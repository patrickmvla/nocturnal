# RFC-001: Nocturnal System Architecture

## Status
Accepted (Revised after OpenCode source challenge — see RFC-002)

## Date
2026-04-07 (Revised 2026-04-07)

## Authors
Patrick Mvula

## Summary

Nocturnal is a production-grade general-purpose agent backend built to deeply understand prompt architecture and context degradation. This RFC describes how the four architectural decisions (ADR-001 through ADR-004) compose into a working system.

The system has one job: accept a user message, run an agentic loop against an LLM, manage context intelligently, and return a response — streamed, observable, and provider-agnostic.

### Revision Notes

This RFC was challenged against OpenCode's actual source code (RFC-002). 11 changes were adopted. Key revisions: separate message/part tables, branded IDs, unified ToolPart, lazy tool initialization, filterCompacted success checking, option merging, tool name repair, media handling, FilePart and StepFinishPart additions, protected tools in pruning.

---

## 1. System Overview

```
                            ┌──────────────────┐
                            │   Client (any)   │
                            │  CLI / UI / API  │
                            └────────┬─────────┘
                                     │ HTTP/SSE
                            ┌────────▼─────────┐
                            │   Hono Server    │
                            │   (Bun runtime)  │
                            └────────┬─────────┘
                                     │
                    ┌────────────────┬┴───────────────┐
                    │                │                 │
           ┌────────▼──────┐ ┌──────▼───────┐ ┌──────▼───────┐
           │  Session API  │ │  Config API  │ │  Health API  │
           │  /sessions/*  │ │  /config/*   │ │  /health     │
           └────────┬──────┘ └──────────────┘ └──────────────┘
                    │
           ┌────────▼──────────────────────────────────────┐
           │              Session Manager                   │
           │  Create, resume, list sessions                 │
           │  Each session holds: messages[], config, state │
           └────────┬──────────────────────────────────────┘
                    │
           ┌────────▼──────────────────────────────────────┐
           │              Agent Loop (core)                 │
           │                                                │
           │  while (true):                                 │
           │    1. Load messages (filterCompacted)           │
           │    2. Check overflow → prune → compact          │
           │    3. Assemble prompt (6 layers)                │
           │    4. Resolve tools (lazy init, per-model)      │
           │    5. Merge options (model + agent + variant)   │
           │    6. Call LLM (AI SDK streamText)              │
           │    7. Process response                          │
           │    8. If tool calls → execute → continue        │
           │    9. If done → break                           │
           │   10. Check circuit breakers                    │
           └───┬────────────┬──────────────┬───────────────┘
               │            │              │
      ┌────────▼───┐ ┌─────▼──────┐ ┌─────▼──────┐
      │  Prompt    │ │  Context   │ │  Tool      │
      │  Assembler │ │  Manager   │ │  Executor  │
      │  (ADR-001) │ │  (ADR-002) │ │            │
      └────────────┘ └────────────┘ └────────────┘
               │            │              │
      ┌────────▼────────────▼──────────────▼───────────────┐
      │              Database (Drizzle + bun:sqlite)         │
      │  message table ──┐                                   │
      │  part table ─────┘ separate rows, JSON data columns  │
      │  session table                                       │
      │  Branded IDs (ses_, msg_, prt_) with time-ordering   │
      └────────────────────────┬───────────────────────────┘
                               │
      ┌────────────────────────▼───────────────────────────┐
      │              LLM Layer (ADR-003)                     │
      │  AI SDK (streamText/generateText)                    │
      │  Provider adapters (Anthropic, OpenAI, Google, etc.) │
      │  Native SDK escape hatches (batch, cache mgmt)       │
      └────────────────────────────────────────────────────┘
```

---

## 2. Data Flow

A single user message goes through five phases:

### Phase 1: Receive

```
Client sends POST /sessions/:id/messages
Body: { content: "...", files?: [...] }

→ Session Manager loads or creates session
→ User message stored in message table (branded msg_ ID, ascending)
→ TextPart and/or FileParts stored in part table (branded prt_ IDs)
→ Agent loop starts
```

### Phase 2: Context Check (ADR-002)

