# ADR-0014: Cross-modal multi-provider eval gate

**Status:** Accepted
**Landed:** v0.27.x

## Context

Some gbrain outputs cannot be graded by a deterministic test:

- A morning pulse summary.
- A think-tank brainstorm.
- A meeting brief.
- A skill's response to an ambiguous user message.

These are open-ended generations. "Did the test pass" is the wrong
question; "is this output good enough" is the right one. Self-grading
(same model that produced the output also evaluates it) is well known
to be miscalibrated — the model's biases are highly correlated with
its own outputs.

## Decision

Cross-modal eval: three frontier models from three different providers
score the same OUTPUT against the same TASK on a 5-dimension rubric.
Verdict is `pass | fail | inconclusive`:

- **Pass** requires every dimension mean ≥ 7 AND every dimension's
  minimum across models ≥ 5.
- **Inconclusive** when fewer than 2/3 models returned parseable scores
  (so a misbehaving judge never silently flips to pass on an empty
  array).
- **Fail** otherwise.

Default slots: `openai:gpt-4o` / `anthropic:claude-opus-4-7` /
`google:gemini-1.5-pro`. The gate uses the AI gateway (ADR-0012), so
config/auth/aliasing comes from the recipe registry.

Up to 3 cycles per task; stops early on PASS or INCONCLUSIVE. Receipts
land at `~/.gbrain/eval-receipts/<slug>-<sha8>.json` and are bound to
the SKILL.md sha-8 so a receipt for a stale skill version is detectable.

Batch mode (`gbrain eval cross-modal --batch <jsonl>`) fans out across
a LongMemEval-shape file with a `--max-usd` cap.

## Consequences

- Open-ended outputs gain a regression gate that doesn't rely on a
  single model's judgment.
- The cost is real: 3 frontier-model calls per cycle. Default
  `--cycles 1` in non-TTY contexts caps accidental scripted spend.
- Provider diversity matters more than model frontier-ness. Three Opus
  instances would be useless; one OpenAI + one Anthropic + one Google
  is the load-bearing property.

## Alternatives considered

- **Single-judge ensemble (same model, N samples).** Rejected: same
  miscalibration cluster.
- **Human-in-the-loop grading only.** Rejected: doesn't scale; gbrain
  ships features daily.
- **Rubric-on-rules (BLEU/ROUGE-shaped).** Rejected: open-ended outputs
  don't have golden references.

## References

- `src/commands/eval-cross-modal.ts`
- `src/core/cross-modal-eval/{runner,aggregate,judges,json-repair}.ts`
- `skills/conventions/cross-modal.md`
- `docs/eval-cross-modal.md` (if present)
