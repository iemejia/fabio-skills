---
applyTo: "fabio/SKILL.md"
---

# SKILL.md Instructions

This guide defines how the main skill file should be structured and maintained. It applies to `fabio/SKILL.md`.

---

## 1) Frontmatter Requirements

The YAML frontmatter between `---` markers must contain:

```yaml
---
name: fabio                    # Must match directory name, lowercase + hyphens only
description: "..."             # Max 1024 chars, includes activation trigger keywords
license: MIT
compatibility: "..."           # Max 500 chars, runtime requirements
metadata:
  author: iemejia
  version: "X.Y.Z"            # Must match latest fabio release
  repository: https://github.com/iemejia/fabio
---
```

**Rules:**
- `name` must be lowercase alphanumeric + hyphens, no leading/trailing hyphens, no consecutive hyphens
- `description` must include keywords that trigger agent activation (workspace, lakehouse, Fabric, CLI, etc.)
- `version` must track the latest fabio release

---

## 2) Body Size Constraint

- **Maximum 500 lines** — this is the activation payload loaded into agent context
- If adding content pushes past 500 lines, move detail to `references/`
- Every line must earn its place — no padding, no general knowledge

---

## 3) Content Structure

The body should follow this order:

1. **Overview** — What fabio is (1-2 sentences)
2. **Installation** — How to get the binary
3. **Authentication** — All auth methods
4. **Core Concepts** — Output format, error codes, global options, LRO
5. **Key Workflow** — One canonical end-to-end example
6. **Critical API Behaviors** — Top 10-12 behaviors that cause failures without documentation
7. **Command Groups** — Categorized list of all groups (names only, details in COMMANDS.md)
8. **Composability Patterns** — Piping, chaining, variable capture

---

## 4) Command Documentation Style

Commands in SKILL.md should show:
- Minimum required flags only
- Shell variables for IDs (`$WS`, `$LH`, `$WH`, `$CAP`)
- Composable patterns (pipe to `jq`, capture to variables)

```bash
# Good — composable, minimal
WS=$(fabio workspace list --query id --limit 1 -o plain)
fabio lakehouse create --workspace $WS --name "MyLake"

# Bad — hardcoded, verbose
fabio lakehouse create --workspace "12345678-1234-1234-1234-123456789abc" --name "MyLake" --description "A lakehouse"
```

---

## 5) API Behaviors in SKILL.md

Only include behaviors in SKILL.md that:
- Cause immediate failure if not known (PascalCase requirement, wrong endpoint)
- Apply to commonly-used commands (lakehouse, warehouse, notebook)
- Cannot be inferred from `--help` or general API knowledge

Detailed behaviors and edge cases go in `references/API-BEHAVIORS.md`.

---

## 6) Version Updates

When updating for a new fabio release:
1. Update `metadata.version` in frontmatter
2. Add new command groups to the categorized list
3. Add new critical API behaviors (if top-12 worthy)
4. Update subcommand/group counts in the Overview
5. Verify the file stays under 500 lines
