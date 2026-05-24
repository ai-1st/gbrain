# ADR-0011: LLM subagents on top of Minions

**Status:** Accepted
**Landed:** v0.15

## Context

Several gbrain features need to run multi-turn LLM tool-use loops that
take minutes or hours: the dream cycle's synthesize phase processes
each meeting transcript with a fresh subagent, brainstorm runs N close +
M far ideators in parallel, agent run is the user-facing entrypoint to
long-running tasks.

A naive design (spawn a child process per loop, pipe stdout to the user)
breaks on every restart. Multi-turn LLM state is too expensive to
re-derive — context tokens compound, prompt caching has a 5-minute TTL,
sub-task fan-out needs durable join points.

## Decision

Subagents are a job *kind* in the Minions queue (ADR-0010). The
`subagent` handler reads `subagent_messages` (the durable conversation
log), calls the LLM, writes the assistant turn + any tool calls, then
either continues or returns. Tool calls write through a two-phase
mechanism: `subagent_tool_executions` row is created `pending`, the tool
runs, the row is updated `complete` or `failed`. Mid-dispatch crash =
replay reconciliation picks up where it left off.

Parent-child topologies use the queue's existing fan-out + `child_done`
inbox; the `subagent_aggregator` handler claims only after every child
posts its terminal outcome.

Rate-leases (`subagent_rate_leases` table) cap outbound Anthropic
concurrency per `(provider, model)` tuple, surviving cross-process via
`pg_advisory_xact_lock`.

## Consequences

- Subagent jobs survive worker restart, host reboot, and Anthropic
  rate limits without losing the loop.
- The same `gbrain jobs` CLI manages shell jobs, syncs, and subagents
  uniformly. One mental model.
- Cost-bounding is enforced via the budget tracker (`BudgetTracker`
  primitive) installed via `AsyncLocalStorage` around the subagent's
  gateway calls.
- The cost: the schema carries first-class LLM state. Adding a new
  provider means deciding whether its message shape fits the
  Anthropic-shaped log; v0.31.12 hardcoded the subagent loop to
  Anthropic-direct because the tool-use grammar is Anthropic-specific.

## Alternatives considered

- **Subagent state in process memory + checkpoint to disk.** Rejected:
  crash recovery is brittle, and the queue's join semantics
  (`child_done` inbox) are already exactly what fan-in needs.
- **Separate subagent service.** Rejected: doubles the persistence
  layer and the supervisor's responsibilities for no benefit at
  gbrain's scale.

## References

- `src/core/minions/handlers/subagent.ts`
- `src/core/minions/handlers/subagent-aggregator.ts`
- `src/core/minions/rate-leases.ts`
- `src/commands/agent.ts`, `src/commands/agent-logs.ts`
