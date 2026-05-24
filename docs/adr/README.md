# Architecture Decision Records

These ADRs are reconstructed from CHANGELOG entries, design docs in `docs/designs/`
and `docs/architecture/`, ethos essays in `docs/ethos/`, and code paths cited
throughout `CLAUDE.md`. The originals lived in private plan files at
`~/.claude/plans/` per the project's policy of keeping review-process artifacts
out of the public repo; what survives here is the conclusion, not the deliberation.

Each ADR captures one load-bearing decision: the problem, the choice, the
trade-offs, the alternatives considered. They are written after the fact, so
"landed" cites the approximate release rather than the date the decision was made.

## Foundational

- [ADR-0001: Markdown as system of record](0001-markdown-as-system-of-record.md)
- [ADR-0002: Two-axis organization (brain × source)](0002-two-axis-brain-source.md)
- [ADR-0003: Pluggable storage engines via factory](0003-pluggable-engines.md)
- [ADR-0004: Contract-first operations](0004-contract-first-operations.md)

## Retrieval

- [ADR-0005: Hybrid retrieval — vector + keyword + RRF + graph](0005-hybrid-retrieval.md)
- [ADR-0006: Auto-link graph materialization on put_page](0006-auto-link-graph.md)
- [ADR-0007: Source-aware ranking](0007-source-aware-ranking.md)
- [ADR-0015: Named search-mode bundles](0015-search-mode-bundles.md)

## Agent integration

- [ADR-0008: Trust boundary via OperationContext.remote](0008-trust-boundary-remote.md)
- [ADR-0009: Thin harness, fat skills](0009-thin-harness-fat-skills.md)
- [ADR-0019: MCP as agent protocol, OAuth 2.1 for HTTP](0019-mcp-oauth.md)
- [ADR-0020: Skillpacks as scaffolding, not managed-block](0020-skillpacks-as-scaffolding.md)

## Orchestration

- [ADR-0010: Postgres-native job queue (Minions)](0010-postgres-native-queue.md)
- [ADR-0011: LLM subagents on top of Minions](0011-llm-subagents-on-minions.md)
- [ADR-0012: AI gateway as single LLM seam](0012-ai-gateway-seam.md)

## Quality and reasoning

- [ADR-0013: Op-layer eval capture, off by default](0013-eval-capture.md)
- [ADR-0014: Cross-modal multi-provider eval gate](0014-cross-modal-eval-gate.md)
- [ADR-0016: Calibration as first-class brain layer](0016-calibration-layer.md)
- [ADR-0017: Trajectory via typed claim columns](0017-trajectory-typed-claims.md)
- [ADR-0018: Brainstorm grounded in user's brain, not training data](0018-brainstorm-grounding.md)

## How these were chosen

Decisions made the cut when (a) reversing them would force a redesign of multiple
subsystems, (b) the trade-off was non-obvious enough that future contributors
might re-litigate it, or (c) a CHANGELOG entry called it out as a deliberate
choice between named alternatives. Micro-decisions (default flag values, env-var
names, retry counts) live in code comments and PR descriptions, not here.
