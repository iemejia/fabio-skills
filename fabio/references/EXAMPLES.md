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

## Gateway Lifecycle Management

```bash
# Check VNet gateway connectivity status
fabio gateway check-status --id $GW

# Check individual member connectivity status (on-premises gateways)
fabio gateway check-member-status --id $GW --member-id $MEMBER_ID

# Restart gateway (LRO — polls until complete, requires Admin)
fabio gateway restart --id $GW

# Shut down gateway (LRO — polls until complete, requires Admin)
fabio gateway shutdown --id $GW
```

## Lakehouse Sync: rsync-Inspired Patterns

```bash
# Sync only new/modified CSV files (include filter)
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 --delete \
  --include "*.csv"

# Sync large files only, skip tiny ones
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 \
  --min-size 1M --max-size 500M

# Safety: skip ALL deletions if more than 10 files would be removed
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 --delete \
  --max-delete 10

# Move semantics: delete source files after successful transfer
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 \
  --remove-source-files

# Refresh existing files only (don't create new ones at dest)
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 \
  --existing

# Itemize: show per-file actions on stderr for audit trail
fabio lakehouse sync --workspace $WS --id $LH \
  --dest-workspace $WS2 --dest-id $LH2 --delete \
  --itemize 2>sync-audit.log
```

## Lakehouse Sync: Local-to-Remote

Sync a local directory to a remote lakehouse path, uploading only new or changed files:

```bash
# Basic local sync (size comparison)
fabio lakehouse sync --local ./data \
  --dest-workspace $WS --dest-id $LH --dest-path Files/data

# Use checksum (MD5) for more accurate change detection
fabio lakehouse sync --local ./data \
  --dest-workspace $WS --dest-id $LH --dest-path Files/data \
  --checksum

# Move semantics: delete local files after upload
fabio lakehouse sync --local ./exports \
  --dest-workspace $WS --dest-id $LH --dest-path Files/archive \
  --remove-source-files

# Sync only parquet files, skip subdirectories
fabio lakehouse sync --local ./processed \
  --dest-workspace $WS --dest-id $LH --dest-path Files/processed \
  --include "*.parquet" --no-recursive
```

## Copy Job Reset

```bash
# Reset ALL entities in a copy job for re-processing
fabio copy-job reset --workspace $WS --id $CJ --all

# Reset specific entities (comma-separated UUIDs)
fabio copy-job reset --workspace $WS --id $CJ \
  --entity-ids "11111111-0000-0000-0000-000000000001,11111111-0000-0000-0000-000000000002"
```

## Data Build Tool Job (dbt, preview)

```bash
# List dbt jobs in workspace
fabio data-build-tool-job list --workspace $WS -o table

# Create a dbt job
DBTJ=$(fabio data-build-tool-job create \
  --workspace $WS --name "nightly-dbt-run" -q id -o plain)

# Run a dbt job and wait for completion
fabio data-build-tool-job run --workspace $WS --id $DBTJ \
  --wait --timeout 1800

# Get definition
fabio data-build-tool-job get-definition --workspace $WS --id $DBTJ
```

## Organizational App (OrgApp & Audience)

```bash
# Create an Organizational App
APP=$(fabio org-app create \
  --workspace $WS --name "SalesReports" -q id -o plain)

# Create an audience for the app
AUD=$(fabio org-app-audience create \
  --workspace $WS --name "SalesTeam" -q id -o plain)

# Get definition
fabio org-app get-definition --workspace $WS --id $APP
fabio org-app-audience get-definition --workspace $WS --id $AUD

# List all apps and audiences
fabio org-app list --workspace $WS -o table
fabio org-app-audience list --workspace $WS -o table
```

## Deploy CI/CD (Stateless Workspace Deployment)

### GitHub Actions — OIDC (Recommended, secretless)

```yaml
name: Deploy Fabric
on: [push]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install fabio
        run: bash .agents/skills/fabio/scripts/install.sh

      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          allow-no-subscriptions: true   # Fabric-only, no Azure subscription needed

      - name: Deploy to Fabric workspace
        run: |
          fabio deploy apply --source ./fabric-items --workspace "${{ vars.FABRIC_WS_ID }}"
```

### GitHub Actions — Client Secret (simpler, no extra actions)

