# fabio Workflow Examples

Practical examples for common Microsoft Fabric workflows using fabio.

## End-to-End: CSV to Queryable Delta Table

```bash
# Setup
fabio auth login

# Create workspace with capacity
WS=$(fabio workspace create --name "sales-analytics" -q id -o plain)
fabio workspace assign-capacity --id $WS --capacity $CAPACITY_ID

# Create lakehouse
LH=$(fabio lakehouse create --workspace $WS --name "SalesLakehouse" -q id -o plain)

# Upload CSV files (glob patterns, parallel upload)
fabio lakehouse upload --workspace $WS --id $LH --source "data/*.csv" --dest Files/raw/

# Load into Delta table (one-step upload + load)
fabio lakehouse upload-table --workspace $WS --id $LH \
  --source sales.csv --table sales --mode Overwrite --format Csv

# Or two-step: upload first, then load existing file
fabio lakehouse load-table --workspace $WS --id $LH \
  --path Files/raw/orders.csv --table orders --mode Overwrite --format Csv

# Verify tables
fabio lakehouse list-tables --workspace $WS --id $LH -o table
```

## Real-Time Intelligence Pipeline

```bash
# Create Eventhouse + KQL Database
EH=$(fabio eventhouse create --workspace $WS --name "TelemetryHub" -q id -o plain)
KDB=$(fabio kql-database create --workspace $WS --name "SensorDB" --eventhouse-id $EH -q id -o plain)

# Create table and ingestion mapping via KQL
fabio kql-database query --workspace $WS --id $KDB \
  --kql ".create table SensorEvents (DeviceId: string, Temperature: real, Timestamp: datetime)"

fabio kql-database query --workspace $WS --id $KDB \
  --kql ".create table SensorEvents ingestion json mapping 'JsonMapping' '[{\"column\":\"DeviceId\",\"path\":\"$.deviceId\"},{\"column\":\"Temperature\",\"path\":\"$.temperature\"},{\"column\":\"Timestamp\",\"path\":\"$.timestamp\"}]'"

# Create Eventstream with custom endpoint source
ES=$(fabio eventstream create --workspace $WS --name "SensorIngestion" -q id -o plain)
fabio eventstream add-source --workspace $WS --id $ES \
  --name "app-source" --source-type CustomEndpoint

# Wire to KQL Database (DirectIngestion)
fabio eventstream add-destination --workspace $WS --id $ES \
  --name "kql-sink" --destination-type Eventhouse --input-node "app-source-stream" \
  --properties '{"dataIngestionMode":"DirectIngestion","workspaceId":"'$WS'","itemId":"'$KDB'","tableName":"SensorEvents","connectionName":"es-conn","mappingRuleName":"JsonMapping"}'

# Get Event Hub connection for sending events
fabio eventstream get-source-connection --workspace $WS --id $ES --source-id $SRC_ID

# Query ingested data
fabio kql-database query --workspace $WS --id $KDB \
  --kql "SensorEvents | where Timestamp > ago(1h) | summarize avg(Temperature) by DeviceId"
```

## Notebook ETL Workflow

```bash
# Create notebook bound to lakehouse
NB=$(fabio notebook create --workspace $WS --name "ETL-Pipeline" \
  --lakehouse $LH --source etl_notebook.py -q id -o plain)

# Run and wait for completion (default timeout 600s)
fabio notebook run --workspace $WS --id $NB --wait

# Run with custom timeout
fabio notebook run --workspace $WS --id $NB --wait --timeout 1800

# Check status of a running notebook
fabio notebook status --workspace $WS --id $NB

# Stop a long-running notebook
fabio notebook stop --workspace $WS --id $NB

# Update notebook code
fabio notebook update-definition --workspace $WS --id $NB --source updated_etl.py
```

## Direct Lake Semantic Model + Report

```bash
# Get the SQL endpoint ID for the lakehouse
SQLEP=$(fabio sql-endpoint list --workspace $WS -q "data[0].id" -o plain)

# Create Direct Lake semantic model from TMDL
SM=$(fabio semantic-model create --workspace $WS --name "SalesModel" \
  --file model.tmdl --connection $SQLEP -q id -o plain)

# Frame the model (required for Direct Lake to work)
fabio semantic-model refresh --workspace $WS --id $SM

# Make editable in portal (convert from definition-managed to service-managed)
fabio semantic-model takeover --workspace $WS --id $SM

# Query with DAX
fabio semantic-model query --workspace $WS --id $SM \
  --dax "EVALUATE SUMMARIZECOLUMNS('Sales'[Country], \"Total\", SUM('Sales'[Revenue]))"

# Create a report bound to the semantic model
fabio report create --workspace $WS --name "Sales Dashboard" --dataset $SM
```

