# ADR-0017: Trajectory via typed claim columns

**Status:** Accepted
**Landed:** v0.35.7 (typed columns), v0.40.2.0 (event-typed)

## Context

A real personal brain accumulates the same entity's facts over time:
widget-co's MRR was $30K in Jan, $45K in Mar, $38K in Jun. Free-text
takes capture each datum, but reasoning over the trajectory ("MRR is
declining") needs structure: a metric label, a value, a unit, a
timestamp.

Pre-v0.35.7, gbrain stored these as unstructured fact rows. "Show me
widget-co's MRR over time" required NLU at query time, with no
machine-checkable supersession.

## Decision

Migration v67 added four nullable typed columns to the `facts` table:
`claim_metric`, `claim_value`, `claim_unit`, `claim_period`, plus a
partial index on `(entity_slug, claim_metric, valid_from)
WHERE claim_metric IS NOT NULL`. The fact-extraction prompt was
extended to populate them when the markdown carries a typed
("MRR: $45K") shape; free-text claims continue to write the legacy
columns only.

The fence widens from 10 to 14 cells when any row has typed data; the
renderer stays at 10 cells when none do (no churn diff on existing
fences).

`BrainEngine.findTrajectory(opts)` returns chronological typed-claim
history. Pure-function math in `src/core/trajectory.ts`:

- `detectRegressions(points, threshold)` flags consecutive-pair
  drops > threshold (default 10%, env-overridable).
- `computeDriftScore(points)` returns
  `1 - mean(cosine(emb[i], emb[i-1]))` over existing embeddings, NULL
  with <3 embedded points.

v0.40.2.0 added migration v89 with a nullable `event_type` column so
the same substrate carries event-shaped rows (`meeting`, `job_change`,
`location_change`). `TrajectoryOpts.kind?: 'metric' | 'event' | 'all'`
filters at the engine layer.

`gbrain think` consumes the trajectory: temporal and knowledge_update
intents route entity candidates through `findTrajectory`, splice a
`<trajectory>` block into the answer-gen prompt, and let the agent
ground its answer in real chronology.

## Consequences

- "Show me widget-co's MRR" is a single SQL query, not an LLM extraction
  pass.
- Founder scorecard and contradiction probe both consume the same
  typed columns, so they can't drift.
- Back-compat preserved through nullable columns; pre-typed brains keep
  working until they upgrade.
- The cost: extraction prompts grew to populate the fields, fact rows
  are wider. Both negligible.

## Alternatives considered

- **Separate `claims` table.** Rejected: every fact already lives in
  `facts`; a parallel table would split the source of truth.
- **JSON column for typed payload.** Rejected: indexable columns and
  JSON columns degrade differently under Postgres planners; the
  partial index needs real columns.

## References

- `src/core/trajectory.ts`, `trajectory-format.ts`
- `src/commands/eval-trajectory.ts`, `founder-scorecard.ts`
- Migration v67 (typed-claim columns), v89 (event_type)
- `src/core/think/index.ts` (trajectory injection)
