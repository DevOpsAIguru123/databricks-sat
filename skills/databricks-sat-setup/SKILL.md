---
name: databricks-sat-setup
description: Use when installing, deploying, validating, or debugging Databricks Security Analysis Tool (SAT) on Azure Databricks, including SAT dashboards, Lakeview filters, Secrets Scanner, BrickHound permissions analysis, Databricks Asset Bundles, Terraform signature issues, and empty SAT result tables.
---

# Databricks SAT Setup

## Overview

Use this skill to set up Databricks Security Analysis Tool (SAT) end to end, then prove which parts are complete. Treat "dashboard is empty" as a data-flow debugging problem: job status -> table counts -> dashboard query parameters -> published Lakeview definition.

Keep two supported setup modes distinct:

- Workspace-only mode: no Databricks account admin API access. This still supports workspace security checks, Azure subscription/resource checks, Unity Catalog checks visible from the workspace, governance summary, and secrets scanner. Account-console identity/workspace-assignment and multi-workspace BrickHound details may be unavailable or partial.
- Full account mode: Databricks account admin API access is available through an account-console profile. This supports account-level SCIM, account workspaces, workspace assignments, account metastores, and fuller BrickHound permission graph collection.

Do not treat workspace-only mode as failed merely because account API calls fail. Mark the unavailable account-level surfaces clearly, then verify the workspace-level tables and dashboard filters.

## Standard Workflow

1. Confirm workspace context:
   - `databricks auth profiles`
   - `databricks current-user me --profile <profile> -o json`
   - `databricks metastores current --profile <profile> -o json`
   - `databricks warehouses list --profile <profile> -o json`
2. Clone SAT under the project folder:
   - `git clone https://github.com/databricks-industry-solutions/security-analysis-tool.git`
3. Choose the setup mode:
   - Workspace-only mode: use the workspace profile and Azure app registration. Do not require Databricks account admin API access.
   - Full account mode: use both a workspace profile and an account-console profile.
4. For Azure, create or reuse an Azure app registration and service principal. Grant Azure Reader on the subscription. Add the app as a Databricks workspace service principal with workspace admin access for SAT checks.
5. Run the DABS installer with Python 3.11 and explicit answers rather than relying on interactive prompts when automating.
6. If bundle deploy fails with `openpgp: key expired`, rerun deploy with the local Terraform binary:
   - `DATABRICKS_TF_EXEC_PATH=/opt/homebrew/bin/terraform DATABRICKS_TF_VERSION=$(terraform version -json | jq -r .terraform_version) ...`
7. Start/deploy the BrickHound app only after the SAT bundle creates it:
   - `databricks apps start <brickhound-app-name> --profile <profile>`
   - `databricks apps deploy <brickhound-app-name> --source-code-path /Workspace/Applications/SAT/files/app/brickhound --profile <profile>`
8. Run SAT jobs sequentially. Do not start driver, secrets scanner, and permissions analysis together on the first run.

## Setup Modes

### Workspace-Only Mode

Use this when the user has workspace admin access and Azure Reader access but does not have Databricks account admin API access.

Expected working areas:

- SAT Initializer and Driver
- Governance, identity, compute, workspace, DBSQL, and Unity Catalog checks available from the workspace
- Azure resource checks from the Azure app registration
- Secrets Scanner
- Dashboard sections backed by `security_checks`, `security_best_practices`, `account_info`, `account_workspaces`, and `workspace_run_complete`

Expected limitations:

- `databricks account ...` commands may fail with `Not Found`, `PERMISSION_DENIED`, or `Unable to load OAuth Config`.
- Account-level SCIM users/groups/service principals may be unavailable.
- Account workspace assignments and account metastore APIs may be unavailable.
- BrickHound permission graph may be partial unless the permissions-analysis job can authenticate to account APIs and complete.

Validation for this mode:

```bash
databricks current-user me --profile <workspace-profile> -o json
databricks metastores current --profile <workspace-profile> -o json
databricks warehouses list --profile <workspace-profile> -o json
```

Use workspace SQL table counts to prove success. Core tables should be populated even if account API calls are not.

### Full Account Mode

Use this when the user has Databricks account admin access or can authenticate a principal with account-level permissions.

Create an account-console profile separately from the workspace profile:

```bash
databricks auth login \
  --host https://accounts.azuredatabricks.net \
  --account-id <account-id> \
  --profile <account-profile>
```

Validate account API access:

```bash
databricks account workspaces list --profile <account-profile> -o json
databricks account users list --profile <account-profile> -o json
databricks account groups list --profile <account-profile> -o json
databricks account service-principals list --profile <account-profile> -o json
```