## Warehouse Operations

```bash
# Create warehouse
WH=$(fabio warehouse create --workspace $WS --name "AnalyticsWH" -q id -o plain)

# Query inline
fabio warehouse query --workspace $WS --id $WH \
  --sql "SELECT TOP 100 * FROM dbo.fact_sales ORDER BY order_date DESC"

# Query from file
fabio warehouse query --workspace $WS --id $WH --sql @queries/monthly_report.sql

# Query from stdin
echo "SELECT COUNT(*) as total FROM dbo.customers" | \
  fabio warehouse query --workspace $WS --id $WH

# Get connection string for external tools (Power BI, SSMS)
fabio warehouse connection-string --workspace $WS --id $WH

# Create restore point before migration
fabio warehouse create-restore-point --workspace $WS --id $WH --name "pre-migration"
```

## SQL Database Import

```bash
# Create SQL database
DB=$(fabio sql-database create --workspace $WS --name "OrdersDB" -q id -o plain)

# Import CSV (auto-creates table with inferred types)
fabio sql-database import --workspace $WS --id $DB \
  --file orders.csv --table orders --drop-if-exists

# Import JSON
fabio sql-database import --workspace $WS --id $DB \
  --file events.json --table events --batch-size 500

# Query
fabio sql-database query --workspace $WS --id $DB \
  --sql "SELECT status, COUNT(*) as cnt FROM orders GROUP BY status"
```

## Git Integration

```bash
# Connect workspace to GitHub (requires --connection-id)
fabio git connect --workspace $WS \
  --provider github --owner myorg --repo fabric-project \
  --branch main --directory "/" --connection-id $CONN_ID

# Connect workspace to Azure DevOps (no connection ID needed)
fabio git connect --workspace $WS \
  --provider azure-devops --org myorg --project myproject --repo myrepo \
  --branch main

# Initialize (required after connect)
# Use prefer-workspace when workspace has items and repo is empty
fabio git init --workspace $WS --strategy prefer-workspace --wait

# Use prefer-remote when repo has content to pull into workspace
fabio git init --workspace $WS --strategy prefer-remote --wait

# Check status
fabio git status --workspace $WS

# Commit all changes
fabio git commit --workspace $WS --all -m "Add new lakehouse tables"

# Commit specific items
fabio git commit --workspace $WS -m "Update notebook" \
  --items '[{"objectId":"<item-id>"}]'

# Pull remote changes
fabio git pull --workspace $WS

# Pull with conflict resolution
fabio git pull --workspace $WS --conflict-resolution prefer-remote

# Switch branches
fabio git checkout --workspace $WS --branch feature/new-pipeline

# Show tracked items
fabio git show-tracked --workspace $WS

# Show connection details
fabio git connection show --workspace $WS

# Disconnect workspace from git
fabio git disconnect --workspace $WS
```

### Azure DevOps Setup for Git Integration

```bash
# Prerequisite: The Fabric user must have access to the Azure DevOps org
# Add user to org (run as org owner)
az devops user add --email-id "user@tenant.onmicrosoft.com" \
  --license-type express --org https://dev.azure.com/myorg

# Add to project Contributors group
az devops security group membership add \
  --group-id "$CONTRIBUTOR_GROUP_DESCRIPTOR" \
  --member-id "$USER_DESCRIPTOR" \
  --org https://dev.azure.com/myorg

# Ensure repo has at least one commit (required for Fabric to connect)
# Empty repos have no defaultBranch and will fail git connect

# Connect (Automatic credentials — no Fabric connection needed)
fabio git connect --workspace $WS \
  --provider azure-devops --org myorg --project myproject --repo myrepo \
  --branch main

# Full CI/CD workflow
WS=$(fabio workspace create --name "dev-env" -q id -o plain)
fabio workspace assign-capacity --id $WS --capacity $CAP
fabio lakehouse create --workspace $WS --name "DataLake"
fabio git connect --workspace $WS --provider azure-devops \
  --org myorg --project myproject --repo myrepo --branch main
fabio git init --workspace $WS --strategy prefer-workspace --wait
fabio git commit --workspace $WS --all -m "Initial workspace setup"
```

