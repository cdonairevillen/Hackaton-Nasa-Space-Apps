# DONKI — Guía de uso

> Cómo pedir datos a DONKI, qué devuelve y cómo lo usamos en el juego.
> DONKI (*Space Weather Database Of Notifications, Knowledge, Information*) es el registro de eventos de clima espacial del CCMC (Community Coordinated Modeling Center) de la NASA.

---

## 1. Acceso

### 1.1 Dos puertas al mismo catálogo

| Puerta | URL base | Clave |
|---|---|---|
| **api.nasa.gov** (la que usamos) | `https://api.nasa.gov/DONKI/{TIPO}` | `api_key` obligatoria |
| CCMC directo (plan B) | `https://kauai.ccmc.gsfc.nasa.gov/DONKI/WS/get/{TIPO}` | Los ejemplos de la documentación no usan clave |

Usamos `api.nasa.gov` como puerta principal. La del CCMC queda como alternativa si api.nasa.gov falla o nos limita.

### 1.2 Clave y límites

| Clave | Límite |
|---|---|
| `DEMO_KEY` | 30 peticiones por hora y 50 por día, **por IP** |
| Clave personal (gratis en api.nasa.gov) | 1.000 peticiones por hora, por clave |

- La clave se pide en https://api.nasa.gov con un email y llega al momento. **Pedidla en la primera hora.**
- Los límites se aplican a **todas** las APIs de api.nasa.gov juntas, no solo a DONKI.
- Cada respuesta incluye las cabeceras `X-RateLimit-Limit` y `X-RateLimit-Remaining`. Si se supera el límite, la respuesta es `429`.
- La clave va en `.env` como `NASA_API_KEY` y **nunca** en el frontend: todas las peticiones a DONKI salen del backend.

---

## 2. Formato de las peticiones

Todas siguen el mismo patrón:

```
GET https://api.nasa.gov/DONKI/{TIPO}?startDate=YYYY-MM-DD&endDate=YYYY-MM-DD&api_key=CLAVE
```

- Fechas en formato `yyyy-MM-dd`, en UTC.
- Si se omiten `startDate` y `endDate`, por defecto se devuelven los **últimos 30 días** (7 días en las simulaciones WSA-Enlil).
- La respuesta es una **lista JSON** de eventos.

### 2.1 Tipos de evento

| Tipo | Qué es | ¿Lo usamos? |
|---|---|---|
| `FLR` | Llamarada solar | **Sí**: el aviso |
| `SEP` | Partículas solares energéticas | **Sí**: el pico de radiación |
| `CME` | Eyección de masa coronal | **Sí**: la amenaza con cuenta atrás |
| `GST` | Tormenta geomagnética (en la Tierra) | Opcional: contexto ("hubo auroras") |
| `CMEAnalysis` | Análisis de las CME (velocidad, ángulo) | Opcional |
| `IPS` | Choque interplanetario | No |
| `MPC` | Cruce de la magnetopausa | No |
| `RBE` | Mejora del cinturón de radiación | No |
| `HSS` | Corriente de viento solar rápido | No |
| `WSAEnlilSimulations` | Simulaciones de propagación | No |
| `notifications` | Avisos emitidos por DONKI | Opcional |

### 2.2 Ejemplos

```bash
# Llamaradas de la tormenta de mayo de 2024
curl "https://api.nasa.gov/DONKI/FLR?startDate=2024-05-08&endDate=2024-05-15&api_key=$NASA_API_KEY"

# Partículas solares energéticas en el mismo periodo
curl "https://api.nasa.gov/DONKI/SEP?startDate=2024-05-08&endDate=2024-05-15&api_key=$NASA_API_KEY"

# Eyecciones de masa coronal
curl "https://api.nasa.gov/DONKI/CME?startDate=2024-05-08&endDate=2024-05-15&api_key=$NASA_API_KEY"

# Análisis de CME: solo la estimación más precisa y velocidad mínima de 500 km/s
curl "https://api.nasa.gov/DONKI/CMEAnalysis?startDate=2024-05-08&endDate=2024-05-15&mostAccurateOnly=true&speed=500&api_key=$NASA_API_KEY"

# Notificaciones de un tipo concreto (all, FLR, SEP, CME, IPS, MPC, GST, RBE, report)
curl "https://api.nasa.gov/DONKI/notifications?startDate=2024-05-08&endDate=2024-05-15&type=FLR&api_key=$NASA_API_KEY"

# Plan B: CCMC directo
curl "https://kauai.ccmc.gsfc.nasa.gov/DONKI/WS/get/FLR?startDate=2024-05-08&endDate=2024-05-15"
```

