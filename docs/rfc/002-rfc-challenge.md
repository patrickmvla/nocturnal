# RFC-001 Challenge: OpenCode Source vs Our Design

## Date
2026-04-07

## Purpose
Line-by-line comparison of OpenCode's actual implementation against RFC-001. Every gap is a question we need to answer: adopt, adapt, or intentionally skip.

---

## CRITICAL GAPS (Architecture-level differences)

### 1. Effect Framework — OpenCode's Entire Runtime

**What OpenCode does**: The entire codebase is built on the `effect` library — algebraic effects for dependency injection, service composition, error handling, and concurrency. Every function returns `Effect.Effect<T, E>`. Services are defined as `ServiceMap.Service`. Dependencies are declared, composed as layers, and injected at runtime.

**What our RFC says**: Plain TypeScript. No mention of Effect or any DI framework.

**The real question**: Effect gives OpenCode testability (swap service implementations), cancellation (abort signals thread through the effect system), and structured error handling (typed errors propagate without try/catch). Do we need this, or is it over-engineering for our stage?

**Recommendation**: Skip for now. Effect has a steep learning curve and adds significant complexity. We can achieve testability with simple dependency injection (pass interfaces to functions). If we outgrow that, Effect is an ADR-level decision later. The goal is to UNDERSTAND the architecture, not cargo-cult every dependency.

---

### 2. Database Schema — Messages and Parts Are Separate Tables

**What OpenCode does**: 
```
message table: id, session_id, time_created, time_updated, data (JSON blob of MessageV2.Info)
part table:    id, message_id, session_id, time_created, time_updated, data (JSON blob of Part)
```

Messages and parts are **separate database rows** linked by `message_id`. Parts can be updated independently without touching the message row. The `data` column is a JSON blob containing the typed content.

**What our RFC says**: `MessageStore` interface with `saveMessage()` that takes a `Message` with embedded `parts[]`. We treat parts as nested within messages, not as independent entities.

**Why this matters**: 
- OpenCode can update a single part's state (e.g., setting `compactedAt`) with a surgical `UPDATE part SET data = ... WHERE id = ?` 
- With embedded parts, updating one part means rewriting the entire message's JSON blob
- At scale (messages with 20+ tool calls), this is the difference between updating 1 row vs rewriting a large JSON document

**Recommendation**: Adopt. Separate `message` and `part` tables. This is the right design for ADR-002's pruning (which mutates part state frequently) and for ADR-004's immutability contract (message content stays untouched, part state changes independently).

---

### 3. ID System — Branded, Ordered, Prefixed

**What OpenCode does**: Custom ID generation with:
- Branded TypeScript types (`SessionID`, `MessageID`, `PartID`) — prevents mixing IDs across entities
- Prefixed (`ses_`, `msg_`, `prt_`) — immediately readable in logs and debugging
- Time-ordered: sessions use **descending** order (newest first in lexicographic sort), messages/parts use **ascending** (oldest first)
- 6-byte timestamp + monotonic counter + 14 bytes random base62

**What our RFC says**: `id: string` with `generateId()`. No branding, no ordering, no prefix.

**Why this matters**: Descending session IDs mean `SELECT * FROM session ORDER BY id` gives newest first without a secondary sort. Ascending message IDs mean `SELECT * FROM message WHERE session_id = ? ORDER BY id` gives chronological order. The IDs ARE the sort order. No indexes needed on timestamps for basic queries.

**Recommendation**: Adopt. Branded IDs prevent bugs (passing a MessageID where a SessionID is expected). Prefixed IDs make debugging trivial. Time-ordered IDs eliminate timestamp indexes. This is cheap to implement and pays off immediately.

---

### 4. Part Types — We're Missing Half

**What OpenCode has (12 part types)**:
| Part Type | Purpose |
|-----------|---------|
| `text` | Model's text response |
| `reasoning` | Thinking blocks with signatures |
| `file` | File attachments (images, PDFs) |
| `tool` | Tool call + result + state lifecycle (pending→running→completed→error) |
| `snapshot` | File system snapshots for revert |
| `patch` | Code diffs/patches |
| `agent` | Subtask delegation to sub-agents |
| `compaction` | Compaction boundary marker |
| `subtask` | Subtask tracking |
| `retry` | Retry markers |
| `step-start` | Step boundary start |
| `step-finish` | Step boundary end with per-step cost/tokens |

**What our RFC has (6 part types)**: `text`, `tool-call`, `tool-result`, `compaction`, `reasoning`, `step-start`

