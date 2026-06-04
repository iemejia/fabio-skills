# Sync fabio CLI Release to fabio-skills

You are updating the `fabio-skills` repository — an Agent Skills package that teaches AI agents how to use the `fabio` CLI for Microsoft Fabric. A new version of the upstream `fabio` CLI (github.com/iemejia/fabio) has been released. Your job is to analyze ALL changes in the release and update the skill files to reflect the current state of fabio.

## CRITICAL: You MUST make changes

Every fabio release contains something worth updating in this skills repo. If you find yourself making no changes, you are being too conservative. At minimum:
- The `metadata.version` in `fabio/SKILL.md` frontmatter MUST be updated to match the released version
- The `README.md` stats (command groups, subcommands) MUST match the upstream AGENTS.md

## Context Sources (Priority Order)

1. **Release Notes** — Human-curated summary of what's new (provided below as "Release Notes")
2. **AGENTS.md** — Canonical feature inventory with all commands, behaviors, and decisions (provided below)
3. **Commit log** — List of commits since previous release (provided below)
4. **Diff** — Full code diff at `/tmp/fabio-changes.diff` (load on demand for details)
5. **Full repo** — Complete fabio source at `./fabio-upstream/` (browse for implementation details)

## Repository Structure

```
fabio-skills/
├── fabio/
│   ├── SKILL.md              # Main skill file (<500 lines, loaded on activation)
│   ├── references/
│   │   ├── COMMANDS.md       # Full command reference (all subcommands + flags)
│   │   ├── API-BEHAVIORS.md  # Critical API quirks, gotchas, error patterns
│   │   └── EXAMPLES.md       # End-to-end workflow examples
│   └── scripts/
│       └── install.sh        # Binary installer
├── README.md                 # User-facing docs with stats and install instructions
└── AGENTS.md                 # Repo conventions (do not modify)
```

## What to Update (check ALL of these)

### 1. Version Bump (ALWAYS required)

Check `./fabio-upstream/Cargo.toml` for the current version and update:
- `metadata.version` in `fabio/SKILL.md` frontmatter

### 2. README.md Stats

Compare the numbers in `README.md` against the upstream `AGENTS.md` Progress section:
- Total command groups count
- Total subcommands count
- Any new highlights or features worth mentioning

### 3. New Commands → Update `fabio/references/COMMANDS.md`

Look for new `.rs` files in `fabio-upstream/src/commands/` or new subcommand entries in existing files. For each new command:
- Add it to the appropriate section in COMMANDS.md
- Use the exact format: `fabio <group> <subcommand> --flag <value>   Description`
- Include ALL flags with their types and descriptions
- Group under the correct functional category

Signs of new commands:
- New files in `src/commands/`
- New variants in `Command` enum in `src/cli.rs`
- New subcommand structs with `#[derive(Parser)]`

### 4. New Command Groups → Update `fabio/SKILL.md`

If entirely new command groups are added (new item types, new top-level commands):
- Add them to the "Command Groups" section in SKILL.md under the appropriate category
- Keep SKILL.md under 500 lines — move details to references/

### 5. API Behaviors & Quirks → Update `fabio/references/API-BEHAVIORS.md`

Look for discoveries in the diff that reveal:
- New error patterns or error codes
- PascalCase requirements for new endpoints
- LRO behaviors for new operations
- Token scoping changes
- Endpoint URL patterns (workspace-scoped vs tenant-scoped)
- Rate limiting behaviors
- Format requirements (JSON structure, required fields)
- Things that broke during testing (these are the most valuable)

Signs of API behaviors in the code:
- Comments explaining WHY something is done a certain way
- Error handling with specific HTTP status codes
- Custom serialization or field reordering
- Retry logic or special headers
- Test assertions that document expected API responses

### 6. New Workflows → Update `fabio/references/EXAMPLES.md`

If new commands enable new end-to-end workflows:
- Add practical, composable examples using shell variables ($WS, $LH, etc.)
- Show the full workflow from creation to usage
- Include error recovery patterns where relevant
- Use the pipe/compose patterns consistent with existing examples

### 7. New Features (non-command changes)

For changes that aren't new commands but still affect agent usage:
- New global flags (e.g., `--output csv`, new `--format` options)
- New authentication methods
- New output formats
- Shell completions or other DX improvements
- CI/CD integration features (GitHub Actions docs, etc.)

Update the relevant section in SKILL.md or EXAMPLES.md.

## Content Guidelines

### What to Include
- Commands and flags that an AI agent would need to construct correct CLI invocations
- API behaviors that would cause silent failures or confusing errors without documentation
- Exact error messages and their meanings
- Workarounds for API limitations
- Concrete examples with shell variables showing composability
- New capabilities that expand what agents can do with fabio

### What NOT to Include
- General knowledge (what a lakehouse is, how REST APIs work)
- Internal implementation details (Rust code structure, internal function names)
- Test infrastructure or CI details
- Things obvious from `--help` output alone (basic flag descriptions without quirks)

### Style Rules
- Use PascalCase for Fabric API values exactly as the API requires them
- Use shell variables ($WS, $LH, $CAP) in examples for composability
- Document the MINIMUM required flags for each command
- Note when commands are workspace-scoped vs tenant-scoped
- Keep descriptions procedural and actionable

## How to Analyze the Release

1. **Start with the Release Notes** (provided inline) — they highlight what's new and important
2. **Cross-reference with AGENTS.md** (provided inline) — the "Progress" section is the canonical list of all features, behaviors, and decisions
3. **Read `/tmp/fabio-changes-summary.txt`** for an overview of what files changed since the previous release
4. **Browse `./fabio-upstream/src/cli.rs`** to see the full command tree (all subcommands)
5. **Check `./fabio-upstream/Cargo.toml`** for the version number
6. **Check `./fabio-upstream/src/commands/`** for implementation details of new commands
7. **Check `./fabio-upstream/tests/`** for expected behaviors (tests document API contracts)
8. **Read `/tmp/fabio-changes.diff`** for detailed code changes (large — use selectively)

## Validation Checklist

Before finishing, verify:
- [ ] `metadata.version` in SKILL.md matches the version in `./fabio-upstream/Cargo.toml`
- [ ] SKILL.md is under 500 lines
- [ ] SKILL.md frontmatter has valid `name`, `description`, `compatibility` fields
- [ ] New commands in COMMANDS.md match the actual CLI (check against src/cli.rs)
- [ ] API behaviors documented are based on actual code/test evidence (not speculation)
- [ ] Examples use consistent variable naming ($WS, $LH, $WH, $CAP, etc.)
- [ ] No duplicate entries in COMMANDS.md
- [ ] README.md stats are current

## Output

After making changes, write a file at `/tmp/pr-body.md` with:
1. A summary of what was updated and why
2. List of new commands added (if any)
3. List of new API behaviors documented (if any)
4. List of new examples added (if any)
5. Any other updates (version bump, stats, features)
6. Reference to the upstream release tag

Format the PR body in markdown suitable for a GitHub Pull Request.
