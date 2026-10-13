---
name: indexing
description: >
  PostgreSQL indexing patterns and Prisma's limitations. Use when adding indexes,
  diagnosing slow queries, dealing with N+1 problems, or needing partial, expression,
  GiST, or BRIN indexes. Triggers on "index", "@@index", "performance", "slow query",
  "EXPLAIN", "N+1", "partial index", "GIN", "GiST", "BRIN".
---

# Indexing & Query Performance

## Rules

- Always add `@@index` on foreign key fields. Prisma does not create these automatically.
- Run `EXPLAIN ANALYZE` on any query that touches more than ~1000 rows or joins 3+ tables before assuming Prisma's generated SQL is fine.
- Enable query logging in development: `new PrismaClient({ adapter, log: [{ emit: "event", level: "query" }] })` then `prisma.$on("query", e => …)`. Read the SQL Prisma actually generates.

## What Prisma Handles Well

- B-tree indexes via `@@index` and `@@unique`.
- Composite indexes via `@@index([fieldA, fieldB])`.
- Access methods via `@@index([field], type: Gin | Gist | SpGist | Brin | Hash)`, with operator classes via `ops:`.
- Partial indexes via `where:` (Prisma 7.4+, behind the `partialIndexes` preview feature). Without the flag, write them in a custom migration and use Prisma 7.5 or later, so `migrate dev` does not drop them as drift.

## What Prisma Cannot Express (Use Custom Migrations)

### Partial Indexes

For tables with soft deletes or status-filtered queries where most reads exclude a subset of rows.

```sql
-- In a custom migration (prisma migrate --create-only, then edit the SQL)
CREATE INDEX idx_clients_email_active ON clients(email) WHERE deleted_at IS NULL;
```

Without this, every query filtering `WHERE deleted_at IS NULL` scans deleted rows too.

### Expression Indexes

For case-insensitive lookups or computed values.

```sql
CREATE INDEX idx_clients_email_lower ON clients(LOWER(email));
```

Prisma has no way to express `LOWER()` in an `@@index`. Without this, case-insensitive email lookups do a sequential scan or rely on `ILIKE` which cannot use a standard B-tree index.

### GiST and BRIN

Both are schema-level when they index plain columns:

```prisma
@@index([ipAddress(ops: InetOps)], type: Gist)   // ipAddress String @db.Inet
@@index([createdAt], type: Brin)   // large append-only tables
```

Expression-based GiST indexes still need a custom migration:

```sql
CREATE INDEX idx_matters_date_range ON matters USING gist(tsrange(start_date, end_date));
```

## N+1 and Join Behavior

Without the `relationJoins` preview feature (still Preview in 7.x), `include` sends one query per relation level. With it, `relationLoadStrategy` defaults to `join` (LATERAL joins with JSON aggregation on PostgreSQL).

```typescript
// This generates 1 query for clients + 1 query for matters + 1 query for documents
const clients = await prisma.client.findMany({
  include: { matters: { include: { documents: true } } }
});
```

For hot paths, enable `relationJoins` in the Prisma preview features to use lateral joins instead. Verify it is active in your project before assuming nested includes are single-query.

If `relationJoins` is unavailable or insufficient, use `$queryRaw` with an explicit JOIN:

```typescript
const results = await prisma.$queryRaw<ClientWithMatters[]>`
  SELECT c.*, json_agg(m.*) as matters
  FROM clients c
  LEFT JOIN matters m ON m.client_id = c.id
  WHERE c.deleted_at IS NULL
  GROUP BY c.id
  ORDER BY c.created_at DESC
  LIMIT 50
`;
```

## Correct

```prisma
model Matter {
  id        String    @id @default(dbgenerated("uuidv7()")) @db.Uuid
  clientId  String    @map("client_id") @db.Uuid
  client    Client    @relation(fields: [clientId], references: [id])
  status    String    @default("open")
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz(3)

  @@index([clientId])           // explicit FK index
  @@index([status, createdAt])  // composite for filtered + sorted queries
  @@map("matters")
}
```

Plus a custom migration for the partial index:

```sql
CREATE INDEX idx_matters_active_by_client ON matters(client_id, created_at)
  WHERE deleted_at IS NULL;
```

## Incorrect

```prisma
model Matter {
  id        String   @id @default(uuid())
  clientId  String   @map("client_id") @db.Uuid
  client    Client   @relation(fields: [clientId], references: [id])
  status    String   @default("open")
  createdAt DateTime @default(now()) @map("created_at")

  // no @@index on clientId, no composite index
  // no custom migration for partial index on soft delete
  @@map("matters")
}
```

Every `findMany({ where: { clientId } })` does a sequential scan. Filtered queries on active records scan the full table including soft-deleted rows.
