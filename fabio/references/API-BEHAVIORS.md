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

### load-table Multi-Schema Support
- `--schema` flag routes to the multi-schema endpoint: `POST /workspaces/{ws}/lakehouses/{id}/schemas/{schemaName}/tables/{table}/load?beta=true`
- Without `--schema`, uses the default endpoint (no schema path segment)
- This is a **beta endpoint** — requires `?beta=true` query parameter

### create-shortcut Conflict Policy
- `--conflict-policy` maps to the `shortcutConflictPolicy` query parameter
- Values: `Abort` (fail if name conflicts), `GenerateUniqueName` (auto-suffix to avoid conflict)
- Default behavior (no flag): server determines policy (typically Abort)

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
- No individual role create/update via upsert — always send the full set
- `create` uses the native single-role POST endpoint: `POST /workspaces/{ws}/items/{id}/dataAccessRoles?dataAccessRoleConflictPolicy={policy}`
  - `--conflict-policy Abort` (default) fails if role with that name exists
  - `--conflict-policy Overwrite` replaces an existing role with the same name
- `show` and `delete` use native per-roleName endpoints:
  - `GET /workspaces/{ws}/items/{id}/dataAccessRoles/{roleName}`
  - `DELETE /workspaces/{ws}/items/{id}/dataAccessRoles/{roleName}`

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

## Error Handling

### isRetriable Field
API error responses may include `"isRetriable": true|false` in the error body. fabio parses this and surfaces it in structured error output:
```json
{"error": {"code": "API_ERROR", "message": "...", "retriable": true}}
```
The `retriable` field is **omitted when absent** (not present for all errors). When `true`, the operation is safe to retry immediately.

## Item API Behaviors

### list Server-Side Filters
`item list` supports server-side filtering via query parameters:
- `--folder <folder-id>` → `rootFolderId={id}` query param (items in that folder only)
- `--recursive` → `recursive=true` (include items in subfolders)
- `--include <types>` → `include={types}` (comma-separated item types to include)

### move-to-folder
```
POST /workspaces/{ws}/items/{id}/move
Body: {"targetFolderId": "<folder-id>"}
```
Moves an item to a different folder within the same workspace.

### create-external-data-share Recipient Types
```json
{
  "recipient": {
    "recipientType": "User",
    "objectId": "<principal-id>"
  }
}
```
Supported `recipientType` values: `User`, `ServicePrincipal`.

## Notebook API Behaviors

### Parameterized Run
`notebook run` extended parameters:
- `--parameters <json>`: JSON array of parameter objects: `[{"name":"p1","value":"v1","type":"Text"}]`
  - Valid types: `Text`, `Bool`, `Int`, `Float`
- `--compute-type <type>`: Sets `executionData.computeType` in the job request body
- `--execution-data <json>`: Full JSON object merged into the job request body (overrides individual flags)

### get-definition --strip-output
When `--strip-output` is provided, fabio strips cell outputs from the `.ipynb` notebook before returning the definition. Useful for clean source control diffs.

## Dataflow API Behaviors

### dataflow run
```
POST /workspaces/{ws}/dataflows/{id}/jobs/instances?jobType={type}
```
- `--job-type execute` (default): standard dataflow execution
- `--job-type apply-changes`: apply pending dataflow changes
- `--execute-option`: passed as `executeOption` in the request body
- `--parameters`: JSON object passed as `executionParameters` in the request body
- Supports `--wait`, `--timeout`, `--cancel-on-timeout` (same as notebook run)

### dataflow execute-query
```
POST /workspaces/{ws}/dataflows/{id}/executeQuery
Body: {"queryName": "...", "customMashup": "..."}
```
- Returns **binary Arrow IPC stream** (not JSON)
- Use `--file <path>` to save the binary output to a file
- If no `--file` is specified, binary data is written to stdout

## Connection API Behaviors

### Credential Types
In addition to previously supported types, these credential types are now supported in `connection create`:
- `WorkspaceIdentity`: Uses workspace-managed identity (no explicit credential values needed)
- `KeyPair`: Public/private key pair credentials

## RTI (Real-Time Intelligence) API Behaviors

### nl-to-kql
Translates natural language to KQL using Fabric's RTI service:
```
POST /workspaces/{ws}/realTimeIntelligence/nltokql?beta=true
Body:
{
  "itemIdForBilling": "<item-id>",
  "clusterUrl": "https://<cluster>.z0.kusto.fabric.microsoft.com",
  "databaseName": "<db-name>",
  "naturalLanguage": "show me top 10 events by count",
  "userShots": [...],       // optional: few-shot examples
  "chatMessages": [...]     // optional: conversation history
}
```
- **Beta endpoint** — requires `?beta=true`
- Response: `{"kqlQuery": "...", "explanation": "..."}`
- `--item-id` (`itemIdForBilling`) is the Eventhouse or KQL database item ID used for billing attribution

## Environment API Behaviors

### upload-staging-library
```
POST /workspaces/{ws}/environments/{id}/staging/libraries
Content-Type: application/octet-stream
Body: <binary file content>
```
- Uses raw binary upload (not multipart form, not base64)
- `--library-name` overrides the filename; defaults to the source filename
- Supported library types: `.whl`, `.jar`, `.tar.gz`

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
