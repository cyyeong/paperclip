# Product Requirements Document — Paperclip

> **Reverse-engineered from repository [`paperclipai/paperclip`](https://github.com/paperclipai/paperclip) as of 2026-04-07.**
> Source of truth: implementation code, not documentation alone. All claims verified against source unless marked otherwise.

---

## 1. Executive Summary

Paperclip is an open-source **control plane for autonomous AI agent companies**. It orchestrates multiple AI agents as employees within a company structure — complete with org charts, budgets, governance, task management, and heartbeat-driven execution scheduling. Paperclip does **not** execute agent code itself; it triggers agents via adapters (CLI wrappers, HTTP webhooks, subprocess spawners) and tracks their work, costs, and state.

A single Paperclip instance supports **multiple companies**, each fully isolated. Each company has a goal hierarchy, a tree-structured org chart of AI agents, a task system with atomic single-assignee checkout, budget enforcement with auto-pause, approval governance gates, and a full activity audit trail.

**Technology stack**: Node.js 20+ / TypeScript monorepo, Express.js 5 REST API, React 19 + Vite 6 SPA, PostgreSQL 17 (or embedded PGlite), Drizzle ORM, Better Auth, pnpm 9 workspaces.

**Distribution**: npm package (`paperclipai`), Docker image (`ghcr.io/paperclipai/paperclip`), or from source.

**Current state (2026-04-07)**: V1 is substantially implemented. The core control plane (companies, agents, issues, heartbeats, budgets, approvals, adapters, routines, plugins) is functional. A plugin SDK exists with worker isolation. Nine adapter types are available (7 shipped as packages, 2 as plugin-only external adapters). Public API documentation pages are stubs. The planned marketplace (ClipHub) is unimplemented. A teams/milestones/initiatives layer described in `doc/TASKS.md` is not yet in the database schema.

---

## 2. System Overview

Paperclip operates as two conceptual layers:

### Layer 1 — Control Plane (Paperclip server)

The central nervous system. Manages:

| Domain | Capability |
|--------|-----------|
| **Company management** | Multi-company tenancy, company goals, branding, settings |
| **Agent registry** | Hire/configure/pause/terminate agents; org chart hierarchy |
| **Task system** | Issues with atomic checkout, status lifecycle, parent/child hierarchy, blocking relations |
| **Heartbeat scheduling** | Timer-based polling loop (default 30s) that checks agent heartbeat policies and triggers wakeups |
| **Routine automation** | Cron-scheduled, webhook-triggered, or API-triggered recurring task templates |
| **Budget enforcement** | Per-agent/company/project budgets with soft alerts (80%) and hard-stop auto-pause (100%) |
| **Governance** | Approval gates for agent hiring and CEO strategy; board override powers |
| **Cost tracking** | Token/LLM cost events per run, aggregated by agent/provider/model/project |
| **Activity audit** | Full mutation log with actor, action, entity, before/after details |
| **Plugin system** | External code extensions with worker isolation, event subscriptions, scheduled jobs, webhooks, UI slots |
| **Secrets management** | Encrypted at-rest secret storage with versioning and rotation |
| **Real-time events** | WebSocket (and EventEmitter-based) company-scoped event streaming |
| **Storage** | Pluggable file storage (local disk or S3-compatible) |

### Layer 2 — Execution Services (Adapters)

Agents run externally. Adapters bridge Paperclip and agent runtimes:

| Adapter | Mechanism | Status |
|---------|-----------|--------|
| `claude_local` | Spawns `claude` CLI process | **Implemented** |
| `codex_local` | Spawns `codex` CLI process | **Implemented** |
| `cursor` | Spawns Cursor CLI | **Implemented** |
| `gemini_local` | Spawns Gemini CLI | **Implemented** |
| `opencode_local` | Spawns `opencode` CLI | **Implemented** |
| `pi_local` | Spawns Pi CLI | **Implemented** |
| `openclaw_gateway` | HTTP to OpenClaw gateway | **Implemented** |
| `process` | Generic subprocess spawner | **Implemented** |
| `http` | HTTP webhook adapter | **Implemented** |
| `hermes_local` | External plugin adapter | **Plugin-only** (not built-in; HenkDz fork) |
| `droid_local` | External plugin adapter | **Plugin-only** (referenced in registry comments) |

---

## 3. Problem Statement and Intended Value

### Problem

When an entire workforce is AI agents, standard task management tools are insufficient. The gaps:

1. **No organizational structure** — agents operate as isolated scripts, not as coordinated employees
2. **No budget control** — runaway token costs with no automatic enforcement
3. **No governance** — no approval gates for consequential decisions (hiring, strategy)
4. **No persistent context** — agents lose state across invocations; no multi-turn session continuity
5. **No atomic task ownership** — multiple agents can collide on the same work
6. **No audit trail** — mutations happen invisibly without accountability
7. **No recurring automation** — no cron-like scheduling for agent work

### Intended Value

Paperclip aims to be the **default operating system for AI companies** — a single pane of glass where a human operator can:

- See the entire company at a glance (who's doing what, how much it costs, whether it's working)
- Delegate to a CEO agent that autonomously breaks goals into tasks, hires agents, and manages execution
- Enforce financial and governance controls automatically
- Import/export portable company templates as markdown packages

**Success metric** (from `doc/GOAL.md`): Paperclip becomes the default foundation for thousands of autonomous companies.

---

## 4. Current Repository Scope

| Area | Status | Evidence |
|------|--------|---------|
| Company CRUD + multi-tenancy | **Implemented** | `companies` schema, routes, services |
| Agent lifecycle (hire/pause/terminate) | **Implemented** | `agents` schema, 7+ agent statuses |
| Org chart (strict tree hierarchy) | **Implemented** | `reportsTo` FK, `/api/companies/:id/org` endpoint |
| Issue system with atomic checkout | **Implemented** | `issues` schema, checkout route, 409 conflict handling |
| Issue blocking relations | **Implemented** | `issue_relations` schema |
| Heartbeat scheduler loop | **Implemented** | `setInterval` in `index.ts`, `tickTimers()`, 30s default |
| Routine scheduling (cron) | **Implemented** | `routines`/`routine_triggers` schema, `tickScheduledTriggers()` |
| Budget enforcement + auto-pause | **Implemented** | `budget_policies`, `budget_incidents`, `evaluateCostEvent()` |
| Approval governance | **Implemented** | `approvals` schema, hire + CEO strategy types |
| Cost tracking | **Implemented** | `cost_events` schema, provider/model/token breakdown |
| Activity audit log | **Implemented** | `activity_log` schema, logged on all mutations |
| Secrets encryption | **Implemented** | `company_secrets`/`company_secret_versions`, local_encrypted provider |
| Plugin system with worker isolation | **Implemented** | SDK, worker manager, job scheduler, event bus, 4 example plugins |
| External adapter plugins | **Implemented** | Plugin loader, runtime registry, override/pause mechanism |
| WebSocket real-time events | **Implemented** | `ws` server, company-scoped subscriptions |
| Storage abstraction | **Implemented** | Local disk + S3 providers |
| Auth (sessions + API keys) | **Implemented** | Better Auth, hashed agent API keys (`pcp_` prefix) |
| CLI (onboard/configure/run) | **Implemented** | 15+ commands in `cli/src/commands/` |
| Company import/export | **Implemented** | Agent Companies spec v1, CLI + API endpoints |
| Execution workspaces (git worktree) | **Implemented** | `execution_workspaces` schema, workspace runtime services |
| Feedback/voting system | **Implemented** | `feedback_votes`, trace bundles, export |
| Agent eval framework | **Partial** | Promptfoo Phase 0 with 8 core cases |
| Public API docs | **Stub only** | All files in `docs/api/` have empty frontmatter |
| Adapter-specific docs | **Stub only** | All files in `docs/adapters/` have empty frontmatter |
| CLI reference docs | **Stub only** | All files in `docs/cli/` have empty frontmatter |
| Teams/milestones/initiatives (from TASKS.md) | **Not implemented** | No `teams` table in schema; doc describes target model |
| ClipHub marketplace | **Not implemented** | Design doc in `doc/CLIPHUB.md`, no code |
| Multi-member boards | **Not implemented** | Spec says V1 = single human board |
| Self-healing / auto-recovery | **Not implemented** | Listed as out-of-scope in `doc/PRODUCT.md` |
| MCP server integration | **Not implemented** | `doc/TASKS-mcp.md` defines MCP function contracts; no server code |

---

## 5. Personas / Actors / Operators

### Board Operator (Human)

The single human who controls the Paperclip instance. In `local_trusted` mode, this is an implicit local user. In `authenticated` mode, this is a Better Auth-authenticated user with instance admin role.

**Powers**: Full CRUD on all entities. Approve/reject agent hires and CEO strategy. Pause/resume/terminate any agent. Override budgets. Reassign tasks. Direct access to all companies in the instance.

**Primary interface**: React web UI at the server's HTTP port (default 3100).

### Agent (AI)

An AI agent registered in a company. Authenticates via bearer API key (hashed at rest, `pcp_` prefix) or run-scoped JWT (for local adapters).

**Powers**: Scoped to own company. Can read own identity, assignments, issue details. Can checkout/update/comment on owned tasks. Can create subtasks (delegation). Can request hires (managers/CEOs). Can report costs. Cannot access other companies.

**Primary interface**: REST API calls during heartbeat execution windows.

### CLI User

A human or automation using the `paperclipai` CLI tool. Authenticated via session token or API key.

**Powers**: Onboard/configure instances, create/manage companies, list/create/update issues, approve/reject hiring requests, manage plugins.

### Plugin (Extension Code)

Third-party or first-party code running in isolated worker processes. Communicates with the host via JSON-RPC 2.0 protocol.

**Powers**: Subscribe to domain events, register tools, run scheduled jobs, receive webhooks, expose UI slots, store scoped state. Cannot bypass company isolation.

---

## 6. Primary Use Cases and End-to-End Workflows

### UC-1: Create and Launch a Company

1. Board operator opens web UI, creates a company with name and description
2. Sets a top-level company goal (specific, measurable)
3. Creates a CEO agent: selects adapter type (e.g. `claude_local`), configures model, working directory, prompt/instructions, budget
4. Builds org chart: creates direct reports (CTO, engineers, etc.), each with adapter config and budget
5. Enables heartbeats for agents
6. CEO wakes on first heartbeat, reads company goal, proposes strategy via approval request
7. Board approves CEO strategy
8. CEO breaks strategy into tasks, assigns to agents, monitors progress
9. Agents wake on assignment, checkout tasks, execute work, update status

**Key invariant**: Atomic single-assignee checkout prevents two agents from working the same task. `409 Conflict` returned on collision.

### UC-2: Agent Heartbeat Execution Cycle

1. Server's `setInterval` loop (30s default) calls `heartbeat.tickTimers(now)`
2. For each active agent with elapsed heartbeat interval, `enqueueWakeup()` inserts an `agentWakeupRequests` row
3. `resumeQueuedRuns()` picks queued wakeups, serializes per-agent via `withAgentStartLock()`
4. Budget check: `budgetService.getInvocationBlock()` — if any scope (agent/company/project) is paused, block invocation
5. Session resolution: load prior `agentTaskSessions` for session continuity, resolve workspace (project primary or task session)
6. Adapter execution: `registry.getServerAdapter(type).execute(context)` — spawns CLI process or sends HTTP request
7. Agent process runs: agent calls Paperclip REST API to get assignments, checkout task, do work, update status
8. Result capture: adapter parses stdout/exit code, extracts token usage, cost, session state, summary
9. Cost recording: `costService.createEvent()` inserts cost event; `budgetService.evaluateCostEvent()` checks thresholds
10. Session persistence: session params saved for next wake; live events emitted for UI updates
11. Activity logged for audit trail

### UC-3: Routine-Triggered Automation

1. Board operator creates a routine with cron trigger (e.g. `0 */6 * * *` — every 6 hours)
2. Server's `setInterval` calls `routines.tickScheduledTriggers(now)` each tick
3. For due triggers: routine creates an issue from template with interpolated variables
4. Issue assigned to configured agent; heartbeat wakeup queued
5. Agent wakes, checks out issue, executes work
6. Concurrent runs coalesced — if prior run still live, new trigger skips or queues based on concurrency policy
7. Catch-up: missed triggers backfilled up to 25 runs

### UC-4: Budget Hard-Stop

1. Agent completes a heartbeat run; adapter reports 50,000 tokens consumed
2. `costService.createEvent()` records the cost event
3. `budgetService.evaluateCostEvent()` calculates total spend for agent's monthly window
4. If spend ≥ 80% of budget: soft alert (activity log entry, incident record)
5. If spend ≥ 100% of budget: hard stop — agent auto-paused, in-flight work cancelled via `hooks.cancelWorkForScope()`
6. Budget incident created; board operator notified
7. Board can approve budget override to resume, which raises limit and unpauses

### UC-5: Agent Hiring via Governance

1. CEO agent decides a new engineer is needed
2. CEO calls `POST /api/companies/{companyId}/agent-hires` with proposed agent config
3. Approval record created with status `pending`; board operator sees it in Approvals UI
4. Board reviews hire request: name, role, adapter type, capabilities, budget
5. Board approves → agent created, added to org chart under requester
6. Board rejects → approval closed, CEO notified on next heartbeat

### UC-6: Company Import/Export

1. Export: `paperclipai company export <company-id> --out ./my-company/`
2. Produces markdown package: `COMPANY.md`, `agents/<slug>/AGENTS.md`, `projects/`, `skills/`, `.paperclip.yaml` sidecar
3. Import: `paperclipai company import ./my-company/` or from GitHub URL
4. Preview shows what will be created; collision strategies: rename/skip/replace
5. On apply: companies, agents, projects, skills created in Paperclip instance

---

## 7. Architecture Overview

### Monorepo Structure

```
paperclip/                          # pnpm 9 workspace root
├── server/                         # Express.js REST API + orchestration
├── ui/                             # React 19 + Vite 6 SPA
├── cli/                            # CLI tool (paperclipai)
├── packages/
│   ├── db/                         # Drizzle ORM schema + migrations (75 tables)
│   ├── shared/                     # Types, constants, validators, API paths
│   ├── adapter-utils/              # Adapter interface contracts
│   ├── adapters/                   # 7 adapter packages
│   │   ├── claude-local/
│   │   ├── codex-local/
│   │   ├── cursor-local/
│   │   ├── gemini-local/
│   │   ├── opencode-local/
│   │   ├── pi-local/
│   │   └── openclaw-gateway/
│   └── plugins/                    # Plugin SDK + examples
│       ├── sdk/
│       ├── create-paperclip-plugin/
│       └── examples/               # 4 example plugins
├── skills/                         # Agent skills (SKILL.md files)
├── evals/                          # Promptfoo eval framework
├── scripts/                        # Build, dev, release automation
├── tests/                          # E2E Playwright tests
├── doc/                            # Internal design documents
└── docs/                           # Public Mintlify documentation site
```

### Request Flow Diagram

```
Human Operator ──→ React UI ──→ Express REST API ──→ PostgreSQL
                                    │
                                    ├── Heartbeat Scheduler (setInterval 30s)
                                    │       │
                                    │       ▼
                                    │   tickTimers() → enqueueWakeup()
                                    │       │
                                    │       ▼
                                    │   resumeQueuedRuns() → withAgentStartLock()
                                    │       │
                                    │       ▼
                                    │   Adapter Registry → getServerAdapter(type)
                                    │       │
                                    │       ▼
                                    │   adapter.execute(context)
                                    │       │
                                    │       ▼
                                    │   Agent Process (CLI/HTTP)
                                    │       │
                                    │       ▼
                                    │   Agent calls REST API ──→ checkout, update, comment
                                    │       │
                                    │       ▼
                                    │   Result capture → cost tracking → session save
                                    │
                                    ├── Routine Scheduler (same interval)
                                    │       │
                                    │       ▼
                                    │   tickScheduledTriggers() → create issue → wakeup
                                    │
                                    ├── Plugin System
                                    │       │
                                    │       ▼
                                    │   Worker Manager → JSON-RPC → Plugin Workers
                                    │
                                    └── WebSocket Server
                                            │
                                            ▼
                                        Company-scoped live events → UI
```

### Key Architectural Decisions

| Decision | Rationale | Source |
|----------|-----------|--------|
| Control plane, not execution plane | Agents run wherever they run; Paperclip orchestrates, not hosts | `doc/GOAL.md`, `doc/PRODUCT.md` |
| Company-scoped isolation | Every entity belongs to exactly one company; strict tenant boundaries | Schema FKs, route guards, `SPEC-implementation.md` |
| Single-assignee atomic checkout | Prevents concurrent work; SQL-level race protection, 409 on conflict | `issues.ts` service, checkout route |
| Adapter-agnostic | Any runtime that can call HTTP works; no vendor lock-in | `adapter-utils/types.ts` interface |
| Embedded DB by default | Zero-config local mode; `DATABASE_URL` unset → PGlite auto-starts | `config.ts`, `index.ts` startup |
| Event-driven + polling hybrid | 30s polling loop for heartbeats/routines; event-driven for assignments/comments | `index.ts` setInterval, wakeup requests |
| Plugin worker isolation | Plugins run in separate processes; host bridges via JSON-RPC 2.0 | `plugin-worker-manager.js`, `protocol.ts` |

---

## 8. Major Components and Responsibilities

### 8.1 Server (`server/`)

**Entry point**: `server/src/index.ts` — orchestrates startup:
1. Load config (env vars → YAML file → defaults)
2. Initialize database (embedded PGlite or external Postgres)
3. Apply migrations (auto or prompted)
4. Create Express app via `app.ts` factory
5. Start HTTP server + WebSocket server
6. Start heartbeat/routine scheduler loop (`setInterval`)
7. Reconcile runtime services

**Express app** (`server/src/app.ts`):
- Middleware pipeline: JSON parser (10MB limit) → HTTP logger → hostname guard → actor middleware (auth) → CSRF guard
- 24+ route groups mounted at `/api`
- Plugin UI static routes
- UI serving (Vite dev middleware or static SPA)

**Services layer** (`server/src/services/`) — 60+ files:

| Service | Responsibility |
|---------|---------------|
| `heartbeat.ts` | Core execution orchestrator — wakeup queuing, session resolution, adapter invocation, result capture, cost tracking |
| `routines.ts` | Cron parsing, scheduled trigger evaluation, variable interpolation, routine run management |
| `agents.ts` | Agent CRUD, API key management (SHA256 hashed), config revision tracking, wakeup dispatch |
| `issues.ts` | Issue lifecycle, status transitions, atomic checkout with lock validation, relation management |
| `budgets.ts` | Multi-scope budget policies, threshold evaluation (80% warn / 100% hard-stop), auto-pause, incident tracking |
| `costs.ts` | Cost event recording, multi-dimensional aggregation (by agent/provider/model/project), window analysis |
| `approvals.ts` | Approval workflow: pending → approved/rejected/revision_requested |
| `access.ts` | RBAC: company memberships, permission grants, instance admin bypass |
| `activity-log.ts` | Mutation audit logging for all domain operations |
| `live-events.ts` | EventEmitter-based real-time event broadcast (company-scoped) |
| `secrets.ts` | Secret storage with versioning, encryption delegation, sensitive key detection |
| `plugin-lifecycle.ts` | Plugin state machine (installed → ready → disabled → error → uninstalled) |
| `plugin-worker-manager.js` | Process pool for plugin workers |
| `plugin-job-scheduler.ts` | Cron-based scheduled jobs for plugins |
| `plugin-tool-dispatcher.ts` | Routes tool calls to appropriate plugin workers |
| `plugin-event-bus.ts` | Event routing to subscribed plugins |
| `workspace-runtime.ts` | Execution workspace provisioning (git worktree, local fs) |
| `local-service-supervisor.ts` | Process supervision for workspace runtime services |
| `board-auth.ts` | Board user session and key management |
| `feedback.ts` | Vote collection, trace bundles, export management |
| `documents.ts` | Document/revision management for issues |
| `finance.ts` | Billing and finance event tracking |

**Middleware** (`server/src/middleware/`):

| Middleware | Function |
|-----------|----------|
| `auth.ts` | Extracts actor context (board/agent/none) from JWT, API key, or session cookie |
| `error-handler.ts` | Maps HttpError, ZodError to JSON responses |
| `logger.ts` | Pino HTTP logging with dual output (console + file) |
| `validate.ts` | Zod schema validation for request bodies |
| `board-mutation-guard.ts` | CSRF protection for board POST/PUT/DELETE |
| `private-hostname-guard.ts` | Hostname allowlisting for authenticated private mode |

**Adapter subsystem** (`server/src/adapters/`):
- `registry.ts` — Central adapter registry; supports built-in and external (plugin-loaded) adapters with override/pause/fallback
- `plugin-loader.ts` — Dynamic loading of external adapters from npm packages or local file paths
- `http/` — HTTP webhook adapter implementation
- `process/` — Generic subprocess spawner

### 8.2 UI (`ui/`)

React 19 SPA with Vite 6 build tooling. Served by the Express server (via Vite dev middleware in dev, static files in production).

**Key technology**: TanStack Query for server state, Radix UI + Tailwind CSS 4 + shadcn/ui for components, Lexical/MDXEditor for rich text, dnd-kit for drag-and-drop.

**Page structure** (~40 pages):
- Dashboard, Activity log, Org chart
- Agent management (list, detail, create, adapter manager)
- Issue management (list, detail, kanban, my issues)
- Project management (list, detail, workspace detail)
- Routines (list, detail)
- Approvals (list, detail)
- Goals (list, detail)
- Cost tracking
- Company settings, instance settings
- Company import/export
- Plugin manager
- Auth, board claim, CLI auth
- Design guide showcase

**Context providers** (10): Company scope, live updates (SSE), theme, breadcrumbs, dialogs, sidebar, editor autocomplete, general settings, panel, toast.

**API client layer** (`ui/src/api/`) — 23 modules wrapping REST endpoints with TanStack Query integration.

### 8.3 CLI (`cli/`)

Published as `paperclipai` on npm. Built with esbuild into a single executable. Uses `commander.js` for command structure and `@clack/prompts` for interactive prompts.

**Command categories**:

| Category | Commands |
|----------|---------|
| Setup | `onboard` (first-run wizard), `configure`, `doctor` (with `--repair`), `run`, `env` |
| Company | `company create/list/export/import/skills` |
| Issues | `issue list/create/update` |
| Agents | `agent list/hire` |
| Approvals | `approval list/approve/reject` |
| Activity | `activity list` |
| Dashboard | `dashboard` |
| Auth | `auth bootstrap-ceo`, `allowed-hostname` |
| Plugins | `plugin install/uninstall/enable/disable` |
| Context | `context set/show` (manage connection profiles) |
| Database | `db:backup` |
| Heartbeat | `heartbeat run` (manual single execution) |

### 8.4 Database Package (`packages/db/`)

Drizzle ORM schema with **75 exported tables** across 61 schema files. PostgreSQL 17 dialect. Migrations generated via `drizzle-kit` from compiled schema.

**Core domain tables**: `companies`, `agents`, `agent_api_keys`, `issues`, `issue_relations`, `issue_comments`, `goals`, `projects`, `approvals`, `routines`, `routine_triggers`, `heartbeat_runs`, `cost_events`, `activity_log`, `budget_policies`

**Plugin tables**: `plugins`, `plugin_config`, `plugin_company_settings`, `plugin_state`, `plugin_entities`, `plugin_jobs`, `plugin_job_runs`, `plugin_webhooks`, `plugin_logs`

**Auth tables**: `auth_users`, `auth_sessions`, `auth_accounts`, `auth_verifications`, `board_api_keys`, `company_memberships`, `principal_permission_grants`, `invites`, `join_requests`, `instance_user_roles`

**Execution tables**: `execution_workspaces`, `workspace_operations`, `workspace_runtime_services`, `agent_task_sessions`, `agent_wakeup_requests`, `agent_runtime_state`

### 8.5 Shared Package (`packages/shared/`)

Single source of truth for cross-package contracts:
- **Constants**: 60+ status/enum arrays (`AGENT_STATUSES`, `ISSUE_STATUSES`, `ADAPTER_TYPES`, `AGENT_ROLES`, etc.)
- **Types**: TypeScript interfaces for all domain entities (agent, issue, company, approval, budget, routine, plugin, cost, etc.)
- **Validators**: Zod schemas for API request validation
- **API paths**: Route constant strings (`/api/companies`, `/api/agents`, etc.)
- **Config schema**: Zod schemas for server configuration (database, storage, secrets, auth)

### 8.6 Adapter Utils (`packages/adapter-utils/`)

Defines the **adapter contract** that all adapter implementations must fulfill:

| Interface | Purpose |
|-----------|---------|
| `AdapterExecutionContext` | Input to `execute()`: runId, agent, runtime session, config, logging callbacks |
| `AdapterExecutionResult` | Output: exitCode, signal, usage (tokens), cost, session state, result JSON |
| `AdapterSessionCodec` | Session serialization: `deserialize()`, `serialize()`, `getDisplayId()` |
| `AdapterEnvironmentTestResult` | Diagnostic checks for adapter prerequisites |
| `AdapterInvocationMeta` | Logging metadata: command, cwd, env, prompt |

Also provides: session compaction utilities, log redaction, billing inference, CLI invocation helpers.

### 8.7 Adapter Packages (`packages/adapters/`)

Seven adapter packages, each with three modules:

| Module | Exports | Used by |
|--------|---------|---------|
| **Server** | `execute()`, `testEnvironment()`, `listSkills()`, `syncSkills()`, `sessionCodec`, `models`, `detectModel()` | Server heartbeat execution |
| **UI** | stdout parser for run viewer, config form fields, model picker | React UI |
| **CLI** | Terminal formatter for `--watch` mode | CLI tool |

**Common configuration fields** across adapters: `cwd`, `instructionsFilePath`, `model`, `env` (with secret refs), `workspaceStrategy`, `timeoutSec`, `graceSec`.

### 8.8 Plugin System (`packages/plugins/`)

Full plugin SDK with worker isolation:

- **`definePlugin()`** function: registers setup, health, config validation, webhook, shutdown handlers
- **JSON-RPC 2.0 protocol** between host and worker
- **Capabilities**: `event.subscribe`, `agent.tools.register`, `job.schedule`, `webhook.receive`, `ui.slot`, `launcher.register`
- **State scoping**: instance, company, project, issue — namespaced key-value store
- **Lifecycle**: installed → ready → disabled → error → upgrade_pending → uninstalled
- **Capability approval**: upgraded plugins with new capabilities require operator approval before activation
- **Job scheduler**: cron-based with run history
- **Event bus**: domain events (issue.created, agent.status_changed, etc.) routed to subscribed plugins
- **4 example plugins**: hello-world, kitchen-sink, file-browser, authoring-smoke

---

## 9. Core Engine / Runtime / Orchestration Model

This is the most critical section. Paperclip's core engine is **not** an agent runtime — it is a scheduling and orchestration loop that coordinates external agent processes.

### 9.1 Entry Point and Startup Sequence

**File**: `server/src/index.ts`

1. **Config load** — `loadConfig()` reads env vars, `.paperclip.yaml`, `.env` files. Priority: env > file > defaults.
2. **Database init** — If `DATABASE_URL` is unset, starts embedded PGlite in `~/.paperclip/instances/default/db/`. Otherwise connects to external Postgres.
3. **Migration check** — Preflight query checks pending migrations. In dev mode, auto-applies. In production, requires explicit confirmation or `PAPERCLIP_MIGRATION_AUTO_APPLY=true`.
4. **Express app creation** — `createApp()` in `app.ts` constructs the full middleware + route stack.
5. **HTTP server start** — Binds to `config.port` (default 3100) on `config.host`.
6. **WebSocket server** — `ws` library WebSocketServer in no-server mode, upgraded from HTTP.
7. **Scheduler loop start** — `setInterval(callback, config.heartbeatSchedulerIntervalMs)` — the **central control loop**.
8. **Runtime reconciliation** — Checks and reconciles workspace runtime services.
9. **Board claim challenge** — If migrating from `local_trusted` to `authenticated`, emits claim URL.

### 9.2 The Central Control Loop

**File**: `server/src/index.ts`, lines ~589–620

The single `setInterval` (default 30s, configurable via `HEARTBEAT_SCHEDULER_INTERVAL_MS`, minimum 10s) drives three concurrent operations each tick:

```
Every 30 seconds:
  1. heartbeat.tickTimers(now)         → Check all agents for due heartbeats
  2. routines.tickScheduledTriggers(now) → Check all cron triggers for due runs
  3. heartbeat.reapOrphanedRuns()      → Clean up stale runs (5-min threshold)
     → heartbeat.resumeQueuedRuns()    → Process queued wakeup requests
```

This is a **pull-based** model: the server polls on a fixed interval rather than using event-driven scheduling. Immediate wakeups (assignment, mention, manual invoke) are handled via the same `enqueueWakeup()` → `resumeQueuedRuns()` path but triggered from API routes directly.

### 9.3 Heartbeat Tick — Agent Due Check

**File**: `server/src/services/heartbeat.ts`, `tickTimers()`

For each active agent in every company:
1. Skip if agent is paused, terminated, or pending approval
2. Skip if heartbeat is disabled in agent config
3. Parse heartbeat policy (interval in seconds from `heartbeatPolicy`)
4. Calculate elapsed time since `lastHeartbeatAt`
5. If elapsed ≥ interval: insert `agentWakeupRequests` row with `source: "timer"`

Returns `{ checked, enqueued, skipped }` counters.

### 9.4 Wakeup Queue Processing

**File**: `server/src/services/heartbeat.ts`, `resumeQueuedRuns()`

Processes the `agentWakeupRequests` table:
1. Query pending wakeup requests ordered by creation time
2. For each request, acquire per-agent start lock via `withAgentStartLock()`
3. Lock prevents concurrent runs of the same agent (promise-based, in-memory Map)
4. Check `MAX_CONCURRENT_RUNS` (default 1, max 10) — skip if at capacity
5. Proceed to execution pipeline

### 9.5 Execution Pipeline (Single Run)

**File**: `server/src/services/heartbeat.ts`

```
┌──────────────────────────────────────────────────────────────┐
│ 1. BUDGET GATE                                               │
│    budgetService.getInvocationBlock(agentId, companyId)      │
│    → If blocked (agent/company/project paused): abort        │
├──────────────────────────────────────────────────────────────┤
│ 2. SESSION RESOLUTION                                        │
│    deriveTaskKeyWithHeartbeatFallback()                      │
│    → Stable key: issueId if task-triggered, "__heartbeat__"  │
│      if timer-triggered                                      │
│    buildExplicitResumeSessionOverride()                      │
│    → Load prior agentTaskSessions row                        │
│    → Deserialize via adapter's sessionCodec                  │
│    shouldResetTaskSessionForWake()                           │
│    → Fresh session if forceFreshSession or issue_assigned     │
├──────────────────────────────────────────────────────────────┤
│ 3. WORKSPACE RESOLUTION                                      │
│    resolveRuntimeSessionParamsForWorkspace()                 │
│    → Prioritize: explicit workspace > project primary >      │
│      agent fallback (~/.paperclip/.../workspaces/<agentId>) │
│    ensureManagedProjectWorkspace()                           │
│    → Clone repo if needed (10-min timeout)                   │
├──────────────────────────────────────────────────────────────┤
│ 4. CONTEXT ENRICHMENT                                        │
│    enrichWakeContextSnapshot()                               │
│    → Add issue ID, task ID, comment IDs, wake reason        │
│    buildPaperclipWakePayload()                               │
│    → Inline issue summary + up to 8 comments (4KB each,     │
│      12KB total cap) for fat-context delivery               │
├──────────────────────────────────────────────────────────────┤
│ 5. ADAPTER INVOCATION                                        │
│    registry.getServerAdapter(agent.adapterType)              │
│    → Returns built-in or external (plugin) adapter          │
│    adapter.execute(executionContext)                          │
│    → Spawns CLI process / sends HTTP request                │
│    → Streams logs via onLog callback                        │
│    → Waits for exit                                          │
├──────────────────────────────────────────────────────────────┤
│ 6. RESULT PROCESSING                                         │
│    normalizeUsageTotals(result)                              │
│    → Extract input/cached/output token counts               │
│    costService.createEvent()                                │
│    → Insert cost_events row with provider/model/tokens/cost │
│    budgetService.evaluateCostEvent()                        │
│    → Check soft (80%) and hard (100%) thresholds            │
│    → Auto-pause if exceeded                                  │
├──────────────────────────────────────────────────────────────┤
│ 7. SESSION PERSISTENCE                                       │
│    Save sessionIdAfter, sessionParams to agentTaskSessions  │
│    Session compaction if size exceeds threshold              │
│    Update agent lastHeartbeatAt                              │
├──────────────────────────────────────────────────────────────┤
│ 8. LIVE EVENT EMISSION                                       │
│    publishLiveEvent(companyId, "run.completed", {...})       │
│    → WebSocket broadcast to connected UI clients            │
├──────────────────────────────────────────────────────────────┤
│ 9. ACTIVITY LOG                                              │
│    activityLog.record(actor, action, entity, details)       │
│    → Permanent audit trail entry                             │
└──────────────────────────────────────────────────────────────┘
```

### 9.6 Adapter Execution Model

**Files**: `packages/adapter-utils/src/types.ts`, `packages/adapters/*/src/server/execute.ts`

Each adapter's `execute()` function receives an `AdapterExecutionContext` containing:
- `runId` — unique run identifier
- `agent` — agent identity and config
- `runtime` — session params (sessionId, cwd, workspaceId)
- `config` — adapter-specific config blob
- `context` — wake reason, task info, inline payload
- `onLog(line)` — streaming log callback
- `onInvocationMeta(meta)` — records command, cwd, env for debugging

Returns `AdapterExecutionResult`:
- `exitCode`, `signal` — process termination status
- `usage` — `{ inputTokens, cachedInputTokens, outputTokens }`
- `cost` — `{ costCents, provider, biller, model, billingType }`
- `sessionIdAfter` — session ID for continuation
- `sessionParamsAfter` — serialized session state
- `result` — optional JSON result payload

**CLI adapter pattern** (claude_local, codex_local, etc.):
1. Build environment variables (Paperclip agent vars + LLM provider keys + adapter-specific)
2. Prepare skills directory (symlinks to skill files)
3. Spawn CLI process with constructed arguments
4. Stream stdout/stderr through parser
5. Parse JSON stream for session ID, model, tokens, cost, summary, errors
6. Return normalized result

### 9.7 Session Continuity

**Sessioned adapters**: `claude_local`, `codex_local`, `cursor`, `gemini_local`, `opencode_local`, `pi_local`

Sessions are keyed by **task key** (issueId if task-triggered, `"__heartbeat__"` if timer-triggered):
- Session params stored in `agentTaskSessions` table per (agentId, taskKey)
- On next wake for same task, prior session params loaded and passed to adapter
- Adapter's `sessionCodec.deserialize()` reconstructs state (e.g. Claude session ID, Codex thread)
- If `forceFreshSession` flag set or trigger is `issue_assigned`: session reset

**Session compaction**: When session state exceeds size threshold, `resolveSessionCompactionPolicy()` determines compaction strategy (lazy, always, never).

### 9.8 Routine Scheduling Engine

**File**: `server/src/services/routines.ts`

Routines are templates for recurring work:
- Each routine has one or more **triggers** (cron schedule, webhook endpoint, API)
- Cron evaluation: `nextCronTickInTimeZone()` computes next fire time; `matchesCronMinute()` validates against zoned date
- **Variable interpolation**: `{{variable_name}}` in routine body templates resolved from trigger source, schedule defaults, or API payload
- **Concurrency policy**: configurable per-routine — skip if prior run live, or queue
- **Catch-up**: missed triggers backfilled up to `MAX_CATCH_UP_RUNS = 25`
- On trigger: creates issue from template → assigns to configured agent → queues wakeup

### 9.9 Orphan Recovery

**File**: `server/src/services/heartbeat.ts`, `reapOrphanedRuns()`

Runs every scheduler tick. Finds `heartbeat_runs` in non-terminal status older than 5 minutes without activity. Marks them as failed (orphaned). Then `resumeQueuedRuns()` picks up any queued wakeups that were blocked by the now-reaped run.

### 9.10 Real-Time Event System

**Files**: `server/src/services/live-events.ts`, `server/src/realtime/live-events-ws.ts`

- **EventEmitter-based**: In-memory, no persistence. `publishLiveEvent()` emits to company-scoped channel.
- **WebSocket server**: Authenticates via Bearer token (board key or agent API key). Parses company ID from URL path. Subscribes to company channel.
- **Event types**: run.queued, run.completed, agent.status_changed, issue.updated, etc. (defined in `@paperclipai/shared`).
- **Global events**: `publishGlobalLiveEvent()` broadcasts to `"*"` channel for instance-wide events.
- **No SSE endpoint found** — despite `LiveUpdatesProvider` name in UI context, transport appears to be WebSocket-only.

---

## 10. Configuration and Extensibility Model

### 10.1 Configuration Hierarchy

Configuration is resolved with the following priority (highest wins):

1. **Environment variables** — `PAPERCLIP_*` prefix, `PORT`, `HOST`, `DATABASE_URL`, LLM provider keys
2. **Config file** — `.paperclip/.paperclip.yaml` (or `PAPERCLIP_CONFIG_FILE` override)
3. **Dotenv files** — `.env` in cwd, `~/.paperclip/.env`
4. **Computed defaults** — home-aware paths, embedded DB, local disk storage

### 10.2 Environment Variables (Complete Reference)

**Server core**:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `3100` | HTTP server port |
| `HOST` | `127.0.0.1` | Bind address |
| `DATABASE_URL` | (unset → embedded) | PostgreSQL connection string |
| `PAPERCLIP_HOME` | `~/.paperclip` | Root data directory |
| `PAPERCLIP_INSTANCE_ID` | `default` | Instance isolation key |
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` | `local_trusted` or `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | `private` or `public` (authenticated mode only) |
| `PAPERCLIP_PUBLIC_URL` | (auto-detected) | Explicit public base URL |
| `HEARTBEAT_SCHEDULER_INTERVAL_MS` | `30000` | Scheduler tick interval (min 10000) |
| `PAPERCLIP_MIGRATION_AUTO_APPLY` | `false` | Auto-apply DB migrations |

**Secrets**:

| Variable | Purpose |
|----------|---------|
| `PAPERCLIP_SECRETS_MASTER_KEY` | Direct master key for encryption |
| `PAPERCLIP_SECRETS_MASTER_KEY_FILE` | Path to master key file |
| `PAPERCLIP_SECRETS_STRICT_MODE` | Enforce secret refs for sensitive keys |

**Agent runtime** (injected into agent processes):

| Variable | Purpose |
|----------|---------|
| `PAPERCLIP_AGENT_ID` | Agent UUID |
| `PAPERCLIP_COMPANY_ID` | Company UUID |
| `PAPERCLIP_API_URL` | Server base URL |
| `PAPERCLIP_API_KEY` | Bearer token (run JWT or API key) |
| `PAPERCLIP_RUN_ID` | Current heartbeat run UUID |
| `PAPERCLIP_TASK_ID` | Assigned task UUID (if task-triggered) |
| `PAPERCLIP_WAKE_REASON` | Trigger type (timer, issue_assigned, comment_mention, etc.) |
| `PAPERCLIP_WAKE_COMMENT_ID` | Comment that triggered wake |
| `PAPERCLIP_APPROVAL_ID` | Approval that triggered wake |
| `PAPERCLIP_APPROVAL_STATUS` | Approval resolution status |
| `PAPERCLIP_LINKED_ISSUE_IDS` | Comma-separated linked issue IDs |

**LLM provider keys**: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`

### 10.3 Config File Sections

The YAML config file (`.paperclip.yaml`) supports these sections:

| Section | Key Settings |
|---------|-------------|
| `server` | port, host, deploymentMode, exposure, publicUrl |
| `database` | url, backupIntervalMinutes, backupRetentionCount, backupDir |
| `storage` | provider (local_disk / s3), bucket, region, endpoint, credentials |
| `secrets` | provider (local_encrypted), masterKeyPath, strictMode |
| `auth` | baseUrlMode (auto / explicit), signupEnabled |
| `heartbeat` | schedulerEnabled, schedulerIntervalMs |
| `ui` | serveUi (boolean), devMiddleware |
| `companyDeletion` | enabled (default true in local_trusted only) |

### 10.4 Adapter Extensibility

**Built-in adapters** (9 types): Shipped in `packages/adapters/`, registered at startup via `registry.ts`.

**External adapter plugins**: Loaded dynamically via:
- `~/.paperclip/adapter-plugins.json` — file-based registration for dev
- Board UI → Adapter Manager — install from npm or local path
- Runtime registration: `registerServerAdapter(type, module)` adds to in-memory registry
- Override pattern: external adapter can shadow a built-in; builtin preserved as fallback
- Pause mechanism: `setOverridePaused(type)` reverts to builtin without removing external

**Adapter configuration per agent**:
- `adapterType` — selects which adapter to use
- `adapterConfig` — JSON blob passed to adapter's `execute()` (model, cwd, env bindings, instructions path, workspace strategy)
- `heartbeatPolicy` — interval in seconds between automatic wakeups
- `timeoutSec` / `graceSec` — execution time limits

### 10.5 Plugin Extensibility

**Plugin capabilities** (declared in manifest):
- `event.subscribe` — listen to domain events
- `agent.tools.register` — provide tools callable during agent runs
- `job.schedule` — register cron-based jobs
- `webhook.receive` — expose webhook endpoints
- `ui.slot` — inject UI components into Paperclip pages
- `launcher.register` — add items to the UI launcher

**Plugin config**: Two levels:
- `plugin_config` — instance-wide operator settings
- `plugin_company_settings` — company-level overrides

**Plugin state**: Scoped key-value store (`plugin_state` table) with scope kinds: instance, company, project, issue.

### 10.6 Skill Injection

Skills are SKILL.md files with frontmatter metadata. Adapters that support skills (claude_local, codex_local, etc.) make them discoverable during agent execution:
- `claude_local`: Creates temporary directory with symlinks to skill files, passes `--skills-dir`
- `codex_local`: Uses global directory for skill discovery
- Skills can be company-scoped (stored in `company_skills` table) or adapter-managed

### 10.7 What Is Hardcoded vs. Configurable

**Hardcoded**:
- Budget threshold percentages: 80% (soft warn), 100% (hard stop) — in `budgets.ts`
- Max concurrent runs per agent: default 1, max 10 — in `heartbeat.ts`
- Max inline wake comments: 8, max 4KB each, 12KB total — in `heartbeat.ts`
- Orphan reap threshold: 5 minutes — in `index.ts`
- Max catch-up runs for routines: 25 — in `routines.ts`
- Agent API key prefix: `pcp_` — in `agents.ts`
- Scheduler minimum interval: 10 seconds — in `config.ts`
- JSON body parser limit: 10MB — in `app.ts`

**Configurable**:
- All paths (home, data, backup, secrets, storage)
- Database provider and connection
- Storage provider (local/S3)
- Auth mode and exposure
- Heartbeat interval
- Adapter type per agent
- Budget amounts per agent/company/project
- Routine cron expressions and variables
- Plugin installation and configuration

---

## 11. Interfaces and External Integrations

### 11.1 REST API Surface

The API is mounted at `/api` with 24+ route groups. Key endpoint families:

| Route Group | Methods | Key Endpoints |
|-------------|---------|---------------|
| Health | GET | `/api/health` — status + version |
| Companies | GET, POST, PATCH, DELETE | CRUD, stats, member management |
| Agents | GET, POST, PATCH, DELETE | CRUD, org chart, instructions, permissions, keys, wakeup |
| Issues | GET, POST, PATCH | CRUD, checkout (`POST /api/issues/:id/checkout`), release, comments, attachments, feedback |
| Projects | GET, POST, PATCH | CRUD, workspaces, runtime control |
| Goals | GET, POST, PATCH, DELETE | CRUD, project associations |
| Approvals | GET, POST, PATCH | Create, resolve, request revision, list by status |
| Routines | GET, POST, PATCH, DELETE | CRUD, triggers, manual trigger, runs |
| Costs | GET | Summary, by-agent, by-provider, by-model, by-project, window analysis |
| Activity | GET | Filtered event log (by agent, entity type, entity ID, time range) |
| Dashboard | GET | Aggregated metrics per company |
| Secrets | GET, POST, PATCH, DELETE | CRUD with versioning and rotation |
| Plugins | GET, POST, PATCH, DELETE | Install, uninstall, enable/disable, health, UI slots, webhooks |
| Adapters | GET, POST, DELETE | List, install external, unregister |
| Access | GET, POST, PATCH, DELETE | Invites, members, permissions, board auth, CLI auth |
| Assets | POST, GET | File upload/download |
| Instance Settings | GET, PATCH | Global + experimental settings |
| LLMs | GET | Model configuration, agent configuration docs |
| Execution Workspaces | GET, POST, PATCH | Workspace runtime control, readiness |
| Sidebar Badges | GET | UI badge counts |
| Company Skills | GET, POST | Skill discovery and sync |

**Authentication methods**:
- **Board**: Better Auth session cookie (authenticated mode) or implicit local user (local_trusted)
- **Agent**: Bearer token (`Authorization: Bearer pcp_...`) — hashed against `agent_api_keys` table
- **Agent JWT**: Run-scoped JWT for local adapters (`supportsLocalAgentJwt` flag)
- **CLI**: Session token via `paperclipai auth` flow, or CLI auth challenge

**Error conventions**: `400` (validation), `401` (unauthenticated), `403` (forbidden), `404` (not found), `409` (conflict — checkout collision), `422` (unprocessable), `500` (server error).

### 11.2 WebSocket Interface

**Endpoint**: `/api/companies/:companyId/events/ws`

- Auth: Bearer token in first message or upgrade header
- Subscribes to company-scoped live events
- Event format: `{ id, companyId, type, createdAt, payload }`
- Used by UI for real-time updates (agent status, run progress, issue changes)

### 11.3 External Service Integrations

| Integration | Purpose | Status |
|-------------|---------|--------|
| Better Auth | User authentication (sessions, accounts) | **Implemented** |
| Embedded PGlite | Zero-config local database | **Implemented** |
| External PostgreSQL | Production database | **Implemented** |
| S3-compatible storage | File storage (AWS S3, MinIO, R2) | **Implemented** |
| Claude Code CLI | Agent execution | **Implemented** (adapter) |
| Codex CLI | Agent execution | **Implemented** (adapter) |
| Cursor CLI | Agent execution | **Implemented** (adapter) |
| Gemini CLI | Agent execution | **Implemented** (adapter) |
| OpenCode CLI | Agent execution | **Implemented** (adapter) |
| Pi CLI | Agent execution | **Implemented** (adapter) |
| OpenClaw gateway | Remote agent gateway | **Implemented** (adapter) |
| GitHub CLI (`gh`) | Used in Dockerfile, CI workflows | **Implemented** |
| Tailscale | Private network access | **Documented**, config support |
| ClipHub | Public company template marketplace | **Not implemented** |
| Telemetry backend | Usage analytics | **Partial** — client setup exists, backend external |

### 11.4 Agent-Facing API Contract (Heartbeat Protocol)

The minimum API surface agents must use during heartbeat execution:

1. `GET /api/agents/me` — identity
2. `GET /api/agents/me/inbox-lite` — assignments (optimized)
3. `POST /api/issues/:id/checkout` — atomic task acquisition
4. `GET /api/issues/:id/heartbeat-context` — task context (optimized)
5. `PATCH /api/issues/:id` — status update + comment (with `X-Paperclip-Run-Id` header)
6. `POST /api/companies/:companyId/issues` — create subtasks (delegation)
7. `POST /api/issues/:id/release` — release task ownership

Optional: `POST /api/companies/:companyId/cost-events` (manual cost reporting), `POST /api/companies/:companyId/agent-hires` (hiring), `POST /api/companies/:companyId/approvals` (CEO strategy).

---

## 12. Data / State / Artifact Flow

### 12.1 Data Model (Entity Relationships)

```
Company (1) ──────────── (*) Agent
   │                        │
   │                        ├── agent_api_keys (*)
   │                        ├── agent_config_revisions (*)
   │                        ├── agent_task_sessions (*)
   │                        ├── agent_wakeup_requests (*)
   │                        └── agent_runtime_state (1)
   │
   ├── (*) Goal ────────── (*) Project
   │                        │
   │                        ├── project_workspaces (*)
   │                        └── project_goals (*)
   │
   ├── (*) Issue
   │       │
   │       ├── issue_comments (*)
   │       ├── issue_attachments (*)
   │       ├── issue_relations (*) ── blocks/blocked_by
   │       ├── issue_labels (*)
   │       ├── issue_documents (*)
   │       ├── issue_work_products (*)
   │       ├── issue_approvals (*)
   │       ├── issue_read_states (*)
   │       └── issue_inbox_archives (*)
   │
   ├── (*) Routine
   │       ├── routine_triggers (*)
   │       └── routine_runs (*)
   │
   ├── (*) Heartbeat Run
   │       └── heartbeat_run_events (*)
   │
   ├── (*) Cost Event
   ├── (*) Finance Event
   ├── (*) Approval
   │       └── approval_comments (*)
   ├── (*) Activity Log Entry
   ├── (*) Company Secret
   │       └── company_secret_versions (*)
   ├── (*) Company Skill
   ├── (*) Label
   ├── (*) Execution Workspace
   │       ├── workspace_operations (*)
   │       └── workspace_runtime_services (*)
   ├── (*) Document
   │       └── document_revisions (*)
   └── (*) Budget Policy
           └── budget_incidents (*)
```

### 12.2 Issue Status Lifecycle (State Machine)

```
backlog ──→ todo ──→ in_progress ──→ in_review ──→ done (terminal)
                        │                │
                        ▼                │
                     blocked ────────────┘
                        │
                        ▼
                    (returns to todo or in_progress when unblocked)

Any non-terminal ──→ cancelled (terminal)
```

**Transition rules**:
- `in_progress` requires atomic checkout (only one agent)
- `blocked` requires comment explaining why
- `done` and `cancelled` are terminal — no further transitions
- `startedAt` auto-set on first transition to `in_progress`
- `completedAt` auto-set on `done`
- `cancelledAt` auto-set on `cancelled`

### 12.3 Agent Status Lifecycle

```
active ──→ idle ──→ running ──→ idle (cycle)
  │                    │
  │                    ▼
  │                  error
  │
  ▼
paused (budget/manual) ──→ active (resume)
  │
  ▼
terminated (terminal, irreversible)

pending_approval (pre-active, awaiting hire approval)
```

### 12.4 Approval Lifecycle

```
pending ──→ approved (terminal)
   │
   ├──→ rejected (terminal)
   │
   └──→ revision_requested ──→ resubmitted ──→ pending (cycle)
```

### 12.5 State Persistence

| State | Storage | Lifetime |
|-------|---------|----------|
| Agent session params | `agent_task_sessions` table | Per (agentId, taskKey), overwritten each run |
| Agent runtime state | `agent_runtime_state` table | Per agent, updated on status change |
| Heartbeat run logs | `heartbeat_runs.logs` (compressed) | Permanent audit |
| Run events | `heartbeat_run_events` table | Per-event stream with sequence ordering |
| Cost events | `cost_events` table | Permanent, aggregated for dashboards |
| Activity log | `activity_log` table | Permanent audit trail |
| Secrets | `company_secret_versions` table | Versioned, encrypted at rest |
| Plugin state | `plugin_state` table | Scoped KV store, persistent |
| Feedback votes | `feedback_votes` table | Local storage, optional remote sync |
| Wakeup requests | `agent_wakeup_requests` table | Transient (consumed on processing) |

### 12.6 Artifact Flow

1. **Agent execution** → adapter captures stdout → stored in `heartbeat_runs.logs` (compressed)
2. **Cost data** → extracted from adapter result → `cost_events` table → aggregated for dashboard
3. **Session state** → serialized by adapter codec → `agent_task_sessions` → deserialized on next run
4. **File uploads** → `assets` table (metadata) + storage provider (local disk or S3)
5. **Issue attachments** → linked via `issue_attachments` to assets
6. **Company export** → reads entities from DB → writes markdown files + `.paperclip.yaml`
7. **Company import** → parses markdown package → creates entities in DB

---

## 13. Deployment and Operational Model

### 13.1 Deployment Modes

| Mode | Auth | Network | Use Case |
|------|------|---------|----------|
| `local_trusted` | None (implicit local user) | Loopback only (127.0.0.1) | Solo dev, local experimentation |
| `authenticated` + `private` | Better Auth sessions | Private network (Tailscale/VPN/LAN) | Team use on private network |
| `authenticated` + `public` | Better Auth sessions | Internet-facing | Production, explicit public URL required |

Mode is set via `PAPERCLIP_DEPLOYMENT_MODE` env var or `pnpm paperclipai configure --section server`.

### 13.2 Distribution Channels

| Channel | Form | Command |
|---------|------|---------|
| **npm** | `paperclipai` package | `npx paperclipai onboard --yes` |
| **Docker** | `ghcr.io/paperclipai/paperclip` | `docker compose up --build` |
| **Source** | Git clone + `pnpm install` | `pnpm dev` |
| **Podman Quadlet** | systemd unit files | `systemctl start paperclip` |

### 13.3 Docker Architecture

**Dockerfile** — Multi-stage build:

| Stage | Purpose |
|-------|---------|
| `base` | Node.js LTS, system deps (git, curl, gh CLI), pnpm enabled |
| `deps` | Install pnpm workspace dependencies |
| `build` | Build UI, plugins, server |
| `production` | Runtime image with globally installed agent CLIs (claude-code, codex, opencode-ai) |

Volumes: `/paperclip` for persistent data (DB, secrets, workspaces, storage).

**Compose files**:
- `docker-compose.quickstart.yml` — Embedded DB, single container
- `docker-compose.yml` — Full stack with external PostgreSQL
- `docker-compose.untrusted-review.yml` — Isolated PR review environment

### 13.4 Data Directories

| Path | Content |
|------|---------|
| `~/.paperclip/` | Root data directory |
| `~/.paperclip/instances/default/db/` | Embedded PGlite data |
| `~/.paperclip/instances/default/secrets/master.key` | Encryption master key |
| `~/.paperclip/instances/default/data/storage/` | Local file storage |
| `~/.paperclip/instances/default/workspaces/<agentId>/` | Agent working directories |
| `~/.paperclip/backups/` | Database backups |
| `~/.paperclip/.paperclip.yaml` | Config file |
| `~/.paperclip/adapter-plugins.json` | External adapter registrations |
| `~/.paperclip/context.json` | CLI connection profiles |

### 13.5 Operational Commands

| Command | Purpose |
|---------|---------|
| `pnpm paperclipai onboard` | First-run interactive setup wizard |
| `pnpm paperclipai run` | Auto-onboard + health check + start server |
| `pnpm paperclipai doctor --repair` | Diagnostic checks with auto-fix |
| `pnpm paperclipai configure --section <name>` | Edit config sections interactively |
| `pnpm paperclipai db:backup` | Manual database backup |
| `curl localhost:3100/api/health` | Health probe |

### 13.6 Database Backup

- Configurable interval (`backupIntervalMinutes`) and retention count (`backupRetentionCount`)
- Automated via `setInterval` in server startup
- Script: `scripts/backup-db.sh`
- Backup directory: `~/.paperclip/backups/` (or `PAPERCLIP_BACKUP_DIR`)

### 13.7 Worktree Support

For developers working on multiple branches simultaneously:
- `pnpm paperclipai worktree init` creates isolated instance (independent port, DB, config)
- Seeding modes: minimal, full, none
- Instance detection via local service registry to prevent port conflicts

---

## 14. CI/CD, Automation, and Repository Governance

### 14.1 GitHub Actions Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `pr.yml` | Pull request | Policy checks, typecheck, tests, build, e2e, canary dry-run |
| `release.yml` | Push to master / manual dispatch | Canary auto-publish on push; stable publish on manual trigger |
| `e2e.yml` | Manual / scheduled | Playwright browser tests |
| `docker.yml` | Push to master / tags | Multi-platform Docker image build + push to ghcr.io |
| `release-smoke.yml` | Post-release | Playwright smoke tests against published release |
| `refresh-lockfile.yml` | Scheduled | Automated dependency updates |

### 14.2 PR Policy Enforcement

The `pr.yml` workflow enforces:
- Lockfile integrity (pnpm-lock.yaml not manually edited — CI owns it)
- Dockerfile dependency stage validation
- All package.json files present in Docker deps stage
- TypeScript typecheck passes across all packages
- Vitest test suite passes
- Full monorepo build succeeds
- Canary dry-run succeeds
- E2E Playwright tests pass

### 14.3 Release Model

**Versioning**: Calendar-based `YYYY.MDD.P` where MDD = month + zero-padded day. Examples: `2026.318.0` (March 18, 2026, first stable), `2026.318.0-canary.3` (fourth canary).

**Canary releases**: Automatic on every push to `master`. Published to npm under `canary` dist-tag. Git tagged as `canary/vYYYY.MDD.P-canary.N`.

**Stable releases**: Manual promotion via `workflow_dispatch`. Publishes under `latest` dist-tag. Creates GitHub Release with notes from `releases/vYYYY.MDD.P.md`.

**npm publishing**: OIDC trusted publishing via GitHub Actions (no long-lived npm token). CLI bundled via esbuild into single executable.

**Rollback**: Repoint `latest` dist-tag to prior stable version. Script: `scripts/rollback-latest.sh`.

### 14.4 PR Template Requirements

Every PR must include (from `.github/PULL_REQUEST_TEMPLATE.md`):
1. **Thinking Path** — 5–8 step trace from project context to the change
2. **What Changed** — concrete bullet list
3. **Verification** — how to confirm it works
4. **Risks** — what could break
5. **Model Used** — exact AI model (provider, ID, context window) or "None — human-authored"
6. **Checklist** — all items checked

### 14.5 Automated Code Quality

- **Greptile** integration: Required 5/5 score with all comments addressed
- **Forbidden token check**: `scripts/check-forbidden-tokens.mjs` scans build output for leaked workspace paths
- **TypeScript strict mode**: Across all packages
- **Zod validation**: Request bodies validated at API boundaries

---

## 15. Security / Access / Trust Boundaries

### 15.1 Authentication

| Actor | Mechanism | Storage |
|-------|-----------|---------|
| Board operator | Better Auth session (cookie) | `auth_sessions` table |
| Board operator | Board API key | `board_api_keys` table (hashed) |
| Agent | Bearer API key (`pcp_` prefix) | `agent_api_keys` table (SHA256 hashed) |
| Agent (local) | Run-scoped JWT | Signed by server, verified per-request |
| CLI | Session token or CLI auth challenge | `cli_auth_challenges` table |

### 15.2 Authorization Model

**Company-scoped RBAC**:
- `company_memberships` table: (companyId, principalType, principalId, role, status)
- `principal_permission_grants` table: specific permission grants per membership
- Principal types: `user`, `agent`
- Instance admin: bypasses all company-level checks via `instance_user_roles`

**Agent permissions**: Scoped to own company. Cannot:
- Access other companies
- Modify other agents' config
- Create agents directly (must use hire approval flow)
- Bypass budget enforcement

**Board powers**: Full CRUD on all entities within their companies. Can pause/resume/terminate any agent, override budgets, reassign tasks, approve/reject all requests.

### 15.3 Secret Management

- Provider: `local_encrypted` (default)
- Master key: auto-generated at `~/.paperclip/instances/default/secrets/master.key`
- Never leaves the machine (or container volume)
- Secrets stored in `company_secrets` / `company_secret_versions` tables
- Versioned with rotation support
- Strict mode: blocks inline storage of sensitive env values (matching `*_API_KEY`, `*_TOKEN`, `*_SECRET`)
- Migration tool: `pnpm secrets:migrate-inline-env --apply`

### 15.4 Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│ TRUSTED: Server Process                                 │
│  ├── All services, routes, middleware                   │
│  ├── Plugin host (bridges to workers)                   │
│  └── Adapter registry                                   │
├─────────────────────────────────────────────────────────┤
│ SEMI-TRUSTED: Plugin Workers                            │
│  ├── Run in separate processes                          │
│  ├── Communicate via JSON-RPC 2.0                       │
│  ├── Access scoped state and domain data                │
│  └── Cannot bypass company isolation                    │
├─────────────────────────────────────────────────────────┤
│ SEMI-TRUSTED: Agent Processes                           │
│  ├── Run externally (CLI, HTTP)                         │
│  ├── Authenticate via API key or run JWT                │
│  ├── Scoped to company via key association              │
│  └── Cannot access other companies                      │
├─────────────────────────────────────────────────────────┤
│ UNTRUSTED: External Network                             │
│  ├── Hostname guard (authenticated private mode)        │
│  ├── CSRF protection for board mutations                │
│  └── Auth required for all non-health endpoints         │
└─────────────────────────────────────────────────────────┘
```

### 15.5 Security Measures

- API key hashing: SHA256 at rest (never stored in plaintext)
- CSRF protection: Board mutation guard for POST/PUT/DELETE
- Hostname allowlisting: Private mode restricts to configured hostnames
- Input validation: Zod schemas at API boundaries
- Secret redaction: Sensitive values masked in logs and API responses
- Path redaction: Home directory paths scrubbed from adapter logs
- Company isolation: All queries filtered by `companyId` FK
- Plugin capability approval: Upgraded plugins with new capabilities require operator confirmation
- Forbidden token scanning: Build output checked for leaked paths

---

## 16. Testing / Validation / Quality Controls

### 16.1 Test Framework

| Layer | Framework | Location | Count |
|-------|-----------|----------|-------|
| Unit/Integration | Vitest | `server/__tests__/` | 123 test files |
| Unit | Vitest + jsdom | `ui/src/` | Scattered `.test.tsx` files |
| Unit | Vitest | `cli/src/__tests__/` | CLI-specific tests |
| Unit | Vitest | `packages/adapters/*/` | Adapter-specific tests |
| E2E | Playwright (Chromium) | `tests/e2e/` | `onboarding.spec.ts` + config |
| Release smoke | Playwright | `tests/release-smoke/` | Post-publish validation |
| Agent evals | Promptfoo | `evals/promptfoo/` | 8 core behavior cases |

### 16.2 Vitest Configuration

Root `vitest.config.ts` defines workspace projects:
- `packages/db`
- `packages/adapters/codex-local`
- `packages/adapters/opencode-local`
- `server`
- `ui`
- `cli`

### 16.3 Test Patterns

**Server tests** (123 files): Use `vitest` + `supertest` for HTTP testing. Mock database layer. Test coverage includes:
- Route endpoint behavior (status codes, response shapes)
- Service logic (state transitions, budget calculations, checkout conflicts)
- Adapter parsing (stdout stream parsing, token extraction)
- Approval workflows
- Plugin lifecycle
- Routine scheduling

**UI tests**: Use `@vitest-environment jsdom` for browser simulation. Mock router, API client.

**E2E tests**: Playwright config starts server via `pnpm paperclipai run` with 120s timeout. Tests onboarding flow. LLM assertions skipped by default (`SKIP_LLM_ASSERTIONS=true`).

### 16.4 Eval Framework

**Status**: Phase 0 (bootstrap). Promptfoo framework with 8 deterministic behavior test cases:

| Case | Tests |
|------|-------|
| Assignment pickup | Agent correctly fetches and checks out assigned tasks |
| Progress update | Agent updates issue status and comments properly |
| Blocked reporting | Agent sets blocked status with explanation |
| No-work exit | Agent exits gracefully when no tasks available |
| Checkout-before-work | Agent always calls checkout before starting work |
| 409 conflict handling | Agent handles checkout conflict correctly (no retry) |
| Approval required | Agent requests approval for governed actions |
| Company boundary | Agent cannot access cross-company data |

**Planned phases**: TypeScript harness (P1), pairwise scoring (P2), efficiency metrics (P3), production case ingestion (P4).

### 16.5 Verification Commands

```sh
pnpm -r typecheck       # TypeScript strict check across all packages
pnpm test:run           # Vitest across all configured projects
pnpm build              # Full monorepo build
pnpm test:e2e           # Playwright end-to-end tests
pnpm evals:smoke        # Promptfoo agent behavior evals
```

---

## 17. Portability and Reimplementation Guidance

This section structures Paperclip's architecture for reimplementation in another language/platform.

### 17.1 Language-Agnostic Domain Model

**Core entities** (implement as database tables):

| Entity | Key Fields | Invariants |
|--------|-----------|------------|
| Company | id, name, description, status, budgetMonthlyCents, issuePrefixCounter | Multi-tenant root, company-scoped isolation |
| Agent | id, companyId, name, role, adapterType, adapterConfig, reportsTo, status, budgetMonthlyCents, heartbeatPolicy | Strict tree hierarchy (single parent), single adapter |
| Issue | id, companyId, title, description, status, priority, assigneeAgentId, parentId, projectId, goalId | Single assignee, atomic checkout, parent/child hierarchy |
| Goal | id, companyId, title, description, parentId | Hierarchical goal tree |
| Project | id, companyId, name, status | Groups related work |
| Approval | id, companyId, type, status, requestorAgentId, payload | Polymorphic payload (hire, strategy) |
| Routine | id, companyId, name, bodyTemplate, assigneeAgentId | Template + triggers |
| RoutineTrigger | id, routineId, kind (schedule/webhook/api), cronExpression, timezone | Cron parsing required |
| HeartbeatRun | id, agentId, companyId, status, exitCode, usage, cost, sessionParams, logs | Full execution record |
| CostEvent | id, companyId, agentId, runId, provider, model, inputTokens, outputTokens, costCents | Multi-dimensional aggregation |
| BudgetPolicy | id, scope (company/agent/project), scopeId, limitCents, windowType | Two thresholds: 80% warn, 100% hard stop |

### 17.2 Critical Interfaces (Language-Agnostic)

**Adapter Interface**:
```
execute(context: ExecutionContext) → ExecutionResult
testEnvironment() → TestResult
sessionCodec.serialize(state) → bytes
sessionCodec.deserialize(bytes) → state
```

**Storage Interface**:
```
putObject(key, stream, metadata) → void
getObject(key) → { stream, metadata }
headObject(key) → metadata
deleteObject(key) → void
```

**Secret Provider Interface**:
```
encrypt(plaintext, masterKey) → ciphertext
decrypt(ciphertext, masterKey) → plaintext
```

### 17.3 Critical Algorithms

**Atomic checkout** (SQL-level):
```sql
UPDATE issues SET assignee_agent_id = :agentId, status = 'in_progress'
WHERE id = :issueId AND (assignee_agent_id IS NULL OR assignee_agent_id = :agentId)
  AND status NOT IN ('done', 'cancelled')
RETURNING *;
-- If 0 rows affected → 409 Conflict
```

**Budget evaluation** (per cost event):
```
observed = SUM(cost_events.costCents) WHERE scope AND window
if observed >= limit: hard_stop → pause scope, cancel work
elif observed >= limit * 0.8: soft_warn → log alert
```

**Heartbeat tick** (per interval):
```
for each active agent:
  if heartbeat disabled: skip
  elapsed = now - agent.lastHeartbeatAt
  if elapsed >= agent.heartbeatPolicy.intervalSeconds:
    enqueue wakeup(agentId, source=timer)
```

**Cron matching** (per minute):
```
for each field in [minute, hour, day, month, weekday]:
  if field == '*': match
  elif field is list: check if current value in list
  elif field is range: check if current value in range
  elif field is step: check if current value matches step
```

### 17.4 State Transitions to Implement

See sections 12.2 (Issue), 12.3 (Agent), 12.4 (Approval) for complete state machines.

Additionally:
- **Plugin lifecycle**: installed → ready → disabled → error → upgrade_pending → uninstalled
- **Routine run**: queued → running → succeeded/failed/cancelled
- **Budget incident**: open → resolved

### 17.5 External Dependencies to Replace

| TypeScript Dependency | Purpose | Replacement Strategy |
|----------------------|---------|---------------------|
| Express.js 5 | HTTP framework | Any HTTP framework (Flask, Gin, Actix, etc.) |
| Drizzle ORM | Database ORM | Any PostgreSQL ORM |
| Better Auth | Authentication | Session + API key auth library |
| Zod | Validation | JSON Schema or language-native validation |
| ws | WebSocket | Any WebSocket library |
| Pino | Logging | Any structured logger |
| embedded-postgres (PGlite) | Zero-config DB | SQLite equivalent or embedded Postgres binding |
| commander.js | CLI framework | Any CLI framework (Click, Cobra, Clap) |
| esbuild | Build tool | Language-native build |
| React + Vite | Frontend | Any SPA framework or server-rendered UI |
| TanStack Query | Server state | Any data fetching library |
| sharp | Image processing | Any image library |

### 17.6 Migration Risks

1. **Session codec compatibility**: Adapters serialize session state in adapter-specific formats. New implementation must match format if preserving sessions.
2. **PostgreSQL-specific SQL**: Some queries use PostgreSQL features (JSONB, lateral joins). Switching DB requires query adaptation.
3. **Plugin worker isolation**: Current model uses Node.js child processes with JSON-RPC. Other languages need equivalent IPC mechanism.
4. **Embedded PGlite**: TypeScript-specific. Replace with language-appropriate embedded DB.
5. **Adapter CLI spawning**: Process.spawn semantics differ across languages. Must handle stdout streaming, signal forwarding, timeout management.
6. **Calendar versioning**: `YYYY.MDD.P` format is custom. Reimplementation can use standard semver.

---

## 18. Known Gaps / Risks / Technical Debt

### 18.1 Documentation Gaps

| Area | Status | Detail |
|------|--------|--------|
| API reference docs | **Stub only** | All 12 files in `docs/api/` contain empty frontmatter |
| Adapter docs | **Stub only** | All 9 files in `docs/adapters/` contain empty frontmatter |
| CLI reference docs | **Stub only** | All 3 files in `docs/cli/` contain empty frontmatter |
| Issue documents plan | **Stub** | `docs/plans/2026-03-13-issue-documents-plan.md` has only title |
| Agent config UI spec | **Stub** | `docs/specs/agent-config-ui.md` has only title |

### 18.2 Implementation Gaps

| Feature | Status | Evidence |
|---------|--------|---------|
| Teams entity | **Not implemented** | `doc/TASKS.md` describes teams with workflow states, identifiers, team-scoped issues. No `teams` table in schema. |
| Milestones | **Not implemented** | `doc/TASKS.md` describes milestones as project subdivisions. Not in schema. |
| Initiatives | **Not implemented** | `doc/TASKS.md` describes highest-level planning entity. Not in schema. |
| Issue identifiers (`ENG-123`) | **Not verified** | `doc/TASKS.md` describes team-key numbering. May be partially implemented via `companies.issuePrefixCounter`. |
| Workflow states per team | **Not implemented** | `doc/TASKS.md` describes team-specific workflow states. Current implementation uses global status enum. |
| Point estimates | **Not implemented** | `doc/TASKS.md` describes exponential scale (1,2,4,8,16). Not in schema. |
| Threaded comments (reply chains) | **Not implemented** | `issue_comments` table exists but has no `parent_id` column. Flat comments only. |
| Label groups (mutual exclusion) | **Not implemented** | `labels` table has only id, companyId, name, color. No group/parent columns. |
| ClipHub marketplace | **Not implemented** | Detailed design in `doc/CLIPHUB.md`. No code, no separate service. |
| MCP function contracts | **Not implemented** | `doc/TASKS-mcp.md` defines full MCP API surface. No MCP server code found. |
| Multi-member boards | **Not implemented** | V1 spec explicitly limits to single human board. |
| Knowledge bases | **Not implemented** | Listed as out-of-scope in `doc/PRODUCT.md`. |
| Automatic self-healing | **Not implemented** | Listed as out-of-scope in `doc/PRODUCT.md`. |

### 18.3 Technical Risks

1. **Single-process scheduler**: The heartbeat scheduler runs in a `setInterval` on the main Node.js process. No distributed locking — if multiple server instances run, heartbeats could fire redundantly. **Risk**: Not horizontally scalable without external coordination.

2. **In-memory event system**: `live-events.ts` uses `EventEmitter` with no persistence. Events lost on server restart. Connected WebSocket clients reconnect but miss events during downtime. **Risk**: UI stale state after restart.

3. **In-memory agent start locks**: `withAgentStartLock()` uses a JavaScript `Map`. Lost on server restart. **Risk**: Brief window of concurrent agent runs on restart.

4. **Plugin worker process model**: Workers are Node.js child processes on the same machine. No support for distributed plugin execution. **Risk**: Plugin failures can consume server resources.

5. **Embedded PGlite limitations**: Single-writer, not designed for high concurrency. Suitable for dev/single-user but not production multi-user. **Risk**: Data corruption under heavy concurrent access.

6. **No rate limiting**: No visible rate limiting middleware for API endpoints. **Risk**: API abuse in public deployment mode.

7. **Session state size**: Agent session params stored in database without size limits beyond compaction policy. **Risk**: Unbounded growth for long-running sessions.

### 18.4 Technical Debt

1. **Fork-specific patches**: The HenkDz fork includes QoL patches (stderr_group, tool_group, dashboard excerpt) that must be manually re-applied when merging upstream changes.
2. **Plugin worker manager in JavaScript**: `plugin-worker-manager.js` is plain JS, not TypeScript — inconsistent with rest of codebase.
3. **Adapter type validation**: `packages/shared/src/adapter-type.ts` accepts any string (min 1 char). Validation of known types happens at route level rather than schema level.
4. **Dev runner complexity**: `scripts/dev-runner.ts` is a substantial process manager (file watching, migration preflight, child process lifecycle) that could benefit from being a proper tool.
5. **Empty documentation pages**: 24+ stub files in `docs/` that are published to the documentation site with no content.

---

## 19. Future Plans Found in the Repository

### 19.1 Explicit Roadmap Items

| Plan | Source | Status |
|------|--------|--------|
| **ClipHub marketplace** | `doc/CLIPHUB.md`, `docs/specs/cliphub-plan.md` | Detailed design. V1 scope defined. No implementation. Separate service (React + Hono + PostgreSQL). |
| **Teams + workflow states** | `doc/TASKS.md` | Full data model designed. 28 fields per issue. Team-specific workflow states. Not yet in schema. |
| **MCP function contracts** | `doc/TASKS-mcp.md` | Complete MCP API surface defined (issues, workflow states, teams, projects, milestones, labels, relations, comments, initiatives). Not implemented. |
| **Issue documents** | `docs/plans/2026-03-13-issue-documents-plan.md` | Stub file only — plan not written yet. |
| **Agent config UI spec** | `docs/specs/agent-config-ui.md` | Stub file only. |
| **Advanced eval phases** | `evals/README.md` | Phase 1: TypeScript harness. Phase 2: Pairwise scoring. Phase 3: Efficiency metrics. Phase 4: Production ingestion. |
| **Plugins, advanced workflows** | `doc/SPEC-implementation.md` (post-V1 backlog) | Listed as post-V1: plugins, advanced workflow customization, dependencies, realtime transport, ClipHub. |
| **Multi-member boards** | `doc/SPEC.md` | Explicitly out-of-scope for V1 ("single human board"). |
| **External revenue/expense tracking** | `doc/SPEC.md` | Explicitly out-of-scope. |
| **Agent Companies spec** | `docs/companies/companies-spec.md` | v1-draft specification. Markdown-first package format. Import/export implemented. |

### 19.2 Post-V1 Backlog (from `doc/SPEC-implementation.md`)

- Advanced workflow customization (custom status flows beyond the fixed lifecycle)
- Issue dependency management (beyond current blocking relations)
- Real-time transport upgrade (beyond current WebSocket + EventEmitter)
- ClipHub integration (browse + install company templates from marketplace)
- Plugin marketplace
- Delegated hiring authority (agents hiring without board approval in some contexts)

### 19.3 Inferred Future Directions

Based on code patterns and stubs:
- **Expanded adapter ecosystem**: The external adapter plugin system is built to support unlimited adapter types
- **Richer routine engine**: Variable interpolation, webhook triggers, and concurrency policies suggest plans for workflow automation
- **Agent skill marketplace**: Company skills + skill sync across agents hint at shareable skill packages
- **Finance tracking**: `finance_events` table exists but finance service is minimal — suggests planned billing/invoicing features
- **Execution workspace providers**: Schema supports `local_fs` and `git_repo` workspace types — suggests plans for cloud/container workspaces

---

## 20. Open Questions / Uncertainties

1. **SSE vs. WebSocket**: The UI context provider is named `LiveUpdatesProvider` and the `docs/agents-runtime.md` mentions SSE, but only WebSocket transport was found in server code. May have been migrated or may use both. **Uncertainty: medium.**

2. **Issue identifiers format**: `doc/TASKS.md` describes `TEAM_KEY-NUMBER` identifiers (e.g. `ENG-123`). The `companies.issuePrefixCounter` column exists, but the team-key prefix mechanism requires a `teams` table that doesn't exist. Current implementation likely uses a simpler scheme. **Uncertainty: medium.**

3. **Threaded comments**: `doc/TASKS.md` describes `parent_id` on comments for threading. The `issue_comments` table exists, but column-level schema was not fully verified for parent_id support. **Uncertainty: low.**

4. **Label group nesting**: `doc/TASKS.md` describes one-level nesting with mutual exclusion within groups. `labels` table exists but group semantics not verified at column level. **Uncertainty: low.**

5. **OpenClaw adapter completeness**: `openclaw_gateway` adapter exists in source but OpenClaw integration complexity (Docker sandbox, device pairing, auth headers) suggests it may not be fully production-ready. **Uncertainty: medium.**

6. **Telemetry backend**: `telemetry.ts` sets up an analytics client, and `feedback-voting.md` references remote sync. The backend service is external and not included in the repository. Unclear whether it exists as a production service. **Uncertainty: medium.**

7. **Plugin UI sandboxing**: `docs` and skill files note "plugin UI is not sandboxed." This means plugin-provided UI components run with full access to the React app context. **Confirmed but notable security consideration.**

8. **Adapter type expansion**: The adapter registry accepts arbitrary string types, and external adapters can register at runtime. The set of adapters may grow beyond the 9 currently in `AGENT_ADAPTER_TYPES`. **Confirmed behavior.**

9. **Finance events**: `finance_events` table exists in schema and `finance.ts` service exists. Unclear what billing model is implemented or planned beyond cost tracking. **Uncertainty: medium.**

---

## 21. Appendix: Important Files and Their Roles

### Core Architecture

| File | Role |
|------|------|
| `server/src/index.ts` | Server startup orchestration, scheduler loop, database init |
| `server/src/app.ts` | Express app factory, middleware stack, route mounting, plugin setup |
| `server/src/config.ts` | Hierarchical config loading (env → file → defaults) |
| `server/src/services/heartbeat.ts` | Core execution engine — wakeup, session, adapter invocation, cost |
| `server/src/services/routines.ts` | Cron engine, variable interpolation, scheduled trigger evaluation |
| `server/src/adapters/registry.ts` | Adapter registration, override/pause, built-in + external loading |
| `server/src/services/issues.ts` | Issue lifecycle, atomic checkout, status transitions |
| `server/src/services/budgets.ts` | Budget evaluation, auto-pause, incident management |
| `server/src/services/costs.ts` | Cost event recording, multi-dimensional aggregation |
| `server/src/services/agents.ts` | Agent CRUD, API key management, config revision tracking |
| `server/src/services/access.ts` | RBAC, company membership, permission grants |
| `server/src/services/activity-log.ts` | Mutation audit logging |
| `server/src/services/live-events.ts` | In-memory EventEmitter event broadcast |
| `server/src/realtime/live-events-ws.ts` | WebSocket server for real-time UI updates |

### Data Layer

| File | Role |
|------|------|
| `packages/db/src/schema/index.ts` | Central export of all 75 database tables |
| `packages/db/drizzle.config.ts` | Migration generation config |
| `packages/shared/src/constants.ts` | All status enums, adapter types, agent roles |
| `packages/shared/src/types/` | TypeScript interfaces for all domain entities |
| `packages/shared/src/validators/` | Zod request validation schemas |
| `packages/shared/src/api.ts` | API route path constants |

### Adapter Layer

| File | Role |
|------|------|
| `packages/adapter-utils/src/types.ts` | Adapter contract interfaces |
| `packages/adapters/claude-local/src/server/execute.ts` | Claude CLI execution pipeline |
| `packages/adapters/codex-local/src/server/execute.ts` | Codex CLI execution pipeline |
| `server/src/adapters/plugin-loader.ts` | Dynamic external adapter loading |

### Plugin Layer

| File | Role |
|------|------|
| `packages/plugins/sdk/src/define-plugin.ts` | Plugin definition API |
| `packages/plugins/sdk/src/types.ts` | Plugin context, event, scope types |
| `packages/plugins/sdk/src/protocol.ts` | JSON-RPC 2.0 host-worker protocol |
| `server/src/services/plugin-lifecycle.ts` | Plugin state machine |
| `server/src/services/plugin-worker-manager.js` | Worker process pool |
| `server/src/services/plugin-job-scheduler.ts` | Cron-based plugin jobs |

### UI Layer

| File | Role |
|------|------|
| `ui/src/main.tsx` | React app entry point, provider tree |
| `ui/src/api/client.ts` | Base HTTP client for API calls |
| `ui/src/context/CompanyContext.tsx` | Company scope selection |
| `ui/src/context/LiveUpdatesProvider.tsx` | Real-time event subscription |

### CLI Layer

| File | Role |
|------|------|
| `cli/src/index.ts` | CLI entry point, command registration |
| `cli/src/commands/onboard.ts` | First-run setup wizard |
| `cli/src/commands/run.ts` | Bootstrap + server start |
| `cli/src/commands/doctor.ts` | Diagnostic checks with auto-repair |

### DevOps

| File | Role |
|------|------|
| `Dockerfile` | Multi-stage production build |
| `docker/docker-compose.quickstart.yml` | Quick-start Docker deployment |
| `.github/workflows/release.yml` | Canary + stable release pipeline |
| `.github/workflows/pr.yml` | PR checks (policy, typecheck, test, build, e2e) |
| `scripts/dev-runner.ts` | Dev server lifecycle with hot reload |
| `scripts/release.sh` | Release script (canary/stable) |
| `scripts/build-npm.sh` | CLI esbuild packaging |

### Skills and Documentation

| File | Role |
|------|------|
| `skills/paperclip/SKILL.md` | Agent heartbeat protocol reference |
| `skills/paperclip-create-agent/SKILL.md` | Agent hiring workflow |
| `.claude/skills/design-guide/SKILL.md` | UI design system reference |
| `doc/SPEC-implementation.md` | V1 implementation contract |
| `doc/PRODUCT.md` | Product definition |
| `doc/GOAL.md` | Vision and success metrics |
| `docs/companies/companies-spec.md` | Agent Companies v1 package format |

---

## Repository Coverage Summary

| Area | Files/Dirs Examined | Coverage |
|------|-------------------|----------|
| Documentation (`doc/`, `docs/`) | All files read | Complete |
| Server source (`server/src/`) | All key services, routes, middleware, adapters | Complete |
| Database schema (`packages/db/src/schema/`) | All 61 schema files enumerated, index.ts read | Complete |
| Shared types (`packages/shared/src/`) | Constants, types, validators, API paths | Complete |
| Adapter implementations (`packages/adapters/`) | All 8 adapters surveyed; claude-local and codex-local read in detail | Complete |
| Adapter utils (`packages/adapter-utils/`) | Interface types fully read | Complete |
| Plugin SDK (`packages/plugins/sdk/`) | Types, define-plugin, protocol | Complete |
| Plugin examples | Listed, not deep-read | Partial |
| UI source (`ui/src/`) | Structure mapped; entry point, API, context, pages surveyed | Complete |
| CLI source (`cli/src/`) | Entry point, command structure, package.json | Complete |
| CI/CD (`.github/workflows/`) | All 6 workflows read | Complete |
| Scripts (`scripts/`) | dev-runner.ts, dev-service.ts read; others surveyed | Partial |
| Tests (`server/__tests__/`, `tests/`) | Count verified (123 files), patterns sampled | Partial |
| Evals (`evals/`) | README read, cases surveyed | Complete |
| Skills (`skills/`, `.claude/skills/`) | All SKILL.md files read | Complete |
| Root configs | package.json, Dockerfile, pnpm-workspace.yaml, vitest.config.ts, AGENTS.md, CONTRIBUTING.md | Complete |
| Release artifacts (`releases/`) | Listed | Surveyed |

---

## Key Evidence Sources

The most important files grounding this PRD:

1. `server/src/index.ts` — startup sequence, scheduler loop location
2. `server/src/services/heartbeat.ts` — core execution engine
3. `server/src/services/routines.ts` — routine scheduling engine
4. `server/src/services/budgets.ts` — budget enforcement logic
5. `server/src/services/issues.ts` — atomic checkout implementation
6. `server/src/adapters/registry.ts` — adapter registration model
7. `server/src/app.ts` — full middleware and routing stack
8. `packages/db/src/schema/index.ts` — all 75 tables
9. `packages/shared/src/constants.ts` — enum definitions
10. `packages/adapter-utils/src/types.ts` — adapter contract
11. `packages/plugins/sdk/src/types.ts` — plugin contract
12. `doc/SPEC-implementation.md` — V1 implementation contract
13. `doc/TASKS.md` — target task management data model
14. `doc/TASKS-mcp.md` — MCP function contracts
15. `doc/CLIPHUB.md` — marketplace design
16. `skills/paperclip/SKILL.md` — heartbeat protocol reference
17. `.github/workflows/release.yml` — release pipeline
18. `Dockerfile` — production build stages

---

## Doc-Code Mismatches

| # | Documentation Claim | Implementation Reality | Severity |
|---|--------------------|-----------------------|----------|
| 1 | `doc/TASKS.md` describes teams as primary org unit with team-specific workflow states and `TEAM_KEY-NUMBER` identifiers | No `teams` table in schema. Issues use global status enum, not team-specific workflow states. | **High** — represents planned but unimplemented feature set |
| 2 | `doc/TASKS.md` describes milestones, initiatives, point estimates | None of these entities exist in the database schema | **High** — planned, not implemented |
| 3 | `doc/TASKS-mcp.md` defines full MCP function contracts | No MCP server implementation found in codebase | **High** — contract defined, implementation absent |
| 4 | `docs/agents-runtime.md` mentions SSE for live updates | Only WebSocket transport found in server code | **Low** — may be outdated doc reference |
| 5 | `doc/SPEC-implementation.md` lists `process` and `http` as the two adapter types | Implementation has 9+ adapter types including all local CLI adapters | **Low** — spec was written before adapter expansion |
| 6 | Several docs reference `AGENTS.md` in skill/agent context | Also name of root repo governance file; potential confusion | **Low** — naming overlap |
| 7 | `docs/guides/board-operator/managing-agents.md` lists adapter types | 9 types in `AGENT_ADAPTER_TYPES`; external adapters can register additional types at runtime | **Low** — by design |
| 8 | `doc/CLIPHUB.md` describes a full separate service architecture | No ClipHub code exists anywhere | **Medium** — design only, clearly future |
| 9 | `adapter-plugin.md` describes fork-specific behavior (port 3101+, NTFS workarounds) | This is fork documentation mixed into the main repo | **Low** — fork-specific context |
| 10 | `doc/TASKS.md` describes threaded comments with `parent_id` | `issue_comments` schema has no `parent_id` column — flat comments only | **Medium** — planned feature not implemented |
| 11 | `doc/TASKS.md` describes label groups with mutual exclusion | `labels` schema has only basic fields (name, color) — no group support | **Medium** — planned feature not implemented |

---

## Improvement Opportunities

### High Priority

1. **Fill API reference documentation** — 12 stub files in `docs/api/` should be generated from route code or OpenAPI spec
2. **Fill adapter documentation** — 9 stub files in `docs/adapters/` should document each adapter's config, environment requirements, and troubleshooting
3. **Add rate limiting** — No rate limiting middleware visible; critical for `authenticated` + `public` deployment mode
4. **Distributed scheduler support** — Single-process `setInterval` prevents horizontal scaling; consider advisory locks or leader election

### Medium Priority

5. **Implement teams entity** — The `doc/TASKS.md` data model is well-designed and would significantly improve task organization
6. **Implement MCP server** — `doc/TASKS-mcp.md` already defines the full contract; implementation would enable MCP-compatible agent tooling
7. **Add OpenAPI/Swagger spec** — No API specification file exists; would enable auto-generated docs and client SDKs
8. **Convert plugin-worker-manager to TypeScript** — Only `.js` file in the server codebase
9. **Add health check for plugin workers** — Worker crashes detected reactively; proactive health probes would improve reliability
10. **Event persistence** — In-memory EventEmitter loses events on restart; consider write-ahead log or Redis pub/sub

### Low Priority

11. **Remove or complete stub documentation files** — 24+ empty pages create false navigation in the docs site
12. **Consolidate `doc/` and `docs/`** — Two documentation directories with overlapping content
13. **Standardize adapter type validation** — Move from runtime string validation to schema-level enforcement
14. **Add integration test for budget hard-stop** — Critical path that should have dedicated E2E coverage
15. **Document environment variable precedence** — Config hierarchy (env > file > default) should be explicitly documented for operators
