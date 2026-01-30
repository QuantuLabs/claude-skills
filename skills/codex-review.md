# /codex-review - Iterative Review with Codex

Run Codex review on code changes OR architecture plans/documents. Analyze findings, reach consensus, and iterate until both agree.

## Mode Detection

**Automatically detect review type based on $ARGUMENTS:**

1. **Code Review Mode** (default): When $ARGUMENTS mentions code changes, uncommitted files, or is empty
   - Reviews git uncommitted changes
   - Focuses on bugs, type issues, error handling

2. **Plan Review Mode**: When $ARGUMENTS contains path to `.md` files OR mentions "plan", "architecture", "spec", "design"
   - Reviews documentation/specifications
   - Focuses on gaps, risks, inconsistencies, missing elements

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Claude + Codex = Collaborative Review                      │
│                                                             │
│  1. Detect mode (code vs plan)                              │
│  2. Run appropriate Codex review                            │
│  3. Parse findings                                          │
│  4. For each finding:                                       │
│     → Analyze: Is it valid? False positive?                 │
│     → If valid: Apply fix (code) or note for user (plan)    │
│     → If false positive: Note reasoning                     │
│  5. CONSENSUS STEP: Send analysis back to Codex             │
│  6. Repeat until BOTH agree OR max 3 rounds                 │
│  7. Report final consensus status                           │
└─────────────────────────────────────────────────────────────┘
```

---

## CODE REVIEW MODE

### Step 1: Initial Codex Review (Code)

```bash
codex exec "Review the uncommitted changes in this git repo. Check for:
- Type mismatches (nullable used as non-nullable)
- Missing null checks
- BigInt precision issues
- Error handling gaps
- Interface/schema sync issues

List each issue as: file:line - severity (HIGH/MEDIUM/LOW) - description

Context: $ARGUMENTS" 2>&1
```

### Step 2: Analyze & Fix

For each finding:
1. **Read the code** at the specified file:line
2. **Evaluate validity**: Is it real or false positive?
3. **Decision matrix**:
   - Valid HIGH/MEDIUM → Fix immediately
   - Valid LOW → Fix if simple
   - False positive → Document reasoning

### Step 3: Apply Fixes

1. Read the relevant code section
2. Apply the minimal fix
3. Verify the fix compiles (run build)

---

## PLAN REVIEW MODE

### Step 1: Initial Codex Review (Plan)

First, read the plan file(s) specified in $ARGUMENTS, then:

```bash
codex exec "Review this architecture/plan document for:

## Technical Gaps
- Missing error handling strategies
- Undefined failure modes
- Scalability concerns
- Security vulnerabilities
- Performance bottlenecks

## Consistency Issues
- Contradictions between sections
- Undefined terms or concepts referenced elsewhere
- Mismatched numbers/estimates

## Missing Elements
- Rollback/migration strategies
- Monitoring/observability
- Cost edge cases
- Compliance/legal considerations

## Feasibility Concerns
- Unrealistic assumptions
- Technical impossibilities
- Dependency risks

List each issue as: section - severity (HIGH/MEDIUM/LOW) - description

Document content:
[PASTE PLAN CONTENT HERE]

Context: $ARGUMENTS" 2>&1
```

### Step 2: Analyze Findings (Plan)

For each finding:
1. **Re-read the relevant section** of the plan
2. **Cross-reference** with other sections for context
3. **Evaluate validity**:
   - Is this a real gap or already addressed elsewhere?
   - Is the concern valid for the stated scope/timeline?
   - Is this a V1 concern or V2 backlog item?

4. **Decision matrix**:
   - Valid HIGH → Flag for immediate plan update
   - Valid MEDIUM → Recommend addressing before implementation
   - Valid LOW → Note as future consideration
   - False positive → Document why it's not an issue
   - Out of scope → Note as intentional V2 item

### Step 3: Compile Recommendations (Plan)

Group findings into:
1. **Must Fix Before Dev** - Critical gaps that would block implementation
2. **Should Address** - Important but can be tackled during dev
3. **Nice to Have** - Valid but not blocking
4. **Acknowledged Tradeoffs** - Known limitations accepted for V1
5. **False Positives** - With reasoning

---

## CONSENSUS ROUND (Both Modes)

After analyzing ALL findings, communicate back to Codex:

```bash
codex exec "I reviewed your findings and made the following decisions:

## [Fixes Applied / Plan Updates Recommended]
[List each with location and what was changed/recommended]

## False Positives (with reasoning)
[For each false positive, explain WHY with concrete evidence]

## Acknowledged Tradeoffs
[Items that are known limitations, not bugs]

Please verify:
1. Do you agree with my false positive assessments?
2. Are there any remaining issues I missed?
3. Do you have concerns about my analysis?

If you disagree with any assessment, explain why.
If everything looks good, confirm: 'LGTM - no remaining issues.'

Context: $ARGUMENTS" 2>&1
```

### Iterate Until Consensus

**Continue if Codex:**
- Disagrees with a false positive assessment → Re-evaluate
- Finds new issues → Analyze
- Has concerns → Address them

**Stop when:**
- Codex confirms "LGTM" or equivalent
- Both agree all remaining items are intentional/acceptable
- Max 3 consensus rounds reached

---

## Output Format

### Code Review Output

```markdown
## Codex Code Review - Round N

**Changes reviewed**: [file list]

### Findings

| File:Line | Severity | Issue | Assessment | Action |
|-----------|----------|-------|------------|--------|
| src/foo.ts:42 | HIGH | Nullable issue | Valid | Fixed |
| src/bar.ts:15 | MEDIUM | Missing await | False positive | See below |

### False Positive Reasoning
- `src/bar.ts:15` - Not an issue because [reason]

### Fixes Applied
- `src/foo.ts:42` - Added null check

### Consensus Status
- [x] Sent to Codex
- [ ] Codex agrees / disagrees on: [item]
```

### Plan Review Output

```markdown
## Codex Plan Review - Round N

**Document reviewed**: [file path]

### Findings

| Section | Severity | Issue | Assessment | Recommendation |
|---------|----------|-------|------------|----------------|
| Architecture | HIGH | Discord DO viability | Valid | Add prototype validation |
| Cost Model | MEDIUM | Missing egress costs | Valid | Update estimates |
| Security | LOW | No rate limiting | V2 scope | Backlog |

### Must Fix Before Dev
1. **Discord Gateway + DO hibernation** - Need to validate persistent WebSocket viability
2. ...

### Should Address
1. **Cost model completeness** - Add egress, bandwidth, support costs
2. ...

### Acknowledged Tradeoffs
- "Encrypted at rest" vs E2E - Documented, honest marketing
- No in-app chat - By design, add debug console instead

### False Positive Reasoning
- "Missing migration plan" - Intentionally no V0.5→V1 migration (fresh start)

### Consensus Status
- [x] Sent to Codex
- [ ] Codex agrees / disagrees on: [item]
```

---

## Final Report

```markdown
## Codex Review Complete - Consensus Reached

**Mode**: [Code / Plan]
**Rounds**: N
**Total findings**: X
**Valid issues**: Y
**False positives (agreed)**: Z

### Consensus Summary
[What both Claude and Codex agreed on]

### Action Items (Plan Mode)
- [ ] Item 1 - Priority HIGH
- [ ] Item 2 - Priority MEDIUM
...

### Final Status
- [x] All valid issues addressed
- [x] False positives explained and accepted
- [x] Codex confirmed: LGTM
```

---

## Important Notes

1. **Consensus is required** - Don't finish until Codex agrees or explicitly disagrees
2. **Explain reasoning clearly** - Codex needs context to evaluate assessments
3. **Be open to being wrong** - If Codex pushes back, reconsider
4. **Max 3 rounds** - If no consensus after 3 rounds, escalate to user
5. **Scope awareness** - For plans, distinguish V1 vs V2 concerns

## Arguments

- `$ARGUMENTS` - Either:
  - **Code**: Context about changes (e.g., "Updated feedback interfaces")
  - **Plan**: Path to plan file(s) (e.g., "docs/v1/QUANTU_V1_PLAN.md") or description
