# ADR-006: Database Layer

## Status
Accepted

## Date
2026-04-08

## Context

Nocturnal stores sessions, messages, and parts in SQLite (decided in RFC-001). The question is what sits between our TypeScript code and the SQLite database: raw `bun:sqlite` (zero dependencies, full control) or Drizzle ORM (schema-as-code, typed queries, migrations).

This decision affects every component. The message store (ADR-004) reads and writes parts. The context manager (ADR-002) does surgical part state updates. The session manager creates and queries sessions. The API layer returns paginated results. Every one of these touches the database layer.

### The CLAUDE.md Constraint

The project's `CLAUDE.md` says: "Use `bun:sqlite` for SQLite. Don't use `better-sqlite3`." This rules out `better-sqlite3` but doesn't prohibit an ORM built on top of `bun:sqlite`. Drizzle's `drizzle-orm/bun-sqlite` driver wraps `bun:sqlite` directly — it's `bun:sqlite` with a typed query builder on top.

### What OpenCode Proves

OpenCode runs Drizzle ORM on `bun:sqlite` in production with 138K+ stars worth of users. Their implementation:

- `drizzle-orm@1.0.0-beta.19` pinned to a specific commit hash
- 6-line database init: create `Database` from `bun:sqlite`, wrap with `drizzle()`
- Schema definitions using `sqliteTable()`, `text()`, `integer()`, `index()`
- `text({ mode: "json" }).$type<T>()` for typed JSON columns — exactly what RFC-001's message/part tables need
- 10 SQL migration files (timestamped directories) applied at startup
- SQLite pragmas: WAL mode, synchronous=NORMAL, 5s busy timeout, 64MB cache, foreign keys ON
- No reported Drizzle-specific issues in their GitHub tracker

### The Schema We Need (RFC-001)

```
session table:   id (ses_), config (JSON), state (JSON), timestamps
message table:   id (msg_), session_id (FK), data (JSON blob of MessageInfo)
part table:      id (prt_), message_id (FK), session_id, data (JSON blob of Part)
```

Key patterns:
- **JSON data columns**: Message and part content stored as JSON blobs. IDs pulled out as indexed columns for relational queries. This is OpenCode's exact pattern.
- **Surgical part updates**: ADR-002's pruning sets `compactedAt` on individual parts. This is `UPDATE part SET data = ? WHERE id = ?` — needs to be fast and targeted.
- **Time-ordered ID queries**: Branded IDs (ADR-004) encode sort order. `SELECT * FROM message WHERE session_id = ? ORDER BY id` gives chronological order without timestamp indexes.

## Decision

Use **Drizzle ORM (stable 0.45.x)** on top of `bun:sqlite` for schema definition, typed queries, and migration management. Use `drizzle-zod@0.8.3` for Zod schema derivation until Drizzle 1.0 stable ships with built-in support.

### Why Drizzle, Not Raw bun:sqlite

**1. Schema-as-code eliminates type drift.**

With raw `bun:sqlite`, the database schema lives in SQL files and the TypeScript types live in `types.ts`. They can drift apart silently. A column rename in SQL that isn't reflected in TypeScript causes runtime failures, not compile errors.

With Drizzle, the schema IS the TypeScript:

```typescript
const partTable = sqliteTable("part", {
  id: text().$type<PartID>().primaryKey(),
  message_id: text().$type<MessageID>().notNull()
    .references(() => messageTable.id, { onDelete: "cascade" }),
  session_id: text().$type<SessionID>().notNull(),
  data: text({ mode: "json" }).notNull().$type<PartData>(),
  time_created: integer().notNull(),
  time_updated: integer().notNull(),
})
```

One definition. The table schema, the TypeScript types, the foreign key relationships, and the JSON column typing all live in the same place. Change one, the compiler catches everywhere it propagates.

**2. JSON column typing is critical for our architecture.**

RFC-001's message/part tables use JSON data columns: `text({ mode: "json" }).$type<T>()`. This is not a nice-to-have — it's how ADR-004's typed parts system works. Drizzle handles JSON serialization/deserialization automatically with type safety. Raw `bun:sqlite` would require manual `JSON.parse()` / `JSON.stringify()` with type assertions on every read/write.

**3. Migrations are a solved problem.**

Drizzle generates timestamped SQL migration files from schema changes. They're plain SQL, reviewable, version-controllable. The runtime `migrate()` function applies them at startup. OpenCode does exactly this.

With raw `bun:sqlite`, we'd build our own migration runner. It's not hard, but it's solved — and migration bugs are subtle and destructive (data loss, schema corruption). Using a proven migration system is the right call for production.

**4. Queries stay close to SQL.**

Drizzle's query API is intentionally SQL-like:

```typescript
// Drizzle
db.select().from(partTable)
  .where(eq(partTable.message_id, messageId))
  .orderBy(partTable.id)
  .all()

// vs raw bun:sqlite
db.query("SELECT * FROM part WHERE message_id = ? ORDER BY id")
  .all(messageId) as PartRow[]
```

The Drizzle version is 3 more characters and gives you: type checking on column names, type checking on the return type, and refactor safety (rename a column, every query that references it gets a compile error). The raw version is a magic string that silently breaks.

**5. OpenCode proves the combo works.**

This isn't theoretical. OpenCode ships Drizzle + `bun:sqlite` to 138K+ users. Same table structure (messages and parts in separate tables with JSON data columns). Same migration pattern. Same SQLite pragmas. They pinned to a beta version and have zero reported Drizzle issues. We're not being brave — we're following proven ground.

### Why Stable 0.45.x, Not 1.0 Beta