```yaml
      - name: Deploy to Fabric workspace
        env:
          AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
        run: |
          fabio deploy apply --source ./fabric-items --workspace "${{ vars.FABRIC_WS_ID }}"
```

### Export → Modify → Deploy Pattern
```bash
# Export current workspace state to disk
fabio deploy export --workspace $WS --dir ./fabric-export --overwrite

# Preview what would change (plan)
fabio deploy plan --source ./fabric-export --workspace $WS_TARGET

# Apply changes (create/update/rename items to match source)
fabio deploy apply --source ./fabric-export --workspace $WS_TARGET

# Apply with dry-run to preview without executing
fabio deploy apply --source ./fabric-export --workspace $WS_TARGET --dry-run
```

### Multi-Environment Deployment with Parameters
```bash
# Generate parameters.json from scanning GUIDs in definitions
fabio deploy init-params --source ./fabric-export --out parameters.json

# Generate by diffing two environments
fabio deploy init-params --source ./dev-export --compare ./prod-export \
  --source-env dev --compare-env prod --out parameters.json

# Deploy to dev
fabio deploy apply --source ./fabric-export --workspace "dev-workspace" \
  --parameters parameters.json --env dev

# Deploy to prod
fabio deploy apply --source ./fabric-export --workspace "prod-workspace" \
  --parameters parameters.json --env prod
```

### Plan File for Review Before Apply
```bash
# Save plan to file for review
fabio deploy plan --source ./fabric-export --workspace $WS --out plan.json

# Review the plan
cat plan.json | jq '.data.summary'

# Apply the saved plan (verifies workspace hasn't changed)
fabio deploy apply --plan plan.json

# Force apply even if workspace changed
fabio deploy apply --plan plan.json --force
```

### Advanced Deploy Options
```bash
# Deploy only specific item types
fabio deploy apply --source ./fabric-export --workspace $WS \
  --item-types Notebook,DataPipeline

# Delete items in workspace not in source
fabio deploy apply --source ./fabric-export --workspace $WS --delete-orphans

# Increase concurrency for large workspaces
fabio deploy apply --source ./fabric-export --workspace $WS --concurrency 16

# Skip post-deploy hooks (no auto-refresh semantic models)
fabio deploy apply --source ./fabric-export --workspace $WS --no-post-hooks

# Deploy using workspace name instead of ID
fabio deploy apply --source ./fabric-export --workspace "My Workspace Name"
```

## Data Agent (AI-Powered Q&A)

### Create and Configure Data Agent
```bash
# Create data agent
DA=$(fabio data-agent create --workspace $WS --name "SalesAssistant" -q id -o plain)

# Configure AI instructions via definition
fabio data-agent update-definition --workspace $WS --id $DA --file data-agent-config/

# Query the data agent (chat interface)
fabio data-agent query --workspace $WS --id $DA --message "What were total sales last quarter?"

# Publish the agent (makes it available via URL)
fabio data-agent publish --workspace $WS --id $DA
```

## Ontology & Graph Model

### Create Ontology with Entity Types and Data Bindings
```bash
# Create ontology
ONT=$(fabio ontology create --workspace $WS --name "EquipmentOntology" -q id -o plain)

# Update definition with entity types, relationship types, data bindings
fabio ontology update-definition --workspace $WS --id $ONT --dir ./ontology-definition/

# Get definition (with readable decoded output)
fabio ontology get-definition --workspace $WS --id $ONT --decode

# Create graph model linked to ontology
GM=$(fabio graph-model create --workspace $WS --name "EquipmentGraph" --ontology $ONT -q id -o plain)

# Refresh graph (requires portal initialization first)
fabio graph-model refresh-graph --workspace $WS --id $GM

# Execute KQL query on loaded graph
fabio graph-model execute-query --workspace $WS --id $GM \
  --query "nodes() | where type == 'Equipment' | take 10"
```

## Workspace Networking & Security

### Configure Firewall Rules
```bash
# Get current network policy
fabio workspace get-network-policy --id $WS

# Set firewall rules (replaces all existing rules)
fabio workspace set-network-policy --id $WS --type firewall \
  --content '{"rules":[{"displayName":"Office","value":"10.0.0.0/24"},{"displayName":"VPN","value":"192.168.1.0/24"}]}'

# Configure git outbound policy
fabio workspace set-network-policy --id $WS --type git-outbound \
  --content '{"defaultAction":"Deny","rules":[]}'
```

