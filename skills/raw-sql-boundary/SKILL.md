---
name: raw-sql-boundary
description: >
  Decision logic for when to use $queryRaw vs Prisma's query API. Use when deciding
  whether raw SQL is warranted, writing window functions, CTEs, full-text search,
  JSONB operators, or complex aggregations. Triggers on "$queryRaw", "raw sql",
  "window function", "CTE", "full-text search", "JSONB", "ROW_NUMBER", "RANK",
  "Prisma.sql", "EXPLAIN ANALYZE".
---

# Raw SQL Boundary

## Decision Rule

Use Prisma's query API by default. Reach for `$queryRaw` only when Prisma's generated SQL is measurably worse or cannot express the operation at all. The threshold is not "raw SQL would be faster" (it almost always is, marginally). The threshold is "Prisma cannot do this, or Prisma's approach changes the algorithmic complexity."

## When to Use $queryRaw

| Pattern | Why Prisma Falls Short |
|---|---|
| Window functions (`ROW_NUMBER`, `RANK`, `LAG/LEAD`) | Not expressible in Prisma query API |
| CTEs (recursive or not) | Not supported. Needed for hierarchical data like org charts, matter trees, threaded comments |
| Full-text search with ranking | Prisma's `search` filter has no `ts_rank`, no custom dictionaries, no control over tsvector columns |
| JSONB containment and key-existence operators (`@>`, `?`, `?&`) with GIN indexes | Check the generated SQL with query logging before assuming Prisma's Json filters use the index |
| Bulk upsert (`INSERT ... ON CONFLICT`) | Prisma's `upsert` is one query per row |
| Conditional bulk update (`UPDATE ... CASE`) | Prisma's `updateMany` applies uniform data |
| Aggregation beyond `groupBy` | No HAVING with complex expressions, no rollups, no cube |
| `EXPLAIN ANALYZE` | Cannot run through Prisma API |
| Materialized view creation/refresh | DDL, not a query |

## When NOT to Use $queryRaw

- Simple CRUD (create, findMany, update, delete). Prisma's overhead is negligible and you get type safety.
- Filtered queries with standard operators (equals, contains, gt/lt, in). Prisma generates correct SQL for these.
- Pagination with cursor or offset. Prisma handles this fine.
- Simple includes/joins, especially with `relationJoins` enabled.

If you find yourself writing raw SQL for basic CRUD, the problem is probably a missing index or a schema design issue, not a Prisma limitation.

## Keeping Raw SQL Maintainable

Import `Prisma` (for `Prisma.sql`, `Prisma.join`) from the generated client path.

### Always Type the Return

The type argument is an unchecked assertion. Bigint columns come back as `BigInt` and numeric as `Decimal`. Validate or map rows at the boundary, or use TypedSQL (`prisma/sql/*.sql`).

```typescript
// WRONG: Untyped
const results = await prisma.$queryRaw`SELECT * FROM matters`;

// CORRECT: Typed
interface MatterWithRank {
  id: string;
  title: string;
  rank: bigint; // ROW_NUMBER() returns bigint, which arrives as BigInt
}

const results = await prisma.$queryRaw<MatterWithRank[]>`
  SELECT id, title,
    ROW_NUMBER() OVER (PARTITION BY client_id ORDER BY created_at DESC) as rank
  FROM matters
  WHERE deleted_at IS NULL
`;
```

### Co-locate Raw Queries with Their Related Prisma Model

```
src/
  modules/
    matters/
      matters.service.ts       # Prisma client queries
      matters.queries.ts       # $queryRaw queries for matters
      matters.controller.ts
```

Don't scatter raw SQL across services. Keep it next to the Prisma operations it supplements so the next developer (or Claude) knows where to look.

### Use Prisma.sql for Parameterization

```typescript
// WRONG: String interpolation (SQL injection risk)
const results = await prisma.$queryRawUnsafe(
  `SELECT * FROM matters WHERE client_id = '${clientId}'`
);

// CORRECT: Parameterized
const results = await prisma.$queryRaw<Matter[]>`
  SELECT * FROM matters WHERE client_id = ${clientId}::uuid
`;
```

Prisma's tagged template literal handles parameterization automatically. Inside the `$queryRaw` tag, `${}` is always a bound parameter; do not quote it. Do not use string concatenation or template literals outside the `$queryRaw` tag.

## Correct: CTE for Matter Hierarchy

```typescript
// A matter can have sub-matters (e.g. a corporate reorganization with sub-files)
const hierarchy = await prisma.$queryRaw<MatterNode[]>`
  WITH RECURSIVE matter_tree AS (
    SELECT id, title, parent_id, 0 as depth
    FROM matters
    WHERE id = ${rootMatterId}::uuid

    UNION ALL

    SELECT m.id, m.title, m.parent_id, mt.depth + 1
    FROM matters m
    JOIN matter_tree mt ON m.parent_id = mt.id
    WHERE m.deleted_at IS NULL
  )
  SELECT * FROM matter_tree ORDER BY depth, title
`;
```

## Incorrect: Raw SQL for Something Prisma Handles Fine

```typescript
// No reason for raw SQL here
const clients = await prisma.$queryRaw<Client[]>`
  SELECT * FROM clients
  WHERE deleted_at IS NULL
  ORDER BY created_at DESC
  LIMIT 50
`;

// Prisma does this correctly:
const clients = await prisma.client.findMany({
  where: { deletedAt: null },
  orderBy: { createdAt: 'desc' },
  take: 50
});
```
