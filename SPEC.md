# corkboard

> A moderator-shaped agent orchestrator with a bulletin-board view of work,
> built around a narrow Claude Code runtime adapter and shippable as a
> single `npx` command.

**Status:** Draft spec, v0.1
**Working name:** `corkboard` (placeholder — alternatives: `marshal`, `dispatch`, `rookery`)
**License:** MIT
**Distribution:** `npx corkboard` from npm

---

## 1. TL;DR

`corkboard` is a long-running moderator process that sits above one or more
repositories (or arbitrary task domains) and orchestrates work across
ephemeral workers through the bundled Claude Code runtime adapter. The adapter
boundary keeps Claude-specific SDK, CLI, hook, and transcript behavior out of
the moderator and Board. State lives in the FS Board adapter in v0.1. With the
bundled adapter, active tasks are **teleportable**: the operator can interrupt
the worker, take the session under lock, intervene by hand, and hand control
back to the moderator without losing context.

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

Five nouns and two channels. Everything else is implementation.

### 3.1 Task
A unit of work with a status, a scope (which repo / directory / domain), a
prompt or spec, and zero-or-more links to other tasks (depends-on, blocks,
spawned-from). Tasks live on the **Board**.

### 3.2 Worker
An ephemeral agent runtime invocation dispatched to execute one Task.
Has its own session, its own `cwd`, and its own tool policy. Returns a
structured result and dies. Workers do not see other workers' contexts.

### 3.3 Agent adapter
A provider-specific implementation that turns normalized corkboard requests
into model/runtime calls and emits normalized output frames. The core talks
to a runtime adapter boundary, not to `claude -p`, provider SDK message
objects, or provider-native transcript files. The bundled v0.1 adapter is
`claude-code`.

### 3.4 Session
A provider-scoped conversation handle. Identified inside corkboard by a UUID,
with adapter metadata that records the provider session id, transcript
locator, and handoff capability. With the default Claude Code adapter, this
maps to the on-disk JSONL under
`~/.claude/projects/<encoded-cwd>/<id>.jsonl`.

Sessions are first-class artifacts: referenced by ID on the Board, resumable
through the v0.1 Claude Code adapter, and visible to the dashboard through
live normalized frames while a worker is running. Durable transcripts remain
provider-owned; v0.1 uses them internally for handoff validation, not as a
public transcript browsing API.

### 3.5 Board
The bulletin board.
The single source of truth for tasks, links, worker results, decisions, agent
adapter ids, and session IDs.
Pluggable backend (see §8).
In v0.1, that backend is the built-in FS Board adapter.
The moderator and dashboard both read/write through it.

### 3.6 Channels

Two distinct streams flow through the system. Conflating them is the
fastest way to make corkboard incomprehensible.

- **Transcripts.** The rolling back-and-forth between operator / moderator
  / workers — model output, tool calls, intermediate reasoning. Streamed
  live to the browser; **not persisted by corkboard**. Transcript storage is
  owned by the active agent adapter. The default Claude Code adapter reads the
  CLI session JSONL under `~/.claude/projects/`; corkboard does not duplicate
  it.
- **Audit trail.** A small, fixed vocabulary of events and state
  transitions (see §7.3 / §11.5). Append-only, queryable, replayable.
  This is what gets stored, exported, and shipped to a Reqall adapter.

The dashboard subscribes to both: log-cards tail the transcript channel;
the task list, status header, and history view are driven by the audit
channel.

---

## 4. Architecture

```
┌─────────────────────┐         ┌──────────────────────┐
│   Browser (Svelte)  │ ◄──SSE──┤  Fastify (127.0.0.1) │
│   log-cards / audit │ ──REST─►│  /api/* + /events    │
└─────────────────────┘         └──────────┬───────────┘
                                           │ in-process
                                ┌──────────▼───────────┐
                                │  Moderator loop      │
                                │  plan → dispatch     │
                                └──────────┬───────────┘
                                           │
                                ┌──────────▼───────────┐
                                │  Board (FS v0.1)     │   ┌──────────────┐
                                │  audit JSONL + MD    │──►│ ./cb-store/  │
                                │  watch() → events    │   │ events.jsonl │
                                └──────────┬───────────┘   │ tasks/*.md   │
                                           │ spawn         └──────────────┘
                                  ┌────────▼────────┐
                                  │  Workers        │  transcripts live
                                  │  (adapter-owned)│  provider-owned
                                  └─────────────────┘  (not duplicated)
```