```
Load messages via filterCompacted()
  → Walk forward through messages
  → Track completed compaction summaries (assistant with summary=true, finished, no error)
  → Stop at user message with CompactionPart whose compaction SUCCEEDED
  → Return only messages after that boundary (reversed to chronological)
  
Count tokens in current context

If tokens > pruneAt (75% of model window):
  → Walk backwards through messages
  → Skip protected tool outputs (configurable exclusion list)
  → Set compactedAt on ToolParts older than protection window
  → Part state updated independently in part table (no message rewrite)
  → Recount

If tokens > compactAt (85% of model window):
  → Send pruned history to compaction agent
  → Insert CompactionPart into new message
  → Replay last user message
  → Recount

If tokens > hardStopAt (95% of model window):
  → Return ContextOverflowError
```

### Phase 3: Assemble (ADR-001)

```
Build system prompt from 6 layers:

  Layer 1/5: Provider base prompt OR agent override
    → Read from prompt files (anthropic.txt, openai.txt, etc.)
    → Selected based on model identifier string matching
    
  Layer 2: Environment context
    → <env> block: working directory, platform, date, model info
    
  Layer 3: Tool listing
    → Available tools for current agent
    → Conditionally loaded based on agent permissions
    → Per-model filtering (some tools only available for certain providers)
    
  Layer 4: User instructions  
    → Walk up from cwd looking for instruction files
    → Load global instructions
    → Load config-specified instruction files
    
  Layer 6: Per-message injections
    → Injected into last user message, not system prompt
    → Reminders, mode switches, max-step warnings

Merge options: pipe(model.options, agent.options, variant.options)

Convert messages via toModelMessages()
  → Compacted ToolParts → "[Old tool result content cleared]"
  → Pending/running ToolParts → "[Tool execution was interrupted]"
  → ReasoningParts → include signatures for multi-turn
  → CompactionParts → rendered as text summary
  → FileParts → provider-aware media handling
    → Providers without media-in-tool-result support:
      extract attachments, inject as separate user messages
  → StepStartParts, StepFinishParts → skipped
```

### Phase 4: Execute

```
Call AI SDK streamText() with:
  - model (from session config)
  - system prompt array (assembled in Phase 3)
  - messages (converted in Phase 3)
  - tools (resolved in Phase 3, with tool name repair for case-sensitivity)
  - merged options (temperature, topP, maxOutputTokens)
  - providerOptions (caching, thinking, etc.)

Stream response back to client via SSE:
  - text-delta events as tokens arrive
  - tool-call events when model invokes tools
  - tool-result events when tools complete
  - reasoning events for thinking blocks
  - step-finish events with per-step cost/tokens
  - error events on failure

If response contains tool calls:
  → Execute each tool
  → Store ToolPart with state transition: pending → running → completed|error
  → Record StepFinishPart with cost and token usage for this step
  → Loop back to Phase 2 (context may have grown)

If response is text-only (finish_reason: "stop"):
  → Store assistant message with TextPart
  → Record StepFinishPart
  → Break loop
```

### Phase 5: Return

```
Stream completes → client receives final message
Session state updated in store
Circuit breaker counters updated (turns, cost, consecutive tool calls)
Post-loop pruning pass (clean up any tool outputs that grew during final step)
```

---

## 3. Core Components

### 3.1 ID System

All entity IDs are branded TypeScript types with prefix, time-ordering, and randomness:

```typescript
// Branded types prevent mixing IDs across entities
type SessionID = string & { readonly __brand: "SessionID" }
type MessageID = string & { readonly __brand: "MessageID" }
type PartID = string & { readonly __brand: "PartID" }

// Prefixed for instant readability in logs and debugging
// ses_ff3a8b...  msg_00c574...  prt_00c574...

// Time-ordered for free sorting:
//   Sessions: DESCENDING (newest first in lexicographic sort)
//   Messages: ASCENDING (oldest first — chronological)
//   Parts: ASCENDING (oldest first)
```

| Entity | Prefix | Sort Order | Why |
|--------|--------|------------|-----|
| Session | `ses_` | Descending | `SELECT * FROM session ORDER BY id` = newest first |
| Message | `msg_` | Ascending | `SELECT * FROM message WHERE session_id = ? ORDER BY id` = chronological |
| Part | `prt_` | Ascending | Parts within a message in creation order |

