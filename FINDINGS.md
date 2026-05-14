# Spec review findings

Reviewed `SPEC.md` for the agent orchestration bulletin-board design.

## Addressed in this pass

- Agent I/O is no longer hardwired into Claude Code internals.  `SPEC.md` now defines a narrow runtime adapter boundary, normalized input/output frames, adapter-owned transcripts, and Claude Code as the bundled default adapter.
- Teleport is now implemented through the runtime adapter boundary, with
  Claude Code as the v0.1 implementation. Future non-teleportable adapters
  should be modeled as explicit adapter profiles, not optional methods.
- Teleport unlock now uses an adapter-validated return handshake:
  dashboard Resume emits `TELEPORT_UNLOCK_REQUESTED`; the adapter validates
  exclusive return plus transcript `tip`/`transcriptHash`; only validated
  `TELEPORT_UNLOCK` clears the lock, while `TELEPORT_UNLOCK_REJECTED` keeps it
  locked.
- Task creation now allows `workerProfile?`, which matters once multiple
  adapters/profiles exist.
- Audit payload field names now use implementation-friendly camelCase.
- `README.md` now matches the thin single-command CLI described by the spec.
- FS store recovery is now explicit: single-writer lock, fsynced appends,
  rebuildable projections, partial trailing-line recovery, and hard failure
  with repair instructions when `events.jsonl` is ambiguous.
- The v0.1 adapter/storage capability question is closed: the MVP ships a
  minimal `claude-code` adapter boundary and minimal FS Board contract. Session
  fork, transcript pagination/export, public community adapter ABI, alternate
  Board adapters, and `UNSUPPORTED_OPERATION` probing are explicitly out of
  the v0.1 surface.
- `toolPolicy` now has the minimal shape needed for a Claude Code SDK
  coding/writing agent: `readRoots`, `writeRoots`, normalized tool grants,
  shell command grants, network mode, and explicit Claude Code SDK escape
  hatches.
- Pause has been removed from the v0.1 task/session model. Handoff is now
  specified as interrupt-then-lock, `/api/tasks/:id/pause` is gone, and locked
  sessions are the only "do not dispatch here" state.

## Explicitly out of scope (per product direction)

Corkboard is a local-only, single-user tool.
The "server" is just the I/O interface, runs at user privilege, and inherits the operator's existing trust boundary.
Security inside that boundary is the user's responsibility.
The following review items are therefore declined rather than deferred:
- Loopback auth / per-process tokens / origin checks.  Removed from `SPEC.md` §7.3; the API binds `127.0.0.1` and that is the contract.
- Transcript redaction boundaries (previously medium item 5). Transcripts may contain secrets pasted or printed locally; corkboard does not scrub them.  Adapter-supplied hooks remain the user's escape hatch.

## Remaining issues

1. High: task mutation needs concurrency rules.
   `SPEC.md:385` allows `PATCH /api/tasks/:id` to update status, prompt, and
   scope. That is too broad once the moderator and dashboard both write to the
   Board. Require `ifMatchEventId`/revision checks, and forbid prompt/scope
   changes after dispatch except by creating a replacement task.

2. Low: adapter/plugin naming should be unified.
   `SPEC.md:537` uses `corkboard-board-*`; `SPEC.md:627` uses
   `corkboard-agent-*`. That distinction is clear, but discovery should use
   one manifest shape so storage adapters, agent adapters, and hooks install
   the same way.