## Deployment Pipeline (CI/CD)

```bash
# Create deployment pipeline
DP=$(fabio deployment-pipeline create --name "Sales Pipeline" -q id -o plain)

# List stages (Dev -> Test -> Prod)
fabio deployment-pipeline list-stages --id $DP -o table

# Assign workspaces to stages
fabio deployment-pipeline assign-workspace --id $DP --stage-id $DEV_STAGE --workspace $WS_DEV
fabio deployment-pipeline assign-workspace --id $DP --stage-id $PROD_STAGE --workspace $WS_PROD

# Deploy from Dev to Prod
fabio deployment-pipeline deploy --id $DP --source-stage $DEV_STAGE --target-stage $PROD_STAGE
```

## Cross-Lakehouse Data Operations

```bash
# Copy files between lakehouses (parallel, glob patterns)
fabio lakehouse copy-file --workspace $WS --id $LH_SOURCE \
  --source "Files/raw/*.csv" \
  --dest-workspace $WS2 --dest-id $LH_DEST --dest Files/imported/

# Move files (copy + delete source)
fabio lakehouse move-file --workspace $WS --id $LH \
  --source "Files/staging/*" --dest Files/archive/

# Copy Delta table between lakehouses
fabio lakehouse copy-table --workspace $WS --id $LH --table sales \
  --dest-workspace $WS2 --dest-id $LH2

# Sync (only copies new/modified, based on ETag/MD5)
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 --delete
```

## Shortcuts (External Data Access)

```bash
# ADLS Gen2 shortcut
fabio lakehouse create-shortcut --workspace $WS --id $LH \
  --name "external-data" --path Files/ \
  --target-type adls \
  --location "https://storageacct.dfs.core.windows.net/container" \
  --subpath "data/2024/"

# S3 shortcut
fabio lakehouse create-shortcut --workspace $WS --id $LH \
  --name "s3-data" --path Files/ \
  --target-type s3 \
  --location "https://bucket.s3.amazonaws.com" \
  --subpath "prefix/"

# OneLake cross-lakehouse shortcut
fabio lakehouse create-shortcut --workspace $WS --id $LH \
  --name "other-lake" --path Tables/ \
  --target-type onelake \
  --location "$OTHER_WS/$OTHER_LH" \
  --subpath "Tables/shared_table"
```

## Workspace Administration

```bash
# Create workspace with full setup
WS=$(fabio workspace create --name "team-project" -q id -o plain)
fabio workspace assign-capacity --id $WS --capacity $CAP
fabio workspace provision-identity --id $WS

# Role management
fabio workspace add-role-assignment --id $WS \
  --principal $USER_ID --principal-type User --role Contributor

# Organize with folders
fabio workspace create-folder --id $WS --name "Production"
fabio workspace create-folder --id $WS --name "Staging"

# Tag for governance
fabio workspace apply-tags --id $WS --tag-ids '["tag-uuid-1"]'

# Domain assignment
fabio workspace assign-to-domain --id $WS --domain $DOMAIN_ID

# Configure storage tier
fabio workspace modify-default-tier --id $WS --tier Cold
```

## Composability & Scripting Patterns

```bash
# Extract field with --query
WS=$(fabio workspace list --query "data[0].id" -o plain)

# Use --limit for bounded responses
fabio item list --workspace $WS --limit 10

# Dry-run to preview destructive operations
fabio workspace delete --id $WS --dry-run

# Suppress output (errors still on stderr)
fabio lakehouse delete --workspace $WS --id $LH --quiet

# Auto-paginate all results
fabio item list --workspace $WS --all

# Resume from pagination token
fabio item list --workspace $WS --continuation-token "eyJ..."

# Pipe JSON to jq for complex extraction
fabio workspace list | jq '.data[] | select(.capacityId != null) | .id'

# Named profiles for multi-environment
fabio profile save --name dev --workspace $WS_DEV
fabio profile save --name prod --workspace $WS_PROD
fabio profile use --name prod
fabio lakehouse list  # uses prod workspace from profile
```