**Key missing types**:
- **`file`**: We have no way to handle image/PDF attachments in messages
- **`tool` (unified)**: OpenCode merges tool-call and tool-result into a single `ToolPart` with a state machine (`pending`→`running`→`completed`|`error`). We split them into separate `ToolCallPart` and `ToolResultPart`. OpenCode's approach is actually cleaner — one part tracks the full lifecycle
- **`step-finish`**: Tracks per-step cost and token usage. Without this, we can't do per-step cost observability
- **`snapshot`/`patch`**: File system state tracking for undo/revert. Not needed for a general agent (this is coding-agent specific)
- **`agent`/`subtask`**: Sub-agent delegation. Premature for our stage but worth knowing about

**Recommendation**: 
- Adopt unified `ToolPart` (merge call + result + state lifecycle into one part type)
- Add `FilePart` for attachments
- Add `StepFinishPart` for per-step cost tracking
- Skip `snapshot`, `patch`, `agent`, `subtask`, `retry` — these are coding-agent concerns, not general-agent architecture

---

### 5. Tool System — Lazy Init, Per-Model Filtering, Custom Tool Loading

**What OpenCode does**:
- 45+ built-in tools across 45 files
- **Lazy initialization**: `Tool.Info = { id, init: (ctx?) => Promise<Def> }` — tools aren't constructed until first use
- **Per-model tool filtering**: GPT models get `apply_patch` instead of `edit`/`write`. Some tools only available for certain providers
- **Custom tool loading**: Users can add tools from `.opencode/tool/` directories
- **Tool description files**: Separate `.txt` files per tool for descriptions (keeps code clean)
- **Protected tools**: Certain tool outputs immune to pruning (e.g., `skill` tool)

**What our RFC says**: Simple tool registry. Tools defined in code. No lazy init, no per-model filtering, no custom loading.

**Recommendation**:
- Adopt lazy initialization (tools can have expensive setup — DB connections, API auth)
- Adopt per-model tool filtering through the agent permission system (ADR-001 Layer 3)
- Skip custom tool loading for now — premature
- Adopt protected tools concept for ADR-002 pruning

---

### 6. Plugin System — Hooks at Every Layer

**What OpenCode does**: Extensive plugin hooks:
- `experimental.chat.system.transform` — modify system prompt array
- `experimental.chat.messages.transform` — modify messages before LLM call
- `chat.params` — modify temperature, topP, maxOutputTokens
- `chat.headers` — inject custom HTTP headers
- `experimental.session.compacting` — inject context during compaction
- Plugin tools wrapped with `Truncate.output()` (auto-truncation of tool outputs)

**What our RFC says**: No plugin system.

**Recommendation**: Skip entirely. Plugins are for extensibility at scale with external contributors. We don't have users yet. Every plugin hook is a maintenance burden and API contract. Build the core, add hooks when there's demand.

---

### 7. Provider Model Discovery — Dynamic, Not Static

**What OpenCode does**: Fetches model metadata from `https://models.dev/api.json` with 5-minute cache TTL. Falls back to bundled snapshot. Models have rich metadata: cost, limits, capabilities, modalities, status (alpha/beta/deprecated).

**What our RFC says**: Static provider/model config in a config file.

**Recommendation**: Start static, add dynamic discovery later. For our stage, we know exactly which models we're testing with. A config file with model names and context window sizes is sufficient. Dynamic discovery is polish.

---

### 8. Session Forking

**What OpenCode does**: `Session.fork()` clones a session at a message boundary, remapping all message/part IDs to fresh ascending IDs while preserving parent-child relationships. The forked session has a `parentID` reference.

**What our RFC says**: No mention of forking.

**Recommendation**: Skip. Useful for branching conversations, but premature. Add when there's a use case.

---

### 9. Event Sourcing / Sync Tables

**What OpenCode does**: `event` and `event_sequence` tables track all state changes as events. Used for syncing state to clients (WebSocket/SSE push of changes).

**What our RFC says**: No event sourcing. Direct SSE streaming of LLM responses only.

**Recommendation**: Skip. Event sourcing is for multi-client sync (TUI + web UI + IDE extension all watching the same session). We're building a backend API — clients poll or subscribe to the streaming endpoint. If we need multi-client sync later, it's additive.

---

### 10. LLM Call Assembly — More Nuanced Than Our RFC

**What OpenCode actually does in `llm.ts`**:

1. System prompt = `[agent.prompt || SystemPrompt.provider(model), ...input.system, ...user.system?].join("\n")`
2. Plugin hook transforms system array
3. If system header unchanged, rejoin into 2-part structure (cache optimization)
4. Merge options: `pipe(base, mergeDeep(model.options), mergeDeep(agent.options), mergeDeep(variant))`
5. OpenAI OAuth: use `instructions` field instead of system messages
6. Non-OpenAI: prepend system as `{role: "system"}` messages
7. Plugin hooks for params and headers
8. Permission-based tool filtering
9. Tool repair function for case-sensitivity
10. Provider middleware wrapper on language model

