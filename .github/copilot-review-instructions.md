# Copilot Code Review Instructions for fabio-skills

## Review Focus Areas

When reviewing pull requests to this repository, prioritize the following:

### 1. Agent Skills Spec Compliance

Every skill must follow the [agentskills.io specification](https://agentskills.io/specification):
- YAML frontmatter with required fields (`name`, `description`, `compatibility`)
- `name` field: lowercase alphanumeric + hyphens, matches directory name
- `description` field: max 1024 characters, includes trigger keywords for activation
- `compatibility` field: max 500 characters
- `SKILL.md` body under 500 lines (activation payload must be small)
- Detailed content belongs in `references/` (loaded on demand, not at activation)

### 2. fabio CLI Command Correctness

All fabio commands in examples and instructions must be syntactically valid:
- Correct command group and subcommand names (fabio has 70 groups, 771 subcommands)
- Required flags present (`--workspace`, `--id`, etc.)
- PascalCase values where the Fabric API requires them (`Overwrite`, `Append`, `Csv`, `Parquet`)
- Correct endpoint scoping (workspace-scoped vs tenant-scoped commands)
- LRO patterns (`--wait`) where operations are asynchronous
- Output format flags (`-o json`, `-o table`, `-o plain`)

### 3. Progressive Disclosure Structure

The three-stage loading model must be preserved:
- **Discovery** (~100 tokens): Only `name` + `description` from frontmatter
- **Activation** (<5000 tokens): Full `SKILL.md` body — must be concise and actionable
- **Resources** (on demand): Files in `references/` and `scripts/` — loaded only when needed

Flag violations where:
- SKILL.md contains content that belongs in `references/`
- Reference files duplicate what's already in SKILL.md
- General knowledge is included that agents already know (what a lakehouse is, how HTTP works)

### 4. Content Quality for Agent Consumption

Instructions must help AI agents succeed on first attempt:
- Examples use composable shell variables (`$WS`, `$LH`, `$WH`, `$CAP`)
- Error cases and edge cases are documented
- API quirks include the exact error message or behavior that triggered the documentation
- Commands show minimum required flags (not every optional flag)
- Workflows are end-to-end: from authentication to final result

### 5. API Behavior Accuracy

Documented behaviors must match known Fabric API patterns:
- HTTP methods match operation type (GET reads, POST creates, PATCH updates)
- LRO operations correctly documented (202 + Location header polling)
- Token scoping is correct (Fabric scope, Storage scope, ARM scope)
- Workspace-scoped vs tenant-scoped endpoints distinguished
- Pagination patterns (`--all`, `--limit`, `--continuation-token`)
- Rate limiting awareness (bulk APIs preferred over repeated calls)

### 6. Install Script Safety

Changes to `scripts/install.sh` must be:
- Cross-platform (Linux x64/arm64, macOS arm64, Windows noted separately)
- Safe to run without elevated privileges (install to `~/.local/bin` by default)
- Idempotent (re-running should update, not break)
- Using HTTPS for all downloads
- Verifying checksums or using trusted sources (GitHub releases)

## Review Anti-Patterns (Flag These)

- Lowercase API values where PascalCase is required (`overwrite` instead of `Overwrite`)
- SKILL.md exceeding 500 lines
- Commands with incorrect flag names or missing required flags
- Hardcoded workspace/item IDs in examples (should use `$WS`, `$LH` variables)
- General knowledge padding (explaining what Microsoft Fabric is, how REST works)
- Internal implementation details from fabio's Rust codebase
- Broken relative links to `references/` or `scripts/` files
- Duplicate command entries in COMMANDS.md
- Examples that aren't composable (can't be piped or chained)
- Missing `--dry-run` documentation for mutation commands
- Version mismatches between SKILL.md frontmatter and README.md

## Commit Message Format

Verify PR titles and commits follow conventional commits:
```
type(scope): description
```
Valid types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`, `revert`

Examples:
- `feat(commands): add app-backend command reference`
- `fix(api-behaviors): correct KQL token scope documentation`
- `docs(examples): add deploy workflow with parameters`
- `chore(version): bump to v0.18.0`
