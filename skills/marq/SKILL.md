---
name: marq
description: How an AI agent should use the MarQ PM MCP tools (tasks, multi-agent leases, schedule, tags, requirements, notes) and the "marq it", "marq status", "marq next" commands. Use it whenever the user mentions MarQ, asks to update the board or the project status, or types a command starting with "marq" (in English or Spanish, e.g. "haz un marq").
---

# MarQ — skill for agents

MarQ is this team's project board. You work on the code; MarQ keeps the real state of the work so that people and other agents can see it. This skill tells you **how to use its MCP tools** (`pm_*`) and **what to do when the user types a `marq ...` command**.

Guiding principle: **MarQ must reflect reality, not your intentions.** You only change a task's status with evidence (a commit, tests, a file, something the user told you). If there is none, don't mark it. Always reply in the user's language (e.g. English or Spanish) and interpret natural-language triggers in either language (`marq it`, `haz un marq`, `marq status`, `marq next`).

---

## 1. Tool usage rules

### 1.1 Spend few tokens
A project can have hundreds of tasks. **Never ask for the full list.**

| You need | Use |
|---|---|
| Which projects exist | `pm_list_projects` |
| Get oriented in a project | `pm_get_project_tree` with `limit: 0` (summary only: counts by status/priority). Raise `limit` only if needed |
| A subset of tasks | `pm_list_tasks` with filters (`status[]`, `priority[]`, `assignedTo`, `unassigned`, `tags[]`, `requirements[]`, `search`, `available`, `updatedSince`) |
| The detail of **one** task | `pm_get_task` (accepts a UUID or a reference, e.g. `CF-19`) |
| Choose what to do now | `pm_list_tasks` with `available: true`, `sort: "priority"`, `limit: 10` |
| Changes since last time | `pm_list_tasks` with `updatedSince: <ISO>` |

- The listing is **compact** (no description or notes). Ask for `includeDescription: true` only if you need it, and then with a low `limit`.
- Paginate with `offset = next_offset` while `has_more` is `true`. For `marq it`, filtering by status is usually enough: `in_progress`, `review`, `blocked`, `todo`.
- **Always use the reference** (`CF-19`) instead of the UUID when you have it.
- Don't ask again for what you already have in the conversation.

### 1.2 Lease before writing
Several agents may work at the same time. To **modify** a task:

1. `pm_acquire_task { taskId, executionId }` → keep `lease_id` and `fencing_token`. Use a stable `executionId` per session (retrying with the same one returns the same lease).
2. Write passing `leaseId` and `fencingToken` on every call: `pm_update_task_status`, `pm_update_task`, `pm_report_blocker`, `pm_log_time_spent`.
3. If you will take a while, `pm_renew_task_lock` before it expires (15 min by default).
4. When you finish or give up, **`pm_release_task`**, also if something fails.

These do not need a lease: reading, `pm_add_future_note` and `pm_create_task`.

If `pm_acquire_task` says the task is busy (`owner_id`, `retry_after_seconds`): **don't force it or wait in a loop**. Move to another task, note it in the report and carry on. If the error is `stale_lock` / `lock_expired`: your lease expired, acquire again and retry **once**.

To find out what you hold after a restart: `pm_list_my_task_locks`.

### 1.3 Statuses and when to use each

| Status | Means | Evidence needed |
|---|---|---|
| `backlog` | Idea or unplanned work | — |
| `todo` | Planned, ready to start | Has a clear description |
| `in_progress` | Being worked on **right now** | There are unfinished changes or you started in this session |
| `review` | Implemented, awaiting review/validation | Commit or complete changes, pending tests or a person |
| `done` | Finished and verified | Tests pass / commit merged / the user confirms |
| `blocked` | Cannot move forward | A concrete cause: use `pm_report_blocker` |

Rules:
- **Never `done` without verification.** If the code exists but is untested, use `review`.
- A `blocked` task is always marked with `pm_report_blocker` (it records the cause and notifies people), not with a plain status change.
- At most one or two of your own tasks in `in_progress` at a time.
- Pass `comment` in `pm_update_task_status` with **one line** saying why it changes (commit, test, decision).

### 1.4 Creating tasks
- **Search before creating:** `pm_list_tasks { search: "<keywords>" }`. Don't duplicate.
- Title in the imperative and specific ("Add RLS to the `ledger` table"), description in Markdown with an acceptance criterion.
- `priority`: `critical` only if it blocks others or is a production failure; `high` for what is due this sprint; `medium` by default; `low` for improvements.
- `estimatedHours` if you can estimate it; `dependencies` with references of earlier tasks.
- Don't invent references: let MarQ assign the code (`referenceCode` only if the user asks).

