# Junior Astronaut Mission Trainer — Propuesta

> NASA Space Apps Challenge 2026 · Reto: *Build a Junior Astronaut Mission Trainer*
> Equipo de 4 personas · 48 horas
>
> Este documento es la base de información del proyecto. Las decisiones que se tomen durante el desarrollo se registran en `docs/decisions.md` y los cambios en `docs/changelogs.md`. Si algo de aquí queda obsoleto, se actualiza aquí y se anota allí el motivo.

---

## 1. Resumen

Construimos un juego de navegador en el que niños de 7 a 12 años gestionan una base en la Luna o en Marte. El jugador reparte una cantidad limitada de energía entre mantener a la tripulación con vida, protegerla de la radiación y producir comida, y vive las consecuencias de esas decisiones a lo largo de una misión simulada.

**Objetivo educativo:** que el jugador termine la partida entendiendo una restricción real de la supervivencia en el espacio profundo: **ninguna decisión es gratis**. Ahorrar energía hoy puede costar oxígeno mañana, y protegerse de una tormenta solar deja a la base sin reservas esa noche.

**Lo que nos diferencia:**

- Los eventos de radiación del juego son **eventos solares reales** sacados de la base de datos DONKI de la NASA.
- Al final de cada partida, una pantalla **"¿Qué pasó de verdad?"** conecta la decisión del jugador con el evento real que la provocó y con su fuente.
- Se juega en el navegador, sin descargas ni registro.

---

## 2. Mecánica de juego

### 2.1 Recursos

Limitamos el juego a tres sistemas, siguiendo la recomendación del propio reto. Más sistemas abruman a un jugador de esta edad.

| Recurso | Qué representa | Cómo se gana | Cómo se pierde |
|---|---|---|---|
| **Energía** | El presupuesto de la base | Paneles solares (de día) y baterías | Todo lo que se active la consume |
| **Oxígeno / soporte vital** | Lo que mantiene viva a la tripulación | Asignándole energía | Baja de forma constante; más rápido si le quitas energía |
| **Dosis de radiación** | La radiación acumulada por la tripulación | No se recupera | Radiación de fondo constante y picos durante los eventos solares; el blindaje la frena |

El jugador controla **un único reparto**: cuánta energía va a soporte vital, cuánta a blindaje y cuánta al invernadero. Todo lo demás es consecuencia de ese reparto.

### 2.2 El compromiso central

- El **blindaje** reduce la dosis de radiación, pero consume mucha energía.
- El **soporte vital** mantiene el oxígeno, pero si le quitas energía no notas el problema hasta varios soles después. Esta es la consecuencia diferida que pide el reto.
- El **invernadero** (comida) es la inversión a largo plazo: no salva a nadie hoy, pero sin él la misión no aguanta.

No existe la opción segura: cada asignación quita energía a otra cosa.

### 2.3 Eventos

- **Alerta de llamarada o tormenta solar:** basada en un evento real de DONKI. Aparece un aviso con cuenta atrás y el jugador decide si refuerza el blindaje o lleva a la tripulación al refugio, a costa de la energía de esa noche.
- **Noche lunar o marciana:** los paneles dejan de producir y la base vive de baterías.
- **Retraso de comunicaciones (solo Marte):** la ayuda o la información de la Tierra llega tarde, con el retraso real escalado al tiempo del juego.

### 2.4 Fin de partida

- **Derrota:** el oxígeno llega a cero o la dosis de radiación supera el límite de la misión.
- **Victoria:** sobrevivir N soles. *Sol* es el día marciano; lo explicamos dentro del juego.
- **Pantalla final:** qué pasó, por qué y qué evento real lo provocó, con la fuente enlazada. La derrota tiene que sentirse como aprendizaje, no como castigo, para invitar a volver a jugar.

### 2.5 Lenguaje

Los términos que no podemos evitar (dosis de radiación, sol, microgravedad) se explican **en el momento en que aparecen**, con una frase corta y sin jerga.

---

## 3. Formato: tiempo real frente a por turnos

El motor se diseña como una función pura `step(state, actions, dt)`. Con eso **los dos formatos usan el mismo motor**: en tiempo real se llama muchas veces por segundo con un `dt` pequeño, y por turnos se llama una vez por decisión con un `dt` grande. Podemos empezar a construir sin haber elegido y decidir con una partida de prueba real.

### 3.1 Tiempo real

