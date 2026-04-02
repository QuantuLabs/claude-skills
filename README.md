# Claude Skills

Custom skill definitions organized as one folder per skill.

## Structure

Each skill lives in `<name>/` at the repo root and contains:

- `SKILL.md` for the actual skill definition
- `README.md` for a short human-readable summary

## Available Skills

- [`codex-review`](codex-review/): iterative code or plan review using Codex in a consensus loop
- [`gem-review`](gem-review/): iterative code, plan, or codebase review using Gemini CLI in a consensus loop
- [`privilege`](privilege/): static privilege-path analysis for blockchain programs hosted on GitHub

## Notes

- `codex-review` requires [Codex CLI](https://github.com/openai/codex).
- `gem-review` requires [Gemini CLI](https://github.com/google-gemini/gemini-cli) and a configured API key.
- A sample Gemini env file is available at [`config/gemini-cli/.env.example`](config/gemini-cli/.env.example).
- `privilege` expects a GitHub repository URL for an Anchor, Pinocchio, Quasar, native Solana, or Solidity/EVM codebase.

## Layout

```text
codex-review/
  README.md
  SKILL.md
gem-review/
  README.md
  SKILL.md
privilege/
  README.md
  SKILL.md
```

## License

MIT
