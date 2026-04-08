# ADR-002: Context Management Strategy

## Status
Accepted

## Date
2026-04-07

## Context

Context degradation is the #1 killer of LLM-powered systems in production. The research is unambiguous:

- **39% performance drop** in multi-turn conversations (Microsoft/Salesforce)
- **30%+ accuracy loss** for information in the middle of the context window (Lost in the Middle, TACL 2024)
- One company's costs went from **$2K to $47K/month** because conversations carried 150K tokens of history — 10-token responses cost $8 each
- Tool outputs are **100x larger** than user messages (Shopify). They are the primary context bloat vector
- Models don't gracefully degrade — they **fail convincingly**. The agent keeps running with incomplete info, producing confident wrong results. Context failures are invisible

The question isn't WHETHER to manage context — it's HOW.

### The Compaction Problem (Lived Experience)

Pure LLM compaction (Option B) is the most common approach. Anthropic offers it server-side. OpenAI has `/responses/compact`. It sounds clean: context gets too big, summarize it, keep going.

In practice, it feels like grasping at straws. The model works from fragments of what actually happened. It knows SOMETHING was discussed but can't access the original reasoning. It keeps trying to reconstruct context from summaries that are, by definition, lossy.

The research confirms this isn't just a feeling:

- **JetBrains (NeurIPS 2025)**: LLM summarization caused agents to run **13-15% longer** than observation masking. Summaries obscure stopping signals — the agent can't tell it's done because the compressed history doesn't give it enough signal
- **OpenAI's own docs**: "Even minor hallucinations in a summary can propagate forward" — one bad fact in a compaction poisons every turn after it
- **Summary generation API calls** comprised over **7% of total costs** in JetBrains' testing — and that's just the summary step, not counting the trajectory elongation

Compaction is a necessary escape valve. It should not be the first line of defense.

### The Static Threshold Problem

OpenCode's implementation hardcodes context thresholds: protect the last ~40,000 tokens of tool outputs, reserve ~20,000 tokens for compaction buffer. These are magic numbers that don't scale across models.

```
Claude Opus 4.6:    1,000,000 token window  →  40K protected = 4% of window
GPT-4.1:              128,000 token window  →  40K protected = 31% of window
Gemini 3 Pro:       1,000,000 token window  →  40K protected = 4% of window
Haiku 4.5:            200,000 token window  →  40K protected = 20% of window
Local 8K model:         8,000 token window  →  40K doesn't even fit
```

Same static number, completely different behavior per model. On a 1M window you're protecting almost nothing — 96% of recent tool outputs get pruned. On a small model you're trying to protect more than the entire window. This is lazy engineering. The thresholds must be proportional to the model's actual context window.

## Decision

Adopt a **three-tier context management strategy** modeled after OpenCode's architecture, but with **percentage-based thresholds** that scale with the model's context window instead of static token counts.

### Tier 1: Prune (Mechanical, Zero Cost)

**What it does**: Walk backwards through message history. Replace old tool outputs with `"[Old tool result content cleared]"`. Keep the tool CALL in history (preserves conversation structure), drop only the OUTPUT (the heavy payload).

