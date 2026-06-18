# fabio Command Reference

Complete command reference organized by functional area (74 groups, 843+ subcommands).

## Global Flags

These flags apply to all commands:

```
-o, --output <format>       Output format: json (default), table, plain, csv, tsv
-q, --query <jmespath>      JMESPath expression — requires [*].field for list projection
-v, --verbose               HTTP/LRO/auth diagnostics to stderr (for debugging only — do not use in normal agent flows)
    --quiet                 Suppress stdout (errors still go to stderr)
    --dry-run               Preview mutations without executing
    --limit <n>             Truncate list results client-side
    --all                   Auto-paginate all results
    --continuation-token    Resume pagination from a specific token
    --profile <name>        Use a named profile
    --lro-timeout <secs>    Custom LRO polling timeout (default 120s)
    --hard-delete           Permanently delete (skip recycle bin) — supported on all 38 item type deletes
```

> **Note for AI agents**: Use `--verbose` only for debugging HTTP issues. It emits request/response traces to stderr and does NOT affect stdout JSON output.

## Core Commands

### auth
```
fabio auth login             Log in (device code default; see variants below)
  Variants:
    (no flags)               Device code flow (headless/SSH)
    --browser                Browser PKCE (faster; SSO on macOS Enterprise Extension)
    --service-principal --tenant <tid> --client-id <cid> --client-secret <secret>
    --service-principal --tenant <tid> --client-id <cid> --certificate <path> [--certificate-password <pw>]
    --service-principal --tenant <tid> --client-id <cid> --federated-token <token>
    --service-principal --tenant <tid> --client-id <cid> --federated-token-file <path>
    --wam                    Windows WAM broker SSO (Windows only)
fabio auth logout            Log out and clear cached credentials
fabio auth status            Show authentication status and credential source
```

### workspace
```
fabio workspace list                    List all workspaces
fabio workspace show --id <id>          Show workspace details
fabio workspace create --name <name>    Create a new workspace
fabio workspace update --id <id> --name <new-name>  Update properties
fabio workspace delete --id <id>        Delete a workspace
fabio workspace assign-capacity --id <id> --capacity <cap-id>
fabio workspace unassign-capacity --id <id>
fabio workspace provision-identity --id <id>
fabio workspace deprovision-identity --id <id>
fabio workspace list-role-assignments --id <id>
fabio workspace show-role-assignment --id <id> --assignment-id <aid>
fabio workspace add-role-assignment --id <id> --principal <pid> --principal-type <User|Group|ServicePrincipal> --role <Admin|Member|Contributor|Viewer>
fabio workspace update-role-assignment --id <id> --assignment-id <aid> --role <role>
fabio workspace delete-role-assignment --id <id> --assignment-id <aid>
fabio workspace list-folders --id <id>
fabio workspace create-folder --id <id> --name <name>
fabio workspace show-folder --id <id> --folder-id <fid>
fabio workspace update-folder --id <id> --folder-id <fid> --name <name>
fabio workspace delete-folder --id <id> --folder-id <fid>
fabio workspace move-folder --id <id> --folder-id <fid> [--parent-folder <pid>]
fabio workspace apply-tags --id <id> --tag-ids '["uuid1","uuid2"]'
fabio workspace unapply-tags --id <id> --tag-ids '["uuid1"]'
fabio workspace assign-to-domain --id <id> --domain <domain-id>
fabio workspace unassign-from-domain --id <id>
fabio workspace get-onelake-settings --id <id>
fabio workspace modify-default-tier --id <id> --tier <Hot|Cold>
fabio workspace modify-diagnostics --id <id> ...
fabio workspace modify-immutability-policy --id <id> ...
fabio workspace export-lifecycle-policy --id <id>
fabio workspace import-lifecycle-policy --id <id> --file <path>
fabio workspace reset-shortcut-cache --id <id>
fabio workspace get-network-policy --id <id>
fabio workspace set-network-policy --id <id> [--file <path>|--content <json>]
fabio workspace get-firewall-rules --id <id>
fabio workspace set-firewall-rules --id <id> [--file <path>|--content <json>]
fabio workspace get-git-outbound-policy --id <id>
fabio workspace set-git-outbound-policy --id <id> [--file <path>|--content <json>]
fabio workspace get-inbound-azure-resource-rules --id <id>
fabio workspace set-inbound-azure-resource-rules --id <id> [--file <path>|--content <json>]
fabio workspace get-outbound-cloud-connection-rules --id <id>
fabio workspace set-outbound-cloud-connection-rules --id <id> [--file <path>|--content <json>]
fabio workspace get-outbound-gateway-rules --id <id>
fabio workspace set-outbound-gateway-rules --id <id> [--file <path>|--content <json>]
fabio workspace get-settings --id <id>
fabio workspace update-settings --id <id> [--file <path>|--content <json>]
fabio workspace get-dataset-storage-format --id <id>
fabio workspace set-dataset-storage-format --id <id> --format <Large|Small>
```

### item (generic item operations)
```
fabio item list --workspace <ws> [--type <ItemType>] [--folder <folder-id>] [--recursive]
fabio item show --workspace <ws> --id <id>
fabio item create --workspace <ws> --name <name> --type <type>
fabio item update --workspace <ws> --id <id> --name <new-name>
fabio item delete --workspace <ws> --id <id> [--hard-delete]
fabio item copy --workspace <ws> --id <id> --dest-workspace <dest>
fabio item move --workspace <ws> --id <id> --dest-workspace <dest>
fabio item get-definition --workspace <ws> --id <id>
fabio item update-definition --workspace <ws> --id <id> --file <path>
fabio item list-connections --workspace <ws> --id <id>
fabio item exists --workspace <ws> --id <id>
fabio item url --workspace <ws> --id <id>
fabio item inspect --workspace <ws> --id <id>
fabio item move-to-folder --workspace <ws> --id <id> [--folder-id <fid>]
fabio item apply-tags --workspace <ws> --id <id> --tag-ids '["uuid"]'
fabio item unapply-tags --workspace <ws> --id <id> --tag-ids '["uuid"]'
fabio item bulk-create --workspace <ws> --items <json>
fabio item bulk-delete --workspace <ws> --ids '["id1","id2"]' [--hard-delete]
fabio item bulk-export-definitions --workspace <ws> ...
fabio item bulk-import-definitions --workspace <ws> ...
fabio item bulk-move --workspace <ws> --ids '["id1","id2"]' --dest-workspace <dest>
fabio item list-external-data-shares --workspace <ws> --id <id>
fabio item create-external-data-share --workspace <ws> --id <id> --recipient-type <User|ServicePrincipal> --recipient-id <id> ...
fabio item show-external-data-share --workspace <ws> --id <id> --share-id <sid>
fabio item revoke-external-data-share --workspace <ws> --id <id> --share-id <sid>
fabio item delete-external-data-share --workspace <ws> --id <id> --share-id <sid>
fabio item assign-identity --workspace <ws> --id <id> ...
fabio item get-invitation --workspace <ws> --id <id> ...
fabio item accept-invitation --workspace <ws> --id <id> ...
```

