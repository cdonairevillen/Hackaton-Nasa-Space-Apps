# Despliegue

> Cómo publicamos el juego para que los jueces puedan jugarlo desde una URL, sin instalar nada.
> Cualquier cambio en este proceso se anota en `docs/decisions.md`.

---

## 1. ¿Podemos usar una cuenta personal de Google Cloud?

Las normas de participación de Space Apps **no dicen nada** sobre servicios de hosting, nube o cuentas personales. Lo que sí exigen y afecta al despliegue:

- **Demo pública:** el enlace al proyecto y a la demo debe estar en un servicio externo y ser accesible públicamente. La web de Space Apps no aloja proyectos.
- **Licencia abierta:** el código original debe ser libre o tener una licencia OSI. Si no indicamos ninguna, NASA lo distribuye bajo Apache 2.0. Añadimos un `LICENSE` en la raíz.
- **Citar todas las fuentes** usadas, incluidas las de NASA (ver `docs/DATA_SOURCES.md`).
- **Nada de datos personales** en la entrega, y ningún nombre, voz o imagen reconocible de menores de 18 años. Esto encaja con el jugador temporal con nickname generado.

**Conclusión:** usar la cuenta de Google Cloud de un miembro del equipo está permitido. El coste corre a cargo de esa cuenta, así que hay que poner límites (sección 6). Si el evento local tiene normas propias, revisadlas al llegar.

---

## 2. Arquitectura de despliegue

```
                    Navegador del jugador
                             │
                             ▼
              ┌──────────────────────────────┐
              │ Firebase Hosting             │
              │  - frontend/dist (estático)  │
              │  - /api/** ──► Cloud Run     │  (rewrite: mismo dominio, sin CORS)
              └──────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │ Cloud Run: jamt-api          │
              │  FastAPI (contenedor)        │
              │  secretos: Secret Manager    │
              └──────┬───────────────┬───────┘
                     │               │
                     ▼               ▼
        Postgres (Supabase/Neon)   api.nasa.gov (DONKI)
```

| Pieza | Servicio | Motivo |
|---|---|---|
| Frontend | **Firebase Hosting** | Gratis, en la misma cuenta de Google y con rewrite a Cloud Run |
| Backend | **Cloud Run** | Escala a cero, capa gratuita mensual y ya lo conocemos de hourli-fy |
| Base de datos | **Supabase o Neon** (plan gratuito) | Cloud SQL cuesta dinero aunque no se use; un Postgres gestionado gratuito basta para el hackathon |
| Secretos | **Secret Manager** | `DATABASE_URL` y `NASA_API_KEY` fuera del código |

**Por qué el rewrite de Firebase:** el frontend llama a rutas relativas (`/api/...`). En producción, Firebase las redirige a Cloud Run, y en local las redirige el proxy de Vite. Mismo código en los dos entornos y sin problemas de CORS.

Región: `europe-west1`.

---

## 3. Preparación (una sola vez)

Lo hace quien sea titular de la cuenta.

### 3.1 Proyecto de Google Cloud

```bash
gcloud auth login
gcloud projects create jamt-spaceapps-2026 --name="Junior Astronaut"
gcloud config set project jamt-spaceapps-2026
gcloud config set run/region europe-west1

gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com
```

Vincular el proyecto a la cuenta de facturación desde la consola (Facturación → Mis proyectos).

### 3.2 Base de datos

1. Crear un proyecto en Supabase (o Neon) en una región europea.
2. Copiar la cadena de conexión y adaptarla al driver que usemos con SQLAlchemy:
   ```
   postgresql+psycopg://USUARIO:CONTRASEÑA@HOST:5432/postgres
   ```
3. **Supabase con el pooler en modo transacción (puerto 6543):** psycopg necesita desactivar las sentencias preparadas. Si da problemas, usad la conexión directa (puerto 5432), que para el tráfico de un hackathon es suficiente.

### 3.3 Secretos

```bash
printf '%s' 'postgresql+psycopg://...' | gcloud secrets create DATABASE_URL --data-file=-
printf '%s' 'TU_NASA_API_KEY'          | gcloud secrets create NASA_API_KEY --data-file=-
```

Dar acceso de lectura a la cuenta de servicio que usa Cloud Run:

```bash
PROJECT_NUMBER=$(gcloud projects describe jamt-spaceapps-2026 --format='value(projectNumber)')

gcloud projects add-iam-policy-binding jamt-spaceapps-2026 \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

Para actualizar un secreto más adelante:

```bash
printf '%s' 'nuevo-valor' | gcloud secrets versions add DATABASE_URL --data-file=-
```

### 3.4 Firebase

```bash
npm install -g firebase-tools
firebase login
firebase projects:addfirebase jamt-spaceapps-2026
```

`firebase.json` en la raíz del repo:

```json
{
  "hosting": {
    "public": "frontend/dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "/api/**",
        "run": { "serviceId": "jamt-api", "region": "europe-west1" }
      },
      { "source": "**", "destination": "/index.html" }
    ]
  }
}
```

`.firebaserc` en la raíz:

```json
{ "projects": { "default": "jamt-spaceapps-2026" } }
```

---

## 4. Backend en Cloud Run

### 4.1 Requisitos del contenedor

Cloud Run indica el puerto con la variable `PORT` (por defecto 8080). El `Dockerfile` del backend debe escuchar ahí:

```dockerfile
FROM python:3.12-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .

ENV PORT=8080
CMD uv run uvicorn app.main:app --host 0.0.0.0 --port ${PORT}
```

Las rutas de FastAPI llevan el prefijo `/api` (por ejemplo, `/api/health`), para que coincidan con el rewrite de Firebase.

### 4.2 Desplegar

Desde la raíz del repo:

```bash
gcloud run deploy jamt-api \
  --source backend \
  --region europe-west1 \
  --allow-unauthenticated \
  --max-instances 2 \
  --memory 512Mi \
  --set-secrets DATABASE_URL=DATABASE_URL:latest,NASA_API_KEY=NASA_API_KEY:latest
```

`--source` construye la imagen con Cloud Build usando `backend/Dockerfile`. Al terminar devuelve una URL `https://jamt-api-....run.app`.

Comprobación:

```bash
curl https://jamt-api-XXXX.europe-west1.run.app/api/health
```

### 4.3 Migraciones en producción

Las migraciones se ejecutan **desde local** contra la base de datos de producción. No se ejecutan al arrancar el contenedor, para evitar que dos instancias migren a la vez.

```bash
cd backend
DATABASE_URL='postgresql+psycopg://...' uv run alembic upgrade head
DATABASE_URL='postgresql+psycopg://...' uv run python scripts/seed.py
```

---

## 5. Frontend en Firebase Hosting

```bash
cd frontend
npm ci
npm run build          # genera frontend/dist
cd ..
firebase deploy --only hosting
```

La URL pública queda en `https://jamt-spaceapps-2026.web.app`. **Esta es la URL que va en la entrega.**

### 5.1 Proxy en desarrollo

Para que `/api` funcione igual en local, en `frontend/vite.config.ts`:

```ts
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://localhost:8000",
    },
  },
});
```

---

## 6. Control de costes y seguridad

La URL será pública durante el hackathon y el periodo de evaluación, y la factura va a una cuenta personal. Antes del primer despliegue:

1. **Alerta de presupuesto:** Facturación → Presupuestos y alertas → crear un presupuesto de, por ejemplo, 5 € con avisos al 50 %, 90 % y 100 %. La alerta avisa pero **no corta el servicio**.
2. **`--max-instances 2`** en Cloud Run: limita el gasto aunque alguien sature la URL.
3. **Sin instancia mínima** por defecto (escala a cero, con unos segundos de arranque en frío). Si el arranque en frío molesta durante la evaluación, se puede subir temporalmente a `--min-instances 1`, sabiendo que eso sí tiene coste.
4. **Ningún secreto en el repo:** `.env` en `.gitignore`; solo se sube `.env.example`.
5. **Límite en los endpoints que escriben** (`POST /api/players`, `POST /api/runs`): validación estricta con Pydantic y tamaño de cuerpo acotado.

---

## 7. Accesos del equipo

- **Opción simple (recomendada para 48 h):** solo el titular de la cuenta despliega. El resto avisa en el canal del equipo cuando hay algo en `main` listo para desplegar.
- **Si hace falta que otra persona despliegue:** añadirla en IAM con los roles `Cloud Run Developer` y `Service Account User`, y como miembro del proyecto en Firebase.

---

## 8. Objetivos del Makefile

| Objetivo | Comando |
|---|---|
| `make deploy-back` | `gcloud run deploy jamt-api --source backend ...` |
| `make deploy-front` | `npm --prefix frontend run build && firebase deploy --only hosting` |
| `make deploy` | `deploy-back` y después `deploy-front` |
| `make migrate-prod` | `alembic upgrade head` con `DATABASE_URL` de producción (leída de un `.env.prod` local, no versionado) |

---

## 9. Lista de comprobación antes de entregar

- [ ] `https://jamt-spaceapps-2026.web.app` carga en una ventana de incógnito
- [ ] Se puede jugar una partida completa sin registrarse
- [ ] El juego funciona aunque el backend no responda (datos de respaldo)
- [ ] `/api/health` responde
- [ ] La pantalla de créditos cita las fuentes de NASA
- [ ] El README tiene la URL pública y cómo arrancarlo en local
- [ ] Hay un `LICENSE` en la raíz
- [ ] La alerta de presupuesto está activa

---

## 10. Después del hackathon

Mantener el despliegue activo hasta que termine la evaluación. Después:

```bash
gcloud run services delete jamt-api --region europe-west1
firebase hosting:disable
```

O borrar el proyecto entero con `gcloud projects delete jamt-spaceapps-2026`, si no se va a usar más.
