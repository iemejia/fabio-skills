# fabio-skills

[Agent Skills](https://agentskills.io) for [fabio](https://github.com/iemejia/fabio) — the agent-first CLI for Microsoft Fabric.

## Install

```bash
npx skills add iemejia/fabio-skills
```

This installs the `fabio` skill into your project's `.agents/skills/` directory, making it available to GitHub Copilot (Agent mode), Claude Code, OpenCode, Cursor, and any other [compatible agent](https://agentskills.io/clients).

### Install globally (available in all projects)

```bash
npx skills add iemejia/fabio-skills --global
```

### Install for a specific agent only

```bash
npx skills add iemejia/fabio-skills --agent copilot
```

### Verify installation

```bash
npx skills list
```

In VS Code with GitHub Copilot, switch to **Agent mode** and type `/skills` to confirm `fabio` appears in the list.

## Prerequisites

The skill teaches agents how to use `fabio`, but you still need the tool itself and its companion CLIs installed on the machine:

### 1. Install fabio

```bash
# Auto-detect OS/arch, download latest release, install to ~/.local/bin
bash .agents/skills/fabio/scripts/install.sh
```

Or manually:

```bash
VERSION=$(curl -s https://api.github.com/repos/iemejia/fabio/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
curl -fsSL "https://github.com/iemejia/fabio/releases/download/${VERSION}/fabio-linux-x64.tar.gz" | tar xz -C ~/.local/bin
```

Or with `gh`:

```bash
gh release download --repo iemejia/fabio --pattern "fabio-linux-x64.tar.gz"
tar xzf fabio-linux-x64.tar.gz -C ~/.local/bin
```

Or build from source (requires Rust 1.85+):

```bash
cargo install --git https://github.com/iemejia/fabio.git
```

### 2. Install companion CLIs (strongly recommended)

| Tool | Why | Install |
|------|-----|---------|
| **`az`** (Azure CLI) | Required for authentication. fabio delegates all auth to `az login`. Also useful for supplementary Azure operations outside Fabric scope. | [Install Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| **`gh`** (GitHub CLI) | Download fabio releases easily. Useful when Fabric workspaces are connected to GitHub repos via `fabio git connect`. | [Install GitHub CLI](https://cli.github.com/) |

### 3. Authenticate

```bash
az login
fabio auth status
```

## Usage with GitHub Copilot

Once installed, the skill activates automatically when you ask Copilot (in Agent mode) about Microsoft Fabric tasks:

- "Create a lakehouse and upload these CSV files as Delta tables"
- "Query the warehouse for monthly revenue by country"
- "Set up an eventstream pipeline from custom endpoint to KQL database"
- "Run the ETL notebook and wait for completion"
- "Connect this workspace to our GitHub repo"

The agent will use `fabio` commands with structured JSON output, handling authentication, pagination, error recovery, and LRO polling automatically.

## What the skill covers

| Area | Capabilities |
|------|-------------|
| **Data Storage** | Lakehouses, warehouses, SQL databases, file upload/download, Delta table loading |
| **Compute** | Notebooks (run + wait), Spark jobs, data pipelines, environments |
| **Analytics** | Semantic models (Direct Lake TMDL), DAX queries, reports, dashboards |
| **Real-Time** | Eventhouses, eventstreams, KQL databases, KQL queries |
| **Integration** | Git sync, deployment pipelines, connections, domains |
| **Administration** | Workspace management, role assignments, capacity, governance |

## Skill structure

```
fabio/
├── SKILL.md                    # Main skill — instructions loaded on activation
├── scripts/
│   └── install.sh              # Cross-platform binary installer
└── references/
    ├── COMMANDS.md             # Full command reference (37 groups, 267+ subcommands)
    ├── API-BEHAVIORS.md        # Critical API quirks agents must know
    └── EXAMPLES.md             # End-to-end workflow examples
```

The agent loads `SKILL.md` (~240 lines) when activated. Reference files are loaded on demand only when deeper detail is needed for a specific task.

## Other agents

The skill works with any agent that supports the [Agent Skills specification](https://agentskills.io/specification):

| Agent | Skill location |
|-------|---------------|
| GitHub Copilot (VS Code) | `.agents/skills/fabio/` |
| Claude Code | `.agents/skills/fabio/` |
| OpenCode | `.agents/skills/fabio/` or configure in `opencode.json` |
| Cursor | `.cursor/skills/fabio/` |
| Gemini CLI | `.gemini/skills/fabio/` |
| OpenAI Codex | `.agents/skills/fabio/` |

`npx skills add` handles placement automatically based on detected agents.

## Updating

```bash
npx skills update fabio
```

## License

Apache-2.0