| Layer        | Component                       | Lifetime    |
|--------------|---------------------------------|-------------|
| Trigger      | cron / webhook / manual         | per-tick    |
| Brain        | moderator loop via agent adapter | long-lived  |
| Hands        | workers (adapter subprocs/API calls) | per-task |
| Audit store  | Board (FS JSONL+MD by default)  | durable     |
| Transcripts  | Provider session logs/artifacts | durable (owned by adapter/provider) |
| Service      | Fastify on `127.0.0.1` (REST+SSE) | long-lived |
| View         | Browser dashboard (SvelteKit)   | on-demand   |
| Intervention | adapter handoff (`claude --resume <id>` by default) | on-demand |

The moderator never edits files. Workers never plan. Corkboard core never
calls a model/provider directly; all agent I/O goes through
a runtime adapter boundary. The Board is the only shared truth for *audit*;
transcripts are owned by the adapter/provider and exposed only as live
normalized frames in v0.1. The Fastify service and moderator share one Node process — no
internal IPC; the moderator subscribes to
`Board.watch()` for dispatch and the SSE routes subscribe to the same
emitter for browser fanout.

---

## 5. CLI interface

The CLI has no subcommands. Lifecycle (start/stop), task management,
session intervention, and inspection are all UI surfaces — they live in
the browser, in adapter handoff (`claude --resume <id>` for the default
Claude Code adapter), or in a `SIGINT` on the running process. A subcommand
layer would just be a worse rendering of those.

```
npx corkboard --cwd <dir> --storepath <dir> [--port <n>]
```

| Flag             | Default                       | Purpose                                |
|------------------|-------------------------------|----------------------------------------|
| `--cwd <dir>`    | `process.cwd()`               | Root for moderator + default worker scope |
| `--storepath <dir>` | `./cb-store`               | Where audit JSONL + MD records are written |
| `--port <n>`     | `4321`                        | Fastify bind port on `127.0.0.1`       |
| `--config <file>` | `<cwd>/corkboard.config.ts`  | Optional config file for worker profiles |

Running it does one thing: bind the service, open the Board at
`--storepath`, run the moderator loop, print the dashboard URL, and block
until `SIGINT`. Everything else is a request on the API.

Automation also targets the API. Scripts, agents, and other corkboards must
be given the URL of the running service they should talk to, for example
`CORKBOARD_URL=http://127.0.0.1:4321`. v0.1 does not discover, enumerate, or
select among running corkboards, and the CLI does not proxy API calls into a
separate control surface.

The same invocation is the only entrypoint; `npx corkboard --help` prints
flags and exits.

---

## 6. Session teleportation

The defining feature of the bundled Claude Code adapter.

Corkboard models teleport through the runtime adapter boundary. The Claude
Code adapter exposes a copyable CLI command plus lock/unlock semantics that let
the moderator stop touching a mutable session while a human has it. It uses
on-disk JSONL sessions shared between Claude CLI and SDK.

### 6.1 Worker → CLI (debug an in-flight worker)

From the browser dashboard, the log-card for a task has **Cancel** and
**Handoff to CLI** actions. Handoff is interrupt-then-lock: if the worker is
still active, corkboard asks the adapter to stop it; once no adapter writer
remains, corkboard emits `TELEPORT_LOCK` and surfaces a copy-ready
`claude --resume <session-id>` command. If the adapter cannot stop cleanly,
the session stays with the worker and no handoff target is issued.

```
[browser log-card: task-1729 / spectrum]
  status: in-flight  ▸  [handoff] → adapter cancel → TELEPORT_LOCK
  status: awaiting-resume
  session: 7e3a1f0c-...
  [open in CLI]  →  copies: claude --resume 7e3a1f0c-...

$ claude --resume 7e3a1f0c-...
  [interactive CLI opens, full conversation history loaded]
  [user types, agent responds, files get fixed]
  > /exit

[browser log-card: task-1729]
  [resume]  →  POST /api/tasks/1729/resume → adapter validates return
            →  TELEPORT_UNLOCK → moderator redispatches from validated tip
```

While a session has an active `TELEPORT_LOCK`, the moderator does **not**
dispatch new work on that session. v0.1 keeps the task model small: workers
run, are cancelled/interrupted, or are handed to the operator under a lock.
Concurrency on a single session log is unsafe; the moderator treats locked
sessions as "out for repair."

### 6.2 CLI → service (productionize an interactive session)

This is adapter-specific. The Claude Code adapter supports importing an
existing CLI session by id:

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

The moderator has its own durable Claude Code session. Copy the moderator
session id from `GET /api/status` (or the dashboard header) and run
`claude --resume <moderator-session-id>` in any terminal. On `/exit`, click
**Resume moderator** on the dashboard. The same unlock handshake applies:
the adapter validates return, records the transcript tip/hash, and only then
does the loop pick up with the corrected plan.

### 6.4 Claude Code fidelity guarantee

