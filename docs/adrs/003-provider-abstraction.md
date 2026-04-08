# ADR-003: Provider Abstraction

## Status
Accepted

## Date
2026-04-07

## Context

Nocturnal is a multi-provider agent backend. ADR-001 commits us to per-provider prompt files — meaning we already treat models differently at the prompt layer. The question here is: what handles the plumbing BELOW the prompts? The API calls, streaming, tool calling, response parsing, and the dozens of provider-specific behaviors that differ across providers.

### The Provider Fragmentation Problem

Each provider implements the same concepts differently:

**Tool calling** — OpenAI returns tool arguments as a JSON **string** requiring `JSON.parse()`. Anthropic and Google return parsed **objects** directly. This means your tool execution code breaks if you swap providers without an adapter.

**Streaming** — Each provider has its own SSE event format. Anthropic sends `content_block_delta`, OpenAI sends `choices[0].delta.content`, Google has its own format. Building a streaming endpoint means writing provider-specific parsing for each.

**Reasoning/Thinking** — Anthropic has `thinking.type: "adaptive"` with an `effort` parameter and cryptographic signatures that must be passed back in multi-turn tool loops. OpenAI has `reasoning.effort` with encrypted reasoning state via `include: ["reasoning.encrypted_content"]`. Google has `thinkingLevel` with thought signatures that cause 400 errors if missing. Three completely different protocols for the same concept.

**Caching** — Anthropic uses `cache_control` breakpoints with a tools→system→messages hierarchy and lookback windows. Google uses explicit pre-created cache objects with TTLs. OpenAI uses `promptCacheKey`. Incompatible approaches to the same cost optimization.

**Temperature** — Gemini 3 MUST stay at 1.0 or performance degrades. Other providers treat it as a standard 0-2 parameter.

This isn't a theoretical concern. VentureBeat documented that each provider switch costs "days or weeks of development time." A team with 40+ OpenAI API call sites wanting to test Anthropic would need to rewrite every one.

### Why This Decision Matters for Nocturnal Specifically

We need multiple providers for different purposes: production-quality models (Claude, GPT) for the real workload, and free-tier providers (Groq, Google AI Studio, Mistral, OpenRouter, Cerebras, SambaNova) for testing and evals. The ability to switch between them without rewriting the agent loop is architecturally load-bearing.

But we also need provider-specific features. ADR-001 gives each provider its own prompt. ADR-002's compaction needs to interact with Anthropic's server-side compaction API and context editing features. We can't paper over everything — we need the abstraction to have escape hatches.

## Decision

Use **Vercel AI SDK** (`ai` package) as the primary provider abstraction, with **native SDKs as direct dependencies** for provider-specific features that AI SDK doesn't cover.

### Why AI SDK (The Engineering Arguments)

**1. Streaming is the core justification.**

Native SDK streaming requires ~20+ lines of boilerplate per endpoint: constructing `ReadableStream`, `TextEncoder`, async iteration, controller lifecycle, SSE headers. Every team reimplements this differently. AI SDK collapses it to:

```typescript
const result = await streamText({ model, messages, tools });
return result.toDataStreamResponse();
```

This isn't convenience — it's correctness. AI SDK's wire protocol uses typed SSE events (`text-delta`, `tool-input-start`, `tool-input-delta`, `tool-output-available`, `reasoning-start/delta/end`, `error`). Each event type has a defined schema. This structured protocol is what enables client-side hooks to render tool calls, reasoning, and text in real-time without custom parsing. Building this from scratch is where most teams introduce streaming bugs.

**2. Tool calling unification is genuinely hard to get right.**

```typescript
// One definition, works across all providers
tools: {
  weather: tool({
    inputSchema: z.object({ location: z.string() }),
    execute: async ({ location }) => ({ temperature: 72 }),
  }),
}
```

Zod schemas give you: TypeScript type inference on tool inputs/outputs, automatic validation, consistent behavior regardless of whether the provider returns strings (OpenAI) or objects (Anthropic/Google). In AI SDK 6, tool inputs stream by default (partial updates as the model generates), and `needsApproval` enables human-in-the-loop gating. Reimplementing this per provider is significant engineering.

**3. This is what OpenCode uses.**

Our prompt architecture (ADR-001) is modeled after OpenCode. OpenCode uses Vercel AI SDK for all LLM communication — `packages/opencode/src/session/llm.ts` calls AI SDK's streaming functions with provider-specific configuration. Using the same SDK means we can study their implementation directly. When we read their code, we're reading the same abstractions we're using.

**4. The TypeScript ecosystem has converged on it.**

Thomson Reuters (CoCounsel, 1,300 accounting firms) migrated their entire codebase to AI SDK, "deprecating thousands of lines across 10 providers." A team with 40+ OpenAI call sites migrated and "removed several hundred lines of manual error handling, JSON parsing, schema validation." The `ai` package has 20.8K GitHub stars.

This isn't popularity for popularity's sake. Convergence means: more bugs found and fixed, more edge cases handled, more provider updates tracked. When Anthropic ships a new API feature, the AI SDK team adds support — we don't have to.

