# Despliegue bajo subpath

Cómo correr GlitchTip en un path del dominio central (ej.
`https://tu-dominio.com/glitchtip/`) en vez de un subdominio propio.

> **Si podés usar un subdominio, usalo.** El soporte de subpath de GlitchTip es
> parcial y esta guía existe para rodear cuatro defectos distintos. Con subdominio
> propio nada de esto hace falta: `GLITCHTIP_DOMAIN=https://glitchtip.tu-dominio.com`,
> sin `BASE_PATH`, y listo.

## Contexto

GlitchTip corre sobre Django, que separa el `PATH_INFO` (lo que nginx reenvía)
del `SCRIPT_NAME` (el prefijo que Django antepone al generar URLs absolutas).
`BASE_PATH` alimenta el segundo. Hacer que ambas partes coincidan requiere tres
piezas, no una.

## Configuración

### 1. nginx saca el prefijo antes de reenviar

La barra final en `proxy_pass` es lo que hace el strip:

```nginx
location /glitchtip/ {
    proxy_pass http://glitchtip_web/;   # <- la barra final saca /glitchtip/
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    client_max_body_size 40M;
}
```

Django recibe `/accept/5/token/` y resuelve contra su urlconf normalmente,
porque `FORCE_SCRIPT_NAME` solo afecta a la *generación* de URLs, no al ruteo.

### 2. El dominio tiene que incluir el subpath

```env
GLITCHTIP_DOMAIN=https://tu-dominio.com/glitchtip
BASE_PATH=/glitchtip
CSRF_TRUSTED_ORIGINS=https://tu-dominio.com
```

`CSRF_TRUSTED_ORIGINS` va solo con protocolo+host: Django rechaza orígenes con
path.

> **Esto contradice la documentación oficial**, que dice que `GLITCHTIP_DOMAIN`
> debe llevar solo esquema+host ("Set to your domain. Include scheme (http or
> https)."). Seguir esa guía al pie con `BASE_PATH` produce DSN y enlaces de
> correo rotos. La doc oficial no cubre la interacción entre ambas variables.

### 3. Colapsar el prefijo duplicado en nginx

```nginx
rewrite ^/glitchtip/glitchtip/(.*)$ /glitchtip/$1 last;
```

Va a nivel de `server`, fuera de los `location`.

Usar `last` y **no** `permanent`: un 301 se arma con `$scheme`, que vale `http`
cuando el TLS se termina aguas arriba (nginx escuchando en el 80). El redirect
mandaría al usuario de https a http. Con `last` se resuelve internamente, sin
round-trip ni esquema de por medio.

### 4. Reencaminar las rutas de allauth

```nginx
rewrite ^/(reset-password(?:/.*)?)$      /glitchtip/$1 last;
rewrite ^/(profile/confirm-email/.*)$    /glitchtip/$1 last;
```

También a nivel de `server`. Son las rutas de `HEADLESS_FRONTEND_URLS` que
allauth emite por correo (reseteo de contraseña, confirmación de email) y que
salen sin el prefijo — ver el defecto (d).

## Por qué hace falta todo esto

Son cuatro defectos independientes. Ninguno está documentado upstream.

### a) Los mails salen sin el prefijo

`django.setup()` fija el prefijo con `set_script_prefix()`, y su propio docstring
lo aclara: *"Set the **thread-local** urlresolvers script prefix"*. El
almacenamiento es `_prefixes = Local()` (un `asgiref.local.Local`), o sea por
hilo.

Los hilos que atienden requests lo tienen seteado. El worker que renderiza los
mails corre en otro hilo, donde `get_script_prefix()` cae al default `/`. La
plantilla de invitación usa `{% get_domain %}{% url 'invitations_register' %}`,
así que el `{% url %}` genera `/accept/<id>/<token>/` sin prefijo.

Se reproduce en el mismo proceso:

```bash
docker exec glitchtip-web python -c "
import django, os, threading
os.environ.setdefault('DJANGO_SETTINGS_MODULE','glitchtip.settings')
django.setup()
from django.urls import reverse, get_script_prefix

def probe(label):
    print(f'{label:20} prefix={get_script_prefix()!r:14} '
          f'url={reverse(\"invitations_register\", args=[1,\"abc-def\"])}')

probe('hilo principal')
t = threading.Thread(target=probe, args=('hilo background',)); t.start(); t.join()
"
```

```
hilo principal       prefix='/glitchtip/'  url=/glitchtip/accept/1/abc-def/
hilo background      prefix='/'            url=/accept/1/abc-def/
```

Poner el path en `GLITCHTIP_DOMAIN` compensa exactamente esta pérdida: el
`{% get_domain %}` aporta el prefijo que el `{% url %}` no puso.

### b) Alertas y uptime nunca contemplaron BASE_PATH

`apps/alerts/email.py` y `apps/uptime/email.py` arman los enlaces por
concatenación directa:

```python
base_url = settings.GLITCHTIP_URL.geturl()
issue_link = f"{base_url}/{org_slug}/issues/{first_issue.id}"
```

