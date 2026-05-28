# FUTURA — Landing

Landing pública de FUTURA — agentes de IA para clínicas. Sirve la página y dispara llamadas demo contra el backend en `app.futuradigital.es`.

## Stack

- `index.html` — landing single-file (HTML + CSS + JS inline). Carga fuentes desde Google Fonts y el video VSL desde `cliniq.futuradigital.es`.
- `nginx.conf` — config nginx con gzip, caché de assets, `no-store` para `index.html`, `/healthz`.
- `Dockerfile` — `nginx:alpine` + healthcheck.

## Deploy en Dokploy

1. **Create Service → Application** (Git).
2. Repo: este. Branch: `main`. Build Path: `/` (raíz).
3. Build Type: **Dockerfile** → `./Dockerfile`.
4. **Domains**: agregar el dominio público (ej. `cliniq.futuradigital.es`) con cert Let's Encrypt → Container Port `80`.
5. **Auto Deploy**: activado sobre push a `main`.

Tarda ~1 minuto el redeploy. El healthcheck `/healthz` lo expone Dokploy automáticamente.

## CORS del backend

La landing hace `POST https://app.futuradigital.es/api/public/demo-call`. Para que el browser no la bloquee, el dominio donde sirvas esta landing **tiene que estar listado** en la env var `FUTURA_DEMO_ALLOWED_ORIGINS` del service `cliniq-web` (separados por comas).

## Cambiar el endpoint del API

Si querés apuntar la landing a otro backend (staging, local), editá la constante al principio del `<script>` en `index.html`:

```js
const FUTURA_API_BASE = 'https://app.futuradigital.es';
```

## Notas del form "Recibir llamada"

- Normaliza el teléfono a E.164 (acepta `+34611...`, `0034611...`, o 9 dígitos móvil España).
- Rate-limit del lado backend: 60s entre llamadas al mismo número.
- Estado del botón: idle → loading (con spinner) → cooldown 8s (sin spinner) → idle. Guarda `isBusy` previene submits dobles.
