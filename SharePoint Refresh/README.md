# SharePoint Refresh deployment

Use this template when your audit-log CSVs land in a **SharePoint folder** (typically as scheduled drops from `scripts/get-copilot-interactions.ps1` or your own export pipeline) and you want Power BI Service to refresh them automatically — no Fabric capacity required.

## What's in this folder

| File | Purpose |
|---|---|
| `AI-Business-Value-Dashboard-Sharepoint-Refresh.pbit` | Power BI template that iterates a SharePoint folder, unions every CSV, refreshes on schedule |

## When to use this path

| Pick this path if… | Pick another path instead if… |
|---|---|
| Your audit CSVs land in a SharePoint folder (one or many) and you want them auto-unioned | You only have a single static CSV — use the csv-path PBIT at the repo root |
| You want Service-side scheduled refresh without Fabric / Premium | Volume so large that in-dataset JSON parsing hits the 1 GB or 2-hour cap → see [`Fabric/`](../Fabric/) |
| Power BI Pro workspace | You need sub-second refresh / Direct Lake → use [`Fabric/`](../Fabric/) |

## How it works

The template's M-query loops through every CSV in the configured SharePoint folder, unions them all, then runs JSON parsing / fan-out. Drop new CSVs in the folder and the dataset picks them up on the next refresh — no template change needed.

```
Export pipeline (scripts/automation/* or custom)
        ↓
SharePoint folder of audit CSVs
        ↓
PBIT (Sharepoint.Files() iteration → JSON parse → expand)
        ↓
Power BI dataset → scheduled Service refresh
```

## Quick start

### 1. Prepare the SharePoint folder

- Pick (or create) a SharePoint document library / folder where audit CSVs will land
- Ensure the account that will refresh in Service has **read access**
- Note the full URL, e.g. `https://contoso.sharepoint.com/sites/CopilotAnalytics/Shared%20Documents/AuditLogs`

### 2. Open the PBIT

- Open `AI-Business-Value-Dashboard-Sharepoint-Refresh.pbit` in Power BI Desktop
- Supply parameters:

| Parameter | Value |
|---|---|
| **Copilot Interactions File** | The SharePoint folder URL from step 1 |
| **Copilot Licensed Users** | Path or SharePoint URL to your licensed-users CSV |
| **Org Data File** | Path or SharePoint URL to your org-data CSV |
| Optional ones | Leave blank |

- Click **Load**. First refresh parses every CSV; subsequent refreshes pick up new files automatically

### 3. Publish + schedule in Service

- Publish to a Power BI workspace
- Service → dataset Settings → **Data source credentials** → sign in to SharePoint with an account that has folder access
- **Scheduled refresh** → enable, set the cadence to match your export pipeline

## Folder schema requirement

Each CSV must have these columns at minimum:

```
RecordId, CreationDate, RecordType, Operation, AuditData, AssociatedAdminUnits, AssociatedAdminUnitsNames
```

This is what `scripts/get-copilot-interactions.ps1` produces.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Access to the resource is forbidden` | Stored credentials lack SharePoint access | Service → dataset Settings → Data source credentials → re-sign in |
| Refresh succeeds but interactions table is empty | Locale-sensitive type conversion under en-GB silently dropped every row | Verify `Table.TransformColumnTypes` for `CreationDate` includes `, "en-US"` (this template's 29 04+ build already has it) |
| Refresh times out (2 h) or hits 1 GB cap | Volume too large for in-dataset JSON parsing | Switch to the [`Fabric/`](../Fabric/) deployment path |
| `Formula.Firewall: Query references other queries…` | Privacy levels mismatched | Service → dataset Settings → Data source credentials → set **Privacy: None** |

## Compared to the other paths

| | csv-path | **Sharepoint Refresh** | Fabric |
|---|---|---|---|
| Source | Single local file or URL | SharePoint folder (auto-unions all CSVs) | Fabric Lakehouse Delta table |
| Parsing happens in | Power BI dataset | Power BI dataset | Fabric (upstream) |
| Service refresh | Manual / scheduled | **Scheduled, hands-off** | Scheduled, near-instant |
| Volume ceiling | ~100K events comfortably | ~500K events comfortably (Pro) | Millions |
| Setup effort | Lowest | Low — just a SharePoint folder | One-time Lakehouse + notebook |
| Best for | Ad-hoc, one-shot | Recurring with Pro license | Large tenants, Fabric capacity |
