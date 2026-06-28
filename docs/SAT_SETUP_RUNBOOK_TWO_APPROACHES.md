# Databricks SAT Setup Runbook: Workspace-Only And Full Account API

This runbook documents two supported setup approaches for Databricks Security Analysis Tool (SAT) on Azure Databricks:

- Approach A: Workspace-only setup, without Databricks account admin API.
- Approach B: Full account API setup, with Databricks account admin API.

Use the workspace-only approach when the goal is to analyze one workspace and account API access is not available. Use the full account approach when account-level identities, workspace assignments, account metastores, and multi-workspace graph analysis are required.

## Shared Prerequisites

Install and verify these local tools:

```bash
databricks --version
az version
terraform version
python3 --version
uv --version
git --version
```

Fill these values for the target workspace:

| Item | Value |
| --- | --- |
| Workspace profile | `<workspace-profile>` |
| Workspace ID | `<workspace-id>` |
| Workspace host | `<workspace-host>` |
| Databricks account ID | `<account-id>` |
| Azure subscription ID | `<subscription-id>` |
| Azure tenant ID | `<tenant-id>` |
| Catalog/schema | `<catalog>.<schema>` |
| SQL warehouse ID | `<warehouse-id>` |
| SAT dashboard ID | `<dashboard-id>` |
| BrickHound app | `<brickhound-app-name>` |

## Approach A: Workspace-Only Setup

### When To Use

Use this when:

- You are a Databricks workspace admin.
- You can create/use an Azure app registration with Reader on the Azure subscription.
- You do not have Databricks account admin API access.
- You only need the current workspace analyzed.

This mode still supports:

- SAT Initializer and Driver.
- Lakeview dashboard.
- Governance summary.
- Workspace-level checks.
- Azure resource checks.
- Unity Catalog checks visible from the workspace.
- Secrets Scanner.
- BrickHound single-workspace permissions graph.

It does not fully support:

- Account-level users/groups/service principals.
- Account workspace assignments.
- Account-level metastore assignment graph.
- Multi-workspace BrickHound collection.
- Full cross-workspace attack path analysis.

### What This One-Workspace Setup Explicitly Adds

This approach is scoped to one Databricks workspace. It still creates resources in several places: the local project folder, Azure, the Databricks workspace, Unity Catalog, and Databricks Apps.

| Area | Explicitly added or changed | Key files or resource names | Purpose |
| --- | --- | --- | --- |
| Local project | SAT source repository under `<local-sat-project>/security-analysis-tool` | `README.md`, `dabs/main.py`, `dabs/setup.sh`, `dabs/requirements.txt` | Holds SAT source, DABS config, notebooks, dashboard template, and local setup files. |
| Local tooling | Python 3.11 virtual environment and SAT Python dependencies | `.venv/`, `dabs/requirements.txt` | Runs the DABS installer and helper scripts. |
| Local Databricks config | Workspace CLI profile `<workspace-profile>` | `~/.databrickscfg` profile entry | Lets local commands deploy and validate SAT against the target workspace. |
| Azure Entra ID | Azure app registration `<azure-app-registration-name>` | Azure app registration object, client ID, client secret | Provides an application identity for Azure-side SAT checks. |
| Azure subscription IAM | Reader role assignment for the Azure app registration at subscription or resource-group scope | Azure role assignment on `/subscriptions/<subscription-id>` or narrower scope | Allows read-only inspection of Azure Databricks resource/network/encryption metadata. |
| Databricks workspace identity | Workspace service principal mapped to the Azure application | Workspace service principal object for `<azure-app-client-id>` | Lets the same app identity exist inside the Databricks workspace. |
| Databricks workspace permissions | Workspace service principal added to workspace admins, or granted equivalent required permissions | Workspace `admins` group membership or equivalent permissions | Allows SAT to collect workspace configuration and object metadata. |
| Databricks workspace files | `/Workspace/Applications/SAT/files` | `notebooks/security_analysis_initializer`, `notebooks/security_analysis_driver`, `notebooks/security_analysis_secrets_scanner`, `notebooks/permission_analysis_data_collection` | Stores deployed SAT notebooks, dashboard assets, app source, and bundle artifacts. |
| Unity Catalog | Schema `<catalog>.<schema>` | Tables listed in the durable output section below | Stores SAT result tables and BrickHound graph tables. |
| SQL warehouse | Existing or selected SQL warehouse `<warehouse-id>` | Warehouse selected during DABS install | Serves Lakeview dashboard queries and validation SQL. |
| Lakeview dashboard | SAT dashboard `<dashboard-id>` | Local template `dashboards/SAT_Dashboard_definition.json`; deployed dashboard `Security Analysis Tool [SAT]` | Main report for security checks, governance, secrets, and posture. |
| Databricks Jobs | Initializer, Driver, Secrets Scanner, and Permissions Analysis jobs | `<initializer-job-id>`, `<driver-job-id>`, `<secrets-scanner-job-id>`, `<permissions-analysis-job-id>` | Orchestrates setup, scheduled analysis, secret scanning, and permissions graph collection. |
| Job schedules | Driver, Secrets Scanner, and BrickHound schedules | Quartz expressions configured during DABS install | Keeps SAT data refreshed after setup. |
| Databricks App | BrickHound app `<brickhound-app-name>` | Local files `app/brickhound/databricks.yml`, `app/brickhound/app.yaml`, `app/brickhound/app.py`, `app/brickhound/requirements.txt`; deployed path `/Workspace/Applications/SAT/files/app/brickhound` | Optional UI for exploring permissions graph data. |
| Secret scope entries | SAT/BrickHound configuration secrets such as client ID, client secret, tenant ID, account ID, warehouse ID, and schema settings | Secret keys such as `client-id`, `client-secret`, `tenant-id`, `account-console-id`, `sql-warehouse-id`, `analysis_schema_name` | Supplies credentials and configuration to notebooks at runtime. |