El tiempo avanza solo. El jugador ajusta el reparto de energía mientras pasan las horas y los soles, y puede pausar y cambiar la velocidad.

**Ventajas**

- Es más inmersivo y emocionante: la alerta solar con cuenta atrás crea tensión real.
- Se parece más a una sala de control de misión.
- Las consecuencias diferidas se ven en directo, con el oxígeno bajando poco a poco.
- Es el formato que más "parece un juego", lo que ayuda a retener la atención.

**Inconvenientes**

- Puede abrumar a los más pequeños (7–8 años) si todo pasa a la vez.
- Cuesta más reflexionar: el jugador reacciona en lugar de pensar el compromiso.
- El balanceo es más difícil, porque los números dependen de la velocidad del juego.
- Requiere más trabajo de frontend: bucle de juego, animaciones fluidas, pausa y velocidad.
- Es más difícil de testear de forma determinista, aunque el motor puro lo resuelve en gran parte.

**Condiciones para que funcione:** pausa siempre disponible, velocidad base baja, un primer sol que funcione como tutorial y alertas que **pausen el juego automáticamente** al aparecer.

### 3.2 Por turnos

Cada turno es un sol. El jugador decide el reparto, pulsa "Avanzar sol" y ve el resultado.

**Ventajas**

- Cada decisión se piensa: es el formato que mejor enseña el compromiso.
- Es más accesible para los más pequeños y funciona en el aula (un profesor puede proyectarlo y debatir cada turno).
- El balanceo y los tests son más sencillos.
- Necesita menos frontend, así que queda más tiempo para la mecánica y los datos.

**Inconvenientes**

- Es menos emocionante y tiene más riesgo de aburrir.
- Las alertas solares pierden urgencia.
- Se parece más a una hoja de cálculo si no se cuida la presentación.

### 3.3 Recomendación

Construir el motor común desde el principio y **jugar las dos versiones en una partida de prueba hacia la hora 10–12**. Si el tiempo real se entiende con pausa y alertas que pausan, nos quedamos con él. Si no, pasamos a por turnos sin haber perdido trabajo.

Hay una opción intermedia que también vale la pena probar: **tiempo real que se pausa en cada evento**. Tiene la tensión del tiempo real y el momento de reflexión del modo por turnos justo cuando importa.

---

## 4. Arquitectura técnica

### 4.1 Visión general

```
┌──────────────────────────┐        ┌──────────────────────────┐        ┌─────────────┐
│ Frontend (React + TS)    │  HTTP  │ Backend (FastAPI)        │  SQL   │ PostgreSQL  │
│  - motor de simulación   │ ─────▶ │  - proxy y caché DONKI   │ ─────▶ │  (Alembic)  │
│  - bucle de juego        │        │  - escenarios            │        └─────────────┘
│  - UI                    │        │  - jugadores y partidas  │
│  - datos de respaldo     │        └────────────┬─────────────┘
└──────────────────────────┘                     │ HTTPS
                                                 ▼
                                        api.nasa.gov / DONKI
```

**Principio:** el juego se ejecuta entero en el navegador. El backend aporta datos y persistencia, pero **el juego tiene que poder funcionar aunque el backend falle**, usando los datos de respaldo embebidos.

### 4.2 Frontend

- **React + TypeScript** con Vite.
- `engine/` es TypeScript puro, sin React: estado, `step()`, eventos y condiciones de victoria y derrota. Es la pieza más importante del proyecto y se testea con Vitest.
- `game/` contiene el bucle (`requestAnimationFrame` en tiempo real o el botón de avanzar sol por turnos), la pausa y la velocidad.
- Datos de respaldo en JSON con eventos reales curados, para cuando el backend o la API no respondan.

### 4.3 Backend

- **FastAPI + Pydantic + SQLAlchemy 2.0 + Alembic**, con `uv` para el entorno.
- Arquitectura en capas por dominio (`models / schemas / repository / service / router`), igual que en ft_transcendence.

Endpoints:

| Método | Ruta | Función |
|---|---|---|
| `GET` | `/api/space-weather` | Eventos solares (de DONKI cacheados o del snapshot). Indica `live: true/false` |
| `GET` | `/api/scenarios` | Escenarios basados en eventos reales (p. ej. la tormenta de mayo de 2024) |
| `POST` | `/api/players` | Crea el jugador temporal |
| `POST` | `/api/runs` | Guarda el resultado de una partida |
| `GET` | `/api/leaderboard` | Ranking |
| `GET` | `/api/health` | Comprobación para el despliegue |

