# Implementation Plan: Nocturnal

## Date
2026-04-08

## Approach
Horizontal slices — each phase delivers a working system with more capability. We can test, deploy, and validate at every phase boundary.

---

## Phase 1: First Message In, First Response Out

**Goal**: Send a message to a single provider, get a streamed response back. No tools, no context management, no multi-provider. Just the core data flow working end to end.

**What gets built**:
- [ ] Project scaffolding (Bun, Hono, Drizzle, TypeScript config)
- [ ] ID system — branded `ses_`, `msg_`, `prt_` generators with time-ordering
- [ ] Database schema — session, message, part tables with Drizzle + bun:sqlite
- [ ] Message types — `TextPart`, `ReasoningPart` only (enough for basic conversation)
- [ ] Session manager — create, get, list sessions
- [ ] Prompt assembler — Layer 1 (single provider prompt) + Layer 2 (environment block) only
- [ ] LLM integration — AI SDK `streamText()` with one provider (Anthropic or Groq for free testing)
- [ ] SSE streaming — Hono endpoint that streams text-delta events
- [ ] API routes — `POST /sessions`, `POST /sessions/:id/messages`, `GET /sessions/:id/messages`
- [ ] Basic health endpoint

**What it proves**: The data flow works. Message goes in, gets stored with branded IDs in separate tables, prompt is assembled, model responds, response streams back via SSE, assistant message is stored.

