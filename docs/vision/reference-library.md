# Nocturnal Reference Library

## Date
2026-04-07

## Purpose
Consolidated research backing every architectural decision. This is the evidence base for ADRs.

---

# PART 1: CONTEXT DEGRADATION RESEARCH

## Lost in the Middle (Liu et al., 2024 TACL)

**Core finding**: LLMs exhibit a U-shaped performance curve — high accuracy at beginning and end of context, 30%+ degradation in the middle.

| Model | Start (pos 0) | Middle (pos 9) | End (pos 19) |
|-------|--------------|----------------|--------------|
| GPT-3.5-Turbo | 75.8% | 53.8% | 63.2% |
| Claude-1.3 | 59.9% | 56.8% | 60.1% |

**Critical**: GPT-3.5-Turbo's middle accuracy (53.8%) fell BELOW its closed-book baseline (56.1%) — the context actively hurt performance.

**Implication**: Place critical information at beginning or end. Never bury instructions in the middle of context.

## Chroma Context Rot (July 2025)

Tested 18 models across 4 families:

- **Shuffled text beats coherent text** for retrieval — coherent context makes the model get "absorbed" in the flow
- Lower semantic similarity between query and target = faster degradation
- Claude models: lowest hallucination rates but abstain when uncertain
- GPT models: highest hallucination rates
- GPT-4.1 mini: generates random words not present in input
- Thinking mode helps but doesn't solve the fundamental problem

## How Long Contexts Fail (Breunig, 2025)

Four failure modes:

1. **Context Poisoning**: A hallucination enters context, gets repeatedly referenced. Gemini 2.5 playing Pokemon hallucinated game state, causing infinite loops
2. **Context Distraction**: Beyond ~100K tokens, models repeat previous actions rather than synthesize new plans. Correctness falls at ~32K for Llama 3.1 405B
3. **Context Confusion**: Every model performs worse with more tools. 46 tools = failure, 19 tools = success in the same context window
4. **Context Clash**: Multi-turn vs single-prompt shows 39% average performance drop. OpenAI o3 dropped from 98.1 to 64.1. When LLMs take a wrong turn, they do not recover

### Mitigations (from "How to Fix Your Context"):
- **RAG**: Selective retrieval, not context stuffing
- **Tool masking**: Dynamic tool selection improved performance 44%, 77% speed improvement
- **Context quarantine**: Isolate tasks in separate threads — 90.2% improvement
- **Context pruning**: Provence tool reduced content 95% while preserving relevance
- **Context offloading**: External scratchpad — Anthropic showed 54% improvement

## JetBrains Research (NeurIPS 2025)

Compared observation masking vs LLM summarization on SWE-bench:

| Strategy | Result |
|----------|--------|
| Observation masking (keep last 10 turns) | Won in 4/5 settings, 52% cheaper |
| LLM summarization | Agents ran 13-15% longer, summaries obscure stopping signals |
| Both vs unmanaged | 50%+ cost reduction |

**Winner**: Simple observation masking. Keep last 10 turns visible, replace older with placeholders.

---

# PART 2: PROVIDER PROMPT ENGINEERING GUIDES

## Anthropic/Claude

### Prompt Structure
- XML tags are the primary structural tool: `<instructions>`, `<context>`, `<input>`
- Put long documents FIRST, queries LAST — up to 30% quality improvement
- 3-5 diverse examples in `<example>` tags dramatically improve accuracy
- Give Claude a role in the system prompt
- Be specific: "Create an analytics dashboard with X, Y, Z" not "Create a dashboard"
- Explain WHY, not just WHAT — Claude generalizes from motivation

### Prompt Caching
- Cache hierarchy: `tools` > `system` > `messages` — changes at any level invalidate that level and below
- Cache reads cost 10% of base (90% savings)
- Cache writes: 1.25x (5-min TTL) or 2x (1-hour TTL)
- Minimum cacheable: 1,024 tokens (Sonnet 4.5/4), 2,048 (Sonnet 4.6), 4,096 (Opus 4.6, Haiku 4.5)
- Lookback window: 20 blocks — if conversation pushes past 20 blocks from cache, you get misses
- **Anti-pattern**: Timestamps or dynamic content BEFORE cache breakpoint = cache miss every request

### Compaction (Beta)
- Server-side automatic summarization when input exceeds trigger threshold
- Default trigger: 150K tokens (minimum 50K)
- Generates structured summary: state, next steps, learnings
- Content before compaction block is ignored in subsequent requests
- Custom summarization prompt completely replaces default when provided
- Can pause after compaction to inject additional context

### Context Editing (Beta)
- Tool result clearing: clear oldest results when context exceeds threshold, keep N recent
- Thinking block clearing: control which thinking blocks survive
- Both happen server-side — client maintains full history

### Claude 4.6 Specifics
- Adaptive thinking with `effort` parameter (low/medium/high/max)
- More proactive — dial BACK aggressive prompting from older models
- Prefilled responses deprecated — use user messages or tools instead
- Tracks remaining context window via injected XML
- Native subagent orchestration — watch for overuse

