---
name: fabio
description: "Manage Microsoft Fabric artifacts and data using the fabio CLI - an agent-native command-line tool with 851+ subcommands across 74 groups, structured JSON output, composable piping, and machine-readable errors. Use when working with Fabric workspaces, lakehouses, warehouses, notebooks, eventhouses, semantic models, reports, data pipelines, KQL databases, eventstreams, deploy CI/CD, REST passthrough, Power BI API, capacity lifecycle, app-backend (Power Apps), data-build-tool-job (dbt), org-app (Organizational App), azure-databricks-storage (Azure Databricks integration), or any Fabric REST API resource. Covers CRUD operations, file upload/download, SQL/DAX/KQL queries, Git integration, deployment pipelines, CI/CD deploy (plan/apply/export/validate/config-file/git-diff), natural language to KQL, KQL schema discovery and diagnostics, and administration."
license: MIT
compatibility: "Requires fabio binary (Linux/macOS/Windows x64/arm64). Authentication via `fabio auth login` (uses same Microsoft Identity platform as Azure CLI). Network access to api.fabric.microsoft.com, api.powerbi.com, and onelake.dfs.fabric.microsoft.com required."
metadata:
  author: iemejia
  version: "0.31.0-dev"
  repository: https://github.com/iemejia/fabio
---

# fabio — Agent-Native CLI for Microsoft Fabric

## Quick Start

```bash
# Install (auto-detect OS/arch)
bash scripts/install.sh
# Or: cargo install --git https://github.com/iemejia/fabio.git

# Upgrade if already installed
fabio upgrade

# Authenticate (no Azure CLI dependency)
fabio auth login
fabio auth status
```

## Runtime Discovery (Preferred Over Reading Docs)

fabio has built-in introspection. Use these commands instead of reading reference files:

```bash
# Find commands — compact index of all 74 groups + subcommands
fabio context agent

# Deep-dive on one command — all flags, types, output shape
fabio context describe <group> <command>

# Search commands by keyword
fabio context find "upload"

# Multi-step workflow recipes
fabio context workflow <name>
# Available: lakehouse-etl, rti-pipeline, direct-lake-report, cicd-deploy, data-agent-setup

# Best practices
fabio context best-practices <topic>
# Available: throttling, lro, pagination, admin-apis, shortcuts

# Item definition format (for create/update-definition)
fabio context schema <type>

# Output shape example for a specific command
fabio context examples <group> <command>

# List all discoverable topics
fabio context list
```

## Output & Errors

```json
// List operations
{"data": [...], "count": N}

// Single object
{"data": {...}}

// Errors (stderr, non-zero exit)
{"error": {"code": "NOT_FOUND", "message": "...", "hint": "..."}}
```

Error codes: `AUTH_REQUIRED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT`, `RATE_LIMITED`, `CAPACITY_INACTIVE`, `INVALID_INPUT`, `API_ERROR`, `TIMEOUT`, `NETWORK_ERROR`

## Global Flags

| Flag | Purpose |
|------|---------|
| `-o`, `--output` | `json` (default), `table`, `plain`, `csv`, `tsv` |
| `-q`, `--query` | JMESPath expression for field projection |
| `--dry-run` | Preview mutations without executing |
| `--all` | Auto-paginate all pages |
| `--limit` | Limit list results |
| `--quiet` | Suppress stdout |
| `--wait` | Block until async job completes |
| `--timeout` | Timeout for `--wait` (seconds) |
| `--profile` | Use a named profile |
| `--hard-delete` | Permanently delete (skip recycle bin) |
| `--lro-timeout` | LRO polling timeout (default: 120s) |

## Authentication

```bash
# Device code (headless/SSH)
fabio auth login

# Browser PKCE (faster, SSO on macOS)
fabio auth login --browser

# Service principal (CI/CD)
fabio auth login --service-principal --tenant <T> --client-id <C> --client-secret <S>

# Service principal with federated token (GitHub Actions OIDC)
fabio auth login --service-principal --tenant <T> --client-id <C> --federated-token-file <path>

# Windows WAM broker
fabio auth login --wam
```

Credential chain: fabio cache > env vars > managed identity > Azure CLI > Azure Developer CLI

## Critical API Behaviors (Must-Know)

These cause silent failures if ignored:

1. **PascalCase values required** — `Overwrite` not `overwrite`, `Csv` not `csv`, `Parquet` not `parquet`. Load-table ONLY supports Csv and Parquet (not JSON).
2. **Tenant-scoped commands** — `deployment-pipeline`, `connection`, `capacity`, `domain`, `gateway` have NO `--workspace` flag. They operate at tenant level.
3. **LRO awareness** — Create, getDefinition, updateDefinition return 202. Jobs use `--wait` + `--timeout`. Default LRO poll: 2s interval, 120s max.
4. **Token sharing** — Same Fabric token (`https://api.fabric.microsoft.com/.default`) works for Power BI API. Use `fabio rest call --api powerbi` for Power BI endpoints.
5. **KQL uses separate scope** — KQL database queries scope to `{kusto_uri}/.default`, not the standard Fabric scope. Management commands (`.show`) use `/v1/rest/mgmt`; data queries use `/v2/rest/query`.
6. **OneLake atomic rename** — `move-file` and `move-table` use O(1) metadata rename within same lakehouse. Falls back to copy+delete for cross-item moves.
7. **Notebook source format** — `.ipynb` cells require `source: ["line1\n", "line2\n"]` (array of strings, not single string).
8. **Deploy is stateless** — Content-hash diffing against live workspace. No state file. SHA-256 of sorted (path, payload) pairs.
9. **Hard delete on 38 item types** — `--hard-delete` flag permanently removes items (skips recycle bin). Without it, items go to recycle bin.
10. **SQL Database needs F4+ capacity** — F2 fails with error 18456 State 240 for TDS connections.
11. **Report visuals need PBIR-Legacy** — PBIR format cannot render data programmatically. Use `report.json` with `prototypeQuery`.
12. **ARM scope for capacity lifecycle** — suspend/resume/create/delete use `management.azure.com` with Azure RBAC (Contributor role required).

## Composability Patterns

```bash
# Extract a single value
WS=$(fabio workspace list --query 'data[0].id' -o plain)

# Pipe SQL from file
fabio warehouse query --workspace $WS --id $WH --sql @queries/report.sql

# Pipe SQL from stdin
echo "SELECT COUNT(*) FROM dbo.orders" | fabio warehouse query --workspace $WS --id $WH

# Chain create + use
ID=$(fabio lakehouse create --workspace $WS --name "Lake" --query 'data.id' -o plain)
fabio lakehouse upload --workspace $WS --id $ID --source "data/*.csv" --dest Files/raw/
```

## Throttling Awareness

- Prefer bulk/batch APIs: `item bulk-create`, `item bulk-delete`, workspace role batch-assign
- Prefer list APIs + client-side filter over N individual show calls
- Use `--all` for paginated lists (not manual loops with `--continuation-token`)
- Rate-limit retry is automatic for parallel operations
- Deploy uses bounded concurrency (default 8) with rate-limit retry

## Key URLs

| Endpoint | URL |
|----------|-----|
| Fabric REST API | `https://api.fabric.microsoft.com/v1` |
| Power BI REST API | `https://api.powerbi.com/v1.0/myorg` |
| OneLake DFS | `https://onelake.dfs.fabric.microsoft.com` |
| Fabric scope | `https://api.fabric.microsoft.com/.default` |
| Storage scope | `https://storage.azure.com/.default` |
| ARM scope | `https://management.azure.com/.default` |