### 1.4b Schedule (dates and dependencies)
Each task can have **planned dates** (`plannedStart`/`plannedEnd`, format `YYYY-MM-DD`, inclusive: 5th to 5th = 1 day). They are not `started_at`/`completed_at`, which are the actual ones. People see them in the **Schedule** tab (a Gantt chart with dependency arrows), and what you write there appears **live**.

- **Before planning or replanning:** `pm_get_schedule { projectId }`. It returns dates, dependencies (with references like `R-012`), **conflicts**, the critical path with slack, overdue and unscheduled tasks. It is compact: use it instead of listing all the tasks.
- **Plan several at once:** `pm_schedule_tasks { projectId, items: [{ taskId, plannedStart, plannedEnd }] }` (max 100). If one fails (invalid date, leased by another agent…) the rest are applied and the failure comes back in `failed`. The response includes the resulting conflicts: **read them and fix them**.
- **A single task:** `pm_update_task { plannedStart, plannedEnd }`. `null` clears the schedule. With only one date, the task lasts 1 day. `pm_create_task` also accepts `plannedStart`/`plannedEnd`.
- **Finish-to-start dependencies:** a task must **start after** its predecessor ends (starting the same day or earlier is an overlap and is flagged as a conflict). A predecessor that is already `done` creates no conflict.
- **Dependencies cannot be circular:** MarQ rejects with "Circular dependency" the one that would close a cycle. Don't retry the same thing: review the order.
- **How to plan well:** order by dependencies; duration ≈ `estimatedHours` / 8 h per day (minimum 1 day); place each task right after its last predecessor; don't overlap tasks that depend on each other. The **critical path** (`critical: true`, `slack_days ≤ 0`) is the chain that sets the end of the project: that is where a delay costs.
- **Don't move other people's tasks on your own.** Tasks leased by another agent will fail; for people's tasks, change them only if the user asked.
- Every date change is recorded in the task history (`schedule_change`), with the previous and new values.

### 1.4c Tags and requirements
These are two **different** things; don't mix them up.

**Tags** (categories of a task, only for filtering and locating):
- `pm_list_tags { projectId }` returns the existing ones with their task count. **Reuse them**: before tagging, look at which exist instead of inventing variants (`security` and `sec`, `api` and `apis`).
- Add tags on creation (`pm_create_task { tags: ["security", "api"] }`) or with `pm_update_task { tags }`. On update the list **replaces all previous ones**: to add one, send the existing ones plus the new one; `[]` removes them all; if you don't send `tags`, they are left untouched.
- They are short lowercase words (`security`, `api`, `metrics`); MarQ normalizes `#Security` to `security` and spaces to `-`. Maximum **10** per task (more is an error, it is not truncated). One or two per task are usually enough: category or area, not status or priority (there are fields for those).
- To find tasks: `pm_list_tasks { tags: ["security"] }` (with several, the task must have **all**) or `search: "#security"`.

