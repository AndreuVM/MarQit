---
name: marq
description: Cómo un agente de IA debe usar las herramientas MCP de MarQ PM (tareas, reservas entre agentes, cronograma, etiquetas, requisitos, notas) y los comandos "marq it", "marq status", "marq next", etc. Úsala siempre que el usuario mencione MarQ, pida actualizar el tablero o el estado del proyecto, o escriba un comando que empiece por "marq".
---

# MarQ — skill para agentes

MarQ es el tablero de proyectos de este equipo. Tú trabajas en el código; MarQ guarda el estado real del trabajo para que las personas y otros agentes lo vean. Esta skill te dice **cómo usar sus herramientas MCP** (`pm_*`) y **qué hacer cuando el usuario escribe un comando `marq ...`**.

Principio rector: **MarQ debe reflejar la realidad, no tus intenciones.** Solo cambias el estado de una tarea con evidencia (commit, tests, archivo, lo que el usuario te diga). Si no la hay, no lo marcas.

---

## 1. Reglas de uso de las herramientas

### 1.1 Gasta pocos tokens
Un proyecto puede tener cientos de tareas. **Nunca pidas la lista completa.**

| Necesitas | Usa |
|---|---|
| Saber qué proyectos existen | `pm_list_projects` |
| Orientarte en un proyecto | `pm_get_project_tree` con `limit: 0` (solo resumen: conteos por estado/prioridad). Sube `limit` solo si hace falta |
| Un subconjunto de tareas | `pm_list_tasks` con filtros (`status[]`, `priority[]`, `assignedTo`, `unassigned`, `tags[]`, `requirements[]`, `search`, `available`, `updatedSince`) |
| Ver el detalle de **una** tarea | `pm_get_task` (acepta UUID o referencia, p. ej. `CF-19`) |
| Elegir qué hacer ahora | `pm_list_tasks` con `available: true`, `sort: "priority"`, `limit: 10` |
| Cambios desde la última vez | `pm_list_tasks` con `updatedSince: <ISO>` |

- El listado es **compacto** (sin descripción ni notas). Pide `includeDescription: true` solo si la necesitas, y entonces con `limit` bajo.
- Pagina con `offset = next_offset` mientras `has_more` sea `true`. Para (`marq it`) normalmente basta con filtrar por estado: `in_progress`, `review`, `blocked`, `todo`.
- Usa **siempre la referencia** (`CF-19`) en lugar del UUID cuando la tengas.
- No vuelvas a pedir lo que ya tienes en la conversación.

### 1.2 Reserva antes de escribir (leases)
Varios agentes pueden trabajar a la vez. Para **modificar** una tarea:

1. `pm_acquire_task { taskId, executionId }` → guarda `lease_id` y `fencing_token`. Usa un `executionId` estable por sesión (reintentar con el mismo devuelve la misma reserva).
2. Escribe pasando `leaseId` y `fencingToken` en cada llamada: `pm_update_task_status`, `pm_update_task`, `pm_report_blocker`, `pm_log_time_spent`.
3. Si vas a tardar, `pm_renew_task_lock` antes de que venza (por defecto 15 min).
4. Al terminar o abandonar, **`pm_release_task`**, también si algo falla.

No necesitan reserva: leer, `pm_add_future_note` y `pm_create_task`.

Si `pm_acquire_task` devuelve que está ocupada (`owner_id`, `retry_after_seconds`): **no la fuerces ni esperes en bucle**. Salta a otra tarea, déjala anotada en el informe y sigue. Si el error es `stale_lock` / `lock_expired`: tu reserva caducó, vuelve a adquirir y reintenta **una vez**.

Para saber qué tienes reservado tras un reinicio: `pm_list_my_task_locks`.

### 1.3 Estados y cuándo usar cada uno

| Estado | Significa | Evidencia necesaria |
|---|---|---|
| `backlog` | Idea o trabajo sin planificar | — |
| `todo` | Planificado, listo para empezar | Tiene descripción clara |
| `in_progress` | Se está trabajando en ello **ahora** | Hay cambios sin terminar o has empezado en esta sesión |
| `review` | Implementado, falta revisión/validación | Commit o cambios completos, pendiente de tests o de una persona |
| `done` | Terminado y verificado | Tests pasan / commit mergeado / el usuario lo confirma |
| `blocked` | No puede avanzar | Causa concreta: usa `pm_report_blocker` |