No pasan por `reverse()`, así que `BASE_PATH` no los toca. Solo se arreglan si
el prefijo ya viene dentro de `GLITCHTIP_URL`.

### c) El DSN espera el path en el dominio

`ProjectKey.get_dsn()` en `apps/projects/models.py`:

```python
return "%s://%s@%s/%s" % (
    urlparts.scheme,
    self.public_key_hex,
    urlparts.netloc + urlparts.path,   # <- concatena el path explícitamente
    self.project_id,
)
```

Es la evidencia más fuerte de que el diseño *espera* el subpath en
`GLITCHTIP_DOMAIN`. Sin él, cada proyecto nuevo muestra en la UI un DSN que no
recibe eventos y hay que corregirlo a mano.

### d) Los enlaces de allauth no pasan por reverse() ni por el dominio

El reseteo de contraseña y la confirmación de email los emite `allauth`, que
arma la URL con `request.build_absolute_uri()` sobre las rutas de
`HEADLESS_FRONTEND_URLS` (en `glitchtip/settings.py`):

```python
HEADLESS_FRONTEND_URLS = {
    "account_reset_password": "/reset-password",
    "account_confirm_email": "/profile/confirm-email/{key}/",
    "account_reset_password_from_key": "/reset-password/set-new-password/{key}",
    ...
}
```

Son rutas absolutas desde la raíz. `build_absolute_uri()` no antepone el
`SCRIPT_NAME`, y como no pasan por `reverse()` ni por `GLITCHTIP_URL`, ni
`BASE_PATH` ni el path del dominio las alcanzan. El enlace sale sin prefijo y
da 404. Por eso el paso 4.

**Advertencia sobre el esquema.** Esa misma llamada toma el esquema del request.
`SECURE_PROXY_SSL_HEADER` no está definido en GlitchTip, así que Django ignora
el header `X-Forwarded-Proto` y ve `http` cuando el TLS se termina aguas arriba.
El enlace del correo sale entonces como `http://`. Eso **no** es
necesariamente un problema: si el terminador TLS redirige a https por su cuenta,
el flujo funciona igual (verificado en la práctica: un reseteo de contraseña
completo, de punta a punta, sobre un despliegue con TLS terminado aguas arriba).

Conviene confirmarlo en tu propio despliegue desde un navegador externo, porque
desde el host del servidor el dominio no suele ser alcanzable (ver "Clientes
nativos" en el README: hairpin NAT). Si tu terminador **no** redirige, no hay
variable de entorno para corregirlo: habría que definir
`SECURE_PROXY_SSL_HEADER` en los settings de Django.

## El efecto secundario, y por qué el rewrite

Meter el path en el dominio arregla (a), (b) y (c), pero rompe los endpoints que
*además* usan `reverse()`: ahí el prefijo se antepone dos veces.

| Endpoint | Patrón |
|---|---|
| `invite_link` (UI de miembros) | `apps/organizations_ext/schema.py` |
| `heartbeat_endpoint` (uptime) | `apps/uptime/schema.py` |
| Subida de sourcemaps | `apps/files/api.py` |
| Embed de feedback | `apps/event_ingest/embed_api.py` |

Todos comparten la firma `GLITCHTIP_URL.geturl() + reverse(...)` y producen
`/glitchtip/glitchtip/...`. El `rewrite` del paso 3 los colapsa. Esa ruta no es
legítima en ningún caso, así que la regla no tiene falsos positivos.

## Verificación

Después de aplicar los cuatro pasos:

```bash
# El DSN debe incluir el subpath
docker exec glitchtip-web python -c "
import django, os
os.environ.setdefault('DJANGO_SETTINGS_MODULE','glitchtip.settings')
django.setup()
from apps.projects.models import ProjectKey
print(ProjectKey.objects.first().get_dsn())
"

# La URL duplicada debe dar 200 y NO un redirect
curl -s -o /dev/null -w '%{http_code} redirect=%{redirect_url}\n' \
  -H 'Host: tu-dominio.com' \
  'http://127.0.0.1/glitchtip/glitchtip/api/settings/'

# Las rutas de allauth deben dar 200 (antes del paso 4 daban 404)
for p in /reset-password \
         /reset-password/set-new-password/XXX \
         /profile/confirm-email/XXX/ ; do
  printf '%-40s -> ' "$p"
  curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: tu-dominio.com' "http://127.0.0.1$p"
done
```

## Rescatar una invitación ya emitida

Si alguien recibió el mail con el enlace roto, no hace falta reinvitarlo: el
token sigue siendo válido (15 días, `REGISTRATION_TIMEOUT_DAYS`). Alcanza con
agregarle el prefijo a mano:

```
https://tu-dominio.com/accept/5/TOKEN/            <- lo que llegó (404)
https://tu-dominio.com/glitchtip/accept/5/TOKEN/  <- el que funciona
```

## Referencias

- [Install — Documentation | GlitchTip](https://glitchtip.com/documentation/install/)
- [GlitchTip Backend — issues](https://gitlab.com/glitchtip/glitchtip-backend/-/issues)

Verificado contra GlitchTip 6.2.0 / Django 6.0.4.