### lakehouse
```
fabio lakehouse list --workspace <ws>
fabio lakehouse show --workspace <ws> --id <id>
fabio lakehouse create --workspace <ws> --name <name>
fabio lakehouse update --workspace <ws> --id <id> --name <new-name>
fabio lakehouse delete --workspace <ws> --id <id>
fabio lakehouse list-tables --workspace <ws> --id <id>
fabio lakehouse list-files --workspace <ws> --id <id> [--path <dir>]
fabio lakehouse upload --workspace <ws> --id <id> --source <glob> --dest <path>
fabio lakehouse download --workspace <ws> --id <id> --path <remote> --dest <local>
fabio lakehouse upload-table --workspace <ws> --id <id> --source <file> --table <name> --mode <Overwrite|Append> --format <Csv|Parquet>
fabio lakehouse load-table --workspace <ws> --id <id> --path <Files/...> --table <name> --mode <Overwrite|Append> --format <Csv|Parquet>
fabio lakehouse copy-file --workspace <ws> --id <id> --source <glob> --dest-workspace <ws2> --dest-id <lh2> --dest <path>
fabio lakehouse move-file --workspace <ws> --id <id> --source <glob> --dest <path>     # atomic rename for same-item, copy+delete for cross-item
fabio lakehouse delete-file --workspace <ws> --id <id> --path <file>
fabio lakehouse copy-table --workspace <ws> --id <id> --table <name> --dest-workspace <ws2> --dest-id <lh2>
fabio lakehouse move-table --workspace <ws> --id <id> --table <name> --dest-workspace <ws2> --dest-id <lh2>   # atomic rename when dest-id == id (same lakehouse), copy+delete otherwise
fabio lakehouse delete-table --workspace <ws> --id <id> --table <name>
fabio lakehouse sync (--workspace <ws> --id <id> --dest-workspace <ws2> --dest-id <lh2> | --local <dir> --dest-workspace <ws> --dest-id <lh> --dest-path <path>) [--checksum] [--include <patterns>] [--exclude <patterns>] [--no-recursive] [--remove-source-files] [--min-size <size>] [--max-size <size>] [--itemize]
  Remote-to-remote only: [--delete] [--size-only] [--no-overwrite] [--force] [--max-delete <n>] [--existing]
fabio lakehouse create-shortcut --workspace <ws> --id <id> --name <name> --path <path> --target-type <adls|s3|onelake> --location <url> [--subpath <sub>]
fabio lakehouse get-shortcut --workspace <ws> --id <id> --name <name> --path <path>
fabio lakehouse delete-shortcut --workspace <ws> --id <id> --name <name> --path <path>
fabio lakehouse bulk-create-shortcuts --workspace <ws> --id <id> --file <path> [--conflict-policy <Abort|Replace>]
fabio lakehouse get-definition --workspace <ws> --id <id>
fabio lakehouse update-definition --workspace <ws> --id <id> --file <path>
fabio lakehouse refresh-materialized-views --workspace <ws> --id <id>
fabio lakehouse create-materialized-views-schedule --workspace <ws> --id <id> ...
fabio lakehouse update-materialized-views-schedule --workspace <ws> --id <id> ...
fabio lakehouse delete-materialized-views-schedule --workspace <ws> --id <id>
fabio lakehouse run-table-maintenance --workspace <ws> --id <id>
fabio lakehouse optimize-table --workspace <ws> --id <id> --table <name> [--v-order] [--z-order-by <col1,col2>]
fabio lakehouse vacuum-table --workspace <ws> --id <id> --table <name> [--retention <D:HH:MM:SS>]
fabio lakehouse table-schema --workspace <ws> --id <id> --table <name>
fabio lakehouse query --workspace <ws> --id <id> --sql <"query"|@file|stdin>
fabio lakehouse list-livy-sessions --workspace <ws> --id <id>
fabio lakehouse get-livy-session --workspace <ws> --id <id> --session-id <sid>
```

### capacity
```
fabio capacity list
fabio capacity show --id <id>
fabio capacity suspend --subscription <sub> --resource-group <rg> --name <name>
fabio capacity resume --subscription <sub> --resource-group <rg> --name <name>
fabio capacity create --subscription <sub> --resource-group <rg> --name <name> --location <region> --sku <F2-F2048> --admin <email>
fabio capacity update --subscription <sub> --resource-group <rg> --name <name> [--sku <sku>] [--admin <email>]
fabio capacity delete --subscription <sub> --resource-group <rg> --name <name>
fabio capacity list-skus --subscription <sub>
fabio capacity check-name --subscription <sub> --location <region> --name <name>
```

### catalog
```
fabio catalog search --content '{"searchString":"<text>","top":N}'
```

## Data & Compute

### notebook
```
fabio notebook list --workspace <ws>
fabio notebook show --workspace <ws> --id <id>
fabio notebook create --workspace <ws> --name <name> [--lakehouse <lh-id>] [--source <file.py>]
fabio notebook update --workspace <ws> --id <id> --name <new-name>
fabio notebook delete --workspace <ws> --id <id>
fabio notebook get-definition --workspace <ws> --id <id> [--strip-output]
fabio notebook update-definition --workspace <ws> --id <id> --source <file.py>
fabio notebook run --workspace <ws> --id <id> [--wait] [--timeout <secs>] [--parameters <json>] [--compute-type <type>] [--execution-data <json>]
fabio notebook status --workspace <ws> --id <id>
fabio notebook get-job-instance --workspace <ws> --id <id> --instance-id <iid>
fabio notebook stop --workspace <ws> --id <id>
fabio notebook list-livy-sessions --workspace <ws> --id <id>
fabio notebook get-livy-session --workspace <ws> --id <id> --session-id <sid>
```

### warehouse
```
fabio warehouse list --workspace <ws>
fabio warehouse show --workspace <ws> --id <id>
fabio warehouse create --workspace <ws> --name <name>
fabio warehouse update --workspace <ws> --id <id> --name <new-name>
fabio warehouse delete --workspace <ws> --id <id>
fabio warehouse query --workspace <ws> --id <id> --sql <"query"|@file|stdin>
fabio warehouse connection-string --workspace <ws> --id <id>
fabio warehouse get-sql-pools-config --workspace <ws> --id <id>
fabio warehouse update-sql-pools-config --workspace <ws> --id <id> ...
fabio warehouse get-audit-settings --workspace <ws> --id <id>
fabio warehouse update-audit-settings --workspace <ws> --id <id> ...
fabio warehouse set-audit-actions --workspace <ws> --id <id> ...
fabio warehouse list-restore-points --workspace <ws> --id <id>
fabio warehouse create-restore-point --workspace <ws> --id <id> --name <name>
fabio warehouse show-restore-point --workspace <ws> --id <id> --restore-point-id <rpid>
fabio warehouse update-restore-point --workspace <ws> --id <id> --restore-point-id <rpid> ...
fabio warehouse delete-restore-point --workspace <ws> --id <id> --restore-point-id <rpid>
fabio warehouse restore-to-point --workspace <ws> --id <id> --restore-point-id <rpid> ...
```

### sql-database
```
fabio sql-database list --workspace <ws>
fabio sql-database show --workspace <ws> --id <id>
fabio sql-database create --workspace <ws> --name <name>
fabio sql-database update --workspace <ws> --id <id> --name <new-name>
fabio sql-database delete --workspace <ws> --id <id>
fabio sql-database query --workspace <ws> --id <id> --sql <"query"|@file|stdin>
fabio sql-database connection-string --workspace <ws> --id <id>
fabio sql-database import --workspace <ws> --id <id> --file <path> --table <name> [--drop-if-exists] [--no-create-table] [--batch-size <n>]
fabio sql-database get-definition --workspace <ws> --id <id>
fabio sql-database update-definition --workspace <ws> --id <id> --file <path>
fabio sql-database start-mirroring --workspace <ws> --id <id>
fabio sql-database stop-mirroring --workspace <ws> --id <id>
fabio sql-database revalidate-cmk --workspace <ws> --id <id>
fabio sql-database get-audit-settings --workspace <ws> --id <id>
fabio sql-database update-audit-settings --workspace <ws> --id <id> ...
fabio sql-database list-deleted --workspace <ws>
```

### sql-endpoint
```
fabio sql-endpoint list --workspace <ws>
fabio sql-endpoint show --workspace <ws> --id <id>
fabio sql-endpoint connection-string --workspace <ws> --id <id>
fabio sql-endpoint query --workspace <ws> --id <id> --sql "<T-SQL>"
  # Input modes: --sql "inline SQL", --sql @file.sql, or pipe via stdin
fabio sql-endpoint refresh-metadata --workspace <ws> --id <id>
fabio sql-endpoint get-audit-settings --workspace <ws> --id <id>
fabio sql-endpoint update-audit-settings --workspace <ws> --id <id> ...
fabio sql-endpoint set-audit-actions --workspace <ws> --id <id> ...
```

### data-pipeline
```
fabio data-pipeline list --workspace <ws>
fabio data-pipeline show --workspace <ws> --id <id>
fabio data-pipeline create --workspace <ws> --name <name>
fabio data-pipeline update --workspace <ws> --id <id> --name <new-name>
fabio data-pipeline delete --workspace <ws> --id <id>
fabio data-pipeline run --workspace <ws> --id <id>
fabio data-pipeline get-definition --workspace <ws> --id <id>
fabio data-pipeline update-definition --workspace <ws> --id <id> --file <path>
fabio data-pipeline create-schedule --workspace <ws> --id <id> (--file <path> | --content '<json>')
fabio data-pipeline list-schedules --workspace <ws> --id <id>
fabio data-pipeline get-schedule --workspace <ws> --id <id> --schedule-id <schedule-id>
fabio data-pipeline update-schedule --workspace <ws> --id <id> --schedule-id <schedule-id> (--file <path> | --content '<json>')
fabio data-pipeline delete-schedule --workspace <ws> --id <id> --schedule-id <schedule-id>
fabio data-pipeline list-instances --workspace <ws> --id <id>
fabio data-pipeline get-instance --workspace <ws> --id <id> --instance-id <instance-id>
```

