# ADR-0006: Auto-link graph materialization on put_page

**Status:** Accepted
**Landed:** v0.12 (Knowledge Graph layer)

## Context

The hybrid retrieval stack (ADR-0005) leans on a real knowledge graph for
relational queries. The graph has to come from somewhere. Two options:

- **Lazy, query-time** — parse wikilinks every time someone walks the
  graph. Cheap to write, expensive to read, and the read cost compounds
  with brain size.
- **Eager, write-time** — every time a page is written, parse its
  wikilinks and materialize the edges into the `page_links` table. The
  read path is then a simple SQL traversal.

Eager materialization needs a hook on the write path that fires on every
put_page, every sync, every reverse-write from the dream cycle, without
the caller having to remember.

## Decision

The `put_page` operation handler (in `src/core/operations.ts`) calls the
auto-link post-hook after every successful write. The hook calls
`extractPageLinks` from `src/core/link-extraction.ts`, infers edge types
from anchor patterns (`attended`, `works_at`, `invested_in`, `founded`,
`advises`, `source`, `mentions`), and upserts into `page_links` and
`timeline_entries` via the v0.12.1 batch insert paths
(`addLinksBatch`, `addTimelineEntriesBatch`).

The hook is skipped only when `ctx.remote === true` AND
`ctx.trustedWorkspace` is unset — that is, when an untrusted MCP caller
writes to its sandboxed namespace. The autopilot cycle's `extract` phase
then runs as a sweeper to catch anything the hook missed (subagent
writes, bulk imports).

## Consequences

- `gbrain graph-query <slug>` is fast on any brain because the work is
  already done.
- Edge-type inference is a heuristic. Wrong types are caught by the
  `extract` phase's idempotent re-walk and by the contradictions probe.
- The cost: every put_page now does linker work. Batch inserts and
  `ON CONFLICT DO NOTHING` keep the per-write cost in microseconds.
- Migration v12 (v0.12.1) is a one-time backfill that runs the same
  extractor against every existing page so pre-graph brains catch up.

## Alternatives considered

- **Edge inference in a background job, not a write hook.** Considered.
  Rejected because then `gbrain graph-query` immediately after `put_page`
  returns stale results, breaking the "I wrote it, now I can query it"
  invariant agents rely on.
- **No edge-type inference, all edges generic.** Rejected because the
  graph then can't answer "what did Alice invest in" — only "who is
  Alice connected to."

## References

- `src/core/link-extraction.ts`
- `src/core/operations.ts` (auto-link post-hook on put_page)
- `src/commands/extract.ts` (sweeper)
- `src/commands/graph-query.ts`