IDs encode a 6-byte timestamp (with monotonic counter) plus random bytes in base62. Descending IDs use bitwise NOT on the timestamp. No secondary indexes needed for basic time-ordered queries.

### 3.2 Session Manager

A session is a conversation with state:

```typescript
interface Session {
  id: SessionID
  createdAt: number
  updatedAt: number
  
  // Configuration
  config: SessionConfig
  
  // State
  state: {
    status: "active" | "paused" | "completed" | "error"
    currentStep: number
    totalTokensUsed: number
    totalCost: number
    compactionCount: number
  }
}

interface SessionConfig {
  // Model selection
  provider: string          // "anthropic" | "openai" | "google" | etc.
  model: string             // "claude-opus-4-6" | "gpt-4.1" | etc.
  
  // Agent selection
  agent: string             // "default" | "explorer" | "planner" | etc.
  
  // Context thresholds (ADR-002, percentage-based)
  context: {
    pruneAt: number          // default: 0.75
    protectionWindow: number // default: 0.15
    compactAt: number        // default: 0.85
    hardStopAt: number       // default: 0.95
    outputReserve: number    // default: 0.05
  }
  
  // Circuit breakers
  limits: {
    maxTurns: number              // default: 100
    maxConsecutiveToolCalls: number // default: 25
    maxCostPerSession: number      // default: 5.00 (USD)
  }
  
  // Provider-specific options (passed to AI SDK providerOptions)
  providerOptions: Record<string, unknown>
}
```

### 3.3 Agent Loop

The heart of the system:

```typescript
async function runLoop(session: Session, userMessage: Message): AsyncGenerator<StreamEvent> {
  let step = 0
  
  while (true) {
    step++
    
    // 1. Load post-compaction messages (with success checking)
    const allMessages = await store.getMessages(session.id)
    const messages = filterCompacted(allMessages)
    
    // 2. Context management (ADR-002)
    const tokenCount = countTokens(messages, session.config)
    const modelLimit = getModelContextWindow(session.config.model)
    
    if (tokenCount > modelLimit * session.config.context.pruneAt) {
      await pruneToolOutputs(messages, session.config.context)
    }
    
    const postPruneCount = countTokens(messages, session.config)
    if (postPruneCount > modelLimit * session.config.context.compactAt) {
      await compact(session, messages)
      continue  // restart loop with compacted history
    }
    
    if (postPruneCount > modelLimit * session.config.context.hardStopAt) {
      throw new ContextOverflowError(session.id, postPruneCount, modelLimit)
    }
    
    // 3. Assemble prompt (ADR-001)
    const system = assembleSystemPrompt(session.config)
    
    // 4. Resolve tools (lazy init, per-model filtering)
    const tools = await resolveTools(session.config.agent, session.config.model)
    
    // 5. Merge options (model defaults + agent overrides)
    const options = mergeOptions(
      getModelDefaults(session.config.model),
      getAgentOptions(session.config.agent),
    )
    
    // 6. Convert messages with provider-aware media handling
    const modelMessages = toModelMessages(messages, {
      provider: session.config.provider,
      stripMedia: false,
    })
    
    // 7. Inject per-message reminders (Layer 6)
    injectReminders(modelMessages, session.state, step)
    
    // 8. Call LLM (ADR-003) with tool name repair
    const result = await streamText({
      model: getModel(session.config),
      system,
      messages: modelMessages,
      tools,
      ...options,
      providerOptions: session.config.providerOptions,
      experimental_repairToolCall: repairToolName(tools),
    })
    
    // 9. Stream and process response
    for await (const event of result) {
      yield event
    }
    
    // 10. Store assistant message and step finish
    const assistantMessage = buildMessage(result)
    await store.saveMessage(session.id, assistantMessage)
    await store.savePart(assistantMessage.id, {
      type: "step-finish",
      cost: result.usage.totalCost,
      tokens: {
        input: result.usage.promptTokens,
        output: result.usage.completionTokens,
        reasoning: result.usage.reasoningTokens ?? 0,
        cache: { read: result.usage.cacheReadTokens ?? 0, write: result.usage.cacheWriteTokens ?? 0 },
      }
    })
    
    // 11. Check finish condition
    if (result.finishReason === "stop") break
    
    // 12. Execute tool calls
    if (result.toolCalls?.length) {
      for (const call of result.toolCalls) {
        // Update tool part state: pending → running
        await store.updatePartState(call.partId, { status: "running" })
        
        const toolResult = await executeTool(call)
        
        // Update tool part state: running → completed|error
        await store.updatePartState(call.partId, {
          status: toolResult.error ? "error" : "completed",
          output: toolResult.output,
          error: toolResult.error,
        })
      }
    }
    
    // 13. Circuit breakers
    if (step >= session.config.limits.maxTurns) {
      yield { type: "max-turns-reached" }
      break
    }
    
    session.state.currentStep = step
    await store.updateSession(session)
  }
  
  // Post-loop: final prune pass
  await pruneToolOutputs(
    await store.getMessages(session.id),
    session.config.context,
  )
}
```