If these fail, keep the SAT deployment in workspace-only mode until account admin access is granted or account-console OAuth is fixed.

For Azure Databricks, a workspace profile with `host=https://adb-...azuredatabricks.net` is not enough for account API validation. Account API calls should target `https://accounts.azuredatabricks.net` with `account_id`.

## Job Order

Run in this order:

1. SAT Initializer Notebook (one-time)
2. SAT Driver Notebook
3. SAT Secrets Scanner
4. SAT Permissions Analysis - Data Collection (Experimental)

Avoid concurrent first runs. Concurrent SAT jobs can collide on Delta metadata and fail with `DELTA_METADATA_CHANGED`.

## Validation Queries

Use the SQL warehouse and SDK when the CLI lacks `statement-execution`:

```python
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.sql import StatementState
import time

w = WorkspaceClient(profile="<profile>")
stmt = """
SELECT 'security_checks' AS table_name, count(*) AS rows FROM <catalog>.<schema>.security_checks
UNION ALL SELECT 'security_best_practices', count(*) FROM <catalog>.<schema>.security_best_practices
UNION ALL SELECT 'account_info', count(*) FROM <catalog>.<schema>.account_info
UNION ALL SELECT 'workspace_run_complete', count(*) FROM <catalog>.<schema>.workspace_run_complete
UNION ALL SELECT 'clusters_secret_scan_results', count(*) FROM <catalog>.<schema>.clusters_secret_scan_results
UNION ALL SELECT 'notebooks_secret_scan_results', count(*) FROM <catalog>.<schema>.notebooks_secret_scan_results
"""
r = w.statement_execution.execute_statement(warehouse_id="<warehouse-id>", statement=stmt, wait_timeout="30s")
while r.status.state in {StatementState.PENDING, StatementState.RUNNING}:
    time.sleep(1)
    r = w.statement_execution.get_statement(r.statement_id)
print(r.status.state.value, r.status.error)
print(r.result.data_array if r.result else None)
```

Core SAT dashboard needs:

- `security_checks` > 0
- `security_best_practices` > 0
- `account_info` > 0
- `workspace_run_complete` has `completed = true`

Secrets dashboard needs the Secrets Scanner to finish. A clean scan may produce sentinel rows with `secrets_found = 0`; detailed findings tables will be empty by design.

BrickHound permissions analysis needs `brickhound_vertices`, `brickhound_edges`, and `brickhound_collection_metadata`, which are created only after the permissions analysis job succeeds.

In workspace-only mode, missing BrickHound account-level rows can be expected. In full account mode, treat missing BrickHound tables as incomplete until the permissions-analysis job succeeds.

## Dashboard Debugging

If Lakeview dashboard panels are empty while tables have rows:

1. Fetch the dashboard definition:
   - `databricks lakeview get <dashboard-id> --profile <profile> -o json > /tmp/sat_lakeview.json`
2. Inspect `serialized_dashboard` for:
   - catalog/schema references such as `` `sat`.security_analysis `` that should be `<catalog>.<schema>`
   - empty `workspaceid`, `workspace_id`, or `date_param` defaults
3. Run the exact dataset SQL manually with concrete values.
4. Patch defaults, update, and publish:
   - `databricks lakeview update <dashboard-id> --json @body.json --profile <profile>`
   - `databricks lakeview publish <dashboard-id> --embed-credentials --warehouse-id <warehouse-id> --profile <profile>`

Read [dashboard-patches.md](references/dashboard-patches.md) when dashboard filters or secret tiles show "no data" despite successful jobs.

## Workspace-Specific Values Template

Fill these placeholders from the target workspace before running commands:

- Profile: `<workspace-profile>`
- Account profile: create `<account-profile>` only after account-console login succeeds
- Workspace ID: `<workspace-id>`
- Account ID: `<account-id>`
- Host: `<workspace-host>`
- Account host: `https://accounts.azuredatabricks.net`
- Catalog/schema: `<catalog>.<schema>`
- SQL warehouse: `<warehouse-id>`
- Workspace dashboard filter value: `<workspace-name> (<workspace-id>)`
- SAT dashboard ID: `<dashboard-id>`

Expected post-setup state:

- Workspace-only mode is working when the workspace profile is valid, core SAT tables are populated, dashboard filters are patched, and the permissions-analysis job has created BrickHound tables.
- Full account mode is working only after account API calls succeed from an account-console profile.
- Permissions analysis is incomplete until job `<permissions-analysis-job-id>` succeeds and `brickhound_vertices`, `brickhound_edges`, and `brickhound_collection_metadata` exist.

For any other workspace, discover values from the CLI and tables instead of reusing these.
