# AGENTS.md

## Repository Overview

This repository contains [Agent Skills](https://agentskills.io) for [fabio](https://github.com/iemejia/fabio), an agent-first CLI for Microsoft Fabric.

## Structure

```
.
├── AGENTS.md                   # This file — context for AI agents working on this repo
├── README.md                   # User-facing docs (install via npx skills add)
├── LICENSE                     # Apache-2.0
├── .github/
│   ├── copilot-instructions.md          # General Copilot coding context for this repo
│   ├── copilot-review-instructions.md   # Custom instructions for Copilot PR reviews
│   ├── instructions/
│   │   ├── skill-content.instructions.md  # Scoped: applies to fabio/SKILL.md
│   │   └── references.instructions.md    # Scoped: applies to fabio/references/**
│   ├── prompts/
│   │   └── sync-fabio-skill.md          # Prompt for automated fabio release sync
│   └── workflows/
│       ├── copilot-review.yml           # Adds Copilot as native PR reviewer
│       ├── semantic-pull-request.yml    # Enforces conventional commit PR titles
│       ├── ai-review.yml               # Custom AI review with verdict (Copilot CLI)
│       ├── validate.yml                 # Structural + content quality validation
│       ├── eval.yml                     # Skill evaluation framework
│       ├── gitleaks.yml                 # Secret scanning
│       ├── dependency-review.yml        # Dependency security review
│       ├── dependabot-auto-merge.yml    # Auto-merge dependabot PRs
│       └── sync-with-fabio-releases.yml # Triggered on new fabio releases
└── fabio/                      # The skill directory (name must match SKILL.md `name` field)
    ├── SKILL.md                # Required: frontmatter + instructions (<500 lines)
    ├── scripts/
    │   └── install.sh          # Cross-platform binary installer
    └── references/
        ├── COMMANDS.md         # Full command reference
        ├── API-BEHAVIORS.md    # Critical API quirks and patterns
        └── EXAMPLES.md         # End-to-end workflow examples
```

## Conventions

### Skill Format

This project follows the [Agent Skills specification](https://agentskills.io/specification):

- Each skill is a directory containing a `SKILL.md` file with YAML frontmatter
- The directory name must match the `name` field in frontmatter (lowercase, hyphens only)
- `SKILL.md` body should stay under 500 lines (agents load it fully on activation)
- Detailed reference material goes in `references/` (loaded on demand)
- Executable helpers go in `scripts/` (must be cross-platform where possible)
- The `description` field (max 1024 chars) determines when agents activate the skill

### Progressive Disclosure

The skill is structured for three-stage loading:

1. **Discovery** (~100 tokens): Only `name` and `description` from frontmatter are read at startup
2. **Activation** (<5000 tokens): Full `SKILL.md` body loaded when task matches description
3. **Resources** (on demand): Files in `references/` and `scripts/` loaded only when needed

### Content Guidelines

- Focus on what agents would NOT know without the skill (API quirks, PascalCase requirements, token scoping, LRO patterns)
- Omit general knowledge (what a lakehouse is, how HTTP works)
- Provide concrete commands and examples, not abstract descriptions
- Document gotchas from real testing against live Fabric APIs
- Keep instructions procedural: step-by-step, not declarative

## Working on This Repository

### Adding or updating skill content

1. Keep `SKILL.md` concise — move detailed content to `references/`
2. When adding API behaviors, include the exact error message or code that triggered the discovery
3. Command examples should use shell variables (`$WS`, `$LH`) for IDs to show composability
4. Test that `SKILL.md` frontmatter validates:
   - `name`: lowercase alphanumeric + hyphens, matches directory name
   - `description`: 1-1024 characters, includes trigger keywords
   - `compatibility`: environment requirements (max 500 chars)

### Updating for new fabio releases

When fabio releases a new version:

1. Update `metadata.version` in `fabio/SKILL.md` frontmatter
2. Add new commands to `references/COMMANDS.md`
3. Document new API behaviors in `references/API-BEHAVIORS.md`
4. Add workflow examples for new features to `references/EXAMPLES.md`
5. Update `scripts/install.sh` only if the release asset naming changes

### Commit and PR conventions

PR titles and commit messages must follow [Conventional Commits](https://www.conventionalcommits.org/):

```
type(scope): description
```

Valid types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`, `revert`

Common scopes for this repo: `commands`, `api-behaviors`, `examples`, `version`, `install`, `skill`

Examples:
- `feat(commands): add app-backend command reference`
- `fix(api-behaviors): correct KQL token scope documentation`
- `docs(examples): add deploy workflow with parameters`
- `chore(version): bump to v0.18.0`

This is enforced by CI — PRs with non-conforming titles will fail the `Semantic PR Validation` check.

### Code review

Every PR is automatically reviewed by:

1. **Copilot (native reviewer)** — provides inline suggestions based on `.github/copilot-review-instructions.md`
2. **AI Review (Copilot CLI)** — produces a structured approve/reject verdict for skill content changes in `fabio/`
3. **Structural validation** — CI checks frontmatter, line counts, links, and content quality

Copilot's review focuses on:
- Agent Skills spec compliance (frontmatter, 500-line limit, progressive disclosure)
- fabio CLI command correctness (valid groups, subcommands, flags, PascalCase values)
- API behavior accuracy (correct scoping, LRO patterns, token requirements)
- Content quality for agent consumption (composable examples, error coverage, no general knowledge padding)

### Validating changes

```bash
# Check frontmatter compliance
npx skills init --check fabio/

# Verify skill is detected
npx skills list

# Test in VS Code: switch to Agent mode, type /skills, confirm "fabio" appears
```

## Key Dependencies

| Dependency | Role | Required |
|---|---|---|
| `fabio` binary | The CLI this skill teaches agents to use | Yes (runtime) |
| `az` (Azure CLI) | Supplementary Azure operations (networking, IAM, storage) outside Fabric scope | Recommended |
| `gh` (GitHub CLI) | Release downloads, Git integration | Recommended |
| `npx skills` | Skill installation and management | For installation only |

## Upstream References

- fabio repository: https://github.com/iemejia/fabio
- fabio releases: https://github.com/iemejia/fabio/releases
- Agent Skills spec: https://agentskills.io/specification
- Fabric REST API: https://learn.microsoft.com/rest/api/fabric/
- Fabric REST API specs (OpenAPI): https://github.com/Azure/azure-rest-api-specs/tree/main/specification/fabric