### 3.4 Prompt Assembler (ADR-001)

```typescript
function assembleSystemPrompt(config: SessionConfig): string {
  const agent = getAgent(config.agent)
  
  // Layer 1 or 5: provider base OR agent override
  const basePrompt = agent.prompt 
    ?? loadProviderPrompt(config.provider, config.model)
  
  // Layer 2: environment
  const env = buildEnvironmentBlock(config)
  
  // Layer 3: tools listing
  const toolsListing = buildToolsListing(agent)
  
  // Layer 4: user instructions
  const instructions = loadInstructions(config)
  
  return [basePrompt, env, toolsListing, instructions].join("\n")
}

// Option merging: model defaults < agent overrides
function mergeOptions(modelDefaults: LLMOptions, agentOverrides: LLMOptions): LLMOptions {
  return deepMerge(modelDefaults, agentOverrides)
}
```

The 2-part cache split: `basePrompt` is stable (cache breakpoint here), everything else is dynamic.

### 3.5 Context Manager (ADR-002)

```typescript
// Protected tools: these tool outputs are never pruned
const PROTECTED_TOOLS: Set<string> = new Set(["skill"])

async function pruneToolOutputs(
  messages: Message[], 
  config: ContextConfig
): Promise<void> {
  const modelLimit = getModelContextWindow(config.model)
  const protectTokens = modelLimit * config.protectionWindow
  
  let recentTokens = 0
  
  for (let i = messages.length - 1; i >= 0; i--) {
    const parts = await store.getParts(messages[i].id)
    for (const part of parts) {
      if (part.type === "tool" 
          && part.state.status === "completed"
          && !part.state.compactedAt
          && !PROTECTED_TOOLS.has(part.toolName)) {
        const partTokens = estimateTokens(part.state.output)
        recentTokens += partTokens
        
        if (recentTokens > protectTokens) {
          part.state.compactedAt = Date.now()
          // Surgical update: only this part row in the part table
          await store.updatePartState(part.id, part.state)
        }
      }
    }
  }
}

// filterCompacted: only honor SUCCESSFUL compactions
function filterCompacted(messages: MessageWithParts[]): MessageWithParts[] {
  const result: MessageWithParts[] = []
  const completedCompactions = new Set<string>()
  
  for (const msg of messages) {
    result.push(msg)
    
    // Track assistant messages that are successful compaction summaries
    if (msg.info.role === "assistant" && msg.info.summary && msg.info.finish && !msg.info.error) {
      completedCompactions.add(msg.info.parentId)
    }
    
    // Stop at user message with compaction part that has a completed summary
    if (msg.info.role === "user" 
        && completedCompactions.has(msg.info.id)
        && msg.parts.some(p => p.type === "compaction")) {
      break
    }
  }
  
  result.reverse()
  return result
}
```

### 3.6 Database Schema (Drizzle ORM + bun:sqlite)

Messages and parts are **separate tables**. Part state mutations (pruning, tool status) happen with surgical row updates, not message blob rewrites.