### OneLake Storage Configuration
```bash
# View OneLake settings (tier, diagnostics, immutability)
fabio workspace get-onelake-settings --id $WS

# Set default storage tier
fabio workspace modify-default-tier --id $WS --tier Cold

# Export lifecycle policy
fabio workspace export-lifecycle-policy --id $WS > lifecycle.json

# Import lifecycle policy
fabio workspace import-lifecycle-policy --id $WS --file lifecycle.json
```

## Apache Airflow Job

### Manage Airflow Environment
```bash
# Create Airflow job
AF=$(fabio apache-airflow-job create --workspace $WS --name "ETLOrchestrator" -q id -o plain)

# Start the Airflow environment
fabio apache-airflow-job start-environment --workspace $WS --id $AF

# Check environment status
fabio apache-airflow-job get-environment --workspace $WS --id $AF

# Upload a DAG file
fabio apache-airflow-job upload-file --workspace $WS --id $AF \
  --path "dags/etl_pipeline.py" --content "$(cat my_dag.py)"

# List files in the Airflow workspace
fabio apache-airflow-job list-files --workspace $WS --id $AF

# Deploy requirements
fabio apache-airflow-job deploy-requirements --workspace $WS --id $AF --file requirements.txt

# Stop the environment when done
fabio apache-airflow-job stop-environment --workspace $WS --id $AF
```

## Warehouse Snapshots & Restore

### Manage Restore Points
```bash
# List restore points
fabio warehouse list-restore-points --workspace $WS --id $WH

# Create a restore point before migration
fabio warehouse create-restore-point --workspace $WS --id $WH --name "pre-migration-backup"

# Restore warehouse to a point
fabio warehouse restore-to-point --workspace $WS --id $WH --restore-point-id $RP_ID

# Delete old restore point
fabio warehouse delete-restore-point --workspace $WS --id $WH --restore-point-id $RP_ID
```

### Warehouse Audit Settings
```bash
# Get audit settings
fabio warehouse get-audit-settings --workspace $WS --id $WH

# Update audit settings
fabio warehouse update-audit-settings --workspace $WS --id $WH \
  --content '{"state":"Enabled","retentionDays":90,"auditActionsAndGroups":["SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP","BATCH_COMPLETED_GROUP"]}'
```

## Semantic Model Power BI Operations

### Clone and Export

```bash
# Clone a semantic model within the same workspace
fabio semantic-model clone --workspace $WS --id $SM --name "SalesModel-Copy"

# Clone to another workspace
fabio semantic-model clone --workspace $WS --id $SM --name "SalesModel-Prod" --target-workspace $WS_PROD

# Export as .pbix file
fabio semantic-model export-pbix --workspace $WS --id $SM --file ./exports/sales.pbix

# Import a .pbix file (overwrite if exists)
fabio semantic-model import-pbix --workspace $WS --name "ImportedModel" \
  --file ./models/sales.pbix --name-conflict Overwrite
```

### User Management

```bash
# List users with access to the semantic model
fabio semantic-model list-users --workspace $WS --id $SM

# Grant ReadWrite access to a user
fabio semantic-model add-user --workspace $WS --id $SM \
  --content '{"identifier":"user@company.com","principalType":"User","datasetUserAccessRight":"ReadWrite"}'

# Remove a user
fabio semantic-model delete-user --workspace $WS --id $SM --user "user@company.com"

# Check refresh history
fabio semantic-model refresh-status --workspace $WS --id $SM --top 5
```

### Parameters and Data Sources

```bash
# List M parameters
fabio semantic-model list-parameters --workspace $WS --id $SM

# Update parameters (change database connection)
fabio semantic-model update-parameters --workspace $WS --id $SM \
  --content '{"updateDetails":[{"name":"DatabaseServer","newValue":"prod-server.database.windows.net"}]}'

# List data sources
fabio semantic-model list-datasources --workspace $WS --id $SM
```

## REST Passthrough

### Direct API Calls

