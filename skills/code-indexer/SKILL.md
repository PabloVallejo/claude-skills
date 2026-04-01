# Code Indexer: Codebase Mapping Skill

Scans any codebase and produces a comprehensive `codebase-map.json` — a machine-readable index of project structure, tech stack, data models, API surface, conventions, CI commands, and more.

The map is useful as persistent context for AI coding assistants. Generate it once, refresh when the codebase changes significantly, and reference it in future conversations to skip redundant codebase scanning.

Trigger on: "code-indexer", "index the codebase", "map the codebase", "refresh the codebase map", "generate codebase index".

---

## How It Works

1. Spawns parallel mapper agents (up to 4) that each scan a focus area of the codebase
2. Each agent writes a partial map covering its focus area
3. Results are merged into a single `codebase-map.json`
4. The map is stored in `.code-indexer/codebase-map.json` (gitignored by default)

### Focus Areas

| Agent | Scans | Sections |
|-------|-------|----------|
| Tech | Dependencies, frameworks, infra, env vars | `tech_stack`, `commands`, `integrations`, `env_vars` |
| Architecture | Directory structure, models, API surface, data flow | `directory_structure`, `models`, `api_surface`, `model_navigation`, `db_schema` |
| Quality | Code style, naming, linting rules, conventions | `conventions`, `linter_rules`, `code_samples` |
| Concerns | Test infra, recent activity, view structure | `test_infrastructure`, `view_structure`, `recent_commits`, `key_files` |

---

## Prerequisites

- Git repository with at least one commit
- Claude Code CLI

---

## Process

### Step 1: Setup

Ensure output directory exists and is gitignored:

```bash
mkdir -p .code-indexer
grep -qxF '.code-indexer/' .gitignore 2>/dev/null || echo '.code-indexer/' >> .gitignore
```

### Step 2: Detect Repo Structure

```bash
if [ -f .gitmodules ] || git submodule status 2>/dev/null | grep -q '^'; then
  REPO_TYPE="multi"
  REPOS=$(git submodule status | awk '{print $2}')
else
  REPO_TYPE="single"
  REPOS="."
fi
```

### Step 3: Check Freshness (Skip If Fresh)

If a map already exists, check if it's stale:

```bash
if [ -f .code-indexer/codebase-map.json ]; then
  # Get the indexed commit SHA
  INDEXED_SHA=$(python3 -c "
import json
d = json.load(open('.code-indexer/codebase-map.json'))
commits = d.get('generated_from_commit', {})
print(list(commits.values())[0] if commits else '')
" 2>/dev/null)

  if [ -n "$INDEXED_SHA" ]; then
    CHANGES=$(git diff --name-only "$INDEXED_SHA"..HEAD -- \
      '*.rb' '*.swift' '*.ts' '*.tsx' '*.py' '*.go' '*.rs' '*.java' \
      'package.json' 'Gemfile' 'Cargo.toml' 'go.mod' \
      2>/dev/null | wc -l | tr -d ' ')
    echo "Changes since last index: $CHANGES"
  else
    CHANGES=999
  fi
else
  CHANGES=999
fi
```

| Condition | Action |
|---|---|
| No map exists | Run full index |
| 0-10 changes | Map is fresh — tell user, offer to force refresh |
| 11+ changes | Map is stale — run full index |

If map is fresh, ask the user:
> Codebase map is up to date (only {N} file changes since last index). Want to force a full refresh anyway?

### Step 4: Spawn Parallel Mapper Agents

Read the mapper prompt from the `prompts/` directory relative to this skill:

```
~/.claude/skills/code-indexer/prompts/mapper.md
```

Spawn **4 parallel agents**, each with the mapper prompt prepended with their focus area and output path:

**Agent 1: Tech**
```
Agent(
  description="Index codebase: tech stack",
  subagent_type="general-purpose",
  prompt="<mapper prompt contents>

CONTEXT FOR THIS RUN:
- Focus: tech
- Project root: {absolute path}
- Repo to scan: {repo_name} at {repo_path}
- Write output to: {PROJECT_ROOT}/.code-indexer/partial-tech.json
- Sections to produce: tech_stack, commands, integrations, env_vars

Scan the codebase for your focus area and write your sections as a JSON object."
)
```

**Agent 2: Architecture**
```
Focus: architecture
Sections: directory_structure, models, api_surface, model_navigation, db_schema
Output: .code-indexer/partial-architecture.json
```

**Agent 3: Quality**
```
Focus: quality
Sections: conventions, linter_rules, code_samples
Output: .code-indexer/partial-quality.json
```

**Agent 4: Concerns**
```
Focus: concerns
Sections: test_infrastructure, view_structure, recent_commits, key_files, services, jobs
Output: .code-indexer/partial-concerns.json
```

For **multi-repo** setups, spawn one set of 4 agents per repo (they don't conflict since they write to different files).

### Step 5: Merge Results

After all agents complete, merge partial maps into one:

```bash
python3 -c "
import json, glob, os
from datetime import datetime, timezone

partials = sorted(glob.glob('.code-indexer/partial-*.json'))
merged_repo = {}
for f in partials:
    with open(f) as fh:
        merged_repo.update(json.load(fh))

# For single repo, wrap in repos structure
result = {
    'generated_at': datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'),
    'generated_from_commit': {'repo': '$(git rev-parse HEAD)'},
    'project_type': 'single',
    'repos': {
        os.path.basename(os.getcwd()): merged_repo
    }
}

with open('.code-indexer/codebase-map.json', 'w') as fh:
    json.dump(result, fh, indent=2)

print('Merged', len(partials), 'partial maps')
"

# Clean up partials
rm -f .code-indexer/partial-*.json
```

### Step 6: Report

```
Codebase map generated at .code-indexer/codebase-map.json

Indexed: {N} repos, {M} models, {P} mutations, {Q} queries, {R} services
Map size: {file size}

The map can be referenced in future conversations for instant codebase context.
To refresh: run /code-indexer again.
```

---

## Output Structure

The map is stored at `.code-indexer/codebase-map.json`. See `prompts/mapper.md` for the full JSON schema.

Key sections per repo:
- `tech_stack` — language, framework, database, dependencies
- `commands` — dev, test, lint, build, format commands
- `directory_structure` — directory purposes
- `models` — data models with fields, associations, notes
- `api_surface` — REST endpoints, GraphQL queries/mutations/types
- `services` / `jobs` — business logic and background processing
- `conventions` — naming, error handling, auth, testing patterns
- `linter_rules` — non-default linter/formatter rules
- `code_samples` — copy-paste templates for common patterns
- `model_navigation` — relationship traversal chains
- `db_schema` — tables, columns, indexes
- `integrations` — third-party services with env vars
- `test_infrastructure` — test framework, factories, helpers
- `key_files` — frequently referenced files catalog
- `env_vars` — all environment variables with purposes
- `recent_commits` — snapshot of recent work

---

## File Layout

```
.code-indexer/
└── codebase-map.json    # Generated index (gitignored)
```

Skill files (installed to `~/.claude/skills/code-indexer/`):
```
code-indexer/
├── SKILL.md             # This file (orchestration)
└── prompts/
    └── mapper.md        # Agent prompt for scanning
```