```typescript
// SQLite pragmas: WAL mode, synchronous=NORMAL, 5s busy timeout, 
// 64MB cache, foreign keys ON

const sessionTable = sqliteTable("session", {
  id: text().$type<SessionID>().primaryKey(),       // ses_ prefix, descending
  slug: text().notNull(),
  directory: text().notNull(),
  title: text().notNull(),
  config: text({ mode: "json" }).$type<SessionConfig>().notNull(),
  state: text({ mode: "json" }).$type<SessionState>().notNull(),
  time_created: integer().notNull(),
  time_updated: integer().notNull(),
})

const messageTable = sqliteTable("message", {
  id: text().$type<MessageID>().primaryKey(),       // msg_ prefix, ascending
  session_id: text().$type<SessionID>().notNull()
    .references(() => sessionTable.id, { onDelete: "cascade" }),
  time_created: integer().notNull(),
  time_updated: integer().notNull(),
  data: text({ mode: "json" }).notNull(),           // MessageInfo (role, model, agent, tokens, cost, etc.)
}, (table) => [
  index("message_session_idx").on(table.session_id, table.time_created, table.id)
])

const partTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),          // prt_ prefix, ascending
  message_id: text().$type<MessageID>().notNull()
    .references(() => messageTable.id, { onDelete: "cascade" }),
  session_id: text().$type<SessionID>().notNull(),
  time_created: integer().notNull(),
  time_updated: integer().notNull(),
  data: text({ mode: "json" }).notNull(),           // Part data (type discriminated union)
}, (table) => [
  index("part_message_idx").on(table.message_id, table.id),
  index("part_session_idx").on(table.session_id),
])
```

**Why separate tables**: ADR-002's pruning sets `compactedAt` on individual tool parts. With embedded parts, that means rewriting the entire message JSON blob (potentially 100+ KB for a message with many tool calls). With separate rows, it's a targeted `UPDATE part SET data = ? WHERE id = ?` on a single small row.

**JSON data columns**: The `data` column holds everything except IDs and foreign keys. IDs are pulled out as indexed columns for relational queries. The JSON blob holds the typed content. This is the same pattern OpenCode uses — relational structure for queries, JSON flexibility for typed content.

### 3.7 Message Format (ADR-004, Revised)

#### Message Info

```typescript
// Discriminated union by role
type MessageInfo = UserInfo | AssistantInfo

interface UserInfo {
  role: "user"
  time: { created: number }
  agent: string
  model: { provider: string; model: string }
  system?: string           // per-message system prompt injection
}

interface AssistantInfo {
  role: "assistant"
  time: { created: number; completed?: number }
  parentId: MessageID        // links to the user message that triggered this
  provider: string
  model: string
  agent: string
  summary?: boolean          // true if this is a compaction summary
  cost: number
  tokens: {
    input: number
    output: number
    reasoning: number
    cache: { read: number; write: number }
  }
  finish?: string            // "stop" | "tool_calls" | "max_tokens" | "error"
  error?: { name: string; message: string }
}
```

#### Part Types (8 types)

```typescript
type MessagePart =
  | TextPart
  | ToolPart              // UNIFIED: call + result + state lifecycle
  | FilePart              // NEW: attachments (images, PDFs)
  | ReasoningPart
  | CompactionPart
  | StepStartPart
  | StepFinishPart        // NEW: per-step cost and token tracking
  | RetryPart             // retry markers

// --- Unified ToolPart (replaces separate ToolCallPart + ToolResultPart) ---
interface ToolPart {
  type: "tool"
  toolName: string
  callId: string
  state: ToolState         // discriminated union by status
}

type ToolState =
  | { status: "pending"; input: Record<string, unknown> }
  | { status: "running"; input: Record<string, unknown> }
  | { status: "completed"; input: Record<string, unknown>; output: unknown; compactedAt?: number }
  | { status: "error"; input: Record<string, unknown>; error: string }

// --- FilePart (NEW) ---
interface FilePart {
  type: "file"
  mime: string             // "image/png", "application/pdf", etc.
  filename?: string
  url: string              // data URL or file path
}

// --- StepFinishPart (NEW) ---
interface StepFinishPart {
  type: "step-finish"
  cost: number
  tokens: {
    input: number
    output: number
    reasoning: number
    cache: { read: number; write: number }
  }
}

// --- Remaining parts (unchanged from ADR-004) ---
interface TextPart {
  type: "text"
  content: string
}

interface ReasoningPart {
  type: "reasoning"
  content: string
  signature?: string       // provider-specific thinking signature
  redacted?: boolean
}

interface CompactionPart {
  type: "compaction"
  summary: string
  compactedAt: number
}

interface StepStartPart {
  type: "step-start"
  stepIndex: number
}

interface RetryPart {
  type: "retry"
  attempt: number
  error: string
  time: number
}
```