Important local source files for the one-workspace setup:

| File | Role |
| --- | --- |
| `dabs/main.py` | Interactive/automated SAT DABS installer entrypoint. |
| `dabs/sat/config.py` | Installer configuration model and generated settings handling. |
| `dabs/sat/utils.py` | Helper functions for profiles, catalogs, warehouses, and deployment inputs. |
| `dashboards/SAT_Dashboard_definition.json` | Lakeview dashboard template imported during setup. |
| `notebooks/security_analysis_initializer.py` | One-time initializer notebook for SAT metadata, config, tables, and dashboard import. |
| `notebooks/security_analysis_driver.py` | Main SAT collection and security-check evaluation notebook. |
| `notebooks/security_analysis_secrets_scanner.py` | Secrets scanner job notebook. |
| `notebooks/permission_analysis_data_collection.py` | BrickHound permissions graph collection notebook. |
| `notebooks/Setup/1. list_account_workspaces_to_conf_file.py` | Generates workspace inventory/config input. In workspace-only mode this can resolve to the current workspace. |
| `notebooks/Setup/3. test_connections.py` | Tests configured workspace/API connectivity. |
| `notebooks/Setup/4. enable_workspaces_for_sat.py` | Enables selected workspaces for analysis. |
| `notebooks/Setup/5. import_dashboard_template_lakeview.py` | Imports and configures the Lakeview dashboard template. |
| `notebooks/Setup/7. update_sat_check_configuration.py` | Updates enabled check definitions/configuration. |
| `notebooks/Setup/8. update_workspace_configuration.py` | Updates workspace-specific SAT configuration. |
| `notebooks/Utils/accounts_bootstrap.py` | Account/workspace bootstrap helpers. Full account mode uses more of this surface. |
| `notebooks/Utils/workspace_bootstrap.py` | Workspace API bootstrap helpers. |
| `notebooks/Utils/common.py` | Shared table creation, writing, and utility functions. |
| `notebooks/Utils/sat_checks_config.py` | SAT check definitions/configuration. |
| `notebooks/Includes/workspace_analysis.py` | Workspace check evaluation logic. |
| `notebooks/Includes/workspace_settings.py` | Workspace settings collection helpers. |
| `notebooks/Includes/workspace_stats.py` | Workspace stats collection helpers. |
| `notebooks/brickhound/00_config.py` | BrickHound table names and analysis config. |
| `notebooks/brickhound/01_principal_resource_analysis.py` | BrickHound principal/resource analysis notebook. |
| `notebooks/brickhound/02_escalation_paths.py` | BrickHound escalation path analysis notebook. |
| `notebooks/brickhound/03_impersonation_analysis.py` | BrickHound impersonation analysis notebook. |
| `notebooks/brickhound/04_advanced_reports.py` | BrickHound advanced report notebook. |
| `configs/trufflehog_detectors.yaml` | TruffleHog detector configuration used by secret scanning. |

