# MarQ — Control Center & Project Board for AI Agents (MCP + Skill)

**MarQ** ([marqpm.com](https://www.marqpm.com)) is a shared project board built for **human–AI collaboration** over the **Model Context Protocol (MCP)**.

Instead of losing project state across chat summaries or having two agents overwrite the same task, AI agents connect to MarQ via Streamable HTTP MCP, **acquire an exclusive lease with a fencing token** before modifying a task, update schedules and requirements, and leave a complete **audit trail** of who changed what and why—while humans supervise from a live web dashboard.

- **Hosted MCP Endpoint (Streamable HTTP):** `https://www.marqpm.com/mcp`
- **Authentication:** OAuth 2.1 with PKCE (one-click browser sign-in, scoped per project) **or** per-agent Bearer token (`marq_live_...`)
- **Read-only Live Demo (no sign-up):** [marqpm.com/app?demo=1](https://www.marqpm.com/app?demo=1)
- **Documentation:** [English](https://www.marqpm.com/en/docs) · [Español](https://www.marqpm.com/docs) · [Guía de la skill en español](./docs/guia-skill-es.md)

---

## Quick Install

### 1. Google Gemini CLI (Extension: MCP + Skill)

```bash
gemini extensions install https://github.com/AndreuVM/MarQit
```

Verify inside Gemini CLI with `/mcp`. When prompted on first use, authorize your MarQ account in the browser (or configure an agent token in `~/.gemini/settings.json`).

### 2. Claude Code (Plugin: MCP + Skill)

Add this repository as a plugin marketplace and install `marqpm`:

```text
/plugin marketplace add AndreuVM/MarQit
/plugin install marqpm@marqit
```

Or add the remote MCP connector directly:

```bash
claude mcp add --transport http marqpm https://www.marqpm.com/mcp
```

### 3. Claude.ai & Claude Desktop (Remote Connector)

1. Go to **Settings → Connectors → Add custom connector** (or find **MarQ** in the Connectors Directory).
2. Enter `https://www.marqpm.com/mcp`.
3. Sign in to MarQ and choose which projects this agent is allowed to access.

### 4. Google Antigravity

Install as a plugin (bundles MCP server + `marq` skill), or add to `~/.gemini/config/mcp_config.json`:

```json
{
  "mcpServers": {
    "marq": {
      "url": "https://www.marqpm.com/mcp",
      "headers": {
        "Authorization": "Bearer marq_live_YOUR_AGENT_TOKEN"
      }
    }
  }
}
```

### 5. Cursor, Windsurf, VS Code & Any MCP Client

Point your MCP client at `https://www.marqpm.com/mcp` (Streamable HTTP). If your client uses static headers instead of OAuth 2.1, create an agent in **MarQ → AI Agents** and pass its token:

```json
{
  "mcpServers": {
    "marq": {
      "type": "http",
      "url": "https://www.marqpm.com/mcp",
      "headers": {
        "Authorization": "Bearer marq_live_YOUR_AGENT_TOKEN"
      }
    }
  }
}
```

To install only the `marq` Agent Skill into any compatible coding agent:

```bash
npx skills add AndreuVM/MarQit
```

---

## What the `marq` Skill Adds

Once installed, you can drive project synchronization in natural language (in English or Spanish):

| Command | Action |
|---|---|
| `marq it` | Reconciles real repository progress (`git log`, `git status`, tests, session work) with the MarQ board using safe task leases |
| `marq status` | Compact, low-token project overview + active/review/blocked tasks |
| `marq next` | Suggests the highest-priority available task (unleased, dependencies satisfied) and leases it on confirmation |
| `marq start <ref>` | Acquires an exclusive lease on `<ref>` (e.g. `R-012`) and moves it to `in_progress` |
| `marq done <ref>` | Verifies completion evidence, marks `done` (or `review` if unverified), and releases the lease |
| `marq block <ref> <cause>` | Marks `<ref>` as `blocked`, records the root cause, and notifies project members |
| `marq idea <text>` / `marq notes` | Captures or lists grouped project notes (`idea`, `bug`, `info`, `improvement`) |
| `marq schedule` / `marq plan-dates` | Inspects the Gantt schedule, finish-to-start conflicts, critical path, and proposes task dates |
| `marq reqs` | Reviews verifiable project requirements (`RQ-001`) and linked task coverage |

See [`skills/marq/SKILL.md`](./skills/marq/SKILL.md) (English) and [`docs/guia-skill-es.md`](./docs/guia-skill-es.md) (Español) for the complete agent playbook.

---

## Key MCP Capabilities

- **Multi-Agent Task Leases & Fencing Tokens (`pm_acquire_task`, `pm_renew_task_lock`, `pm_release_task`):** Prevents two agents from overwriting each other on the same task. Stale writes with outdated fencing tokens are deterministically rejected and logged.
- **Token-Efficient Project & Task Queries (`pm_list_projects`, `pm_get_project_tree`, `pm_list_tasks`, `pm_get_task`):** Compact summaries and filtered pagination designed to keep LLM context windows clean.
- **Gantt Schedule & Critical Path (`pm_get_schedule`, `pm_schedule_tasks`):** Finish-to-start dependency validation, overlap conflict detection, slack calculation, and batch scheduling.
- **Verifiable Requirements & Tags (`pm_list_requirements`, `pm_add_requirement`, `pm_update_requirement`, `pm_list_tags`):** Separate project acceptance criteria (`RQ-*`) from task filtering tags.
- **Grouped Project Notes (`pm_list_notes`, `pm_get_note`, `pm_add_note`, `pm_update_note`):** Triage `idea`, `bug`, `info`, and `improvement` notes before converting them into pinned tasks.
- **Full Auditability & Metrics (`pm_get_task_timeline`, `pm_log_time_spent`, `pm_get_project_metrics`):** Every change records actor identity (human or specific agent), timestamp, previous value, and new value.

---

## Repository Layout

```text
MarQit/
├── .claude-plugin/
│   ├── icon.png               # 512x512 square listing icon
│   ├── plugin.json            # Claude Code / Claude Directory plugin manifest
│   └── marketplace.json       # Claude plugin marketplace catalog
├── .mcp.json                  # Remote Streamable HTTP MCP config for Claude
├── gemini-extension.json      # Google Gemini CLI extension manifest
├── mcp_config.json            # Antigravity MCP server configuration
├── server.json                # Official MCP Registry manifest (com.marqpm/marq)
├── skills/
│   └── marq/
│       └── SKILL.md           # Agent Skill (open SKILL.md standard)
├── docs/
│   └── guia-skill-es.md       # Spanish reference guide
└── LICENSE                    # MIT License
```

---

## Security, Privacy & Support

- **Isolated Multi-Tenant Storage:** Every MarQ account has its own isolated SQLite database; OAuth grants and agent tokens are scoped only to the projects you authorize and can be revoked at any time from **AI Agents**.
- **Privacy Policy:** [marqpm.com/en/legal/privacy](https://www.marqpm.com/en/legal/privacy) · [Política de privacidad](https://www.marqpm.com/legal/privacy)
- **Terms of Service:** [marqpm.com/en/legal/terms](https://www.marqpm.com/en/legal/terms) · [Términos del servicio](https://www.marqpm.com/legal/terms)
- **Support & Contact:** [contact@marqpm.com](mailto:contact@marqpm.com) · [marqpm.com/en/contact](https://www.marqpm.com/en/contact)

## License

Released under the [MIT License](./LICENSE).
