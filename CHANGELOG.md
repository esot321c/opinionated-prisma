# Changelog

All notable changes to this project will be documented in this file.

## [1.1.0] - 2026-10-13

### Added

- **opinionated-prisma**: Scope section (Prisma ORM 7.x on PostgreSQL; the repository's own conventions win) and a Prisma 7 baseline: `prisma-client` generator with required `output`, connection URLs in `prisma.config.ts`, required driver adapter, `migrate dev` no longer running `generate` or the seed, `$use` replaced by `$extends`
- **schema-design**: map every `DateTime` to `@db.Timestamptz(3)`
- **opinionated-prisma**, **schema-design**: UUIDv5 exception for ids that must be derived from their input (stateless code that cannot check for duplicates); with a database, a v7 id plus a unique constraint on the key
- **transactions**: project-wide `transactionOptions` on `PrismaClient`

### Changed

- UUID v7 on PostgreSQL 17 and earlier now uses Prisma's `@default(uuid(7))` instead of the `pg_uuidv7` extension; existing `pg_uuidv7` projects keep it. The Docker init-script setup is removed
- **indexing**: GIN, GiST, SP-GiST, BRIN and Hash indexes and operator classes are expressed in the schema; partial indexes via the `partialIndexes` preview feature (7.4+) or a custom migration (7.5+ preserves them); `relationJoins` and query-logging guidance updated for Prisma 7
- **migration-safety**: how Prisma 7.4+ applies migration SQL (statements sent separately, `CONCURRENTLY` allowed, migrations no longer atomic); Prisma 7 `migrate diff` flags; down migrations as `down.sql` generated with `migrate diff`; `SET NOT NULL` via a validated `CHECK`; corrected lock and failure descriptions
- **transactions**: bulk raw writes use `$executeRaw`; nested bulk creates run sequentially in the transaction; `createManyAndReturn` for returned rows
- **raw-sql-boundary**: return-type assertions are unchecked (`BigInt`, `Decimal`); injection example uses `$queryRawUnsafe`; JSONB guidance asks for a check of the generated SQL

### Removed

- SessionStart hooks (`hooks/`) for Claude Code and Cursor, and the matching first-message injection in the OpenCode plugin. Skills trigger from their own descriptions

## [1.0.2] - 2026-09-04

### Changed

- UUID v7 guidance updated for PostgreSQL 18: the native `uuidv7()` function is now the primary recommendation; the `pg_uuidv7` extension is documented as the fallback for PostgreSQL 17 and below
- Schema examples across skills switched from `uuid_generate_v7()` to `uuidv7()`
- Initial setup behavior now determines the Postgres major version before choosing the ID default

## [1.0.1] - 2026-05-15

### Fixed

- Remove duplicate `hooks` declaration from Claude Code plugin manifest that caused a load error

## [1.0.0] - 2026-05-15

### Added

- Multi-platform plugin support (Claude Code, Cursor, Codex, Gemini CLI, OpenCode)
- **schema-design** skill: naming conventions, UUID v7 primary keys, enum decisions, soft delete patterns
- **indexing** skill: Prisma indexing gaps, partial/expression/GiST/BRIN indexes, N+1 join behavior
- **migration-safety** skill: safe `prisma migrate` workflows, concurrent index creation, constraint validation
- **transactions** skill: interactive vs sequential transactions, bulk create/upsert/update patterns
- **raw-sql-boundary** skill: decision logic for `$queryRaw` vs Prisma query API
- **opinionated-prisma** skill: overview, prerequisites, UUID v7 setup, skill routing
- Cross-platform session hooks (Windows + Unix)
- Marketplace metadata for plugin installation
- Per-platform installation commands (CLI syntax for Claude Code, Codex, Cursor, Gemini CLI, OpenCode)
- CLAUDE.md with repo instructions for Claude
- CONTRIBUTING.md with contributor guidelines and fork-first philosophy
- CODE_OF_CONDUCT.md (Contributor Covenant v2.0)
- CHANGELOG.md

### Changed

- Restructured from single monolithic skill to six independent skills, each in its own directory
- README rewritten with "How It Works" section, richer skill descriptions, per-platform install commands, and updating instructions