To ensure Claude SDK ↔ Claude CLI teleport is lossless, the Claude Code
adapter pins:

```ts
const toolPolicy = resolveToolPolicy(workerProfile)

{
  systemPrompt: { type: 'preset', preset: 'claude_code' },
  settingSources: ['user', 'project'],
  resume: sessionId,
  cwd: workerProfile.cwd,
  allowedTools: compileClaudeAllowedTools(toolPolicy),
  disallowedTools: toolPolicy.claudeCode?.disallowedTools,
  permissionMode: toolPolicy.claudeCode?.permissionMode ?? 'default',
}
```

Without these, the SDK resumes with a minimal prompt and CLAUDE.md is ignored
— the model "remembers what" but "forgets how." This is adapter-local
behavior, not a core corkboard contract. `corkboard` sets it by default for
the Claude Code adapter; opt out per-worker if you need it.

### 6.5 Hard rules

- A teleported session is **locked** to the CLI for its duration. Entering
  teleport emits `TELEPORT_LOCK { lock.id, session.id, task.id, by, at,
  baseTip, baseTranscriptHash }`.
- Dashboard **Resume** does not unlock a session. It emits
  `TELEPORT_UNLOCK_REQUESTED { lock.id, session.id, task.id, by, at }`.
  The session remains locked until the adapter validates the return.
- Adapter return validation must prove that the request matches the active
  lock, that the handoff writer is no longer active, and that the adapter can
  read a stable transcript cursor. On success, corkboard emits
  `TELEPORT_UNLOCK { lock.id, session.id, task.id, tip, transcriptHash,
  returnedState, duration }`. On failure, corkboard emits
  `TELEPORT_UNLOCK_REJECTED { lock.id, session.id, task.id, reason,
  retryable }` and leaves the lock in place.
- The moderator may redispatch only after a validated `TELEPORT_UNLOCK`, and
  it must resume from the `tip` and `transcriptHash` recorded on that event.
- The moderator detects locks via the Board (driven by those events), not
  by polling the JSONL.
- Future adapters that cannot guarantee exclusive mutable session access must
  use a non-teleportable adapter profile; the dashboard hides handoff controls
  for those sessions.
- Cross-host teleport requires adapter-level transcript/session sync; v1 is
  single-host.

### 6.6 Unlock handshake

The unlock handshake is deliberately event-first:

1. `TELEPORT_LOCK` records the session being handed to the operator, a
   unique `lock.id`, and the transcript cursor/hash at the handoff boundary.
2. `TELEPORT_UNLOCK_REQUESTED` records the operator's request to return the
   session. This is a pending request, not permission for the moderator to
   resume.
3. The agent adapter validates the return. It must compare the request to
   the active lock, confirm that the human handoff has exited or otherwise
   released exclusive write access, and return a stable transcript `tip` plus
   `transcriptHash`.
4. `TELEPORT_UNLOCK` clears the lock and becomes the only resume point the
   moderator is allowed to trust. `TELEPORT_UNLOCK_REJECTED` keeps the lock
   active and should surface the rejection reason in the dashboard.

