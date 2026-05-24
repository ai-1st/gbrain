# ADR-0019: MCP as agent protocol, OAuth 2.1 for HTTP

**Status:** Accepted
**Landed:** v0.16 (stdio MCP), v0.26.0 (HTTP MCP + OAuth 2.1)

## Context

Agents need a protocol to call gbrain. Three options were live in
early 2024–2026:

- A bespoke gbrain protocol (JSON over stdin/stdout). Whatever shape we
  invent, every new agent has to support.
- gRPC. Heavy toolchain, schema generation, not what the ecosystem
  converged on.
- **Model Context Protocol (MCP).** Anthropic's spec; Claude Desktop,
  Cursor, every recent agent framework speaks it. Two transports: stdio
  (process-pair) and HTTP (network).

HTTP MCP needs auth. Static bearer tokens work for personal use but
break when the user wants to grant a hosted agent (ChatGPT, Perplexity)
read-only access to their brain. The right shape is real OAuth — but
OAuth 2.0 is famously footgun-laden, and OAuth 2.1 closes most of those
footguns.

## Decision

GBrain speaks MCP, period. Both transports:

- **stdio** — `gbrain serve` for Claude Desktop, Cursor, OpenClaw.
- **HTTP** — `gbrain serve --http` for ChatGPT, Perplexity, hosted
  agents, the admin SPA.

HTTP MCP carries full OAuth 2.1: PKCE-enforced authorization-code grant
for public clients (ChatGPT, Cursor), `client_credentials` grant for
service clients (Perplexity), refresh-token rotation with stolen-token
detection, atomic `DELETE ... RETURNING` on every single-use token to
close TOCTOU races (RFC 6749 §10.4 and §10.5).

Tokens are SHA-256 hashed before storage. Client secrets are
one-time-reveal. The admin dashboard at `/admin` (React 19 SPA served
out of `admin/dist/`) registers clients, mints tokens, watches a live
SSE activity feed, and revokes anything that looks wrong.

The v0.26.9 hardening pass closed every cross-listing of the spec it
could find: F1+F2 fold `client_id` atomically into delete-returning
clauses, F3 enforces refresh-scope-subset against the original grant
(not the client's current allowed scopes), F4 binds `client_id` on
revoke, F5 swaps bare `catch {}` for the explicit
`isUndefinedColumnError` predicate, F7c validates `redirect_uri`
against the value stored at `/authorize`.

Source scoping (ADR-0002) extends OAuth: each client carries a
`source_id` (write authority) and `federated_read` (TEXT[] read scope).
A client bound to `dept-x` cannot see `dept-y` rows.

## Consequences

- Adding agent support is "configure an OAuth client" — no per-agent
  bespoke code.
- Legacy bearer tokens (`access_tokens` table) grandfather to
  `read+write+admin` on the OAuth server so pre-v0.26 clients keep
  working without migration.
- The HTTP server has to ship Express 5 + the MCP SDK + an SPA. That's
  the carrying cost.
- v0.34.1 added `--bind 127.0.0.1` default so personal-laptop installs
  don't accidentally publish to the LAN.

## Alternatives considered

- **Bespoke protocol.** Rejected: ecosystem convergence on MCP.
- **HTTP MCP with bearer tokens only.** Rejected: no scope, no
  rotation, no per-client revoke. Bad fit for hosted agents.

## References

- `src/mcp/server.ts`, `src/mcp/dispatch.ts`
- `src/commands/serve.ts`, `serve-http.ts`
- `src/core/oauth-provider.ts`
- `admin/` (React 19 SPA)
- `docs/mcp/`
