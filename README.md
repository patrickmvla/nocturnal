# Nocturnal

A production-grade general-purpose agent backend built to understand prompt architecture and context degradation at a deep level.

## Why

The market created "AI Engineer" as a role but most practitioners lack understanding of the two hardest problems in LLM-powered systems: how to structure prompts so models behave correctly across providers, and how to manage context as conversations grow. Nocturnal is a deliberate, rigorous study of both — implemented as a real system, not a tutorial.

## Architecture

Every decision is documented with evidence:

- **7 ADRs** covering prompt layering, context management, provider abstraction, message format, Zod versioning, database layer, and dependency manifest
- **System architecture RFC** challenged against OpenCode's actual source code (21 findings)
- **7-phase implementation plan** delivering working software at each boundary

See `docs/` for the full paper trail.

## Reference

Architecture modeled after [OpenCode](https://github.com/anomalyco/opencode) (138K+ stars) — studied line by line, reimplemented with deliberate improvements including percentage-based context thresholds that scale with model context windows.

## Stack

| Component | Choice |
|-----------|--------|
| Runtime | Bun |
| Server | Hono |
| LLM | Vercel AI SDK v6 + native SDKs |
| Database | Drizzle ORM + bun:sqlite |
| Validation | Zod 3.25 (dual v3/v4 import) |
| Language | TypeScript |

## Setup

```bash
bun install
bun run dev
```

## Status

Pre-implementation. Architecture complete. Phase 1 next.
