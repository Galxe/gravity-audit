---
name: "🔍 Audit Finding (Sub-issue)"
about: Record a single concrete audit finding
title: "[<REPO>-<ID>] [<SEVERITY>] <short title>"
labels: "status: needs-triage, source: ai-audit"
---

## Info

- **Finding ID**: <!-- GRETH-020 / GSDK-017 / ... -->
- **Severity**: <!-- CRITICAL / HIGH / MEDIUM / LOW / INFO -->
- **Repository**: <!-- gravity-reth / gravity-sdk / contracts -->
- **Audit Round**: <!-- Parent Issue #<number> -->
- **Code Author**: <!-- @github_handle (via git blame) -->

## Description

**File**: `<!-- crates/xxx/src/xxx.rs:line_start-line_end -->`

<!-- Detailed description of the issue, including relevant code snippet -->

```rust
// relevant code
```

## Impact

<!-- Describe the attack scenario and blast radius -->

**Exploitability**: <!-- Easy / Medium / Hard -->
**Blast Radius**: <!-- Single node / All validators / On-chain funds / ... -->

## Steps to Reproduce

<!-- PoC if available -->

1.
2.

## Recommendation

<!-- Specific fix or mitigation -->

## References

- Original report: <!-- audits/2026-03-05-phase2/ -->
- Related finding: <!-- #<issue_number> if applicable -->
