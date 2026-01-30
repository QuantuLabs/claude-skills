# Claude Skills

Custom skills for [Claude Code](https://claude.ai/claude-code) - Anthropic's official CLI for Claude.

## Installation

Copy skills to your Claude Code commands directory:

```bash
cp skills/*.md ~/.claude/commands/
```

Or symlink for auto-updates:

```bash
ln -sf $(pwd)/skills/*.md ~/.claude/commands/
```

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [Codex Review](skills/codex-review.md) | `/codex-review` | Iterative code/plan review with OpenAI Codex CLI |
| [Gemini Review](skills/gem-review.md) | `/gem-review` | Iterative code/plan review with Gemini CLI |
| [Hivemind](skills/hive.md) | `/hive` | Multi-AI consensus orchestration (GPT + Gemini) |
| [Hive Config](skills/hive-config.md) | `/hive-config` | Configure Hivemind settings |

## Multi-AI Review Skills

The review skills (`codex-review`, `gem-review`) enable Claude to collaborate with other AI models for:

- **Code Review**: Analyze uncommitted changes for bugs, type issues, error handling
- **Plan Review**: Review architecture docs for gaps, risks, inconsistencies
- **Consensus Loop**: Iterate until Claude and the external AI agree

```
┌─────────────────────────────────────────────────────────────┐
│  Claude + External AI = Collaborative Review                │
│                                                             │
│  1. Detect mode (code vs plan)                              │
│  2. Run external AI review                                  │
│  3. Parse findings                                          │
│  4. For each finding:                                       │
│     → Analyze: Is it valid? False positive?                 │
│     → If valid: Apply fix (code) or note (plan)             │
│     → If false positive: Document reasoning                 │
│  5. CONSENSUS STEP: Send analysis back                      │
│  6. Repeat until BOTH agree OR max 3 rounds                 │
│  7. Report final consensus status                           │
└─────────────────────────────────────────────────────────────┘
```

## Hivemind Skill

The `/hive` skill queries multiple AI models and orchestrates consensus:

```
┌─────────────────────────────────────────────────────────────┐
│  Claude = Orchestrator                                      │
│                                                             │
│  1. Gather context (if codebase question)                   │
│  2. Call hivemind → get raw responses                       │
│  3. Analyze: consensus? divergences?                        │
│  4. If significant divergences:                             │
│     → Formulate targeted questions per model                │
│     → Call hivemind again with queries[]                    │
│  5. Repeat until consensus OR divergences are trivial       │
│  6. Synthesize + add Claude's perspective                   │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

### For Codex Review

Install [Codex CLI](https://github.com/openai/codex):
```bash
npm install -g @openai/codex
```

### For Gemini Review

Install [Gemini CLI](https://github.com/google-gemini/gemini-cli):
```bash
npm install -g @google/gemini-cli
```

Configure API key:
```bash
echo 'GEMINI_API_KEY=your-key' > ~/.gemini/.env
```

### For Hivemind

Requires the [hivemind-mcp](https://github.com/your-org/hivemind-mcp) server configured in Claude Code.

## License

MIT
