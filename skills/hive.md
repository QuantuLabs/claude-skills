# /hive - Ask the Hivemind

Query multiple AI models and orchestrate consensus through intelligent deliberation.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  YOU (Claude) = Orchestrator                                │
│                                                             │
│  1. Gather context (if codebase question)                   │
│  2. Call hivemind → get raw responses                       │
│  3. Analyze: consensus? divergences?                        │
│  4. If significant divergences:                             │
│     → Formulate targeted questions per model                │
│     → Call hivemind again with queries[]                    │
│  5. Repeat until consensus OR divergences are trivial       │
│  6. Synthesize + add your perspective                       │
└─────────────────────────────────────────────────────────────┘
```

## Orchestration Flow

### Round 1: Initial Query

```typescript
hivemind({
  question: "User's question",
  context: "[raw context if any]"
})
// Returns: { responses: [{ provider, model, content }] }
```

### Analyze Responses

After each round, analyze:
1. **Agreements**: What do all models agree on?
2. **Divergences**: Where do they disagree?
3. **Significance**: Are divergences important or trivial?

**Decision matrix:**
- All agree → Consensus reached, go to synthesis
- Minor divergences (style, wording) → Consensus reached, note the nuances
- Significant divergences → Need another round with targeted questions

### Round N: Targeted Follow-up

When divergences exist, formulate SPECIFIC questions for each model:

```typescript
hivemind({
  question: "Original question",  // Required but queries[] takes precedence
  context: "[same context]",
  queries: [
    {
      provider: "openai",
      question: "You said X, but Gemini argues Y because Z. What's your response to this specific point?"
    },
    {
      provider: "google",
      question: "OpenAI disagrees with your point about W. Can you elaborate on why you believe this?"
    }
  ]
})
```

### When to Stop

Stop iterating when:
- Models reach consensus
- Divergences become trivial (opinion vs fact)
- You've done 3 rounds (diminishing returns)
- The disagreement is fundamental and won't resolve

**Don't waste rounds on:**
- Stylistic differences
- Equally valid approaches
- Matters of opinion

### Synthesis

After consensus (or max rounds):
1. Summarize agreed points
2. Note remaining divergences (if any)
3. Add YOUR perspective as the orchestrator
4. Provide actionable recommendation

## Example Orchestration

```
User: /hive Should I use Redis or Memcached for session caching?

Round 1:
→ Call hivemind with question
→ GPT: "Redis - persistence, data structures"
→ Gemini: "Memcached - simpler, faster for pure caching"

Analysis: Significant divergence on recommendation

Round 2:
→ queries: [
    { provider: "openai", question: "Gemini argues Memcached is faster for pure caching. Do you agree, and if not, why is Redis still better for sessions specifically?" },
    { provider: "google", question: "GPT points out Redis has persistence. Is this important for sessions, or is Memcached's speed advantage more relevant?" }
  ]
→ GPT: "For sessions, persistence matters for user experience during restarts..."
→ Gemini: "Fair point on persistence. For sessions specifically, Redis edge case handling is valuable..."

Analysis: Converging on Redis for sessions, Memcached for pure cache

Synthesis:
- Consensus: Redis for sessions (persistence + data structures)
- Both agree: Memcached better for simple key-value cache
- My take: Redis. Session loss = user frustration. The performance difference is negligible for most apps.
```

## Output Format

```markdown
## Hivemind Consensus

**Question**: [question]
**Rounds**: [N]

### Responses

**GPT-5.2**: [summary]
**Gemini 3 Pro**: [summary]

### Analysis

**Agreements**:
- [point 1]
- [point 2]

**Divergences** (if any):
- [point]: GPT says X, Gemini says Y

### Consensus
[Synthesized answer incorporating all perspectives]

### My Take (Claude)
[Your analysis, where you agree/disagree, final recommendation]
```

## Phase 1: Context Gathering

For codebase questions, gather context BEFORE calling hivemind:

1. Launch Explore agents for relevant areas
2. Read critical source files
3. Pass RAW outputs as context (don't pre-digest)

Skip for general questions (no codebase context needed).

## Arguments

- `$ARGUMENTS` - The question to ask the Hivemind
