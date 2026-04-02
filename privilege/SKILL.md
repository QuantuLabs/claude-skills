---
name: privilege
description: >
  Static privilege-path audit for blockchain programs in GitHub repositories.
  Clones the repo, detects the framework (Anchor, Pinocchio, Quasar, native Solana, or Solidity/EVM),
  inventories authorities, signer gates, role checks, upgrade paths, and delegated execution,
  then maps escalation chains and scores the resulting findings.

  Trigger this skill ONLY when the user explicitly invokes `/privilege <github_repo_url>`,
  OR asks to audit privileges, authorities, or access control for a blockchain repo and provides a GitHub URL.
  Do NOT trigger on general security questions, smart contract reviews without a repo URL,
  or non-blockchain code audits.
---

# Privilege — Blockchain Privilege Path Auditor

You are performing a static privilege audit.
Goal: identify who can mutate critical state, move value, upgrade code, grant roles, or bypass controls, and explain each path with evidence.

Stay strict:
- No speculation. If control flow is unclear, flag it as `Needs manual review`.
- A real finding ties together three things: holder, action, and guard or missing guard.
- Do not treat every `authority` field, `owner` variable, or `onlyOwner` modifier as a vulnerability by itself.

---

## Trigger conditions

This skill activates when:
1. The user runs `/privilege <github_repo_url>` (primary trigger)
2. The user asks to audit privileges, scan authorities, or analyze access control in a blockchain repo and provides a GitHub URL

A valid invocation always includes a GitHub repository URL. If no URL is provided, ask for one before proceeding.

---

## Execution pipeline

Follow these steps in order. Do not skip steps.

### Step 1 — Preflight and clone

Require a GitHub repository URL. If the user gives a branch or subdirectory URL, analyze what is reachable and note any scope limitation.

Work in a temporary directory and clean it up at the end, even on failure.

```bash
TMP_DIR=$(mktemp -d -t privilege-audit.XXXXXX)
trap 'rm -rf "$TMP_DIR"' EXIT
git clone --depth 1 <repo_url> "$TMP_DIR/repo"
cd "$TMP_DIR/repo"
```

### Step 2 — Detect framework and scope

Prefer `rg --files` and `rg -n` for discovery. If `rg` is unavailable, fall back to `find` and `grep -RIn`.

Detect the framework using these markers:
- **Anchor (Solana)**: `Anchor.toml` or `use anchor_lang` in `.rs` files
- **Pinocchio (Solana)**: `pinocchio` dependency or imports, `pinocchio::program_entrypoint!`, `AccountView`, `Address`, `pinocchio_system`
- **Quasar (Solana)**: `quasar-lang` dependency, `use quasar_lang::prelude::*`, `#[program]`, `#[derive(Accounts)]`, `Ctx<...>`, `declare_id!`
- **Native Solana**: `use solana_program` without Anchor, Pinocchio, or Quasar markers
- **Solidity/EVM**: `.sol` files with `pragma solidity`
- **Other**: flag as unsupported, report what was found, stop

If multiple frameworks appear in one repo, prioritize the directories that actually contain deployed programs or contracts and note the mixed layout in the report.

Identify analysis roots:
- **Anchor**: `programs/*/src/`, especially `lib.rs`, `instructions/`, `state/`, and files containing `authority`, `admin`, or `owner`
- **Pinocchio**: `src/lib.rs`, `src/instructions/`, `src/processor/`, `src/state/`, and files importing `pinocchio` or `pinocchio_system`
- **Quasar**: `programs/*/src/` or `src/lib.rs`, plus files using `#[program]`, `#[derive(Accounts)]`, `Ctx<...>`, or `quasar_lang::prelude::*`
- **Native Solana**: `src/entrypoint.rs`, `processor.rs`, `instruction.rs`, `state.rs`
- **Solidity/EVM**: `contracts/`, `src/`, proxy or upgrade files, and access-control helpers

Report what was detected before moving on:
> Framework: Anchor (Solana)
> Programs found: 3 (registry, marketplace, staking)
> Proceeding with analysis...

### Step 3 — Build the privilege inventory

