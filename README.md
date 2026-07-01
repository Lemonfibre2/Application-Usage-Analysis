# Application Usage Dashboard

A Databricks-based data pipeline and AI/BI dashboard for tracking user adoption, lab activity, and interaction patterns across RNBI applications.

---

## Project Overview

This project processes audit log data from multiple labs and applications, transforms it through a Medallion architecture (Bronze → Silver → Gold), and visualizes it through a Databricks AI/BI Dashboard. It supports multi-month data ingestion, multi-lab user deduplication, and interactive filtering by application, month, lab, and user.

---

## Architecture

```
CSV Files (audit_*.csv)
        ↓
  Bronze Layer        — Raw ingestion, no transformations
        ↓
  Silver Layer        — Cleaning, timestamp standardization, null filtering
        ↓
  Gold Layer          — Lab enrichment, JOIN with user/LDAP data
        ↓
  Dashboard (.json)   — Databricks AI/BI Dashboard with 16+ datasets
```

---

## Repository Structure

```
Application_Analysis/
│
├── 01_Raw_bronze.py                          # Bronze ingestion notebook
├── 02_silver_cleaning.py                     # Silver cleaning notebook
├── 03_gold_aggregation.py                    # Gold aggregation notebook
└── Application_Usage_Dashboard_Final.json    # Databricks AI/BI Dashboard
```

---

## Pipeline Details

### 01_Raw_bronze.py — Bronze Layer
- Reads all `audit_*.csv` files from the Databricks Volume using a wildcard path
- Reads with `inferSchema=False` to ensure all columns are treated as strings
- Writes raw data as-is to `workspace.bronze.audit_bronze` (Delta table)
- Also ingests `user_details.csv` and `ladp.csv` for lab/user mapping

**Important:** CSV files must be pre-cleaned to remove duplicate header rows before uploading to the volume. Use the provided cleaning script.

### 02_silver_cleaning.py — Silver Layer
- Standardizes `start_datetime` to `yyyy/MM/dd HH:mm:ss` format using:
  ```sql
  COALESCE(
    TRY_TO_TIMESTAMP(..., 'yyyy-MM-dd HH:mm:ss'),
    TRY_TO_TIMESTAMP(..., 'yyyy/MM/dd HH:mm:ss')
  )
  ```
- Normalizes `user` to UPPERCASE and trims whitespace
- Filters out null users and null data providers
- Filters out duplicate header rows and malformed rows
- Produces `workspace.silver.audit_defects_silver`

### 03_gold_aggregation.py — Gold Layer
- Joins audit data with user details and LDAP data to enrich with lab, region, and display name
- Uses `ROW_NUMBER()` deduplication on JOIN sources to prevent row multiplication
- Produces:
  - `workspace.gold.user_lab_application_gold` — main table used by dashboard
  - `workspace.gold.defects_user_lab_gold` — defects-specific table
  - `workspace.gold.user_lab_application_summary_gold` — aggregated summary

---

## Dashboard Details

The Databricks AI/BI Dashboard (`Application_Usage_Dashboard_Final.json`) contains **16 datasets** and the following visualizations:

### KPI Widgets
| KPI | Description |
|---|---|
| Total Usage | Deduplicated event count |
| Active Users | Distinct user count |
| Active Labs | Distinct lab count (excludes Multi-lab-access and Unknown) |
| Usage Hours | Total hours spent (sum of event_duration_seconds / 3600) |
| Avg Session (min) | Average session duration in minutes |
| Last Activity | Most recent event timestamp |

### Charts
- **Daily Application Usage** — Line chart of total usage per day
- **Monthly Application Usage** — Bar/pie chart of usage per month
- **Top Labs by Application Usage** — Stacked bar chart by lab and application
- **Monthly Usage by Lab** — Stacked bar chart comparing months side by side
- **Top Users by Application Usage** — Bar chart of top 20 users
- **Lab Usage by Application (Heatmap)** — Heatmap of lab vs application usage

### Tables
- **Activity Details** — Row-level event log
- **Unknown Lab Users** — Users with no lab recorded
- **Multi-Lab User Access** — Users with access to more than one lab and their labs

