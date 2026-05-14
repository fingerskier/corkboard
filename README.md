# Bebe

Bulletin-board style orchestration dashboard for AI agents.

Working name: **corkboard**. One Node process runs a moderator (Claude
Agent SDK loop), a Fastify service on `127.0.0.1`, and dispatches
ephemeral `claude -p` workers. The only UI is a browser dashboard
(SvelteKit) served from the same process; live updates come over SSE,
mutations over REST. The CLI is intentionally thin — `init`, `up`,
`down`, `status`.

See [SPEC.md](./SPEC.md) for the full design.