#### Conversion: toModelMessages()

```typescript
function toModelMessages(messages: MessageWithParts[], options: ConvertOptions): ModelMessage[] {
  const supportsMediaInToolResults = checkMediaSupport(options.provider)
  const mediaExtractions: ModelMessage[] = []  // injected user messages for extracted media
  
  for (const msg of messages) {
    for (const part of msg.parts) {
      switch (part.type) {
        case "tool":
          if (part.state.status === "completed" && part.state.compactedAt) {
            // Pruned: replace output with placeholder
            part.state.output = "[Old tool result content cleared]"
          }
          if (part.state.status === "pending" || part.state.status === "running") {
            // Interrupted: required by Anthropic API
            part.state = { ...part.state, status: "error", error: "[Tool execution was interrupted]" }
          }
          // Media extraction for providers without media-in-tool-result support
          if (!supportsMediaInToolResults && hasMediaAttachments(part.state.output)) {
            mediaExtractions.push(extractMediaAsUserMessage(part))
          }
          break
          
        case "step-start":
        case "step-finish":
          // Skipped — model doesn't need step boundaries
          continue
          
        case "file":
          // Handled by AI SDK's content part conversion
          break
          
        case "reasoning":
          // Include signature for multi-turn tool loops
          break
      }
    }
  }
  
  // Inject extracted media messages at appropriate positions
  return [...convertedMessages, ...mediaExtractions]
}
```

### 3.8 Tool System

Tools use **lazy initialization** — constructed on first use, not at boot:

```typescript
interface ToolInfo {
  name: string
  init: () => Promise<ToolDef>    // lazy — only called when tool is first needed
}

interface ToolDef {
  description: string
  inputSchema: ZodSchema
  execute: (args: unknown, ctx: ToolContext) => Promise<ToolOutput>
}

interface ToolOutput {
  title?: string
  output: unknown
  metadata?: Record<string, unknown>
  attachments?: FilePart[]
}

interface ToolContext {
  sessionId: SessionID
  messageId: MessageID
  agent: string
  abort: AbortSignal
}
```

**Tool resolution** is per-agent AND per-model:

```typescript
async function resolveTools(agent: string, model: string): Promise<Record<string, AITool>> {
  const agentInfo = getAgent(agent)
  const allTools = await initializeTools()  // lazy init
  
  // Filter by agent permissions
  let tools = filterByPermissions(allTools, agentInfo.permissions)
  
  // Filter by model capabilities (some tools only work with certain providers)
  tools = filterByModel(tools, model)
  
  return tools
}
```

**Tool name repair** — case-insensitive fallback when model returns wrong casing:

```typescript
function repairToolName(tools: Record<string, AITool>) {
  return (toolCall: ToolCall) => {
    if (tools[toolCall.toolName]) return toolCall
    
    // Try case-insensitive match
    const match = Object.keys(tools).find(
      name => name.toLowerCase() === toolCall.toolName.toLowerCase()
    )
    if (match) return { ...toolCall, toolName: match }
    
    // Route to invalid tool handler
    return { ...toolCall, toolName: "invalid" }
  }
}
```

### 3.9 Agent Definitions

```typescript
interface Agent {
  name: string
  description?: string
  
  // Prompt override (replaces Layer 1 when set)
  prompt?: string
  
  // Permission rules for tools
  permissions: PermissionRuleset    // { toolName: "allow" | "deny" | "ask" }
  
  // Model override (optional — defaults to session model)
  model?: { provider: string; model: string }
  
  // LLM option overrides (merged on top of model defaults)
  options?: {
    temperature?: number
    topP?: number
    maxOutputTokens?: number
  }
  
  // Max steps before forced stop
  maxSteps?: number
}
```

Built-in agents:

| Agent | Purpose | Prompt Override | Tools | Options |
|-------|---------|----------------|-------|---------|
| `default` | General conversation | None (uses provider base) | All | Model defaults |
| `compaction` | Summarize conversation (ADR-002) | `compaction.txt` | None (all denied) | — |
| `explorer` | Read-only information gathering | `explorer.txt` | Read-only subset | — |
| `title` | Generate session title | `title.txt` | None | `temperature: 0.5` |

---

## 4. API Surface

Hono HTTP server running on Bun. RESTful endpoints with SSE for streaming.

