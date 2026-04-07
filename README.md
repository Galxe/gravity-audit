# GravityAIAudit

Centralized security audit issue tracking for Gravity L1.

All audit findings across every round (internal AI audits, external professional audits) are managed here, covering `gravity-reth`, `gravity-sdk`, and related contracts.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Issue Hierarchy](#issue-hierarchy)
- [Label System](#label-system)
- [Triage & Fix Workflow](#triage--fix-workflow)
- [AI Automation](#ai-automation)
- [Code Submodules](#code-submodules)

---

## Repository Structure

```
GravityAIAudit/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── audit-round.md        # Parent Issue template (audit round)
│   │   └── finding.md            # Sub-issue template (single finding)
│   └── workflows/
│       └── auto-triage.yml       # Auto-label workflow
├── audits/
│   ├── 2026-02-23-gravity-reth/  # Raw report archives per round
│   ├── 2026-02-28-gravity-sdk/
│   └── 2026-03-05-phase2/
├── gravity-reth/                  # git submodule
├── gravity-sdk/                   # git submodule
└── README.md
```

---

## Issue Hierarchy

### Parent Issue (Audit Round)

One Parent Issue per audit round, capturing high-level metadata.

**Naming format:**
```
[AUDIT] <Source> <Round> — <Target Repo> <Version/Date>
```

**Examples:**
- `[AUDIT] AI Phase 1 — gravity-reth 2026-02-23`
- `[AUDIT] AI Phase 2 — gravity-reth + gravity-sdk 2026-03-05`
- `[AUDIT] Trail of Bits Round 1 — gravity-reth v1.0.0`

**Required fields (Issue body):**
```
- **Audit Source**: AI (Claude Opus 4.6) / Trail of Bits / OtterSec / ...
- **Scope**: gravity-reth crates/..., gravity-sdk crates/...
- **Code Version**: commit hash or tag
- **Report Link**: link to raw report under audits/
- **Stats**: CRITICAL x, HIGH x, MEDIUM x, LOW x, INFO x
```

### Sub-issue (Single Finding)

One Sub-issue per finding, linked to the corresponding Parent Issue.

**Naming format:**
```
[<REPO>-<ID>] [<SEVERITY>] <short title>
```

**Examples:**
- `[GRETH-020] [CRITICAL] Pipeline permanent deadlock via barrier timeout gap`
- `[GSDK-017] [HIGH] Nested mutex holding in BlockBufferManager`

See the Issue template for required fields.

---

## Label System

### Severity

| Label | Color | Meaning |
|-------|-------|---------|
| `severity: critical` | 🔴 `#B60205` | Can cause chain state fork, fund loss, or permanent DoS |
| `severity: high` | 🟠 `#E11D48` | Exploitable security issue |
| `severity: medium` | 🟡 `#F97316` | Defense-in-depth concern, plan to fix |
| `severity: low` | 🟢 `#EAB308` | Low risk, suggested improvement |
| `severity: info` | ⚪ `#6B7280` | Informational, no security impact |

### Target Repository

| Label | Meaning |
|-------|---------|
| `repo: gravity-reth` | Issue in gravity-reth |
| `repo: gravity-sdk` | Issue in gravity-sdk |
| `repo: contracts` | Issue in on-chain contracts |
| `repo: cross` | Cross-repository issue |

### Status

| Label | Meaning |
|-------|---------|
| `status: needs-triage` | Newly opened, awaiting confirmation (default) |
| `status: confirmed` | Confirmed as a real issue |
| `status: invalid` | False positive, confirmed non-issue |
| `status: wont-fix` | Known issue, decided not to fix (reason required in comment) |
| `status: in-progress` | Fix in progress |
| `status: fixed` | Fix committed, awaiting verification |
| `status: verified` | Fix verified and closed |

### Source

| Label | Meaning |
|-------|---------|
| `source: ai-audit` | Found by AI automated audit |
| `source: manual-audit` | Found by professional manual audit |
| `source: community` | Found by community / bug bounty |

### Ownership

Create per-developer labels using GitHub handles, e.g.:
- `owner: @alice`
- `owner: @bob`

---

## Triage & Fix Workflow

### Creating Issues (After an Audit)

```
1. Create a Parent Issue for the audit round
2. For each finding, run:
     git blame <file> -L <line_start>,<line_end>
   to locate the code author
3. Create a Sub-issue in GravityAIAudit
   - Apply labels: severity: xxx / repo: xxx / source: xxx / owner: @xxx
   - Add the Sub-issue number to the Parent Issue tasklist
4. Notify relevant developers
```

### Fix Workflow (Developer View)

1. Filter Issues by your `owner:` label
2. Create a fix branch in `gravity-reth` or `gravity-sdk`
3. In the PR description, reference the Issue:
   ```
   Fixes Galxe/GravityAIAudit#<issue_number>
   ```
4. After the PR is merged, add a comment to the Issue and close it

### Reject / Invalid Workflow

If a finding is a false positive or will not be fixed:

1. Add a comment explaining the rationale
2. Change the label to `status: invalid` or `status: wont-fix`
3. Close the Issue (developer or Tech Lead)

### Progress Tracking

Use GitHub Tasklist syntax in the Parent Issue body:

```markdown
## Findings

### CRITICAL
- [ ] #12 [GRETH-020] Pipeline permanent deadlock
- [ ] #13 [GRETH-021] Cache eviction of unpersisted trie nodes

### HIGH
- [x] #14 [GRETH-022] BLOCKHASH opcode unimplemented ✅ fixed in gravity-reth#276
- [ ] #15 [GRETH-023] Token loss during epoch transitions
```

GitHub renders this as a progress bar on the Parent Issue automatically.

---

## AI Automation

### Available Now (Claude Code)

```bash
# From within the GravityAIAudit directory, ask Claude Code to:

# Bulk-create Issues from an audit report
"Read audits/2026-03-05-phase2/ and create a Sub-issue for each HIGH+
 finding, running git blame to identify the code author and apply the
 correct owner label."

# Daily progress summary
"Scan all open issues, group by severity and owner, output a markdown table."
```

### Advanced: GitHub Actions

`.github/workflows/auto-triage.yml` handles:

- **Auto-label on open**: Parses `[SEVERITY]` and `[GRETH-` / `[GSDK-` from the Issue title, applies `severity:` and `repo:` labels automatically
- **Semantic close** *(planned)*: Agent reads comment semantics (e.g. "confirmed false positive"), applies `status: invalid` and closes
- **Auto blame & assign** *(planned)*: Agent parses file paths and line numbers from the Issue body, runs `git blame` against the submodule, applies `owner:` label

---

## Code Submodules

Audited codebases are included as submodules for direct cross-reference when reviewing Issues.

```bash
# After cloning this repo
git submodule update --init --recursive

# Update to latest
git submodule update --remote gravity-reth
git submodule update --remote gravity-sdk
```

| Submodule | Repository | Description |
|-----------|-----------|-------------|
| `gravity-reth/` | https://github.com/Galxe/gravity-reth | EVM execution layer |
| `gravity-sdk/` | https://github.com/Galxe/gravity-sdk | AptosBFT consensus layer |

---

## Completed Audit Rounds

| Date | Source | Scope | Findings | Parent Issue |
|------|--------|-------|----------|--------------|
| 2026-02-23 | AI (Claude Opus 4.6) | gravity-reth | CRITICAL:0 HIGH:3 MED:8 LOW:8 | [#1](#) |
| 2026-02-28 | AI (Claude Opus 4.6) | gravity-sdk | CRITICAL:0 HIGH:2 MED:5 LOW:4 | [#2](#) |
| 2026-03-05 | AI Phase 2 (Multi-agent) | gravity-reth + gravity-sdk | CRITICAL:2 HIGH:16 MED:25 LOW:16 | [#3](#) |

---

*Maintained by the Galxe Engineering Team*
