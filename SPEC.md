# corkboard

> A moderator-shaped agent orchestrator with a bulletin-board view of work,
> built on the Claude Agent SDK and shippable as a single `npx` command.

**Status:** Draft spec, v0.1
**Working name:** `corkboard` (placeholder — alternatives: `marshal`, `dispatch`, `rookery`)
**License:** MIT
**Distribution:** `npx corkboard` from npm

---

## 1. TL;DR

`corkboard` is a long-running moderator process that sits above one or more
repositories (or arbitrary task domains) and orchestrates work across
ephemeral Claude Code workers. State lives in a pluggable bulletin board.
Any task in flight is **teleportable**: the operator can drop into the live
session from the CLI (`claude --resume <id>`), intervene by hand, and hand
control back to the moderator without losing context.

The whole thing runs from a single `npx corkboard` invocation.

---

## 2. Vision

Three observations drive the design:

1. **The agent loop is general.** The same harness that fixes a bug can plan
   a refactor, audit a directory, draft a chapter, or triage a backlog. The shape of the work is "task graph + scoped execution context."
2. **Long-running orchestration needs a brain that persists.** Workers should be ephemeral and isolated; the planner should not be.
3. **The human is part of the loop.** Not as a babysitter, but as an escalation path.

`corkboard` is the smallest plausible thing that satisfies all three.

---

## 3. Core concepts

Four nouns. Everything else is implementation.

### 3.1 Task
A unit of work with a status, a scope (which repo / directory / domain), a
prompt or spec, and zero-or-more links to other tasks (depends-on, blocks,
spawned-from). Tasks live on the **Board**.

### 3.2 Worker
An ephemeral Claude Code instance dispatched to execute one Task.
Has its own session, its own `cwd`, its own `allowedTools` allowlist.Returns a structured result and dies.
Workers do not see other workers' contexts.

### 3.3 Session
The on-disk conversation log (`~/.claude/projects/<encoded-cwd>/<id>.jsonl`) produced by any Claude Code invocation.
Identified by a UUID.
Sessions are first-class artifacts: stored on the Board, referenced by ID, resumable from either the SDK or the CLI.

### 3.4 Board
The bulletin board.
The single source of truth for tasks, links, worker results, decisions, and session IDs.
Pluggable backend (see §8).
The moderator and dashboard both read/write through it.

---

## 4. Architecture

```
┌─────────────────────┐         ┌──────────────────────┐
│   Browser (Svelte)  │ ◄──SSE──┤  Fastify (127.0.0.1) │
│   log-cards         │ ──REST─►│  /api/* + /events    │
└─────────────────────┘         └──────────┬───────────┘
                                           │ in-process
                                ┌──────────▼───────────┐
                                │  Moderator (SDK loop)│
                                │  plan → dispatch     │
                                └──────────┬───────────┘
                                           │
                                ┌──────────▼───────────┐
                                │  Board (pluggable)   │
                                │  watch() → events    │
                                └──────────┬───────────┘
                                           │ spawn
                                  ┌────────▼────────┐
                                  │  Workers        │
                                  │  (claude -p)    │
                                  └─────────────────┘
```

| Layer        | Component                       | Lifetime    |
|--------------|---------------------------------|-------------|
| Trigger      | cron / webhook / manual         | per-tick    |
| Brain        | moderator SDK loop              | long-lived  |
| Hands        | workers (`claude -p` subprocs)  | per-task    |
| State        | Board (pluggable)               | durable     |
| Service      | Fastify on `127.0.0.1` (REST+SSE) | long-lived |
| View         | Browser dashboard (SvelteKit)   | on-demand   |
| Intervention | `claude --resume <id>`          | on-demand   |

The moderator never edits files. Workers never plan. The Board is the only
shared truth. The Fastify service and moderator share one Node process —
no internal IPC; the moderator subscribes to `Board.watch()` for dispatch
and the SSE routes subscribe to the same emitter for browser fanout.

---

## 5. CLI interface

The CLI is deliberately thin: lifecycle and config only. All per-task
interaction (create, list, inspect, cancel, link, resume) happens in the
browser dashboard. Direct session intervention is done with the raw
`claude --resume <session-id>` CLI (see §6).

```
npx corkboard <command> [options]
```

### 5.1 Commands

| Command            | Purpose                                              |
|--------------------|------------------------------------------------------|
| `corkboard init`   | Scaffold `corkboard.config.ts` in cwd                |
| `corkboard up`     | Start moderator + service; print dashboard URL       |
| `corkboard down`   | Graceful shutdown; persist state                     |
| `corkboard status` | One-shot snapshot of moderator + counts, exits 0     |