### copy-job
```
fabio copy-job list --workspace <ws>
fabio copy-job show --workspace <ws> --id <id>
fabio copy-job create --workspace <ws> --name <name>
fabio copy-job update --workspace <ws> --id <id> --name <new-name>
fabio copy-job delete --workspace <ws> --id <id>
fabio copy-job get-definition --workspace <ws> --id <id>
fabio copy-job update-definition --workspace <ws> --id <id> --file <path>
fabio copy-job reset --workspace <ws> --id <id> (--all | --entity-ids <uuid,uuid>)   Reset all entities or specific entities for re-processing
```

### dataflow
```
fabio dataflow list --workspace <ws>
fabio dataflow show --workspace <ws> --id <id>
fabio dataflow create --workspace <ws> --name <name>
fabio dataflow update --workspace <ws> --id <id> --name <new-name>
fabio dataflow delete --workspace <ws> --id <id>
fabio dataflow get-definition --workspace <ws> --id <id>
fabio dataflow update-definition --workspace <ws> --id <id> --file <path>
fabio dataflow discover-parameters --workspace <ws> --id <id>
fabio dataflow run --workspace <ws> --id <id> [--wait] [--timeout <secs>]
fabio dataflow execute-query --workspace <ws> --id <id> --query-name <name> [--file <output>]
```

### environment
```
fabio environment list --workspace <ws>
fabio environment show --workspace <ws> --id <id>
fabio environment create --workspace <ws> --name <name>
fabio environment update --workspace <ws> --id <id> --name <new-name>
fabio environment delete --workspace <ws> --id <id>
fabio environment publish --workspace <ws> --id <id>
fabio environment cancel-publish --workspace <ws> --id <id>
fabio environment get-spark-settings --workspace <ws> --id <id>
fabio environment get-staging-spark-settings --workspace <ws> --id <id>
fabio environment get-definition --workspace <ws> --id <id>
fabio environment update-definition --workspace <ws> --id <id> --file <path>
fabio environment list-libraries --workspace <ws> --id <id>
fabio environment export-libraries --workspace <ws> --id <id>
fabio environment list-staging-libraries --workspace <ws> --id <id>
fabio environment delete-staging-library --workspace <ws> --id <id> --library <name>
fabio environment export-staging-libraries --workspace <ws> --id <id>
fabio environment import-staging-libraries --workspace <ws> --id <id> --file <path>
fabio environment remove-staging-library --workspace <ws> --id <id> --library <name>
fabio environment update-staging-spark-compute --workspace <ws> --id <id> ...
```

### data-agent
```
fabio data-agent list --workspace <ws>
fabio data-agent show --workspace <ws> --id <id>
fabio data-agent create --workspace <ws> --name <name> [--description <desc>]
fabio data-agent update --workspace <ws> --id <id> --name <new-name>
fabio data-agent delete --workspace <ws> --id <id>
fabio data-agent get-definition --workspace <ws> --id <id>
fabio data-agent update-definition --workspace <ws> --id <id> --file <path>

# Configuration management
fabio data-agent get-config --workspace <ws> --id <id>
fabio data-agent update-config --workspace <ws> --id <id>
  [--instructions <text>] [--instructions-file <path>]
  [--enable-preview-runtime] [--disable-preview-runtime]

# Datasource management
fabio data-agent list-datasources --workspace <ws> --id <id>
fabio data-agent show-datasource --workspace <ws> --id <id> --datasource <name-or-id>
fabio data-agent add-datasource --workspace <ws> --id <id> --artifact <name-or-id>
  [--artifact-type <Lakehouse|Warehouse|KQLDatabase|SemanticModel|...>]
  [--artifact-workspace <ws-id>] [--instructions <text>]
fabio data-agent remove-datasource --workspace <ws> --id <id> --datasource <name-or-id>
fabio data-agent select-tables --workspace <ws> --id <id> --datasource <name-or-id>
  (--tables <t1,t2> | --all-tables) [--unselect]

# Element descriptions
fabio data-agent list-elements --workspace <ws> --id <id> --datasource <name-or-id>
fabio data-agent describe-element --workspace <ws> --id <id> --datasource <name-or-id>
  --path <dbo.table.column> [--description <text>]

# Few-shot examples
fabio data-agent list-fewshots --workspace <ws> --id <id> --datasource <name-or-id>
fabio data-agent add-fewshot --workspace <ws> --id <id> --datasource <name-or-id>
  --question <text> --answer <sql-or-kql>   (--sql is alias for --answer)
fabio data-agent remove-fewshot --workspace <ws> --id <id> --datasource <name-or-id>
  --fewshot-id <uuid>
fabio data-agent upload-fewshots --workspace <ws> --id <id> --datasource <name-or-id>
  --file <path>   (JSON: [{"question":"...","query":"..."}] or CSV with question/query columns)

# Query and publish
fabio data-agent query --workspace <ws> --id <id> [--prompt <text>|stdin]
  [--stage sandbox|production] [--timeout <secs>] [--show-steps]
fabio data-agent publish --workspace <ws> --id <id> [--description <text>] [--to-m365]
```

### ontology
```
fabio ontology list --workspace <ws>
fabio ontology show --workspace <ws> --id <id>
fabio ontology create --workspace <ws> --name <name>
fabio ontology update --workspace <ws> --id <id> --name <new-name>
fabio ontology delete --workspace <ws> --id <id>
fabio ontology get-definition --workspace <ws> --id <id> [--decode] [--dir <path>]
fabio ontology update-definition --workspace <ws> --id <id> --file <path> [--dir <path>]
```

### graphql-api
```
fabio graphql-api list --workspace <ws>
fabio graphql-api show --workspace <ws> --id <id>
fabio graphql-api create --workspace <ws> --name <name>
fabio graphql-api update --workspace <ws> --id <id> --name <new-name>
fabio graphql-api delete --workspace <ws> --id <id>
fabio graphql-api get-definition --workspace <ws> --id <id>
fabio graphql-api update-definition --workspace <ws> --id <id> --file <path>
```

### warehouse-snapshot
```
fabio warehouse-snapshot list --workspace <ws>
fabio warehouse-snapshot show --workspace <ws> --id <id>
fabio warehouse-snapshot create --workspace <ws> --name <name> --warehouse-id <wh-id>
fabio warehouse-snapshot update --workspace <ws> --id <id> --name <new-name>
fabio warehouse-snapshot delete --workspace <ws> --id <id>
```

## Analytics & Reporting

### semantic-model
```
fabio semantic-model list --workspace <ws>
fabio semantic-model show --workspace <ws> --id <id>
fabio semantic-model create --workspace <ws> --name <name> --file <model.tmdl|model.bim> [--connection <sql-endpoint-id>]
fabio semantic-model update --workspace <ws> --id <id> --name <new-name>
fabio semantic-model delete --workspace <ws> --id <id>
fabio semantic-model get-definition --workspace <ws> --id <id>
fabio semantic-model update-definition --workspace <ws> --id <id> --file <path>
fabio semantic-model query --workspace <ws> --id <id> --dax <"query">
fabio semantic-model bind-connection --workspace <ws> --id <id> --connection <conn-id>
fabio semantic-model unbind-connection --workspace <ws> --id <id>
fabio semantic-model refresh --workspace <ws> --id <id>
fabio semantic-model takeover --workspace <ws> --id <id>
# Power BI API commands (via api.powerbi.com)
fabio semantic-model list-parameters --workspace <ws> --id <id>
fabio semantic-model update-parameters --workspace <ws> --id <id> --content <json>
fabio semantic-model list-datasources --workspace <ws> --id <id>
fabio semantic-model update-datasources --workspace <ws> --id <id> --content <json>
fabio semantic-model list-users --workspace <ws> --id <id>
fabio semantic-model add-user --workspace <ws> --id <id> --content <json>
fabio semantic-model delete-user --workspace <ws> --id <id> --user <email-or-id>
fabio semantic-model refresh-status --workspace <ws> --id <id> [--top <N>]
fabio semantic-model list-upstream --workspace <ws> --id <id>
fabio semantic-model clone --workspace <ws> --id <id> --name <new-name> [--target-workspace <ws2>]
fabio semantic-model export-pbix --workspace <ws> --id <id> --file <output.pbix>
fabio semantic-model import-pbix --workspace <ws> --name <name> --file <input.pbix> [--name-conflict <Abort|Overwrite|CreateOrOverwrite|GenerateUniqueName>]
```

