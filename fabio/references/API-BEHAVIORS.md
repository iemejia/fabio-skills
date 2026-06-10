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
| Fabric REST API | `https://api.fabric.microsoft.com/.default` |
| OneLake DFS/Blob | `https://storage.azure.com/.default` |
| KQL (Kusto) queries | `{kusto_cluster_uri}/.default` |
| SQL (TDS) queries | SQL-scoped AAD token via `require_sql_auth()` |
| Power BI REST API | `https://api.fabric.microsoft.com/.default` (same token — reused) |
| ARM API (capacity lifecycle) | `https://management.azure.com/.default` |

### Authentication Methods

All login methods share the same `~/.fabio/token_cache.json` cache. On Windows, the cache is encrypted with DPAPI (matching Azure CLI behavior).

| Method | Command | Notes |
|--------|---------|-------|
| Device code | `fabio auth login` | Headless/SSH; user must visit URL and enter code |
| Browser PKCE | `fabio auth login --browser` | Faster; SSO on macOS with Enterprise Extension |
| Service principal (secret) | `fabio auth login --service-principal --tenant T --client-id C --client-secret S` | CI/CD |
| Service principal (cert PEM) | `fabio auth login --service-principal --tenant T --client-id C --certificate /path/cert.pem` | |
| Service principal (cert PFX) | `fabio auth login --service-principal --tenant T --client-id C --certificate /path/cert.pfx --certificate-password pw` | |
| Federated token (OIDC) | `fabio auth login --service-principal --tenant T --client-id C --federated-token <jwt>` | GitHub Actions OIDC |
| Federated token file | `fabio auth login --service-principal --tenant T --client-id C --federated-token-file /path/token` | File is trimmed of whitespace |
| Windows WAM broker | `fabio auth login --wam` | Windows only; SSO with current Windows account; no browser/code |

**SP error handling**: Empty strings for `--tenant`, `--client-id`, `--client-secret`, `--certificate`, `--federated-token` are treated as "not provided" with structured JSON error output.

**Security**: `--verbose` output and `--dry-run` previews automatically redact sensitive JSON fields (password, client_secret, credentials, access_token, key, connectionString, etc.) before logging. Redaction is recursive and case-insensitive.

### CI/CD Authentication

`DefaultAzureCredential` with client secret environment variables works correctly in CI as of v0.16.0. Set these three variables before running fabio:

```bash
export AZURE_CLIENT_ID="<app-id>"
export AZURE_TENANT_ID="<tenant-id>"
export AZURE_CLIENT_SECRET="<secret>"
fabio auth status   # confirms env-var credential source
```

Or use `fabio auth login --service-principal` directly (credentials stored in token cache):

```bash
fabio auth login --service-principal --tenant $TENANT_ID --client-id $CLIENT_ID --client-secret $CLIENT_SECRET
```

**GitHub Actions — OIDC federated credentials (recommended, secretless):**

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: azure/login@v3
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      allow-no-subscriptions: true   # Fabric-only auth doesn't need a subscription
  - run: fabio workspace list
```

**GitHub Actions — service principal with client secret (simpler, no extra actions):**

```yaml
steps:
  - env:
      AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
      AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
    run: fabio workspace list
```

> **Fix in v0.16.0**: Prior versions panicked at runtime with "The reqwest feature is required to use the default HTTP client" when using client secret env vars. The `reqwest` and `tokio` features are now enabled on `azure_identity`/`azure_core`.

## Query Filtering (--query / JMESPath)

The `--query` / `-q` flag uses full **JMESPath** expressions (see [jmespath.org](https://jmespath.org)).

**Breaking change in v0.18.0**: `--query` on lists now requires explicit `[*].field` syntax. Old dot-notation implicit array projection no longer works.

```bash
# List projection (REQUIRED [*] prefix for lists):
fabio workspace list --query '[*].displayName'
fabio lakehouse list-tables --workspace $WS --id $LH --query '[*].name'

# Filter expressions:
fabio item list --workspace $WS --query '[?type==`Notebook`].displayName'

# Pipe and functions:
fabio workspace list --query 'length(@)'
fabio workspace list --query 'sort_by(@, &displayName)[*].id'

