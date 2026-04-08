# Nocturnal

## What This Is

A production-grade general-purpose agent backend built to deeply understand prompt architecture and context degradation. The architecture is modeled after OpenCode (anomalyco/opencode) — a production coding agent with 138K+ GitHub stars — studied line by line, then reimplemented with deliberate improvements.

Not a toy. Not a tutorial. Not a wrapper. Every decision is documented. Every line is understood.

## Current Status

All architecture and documentation is complete. Phase 1 (first message in, first response out) is next.

- 7 ADRs accepted (docs/adrs/)
- System architecture RFC revised after challenging against OpenCode's actual source (docs/rfc/)
- 7-phase implementation plan ready (docs/implementation/plan.md)
- Repo live at patrickmvla/nocturnal

## How We Work

**Patrick must understand every line.** After completing each phase, do not move to the next until he confirms he understands everything — the logic, the data flow, every line of code, and why it's written that way. The more questions he asks, the better. If he isn't asking questions, stop and say: "wtf are you doing man — ask me something, we don't move on until this clicks."

**ADRs are the law.** Implementation follows the ADR decisions. If something needs to change, update the ADR first with evidence. Never silently deviate.

**No rushing.** If it needs research, do the research. If it needs an ADR, write the ADR. The goal is understanding, not speed.

**Documentation must have teeth.** Real evidence — specific data points, issue numbers, benchmarks, known problems. Not summaries. Not generic descriptions. The first draft of ADR-001 was rejected because it read like a book report. That is the bar.

**Branch per phase.** Never commit directly to main.

## Before Writing Code

Read the architecture docs first. They are the source of truth:

1. `docs/vision/problem-statement.md` — what Nocturnal is and why it exists
2. `docs/adrs/001-prompt-layering-strategy.md` — 6-layer prompt stack modeled after OpenCode
3. `docs/adrs/002-context-management-strategy.md` — 3-tier context management with percentage-based thresholds
4. `docs/adrs/003-provider-abstraction.md` — AI SDK as primary, native SDKs as escape hatches
5. `docs/adrs/004-message-format.md` — typed parts system with unified ToolPart
6. `docs/adrs/005-zod-version.md` — dual import strategy (v3 for AI SDK, v4 for everything else)
7. `docs/adrs/006-database-layer.md` — Drizzle ORM stable on bun:sqlite
8. `docs/adrs/007-dependency-manifest.md` — 16 dependencies, every one justified
9. `docs/rfc/001-system-architecture.md` — complete system design
10. `docs/rfc/002-rfc-challenge.md` — 21 findings from OpenCode source comparison
11. `docs/implementation/plan.md` — 7 phases, horizontal slices

## Reference Implementation

OpenCode (github.com/anomalyco/opencode, branch: dev) is the primary reference. Key files:

- `packages/opencode/src/session/prompt.ts` — agent loop
- `packages/opencode/src/session/llm.ts` — LLM calling and system prompt assembly
- `packages/opencode/src/session/system.ts` — provider prompt selection
- `packages/opencode/src/session/compaction.ts` — pruning and compaction
- `packages/opencode/src/session/message-v2.ts` — message types and filterCompacted
- `packages/opencode/src/session/overflow.ts` — overflow detection

Fork at patrickmvla/opencode. Cloned locally at `/home/mvula/audhd/opencode` (dev branch, shallow clone). Use `Read` tool to study source files directly.

## Architecture Summary

**Prompt layering (ADR-001):** 6 layers assembled at call time — provider base prompt, environment context, tool listing, user instructions, agent override, per-message injections. Each provider gets a tuned prompt because models think differently.

**Context management (ADR-002):** Three tiers — prune tool outputs first (free, mechanical), then LLM compaction (expensive, semantic), then hard stop. All thresholds are percentage-based relative to the model's context window. This is a deliberate improvement over OpenCode's static 40K/20K constants.

**Provider abstraction (ADR-003):** AI SDK v6 for streaming and tool calling unification. Native Anthropic and OpenAI SDKs as escape hatches for batch API and features AI SDK hasn't modeled. Free-tier providers (Groq, Google AI Studio, Mistral, OpenRouter) for evals.

**Message format (ADR-004):** Typed parts — text, tool (unified with state machine), file, reasoning, compaction, step-start, step-finish, retry. Messages and parts stored in separate database tables. Branded IDs with prefixes (ses_, msg_, prt_) and time-ordering baked into the ID.

**Zod (ADR-005):** zod@^3.25.0 with dual imports. `zod/v3` at the AI SDK boundary only (tool schemas). `zod/v4` everywhere else. AI SDK + Zod v4 has unresolved compatibility issues — empty input_schema on tool calls.

**Database (ADR-006):** Drizzle ORM stable (0.45.x) on bun:sqlite. JSON data columns with TypeScript typing. Separate message and part tables so pruning can surgically update a single part row without rewriting the entire message blob.

**Dependencies (ADR-007):** 13 production, 3 dev. Every one traced to an ADR.

## Research

Extensive research is in docs/vision/:

- `production-problems-research.md` — failure data from enterprise and academic sources
- `startup-pain-research.md` — raw complaints from Reddit, HN, Indie Hackers, founder postmortems
- `reference-library.md` — Lost in the Middle, Chroma Context Rot, provider prompt guides, production architecture patterns
- `provider-abstraction-research.md` — AI SDK vs native SDKs with feature gap analysis

## Tech Stack

| Component | Choice | Version |
|-----------|--------|---------|
| Runtime | Bun | latest |
| Server | Hono | ^4.x |
| LLM | Vercel AI SDK + native SDKs | ai@^6.x |
| Database | Drizzle ORM + bun:sqlite | drizzle-orm@~0.45.x |
| Validation | Zod (dual v3/v4) | zod@^3.25.0 |
| Language | TypeScript | ^5.5 |

## Bun Conventions

- `bun <file>` not `node`
- `bun test` not jest/vitest
- `bun install` not npm/yarn/pnpm
- `bun run <script>` not npm run
- Bun loads `.env` automatically — no dotenv
- `bun:sqlite` for SQLite — no better-sqlite3
- `WebSocket` is built-in — no ws
- `fetch` is global — no node-fetch

For more information, read the Bun API docs in `node_modules/bun-types/docs/**.md`.
