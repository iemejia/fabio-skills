---
applyTo: "fabio/references/**"
---

# Reference File Instructions — `fabio/references/**`

This guide defines how reference files should be structured. These files are loaded on demand (not at activation) and contain detailed content that would bloat SKILL.md.

---

## 1) File Roles

| File | Purpose | Loaded When |
|------|---------|-------------|
| `COMMANDS.md` | Complete command reference with all flags | Agent needs exact flag names/types |
| `API-BEHAVIORS.md` | API quirks, gotchas, error patterns | Agent hits unexpected API behavior |
| `EXAMPLES.md` | End-to-end workflow examples | Agent needs to compose multi-step operations |

---

## 2) COMMANDS.md Format

### Command entry structure

```
fabio <group> <subcommand> [flags]   Description
```

### Flag documentation

```
  --workspace, -w <GUID>         Workspace ID (required)
  --id <GUID>                    Item ID
  --name, -n <string>            Display name
  --output, -o <format>          json|table|plain|csv|tsv
  --dry-run                      Preview without executing
```

### Rules:
- One entry per subcommand (no duplicates)
- Group commands by functional category (same categories as SKILL.md)
- Include ALL flags for each command (this is the detailed reference)
- Mark required vs optional flags
- Show default values where relevant
- Note workspace-scoped vs tenant-scoped

---

## 3) API-BEHAVIORS.md Format

Each behavior entry should include:

```markdown
### N. Behavior Title

**Symptom**: What goes wrong without this knowledge
**Cause**: Why the API behaves this way
**Fix**: What to do correctly

\`\`\`bash
# Correct usage
fabio <correct command>

# Wrong (causes error)
fabio <wrong command>
# Error: <exact error message>
\`\`\`
```

### Rules:
- Each entry must be based on real testing against live Fabric APIs
- Include the exact error message or HTTP status code
- Show both correct and incorrect usage where helpful
- Number entries for easy cross-referencing from SKILL.md
- Group by category (lakehouse, warehouse, KQL, deploy, etc.)

---

## 4) EXAMPLES.md Format

Each example should be a complete, runnable workflow:

```markdown
## Workflow: <Title>

<1-2 sentence description of what this achieves>

\`\`\`bash
# Step 1: <what this does>
<command>

# Step 2: <what this does>
<command>
\`\`\`

**Notes:**
- <any important caveats>
- <timing/capacity considerations>
```

### Rules:
- All examples must use shell variables (`$WS`, `$LH`, `$WH`, `$CAP`)
- Show the full workflow from prerequisites to final result
- Include error recovery patterns where relevant
- Demonstrate composability (piping, variable capture, jq)
- Group workflows by use case (data engineering, analytics, CI/CD, admin)

---

## 5) Content Quality Standards

### Include:
- Behaviors that cause silent failures or confusing errors
- Exact error messages from the Fabric API
- Workarounds for known API limitations
- Timing constraints (cold start, LRO timeouts, capacity warmup)
- PascalCase values exactly as the API requires

### Exclude:
- General knowledge agents already have
- fabio internal implementation details
- Information available from `fabio <command> --help`
- Speculative behaviors not verified against live APIs
- Deprecated commands or behaviors (remove them)

---

## 6) Cross-Referencing

- SKILL.md may reference behaviors by number: "See [API-BEHAVIORS.md](references/API-BEHAVIORS.md) #3"
- COMMANDS.md entries should note related API behaviors
- EXAMPLES.md should reference prerequisite commands from COMMANDS.md
- All relative links must resolve (validated in CI)