**Requirements** (what the project must satisfy; they appear in the project's **Info** tab):
- `pm_list_requirements { projectId, onlyPending? }` → requirements with code `RQ-003`, status and summary.
- `pm_add_requirement { projectId, title, description? }`: a **verifiable** sentence ("Authentication is mandatory in local mode"), not a task ("Implement login"). If it is work to do, create a task.
- `pm_update_requirement { projectId, requirementId, met }`: set `met: true` **only when you have checked that it holds** (same criterion as `done`). It can also change the title or description.
- `pm_delete_requirement` only if the user asks or it no longer applies; if it is satisfied, mark it, don't delete it.
- **Link each task to the requirements it helps satisfy**: `pm_create_task { requirements: ["RQ-001"] }` or `pm_update_task { requirements: [...] }`. On update the list **replaces all previous ones** (to add one, send the ones it already had plus the new one; `[]` unlinks everything; if you don't send `requirements`, they are left untouched). A code that does not exist is an error and nothing is saved: check them first with `pm_list_requirements`. A task can cover several requirements and a requirement can have several tasks.
- To find **which tasks cover a requirement**: `pm_list_tasks { requirements: ["RQ-003"] }` (with several, the task must cover **all**). `requirements: ["none"]` lists the tasks that cover none. `pm_list_requirements` shows `tasks: "done/linked"` per requirement.
- Marking a task `done` does **not** mark the requirement as met: a requirement is met when you check its condition, normally with all its tasks done. Before marking it, review its tasks with `pm_list_tasks { requirements: [...] }`.
- Requirements need no lease and have no dates or dependencies. `pm_get_project_tree` includes `requirements_summary` and the most used tags.

### 1.4d Project notes (ideas, bugs, informative notes and improvements)
A **note** captures a new idea, a bug found, an informative note (for instance about a task) or a pending improvement that is **not detailed yet** into an executable task. They appear in the project's **Notes** tab, are classified by group (`group`: `idea` | `bug` | `info` | `improvement`), and both people and you can create them. They are not tasks, nor requirements, nor the internal annotations of a task (`pm_add_future_note`).
- `pm_list_notes { projectId, status?, group?, all? }` → by default the **open** ones (waiting to be detailed), newest first, with code `N-003`, title, `group`, a content excerpt and the pinned tasks. `status` = `open` | `converted` | `archived`; `group` = `idea` | `bug` | `info` | `improvement`; `all: true` returns all of them. `pm_get_project_tree` includes `notes_summary` (how many there are in each status and in `by_group`).
- `pm_get_note { projectId, noteId }`: the whole note (full content, group, status and pinned tasks with their status). Read it before detailing it.
- `pm_add_note { projectId, title, content?, group?, tasks? }`: jot down a note classifying it with `group` (`idea` by default, `bug` for bugs found, `info` for informative notes/context about tasks, `improvement` for technical refinements). One line in `title` is enough; add `content` only if you have detail worth keeping. If it is already clear and is work to do, it is a **task** (`pm_create_task`); if it is a condition the project must satisfy, a **requirement**.
- **Detailing a note into concrete tasks:** read the note (`pm_get_note`), create the tasks (`pm_create_task`) and then `pm_update_note { projectId, noteId, addTasks: ["R-021", "R-022"], status: "converted" }`. The tasks stay **pinned** to the note and the note becomes `converted`. You can also change `group` via `pm_update_note`.
- **Pinning a task as mere reference** (`addTasks`) does **not** change the note's status: only mark `converted` when the idea or bug has been detailed into tasks. `removeTasks` unpins. A task that does not exist in the project is an error and nothing is saved. From the task, `pm_get_task` shows `linked_notes` with the codes of the notes that mention it.
- `status: "archived"` for a discarded note (it is not lost) and `"open"` to reopen it. `pm_delete_note` only if the user asks.
- They need no lease and have no dates or dependencies. Each project allows up to 500 notes.

### 1.5 Time, annotations and alerts
- `pm_log_time_spent`: only time you **know** was spent (measured in the session or stated by the user). Don't estimate by eye or log by default.
- `pm_add_future_note`: decisions, bugs, risks, prerequisites and ideas (`category`: `future_idea`, `bug`, `prerequisite`, `risk`, `architecture`, `general`). They are annotations on **one task** (not the project notes of 1.4d) and do **not** need a lease. Use them instead of bloating the description.
- `pm_broadcast_notification`: only for alerts people must see (a deployment, a critical blocker). One alert per event; don't use it as a log.
- `pm_get_user_notifications` with your identity (`agent:<id>`) at the start of a long session to see whether you were warned about expired leases.

### 1.5b History, annotations and GitHub (read-only, except notifications)
- `pm_get_task_timeline { taskId }`: transition history of a task and time in each status. Use it to answer "what happened with CF-19?", not to list the project. On the Free plan it hides events older than 30 days (`history_truncated: true`).
- `pm_get_future_notes { taskId }`: notes with decisions, risks and prerequisites. Read them before picking up someone else's task.
- `pm_mark_notification_read { notificationId, userId }`: mark it read once you have dealt with it.
- `pm_inspect_github_repo { repositoryUrl }`: inspects a GitHub repository (metadata, estimated progress and suggested tasks). Only to **start a new project** or rescue an existing one; the suggested tasks are a proposal: show them to the user before creating them. Requires the Pro plan (`plan_feature_unavailable` otherwise).

### 1.6 What MarQ does not do (via MCP)
There are no tools to **create, edit or delete projects**, nor to **delete tasks**. If the project does not exist, tell the user so they create it in the app; don't substitute another one with a similar name.

### 1.7 Security
- The MCP token (`marq_live_...`) is a credential: never write it in commits, repo files, issues or in your reply.
- Never include secrets in titles, descriptions, notes or comments.
- If MarQ answers with an authentication or permissions error (401/403), **don't retry in a loop or look for shortcuts**: tell the user the exact message.

---

## 2. Commands

The user can type these commands in natural language ("marq it", "do a marq", "marq this"). Interpret them the same way.

### `marq it` — sync the whole project with MarQ
This is the main command: it brings the board up to date with what has really happened in the repository and in this conversation.

**Step 0 — Identify the project**
1. `pm_list_projects`.
2. Pick the project whose name matches the repository/folder (`package.json`, directory name, git remote) or what the user says (`marq it casal-finance`).
3. If there are several candidates or none is clear, ask once. If it does not exist, stop and warn (see 1.6).

**Step 1 — Read the board (cheaply)**
- `pm_get_project_tree { limit: 0 }` for the counts.
- `pm_list_tasks` with `status: ["in_progress","review","blocked"]` (everything "live"), and `status: ["todo"]` with `limit` 25 only if needed to match new work.
- To match new work with existing tasks, search with `search` by reference and keywords; don't load the whole backlog.

**Step 2 — Gather evidence of the real work**
- `git log` since the last update (use the most recent `updated_at` of the live tasks as the starting point, or the `--since` the user gives), `git status` and `git diff --stat`.
- Task references in commit messages/branches (`CF-19`, `R-012`) → the strongest evidence.
- Test/build results if you ran them in this session (don't invent them; if you didn't run them, say they are not verified).
- What the user and you did in this conversation.
- Planning files in the repo if they exist (TODO, roadmap, `docs/`).

**Step 3 — Reconcile (decide the changes)**
For each piece of work, one of these actions:

| Situation | Action in MarQ |
|---|---|
| Existing task, work finished **and verified** | → `done` |
| Existing task, implemented but unverified | → `review` |
| Existing task, started | → `in_progress` |
| `in_progress` task with no activity and no reason | propose going back to `todo` (ask, don't do it yourself) |
| Blocker detected (external failure, dependency, pending decision) | `pm_report_blocker` with the cause |
| Work done without a task | create the task (searching first) and put it in the right status |
| Pending work identified (TODOs, agreed next steps) | create tasks in `backlog`/`todo` with priority and dependencies |
| Decision, risk or idea that came up | `pm_add_future_note` on the related task |
| Known time | `pm_log_time_spent` |

**Step 4 — Apply**
- If the changes are backed by clear evidence, **apply them directly**; don't ask permission for each task.
- **Ask first** (once, with the list) when you will create **more than 10** tasks, when there are ambiguous changes (you don't know which task a commit belongs to), or when you will move a task back a status.
- For each task you modify: `acquire → write → release`. Do it task by task (don't keep many leases open) and always release, also on errors.
- If a task is leased by another agent, **skip it** and note it in the report.

**Step 5 — Final report (short)**
```
marq it · <Project>
✔ Done:        CF-19 → done · CF-21 → review
▶ In progress: CF-22 → in_progress
⛔ Blocked:    CF-24 (no Supabase access)
＋ Created:    CF-31 "Migrate tables to RLS" (high)
⏱ Time:        CF-19 +45 min
⚠ Skipped:     CF-20 leased by agent:other until 13:40
Board now: 19 todo · 2 in progress · 1 in review · 1 blocked
```
Only what has changed, with references. Without repeating descriptions.

### Other commands

| Command | What to do |
|---|---|
| `marq it <project>` | Same as `marq it` on a specific project |
| `marq plan` / `marq it --dry` | Steps 0–3 and show the proposal **without writing anything** |
| `marq status` | Project summary: `pm_get_project_tree { limit: 0 }` + live tasks (`in_progress`, `review`, `blocked`). Read-only |
| `marq next` | Propose the next task: `available: true`, `sort: "priority"`, `limit: 5`. If the user accepts: `pm_acquire_task` and move it to `in_progress` |
| `marq start <ref>` | `pm_get_task` → `pm_acquire_task` → `in_progress`. Keep the lease while you work |
| `marq done <ref> [comment]` | Verify (tests/commit) → `done` with a comment → `pm_release_task`. If there is no verification, use `review` and say so |
| `marq block <ref> <cause>` | `pm_report_blocker` with the cause |
| `marq log <ref> <30m\|1h>` | `pm_log_time_spent` |
| `marq add <title>` | Search for duplicates → `pm_create_task`. Ask for priority only if it can't be inferred |
| `marq note <ref> <text>` | `pm_add_future_note`: annotation on **one task** (pick a category) |
| `marq idea <text>` | `pm_add_note`: jot down a quick idea in the project notes. Look first for an equal one (`pm_list_notes`) |
| `marq notes` | `pm_list_notes`: the open notes. Summarize in 5 lines at most (code, title and whether they have pinned tasks). Read-only |
| `marq notes <N-003>` | `pm_get_note`: the whole note and its pinned tasks. Read-only |
| `marq detail <N-003>` | Read the note, **propose** the concrete tasks and, once the user approves them, create them and pin them with `pm_update_note { addTasks, status: "converted" }` |
| `marq find <text>` | `pm_list_tasks { search }` (with `#tag` it searches by tag) |
| `marq tag <ref> <tags>` | `pm_list_tags` to reuse the existing ones and `pm_update_task { tags }` (remember: it replaces all; keep the ones it already had) |
| `marq reqs` | `pm_list_requirements`. Summarize in 3 lines (met/total and the pending ones). Read-only |
| `marq reqs <RQ-003>` | `pm_list_tasks { requirements: ["RQ-003"] }`: the tasks that cover that requirement and how they are going. Read-only |
| `marq mine` | Your tasks: `pm_list_my_task_locks` and `pm_list_tasks { assignedTo: "<your id>" }` |
| `marq schedule` | Project schedule: `pm_get_schedule`. Summarize in 5 lines (range, conflicts, critical path, overdue, unscheduled). Read-only |
| `marq plan-dates` | Propose dates for the unscheduled tasks following 1.4b and, if the user accepts, apply them with `pm_schedule_tasks` |
| `marq metrics` | `pm_get_project_metrics` and summarize in 5 lines |
| `marq history <ref>` | `pm_get_task_timeline` (and `pm_get_future_notes` if in doubt): summarize in 3-4 lines what has happened with the task. Read-only |
| `marq inspect <owner/repo>` | `pm_inspect_github_repo`: summarize the repository and **propose** the suggested tasks without creating them until the user approves |
| `marq release` | Release all your leases (`pm_list_my_task_locks` → `pm_release_task`) |

---

## 3. Recommended flow in a normal session

1. **At the start:** `marq status` (cheap) and, if relevant, `pm_get_user_notifications`. If there are open notes related to what you are about to do (`notes_summary`), take a look with `marq notes`.
2. **Before touching something:** lease the task (`marq start`).
3. **While working:** notes with decisions and risks; renew the lease on long jobs.
4. **When finished:** verify, `marq done` or `review`, release.
5. **When closing the session or making an important commit:** `marq it`.

## 4. Common errors

| Response | What to do |
|---|---|
| `locked_by_other` / task busy | Move to another task; report the owner and `retry_after_seconds` |
| `lock_expired` / `stale_lock` | Acquire again and retry once |
| `lock_required` | `pm_acquire_task` is missing; acquire and resend with `leaseId` and `fencingToken` |
| `dependencies_unmet` | Work on the dependencies first or tell the user it is blocked by them |
| `agent_identity_required` | The session does not use its own agent token; tell the user |
| Project or task not found | Check with `pm_list_projects` / `search`; don't invent IDs |
| `plan_limit_reached` / `plan_feature_unavailable` (MCP error or HTTP 402) | The account exceeded a plan limit (projects, agents, collaborators, tasks per project) or a feature not included. **Don't retry or create alternative tasks to get around it.** Tell the user which limit it is (`resource`, `limit`, `used`) and that they can upgrade their plan or free up space in the app. Carry on with what you can do |
| `project_read_only` / `collaborator_read_only` / `agent_suspended_by_plan` (403) | The account's plan left that project read-only, that collaborator as a reader, or that agent suspended. You can still **read**; don't try to write another way. Tell the user: they can choose what stays active in "Plan and usage" or upgrade the plan |
| 401 / 403 | Credentials or permissions: report the exact message and stop |

## 5. What you must never do
- Ask for the full tree of a large project.
- Mark `done` without verification.
- Create duplicate tasks or tasks with vague titles ("various fixes").
- Leave leases open when you finish.
- Force or ignore another agent's lease.
- Write tokens or secrets in MarQ or in the repository.
- Retry or work around a plan limit (`plan_limit_reached`): report it and carry on.
- Modify many tasks without showing a summary of what was done afterwards.