Reglas:
- **Nunca `done` sin verificación.** Si el código existe pero no está probado, `review`.
- Una tarea `blocked` siempre se marca con `pm_report_blocker` (deja la causa y avisa a las personas), no con un simple cambio de estado.
- Máximo una o dos tareas tuyas en `in_progress` a la vez.
- Pasa `comment` en `pm_update_task_status` con **una línea** de por qué cambia (commit, test, decisión).

### 1.4 Crear tareas
- **Busca antes de crear:** `pm_list_tasks { search: "<palabras clave>" }`. No dupliques.
- Título en imperativo y concreto (“Añadir RLS a la tabla `ledger`”), descripción en Markdown con criterio de aceptación.
- `priority`: `critical` solo si bloquea a otros o es un fallo en producción; `high` para lo que toca en este sprint; `medium` por defecto; `low` para mejoras.
- `estimatedHours` si lo puedes estimar; `dependencies` con referencias de tareas previas.
- No inventes referencias: deja que MarQ asigne el código (`referenceCode` solo si el usuario lo pide).

### 1.4b Cronograma (fechas y dependencias)
Cada tarea puede tener **fechas previstas** (`plannedStart`/`plannedEnd`, formato `AAAA-MM-DD`, inclusivas: del 5 al 5 = 1 día). No son `started_at`/`completed_at`, que son las reales. Las personas las ven en la pestaña **Cronograma** (diagrama de Gantt con flechas de dependencia), y lo que escribas ahí aparece **en vivo**.

- **Antes de planificar o replanificar:** `pm_get_schedule { projectId }`. Devuelve fechas, dependencias (con referencias `R-012`), **conflictos**, ruta crítica con holguras, atrasadas y tareas sin planificar. Es compacto: úsalo en vez de listar todas las tareas.
- **Planificar varias a la vez:** `pm_schedule_tasks { projectId, items: [{ taskId, plannedStart, plannedEnd }] }` (máx. 100). Si una falla (fecha inválida, reservada por otro agente…) las demás se aplican y el fallo vuelve en `failed`. La respuesta incluye los conflictos resultantes: **léelos y corrígelos**.
- **Una sola tarea:** `pm_update_task { plannedStart, plannedEnd }`. `null` borra la planificación. Con una sola fecha, la tarea dura 1 día. `pm_create_task` también acepta `plannedStart`/`plannedEnd`.
- **Dependencias fin→inicio:** una tarea debe **empezar después** de que termine su predecesora (si empieza el mismo día o antes, es un solape y se marca como conflicto). Una predecesora ya `done` no genera conflicto.
- **Las dependencias no pueden ser circulares:** MarQ rechaza con "Circular dependency" (las herramientas responden en inglés) la que cerraría un ciclo. No reintentes lo mismo: revisa el orden.
- **Cómo planificar bien:** ordena por dependencias; duración ≈ `estimatedHours` / 8 h por día (mínimo 1 día); pon cada tarea justo después de su última predecesora; no solapes lo que depende entre sí. La **ruta crítica** (`critical: true`, `slack_days ≤ 0`) es la cadena que fija el final del proyecto: ahí es donde un retraso cuesta.
- **No muevas por tu cuenta las tareas de otros.** Las que están reservadas (lease) por otro agente fallarán; las de personas, cámbialas solo si el usuario te lo ha pedido.
- Cada cambio de fechas queda en el historial de la tarea (`schedule_change`), con el valor anterior y el nuevo.

### 1.4c Etiquetas y requisitos
Son dos cosas **distintas**; no las mezcles.

**Etiquetas** (categorías de una tarea, solo para filtrar y localizar):
- `pm_list_tags { projectId }` devuelve las que ya existen con su número de tareas. **Reutilízalas**: antes de etiquetar, mira cuáles hay en vez de inventar variantes (`seguridad` y `security`, `api` y `apis`).
- Pon etiquetas al crear (`pm_create_task { tags: ["seguridad", "api"] }`) o con `pm_update_task { tags }`. En la actualización la lista **sustituye a todas las anteriores**: para añadir una, envía las existentes más la nueva; `[]` las quita todas; si no envías `tags`, no se tocan.
- Son palabras cortas en minúsculas (`seguridad`, `api`, `métricas`); MarQ normaliza `#Seguridad` a `seguridad` y los espacios a `-`. Máximo **10** por tarea (más es un error, no se recorta). Una o dos por tarea suelen bastar: categoría o área, no estado ni prioridad (para eso ya hay campos).
- Para encontrar tareas: `pm_list_tasks { tags: ["seguridad"] }` (con varias, la tarea debe tener **todas**) o `search: "#seguridad"`.