```bash
# GET request to Fabric API
fabio rest call --method GET --path "/workspaces" --query-params "roles=Admin"

# POST with body from file
fabio rest call --method POST --path "/workspaces/$WS/items" --body @item.json

# POST with stdin body
echo '{"displayName":"test"}' | fabio rest call --method POST --path "/workspaces/$WS/items" --body @-

# Call Power BI API directly
fabio rest call --method GET --path "/groups/$WS/datasets" --api powerbi

# LRO-aware call (poll until complete)
fabio rest call --method POST --path "/workspaces/$WS/items/$ID/getDefinition" --body '{}' --poll

# Dry-run for mutations
fabio rest call --method DELETE --path "/workspaces/$WS/items/$ID" --dry-run
```

## Natural Language to KQL

```bash
# Translate a question to KQL
fabio rti nl-to-kql --workspace $WS --item-id $KDB \
  --cluster-url "https://$EH_ID.eastus.kusto.fabric.microsoft.com" \
  --database "SensorDB" \
  --question "What is the average temperature in the last 24 hours?"

# With few-shot examples for better accuracy
fabio rti nl-to-kql --workspace $WS --item-id $KDB \
  --cluster-url "$KUSTO_URI" \
  --database "SensorDB" \
  --question "Show devices with temperature above 80" \
  --user-shots '[{"naturalLanguage":"count events","kqlQuery":"SensorEvents | count"}]'
```

## Capacity Lifecycle Management (ARM API)

```bash
# List available SKUs in your subscription
fabio capacity list-skus --subscription $SUB_ID

# Check if a capacity name is available
fabio capacity check-name --subscription $SUB_ID --location eastus --name "myanalyticsf4"

# Create a new F4 capacity
fabio capacity create --subscription $SUB_ID --resource-group "fabric-rg" \
  --name "myanalyticsf4" --location eastus --sku F4 --admin "admin@company.com"

# Suspend capacity (save costs when not in use)
fabio capacity suspend --subscription $SUB_ID --resource-group "fabric-rg" --name "myanalyticsf4"

# Resume capacity
fabio capacity resume --subscription $SUB_ID --resource-group "fabric-rg" --name "myanalyticsf4"

# Scale up to F8
fabio capacity update --subscription $SUB_ID --resource-group "fabric-rg" \
  --name "myanalyticsf4" --sku F8

# Delete capacity
fabio capacity delete --subscription $SUB_ID --resource-group "fabric-rg" --name "myanalyticsf4"
```

## Deploy Validate (Pre-Flight Checks)

```bash
# Validate source directory before planning
fabio deploy validate --source ./fabric-items

# Full CI/CD pipeline
fabio deploy validate --source ./fabric-items && \
fabio deploy plan --source ./fabric-items --workspace $WS && \
fabio deploy apply --source ./fabric-items --workspace $WS
```

## Item Bulk Operations

```bash
# Bulk create items (parallel, rate-limit aware)
fabio item bulk-create --workspace $WS --items '[
  {"displayName":"Lake1","type":"Lakehouse"},
  {"displayName":"Lake2","type":"Lakehouse"},
  {"displayName":"Notebook1","type":"Notebook"}
]'

# Bulk delete items
fabio item bulk-delete --workspace $WS --ids '["id1","id2","id3"]'

# Hard delete (skip recycle bin)
fabio item bulk-delete --workspace $WS --ids '["id1","id2"]' --hard-delete

# Check if an item exists (never errors on 404)
fabio item exists --workspace $WS --id $ITEM_ID

# Get portal URL for an item
fabio item url --workspace $WS --id $ITEM_ID

# Move item to a folder (or root)
fabio item move-to-folder --workspace $WS --id $ITEM_ID --folder-id $FOLDER_ID
fabio item move-to-folder --workspace $WS --id $ITEM_ID  # moves to root
```

## Notebook Run with Parameters

```bash
# Run with parameters (wait for completion)
fabio notebook run --workspace $WS --id $NB --wait --timeout 900 \
  --parameters '[{"name":"start_date","value":"2024-01-01","type":"Text"},{"name":"batch_size","value":"1000","type":"Int"}]'

# Get definition without outputs (for git)
fabio notebook get-definition --workspace $WS --id $NB --strip-output

# Run with custom compute type
fabio notebook run --workspace $WS --id $NB --wait \
  --compute-type "Spark" --execution-data '{"configuration":{"conf":{"spark.dynamicAllocation.enabled":"true"}}}'
```

## Dataflow Operations

