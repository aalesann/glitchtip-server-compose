# GlitchTip server compose

Stack Docker para levantar [GlitchTip](https://glitchtip.com) (error tracking self-hosted, compatible con el protocolo de Sentry) como servicio propio, reutilizable por cualquier proyecto que use un SDK de Sentry (`@sentry/node`, `@sentry/nestjs`, `@sentry/react`, etc.) apuntando el DSN acá.

Imagen oficial `glitchtip/glitchtip:6` con `SERVER_ROLE: all_in_one` (web + worker + migraciones en un solo contenedor), Postgres 18 y Valkey como dependencias.

## Requisitos

- Docker y Docker Compose v2

## Desarrollo local

1. Crear el `.env`:
   ```bash
   cp .env.example .env
   ```
   Completar `POSTGRES_PASSWORD` (el resto de las variables ya tienen default de desarrollo en `compose.yml`).

2. Levantar:
   ```bash
   docker compose up -d
   ```
   Esto publica GlitchTip en `http://localhost:8000`.

3. Entrar a `http://localhost:8000` y crear el primer usuario — con `ENABLE_ADMIN: "True"` (default de dev) ese usuario queda como admin sin pasos extra.

4. Crear un proyecto por cada aplicación/componente que vaya a reportar errores acá (por ejemplo `sgp-backend` y `sgp-frontend`). Cada proyecto expone un DSN (`http://<public_key>@host:puerto/<project_id>`) para pegar en la app consumidora.

> Si una app consumidora corre dentro de otro contenedor Docker (no en el browser), el DSN necesita `host.docker.internal` en vez de `localhost` como host — GlitchTip corre publicado en el host, no en la red interna de esa otra app.

## Producción

En producción, `glitchtip-web` es alcanzable principalmente vía la red Docker `glitchtip-network` — un reverse proxy central (nginx u otro) se une a esa red y lo resuelve por nombre de contenedor (`glitchtip-web:8000`). Además se publica en `127.0.0.1:8000` (solo loopback, no accesible desde fuera del host) para clientes que no son contenedores — ver "Clientes nativos (PM2, procesos sin Docker)" más abajo.

1. `cp .env.prod.example .env.prod` y completar todo (acá no hay defaults, `compose.prod.yml` exige explícitamente `SECRET_KEY`, `GLITCHTIP_DOMAIN` y el resto):
   - `SECRET_KEY`: `openssl rand -hex 32`
   - `GLITCHTIP_DOMAIN`: URL pública completa, con esquema (`https://glitchtip.tu-dominio.com`)
   - `EMAIL_URL`: SMTP real (`smtp+tls://usuario:password@host:587`) — sin esto no salen los mails de invitación/reset de contraseña.
     El esquema **debe** ser `smtp+tls://` en el 587 (o `smtp+ssl://` en el 465): con `smtp://` a secas Django no negocia
     STARTTLS y proveedores como AWS SES rechazan el login con `530 Must issue a STARTTLS command first`
   - `DEFAULT_FROM_EMAIL`

2. Levantar:
   ```bash
   docker compose -f compose.prod.yml --env-file .env.prod up -d
   ```

3. En el compose del reverse proxy central, declarar `glitchtip-network` como red externa (`external: true`, mismo `name: glitchtip-network` que define este stack) y sumarla a la lista de `networks` del servicio de nginx. Agregar el `upstream`/`location` correspondiente apuntando a `glitchtip-web:8000`.

   - **Con subdominio propio** (recomendado si podés): `location / { proxy_pass http://glitchtip_web; ... }` en un `server` nuevo para ese subdominio. No hace falta `BASE_PATH`.
   - **Bajo un path del dominio central** (si no controlás el DNS/dominio y solo podés sumar un `location` al `server` existente): ver sección "Subpath" abajo.

### Subpath (ej. `https://tu-dominio.com/glitchtip/`)

`BASE_PATH` **solo no alcanza**. Hacen falta cuatro piezas juntas:

1. nginx saca el prefijo antes de reenviar (la barra final en `proxy_pass`):
   ```nginx
   location /glitchtip/ {
       proxy_pass http://glitchtip_web/;
       proxy_set_header Host $host;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       client_max_body_size 40M;
   }
   ```
2. El dominio incluye el subpath — al contrario de lo que dice la doc oficial:
   ```env
   GLITCHTIP_DOMAIN=https://tu-dominio.com/glitchtip
   BASE_PATH=/glitchtip
   CSRF_TRUSTED_ORIGINS=https://tu-dominio.com
   ```
3. Una regla a nivel `server` que colapsa el prefijo duplicado:
   ```nginx
   rewrite ^/glitchtip/glitchtip/(.*)$ /glitchtip/$1 last;
   ```
4. Dos reglas más para las rutas que allauth emite por correo:
   ```nginx
   rewrite ^/(reset-password(?:/.*)?)$   /glitchtip/$1 last;
   rewrite ^/(profile/confirm-email/.*)$ /glitchtip/$1 last;
   ```

Sin el paso 2 quedan rotos los DSN de cada proyecto nuevo y los enlaces de todos
los mails (invitación, alertas, uptime). Sin el paso 3, los endpoints que usan
`reverse()` duplican el prefijo. Sin el paso 4, el reseteo de contraseña y la
confirmación de email dan 404.

**El detalle completo, con la causa de cada defecto y cómo reproducirlo, está en
[docs/despliegue-subpath.md](docs/despliegue-subpath.md).**

### Clientes nativos (PM2, procesos sin Docker)

Una app que corre como proceso nativo en el **mismo host** que este stack (ej. con PM2, sin contenedor) no puede resolver `glitchtip-web` por DNS de Docker ni unirse a `glitchtip-network`. Tampoco debería usar el dominio público: si el host resuelve su propio dominio hacia afuera, muchos proveedores no permiten el *hairpin NAT* (el tráfico sale a internet y no puede volver a entrar al mismo host — se ve como timeout/socket hang up).

La solución es el puerto publicado en loopback (`127.0.0.1:8000`, ver arriba). El DSN para estos clientes:

```
SENTRY_DSN=http://<public_key>@127.0.0.1:8000/<project_id>
```

Sin `/glitchtip` (no pasa por nginx) y sin TLS (loopback, no hace falta).

## Variables de entorno

| Variable | Dev | Prod | Descripción |
|----------|-----|------|-------------|
| `POSTGRES_USER` | requerida | requerida | Usuario de Postgres |
| `POSTGRES_PASSWORD` | requerida | requerida | Contraseña de Postgres |
| `POSTGRES_DB` | requerida | requerida | Nombre de la base |
| `SECRET_KEY` | default de dev | **requerida**, sin default | Clave de Django — `openssl rand -hex 32` |
| `GLITCHTIP_DOMAIN` | default `http://localhost:8000` | **requerida**, sin default | URL pública completa, con esquema |
| `ALLOWED_HOSTS` | default `*` | recomendada | Hosts aceptados, separados por coma. Sin esto queda en `*` y Django avisa en cada arranque. Incluir también los hosts internos desde los que se publican eventos (ej. `glitchtip-web`, `127.0.0.1`) |
| `EMAIL_URL` | default `consolemail://` (los mails se imprimen en los logs, no hace falta SMTP real) | requerida (SMTP real) | Formato `smtp+tls://usuario:password@host:587` — con `smtp://` no hay STARTTLS y SES devuelve `530` |
| `DEFAULT_FROM_EMAIL` | default `dev@localhost` | requerida | Remitente de los mails que envía GlitchTip |
| `BASE_PATH` | no usado | opcional | Solo si corre bajo subpath (ej. `/glitchtip`, sin barra final). Requiere además que `GLITCHTIP_DOMAIN` incluya el path y una regla `rewrite` en nginx — ver [docs/despliegue-subpath.md](docs/despliegue-subpath.md) |
| `CSRF_TRUSTED_ORIGINS` | no usado | requerida si usás `BASE_PATH` | Origen completo con esquema (ej. `https://tu-dominio.com`) |
| `ENABLE_USER_REGISTRATION` | default `True` | `False` si la instancia da a internet | Con `False` nadie puede registrarse solo; se suma gente por invitación, y solo a emails que ya tengan cuenta |
| `ENABLE_ORGANIZATION_CREATION` | default `False` | opcional | Permite que cada usuario cree su propia organización. La primera organización del servidor siempre se puede crear, aunque esté en `False` |

## Troubleshooting

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `glitchtip-postgres` no pasa el healthcheck / no arranca | Desde `postgres:18` la imagen oficial espera el volumen montado en `/var/lib/postgresql` (sin `/data` al final) — layout nuevo compatible con `pg_ctlcluster`. Un mount en `/var/lib/postgresql/data` no arranca. | Ya está corregido en ambos compose de este repo. Si lo cambiaste, revertí el mount y `docker compose down -v && docker compose up -d` (perdés los datos del volumen, solo aceptable si todavía no tenés nada real cargado). |
| El proyecto consumidor no reporta nada aunque el DSN esté bien | Está usando `localhost` desde un contenedor Docker, o `host.docker.internal` desde el browser | Revisar la nota del paso 4 de "Desarrollo local" — el host del DSN depende de dónde corre el código que llama a `Sentry.init`, no de dónde corre GlitchTip. |
| No llega ningún mail (invitaciones, reset de contraseña) y no hay error visible en la UI | `EMAIL_URL` con esquema `smtp://`: Django deja `EMAIL_USE_TLS=False`, no negocia STARTTLS y el servidor corta el login. Con AWS SES el error es `530 Must issue a STARTTLS command first`. | Usar `smtp+tls://` (puerto 587) o `smtp+ssl://` (465). Probar el envío real con:<br>`docker exec glitchtip-web python -c "import django,os;os.environ.setdefault('DJANGO_SETTINGS_MODULE','glitchtip.settings');django.setup();from django.core.mail import send_mail;print(send_mail('test','test',None,['vos@dominio.com']))"` |
| Al crear una organización la UI muestra `[object Object]` y la API devuelve `403 Organization creation is not open` | `ENABLE_ORGANIZATION_CREATION` está en `False` (su default). Solo la **primera** organización del servidor es libre; después únicamente un superusuario puede crear más. | Poner `ENABLE_ORGANIZATION_CREATION=True` si querés que cada usuario arme la suya, o marcar superusuario a la cuenta administradora. El `[object Object]` es el frontend que no sabe renderizar el cuerpo del 403. |
| Una variable del `.env` no llega al contenedor y no se entiende por qué | Una variable exportada en el shell (ej. `SECRET_KEY` en `~/.bashrc`) tiene **precedencia** sobre `--env-file` en Docker Compose, y lo pisa en silencio. | Verificar siempre el valor efectivo con `docker exec glitchtip-web printenv NOMBRE_VAR`, o revisar `docker compose -f compose.prod.yml --env-file .env.prod config`. |
| Todos los clientes reciben `400 Bad Request` después de fijar `ALLOWED_HOSTS` | Falta en la lista algún host desde el que se publican eventos. Django compara contra el header `Host`, que no es el dominio público cuando el cliente entra por la red interna de Docker o por loopback. | Incluir también el nombre del servicio (`glitchtip-web`) y `127.0.0.1`/`localhost` además del dominio público. Django ignora el puerto al comparar. |
| El enlace de invitación que llega por mail da 404, o el DSN de un proyecto nuevo no recibe eventos | Despliegue bajo subpath con `BASE_PATH` pero `GLITCHTIP_DOMAIN` sin el path. | Ver [docs/despliegue-subpath.md](docs/despliegue-subpath.md). Una invitación ya emitida se rescata agregándole el prefijo al enlace a mano. |

## Estructura

```
.
├── compose.yml            # Desarrollo — publica el puerto 8000 al host
├── compose.prod.yml       # Producción — sin puertos, solo red Docker
├── .env.example           # Plantilla de variables (dev)
├── .env.prod.example      # Plantilla de variables (prod)
└── docs/
    └── despliegue-subpath.md   # Correr bajo un path del dominio central
```
