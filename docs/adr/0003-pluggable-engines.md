# ADR-0003: Pluggable storage engines via factory

**Status:** Accepted
**Landed:** v0.7 (PGLite engine), formalized through v0.12

## Context

GBrain needs to run in two very different deployment shapes:

- A new user on a laptop with no infrastructure who wants to try it in
  thirty seconds.
- A power user with 100K+ pages, multiple devices, and a need for
  background workers and HTTP access.

Picking one storage engine forces a choice between "works for everyone" and
"works well at scale." Postgres + pgvector is the right answer for the
second user; standing up Postgres is the wrong friction for the first.

## Decision

The `BrainEngine` interface in `src/core/engine.ts` defines ~40 methods.
Two concrete implementations satisfy it:

- `PostgresEngine` — Supabase or self-hosted Postgres + pgvector. Real
  background workers, HNSW indexes, RLS-capable.
- `PGLiteEngine` — Postgres 17.5 compiled to WASM, embedded. Zero
  infrastructure. Single-writer. pgvector included.

`gbrain init` defaults to PGLite and prompts to upgrade to Postgres when
the user has 1000+ files. `gbrain migrate --to {pglite,postgres}` moves
data bidirectionally.

The factory in `src/core/engine-factory.ts` dynamically imports the
configured engine so the binary doesn't pay for both.

## Consequences

- The same CLI and MCP surface work on both engines. Tests run hermetically
  against PGLite in-memory; E2E runs against real Postgres.
- Every engine method must be implemented on both sides. The
  engine-parity E2E tests (`test/e2e/engine-parity.test.ts`,
  `test/e2e/schema-drift.test.ts`) gate against drift.
- Schema migrations declare engine-specific SQL via `sqlFor.{postgres,pglite}`
  so concurrent-index DDL stays Postgres-only.
- The cost: a feature that needs Postgres-only behavior (parallel workers,
  RLS, advisory locks) must either degrade gracefully on PGLite or be
  guarded by `engine.kind === 'postgres'`.

## Alternatives considered

- **PGLite-only.** Rejected: caps the system at single-writer, no real
  workers, no Supabase integration.
- **Postgres-only with a Docker shim for zero-config.** Rejected: Docker
  is friction for the laptop user; the install-in-thirty-seconds promise
  fails.
- **SQLite + custom vector layer.** Rejected: pgvector and the SQL dialect
  parity between PGLite and Postgres are the whole reason this architecture
  is cheap.

## References

- `src/core/engine.ts` (interface)
- `src/core/engine-factory.ts`
- `src/core/postgres-engine.ts`, `src/core/pglite-engine.ts`
- `docs/ENGINES.md`
