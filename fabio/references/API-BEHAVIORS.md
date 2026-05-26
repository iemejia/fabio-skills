# fabio API Behaviors & Quirks

Critical API behaviors that agents must know for correct operation. These are based on extensive testing against the live Fabric REST API.

## Output Envelope

### Standard Envelope
```json
// List commands
{"data": [...], "count": N}

// With pagination
{"data": [...], "count": N, "continuationToken": "eyJ..."}

// Single object
{"data": {"id": "...", "displayName": "...", ...}}
```

### Error Envelope (stderr)
```json
{"error": {"code": "NOT_FOUND", "message": "Item not found", "hint": "Run: fabio item list --workspace <ws>"}}
```

## Authentication & Token Scoping

| API Surface | Token Scope |
|---|---|
| Fabric REST API | `https://analysis.windows.net/powerbi/api/.default` |
| OneLake DFS/Blob | `https://storage.azure.com/.default` |
| KQL (Kusto) queries | `{kusto_cluster_uri}/.default` |
| SQL (TDS) queries | SQL-scoped AAD token via `require_sql_auth()` |
| Power BI REST API | `https://analysis.windows.net/powerbi/api/.default` |

## Endpoint Scoping

### Workspace-scoped (most commands)
```
POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/{itemType}s
GET  https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/{itemType}s/{itemId}
```

### Tenant-scoped (no workspace prefix)
- `fabio connection list/show/create/update/delete`
- `fabio capacity list/show`
- `fabio deployment-pipeline list/show/create/update/delete`
- `fabio gateway list/show/create/update/delete`
- `fabio domain list/show/create/update/delete`
- `fabio admin *` (all admin commands)

## Long-Running Operations (LRO)

### Standard Pattern
1. Client sends POST/PUT/PATCH
2. Server returns 202 with `Location` header and optional `x-ms-operation-id`
3. Client polls Location URL every 2 seconds
4. Terminal states: `Succeeded` or `Failed`
5. On success, follow the resource URL in the response

### LRO Commands
- All `get-definition` and `update-definition` operations
- `item copy`, `item move`, `item bulk-*`
- `notebook run` (with `--wait`)
- `git status`, `git commit`, `git pull`, `git init`
- `sql-endpoint refresh-metadata`
- `lakehouse bulk-create-shortcuts`

### Notebook Job States
`NotStarted` -> `InProgress` -> `Completed` | `Failed` | `Cancelled`

Cold start on small capacity: 2-5 minutes from `NotStarted` to `InProgress`.

## Lakehouse Operations

### load-table Critical Rules
- **Mode values are PascalCase**: `Overwrite` or `Append` (NOT `overwrite`)
- **Format is PascalCase**: `Csv` or `Parquet` (NOT `csv`)
- **JSON is NOT supported**: Must convert to CSV/Parquet first
- **`format` goes inside `formatOptions`**: The discriminated union requires format field in the options object
- **CSV-specific fields with Parquet cause rejection**: Never send `header`/`delimiter` with Parquet format

### list-tables Response Key
Lakehouse tables use `"data"` key in the response. All other Fabric list endpoints use `"value"`.

### OneLake File Operations

#### Upload (DFS 3-step)
1. `PUT /{ws}/{lh}/{path}?resource=file` (create)
2. `PATCH /{ws}/{lh}/{path}?action=append&position=0` (append data)
3. `PATCH /{ws}/{lh}/{path}?action=flush&position={size}` (flush/commit)

#### Server-Side Copy (Blob API)
```
PUT https://onelake.blob.fabric.microsoft.com/{ws}/{lh}/{dest-path}
x-ms-copy-source: https://onelake.blob.fabric.microsoft.com/{ws}/{lh}/{src-path}
```
Returns 202 with pending status. Poll via HEAD for completion.

#### No Native Rename/Move
OneLake rejects `x-ms-rename-source` header. Move = copy + delete.

#### Recursive Delete
```
DELETE https://onelake.dfs.fabric.microsoft.com/{ws}/{lh}/Tables/{name}?recursive=true
```

#### File Listing Quirk
When `directory` parameter is specified in DFS listing, paths appear doubled (e.g., `Files/Files/myfile.csv`). fabio normalizes this automatically. Use root listing (no `directory` param) to get real paths prefixed with item ID.

### Sync Command
Compares source and destination using ETag/MD5. Only copies new/modified files. `--delete` removes files in destination that don't exist in source.

## Warehouse & SQL Database

