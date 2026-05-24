# ADR-0012: AI gateway as single LLM seam

**Status:** Accepted
**Landed:** v0.31.12

## Context

GBrain calls LLM providers from many places: chat (think, brainstorm,
synthesize, judge), embeddings (import, search expansion), multimodal
embeddings (image ingestion), and reranking. Each call needs the same
plumbing — config-and-env key resolution, model alias expansion,
recipe-shape validation, dimension passthrough for flexible-dim models,
prompt-cache headers, error classification, budget enforcement.

Pre-v0.31, every call site instantiated `new Anthropic()` or
`new OpenAI()` directly and re-implemented some subset of the plumbing.
This is how v0.31.6 shipped with a phantom `claude-sonnet-4-6-20250929`
chat default that 404'd silently on every install: nobody owned the
canonical model registry.

## Decision

`src/core/ai/gateway.ts` is the single seam. Every LLM call in gbrain
walks through one of:

- `gateway.chat(params)`
- `gateway.embed(texts)`
- `gateway.embedQuery(text)`
- `gateway.embedMultimodal(inputs)`
- `gateway.rerank(query, docs)`

The gateway owns:

- Provider recipes (Anthropic, OpenAI, Voyage, ZeroEntropy,
  OpenRouter, Google, Gemini multimodal via proxy).
- The model-tier resolver (`resolveModel`) and the 8-step alias chain.
- Per-recipe knobs (`chars_per_token`, `safety_factor`,
  `max_batch_tokens`, dimension validation).
- Provider-shim translation (Voyage `dimensions → output_dimension`,
  ZeroEntropy `/embeddings → /models/embed` wire rewrite).
- Budget tracking via `withBudgetTracker` + `AsyncLocalStorage`.
- Test seams (`__setEmbedTransportForTests`,
  `__setChatTransportForTests`) so tests stub without `mock.module`.

`gbrain models doctor` probes the gateway with 1-token reachability
calls per configured model — the structural fix for the silent
no-op bug class.

## Consequences

- Adding a provider means one recipe file. Adding a feature (prompt
  caching, structured outputs, JSON mode) means one gateway change.
- Per-call config + auth comes from `~/.gbrain/config.json` AND env;
  closed the v0.30 bug class where stdio MCP launches lost the API key
  because the Anthropic SDK only read env.
- The gateway is the cost choke point. A budget cap installed around a
  cycle phase enforces against every embed, every chat, every rerank
  the phase makes.
- The cost: one file gets large. The recipe table is the carrying
  cost of the abstraction.

## Alternatives considered

- **LiteLLM or LangChain LLM wrappers.** Considered. Rejected because
  gbrain's correctness depends on per-recipe knobs the upstream
  abstractions don't surface (Voyage dimension passthrough, the
  ZeroEntropy wire rewrite, AnthropicCacheControlHeader).
- **One adapter file per provider, no central gateway.** What gbrain
  had pre-v0.31. The v0.31.6 model-id incident is exactly what
  happens when nobody owns the canonical registry.

## References

- `src/core/ai/gateway.ts`
- `src/core/ai/recipes/`
- `src/core/model-config.ts`
- `src/commands/models.ts` (`gbrain models doctor`)