`tip` is an adapter-native transcript cursor that can be used to resume or
page from a precise point. `transcriptHash` is the adapter's canonical hash
of transcript content through that `tip`. `returnedState` is one of
`cleanExit`, `aborted`, or `unknown`; `unknown` is acceptable only when the
adapter can still prove that no handoff writer remains active.

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
│    task-1733 (blk)  │  │ [cancel][handoff]          │ │
│                     │  └────────────────────────────┘ │
│  [+ new task]       │  ┌─ task-1731 / believer ─────┐ │
│                     │  │ ...                        │ │
└─────────────────────┴──────────────────────────────────┘
```

- **Left rail.** Task list from `GET /api/tasks`, live-updated by
  `TASK_CREATED` / `TASK_STATUS` events on `/api/events`. Click a task to
  open or focus its log-card.
- **Log-cards.** One per active or pinned task. Each card subscribes to
  `GET /api/tasks/:id/stream` (SSE) for normalized transcript frames. Card
  controls call REST (`/cancel`, `/handoff`, `/resume`, etc.); the handoff
  control surfaces the adapter-provided command or URL when available.
- **Header.** Moderator status (running/dispatch-held/replanning) + counts, fed
  by `GET /api/status` and `REPLAN` / `PLAN_TICK` events.

### 7.2 Read-only export

`GET /api/export?format=json|md` dumps the current Board. Useful for
sharing state, archiving, or piping into Reqall.

### 7.3 HTTP service API

Fastify on `127.0.0.1:${PORT}`. REST for mutations and snapshots; SSE for
streams. JSON in/out. No auth, no tokens, no origin checks. Corkboard is a
local-only, single-user tool; the loopback port runs at the operator's user
privilege and inherits whatever trust boundary the operator's machine already
has. If you need to expose it to another host, tunnel it yourself.

#### REST

| Method | Path                        | Purpose                                   |
|--------|-----------------------------|-------------------------------------------|
| GET    | `/api/status`               | Moderator state + counts                  |
| GET    | `/api/tasks`                | List tasks; `?status=open` etc.           |
| POST   | `/api/tasks`                | Create task (body: `{scope,prompt,resume?,workerProfile?}`) |
| GET    | `/api/tasks/:id`            | Full task + linked sessions               |
| PATCH  | `/api/tasks/:id`            | Update status / prompt / scope            |
| POST   | `/api/tasks/:id/cancel`     | Soft-cancel through adapter; terminates worker if supported |
| POST   | `/api/tasks/:id/handoff`    | Interrupt active worker if needed, lock session, return adapter command/URL |
| POST   | `/api/tasks/:id/resume`     | Request return from adapter handoff; validates before redispatch |
| POST   | `/api/tasks/:id/link`       | Body: `{to, kind}`                        |
| GET    | `/api/sessions/:id`         | Session metadata + lock/capability state  |
| GET    | `/api/export`               | `?format=json\|md` (see §7.2)             |

These are the complete v0.1 routes. Session fork, transcript pagination, and
provider transcript export are not mounted in v0.1. Clients that call an
unmounted future route receive the standard Fastify 404 response. Mounted
routes use this JSON error body:

```json
{
  "error": {
    "code": "INVALID_STATE",
    "message": "human readable summary",
    "details": {}
  }
}
```

The small initial code set is `NOT_FOUND`, `INVALID_STATE`,
`CONFLICTING_WRITE`, `VALIDATION_FAILED`, and `ADAPTER_ERROR`. Do not add
`UNSUPPORTED_OPERATION` in v0.1; unsupported operations should not be part of
the mounted API surface.

#### SSE

Two streams, one per channel (see §3.6):

| Path                          | Channel    | Stream                                                  |
|-------------------------------|------------|---------------------------------------------------------|
| `GET /api/events`             | Audit      | All audit events (vocabulary below). Used to drive task list, status header, history view. |
| `GET /api/tasks/:id/stream`   | Transcript | Live normalized adapter frames for one task. Not persisted by corkboard. Closes on worker exit. |

Reconnect on `/api/events`: clients send `Last-Event-ID` and the server
replays from the audit log (§11.5), which **is** the persistent store —
the audit JSONL doubles as the replay buffer. `/api/tasks/:id/stream`
does not replay; reconnects only pick up live output.

#### Audit event vocabulary

Fixed, deliberately small. Adding events requires a real question to
answer; if no one will ask it later, don't ship it.

| Event              | Payload                                                              |
|--------------------|----------------------------------------------------------------------|
| `TASK_CREATED`     | `task.id`, `source` (`cli\|moderator\|webhook`), `spec`              |
| `TASK_LINKED`      | `from`, `to`, `kind` (`depends-on\|blocks\|spawned-from`)            |
| `TASK_STATUS`      | `task.id`, `from`, `to`, `reason`                                    |
| `TASK_CANCELLED`   | `task.id`, `by`                                                      |
| `DISPATCH`         | `task.id`, `worker.profile`, `agent.adapter`, `session.id`, `promptHash`, `git.sha`, `configSnapshot` |
| `RESULT`           | `task.id`, `session.id`, `agent.adapter`, `outcome` (`ok\|fail\|blocked`), `filesTouched[]`, `tokens`, `duration`, `summary` |
| `PLAN_TICK`        | `moderator.session.id`, `turn`, `decision`, `candidates[]`, `chosen`, `rationaleHash` |
| `REPLAN`           | `moderator.session.id`, `reason`, `beforeHash`, `afterHash`          |
| `SESSION_OPEN`     | `session.id`, `agent.adapter`, `kind` (`worker\|moderator`), `cwd`, `externalId?`, `transcriptLocator?` |
| `SESSION_CLOSE`    | `session.id`, `reason` (`done\|error\|teleport`)                     |
| `TELEPORT_LOCK`    | `lock.id`, `session.id`, `task.id`, `by` (operator), `at`, `baseTip`, `baseTranscriptHash` |
| `TELEPORT_UNLOCK_REQUESTED` | `lock.id`, `session.id`, `task.id`, `by`, `at`               |
| `TELEPORT_UNLOCK`  | `lock.id`, `session.id`, `task.id`, `tip`, `transcriptHash`, `returnedState` (`cleanExit\|aborted\|unknown`), `duration` |
| `TELEPORT_UNLOCK_REJECTED` | `lock.id`, `session.id`, `task.id`, `reason` (`staleLock\|stillActive\|missingTranscript\|hashMismatch\|unsupported\|adapterError`), `retryable` |
| `HOOK_REJECT`      | `worker.session.id`, `tool`, `argsHash`, `rule`, `reason`            |
| `SCOPE_VIOLATION`  | `worker.session.id`, `attemptedPath`, `allowedRoot`                  |
| `VERIFY`           | `task.id`, `kind` (`lint\|test\|typecheck`), `outcome`, `by`         |
| `COST`             | `task.id`, `agent.adapter`, `model`, `inputTokens`, `outputTokens`, `usd` |
| `ERROR`            | `source`, `kind`, `payload`, `recoverable`                           |

Each event on the wire is `{ id, ts, type, ...payload }`. `id` is a
monotonic sequence used for `Last-Event-ID` resume.

---

## 8. Storage backend

The Board stores **audit data only** (events + task records). Transcripts
are not persisted by corkboard — they live wherever the active agent adapter
stores them and are read through that adapter when needed. With the bundled
Claude Code adapter, that means JSONLs under `~/.claude/projects/`.

v0.1 ships filesystem-only. Database-backed Boards are later explicit adapter
work, not part of the initial FS contract.

| Adapter   | Use case                                | Notes                    |
|-----------|-----------------------------------------|--------------------------|
| `fs`      | **Default.** Zero-config, human-readable, greppable. | `--storepath ./cb-store/` |
| `reqall`  | Cross-project / shared memory.          | Community adapter (v0.3) |
| `sqlite`  | Single-file DB for higher event rates.  | Community adapter        |

### 8.1 FS layout (default)

Everything under `--storepath` (default `./cb-store`):

```
cb-store/
├── events.jsonl            # append-only audit channel; one event per line
├── events.idx              # sparse index: byte offsets by event id
├── tasks/
│   ├── 1729.md             # one MD file per task; frontmatter + body
│   └── 1731.md
├── sessions/
│   └── 7e3a1f0c-....json   # session metadata (id, adapter, cwd, locks)
└── config.snapshot.json    # captured at startup, referenced by DISPATCH
```

`events.jsonl` is the source of truth; the `tasks/` and `sessions/` files
are materialized projections rebuildable by replaying events from
position 0. The dashboard reads projections; the SSE replay reads the
JSONL directly.

The FS adapter is deliberately simple in v0.1:

- A store has one writer. On startup, corkboard acquires an exclusive
  `writer.lock` under `--storepath`; if another live process holds it, startup
  fails with a user-facing message instead of sharing the store.
- Each event append writes one complete newline-terminated JSON record, flushes
  it, and fsyncs `events.jsonl` before the append is acknowledged.
- `tasks/`, `sessions/`, and `events.idx` are disposable projections. If any
  projection file is missing, unreadable, invalid, or inconsistent, corkboard
  moves the bad file aside when possible and rebuilds projections by replaying
  `events.jsonl` from position 0.
- On startup, corkboard scans `events.jsonl`. A partial trailing line may be
  truncated back to the last valid newline. Invalid JSON or an invalid event in
  the middle of the log is not skipped silently; corkboard stops, reports the
  path, byte offset, and reason, and asks the user to repair or restore the log.
- If a best-effort rebuild fails, the store remains unavailable until the user
  fixes the corrupted file. Corkboard must not invent events or continue from an
  ambiguous state.

### 8.2 Adapter interface

A backend is a class implementing `Board`:

```ts
interface Board {
  task: { create, get, list, update, link }
  session: { register, get, list, lock, requestUnlock, unlock, rejectUnlock }
  events: { append(event), readFrom(id), watch(cb): Unsubscribe }
}
```

This is the whole v0.1 storage contract. The FS adapter does not provide
full-text search, session fork, provider transcript storage, secondary query
indexes, multi-writer coordination, or cross-host sync. Those require explicit
new Board methods in later versions, not optional behavior hidden behind the
initial interface.

`session.register` stores normalized session metadata only: corkboard id,
agent adapter id, provider-native external id, cwd/scope, transcript locator,
capabilities, and lock state. It does not persist provider transcripts.

Later versions can allow `@scope/corkboard-board-foo` packages wired through
config. Postgres, SQLite, S3-backed JSON, Notion, Linear — all plausible, but
not part of the v0.1 FS storage surface.

---

## 9. Agent runtime adapter

Storage adapters and agent adapters are separate extension points. Storage
answers "where does the Board live?" Agent runtime adapters answer "how does
corkboard talk to this model/runtime, and how are provider-native events
translated into corkboard frames?"

v0.1 ships one runtime adapter: the bundled Claude Code adapter. The adapter
boundary exists so Claude-specific SDK, CLI, transcript, and hook behavior does
not leak into the moderator or Board, not to promise a stable community adapter
ABI on day one.

The v0.1 adapter contract is intentionally narrow and has no optional
operations:

```ts
interface ClaudeCodeRuntimeAdapter {
  id: 'claude-code'
  kind: 'local-process'
  capabilities(): AgentCapabilities
  start(input: AgentStartInput): AsyncIterable<AgentFrame>
  resume(input: AgentResumeInput): AsyncIterable<AgentFrame>
  cancel(session: SessionRef): Promise<void>
  handoff(session: SessionRef): Promise<HandoffTarget>
  validateHandoffReturn(input: HandoffReturnInput): Promise<HandoffReturn>
}
```

`capabilities()` is descriptive metadata for the dashboard and session record,
not a negotiation surface in v0.1. A Claude Code adapter that cannot start,
resume, cancel, create a CLI handoff target, and validate handoff return is not
a valid v0.1 adapter.

```ts
interface AgentCapabilities {
  runtime: 'claude-code'
  handoff: { kind: 'cli'; commandPattern: 'claude --resume <session-id>' }
  transcriptCursor: { kind: 'claude-jsonl-tip'; hash: 'sha256' }
  cancellation: 'abort-signal-then-process-kill'
  usage: 'agent-frame'
}
```

```ts
interface HandoffTarget {
  kind: 'cli' | 'url' | 'other'
  label: string
  command?: string
  url?: string
  baseTip: string
  baseTranscriptHash: string
}

