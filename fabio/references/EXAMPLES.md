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
# Connect workspace to GitHub
fabio git connect --workspace $WS \
  --provider GitHub --org myorg --repo fabric-project \
  --branch main --directory "/"

# Initialize (required after connect)
fabio git init --workspace $WS

# Check status
fabio git status --workspace $WS

# Commit all changes
fabio git commit --workspace $WS --all -m "Add new lakehouse tables"

# Pull remote changes
fabio git pull --workspace $WS

# Switch branches
fabio git checkout --workspace $WS --branch feature/new-pipeline

# Show tracked items
fabio git show-tracked --workspace $WS
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
# FORBIDDEN         -> Check workspace role assignment
# NOT_FOUND         -> Verify ID with list command
# CONFLICT          -> Name already exists, choose different name
# RATE_LIMITED      -> Retry after delay (fabio retries automatically)
# CAPACITY_INACTIVE -> Resume/assign capacity to workspace
# INVALID_INPUT     -> Check PascalCase values, required fields
# TIMEOUT           -> Increase --timeout or check capacity status
```