Para ver los límites restantes, añadid `-i` a `curl` y mirad las cabeceras `X-RateLimit-*`.

---

## 3. Qué devuelve cada tipo

Campos principales. **Verificadlos con una respuesta real** antes de fijar los schemas de Pydantic: DONKI puede añadir campos o devolver `null` en algunos.

### 3.1 `FLR` — Llamarada solar

| Campo | Ejemplo | Uso |
|---|---|---|
| `flrID` | `"2024-05-11T01:10:00-FLR-001"` | Identificador |
| `beginTime` / `peakTime` / `endTime` | `"2024-05-11T01:10Z"` | Cuándo ocurrió |
| `classType` | `"X5.8"` | **Intensidad**: letra (A, B, C, M, X) y número |
| `sourceLocation` | `"S17W21"` | Posición en el Sol |
| `activeRegionNum` | `13664` | Mancha solar de origen |
| `linkedEvents` | `[{"activityID": "...-CME-001"}]` | Eventos que provocó |
| `link` | URL a la ficha en DONKI | **Atribución en el juego** |

### 3.2 `SEP` — Partículas solares energéticas

| Campo | Uso |
|---|---|
| `sepID` | Identificador |
| `eventTime` | Cuándo empezó el aumento de partículas |
| `instruments` | Qué instrumento lo detectó (p. ej. GOES) |
| `linkedEvents` | La llamarada o CME de origen |
| `link` | Atribución |

Las SEP son **el riesgo de radiación real para los astronautas**. Son el evento que más dosis añade en el juego.

### 3.3 `CME` — Eyección de masa coronal

| Campo | Uso |
|---|---|
| `activityID` | Identificador |
| `startTime` | Cuándo salió del Sol |
| `sourceLocation` | Posición en el Sol |
| `cmeAnalyses[]` | Análisis: `speed` (km/s), `halfAngle`, `isMostAccurate` |
| `cmeAnalyses[].enlilList[]` | Simulaciones; incluye la **hora estimada de llegada** del choque |
| `linkedEvents` | Eventos relacionados |
| `link` | Atribución |

La diferencia entre `startTime` y la llegada estimada (normalmente de 1 a 3 días) es la **cuenta atrás** del juego.

### 3.4 `GST` — Tormenta geomagnética

| Campo | Uso |
|---|---|
| `gstID` | Identificador |
| `startTime` | Inicio |
| `allKpIndex[]` | Índice Kp a lo largo del tiempo (0–9; 9 es extremo) |
| `linkedEvents` | La CME que la causó |

---

## 4. Particularidades a tener en cuenta

- **Respuesta vacía:** cuando no hay eventos en el rango, DONKI puede devolver un cuerpo vacío en lugar de `[]`. El cliente debe tratarlo como lista vacía.
- **Datos recientes incompletos:** los eventos se catalogan con algo de retraso, así que los últimos días pueden estar incompletos. Los escenarios del juego se basan en periodos históricos ya cerrados.
- **Rangos de fechas largos:** pedid por tramos (por ejemplo, un mes por petición) en lugar de años enteros.
- **Campos `null`:** `linkedEvents`, `cmeAnalyses` o `activeRegionNum` pueden venir a `null`. Los schemas deben aceptarlo.
- **Punto de vista terrestre:** DONKI cataloga eventos observados desde la Tierra y su entorno. Para la Luna la correspondencia es razonable. Para Marte no: una CME dirigida a la Tierra no tiene por qué llegar a Marte. Si el juego se ambienta en Marte, lo indicamos en pantalla ("evento real; en tu base lo simulamos como si llegara a Marte").

---

## 5. Uso en el backend

### 5.1 Cliente

`backend/app/space_weather/donki_client.py`:

