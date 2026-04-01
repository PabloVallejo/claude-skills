# Code Indexer

Scans any codebase and produces a comprehensive `codebase-map.json` — a machine-readable index of your project's structure, tech stack, data models, API surface, conventions, and more.

## Why

AI coding assistants spend significant time re-scanning the same codebase every conversation. The code indexer generates a persistent map that can be loaded instantly, skipping redundant exploration and giving the assistant deep context from the start.

## Install

```bash
cp -r skills/code-indexer ~/.claude/skills/code-indexer
```

## Usage

In Claude Code, say any of:

- "index the codebase"
- "map the codebase"
- "generate codebase index"
- "refresh the codebase map"

Or use the slash command:

```
/code-indexer
```

## What It Produces

A single JSON file at `.code-indexer/codebase-map.json` (automatically gitignored) containing:

```json
{
  "generated_at": "2026-04-01T19:00:00Z",
  "generated_from_commit": { "my-repo": "abc1234" },
  "project_type": "single",
  "repos": {
    "my-repo": {
      "tech_stack": { "language": "ruby 3.4", "framework": "rails 8.1", ... },
      "commands": { "test": "bundle exec rspec", "lint": "rubocop", ... },
      "models": [ { "name": "User", "fields": [...], "associations": [...] } ],
      "api_surface": { "rest": [...], "graphql_mutations": [...] },
      "conventions": { "naming": "snake_case", "error_handling": "..." },
      "code_samples": { "mutation": "...", "model": "..." },
      "linter_rules": { ... },
      "db_schema": { ... },
      "integrations": [ ... ],
      "test_infrastructure": { ... },
      "key_files": [ ... ],
      ...
    }
  }
}
```

### Key Sections

| Section | What It Captures |
|---------|-----------------|
| `tech_stack` | Language, framework, database, key dependencies |
| `commands` | Dev, test, lint, build, format commands |
| `models` | Data models with fields, associations, validations |
| `api_surface` | REST endpoints, GraphQL queries/mutations with full signatures |
| `conventions` | Naming, error handling, auth, testing patterns |
| `code_samples` | Copy-paste templates for common patterns |
| `linter_rules` | Non-default rules that differ from framework defaults |
| `model_navigation` | Relationship traversal chains (e.g., `order.customer.address`) |
| `db_schema` | Tables, columns, indexes, foreign keys |
| `integrations` | Third-party services with env vars needed |
| `test_infrastructure` | Test framework, factories, helpers, mocking patterns |

## How It Works

1. Spawns 4 parallel agents, each scanning a focus area (tech, architecture, quality, concerns)
2. Each agent reads actual files (not guessing) and writes a partial JSON map
3. Results are merged into `codebase-map.json`

The map auto-detects staleness: if fewer than 11 structural files changed since the last index, it reports the map is fresh and offers to skip.

## Multi-Repo Support

Works with git submodule setups. Each repo is scanned independently and the results are merged into a single map under `repos: { repo1: {...}, repo2: {...} }`.

## Refreshing

Run the skill again anytime. It will check staleness and only re-index if needed. To force a full refresh, tell Claude "force refresh the codebase index".

## Using the Map in Other Skills

Reference `.code-indexer/codebase-map.json` in your own skills or CLAUDE.md:

```markdown
## Context
Before starting work, check if `.code-indexer/codebase-map.json` exists.
If it does, read it for instant codebase context instead of scanning files.
```