**Requisitos** (lo que el proyecto debe cumplir; aparecen en la pestaña **Información** del proyecto):
- `pm_list_requirements { projectId, onlyPending? }` → requisitos con código `RQ-003`, estado y resumen.
- `pm_add_requirement { projectId, title, description? }`: una frase **verificable** («Autenticación obligatoria en modo local»), no una tarea («Implementar login»). Si es trabajo a realizar, crea una tarea.
- `pm_update_requirement { projectId, requirementId, met }`: marca `met: true` **solo cuando hayas comprobado que se cumple** (mismo criterio que `done`). También cambia título o descripción.
- `pm_delete_requirement` solo si el usuario lo pide o ya no aplica; si se cumple, márcalo, no lo borres.
- **Enlaza cada tarea con los requisitos que ayuda a cumplir**: `pm_create_task { requirements: ["RQ-001"] }` o `pm_update_task { requirements: [...] }`. En la actualización la lista **sustituye a todas las anteriores** (para añadir uno, envía los que ya tenía más el nuevo; `[]` desenlaza todo; si no envías `requirements`, no se tocan). Un código que no existe es un error y no se guarda nada: míralos antes con `pm_list_requirements`. Una tarea puede cubrir varios requisitos y un requisito puede tener varias tareas.
- Para saber **qué tareas cubren un requisito**: `pm_list_tasks { requirements: ["RQ-003"] }` (con varios, la tarea debe cubrir **todos**). `requirements: ["none"]` lista las tareas que no cubren ninguno. `pm_list_requirements` muestra por requisito `tasks: "hechas/enlazadas"`.
- Marcar una tarea como `done` **no** marca el requisito como cumplido: el requisito se cumple cuando tú compruebas la condición, normalmente con todas sus tareas hechas. Antes de marcarlo, revisa sus tareas con `pm_list_tasks { requirements: [...] }`.
- Los requisitos no necesitan reserva ni tienen fechas ni dependencias. `pm_get_project_tree` incluye `requirements_summary` y las etiquetas más usadas.

### 1.4d Notas del proyecto (ideas rápidas)
Una **nota** es una idea que se te ocurre al momento y que **todavía no está detallada** como una tarea: algo para implementar o revisar más adelante. Aparecen en la pestaña **Notas** del proyecto y las pueden crear tanto las personas como tú. No son tareas, ni requisitos, ni las anotaciones de una tarea (`pm_add_future_note`).
- `pm_list_notes { projectId, status?, all? }` → por defecto las **abiertas** (pendientes de detallar), de la más reciente a la más antigua, con código `N-003`, título, un extracto del contenido y las tareas ancladas. `status` = `open` | `converted` | `archived`; `all: true` trae todas. `pm_get_project_tree` incluye `notes_summary` (cuántas hay en cada estado).
- `pm_get_note { projectId, noteId }`: la nota entera (contenido completo y tareas ancladas con su estado). Léela antes de detallarla.
- `pm_add_note { projectId, title, content?, tasks? }`: apunta una idea. Basta una línea en `title`; añade `content` solo si tienes detalle que merezca conservarse. Si ya está clara y es trabajo a realizar, es una **tarea** (`pm_create_task`); si es una condición que el proyecto debe cumplir, un **requisito**.
- **Detallar una nota en tareas concretas:** lee la nota (`pm_get_note`), crea las tareas (`pm_create_task`) y después `pm_update_note { projectId, noteId, addTasks: ["R-021", "R-022"], status: "converted" }`. Las tareas quedan **ancladas** a la nota y la nota pasa a `converted`.
- **Anclar una tarea como mera referencia** (`addTasks`) **no** cambia el estado de la nota: solo marca `converted` cuando la idea ya está detallada. `removeTasks` desancla. Una tarea que no existe en el proyecto es un error y no se guarda nada. Desde la tarea, `pm_get_task` muestra `linked_notes` con los códigos de las notas que la mencionan.
- `status: "archived"` para una idea descartada (no se pierde) y `"open"` para reabrirla. `pm_delete_note` solo si el usuario lo pide.
- No necesitan reserva ni tienen fechas ni dependencias. Cada proyecto admite hasta 500 notas.