### Sessions

```
POST   /sessions                    Create new session
GET    /sessions                    List sessions (ordered by ses_ ID = newest first)
GET    /sessions/:id                Get session details
PATCH  /sessions/:id                Update session config
DELETE /sessions/:id                Delete session (cascades to messages and parts)
```

### Messages

```
POST   /sessions/:id/messages       Send message (streams response via SSE)
GET    /sessions/:id/messages        Get message history (ordered by msg_ ID = chronological)
GET    /sessions/:id/messages/:mid   Get single message with all parts
```

### Config

```
GET    /config/providers             List available providers and models
GET    /config/agents                List available agents
GET    /config/tools                 List available tools
```

### Health

```
GET    /health                       Server health check
GET    /health/providers             Provider connectivity check
```

### Streaming Response Format

`POST /sessions/:id/messages` returns an SSE stream:

```
event: text-delta
data: {"content": "Here"}

event: text-delta
data: {"content": " is"}

event: tool-call
data: {"toolCallId": "tc_1", "toolName": "readFile", "args": {"path": "..."}, "status": "pending"}

event: tool-status
data: {"toolCallId": "tc_1", "status": "running"}

event: tool-result
data: {"toolCallId": "tc_1", "status": "completed", "result": "..."}

event: reasoning
data: {"content": "Let me think about..."}

event: step-finish
data: {"step": 2, "cost": 0.003, "tokens": {"input": 1500, "output": 350}}

event: done
data: {"messageId": "msg_00c574...", "finishReason": "stop", "totalCost": 0.012}

event: error
data: {"code": "CONTEXT_OVERFLOW", "message": "..."}
```

---

## 5. Project Structure

```
nocturnal/
├── docs/
│   ├── vision/                  # Problem statement, research
│   ├── adrs/                    # Architecture decisions
│   ├── rfc/                     # This document + challenge
│   ├── api/                     # API contracts (future)
│   └── implementation/          # Build plan (next)
│
├── src/
│   ├── index.ts                 # Hono server entry point
│   │
│   ├── api/                     # HTTP route handlers
│   │   ├── sessions.ts
│   │   ├── messages.ts
│   │   ├── config.ts
│   │   └── health.ts
│   │
│   ├── core/                    # The engine
│   │   ├── loop.ts              # Agent loop (§3.3)
│   │   ├── prompt.ts            # Prompt assembler (§3.4)
│   │   ├── context.ts           # Context manager — prune, compact, overflow (§3.5)
│   │   └── stream.ts            # SSE streaming helpers
│   │
│   ├── messages/                # Message format (ADR-004)
│   │   ├── types.ts             # Message, Part type definitions (branded IDs)
│   │   ├── convert.ts           # toModelMessages(), filterCompacted()
│   │   └── tokens.ts            # Token counting/estimation
│   │
│   ├── providers/               # Provider abstraction (ADR-003)
│   │   ├── registry.ts          # Provider/model registry
│   │   └── options.ts           # Option merging, provider-specific config
│   │
│   ├── prompts/                 # Prompt files (ADR-001)
│   │   ├── anthropic.txt
│   │   ├── openai.txt
│   │   ├── gemini.txt
│   │   ├── default.txt
│   │   └── agents/
│   │       ├── compaction.txt
│   │       ├── explorer.txt
│   │       └── title.txt
│   │
│   ├── agents/                  # Agent definitions (§3.9)
│   │   └── registry.ts
│   │
│   ├── tools/                   # Tool definitions (§3.8)
│   │   ├── registry.ts          # Lazy init, per-model filtering, name repair
│   │   └── built-in/
│   │       └── ...
│   │
│   ├── id/                      # Branded ID generation (§3.1)
│   │   └── index.ts             # ses_, msg_, prt_ generators
│   │
│   └── db/                      # Database (§3.6)
│       ├── schema.ts            # Drizzle table definitions
│       ├── client.ts            # bun:sqlite init with pragmas
│       └── migrate.ts           # Migration runner
│
├── migrations/                  # Drizzle migration SQL files
│   └── 0001_initial/
│       └── migration.sql
│
├── tests/
│   ├── core/                    # Agent loop, prompt assembly, context management
│   ├── messages/                # Conversion, filtering, token counting
│   ├── id/                      # ID generation, ordering
│   └── api/                     # HTTP endpoint tests
│
├── CLAUDE.md
├── package.json
├── drizzle.config.ts
└── tsconfig.json
```

