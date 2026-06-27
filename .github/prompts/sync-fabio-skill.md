# Sync fabio CLI Release to fabio-skills

You are updating the `fabio-skills` repository — an Agent Skills package that teaches AI agents how to use the `fabio` CLI for Microsoft Fabric. A new version of the upstream `fabio` CLI (github.com/iemejia/fabio) has been released. Your job is to analyze the release and update the skill to reflect the current state.

## Architecture (How Agent Discovery Works)

fabio's agent discoverability is **runtime-based**. The binary IS the documentation:

- `fabio context agent` — machine-readable command schema (all flags, types, 222+ examples)
- `fabio context describe <group> <cmd>` — deep-dive on any command
- `fabio context find "<query>"` — keyword search across all commands
- `fabio context workflow <name>` — multi-step recipes
- `fabio context best-practices <topic>` — operational guidance
- `fabio context examples <group> <cmd>` — output shape examples

The skill's job is NOT to duplicate this. The skill is a **bootstrapping document** that:
1. Gets agents started (install + auth)
2. Teaches them to use runtime introspection
3. Documents the critical API behaviors they can't discover from `--help`

## Repository Structure

```
fabio-skills/
├── fabio/
│   ├── SKILL.md              # Bootstrapping document (~164 lines, loaded on activation)
│   ├── references/
│   │   └── API-BEHAVIORS.md  # Critical API quirks that cause silent failures
│   └── scripts/
│       └── install.sh        # Binary installer
├── tests/
│   └── promptfooconfig.yaml  # 77-case eval testing skill quality
├── README.md                 # User-facing docs with install instructions
└── AGENTS.md                 # Repo conventions (do not modify)
```

**There is NO `references/COMMANDS.md` or `references/EXAMPLES.md`.** Those were deleted because agents get command info from the binary at runtime.

## What to Update (check ALL of these)

### 1. Version Bump (ALWAYS required)

Check `./fabio-upstream/Cargo.toml` for the current version:
- Update `metadata.version` in `fabio/SKILL.md` frontmatter

### 2. SKILL.md Updates (only when needed)

The SKILL.md is a 164-line bootstrapping document. Only update it for:
- **New authentication methods** (added to the Authentication section)
- **New global flags** that affect agent safety (like `--readonly`, `--enable-commands`)
- **New critical API behaviors** (added to the "Critical API Behaviors" numbered list)
- **Major new capabilities** that change the tool's scope (new command categories)

Do NOT add individual command details — agents use `fabio context agent` for that.

### 3. API Behaviors → Update `fabio/references/API-BEHAVIORS.md`

This is the highest-value file. Look for discoveries that reveal:
- New error patterns or error codes
- PascalCase requirements for new endpoints
- LRO behaviors for new operations
- Token scoping changes
- Endpoint URL patterns (workspace-scoped vs tenant-scoped)
- Rate limiting behaviors
- Format requirements (JSON structure, required fields)
- Things that broke during testing

Signs of API behaviors in the code:
- Comments explaining WHY something is done a certain way
- Error handling with specific HTTP status codes
- Custom serialization or field reordering
- Retry logic or special headers
- Test assertions that document expected API responses
- Entries in the "API Behaviors Discovered" sections of AGENTS.md

### 4. README.md Stats

Update if the release significantly changes:
- Total command group count
- Total subcommand count
- Major new feature highlights

### 5. Eval Test Cases (when new patterns emerge)

If the release introduces new patterns that agents need to handle correctly, add test cases to `tests/promptfooconfig.yaml`:
- New PascalCase value requirements
- New tenant-scoped commands (no --workspace)
- New LRO-aware operations (--wait)
- New safety features (--readonly, --enable-commands)
- New workflow patterns

## Content Guidelines

### What to Include in API-BEHAVIORS.md
- API behaviors that cause **silent failures** without documentation
- Exact error messages and their meanings
- PascalCase requirements for specific endpoints
- Workspace-scoped vs tenant-scoped distinctions
- Required query parameters (`?beta=true`, `?preview=true`)
- Non-standard response envelope keys
- LRO polling patterns specific to new item types
- Capacity/SKU requirements for specific features

### What NOT to Include Anywhere
- Command flags and descriptions (agents get from `fabio context agent`)
- Command examples (agents get from `fabio context describe`)
- Output shapes (agents get from `fabio context examples`)
- Workflow recipes (agents get from `fabio context workflow`)
- General knowledge (what a lakehouse is, how HTTP works)
- Internal implementation details

### Style Rules
- PascalCase for Fabric API values exactly as required
- Shell variables ($WS, $LH, $CAP) in any examples
- Keep SKILL.md under 500 lines (currently ~164)
- Keep API-BEHAVIORS.md focused on gotchas, not general reference

## How to Analyze the Release

1. **Release Notes** (provided inline) — what's new and important
2. **AGENTS.md** (provided inline) — canonical feature inventory, API behaviors section
3. **Commit log** (provided inline) — all commits since previous release
4. **`/tmp/fabio-changes-summary.txt`** — overview of file changes
5. **`./fabio-upstream/Cargo.toml`** — version number
6. **`./fabio-upstream/src/commands/`** — new command implementations
7. **`/tmp/fabio-changes.diff`** — detailed code changes (large, use selectively)

## Validation Checklist

Before finishing, verify:
- [ ] `metadata.version` in SKILL.md matches `./fabio-upstream/Cargo.toml`
- [ ] SKILL.md is under 500 lines
- [ ] SKILL.md frontmatter has valid `name`, `description`, `compatibility` fields
- [ ] API behaviors documented are based on actual code/test evidence (not speculation)
- [ ] No content duplicates what `fabio context agent/describe/workflow` provides
- [ ] README.md stats are current (if changed)

## Output

After making changes, write a file at `/tmp/pr-body.md` with:
1. Summary of what was updated and why
2. New API behaviors documented (if any)
3. SKILL.md changes (if any, with justification)
4. Eval test cases added (if any)
5. Reference to the upstream release tag

Format the PR body in markdown suitable for a GitHub Pull Request.