## OpenAI

### Prompt Structure
- Developer messages (reasoning models) vs system messages (GPT)
- Structure: Identity → Instructions → Examples → Context
- Instruction hierarchy: Root (safety) > Developer > User

### Tool Calling
- Strict mode (`strict: true`) always recommended
- Keep under 20 tools upfront, under 100 total
- Flat schemas easier for models than deeply nested
- Tool definitions consume input tokens — minimize count and description length
- Reasoning models: do NOT add explicit CoT for tool calls — degrades performance

### Context Management
- **Trimming**: Keep last N turns, discard older. Deterministic, no drift risk
- **Summarization**: Compress older messages, keep last N verbatim. Risk: "minor hallucinations in summary propagate forward"
- **Compaction** (GPT-5.2+): `/responses/compact` endpoint, loss-aware compression, encrypted opaque items

### GPT-5.2+ Specifics
- More deliberate scaffolding, lower verbosity, stronger instruction adherence
- For long context (>10K): produce internal outline, re-state constraints, anchor claims to sections
- Preserve `phase` metadata on assistant items — dropping causes significant degradation

## Google/Gemini

### Prompt Structure
- XML tags (`<role>`, `<constraints>`, `<context>`, `<task>`) or Markdown headings
- Context first, questions last. Anchor with "Based on the information above..."
- Always include few-shot examples
- Concise instructions — verbose prompts cause over-analysis

### Gemini 3 Specifics
- **Temperature MUST stay at 1.0** — lower values cause looping/degradation
- Thinking level parameter: minimal/low/medium/high
- If previously using CoT: try `thinking_level: "high"` with simplified prompts instead
- Thought signatures must be returned exactly as received (400 error if missing)
- "Will sometimes ignore instructions to maintain persona adherence" — review carefully

## Key Provider Differences

| Aspect | Anthropic | OpenAI | Google |
|--------|-----------|--------|--------|
| Structure | XML tags | Identity→Instructions→Examples→Context | XML or Markdown |
| Long context | Docs top, query bottom (30% improvement) | Varies | Context first, questions last |
| Temperature | Standard | Standard | Must be 1.0 for Gemini 3 |
| CoT prompting | "Think thoroughly" > prescriptive steps | Avoid for reasoning models | Use thinking_level instead |
| Caching | Explicit breakpoints, 90% savings | N/A documented | Context caching |
| Parallel tools | Native, ~100% with prompt guidance | `parallel_tool_calls` param | Supported |

---

# PART 3: PRODUCTION PROMPT ARCHITECTURE PATTERNS

## Pattern 1: Layered Prompt Assembly

The dominant pattern. Production systems assemble prompts from modular components:

**Claude Code**: ~80 modular pieces, ~2.5K system tokens + 14-17K tool definitions. Cache boundary marker separates globally-cacheable from session-specific content.

**OpenCode**: 6 layers — provider base prompt → environment context → skills → user instructions → agent prompt → per-message reminders.

**Cursor**: Fully static system prompt + tool descriptions for maximum cache hits. User-specific context flows through separate message components in XML tags. Project rules fetched via tool calls, not injected into system prompt.

## Pattern 2: Priority-Based Composition (Priompt)

Cursor's open-source library. When you have more content than fits:

- Prompts are JSX components with priority scores
- Binary search finds minimum priority cutoff that fits token budget
- `<scope>` sets priorities, `<first>` provides fallbacks, `<isolate>` creates independent budgets
- Sourcemaps map rendered output back to source for debugging
- **Tradeoff**: Fine-grained priorities create hard-to-cache prompts

## Pattern 3: Static vs Dynamic Separation

The #1 cache optimization:

- **Static**: Base system prompt, tool definitions, role identity — rarely changes
- **Dynamic**: Environment info, conversation history, user instructions — changes per request
- Static content goes FIRST in the prompt to maximize cache prefix matching
- Cursor keeps system prompt 100% static — all personalization goes through messages

## Pattern 4: Prompt Versioning

Production teams treat prompts as deployable artifacts:

- Immutable versions (semantic or SHA-based)
- Environment promotion: Dev → Staging → Production
- Decoupled from code: applications fetch active versions at runtime
- Quality gates: every update runs against baseline evals
- Rollback is instant — no code redeploy needed

## Pattern 5: Tool Schema Management

- Tool definitions are the HEAVIEST component (14-17K tokens in Claude Code)
- Conditional tool inclusion — only load tools relevant to current context
- Flat schemas over deeply nested (easier for models, fewer tokens)
- Keep under 20 tools per request, under 100 total
- Dynamic tool selection improved performance 44% and speed 77%

## Pattern 6: Context Management Tiers

Every production system uses a tiered approach:

**Tier 1 — Prune** (mechanical, cheap):
- Replace old tool outputs with placeholders
- Keep last N turns visible
- Preserve tool call structure, drop tool output content
- OpenCode: protect last 40K tokens, prune beyond that

