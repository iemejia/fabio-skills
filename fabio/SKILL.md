---
name: fabio
description: "Manage Microsoft Fabric artifacts and data using the fabio CLI - an agent-native command-line tool with 820+ subcommands across 75 groups, structured JSON output, composable piping, and machine-readable errors. Use when working with Fabric workspaces, lakehouses, warehouses, notebooks, eventhouses, semantic models, reports, data pipelines, KQL databases, eventstreams, deploy CI/CD, REST passthrough, Power BI API, capacity lifecycle, app-backend (Power Apps), data-build-tool-job (dbt), org-app (Organizational App), azure-databricks-storage (Azure Databricks integration), or any Fabric REST API resource. Covers CRUD operations, file upload/download, SQL/DAX/KQL queries, Git integration, deployment pipelines, CI/CD deploy (plan/apply/export/validate/config-file/git-diff), natural language to KQL, and administration."
license: MIT
compatibility: "Requires fabio binary (Linux/macOS/Windows x64/arm64). Authentication via `fabio auth login` (uses same Microsoft Identity platform as Azure CLI). Strongly recommended companions: az (Azure CLI) for supplementary Azure operations, gh (GitHub CLI) for release downloads. Network access to api.fabric.microsoft.com, api.powerbi.com, and onelake.dfs.fabric.microsoft.com required."
metadata:
  author: iemejia
  version: "0.27.0"
  repository: https://github.com/iemejia/fabio
---

# fabio — Agent-Native CLI for Microsoft Fabric

## Overview

`fabio` is a CLI designed for AI agents first, humans second. It manages the entire Microsoft Fabric platform (820+ subcommands across 75 groups) from the command line with structured JSON output, composable stdin/stdout piping, machine-readable error codes, and non-interactive operation.

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

### Docker (multi-arch: linux/amd64 + linux/arm64)

```bash
# Always latest stable release
docker pull ghcr.io/iemejia/fabio:latest

# Run with service principal auth via env vars
docker run --rm \
  -e AZURE_CLIENT_ID=$AZURE_CLIENT_ID \
  -e AZURE_CLIENT_SECRET=$AZURE_CLIENT_SECRET \
  -e AZURE_TENANT_ID=$AZURE_TENANT_ID \
  ghcr.io/iemejia/fabio:0.27.0 fabio workspace list
```

### Build from Source (requires Rust 1.85+)

```bash
cargo install --git https://github.com/iemejia/fabio.git
```

## Authentication

fabio has its own built-in OAuth2 authentication — no Azure CLI dependency required.

```bash
# Interactive device code flow (headless/SSH environments)
fabio auth login

# Browser-based PKCE flow (faster; SSO on macOS Enterprise Extension)
fabio auth login --browser

# Service principal with client secret
fabio auth login --service-principal --tenant <tid> --client-id <cid> --client-secret <secret>

# Service principal with certificate (PEM or PFX)
fabio auth login --service-principal --tenant <tid> --client-id <cid> --certificate <path> [--certificate-password <pw>]

# Service principal with federated token (OIDC/GitHub Actions)
fabio auth login --service-principal --tenant <tid> --client-id <cid> --federated-token <token>
# Or via file:
fabio auth login --service-principal --tenant <tid> --client-id <cid> --federated-token-file <path>

# Windows WAM broker SSO (Windows only — uses current Windows account)
fabio auth login --wam

# Verify authentication status
fabio auth status
```

