# ADR-0010: Postgres-native job queue (Minions)

**Status:** Accepted
**Landed:** v0.11 (Minions adoption)

## Context

Background work in gbrain has three shapes: deterministic shell jobs
(sync, embed, extract), LLM subagent loops (synthesize, brainstorm,
agent run), and recurring autopilot phases (the dream cycle). All three
need durable state, retry on failure, parent-child fan-out with fan-in,
per-job timeouts, idempotency keys, and cancellation.

The industry default is BullMQ (Redis) or a SaaS queue. Either choice
adds a service dependency and a separate persistence model.

GBrain already has Postgres for the brain data. The question is whether
the queue should share that database or run alongside on Redis.

## Decision

Implement the queue (Minions) on Postgres directly. The schema
(`minion_jobs`, `minion_queues`, `minion_locks`, `subagent_messages`,
`subagent_tool_executions`, `subagent_rate_leases`) lives in the same
database as the brain content. `FOR UPDATE SKIP LOCKED` drives the
claim semantics. Stall detection uses TTL'd lock rows.

The API surface (`MinionQueue`, `MinionWorker`, `MinionSupervisor`) is
BullMQ-inspired but contract-first: handlers live in
`src/core/minions/handlers/`, registered at worker startup. Schema
migrations evolve the queue tables alongside the brain tables.

PGLite gets the same schema but no real worker; PGLite callers use the
`--follow` inline path which executes the handler in-process.

## Consequences

- One database to back up, one to migrate, one to monitor.
- Postgres advisory locks coordinate cross-process supervisors;
  `pg_try_advisory_xact_lock` handles concurrent submit dedup.
- The supervisor watchdog reconnects on three consecutive health
  failures via `engine.reconnect()`; a saturated pool no longer
  cascades into worker death.
- The cost: Postgres-native queues are not Redis-fast. Throughput tops
  out where the DB's row-locking does. For gbrain's workload
  (single-user personal brain, ~thousands of jobs/day, not millions),
  this is irrelevant.

## Alternatives considered

- **BullMQ on Redis.** Rejected: adds Redis to the stack and splits
  persistence. Loss of the queue would mean unfinished work; with the
  Postgres queue, brain backup IS queue backup.
- **In-process work queue, no durability.** Rejected: subagent loops
  routinely run for hours and must survive worker restarts.

## References

- `src/core/minions/queue.ts`, `worker.ts`, `supervisor.ts`
- `src/core/minions/types.ts`
- `docs/designs/MINIONS_AGENT_ORCHESTRATION.md`
- `skills/minion-orchestrator/SKILL.md`