The one-workspace setup does not grant Databricks account admin. It also does not create a working account-console profile unless you perform the full account API setup later.

After this setup completes, the expected durable outputs are:

- SAT dashboard is published and points to the selected SQL warehouse.
- SAT jobs exist and can be run manually or by schedule.
- Core tables exist in `<catalog>.<schema>`, including `security_checks`, `security_best_practices`, `account_info`, `account_workspaces`, and `workspace_run_complete`.
- Secret scan tables exist after the Secrets Scanner runs.
- BrickHound graph tables exist after the Permissions Analysis job succeeds in `single-workspace` mode.

### Step 1: Authenticate To Workspace

```bash
databricks auth login \
  --host <workspace-host> \
  --profile <workspace-profile>
```

Validate:

```bash
databricks auth profiles
databricks current-user me --profile <workspace-profile> -o json
databricks metastores current --profile <workspace-profile> -o json
databricks warehouses list --profile <workspace-profile> -o json
```

Expected current user for this setup:

- The authenticated principal is a workspace admin.

### Step 2: Clone SAT

```bash
cd <local-sat-project>
git clone https://github.com/databricks-industry-solutions/security-analysis-tool.git
cd security-analysis-tool
```

### Step 3: Create Azure App Registration

Create an Azure app registration and service principal for SAT, then grant Azure Reader on the subscription:

```bash
az ad app create --display-name <azure-app-registration-name>
az ad app credential reset --id <app-id>
az role assignment create \
  --assignee <app-id> \
  --role Reader \
  --scope /subscriptions/<subscription-id>
```

Record the app registration values securely:

- Display name: `<azure-app-registration-name>`
- Application/client ID: `<azure-app-client-id>`

Do not store client secrets in documentation. Put them only in Databricks secrets or local secure tooling during setup.

### Step 4: Create Databricks Workspace Service Principal

Create a matching Databricks workspace service principal and add it to workspace admins. This is needed for SAT workspace checks.

Record the Databricks workspace identity values:

- Databricks service principal ID: `<databricks-service-principal-id>`
- Workspace admins group ID: `<workspace-admins-group-id>`

Validation:

```bash
databricks service-principals list --profile <workspace-profile> -o json
databricks groups list --profile <workspace-profile> -o json
```

### Step 5: Prepare Python Environment

```bash
cd <local-sat-project>/security-analysis-tool
uv venv --python 3.11 --allow-existing .venv
uv pip install --python .venv/bin/python -r dabs/requirements.txt
```

### Step 6: Run SAT DABS Installer

Run the installer from `dabs/main.py`, using these choices:

| Prompt | Value |
| --- | --- |
| Databricks profile | `<workspace-profile>` |
| Databricks account ID | `<account-id>` |
| Catalog | `<catalog>` |
| Schema | `<schema>` |
| Serverless | `True` |
| SQL warehouse | `test`, ID `<warehouse-id>` |
| Time zone | `<timezone>` |
| Driver schedule | `0 0 8 ? * Mon,Wed,Fri` |
| Secrets scanner schedule | `0 0 10 ? * *` |
| BrickHound schedule | `0 0 2 ? * *` |
| Azure tenant/subscription/client | Use the Azure app registration values. |

If bundle deployment fails with Terraform signature/key errors, use the local Terraform binary:

```bash
export DATABRICKS_TF_EXEC_PATH=/opt/homebrew/bin/terraform
export DATABRICKS_TF_VERSION="$(terraform version -json | jq -r .terraform_version)"
```

Then rerun the bundle deployment or installer deploy step.

### Step 7: Start And Deploy BrickHound App

```bash
databricks apps start <brickhound-app-name> \
  --profile <workspace-profile> \
  --timeout 20m

databricks apps deploy <brickhound-app-name> \
  --source-code-path /Workspace/Applications/SAT/files/app/brickhound \
  --profile <workspace-profile>
```

App URL:

```text
<brickhound-app-url>
```

### Step 8: Run Jobs In Order

Run these sequentially:

```bash
databricks jobs run-now <initializer-job-id> --profile <workspace-profile>
databricks jobs run-now <driver-job-id> --profile <workspace-profile>
databricks jobs run-now <secrets-scanner-job-id> --profile <workspace-profile>
databricks jobs run-now <permissions-analysis-job-id> --profile <workspace-profile>
```

Do not run Driver and Permissions Analysis concurrently on the first run. First-run concurrent writes can collide on Delta metadata.

