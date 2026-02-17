# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Repository Overview

A Cursor/Claude skill for **Feature-Sliced Design (FSD v2.1)** in Next.js applications. The skill teaches agents how to implement, refactor, and maintain FSD architecture—layered hierarchy (app, views, widgets, features, entities, shared), import rules, and the custom **views** layer pattern.

## Directory Structure

```
feature-sliced-design/
├── AGENTS.md                    # This file — repo-level agent guidance
├── README.md
├── CLAUDE.md
│
└── skills/
    └── feature-sliced-design/
        ├── SKILL.md             # Required: skill definition (frontmatter + instructions)
        ├── AGENTS.md             # Skill-level: FSD patterns for agents implementing projects
        ├── metadata.json        # Skill metadata (version, abstract, references)
        │
        └── references/          # Supporting docs (load on demand)
            ├── layers-and-segments.md
            ├── examples.md
            └── monorepo.md      # Optional
```

## Key Files

| File | Purpose |
|------|---------|
| **SKILL.md** | Main skill content. Frontmatter (`name`, `description`) + Overview, principles, workflow, configuration. Keep under ~500 lines. |
| **AGENTS.md** | Pattern guidance for agents implementing FSD in user projects. Incorrect/correct examples, impact levels. |
| **references/** | Detailed layer definitions, code examples, monorepo guide. Linked from SKILL.md; read only when needed. |
| **metadata.json** | Version, abstract, external reference URLs. Used for skill packaging/listing. |

## Modifying the Skill

### When Updating SKILL.md

- Keep **Overview** section with reference file links
- Maintain **Reference Files** → **Documentation Library** at the end
- Links use `./references/{filename}.md` (relative to skill directory)
- description frontmatter: include trigger phrases for discoverability

### When Adding Reference Files

1. Add markdown file to `references/`
2. Add link in SKILL.md **Overview** and **Reference Files** section
3. Use descriptive filenames: `kebab-case.md` (e.g., `layers-and-segments.md`)

### When Updating AGENTS.md (skill-level)

- Follow structure: Abstract → Table of Contents (with Impact) → Sections with Incorrect/Correct examples
- Impact levels: CRITICAL, HIGH, MEDIUM
- Keep examples concise; link to references/ for full details

## Conventions

- **Skill directory**: `skills/feature-sliced-design/`
- **Reference files**: `references/` with one-level-deep links from SKILL.md
- **Version**: Sync `metadata.json` version with SKILL.md frontmatter `version`

## Distribution

Users install by copying the skill directory to their Cursor skills path:

```bash
cp -r skills/feature-sliced-design ~/.claude/skills/
```

Or add to project: `.claude/skills/feature-sliced-design/`
