# Journal

A personal engineering journal for documenting problems, solutions, and learnings.

## Purpose

This repo stores structured troubleshooting notes and decision logs. Each entry documents a real problem encountered during development, the options considered, the chosen solution, and command-by-command explanations.

## File Naming Convention

```
<YYYY-MM-DD>-<short-kebab-description>.md
```

Example: `2026-04-14-docker-disk-migration.md`

## Entry Structure

Every journal entry should follow this format:

```markdown
# Title

- **date**: YYYY-MM-DD
- **tags**: comma-separated keywords

---

## Problem
What went wrong and why.

## Options Considered
Table of options with pros/cons.

## Choice: <Option>
**Reason**: Why this option was selected.

## Steps Executed
Numbered steps with commands and explanations.

## Key Concepts
Table of concepts referenced in the entry.
```

## Writing Guidelines

- Write entries so they can be parsed programmatically (consistent headings, frontmatter-style metadata).
- Explain every command and flag — this journal serves as a learning reference.
- Include actual outputs/values where relevant (UUIDs, paths, sizes).
- Add a disk/architecture diagram when the solution involves infrastructure changes.
- Tags should be lowercase, useful for future search.

## Owner

- **Name**: Avinash Kumar
- **Machine**: Arch Linux, 16GB RAM, 32GB root (`/dev/sda2`), 78GB home (`/dev/sda3`), 476GB NVMe SSD (`/dev/nvme0n1`)
