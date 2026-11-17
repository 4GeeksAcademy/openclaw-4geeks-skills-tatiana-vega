# SKILL_LOG: Mi Asistente 4Geeks

Registro de las skills construidas con OpenClaw para consultar mi progreso en 4Geeks.

## Configuración inicial
- **OpenClaw:** v2026.11.4, agente `main`, modelo `openai/gpt-6-luna` (login con cuenta ChatGPT vía Codex OAuth). Gateway local en `127.0.0.1:18789`.
- **Token de estudiante:** obtenido de la cookie `4g_tok` en learn.4geeks.com y guardado en `~/.openclaw/.env` (permisos `600`) como `FOURGEEKS_STUDENT_TOKEN`. El archivo está fuera del repositorio y el token nunca se escribe en ninguna skill.

## Conversación de descubrimiento
- **Prompt inicial:** "Quiero darte la habilidad de conectarte a mi cuenta de 4Geeks usando mi token de estudiante, sin que tenga que desarrollar código de mi parte. El token está guardado en la variable de entorno `FOURGEEKS_STUDENT_TOKEN` (en `~/.openclaw/.env`). Nunca lo escribas en un archivo de skill. ¿Qué debemos hacer?"
- **Qué sugirió OpenClaw:**
  1. Basarse en la documentación oficial de autenticación de la API de BreatheCode en lugar de inventar el flujo.
  2. Confirmar en la documentación el endpoint de solo lectura adecuado para comprobar la sesión.
  3. Crear la skill `check_auth`, que lee `FOURGEEKS_STUDENT_TOKEN` en tiempo de ejecución y hace esa comprobación.
  4. Mostrar solo si la autenticación funciona (o el código de error si falla), sin imprimir ni guardar nunca el token.
  5. Una vez validada la conexión, reutilizar la misma autenticación para `get_projects`.
- **Qué información me pidió:** Ninguna credencial. Confirmó que no necesita que le pegue el token porque lo leerá en tiempo de ejecución desde `~/.openclaw/.env`. Antes de crear nada, pidió mi confirmación para continuar.

## Skills principales

### Skill 1: Autenticar (`check_auth`)
- **Prompt en lenguaje natural:** "Sí, procede. Crea la skill `check_auth` usando `GET https://breathecode.herokuapp.com/v1/auth/user/me` con la cabecera `Authorization: Token $FOURGEEKS_STUDENT_TOKEN`. Si la sesión es válida, que muestre mi nombre, mi id y 'sesión activa'; si no, el código HTTP del error. Cuando la crees, pruébala y muéstrame el resultado."
- **Qué hizo OpenClaw:** Creó una skill local de solo lectura en `~/.openclaw/workspace/skills/check_auth/`, compuesta por `SKILL.md` (instrucciones) y `scripts/check_auth.py` (el ejecutable). El script lee el token en tiempo de ejecución desde `FOURGEEKS_STUDENT_TOKEN` o `~/.openclaw/.env`, nunca lo imprime ni lo pasa como argumento de proceso, y limita la salida a nombre, ID y estado (o al código HTTP si hay error). OpenClaw también comentó que el validador de skills prefiere guion medio (`check-auth`), pero mantuvo el nombre que le pedí.
- **Endpoint:** `GET /v1/auth/user/me`
- **Verificación de seguridad:** Busqué el valor del token en los archivos de la skill y no aparece.
- **Resultado de prueba:**
  ```text
  Nombre: Tatiana Vega Ahumada
  ID: 21764
  sesión activa
  ```