### Query Input Methods
```bash
# Inline SQL
fabio warehouse query --workspace $WS --id $WH --sql "SELECT 1"

# From file (prefix with @)
fabio warehouse query --workspace $WS --id $WH --sql @queries/report.sql

# From stdin (pipe)
echo "SELECT 1" | fabio warehouse query --workspace $WS --id $WH
```

### SQL Database Import Type Inference
Type widening rules (never narrows):
- `Unknown` -> first observation sets type
- `Int` -> `BigInt` -> `Float` -> `NVarChar` (progressive widening)
- NVarChar length: `clamp(observed_max_len * 2, 50, 4000)`
- Batch size: 100 rows per INSERT (configurable via `--batch-size`)
- Timeout: 120s per batch

### SQL Database Capacity Requirement
F4+ capacity required for TDS connections. F2 fails with error 18456 State 240.

## KQL Database Queries

### Query Routing
- Management commands (starting with `.`): `POST {kusto_uri}/v1/rest/mgmt`
- Data queries: `POST {kusto_uri}/v2/rest/query`

### Token Scope
KQL uses a non-standard scope: `{kusto_cluster_uri}/.default` (NOT the standard Fabric scope).

### KQL Queryset Definition Format
```json
{
  "queryset": {
    "version": "1.0.0",
    "dataSources": [{"id": "...", "clusterUri": "...", "type": "...", "databaseName": "..."}],
    "tabs": [{"id": "...", "content": "KQL query\\nwith newlines", "title": "Tab Name", "dataSourceId": "..."}]
  }
}
```
Tab selection in `kql-queryset run` is case-insensitive by title.

## Semantic Model

### Format Requirements
| Scenario | Format | Key Requirement |
|---|---|---|
| Direct Lake | TMDL | Required for `mode: directLake` partitions |
| Import mode | model.bim | `compatibilityLevel: 1604` + `powerBI_V3` |

### Direct Lake Key Points
- Connection flag `--connection` takes the SQL Analytics Endpoint ID (NOT lakehouse ID)
- After creation, call `refresh` to frame the model (without framing, DAX queries fail with error 3242524690)
- `Sql.Database()` second parameter must be SQL endpoint ID (not lakehouse ID)
- Storage mode must be `Abf` (NOT `PremiumFiles`)
- Call `takeover` after creation to make editable in portal

### definition.pbism Format
```json
// For TMDL (v4.2)
{"$schema": "https://developer.microsoft.com/json-schemas/fabric/item/semanticModel/definitionProperties/1.0.0/schema.json", "version": "4.2", "settings": {}}

// For model.bim (v3.0)
{"version": "3.0"}
```

## Report Definitions

### Format Selection
- **PBIR-Legacy** (`report.json`): Required for programmatic visuals that render data (uses `prototypeQuery`)
- **PBIR** (`definition/` folder): Better for source control, but cannot render data programmatically

### definition.pbir (Always Required)
```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
  "version": "4.0",
  "datasetReference": {
    "byConnection": {
      "connectionString": "semanticmodelid=<SEMANTIC-MODEL-UUID>"
    }
  }
}
```

### updateDefinition Rules
- ALWAYS include `definition.pbir` part, even if only updating `report.json`
- Can switch formats by sending appropriate parts
- `prototypeQuery` is REQUIRED in visual config for data to render

## Eventstream

### add-source/add-destination Pattern
These high-level commands fetch the current definition, merge the new node, and update. They abstract the complex eventstream topology.

### Destination Properties (JSON)
```json
{
  "dataIngestionMode": "DirectIngestion",
  "workspaceId": "<ws-id>",
  "itemId": "<kql-db-id>",
  "tableName": "TableName",
  "connectionName": "conn-name",
  "mappingRuleName": "JsonMapping"
}
```

## Ontology API

### Critical: JSON Key Ordering
The Fabric Ontology API uses ordered JSON deserialization for data bindings. `sourceType` MUST be the first key in `sourceTableProperties`. fabio normalizes this automatically via `normalize_data_binding()`.

### Data Binding Structure
```json
{
  "id": "<uuid-format-required>",
  "dataBindingConfiguration": {
    "dataBindingType": "NonTimeSeries",
    "sourceTableProperties": {
      "sourceType": "LakehouseTable",
      "workspaceId": "...",
      "itemId": "...",
      "sourceTableName": "...",
      "sourceSchema": "dbo"
    },
    "propertyBindings": [
      {"sourceColumnName": "col", "targetPropertyId": "prop-id"}
    ]
  }
}
```

## Data Agent API

