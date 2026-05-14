# Separate "channels":
* Transcripts ~ actual back & forth with agents, rolling, not persisted
* Audit trail ~ events, state changes, and metadata that is valuable to persist for query and replay


# Event Types (kept small on purpose)
─────────────────────────────────────────────────
TASK_CREATED        task.id, source (cli|moderator|webhook), spec
TASK_LINKED         from, to, kind (depends-on|blocks|spawned-from)
TASK_STATUS         task.id, from, to, reason
TASK_CANCELLED      task.id, by

DISPATCH            task.id, worker.profile, session.id, prompt-hash,
                    git.sha, config-snapshot
RESULT              task.id, session.id, outcome (ok|fail|blocked),
                    files-touched[], tokens, duration, summary

PLAN_TICK           moderator.session.id, turn, decision, candidates[],
                    chosen, rationale-hash
REPLAN              moderator.session.id, reason, before-hash, after-hash

SESSION_OPEN        session.id, kind (worker|moderator), cwd
SESSION_FORK        from.id, to.id, by
SESSION_CLOSE       session.id, reason (done|error|teleport)

TELEPORT_LOCK       session.id, task.id, by (operator), at
TELEPORT_UNLOCK     session.id, returned-state, duration

HOOK_REJECT         worker.session.id, tool, args-hash, rule, reason
SCOPE_VIOLATION     worker.session.id, attempted-path, allowed-root

VERIFY              task.id, kind (lint|test|typecheck), outcome, by

COST                task.id, model, input-tokens, output-tokens, usd
ERROR               source, kind, payload, recoverable

That's roughly the whole vocabulary.
Resist adding more.
The discipline is: if an event doesn't enable a question someone will ask later, don't create it.


# Storage
- File-system by default: JSONL & MD
- Adapterable ~ community can add DB adapters on demand


# CLI
- why `init`, `up`, `down`, `status`, `teleport`, `fork`, `resume`?  These are UI features
- CLI is `npx corkboard --cwd . --storepath ./cb-store` etc.
