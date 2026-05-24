# ADR-0002: Two-axis organization — brain × source

**Status:** Accepted
**Landed:** v0.18 (sources), v0.19 (brain mounts)

## Context

A single user accumulates knowledge in concentric scopes:

- Their personal notes (always present).
- Subject-specific corpora they want to keep navigable but separate (a wiki
  fork, a gstack code brain, an essays repo).
- Team or organization brains they want to read from but not own
  (CEO-class deployments need to consult multiple team brains).

A single-DB design forces everything to share one keyspace. A many-DBs design
forces every read to be cross-DB-aware. Neither alone matches the actual
mental model.

## Decision

Two orthogonal axes:

- **Brain** = one database (PGLite file, Postgres, or Supabase). Picked by
  `--brain`, `GBRAIN_BRAIN_ID`, or `.gbrain-mount` dotfile. Your default is
  the `host` brain; additional brains register via `gbrain mounts add`.
- **Source** = one named repo of pages inside a brain. Picked by
  `--source`, `GBRAIN_SOURCE`, or `.gbrain-source` dotfile. A brain holds
  many sources; slugs scope per source so `people/alice` in `wiki` and
  `people/alice` in `gstack` do not collide.

Both axes resolve through the same 6-tier precedence chain so the routing
rules are learnable once and apply everywhere.

## Consequences

- Read paths thread `sourceId` (scalar) or `sourceIds` (federated array)
  through every engine method that touches `pages`, `content_chunks`,
  `facts`, or the graph. The `sourceScopeOpts(ctx)` helper centralizes the
  precedence.
- OAuth clients carry `source_id` (write authority) and `federated_read`
  (read scope) so a team-published brain can be served with per-client
  visibility.
- Migration v60-v65 (PR #861, #876) backfills `source_id='default'` for
  pre-v0.34 deployments so the contract is uniform.
- The cost: every new operation must decide its source-scoping behavior
  explicitly. A handler that forgets `sourceScopeOpts(ctx)` silently
  collapses to the global view.

## Alternatives considered

- **Single namespace with slug prefixing.** Considered but rejected because
  prefixes leak into URLs, frontmatter, and user mental models.
- **One DB per source.** Rejected because cross-source graph queries
  ("what does my wiki know about a person in my gstack brain?") then need
  cross-DB joins, defeating the point.

## References

- `docs/architecture/brains-and-sources.md`
- `docs/architecture/topologies.md`
- `skills/conventions/brain-routing.md`
- `src/core/source-resolver.ts:resolveSourceWithTier()`
- `src/core/operations.ts:sourceScopeOpts()`