### Global Filters
- Application Name
- Month
- Unknown Lab Users
- Lab (applies to Multi-Lab User Access table only)

---

## Key Design Decisions

### Multi-lab User Deduplication
Users who access an application from multiple labs generate duplicate rows in the source data (one per lab per event). The dashboard uses a two-step CTE to:
1. Detect events with `COUNT(DISTINCT lab_name) > 1` per `(user, application_name, start_datetime)`
2. Label those users as `Multi-lab-access` and keep only one row per event using `ROW_NUMBER()`

### Application Display Name Logic
```sql
CASE
    WHEN application_name = 'RNBI_MANUFACTURING_V2' THEN 'Optical Analysis'
    WHEN application_name = 'RNBI_MANUFACTURING_RxSR_RxMFGOA' THEN 'Servicerate Analysis/Order Analysis'
    WHEN application_name LIKE 'RNBI_MANUFAC_%' THEN REGEXP_REPLACE(application_name, '^RNBI_MANUFAC_', '')
    WHEN application_name LIKE 'RNBI%' THEN REGEXP_REPLACE(application_name, '^RNBI_?', '')
    ELSE application_name
END
```

### Applications Tracked
```
RNBI_AEL_BCP_ANALYSIS, RNBI_BUDGET_FORECAST_V2, RNBI_LEADTIME,
RNBI_MANUFAC_DEFECTS_ANALYSIS, RNBI_MANUFAC_EMFitting,
RNBI_MANUFAC_HC_BASKET_TRACKING, RNBI_MANUFACTURING_RxSR_RxMFGOA,
RNBI_MANUFACTURING_V2, DEFECTS_ANALYSIS, HC_BASKET_TRACKING,
EMFitting, PCKG, fcsd_dashboard
```

---

## Setup Instructions

### Prerequisites
- Databricks workspace with Unity Catalog enabled
- Databricks Volume at `/Volumes/workspace/bronze/raw/defects_project/csv/`
- Schemas created: `workspace.bronze`, `workspace.silver`, `workspace.gold`

### 1. Prepare CSV Files
Before uploading, remove duplicate header rows from audit CSV files:
```python
df = pd.read_csv('audit_xxx.csv', dtype=str)
split_idx = df.index[df['Data Provider'] == 'User'].tolist()
if split_idx:
    df = df.iloc[:split_idx[0]]
df.to_csv('audit_xxx_clean.csv', index=False)
```

### 2. Upload Files to Databricks Volume
Upload the following to `/Volumes/workspace/bronze/raw/defects_project/csv/`:
- `audit_janfeb.csv`
- `audit_mar.csv`
- `audit_apr.csv`
- `audit_may.csv` (and future months)
- `user_details.csv`
- `ladp.csv`

### 3. Run the Pipeline
Run notebooks in order:
```
01_Raw_bronze.py → 02_silver_cleaning.py → 03_gold_aggregation.py
```

### 4. Import the Dashboard
1. In Databricks, go to **Dashboards**
2. Click **Import** and upload `Application_Usage_Dashboard_Final.json`
3. The dashboard will auto-connect to the gold table

---

## Adding New Months

1. Export the new month's audit CSV from the source system
2. Clean it using the duplicate header removal script
3. Upload to `/Volumes/workspace/bronze/raw/defects_project/csv/`
4. Rerun the pipeline — the wildcard `audit_*.csv` picks it up automatically
5. No dashboard changes needed

---

## Known Issues & Notes

- `start_datetime` is stored as a string in Silver/Gold (`yyyy/MM/dd HH:mm:ss`) — use `TO_TIMESTAMP(start_datetime, 'yyyy/MM/dd HH:mm:ss')` for date arithmetic
- Source CSV files from some months contain a duplicate header row mid-file causing column shifting — always pre-clean before uploading
- `Multi-lab-access` and `Unknown` are excluded from the Active Labs KPI count
- The `usage_by_object` and `usage_by_event_type` datasets query a different source table (`defects_user_lab_gold`) and are not affected by application filters
