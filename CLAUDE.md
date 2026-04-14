# Journal

A personal engineering journal for documenting problems, solutions, and learnings.

## Hugo Static Site

This repo is a Hugo site using the PaperMod theme, deployed to GitHub Pages via GitHub Actions.

- **Local dev**: `hugo server -D` → `http://localhost:1313/journal/`
- **Production**: `https://avisord.github.io/journal/`
- **Theme**: PaperMod (git submodule in `themes/PaperMod/`)

## Creating New Entries

```bash
hugo new posts/<YYYY-MM-DD>-<short-kebab-description>.md
```

This uses the archetype at `archetypes/posts.md`. Remove `draft: true` when ready to publish.

## Entry Format

All entries live in `content/posts/` and must have YAML frontmatter:

```yaml
---
title: "Descriptive Title"
date: YYYY-MM-DD
tags: [tag1, tag2, tag3]
---
```

Do NOT use a `# Title` heading in the body — Hugo renders the title from frontmatter.

## Entry Structure

```markdown
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

- Explain every command and flag — this journal serves as a learning reference.
- Include actual outputs/values where relevant (UUIDs, paths, sizes).
- Add a disk/architecture diagram when the solution involves infrastructure changes.
- Tags should be lowercase, useful for future search.

## Owner

- **Name**: Avinash Kumar
- **Machine**: Arch Linux, 16GB RAM, 32GB root (`/dev/sda2`), 78GB home (`/dev/sda3`), 476GB NVMe SSD (`/dev/nvme0n1`)
