<<<<<<< HEAD
# Bebe

Bulletin-board style orchestration dashboard for AI agents.

Working name: **corkboard**. One Node process runs a moderator, a Fastify
service on `127.0.0.1`, and dispatches ephemeral workers through the bundled
Claude Code runtime adapter. v0.1 uses a minimal FS Board store.
The only UI is a browser dashboard (SvelteKit) served from the same process;
live updates come over SSE, mutations over REST. The CLI is intentionally
thin: `npx corkboard --cwd . --storepath ./cb-store`.

See [SPEC.md](./SPEC.md) for the full design.
=======
# `corkboard`
Bulletin-board style orchestration dashboard for AI agents
>>>>>>> 3c175463e54790084fa46348f6c34766f8b557d4