For every externally callable instruction or function, extract:
- entrypoint or function name
- file path and line number
- privileged actor: EOA, PDA, multisig, role, upgrade authority, or any signer
- guard: signer check, role check, ownership check, seed constraint, timelock, threshold, or missing guard
- sensitive action: upgrade, withdraw, transfer, mint, burn, freeze, pause, reconfigure, close, or arbitrary call
- mutation target: state account, vault, token mint, proxy implementation, admin slot, etc.
- delegation path: CPI, proxy, `delegatecall`, signer seeds, or nested admin transfer

Record each match with surrounding evidence, but do not dump raw grep output into the final answer. Normalize repeated hits into one finding per privileged action.

#### Solana patterns to scan

| Category | What to look for |
|---|---|
| Authority keys | Fields named `authority`, `admin`, `owner`, `creator`, `governor`, `operator` in account structs |
| Signer and ownership constraints | `Signer<'info>`, `#[account(signer)]`, `has_one = authority`, `owner =`, `address =`, `constraint =` |
| Access gates | `require!()` or `if` blocks checking `ctx.accounts.authority.key()`, `.is_signer`, role enums, governance checks |
| Upgrade authority | `set_authority`, `upgrade_authority`, `BpfUpgradeable` references |
| PDA derivation | `seeds = [...]`, `bump`, `find_program_address` — record exact seed composition |
| CPI / delegation | `invoke_signed`, `CpiContext`, cross-program calls that pass signer seeds |
| Missing checks | `UncheckedAccount`, `AccountInfo` without `Signer` or `has_one`, `/// CHECK:` comments |
| Mutable exposure | `#[account(mut)]`, `realloc =`, lamport mutation, token state mutation without clear ownership or authority guard |
| Close / drain | `close = destination` — check if destination is constrained or attacker-controllable |
| Multisig / governance | Threshold checks, multiple signer requirements, timelock patterns |

Framework-specific Solana markers:
- **Anchor**: `Context<...>`, `Account<'info, T>`, `Program<'info, _>`, `InterfaceAccount`, `has_one`, `constraint =`, SPL authority setters
- **Pinocchio**: `pinocchio::program_entrypoint!`, `AccountView`, `Address`, manual `is_signer()` helpers, direct owner or address comparisons, `invoke_signed`, `pinocchio_system::instructions::*`
- **Quasar**: `#[program]`, `#[derive(Accounts)]`, `Ctx<...>`, `Account<T>`, `&'info Signer`, `Program<System>`, `#[account(init|mut|close|realloc|seeds|bump)]`, `require!`, `require_eq!`, `require_keys_eq!`, `quasar_lang::cpi::*`

#### Solidity / EVM patterns to scan

| Category | What to look for |
|---|---|
| Owner patterns | `onlyOwner`, `Ownable`, `msg.sender == owner` |
| Role-based | `AccessControl`, `hasRole`, `grantRole`, `revokeRole`, `DEFAULT_ADMIN_ROLE` |
| Proxy / upgrade | `delegatecall`, `TransparentUpgradeableProxy`, `UUPSUpgradeable`, `_authorizeUpgrade`, `upgradeTo`, `upgradeToAndCall`, `initializer`, `reinitializer` |
| External calls | `.call{value:}`, `delegatecall`, unguarded external calls, arbitrary target setters |
| Privilege transfer | `transferOwnership`, `renounceOwnership`, `changeAdmin`, admin setters, privileged `set*` functions |
| Value and control surfaces | `withdraw`, `sweep`, `mint`, `burn`, `pause`, `unpause`, `selfdestruct` |

### Step 4 — Map privilege and escalation chains

For each normalized finding, trace the full path:

1. **Who holds the privilege?** (specific key, PDA, multisig, any signer)
2. **What can they do?** (the instruction or function they can call)
3. **Under what conditions?** (constraints, timelocks, thresholds)
4. **Can this privilege escalate?** (can holder A grant themselves privilege B, or call an instruction that modifies authority)
5. **Is delegation possible?** (CPI chains, proxy calls, role granting)

Build a directed graph in this form: `Source -> Action -> Target` and annotate each edge with the relevant condition.

Important:
- Merge duplicates. Multiple pattern hits inside the same privileged path should become one clear finding.
- Distinguish direct control from delegated control.
- If a role can grant itself or replace another authority, treat that as escalation even if the initial role looks limited.
- If macros, generated code, proxy fallback logic, or dynamic dispatch hide the exact target, mark the path `Needs manual review`.

