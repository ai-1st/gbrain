# ADR-0008: Trust boundary via OperationContext.remote

**Status:** Accepted
**Landed:** v0.16 introduced; v0.26.9 made the field required (fail-closed)

## Context

The same operations module (ADR-0004) is reached by two very different
callers:

- The local CLI, typed by the user on their own machine. Full trust.
  Can submit shell jobs, dream-cycle phases, file uploads anywhere.
- The MCP server, called by an LLM agent over JSON-RPC. The agent may
  follow instructions embedded in untrusted content the user
  ingested (a prompt injection in an email, a poisoned article). Cannot
  be allowed to submit shell jobs or write outside its sandbox.

A naive design — separate handler paths per transport — diverges in
weeks. Putting trust in the handler signature makes it impossible to
forget.

## Decision

`OperationContext` carries a `remote: boolean` field. The CLI dispatch
in `src/cli.ts` sets `remote: false`. The MCP server in
`src/mcp/server.ts` sets `remote: true`. Subagents always set
`remote: true`.

Sensitive operations check it explicitly:

- `submit_job` rejects `name: 'shell'` or `'subagent'` when
  `remote !== false`.
- `file_upload` tightens path confinement when `remote === true`.
- `put_page` requires the slug to match the caller's
  `allowedSlugPrefixes` when `remote === true`.
- `auto_link` skips the post-hook when `remote === true` AND
  `trustedWorkspace` is unset.

As of v0.26.9 the field is **required** in the TypeScript type. The
compiler is the first defense against transports that forget to set it.
The check is `ctx.remote === false` (trusted-only sites) or
`ctx.remote !== false` (untrust unless explicit-false) — fail-closed
under cast bypass.

## Consequences

- An HTTP MCP request handler that forgets to set `remote: true`
  triggers a compile error, not a runtime escalation.
- The v0.26.9 hardening closed an OAuth-token-to-shell-RCE path that
  existed for several releases because the HTTP transport had an inlined
  context builder missing the field.
- Every new operation must decide its trust posture explicitly. A
  handler that ignores `ctx.remote` is auditable in `git grep`.

## Alternatives considered

- **Separate operation registries per transport.** Rejected: drift
  guaranteed, as shown by the v0.26.9 incident.
- **Trust at the HTTP layer (allow/deny ops by name).** Considered;
  retained for OAuth scope enforcement but not sufficient alone — the
  trust posture also affects parameter validation (path confinement,
  slug allowlists), which lives inside the handler.

## References

- `src/core/operations.ts:OperationContext`
- `src/mcp/dispatch.ts:buildOperationContext`
- `test/trust-boundary-contract.test.ts`