### report
```
fabio report list --workspace <ws>
fabio report show --workspace <ws> --id <id>
fabio report create --workspace <ws> --name <name> [--dataset <semantic-model-id>]
fabio report update --workspace <ws> --id <id> --name <new-name>
fabio report delete --workspace <ws> --id <id>
fabio report get-definition --workspace <ws> --id <id>
fabio report update-definition --workspace <ws> --id <id> --file <path> [--report-json <path>]
fabio report publish-to-web --workspace <ws> --id <id>
```

## Data Science

### operations-agent
```
fabio operations-agent list --workspace <ws>
fabio operations-agent show --workspace <ws> --id <id>
fabio operations-agent create --workspace <ws> --name <name>
fabio operations-agent update --workspace <ws> --id <id> --name <new-name>
fabio operations-agent delete --workspace <ws> --id <id>
fabio operations-agent get-definition --workspace <ws> --id <id>
fabio operations-agent update-definition --workspace <ws> --id <id> --file <path>
```

### apache-airflow-job
```
fabio apache-airflow-job list --workspace <ws>
fabio apache-airflow-job show --workspace <ws> --id <id>
fabio apache-airflow-job create --workspace <ws> --name <name>
fabio apache-airflow-job update --workspace <ws> --id <id> --name <new-name>
fabio apache-airflow-job delete --workspace <ws> --id <id>
fabio apache-airflow-job get-definition --workspace <ws> --id <id>
fabio apache-airflow-job update-definition --workspace <ws> --id <id> --file <path>
fabio apache-airflow-job start-environment --workspace <ws> --id <id>
fabio apache-airflow-job stop-environment --workspace <ws> --id <id>
fabio apache-airflow-job get-environment --workspace <ws> --id <id>
fabio apache-airflow-job list-files --workspace <ws> --id <id>
fabio apache-airflow-job get-file --workspace <ws> --id <id> --path <file-path>
fabio apache-airflow-job upload-file --workspace <ws> --id <id> --path <file-path> --content <text>
fabio apache-airflow-job delete-file --workspace <ws> --id <id> --path <file-path>
fabio apache-airflow-job get-compute --workspace <ws> --id <id>
fabio apache-airflow-job get-workspace-settings --workspace <ws> --id <id>
fabio apache-airflow-job deploy-requirements --workspace <ws> --id <id> --file <path>
```

### ml-model
```
fabio ml-model list --workspace <ws>
fabio ml-model show --workspace <ws> --id <id>
fabio ml-model create --workspace <ws> --name <name>
fabio ml-model update --workspace <ws> --id <id> --name <new-name>
fabio ml-model delete --workspace <ws> --id <id>
```

### ml-experiment
```
fabio ml-experiment list --workspace <ws>
fabio ml-experiment show --workspace <ws> --id <id>
fabio ml-experiment create --workspace <ws> --name <name>
fabio ml-experiment update --workspace <ws> --id <id> --name <new-name>
fabio ml-experiment delete --workspace <ws> --id <id>
```

### spark
```
fabio spark get-settings --workspace <ws>
fabio spark update-settings --workspace <ws> [--file <path>|--content <json>]
fabio spark list-pools --workspace <ws>
fabio spark get-pool --workspace <ws> --pool-id <pid>
fabio spark create-pool --workspace <ws> [--file <path>|--content <json>]
fabio spark update-pool --workspace <ws> --pool-id <pid> [--file <path>|--content <json>]
fabio spark delete-pool --workspace <ws> --pool-id <pid>
```

### spark-job-definition
```
fabio spark-job-definition list --workspace <ws>
fabio spark-job-definition show --workspace <ws> --id <id>
fabio spark-job-definition create --workspace <ws> --name <name>
fabio spark-job-definition update --workspace <ws> --id <id> --name <new-name>
fabio spark-job-definition delete --workspace <ws> --id <id>
fabio spark-job-definition get-definition --workspace <ws> --id <id>
fabio spark-job-definition update-definition --workspace <ws> --id <id> --file <path>
fabio spark-job-definition run --workspace <ws> --id <id>
```

### data-build-tool-job
```
fabio data-build-tool-job list --workspace <ws>
fabio data-build-tool-job show --workspace <ws> --id <id>
fabio data-build-tool-job create --workspace <ws> --name <name> [--description <desc>]
fabio data-build-tool-job update --workspace <ws> --id <id> [--name <new-name>] [--description <desc>]
fabio data-build-tool-job delete --workspace <ws> --id <id>
fabio data-build-tool-job get-definition --workspace <ws> --id <id>
fabio data-build-tool-job update-definition --workspace <ws> --id <id> --file <path>
fabio data-build-tool-job run --workspace <ws> --id <id> [--wait] [--timeout <secs>] [--cancel-on-timeout]
```
Notes:
- Preview item type (dbt integration)
- `run` uses item-specific job endpoint (not the generic items endpoint)
- `--wait` polls every 5s; default timeout 600s; terminal statuses: Completed, Failed, Cancelled

## Power Apps

### app-backend
```
fabio app-backend list --workspace <ws>
fabio app-backend show --workspace <ws> --id <id>
fabio app-backend create --workspace <ws> --name <name> [--description <desc>]
fabio app-backend update --workspace <ws> --id <id> [--name <new-name>] [--description <desc>]
fabio app-backend delete --workspace <ws> --id <id> [--hard-delete]
```

### org-app
```
fabio org-app list --workspace <ws>
fabio org-app show --workspace <ws> --id <id>
fabio org-app create --workspace <ws> --name <name> [--description <desc>]
fabio org-app update --workspace <ws> --id <id> [--name <new-name>] [--description <desc>]
fabio org-app delete --workspace <ws> --id <id>
fabio org-app get-definition --workspace <ws> --id <id>
fabio org-app update-definition --workspace <ws> --id <id> --file <path>
```

### org-app-audience
```
fabio org-app-audience list --workspace <ws>
fabio org-app-audience show --workspace <ws> --id <id>
fabio org-app-audience create --workspace <ws> --name <name> [--description <desc>]
fabio org-app-audience update --workspace <ws> --id <id> [--name <new-name>] [--description <desc>]
fabio org-app-audience delete --workspace <ws> --id <id>
fabio org-app-audience get-definition --workspace <ws> --id <id>
fabio org-app-audience update-definition --workspace <ws> --id <id> --file <path>
```

## Real-Time Intelligence

### eventhouse
```
fabio eventhouse list --workspace <ws>
fabio eventhouse show --workspace <ws> --id <id>
fabio eventhouse create --workspace <ws> --name <name>
fabio eventhouse update --workspace <ws> --id <id> --name <new-name>
fabio eventhouse delete --workspace <ws> --id <id>
fabio eventhouse get-definition --workspace <ws> --id <id>
fabio eventhouse update-definition --workspace <ws> --id <id> --file <path>
```

### kql-database
```
fabio kql-database list --workspace <ws>
fabio kql-database show --workspace <ws> --id <id>
fabio kql-database create --workspace <ws> --name <name> --eventhouse-id <eh-id>
fabio kql-database update --workspace <ws> --id <id> --name <new-name>
fabio kql-database delete --workspace <ws> --id <id>
fabio kql-database query --workspace <ws> --id <id> --kql <"query">
fabio kql-database get-definition --workspace <ws> --id <id>
fabio kql-database update-definition --workspace <ws> --id <id> --file <path>
fabio kql-database list-shortcuts --workspace <ws> --id <id>
fabio kql-database create-shortcut --workspace <ws> --id <id> ...
fabio kql-database get-shortcut --workspace <ws> --id <id> --name <name>
fabio kql-database delete-shortcut --workspace <ws> --id <id> --name <name>
fabio kql-database bulk-create-shortcuts --workspace <ws> --id <id> --file <path>

# ── Schema discovery (RTI intelligence) ──
fabio kql-database list-entities --workspace <ws> --id <id>   List tables, views, external tables, functions
fabio kql-database describe --workspace <ws> --id <id>        Full database schema (all columns in all tables)
fabio kql-database describe-entity --workspace <ws> --id <id> --entity-name <name>   Schema for one entity
fabio kql-database sample --workspace <ws> --id <id> --entity-name <name> [--count <n>]  Sample rows from entity

# ── Inline ingestion ──
fabio kql-database ingest --workspace <ws> --id <id> --table <name> --data <csv-text>   Inline CSV ingestion
  --table <name>    Target table name (must exist)
  --data <csv>      Inline CSV data (header row + data rows, newlines as \n)
  --dry-run         Validate without ingesting

# ── Query analysis ──
fabio kql-database show-queryplan --workspace <ws> --id <id> --kql <"query">   Execution plan without running
fabio kql-database diagnostics --workspace <ws> --id <id>     Cluster health, capacity, ingestion failures

# ── Portal deeplinks ──
fabio kql-database deeplink --workspace <ws> --id <id> --kql <"query">    Fabric or ADX Web Explorer URL
  Fabric URL: https://app.fabric.microsoft.com/groups/{ws}/kqlDatabases/{id}?query={encoded}
  ADX URL:    https://dataexplorer.azure.com/clusters/{uri}/databases/{db}?query={encoded}
```

