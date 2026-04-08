# ADR-005: Zod Version Strategy

## Status
Accepted

## Date
2026-04-08

## Context

Zod is the schema validation library for the entire Nocturnal stack. It defines message types (ADR-004), tool input schemas (AI SDK tool calling), API request validation, database serialization shapes, and configuration validation. Every component touches Zod. Getting the version wrong means either a painful migration later or fighting compatibility bugs now.

Zod v4 shipped mid-2025 with massive improvements: 14.7x faster string parsing, 2.3x smaller bundle (5.36 KB vs 12.47 KB), recursive types without `z.lazy()`, built-in JSON Schema generation, and cleaner APIs. The breaking changes are significant — `.merge()` removed, `.strict()` replaced with `z.strictObject()`, string validators moved to top-level (`z.email()` instead of `z.string().email()`), error customization completely reworked, `.deepPartial()` removed entirely.

### The AI SDK Compatibility Problem

This is the decision's load-bearing constraint. ADR-003 commits us to Vercel AI SDK for tool calling. AI SDK uses Zod schemas to define tool inputs:

```typescript
tools: {
  weather: tool({
    inputSchema: z.object({ location: z.string() }),
    execute: async ({ location }) => ({ temperature: 72 }),
  }),
}
```

**AI SDK + Zod v4 is not reliably production-safe as of April 2026.**

Evidence:
- **Issue #7291** (vercel/ai): Initial Zod v4 support request. Added in AI SDK v5 beta.19 via PR #7298, but issues persisted.
- **Issue #10014** (vercel/ai): `generateObject` failures with Zod v4.1.12+. Users report schemas serialize incorrectly.
- **Issue #12020** (vercel/ai, April 2026): Tool `input_schema` arrives **empty** when using Anthropic provider. Affects Zod v3 and v4, but indicates the integration is still rough.
- **Root cause**: AI SDK internally depends on `zod-to-json-schema` which expects Zod v3 internals (`ZodFirstPartyTypeKind`). Zod v4 has built-in `.toJSONSchema()` making this unnecessary, but AI SDK hasn't fully migrated.
- **Community reports**: Some users report success after clearing `node_modules`; others cannot get it working at all. Unreliable.

If we go pure Zod v4 and tool schemas break silently (empty `input_schema` means the model gets no schema = hallucinated tool arguments), the agentic loop from ADR-002 falls apart. This is not an acceptable risk.

### The Drizzle Compatibility Situation

Drizzle ORM also uses Zod for schema generation:
- `drizzle-zod` v0.8.1+ claims Zod v4 support, but users report `Missing './v4' specifier` errors
- Starting from Drizzle ORM v0.42, `drizzle-zod` is deprecated in favor of built-in `drizzle-orm/zod`
- For new projects: use `drizzle-orm/zod` directly, which handles the version internally

### Zod's Built-in Migration Path

Zod v3.25.x ships BOTH v3 and v4 code internally, accessible via subpath imports:

```typescript
import * as z3 from "zod/v3"   // Zod v3 API (stable)
import * as z4 from "zod/v4"   // Zod v4 API (new)
```

This is intentional. The Zod team designed this so libraries can gradually migrate:
- One dependency (`zod@^3.25.0`) gives you both versions
- A codemod exists: `npx @zod/codemod --transform v3-to-v4 ./src`
- When ready to commit to v4: bump to `zod@^4.0.0`, update imports, done

## Decision

Install `zod@^3.25.0` and use **dual imports**:

```typescript
// For AI SDK tool schemas — must use v3 until AI SDK fully supports v4
import { z } from "zod/v3"

// For everything else — message types, config, API validation, DB schemas
import { z } from "zod/v4"
```

### What Uses v3 (AI SDK boundary only)
- Tool input schema definitions passed to AI SDK's `tool()` function
- Any schema that flows through `streamText()` / `generateText()` tool parameters
- Structured output schemas if using AI SDK's `output` parameter

### What Uses v4 (Everything else)
- Message types (`MessageInfo`, `MessagePart` discriminated unions)
- Session config validation
- API request/response validation
- Database serialization schemas
- Configuration file parsing
- Any internal validation not touching AI SDK

### Why Not Just Stay on v3 Everywhere

Zod v4's improvements are too significant to ignore for a new project:

| Metric | v3 | v4 | Impact |
|--------|----|----|--------|
| String parsing | 363 μs/iter | 24,674 ns/iter | **14.7x faster** |
| Object parsing | 805 μs/iter | 124 μs/iter | **6.5x faster** |
| Bundle size | 12.47 KB | 5.36 KB | **2.3x smaller** |
| TS instantiations | baseline | 100x reduction | **Faster type checking** |

For message validation running on every part of every message in a potentially long conversation, 6.5x faster object parsing is material. For a system that estimates tokens by parsing message structures (ADR-002), parse speed matters.

Additionally, v4's recursive types without `z.lazy()` is directly useful for nested message structures, and built-in `z.toJSONSchema()` eliminates the `zod-to-json-schema` dependency.

### Migration Trigger

When AI SDK fully supports Zod v4 (tracked via vercel/ai issues):
1. Bump to `zod@^4.0.0`
2. Run `npx @zod/codemod --transform v3-to-v4 ./src`
3. Replace all `zod/v3` imports with `zod` (which is v4 in `zod@^4.0.0`)
4. Remove dual-import pattern
5. Test all tool calling end-to-end

This should be a single PR with no architectural changes — the schemas are the same shapes, just different import paths.

## Options Considered

### Option A: Pure Zod v3
- **Pros:** Maximum compatibility. AI SDK works. Drizzle works. All examples and docs reference v3.
- **Cons:** We write a new project against old APIs that are already deprecated. 14.7x slower parsing. 2.3x larger bundle. Need `z.lazy()` for recursive types. Need `zod-to-json-schema` for JSON Schema generation. We'll have to migrate later anyway — every library is moving to v4.

### Option B: Pure Zod v4
- **Pros:** Best performance. Best API. Best bundle size. No migration needed later.
- **Cons:** AI SDK tool calling may break silently (empty `input_schema`). This is a showstopper — the entire agentic loop depends on working tool schemas. We'd be gambling the core architecture on an unresolved compatibility issue.

### Option C: Dual Import (v3 for AI SDK, v4 for everything else) — SELECTED
- **Pros:** AI SDK compatibility preserved. v4 benefits for all internal code. Clear boundary (AI SDK boundary = v3, everything else = v4). Built-in migration path when AI SDK catches up. One dependency, two import paths. Codemod available for final migration.
- **Cons:** Two import conventions to maintain. Contributors must know which `z` to use where. Risk of accidentally using v4 in an AI SDK context (caught at runtime, but annoying). Slightly larger bundle since both versions are included in `zod@3.25.x`.

### Option D: Wait for AI SDK v4 support, then decide
- **Pros:** No risk. Perfect information.
- **Cons:** Blocks Phase 1 indefinitely. We can't start building without schema validation. The dual-import strategy gives us the same result without waiting.

## Consequences

### What becomes easier
- **Internal validation**: 6.5x faster object parsing for message types, config, API validation
- **Recursive types**: No `z.lazy()` wrappers for nested message structures
- **JSON Schema**: Built-in `z.toJSONSchema()` for API documentation and tool definitions (non-AI-SDK contexts)
- **Future migration**: When AI SDK supports v4, it's a codemod + import change, not a rewrite

### What becomes harder
- **Developer discipline**: Must use `zod/v3` for AI SDK tool schemas, `zod/v4` everywhere else. Need lint rules or conventions to enforce this boundary
- **Two mental models**: v3 and v4 APIs differ significantly. Working near the AI SDK boundary means knowing both
- **Testing**: Tool schemas need testing with AI SDK specifically to catch any serialization issues early

### Convention

```
src/tools/**/*.ts        → import { z } from "zod/v3"    (AI SDK boundary)
src/**/*.ts (everything else) → import { z } from "zod/v4"
```

A lint rule or code comment convention enforces this. The boundary is small — only tool definition files import v3.

## References

- Zod v4 release notes: zod.dev/v4
- Zod v4 migration guide: zod.dev/v4/changelog
- Zod v4 versioning strategy (dual-import design): zod.dev/v4/versioning
- AI SDK + Zod v4 issues: vercel/ai #7291, #10014, #12020
- AI SDK Zod v4 discussion: vercel/ai #7289
- Drizzle + Zod v4: drizzle-team/drizzle-orm #4406, #4820
- Zod v4 performance benchmarks: zod.dev/v4 (official)
- Zod v4 controversy (bundling v4 inside v3): colinhacks/zod #4923