interface HandoffReturnInput {
  lockId: string
  session: SessionRef
  taskId?: string
  baseTip: string
  baseTranscriptHash: string
}

interface HandoffReturn {
  sessionId: string
  tip: string
  transcriptHash: string
  returnedState: 'cleanExit' | 'aborted' | 'unknown'
}
```

Normalized input is boring on purpose: task id, role (`worker` or
`moderator`), prompt, scope/cwd, optional prior session, model options, env,
and tool policy. Normalized output is also boring: `session_opened`,
`transcript`, `tool_call`, `tool_result`, `usage`, `result`, and `error`
frames. Adapters may include a provider-native `raw` field for debugging,
but core Board events must not depend on provider SDK object shapes.

The bundled adapter is `claude-code`:

- worker start/resume maps to Claude Code SDK / `claude -p`
- session ids and transcript locators map to Claude Code JSONLs
- tool policy maps to `allowedTools`, `disallowedTools`, and PreToolUse hooks
- handoff maps to `claude --resume <session-id>`

Community adapters should ship as npm packages, e.g.
`@scope/corkboard-agent-openai` or `@scope/corkboard-agent-local`, but that is
post-v0.1 work. When community adapters are introduced, define explicit
versioned adapter profiles rather than adding a bag of optional methods to the
v0.1 Claude Code boundary. Teleport/handoff can then be a named profile instead
of an implicit maybe-supported method.

---

## 10. Configuration

Config is optional. With no config file present, corkboard runs with FS
storage at `--storepath`, a single default worker profile (the current
`--cwd` with the bundled `claude-code` adapter and a conservative tool
allowlist), and the default moderator settings. A `corkboard.config.ts` is
only needed to define multiple worker profiles or tune the moderator in v0.1.
Adding agent adapters or swapping the Board adapter is post-v0.1 work.

```ts
import { defineConfig } from 'corkboard'