**What it does NOT have**: Tools, context management, multi-provider, agent loop (it's a single LLM call, not a loop).

---

## Phase 2: Tool Calling Loop

**Goal**: The agent can call tools, get results, and keep going. This is where it becomes an agent, not a chatbot.

**What gets built**:
- [ ] Unified `ToolPart` with state machine (pending → running → completed | error)
- [ ] Tool system — `ToolInfo` with lazy init, `ToolDef` with Zod schema + execute
- [ ] Tool registry — register, resolve per agent
- [ ] A few built-in tools to test with (e.g., a simple calculator, echo, or mock tools)
- [ ] Agent loop — the `while(true)` loop: call LLM → if tool calls → execute → store ToolPart → loop back → if stop → break
- [ ] `StepStartPart` and `StepFinishPart` — per-step cost tracking
- [ ] Tool name repair — case-insensitive fallback
- [ ] Circuit breakers — max turns, max consecutive tool calls
- [ ] `toModelMessages()` — basic conversion (ToolPart → model format, pending tools → "[interrupted]")
- [ ] Updated SSE events — tool-call, tool-status, tool-result, step-finish

**What it proves**: The agentic loop works. Model calls a tool, tool executes, result goes back, model continues or stops. Circuit breakers prevent runaway loops. Every step tracks cost.

**What it does NOT have**: Context management (conversations can overflow), multi-provider prompts, user instructions.

---

## Phase 3: Context Management

**Goal**: Long conversations don't break. Pruning, compaction, and overflow detection work automatically with percentage-based thresholds.

**What gets built**:
- [ ] Token counting/estimation module
- [ ] `filterCompacted()` — with success-checking (only honor completed compactions)
- [ ] Tier 1: Prune — walk backwards, set `compactedAt` on old ToolParts, skip protected tools
- [ ] `toModelMessages()` updated — compacted ToolParts → "[Old tool result content cleared]"
- [ ] Tier 2: Compact — compaction agent, structured summary (Goal/Instructions/Discoveries/Accomplished/Files), `CompactionPart` insertion
- [ ] Tier 3: Hard stop — `ContextOverflowError`
- [ ] Percentage-based thresholds — `pruneAt`, `protectionWindow`, `compactAt`, `hardStopAt` all relative to model context window
- [ ] Protected tools — configurable exclusion list for pruning
- [ ] Compaction agent definition — `compaction.txt` prompt, no tools
- [ ] Post-loop prune pass
- [ ] Session config for threshold overrides

**What it proves**: Context management works. A long conversation with many tool calls prunes old outputs, compacts when needed, and hard-stops on overflow. Thresholds scale correctly across different model context windows.

**What it does NOT have**: Multi-provider prompts, user instructions, file attachments.

---

## Phase 4: Multi-Provider Prompt Architecture

**Goal**: Full 6-layer prompt assembly. Switch providers and the agent gets the right prompt. The prompt architecture from ADR-001 is fully operational.

**What gets built**:
- [ ] Provider base prompts — `anthropic.txt`, `openai.txt`, `gemini.txt`, `default.txt`
- [ ] Provider prompt selection — model ID string matching (like OpenCode's `system.ts`)
- [ ] Layer 3: Tool/skill listing in system prompt
- [ ] Layer 4: User instruction discovery — walk up directory tree, global instructions, config-specified files
- [ ] Layer 6: Per-message injections — reminders, max-step warnings
- [ ] Option merging — model defaults + agent overrides
- [ ] Provider-specific system message handling
- [ ] 2-part cache split — stable header + dynamic tail
- [ ] Multiple AI SDK providers configured — `@ai-sdk/anthropic`, `@ai-sdk/openai`, `@ai-sdk/google`, `@ai-sdk/groq`
- [ ] `GET /config/providers` endpoint
- [ ] `FilePart` support — image/PDF attachments in messages
- [ ] Media handling in `toModelMessages()` — provider-aware attachment extraction
- [ ] `RetryPart` for retry tracking

**What it proves**: The full prompt architecture works. Each provider gets tuned prompts. Switching models mid-session uses the right prompt. Cache optimization is in place. User instructions are discovered and injected.

---

## Phase 5: Agents and Specialization

**Goal**: Multiple agents with different personalities, permissions, and tool sets. Not just the default agent.

**What gets built**:
- [ ] Agent registry with permission system (`allow`, `deny` per tool)
- [ ] Agent-level prompt overrides (Layer 5 replacing Layer 1)
- [ ] Agent-level option overrides (temperature, topP)
- [ ] Per-model tool filtering
- [ ] Explorer agent — read-only tools, exploration prompt
- [ ] Title agent — auto-generate session titles
- [ ] `PATCH /sessions/:id` — change agent mid-session
- [ ] `GET /config/agents` endpoint

**What it proves**: Agents are real specializations, not just prompt swaps. The explorer can't write files. The compaction agent can't use tools. The default agent has full access. Permissions are enforced at the tool resolution layer.

---

## Phase 6: Hardening and Observability

**Goal**: Production-grade reliability. Cost tracking, error handling, session lifecycle, observability.

**What gets built**:
- [ ] Per-session cost tracking (accumulated from StepFinishParts)
- [ ] Max cost circuit breaker enforcement
- [ ] Structured error handling — typed errors for overflow, tool failure, provider errors
- [ ] Session lifecycle — pause, resume, completed, error states
- [ ] Request validation with Zod on all API endpoints
- [ ] Graceful shutdown — abort in-flight streams on server stop
- [ ] Database migrations runner
- [ ] Structured logging
- [ ] `DELETE /sessions/:id` with cascade cleanup
- [ ] Provider health checks — `GET /health/providers`

**What it proves**: The system handles failures gracefully. Cost limits are enforced. Sessions have proper lifecycle. The database is migrateable. Errors are typed and informative.

---

## Phase 7: Eval Foundation

**Goal**: Prove the prompt architecture works with measurable evidence.

**What gets built**:
- [ ] Eval runner — send scripted conversations, measure outcomes
- [ ] Context degradation eval — does the model lose instructions over long conversations?
- [ ] Provider comparison eval — same conversation across different models, compare quality
- [ ] Compaction quality eval — does the summary preserve critical information?
- [ ] Prune impact eval — does pruning degrade response quality?
- [ ] Cost efficiency eval — tokens used per conversation across threshold configs
- [ ] Results output in structured format (JSON, not vibes)

**What it proves**: The architecture holds up under measurement. Not "it seems to work" — "here are the numbers."

---

## Build Principles

1. **Test at every phase boundary** — before moving to the next phase, the current one passes its "what it proves" criteria
2. **Read every line** — no copy-paste. If it goes in, you understand why it's there
3. **Branch per phase** — each phase is a feature branch, merged when complete
4. **ADRs are the law** — implementation follows the ADR decisions. If something needs to change, update the ADR first
5. **Minimal viable at each phase** — don't gold-plate Phase 1 with Phase 4 concerns

---

## Dependencies

```
Phase 1: No dependencies (start here)
Phase 2: Depends on Phase 1 (needs message storage, LLM call)
Phase 3: Depends on Phase 2 (needs tool parts, agent loop)
Phase 4: Depends on Phase 1 (prompt assembly), can partially parallel with Phase 3
Phase 5: Depends on Phase 4 (needs full prompt architecture)
Phase 6: Depends on Phase 2-5 (hardens everything built so far)
Phase 7: Depends on Phase 1-6 (evaluates the complete system)
```

```
Phase 1 ──→ Phase 2 ──→ Phase 3 ──→ Phase 5 ──→ Phase 6 ──→ Phase 7
                 │                      ↑
                 └──→ Phase 4 ──────────┘
```

Phases 3 and 4 can overlap — context management and prompt architecture are independent concerns that come together in Phase 5.
