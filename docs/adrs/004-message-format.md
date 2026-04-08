# ADR-004: Message Format

## Status
Accepted

## Date
2026-04-07

## Context

Messages are the bloodstream of an agent system. Every feature touches them: the prompt assembler reads them, the context manager prunes them, the compactor summarizes them, the model consumes them, the UI renders them, and storage persists them. The message format isn't a data modeling exercise — it's the architectural backbone that every other ADR depends on.

### The Three Consumers Problem

A message in Nocturnal has three consumers with conflicting needs:

**The model** wants provider-specific shapes. Anthropic expects content blocks with `type: "text"` and `type: "tool_use"`. OpenAI expects `content` strings and `tool_calls` arrays with JSON string arguments. Google has its own `parts` format. A tool result that works for Claude breaks GPT and vice versa. The Vercel AI SDK (ADR-003) handles this translation — but it needs a consistent input format to translate FROM.

**Storage** wants the complete picture, forever. ADR-002 says messages are never deleted — pruned tool outputs stay in history with a marker, compaction boundaries are inserted as special message parts, the full conversation is always recoverable. Storage also needs metadata the model never sees: timestamps, token counts, which step generated the message, whether compaction has touched it.

**The UI** (future) wants display-friendly structure. Streaming text chunks need to be appendable. Tool calls need status indicators (pending/running/complete/error). Reasoning blocks need to render differently from response text. Compaction summaries need to be visually distinct from real conversation.

### Why This Can't Be a Single Format

A single message format forces every consumer to handle every concern:

- Model-facing code would skip over UI metadata it doesn't care about
- Storage would need to carry provider-specific formatting it shouldn't know about
- The UI would parse through compaction state and token counts to find displayable text
- Pruning would need to modify the same object the model and UI read, creating mutation bugs

More concretely: ADR-002's pruning needs to mark a tool output as "cleared" without deleting the message. If the message format is just `{ role, content }`, there's nowhere to put that state. You'd need to either mutate the content string (fragile) or maintain a separate lookup table (complex). A parts-based format with per-part state solves this structurally.

### What OpenCode Proved

OpenCode's `message-v2.ts` implements a parts-based message format that powers everything:

- Each message has an array of **typed parts**: text, tool-call, tool-result, compaction, step-start
- Each part carries its own **state** — including a `compacted` timestamp that marks when a tool output was pruned
- `toModelMessages()` converts parts to provider format, replacing compacted tool results with `"[Old tool result content cleared]"`
- `filterCompacted()` walks messages backwards, finding compaction boundaries to window history
- The message itself is immutable in storage — only the conversion function changes what the model sees

This is the design that makes ADR-002's three-tier context management actually work. Without typed parts and per-part state, you can't prune tool outputs, mark compaction boundaries, or window history.

## Decision

Adopt a **rich internal message format with typed parts**, modeled after OpenCode's `MessageV2`. Messages are the single source of truth, stored with full fidelity. Conversion to model format and UI format happens at read time, never by mutating the stored message.

### Message Structure

```typescript
interface Message {
  id: string
  sessionId: string
  role: "user" | "assistant"
  
  // The core: typed parts array
  parts: MessagePart[]
  
  // Metadata (never sent to model)
  metadata: {
    createdAt: number          // timestamp
    stepIndex: number          // which agent loop step produced this
    model?: string             // which model generated this (assistant only)
    tokenCount?: number        // estimated tokens for this message
    parentId?: string          // for threading/compaction reference
    summary?: boolean          // true if this is a compaction summary
    finish?: boolean           // true if assistant finished (not mid-tool-loop)
    finishReason?: string      // "stop" | "tool_calls" | "max_tokens" | "error"
    error?: string             // error message if failed
  }
}
```

### Part Types

Each part type represents a distinct kind of content that flows through the system differently:

```typescript
type MessagePart =
  | TextPart
  | ToolCallPart
  | ToolResultPart
  | CompactionPart
  | ReasoningPart
  | StepStartPart

interface TextPart {
  type: "text"
  content: string
}

interface ToolCallPart {
  type: "tool-call"
  toolCallId: string
  toolName: string
  args: Record<string, unknown>
  state: {
    status: "pending" | "running" | "complete" | "error"
  }
}

interface ToolResultPart {
  type: "tool-result"
  toolCallId: string
  toolName: string
  result: unknown
  state: {
    status: "complete" | "error"
    compactedAt?: number    // timestamp when pruned (ADR-002 Tier 1)
  }
}

interface CompactionPart {
  type: "compaction"
  summary: string           // the structured summary from compaction agent
  compactedAt: number       // when compaction happened
}

interface ReasoningPart {
  type: "reasoning"
  content: string
  signature?: string        // provider-specific thinking signature (Anthropic/Google)
  redacted?: boolean        // true if thinking content was stripped but signature preserved
}

interface StepStartPart {
  type: "step-start"
  stepIndex: number
}
```

