---
name: fabio
description: "Manage Microsoft Fabric artifacts and data using the fabio CLI - an agent-first command-line tool with structured JSON output, composable piping, and machine-readable errors. Use when working with Fabric workspaces, lakehouses, warehouses, notebooks, eventhouses, semantic models, reports, data pipelines, KQL databases, eventstreams, or any Fabric REST API resource. Covers CRUD operations, file upload/download, SQL/DAX/KQL queries, Git integration, deployment pipelines, and administration."
license: Apache-2.0
compatibility: "Requires fabio binary (Linux/macOS/Windows x64/arm64), Azure CLI (az), and az login credentials. Strongly recommended companions: gh (GitHub CLI) for release downloads and az for authentication and supplementary Azure operations. Network access to api.fabric.microsoft.com and onelake.dfs.fabric.microsoft.com required."
metadata:
  author: iemejia
  version: "0.7.0"
  repository: https://github.com/iemejia/fabio
---

# fabio — Agent-First CLI for Microsoft Fabric

## Overview

`fabio` is a CLI designed for AI agents first, humans second. It manages the entire Microsoft Fabric platform from the command line with structured JSON output, composable stdin/stdout piping, machine-readable error codes, and non-interactive operation.

## Installation

### Download Pre-built Binary (Recommended)

Run the bundled install script to download the latest release:

```bash
# Auto-detect OS/arch and install to ~/.local/bin (or /usr/local/bin)
bash scripts/install.sh
```

Or manually download from GitHub Releases:

```bash
# Get latest version tag
VERSION=$(curl -s https://api.github.com/repos/iemejia/fabio/releases/latest | grep '"tag_name"' | cut -d'"' -f4)

# Linux x64
curl -fsSL "https://github.com/iemejia/fabio/releases/download/${VERSION}/fabio-linux-x64.tar.gz" | tar xz -C /usr/local/bin

# Linux arm64
curl -fsSL "https://github.com/iemejia/fabio/releases/download/${VERSION}/fabio-linux-arm64.tar.gz" | tar xz -C /usr/local/bin

# macOS arm64 (Apple Silicon)
curl -fsSL "https://github.com/iemejia/fabio/releases/download/${VERSION}/fabio-macos-arm64.tar.gz" | tar xz -C /usr/local/bin

# Windows x64 (PowerShell)
# Invoke-WebRequest "https://github.com/iemejia/fabio/releases/download/${VERSION}/fabio-windows-x64.zip" -OutFile fabio.zip
# Expand-Archive fabio.zip -DestinationPath $env:USERPROFILE\.local\bin
```

### Build from Source (requires Rust 1.85+)

```bash
cargo install --git https://github.com/iemejia/fabio.git
```

## Authentication

fabio delegates authentication to the Azure credential chain. No built-in token storage.

```bash
# Step 1: Login via Azure CLI
az login

# Step 2: Verify fabio can authenticate
fabio auth status
```