### Skill 2: Obtener mis proyectos (`get_my_projects`)
- **Prompt en lenguaje natural:** "Ahora quiero saber qué proyectos tengo asignados. Crea la skill `get_my_projects` que use la misma autenticación que `check_auth` y llame a `GET https://breathecode.herokuapp.com/v1/assignment/user/me/task?task_type=PROJECT`. Debe listar cada proyecto con su título y su estado, traducido así: `task_status=PENDING` → pendiente; `task_status=DONE` y `revision_status=PENDING` → entregado; `revision_status=APPROVED` o `REJECTED` → calificado (indicando aprobado o rechazado). Solo esa responsabilidad, sin resúmenes. Pruébala y muéstrame el resultado."
- **Qué hizo OpenClaw:** Antes de escribir nada, revisó la skill `check_auth` para reutilizar el mismo patrón de autenticación, con el token solo en memoria y sin duplicar credenciales. Después creó `get_my_projects`, que lista título y estado de cada proyecto y nada más.
- **Endpoint:** `GET /v1/assignment/user/me/task?task_type=PROJECT`
- **Mapeo de estados:**
  | `task_status` | `revision_status` | Estado mostrado |
  |---|---|---|
  | `PENDING` | — | pendiente |
  | `DONE` | `PENDING` | entregado |
  | — | `APPROVED` | calificado — aprobado |
  | — | `REJECTED` | calificado — rechazado |
- **Resultado de prueba (extracto, 52 proyectos en total):**
  ```text
  - Milestone 5 — Backend: Inventory Management with ORM & Dual Database — pendiente
  - Command Line Challenge — calificado — aprobado
  - Build a Digital Postcard with HTML/CSS — calificado — aprobado
  - Connecting the Lock: Authentication Flows in the Frontend — pendiente
  - AI basic Inventory Agent Loop — entregado
  - My 4Geeks Assistant — Teaching OpenClaw to Track Your Progress — calificado — rechazado
  - Securing the API: Authentication and Route Restriction in FastAPI — calificado — rechazado
  - Todo List CLI with Python — entregado
  - Milestone 4 — AI-driven Engineering — calificado — aprobado
  ...
  ```
- **Observación:** Algunos proyectos aparecen duplicados con estados distintos (por ejemplo, *Command Line Challenge* sale como aprobado y como pendiente), probablemente porque se asignaron en más de una cohorte. Esto se tiene en cuenta en la Skill 3.

### Skill 3: Obtener trabajo pendiente (`get_pending_work`)
- **Prompt en lenguaje natural:** "Ahora quiero que me digas qué me falta por entregar. Crea la skill `get_pending_work` con la misma autenticación, usando `GET https://breathecode.herokuapp.com/v1/assignment/user/me/task?task_type=PROJECT`. Debe devolver solo lo que realmente me falta: proyectos con `task_status=PENDING` y proyectos con `revision_status=REJECTED` (marcados como 'rechazado — reentregar'). Ojo: hay proyectos duplicados por título; si un título tiene alguna copia aprobada o entregada, no lo cuentes como pendiente. Muestra la lista y el total. Pruébala y muéstrame el resultado."
- **Qué hizo OpenClaw:** Reutilizó el cargador de credenciales y el parser de proyectos que ya existían, en lugar de duplicarlos. Agrupa los proyectos por título: si alguna copia está aprobada o entregada, descarta el título; si la que queda está rechazada, la marca como "rechazado — reentregar".
- **Endpoint:** `GET /v1/assignment/user/me/task?task_type=PROJECT`
- **Diferencia con la Skill 2:** La Skill 2 lista todos los proyectos tal como los devuelve la API. Esta responde a otra pregunta: qué me falta por hacer.
- **Resultado de prueba:**
  ```text
  - Milestone 5 — Backend: Inventory Management with ORM & Dual Database — pendiente
  - Connecting the Lock: Authentication Flows in the Frontend — pendiente
  - The Missing Piece: Password Reset Flow — pendiente
  - My 4Geeks Assistant — Teaching OpenClaw to Track Your Progress — rechazado — reentregar
  - Company Monorepo Containerization — pendiente
  - Applying Spec Driven Development - Financial dashboard — rechazado — reentregar
  - Centralized Incident Manager — pendiente
  - Error Handling — pendiente
  - Milestone 5 — Backoffice: Inventory Management Interface — pendiente
  - EduTrack Data Audit — Related Tables — pendiente
  - Frontend Performance Audit — pendiente
  - Backend Serialization Audit — pendiente
  - Performance Optimisation: Caching — pendiente
  - Securing the API: Authentication and Route Restriction in FastAPI — rechazado — reentregar
  - Company Incident File Analyzer — rechazado — reentregar
  - Supplier Directory — Lightweight Storage API — rechazado — reentregar

  Total: 16
  ```