### Why These Specific Part Types

Each type exists because something in the system needs to treat it differently:

| Part Type | Why It's Separate |
|-----------|------------------|
| `text` | The model's actual response. Always displayed, never pruned. |
| `tool-call` | Preserved during pruning (ADR-002) — the model needs to see WHAT it called even after outputs are cleared. Has lifecycle status for UI rendering. |
| `tool-result` | The pruning target (ADR-002 Tier 1). The `compactedAt` field is the flag that triggers replacement with placeholder text during model conversion. 100x larger than user messages (Shopify data). |
| `compaction` | The boundary marker (ADR-002 Tier 2). `filterCompacted()` uses this to window history — everything before a completed compaction is invisible to future turns. |
| `reasoning` | Provider-specific thinking blocks. Signatures must be preserved and passed back for Anthropic/Google multi-turn tool loops. Can be stripped or redacted independently of text content. |
| `step-start` | Marks agent loop iterations. Enables observability ("step 7 is where things went wrong") without adding metadata to content parts. |

### Conversion Functions

Two conversion paths — one for the model, one for the future UI:

**`toModelMessages(message[], options)`** — Converts parts to the format AI SDK expects:

```
For each message:
  For each part:
    TextPart         → { type: "text", text: content }
    ToolCallPart     → { type: "tool-call", toolCallId, toolName, args }
    ToolResultPart   → if compactedAt: { type: "tool-result", result: "[Old tool result content cleared]" }
                       else: { type: "tool-result", result }
    ReasoningPart    → { type: "reasoning", text: content, signature } (if provider supports it)
    CompactionPart   → { type: "text", text: summary } (compaction summary becomes regular text)
    StepStartPart    → skipped (model doesn't need step boundaries)
```

Options:
- `stripMedia: boolean` — replace images/files with text references (for compaction input)
- `stripReasoning: boolean` — remove thinking blocks (for cost reduction)

**`filterCompacted(messages[])`** — Windows history to only include post-compaction messages:

```
Walk from newest to oldest:
  Collect each message
  If message has a CompactionPart AND its compaction is complete:
    Stop — everything older is behind the wall
Return collected messages (reversed to chronological order)
```

This is how ADR-002's Tier 2 works: compaction inserts a message with a `CompactionPart`, and `filterCompacted` hides everything before it. The old messages still exist in storage — they're just not loaded into context.

**`toUIMessages(message[])`** (future) — Converts parts to display-friendly format:

```
For each message:
  For each part:
    TextPart         → rendered as markdown
    ToolCallPart     → rendered with status badge (pending/running/complete/error)
    ToolResultPart   → collapsible, shows "[content cleared]" if compacted
    ReasoningPart    → collapsible thinking block with distinct styling
    CompactionPart   → visual divider: "Conversation compacted at [time]"
    StepStartPart    → step boundary indicator (optional, debug view)
```

### Storage Contract

Messages are stored with full fidelity — every part, every state field, every metadata field. The storage layer doesn't interpret parts or apply transformations. It's append-only for message creation, with the single exception of part state mutations:

**Allowed mutations:**
- Setting `toolResultPart.state.compactedAt` during pruning (ADR-002 Tier 1)
- Setting `toolCallPart.state.status` during tool execution
- Setting `message.metadata.finish` and `finishReason` when assistant completes

**Never mutated:**
- Part content (text, tool args, tool results) — once written, immutable
- Message role, id, sessionId — identity is permanent
- CompactionPart summary — once generated, it's the historical record

This immutability contract means any message can be replayed from storage to reconstruct the exact conversation state at any point in time.

## Options Considered

### Option A: Single Message Format
- **Pros:** Simple. One type to learn. No conversion functions.
- **Cons:** Every consumer handles every concern. Model code skips UI metadata. Storage carries provider-specific formatting. Pruning requires mutating content strings or maintaining separate lookup tables. No structural way to represent "this tool output was here but is now cleared." Doesn't support ADR-002's tiered context management — there's no place to put compaction boundaries or prune markers. This is the shape most tutorials use. It's also why most AI apps break in production.

