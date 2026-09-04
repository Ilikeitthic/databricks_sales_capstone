# Sales Capstone - Databricks ETL Pipeline

A production-grade Databricks Asset Bundle (DAB) that processes Indian retail sales data through a **Bronze → Silver → Gold** medallion architecture, with data quality gates, a CI/CD pipeline, and SQL dashboard queries.

---

## Project Structure

```
databricks_sales_capstone/
│
├── databricks.yml                     # Bundle definition
│
├── resources/
│   └── sales_job.yml                  # Databricks Job: sales_etl_job
│
├── src/
│   ├── 00_data_quality_report.ipynb   # Stage 2 - Flag bad records
│   ├── 01_bronze_ingestion.ipynb      # Stage 3 - Raw → Bronze Delta
│   ├── 02_silver_cleaning.ipynb       # Stage 4 - Bronze → Silver Delta
│   └── 03_gold_analytics.ipynb        # Stage 5 - Silver → Gold summary
│
├── .github/
│   └── workflows/
│       └── deploy.yml                 # CI/CD: validate → deploy → run job
│
├── DashBoardQueries/
│   ├── 01_monthly_net_sales.dbquery.ipynb   # Chart 1 - Monthly net sales trend
│   ├── 02_sales_by_state.dbquery.ipynb      # Chart 2 - Net sales by state
│   ├── 03_sales_by_category.dbquery.ipynb   # Chart 3 - Net sales by category
│   ├── 04_orders_by_month.dbquery.ipynb     # Chart 4 - Orders by month
│   └── 05_overall_kpis.dbquery.ipynb        # Chart 5 - Overall KPI summary
│
└── README.md
```

---

## Schema

| Column | Type | Description |
|---|---|---|
| order_id | string | Unique order identifier |
| order_date | date | Date of the order |
| customer_id | string | Customer identifier |
| customer_name | string | Customer name |
| city | string | Customer city |
| state | string | Customer state |
| product_id | string | Product identifier |
| product_name | string | Product name |
| category | string | Product category |
| quantity | int | Units purchased |
| unit_price | double | Unit price (₹) |
| discount_pct | double | Discount percentage |
| gross_amount | double | Pre-discount total |
| discount_amount | double | Discount value |
| net_amount | double | Final sale amount |
| payment_method | string | Payment method |
| order_status | string | Completed / Cancelled / Returned |

---

## Pipeline Stages

### Stage 1 - Source Data
Upload `sales_source_1500.csv` to the Unity Catalog volume path:
`/Volumes/workspace/default/sales_data/sales_source_1500.csv`

### Stage 2 - Data Quality (`00_data_quality_report.ipynb`)
Applies quality flags before ingestion:

| Flag | Rule |
|---|---|
| `VALID` | All checks pass |
| `INVALID_CUSTOMER` | `customer_id` IS NULL or empty |
| `INVALID_QUANTITY` | `quantity <= 0` |
| `INVALID_AMOUNT` | `net_amount < 0` |
| `INVALID_PRODUCT` | `product_id` is NULL or `'UNKNOWN'` |

### Stage 3 - Bronze Layer (`capstone_bronze_sales`)
Notebook `01_bronze_ingestion.ipynb` reads the flagged staging table and writes all rows - with an `_ingested_at` audit column - to the Bronze Delta table.

### Stage 4 - Silver Layer (`capstone_silver_sales`)
Notebook `02_silver_cleaning.ipynb`:
- Casts all columns to correct types
- Trims string whitespace
- Fills safe null defaults
- Removes `INVALID` records
- Adds `year`, `month`, `month_name` derived columns

### Stage 5 - Gold Layer (`capstone_gold_sales_summary`)
Notebook `03_gold_analytics.ipynb` aggregates by `year / month / state / category`:

| Metric | Formula |
|---|---|
| `total_orders` | COUNT DISTINCT order_id |
| `units_sold` | SUM quantity |
| `gross_sales` | SUM gross_amount |
| `total_discount` | SUM discount_amount |
| `net_sales` | SUM net_amount |
| `avg_order_value` | SUM(net_amount) / COUNT DISTINCT order_id |
| `last_updated` | Pipeline run timestamp |

Table is partitioned by `year` and Z-ORDERed by `month`, `state`, `category`.

---

## Databricks Job

Job name: **`sales_etl_job`**

Task execution order:

```
data_quality_report → bronze_ingestion → silver_cleaning → gold_analytics
```

---

## CI/CD (GitHub Actions)

| Trigger | Action |
|---|---|
| Pull Request to `main` | Bundle validate only |
| Push to `main` | Validate → Deploy dev → Run job |
| `workflow_dispatch` | Validate → Deploy dev → Deploy prod |

### Required GitHub Secrets

| Secret | Value |
|---|---|
| `DATABRICKS_HOST` | `https://dbc-e490342b-3bd9.cloud.databricks.com` |
| `DATABRICKS_CLIENT_ID` | Service principal client ID |
| `DATABRICKS_CLIENT_SECRET` | Service principal client secret |

### Required Bundle Variables

| Variable | Value |
|---|---|
| `databricks_host` | `https://dbc-e490342b-3bd9.cloud.databricks.com` (set as default in `databricks.yml`) |
| `notification_email` | Email address for job failure alerts |

---

## Stage 6 - Dashboard Queries

Databricks SQL notebooks are in the [`DashBoardQueries/`](DashBoardQueries/) folder.
Open each `.dbquery.ipynb` file directly in your Databricks workspace to run and pin to a dashboard.

| Notebook | Chart | Visualisation |
|---|---|---|
| [`01_monthly_net_sales.dbquery.ipynb`](DashBoardQueries/01_monthly_net_sales.dbquery.ipynb) | Monthly Net Sales | Line chart |
| [`02_sales_by_state.dbquery.ipynb`](DashBoardQueries/02_sales_by_state.dbquery.ipynb) | Sales by State | Bar chart |
| [`03_sales_by_category.dbquery.ipynb`](DashBoardQueries/03_sales_by_category.dbquery.ipynb) | Sales by Category | Bar / Pie chart |
| [`04_orders_by_month.dbquery.ipynb`](DashBoardQueries/04_orders_by_month.dbquery.ipynb) | Orders by Month | Bar chart |
| [`05_overall_kpis.dbquery.ipynb`](DashBoardQueries/05_overall_kpis.dbquery.ipynb) | Overall KPIs | Counter tiles |
| [`06_dq_trend.dbquery.ipynb`](DashBoardQueries/06_dq_trend.dbquery.ipynb) | Data Quality Trend | Stacked bar chart |

---

## Deployment

All deployments are triggered via **GitHub Actions** - no local CLI required.

| Action | How to trigger |
|---|---|
| Deploy + run job | Push or merge to `main` |
| Manual deploy + run job | Trigger **Run workflow** manually from the Actions tab |

Ensure the following secrets are set in **GitHub → Settings → Secrets and variables → Actions**:
- `DATABRICKS_HOST`
- `DATABRICKS_CLIENT_ID`
- `DATABRICKS_CLIENT_SECRET`
