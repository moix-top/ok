# Plan: `ok` · ok.ru sin distracciones (ok.moix.top)

> Estado (2026-10-22): **solo plan, sin código**. Pendiente de retomar empezando por la Fase 0.

La idea es replicar `moix-top/yt` (yt.moix.top), pero con la plataforma rusa **ok.ru**
(Odnoklassniki) en lugar de YouTube: tres cosas y nada más, **por ver**, **vistos** y el
**segundo exacto** donde dejaste cada vídeo, sincronizadas entre la web y el móvil.

## Cómo es `yt` (referencia: `/LINUXDATA/repos/github.com/moix-top/yt`)

- `web/`: Next.js 14 + Auth.js (Google) + DynamoDB, desplegada en Vercel (*Root Directory* = `web`).
  - Una sola API: `POST /api/sync` (`web/lib/types.ts`). El cliente manda `{ since, changes[] }`
    y recibe `{ now, videos[] }` con todo lo cambiado desde su cursor.
  - Modelo `Video`: `videoId, title, channel, durationSec, positionSec/positionAt, queued/queuedAt,
    watched/watchedAt, updatedAt`. Cada uno de los tres campos lleva su propio reloj y se fusiona por
    separado (gana el último, `lib/merge.ts`). `updatedAt` lo pone el servidor y hace de cursor.
  - `lib/services/videoService.ts`: la pk siempre se calcula en el servidor, nunca hay Scan y los
    ítems se construyen campo a campo. Tokens de dispositivo `yt_…`: solo se guarda su SHA-256, hay un
    máximo de 10 y se revocan por un prefijo de 12 caracteres.
  - `/api/meta`: título y canal vía oEmbed de YouTube, desde el servidor (oEmbed no manda CORS).
  - `watch/[id]`: IFrame API de YouTube (`youtube-nocookie`, `rel=0`). Guarda cada 5 s, al pausar
    y al salir. Al terminar marca visto, lo saca de "por ver", pone la posición a 0 y tapa la
    pantalla final.
  - Es una PWA instalable y aparece en *Compartir* de Android (`app/share`). `SpatialNav` sirve
    para la navegación con mando o teclado.
  - Dev: `DEV_LOGIN_EMAIL=yo@example.com AUTH_SECRET=x npm run dev`. Tests con vitest (`tests/sync.test.ts`).
- `android/`: fork de NewPipe = `overlay/` (paquete `ytsync`: `SyncEngine`, `SyncPrefs`,
  `SyncSettingsFragment`, `SyncWorker`, `YtSync`) + `patches/` (3 ganchos, unas 11 líneas). Se
  sincroniza al abrir y cerrar la app y cada 15 min, con el token pegado en los ajustes.
- `.github/workflows/`: `web-ci.yml`, `deploy-infra.yml` (DynamoDB) y `android.yml`. Este último
  comprueba cada día si hay nueva release de NewPipe; si la hay, aplica parches, compila, firma y
  publica una GitHub Release que Obtainium instala sola. Si algo falla, GLM (`GLM_API_KEY`) propone
  el arreglo como PR.

## Qué se puede copiar de `yt` y qué no

| Pieza de `yt` | En `ok` |
|---|---|
| `web/` (Auth.js, DynamoDB, tokens, `merge.ts`, `useLibrary`, PWA, `share/`, `SpatialNav`) | Se copia casi entero |
| Infra (`infra/dynamodb.yml`, `setup-aws.sh`, `setup-github-secrets.sh`, workflows web/infra) | Se copia y se renombra (tabla `ok`, token `ok_`) |
| `/api/meta` (oEmbed de YouTube) | Se reescribe para ok.ru |
| `watch/[id]` (IFrame API de YouTube) | Se reescribe. Es el punto más delicado |
| `android/` (fork de NewPipe) | No sirve: NewPipe no soporta ok.ru |

## Fase 0: pruebas previas (deciden todo lo demás)

