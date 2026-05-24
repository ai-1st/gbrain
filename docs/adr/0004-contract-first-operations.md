# ADR-0004: Contract-first operations

**Status:** Accepted
**Landed:** v0.5 (foundational), grew to ~47 ops by v0.29

## Context

GBrain exposes the same brain through two transports: a local CLI a user
types, and an MCP server an agent calls over JSON-RPC. Hand-writing
parallel surfaces guarantees drift — the CLI grows a flag the MCP server
doesn't expose, the MCP op gets a new param the CLI ignores, security
gates fire in one path but not the other.

## Decision

`src/core/operations.ts` is the single source of truth. Each operation
declares its name, params (with JSON-schema-able `ParamDef` shape), scope
(`read | write | admin`), `localOnly` flag, and handler. The CLI dispatch
in `src/cli.ts` and the MCP server in `src/mcp/server.ts` are both
*generated* from this list at runtime.

The handler signature is `(ctx, params) => Promise<result>`. `ctx` carries
the engine, config, logger, and a `remote` flag that distinguishes trusted
local CLI callers from untrusted agent-facing callers (ADR-0008).

## Consequences

- Adding an op means editing one file. The CLI gets the flag, the MCP
  server gets the tool definition, the subagent allowlist sees it (if
  declared so), the HTTP scope check applies.
- The `paramDefToSchema()` helper in `src/mcp/tool-defs.ts` is the single
  ParamDef → JSON Schema mapper, consumed by stdio MCP, HTTP MCP
  `tools/list`, and the subagent tool registry. Three previously-divergent
  inline mappers collapsed to one in v0.35.3.
- Validation, scope enforcement, dry-run, and error envelope are uniform.
  `dispatchToolCall` in `src/mcp/dispatch.ts` is the shared entry.
- The cost: handlers must be pure functions of `(ctx, params)`. Hidden
  state (process-globals, module-level caches) breaks parity tests.

## Alternatives considered

- **Separate CLI and MCP codebases.** Standard practice. Rejected because
  every CHANGELOG would carry "remember to also expose in MCP."
- **OpenAPI-generated stubs.** Heavier toolchain than needed; ParamDef +
  TypeScript checks already give the same coverage.

## References

- `src/core/operations.ts`
- `src/mcp/dispatch.ts`, `src/mcp/tool-defs.ts`
- `test/parity.test.ts` (CLI-MCP contract parity)