## GraphQL API

```bash
# Create GraphQL API
GQL=$(fabio graphql-api create --workspace $WS --name "OrdersAPI" -q id -o plain)

# Update schema
fabio graphql-api update-definition --workspace $WS --id $GQL --file schema.graphql

# Execute a query
fabio graphql-api query --workspace $WS --id $GQL \
  --query '{ orders(first: 10) { id status total } }'
```

## KQL Queryset (Saved Queries)

```bash
# Run a saved query tab by name
fabio kql-queryset run --workspace $WS --id $QS --tab "Hourly Summary"

# Run by index (0-based)
fabio kql-queryset run --workspace $WS --id $QS --tab 0
```

## Error Handling for Agents

```bash
# Errors are JSON on stderr with machine-readable codes
fabio workspace show --id "nonexistent" 2>&1
# {"error":{"code":"NOT_FOUND","message":"...","hint":"Run: fabio workspace list"}}

# Check exit code
fabio auth status
if [ $? -ne 0 ]; then
  echo "Not authenticated, running fabio auth login"
  fabio auth login
fi

# Common error codes and recovery:
# AUTH_REQUIRED     -> Run: fabio auth login
# FORBIDDEN         -> Check workspace role assignment (or admin scope for admin commands)
# NOT_FOUND         -> Verify ID with list command
# CONFLICT          -> Name already exists, choose different name
# RATE_LIMITED      -> Retry after delay (fabio retries automatically)
# CAPACITY_INACTIVE -> Resume/assign capacity to workspace
# INVALID_INPUT     -> Check PascalCase values, required fields
# TIMEOUT           -> Increase --timeout or check capacity status
```

## Tenant Administration

### List and Manage Tenant Settings

```bash
# List all tenant settings (165+ settings)
fabio admin list-tenant-settings -o table

# Show a specific setting
fabio admin show-tenant-setting --name "ExportToImage"

# Enable a tenant setting
fabio admin update-tenant-setting --name "ExportToImage" \
  --content '{"enabled": true}'

# Enable with security group scoping
fabio admin update-tenant-setting --name "ExportToImage" \
  --content '{"enabled": true, "enabledSecurityGroups": [{"graphId": "<group-id>"}]}'

# Disable a setting
fabio admin update-tenant-setting --name "AllowExternalDataSharingSwitch" \
  --content '{"enabled": false}'

# Check if a setting is delegatable to capacity
fabio admin show-tenant-setting --name "PlatformMonitoringTenantSetting" | \
  jq '.data.delegateToCapacity'
```

### Capacity-Level Setting Overrides

```bash
# List overrides for a capacity
fabio admin list-capacity-tenant-setting-overrides --capacity $CAP_ID

# Create/update a capacity override (only for delegatable settings)
fabio admin update-capacity-tenant-setting-override \
  --capacity $CAP_ID --name "PlatformMonitoringTenantSetting" \
  --content '{"enabled": true}'

# Revert to tenant default (disable override)
fabio admin update-capacity-tenant-setting-override \
  --capacity $CAP_ID --name "PlatformMonitoringTenantSetting" \
  --content '{"enabled": false}'
```

### Tag Lifecycle (Governance)

```bash
# Create a governance tag
TAG_RESPONSE=$(fabio admin create-tag \
  --content '{"createTagsRequest": [{"displayName": "Production"}]}')
TAG_ID=$(echo $TAG_RESPONSE | jq -r '.data.tags[0].id')

# List all tags
fabio admin list-tags -o table

# Update tag metadata
fabio admin update-tag --id $TAG_ID \
  --content '{"displayName": "Production", "description": "Production-ready items"}'

# Apply tag to workspace
fabio workspace apply-tags --id $WS --tag-ids "[\"$TAG_ID\"]"

# Apply tag to item
fabio item apply-tags --workspace $WS --id $ITEM_ID --tag-ids "[\"$TAG_ID\"]"

# Clean up
fabio workspace unapply-tags --id $WS --tag-ids "[\"$TAG_ID\"]"
fabio admin delete-tag --id $TAG_ID
```

### Domain Management