### Step 9: Validate Workspace-Only Setup

Validate core SAT data:

```sql
SELECT 'security_checks' AS table_name, count(*) AS rows
FROM <catalog>.<schema>.security_checks
UNION ALL
SELECT 'security_best_practices', count(*)
FROM <catalog>.<schema>.security_best_practices
UNION ALL
SELECT 'account_info', count(*)
FROM <catalog>.<schema>.account_info
UNION ALL
SELECT 'account_workspaces', count(*)
FROM <catalog>.<schema>.account_workspaces
UNION ALL
SELECT 'workspace_run_complete', count(*)
FROM <catalog>.<schema>.workspace_run_complete
WHERE completed = true;
```

Healthy workspace-only result expectations:

| Table | Expected result |
| --- | --- |
| `security_checks` | Nonzero rows after the driver job. |
| `security_best_practices` | Nonzero check definitions. |
| `account_info` | Rows for collected workspace/account facts. |
| `account_workspaces` | At least the analyzed workspace. |
| `workspace_run_complete` | At least one completed run. |

Validate secrets scanner:

```sql
SELECT 'clusters_secret_scan_results' AS table_name, count(*) AS rows
FROM <catalog>.<schema>.clusters_secret_scan_results
UNION ALL
SELECT 'notebooks_secret_scan_results', count(*)
FROM <catalog>.<schema>.notebooks_secret_scan_results;
```

Clean scans can have rows with `secrets_found = 0`. Detail tables can be empty if no secrets were found.

Validate BrickHound:

```sql
SELECT 'vertices' AS table_name, count(*) AS rows
FROM <catalog>.<schema>.brickhound_vertices
UNION ALL
SELECT 'edges', count(*)
FROM <catalog>.<schema>.brickhound_edges
UNION ALL
SELECT 'metadata', count(*)
FROM <catalog>.<schema>.brickhound_collection_metadata;
```

BrickHound data is present when these queries return nonzero vertex and edge counts. The latest collection mode should be `single-workspace` for this approach.

### Step 10: Patch Or Publish Dashboard If Needed

Fetch dashboard:

```bash
databricks lakeview get <dashboard-id> \
  --profile <workspace-profile> \
  -o json > /tmp/sat_lakeview.json
```

Common dashboard fixes:

- Set `workspaceid` default to `<workspace-name> (<workspace-id>)`.
- Set `workspace_id` default to `<workspace-id>`.
- Set `date_param` to the latest completed run date.
- Publish with embedded credentials and warehouse ID `<warehouse-id>`.

Publish:

```bash
databricks lakeview publish <dashboard-id> \
  --profile <workspace-profile> \
  --embed-credentials \
  --warehouse-id <warehouse-id>
```

## Approach B: Full Account API Setup

### When To Use

Use this when:

- You have Databricks account admin access.
- You need all workspaces in the Databricks account.
- You need account-level SCIM users/groups/service principals.
- You need workspace assignment data.
- You need account metastore and assignment information.
- You need fuller BrickHound cross-workspace graph analysis.

### Additional Prerequisites

You need one of these:

- A Databricks user with account admin permissions.
- A service principal with account-level permissions and valid account OAuth configuration.

For Azure Databricks, account APIs use:

```text
https://accounts.azuredatabricks.net
```

They do not use the workspace URL.

### Step 1: Create Account Console Profile

```bash
databricks auth login \
  --host https://accounts.azuredatabricks.net \
  --account-id <account-id> \
  --profile <account-profile>
```

Validate:

```bash
databricks account workspaces list --profile <account-profile> -o json
databricks account users list --profile <account-profile> -o json
databricks account groups list --profile <account-profile> -o json
databricks account service-principals list --profile <account-profile> -o json
databricks account metastores list --profile <account-profile> -o json
```

If these fail, stay in workspace-only mode until account permissions or account OAuth are fixed.

### Step 2: Configure Account Admin Service Principal For BrickHound

For full multi-workspace BrickHound, the permissions notebook expects account-capable service principal secrets in the SAT/BrickHound secret scope. On Azure, it also needs tenant ID for MSAL token acquisition.

Expected secret values:

| Secret key | Purpose |
| --- | --- |
| `account-console-id` | Databricks account ID. |
| `client-id` | Service principal client/application ID. |
| `client-secret` | Service principal client secret. |
| `tenant-id` | Azure tenant ID for MSAL. |

The service principal must be able to call account APIs and must have access to target workspaces.

