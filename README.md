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

En producción **no se publica ningún puerto al host** — `compose.prod.yml` deja `glitchtip-web` alcanzable solo vía la red Docker `glitchtip-network`, para que un reverse proxy central (nginx u otro) se una a esa red y la resuelva por nombre de contenedor (`glitchtip-web:8000`), en vez de necesitar loopback.

1. `cp .env.prod.example .env.prod` y completar todo (acá no hay defaults, `compose.prod.yml` exige explícitamente `SECRET_KEY`, `GLITCHTIP_DOMAIN` y el resto):
   - `SECRET_KEY`: `openssl rand -hex 32`
   - `GLITCHTIP_DOMAIN`: URL pública completa, con esquema (`https://glitchtip.tu-dominio.com`)
   - `EMAIL_URL`: SMTP real (`smtp://usuario:password@host:puerto`) — sin esto no salen los mails de invitación/reset de contraseña
   - `DEFAULT_FROM_EMAIL`

2. Levantar:
   ```bash
   docker compose -f compose.prod.yml --env-file .env.prod up -d
   ```

3. En el compose del reverse proxy central, declarar `glitchtip-network` como red externa (`external: true`, mismo `name: glitchtip-network` que define este stack) y sumarla a la lista de `networks` del servicio de nginx. Agregar el `upstream`/`location` correspondiente apuntando a `glitchtip-web:8000`.

   - **Con subdominio propio** (recomendado si podés): `location / { proxy_pass http://glitchtip_web; ... }` en un `server` nuevo para ese subdominio. No hace falta `BASE_PATH`.
   - **Bajo un path del dominio central** (si no controlás el DNS/dominio y solo podés sumar un `location` al `server` existente): ver sección "Subpath" abajo.

### Subpath (ej. `https://tu-dominio.com/glitchtip/`)

GlitchTip corre sobre Django, que separa el `PATH_INFO` (lo que nginx reenvía) del `SCRIPT_NAME` (el prefijo que Django antepone al generar URLs absolutas). Para que ambas partes coincidan:

1. nginx tiene que sacar el prefijo antes de reenviar — la barra final en `proxy_pass` es lo que hace ese strip:
   ```nginx
   location /glitchtip/ {
       proxy_pass http://glitchtip_web/;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
       client_max_body_size 40M;
   }
   ```
2. GlitchTip necesita saber ese mismo prefijo para reconstruirlo en los links que genera (`BASE_PATH`, sin barra final):
   ```env
   GLITCHTIP_DOMAIN=https://tu-dominio.com
   BASE_PATH=/glitchtip
   CSRF_TRUSTED_ORIGINS=https://tu-dominio.com
   ```
   `GLITCHTIP_DOMAIN` va solo con protocolo+host — el path lo agrega `BASE_PATH` aparte.

## Variables de entorno

| Variable | Dev | Prod | Descripción |
|----------|-----|------|-------------|
| `POSTGRES_USER` | requerida | requerida | Usuario de Postgres |
| `POSTGRES_PASSWORD` | requerida | requerida | Contraseña de Postgres |
| `POSTGRES_DB` | requerida | requerida | Nombre de la base |
| `SECRET_KEY` | default de dev | **requerida**, sin default | Clave de Django — `openssl rand -hex 32` |
| `GLITCHTIP_DOMAIN` | default `http://localhost:8000` | **requerida**, sin default | URL pública completa, con esquema |
| `EMAIL_URL` | default `consolemail://` (los mails se imprimen en los logs, no hace falta SMTP real) | requerida (SMTP real) | Formato `smtp://usuario:password@host:puerto` |
| `DEFAULT_FROM_EMAIL` | default `dev@localhost` | requerida | Remitente de los mails que envía GlitchTip |
| `BASE_PATH` | no usado | opcional | Solo si corre bajo subpath (ej. `/glitchtip`, sin barra final) — ver sección "Subpath" |
| `CSRF_TRUSTED_ORIGINS` | no usado | requerida si usás `BASE_PATH` | Origen completo con esquema (ej. `https://tu-dominio.com`) |

## Troubleshooting

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `glitchtip-postgres` no pasa el healthcheck / no arranca | Desde `postgres:18` la imagen oficial espera el volumen montado en `/var/lib/postgresql` (sin `/data` al final) — layout nuevo compatible con `pg_ctlcluster`. Un mount en `/var/lib/postgresql/data` no arranca. | Ya está corregido en ambos compose de este repo. Si lo cambiaste, revertí el mount y `docker compose down -v && docker compose up -d` (perdés los datos del volumen, solo aceptable si todavía no tenés nada real cargado). |
| El proyecto consumidor no reporta nada aunque el DSN esté bien | Está usando `localhost` desde un contenedor Docker, o `host.docker.internal` desde el browser | Revisar la nota del paso 4 de "Desarrollo local" — el host del DSN depende de dónde corre el código que llama a `Sentry.init`, no de dónde corre GlitchTip. |

## Estructura

```
.
├── compose.yml            # Desarrollo — publica el puerto 8000 al host
├── compose.prod.yml       # Producción — sin puertos, solo red Docker
├── .env.example            # Plantilla de variables (dev)
└── .env.prod.example       # Plantilla de variables (prod)
```