**5. Bundle size and runtime compatibility.**

AI SDK: 19.5 kB gzipped. OpenAI SDK: 129.5 kB. The SDK was designed for V8 Edge Runtime from day one. LangChain JS is fundamentally incompatible with edge due to Node.js `fs` dependency.

### Why Not AI SDK Alone (The Escape Hatches)

AI SDK has real gaps. We will carry native SDKs as direct dependencies for:

| Feature | Why Native SDK | Provider |
|---------|---------------|----------|
| Batch API | 50% cost savings, not supported in AI SDK | Anthropic, OpenAI |
| Cache lifecycle | Can't create/manage caches through AI SDK | Google |
| Bleeding-edge features | AI SDK lags weeks-months behind native releases | All |
| Debugging | When the abstraction hides the failure, go direct | All |

The `providerOptions` escape hatch handles most provider-specific config (caching, thinking, compaction) — but features the SDK hasn't modeled at all (batch API, realtime voice) require native access.

### What We're Knowingly Accepting

**Known bugs in AI SDK:**
- Issue #11602: Anthropic thinking signatures lost during multi-step tool calls — this directly affects our agentic loop
- Issue #11685: PDFs downloaded unnecessarily instead of passing URLs
- Issue #13843: Non-image file parts silently dropped for OpenAI Responses API
- Zod `.url()` validator causes silent serialization failures through providers — use `.string()` and validate post-generation

**Abstraction costs:**
- ~200ms overhead through AI Gateway on small prompts (measured via benchmarks). Acceptable for most use cases, but worth knowing.
- Version compatibility breakage: `@ai-sdk/anthropic@3.x` was incompatible with `ai@5.x` at runtime. Pinning versions matters.
- "All-or-nothing" control: if we need fine-grained control beyond `streamText()`, we may duplicate logic. The agent loop in ADR-002 lives outside `streamText()` anyway — we control the loop, AI SDK handles the individual LLM call.

**The honest provider switching story:**
- Infrastructure switch is real — one line to change model
- AI behavior switch is NOT real — prompts don't transfer, output formats differ, tool calling reliability varies (Anthropic 8.4, Google 7.9, OpenAI 6.3 on Q1 2026 benchmarks)
- ADR-001 already handles this: per-provider prompt files exist precisely because models behave differently. We don't pretend they're interchangeable.

## Options Considered

### Option A: Direct Provider SDKs
- **Pros:** Maximum control. No abstraction overhead. First access to new features. "I know where all the state exists" (HN user dennisy). No version compatibility surprises from a third party.
- **Cons:** Every provider is custom code. Streaming boilerplate per provider (~20+ lines each). Tool calling differs (string vs object parsing). Every new provider = new integration. When Anthropic ships a new API, YOU maintain the adapter. The team with 40+ OpenAI call sites wanting to test Anthropic illustrates the pain — rewrite every site. We'd be solving a solved problem.

### Option B: Vercel AI SDK with Native SDK Escape Hatches — SELECTED
- **Pros:** Streaming solved correctly. Tool calling unified with type safety. Same abstraction OpenCode uses (direct code study). TypeScript ecosystem standard (Thomson Reuters, 20.8K stars). `providerOptions` for provider-specific features. Native SDKs available for gaps (batch API, cache management). Provider adapters handle API differences. 19.5 kB bundle. Edge-compatible.
- **Cons:** 200ms gateway overhead on small prompts. Known bugs (thinking signatures, PDF handling, file part dropping). Version compatibility risks. Lag time on new provider features. Can't control the abstraction — dependent on Vercel's team. Zod serialization edge cases.

### Option C: Custom Abstraction Layer
- **Pros:** Full control. Tailored to Nocturnal's exact needs. No third-party dependency.
- **Cons:** We'd be building AI SDK from scratch. Streaming, tool calling, provider adapters, type safety — all reimplemented. Every provider update is our maintenance burden. This is literally what AI SDK exists to prevent. Engineering time spent on plumbing instead of prompt architecture and context management (the actual problems we're here to solve).

### Option D: LiteLLM
- **Pros:** 100+ provider support. OpenAI-compatible proxy.
- **Cons:** Python-based. Adds a network hop (proxy architecture). Another service to deploy and maintain. Not TypeScript-native. Doesn't solve frontend streaming. Wrong ecosystem for a Bun/Hono/TypeScript backend.

## Architecture