### Step 3: Use Classic Compute For Multi-Workspace Collection

The SAT permissions notebook explicitly treats serverless compute as current-workspace only. For multi-workspace collection:

- Use classic compute.
- Ensure the service principal credentials are available in the expected secret scope.
- Ensure workspace and account IP allow lists permit calls from the compute environment.

If running on serverless, BrickHound restricts collection to the current workspace even when credentials exist.

### Step 4: Run Same SAT Jobs

Run the same job order:

1. Initializer.
2. Driver.
3. Secrets scanner.
4. Permissions Analysis.

For full account mode, confirm the Permissions Analysis job metadata shows `multi-workspace` instead of `single-workspace`.

Validation:

```sql
SELECT run_id, collection_timestamp, vertices_count, edges_count,
       collection_mode, workspaces_collected, workspaces_failed
FROM <catalog>.<schema>.brickhound_collection_metadata
ORDER BY collection_timestamp DESC
LIMIT 5;
```

Expected full-account signs:

- `collection_mode = 'multi-workspace'`.
- `workspaces_collected` contains all intended workspaces.
- Vertex types include account-level identities such as `AccountUser`, `AccountGroup`, or `AccountServicePrincipal`.
- Edges include workspace-access and account-level relationships where available.

## Troubleshooting

### Account API Fails From Workspace Profile

Symptom:

```text
databricks account workspaces list --profile <workspace-profile>
Error: Not Found
```

Cause:

- The profile points at a workspace host, not the account console host.
- Or the user/principal is not account admin.
- Or Azure account OAuth is not configured/available.

Fix:

- Create a separate account profile with host `https://accounts.azuredatabricks.net`.
- Validate account APIs with that account profile.

### SDK AccountClient Fails With Unable To Load OAuth Config

Symptom:

```text
400 Bad Request: Unable to load OAuth Config
```

Cause:

- Account-level OAuth path is not usable for the current profile/principal.
- Account console profile or service principal setup is missing.

Fix:

- Use account-console login with account ID.
- Use an account-admin user or account-capable service principal.
- Keep workspace-only mode if account API is not required.

### Governance Summary Is Empty

Likely causes:

- Dashboard `workspaceid` default is empty or wrong.
- Dashboard `date_param` default is empty or wrong.
- Dashboard was updated but not republished.

Fix:

- Set workspace filter to `<workspace-name> (<workspace-id>)`.
- Set date filter to the latest `workspace_run_complete.chk_date`.
- Republish the Lakeview dashboard with embedded credentials.

### Secret Tiles Show No Data

If the scan found no secrets, detail views can be empty by design. Summary counters should show zero.

Recommended clean-scan dashboard behavior:

- Secret workspace dropdown includes scanned workspaces even when `secrets_found = 0`.
- Selected-workspace secret metadata counts only workspaces with actual findings.

### BrickHound Tables Missing

Cause:

- Permissions Analysis job has not completed.
- The only previous run was canceled.

Fix:

```bash
databricks jobs run-now <permissions-analysis-job-id> --profile <workspace-profile>
```

Then validate:

```sql
SELECT table_name
FROM <catalog>.information_schema.tables
WHERE table_schema = 'security_analysis'
  AND table_name LIKE 'brickhound%'
ORDER BY table_name;
```

After the job succeeds, these tables should exist:

- `brickhound_vertices`
- `brickhound_edges`
- `brickhound_collection_metadata`

### Delta Metadata Collision During First Run

Symptom:

```text
MetadataChangedException [DELTA_METADATA_CHANGED]
```

Cause:

- Multiple SAT jobs write or alter Delta tables concurrently during first setup.

Fix:

- Run Initializer, Driver, Secrets Scanner, and Permissions Analysis sequentially.
- Rerun the failed job after the first writer completes.

## Completion Checklist

Workspace-only setup is complete when:

- Workspace auth is valid.
- Initializer succeeds.
- Driver succeeds.
- Secrets Scanner succeeds.
- Permissions Analysis succeeds in `single-workspace` mode.
- Core SAT tables have rows.
- BrickHound tables have rows.
- Lakeview dashboard is published and filters point to the correct workspace/date.

Full account setup is complete when all workspace-only checks pass and:

- Account console profile validates.
- Account workspaces/users/groups/service-principals APIs succeed.
- Permissions Analysis runs in `multi-workspace` mode.
- Account-level identities and workspace assignments appear in BrickHound graph data.
