# ADR-0015: Named search-mode bundles

**Status:** Accepted
**Landed:** v0.32.3

## Context

The hybrid retrieval stack (ADR-0005) exposes a knob explosion: cache
on/off + cache similarity threshold + cache TTL + intent weighting +
token budget + LLM expansion + searchLimit. Pre-v0.32 these were
per-key config entries with no documented combination guidance, so the
field of "good search settings" was a path-dependent walk through ten
flags.

Worse, the *agent cost* of a search call is dominated by the
downstream model's context-token spend on the returned payload, not by
gbrain itself. The right knob settings for "Haiku 4.5 downstream" differ
by 5x from "Opus 4.7 downstream." Per-knob config invites mismatched
pairings (tokenmax+Haiku wastes capacity; conservative+Opus starves it).

## Decision

Three named modes bundle the knobs into one config key
(`search.mode`):

- **conservative** — 4K-token budget, no LLM expansion, top 10 results.
  Pairs with Haiku-class downstreams (~$40/mo at 10K queries/month).
- **balanced** — 12K budget, no expansion, top 25. Default. Pairs with
  Sonnet-class (~$300/mo at 10K).
- **tokenmax** — no budget, LLM expansion ON, top 50. Pairs with
  Opus-class (~$1000/mo at 10K).

Resolution chain: per-call `SearchOpts` → per-key override → mode
bundle → balanced fallback.

Mode resolution lives in bare `hybridSearch`, not just the cached
wrapper, so eval replay and the LongMemEval harness measure the same
mode-affected behavior as the production `query` op.

Cache-key contamination defense: the `knobs_hash` column (migration
v56) folds mode into the cache key so a tokenmax write can't be served
to a conservative read.

`gbrain search modes` shows what's active. `gbrain search tune` reads
captured eval data (ADR-0013) and recommends a mode based on the
user's actual query mix.

## Consequences

- Install picker in `gbrain init` collects intended downstream model
  and picks the matching mode. Mismatched pairings are still
  configurable but no longer the default.
- Cost is legible: the matrix in `CLAUDE.md` and the install picker
  show $/month at typical volume per pairing.
- Future modes (e.g. `frontier` once 1M-context becomes cheap) are
  additive without breaking existing configs.

## Alternatives considered

- **Auto-tune per query.** Considered for a future revision; needs
  more captured-eval data than gbrain has shipped against.
- **Keep per-knob, document better.** Rejected: documentation never
  catches up to flag drift, as proven by the pre-v0.32 state.

## References

- `src/core/search/mode.ts`
- `src/commands/search-modes.ts`, `search-stats.ts`, `search-tune.ts`
- `docs/eval/SEARCH_MODE_METHODOLOGY.md`
- Cost matrix in `CLAUDE.md` (the "Search Mode" section)
