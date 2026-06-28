# SAT Dashboard Patches

Use this reference only after SAT jobs have succeeded but Lakeview dashboard panels still show no data.

## Patch Workspace And Date Defaults

Set every `workspaceid` parameter to the exact value from:

```sql
SELECT concat(workspace_name, ' (', workspace_id, ')') AS workspace_param
FROM <catalog>.<schema>.account_workspaces
WHERE analysis_enabled = true;
```

Set every `date_param` parameter to a completed run date from:

```sql
SELECT chk_date
FROM <catalog>.<schema>.workspace_run_complete
WHERE completed = true
ORDER BY chk_date DESC, run_id DESC
LIMIT 1;
```

Patch with Python:

```python
import json
from pathlib import Path

obj = json.loads(Path("/tmp/sat_lakeview.json").read_text())
d = json.loads(obj["serialized_dashboard"])
for ds in d.get("datasets", []):
    for p in ds.get("parameters", []) or []:
        if p.get("keyword") == "workspaceid":
            p["defaultSelection"] = {"values": {"dataType": "STRING", "values": [{"value": "<workspace-name> (<workspace-id>)"}]}}
        if p.get("keyword") == "workspace_id":
            p["defaultSelection"] = {"values": {"dataType": "STRING", "values": [{"value": "<workspace-id>"}]}}
        if p.get("keyword") == "date_param":
            p["defaultSelection"] = {"values": {"dataType": "DATE", "values": [{"value": "<yyyy-mm-dd>"}]}}

body = {
    "display_name": obj["display_name"],
    "warehouse_id": obj["warehouse_id"],
    "serialized_dashboard": json.dumps(d),
}
Path("/tmp/sat_lakeview_update_body.json").write_text(json.dumps(body))
```

Then:

```bash
databricks lakeview update <dashboard-id> --profile <profile> --json @/tmp/sat_lakeview_update_body.json
databricks lakeview publish <dashboard-id> --profile <profile> --embed-credentials --warehouse-id <warehouse-id>
```

## Governance Summary

If Governance Summary is empty, run the backing dataset query with concrete filters. Expected healthy values can include Governance rows such as High/Medium/Low counts. Common issue: empty `date_param` default on summary datasets.

## Secrets Tiles

Secrets Scanner can succeed and find no secrets. In that case:

- global counters should show `0`, not no data
- selected-workspace counters should show `0`, not no data
- detail tables can remain empty because there are no findings to list

If the workspace dropdown is empty after a clean scan, patch the `Secret Scanner Workspaces` dataset so it does not filter to `secrets_found > 0`. It should list scanned workspaces even when findings are zero.

Also check selected-workspace metadata: `workspaces_with_secrets` should count only workspaces with notebook or cluster findings, not all scanned workspaces.
