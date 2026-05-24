# ADR-0013: Op-layer eval capture, off by default

**Status:** Accepted
**Landed:** v0.25.0

## Context

Retrieval quality drifts. A change that improves P@5 on a synthetic
corpus may regress real user queries that the synthetic doesn't cover.
The only ground truth is the queries the user actually ran and the
results they kept (clicked, cited, edited).

Capturing real queries means writing them to a table the user can later
export. Two design pitfalls:

- If capture lives in every transport (CLI, MCP, HTTP), one of them
  drifts and a class of real queries is lost.
- If capture is on for every user by default, a quiet personal brain
  becomes a chatty privacy surface.

## Decision

Capture lives at the op layer — `src/core/eval-capture.ts` wraps the
`query` and `search` operation handlers. One call site catches MCP,
CLI, and subagent tool-bridge from the same place.

Capture is **off by default**. Resolution order:

1. Explicit `eval.capture` config flag (file plane, not DB).
2. `GBRAIN_CONTRIBUTOR_MODE=1` env var.
3. Off.

Contributors opt in with `export GBRAIN_CONTRIBUTOR_MODE=1` in their
shell profile. Production users get a quiet brain.

PII scrubbing (`src/core/eval-capture-scrub.ts`) is independent and
defaults to true regardless: emails, phones, SSN, Luhn-verified credit
cards, JWT-shaped tokens, bearer tokens get redacted before write.

`gbrain eval export` streams captured rows as stable NDJSON
(`schema_version: 1`) to stdout. `gbrain eval replay` re-runs the
captured queries against the current brain and prints Jaccard@k drift.

## Consequences

- Retrieval changes can be gated on real-user data before merging.
- Captured rows are git-friendly: NDJSON, scrubbed, schema-stable.
- The cost: every `query` and `search` op pays for a fire-and-forget
  write to `eval_candidates` when capture is enabled. Failures route
  to `eval_capture_failures` so doctor sees drops cross-process.
- Pre-v0.31 brains without the table fail open (informational
  warning in doctor).

## Alternatives considered

- **Transport-layer capture (every MCP transport, every CLI command).**
  Rejected — drift class proven in practice elsewhere.
- **Always-on capture.** Rejected: privacy posture wrong for personal
  brain.
- **External logging service.** Rejected: data leaves the user's
  machine, defeats the personal-brain promise.

## References

- `src/core/eval-capture.ts`
- `src/core/eval-capture-scrub.ts`
- `src/commands/eval-export.ts`, `eval-replay.ts`, `eval-prune.ts`
- `docs/eval-bench.md`, `docs/eval-capture.md`