### eventstream
```
fabio eventstream list --workspace <ws>
fabio eventstream show --workspace <ws> --id <id>
fabio eventstream create --workspace <ws> --name <name>
fabio eventstream update --workspace <ws> --id <id> --name <new-name>
fabio eventstream delete --workspace <ws> --id <id>
fabio eventstream get-definition --workspace <ws> --id <id>
fabio eventstream update-definition --workspace <ws> --id <id> --file <path>
fabio eventstream get-topology --workspace <ws> --id <id>
fabio eventstream pause --workspace <ws> --id <id>
fabio eventstream resume --workspace <ws> --id <id>
fabio eventstream get-source --workspace <ws> --id <id> --source-id <sid>
fabio eventstream get-source-connection --workspace <ws> --id <id> --source-id <sid>
fabio eventstream pause-source --workspace <ws> --id <id> --source-id <sid>
fabio eventstream resume-source --workspace <ws> --id <id> --source-id <sid>
fabio eventstream get-destination --workspace <ws> --id <id> --dest-id <did>
fabio eventstream get-destination-connection --workspace <ws> --id <id> --dest-id <did>
fabio eventstream pause-destination --workspace <ws> --id <id> --dest-id <did>
fabio eventstream resume-destination --workspace <ws> --id <id> --dest-id <did>
fabio eventstream add-source --workspace <ws> --id <id> --name <name> --source-type <CustomEndpoint|...>
fabio eventstream add-destination --workspace <ws> --id <id> --name <name> --destination-type <Eventhouse|Lakehouse|...> --input-node <node-id> --properties <json>

# ── Builder helpers (high-level) ──
fabio eventstream add-sample-source --workspace <ws> --id <id> --name <name>   Add SampleData source
fabio eventstream add-derived-stream --workspace <ws> --id <id> --name <name> --input-node <node>  Add filtered/transformed stream

# ── Definition validation ──
fabio eventstream validate --file <path>   Client-side definition validation (no API call)
  --file <path>    Local eventstream definition JSON file
  Checks: node reference integrity, duplicates, required fields

# ── Component catalog ──
fabio eventstream list-components [--category <source|destination>]   List available source/destination types
```

### kql-queryset
```
fabio kql-queryset list --workspace <ws>
fabio kql-queryset show --workspace <ws> --id <id>
fabio kql-queryset create --workspace <ws> --name <name>
fabio kql-queryset update --workspace <ws> --id <id> --name <new-name>
fabio kql-queryset delete --workspace <ws> --id <id>
fabio kql-queryset get-definition --workspace <ws> --id <id>
fabio kql-queryset update-definition --workspace <ws> --id <id> --file <path>
fabio kql-queryset run --workspace <ws> --id <id> --tab <"tab name"|index>
```

### reflex (Data Activator)
```
fabio reflex list --workspace <ws>
fabio reflex show --workspace <ws> --id <id>
fabio reflex create --workspace <ws> --name <name>
fabio reflex update --workspace <ws> --id <id> --name <new-name>
fabio reflex delete --workspace <ws> --id <id>
fabio reflex get-definition --workspace <ws> --id <id>
fabio reflex update-definition --workspace <ws> --id <id> --file <path>
fabio reflex configure-kql-source --workspace <ws> --id <id> ...

# ── High-level trigger creation (auto-generates full entity hierarchy) ──
fabio reflex create-trigger --workspace <ws> --name <name>
  --eventhouse-id <eh-id>   Eventhouse containing the data source
  --database <db-name>      KQL database name
  --table <table-name>      Source table to monitor
  --condition <kql-expr>    KQL condition expression (e.g., "State == 'ILLINOIS' and EventType == 'Flood'")
  --action <email|teams>    Notification action type
  --recipients <emails>     Comma-separated recipient addresses
  [--message <text>]        Optional notification message body
  [--interval <secs>]       Check interval in seconds (default: 60)
  [--dry-run]               Show planned entity count and parameters without creating
```

### kql-dashboard
```
fabio kql-dashboard list --workspace <ws>
fabio kql-dashboard show --workspace <ws> --id <id>
fabio kql-dashboard create --workspace <ws> --name <name>
fabio kql-dashboard update --workspace <ws> --id <id> --name <new-name>
fabio kql-dashboard delete --workspace <ws> --id <id>
fabio kql-dashboard get-definition --workspace <ws> --id <id>
fabio kql-dashboard update-definition --workspace <ws> --id <id> --file <path>
```

### anomaly-detector
```
fabio anomaly-detector list --workspace <ws>
fabio anomaly-detector show --workspace <ws> --id <id>
fabio anomaly-detector create --workspace <ws> --name <name>
fabio anomaly-detector update --workspace <ws> --id <id> --name <new-name>
fabio anomaly-detector delete --workspace <ws> --id <id>
fabio anomaly-detector get-definition --workspace <ws> --id <id>
fabio anomaly-detector update-definition --workspace <ws> --id <id> --file <path>
```

### event-schema-set
```
fabio event-schema-set list --workspace <ws>
fabio event-schema-set show --workspace <ws> --id <id>
fabio event-schema-set create --workspace <ws> --name <name>
fabio event-schema-set update --workspace <ws> --id <id> --name <new-name>
fabio event-schema-set delete --workspace <ws> --id <id>
fabio event-schema-set get-definition --workspace <ws> --id <id>
fabio event-schema-set update-definition --workspace <ws> --id <id> --file <path>
```

## Graph & Digital Twins

### graph-model
```
fabio graph-model list --workspace <ws>
fabio graph-model show --workspace <ws> --id <id>
fabio graph-model create --workspace <ws> --name <name> [--ontology <ontology-id>]
fabio graph-model update --workspace <ws> --id <id> --name <new-name>
fabio graph-model delete --workspace <ws> --id <id>
fabio graph-model get-definition --workspace <ws> --id <id>
fabio graph-model update-definition --workspace <ws> --id <id> --file <path>
fabio graph-model refresh-graph --workspace <ws> --id <id>
fabio graph-model execute-query --workspace <ws> --id <id> --query <"KQL">
fabio graph-model get-queryable-graph-type --workspace <ws> --id <id>
```

### graph-query-set
```
fabio graph-query-set list --workspace <ws>
fabio graph-query-set show --workspace <ws> --id <id>
fabio graph-query-set create --workspace <ws> --name <name>
fabio graph-query-set update --workspace <ws> --id <id> --name <new-name>
fabio graph-query-set delete --workspace <ws> --id <id>
fabio graph-query-set get-definition --workspace <ws> --id <id>
fabio graph-query-set update-definition --workspace <ws> --id <id> --file <path>
```

### digital-twin-builder
```
fabio digital-twin-builder list --workspace <ws>
fabio digital-twin-builder show --workspace <ws> --id <id>
fabio digital-twin-builder create --workspace <ws> --name <name>
fabio digital-twin-builder update --workspace <ws> --id <id> --name <new-name>
fabio digital-twin-builder delete --workspace <ws> --id <id>
fabio digital-twin-builder get-definition --workspace <ws> --id <id>
fabio digital-twin-builder update-definition --workspace <ws> --id <id> --file <path>
```

