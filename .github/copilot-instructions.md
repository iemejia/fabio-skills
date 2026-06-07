# Copilot Instructions for fabio-skills

## Overview

This repository contains [Agent Skills](https://agentskills.io) for [fabio](https://github.com/iemejia/fabio), an agent-first CLI for Microsoft Fabric. The skill teaches AI agents how to use fabio's 771 subcommands across 70 command groups to manage Fabric artifacts.

## Repository Structure

```
fabio-skills/
├── AGENTS.md                   # AI agent context (repo conventions)
├── README.md                   # User-facing docs (install via npx skills add)
├── LICENSE                     # Apache-2.0
└── fabio/                      # Skill directory
    ├── SKILL.md                # Main skill file (<500 lines, YAML frontmatter)
    ├── scripts/
    │   └── install.sh          # Cross-platform binary installer
    └── references/
        ├── COMMANDS.md         # Full command reference (all groups + subcommands)
        ├── API-BEHAVIORS.md    # Critical API quirks and patterns
        └── EXAMPLES.md         # End-to-end workflow examples
```

## Content Conventions

### What Belongs in This Repo

- Commands and flags that agents need to construct correct CLI invocations
- API behaviors that would cause silent failures without documentation
- Exact error messages and their meanings
- Workarounds for Fabric API limitations
- Concrete examples with shell variables showing composability
- PascalCase values where the API requires them (Overwrite, Append, Csv, Parquet)

### What Does NOT Belong

- General knowledge (what a lakehouse is, how REST APIs work)
- Internal fabio implementation details (Rust code, internal functions)
- Test infrastructure or CI internals from the fabio repo
- Information obvious from `--help` output alone

### Progressive Disclosure Model

Content is structured for three-stage agent loading:

1. **Discovery** (~100 tokens): `name` + `description` from SKILL.md frontmatter
2. **Activation** (<5000 tokens): Full SKILL.md body — concise, actionable
3. **Resources** (on demand): `references/` and `scripts/` — loaded only when needed

Keep SKILL.md under 500 lines. Move detailed content to `references/`.

## Style Rules

- Use PascalCase for Fabric API values exactly as the API requires
- Use shell variables (`$WS`, `$LH`, `$WH`, `$CAP`) in examples for composability
- Document minimum required flags for each command
- Note when commands are workspace-scoped vs tenant-scoped
- Keep descriptions procedural and actionable
- Prefer concrete commands over abstract descriptions
- Include the exact error that triggered a documented behavior

## Formatting

- Markdown files use GitHub-flavored markdown
- Code blocks use `bash` or `json` language tags
- Command signatures: `fabio <group> <subcommand> --flag <value>   Description`
- Headings follow hierarchy (no skipping levels)
- No trailing whitespace
- LF line endings (no CRLF)

## Agent Skills Specification

The skill must comply with [agentskills.io/specification](https://agentskills.io/specification):

- **name**: lowercase alphanumeric + hyphens, matches directory name (`fabio`)
- **description**: max 1024 chars, includes keywords that trigger activation
- **compatibility**: max 500 chars, describes runtime requirements
- **metadata**: `author`, `version`, `repository` fields
- **SKILL.md body**: under 500 lines, procedural instructions

## fabio CLI Conventions

When writing commands, follow these patterns:

- Structured output: `--json` or `-o json` for machine-readable output
- Composability: pipe `--query` with `-o plain` for single values
- Mutations: always document `--dry-run` support
- Lists: document `--all`, `--limit`, `--continuation-token`
- LRO operations: document `--wait` for async operations
- Error codes: `AUTH_REQUIRED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `INVALID_INPUT`

## Commit Conventions

- Use conventional commits: `type(scope): description`
- Valid types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`, `revert`
- Examples:
  - `feat(commands): add graphql-api command reference`
  - `fix(api-behaviors): correct load-table format values`
  - `docs(examples): add deploy workflow with parameters`
  - `chore(version): bump to v0.18.0`

## Upstream References

- fabio repository: https://github.com/iemejia/fabio
- fabio releases: https://github.com/iemejia/fabio/releases
- Agent Skills spec: https://agentskills.io/specification
- Fabric REST API: https://learn.microsoft.com/rest/api/fabric/

## Maintenance Matrix

| Changed | Must Also Update |
|---------|-----------------|
| `fabio/SKILL.md` frontmatter version | `README.md` stats |
| New command group in SKILL.md | `references/COMMANDS.md` (full flags) |
| New API behavior discovered | `references/API-BEHAVIORS.md` |
| New end-to-end workflow | `references/EXAMPLES.md` |
| `scripts/install.sh` | Only if release asset naming changes |
