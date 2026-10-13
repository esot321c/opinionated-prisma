---
name: opinionated-prisma
description: >
  Overview and prerequisites for Prisma ORM + PostgreSQL patterns.
  Use when first setting up a database, establishing ID strategy, or needing
  guidance on which specific skill to use. Triggers on "prisma", "database",
  "postgresql", "postgres".
---

# Prisma + PostgreSQL

Opinionated patterns for building with Prisma on self-hosted PostgreSQL. This plugin
provides separate skills for each concern — use the one that matches your current task.

## Available Skills

| Skill | Use When |
|---|---|
| `schema-design` | Designing models, choosing field types, naming conventions, primary keys, enums, soft delete |
| `indexing` | Adding indexes, diagnosing slow queries, N+1 problems, partial/expression/GiST/BRIN indexes |
| `migration-safety` | Planning or running `prisma migrate`, adding columns to large tables, constraint changes |
| `transactions` | Writing transactions, bulk creates, bulk upserts, choosing interactive vs sequential |
| `raw-sql-boundary` | Deciding whether to use `$queryRaw`, window functions, CTEs, full-text search, JSONB operators |

Invoke the specific skill via the Skill tool when the task narrows to one of these areas.

## Scope

Written for Prisma ORM 7.x on PostgreSQL. Prisma 8 replaces the query and migration APIs; do not apply this guidance to a Prisma 8 project. Where the repository's CLAUDE.md, schema comments or existing migrations set a convention, the repository wins.

## Prisma 7 baseline

- Generator: `provider = "prisma-client"` with a required `output` (e.g. `"../generated/prisma"`). Import `PrismaClient` and the `Prisma` namespace from the generated path (e.g. `../generated/prisma/client`), not `@prisma/client`.
- Connection URLs live in `prisma.config.ts` (`datasource.url`), not in the `datasource` block. The CLI does not load `.env`; load it in `prisma.config.ts`.
- `PrismaClient` requires a driver adapter: `new PrismaClient({ adapter: new PrismaPg({ connectionString }) })` from `@prisma/adapter-pg`.
- `prisma migrate dev` no longer runs `prisma generate` or the seed; run them explicitly.
- `$use` middleware is removed; use `$extends` with `query` hooks.

## Primary keys: UUIDv7

PostgreSQL 18 and later has `uuidv7()` built in:

```prisma
id String @id @default(dbgenerated("uuidv7()")) @db.Uuid
```

PostgreSQL 17 and earlier: `id String @id @default(uuid(7)) @db.Uuid`. Prisma Client generates the value, so raw SQL inserts must supply their own id. If the project already uses `pg_uuidv7` (`uuid_generate_v7()`), keep it.

### Exception: ids derived from their input (UUIDv5)

Use a UUIDv5 (SHA-1 of a namespace UUID plus a name) when an id must be derived from its input, so that a retry or replay names the same row by construction. This applies when the code creating the record cannot check for a duplicate, typically a stateless service with no database to hold a unique constraint. Example: a gateway that accepts orders without writing to the database derives the order id from the account id and the client's idempotency key; the worker that later writes the order rejects a repeated id.

```typescript
import { v5 as uuidv5 } from "uuid";

const ORDER_ID_NAMESPACE = "6f1c2a4e-…"; // a fixed UUID chosen once for this kind of id
const orderId = uuidv5(`${accountId}:${idempotencyKey}`, ORDER_ID_NAMESPACE);
```

- Put the tenant or scope id in the hashed name, so two tenants using the same key cannot collide.
- Generate the id in application code (`v5` from the `uuid` npm package, `uuid.NewSHA1` in Go). Prisma's `uuid()` supports versions 4 and 7 only, and PostgreSQL has no built-in v5 (`uuid_generate_v5()` is in the `uuid-ossp` extension). The column has no default: `id String @id @db.Uuid`.
- v5 ids are not time-ordered, so these inserts lose v7's index locality. Accept that for the rows that need it.

When a database is available, use a v7 id plus a unique constraint on the idempotency or natural key (`@@unique([accountId, idempotencyKey])`), not a v5 id.

## Initial Setup Behavior

1. Read `schema.prisma`, `prisma.config.ts`, the Prisma version in `package.json`, the PostgreSQL version (compose image tag or `SELECT version()`), and the existing migrations.
2. Follow the ID strategy already in the schema. Do not mix strategies.
3. New schema: `uuidv7()` on PostgreSQL 18+, `uuid(7)` otherwise.
4. Never change database infrastructure (images, extensions, init scripts) without asking.
