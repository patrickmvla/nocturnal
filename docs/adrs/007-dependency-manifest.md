# ADR-007: Dependency Manifest

## Status
Accepted

## Date
2026-04-08

## Context

Every dependency is a decision. Every dependency is code we don't control, a version we have to track, a potential breaking change, a supply chain risk, and a bundle size cost. For a project that values understanding every line, each dependency must justify its existence.

This ADR locks in every dependency Nocturnal starts with, the version range, and why it's there. If it's not in this list, it doesn't get installed without a new ADR or an update to this one.

### Guiding Principles

1. **Bun-native first** — if Bun provides it built-in, don't add a dependency (`.env` loading, SQLite, test runner, file I/O)
2. **One job per dependency** — no frameworks that do 10 things when we need 1
3. **Prior art** — prefer what OpenCode uses when it's the right choice (we can study their code)
4. **Pin ranges tightly** — `^` for minor updates on stable packages, exact pins for pre-1.0 or beta packages

## Decision

### Production Dependencies

| Package | Version | Why | ADR |
|---------|---------|-----|-----|
| `hono` | `^4.x` | HTTP framework. Lightweight (14KB), Bun-native, middleware ecosystem. The server runtime. | RFC-001 |
| `ai` | `^6.x` | Vercel AI SDK core. `streamText()`, `generateText()`, tool execution, SSE streaming protocol. The LLM abstraction layer. | ADR-003 |
| `@ai-sdk/anthropic` | `^4.x` | Anthropic provider adapter. Claude models. Primary production provider. | ADR-003 |
| `@ai-sdk/openai` | `^4.x` | OpenAI provider adapter. GPT models. | ADR-003 |
| `@ai-sdk/google` | `^4.x` | Google provider adapter. Gemini models. | ADR-003 |
| `@ai-sdk/groq` | `^1.x` | Groq provider adapter. Fast inference for testing. | ADR-003 |
| `@ai-sdk/mistral` | `^1.x` | Mistral provider adapter. Free tier for evals. | ADR-003 |
| `@ai-sdk/openrouter` | `^0.x` | OpenRouter adapter. One API key, many models for broad eval coverage. | ADR-003 |
| `@anthropic-ai/sdk` | `^1.x` | Native Anthropic SDK. Escape hatch for batch API and bleeding-edge features. | ADR-003 |
| `openai` | `^5.x` | Native OpenAI SDK. Escape hatch for batch API. | ADR-003 |
| `zod` | `^3.25.0` | Schema validation. Dual import: `zod/v3` for AI SDK tools, `zod/v4` for everything else. Ships both versions internally. | ADR-005 |
| `drizzle-orm` | `0.45.x` | ORM for SQLite. Schema-as-code, typed JSON columns, SQL-like queries. Stable line. | ADR-006 |
| `drizzle-zod` | `0.8.x` | Zod schema derivation from Drizzle tables. Bridges ADR-005 and ADR-006. | ADR-005, ADR-006 |

**13 production dependencies.** Each traced to an ADR.

### Development Dependencies

| Package | Version | Why |
|---------|---------|-----|
| `drizzle-kit` | `0.45.x` | Migration generation and schema management. Run under Node if Bun has issues (#5515). |
| `typescript` | `^5.5` | Type checking. Required for Zod v4 compatibility. |
| `@types/bun` | `latest` | Bun type definitions. |

**3 dev dependencies.**

### What We Are NOT Installing

| Package | Why Not |
|---------|---------|
| `dotenv` | Bun loads `.env` automatically |
| `express` / `fastify` | Hono is the server framework (CLAUDE.md) |
| `better-sqlite3` | `bun:sqlite` is built-in and faster (CLAUDE.md) |
| `jest` / `vitest` | `bun test` is built-in |
| `ws` | `WebSocket` is built-in to Bun |
| `node-fetch` | `fetch` is global in Bun |
| `uuid` | Custom branded ID system (RFC-001 §3.1) |
| `lodash` | Utility functions written inline |
| `winston` / `pino` | `console` for now. Structured logging is Phase 6 |
| `langchain` | "Unnecessarily bloated abstraction" — every team that succeeded used thin custom code (research) |
| `zod-to-json-schema` | Zod v4 has built-in `z.toJSONSchema()` |
| `effect` | Steep learning curve, premature for our stage (RFC-002 challenge) |
| `ioredis` | No Redis needed. SQLite is the store |
| `pg` / `postgres` | No Postgres needed. SQLite is the store |

### Dependency Count

```
Production:  13 packages
Development:  3 packages
Total:       16 packages
```

For comparison:
- OpenCode: 80+ direct dependencies (including TUI framework, PTY, tree-sitter, MCP SDK, Effect, etc.)
- A typical Next.js starter: 20+ dependencies before you write a line of code

We're at 16 total. Every one justified. Every one traced to a decision.

## Package.json

```json
{
  "name": "nocturnal",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "bun --hot src/index.ts",
    "start": "bun src/index.ts",
    "test": "bun test",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:studio": "drizzle-kit studio"
  },
  "dependencies": {
    "hono": "^4.7.0",
    "ai": "^6.0.0",
    "@ai-sdk/anthropic": "^4.0.0",
    "@ai-sdk/openai": "^4.0.0",
    "@ai-sdk/google": "^4.0.0",
    "@ai-sdk/groq": "^1.0.0",
    "@ai-sdk/mistral": "^1.0.0",
    "@ai-sdk/openrouter": "^0.1.0",
    "@anthropic-ai/sdk": "^1.0.0",
    "openai": "^5.0.0",
    "zod": "^3.25.0",
    "drizzle-orm": "~0.45.0",
    "drizzle-zod": "~0.8.0"
  },
  "devDependencies": {
    "drizzle-kit": "~0.45.0",
    "typescript": "^5.5.0",
    "@types/bun": "latest"
  }
}
```

Note: Drizzle uses `~` (patch only) instead of `^` (minor) because 0.x versions can ship breaking changes in minor bumps.

## Consequences

### What becomes easier
- **Dependency audits**: One document lists everything, with justification
- **Onboarding**: New contributors see exactly what's used and why
- **Supply chain risk**: 16 packages is a small surface area
- **Bundle size**: Minimal footprint, no bloat

### What becomes harder
- **Adding dependencies**: Requires updating this ADR or writing a new one. This is intentional friction — it forces the question "do we really need this?"
- **Missing utilities**: No lodash, no utility libraries. Write it yourself or justify the dependency

### Update Policy

- **Patch updates** (`~`): Applied freely. Bug fixes and security patches.
- **Minor updates** (`^`): Applied after checking changelogs. Run tests before merging.
- **Major updates**: Requires a new ADR or update to this one. Major versions mean API changes.
- **AI SDK provider packages**: Updated together as a set to avoid version mismatch.
- **Drizzle**: Pinned to stable `~0.45.x`. Moving to 1.0 stable requires an ADR update.
- **Zod**: Stays on `^3.25.x` until AI SDK fully supports v4 (tracked in ADR-005).

## References

- ADR-003: Provider Abstraction (AI SDK + native SDK decisions)
- ADR-005: Zod Version Strategy (dual import, v3.25.x)
- ADR-006: Database Layer (Drizzle stable on bun:sqlite)
- RFC-001: System Architecture (Hono, Bun, branded IDs)
- CLAUDE.md: Bun-native constraints (no express, no better-sqlite3, no dotenv)
- OpenCode `package.json`: 80+ dependencies as counter-example