### digital-twin-builder-flow
```
fabio digital-twin-builder-flow list --workspace <ws>
fabio digital-twin-builder-flow show --workspace <ws> --id <id>
fabio digital-twin-builder-flow create --workspace <ws> --name <name> --dtb-id <digital-twin-builder-id>
fabio digital-twin-builder-flow update --workspace <ws> --id <id> --name <new-name>
fabio digital-twin-builder-flow delete --workspace <ws> --id <id>
fabio digital-twin-builder-flow get-definition --workspace <ws> --id <id>
fabio digital-twin-builder-flow update-definition --workspace <ws> --id <id> --file <path>
```

### map
```
fabio map list --workspace <ws>
fabio map show --workspace <ws> --id <id>
fabio map create --workspace <ws> --name <name>
fabio map update --workspace <ws> --id <id> --name <new-name>
fabio map delete --workspace <ws> --id <id>
fabio map get-definition --workspace <ws> --id <id>
fabio map update-definition --workspace <ws> --id <id> --file <path>
```

## Mirroring

### mirrored-database
```
fabio mirrored-database list --workspace <ws>
fabio mirrored-database show --workspace <ws> --id <id>
fabio mirrored-database create --workspace <ws> --name <name>
fabio mirrored-database update --workspace <ws> --id <id> --name <new-name>
fabio mirrored-database delete --workspace <ws> --id <id>
fabio mirrored-database get-definition --workspace <ws> --id <id>
fabio mirrored-database update-definition --workspace <ws> --id <id> --file <path>
fabio mirrored-database start --workspace <ws> --id <id>
fabio mirrored-database stop --workspace <ws> --id <id>
fabio mirrored-database status --workspace <ws> --id <id>
fabio mirrored-database table-status --workspace <ws> --id <id>
```

### mirrored-catalog
```
fabio mirrored-catalog list --workspace <ws>
fabio mirrored-catalog show --workspace <ws> --id <id>
fabio mirrored-catalog create --workspace <ws> --name <name>
fabio mirrored-catalog update --workspace <ws> --id <id> --name <new-name>
fabio mirrored-catalog delete --workspace <ws> --id <id>
fabio mirrored-catalog get-definition --workspace <ws> --id <id>
fabio mirrored-catalog update-definition --workspace <ws> --id <id> --file <path>
fabio mirrored-catalog refresh-metadata --workspace <ws> --id <id>
fabio mirrored-catalog mirroring-status --workspace <ws> --id <id>
fabio mirrored-catalog tables-status --workspace <ws> --id <id>
```

### mirrored-databricks-catalog
```
fabio mirrored-databricks-catalog list --workspace <ws>
fabio mirrored-databricks-catalog show --workspace <ws> --id <id>
fabio mirrored-databricks-catalog create --workspace <ws> --name <name>
fabio mirrored-databricks-catalog update --workspace <ws> --id <id> --name <new-name>
fabio mirrored-databricks-catalog delete --workspace <ws> --id <id>
fabio mirrored-databricks-catalog get-definition --workspace <ws> --id <id>
fabio mirrored-databricks-catalog update-definition --workspace <ws> --id <id> --file <path>
fabio mirrored-databricks-catalog discover-catalogs --workspace <ws> --id <id>
fabio mirrored-databricks-catalog refresh-metadata --workspace <ws> --id <id>
fabio mirrored-databricks-catalog mirroring-status --workspace <ws> --id <id>
```

### mirrored-warehouse
```
fabio mirrored-warehouse list --workspace <ws>
```

### cosmos-db-database
```
fabio cosmos-db-database list --workspace <ws>
fabio cosmos-db-database show --workspace <ws> --id <id>
fabio cosmos-db-database create --workspace <ws> --name <name>
fabio cosmos-db-database update --workspace <ws> --id <id> --name <new-name>
fabio cosmos-db-database delete --workspace <ws> --id <id>
fabio cosmos-db-database get-definition --workspace <ws> --id <id>
fabio cosmos-db-database update-definition --workspace <ws> --id <id> --file <path>
```

### snowflake-database
```
fabio snowflake-database list --workspace <ws>
fabio snowflake-database show --workspace <ws> --id <id>
fabio snowflake-database create --workspace <ws> --name <name> --connection <json>
fabio snowflake-database update --workspace <ws> --id <id> --name <new-name>
fabio snowflake-database delete --workspace <ws> --id <id>
fabio snowflake-database get-definition --workspace <ws> --id <id>
fabio snowflake-database update-definition --workspace <ws> --id <id> --file <path>
```

### mounted-data-factory
```
fabio mounted-data-factory list --workspace <ws>
fabio mounted-data-factory show --workspace <ws> --id <id>
fabio mounted-data-factory create --workspace <ws> --name <name> --resource-id <ARM-resource-id>
fabio mounted-data-factory update --workspace <ws> --id <id> --name <new-name>
fabio mounted-data-factory delete --workspace <ws> --id <id>
fabio mounted-data-factory get-definition --workspace <ws> --id <id>
fabio mounted-data-factory update-definition --workspace <ws> --id <id> --file <path>
```

### azure-databricks-storage
```
fabio azure-databricks-storage list --workspace <ws>
fabio azure-databricks-storage show --workspace <ws> --id <id>
fabio azure-databricks-storage create --workspace <ws> --name <name> [--description <desc>]
fabio azure-databricks-storage update --workspace <ws> --id <id> [--name <new-name>] [--description <desc>]
fabio azure-databricks-storage delete --workspace <ws> --id <id> [--hard-delete]
fabio azure-databricks-storage get-definition --workspace <ws> --id <id> [--decode]
fabio azure-databricks-storage update-definition --workspace <ws> --id <id> (--file <path> | --content '<json>')
```

## Integration & DevOps

### git
```
fabio git status --workspace <ws>
fabio git commit --workspace <ws> --all -m <"message">
fabio git pull --workspace <ws>
fabio git connect --workspace <ws> --provider <GitHub|AzureDevOps> --org <org> --project <proj> --repo <repo> --branch <branch> --directory <dir>
fabio git disconnect --workspace <ws>
fabio git init --workspace <ws>
fabio git checkout --workspace <ws> --branch <branch>
fabio git connection show --workspace <ws>
fabio git credentials show --workspace <ws>
fabio git credentials update --workspace <ws> ...
fabio git show-tracked --workspace <ws>
```

### connection
```
fabio connection list
fabio connection show --id <id>
fabio connection create --name <name> --type <type> --params <json>
fabio connection update --id <id> ...
fabio connection delete --id <id>
fabio connection list-supported-types
fabio connection test-connection --id <id>
fabio connection list-role-assignments --id <id>
fabio connection add-role-assignment --id <id> ...
fabio connection show-role-assignment --id <id> --assignment-id <aid>
fabio connection update-role-assignment --id <id> --assignment-id <aid> ...
fabio connection delete-role-assignment --id <id> --assignment-id <aid>
```

### deployment-pipeline
```
fabio deployment-pipeline list
fabio deployment-pipeline show --id <id>
fabio deployment-pipeline create --name <name>
fabio deployment-pipeline update --id <id> --name <new-name>
fabio deployment-pipeline delete --id <id>
fabio deployment-pipeline list-stages --id <id>
fabio deployment-pipeline show-stage --id <id> --stage-id <sid>
fabio deployment-pipeline update-stage --id <id> --stage-id <sid> ...
fabio deployment-pipeline list-stage-items --id <id> --stage-id <sid>
fabio deployment-pipeline assign-workspace --id <id> --stage-id <sid> --workspace <ws>
fabio deployment-pipeline unassign-workspace --id <id> --stage-id <sid>
fabio deployment-pipeline list-operations --id <id>
fabio deployment-pipeline show-operation --id <id> --operation-id <oid>
fabio deployment-pipeline list-role-assignments --id <id>
fabio deployment-pipeline add-role-assignment --id <id> ...
fabio deployment-pipeline delete-role-assignment --id <id> --assignment-id <aid>
fabio deployment-pipeline deploy --id <id> --source-stage <sid> --target-stage <tid> ...
```

### job-scheduler
```
fabio job-scheduler list-instances --workspace <ws> --item-id <id>
fabio job-scheduler get-instance --workspace <ws> --item-id <id> --instance-id <iid>
fabio job-scheduler run-on-demand --workspace <ws> --item-id <id> --job-type <type> [--wait] [--timeout <secs>] [--cancel-on-timeout]
fabio job-scheduler cancel-instance --workspace <ws> --item-id <id> --instance-id <iid>
fabio job-scheduler list-schedules --workspace <ws> --item-id <id>
fabio job-scheduler get-schedule --workspace <ws> --item-id <id> --schedule-id <sid>
fabio job-scheduler create-schedule --workspace <ws> --item-id <id> ...
fabio job-scheduler update-schedule --workspace <ws> --item-id <id> --schedule-id <sid> ...
fabio job-scheduler delete-schedule --workspace <ws> --item-id <id> --schedule-id <sid>
```