```
┌─────────────────────────────────────────┐
│            Nocturnal Agent Loop          │
│         (ADR-002: context mgmt)         │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │     Prompt Assembly (ADR-001)     │  │
│  │  Provider prompt + env + tools    │  │
│  │  + instructions + agent + inject  │  │
│  └────────────┬──────────────────────┘  │
│               │                         │
│  ┌────────────▼──────────────────────┐  │
│  │      Vercel AI SDK (primary)      │  │
│  │  streamText() / generateText()    │  │
│  │  Tool execution, streaming,       │  │
│  │  providerOptions for specific     │  │
│  │  features (caching, thinking,     │  │
│  │  compaction)                      │  │
│  └────────────┬──────────────────────┘  │
│               │                         │
│  ┌────────────▼──────────────────────┐  │
│  │     Provider Adapters (AI SDK)    │  │
│  │  @ai-sdk/anthropic                │  │
│  │  @ai-sdk/openai                   │  │
│  │  @ai-sdk/google                   │  │
│  │  @ai-sdk/groq, mistral, etc.     │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │   Native SDKs (escape hatches)    │  │
│  │  @anthropic-ai/sdk  → batch API   │  │
│  │  openai             → batch API   │  │
│  │  @google/genai      → cache mgmt  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Dependency Strategy

```json
{
  "dependencies": {
    "ai": "^6.x",
    "@ai-sdk/anthropic": "^4.x",
    "@ai-sdk/openai": "^4.x",
    "@ai-sdk/google": "^4.x",
    "@ai-sdk/groq": "^1.x",
    "@ai-sdk/mistral": "^1.x",
    "@ai-sdk/openrouter": "^0.x",
    "@anthropic-ai/sdk": "^1.x",
    "openai": "^5.x"
  }
}
```

AI SDK providers for ALL models (production + testing). Native SDKs only for Anthropic and OpenAI — the providers where we need escape hatches (batch API, bleeding-edge features). Testing providers are consumed purely through AI SDK.

### Testing and Eval Providers

For evals and development, we'll use free-tier providers rather than burning production API credits. The AI SDK ecosystem covers these well:

| Provider | AI SDK Package | Use Case |
|----------|---------------|----------|
| Groq | `@ai-sdk/groq` | Fast inference, quick iteration |
| Google AI Studio | `@ai-sdk/google` | Gemini models, generous free tier |
| Mistral | `@ai-sdk/mistral` | European models, free tier |
| OpenRouter | `@ai-sdk/openrouter` | One API key, dozens of models — best for broad eval coverage |

OpenRouter is particularly valuable for evals — one integration gives access to free and cheap models across multiple families, letting us test how the prompt architecture holds across diverse models without managing separate API keys.

These providers don't need native SDK escape hatches. They're consumed purely through AI SDK. If a specific provider has gaps (e.g., AI SDK doesn't expose a feature), we add the native SDK as needed — not preemptively.

## Consequences

### What becomes easier
- **Streaming**: Solved correctly across all providers. We focus on what to stream, not how.
- **Tool calling**: One Zod schema, works everywhere. Type-safe inputs and outputs.
- **Provider switching for testing**: Free-tier models for evals, production models for quality — same agent loop code.
- **Code study**: Same abstractions as OpenCode. Their `llm.ts` reads directly into our understanding.
- **Maintenance**: Provider API updates handled by AI SDK team. We consume updates, not write adapters.

### What becomes harder
- **Debugging provider-specific issues**: The abstraction can hide failures. Thinking signatures lost silently (issue #11602). PDFs downloaded unnecessarily (issue #11685). When things break weirdly, we need to know when to bypass AI SDK and test with native SDKs directly.
- **Version management**: Pinning AI SDK, provider packages, AND native SDKs. Compatibility matrix gets complex.
- **New feature access**: When a provider ships something new, we wait for AI SDK support or drop to native SDK. Need to plan for this lag.

### Relationship to Other ADRs

**ADR-001 (Prompt Layering)**: The prompt assembly layer sits ABOVE AI SDK. We build the system prompt array, then hand it to `streamText()` / `generateText()`. AI SDK doesn't know about our layers — it just receives the assembled prompt.

**ADR-002 (Context Management)**: The agent loop controls the conversation flow — prune, compact, continue. AI SDK handles individual LLM calls within that loop. Anthropic's server-side compaction and context editing are accessible via `providerOptions.anthropic.contextManagement` — but our Tier 1 pruning happens BEFORE the AI SDK call (we modify the messages array ourselves).

## References

### Production Evidence
- Thomson Reuters CoCounsel: migrated 10 providers to AI SDK, "deprecating thousands of lines"
- Migration story: 40+ call sites, "removed several hundred lines of manual error handling"
- OpenCode: `packages/opencode/src/session/llm.ts` — AI SDK streamText/generateText usage

### Known Issues
- Issue #11602: Thinking signatures lost in multi-step tool calls (Anthropic)
- Issue #11685: PDFs downloaded unnecessarily (Anthropic)
- Issue #13843: File parts silently dropped (OpenAI)
- Zod `.url()` serialization failures — silent, only caught post-deployment

### Benchmarks and Data
- AI Gateway overhead: ~200ms on small prompts (dev.to benchmark)
- Tool calling reliability: Anthropic 8.4, Google 7.9, OpenAI 6.3 (Q1 2026)
- Bundle size: AI SDK 19.5 kB vs OpenAI SDK 129.5 kB
- AI SDK GitHub: 20.8K stars, V3 provider spec

### Research
- VentureBeat: Provider switch costs "days or weeks of development time"
- HN user dennisy: "Much happier building from scratch — know where all state exists"
- Ask HN: Concrete drawbacks of Vercel AI SDK (item 45465629)
- Strapi: OpenAI SDK vs AI SDK comparison
