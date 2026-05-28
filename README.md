# FUTURA — Landing

Landing pública de FUTURA — agentes de IA para clínicas. Single-file HTML servido por nginx en Docker, deployable a Dokploy (o cualquier host que entienda `Dockerfile`).

La landing dispara llamadas demo contra el backend Next.js en `https://app.futuradigital.es` (endpoint público `/api/public/demo-call`).

---

## Estructura del repo

| Archivo | Qué hace |
|---|---|
| `index.html` | Landing completa (HTML + CSS + JS inline). VSL embebido + form "Recibir llamada" + iframe del calendario GHL. |
| `nginx.conf` | Config nginx: gzip, caché 30d para assets, `Cache-Control: no-store` para `index.html`, endpoint `/healthz`. |
| `Dockerfile` | `nginx:alpine` + copy + healthcheck que hace `wget /healthz`. |

---

## Deploy en Dokploy — paso a paso

### 0. Prerequisitos

- Acceso al Dokploy panel (`https://vpsdokploy.futuradigital.es` para FUTURA).
- Este repo ya creado en GitHub y la cuenta GitHub conectada a Dokploy (Dokploy → Settings → Git Providers → GitHub App). Si todavía no está, instalá la GitHub App de Dokploy en tu cuenta/org y dale acceso a este repo.
- Acceso DNS del dominio que vas a usar (para apuntarlo al VPS).

### 1. Crear el Service de tipo Application

1. Abrí el proyecto (ej. `Cliniq Production`) → **Create Service → Application**.
2. **NO** elegís Database, ni Compose, ni Application Docker. **Application** a secas (Git source).
3. Datos del service:
   - **Name**: `futura-landing` (o `cliniq-landing`, como prefieras — define el hostname interno del container en la red `dokploy-network`).
   - **Description**: opcional.

### 2. Conectar Git

En el tab **General** (o **Source**, según versión):

| Campo | Valor |
|---|---|
| Source Type | `Git` |
| Provider | `GitHub` (el que conectaste en Settings) |
| Repository | `<tu-usuario>/futura-landing` |
| Branch | `main` |
| Build Path | `/` (raíz del repo — el `Dockerfile` está en root) |

### 3. Build Type = Dockerfile

En **Build**:

| Campo | Valor |
|---|---|
| Build Type | `Dockerfile` |
| Dockerfile Path | `./Dockerfile` |
| Docker Context Path | `.` |
| Docker Build Stage | (vacío — el Dockerfile no usa multi-stage) |

NO necesitás Buildpacks, Nixpacks ni Static — el `Dockerfile` ya hace todo (`nginx:alpine` + COPY + healthcheck).

### 4. Variables de entorno

Ninguna. La landing es 100% estática, no consume secrets. Skipeá el tab Environment.

(Si en el futuro mové `FUTURA_API_BASE` a una env var pasada al build, agregás `ARG` en el Dockerfile y `Build Args` en Dokploy. Por ahora está hardcodeada en `index.html`.)

### 5. Dominio (Traefik + Let's Encrypt)

En **Domains → Add Domain**:

| Campo | Valor |
|---|---|
| Host | `cliniq.futuradigital.es` (o el subdominio que decidas, ej. `landing.futuradigital.es`) |
| Path | `/` |
| Container Port | `80` |
| HTTPS | ✅ |
| Certificate Provider | `letsencrypt` |

**Antes de guardar**: en tu DNS (Hostinger / Cloudflare / donde manejes el dominio):
- Crear/actualizar un `A` record → IP del VPS Dokploy (para FUTURA: `72.60.212.232`).
- Esperá unos minutos a que propague (verificá con `dig +short cliniq.futuradigital.es` o `nslookup`).
- Recién después le das **Save** al dominio en Dokploy — si no, Let's Encrypt va a fallar la validación HTTP-01.

Si el dominio ya estaba en uso por otro hosting (ej. Hostinger viejo), bajá el sitio anterior antes de cortarlo o vas a tener cache raro durante la propagación.

### 6. Auto-deploy on push

En **Deployments → Auto Deploy**:

- ✅ Activar "Auto Deploy on Push"
- **Watch Paths**: dejá vacío. Como el repo SOLO contiene la landing, cualquier push a `main` tiene que redeployar. (Si en algún momento metés `docs/`, `scripts/`, etc., ahí sí filtrás con `index.html`, `Dockerfile`, `nginx.conf`.)

Dokploy genera un webhook URL que se registra automáticamente en GitHub vía la GitHub App.

### 7. Primer deploy

Click **Deploy** en el botón superior derecho. Vas a ver el log en vivo:

1. `Cloning repository...`
2. `docker build -t <imagename> .`
3. `Successfully built ...`
4. `Starting container...`
5. Healthcheck verde a los ~10s.

Toda la build tarda **~30-60s** (es solo `FROM nginx:alpine` + dos `COPY`).

Apenas pase a verde, abrí `https://cliniq.futuradigital.es` en el browser → tenés que ver la landing.

### 8. Verificar CORS del backend

La landing hace `POST https://app.futuradigital.es/api/public/demo-call`. Si el browser bloquea con error de CORS:

1. Abrí Dokploy → **cliniq-web → Environment**.
2. Buscá la env var `FUTURA_DEMO_ALLOWED_ORIGINS`.
3. Confirmá que tu dominio nuevo de la landing está en la lista, separado por comas. Ejemplo:
   ```
   FUTURA_DEMO_ALLOWED_ORIGINS=https://cliniq.futuradigital.es,https://www.cliniq.futuradigital.es
   ```
4. Si lo agregaste o cambiaste, **redeploy de `cliniq-web`** para que tome efecto.

### 9. Smoke test end-to-end

1. Abrir `https://<tu-dominio>` en el browser.
2. Bajar hasta la sección "Pruébalo en vivo".
3. Cargar nombre + teléfono propio.
4. Click "Recibir llamada".
5. Status box verde → el teléfono suena en ~10s.

Si no suena:
- DevTools → Network → mirá el response del `POST /api/public/demo-call`.
- `429 rate_limited` → ya hubo otra llamada al mismo número en los últimos 60s.
- `503 no_phone` o `no_agent` → falta config en el tenant demo (DB).
- `502` con `reason: telephony_provider_permission_denied` → Zadarma/Twilio no tiene habilitado el país destino.

---

## Cambiar el endpoint del API

La constante está hardcodeada al inicio del `<script>` en `index.html`:

```js
const FUTURA_API_BASE = 'https://app.futuradigital.es';
```

Si querés apuntar a staging / local, editá esa línea y commiteá. Auto-deploy de Dokploy se encarga del resto.

---

## Sobre el form "Recibir llamada"

- **Normalización E.164**: acepta `+34611...`, `0034611...`, o 9 dígitos móvil España (le agrega `+34`).
- **Rate-limit backend**: 60s entre llamadas al mismo número (lo enforcea `/api/public/demo-call`).
- **Estados del botón**:
  - `idle` → habilitado, label "Recibir llamada"
  - `loading` → spinner girando, label "Llamando..." (durante el `fetch`)
  - `cooldown` → 8s sin spinner, label "Llamada disparada ✓", botón disabled (evita doble disparo)
  - vuelve a `idle`
- **Guarda `isBusy`**: previene que clicks múltiples encolen requests. Si presionás 3 veces seguidas, solo el primero dispara.

---

## Troubleshooting

| Síntoma | Causa probable | Cómo verificar |
|---|---|---|
| Dokploy: build falla en `COPY` | El `Dockerfile Path` o `Build Path` están mal | Logs del deploy en Dokploy |
| 502 / 503 al abrir el dominio | Container no levantó o Traefik no rutea | Dokploy → Logs del container; debe estar healthy |
| Certificado SSL inválido | Let's Encrypt no validó (DNS no propagado o puerto 80 cerrado) | `curl -v https://<dominio>` y revisar Dokploy → Domain logs |
| Form devuelve "CORS error" | Origin no está en `FUTURA_DEMO_ALLOWED_ORIGINS` | DevTools Network → response headers del OPTIONS |
| Form devuelve 429 | Rate-limit por número | Esperar 60s o usar otro número |
| Cambio en `index.html` no se ve | Browser caché o no se redeployó | Hard refresh (Cmd+Shift+R); confirmar que el último commit en `main` está deployado en Dokploy |

---

## Roll-back

Si un deploy rompe algo:

1. Dokploy → **Deployments** → buscá un deployment anterior verde.
2. Click **Redeploy** sobre ese.

O desde Git:

```bash
git revert <commit-bad>
git push
```

Auto-deploy reaplica el revert en ~1 min.

---

## Ver también

- Backend que recibe las llamadas: el monorepo principal (`cliniq-web` en Dokploy, repo `llamadaSalientes`, carpeta `apps/web`).
- Endpoint que dispara la llamada: `apps/web/app/api/public/demo-call/route.ts`.
- Agente Retell que atiende: `agent_b5ed188b95c62ad43b0c8e2d81` ("FUTURA — Demo Outbound (Sofía)").