### domain
```
fabio domain list
fabio domain show --id <id>
fabio domain create --name <name>
fabio domain update --id <id> --name <new-name>
fabio domain delete --id <id>
fabio domain list-workspaces --id <id>
fabio domain assign-workspaces --id <id> --content <json>
fabio domain unassign-workspaces --id <id> --content <json>
fabio domain assign-by-capacity --id <id> --content <json>
fabio domain assign-by-principal --id <id> --principal-type <type> --content <json>
```

### variable-library
```
fabio variable-library list --workspace <ws>
fabio variable-library show --workspace <ws> --id <id>
fabio variable-library create --workspace <ws> --name <name>
fabio variable-library update --workspace <ws> --id <id> --name <new-name>
fabio variable-library delete --workspace <ws> --id <id>
fabio variable-library get-definition --workspace <ws> --id <id>
fabio variable-library update-definition --workspace <ws> --id <id> --file <path>
```

### user-data-function
```
fabio user-data-function list --workspace <ws>
fabio user-data-function show --workspace <ws> --id <id>
fabio user-data-function create --workspace <ws> --name <name>
fabio user-data-function update --workspace <ws> --id <id> --name <new-name>
fabio user-data-function delete --workspace <ws> --id <id>
fabio user-data-function get-definition --workspace <ws> --id <id>
fabio user-data-function update-definition --workspace <ws> --id <id> --file <path>
```

## CI/CD Deployment

### deploy
```
fabio deploy plan --source <DIR> --workspace <ID|NAME>
  [--config <FILE>] [--env <NAME>]
  [--git-diff <REF>]
  [--item-types <T1,T2>] [--delete-orphans] [--allow-unresolved] [--force-all]
  [--exclude-regex <PATTERN>] [--include-items <Name.Type,...>]
  [--include-folders <PATH,...>] [--exclude-folders <PATH,...>]
  [--allow-delete-types <TYPE,...>]
  [--no-folders] [--no-workspace-id-replace]
  [--out <FILE>] [--parameters <FILE> --env <NAME>]

fabio deploy apply --source <DIR> --workspace <ID|NAME>
  [--plan <FILE>] [--config <FILE>] [--env <NAME>]
  [--git-diff <REF>]
  [--item-types <T1,T2>] [--delete-orphans] [--allow-unresolved]
  [--fail-fast] [--force] [--force-all] [--concurrency <N>]
  [--exclude-regex <PATTERN>] [--include-items <Name.Type,...>]
  [--include-folders <PATH,...>] [--exclude-folders <PATH,...>]
  [--allow-delete-types <TYPE,...>]
  [--no-folders] [--no-workspace-id-replace]
  [--shortcut-exclude-regex <PATTERN>]
  [--parameters <FILE> --env <NAME>] [--no-post-hooks]

fabio deploy export --workspace <ID|NAME> --dir <DIR> [--item-types <T1,T2>] [--overwrite] [--dry-run]
fabio deploy init-params --source <DIR> [--compare <DIR>] [--source-env <NAME>] [--compare-env <NAME>] [--out <FILE>]
fabio deploy validate --source <DIR>
```

**New flags (v0.22.0):**

| Flag | Description |
|------|-------------|
| `--config <FILE>` | Deploy config file (JSON or YAML) with per-environment workspace, source, parameters |
| `--git-diff <REF>` | Only deploy items changed since this git reference (branch, tag, or commit SHA) |
| `--exclude-regex <PATTERN>` | Exclude items whose display name matches this regex |
| `--include-items <Name.Type,...>` | Only include specific items (e.g., `MyNB.Notebook,SalesLH.Lakehouse`) |
| `--include-folders <PATH,...>` | Only include items in these workspace folder paths |
| `--exclude-folders <PATH,...>` | Exclude items in these workspace folder paths |
| `--allow-delete-types <TYPE,...>` | Permit deletion of protected types (Lakehouse, Warehouse, SQLDatabase, Eventhouse, KQLDatabase) |
| `--no-folders` | Skip workspace folder management |
| `--no-workspace-id-replace` | Skip automatic `00000000-...` workspace UUID replacement |
| `--shortcut-exclude-regex <PATTERN>` | Filter shortcuts during reconciliation (apply only) |

**v0.23.0 changes:**

- `--parameters <FILE>` now accepts **YAML** (`.yml`/`.yaml`) in addition to JSON — auto-detected by extension. fabric-cicd's `parameter.yml` files work without conversion.

## Configuration & Tooling

### profile
```
fabio profile save --name <name> [--workspace <ws>] [--capacity <cap>] [--default-output <format>] [--private-link-workspace <ws-id>]
fabio profile use --name <name>
fabio profile list
fabio profile show --name <name>
fabio profile delete --name <name>
```

Flags for `profile save`:
- `--workspace <ID>` — default workspace (injects `FABIO_WORKSPACE`)
- `--capacity <ID>` — default capacity (used by `workspace assign-capacity` and `gateway create`)
- `--default-output <format>` — default output format (`json`, `table`, `plain`, `csv`, `tsv`) (injects `FABIO_OUTPUT`)
- `--private-link-workspace <ID>` — routes all Fabric/OneLake API calls through private link URLs

`profile save` merges with existing profile — omitted fields preserve their current values.

### jobs (local async job ledger)
```
fabio jobs list
fabio jobs get --id <job-id>
fabio jobs prune
```

### context (agent introspection)

The `fabio context` command family provides three-layer knowledge for AI agents:

```
# Layer 1: Tool layer — what commands exist
fabio context agent    Machine-readable command schema (auth_scope, returns, flags, mutability)

# Layer 2: Domain layer — how to use Fabric
fabio context schema <TYPE>             Item definition format (22 types: Notebook, Lakehouse, etc.)
fabio context workflow <NAME>           Workflow recipes: rti-pipeline, direct-lake-report, cicd-deploy, lakehouse-etl, data-agent-setup
fabio context best-practices <TOPIC>    Topics: throttling, lro, pagination, admin-apis
fabio context examples <GROUP> <CMD>    Output shape example (20 registered)
fabio context list                      Discover all available topics

# Layer 3: Environment layer — what's in YOUR workspace
fabio context tenant --workspace <ws>     Extract graph of items and relationships from workspace(s)
  --workspace <id|name>     Workspace to scan (repeatable — pass multiple times for multi-workspace)
  --deep                    Fetch item definitions to discover embedded UUID references (slower)
  --include-connections     Include connection objects as graph edges
  --item-types <T1,T2>      Filter to specific item types (comma-separated, case-insensitive)
  --no-properties           Skip type-specific GET calls; inventory only (~3s for 20 workspaces)
  --format <graph|jsonld>   Output format: graph (default JSON) or jsonld (RDF-compatible JSON-LD)
  --output-file <path>      Write graph JSON to file instead of stdout
  --merge <existing.json>   Load existing graph file and union new nodes/edges into it (idempotent)
  --concurrency <n>         Concurrent API calls (default 8)
  --dry-run                 Preview what would be scanned without API calls
```

All knowledge layers except `tenant` are offline (no API calls, embedded in binary).

Output structure (default format):
```json
{"data": {
  "nodes": [{"id":"<uuid>","type":"Notebook","name":"...","workspaceId":"...","properties":{...}}],
  "edges": [{"source":"<uuid>","target":"<uuid>","relationship":"default_lakehouse"}],
  "workspaces": [{"id":"<uuid>","name":"..."}],
  "summary": {"totalNodes":N,"totalEdges":N,"itemTypes":{...}}
}}
```

Relationship types discovered: `child_of`, `has_endpoint`, `default_lakehouse`, `bound_to_model`, `reads_from`, `streams_to`, `queries`, `executes`, `definition_ref`, `workspace_ref`, `connected_via`

### completions
```
fabio completions bash         Print bash completion script
fabio completions zsh          Print zsh completion script
fabio completions fish         Print fish completion script
fabio completions powershell   Print PowerShell completion script
fabio completions elvish       Print elvish completion script
```