### 1.5 Tiempo, anotaciones y avisos
- `pm_log_time_spent`: solo tiempo que **sabes** que se ha dedicado (medido en la sesión o dicho por el usuario). No estimes a ojo ni imputes por defecto.
- `pm_add_future_note`: decisiones, riesgos, prerrequisitos e ideas (`category`: `future_idea`, `prerequisite`, `risk`, `architecture`, `general`). Son anotaciones de **una tarea** (no son las notas del proyecto de 1.4d) y **no** requieren reserva. Úsalas en vez de ensanchar la descripción.
- `pm_broadcast_notification`: solo para avisos que las personas deban ver sí o sí (despliegue, bloqueo crítico). Un aviso por evento; no lo uses como log.
- `pm_get_user_notifications` con tu identidad (`agent:<id>`) al empezar una sesión larga para ver si te avisaron de reservas vencidas.

### 1.5b Historial, anotaciones y GitHub (solo lectura, salvo notificaciones)
- `pm_get_task_timeline { taskId }`: historial de transiciones de una tarea y tiempo en cada estado. Úsalo para responder «¿qué pasó con CF-19?», no para listar el proyecto. En el plan Gratis oculta eventos de más de 30 días (`history_truncated: true`).
- `pm_get_future_notes { taskId }`: notas de decisiones, riesgos y prerrequisitos. Léelas antes de retomar una tarea ajena.
- `pm_mark_notification_read { notificationId, userId }`: márcala leída cuando ya la hayas atendido.
- `pm_inspect_github_repo { repositoryUrl }`: inspecciona un repositorio de GitHub (metadatos, progreso estimado y tareas sugeridas). Solo para **arrancar un proyecto nuevo** o rescatar uno existente; las tareas sugeridas son una propuesta: enséñalas al usuario antes de crearlas. Requiere plan Pro (`plan_feature_unavailable` si no).

### 1.6 Lo que MarQ no hace (desde MCP)
No hay herramientas para **crear, editar o borrar proyectos**, ni para **borrar tareas**. Si el proyecto no existe, díselo al usuario para que lo cree en la aplicación; no lo sustituyas por otro con nombre parecido.

### 1.7 Seguridad
- El token MCP (`marq_live_...`) es una credencial: nunca lo escribas en commits, archivos del repo, issues ni en tu respuesta.
- Nunca incluyas secretos en títulos, descripciones, notas ni comentarios.
- Si MarQ responde con un error de autenticación o permisos (401/403), **no reintentes en bucle ni busques atajos**: informa al usuario con el mensaje exacto.

---

## 2. Comandos

El usuario puede escribir estos comandos en lenguaje natural (“marq it”, “haz un marq”, “marquea esto”). Interprétalos igual.

### `marq it` — sincronizar todo el proyecto con MarQ
Es el comando principal: pone el tablero al día con lo que realmente ha pasado en el repositorio y en esta conversación.

**Paso 0 — Identificar el proyecto**
1. `pm_list_projects`.
2. Elige el proyecto cuyo nombre coincida con el repositorio/carpeta (`package.json`, nombre del directorio, remoto de git) o con lo que diga el usuario (`marq it casal-finance`).
3. Si hay varios candidatos o ninguno claro, pregunta una sola vez. Si no existe, para y avisa (ver 1.6).

**Paso 1 — Leer el tablero (barato)**
- `pm_get_project_tree { limit: 0 }` para los conteos.
- `pm_list_tasks` con `status: ["in_progress","review","blocked"]` (todo lo “vivo”), y `status: ["todo"]` con `limit` 25 solo si hace falta casar trabajo nuevo.
- Para casar trabajo nuevo con tareas existentes, busca con `search` por referencia y por palabras clave; no cargues el backlog entero.

**Paso 2 — Reunir evidencia del trabajo real**
- `git log` desde la última actualización (usa el `updated_at` más reciente de las tareas vivas como punto de partida, o `--since` que indique el usuario), `git status` y `git diff --stat`.
- Referencias de tarea en mensajes de commit/ramas (`CF-19`, `R-012`) → son la evidencia más fuerte.
- Resultado de tests/build si los has ejecutado en esta sesión (no los inventes; si no los has ejecutado, di que no están verificados).
- Lo que el usuario y tú habéis hecho en esta conversación.
- Archivos de planificación del repo si existen (TODO, roadmap, `docs/`).