Supported credential sources (via DefaultAzureCredential chain):
- **fabio auth login** (preferred — independent OAuth2, no `az` needed): device code, browser PKCE, service principal (secret/cert/federated), Windows WAM
- Environment variables (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`)
- Managed Identity (when running on Azure)
- Azure CLI (`az login`) as fallback

## Companion Tools (Strongly Recommended)

fabio works best alongside these CLIs:

| Tool | Purpose | Install |
|------|---------|---------|
| `az` (Azure CLI) | Supplementary Azure operations (networking, IAM, storage accounts) that fall outside Fabric scope. | [Install Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| `gh` (GitHub CLI) | Download fabio releases (`gh release download`), manage issues, and authenticate with GitHub repos for Fabric Git integration | [Install GitHub CLI](https://cli.github.com/) |

```bash
# Verify companions are available
az --version
gh --version

# Use gh to download the latest fabio release
gh release download --repo iemejia/fabio --pattern "fabio-linux-x64.tar.gz" --dir /tmp
tar xzf /tmp/fabio-linux-x64.tar.gz -C ~/.local/bin

# Authenticate with fabio
fabio auth login
fabio auth status
```

`fabio auth login` handles authentication independently using the Microsoft Identity platform. `az` remains useful for Azure operations outside Fabric scope (networking, IAM, storage). `gh` simplifies downloading fabio binaries and is useful when Fabric workspaces are connected to GitHub repositories via `fabio git connect`.

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

Use `-o table` for human-readable output, `-o plain` for one-value-per-line shell scripting, `-o csv` or `-o tsv` for tabular data export.

### Error Codes

Machine-readable error codes: `AUTH_REQUIRED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `CAPACITY_INACTIVE`, `INVALID_INPUT`, `API_ERROR`, `TIMEOUT`, `NETWORK_ERROR`

Errors include `hint` fields with actionable suggestions for self-correction.

### Global Options

| Flag | Purpose |
|------|---------|
| `-o`, `--output` | `json` (default), `table`, `plain`, `csv`, `tsv` |
| `--json` | Shorthand for `--output json` |
| `-q`, `--query` | JMESPath expression (jmespath.org) — use `[*].field` for list projection |
| `-v`, `--verbose` | HTTP/LRO/auth diagnostics to stderr (for debugging only) |
| `--quiet` | Suppress stdout (errors still on stderr) |
| `--force` | Skip confirmation for destructive ops |
| `--dry-run` | Preview mutations without executing |
| `--limit` | Limit list results |
| `--all` | Auto-paginate all pages |
| `--continuation-token` | Resume pagination |
| `--profile` | Use a named profile |
| `--lro-timeout` | Custom LRO polling timeout |
| `--hard-delete` | Permanently delete (skip recycle bin) — supported on all 38 item type delete commands |

### Long-Running Operations (LRO)

Many operations are async. Use `--wait` to block until completion:

```bash
fabio notebook run --workspace $WS --id $NB --wait --timeout 600
```

LRO pattern: 2s polling interval, 120s max wait by default. Status transitions: `NotStarted` -> `InProgress` -> `Succeeded`/`Failed`.

### Agent Introspection

```bash
# Get machine-readable command schema (v2 — includes auth_scope and returns for all commands)
fabio agent-context
```

Schema version 2 includes `auth_scope` (`fabric`, `arm`, or `local`) and `returns` (`list`, `object`, or `void`) annotations for all 592+ subcommands across all command groups, plus `output_fields`, `workflows`, `output_conventions`, and 35 item type `definition_paths`.

## Workflow: From Sign-In to Queryable Data

```bash
# 1. Authenticate
fabio auth login

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
4. **Tenant-scoped endpoints**: `deployment-pipeline` (distinct from `deploy`), Connections, Capacities, Gateways have NO `/workspaces/` prefix.
5. **OneLake atomic rename for same-item moves**: `move-file` and `move-table` use `x-ms-rename-source` header for O(1) metadata rename within the same lakehouse (returns 201). Automatically falls back to copy + delete for cross-item/cross-workspace moves (403 from rename attempt).
6. **Definition operations are LRO**: Both `getDefinition` and `updateDefinition` use 202 + Location header polling.
7. **Notebook source must be array of strings**: The `.ipynb` format requires `source: ["line1\n", "line2\n"]`, not a single string.
8. **SQL Database needs F4+ capacity**: F2 fails with error 18456 State 240 for TDS connections.
9. **Report visuals require PBIR-Legacy with prototypeQuery**: PBIR format cannot render data programmatically. Use `report.json` with `prototypeQuery` in each visual's config.
10. **Deploy uses content-hash diffing**: SHA-256 of sorted definition parts; no state file; always queries live workspace.
11. **Power BI and Fabric share one token**: The Fabric token (`https://api.fabric.microsoft.com/.default`) works for both `api.fabric.microsoft.com` and `api.powerbi.com`. No separate scope needed.
12. **Capacity lifecycle via ARM API**: suspend/resume/create/update/delete use `management.azure.com` with separate ARM scope — requires Azure RBAC (Contributor) on the capacity resource.

## Command Groups

fabio has 74 command groups with 800+ subcommands covering the full Fabric API surface. See [references/COMMANDS.md](references/COMMANDS.md) for the complete reference.

**Core**: auth, workspace, item, lakehouse, capacity, catalog, context
**Data & Compute**: notebook, warehouse, sql-database, sql-endpoint, data-pipeline (including schedule management: create/list/get/update/delete-schedule; and instance history: list-instances, get-instance), copy-job, dataflow, environment, data-agent, ontology
**Analytics**: report, semantic-model (including 12 Power BI API commands: clone, export-pbix, import-pbix, list-users, etc.), paginated-report, dashboard, datamart
**Real-Time Intelligence**: eventhouse, eventstream, kql-database, kql-queryset, kql-dashboard, reflex, anomaly-detector, event-schema-set, rti (nl-to-kql)
**Power Apps & Apps**: app-backend (preview — Power Apps backend services: list, show, create [LRO], update, delete [--hard-delete]), org-app (Organizational App packages), org-app-audience (audience targeting for Org Apps)
**Data Science**: ml-model, ml-experiment, operations-agent, spark, spark-job-definition, apache-airflow-job, data-build-tool-job (dbt integration, preview: list/show/create/update/delete/get-definition/update-definition/run [--wait])
**Graph & Digital Twins**: graphql-api, graph-model, graph-query-set, digital-twin-builder, digital-twin-builder-flow, map
**Mirroring**: mirrored-database, mirrored-catalog, mirrored-databricks-catalog, mirrored-warehouse, cosmos-db-database, snowflake-database, mounted-data-factory, azure-databricks-storage (AzureDatabricksStorage Fabric item: list/show/create/update/delete/get-definition/update-definition; definition format: AzureDatabricksStorageV1, part path: definition.json)
**Integration**: git, connection, deployment-pipeline (Fabric Deployment Pipeline resource — NOT `deploy`; tenant-scoped, no --workspace), domain, job-scheduler, variable-library, user-data-function
**CI/CD**: deploy (plan, apply, export, init-params, validate — stateless content-hash diffing, parameter substitution, rename detection, post-deploy hooks)
**Security**: onelake-security, managed-private-endpoint, gateway (including lifecycle: check-status, check-member-status, restart [LRO], shutdown [LRO])
**Admin**: admin (49 subcommands for tenant administration — tenant settings, workspaces, items, users, domains, tags, labels, sharing links, external data shares, workloads)
**Tooling**: profile, jobs, feedback, agent-context, operation, rest (raw REST passthrough with Power BI API support), completions (shell tab-completion scripts), upgrade (self-update: check/download/verify/replace binary)

## CI/CD Deployment (fabio deploy)

A stateless CI/CD engine that deploys Fabric items via content-hash diffing (no state file). Full fabric-cicd compatibility — source directories exported by Microsoft's [fabric-cicd](https://github.com/microsoft/fabric-cicd) Python library work identically with fabio, including YAML parameter files (`parameter.yml`).

```bash
# Export current workspace state to disk
fabio deploy export --workspace $WS --dir ./fabric-items --overwrite

# Validate source directory (local-only pre-flight checks, no API calls)
fabio deploy validate --source ./fabric-items

# Plan changes (compare source directory against live workspace)
fabio deploy plan --source ./fabric-items --workspace $WS

# Apply changes (create/update/rename items to match source)
fabio deploy apply --source ./fabric-items --workspace $WS

# Deploy only items changed since a git ref (selective deploy)
fabio deploy plan --source ./fabric-items --workspace $WS --git-diff main

# Deploy with config file (per-environment workspace mapping)
fabio deploy apply --config deploy.yaml --env prod

# Deploy with filtering
fabio deploy apply --source ./fabric-items --workspace $WS \
  --exclude-regex "^Test" --include-folders "/ETL,/Reports"

# Generate parameters.json for multi-environment deploys
fabio deploy init-params --source ./fabric-items --out parameters.json

# Deploy with environment-specific parameters
fabio deploy apply --source ./fabric-items --workspace $WS_PROD \
  --parameters parameters.json --env prod
```

Key behaviors:
- **Stateless**: Always queries live workspace (no `.tfstate` equivalent)
- **Content-hash diffing**: SHA-256 of sorted (path, payload) pairs detects actual changes
- **fabric-cicd compatible**: Parses `.children/` KQL discovery, `.pbi/` exclusion, Report `byPath`→`byConnection`, notebook ordering, `ItemDisplayNameNotAvailableYet` retry
- **Config file** (`--config deploy.yaml --env prod`): JSON or YAML with per-environment workspace mapping, filtering, and option defaults
- **Git-diff selective deploy** (`--git-diff <ref>`): Only deploys items changed since a git reference
- **Workspace folder management**: Infers folder hierarchy from source directory, creates/moves/deletes automatically
- **Workspace ID auto-replacement**: Replaces `00000000-...` placeholder with target workspace UUID (regex on workspace-reference keys only; opt-out: `--no-workspace-id-replace`)
- **Protected type deletion**: Lakehouse, Warehouse, SQLDatabase, Eventhouse, KQLDatabase require `--allow-delete-types` to be deleted
- **Rename detection**: Two-pass matching by (type, name) then by logicalId
- **Parallel execution**: Bounded concurrency (default 8) per type batch with rate-limit retry
- **46 item types in dependency order**: Storage → compute → code → models → reactive → APIs
- **Post-deploy hooks**: SemanticModel → refresh, Environment → publish (opt-out via `--no-post-hooks`)
- **Parameter substitution**: find_replace, key_value_replace, spark_pool, semantic_model_binding
- **Parameter file format**: `--parameters` accepts JSON (`.json`) or YAML (`.yml`/`.yaml`) — auto-detected by extension; fabric-cicd's `parameter.yml` files work directly
- **`.platform` in parts but excluded from hash**: Sent to API for metadata updates, but excluded from content hash (API rewrites `logicalId`, breaking idempotency if hashed)

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

- Prefer bulk/batch APIs over repeated individual calls (e.g., `item bulk-create`, `item bulk-delete`, workspace role batch-assign, domain batch-assign)
- Prefer list APIs over repeated single-resource requests (single list call + client-side filter rather than N individual show calls)
- Use `--all` for paginated lists instead of manual loop with `--continuation-token`
- The CLI handles rate-limit retry automatically for parallel operations
- Spark cold start on small capacity: 2-5 minutes for first notebook run
- Deploy uses bounded concurrency (default 8) with automatic rate-limit retry

## Key URLs

- Fabric REST API: `https://api.fabric.microsoft.com/v1`
- Power BI REST API: `https://api.powerbi.com/v1.0/myorg`
- OneLake DFS: `https://onelake.dfs.fabric.microsoft.com`
- OneLake Blob: `https://onelake.blob.fabric.microsoft.com`
- Fabric scope: `https://api.fabric.microsoft.com/.default`
- Storage scope: `https://storage.azure.com/.default`
- ARM scope: `https://management.azure.com/.default`
- Repository: `https://github.com/iemejia/fabio`
- Releases: `https://github.com/iemejia/fabio/releases`