**What our RFC says**: Simple `assembleSystemPrompt()` → `streamText()`. Missing:
- Option merging (model options + agent options + variant)
- Provider-specific system message handling (some providers use `instructions` field)
- Tool repair for case-sensitivity
- Provider middleware transforms

**Recommendation**: 
- Adopt option merging (model defaults + agent overrides)
- Adopt provider-specific system message handling (not all providers support `{role: "system"}` the same way — AI SDK handles some of this, but edge cases exist)
- Adopt tool repair (models sometimes return tool names with wrong casing — cheap defensive code)
- Skip middleware transforms for now (provider-specific prompt transforms are handled by AI SDK)

---

### 11. Overflow Detection — Their Static Numbers Confirmed

**OpenCode's actual code** (`overflow.ts`):
```typescript
const COMPACTION_BUFFER = 20_000
// ...
const reserved = cfg.compaction?.reserved ?? Math.min(COMPACTION_BUFFER, maxOutputTokens)
const usable = model.limit.input ? model.limit.input - reserved : context - maxOutputTokens
return count >= usable
```

**Confirmed**: Static `20_000` token buffer. No percentage-based scaling. On a 1M context window, this is 2% reserved. On a 128K window, it's 15.6%.

**Our ADR-002 improvement stands**: Percentage-based thresholds that scale with the model's context window. This is a genuine architectural improvement over OpenCode.

---

### 12. Pruning — Their Static Constants Confirmed

**OpenCode's actual code** (`compaction.ts`):
```typescript
const PRUNE_PROTECT = 40_000
const PRUNE_MINIMUM = 20_000
```

Protect last 40K tokens of tool outputs, require at least 20K pruneable before acting.

**Our ADR-002 improvement confirmed**: These should be `model.contextWindow * percentage`, not static numbers.

---

### 13. filterCompacted — Subtlety We Missed

**OpenCode's actual implementation**:
```typescript
function filterCompacted(msgs) {
  const result = []
  const completed = new Set()
  for (const msg of msgs) {
    result.push(msg)
    if (msg.role === "user" && completed.has(msg.id) && msg.parts.some(p => p.type === "compaction"))
      break
    if (msg.role === "assistant" && msg.summary && msg.finish && !msg.error)
      completed.add(msg.parentID)
  }
  result.reverse()
  return result
}
```

**Subtlety**: It doesn't just look for a compaction part — it requires that the compaction SUCCEEDED (assistant with `summary: true`, finished, no error). A failed compaction is invisible to filtering. This prevents a failed compaction attempt from hiding valid history.

**Our RFC**: Simpler version that just looks for `CompactionPart`. Should adopt the success-checking logic.

---

### 14. toModelMessages — Media Handling Per Provider

**What OpenCode does**: Checks `supportsMediaInToolResults` per provider. For providers that don't support media in tool results (most except Anthropic and Google), it extracts image attachments from tool results and injects them as separate user messages with `"Attached image(s) from tool result:"`.

**What our RFC says**: No mention of media handling in conversion.

**Recommendation**: Adopt. This is a real-world concern — if a tool returns screenshots or images, the conversion needs to handle provider differences.

---

## SUMMARY: What to Change in RFC-001

### Must Adopt (architectural impact)
1. **Separate message and part tables** — parts as independent DB rows
2. **Branded, ordered, prefixed IDs** — `ses_`, `msg_`, `prt_` with time-ordering
3. **Unified ToolPart** — merge tool-call and tool-result into single part with state machine
4. **Add FilePart** — for attachments
5. **Add StepFinishPart** — per-step cost/token tracking
6. **filterCompacted success checking** — only honor completed compactions
7. **Protected tools in pruning** — some tool outputs immune to pruning

### Should Adopt (quality impact)
8. **Lazy tool initialization** — tools constructed on first use
9. **Option merging** — model defaults + agent overrides + variant
10. **Tool name repair** — case-insensitive matching fallback
11. **Media handling in toModelMessages** — provider-aware attachment extraction

### Intentionally Skip (premature or coding-agent specific)
12. Effect framework — steep learning curve, plain TS is fine for now
13. Plugin system — no users to extend for yet
14. Session forking — no branching use case
15. Event sourcing — single-client API, not multi-client sync
16. Dynamic model discovery — static config is fine at our scale
17. Custom tool loading from directories — premature
18. Snapshot/patch/revert parts — coding-agent specific
19. Agent/subtask parts — premature multi-agent

### Confirmed Improvements Over OpenCode
20. **Percentage-based context thresholds** (ADR-002) — OpenCode uses static 40K/20K
21. **Percentage-based overflow detection** (ADR-002) — OpenCode uses static 20K buffer