### 4.4 Base de datos

| Tabla | Campos principales |
|---|---|
| `players` | `id` (UUID), `nickname`, `created_at` |
| `runs` | `id`, `player_id`, `site` (moon/mars), `mode` (realtime/turns), `seed`, `sols_survived`, `score`, `end_cause`, `created_at` |
| `space_weather_events` | `id`, `donki_id`, `event_type`, `start_time`, `intensity`, `payload` (JSONB), `fetched_at` |
| `scenarios` | `id`, `name`, `site`, `description`, `event_ids` |

### 4.5 Jugador temporal

- Al entrar se genera un **UUID** que se guarda en `localStorage` y un **nickname generado automáticamente** (por ejemplo, "Zorro-Lunar-42").
- **El niño no escribe su nombre.** Así no recogemos datos personales de menores, y no hacen falta contraseña ni registro.
- Si no da tiempo a implementarlo, el juego funciona igual sin ranking.

### 4.6 Despliegue

- **Frontend:** estático en Vercel o Netlify.
- **Backend:** Google Cloud Run (lo conocemos de hourli-fy).
- **Base de datos:** Postgres gestionado (Supabase o Neon).
- **Docker Compose** solo para el desarrollo local. Lo que se entrega es la URL pública.

---

## 5. Estructura de carpetas

Basada en lo que ya usamos en ft_transcendence (backend en capas por dominio, `uv`, Alembic) y hourli-fy (FastAPI + React + Postgres).

```
junior-astronaut/
├── PROPUESTA.md                  # este documento
├── README.md                     # cómo arrancar, cómo jugar, URL de la demo
├── Makefile                      # punto de entrada único
├── docker-compose.yml            # postgres + backend + frontend (desarrollo)
├── .env.example                  # NASA_API_KEY, DATABASE_URL, VITE_API_URL
├── .gitignore
│
├── docs/
│   ├── decisions.md              # registro de decisiones (qué, por qué, cuándo)
│   ├── changelogs.md             # registro de cambios
│   └── DATA_SOURCES.md           # datasets de la NASA usados y por qué
│
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── alembic/
│   │   ├── env.py
│   │   └── versions/
│   ├── app/
│   │   ├── main.py               # crea la app, CORS, monta routers
│   │   ├── core/
│   │   │   ├── config.py         # settings (pydantic-settings)
│   │   │   └── database.py       # engine, sesión, Base
│   │   ├── players/
│   │   │   ├── models.py
│   │   │   ├── schemas.py
│   │   │   ├── repository.py
│   │   │   ├── service.py
│   │   │   └── router.py
│   │   ├── runs/
│   │   │   └── (mismas capas)
│   │   ├── scenarios/
│   │   │   └── (mismas capas)
│   │   └── space_weather/
│   │       ├── (mismas capas)
│   │       └── donki_client.py   # llamadas a api.nasa.gov
│   ├── data/
│   │   └── snapshots/            # JSON reales de DONKI descargados
│   ├── scripts/
│   │   ├── fetch_donki.py        # descarga eventos históricos → snapshots
│   │   └── seed.py               # carga snapshots y escenarios en la BD
│   └── tests/
│       ├── conftest.py
│       ├── test_players.py
│       ├── test_runs.py
│       └── test_space_weather.py
│
└── frontend/
    ├── package.json
    ├── Dockerfile
    ├── vite.config.ts
    ├── index.html
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── engine/               # TS puro, sin React
        │   ├── types.ts          # GameState, Action, GameEvent
        │   ├── constants.ts      # tasas y límites (con fuente en comentario)
        │   ├── step.ts           # step(state, actions, dt)
        │   ├── events.ts         # eventos solares, noche, comunicaciones
        │   ├── outcome.ts        # victoria / derrota
        │   └── engine.test.ts
        ├── game/
        │   ├── useGameLoop.ts    # bucle tiempo real / avance por turnos
        │   └── GameScreen.tsx
        ├── components/           # paneles de recursos, alertas, sliders
        ├── pages/                # inicio, juego, resultado, créditos de datos
        ├── api/
        │   ├── client.ts
        │   └── types.ts          # espejo de los schemas de Pydantic
        ├── data/
        │   └── fallback/         # eventos reales curados (sin backend)
        └── assets/
```

### 5.1 Makefile