### Definition Persistence Rule
Single-part `updateDefinition` with only a datasource file is silently dropped. Must include ALL parts together:
- `data_agent.json` + `stage_config.json` + `datasource.json`

### Data Source Types
`unknown`, `lakehouse_tables`, `lakehouse`, `data_warehouse`, `kusto`, `semantic_model`, `graph`, `mirrored_database`, `mirrored_azure_databricks`

### Publishing
Publishing is portal-only. No REST API endpoint for publish. The portal "Publish" button activates the server-side chat endpoint.

## Definition Operations (Generic Pattern)

### getDefinition
```
POST /workspaces/{ws}/{type}s/{id}/getDefinition
Body: {}
```
Returns 202 (LRO). Result contains base64-encoded definition parts:
```json
{
  "definition": {
    "parts": [
      {"path": "file.json", "payload": "<base64>", "payloadType": "InlineBase64"}
    ]
  }
}
```

### updateDefinition
```
POST /workspaces/{ws}/{type}s/{id}/updateDefinition
Body: {"definition": {"parts": [{"path": "...", "payload": "<base64>", "payloadType": "InlineBase64"}]}}
```

## Job Types (for job-scheduler run-on-demand)
- Notebook: `RunNotebook`
- Data Pipeline: `Pipeline` (PascalCase)
- Spark Job: `sparkjob` (lowercase)

## Connection Parameters Format
User-provided JSON `{"key": "value"}` is converted to the API format:
```json
[{"dataType": "Text", "name": "key", "value": "value"}]
```

## Naming Restrictions
- Digital Twin Builder names: no hyphens allowed
- MirroredAzureDatabricksCatalog names: no hyphens allowed
- Workspace names: most characters allowed
- Item names: alphanumeric + spaces + underscores (varies by type)

## Rate Limiting
- Spark on small capacity: LRO reports 430 `TooManyRequestsForCapacity`
- fabio retries automatically for parallel operations
- Prefer bulk APIs over repeated individual calls

## Capacity Requirements
- SQL Database TDS: requires F4+ capacity
- Spark notebooks: any capacity (but cold start delay on small)
- Most Fabric operations: any capacity with active state
- `CAPACITY_INACTIVE` error if workspace capacity is paused/deallocated

## Apache Airflow Specifics
- File operations require `?beta=true` query parameter
- File upload uses `Content-Type: text/plain` (JSON body rejected)
- DAGs are files, not definitions

## OneLake Security
- `upsert` replaces ALL roles atomically (PUT semantics, not PATCH)
- No individual role create/update — always send the full set

## Git Integration (Azure DevOps)

### Cross-Service Identity Requirement
Fabric's git integration uses the authenticated user's identity to access Azure DevOps. The user (OID from the Fabric token) must be:
1. A member of the Azure DevOps organization
2. Have at least Contributor access to the project/repo

Without this, `git connect` returns `InsufficientPrivileges` (403). The error looks like a workspace permission issue but is actually Azure DevOps rejecting the identity.

### Same AAD Tenant Required
The Azure DevOps organization must be backed by (connected to) the same Azure AD tenant as the Fabric workspace. Cross-tenant git integration is not supported with "Automatic" credentials.

### Azure DevOps vs GitHub Credentials
| Provider | Connection ID | Credential Mode |
|----------|---------------|-----------------|
| `azure-devops` | NOT required | "Automatic" — Fabric uses caller's OAuth token directly |
| `github` | ALWAYS required | Must pre-configure a `GitHubSourceControl` connection |

### `directoryName` is Required
The Fabric API rejects `git connect` without a `directoryName` field in `gitProviderDetails`. fabio defaults to `"/"` (repo root). Omitting it returns: `InvalidInput: The DirectoryName field is required.`

### Permission Propagation Delay
After adding a user to an Azure DevOps org/project, permissions take 5-10 seconds to propagate. Fabric's git connect can fail with 403 immediately after granting access. Retry after a brief wait.

### Multiple Workspaces on Same Repo
Different Fabric workspaces can connect to the same Azure DevOps repo and branch (same `directoryName`). Each workspace maintains independent sync state. Useful for CI/CD workspace-per-environment patterns.

### Repo Must Have a Branch
Azure DevOps repos without any commits have no `defaultBranch`. You must push an initial commit to create `main` before Fabric can connect.

## Gateway Operations
- PATCH requires `"type"` field in body or fails silently
- Non-existent principal returns 500 (not clean validation)
- Duplicate role assignment returns 409 with typo "assignemnt" in error message