```bash
# List M parameters in a dataflow
fabio dataflow discover-parameters --workspace $WS --id $DF

# Run a dataflow (wait for completion)
fabio dataflow run --workspace $WS --id $DF --wait --timeout 600

# Execute a query and save results as Arrow IPC
fabio dataflow execute-query --workspace $WS --id $DF \
  --query-name "TransformedData" --file ./output/results.arrow
```

## Lakehouse Table Maintenance

```bash
# Optimize a table (V-Order compaction)
fabio lakehouse optimize-table --workspace $WS --id $LH --table sales --v-order

# Optimize with Z-Order clustering
fabio lakehouse optimize-table --workspace $WS --id $LH --table sales \
  --v-order --z-order-by "country,date"

# Vacuum a table (remove old files, 7-day retention)
fabio lakehouse vacuum-table --workspace $WS --id $LH --table sales --retention "7:00:00:00"

# Get table schema without Spark (reads Delta log)
fabio lakehouse table-schema --workspace $WS --id $LH --table sales

# Query lakehouse via SQL endpoint
fabio lakehouse query --workspace $WS --id $LH --sql "SELECT COUNT(*) FROM dbo.sales"
```

## Authentication Workflows

### CI/CD with Service Principal (GitHub Actions OIDC)

```bash
# In GitHub Actions (OIDC federated token — no secrets required)
# In workflow YAML: permissions: { id-token: write }

# Fetch the OIDC JWT from GitHub's token endpoint
OIDC_TOKEN=$(curl -sS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
  "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=api://AzureADTokenExchange" | jq -r .value)

# Use --federated-token directly with the JWT value
fabio auth login --service-principal \
  --tenant $AZURE_TENANT_ID \
  --client-id $AZURE_CLIENT_ID \
  --federated-token "$OIDC_TOKEN"

# Or write to file and use --federated-token-file
echo "$OIDC_TOKEN" > /tmp/oidc_token
fabio auth login --service-principal \
  --tenant $AZURE_TENANT_ID \
  --client-id $AZURE_CLIENT_ID \
  --federated-token-file /tmp/oidc_token

# Or using Azure Login action first (sets AZURE_* env vars):
# - uses: azure/login@v3
# Then env vars are picked up by DefaultAzureCredential automatically
fabio auth status
```

### CI/CD with Service Principal (Client Secret)

```bash
fabio auth login --service-principal \
  --tenant $TENANT_ID \
  --client-id $CLIENT_ID \
  --client-secret $CLIENT_SECRET

fabio auth status
fabio workspace list
```

### Browser PKCE (Interactive / macOS SSO)

```bash
# Opens system browser — SSO on macOS with Enterprise SSO Extension
fabio auth login --browser
fabio auth status
```

### Windows WAM Broker SSO

```powershell
# Windows only — uses current Windows account, no browser or device code needed
fabio auth login --wam
fabio auth status
```

## JMESPath Query Examples

```bash
# List projection — MUST use [*].field for lists (breaking change in v0.18.0)
fabio workspace list --query '[*].displayName'
fabio lakehouse list-tables --workspace $WS --id $LH --query '[*].name'

# Filter by field value
fabio item list --workspace $WS --query '[?type==`Notebook`].displayName'

# Multiselect hash (reshape output)
fabio workspace list --query '[*].{name: displayName, id: id}'

# Count items
fabio workspace list --query 'length(@)'

# Sort by field
fabio workspace list --query 'sort_by(@, &displayName)[*].displayName'

# Pipe to first element
fabio item list --workspace $WS --query '[?type==`Lakehouse`].id | [0]' -o plain

# Nested field access (backward-compatible)
fabio workspace show --id $WS --query 'data.capacityId'
```

## Atomic File/Table Move (Same-Item)

```bash
# Same-item file moves use atomic x-ms-rename-source (O(1), no data transfer)
fabio lakehouse move-file --workspace $WS --id $LH \
  --source "Files/staging/data.csv" --dest "Files/processed/data.csv"

# Output includes "method": "rename" for same-item, "copy_delete" for cross-item
fabio lakehouse move-file --workspace $WS --id $LH \
  --source "Files/raw/*.parquet" --dest "Files/archive/"

# Same-item move (same --dest-id as --id) uses atomic directory rename
# Cross-item move (different --dest-id) uses copy+delete
fabio lakehouse move-table --workspace $WS --id $LH \
  --table raw_orders --dest-workspace $WS2 --dest-id $LH2
```
