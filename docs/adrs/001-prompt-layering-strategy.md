# ADR-001: Prompt Layering Strategy

## Status
Accepted

## Date
2026-04-07

## Context

The central question in building any LLM-powered system is: how do you structure what you send to the model?

Most AI startups treat this as a string concatenation problem — one big system prompt, maybe with some variables injected. Research shows this is how most of them fail. 95% of generative AI pilots never reach production (MIT/Fortune). Prompt tweaks don't scale reliability (HN consensus across hundreds of threads). A 39% performance drop occurs in multi-turn conversations (Microsoft/Salesforce). The technology isn't the hard part — the architecture is.

We studied four approaches used in production systems: flat prompts, fixed-layer stacks, priority-based composition, and static/dynamic splits. We chose to model our architecture after OpenCode (anomalyco/opencode) — and the reason is specific.

### Why OpenCode's Architecture

OpenCode is an open-source coding agent with 138K+ GitHub stars. It supports Claude, GPT, Gemini, Kimi, and other models through a single interface. What makes it architecturally interesting isn't just that it supports multiple providers — it's HOW it does it.

**The key insight: provider-specific prompt tuning unlocks model capability that generic prompts can't.**

In Andrea Grandi's hands-on comparison (same real-world coding task, multiple models), GPT-4.1 through OpenCode produced "much better results than using it from within VSCode." The model didn't change. The architecture around it did. OpenCode's agentic loop — tool chaining, file access, command execution — combined with a prompt specifically tuned for how GPT models work best, unlocked performance that a generic interface left on the table.

OpenCode achieves this through **per-provider base prompts** that are dramatically different from each other:

| Provider File | Character |
|--------------|-----------|
| `anthropic.txt` | Task management focus, TodoWrite tool usage, professional objectivity |
| `beast.txt` (GPT/o1/o3) | Aggressive autonomous agent — "keep going until solved," recursive web research, never stop |
| `gpt.txt` (GPT-5+) | Minimalist — "fewer than 4 lines," one-word answers preferred, zero preamble |
| `gemini.txt` | Convention-focused, security-first, detailed workflows for new applications |
| `kimi.txt` | Tool use and parallel execution emphasis, modular approach |

Each model gets instructions tuned to how IT thinks best. Claude gets structure and task management. GPT gets aggression and autonomy (or extreme minimalism for newer models). Gemini gets convention guardrails. This isn't just formatting differences — it's fundamentally different agent behaviors per model.