---

## 6. Configuration

```typescript
interface NocturnalConfig {
  // Server
  port: number                    // default: 3000
  host: string                    // default: "0.0.0.0"
  
  // Default provider/model (overridable per session)
  defaultProvider: string         // e.g., "anthropic"
  defaultModel: string            // e.g., "claude-sonnet-4-6"
  
  // Provider API keys (loaded from .env by Bun automatically)
  providers: {
    anthropic?: { apiKey: string }
    openai?: { apiKey: string }
    google?: { apiKey: string }
    groq?: { apiKey: string }
    mistral?: { apiKey: string }
    openrouter?: { apiKey: string }
  }
  
  // Context defaults (ADR-002)
  context: ContextThresholds
  
  // Circuit breaker defaults
  limits: CircuitBreakerConfig
  
  // Storage
  storage: {
    type: "sqlite"
    path: string                  // default: "./data/nocturnal.db"
  }
  
  // Agent overrides
  agents?: Record<string, Partial<Agent>>
  
  // Instructions (ADR-001, Layer 4)
  instructions?: string[]         // paths or URLs to instruction files
}
```

---

## 7. What This RFC Does NOT Cover

These are real concerns that will be addressed in future work, not this design:

- **Authentication and authorization** — who can create sessions, access messages
- **Rate limiting** — per-user, per-provider rate management
- **Persistence scaling** — beyond SQLite (Postgres, Redis, etc.)
- **Horizontal scaling** — multiple instances, shared state
- **Eval framework** — how we test prompt architecture quality
- **UI protocol** — beyond SSE streaming format
- **Plugin/extension system** — third-party tools, custom agents (intentionally deferred — see RFC-002 challenge)
- **Observability and tracing** — structured logging, OpenTelemetry
- **Deployment** — hosting, CI/CD, infrastructure
- **Session forking** — branching conversations at a message boundary
- **Event sourcing** — multi-client sync via event tables
- **Dynamic model discovery** — fetching model metadata from external APIs

Each of these deserves its own ADR or RFC when the time comes.

---

## 8. Success Criteria

This RFC is successful when:

1. **A user can send a message and receive a streamed response** through the API
2. **The prompt is assembled from 6 layers** with the correct provider-specific base prompt
3. **Context management fires automatically** — pruning at 75%, compaction at 85%, hard stop at 95% (percentage-based, scaling with model context window)
4. **Tool calls execute and loop correctly** — model calls a tool, ToolPart transitions through pending→running→completed, loop continues
5. **Provider switching works** — change the model in session config, same conversation continues with new provider prompt
6. **Messages and parts are stored with full fidelity** — separate tables, branded IDs, immutable content, mutable part state
7. **Circuit breakers prevent runaway costs** — max turns, max cost, max consecutive tool calls
8. **Per-step cost tracking** — every agent loop step records tokens and cost via StepFinishPart

When all eight criteria are met, Nocturnal is a working agent backend.

---

## References

- ADR-001: Prompt Layering Strategy
- ADR-002: Context Management Strategy
- ADR-003: Provider Abstraction
- ADR-004: Message Format
- RFC-002: RFC Challenge (OpenCode source comparison, 21 findings)
- OpenCode: github.com/anomalyco/opencode — primary reference implementation
  - `packages/opencode/src/session/prompt.ts` — agent loop
  - `packages/opencode/src/session/llm.ts` — LLM calling, option merging
  - `packages/opencode/src/session/system.ts` — provider prompt selection
  - `packages/opencode/src/session/compaction.ts` — pruning and compaction
  - `packages/opencode/src/session/message-v2.ts` — message types, filterCompacted
  - `packages/opencode/src/session/session.sql.ts` — database schema
  - `packages/opencode/src/agent/agent.ts` — agent definitions
  - `packages/opencode/src/id/id.ts` — branded ID generation
- Vercel AI SDK: ai-sdk.dev — provider abstraction and streaming
- Drizzle ORM: orm.drizzle.team — database schema and migrations
- Bun: bun.sh — runtime
- Hono: hono.dev — HTTP framework