| Objetivo | Qué hace |
|---|---|
| `make up` / `make down` | Levanta / para los contenedores |
| `make dev` | Backend y frontend en modo desarrollo con recarga |
| `make migrate` | `alembic upgrade head` |
| `make seed` | Carga snapshots y escenarios en la BD |
| `make fetch-data` | Descarga datos frescos de DONKI a `backend/data/snapshots/` |
| `make db-reset` | Borra la BD, migra y vuelve a cargar los datos |
| `make test` | `test-back` (pytest) y `test-front` (Vitest) |
| `make lint` | ruff y eslint |
| `make clean` | Borra contenedores, volúmenes y cachés |

> En Windows, `make` se ejecuta desde WSL o Git Bash.

---

## 6. Equipo y cronograma

### 6.1 Reparto

| Rol | Responsabilidad | Entregable principal |
|---|---|---|
| **A · Motor y diseño de juego** | Reglas, números, eventos, balanceo | `frontend/src/engine/` con tests |
| **B · Frontend e interfaz** | Bucle de juego, pantallas, accesibilidad, explicaciones en contexto | `game/`, `components/`, `pages/` |
| **C · Backend y datos** | FastAPI, Alembic, cliente DONKI, seed, ranking | `backend/` con tests |
| **D · Integración, despliegue y entrega** | Docker, Makefile, despliegue temprano, `DATA_SOURCES.md`, materiales de entrega | URL pública y entrega |

El **contrato de la API** (schemas de Pydantic ↔ `api/types.ts`) se fija en la primera hora entre B y C, y cualquier cambio se anota en `docs/decisions.md`.

### 6.2 Cronograma

| Horas | Fase | Objetivo |
|---|---|---|
| 0–3 | Arranque | Repo, Makefile, Docker, contrato de API, elegir Luna o Marte, pedir la API key de la NASA |
| 3–10 | Porción vertical mínima | Motor con 3 recursos, interfaz básica jugable, `/space-weather` con fallback. **Despliegue en la hora 8**, aunque esté vacío |
| 10–12 | Partida de prueba | Jugar ambos formatos y **decidir tiempo real o por turnos** |
| 12–20 | Funcionalidad completa | Eventos reales, fin de partida, pantalla "¿Qué pasó de verdad?", jugador temporal y ranking |
| 20–28 | Descanso escalonado | Nunca los cuatro dormidos a la vez |
| 28–40 | Pulido de mecánica | Playtesting, balanceo, explicaciones en contexto, créditos de datos |
| **40** | **Congelación** | A partir de aquí solo se arreglan bugs |
| 40–48 | Entrega | README, `DATA_SOURCES.md`, demo o vídeo, formulario de entrega |

---

## 7. Riesgos y cómo abordarlos

### 7.1 Querer abarcar demasiado

**Riesgo.** Con 48 horas y cuatro personas con ideas, lo más probable es que el proyecto crezca más rápido de lo que se construye: más recursos, más eventos, más pantallas. El resultado típico es llegar al final con muchas piezas a medias y ninguna partida completa.

**Cómo abordarlo.** Definimos un **mínimo jugable**: tres recursos, un tipo de evento solar real, victoria y derrota, y la pantalla final. Hasta que eso funcione de principio a fin y esté desplegado, no se añade nada más. Cualquier idea nueva va a una lista de "si da tiempo" en `docs/decisions.md`. Además, la hora 40 es un límite firme: después no entran funcionalidades nuevas.

### 7.2 El tiempo real abruma a los jugadores más pequeños

**Riesgo.** Un niño de 7 años que tiene que vigilar tres barras mientras el tiempo corre puede frustrarse y dejar de jugar sin haber entendido nada. Si el jugador reacciona en vez de pensar, el juego deja de enseñar el compromiso, que es justo lo que evalúan los jueces.

**Cómo abordarlo.** Pausa siempre visible, velocidad base lenta, alertas que pausan el juego automáticamente y un primer sol que funcione como tutorial guiado. Si la partida de prueba de la hora 10–12 demuestra que no se entiende, pasamos al modo por turnos. Como el motor es el mismo, el cambio no cuesta trabajo perdido.

### 7.3 La API de la NASA falla o nos limita

**Riesgo.** Con la clave de demostración, la API de la NASA permite muy pocas peticiones por hora. Además puede estar caída o lenta justo cuando la prueban los jueces. Si el juego depende de esa llamada, una caída externa nos deja sin demo.