**Tier 2 — Compact** (semantic, expensive):
- LLM generates structured summary of conversation
- Summary replaces old messages
- Template: Goal, Instructions, Discoveries, Accomplished, Relevant Files
- Risk: "minor hallucinations in summary propagate forward"

**Tier 3 — Hard stop**:
- If compaction itself overflows, throw error
- Circuit breakers at cost/turn limits

## Pattern 7: The Agentic Loop

Canonical formula: `Agent = LLM + Memory + Planning + Tool Use`

```
while (true):
  load messages (only post-compaction history)
  check if done
  increment step counter
  check overflow → prune → compact if needed
  inject reminders
  build system prompt array
  resolve available tools
  stream LLM response
  if tool calls → process → continue
  if done → break
```

Production constraints:
- Agents consume 4x more tokens than chat, 15x in multi-agent
- 68% of production agents need human intervention within 10 steps
- Start single-agent, multi-agent only when tasks genuinely require specialization
- Circuit breakers: P95 cost thresholds, max turn limits, max step limits

## Pattern 8: UIMessage vs ModelMessage Separation

Vercel AI SDK pattern:
- **UIMessage**: Source of truth for application state, persisted to DB
- **ModelMessage**: Streamlined representation sent to model
- Separation makes persistence straightforward and provider-switching clean

---

# PART 4: HARD-WON PRODUCTION WISDOM

## From Companies That Shipped

1. **"LLMs generate proposals, deterministic systems enforce invariants"** — creativity upstream, boring execution downstream
2. **"Shrink the action space, don't improve the prompt"** — biggest hallucination reduction comes from limiting what the model CAN do
3. **Simple wrappers beat frameworks** — every team that succeeded used thin custom code
4. **Durable execution from day one** — resume from failure points, not restart
5. **Just-in-time context, not kitchen-sink** — inject when needed, remove when done
6. **Observability is architecture** — 94% of production agents have it; without it, you can't diagnose failures
7. **Model abstraction layer** — switching providers shouldn't require rewriting the system
8. **Tool masking over tool abundance** — expose only what's relevant per request
9. **Storing prompts in databases = disaster** — file-based prompt storage wins
10. **"Every token in context influences behavior, for better or worse"**

## The Numbers

| Metric | Value |
|--------|-------|
| Agent failure rate in production | 41-86% |
| Multi-turn performance drop | 39% |
| Middle-of-context accuracy loss | 30%+ |
| Tool output tokens vs user message | 100x |
| Observation masking cost reduction | 52% |
| Dynamic tool selection speed gain | 77% |
| Context quarantine improvement | 90.2% |
| Prompt cache read savings | 90% |
| Context pruning reduction | 95% |

---

# SOURCES

## Research Papers
- Lost in the Middle (Liu et al., TACL 2024) — aclanthology.org/2024.tacl-1.9
- Chroma Context Rot (2025) — trychroma.com/research/context-rot
- Needle in a Haystack — github.com/gkamradt/LLMTest_NeedleInAHaystack
- ACON: Agent Context Optimization — arxiv.org/pdf/2510.00615
- LLMLingua — llmlingua.com
- Prompt Compression Survey (NAACL 2025) — arxiv.org/html/2410.12388v2
- Google DeepMind 17x Error Trap — towardsdatascience.com

## Provider Docs
- Anthropic Prompt Engineering — docs.anthropic.com/en/docs/build-with-claude/prompt-engineering
- Anthropic Prompt Caching — platform.claude.com/docs/en/docs/build-with-claude/prompt-caching
- Anthropic Compaction — platform.claude.com/docs/en/docs/build-with-claude/compaction
- OpenAI Prompt Engineering — platform.openai.com/docs/guides/prompt-engineering
- OpenAI Function Calling — developers.openai.com/api/docs/guides/function-calling
- OpenAI Session Memory Cookbook — developers.openai.com/cookbook/examples/agents_sdk/session_memory
- Google Gemini Prompting — ai.google.dev/gemini-api/docs/prompting-strategies
- Google Gemini 3 Guide — ai.google.dev/gemini-api/docs/gemini-3

## Production Architecture
- How Claude Code Builds a System Prompt — dbreunig.com/2026/04/04
- How Cursor Works — blog.sshh.io/p/how-cursor-ai-ide-works
- Priompt (Cursor) — github.com/anysphere/priompt
- JetBrains Context Management — blog.jetbrains.com/research/2025/12
- How Long Contexts Fail — dbreunig.com/2025/06/22
- ZenML 1,200 Deployments — zenml.io/blog
- Context Engineering — weaviate.io/blog/context-engineering
- LangChain State of Agent Engineering — langchain.com/state-of-agent-engineering

## Startup Pain
- Pragmatic Engineer: LLM Evals — newsletter.pragmaticengineer.com
- Sketch.dev Agent Loop — sketch.dev/blog/agent-loop
- VentureBeat: Model Migration — venturebeat.com
- TechCrunch: AI Coding Startup Margins — techcrunch.com/2025/08/07
