# Cómo generar errores de prueba

Para verificar que un proyecto está bien conectado, o para tener issues con los
que familiarizarse con la interfaz.

La idea es probar en dos niveles: primero **sin SDK**, para aislar si el problema
está en la red o en el código; después **con el SDK**, que es lo que realmente
va a usar la aplicación.

## Antes de empezar: elegir bien el host del DSN

Es el error más común. El host del DSN depende de **dónde corre el código que
llama a `Sentry.init`**, no de dónde corre GlitchTip:

| Dónde corre tu código | Host del DSN |
|---|---|
| En el browser (React, Vue, etc.) | El dominio público |
| En un contenedor de la misma red Docker | `glitchtip-web:8000` |
| Proceso nativo en el mismo host (PM2, `npm run dev`) | `127.0.0.1:8000` |
| En tu máquina, contra el servidor de Testing | El dominio público |

El DSN lo da la propia interfaz: **Settings → Projects → (tu proyecto) → Client
Keys (DSN)**. Copialo de ahí en vez de armarlo a mano.

## Nivel 0: sin SDK, con curl

Sirve para confirmar que el DSN es válido y que hay conectividad, antes de tocar
una sola línea de la aplicación. No necesita instalar nada.

Del DSN `https://LA_CLAVE@tu-dominio.com/glitchtip/7`, sacás dos datos: la clave
(`LA_CLAVE`) y el id de proyecto (`7`, el número final).

```bash
KEY=LA_CLAVE
PROJECT_ID=7
BASE=https://tu-dominio.com/glitchtip

curl -s -w '\nHTTP %{http_code}\n' \
  -H "Content-Type: application/json" \
  -X POST "$BASE/api/$PROJECT_ID/store/?sentry_key=$KEY" \
  -d "{
    \"event_id\": \"$(cat /proc/sys/kernel/random/uuid | tr -d '-')\",
    \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
    \"platform\": \"other\",
    \"level\": \"error\",
    \"logentry\": {\"formatted\": \"Evento de prueba desde curl\"}
  }"
```

Respuesta esperada:

```
{"event_id": "9b115e781b8848e98097f109536ee69a", "task_id": null}
HTTP 200
```

El issue aparece en la interfaz en unos segundos (la ingesta es asíncrona). Si
en cambio recibís:

- **HTTP 401** → la clave del DSN está mal.
- **HTTP 404** → el id de proyecto está mal, o falta el subpath en la URL.
- **timeout / no conecta** → problema de red, no de GlitchTip. Revisá la tabla
  de hosts de arriba.

## Con el SDK: Node / Express / NestJS

```bash
npm install @sentry/node
```

`Sentry.init()` tiene que ejecutarse **antes** que el resto de la aplicación,
así que va en la primera línea del entrypoint (o en un archivo `instrument.js`
importado primero).

```js
const Sentry = require("@sentry/node");

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV || "development",
});

// Error de prueba
Sentry.captureException(new Error("Error de prueba desde Node"));
```

Para probarlo suelto, sin levantar la app entera:

```bash
node -e "
const Sentry = require('@sentry/node');
Sentry.init({ dsn: process.env.SENTRY_DSN, environment: 'prueba' });
Sentry.captureException(new Error('Error de prueba desde Node'));
Sentry.flush(8000).then(ok => { console.log('flush ok:', ok); process.exit(0); });
"
```

El `flush()` es importante en scripts cortos: el SDK envía en background, y sin
esperar el envío el proceso termina antes y el evento se pierde. En una
aplicación de larga vida no hace falta.

Un error no capturado también se reporta solo, sin llamar a `captureException`:

```js
app.get("/debug-sentry", () => {
  throw new Error("Error de prueba desde Express");
});
```

> En Express 4, además hay que montar el error handler de Sentry después de las
> rutas. Desde Express 5 y en NestJS el SDK lo resuelve solo. Ver la
> documentación del SDK según tu versión.

## Con el SDK: React

```bash
npm install @sentry/react
```

```js
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,   // o process.env.REACT_APP_SENTRY_DSN
  environment: import.meta.env.MODE,
});
```

Y un botón que rompa a propósito:

```jsx
<button onClick={() => { throw new Error("Error de prueba desde React"); }}>
  Romper
</button>
```

Ojo con dos cosas en el frontend:

- El DSN del browser **es público por diseño** (viaja en el bundle). No es una
  credencial secreta: solo permite escribir eventos, no leerlos.
- Un bloqueador de publicidad puede frenar el envío. Si no aparece nada, probá
  en una ventana de incógnito sin extensiones.

## Qué mirar en la interfaz

El issue aparece en **Issues**, agrupado por tipo de error. Un mismo error que
ocurre 50 veces es **un** issue con 50 eventos, no 50 issues.

Dentro del issue vas a encontrar el stack trace, el `environment`, y los tags.
Los eventos enviados con `captureException(new Error(...))` llegan parseados
como excepción (el título arranca con `Error:`); los mensajes sueltos, como
texto plano.

## Separar entornos

Conviene que los errores de tu entorno local no se mezclen con los del servidor
de Testing. Dos formas, combinables:

- **Proyectos distintos** (DSN distintos): la separación más fuerte. Es el
  esquema que ya usamos — cada quien tiene su organización con los proyectos de
  su entorno local, y los proyectos `test-*` son los del servidor de Testing.
- **El campo `environment`** dentro del mismo proyecto: se filtra desde la
  interfaz. Útil para distinguir `development` de `production` en un mismo
  proyecto.

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| El script termina y no llega nada | Falta `await Sentry.flush()` antes de que el proceso salga |
| Nada llega desde el browser | Bloqueador de publicidad, o el DSN apunta a `localhost` en vez del dominio |
| Nada llega desde un contenedor | El DSN usa `localhost`; desde otro contenedor tiene que ser `glitchtip-web:8000` |
| HTTP 404 al enviar | Falta el subpath (`/glitchtip`) en el DSN, o el id de proyecto es incorrecto |
| Llega el evento pero sin stack trace útil | Falta subir los sourcemaps (frontend compilado) |

## Limpiar los issues de prueba

Desde el issue: **Delete**. O varios a la vez, seleccionándolos desde la lista
de Issues. No afecta a la configuración del proyecto ni al DSN.