# Nested fields still work (backward-compatible):
fabio workspace show --id $WS --query 'data.displayName'
```

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

#### Atomic Rename for Same-Item Moves (DFS API)
```
PUT https://onelake.dfs.fabric.microsoft.com/{ws}/{lh}/{dest-path}
x-ms-rename-source: /{ws}/{lh}/{src-path}
x-ms-version: 2021-06-08
```
Returns **201** on success (O(1) metadata operation — no data transfer). Works for **both files and directories** (entire table directory tree renamed atomically). **Fails with 403** for cross-item or cross-workspace moves (auth scope mismatch) — fabio automatically falls back to copy + delete in that case.

#### Server-Side Copy (Blob API)
```
PUT https://onelake.blob.fabric.microsoft.com/{ws}/{lh}/{dest-path}
x-ms-copy-source: https://onelake.blob.fabric.microsoft.com/{ws}/{lh}/{src-path}
```
Returns 202 with pending status. Poll via HEAD for completion.

#### No Native Rename/Move (cross-item only)
For cross-item or cross-workspace moves, OneLake rejects `x-ms-rename-source`. These use copy + delete. Same-item moves use atomic rename (see above).

#### Recursive Delete
```
DELETE https://onelake.dfs.fabric.microsoft.com/{ws}/{lh}/Tables/{name}?recursive=true
```

#### File Listing Quirk
When `directory` parameter is specified in DFS listing, paths appear doubled (e.g., `Files/Files/myfile.csv`). fabio normalizes this automatically. Use root listing (no `directory` param) to get real paths prefixed with item ID.

### Sync Command
Compares source and destination using ETag/MD5. Only copies new/modified files. `--delete` removes files in destination that don't exist in source.

#### Rename Detection
When `--delete` is active, `lakehouse sync` detects renamed/moved files and performs atomic O(1) renames at the destination instead of copy + delete:

- **ETag-based** (zero extra API calls): When source-only and dest-only files match by ETag + size, the file is renamed atomically. Works for files uploaded with `fabio lakehouse upload` (which stores Content-MD5 on flush, preserving ETag across renames).
- **Checksum-based** (`--checksum --delete`): A second pass compares Content-MD5 via HEAD requests. Only fires when exactly one candidate matches. Handles Fabric-generated files (Spark/pipelines) that lack Content-MD5, falling back to unique-size matching.

Output includes a `"renamed"` count for files handled via atomic rename.

**Note**: Fabric-generated files (Spark, data pipelines, load-table) do NOT have Content-MD5 stored. Their ETags change on rename, so they are not detectable by the ETag pass — only by checksum + unique-size fallback.

#### Content-MD5 on Upload
`lakehouse upload` stores an MD5 hash via `x-ms-content-md5` header on DFS flush. OneLake preserves this hash across server-side copy and atomic rename, enabling content-based matching.

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

### Publishing via CI/CD (Officially Supported)
`fabio data-agent publish` copies all `Files/Config/draft/*` parts to `Files/Config/published/*` and adds `publish_info.json` via `updateDefinition`. This is the officially documented CI/CD publish path (confirmed June 2026 — no portal required).

```bash
fabio data-agent publish --workspace $WS --id $DA
```

The publish_info.json format:
```json
{"$schema": "https://developer.microsoft.com/json-schemas/fabric/item/dataAgent/definition/publishInfo/1.0.0/schema.json", "description": "<publish description>"}
```

Publishing activates the OpenAI Assistants-compatible endpoint:
`https://api.fabric.microsoft.com/v1/workspaces/{wsId}/dataagents/{agentId}/aiassistant/openai`

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
- VNet gateway creation takes 60-90 seconds (no LRO — returns 201 directly after delay)
- Roles: `Admin`, `ConnectionCreator`, `ConnectionCreatorWithResharing`
- Cannot demote last Admin (returns error)
- `inactivityMinutesBeforeSleep` must be one of: 30, 60, 90, 120, 150, 240, 360, 480, 720, 1440
- `numberOfMemberGateways` must be 1-9

## Admin API

### Required Permissions
All admin endpoints require **Fabric Admin** role (tenant-level). Standard workspace Admin/Member roles are NOT sufficient. Errors include: `"The caller does not have sufficient scopes to perform this operation"`.

### Non-Standard Response Keys
Unlike most Fabric APIs that use `"value"` as the array key, admin endpoints use varied keys:

| Endpoint | Response Array Key |
|----------|-------------------|
| `/admin/workspaces` | `"workspaces"` |
| `/admin/items` | `"itemEntities"` |
| `/admin/workspaces/{id}/users` | `"accessDetails"` |
| `/admin/items/{id}/users` | `"accessDetails"` |
| `/admin/users/{id}/access` | `"accessEntities"` |
| `/admin/domains` | `"domains"` |
| `/admin/tenantsettings` | `"tenantSettings"` |
| `/admin/tags` | `"value"` (standard) |
| `/admin/workloads` | `"value"` (standard) |
| `/admin/workloads/assignments` | `"value"` (standard) |

fabio normalizes all these into the standard `{"data": [...], "count": N}` envelope.

### Admin Workspace Fields
Admin workspace responses use `name` (NOT `displayName`). Fields: `id`, `name`, `state`, `type`, `capacityId`, `tags`.

### Admin Item Fields
Admin item responses use `name` (NOT `displayName`). Fields: `id`, `type`, `name`, `state`, `lastUpdatedDate`, `creatorPrincipal`, `workspaceId`, `capacityId`.

### Tenant Settings

**Structure:**
```json
{
  "settingName": "ExportToImage",
  "title": "Export to image",
  "enabled": true,
  "tenantSettingGroup": "Export and sharing settings",
  "canSpecifySecurityGroups": true,
  "delegateToCapacity": false,
  "delegateToDomain": false,
  "enabledSecurityGroups": [],
  "excludedSecurityGroups": []
}
```

**Update body (minimum):**
```json
{"enabled": true}
```

**Update response:** Returns ALL settings in the SAME group (not just the updated one).

**Capacity override rules:**
- Only settings with `"delegateToCapacity": true` can have capacity-level overrides
- Attempting to override a non-delegatable setting returns: "The request could not be processed due to missing or invalid information"
- Override body: `{"enabled": true|false, "delegateToWorkspace"?: bool}`

**Domain override rules:**
- Only settings with `"delegateToDomain": true` can have domain-level overrides
- Same pattern as capacity overrides

### Tag Operations

**Create body:**
```json
{"createTagsRequest": [{"displayName": "Production"}]}
```
Optional scope: `{"type": "Tenant"}` or `{"type": "Domain", "domainId": "<uuid>"}`.

**Response:** `{"tags": [{"id": "...", "displayName": "...", "scope": {...}}]}`

**Rate limits:** Tag operations limited to 25 requests/minute.

### Domain Workspace Assignment

**By capacities:** Assigns ALL workspaces on that capacity to the domain.
```json
{"capacitiesIds": ["<uuid>"]}
```

**By principals:** Assigns all workspaces owned/administered by those principals.
```json
{"principals": [{"id": "<uuid>", "type": "User"}]}
```
Requires `--principal-type` flag.

**Additive behavior:** `assign-domain-workspaces-by-principals` only assigns workspaces NOT already assigned to another domain.

### Domain Role Sync
- `sync-domain-roles-to-subdomains` requires `--role` flag
- Only Contributors can be synced — syncing Admins returns: "Syncing admins to subdomains is not supported"

### Bulk Role Assignment
```json
{"type": "Contributors", "principals": [{"id": "<uuid>", "type": "User"}]}
```
Type values: `"Contributors"` or `"Admins"`.

### Sharing Links (LRO)
Both sharing link commands are LRO (return 202, must poll):
- `remove-all-sharing-links`: `{"sharingLinkType": "OrgLink"}` — type values: `OrgLink`, `GuestLink`, `AnonymousLink`, `SpecificPeopleLink`
- `bulk-remove-sharing-links`: Only supports Report type. Other types return "not supported for the requested item type"

### Labels (Microsoft Purview Required)
- `bulk-set-labels` requires M365 E5 licensing + Purview label policy configured in tenant
- `bulk-remove-labels` works without Purview (returns per-item status even if no label set)

### External Data Shares
- `list-external-data-shares` requires tenant setting `AllowExternalDataSharingSwitch` enabled
- Without it: FORBIDDEN with "tenant setting 'External data sharing' is disabled"

### Workload Assignment Body Format
Discriminated union with `type` field:
```json
// Tenant-level
{"type": "Tenant", "workloadId": "<id>"}

// Capacity-level
{"type": "Capacity", "workloadId": "<id>", "capacityId": "<uuid>"}

// Workspace-level
{"type": "Workspace", "workloadId": "<id>", "workspaceId": "<uuid>"}
```

### Workspace Restore
- `POST /admin/workspaces/{id}/restore` with `{"restoredWorkspaceName": "<name>", "capacityId": "<uuid>"}`
- Note: The `restoredWorkspaceName` parameter is ignored by server — workspace keeps original name

### Temporary Admin Access
- `grant-admin-access` / `remove-admin-access` manage TEMPORARY admin access only
- Returns NOT_FOUND if the caller already has permanent Admin access to the workspace

### Admin Error Enrichment
fabio provides 6 targeted error patterns for admin commands:
1. **External data sharing disabled** → exact setting name + CLI enable command
2. **Tenant setting disabled** → Admin Portal path + CLI command
3. **Item type not supported** → only Report type works for sharing link removal
4. **Purview labels not configured** → M365 E5 + licensing prerequisites
5. **Feature not available** → tenant admin feature flag guidance
6. **Sync admins not supported** → suggests `--role Contributor`

## Deploy Command

### Stateless Content-Hash Diffing
Deploy uses SHA-256 content hashing over sorted `path + \x00 + payload` pairs to detect changes. No state file exists — always queries the live workspace. No `.tfstate` equivalent.

### Source Directory Format
```
{DisplayName}.{ItemType}/
├── .platform                    # Required: {"$schema":"...","metadata":{"type":"...","displayName":"..."},"config":{"logicalId":"..."}}
├── definition-part.json         # Base64 payload parts (varies by item type)
└── creationPayload.json         # Optional: merged into creation body as `creationPayload` field
```

### Workspace Resolution
- GUID detection: 36 characters with 4 dashes → used directly as workspace ID
- Display name: resolved via `GET /workspaces?displayName=<name>` lookup

### Changeset Actions
| Action | Behavior |
|--------|----------|
| `Create` | POST to create item (with definition if present) |
| `Update` | POST updateDefinition (content hash differs) |
| `Rename` | PATCH displayName + updateDefinition |
| `Delete` | DELETE item (sequential, never parallel) |
| `Skip` | Content hash matches — no action needed |

### Rename Detection
Two-pass matching algorithm:
1. **First pass**: Match source items to deployed items by `(type, name)` pairs
2. **Second pass**: Unmatched source items with `logicalId` in `.platform` get candidates checked via `getDefinition` on deployed items — compares `logicalId` from their `.platform` part

### Logical ID Resolution
String replacement (`String::replace`) in base64 payloads at apply time. Resolves items created earlier in the same deploy session. Example: a report referencing a semantic model created in the same batch.

### Parameter Substitution
Applied in order (each stage feeds into the next):
1. **find_replace** — simple string replacement in payloads
2. **key_value_replace** — structured key-value pairs
3. **spark_pool** — Spark pool name/ID substitution
4. **semantic_model_binding** — semantic model ID replacement in report bindings

### Post-Deploy Hooks
| Item Type | Hook Action | Notes |
|-----------|-------------|-------|
| SemanticModel | `POST /refreshes` | Frames Direct Lake models |
| Environment | `POST /staging/publish` | Publishes staged changes |

- Failures are **non-fatal** (reported in output, don't fail the deploy)
- Hooks **never fire** during `--dry-run`
- Opt-out via `--no-post-hooks`

### Plan Staleness Detection
Workspace fingerprint = SHA-256 of sorted `(id, type, name)` tuples. If fingerprint changes between plan and apply, deploy fails unless `--force` is specified.

### Deploy Ordering
42 item types in `DEPLOY_ORDER` — deployed in dependency order:
```
storage → compute → code → models → reactive → APIs → ML → graph → viz
```

### Concurrency
- Default: 8 concurrent operations (semaphore-bounded `tokio::spawn`)
- `DataPipeline`: always sequential (ordering dependencies)
- Deletes: always sequential

### Empty Definitions
Items with no definition parts (Lakehouse, MLModel):
- On create: omit `definition` field entirely
- On update: skip `updateDefinition` call

## Workspace API Behaviors

### Folder Management
Standard CRUD at `/workspaces/{ws}/folders`:
- Create: `POST` with `{"displayName": "...", "parentFolderId": "<id>" | null}`
- Move items: `POST /workspaces/{ws}/folders/{id}/move` with body:
  ```json
  {"targetFolderId": "<id>" | null}
  ```
  `null` moves to workspace root.

### Tags
- Apply: `POST /workspaces/{ws}/applyTags` with `{"tagIds": ["<uuid>", ...]}`
- Unapply: `POST /workspaces/{ws}/unapplyTags` with `{"tagIds": ["<uuid>", ...]}`

### Domain Assignment
- Assign: `POST /workspaces/{ws}/assignToDomain` with `{"domainId": "<uuid>"}`
- Unassign: `POST /workspaces/{ws}/unassignFromDomain`

### OneLake Settings
- `POST /workspaces/{ws}/modifyDefaultTier?defaultTier={value}`
- **IMPORTANT**: The tier value goes as a **query parameter**, not in the request body
- Values: `Hot`, `Cool`, `Cold`

### Lifecycle Policies
- Export: `POST /workspaces/{ws}/exportLifecyclePolicy` (returns JSON)
- Import: `POST /workspaces/{ws}/importLifecyclePolicy` (accepts JSON body)

### Network Policies
| Policy | Endpoint |
|--------|----------|
| Firewall rules | `/workspaces/{ws}/networkPolicies/firewallRules` |
| Git outbound | `/workspaces/{ws}/networkPolicies/gitOutbound` |
| Inbound Azure resources | `/workspaces/{ws}/networkPolicies/inboundAzureResources` |
| Outbound cloud connections | `/workspaces/{ws}/networkPolicies/outboundCloud` |
| Outbound gateways | `/workspaces/{ws}/networkPolicies/outboundGateways` |

- **OAP outbound restriction** requires F64+ capacity
- **Inbound** works on Trial capacity

### Identity Provisioning
- `POST /workspaces/{ws}/provisionIdentity` is **LRO** (returns 202)
- Response includes `applicationId` + `servicePrincipalId`

## Item API Behaviors

### Type Filter on List
`GET /workspaces/{ws}/items?type={PascalCase}` — type value must be PascalCase (e.g., `Notebook`, `SemanticModel`, `DataPipeline`).

### Copy Pattern
1. `POST /workspaces/{ws}/{type}s/{id}/getDefinition` (LRO) — get definition parts
2. `GET /workspaces/{ws}/{type}s/{id}` — get metadata (displayName, description)
3. `POST /workspaces/{destWs}/items` with definition (LRO) — create in destination

### Move Pattern
Copy + DELETE source item. No native move API exists.

### Bulk Operations (All LRO)
| Operation | Endpoint |
|-----------|----------|
| `bulkExportDefinitions` | `POST /workspaces/{ws}/items/bulkExportDefinitions` |
| `bulkImportDefinitions` | `POST /workspaces/{ws}/items/bulkImportDefinitions` |
| `bulkMove` | `POST /workspaces/{ws}/items/bulkMove` |

### External Data Shares
Standard CRUD at `/workspaces/{ws}/items/{id}/externalDataShares`. Requires tenant setting `AllowExternalDataSharingSwitch` enabled.

### Identity Assignment
`POST /workspaces/{ws}/items/{id}/assignIdentity` — assigns workspace managed identity to the item.

## Cross-Database Query Behaviors

### Three-Part Naming Support
| Source Endpoint | Three-Part Naming | Notes |
|----------------|-------------------|-------|
| Lakehouse SQL endpoint | YES | Can query other DBs in same workspace |
| Warehouse | YES | Same TDS endpoint, sees `sys.databases` |
| SQL Database | NO | Error 40515: "Reference to database and/or server name is not supported" |

### Direction
Cross-database querying is **one-way**:
- Lakehouse/Warehouse → SQL Database: **works**
- SQL Database → Lakehouse/Warehouse: **does NOT work**

### Practical Pattern
Use the **lakehouse SQL endpoint as query hub** for JOINs across item types:
```sql
SELECT l.col
FROM dbo.local_table l
JOIN SqlDb.dbo.remote_table r ON l.id = r.id
```

## Report Definition Formats

### PBIR-Legacy vs PBIR
| Aspect | PBIR-Legacy | PBIR |
|--------|-------------|------|
| Structure | Single `report.json` | `definition/` folder tree |
| Data rendering | Works with `prototypeQuery` | Stores correctly but renders NO data |
| Future | Deprecated at GA | Only supported format at GA |

### definition.pbir v2.0 Schema (Recommended)
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

### PBIR-Legacy Visual Config
Required fields for visuals that render data:
- `visualType` — the chart/visual type
- `projections` — field-to-role bindings
- `dataTransforms` — metadata for rendering engine
- `prototypeQuery` — **REQUIRED** for data to actually render

### PBIR visual.json Limitation
PBIR visuals use `query.queryState` which stores correctly but **renders NO data** in the portal. The PBIR schema does not support `prototypeQuery`.

### Supported visualType Values
`card`, `barChart`, `columnChart`, `lineChart`, `pieChart`, `donutChart`, `tableEx`, `matrix`, `map`, `scatterChart`, `slicer`, `kpi`

### Projection Role Names by visualType
| visualType | Roles |
|------------|-------|
| `card` | `Values` |
| `barChart` / `columnChart` | `Category` + `Y` |
| `lineChart` | `Category` + `Y` (+ optional `Series`) |
| `pieChart` / `donutChart` | `Category` + `Y` |
| `tableEx` | `Values` (array) |
| `matrix` | `Rows` + `Columns` + `Values` |
| `map` | `Category` + `Size` + `Color` |
| `scatterChart` | `Category` + `X` + `Y` + `Size` |
| `slicer` | `Values` |
| `kpi` | `Indicator` + `TrendAxis` + `Goal` |

### updateDefinition Rule
ALWAYS include `definition.pbir` part, even if only updating `report.json` or visual files.

## Eventstream Behaviors

### Definition Format
Two files:
- `eventstream.json` — topology (sources, destinations, streams, operators, compatibilityLevel)
- `eventstreamProperties.json` — retention (`retentionTimeInDays`: 1-90) and throughput (`eventThroughputLevel`: Low/Medium/High)

### Source Types
`CustomEndpoint`, `AzureEventHub`, `AzureIoTHub`, `SampleData`, `AmazonKinesis`, `ApacheKafka`, `ConfluentCloud`, `GooglePubSub`, CDC types (`AzureSQLDBCDC`, `MySQLCDC`, `PostgreSQLCDC`), Fabric events (`FabricWorkspaceItemEvents`, `FabricJobEvents`, `FabricOneLakeEvents`)

### Destination Types
`Eventhouse`, `Lakehouse`, `CustomEndpoint`, `Activator`

### Eventhouse Destination Critical Rule
The `itemId` field must be the **KQL Database item ID** (NOT the Eventhouse ID). Using the Eventhouse ID causes: "Unable to extract cluster URL from the Eventhouse KQL database item ID".

### Two Ingestion Modes
| Mode | Table Creation | Requirements |
|------|---------------|--------------|
| `ProcessedIngestion` | Auto-creates table (with system columns) | `inputSerialization` in properties |
| `DirectIngestion` | Requires pre-created table + mapping | `connectionName` + `mappingRuleName` |

### Graph-Like Topology
Nodes reference each other by `name` via `inputNodes` arrays. Structure: source → stream → destination/operator. Names must be unique across all node types.

### No Individual Source/Destination CRUD
Sources and destinations can only be created/deleted via `update-definition` (full definition replacement). Individual `GET .../sources/{id}` and `GET .../destinations/{id}` are read-only.

### add-source / add-destination Helpers
High-level commands that:
1. Fetch current definition
2. Merge in the new node
3. Auto-create default streams
4. Push updated definition via `updateDefinition`

## Data Agent Behaviors (Expanded)

### Data Source Configuration
Configured via `datasource.json` parts in the definition. Single-part updates are **silently dropped** — must include ALL parts together:
- `data_agent.json` + `stage_config.json` + `datasource.json`

### Path Convention
```
Files/Config/{stage}/{type}-{DisplayName}/datasource.json
```
Where:
- `stage`: `draft` or `published`
- `type`: full type value (e.g., `lakehouse-tables`, `data-warehouse`, `kusto`, `graph`)

### Data Source Type Enum
`unknown`, `lakehouse_tables`, `lakehouse`, `data_warehouse`, `kusto`, `semantic_model`, `graph`, `mirrored_database`, `mirrored_azure_databricks`

### Full Definition Required
Single-part `updateDefinition` with only the datasource file is silently dropped (202 accepted but not persisted). Must include ALL definition parts together for persistence.

## Environment API

### Staging/Publish Workflow
Environments use a two-stage model:
1. Make changes (libraries, Spark settings) — stored in **staging**
2. `POST /workspaces/{ws}/environments/{id}/staging/publish` — promotes to live

### Library Management
- Libraries exist in both **published** (active) and **staging** (pending) states
- Export/import available for both states
- Changes to libraries require publish to take effect

### Publish Behavior
- Publish is **fire-and-forget** (NOT LRO — returns immediately)
- Cancel via `POST /workspaces/{ws}/environments/{id}/staging/cancelPublish`
- Check publish state via `GET /workspaces/{ws}/environments/{id}`

## Mirrored Database/Catalog Behaviors

### Mirrored Catalog
- Requires **tenant feature flag** to be enabled
- Without it, mutations fail (list may still work)

### Mirrored Databricks Catalog
- Creates **without** external connection (unlike other mirrored types)
- Uses `discover-catalogs` to enumerate available Databricks catalogs

### Naming Constraints
- `MirroredAzureDatabricksCatalog`: **no hyphens** allowed in display name
- Standard `MirroredDatabase`: standard naming rules apply

## Apache Airflow Job Behaviors

### File Operations
- All file endpoints require `?beta=true` query parameter
- Without it, returns 404 or unsupported error

### File Upload
- Content-Type: `text/plain` (JSON body is rejected)
- Files are DAGs, not Fabric definitions

### Environment Lifecycle States
```
Initial → Starting → Started → Stopping → Stopped
```
- Start: `POST /workspaces/{ws}/apacheAirflowJobs/{id}/startEnvironment`
- Stop: `POST /workspaces/{ws}/apacheAirflowJobs/{id}/stopEnvironment`
- Get state: `GET /workspaces/{ws}/apacheAirflowJobs/{id}/getEnvironment`

## Power BI REST API Integration

### Single Token for Both APIs
The Fabric token (`https://api.fabric.microsoft.com/.default` scope) is accepted by both `api.fabric.microsoft.com` and `api.powerbi.com`. No separate Power BI scope is needed.

### Power BI API Base URL
`https://api.powerbi.com/v1.0/myorg`. Workspaces are referenced as "groups": `/groups/{workspace-id}/datasets/{dataset-id}`.

### `datasets` = semantic models
The Power BI REST API uses the legacy term "datasets" for what Fabric calls "semantic models". The ID is the same UUID.

### `--api powerbi` flag on `fabio rest call`
Routes requests to the Power BI API instead of Fabric. Dry-run output includes `"api": "powerbi"` field. Env var `FABIO_POWERBI_ENDPOINT` overrides the base URL (for sovereign clouds).

### Semantic Model Power BI Commands
12 subcommands via Power BI REST API:
- `list-parameters`: `GET /groups/{ws}/datasets/{id}/parameters`
- `update-parameters`: `POST /groups/{ws}/datasets/{id}/Default.UpdateParameters`
- `list-datasources`: `GET /groups/{ws}/datasets/{id}/datasources`
- `update-datasources`: `POST /groups/{ws}/datasets/{id}/Default.UpdateDatasources`
- `list-users`: `GET /groups/{ws}/datasets/{id}/users`
- `add-user`: `POST /groups/{ws}/datasets/{id}/users`
- `delete-user`: `DELETE /groups/{ws}/datasets/{id}/users/{user}`
- `refresh-status`: `GET /groups/{ws}/datasets/{id}/refreshes?$top=N`
- `list-upstream`: `GET /groups/{ws}/datasets/{id}/upstreamDatasets`
- `clone`: `POST /groups/{ws}/datasets/{id}/Default.Clone`
- `export-pbix`: `POST /groups/{ws}/datasets/{id}/Default.Export` (binary download)
- `import-pbix`: `POST /groups/{ws}/imports` (multipart/form-data)

### import-pbix nameConflict Values
`Abort` (default), `Overwrite`, `CreateOrOverwrite`, `GenerateUniqueName`

### add-user accessRight Values
`Read`, `ReadWrite`, `ReadWriteReshare`, `ReadWriteReshareExplore`, `ReadExplore`, `ReadReshareExplore`, `ReadWriteExplore`

## Capacity ARM API Lifecycle

### Dual API Design
- Read operations (list/show): Fabric API (`api.fabric.microsoft.com/v1/capacities`)
- Lifecycle operations (suspend/resume/create/update/delete): ARM API (`management.azure.com`)

### ARM API Details
- API version: `2023-11-01`
- Resource path: `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Fabric/capacities/{name}`
- Auth scope: `https://management.azure.com/.default` (separate from Fabric scope)
- Requires Azure RBAC (Contributor) on the capacity resource

### Capacity Name Constraints
3-63 chars, pattern `^[a-z][a-z0-9]*$` (lowercase only, starts with letter)

### SKU Values
F2, F4, F8, F16, F32, F64, F128, F256, F512, F1024, F2048 (Fabric tier)

### State Values
`Active`, `Inactive` (paused/suspended), `Provisioning`, `Deleting`

## REST Passthrough Command

### Usage
```bash
fabio rest call --method GET --path "/workspaces/{ws}/items" [--body <json|@file|@->] [--query-params "key=value"] [--poll] [--api <fabric|powerbi>]
```

### Body Resolution
- Inline JSON: `--body '{"key": "value"}'`
- From file: `--body @path/to/file.json`
- From stdin: `--body @-`

### `--poll` flag
Enables LRO polling on the response (follows Location header)

### `--api powerbi`
Routes to `https://api.powerbi.com/v1.0/myorg` instead of Fabric base URL

## RTI (Real-Time Intelligence) NL-to-KQL

### Endpoint
`POST /workspaces/{ws}/realTimeIntelligence/nltokql?beta=true`

### Request Body (Required Fields)
```json
{
  "itemIdForBilling": "<kql-database-or-eventhouse-id>",
  "clusterUrl": "<kusto-uri>",
  "databaseName": "<db-name>",
  "naturalLanguage": "<question>"
}
```

### Optional Fields
- `userShots`: Array of `{"naturalLanguage":"...","kqlQuery":"..."}` examples
- `chatMessages`: Array of `{"role":"User|Assistant","content":"..."}` for multi-turn

### Response
Returns JSON with `kqlQuery` field containing the generated KQL, plus `explanation` and metadata.

## CSV/TSV Output Format

All commands support `--output csv` and `--output tsv`. RFC 4180 quoting for CSV. Useful for piping to spreadsheet tools or data pipelines.

## Hard Delete

### `--hard-delete` on All Item Deletes
38 item type delete commands support `--hard-delete` flag to permanently delete (skip recycle bin). Appends `?hardDelete=true` to URL.

Non-item deletes (connection, deployment-pipeline, domain, gateway, managed-private-endpoint, onelake-security, profile, workspace) do NOT have this flag.

## Error `isRetriable` Field

API responses may include `error.isRetriable: bool`. When present, serialized in the structured error output as `"retriable": true/false`. Useful for agent retry logic.

## Item Exists/URL/Inspect

### `item exists`
Returns `{"exists": true/false}` — never errors on 404 (unlike `item show`).

### `item url`
Returns the Fabric portal URL for the item.

### `item inspect`
Aggregates metadata + definition + connections in a single response (reduces API calls).

## Notebook `--strip-output`

`get-definition --strip-output` clears `outputs` and `execution_count` from ipynb cells. Gracefully passes through `.py` format notebooks. Useful for git-friendly exports.

## Notebook Run with Parameters

```bash
fabio notebook run --workspace $WS --id $NB --wait \
  --parameters '[{"name":"p1","value":"v1","type":"Text"}]' \
  --compute-type "Spark"
```

Parameter type values: `Text`, `Int`, `Long`, `Double`, `Bool`, `DateTime`

`--execution-data` and `--parameters` accept `@file.json` (read from file) and `@-` (read from stdin), matching the `fabio rest call --body` convention:

```bash
fabio notebook run --workspace $WS --id $NB --execution-data @params.json
echo '{"type":"Full"}' | fabio notebook run --workspace $WS --id $NB --execution-data @-
```

## Profile-Aware Defaults (FABIO_WORKSPACE and FABIO_OUTPUT)

When a profile is active (`fabio profile use --name <name>`), its `workspace` and `output` defaults are injected as environment variable fallbacks for all commands.

**Precedence (highest to lowest):**
1. Explicit CLI flag (`--workspace`, `--output`)
2. External environment variable (`FABIO_WORKSPACE`, `FABIO_OUTPUT`)
3. Active profile default

This means setting `FABIO_WORKSPACE` in the shell always overrides the profile, which overrides nothing.

```bash
# Save profile with workspace default
fabio profile save --name dev --workspace $DEV_WS --output json

# Activate profile — subsequent commands use $DEV_WS as workspace default
fabio profile use --name dev

# Override for a single command via env var
FABIO_WORKSPACE=$PROD_WS fabio lakehouse list
```

## Deploy Validate

Local-only pre-flight checks on source directory (validates .platform files, item types, definition structure, logical ID references). No API calls required.

```bash
fabio deploy validate --source ./fabric-items
```

## Private Link URL Routing

When `private_link_workspace` is configured via profile, URLs are transformed for private network access. Use `fabio profile save --name private --private-link-workspace <ws-id>` to configure.

## Dataflow Execute Query

`POST /workspaces/{ws}/dataflows/{id}/executeQuery` returns binary Apache Arrow IPC stream (NOT JSON). Save with `--file` flag. Requires Contributor role.

## App Backend (preview)

- **Endpoint pattern**: `/workspaces/{ws}/appBackends` and `/workspaces/{ws}/appBackends/{id}`
- **Create is LRO**: `POST /workspaces/{ws}/appBackends` returns 202 and is polled to completion
- **Hard delete**: `--hard-delete` appends `?hardDelete=true` to permanently delete (skip recycle bin)
- **Update requires at least one field**: `--name` or `--description` is mandatory; omitting both returns `INVALID_INPUT`
- **agent-context coverage**: `fabio agent-context` includes full `app-backend` schema with `--hard-delete` flag typed as bool