export default defineConfig({
  board: {
    adapter: 'fs',           // the only v0.1 Board adapter
    // path is overridden by --storepath
  },

  workers: {
    spectrum: {
      cwd: '~/code/osteostrong/spectrum',
      model: 'claude-sonnet-4-5',
      toolPolicy: {
        readRoots: ['.'],
        writeRoots: ['.'],
        allow: ['Read', 'Edit', 'Write', 'Grep', 'Glob', 'Bash'],
        shell: { allow: ['npm:*', 'node:*', 'git:status', 'git:diff'] },
        network: 'deny',
        claudeCode: {
          extraAllowedTools: ['TodoWrite'],
        },
      },
      claudeCode: {
        claudeMd: true,
        hooks: ['./hooks/no-prod-secrets.ts'],
      },
    },
    believer: {
      cwd: '~/code/osteostrong/believer_app',
      toolPolicy: {
        readRoots: ['.'],
        writeRoots: ['.'],
        allow: ['Read', 'Edit', 'Write', 'Grep', 'Glob'],
        shell: { allow: [] },
        network: 'deny',
      },
    },
    docs: {
      cwd: '~/notes',
      toolPolicy: {
        readRoots: ['.'],
        writeRoots: ['.'],
        allow: ['Read', 'Edit', 'Write', 'Grep', 'Glob', 'WebSearch'],
        shell: { allow: [] },
        network: 'web-tools',
      },
    },
  },

  moderator: {
    model: 'claude-sonnet-4-5',
    maxParallelWorkers: 1,  // v0.1 is serial
    replanEvery: 5,        // re-evaluate plan every 5 worker completions
    sessionPersist: true,  // moderator's own session survives restart
  },
})
```

`toolPolicy` is the minimal policy shape needed to run a Claude Code SDK
coding/writing worker. Paths are resolved relative to the worker `cwd` unless
absolute, and symlinks are resolved before enforcement. If a profile omits
`toolPolicy`, Corkboard uses the default policy shown below; if a profile
defines one, the non-optional fields are required.

```ts
type ToolGrant =
  | 'Read'
  | 'Edit'
  | 'Write'
  | 'MultiEdit'
  | 'Grep'
  | 'Glob'
  | 'Bash'
  | 'WebFetch'
  | 'WebSearch'