### Option B: UIMessage vs ModelMessage (AI SDK Pattern)
- **Pros:** Clean separation between app state and model input. AI SDK provides `convertToModelMessages()` out of the box. Two types is manageable cognitive load.
- **Cons:** Two types isn't enough. Neither UIMessage nor ModelMessage has a place for: compaction boundaries, prune markers, per-part state, step tracking, reasoning signatures. You'd need to extend one or both with Nocturnal-specific fields, at which point you're building Option C anyway but starting from the wrong base. The AI SDK's UIMessage is designed for chat UIs, not agent systems with context management.

### Option C: Rich Internal Format with Typed Parts (OpenCode) — SELECTED
- **Pros:** Each part type maps to a specific system concern. Per-part state enables ADR-002's pruning without content mutation. Compaction boundaries are first-class parts, not metadata hacks. Conversion functions are pure transforms — no side effects, easy to test. The stored message is the single source of truth. OpenCode's `message-v2.ts` provides a reference implementation. Reasoning signatures preserved structurally (Anthropic/Google multi-turn tool loops). Extensible — new part types can be added without changing existing ones.
- **Cons:** More complex than flat messages. Every feature that touches messages needs to understand parts. Conversion functions are another layer to maintain and debug. The parts array can grow large for complex tool-heavy turns. Serialization/deserialization of typed unions needs careful implementation.

## Consequences

### What becomes easier
- **ADR-002 pruning**: Flip `compactedAt` on a tool result part. The conversion function handles the rest. No string manipulation, no content mutation.
- **ADR-002 compaction**: Insert a message with a `CompactionPart`. `filterCompacted()` handles history windowing. No custom message types or metadata flags.
- **Observability**: Every step, tool call, reasoning block, and compaction event is a distinct, queryable part. "Show me all tool calls in step 5" is a filter, not a parse.
- **Future UI**: Parts map directly to UI components. Text → markdown renderer. Tool call → status badge. Reasoning → collapsible block. No parsing required.
- **Replay and debugging**: Full-fidelity storage means you can reconstruct any conversation state from any point in time.

### What becomes harder
- **Initial complexity**: New contributors need to understand the parts system before they can work with messages. It's not `{ role: "assistant", content: "hello" }` anymore.
- **Serialization**: Typed unions (discriminated by `type` field) need careful JSON serialization/deserialization. TypeScript handles this well with discriminated unions, but runtime validation (Zod) adds boilerplate.
- **Testing**: Conversion functions need test coverage across all part types and state combinations. More part types = more test cases.
- **Migration**: If we change part types or add new ones, stored messages need to remain readable. Forward-compatible schema design matters from day one.

### Relationship to Other ADRs

**ADR-001 (Prompt Layering)**: The prompt assembly layer produces the system prompt. Messages carry the conversation. They're separate concerns — the system prompt is not a message, and messages don't contain system prompt layers. They come together in the final `streamText()` / `generateText()` call.

**ADR-002 (Context Management)**: This ADR IS the data structure that ADR-002 operates on. Pruning sets `compactedAt` on `ToolResultPart`. Compaction inserts `CompactionPart`. `filterCompacted()` reads `CompactionPart` to window history. Without typed parts, ADR-002's three tiers would need a fundamentally different (and worse) implementation.

**ADR-003 (Provider Abstraction)**: `toModelMessages()` produces the format AI SDK expects. AI SDK then translates to provider-specific wire format. Our conversion handles Nocturnal-specific concerns (pruned content, compaction, step markers). AI SDK handles provider-specific concerns (Anthropic content blocks vs OpenAI message format).

## References

### Primary
- OpenCode: `packages/opencode/src/session/message-v2.ts` — the reference implementation of typed parts, `toModelMessages()`, `filterCompacted()`
- OpenCode: `packages/opencode/src/session/compaction.ts` — how `compactedAt` state is set during pruning
- Vercel AI SDK: UIMessage vs ModelMessage separation — `ai-sdk.dev/docs/foundations/prompts`

### Research
- Shopify: tool outputs are 100x larger than user messages — why `ToolResultPart` is the pruning target
- JetBrains (NeurIPS 2025): observation masking (replacing old outputs with placeholders) won in 4/5 settings — validates the `compactedAt` → placeholder approach
- Anthropic: thinking signatures must be passed back unchanged in multi-turn tool loops — why `ReasoningPart.signature` exists
- Google: thought signatures cause 400 errors if missing — why `ReasoningPart.signature` is provider-agnostic
- OpenAI: encrypted reasoning state via `include: ["reasoning.encrypted_content"]` — reasoning is a cross-provider concern that needs its own part type