1. **Reproductor web.** El embed `https://ok.ru/videoembed/<id>` no tiene una API JS documentada como
   la de YouTube. Hay que comprobar dos cosas:
   - si acepta un parámetro de segundo de inicio;
   - si emite `postMessage` con el tiempo actual y el evento de fin.

   Si no hace ninguna de las dos, el plan B es extraer en el servidor las URLs MP4/HLS (como hace
   `yt-dlp`, `odnoklassniki.py`: atributo `data-options` → `flashvars.metadata` → `videos[]` y
   `hlsManifestUrl`) y reproducirlas en un `<video>`/hls.js propio, con control total de `currentTime`.
   - Riesgo del plan B: si las URLs van firmadas con la IP de quien las pide (`srcIp`), las sacadas
     desde Vercel no funcionarían en el navegador. Hay que probarlo.
2. **Metadatos.** Hay que comprobar si funciona `https://connect.ok.ru/oembed?url=…` (título, autor y
   duración). Si no, el plan B es leer `og:title` y el JSON del embed.
3. **Ids y enlaces.** Formatos que habría que aceptar: `ok.ru/video/<num>`, `m.ok.ru/video/<num>`,
   `ok.ru/videoembed/<num>` y los compartidos desde la app de OK.
4. **Bloqueos.** Hay que mirar si ok.ru restringe por región o exige login para algunos vídeos (desde
   España y desde los servidores de Vercel).

## Fase 1: la web

1. Copiar `yt/web` a `ok/web`, cambiar el dominio, los textos y el prefijo de token `yt_` → `ok_`, y
   limpiar referencias a YouTube.
2. `lib/merge.ts`: `parseVideoId`/`isVideoId` para ids numéricos de ok.ru.
3. `lib/types.ts`: el modelo `Video` no cambia. Dejar `videoId` como string numérico.
4. `/api/meta`: oEmbed de ok.ru o, si falla, scraping en el servidor.
5. `watch/[id]`: según lo que salga en la Fase 0:
   - **A)** iframe de ok.ru con inicio y postMessage, con la misma lógica de guardado que `yt`.
   - **B)** `<video>` propio con una ruta `/api/stream/[id]` que solo resuelve las URLs, sin hacer
     de proxy del vídeo.

   Si no hay forma de saber el tiempo actual, el plan C es guardar solo "por ver"/"visto" y una
   posición manual. Solo como último recurso.
6. Tapar la pantalla final y las recomendaciones.
7. Tests: copiar `tests/sync.test.ts` y añadir tests del parser de enlaces de ok.ru.

## Fase 2: el móvil

1. **Solo la PWA** (ya aparece en *Compartir* de Android). No cuesta nada y puede bastar. **Recomendado
   para empezar.**
2. **App Kotlin mínima propia** (`android/`), si la PWA se queda corta (por ejemplo, para la
   reproducción en segundo plano):
   - Media3/ExoPlayer y el extractor de ok.ru portado de `yt-dlp`.
   - El `SyncEngine`/`SyncWorker`/`SyncPrefs` de `yt` reutilizados casi tal cual.
   - Sin Google Play Services, con token `ok_…`, GitHub Release y Obtainium.
   - El ciclo diario "¿nueva versión upstream?" desaparece. En su lugar, un workflow semanal que
     compruebe que el extractor sigue funcionando y abra una incidencia si se rompe.

## Fase 3: puesta en marcha

- Google OAuth con callback `https://ok.moix.top/api/auth/callback/google`.
- Vercel con *Root Directory* = `web`.
- AWS (perfil `radamuz`):
  `./scripts/setup-github-secrets.sh --profile radamuz --repo moix-top/ok` → workflow
  "Deploy infra (DynamoDB)" → `./scripts/setup-aws.sh --profile radamuz`.
- Firma del APK (`android/scripts/setup-signing.sh --repo moix-top/ok`), solo si se hace la Fase 2.2.

## Orden propuesto

Fase 0 (prueba de media hora en el navegador) → copiar y renombrar la web → `meta` + `watch` →
desplegar → decidir sobre el móvil.