OpenCode pins to `1.0.0-beta.19`. We're choosing the stable line instead:

- The 1.0 beta ships breaking changes between beta releases (~20 betas in 3 months)
- We don't need features exclusive to the beta (built-in Zod is the main one, and `drizzle-zod@0.8.3` covers us on stable)
- Stable 0.45.x receives security patches
- If we need to move to 1.0 later, the migration is straightforward — Drizzle's API surface hasn't fundamentally changed

### Known Issues We Accept

**drizzle-kit + Bun**: drizzle-kit commands (generate, migrate, studio) may fail under Bun due to `Could not resolve: "node:sqlite"` (issue #5515). Workaround: run drizzle-kit under Node for schema generation, use Bun for runtime. This only affects development workflow, not production.

**Async transaction rollback**: SQLite is synchronous. Drizzle's async wrapper can't properly rollback if an async operation fails mid-transaction. Workaround: keep transaction callbacks synchronous (OpenCode enforces this with a `NotPromise<T>` constraint).

**1,673 open issues**: Large backlog. SQLite is second-class to PostgreSQL in Drizzle's attention. Mitigation: we're using simple patterns (select, insert, update with JSON columns) — not relational queries, not edge-case features.

### SQLite Configuration

Following OpenCode's production-proven pragmas:

```sql
PRAGMA journal_mode = WAL          -- Write-Ahead Logging for concurrent reads
PRAGMA synchronous = NORMAL        -- Balance durability and performance
PRAGMA busy_timeout = 5000         -- 5s wait on lock contention
PRAGMA cache_size = -64000         -- 64MB page cache
PRAGMA foreign_keys = ON           -- Enforce referential integrity
PRAGMA wal_checkpoint(PASSIVE)     -- Non-blocking checkpoint on open
```

## Options Considered

### Option A: Raw bun:sqlite
- **Pros:** Zero dependencies. Maximum performance (3-6x faster than better-sqlite3, already built into Bun). Full control over every query. No ORM abstraction to debug through. No version stability concerns. Matches CLAUDE.md's "use bun:sqlite" directive literally.
- **Cons:** Manual type safety on every query (type assertions, `JSON.parse()` casts). No schema-as-code — SQL and TypeScript types can drift. DIY migration system. Every query is a magic string that can silently break on schema changes. More code to write and maintain for the same functionality.

### Option B: Drizzle ORM stable (0.45.x) + drizzle-zod — SELECTED
- **Pros:** Schema-as-code with typed JSON columns. Migration generation and runtime application. SQL-like query API that stays close to the database. Production-proven on `bun:sqlite` by OpenCode. `drizzle-zod` for Zod schema derivation. Stable release line with security patches.
- **Cons:** Never hit 1.0 stable. drizzle-kit has Bun-specific issues (run under Node). 1,673 open issues. SQLite is second-class to PostgreSQL. Adds a dependency. `drizzle-zod` is technically deprecated (replaced by built-in in beta).

### Option C: Drizzle ORM beta (1.0.0-beta.x)
- **Pros:** Built-in Zod support via `drizzle-orm/zod`. Latest features. What OpenCode uses.
- **Cons:** Breaking changes between beta releases. ~20 betas in 3 months. Pinning to a specific commit hash (like OpenCode does) is fragile. For a project that values understanding every line, using an unstable dependency adds noise.

### Option D: Kysely
- **Pros:** Type-safe query builder (not ORM). Simpler, more stable API than Drizzle. Respected in the TypeScript community.
- **Cons:** `bun:sqlite` support via community packages only (not first-party). No schema-as-code (you define TypeScript interfaces manually). No Zod integration. Less adoption than Drizzle. Migration system is simpler but less featured. We can't study OpenCode's implementation patterns directly.

## Consequences

### What becomes easier
- **ADR-004 message types**: `text({ mode: "json" }).$type<PartData>()` gives typed JSON columns with automatic serialization
- **ADR-002 part updates**: Surgical `UPDATE part SET data = ? WHERE id = ?` is a typed Drizzle operation, not a raw SQL string
- **Schema evolution**: `drizzle-kit generate` creates reviewable SQL migration files
- **Refactoring**: Rename a column → compiler errors on every query that references it
- **Studying OpenCode**: Same ORM, same patterns, direct comparison

### What becomes harder
- **Development setup**: drizzle-kit may need Node for schema generation (Bun issue #5515)
- **Debugging**: ORM abstraction between our code and SQL. When queries behave unexpectedly, need to check both Drizzle and SQLite
- **Dependency management**: Another versioned dependency to track and update

### Relationship to Other ADRs

**ADR-004 (Message Format)**: Drizzle's `text({ mode: "json" }).$type<T>()` is the implementation of RFC-001's "JSON data columns" pattern. The typed JSON column IS how MessageInfo and Part data get stored.

**ADR-005 (Zod Version)**: `drizzle-zod@0.8.3` supports both Zod 3.25+ and Zod 4.0+. Since we're on `zod@3.25.x` with dual imports, this works. When we move to Drizzle 1.0 stable, the built-in `drizzle-orm/zod` takes over.

## References

- OpenCode database implementation: `packages/opencode/src/storage/db.ts`, `db.bun.ts`
- OpenCode schema: `packages/opencode/src/session/session.sql.ts`
- Drizzle ORM docs: orm.drizzle.team
- Drizzle + bun:sqlite: orm.drizzle.team/docs/get-started/bun-new
- drizzle-kit Bun issue: drizzle-team/drizzle-orm #5515
- Bun SQLite docs: bun.sh/docs/api/sqlite
- Drizzle GitHub: 33.7K stars, 0.45.x stable, 1.0.0-beta.20
