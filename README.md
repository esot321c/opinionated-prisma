# Opinionated Prisma Skills

An opinionated AI coding skill plugin for building with Prisma ORM on self-hosted PostgreSQL.

## This Plugin Makes Decisions

This is not a reference guide. It is a set of enforced opinions about how to use Prisma with PostgreSQL. It chooses UUID v7 over v4, strings over database enums, snake_case mapping, and raw SQL only when Prisma changes algorithmic complexity.

**If your project has different needs, fork this plugin rather than using it directly.** The value is in the decisions being made consistently, not in covering every possible approach.

## How It Works

Skills trigger automatically based on what you're doing. When your coding agent sees you working with Prisma schemas, database queries, migrations, or transactions, the relevant skill activates and guides the agent's decisions.

You don't invoke skills manually. Ask your agent to "add a users table" and the schema-design skill kicks in. Ask it to "add an index to a large table" and migration-safety takes over with the safe `CONCURRENTLY` pattern. The agent checks for relevant skills before any database task.

## What's Inside

### Skills

- **opinionated-prisma** — Entry point. Sets the Prisma 7 baseline and the UUID v7 strategy, reads the existing schema, config and migrations, and routes to the appropriate skill for the task at hand.

- **schema-design** — Activates when designing models or adding fields. Enforces snake_case mapping via `@@map`/`@map`, UUID v7 primary keys, `@@index` on every foreign key, string fields over database enums, separate slug fields for URLs, and soft delete via `deletedAt`.

- **indexing** — Activates when adding indexes or diagnosing slow queries. Covers what Prisma handles (B-tree, composite, GIN, GiST, BRIN, Hash) and what requires custom migrations (partial indexes without the preview flag, expression indexes). Addresses N+1 join behavior and when to enable `relationJoins`.

- **migration-safety** — Activates when planning or running `prisma migrate`. Enforces the `--create-only` review workflow, three-step required column additions (nullable, backfill, NOT NULL), `CREATE INDEX CONCURRENTLY` on production tables serving live traffic, and `NOT VALID`/`VALIDATE` for constraints.

- **transactions** — Activates when writing transactions or bulk operations. Chooses interactive vs sequential transactions, enforces explicit timeouts, handles bulk create with relations via transaction loops, and bulk upsert via `INSERT ... ON CONFLICT`.

- **raw-sql-boundary** — Activates when deciding whether to use `$queryRaw`. Draws the line: raw SQL only when Prisma cannot express the operation (window functions, CTEs, full-text search, JSONB operators) or when it changes algorithmic complexity (bulk upsert). Not for marginal speed gains.

## Installation

Installation differs by platform. If you use more than one, install separately for each.

### Claude Code

Register the marketplace:

```
/plugin marketplace add esot321c/opinionated-prisma
```

Install the plugin:

```
/plugin install opinionated-prisma@opinionated-prisma
```

### Codex CLI

Open the plugin search interface:

```
/plugins
```

Search for `opinionated-prisma` and select Install Plugin.

### Gemini CLI

Install the extension:

```
gemini extensions install https://github.com/esot321c/opinionated-prisma
```

Update later:

```
gemini extensions update opinionated-prisma
```

### Cursor

In Cursor Agent chat:

```
/add-plugin opinionated-prisma
```

Or search for "opinionated-prisma" in the plugin marketplace.

### OpenCode

Tell OpenCode:

```
Fetch and follow instructions from https://raw.githubusercontent.com/esot321c/opinionated-prisma/refs/heads/main/.opencode/INSTALL.md
```

### Manual (Any Platform)

Clone into your skills directory:

```bash
git clone https://github.com/esot321c/opinionated-prisma.git ~/.claude/skills/opinionated-prisma
```

### Project-Level

Add to a specific project:

```bash
git clone https://github.com/esot321c/opinionated-prisma.git .claude/skills/opinionated-prisma
```

## Prerequisites

Prisma ORM 7.x. PostgreSQL 18 (built-in `uuidv7()`), or `@default(uuid(7))` on older versions. See the `opinionated-prisma` skill for the Prisma 7 baseline.

## Opinions

This plugin makes these decisions rather than presenting options:

- UUID v7 via `uuidv7()` (PG18) or `uuid(7)`, not v4/CUID/ULID
- String fields for status/category, database enums only for type discriminators
- Slugs for user-facing URLs, UUIDs for internal keys
- `@@map` and `@map` to snake_case on every model and field
- `@@index` on every foreign key (Prisma does not auto-create these)
- Normalize by default, denormalize only with measured justification
- `$queryRaw` only when Prisma changes algorithmic complexity, not for marginal speed
- Three-step migration for adding required columns (nullable, backfill, NOT NULL)
- Interactive transactions when operations depend on each other, sequential when they don't

If any of these don't fit your project, fork and adjust. PRs to make decisions configurable will not be accepted.

## Updating

Plugin updates are generally automatic depending on your platform. To manually update:

- **Claude Code**: `/plugin update opinionated-prisma`
- **Gemini CLI**: `gemini extensions update opinionated-prisma`
- **Manual installs**: `git pull` in the cloned directory

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. The short version: this plugin is opinionated by design. If you disagree with a decision, fork it. PRs that water down specific guidance into "it depends" will be closed.

## License

MIT License — see [LICENSE](LICENSE) for details.