- **Verificación manual:** Contrasté el resultado con la lista de la Skill 2: 11 pendientes y 5 rechazados, total 16. Los duplicados que tienen alguna copia aprobada o entregada (*Command Line Challenge*, *Todo List CLI with Python*, *Cinema Seat Manager*, etc.) quedaron excluidos correctamente.

### Skill 4: Obtener resumen de progreso (`get_progress_summary`)
- **Prompt en lenguaje natural:** "Ahora quiero saber qué tan avanzado estoy en el curso. Crea la skill `get_progress_summary` con la misma autenticación, usando `GET https://breathecode.herokuapp.com/v1/assignment/user/me/task` (sin filtrar por tipo). Agrupa por `task_type` (PROJECT, LESSON, EXERCISE, QUIZ) y, para cada tipo, muestra completadas / total y el porcentaje (una tarea cuenta como completada si `task_status=DONE`). Para PROJECT, deduplica por título como en `get_pending_work` y muestra también cuántos están aprobados. Termina con un porcentaje global. No listes las tareas, solo el resumen. Si la API pagina los resultados, recorre todas las páginas. Pruébala y muéstrame el resultado."
- **Qué hizo OpenClaw:** Reutilizó el mismo autenticador. Antes de calcular nada, revisó cómo pagina la respuesta la API, sin mostrar datos de tareas, para asegurarse de recorrer todas las páginas. Cuenta los proyectos por título único y también deduplica los aprobados. Solo devuelve totales y porcentajes, sin listar tareas.
- **Endpoint:** `GET /v1/assignment/user/me/task` (todas las páginas, sin filtro de tipo)
- **Resultado de prueba:**
  ```text
  PROJECT: 33 / 44 (75%); aprobados: 26
  LESSON: 18 / 33 (54.5%)
  EXERCISE: 63 / 169 (37.3%)
  QUIZ: 0 / 0 (0%)
  Global: 114 / 246 (46.3%)
  ```
- **Verificación manual:** Los números de proyectos cuadran con las Skills 2 y 3:
  - 52 tareas − 8 títulos duplicados = 44 proyectos únicos.
  - 44 − 11 pendientes = 33 completados.
  - 44 − 16 que me faltan − 2 entregados sin calificar = 26 aprobados.
  - Global: (33 + 18 + 63) / (44 + 33 + 169) = 114 / 246 = 46.3%.

## Skills extendidas

### Skill 5: Mis cohortes y fechas (`get_my_cohorts`)
- **Necesidad identificada:** Para saber cuánto margen tengo para entregar lo pendiente, el agente necesita conocer en qué cohorte estoy y cuándo termina. Las skills de tareas no tienen esa información.
- **Prompt en lenguaje natural:** "Quiero que también sepas en qué cohorte estoy y cuáles son sus fechas, para tener contexto de mis plazos. Revisa la documentación de la API de BreatheCode y dime qué endpoint de solo lectura da mis cohortes con mi estado educativo y las fechas de inicio y fin. Luego crea la skill `get_my_cohorts` con la misma autenticación, mostrando por cohorte: nombre, mi estado (`educational_status`), fecha de inicio, fecha de fin y, si hay, el día actual del curso. Pruébala y muéstrame el resultado."
- **Qué hizo OpenClaw:** Revisó la guía oficial de cohortes para estudiantes y la documentación general antes de elegir el endpoint. Vio que también existe `/academy/cohort/me`, pero eligió `user/me` porque es la ruta del flujo oficial para estudiantes. Identificó los campos exactos: `educational_status` en cada matrícula, y `name`, `kickoff_date`, `ending_date` y `current_day` dentro de `cohort`. Después creó la skill y la probó en modo solo lectura.
- **Endpoint:** `GET /v1/admissions/user/me`
- **Resultado de prueba (extracto, 18 cohortes):**
  ```text
  - AI Engineering Introduction — GRADUATED — inicio: 2026-02-01 — fin: sin fecha — día actual: 0
  - Working with AI coding agents — ACTIVE — inicio: 2026-07-27 — fin: sin fecha — día actual: 0
  - Personal assistants with Openclaw — ACTIVE — inicio: 2026-05-15 — fin: sin fecha — día actual: 0
  - Advanced personal assistants with Openclaw — ACTIVE — inicio: 2026-05-15 — fin: sin fecha — día actual: 0
  - Container applications with Docker — ACTIVE — inicio: 2026-05-25 — fin: sin fecha — día actual: 0
  - Application telemetry — ACTIVE — inicio: 2026-06-16 — fin: sin fecha — día actual: 0
  - latam-aie-pt-4 — ACTIVE — inicio: 2026-07-20 — fin: 2027-01-20 — día actual: 49
  ...
  ```
