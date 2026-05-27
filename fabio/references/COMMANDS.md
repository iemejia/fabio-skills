# fabio Command Reference

Complete command reference organized by functional area.

## Core Commands

### auth
```
fabio auth login             Log in (validates Azure credentials)
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
fabio workspace set-network-policy --id <id> ...
```

### item (generic item operations)
```
fabio item list --workspace <ws> [--type <ItemType>]
fabio item show --workspace <ws> --id <id>
fabio item create --workspace <ws> --name <name> --type <type>
fabio item update --workspace <ws> --id <id> --name <new-name>
fabio item delete --workspace <ws> --id <id>
fabio item copy --workspace <ws> --id <id> --dest-workspace <dest>
fabio item move --workspace <ws> --id <id> --dest-workspace <dest>
fabio item get-definition --workspace <ws> --id <id>
fabio item update-definition --workspace <ws> --id <id> --file <path>
fabio item list-connections --workspace <ws> --id <id>
fabio item apply-tags --workspace <ws> --id <id> --tag-ids '["uuid"]'
fabio item unapply-tags --workspace <ws> --id <id> --tag-ids '["uuid"]'
fabio item bulk-export-definitions --workspace <ws> ...
fabio item bulk-import-definitions --workspace <ws> ...
fabio item bulk-move --workspace <ws> --ids '["id1","id2"]' --dest-workspace <dest>
fabio item list-external-data-shares --workspace <ws> --id <id>
fabio item create-external-data-share --workspace <ws> --id <id> ...
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
fabio lakehouse move-file --workspace <ws> --id <id> --source <glob> --dest <path>
fabio lakehouse delete-file --workspace <ws> --id <id> --path <file>
fabio lakehouse copy-table --workspace <ws> --id <id> --table <name> --dest-workspace <ws2> --dest-id <lh2>
fabio lakehouse move-table --workspace <ws> --id <id> --table <name> --dest-workspace <ws2> --dest-id <lh2>
fabio lakehouse delete-table --workspace <ws> --id <id> --table <name>
fabio lakehouse sync --workspace <ws> --id <id> --dest-workspace <ws2> --dest-id <lh2> [--delete]
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
fabio lakehouse list-livy-sessions --workspace <ws> --id <id>
fabio lakehouse get-livy-session --workspace <ws> --id <id> --session-id <sid>
```

### capacity
```
fabio capacity list
fabio capacity show --id <id>
```

### catalog
```
fabio catalog search --query <text> [--type <ItemType>]
```

## Data & Compute

### notebook
```
fabio notebook list --workspace <ws>
fabio notebook show --workspace <ws> --id <id>
fabio notebook create --workspace <ws> --name <name> [--lakehouse <lh-id>] [--source <file.py>]
fabio notebook update --workspace <ws> --id <id> --name <new-name>
fabio notebook delete --workspace <ws> --id <id>
fabio notebook get-definition --workspace <ws> --id <id>
fabio notebook update-definition --workspace <ws> --id <id> --source <file.py>
fabio notebook run --workspace <ws> --id <id> [--wait] [--timeout <secs>]
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
fabio data-pipeline create-schedule --workspace <ws> --id <id> ...
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
fabio data-agent create --workspace <ws> --name <name>
fabio data-agent update --workspace <ws> --id <id> --name <new-name>
fabio data-agent delete --workspace <ws> --id <id>
fabio data-agent query --workspace <ws> --id <id> --message <text>
fabio data-agent get-definition --workspace <ws> --id <id>
fabio data-agent update-definition --workspace <ws> --id <id> --file <path>
fabio data-agent publish --workspace <ws> --id <id>
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
fabio semantic-model refresh --workspace <ws> --id <id>
fabio semantic-model takeover --workspace <ws> --id <id>
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
fabio job-scheduler run-on-demand --workspace <ws> --item-id <id> --job-type <type>
fabio job-scheduler cancel-instance --workspace <ws> --item-id <id> --instance-id <iid>
fabio job-scheduler list-schedules --workspace <ws> --item-id <id>
fabio job-scheduler get-schedule --workspace <ws> --item-id <id> --schedule-id <sid>
fabio job-scheduler create-schedule --workspace <ws> --item-id <id> ...
fabio job-scheduler update-schedule --workspace <ws> --item-id <id> --schedule-id <sid> ...
fabio job-scheduler delete-schedule --workspace <ws> --item-id <id> --schedule-id <sid>
```

## Configuration & Tooling

### profile
```
fabio profile save --name <name> [--workspace <ws>] [--output <format>]
fabio profile use --name <name>
fabio profile list
fabio profile show --name <name>
fabio profile delete --name <name>
```

### jobs (local async job ledger)
```
fabio jobs list
fabio jobs get --id <job-id>
fabio jobs prune
```

### agent-context
```
fabio agent-context    # Machine-readable command schema for AI agents
```

### operation (LRO management)
```
fabio operation get-state --id <operation-id>
fabio operation get-result --id <operation-id>
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
```

### onelake-security
```
fabio onelake-security list --workspace <ws> --item-id <id>
fabio onelake-security show --workspace <ws> --item-id <id> --role <name>
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
