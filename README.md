# Claude Skills

A collection of custom skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — Anthropic's official AI coding assistant CLI.

## What are Skills?

Skills are markdown files that extend Claude Code with new slash commands. They provide structured instructions that Claude follows to perform specialized tasks.

## Quick Start

```bash
# Clone the repo
git clone https://github.com/QuantuLabs/claude-skills.git

# Copy skills to Claude Code
cp claude-skills/skills/*.md ~/.claude/commands/

# Or symlink for auto-updates
ln -sf $(pwd)/claude-skills/skills/*.md ~/.claude/commands/
```

Then in Claude Code, type `/codex-review` or `/gem-review` to use them.

## Available Skills

### Multi-AI Review

| Command | Description | Requires |
|---------|-------------|----------|
| `/codex-review` | Iterative code/plan review with OpenAI Codex | [Codex CLI](https://github.com/openai/codex) |
| `/gem-review` | Iterative code/plan review with Gemini CLI | [Gemini CLI](https://github.com/google-gemini/gemini-cli) |

These skills enable Claude to collaborate with external AI models in a consensus loop:

```
┌────────────────────────────────────────────────────────┐
│  Claude + External AI = Collaborative Review           │
│                                                        │
│  1. Run external AI review (code or plan)              │
│  2. Claude analyzes each finding                       │
│  3. Valid issues → Fix immediately                     │
│  4. False positives → Document reasoning               │
│  5. Send analysis back for consensus                   │
│  6. Iterate until both AIs agree (max 3 rounds)        │
└────────────────────────────────────────────────────────┘
```

**Supported modes:**
- **Code Review**: Analyze git uncommitted changes for bugs, type issues, error handling
- **Plan Review**: Review architecture docs for gaps, risks, inconsistencies
- **Codebase Analysis**: Deep architectural review (gem-review only)

## Prerequisites

### Codex CLI (for `/codex-review`)

```bash
npm install -g @openai/codex
```

### Gemini CLI (for `/gem-review`)

```bash
npm install -g @google/gemini-cli

# Configure API key
mkdir -p ~/.gemini
echo 'GEMINI_API_KEY=your-key' > ~/.gemini/.env
```

Get your API key: https://aistudio.google.com/app/apikey

## Creating Your Own Skills

Skills are markdown files in `~/.claude/commands/`. Structure:

```markdown
# /command-name - Short Description

Instructions for Claude to follow when this command is invoked.

## Arguments

- `$ARGUMENTS` - User input passed to the command
```

See the [skills/](skills/) directory for examples.

## Related Projects

- [Hivemind MCP](https://github.com/QuantuLabs/hivemind-mcp) — Multi-AI consensus orchestration server

## Contributing

PRs welcome! To add a new skill:

1. Create `skills/your-skill.md`
2. Follow the existing skill format
3. Update this README
4. Submit a PR

## License

MIT — see [LICENSE](LICENSE)
