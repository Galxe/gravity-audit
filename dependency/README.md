# Dependency Context

This tree records how an audited project **interacts with its upstream / peer dependencies**. Reviewers use it to understand cross-boundary contracts without having to re-derive them from the dependency source each time.

## Purpose

When auditing gravity-reth, most of the interesting security surface lives on its boundary with `gravity-sdk` (consensus layer) and `gravity_chain_core_contracts` (on-chain system contracts). A reviewer who does not understand these boundaries will either miss real bugs or flood the report with noise about already-documented interaction invariants.

Unlike [design-intent](../design-intent/), these files are **not** a bug-exemption source. They are *context that helps the reviewer reason about interface correctness*. A finding that lives on the boundary is still a finding; these notes exist so the reviewer can correctly judge which side of the boundary the bug is on.

## Layout

```
dependency/
└── <project>/
    ├── README.md                       # overview of deps (optional)
    └── <dependency_project>.md         # one file per dependency
```

## Dependency file structure (required)

Every `<dependency_project>.md` MUST follow this structure:

```markdown
# <dependency name>

> One-sentence role of the dependency relative to <project>.

## Overview
- What the dependency is (origin, role, 1–3 sentences).

## How <project> depends on it
- Crates, paths, and bridge modules.

## Interaction Surface
- Per interface / channel / API:
  - Direction (project → dep, dep → project)
  - Purpose
  - Key data types
  - File:line pointers on the <project> side

## Invariants <project> assumes <dep> upholds
- What the project relies on from the dependency.

## Invariants <dep> assumes <project> upholds
- What the project must guarantee back.

## Audit hot spots
- Places on the boundary where bugs are most likely; what the reviewer should check.
```

## How reviewers should use this

1. Before any cross-boundary review task (anything touching the execution/consensus bridge, system-contract ABIs, relayers, etc.), **Read** the relevant `dependency/<project>/<dep>.md`.
2. When a finding appears to cross a boundary, use the **Invariants** sections to decide which side owns the contract, and therefore which side is buggy.
3. If the dependency's real behaviour diverges from what this file claims, update the file; do not silently work around the discrepancy.