```bash
# Create a domain
DOMAIN=$(fabio admin create-domain --name "Sales Analytics" -q id -o plain)

# Assign workspaces to domain
fabio admin assign-domain-workspaces --id $DOMAIN \
  --content '{"workspacesIds": ["'$WS_DEV'", "'$WS_PROD'"]}'

# Assign all workspaces on a capacity to a domain
fabio admin assign-domain-workspaces-by-capacities --id $DOMAIN \
  --content '{"capacitiesIds": ["'$CAP_ID'"]}'

# Assign workspaces owned by specific users
fabio admin assign-domain-workspaces-by-principals --id $DOMAIN \
  --principal-type User \
  --content '{"principals": [{"id": "'$USER_ID'", "type": "User"}]}'

# List workspaces in domain
fabio admin list-domain-workspaces --id $DOMAIN

# Assign contributors to domain
fabio admin bulk-assign-domain-roles --id $DOMAIN \
  --content '{"type": "Contributors", "principals": [{"id": "'$USER_ID'", "type": "User"}]}'

# Sync contributor roles to subdomains
fabio admin sync-domain-roles-to-subdomains --id $DOMAIN --role Contributor

# Unassign all workspaces atomically
fabio admin unassign-all-domain-workspaces --id $DOMAIN

# Delete domain
fabio admin delete-domain --id $DOMAIN
```

### Admin Workspace Operations

```bash
# List all workspaces in tenant (admin view)
fabio admin list-workspaces --all

# Show workspace details (includes capacity assignment, state)
fabio admin show-workspace --id $WS

# List users with access
fabio admin list-workspace-users --id $WS

# Discover which workspaces have git connections
fabio admin discover-git-connections

# List network communication policies
fabio admin list-network-policies
```

### Admin Item Operations

```bash
# List all items tenant-wide (with type filter)
fabio admin list-items --type Report

# Show item admin details (includes creatorPrincipal, defaultIdentity)
fabio admin show-item --workspace $WS --id $ITEM_ID

# List item permissions
fabio admin list-item-users --workspace $WS --id $ITEM_ID

# List all access for a specific user
fabio admin list-user-access --user-id $PRINCIPAL_ID
```

### Sharing Link Management

```bash
# Remove all org-wide sharing links (LRO - polls until complete)
fabio admin remove-all-sharing-links \
  --content '{"sharingLinkType": "OrgLink"}'

# Remove sharing links for specific reports (only Report type supported)
fabio admin bulk-remove-sharing-links \
  --content '{"sharingLinks": [{"itemId": "'$REPORT_ID'", "itemType": "Report", "workspaceId": "'$WS'"}]}'
```

### External Data Shares

```bash
# List all external data shares (requires AllowExternalDataSharingSwitch enabled)
fabio admin list-external-data-shares

# Revoke a specific external data share
fabio admin revoke-external-data-share \
  --workspace $WS --item-id $ITEM_ID --share-id $SHARE_ID
```

### Workload Management

```bash
# List available workloads
fabio admin list-workloads

# List current assignments
fabio admin list-workload-assignments

# Assign workload at tenant level
fabio admin create-workload-assignment \
  --content '{"type": "Tenant", "workloadId": "my-workload-id"}'

# Assign workload to specific capacity
fabio admin create-workload-assignment \
  --content '{"type": "Capacity", "workloadId": "my-workload-id", "capacityId": "'$CAP_ID'"}'

# Remove assignment
fabio admin delete-workload-assignment --id $ASSIGNMENT_ID
```

## Gateway Management

```bash
# List gateways
fabio gateway list -o table

# Create VNet gateway (requires subnet delegated to Microsoft.PowerPlatform/vnetaccesslinks)
GW=$(fabio gateway create --name "DataGateway" \
  --capacity $CAP_ID \
  --subscription $SUB_ID \
  --resource-group $RG \
  --vnet "my-vnet" \
  --subnet "gateway-subnet" \
  --inactivity-minutes 120 \
  --member-count 1 \
  -q id -o plain)

# Update gateway (inactivity timeout)
fabio gateway update --id $GW --inactivity-minutes 240

# Assign roles
fabio gateway add-role-assignment --id $GW \
  --principal $USER_ID --principal-type User --role ConnectionCreator

# List role assignments
fabio gateway list-role-assignments --id $GW -o table

# Delete gateway
fabio gateway delete --id $GW
```
