# ADR-0007: Source-aware ranking

**Status:** Accepted
**Landed:** v0.22.0

## Context

The hybrid stack (ADR-0005) is content-blind. A 50-token chat log and a
5000-word essay are scored the same way: by lexical overlap and embedding
similarity. In practice the chat log wins more often than it should,
because the same multi-word phrase repeats across thousands of throwaway
conversations.

The pre-v0.22 behavior, observed on real brains: queries like "what is
the article-outline pattern" surfaced chat fragments mentioning the
phrase, not the actual article. This is the "source swamp" problem.

A naive fix — boost certain directories with hardcoded weights — is
brittle and locks user filing decisions into search behavior.

## Decision

Apply a per-prefix multiplicative boost at the SQL layer, with a
two-stage CTE so the boost survives HNSW vector indexes:

- Curated content (`originals/`, `concepts/`, `writing/`,
  `people/`, `companies/`, `deals/`) gets `1.2`–`1.5`.
- Bulk content (`daily/`, `media/x/`, `wintermute/chat/`) gets
  `0.5`–`0.8`.
- Hard-excluded prefixes (`test/`, `archive/`, `attachments/`,
  `.raw/`) drop out of results entirely via `NOT (col LIKE ...)`.

`DEFAULT_SOURCE_BOOSTS` and `DEFAULT_HARD_EXCLUDES` ship as defaults.
Env-var (`GBRAIN_SOURCE_BOOST`, `GBRAIN_SEARCH_EXCLUDE`) and per-call
`SearchOpts` override them. `detail !== 'high'` opt-out preserves the
temporal-query workflow where the user *wants* to see chat fragments.

`searchVector` switched to a two-stage CTE: inner CTE keeps
`ORDER BY embedding <=> vec` so HNSW stays usable, outer re-ranks by
`raw_score * source_factor`.

## Consequences

- The article wins over the chat-log swamp; the BrainBench E2E test
  `search-swamp.test.ts` pins the behavior.
- Boosts are SQL-layer, not post-rank, so pagination is consistent.
- LIKE-pattern escaping (covers `%`, `_`, AND `\`) prevents injection
  through user-supplied prefixes.
- The cost: a user with non-standard filing conventions sees the wrong
  shape of results until they override the defaults. The defaults are
  documented; the override is one env var.

## Alternatives considered

- **Post-rank in TypeScript.** Rejected: breaks pagination and forces
  full result load before slicing.
- **Per-page quality score column.** Considered for a future ML
  reranker; the prefix heuristic is good enough and explainable.

## References

- `src/core/search/source-boost.ts`
- `src/core/search/sql-ranking.ts`
- `test/e2e/search-swamp.test.ts`
