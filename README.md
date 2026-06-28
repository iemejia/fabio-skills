# fabio-skills (DEPRECATED)

> **This repository has been superseded.** The fabio agent skill now ships directly in the main repository at [`.agents/skills/fabio/`](https://github.com/iemejia/fabio/tree/main/.agents/skills/fabio).
>
> ## Migration
>
> ```bash
> # 1. Remove the old skill
> npx skills remove fabio
>
> # 2. Install from the main repo (always up to date)
> npx skills add https://github.com/iemejia/fabio
> ```
>
> Or if you cloned manually:
> ```bash
> # Remove old clone
> rm -rf ~/.agents/skills/fabio-skills
>
> # Install new (skill is at .agents/skills/fabio/ inside the repo)
> npx skills add https://github.com/iemejia/fabio
> ```
>
> The new skill is identical in content but updates atomically with the CLI — no more version lag.
>
> This repository is archived for historical reference only.

---

_Original README below:_

# fabio-skills

[Agent Skills](https://agentskills.io) for [fabio](https://github.com/iemejia/fabio) — the agent-first CLI for Microsoft Fabric.

## Install

### Option 1: Using `npx` (recommended)

> **Prerequisites:** Requires [Node.js](https://nodejs.org) v18+ (which bundles `npm` and `npx`). Install via [nodejs.org](https://nodejs.org), or use a version manager like [nvm](https://github.com/nvm-sh/nvm), [fnm](https://github.com/Schniz/fnm), or [volta](https://volta.sh).

```bash
npx skills add iemejia/fabio-skills
```

This installs the `fabio` skill into your project's `.agents/skills/` directory, making it available to GitHub Copilot (Agent mode), Claude Code, OpenCode, Cursor, and any other [compatible agent](https://agentskills.io/clients).

Install globally (available in all projects):

```bash
npx skills add iemejia/fabio-skills --global
```

Install for a specific agent only:

```bash
npx skills add iemejia/fabio-skills --agent github-copilot
```

Verify installation:

```bash
npx skills list
```

In VS Code with GitHub Copilot, switch to **Agent mode** and type `/skills` to confirm `fabio` appears in the list.

### Option 2: Using GitHub CLI (`gh skill`)

> **Prerequisites:** Requires [GitHub CLI](https://cli.github.com/) v2.90.0 or later.

The `gh skill` command can search, preview, install, and update agent skills directly from GitHub repositories.

```bash
# Preview the skill before installing (inspect contents without installing)
gh skill preview iemejia/fabio-skills fabio

# Install interactively (browse available skills in the repo)
gh skill install iemejia/fabio-skills

# Install directly
gh skill install iemejia/fabio-skills fabio

# Install a specific version
gh skill install iemejia/fabio-skills fabio@v0.26.0

# Install pinned to a version (skipped during updates)
gh skill install iemejia/fabio-skills fabio --pin v0.26.0

# Install for a specific agent and scope
gh skill install iemejia/fabio-skills fabio --agent claude-code --scope user
```

Update skills:

```bash
# Check for updates interactively
gh skill update

# Update a specific skill
gh skill update fabio

# Update all installed skills
gh skill update --all
```

In the GitHub Copilot CLI, use `/skills list` to verify and `/skills reload` after installing.

### Option 3: Manual (git clone)

No tooling required beyond `git`. Clone the repository and copy the skill folder to the correct location:

```bash
git clone https://github.com/iemejia/fabio-skills.git /tmp/fabio-skills

# Project-level (current repo only)
cp -r /tmp/fabio-skills/fabio .agents/skills/fabio

# Global (all projects)
cp -r /tmp/fabio-skills/fabio ~/.agents/skills/fabio
```

### Option 4: Direct download (no git required)

```bash
curl -fsSL https://github.com/iemejia/fabio-skills/archive/refs/heads/main.tar.gz | \
  tar xz --strip-components=1 -C .agents/skills/ fabio-skills-main/fabio
```

Or with `gh`:

```bash
gh repo clone iemejia/fabio-skills /tmp/fabio-skills
cp -r /tmp/fabio-skills/fabio .agents/skills/fabio
```

### Option 5: Git submodule (stays updatable)

```bash
git submodule add https://github.com/iemejia/fabio-skills.git .agents/skills-repos/fabio-skills
ln -s .agents/skills-repos/fabio-skills/fabio .agents/skills/fabio
```

### Skill placement by agent

If you install manually, place the skill in the correct directory for your agent:

| Agent | Location |
|-------|----------|
| GitHub Copilot (VS Code) | `.agents/skills/fabio/` or `.github/skills/fabio/` |
| GitHub Copilot CLI | `.agents/skills/fabio/` or `~/.copilot/skills/fabio/` |
| Claude Code | `.agents/skills/fabio/` |
| OpenCode | `.agents/skills/fabio/` or configure in `opencode.json` |
| Cursor | `.cursor/skills/fabio/` |
| Gemini CLI | `.gemini/skills/fabio/` |
| OpenAI Codex | `.agents/skills/fabio/` |
| Global (all agents) | `~/.agents/skills/fabio/` |

The `npx skills add` and `gh skill install` commands handle placement automatically based on detected agents.

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
| **`az`** (Azure CLI) | Supplementary Azure operations outside Fabric scope (networking, IAM, storage). | [Install Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| **`gh`** (GitHub CLI) | Download fabio releases easily. Useful when Fabric workspaces are connected to GitHub repos via `fabio git connect`. | [Install GitHub CLI](https://cli.github.com/) |

### 3. Authenticate

```bash
fabio auth login
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
    ├── COMMANDS.md             # Full command reference (74 groups, 851+ subcommands)
    ├── API-BEHAVIORS.md        # Critical API quirks agents must know
    └── EXAMPLES.md             # End-to-end workflow examples
```

The agent loads `SKILL.md` (~288 lines) when activated. Reference files are loaded on demand only when deeper detail is needed for a specific task.

## Other agents

The skill works with any agent that supports the [Agent Skills specification](https://agentskills.io/specification). See the [Skill placement by agent](#skill-placement-by-agent) table above for the correct directory per agent.

## Updating

```bash
# With npx
npx skills update fabio

# With GitHub CLI
gh skill update fabio
```

## Development

This repository uses a tiered CI pipeline and automated sync to keep the skill accurate and up to date with upstream fabio releases.

### CI Workflows

All workflows live in `.github/workflows/` and run automatically on PRs and pushes:

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| **Tier 1+2: Skill Validation** (`validate.yml`) | PRs/pushes touching `fabio/**` or `README.md` | Structural validation (frontmatter, line count, required files, internal links) and content quality checks (shell syntax, duplicate commands, PascalCase compliance, version consistency, markdown structure) |
| **Tier 3: Skill Eval** (`eval.yml`) | PRs touching `fabio/**` or `tests/**` | Runs [promptfoo](https://promptfoo.dev) evals against GitHub Models to test that LLMs produce correct fabio commands when given the skill instructions. Compares pass rates against the base branch and fails on regressions. |
| **Tier 4: AI Review** (`ai-review.yml`) | PRs touching `fabio/**` | Uses Copilot CLI to perform an automated code review of skill changes, checking command correctness, API behavior accuracy, instruction quality, and spec compliance. Posts findings as a PR comment. |
| **Sync with fabio releases** (`sync-with-fabio-releases.yml`) | `repository_dispatch` (fabio-released) or manual `workflow_dispatch` | Automatically updates the skill when a new fabio version is released. Uses Copilot CLI to analyze the release diff, update SKILL.md, COMMANDS.md, API-BEHAVIORS.md, and EXAMPLES.md, then opens a PR. |

### Running Locally

#### Validation (Tier 1+2)

Run the same structural and content checks that CI runs:

```bash
# Check SKILL.md frontmatter is valid
head -1 fabio/SKILL.md | grep -q "^---" && echo "Frontmatter OK"

# Check line count (must be under 500)
wc -l < fabio/SKILL.md

# Check required files exist
ls fabio/SKILL.md fabio/scripts/install.sh fabio/references/COMMANDS.md fabio/references/API-BEHAVIORS.md fabio/references/EXAMPLES.md

# Validate with npx skills (if Node.js is installed)
npx skills init --check fabio/
```

#### Eval (Tier 3)

Run promptfoo evals locally to test instruction quality. Requires Node.js and a GitHub token with `models:read` scope:

```bash
# Install promptfoo
npm install -g promptfoo

# Run evals (uses GitHub Models via Copilot subscription)
GITHUB_TOKEN=your_pat promptfoo eval -c tests/promptfooconfig.yaml

# View results in browser
promptfoo view
```

The eval config (`tests/promptfooconfig.yaml`) tests 18+ scenarios including CRUD operations, PascalCase compliance, composability patterns, SQL/KQL queries, LRO awareness, CI/CD deploy workflows, and error handling.

#### AI Review (Tier 4)

Run a local AI review of your changes using Copilot CLI:

```bash
# Install Copilot CLI
npm install -g @github/copilot

# Generate a diff and review it
git diff > /tmp/my-changes.diff
copilot "Review this skill diff for correctness: $(cat /tmp/my-changes.diff)"
```

### Automated Sync with fabio Releases

When a new fabio version is released, the `sync-with-fabio-releases` workflow:

1. Fetches the release tag, release notes, and diff from the previous version
2. Checks out the upstream fabio repo at the release tag
3. Assembles context from the release notes, commit log, and fabio's `AGENTS.md`
4. Invokes Copilot CLI with the sync prompt (`.github/prompts/sync-fabio-skill.md`) to update all skill files
5. Validates the result (frontmatter, line count)
6. Opens a PR with labels `automated` and `fabio-sync`

To trigger a manual sync (e.g., to catch up with a missed release):

```bash
# Via GitHub CLI
gh workflow run sync-with-fabio-releases.yml -f tag=v0.17.0

# Or via the Actions tab on GitHub (workflow_dispatch with optional tag input)
```

### Adding or Updating Content

When contributing changes:

1. Keep `fabio/SKILL.md` under 500 lines — move detailed content to `references/`
2. Use shell variables (`$WS`, `$LH`, `$WH`) in command examples for composability
3. Document API quirks with the exact error message that revealed them
4. Test locally with `npx skills init --check fabio/` before pushing
5. If adding new commands, also add a corresponding eval test case in `tests/promptfooconfig.yaml`

## License

MIT
