---
name: schema-design
description: >
  Prisma schema design conventions: naming, primary keys, enums, relations, soft delete.
  Use when designing models, adding fields, choosing between enums and strings, or
  setting up a new Prisma schema. Triggers on "schema", "model", "relation", "enum",
  "field", "prisma.schema", "@@map", "@map", "uuid", "slug".
---

# Schema Design Conventions

## Rules

- Always `@@map` models to snake_case table names, `@map` fields to snake_case columns.
- Always add `@@index` on foreign key fields. Prisma does not create these automatically.
- Use `@default(dbgenerated("uuidv7()"))` with `@db.Uuid` on PostgreSQL 18+ (`@default(uuid(7))` on older versions, or `dbgenerated("gen_random_uuid()")` (v4) when rows are also inserted outside Prisma and need a database default). Do not use `@default(uuid())`, which generates v4 (random, poor index locality). Do not use CUID or ULID as primary keys (string storage, 26-37 bytes vs 16 bytes for native uuid, no Postgres uuid operations).
- Exception: when an id must be derived from its input so a retry names the same row, and the code creating it has no database to enforce a unique constraint, use a UUIDv5 generated in application code from the tenant or scope id plus the key, with no column default (see the `opinionated-prisma` skill). With a database, use a v7 id plus `@@unique` on the key instead.
- For user-facing URLs, add a separate `slug String @unique` field. The primary key is an internal concern; display identifiers are a separate field.
- Auto-increment integer IDs are acceptable only for high-write append-only tables (logs, events) where the 8-byte saving matters at scale.
- Prefer `String` with application-level validation over Prisma/Postgres enums for most status and category fields. Database enums require a migration to add values; string fields don't.
- Use a proper relation instead of `Json` type unless the structure is genuinely dynamic and unqueried (e.g. raw webhook payloads, third-party metadata blobs).
- Every table gets `createdAt` and `updatedAt` with `@map` to snake_case.
- Map every `DateTime` to `@db.Timestamptz(3)`. Prisma's PostgreSQL default is `timestamp(3)` without a time zone.
- Soft delete uses `deletedAt DateTime?` not a boolean flag.

## When to Use Database Enums

Use database enums when:
- The value is a type discriminator (e.g. `entryType` where INVOICE requires a `lineItems` relation and RECEIPT requires an `attachments` relation).
- The set of values is genuinely fixed and structural.

Use string fields when:
- Status flows that evolve with business logic.
- Categories the client might rename or extend.
- Any value where adding an option shouldn't require a deploy.

## Correct

```prisma
model Client {
  id        String    @id @default(dbgenerated("uuidv7()")) @db.Uuid
  email     String    @unique
  slug      String    @unique // "parr-business-law"
  name      String
  matters   Matter[]
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz(3)

  @@map("clients")
}

model Matter {
  id        String    @id @default(dbgenerated("uuidv7()")) @db.Uuid
  clientId  String    @map("client_id") @db.Uuid
  client    Client    @relation(fields: [clientId], references: [id])
  title     String
  status    String    @default("open")
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz(3)

  @@index([clientId])
  @@map("matters")
}
```

## Incorrect

```prisma
model matter {
  id        String   @id @default(uuid())
  client_id String
  client    Client   @relation(fields: [client_id], references: [id])
  data      Json     // storing structured matter details as JSON blob
  status    String   @default("open") // stringly-typed with no app-level validation
  createdAt DateTime @default(now())
  // no @@map, no @@index on FK, no updatedAt, uuid v4
}

// Database enum for values that will change with business requirements
enum MatterStatus {
  OPEN
  CLOSED
  ON_HOLD
  // Need to add UNDER_REVIEW? That's a migration and deploy.
}
```

Problems: no `@@map` so Prisma uses model name as table name with inconsistent casing. No `@@index` on FK, so lookups by `clientId` and cascading deletes scan the table (PostgreSQL does not index referencing columns). `uuid()` gives v4 (random, poor index locality). `Json` field for data that should be relational. Missing `updatedAt`. Database enum for a status that will evolve.
