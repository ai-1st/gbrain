# ADR-0005: Hybrid retrieval — vector + keyword + RRF + graph

**Status:** Accepted
**Landed:** v0.10 (vector + keyword), v0.12 (graph layer)

## Context

Personal knowledge queries have four shapes that no single retrieval strategy
covers:

- **Thematic** ("companies in my portfolio") — vector embeddings catch
  semantic neighbors even when no exact word matches.
- **Literal** (a person's name, a code identifier, an exact phrase) — vector
  drifts; keyword (BM25) hits the literal token.
- **Relational** ("what did Alice invest in this quarter?") — neither vector
  nor keyword sees causal chains; the typed-edge graph does.
- **Mixed** — most real queries are blends.

Vector-only RAG, the industry default, performs poorly on the literal and
relational shapes. Benchmark numbers in `docs/architecture/RETRIEVAL.md`
show ripgrep BM25 and vector-only RAG at roughly the same P@5 on
BrainBench, both well below the hybrid stack.

## Decision

Every search runs three retrievals in parallel:

1. **Keyword (BM25)** via Postgres tsvector / `ts_rank`.
2. **Vector (HNSW)** via pgvector cosine distance.
3. **Graph traversal** when the query mentions an entity slug.

Results from keyword and vector are merged via **Reciprocal Rank Fusion**
(RRF): each ranking votes, scores are `1 / (k + rank)`, sum across
rankings. RRF is parameter-free relative to per-strategy scoring scales,
which matters because BM25 and cosine produce incomparable raw scores.

Then the fused list is re-ranked with source-aware boosts (ADR-0007) and,
optionally, an LLM reranker (v0.35+).

## Consequences

- Search hot path runs three queries instead of one; Postgres handles
  this in parallel.
- The graph layer requires keeping wikilinks materialized as typed edges,
  which is what ADR-0006 (auto-link) buys.
- Tuning is a knob: query expansion (multi-query LLM rewriting), reranker,
  token budget, source boosts, hard excludes. Knob-explosion was the
  motivation for ADR-0015 (search-mode bundles).

## Alternatives considered

- **Vector-only.** Rejected: literal and relational queries collapse.
- **Keyword + graph, no vector.** Rejected: thematic queries collapse.
- **Learned-to-rank meta-model.** Considered for future; RRF is good
  enough as a non-parametric default and avoids per-user training data.

## References

- `docs/architecture/RETRIEVAL.md`
- `src/core/search/hybrid.ts`
- `src/core/search/sql-ranking.ts`
- BrainBench (sibling repo `gbrain-evals`)
