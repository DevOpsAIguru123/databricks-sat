# Databricks SAT Documentation And Skill

This repository contains reusable documentation and a Codex skill for installing, operating, and troubleshooting Databricks Security Analysis Tool (SAT).

## Contents

| Path | Purpose |
| --- | --- |
| `docs/SAT_ARCHITECTURE_AND_ANALYSIS.md` | Explains what SAT is, benefits, architecture, execution flow, report categories, permissions analysis, and setup modes. |
| `docs/SAT_SETUP_RUNBOOK_TWO_APPROACHES.md` | Step-by-step runbook for workspace-only setup and full account API setup. |
| `skills/databricks-sat-setup/` | Codex skill for SAT setup, validation, dashboard troubleshooting, Secrets Scanner, and BrickHound permissions analysis. |

## Notes

- The documents use placeholders such as `<workspace-profile>`, `<account-id>`, `<catalog>.<schema>`, and `<dashboard-id>`.
- Do not commit real client secrets, tokens, workspace IDs, account IDs, tenant IDs, subscription IDs, or user emails.
- Workspace-only SAT can run without Databricks account admin API access.
- Full account mode requires an account-console profile and account-level permissions.