Setup examples:
```bash
# bash — add to ~/.bashrc
eval "$(fabio completions bash)"

# zsh — add to ~/.zshrc
eval "$(fabio completions zsh)"

# fish
fabio completions fish | source

# PowerShell — add to $PROFILE
fabio completions powershell | Out-String | Invoke-Expression
```

### operation (LRO management)
```
fabio operation get-state --id <operation-id>
fabio operation get-result --id <operation-id>
```

### feedback
```
fabio feedback send --message <text>
fabio feedback list
```

### upgrade
```
fabio upgrade [--check] [--target-version <x.y.z>] [--force] [--dry-run]   Check/install latest release or a specific version
```

Safety behaviors:
- SHA256 checksum verification before binary replacement
- Refuses to downgrade without `--force`
- Dev builds (`-dev` suffix) are protected from accidental overwrite; `--check` still works
- Atomic binary replacement (rename-dance on Windows for locked exe)

### rest (raw REST passthrough)
```
fabio rest call --method <GET|POST|PUT|PATCH|DELETE> --path <api-path> [--body <json|@file|@->] [--query-params <key=value>] [--poll] [--api <fabric|powerbi>]
```

### rti (Real-Time Intelligence)
```
fabio rti nl-to-kql --workspace <ws> --item-id <id> --cluster-url <kusto-uri> --database <db-name> --question <"natural language">
```

### paginated-report
```
fabio paginated-report list --workspace <ws>
fabio paginated-report update --workspace <ws> --id <id> --name <new-name>
```

### dashboard
```
fabio dashboard list --workspace <ws>
```

### datamart
```
fabio datamart list --workspace <ws>
```

## Security & Networking

### gateway
```
fabio gateway list
fabio gateway show --id <id>
fabio gateway create --name <name> --capacity <cap-id> --subscription <sub-id> --resource-group <rg> --vnet <vnet-name> --subnet <subnet-name> [--inactivity-minutes <30-1440>] [--member-count <1-9>]
fabio gateway update --id <id> [--name <new-name>] [--inactivity-minutes <n>] [--member-count <n>]
fabio gateway delete --id <id>
fabio gateway list-members --id <id>
fabio gateway update-member --id <id> --member-id <mid> ...
fabio gateway delete-member --id <id> --member-id <mid>
fabio gateway list-role-assignments --id <id>
fabio gateway add-role-assignment --id <id> --principal <pid> --principal-type <User|Group|ServicePrincipal> --role <Admin|ConnectionCreator|ConnectionCreatorWithResharing>
fabio gateway show-role-assignment --id <id> --assignment-id <aid>
fabio gateway update-role-assignment --id <id> --assignment-id <aid> --role <role>
fabio gateway delete-role-assignment --id <id> --assignment-id <aid>
fabio gateway check-status --id <id>                Check VNet gateway connectivity status
fabio gateway check-member-status --id <id> --member-id <mid>   Check individual member connectivity (on-premises)
fabio gateway restart --id <id>                     Restart a VNet gateway (LRO, requires Admin)
fabio gateway shutdown --id <id>                    Shut down a VNet gateway (LRO, requires Admin)
```

### onelake-security
```
fabio onelake-security list --workspace <ws> --item-id <id>
fabio onelake-security show --workspace <ws> --item-id <id> --role <name>
fabio onelake-security create --workspace <ws> --item-id <id> --role <json|@file>
fabio onelake-security upsert --workspace <ws> --item-id <id> --content <json>
fabio onelake-security delete --workspace <ws> --item-id <id> --role <name>
```

### managed-private-endpoint
```
fabio managed-private-endpoint list --workspace <ws>
fabio managed-private-endpoint show --workspace <ws> --id <id>
fabio managed-private-endpoint create --workspace <ws> --name <name> --resource-id <ARM-id> --group-id <blob|sqlServer|dfs|queue>
fabio managed-private-endpoint delete --workspace <ws> --id <id>
```

## Tenant Administration

### admin — Tenant Settings
```
fabio admin list-tenant-settings                               List all tenant settings (165+)
fabio admin show-tenant-setting --name <settingName>           Show a specific tenant setting
fabio admin update-tenant-setting --name <settingName> --content <json>   Enable/disable a setting
fabio admin list-capacity-tenant-setting-overrides --capacity <cap-id>    List capacity overrides
fabio admin show-capacity-tenant-setting-override --capacity <cap-id> --name <settingName>
fabio admin update-capacity-tenant-setting-override --capacity <cap-id> --name <settingName> --content <json>
fabio admin list-domain-tenant-setting-overrides --domain <dom-id>
fabio admin show-domain-tenant-setting-override --domain <dom-id> --name <settingName>
fabio admin update-domain-tenant-setting-override --domain <dom-id> --name <settingName> --content <json>
```

### admin — Workspace Management
```
fabio admin list-workspaces                                    List all tenant workspaces
fabio admin show-workspace --id <ws-id>                        Show workspace (admin view)
fabio admin list-workspace-users --id <ws-id>                  List workspace access details
fabio admin grant-admin-access --id <ws-id>                    Grant temporary admin access
fabio admin remove-admin-access --id <ws-id>                   Remove temporary admin access
fabio admin restore-workspace --id <ws-id> --capacity <cap-id> [--name <restored-name>]
fabio admin discover-git-connections                            List workspaces with git connections
fabio admin list-network-policies                              List workspace communication policies
```

### admin — Item Management
```
fabio admin list-items [--workspace <ws-id>] [--type <ItemType>]   List all items tenant-wide
fabio admin show-item --workspace <ws-id> --id <item-id>           Show item (admin view)
fabio admin list-item-users --workspace <ws-id> --id <item-id>     List item access details
fabio admin list-user-access --user-id <principal-id>              List all access for a user
```

### admin — Domain Management
```
fabio admin list-domains                                        List all domains
fabio admin show-domain --id <dom-id>                           Show domain details
fabio admin create-domain --name <name>                         Create a domain
fabio admin update-domain --id <dom-id> --name <new-name>       Update domain
fabio admin delete-domain --id <dom-id>                         Delete a domain
fabio admin list-domain-workspaces --id <dom-id>                List workspaces in domain
fabio admin assign-domain-workspaces --id <dom-id> --content '{"workspacesIds":["..."]}'
fabio admin unassign-domain-workspaces --id <dom-id> --content '{"workspacesIds":["..."]}'
fabio admin assign-domain-workspaces-by-capacities --id <dom-id> --content '{"capacitiesIds":["..."]}'
fabio admin assign-domain-workspaces-by-principals --id <dom-id> --principal-type <User|Group|ServicePrincipal> --content '{"principals":[{"id":"...","type":"User"}]}'
fabio admin unassign-all-domain-workspaces --id <dom-id>
fabio admin bulk-assign-domain-roles --id <dom-id> --content '{"type":"Contributors","principals":[{"id":"...","type":"User"}]}'
fabio admin bulk-unassign-domain-roles --id <dom-id> --content '{"type":"Contributors","principals":[{"id":"...","type":"User"}]}'
fabio admin sync-domain-roles-to-subdomains --id <dom-id> --role <Contributor|Admin>
```

### admin — Tags
```
fabio admin list-tags                                           List all governance tags
fabio admin create-tag --content '{"createTagsRequest":[{"displayName":"..."}]}'
fabio admin update-tag --id <tag-id> --content '{"displayName":"...","description":"..."}'
fabio admin delete-tag --id <tag-id>                            Delete a tag
```

### admin — Labels (Microsoft Purview)
```
fabio admin bulk-set-labels --content '{"items":[{"id":"...","type":"Report"}],"labelId":"..."}'
fabio admin bulk-remove-labels --content '{"items":[{"id":"...","type":"Report"}]}'
```

### admin — Sharing Links
```
fabio admin remove-all-sharing-links --content '{"sharingLinkType":"OrgLink"}'
fabio admin bulk-remove-sharing-links --content '{"sharingLinks":[{"itemId":"...","itemType":"Report","workspaceId":"..."}]}'
```

### admin — External Data Shares
```
fabio admin list-external-data-shares                           List all external data shares
fabio admin revoke-external-data-share --workspace <ws-id> --item-id <item-id> --share-id <share-id>
```

### admin — Workloads
```
fabio admin list-workloads                                      List available workloads
fabio admin list-workload-assignments                           List workload assignments
fabio admin create-workload-assignment --content '{"type":"Tenant|Capacity|Workspace","workloadId":"..."}'
fabio admin delete-workload-assignment --id <assignment-id>
```