`corkboard up` accepts `--port <n>` and `--storage <dir>` overrides; both
otherwise come from env / config.

### 5.2 Environment variables

| Variable             | Default        | Purpose                          |
|----------------------|----------------|----------------------------------|
| `PORT`               | `4321`         | Fastify bind port (127.0.0.1)    |
| `STORAGE_DIR`        | `./.corkboard` | Board DB + audit log location    |
| `CORKBOARD_CONFIG`   | `./corkboard.config.ts` | Path to config file     |

---

## 6. Session teleportation

The defining feature.
Built on the fact that Claude Code sessions are on-disk JSONL files shared between CLI and SDK.

### 6.1 Worker → CLI (debug an in-flight worker)

From the browser dashboard, the log-card for a task has a **Pause** control
and an **Open in CLI** action that surfaces a copy-ready
`claude --resume <session-id>` command (plus a "Mark awaiting-resume"
button that flips the task's Board status).

```
[browser log-card: task-1729 / spectrum]
  status: in-flight  ▸  [pause] → status: paused
  session: 7e3a1f0c-...
  [open in CLI]  →  copies: claude --resume 7e3a1f0c-...

$ claude --resume 7e3a1f0c-...
  [interactive CLI opens, full conversation history loaded]
  [user types, agent responds, files get fixed]
  > /exit

[browser log-card: task-1729]
  [resume]  →  POST /api/tasks/1729/resume → moderator picks up at new tip
```

While a task is `paused` or `awaiting-resume`, the moderator does **not**
dispatch new work on that session. Concurrency on a single session log is
unsafe; the moderator treats those tasks as "out for repair."

### 6.2 CLI → service (productionize an interactive session)

```
$ claude
  [interactive prototyping; the user solves a problem]
  [session id appears in result message]
  > /exit

[browser dashboard: "+ new task"]
  resume session: <session-id>
  scope: ./packages/foo
  prompt: "Continue from where I left off"
  → POST /api/tasks  →  task-1730 created, will resume on next dispatch
```

Power users can hit the same endpoint directly:
`curl -X POST http://127.0.0.1:4321/api/tasks -d '{"resume":"<id>","scope":"...","prompt":"..."}'`.

### 6.3 Moderator teleport

The moderator has its own session. To intervene in the plan itself, copy
the moderator session id from `GET /api/status` (or the dashboard header)
and run `claude --resume <moderator-session-id>` in any terminal. On
`/exit`, click **Resume moderator** on the dashboard and the loop picks
up with the corrected plan.

### 6.4 Fidelity guarantee

To ensure SDK ↔ CLI teleport is lossless, the SDK side pins:

```ts
{
  systemPrompt: { type: 'preset', preset: 'claude_code' },
  settingSources: ['user', 'project'],
  resume: sessionId,
  allowedTools: [...workerProfile.tools],
}
```

Without these, the SDK resumes with a minimal prompt and CLAUDE.md is ignored — the model "remembers what" but "forgets how."
`corkboard` sets them by default; opt out per-worker if you need it.

### 6.5 Hard rules

- A teleported session is **locked** to the CLI for its duration.
- The moderator detects locks via a Board flag, not by polling the JSONL.
- Cross-host teleport requires syncing the session file; v1 is single-host.

---

## 7. Browser dashboard

The browser is the only UI. SvelteKit, built static, served by Fastify
on `http://127.0.0.1:${PORT}/`. Live updates come from `/api/events`
(SSE). Mutations are REST. No TUI.

### 7.1 Layout

Single page, two regions:

```
┌─ corkboard ────────────────────────────────────────────┐
│  TASKS              │  AGENT LOG-CARDS                 │
│  ────────           │  ──────────────                  │
│  ▸ task-1729 (run)  │  ┌─ task-1729 / spectrum ──────┐ │
│    task-1731        │  │ [stream tail, autoscroll]  │ │
│    task-1733 (blk)  │  │ [pause][cancel][open-cli]  │ │
│                     │  └────────────────────────────┘ │
│  [+ new task]       │  ┌─ task-1731 / believer ─────┐ │
│                     │  │ ...                        │ │
└─────────────────────┴──────────────────────────────────┘
```

- **Left rail.** Task list from `GET /api/tasks`, live-updated by
  `task.created` / `task.updated` events on `/api/events`. Click a task to
  open or focus its log-card.
- **Log-cards.** One per active or pinned task. Each card subscribes to
  `GET /api/tasks/:id/stream` (SSE) for worker stdout. Card controls call
  REST (`/cancel`, `/resume`, etc.); the "Open in CLI" control surfaces
  the `claude --resume <id>` command to copy-paste.
- **Header.** Moderator status (running/paused/replanning) + counts, fed
  by `GET /api/status` and `moderator.replan` events.

### 7.2 Read-only export

`GET /api/export?format=json|md` dumps the current Board. Useful for
sharing state, archiving, or piping into Reqall.

### 7.3 HTTP service API

Fastify on `127.0.0.1:${PORT}`. REST for mutations and snapshots; SSE for
streams. JSON in/out. No auth — loopback-only bind is the boundary.

#### REST

| Method | Path                        | Purpose                                   |
|--------|-----------------------------|-------------------------------------------|
| GET    | `/api/status`               | Moderator state + counts                  |
| GET    | `/api/tasks`                | List tasks; `?status=open` etc.           |
| POST   | `/api/tasks`                | Create task (body: `{scope,prompt,resume?}`) |
| GET    | `/api/tasks/:id`            | Full task + linked sessions               |
| PATCH  | `/api/tasks/:id`            | Update status / prompt / scope            |
| POST   | `/api/tasks/:id/cancel`     | Soft-cancel; SIGTERM worker if in-flight  |
| POST   | `/api/tasks/:id/pause`      | Mark paused; worker SIGSTOP if running    |
| POST   | `/api/tasks/:id/resume`     | Hand control back after CLI teleport      |
| POST   | `/api/tasks/:id/link`       | Body: `{to, kind}`                        |
| GET    | `/api/sessions/:id`         | Session metadata + recent JSONL excerpt   |
| GET    | `/api/sessions/:id/log`     | Paginated full session log                |
| POST   | `/api/sessions/:id/fork`    | Branch session; returns new id            |
| GET    | `/api/export`               | `?format=json\|md` (see §7.2)             |

#### SSE

| Path                          | Stream                                                  |
|-------------------------------|---------------------------------------------------------|
| `GET /api/events`             | Board-wide events: task created / updated / linked, worker started / finished, moderator replan. |
| `GET /api/tasks/:id/stream`   | Tail of worker stdout/stderr + structured SDK messages for one task. Closes on worker exit. |

Event shape:

```ts
type BoardEvent =
  | { type: 'task.created' | 'task.updated', task: Task }
  | { type: 'task.linked', from: Id, to: Id, kind: LinkKind }
  | { type: 'worker.started' | 'worker.done', taskId: Id, result?: WorkerResult }
  | { type: 'worker.stdout', taskId: Id, line: string }
  | { type: 'moderator.replan', plan: PlanSnapshot }
```

Reconnect: clients send `Last-Event-ID` and the server replays from the
audit log (§10.5), which doubles as the replay buffer.

---

## 8. Storage backend

Pluggable. Ship two adapters out of the box:

| Adapter   | Use case                                | Config                   |
|-----------|-----------------------------------------|--------------------------|
| `sqlite`  | Default. Zero-config local file.        | `./corkboard.db`         |
| `reqall`  | Cross-project / shared memory.          | OAuth or token           |

A backend is a class implementing `Board`:

```ts
interface Board {
  task: { create, get, list, update, link, search }
  session: { register, get, lock, unlock, list }
  result: { append, list }
  watch(callback): Unsubscribe   // realtime push to dashboard
}
```

Anyone can publish `@scope/corkboard-adapter-foo` and wire it in via
config. Postgres, S3-backed JSON, Notion, Linear — all plausible.

---

## 9. Configuration

`corkboard.config.ts` in the project root:

```ts
import { defineConfig } from 'corkboard'

export default defineConfig({
  board: {
    adapter: 'sqlite',
    path: './corkboard.db',
  },

  workers: {
    spectrum: {
      cwd: '~/code/osteostrong/spectrum',
      allowedTools: ['Read', 'Edit', 'Grep', 'Glob', 'Bash(npm:*)'],
      claudeMd: true,
      hooks: ['./hooks/no-prod-secrets.ts'],
    },
    believer: {
      cwd: '~/code/osteostrong/believer_app',
      allowedTools: ['Read', 'Edit', 'Grep', 'Glob'],
    },
    docs: {
      cwd: '~/notes',
      allowedTools: ['Read', 'Edit', 'WebSearch'],
    },
  },

  moderator: {
    model: 'claude-sonnet-4-5',
    maxParallelWorkers: 3,
    replanEvery: 5,        // re-evaluate plan every 5 worker completions
    sessionPersist: true,  // moderator's own session survives restart
  },

  dashboard: {
    tui: true,
    web: { enabled: true, port: 4321 },
  },
})
```

---

## 10. Guardrails

### 10.1 Scope enforcement

Every worker registers its `cwd` with the Board. A PreToolUse hook
intercepts every `Edit`, `Write`, and `Bash` call and rejects anything
outside that root. This is enforced in code, not prompt.

### 10.2 Verification workers

A task may declare a `verify` step (lint, type-check, test). The moderator
will not mark a task `done` until a separate verification worker — fresh
context, no edit tools — confirms success. Workers don't grade their own
homework.

### 10.3 Cycle detection

`POST /api/tasks/:id/link` refuses cycles in the dependency graph. The
moderator refuses to dispatch a task with unmet `depends-on`.

### 10.4 Cost ceilings

Per-task and per-day token budgets in config. Exceeding either pauses
dispatch and posts a notice to the Board.

### 10.5 Audit log

Every dispatch, every result, every teleport is appended to an
append-only event log. The log doubles as the SSE replay buffer (§7.3),
and `GET /api/export?format=md` includes the full audit trail. Cheap
insurance.

---

## 11. MVP scope (v0.1)

The smallest thing that's honest:

- [ ] `corkboard init` / `up` / `down` / `status`
- [ ] SQLite Board adapter
- [ ] Single-worker dispatch (serial, no parallelism)
- [ ] Moderator SDK loop: plan → dispatch one task → ingest result → repeat
- [ ] Fastify service on `127.0.0.1`: REST endpoints from §7.3
- [ ] SSE: `/api/events` + `/api/tasks/:id/stream`
- [ ] Browser dashboard (SvelteKit, static, served by Fastify): task list + log-cards
- [ ] Worker → CLI teleport via copy-paste `claude --resume <id>` from the log-card
- [ ] Scope-enforcement PreToolUse hook

Explicit non-goals for v0.1: parallel workers, Reqall adapter,
verification workers, cross-host teleport, in-browser xterm/PTY, auth.
All layer cleanly on top once the core loop is solid.

---

## 12. Phases

| Version | Headline                                                   |
|---------|------------------------------------------------------------|
| v0.1    | MVP above. Single-host, single-operator, serial dispatch, browser UI. |
| v0.2    | Parallel workers + cycle detection + audit log.            |
| v0.3    | Reqall adapter. Verification workers.                      |
| v0.4    | Webhook triggers (GitHub, claude-cron). Cost ceilings.     |
| v1.0    | Cross-host teleport. Multi-operator coordination.          |
| later   | Federated boards. Non-code worker types as first-class.    |

---

## 13. Non-goals

- **A new agent framework.** This is glue around the Agent SDK, not a
  competitor to it. If the SDK gains a feature, we adopt it; we don't
  reinvent it.
- **A Claude Code replacement.** The CLI stays the CLI. Teleport is the
  whole point.
- **A general workflow engine.** No DAG executor with retries and
  exponential backoff. Tasks are intentionally coarse-grained; if you
  need that, use Temporal.
- **Multi-tenant SaaS.** Single-operator, local-first. A hosted version
  could be built later; the spec assumes you run it yourself.

---

## 14. Open questions

- **Worker isolation level.** Subagents-in-process (cheap, shared
  Node runtime) vs. `claude -p` subprocesses (expensive, true isolation,
  visible in `term-party`)? Default to subprocesses for v0.1; offer
  in-process as an opt-in for the patient.
- **Session storage portability.** If a user wants to back up or version
  their corkboard, the JSONL files matter as much as the SQLite DB.
  Should `GET /api/export` bundle them?
- **Multi-operator coordination.** Two humans on the same Board — who
  owns a teleport lock? Probably out of scope for v1, but the data model
  should not preclude it.
- **Plugin distribution.** Adapters and hooks should be npm packages,
  not files. How does discovery work — `corkboard.config.ts` import, or
  a registry?
- **Naming.** `corkboard` is descriptive but unclaimed-checking pending.

---

## 15. Contributing

MIT licensed. PRs welcome. Two principles:

1. **No magic.** Everything observable through the Board, replayable
   through the audit log. If a feature can't be inspected, it doesn't
   ship.
2. **The CLI is the spec.** Anything the moderator can do, the operator
   can do by hand through `claude` + Board edits. The SDK side is an
   accelerator, not a gate.

---

*This spec is itself a corkboard task.*