**Paso 3 — Conciliar (decidir los cambios)**
Para cada elemento de trabajo, una de estas acciones:

| Situación | Acción en MarQ |
|---|---|
| Tarea existente, trabajo terminado **y verificado** | → `done` |
| Tarea existente, implementado sin verificar | → `review` |
| Tarea existente, se ha empezado | → `in_progress` |
| Tarea `in_progress` sin actividad y sin motivo | proponer volver a `todo` (pregunta, no lo hagas solo) |
| Bloqueo detectado (fallo externo, dependencia, decisión pendiente) | `pm_report_blocker` con la causa |
| Trabajo hecho sin tarea | crear la tarea (con búsqueda previa) y ponerla en el estado que corresponda |
| Trabajo pendiente identificado (TODOs, siguientes pasos acordados) | crear tareas en `backlog`/`todo` con prioridad y dependencias |
| Decisión, riesgo o idea surgida | `pm_add_future_note` en la tarea relacionada |
| Tiempo conocido | `pm_log_time_spent` |

**Paso 4 — Aplicar**
- Si los cambios están respaldados por evidencia clara, **aplícalos directamente**; no pidas permiso por cada tarea.
- **Pregunta antes** (una sola vez, con la lista) cuando: vayas a crear **más de 10** tareas, haya cambios ambiguos (no sabes a qué tarea corresponde un commit), o vayas a retroceder una tarea de estado.
- Por cada tarea que modifiques: `acquire → escribir → release`. Hazlo tarea a tarea (no mantengas muchas reservas abiertas) y libera siempre, también ante errores.
- Si una tarea está reservada por otro agente, **sáltala** y anótalo en el informe.

**Paso 5 — Informe final (corto)**
```
marq it · <Proyecto>
✔ Hecho:       CF-19 → done · CF-21 → review
▶ En curso:    CF-22 → in_progress
⛔ Bloqueado:  CF-24 (Supabase sin acceso)
＋ Creadas:    CF-31 "Migrar tablas a RLS" (high)
⏱ Tiempo:      CF-19 +45 min
⚠ Omitido:     CF-20 reservada por agent:otro hasta 13:40
Tablero ahora: 19 todo · 2 en progreso · 1 en revisión · 1 bloqueada
```
Solo lo que ha cambiado, con referencias. Sin repetir descripciones.

### Resto de comandos

| Comando | Qué hacer |
|---|---|
| `marq it <proyecto>` | Igual que `marq it` sobre un proyecto concreto |
| `marq plan` / `marq it --dry` | Pasos 0–3 y mostrar la propuesta **sin escribir nada** |
| `marq status` | Resumen del proyecto: `pm_get_project_tree { limit: 0 }` + vivas (`in_progress`, `review`, `blocked`). Solo lectura |
| `marq next` | Propón la siguiente tarea: `available: true`, `sort: "priority"`, `limit: 5`. Si el usuario acepta: `pm_acquire_task` y pásala a `in_progress` |
| `marq start <ref>` | `pm_get_task` → `pm_acquire_task` → `in_progress`. Mantén la reserva mientras trabajas |
| `marq done <ref> [comentario]` | Verifica (tests/commit) → `done` con comentario → `pm_release_task`. Si no hay verificación, usa `review` y dilo |
| `marq block <ref> <causa>` | `pm_report_blocker` con la causa |
| `marq log <ref> <30m\|1h>` | `pm_log_time_spent` |
| `marq add <título>` | Busca duplicados → `pm_create_task`. Pide prioridad solo si no se deduce |
| `marq note <ref> <texto>` | `pm_add_future_note`: anotación de **una tarea** (elige categoría) |
| `marq idea <texto>` | `pm_add_note`: apunta una idea rápida en las notas del proyecto. Busca antes si ya existe una igual (`pm_list_notes`) |
| `marq notes` | `pm_list_notes`: las notas abiertas. Resúmelas en 5 líneas máximo (código, título y si tienen tareas ancladas). Solo lectura |
| `marq notes <N-003>` | `pm_get_note`: la nota completa y sus tareas ancladas. Solo lectura |
| `marq detail <N-003>` | Lee la nota, **propón** las tareas concretas y, cuando el usuario las apruebe, créalas, ánclalas con `pm_update_note { addTasks, status: "converted" }` |
| `marq find <texto>` | `pm_list_tasks { search }` (con `#etiqueta` busca por etiqueta) |
| `marq tag <ref> <etiquetas>` | `pm_list_tags` para reutilizar las existentes y `pm_update_task { tags }` (recuerda: sustituye todas; conserva las que ya tenía) |
| `marq reqs` | `pm_list_requirements`. Resúmelo en 3 líneas (cumplidos/total y los pendientes). Solo lectura |
| `marq reqs <RQ-003>` | `pm_list_tasks { requirements: ["RQ-003"] }`: las tareas que cubren ese requisito y cómo van. Solo lectura |
| `marq mine` | Tus tareas: `pm_list_my_task_locks` y `pm_list_tasks { assignedTo: "<tu id>" }` |
| `marq schedule` | Cronograma del proyecto: `pm_get_schedule`. Resúmelo en 5 líneas (rango, conflictos, ruta crítica, atrasadas, sin planificar). Solo lectura |
| `marq plan-dates` | Propón fechas para las tareas sin planificar siguiendo 1.4b y, si el usuario acepta, aplícalas con `pm_schedule_tasks` |
| `marq metrics` | `pm_get_project_metrics` y resúmelo en 5 líneas |
| `marq history <ref>` | `pm_get_task_timeline` (y `pm_get_future_notes` si hay dudas): resume en 3-4 líneas qué ha pasado con la tarea. Solo lectura |
| `marq inspect <owner/repo>` | `pm_inspect_github_repo`: resume el repositorio y **propón** las tareas sugeridas sin crearlas hasta que el usuario las apruebe |
| `marq release` | Libera todas tus reservas (`pm_list_my_task_locks` → `pm_release_task`) |