**When it fires**: Context usage exceeds **prune threshold** (default: 75% of model's context window).

**What it protects**: The most recent tool outputs within the **protection window** (default: last 15% of context window). Everything older is eligible for pruning.

**What it never touches**:
- User messages (always preserved verbatim)
- Assistant messages and reasoning (always preserved)
- Tool calls (the request structure stays — you can see what was asked)
- Skill/special tool outputs (configurable exclusion list)

**Cost**: Zero. No LLM call. A loop over an array with string replacement. This is why it's Tier 1 — it's free and removes the largest source of bloat. Tool outputs are 100x larger than user messages (Shopify data). Pruning them is the highest-leverage move at zero cost.

**Why this works**: The model doesn't need the raw output of a file read from 30 turns ago. It needs to know that it READ the file and what it DECIDED based on it. The decision is in the assistant message. The raw content was just input. Dropping the input while keeping the decision preserves the conversation's reasoning chain.

### Tier 2: Compact (Semantic, Expensive)

**What it does**: Send the entire pruned conversation to a compaction agent. The agent produces a structured summary:

```
## Goal
[What the user is trying to accomplish]

## Instructions
[Important instructions, plans, specs still in play]

## Discoveries
[Notable things learned during the conversation]

## Accomplished
[What's done, what's in progress, what's left]

## Relevant Files
[Files read, edited, or created — with status]
```

**When it fires**: After pruning, if context usage still exceeds **compact threshold** (default: 85% of context window). Pruning already removed tool outputs — if we're still over, it means the actual conversation (human messages, reasoning, decisions) is what's filling the window. That's when you need semantic compression.

**What happens after compaction**:
- A compaction boundary is inserted into message history
- `filterCompacted()` makes everything before the boundary invisible to future turns
- The most recent user message is replayed after the summary (so the model doesn't lose what was just asked)
- Messages are NEVER deleted — they're hidden behind the compaction wall. Full history is preserved in storage

**Cost**: One full LLM call. Input = entire pruned conversation (potentially 100K+ tokens). Output = structured summary (1-3K tokens). This is expensive, but it only fires when pruning wasn't enough — which, given that tool outputs dominate context, should be relatively rare.

**Compaction model**: Configurable. Defaults to the same model as the conversation, but can be set to a cheaper model (e.g., Haiku at 60x cheaper than Opus). The summary is structured enough that a smaller model can handle it. This is a significant cost lever.

**Known risks** (from research, not speculation):
- Summary hallucinations propagate forward (OpenAI docs)
- Summaries obscure stopping signals, elongating agent trajectories 13-15% (JetBrains)
- Summary generation itself costs 7%+ of total API spend (JetBrains)
- If the model takes a wrong turn post-compaction, it can't recover by re-reading original context (Breunig: "When LLMs take a wrong turn, they do not recover")

### Tier 3: Hard Stop (Circuit Breaker, Zero Cost)

**What it fires**: If compaction itself overflows — the conversation is so massive that even summarizing it exceeds the context window. Or if cost/turn circuit breakers are hit.

**What it does**: Returns a `ContextOverflowError`. The agent stops. The user is informed.

**Why this exists**: GetOnStack had agents loop for **11 days undetected**, burning **$47K**. Infinite loops and runaway costs are production realities, not edge cases. A hard stop is non-negotiable.

**Circuit breakers** (configurable):
- Max consecutive tool calls without user input
- Max total cost per session
- Max turns per session
- Context overflow after compaction

### Threshold Configuration

All thresholds are **percentage-based**, scaling with the model's context window:

```typescript
interface ContextThresholds {
  // When to start pruning old tool outputs
  pruneAt: number        // default: 0.75 (75% of context window)
  
  // How much recent content to protect from pruning
  protectionWindow: number  // default: 0.15 (last 15% of window)
  
  // When to fire LLM compaction (after pruning)
  compactAt: number      // default: 0.85 (85% of window, post-prune)
  
  // When to hard stop
  hardStopAt: number     // default: 0.95 (95% of window)
  
  // Reserve for model output
  outputReserve: number  // default: 0.05 (5% reserved for response)
}
```

**What this looks like in practice:**

| Model | Window | Prune At | Protect | Compact At | Hard Stop |
|-------|--------|----------|---------|------------|-----------|
| Claude Opus 4.6 | 1,000K | 750K | 150K | 850K | 950K |
| GPT-4.1 | 128K | 96K | 19K | 109K | 122K |
| Haiku 4.5 | 200K | 150K | 30K | 170K | 190K |
| Local 8K | 8K | 6K | 1.2K | 6.8K | 7.6K |

Every model gets thresholds proportional to its actual capacity. No magic numbers.

**These percentages are starting points, not gospel.** The exact values are what evals are for — we'll measure when pruning/compaction fires, how it affects response quality, and tune accordingly. The architecture supports this tuning without code changes.

### The Flow

```
every turn:
  count tokens in current context
  
  if tokens > pruneAt:
    walk backwards through messages
    for each tool output older than protectionWindow:
      replace with "[Old tool result content cleared]"
    recount tokens
  
  if tokens > compactAt:
    send pruned history to compaction agent
    insert compaction boundary
    replay last user message after summary
    recount tokens
  
  if tokens > hardStopAt:
    throw ContextOverflowError
    stop agent
  
  proceed with normal LLM call

after every turn:
  check circuit breakers (cost, turns, consecutive tool calls)
```

## Options Considered

### Option A: Observation Masking Only (JetBrains)
- **Pros:** Cheapest. Won in 4/5 JetBrains test settings. 52% cheaper than summarization. Simple to implement. No LLM calls.
- **Cons:** Abrupt forgetting. No semantic compression — once a turn falls outside the window, its content is completely gone. Fine for coding agents with filesystem state, but a general agent might need to remember decisions from earlier in the conversation. No recovery path when the conversation itself (not tool outputs) fills the window.

### Option B: Pure LLM Compaction
- **Pros:** Semantic compression preserves meaning. Supported server-side by Anthropic and OpenAI. Conceptually simple.
- **Cons:** Burns a full LLM call EVERY time context exceeds threshold. Summaries feel like grasping at straws — the model works from fragments. Hallucinations in summaries propagate forward (OpenAI docs). Agents run 13-15% longer because summaries obscure stopping signals (JetBrains). Summary generation is 7%+ of total costs. No way to keep original context accessible — once compacted, the real conversation is gone from the model's view. Uses compaction as the first resort when pruning alone could have been sufficient.

### Option C: Tiered — Prune, Then Compact, Then Hard Stop (OpenCode-derived) — SELECTED
- **Pros:** Prune is free and removes the largest bloat vector (tool outputs). The model stays on REAL conversation history as long as possible. Compaction only fires when actual conversation content exceeds the window — which is less frequent. Hard stop prevents runaway costs. Matches OpenCode's proven architecture with the improvement of percentage-based thresholds.
- **Cons:** More complex than pure compaction (three code paths instead of one). Percentage-based thresholds need tuning per use case — wrong ratios could cause premature pruning or delayed compaction. Still inherits compaction's risks when Tier 2 fires.

### Option D: MemGPT / Context-as-RAM
- **Pros:** Most flexible. Model autonomously manages its own memory. Supports theoretically infinite conversations.
- **Cons:** Most complex to implement. Requires the model to make good decisions about what to store/retrieve — and models are "ok-to-terrible at information retrieval" (HN). Event-driven write-back adds latency to every turn. The model burns tool calls managing its own memory instead of doing useful work. Unproven at production scale for general agents. Premature complexity for our stage.

## Consequences

### What becomes easier
- **Cost control**: Tier 1 is free. Tier 2 fires less often than pure compaction. Circuit breakers prevent runaway costs.
- **Debugging**: Each tier is independent and observable. "Why did the model forget X?" has a clear answer: was it pruned (check protection window), compacted (check summary), or never in context?
- **Provider scaling**: Percentage-based thresholds automatically adapt to any model's context window. Add a new model with a 2M window — thresholds scale without code changes.
- **Quality**: The model works from real conversation history longer, not summaries. This should directly reduce the "grasping at straws" feeling.

### What becomes harder
- **Threshold tuning**: Five percentages to get right. Wrong values degrade quality in subtle ways. Need evals to find optimal ratios per model class.
- **Testing**: Three code paths means three sets of edge cases. Pruning logic, compaction logic, and the transition between them all need coverage.
- **Compaction quality**: When Tier 2 does fire, all the known risks apply — hallucination propagation, trajectory elongation, context loss. We're mitigating frequency, not eliminating the problem.

### What this ADR does NOT cover
- Specific compaction prompt wording (implementation detail, not architectural decision)
- Which model to use for compaction (configurable, defaults to conversation model)
- Message storage and persistence strategy (future ADR)
- How context management interacts with the agentic loop (future ADR)
- Eval methodology for tuning thresholds (future ADR)

## Relationship to ADR-001

The prompt layering strategy (ADR-001) determines WHAT goes into context. This ADR determines what happens WHEN context gets too full.

Layer 3 (tool/skill listing) is the heaviest static component (14-17K tokens). If the tool listing alone consumes a significant percentage of a small model's window, the prune threshold fires earlier, creating pressure. This means tool listing management (conditional loading from ADR-001) and context management (this ADR) work together — reducing tool tokens delays pruning, giving the actual conversation more room.

Layer 6 (per-message injections) adds tokens every turn. System reminders, plan mode instructions, and contextual instructions all contribute to context growth. These are small individually but compound over long conversations.

## References

### Primary
- OpenCode: `packages/opencode/src/session/compaction.ts` — prune and compact implementation
- OpenCode: `packages/opencode/src/session/overflow.ts` — token counting and overflow detection  
- OpenCode: `packages/opencode/src/session/message-v2.ts` — `filterCompacted()` history windowing
- OpenCode: hardcoded 40K protection, 20K compaction buffer — the static threshold problem we're solving

### Research
- JetBrains (NeurIPS 2025): Observation masking won 4/5 settings, 52% cheaper. Summarization caused 13-15% trajectory elongation. Summary API calls = 7%+ of total costs. Optimal masking window: 10 turns
- Lost in the Middle (TACL 2024): 30%+ accuracy loss in middle of context. GPT-3.5's middle accuracy fell BELOW closed-book baseline
- Chroma Context Rot: All 18 models degrade with length. Shuffled text paradoxically better than coherent for retrieval
- Breunig "How Long Contexts Fail": Context poisoning (hallucinations enter context and propagate), context distraction (>100K tokens → models repeat rather than synthesize), context confusion (>30 tools → failure)
- OpenAI docs: "Minor hallucinations in summaries propagate forward"
- Shopify: Tool outputs are 100x larger than user messages
- GetOnStack: Agent infinite loop ran 11 days, burned $47K — why circuit breakers are non-negotiable
- Microsoft/Salesforce: 39% performance drop in multi-turn conversations

### Cost Data
- Anthropic cache reads: 10% of base (90% savings) — pruning preserves cache structure
- Anthropic Haiku vs Opus: ~60x cost difference — viable compaction model option
- Context window management at scale: $2K → $47K/month from unmanaged 150K token conversations
