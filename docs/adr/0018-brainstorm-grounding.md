# ADR-0018: Brainstorm grounded in user's brain, not training data

**Status:** Accepted
**Landed:** v0.37.0.0

## Context

Open Collider and related "bisociation" tooling generate ideas by
sampling across diverse domains and forcing the model to find
intersections. They sample from training data — the model's prior — so
the cross-domain pairs come from the public internet's center of mass.

GBrain users already have years of their own cross-domain knowledge in
the brain: essays on Y Combinator + readings on Italian Renaissance
patronage + meeting notes on AI labs + notes on Stripe's product. The
interesting bisociations are between *the user's* concepts, not between
concepts the user has never thought about.

A naive port (Open Collider's approach against gbrain) would lose this
advantage and produce ideas the user has already seen.

## Decision

The brainstorm "domain bank" is sampled from the user's own brain, not
from training data. Concretely:

- `SELECT DISTINCT substring(slug from '^[^/]+/[^/]+')` returns the
  user's prefix-stratified directory structure, cached 1h-TTL.
- Tiebreaker: `JOIN page_links` on connection count, so well-connected
  domains outweigh dead-letter ones.
- Fallback when fewer prefixes than M exist: corpus sampling.

Two commands ride this substrate:

- `gbrain brainstorm <question>` — defensible, cite-heavy. 4 close + 6
  far ideators. Judge threshold 4.0/5 (weighted originality / resistance
  / thesis_density / concrete_grounding / cognitive_load
  0.25/0.20/0.20/0.20/0.15). Saves by default.
- `gbrain lsd <question>` — Lateral Synaptic Drift. Inverted judge:
  rejects ideas with resistance > 4.5 ("too obvious"). 2 close + 12 far.
  Axiomatic inversions required. Stale-page bias via
  `pages.last_retrieved_at` (migration v79). Ephemeral by default.

The `last_retrieved_at` signal is updated by the op-layer search/query/
get_page handlers, fire-and-forget with 5-min throttling, opt-out via
`search.track_retrieval`. Internal callers (sync, migrations, dream
cycle) bypass the op layer so the LSD stale signal stays clean.

Crash-resilient checkpoint at `src/core/brainstorm/checkpoint.ts`
persists FULL idea bodies (~50KB per run) so resume MERGES pre-crash
and post-resume ideas before the judge runs — a resume that produces
only second-run output would be silent partial output.

## Consequences

- Generated ideas are recognizably the user's own concepts, recombined.
  Output quality is gated by what's already in the brain (a brain
  with 30 pages produces 30-page-shaped ideas).
- Three-axis eval gate (`gbrain eval brainstorm`): distance +
  usefulness + grounding, conjunctive. Distance alone is gameable.
- The cost: a fresh install with an empty brain can't usefully
  brainstorm; the LSD command stderr-warns about cold-start.

## Alternatives considered

- **Sample from training data (Open Collider's approach).** Rejected:
  loses the user's own cross-domain advantage.
- **Sample from a curated external corpus.** Rejected: violates the
  personal-brain promise; would need separate ingestion.

## References

- `src/core/brainstorm/{domain-bank,orchestrator,judges}.ts`
- `src/commands/brainstorm.ts`, `lsd.ts`, `eval-brainstorm.ts`
- `src/core/last-retrieved.ts`
- Open Collider: github.com/CL-ML/open-collider