type ShellGrant = string // '<exe>:*', '<exe>:<subcommand>', or '<exe>'

interface ToolPolicy {
  readRoots: string[]       // default: ['.']
  writeRoots: string[]      // default: ['.']; [] makes the worker read-only
  allow: ToolGrant[]        // normalized tools Corkboard understands
  shell?: {
    allow: ShellGrant[]     // examples: 'npm:*', 'git:status', 'node:*'
  }
  network: 'deny' | 'web-tools' | 'allow'
  claudeCode?: {
    extraAllowedTools?: string[]  // exact Claude Code SDK allowedTools entries
    disallowedTools?: string[]    // exact Claude Code SDK disallowedTools entries
    permissionMode?: 'default' | 'acceptEdits'
  }
}

const defaultToolPolicy: ToolPolicy = {
  readRoots: ['.'],
  writeRoots: ['.'],
  allow: ['Read', 'Edit', 'Write', 'Grep', 'Glob'],
  shell: { allow: [] },
  network: 'deny',
}
```

For the Claude Code adapter, `allow` plus `claudeCode.extraAllowedTools`
compiles to Claude Code SDK `allowedTools`. `Bash` is only emitted when
`shell.allow` is non-empty, and each shell grant compiles to the narrowest
Claude Code `Bash(...)` pattern the adapter can express. The generated
PreToolUse hook still performs the authoritative check against resolved
`readRoots`, `writeRoots`, parsed shell argv, and `network`; SDK allowlists are
treated as a convenience layer, not the security boundary.
`network: 'web-tools'` permits Claude-native `WebSearch`/`WebFetch` only when
those tools are also present in `allow`; shell-launched network programs still
need explicit `shell.allow` grants. `network: 'allow'` permits network-capable
shell commands only when both `Bash` and the matching `shell.allow` grant are
present.

Port and store path are CLI flags only (`--port`, `--storepath`); they
are not in the config file so that the same config can be used across
machines with different paths.

---

## 11. Guardrails

### 11.1 Scope enforcement

Every worker registers its `cwd`/scope with the Board. Corkboard enforces
scope through the adapter's tool-policy hooks, not through prompt text. The
Claude Code adapter implements this with a PreToolUse hook that intercepts
`Read`, `Edit`, `Write`, `MultiEdit`, `Bash`, `WebFetch`, and `WebSearch`
calls and rejects anything outside the profile's `readRoots`, `writeRoots`,
`shell.allow`, and `network` policy. Read checks accept either `readRoots` or
`writeRoots`; write checks require `writeRoots`. Future adapters that cannot
enforce write scope must be marked read-only or rejected for write-capable
worker profiles.

### 11.2 Verification workers

A task may declare a `verify` step (lint, type-check, test). The moderator
will not mark a task `done` until a separate verification worker — fresh
context, no edit tools — confirms success. Workers don't grade their own
homework.

### 11.3 Cycle detection

`POST /api/tasks/:id/link` refuses cycles in the dependency graph. The
moderator refuses to dispatch a task with unmet `depends-on`.

### 11.4 Cost ceilings

Per-task and per-day token budgets in config. Exceeding either stops further
dispatch and posts a notice to the Board. Agent adapters report usage through
normalized `usage` frames; if an adapter cannot report usage, cost ceilings
for that profile are unavailable unless the adapter supplies an estimator.

### 11.5 Audit log

Every dispatch, every result, every teleport is appended to an
append-only event log — `events.jsonl` in the FS adapter, using the
vocabulary in §7.3 (`TASK_CREATED`, `DISPATCH`, `RESULT`, `REPLAN`,
`TELEPORT_LOCK`, `HOOK_REJECT`, ...). The log is the persistent record
*and* the SSE replay buffer (§7.3); `GET /api/export?format=md` renders
it as a human-readable transcript of decisions. Cheap insurance.

---

## 12. MVP scope (v0.1)

The smallest thing that's honest:

- [ ] `npx corkboard --cwd . --storepath ./cb-store [--port N]` single invocation
- [ ] FS Board adapter: `events.jsonl` + `tasks/*.md` + `sessions/*.json`
- [ ] Minimal `claude-code` runtime adapter boundary
- [ ] Single-worker dispatch (serial, no parallelism)
- [ ] Moderator loop via agent adapter: plan → dispatch one task → ingest result → repeat
- [ ] Task link cycle detection in the Board projection
- [ ] Audit vocabulary (§7.3) emitted at every meaningful state change
- [ ] Fastify service on `127.0.0.1`: REST endpoints from §7.3
- [ ] SSE: `/api/events` (audit, with `Last-Event-ID` replay from JSONL) + `/api/tasks/:id/stream` (live transcript)
- [ ] Browser dashboard (SvelteKit, static, served by Fastify): task list + log-cards
- [ ] Worker → CLI teleport through the Claude Code adapter via copy-paste `claude --resume <id>` from the log-card; emits `TELEPORT_LOCK` / `TELEPORT_UNLOCK_REQUESTED` / `TELEPORT_UNLOCK`
- [ ] Scope-enforcement tool policy hook in the Claude Code adapter (emits `SCOPE_VIOLATION` / `HOOK_REJECT`)

Explicit non-goals for v0.1: parallel workers, Reqall / SQLite adapters,
shipping non-Claude agent adapters in core, a public community adapter ABI,
session fork, transcript pagination/export, verification workers, cross-host
teleport, in-browser xterm/PTY. All layer cleanly on top once the core loop is
solid.

---

## 13. Phases

| Version | Headline                                                   |
|---------|------------------------------------------------------------|
| v0.1    | MVP above. Single-host, single-operator, serial dispatch, browser UI. |
| v0.2    | Parallel workers + richer Board projections.                  |
| v0.3    | Reqall adapter. Verification workers. First community agent adapter. |
| v0.4    | Webhook triggers (GitHub, claude-cron). Cost ceilings.     |
| v1.0    | Cross-host teleport. Multi-operator coordination.          |
| later   | Federated boards. Non-code worker types as first-class.    |

---

## 14. Non-goals

- **A new model API framework.** Corkboard owns orchestration contracts;
  provider SDKs and runtime quirks stay inside agent adapters.
- **A Claude Code replacement.** The bundled adapter should preserve Claude
  Code CLI workflows. Teleport is the whole point for that adapter.
- **A universal teleport guarantee.** Teleport quality depends on adapter
  capabilities. The core only guarantees locks, status, and audit events.
- **A general workflow engine.** No DAG executor with retries and
  exponential backoff. Tasks are intentionally coarse-grained; if you
  need that, use Temporal.
- **Corkboard discovery / CLI interop.** One corkboard may talk to another
  only when a user or agent provides the peer service URL explicitly. v0.1
  does not ship subcommands or discovery logic that find and control other
  running corkboards.
- **Multi-tenant SaaS.** Single-operator, local-first. A hosted version
  could be built later; the spec assumes you run it yourself.

---

## 15. Open questions

- **Worker isolation level.** Subagents-in-process (cheap, shared
  Node runtime) vs. `claude -p` subprocesses (expensive, true isolation,
  visible in `term-party`)? Default to subprocesses for v0.1; offer
  in-process as an opt-in for the patient.
- **Session storage portability.** If a user wants to back up or version
  their corkboard, provider transcript artifacts matter as much as the
  Board store. Should `GET /api/export` ask adapters to bundle them?
- **Multi-operator coordination.** Two humans on the same Board — who
  owns a teleport lock? Probably out of scope for v1, but the data model
  should not preclude it.
- **Plugin distribution.** Storage adapters, agent adapters, and hooks
  should be npm packages, not files. How does discovery work —
  `corkboard.config.ts` import, or a registry?
- **Naming.** `corkboard` is descriptive but unclaimed-checking pending.

---

## 16. Contributing

MIT licensed. PRs welcome. Two principles:

1. **No magic.** Everything observable through the Board, replayable
   through the audit log. If a feature can't be inspected, it doesn't
   ship.
2. **The handoff path is the spec.** Anything the moderator can do, the
   operator can do by hand through adapter handoff + Board edits. The
   runtime adapter is an accelerator, not a gate.

---

*This spec is itself a corkboard task.*