- **Qué aprendí con esta skill:**
  - Mi cohorte principal es **latam-aie-pt-4**: voy en el día 49 y termina el **2027-01-20**.
  - Las demás cohortes corresponden a módulos o cursos sin fecha de fin.
  - Esto explica los proyectos duplicados de la Skill 2: el mismo proyecto se asignó desde cohortes distintas.

### Skill 6: Próximos eventos (`get_upcoming_events`)
- **Necesidad identificada:** Quiero que el agente me avise de los workshops y eventos de 4Geeks sin tener que entrar a la plataforma a buscarlos.
- **Prompt en lenguaje natural:** "Por último, quiero que me avises de los próximos eventos y workshops de 4Geeks a los que puedo asistir. Revisa la documentación de la API de BreatheCode y dime qué endpoint de solo lectura devuelve los eventos disponibles para mí como estudiante. Luego crea la skill `get_upcoming_events` con la misma autenticación, mostrando solo eventos futuros (a partir de hoy), ordenados por fecha: título, fecha y hora de inicio, y si es online o presencial. Máximo 10. Pruébala y muéstrame el resultado."
- **Qué hizo OpenClaw:** Consultó la documentación oficial de eventos para confirmar los filtros de fecha y el campo que indica la modalidad. Identificó `GET /v1/events/all`, que es el catálogo de sesiones públicas: admite el parámetro `start` en ISO 8601 y devuelve `starting_at` y `online_event`. Como ese filtro puede incluir eventos que ya empezaron, la skill los descarta localmente y ordena los futuros por fecha. Hace una única consulta de solo lectura y muestra como máximo 10 eventos.
- **Endpoint:** `GET /v1/events/all?start=<fecha ISO 8601>`
- **Limitación detectada por OpenClaw:** El catálogo es público, así que no comprueba si yo en particular puedo inscribirme en cada evento.
- **Resultado de prueba (consultado el 2026-11-17):**
  ```text
  - El Agente Caza-Ofertas: Automatiza tu Black Friday con IA en 3 días (DÍA 2) — 2026-11-18 18:30 UTC — online
  - El Agente Caza-Ofertas: Automatiza tu Black Friday con IA en 3 días (DÍA 3) — 2026-11-19 18:30 UTC — online
  - Feeling Like an Impostor Doesn't Make You One. — 2026-11-19 23:00 UTC — online
  - Buscar empleo no es suerte, es método: Crea tu Tracker con IA — 2026-11-29 18:00 UTC — online
  ```

## Resumen de skills

| # | Skill | Pregunta que responde | Endpoint |
|---|---|---|---|
| 1 | `check_auth` | ¿Mi token es válido y la sesión está activa? | `GET /v1/auth/user/me` |
| 2 | `get_my_projects` | ¿Qué proyectos tengo y en qué estado están? | `GET /v1/assignment/user/me/task?task_type=PROJECT` |
| 3 | `get_pending_work` | ¿Qué me falta por entregar? | `GET /v1/assignment/user/me/task?task_type=PROJECT` |
| 4 | `get_progress_summary` | ¿Qué tan avanzada estoy en el curso? | `GET /v1/assignment/user/me/task` |
| 5 | `get_my_cohorts` | ¿En qué cohorte estoy y cuándo termina? | `GET /v1/admissions/user/me` |
| 6 | `get_upcoming_events` | ¿Qué eventos de 4Geeks vienen pronto? | `GET /v1/events/all` |

Todas las skills viven en `~/.openclaw/workspace/skills/`, fuera de este repositorio. Todas leen `FOURGEEKS_STUDENT_TOKEN` en tiempo de ejecución desde `~/.openclaw/.env` y ninguna tiene el token escrito.