**This is also where the architecture has known problems.** OpenCode's own issue tracker documents that model switching mid-session can break (issue #11571 — thinking model to non-thinking model causes errors). GPT-5 underperforms Claude in Plan mode because it doesn't use built-in tools effectively (issue #2381). There's an open proposal to unify the prompts (issue #13605), acknowledging the divergence "might not have good justification with recent models." Gemini hallucinated fixtures and rewrote classes badly in Grandi's test.

**We're adopting the architecture knowing both its strengths and its weaknesses.** The layered approach is sound. The provider-aware composition is what makes models perform. The specific prompt content per provider is where iteration happens — and because OpenCode is open source (TypeScript, MIT license), we can study every line of how they got there and where they're still getting it wrong.

## Decision

Adopt a **fixed-layer prompt stack** with 6 layers, each with a single responsibility. The architecture is modeled after OpenCode's `system.ts`, `llm.ts`, and `prompt.ts` — files we can read line by line as reference.

```
Layer 1: Provider Base Prompt
  → Per-provider prompt file (anthropic.txt, openai.txt, gemini.txt, default.txt)
  → Defines identity, tone, behavioral rules, tool usage policies
  → Tuned to how each model class thinks best
  → Can be fully REPLACED by an agent-level prompt (Layer 5)

Layer 2: Environment Context
  → Working directory, platform, date, git status, runtime info
  → Generated dynamically at call time
  → Structured in an <env> block
  → Small, stable per session

Layer 3: Tool/Skill Listing
  → Available tools described for the model
  → Conditionally included based on agent permissions
  → HEAVIEST component — 14-17K tokens in Claude Code
  → Must support conditional loading (research shows >30 tools = confusion,
    dynamic selection improved performance 44%)

Layer 4: User Instructions
  → Project-level instruction files (walked up from working directory)
  → Global-level instruction files
  → Config-specified files (local paths, globs, URLs with timeout)
  → Prefixed with source path for transparency

Layer 5: Agent-Level Prompt
  → Optional per-agent override
  → When set, REPLACES Layer 1 entirely
  → Enables specialized agents: compaction, exploration, summarization, planning
  → Each agent gets exactly the prompt it needs, nothing more

Layer 6: Per-Message Injections
  → NOT part of system prompt — injected into the messages array
  → System reminders, plan mode instructions, mode-switch notices
  → Max-step warnings when agent hits limits
  → Contextual instructions discovered during tool use (e.g., reading a file
    near an instruction file triggers injection)
```

### Assembly

```typescript
const system = [
  agent.prompt ?? providerPrompt(model),  // Layer 5 or Layer 1
  environmentBlock(session),               // Layer 2
  toolListing(agent, tools),               // Layer 3
  userInstructions(config, cwd),           // Layer 4
].join("\n")

// Layer 6 is injected into messages[last_user_message], not system
```

### Cache Strategy

Two-part split for Anthropic prompt caching:

- **Part 1 (stable)**: Layer 1/5 — provider/agent prompt. Rarely changes within a session. Cache breakpoint here.
- **Part 2 (dynamic)**: Layers 2-4 — environment, tools, instructions. May change per request.

Cache reads cost 10% of base input price (90% savings). This structure maximizes prefix matching. Anthropic's docs explicitly recommend this pattern.

**Anti-pattern we will avoid**: Never put timestamps, request IDs, or per-request dynamic content before the cache breakpoint. OpenCode handles this by keeping the provider prompt as a pure, static first block.

## Options Considered

### Option A: Flat Prompt (Single String)
- **Pros:** Simple. Fast to prototype.
- **Cons:** Can't adapt per provider. Can't swap layers independently. No cache optimization. Doesn't scale — research shows this fails in production consistently. Every tool we studied that works at scale (OpenCode, Claude Code, Cursor) uses layered composition. Zero production systems of meaningful scale use flat prompts.

### Option B: Fixed-Layer Stack (OpenCode) — SELECTED
- **Pros:** Provider-aware composition that unlocks model capability (GPT-4.1 "much better" through OpenCode than VSCode). Clear separation of concerns — each layer has one job. Open-source reference (anomalyco/opencode, 138K stars) allows line-by-line study of a production system. Cache-optimizable with 2-part split. Layers can be tested and modified independently. Battle-tested with real users across multiple providers.
- **Cons:** Less flexible than priority-based when context overflows — can't auto-drop individual pieces. Requires maintaining per-provider prompt files (known pain point — OpenCode issue #13605). Fixed layer count may need extension. Model switching mid-session has known issues (OpenCode issue #11571).

### Option C: Priority-Based Composition (Cursor/Priompt)
- **Pros:** Most flexible for token budget management. Binary search finds optimal priority cutoff. Production-proven at Cursor scale (400M+ daily requests).
- **Cons:** Complex (JSX rendering, sourcemaps, priority scoring). Cache-unfriendly with fine-grained priorities. Harder to reason about what actually made it into the prompt. Priompt is open-source but the full system around it is not. Premature optimization — solves a problem we don't have yet.

### Option D: Static/Dynamic Split (Cursor Cache-First)
- **Pros:** Maximum cache hit rate. System prompt never changes.
- **Cons:** ALL dynamic content through messages or tool calls — adds latency and complexity. Project rules fetched via tool calls means the model must burn a step to get its own instructions. Optimizes for cost at expense of developer experience. Cursor's full system is not open-source — can't study it.

## Consequences

### What becomes easier
- **Provider support**: Adding a new model = write a new base prompt file. Everything else stays the same.
- **Debugging**: Inspect each layer independently. "The model got confused" becomes "Layer 3 had 47 tools when it should have had 12."
- **Learning**: Every design decision maps directly to files in OpenCode's source that we can read and challenge.
- **Testing**: Each layer is a pure function — given these inputs, produce this output. Unit testable.
- **Cache optimization**: 2-part split is mechanical to implement.

### What becomes harder
- **Token overflow**: If all 6 layers exceed the window, we need a strategy for what to cut. Fixed layers don't auto-trim. This will be addressed in ADR-002 (Context Management).
- **Prompt drift across providers**: Maintaining 4-6 different base prompts that all need to be good. OpenCode's own team is debating whether to unify these (issue #13605). We'll face the same tension.
- **Layer conflicts**: Instructions in Layer 4 could contradict Layer 1. Need clear precedence: later layers override earlier layers for behavioral instructions.

### Open questions for future ADRs
- How do we handle token overflow when layers exceed budget? (ADR-002)
- How do we abstract provider differences beyond just prompts? (ADR-003)
- Should we version prompts independently from code? (Future ADR)
- At what point do we consider unifying provider prompts vs keeping them separate? (Revisit after initial implementation)

## References

### Primary (OpenCode source)
- `packages/opencode/src/session/system.ts` — Provider prompt selection, environment block, skills listing
- `packages/opencode/src/session/llm.ts` — Final system prompt assembly, 2-part cache split
- `packages/opencode/src/session/prompt.ts` — Agentic loop, reminder injection (Layer 6)
- `packages/opencode/src/session/instruction.ts` — User instruction discovery (Layer 4)
- `packages/opencode/src/session/prompt/*.txt` — Per-provider base prompts (Layer 1)
- github.com/anomalyco/opencode (138K stars, MIT license)

### Evidence
- Andrea Grandi: GPT-4.1 "much better results" through OpenCode than VSCode — andreagrandi.it
- XDA Developers: OpenCode "gets noticeably closer" to Claude Code than other alternatives
- OpenCode issue #13605: Proposal to unify provider prompts
- OpenCode issue #11571: Model switching mid-session errors
- OpenCode issue #2381: GPT-5 underperforms Claude in Plan mode
- OpenCode issue #20258: Default prompt degrades Kimi K2.5 on benchmarks

### Research
- Lost in the Middle (TACL 2024): 30%+ accuracy loss for middle-of-context info — informs layer ordering
- Anthropic prompt caching docs: 90% savings, 2-part structure, 20-block lookback window
- Claude Code architecture: ~80 modular pieces, 14-17K tokens for tool definitions alone
- Context Confusion (Breunig): >30 tools = model confusion, dynamic selection = 44% improvement
- Multi-turn performance drop: 39% (Microsoft/Salesforce)
