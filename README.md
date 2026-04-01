# Claude Skills

Reusable skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — install, configure, and share.

## Available Skills

| Skill | Description |
|-------|-------------|
| [code-indexer](skills/code-indexer/) | Scans any codebase and produces a comprehensive `codebase-map.json` for persistent AI context |

## Installation

Each skill is a self-contained directory that you copy to `~/.claude/skills/`.

```bash
# Clone the repo
git clone https://github.com/PabloVallejo/claude-skills.git

# Install a specific skill
cp -r claude-skills/skills/code-indexer ~/.claude/skills/code-indexer
```

Or install directly without cloning:

```bash
# Download and install a single skill
mkdir -p ~/.claude/skills/code-indexer
curl -sL https://raw.githubusercontent.com/PabloVallejo/claude-skills/main/skills/code-indexer/SKILL.md -o ~/.claude/skills/code-indexer/SKILL.md
mkdir -p ~/.claude/skills/code-indexer/prompts
curl -sL https://raw.githubusercontent.com/PabloVallejo/claude-skills/main/skills/code-indexer/prompts/mapper.md -o ~/.claude/skills/code-indexer/prompts/mapper.md
```

After installing, the skill is available in Claude Code via its trigger phrases.

## Skill Structure

Each skill follows this layout:

```
skill-name/
├── SKILL.md          # Main skill definition (orchestration logic)
├── prompts/          # Agent prompts used by the skill
│   └── *.md
└── README.md         # Usage docs (optional)
```

- **SKILL.md** — The entry point. Claude Code reads this to understand what the skill does, when to trigger it, and how to execute it.
- **prompts/** — Specialized prompts for sub-agents spawned by the skill.

## Creating Your Own Skill

1. Create a directory under `~/.claude/skills/your-skill-name/`
2. Add a `SKILL.md` with:
   - A description of what it does
   - Trigger phrases (when Claude should activate it)
   - Step-by-step execution process
3. Add any agent prompts to `prompts/`
4. Test it by using the trigger phrases in Claude Code

## Contributing

PRs welcome. Each skill should:
- Be self-contained (no external dependencies beyond Claude Code)
- Include clear trigger phrases
- Document what it produces and where
- Work on any codebase (not project-specific)

## License

MIT