```python
from datetime import date

import httpx

DONKI_BASE_URL = "https://api.nasa.gov/DONKI"
DONKI_FALLBACK_URL = "https://kauai.ccmc.gsfc.nasa.gov/DONKI/WS/get"
DONKI_TIMEOUT = 20.0


async def fetch_donki_events(
    event_type: str,
    start: date,
    end: date,
    api_key: str,
) -> list[dict]:
    params = {
        "startDate": start.isoformat(),
        "endDate": end.isoformat(),
        "api_key": api_key,
    }
    async with httpx.AsyncClient(timeout=DONKI_TIMEOUT) as client:
        response = await client.get(f"{DONKI_BASE_URL}/{event_type}", params=params)
    response.raise_for_status()
    if not response.text.strip():
        return []
    return response.json()


async def fetch_donki_events_fallback(
    event_type: str,
    start: date,
    end: date,
) -> list[dict]:
    params = {"startDate": start.isoformat(), "endDate": end.isoformat()}
    async with httpx.AsyncClient(timeout=DONKI_TIMEOUT) as client:
        response = await client.get(f"{DONKI_FALLBACK_URL}/{event_type}", params=params)
    response.raise_for_status()
    if not response.text.strip():
        return []
    return response.json()
```

### 5.2 Flujo de datos

```
scripts/fetch_donki.py  ──►  backend/data/snapshots/*.json  ──►  scripts/seed.py  ──►  Postgres
        (make fetch-data)          (versionados en git)            (make seed)          (space_weather_events)
                                          │
                                          └──► copia curada en frontend/src/data/fallback/
```

1. **`make fetch-data`** descarga los periodos de los escenarios y los guarda como JSON en `backend/data/snapshots/`. Los snapshots **se versionan en git**: son los datos reales del juego y no queremos depender de la API en la demo.
2. **`make seed`** normaliza los snapshots y los carga en `space_weather_events`.
3. **`GET /api/space-weather`** sirve desde Postgres. Solo el modo opcional "clima espacial de esta semana" consulta DONKI en vivo, cachea el resultado y lo marca con `live: true`.
4. Una **copia curada** de los snapshots va al frontend como respaldo, por si el backend no responde.

### 5.3 Formato normalizado

Lo que guardamos en la base de datos y lo que recibe el frontend, sea cual sea el tipo de evento:

```json
{
  "id": "2024-05-11T01:10:00-FLR-001",
  "event_type": "FLR",
  "start_time": "2024-05-11T01:10:00Z",
  "peak_time": "2024-05-11T01:23:00Z",
  "intensity_label": "X5.8",
  "severity": 3,
  "linked_ids": ["2024-05-11T01:25:00-CME-001"],
  "source_url": "https://...",
  "live": false
}
```

`payload` (JSONB) guarda además el evento original completo, por si hace falta algún campo más adelante.

---

## 6. Del evento real a la mecánica del juego

| Evento DONKI | En el juego | Cálculo de `severity` |
|---|---|---|
| `FLR` | **Aviso**: "¡Llamarada detectada!" Abre la ventana de decisión | C → 1, M → 2, X → 3, X10 o más → 4 |
| `CME` | **Cuenta atrás** hasta la llegada estimada | Por velocidad: menos de 500 km/s → 1, de 500 a 1000 → 2, de 1000 a 2000 → 3, más de 2000 → 4 |
| `SEP` | **Pico de dosis de radiación** (el blindaje lo reduce) | Fija en 3 por defecto (DONKI no da el flujo en el resumen) |
| `GST` | **Contexto** en la pantalla final: "en la Tierra se vieron auroras" | Por Kp máximo: 5–6 → 1, 7 → 2, 8 → 3, 9 → 4 |

Los umbrales de `severity` son una **simplificación nuestra para el juego** y se documentan como tal en `docs/DATA_SOURCES.md`. La clase de la llamarada, las fechas y la velocidad de la CME sí son datos reales.

---

## 7. Escenarios propuestos

| Escenario | Periodo | Por qué |
|---|---|---|
| **Tormenta de mayo de 2024** | 2024-05-08 → 2024-05-15 | La tormenta geomagnética más fuerte en unos 20 años, con varias llamaradas de clase X. Muchos la recuerdan por las auroras |
| Periodo tranquilo | Por elegir tras descargar datos | Partida de tutorial con pocos eventos |
| Actividad moderada | Por elegir tras descargar datos | Dificultad media |

Para elegir los periodos tranquilo y moderado: descargad un año de `FLR` por meses y escoged semanas con pocas llamaradas C y con alguna M, respectivamente.

---

## 8. Atribución

En la pantalla final y en la de créditos de datos:

> Datos de clima espacial: **DONKI**, Community Coordinated Modeling Center (CCMC), NASA Goddard Space Flight Center. Acceso a través de api.nasa.gov.

Cada evento mostrado enlaza a su ficha en DONKI con el campo `link` del evento original.
