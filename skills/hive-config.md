# /hive-config - Configure Hivemind Settings

Configure Hivemind settings like grounding search and Claude Code mode.

## Usage

Use the `configure_hive` tool from the hivemind-mcp server to update settings, or `check_status` to view current configuration.

## Examples

### Enable Claude Code mode (skip Anthropic API)
```
configure_hive({ claudeCodeMode: true })
```

### Disable Claude Code mode (use all 3 providers)
```
configure_hive({ claudeCodeMode: false })
```

### Enable grounding search (default)
```
configure_hive({ useGrounding: true })
```

### Disable grounding search
```
configure_hive({ useGrounding: false })
```

### Check current settings
```
check_status()
```

## Settings

- **claudeCodeMode** (boolean, default: true): Skip Anthropic API calls when running inside Claude Code. When enabled, only OpenAI and Google are queried since Claude is already the host. This saves API costs and avoids redundant calls.

- **useGrounding** (boolean, default: true): Enable Google Search grounding for enhanced answers with real-time web data. When enabled, Gemini will use Google Search to ground its responses with current information.

## Arguments

- `$ARGUMENTS` - Optional: "status" to show current config, or settings to change (e.g., "claude off" to disable Claude Code mode)