---

## 3. Flujo recomendado en una sesión normal

1. **Al empezar:** `marq status` (barato) y, si procede, `pm_get_user_notifications`. Si hay notas abiertas relacionadas con lo que vas a hacer (`notes_summary`), echa un vistazo con `marq notes`.
2. **Antes de tocar algo:** reserva la tarea (`marq start`).
3. **Mientras trabajas:** notas con decisiones y riesgos; renueva la reserva en trabajos largos.
4. **Al terminar:** verifica, `marq done` o `review`, libera.
5. **Al cerrar la sesión o hacer commit importante:** `marq it`.

## 4. Errores frecuentes

| Respuesta | Qué hacer |
|---|---|
| `locked_by_other` / tarea ocupada | Salta a otra tarea; informa del propietario y `retry_after_seconds` |
| `lock_expired` / `stale_lock` | Adquiere de nuevo y reintenta una vez |
| `lock_required` | Falta `pm_acquire_task`; adquiere y reenvía con `leaseId` y `fencingToken` |
| `dependencies_unmet` | Trabaja antes en las dependencias o dile al usuario que está bloqueada por ellas |
| `agent_identity_required` | La sesión no usa un token de agente propio; avisa al usuario |
| Proyecto o tarea no encontrada | Verifica con `pm_list_projects` / `search`; no inventes IDs |
| `plan_limit_reached` / `plan_feature_unavailable` (error de MCP o HTTP 402) | La cuenta superó un límite de su plan (proyectos, agentes, colaboradores, tareas por proyecto) o una función no incluida. **No reintentes ni crees tareas alternativas para esquivarlo.** Cuéntale al usuario qué límite es (`resource`, `limit`, `used`) y que puede mejorar su plan o liberar espacio desde la aplicación. Sigue con lo que sí puedes hacer |
| `project_read_only` / `collaborator_read_only` / `agent_suspended_by_plan` (403) | El plan de la cuenta dejó ese proyecto en solo lectura, ese colaborador como lector o ese agente suspendido. Puedes seguir **leyendo**; no intentes escribir de otra forma. Díselo al usuario: puede elegir qué sigue activo en "Plan y uso" o mejorar el plan |
| 401 / 403 | Credencial o permisos: informa con el mensaje exacto y detente |

## 5. Lo que nunca debes hacer
- Pedir el árbol completo de un proyecto grande.
- Marcar `done` sin verificación.
- Crear tareas duplicadas o con títulos vagos (“arreglos varios”).
- Dejar reservas abiertas al terminar.
- Forzar o ignorar la reserva de otro agente.
- Escribir tokens o secretos en MarQ o en el repositorio.
- Reintentar o esquivar un límite del plan (`plan_limit_reached`): se informa y se continúa.
- Modificar muchas tareas sin enseñar después un resumen de lo hecho.
