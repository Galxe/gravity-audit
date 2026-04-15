# Design Intent

This tree records **intentional design decisions** in audited projects — things an automated (or human) security reviewer is likely to flag as bugs, but which are actually by-design and authoritatively confirmed as non-issues.

## Purpose

When Gravity Reviewer (or any auditor) keeps re-reporting the same finding against a module, the root cause is almost always missing design context, not a missing code-level comment. Writing that context inline would bloat the code; writing it here keeps the source clean and gives the reviewer a single authoritative place to look.

## Layout

```
design-intent/
└── <project>/
    ├── README.md         # project-level notes (optional)
    └── <module>.md       # one file per functional module
```

Organize **by functional module, not by file path or line number.** Line numbers drift; module boundaries are stable.

## Module file structure (required)

Every `<module>.md` MUST follow this structure so agents can read it efficiently:

```markdown
# <Module Name>

> One-sentence purpose of this module within the project.

## Scope
- Which crates / paths / files this module covers.
- Keywords a reviewer's task description is likely to contain.

## Design Intent
- Architectural decisions and *why they are the way they are*.
- Invariants the module relies on from upstream/downstream components.

## Known Non-Issues

### <Short title of the class of finding>  — closed-as-intent, ref #<issue>
- **Surface finding** — how a naive reviewer would describe it.
- **Why this is not a bug** — authoritative explanation (usually from the issue's closing comment).
- **Boundary** — the conditions under which this reasoning *stops* applying, i.e. what *would* make it a real bug. This is the most important field: it tells the reviewer when to ignore the non-issue rule and report anyway.
- **Reference** — link to the closed audit issue.
```

## How reviewers should use this

1. Before starting a review task, **Read** the `design-intent/<project>/` directory listing.
2. If the task description plausibly overlaps any module here, **Read the full module file**.
3. When about to report a finding, check whether it matches any "Known Non-Issue" entry — and check the **Boundary** field to confirm the exemption still applies.
4. If the boundary no longer holds, report the finding (and flag the design-intent entry as needing an update).

## How to add new entries

A closed audit issue becomes a design-intent entry when **it is closed by a comment explaining why it is not a bug** (as opposed to being closed by a fix PR). Copy the essential reasoning from the closing comment into the appropriate module file; link back to the issue for provenance.
