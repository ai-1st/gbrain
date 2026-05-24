# ADR-0016: Calibration as first-class brain layer

**Status:** Accepted
**Landed:** v0.36.1.0

## Context

Every user is consistently wrong about specific things. Engineers
underestimate timelines. Founders are optimistic on B2C consumer
demand. Investors overweight pattern-match on past wins. These biases
don't change with one new piece of evidence; they're stable enough that
a brain ought to *know* them and use that knowledge.

Pre-v0.36, gbrain's brain was bias-blind. Two queries on the same
topic from the same user got the same advice surface regardless of
whether the user had previously been correct or wrong about it.

## Decision

Calibration becomes a first-class brain layer with its own schema, its
own cycle phases, and its own surfacing in every advice path.

Three cycle phases (extending the dream cycle):

- **propose_takes** — LLM scans markdown prose, proposes gradeable
  claims to `take_proposals`. Idempotency-keyed on
  `(source_id, page_slug, content_hash, prompt_version)`.
- **grade_takes** — walks unresolved takes older than 6 months, runs a
  judge model, caches verdicts. Auto-resolve disabled by default;
  conservative thresholds (single-judge ≥0.95 or 3/3 ensemble ≥0.85).
- **calibration_profile** — aggregates resolved takes into 2–4 narrative
  pattern statements + active bias tags. Cold-brain skip when <5
  resolved. Output voice-gated through `gateVoice()`.

Surfacing:

- The think prompt (`buildThinkUserMessage`) accepts a
  `withCalibration` option that injects a `<calibration>` block.
- Real-time nudges fire when a pattern matches the page slug + holder
  + confidence > 0.7, with a 14-day cooldown.
- Take forecasts (Brier-trend math, no LLM) print at write time.
- The admin SPA Calibration tab renders server-side SVGs of the
  Brier trend, domain bars, and pattern statements.

Cross-brain rules (D18, 4-rule contract): local-first; mount-fallback
only with `canReadMountsForCtx(ctx)`; attribution via `source_brain_id`
+ `from_mount`; subagents prohibited from reading cross-brain
calibration (closes the OAuth-to-cross-brain-leak surface).

## Consequences

- Advice surfaces compose calibration into their prompts; the agent
  can say "you've been optimistic on Series A timing in 4 of 5 past
  cases" because the brain has measured it.
- Voice-gated outputs prevent clinical/preachy rendering.
- Wave-versioned with `gbrain calibration --undo-wave` so the entire
  layer can be backed out cleanly.
- The cost: real LLM spend in grade_takes. The conservative thresholds
  + 6-month freshness gate keep it bounded.

## Alternatives considered

- **Ad-hoc heuristics ("if user says 'in 2 weeks', add 50%").**
  Rejected: not learnable, not user-specific.
- **External calibration service.** Rejected: violates the personal-
  brain promise.

## References

- `src/core/cycle/{propose-takes,grade-takes,calibration-profile}.ts`
- `src/core/calibration/`
- `skills/conventions/calibration.md`
- `docs/architecture/calibration-quality-gate-spec.md`