**Cómo abordarlo.** Pedimos una API key gratuita en la primera hora. El backend nunca llama a DONKI en cada petición del jugador: descarga, guarda en Postgres y sirve desde ahí. El frontend además lleva un snapshot de eventos reales embebido, de modo que el juego funciona incluso sin backend. Cuando usamos datos de archivo **lo indicamos en pantalla** ("evento real del 10/05/2024") en vez de presentarlos como datos en vivo, como pide el reto.

### 7.4 Los números del juego no son creíbles

**Riesgo.** El reto valora expresamente usar datos reales en lugar de estadísticas inventadas. Si las tasas de radiación o los límites de dosis son inventados, perdemos uno de los puntos fuertes que nos diferencian, y un juez con conocimientos lo detectará.

**Cómo abordarlo.** Cada constante del motor que represente un valor real lleva en un comentario la fuente de la que sale, y todas quedan documentadas en `docs/DATA_SOURCES.md`. Para la radiación de fondo usamos medidas publicadas (el instrumento RAD del rover Curiosity en Marte y el de la sonda Chang'e 4 en la Luna). Donde tengamos que simplificar o escalar un valor para que sea jugable, lo decimos abiertamente: un valor simplificado y declarado es honesto, y uno inventado y presentado como real no lo es.

### 7.5 Un juego entretenido que no enseña, o uno educativo que aburre

**Riesgo.** Es el riesgo que el propio reto señala. Si priorizamos lo visual, el juego puede verse bien y no enseñar ningún compromiso real. Si priorizamos la exactitud, puede convertirse en una hoja de cálculo que ningún niño quiera jugar.

**Cómo abordarlo.** La regla de diseño es que **ninguna opción sea gratis**: si en el balanceo aparece una estrategia que siempre gana sin coste, es un bug de diseño y se corrige. La pantalla "¿Qué pasó de verdad?" es donde se concentra el aprendizaje explícito, para que el juego en sí pueda centrarse en ser divertido. Y probamos con alguien ajeno al equipo antes de la hora 40 para comprobar si entiende el compromiso sin que se lo expliquemos.

### 7.6 Integración tardía entre frontend y backend

**Riesgo.** Si cada parte trabaja aislada hasta el final, la integración de las últimas horas destapa incompatibilidades (nombres de campos, formatos de fecha, CORS) justo cuando no hay tiempo para arreglarlas.

**Cómo abordarlo.** El contrato de la API se fija en la primera hora y se refleja en `api/types.ts`. Desplegamos una versión vacía pero conectada en la hora 8, para que los problemas de despliegue, CORS y variables de entorno aparezcan pronto. El frontend puede trabajar con los datos de respaldo mientras el backend no está listo, así nadie se bloquea.

### 7.7 Despliegue en el último momento

**Riesgo.** Los jueces necesitan una URL que funcione sin instalar nada. Desplegar por primera vez en las últimas horas suele fallar por motivos ajenos al código: variables de entorno, migraciones en producción, cuotas de los servicios gratuitos o arranques en frío.

**Cómo abordarlo.** Una persona (rol D) es responsable del despliegue desde el principio, con un primer despliegue en la hora 8 y redespliegues frecuentes. El README incluye la URL pública y también cómo arrancarlo en local con `make up`, como plan B.

### 7.8 Datos personales de menores

**Riesgo.** El público son niños de 7 a 12 años. Pedir nombres, correos o cualquier dato identificable crea un problema legal y ético innecesario, y los jueces pueden valorarlo negativamente.

**Cómo abordarlo.** No hay registro. El jugador temporal es un UUID con un nickname generado automáticamente; el niño no escribe nada. No guardamos ningún dato que permita identificar a una persona.

### 7.9 Cansancio del equipo

**Riesgo.** Pasadas las primeras 24 horas, la calidad de las decisiones y del código cae mucho. Los bugs introducidos de madrugada se pagan en las horas finales.

**Cómo abordarlo.** Descanso escalonado entre las horas 20 y 28, para que siempre haya alguien despierto y nadie trabaje 48 horas seguidas. Las decisiones importantes (formato del juego, congelación) se toman en horas en las que todo el equipo está despierto, y se anotan en `docs/decisions.md` para que quien se incorpore tras descansar sepa qué ha cambiado.

---

## 8. Entregables

- URL pública del juego, jugable sin registro.
- Repositorio con README, `PROPUESTA.md` y `docs/DATA_SOURCES.md`.
- Pantalla de créditos de datos dentro del juego.
- Materiales de entrega de Space Apps (revisar los requisitos exactos al arrancar, no al final).