Supported credential sources (via DefaultAzureCredential):
- Azure CLI (`az login`)
- Environment variables (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`)
- Managed Identity (when running on Azure)

## Companion Tools (Strongly Recommended)

fabio works best alongside these CLIs:

| Tool | Purpose | Install |
|------|---------|---------|
| `az` (Azure CLI) | **Required** for authentication (`az login`) and supplementary Azure operations (networking, IAM, storage accounts) that fall outside Fabric scope | [Install Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| `gh` (GitHub CLI) | Download fabio releases (`gh release download`), manage issues, and authenticate with GitHub repos for Fabric Git integration | [Install GitHub CLI](https://cli.github.com/) |

```bash
# Verify companions are available
az --version
gh --version

# Use gh to download the latest fabio release
gh release download --repo iemejia/fabio --pattern "fabio-linux-x64.tar.gz" --dir /tmp
tar xzf /tmp/fabio-linux-x64.tar.gz -C ~/.local/bin

# Use az for authentication (required before fabio works)
az login
fabio auth status
```

`az` is essential because fabio delegates all authentication to `DefaultAzureCredential`, which relies on `az login` session tokens. `gh` simplifies downloading fabio binaries and is useful when Fabric workspaces are connected to GitHub repositories via `fabio git connect`.

## Core Concepts

### Output Format

All commands produce structured JSON by default. The envelope format is:

```json
// List operations
{"data": [...], "count": N}

// Single object operations
{"data": {...}}

// Errors (on stderr, non-zero exit)
{"error": {"code": "NOT_FOUND", "message": "...", "hint": "..."}}
```

Use `-o table` for human-readable output, `-o plain` for one-value-per-line shell scripting.

### Error Codes

Machine-readable error codes: `AUTH_REQUIRED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `CAPACITY_INACTIVE`, `INVALID_INPUT`, `API_ERROR`, `TIMEOUT`, `NETWORK_ERROR`

Errors include `hint` fields with actionable suggestions for self-correction.

### Global Options

| Flag | Purpose |
|------|---------|
| `-o`, `--output` | `json` (default), `table`, `plain` |
| `--json` | Shorthand for `--output json` |
| `-q`, `--query` | Dot-notation field projection |
| `--quiet` | Suppress stdout (errors still on stderr) |
| `--force` | Skip confirmation for destructive ops |
| `--dry-run` | Preview mutations without executing |
| `--limit` | Limit list results |
| `--all` | Auto-paginate all pages |
| `--continuation-token` | Resume pagination |
| `--profile` | Use a named profile |

### Long-Running Operations (LRO)

Many operations are async. Use `--wait` to block until completion:

```bash
fabio notebook run --workspace $WS --id $NB --wait --timeout 600
```

LRO pattern: 2s polling interval, 120s max wait by default. Status transitions: `NotStarted` -> `InProgress` -> `Succeeded`/`Failed`.

### Agent Introspection

```bash
# Get machine-readable command schema
fabio agent-context
```

## Workflow: From Sign-In to Queryable Data

```bash
# 1. Authenticate
az login && fabio auth login

# 2. Create workspace with capacity
fabio workspace create --name "analytics"
fabio workspace assign-capacity --id $WS --capacity $CAP

# 3. Create lakehouse
fabio lakehouse create --workspace $WS --name "DataLake"

# 4. Upload data (glob patterns, parallel)
fabio lakehouse upload --workspace $WS --id $LH --source "data/*.csv" --dest Files/raw/

# 5. Load into Delta table
fabio lakehouse load-table --workspace $WS --id $LH \
  --path Files/raw/orders.csv --table orders --mode Overwrite --format Csv

# 6. Query via SQL
fabio warehouse query --workspace $WS --id $WH \
  --sql "SELECT country, SUM(revenue) FROM dbo.orders GROUP BY country"
```

## Critical API Behaviors

These are essential for correct operation. See [references/API-BEHAVIORS.md](references/API-BEHAVIORS.md) for complete details.

1. **load-table requires PascalCase**: Mode values are `Overwrite`/`Append`, format is `Csv`/`Parquet` (not lowercase). JSON format is NOT supported.
2. **Lakehouse tables key is `"data"`**: Unlike all other list endpoints which use `"value"`.
3. **KQL queries split by type**: Management commands (starting with `.`) route to `/v1/rest/mgmt`; data queries to `/v2/rest/query`. Token scope is `{kusto_uri}/.default`.
4. **Tenant-scoped endpoints**: Deployment Pipelines, Connections, Capacities, Gateways have NO `/workspaces/` prefix.
5. **OneLake has no native rename/move**: Must copy + delete. Blob API `PUT` with `x-ms-copy-source` for server-side copy.
6. **Definition operations are LRO**: Both `getDefinition` and `updateDefinition` use 202 + Location header polling.
7. **Notebook source must be array of strings**: The `.ipynb` format requires `source: ["line1\n", "line2\n"]`, not a single string.
8. **SQL Database needs F4+ capacity**: F2 fails with error 18456 State 240 for TDS connections.

## Command Groups

fabio has 37+ command groups covering the full Fabric API surface. See [references/COMMANDS.md](references/COMMANDS.md) for the complete reference.

**Core**: auth, workspace, item, lakehouse, capacity, catalog
**Data & Compute**: notebook, warehouse, sql-database, sql-endpoint, data-pipeline, copy-job, dataflow, environment, data-agent, ontology
**Analytics**: report, semantic-model, paginated-report, dashboard, datamart
**Real-Time Intelligence**: eventhouse, eventstream, kql-database, kql-queryset, kql-dashboard, reflex, anomaly-detector, event-schema-set
**Data Science**: ml-model, ml-experiment, operations-agent, spark, spark-job-definition, apache-airflow-job
**Graph & Digital Twins**: graphql-api, graph-model, graph-query-set, digital-twin-builder, digital-twin-builder-flow, map
**Mirroring**: mirrored-database, mirrored-catalog, mirrored-databricks-catalog, mirrored-warehouse, cosmos-db-database, snowflake-database, mounted-data-factory
**Integration**: git, connection, deployment-pipeline, domain, job-scheduler, variable-library, user-data-function
**Security**: onelake-security, managed-private-endpoint, gateway
**Admin**: admin (49 subcommands for tenant administration)
**Tooling**: profile, jobs, feedback, agent-context, operation

## Composability Patterns

```bash
# Pipe workspace ID into subsequent commands
WS=$(fabio workspace list --query id --limit 1 -o plain)

# Pipe SQL from file
fabio warehouse query --workspace $WS --id $WH --sql @queries/report.sql

# Pipe SQL from stdin
echo "SELECT COUNT(*) FROM dbo.orders" | fabio warehouse query --workspace $WS --id $WH

# Chain create + populate
fabio lakehouse create --workspace $WS --name "NewLake" | \
  jq -r '.data.id' | \
  xargs -I{} fabio lakehouse upload --workspace $WS --id {} --source "data/*" --dest Files/
```

## Throttling Awareness

- Prefer bulk/batch APIs over repeated individual calls
- Use `--all` for paginated lists instead of manual loop with `--continuation-token`
- The CLI handles rate-limit retry automatically for parallel operations
- Spark cold start on small capacity: 2-5 minutes for first notebook run

## Key URLs

- Fabric REST API: `https://api.fabric.microsoft.com/v1`
- OneLake DFS: `https://onelake.dfs.fabric.microsoft.com`
- OneLake Blob: `https://onelake.blob.fabric.microsoft.com`
- Fabric scope: `https://analysis.windows.net/powerbi/api/.default`
- Storage scope: `https://storage.azure.com/.default`
- Repository: `https://github.com/iemejia/fabio`
- Releases: `https://github.com/iemejia/fabio/releases`
