# opinionated-prisma

This is an AI coding skill plugin. It contains no application code — only markdown skill files and platform manifests.

## Repository structure

```
skills/                  # Each subdirectory is an independent skill
  opinionated-prisma/    # Overview, prerequisites, skill routing
  schema-design/         # Naming, IDs, enums, soft delete
  indexing/              # Index types, Prisma gaps, N+1
  migration-safety/      # Safe migration workflows
  transactions/          # Transaction types, bulk operations
  raw-sql-boundary/      # When to use $queryRaw
.claude-plugin/          # Claude Code plugin manifest
.codex-plugin/           # Codex CLI plugin manifest
.cursor-plugin/          # Cursor plugin manifest
.opencode/plugins/       # OpenCode plugin loader
```

## When editing skills

- Every skill has a `SKILL.md` with YAML frontmatter (`name`, `description`) and markdown body.
- The `description` field controls when the skill is triggered. Keep trigger keywords specific and accurate.
- Skills are opinionated. Do not add hedging language, "it depends" qualifiers, or alternative approaches. Each skill states a position and enforces it.
- Use "Correct" / "Incorrect" sections with code examples, not prose explanations of tradeoffs.
- Keep examples concrete (real table names, real field names) not abstract (TableA, fieldX).

## When adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with frontmatter.
2. Update the routing table in `skills/opinionated-prisma/SKILL.md`.
3. Bump version in `package.json`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.codex-plugin/plugin.json`, `.cursor-plugin/plugin.json`, and `gemini-extension.json`.
4. Add an entry to `CHANGELOG.md`.

## Versioning

All six manifest files must have the same version string. When bumping, update all of them.