### Step 5 — Score severity

Assign each finding a severity level using this rubric:

| Severity | Criteria |
|---|---|
| **CRITICAL** | Single key can directly drain funds, mint or burn critical assets, upgrade the program or implementation, or bypass all checks. No multisig or timelock. |
| **HIGH** | Single key has broad privileged control, or a sensitive action is missing a signer, role, or ownership check. |
| **MEDIUM** | Privilege exists and matters, but it is constrained by multisig, timelock, PDA-only control, or another meaningful guard. |
| **LOW** | Weakness is real but limited in practice, such as mutable exposure without a realistic exploit path. |
| **INFO** | Design observation. Not a vulnerability but worth noting for completeness. |

Use these calibration rules:
- A declared admin or owner role is not automatically a vulnerability. It becomes a finding only when its power is unusually broad, weakly guarded, or poorly documented.
- Missing auth on privileged mutation is never below `HIGH`.
- Upgrade authority, proxy admin, arbitrary `delegatecall`, or unrestricted signer-seed delegation should not be rated below `HIGH`.
- Multisig and timelock reduce severity, but do not erase concentration risk.

Calculate a global score:
- Start at 100
- Each CRITICAL: -25
- Each HIGH: -15
- Each MEDIUM: -5
- Each LOW: -1
- Floor at 0

---

## Output format

Default output is **inline in the conversation**. No file, no artifact, no download.
Be concise. No filler, no repeated context.
If the user explicitly asks for a file (PDF, JSON, artifact), adapt.

### Structure

Use this exact layout, in this order:

```
## Privilege Audit — [repo name]

**[framework] · [program count] program(s) · [total lines] L**
**Score: [X]/100**
[N] CRITICAL · [N] HIGH · [N] MEDIUM · [N] LOW · [N] INFO

---

### CRITICAL

**function_name()** `[path/to/file:line]` — [type]
[One or two sentences: what's wrong, why it matters.]

[Repeat for each CRITICAL finding]

---

### HIGH

[Same format, shorter if needed]

---

### MEDIUM

[One-liners OK for MEDIUM and below]

---

### LOW

[One-liners]

---

### Escalation Chains

**C1 [SEVERITY]** — [source] → [action] = [consequence]
[Repeat for each chain]

---

### Reco

1. [One sentence]
2. [One sentence]
3. [One sentence]

> Static privilege analysis. Not a comprehensive audit.
```

### Formatting rules

- Group findings by severity with H3 headers (### CRITICAL, ### HIGH, etc.)
- Each finding: **function_name()** in bold + file path + line + type on first line, description below
- Cite the exact guard, missing guard, or escalation step in the description
- No markdown tables — they break on mobile. Use the flat format above instead.
- Privilege chains: one line each, bold chain ID + severity, arrow notation
- Recommendations: numbered, one sentence max each
- Skip empty severity groups (if zero HIGH, skip the HIGH section)
- Code references in backticks: `function_name`
- Keep the whole output scannable on a phone screen

---

## Edge cases and constraints

- **Monorepo with multiple programs**: analyze each program separately, then cross-reference CPI privilege chains between them.
- **No privileges found**: report "No explicit privilege patterns detected" and explain whether the design appears permissionless or whether access control is too indirect for static confidence.
- **Obfuscated patterns**: if authority checks use non-standard naming, flag as INFO with "Non-standard access control — manual review recommended."
- **Large repos (>50 files)**: prioritize `lib.rs`, `processor.rs`, `instructions/` directory, and any file containing `authority` or `admin`. Mention skipped files.
- **Generated code or macros**: summarize what is statically visible and mark unresolved paths as `Needs manual review`.
- **Private repos / auth errors**: if clone fails, report the error and stop. Do not guess.

---

## What this skill does NOT do

- No on-chain verification (this is static analysis only)
- No formal verification or symbolic execution
- No runtime testing or fuzzing
- No audit certification — this is a preliminary privilege map, not a professional audit

### Step 6 — Cleanup

Cleanup is mandatory. Remove the temporary clone after the analysis finishes. If you used a shell `trap`, it will already happen automatically.

---

State this disclaimer at the end of every output:
> This is a static privilege analysis, not a comprehensive security audit. Findings should be verified manually and on-chain before making security decisions.
