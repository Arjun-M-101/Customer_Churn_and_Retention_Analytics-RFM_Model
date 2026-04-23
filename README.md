# Customer Churn & Retention Analytics — RFM Model

<img width="1157" height="655" alt="Dashboard Preview" 
src="https://github.com/user-attachments/assets/b6553344-4638-4b49-b7f5-9fae8654da33" />

## Business Problem

A UK-based e-commerce retailer had 541,909 raw transactions but no way to
identify which customers were silently churning, which segments drove revenue,
or how much revenue was recoverable through targeted retention campaigns.

This project builds a full end-to-end analytics pipeline: from raw transactional
data → Python EDA → SQL data warehouse → Power BI RFM segmentation dashboard
with a live revenue recovery simulator.

## Pipeline Architecture

```text
Raw CSV — 541,909 rows (UK e-commerce transactions, Dec 2010 – Dec 2011)
│
▼ Python (Pandas) — EDA, cleaning, RFM aggregation
Cleaned RFM table — 397,884 transactions → 4,338 unique customers
│
▼ SQL Server (SSMS) — staging, schema hardening, Primary Key enforcement
dbo.Fact_RFM — 4,338 rows | CustomerID (PK, INT) | Recency | Frequency | Monetary
│
▼ Power BI Desktop — DAX scoring, relational model, interactive dashboard
E-Commerce Customer Analytics Dashboard
```

## Tools & Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| EDA & Cleaning | Python (Pandas, Datetime) | Data exploration, cleaning, RFM aggregation |
| Data Warehouse | SQL Server + SSMS | Staging, schema hardening, PK enforcement |
| BI & Visualisation | Power BI Desktop (DAX) | Calculated columns, measures, interactive dashboard |

## Dataset

- **Source:** Kaggle — [E-Commerce Data by carrie1](https://www.kaggle.com/datasets/carrie1/ecommerce-data) (UCI Online Retail Dataset)
- **Raw size:** 541,909 rows — individual product-level transactions
- **After cleaning:** 397,884 rows | 4,338 unique customers

---

## What I Built

### Phase 1 — Python: EDA & RFM Aggregation

**File:** `notebooks/churn_rfm_eda.ipynb`

- Loaded raw dataset with `encoding='ISO-8859-1'` (required for UK pound symbol)
- **EDA findings:** ~135,000 rows (25%) had null CustomerID — anonymous guest checkouts
- **Cleaning decisions (documented):**
  - Dropped null CustomerID rows — cannot segment customers without an identifier
  - Dropped negative Quantity rows — product returns would understate true spend history
  - Dropped zero UnitPrice rows — data entry errors, not real transactions
- Calculated `TotalSum = Quantity × UnitPrice` as the Monetary value per line item
- Set `snapshot_date = max(InvoiceDate) + 1 day` as the Recency reference point
- Aggregated 397,884 transactions into 4,338 customer-level rows using `groupby`:
  - **Recency** = days since customer's last invoice from snapshot date
  - **Frequency** = count of distinct invoices per customer
  - **Monetary** = sum of TotalSum per customer
- Resolved float-index export issue by explicitly reconstructing a clean DataFrame
  with `rfm.index.astype(int)` and `index=False` before CSV export

**Output:** `rfm_data.csv` — 4,338 rows, 4 columns (CustomerID, Recency, Frequency, Monetary)

---

### Phase 2 — SQL Server: Data Warehouse & Schema Hardening

**File:** `sql/rfm_schema_hardening.sql`

- Created dedicated `Ecommerce_Analytics` database
- Staged `rfm_data.csv` import into `dbo.Fact_RFM` (initial float types for safe landing)
- Applied schema hardening script:
  - `ALTER COLUMN CustomerID INT NOT NULL` — removed float decimals, enforced integer type
  - `ALTER COLUMN Monetary DECIMAL(18,2) NOT NULL` — financial precision, prevents rounding errors
  - `ADD CONSTRAINT PK_FactRFM_CustomerID PRIMARY KEY (CustomerID)` — enforces uniqueness, optimises Power BI joins
  - Verified zero nulls via `SELECT COUNT(*) WHERE CustomerID IS NULL`
- Final table grain: **one row per unique customer**, indexed for fast BI queries

**Final schema:**

| Column | Type | Constraint | Role |
|--------|------|-----------|------|
| CustomerID | INT | PRIMARY KEY, NOT NULL | Customer identifier |
| Recency | INT | NOT NULL | Days since last purchase |
| Frequency | INT | NOT NULL | Total invoice count |
| Monetary | DECIMAL(18,2) | NOT NULL | Total spend |

---

### Phase 3 — Power BI: RFM Engine & Interactive Dashboard

**File:** `dashboard/Ecommerce_RFM_Analytics_v1.pbix`

#### Data Model (Star Schema)

- **Fact_RFM** loaded from SQL Server (Import mode)
- **data** table loaded from raw `data.csv` — provides InvoiceDate for time-series
- **Relationship:** `data[CustomerID]` → `Fact_RFM[CustomerID]` (Many-to-One)
- **Cross-filter direction:** Both — enables full slicer cross-filtering across all visuals
- **Disabled Auto Date/Time** (File → Options → Current File → Data Load) — resolved DateTime parsing errors in Power Query
- **Power Query:** Removed error rows from InvoiceDate column, changed type to Date/Time using English (United Kingdom) locale

#### DAX Calculated Columns (on Fact_RFM)

Row-level scoring using RANKX quintile method — assigns each customer a score of 1–5 on each RFM dimension:

```dax
-- Recency Score: lower days = more recent = higher score (ASC ranking)
R_Score =
VAR R_Rank = RANKX(ALL('Fact_RFM'), 'Fact_RFM'[Recency], , ASC, Skip)
VAR Total  = COUNTROWS(ALL('Fact_RFM'))
RETURN
IF(R_Rank <= Total * 0.2, 5,
   IF(R_Rank <= Total * 0.4, 4,
      IF(R_Rank <= Total * 0.6, 3,
         IF(R_Rank <= Total * 0.8, 2, 1))))

-- Frequency Score: higher invoice count = higher score (DESC ranking)
F_Score =
VAR F_Rank = RANKX(ALL('Fact_RFM'), 'Fact_RFM'[Frequency], , DESC, Skip)
VAR Total  = COUNTROWS(ALL('Fact_RFM'))
RETURN
IF(F_Rank <= Total * 0.2, 5,
   IF(F_Rank <= Total * 0.4, 4,
      IF(F_Rank <= Total * 0.6, 3,
         IF(F_Rank <= Total * 0.8, 2, 1))))

-- Monetary Score: higher spend = higher score (DESC ranking)
M_Score =
VAR M_Rank = RANKX(ALL('Fact_RFM'), 'Fact_RFM'[Monetary], , DESC, Skip)
VAR Total  = COUNTROWS(ALL('Fact_RFM'))
RETURN
IF(M_Rank <= Total * 0.2, 5,
   IF(M_Rank <= Total * 0.4, 4,
      IF(M_Rank <= Total * 0.6, 3,
         IF(M_Rank <= Total * 0.8, 2, 1))))

-- 3-digit composite RFM code (e.g. 555 = top quintile on all three)
RFM_Cell = 'Fact_RFM'[R_Score] * 100 + 'Fact_RFM'[F_Score] * 10 + 'Fact_RFM'[M_Score]

-- Customer segment classification based on composite score
Customer_Segment =
IF('Fact_RFM'[RFM_Cell] >= 444, "Champions",
   IF('Fact_RFM'[RFM_Cell] >= 333, "Loyal Customers",
      IF('Fact_RFM'[RFM_Cell] >= 222, "At Risk",
         "Hibernating")))
```

#### DAX Measures (dynamic — respond to all filters and slicers)

```dax
Total Revenue = SUM('Fact_RFM'[Monetary])

Customer Count = DISTINCTCOUNT('Fact_RFM'[CustomerID])

-- Isolated to At-Risk segment regardless of active filters (ALL removes context)
At Risk Revenue =
CALCULATE(
    [Total Revenue],
    ALL(Fact_RFM),
    'Fact_RFM'[Customer_Segment] = "At Risk"
)

-- Revenue recovery projection: At-Risk revenue × slider value
-- Uses SELECTEDVALUE to bridge the disconnected Retention Rate parameter table
Projected Recovery =
VAR AtRiskRev = CALCULATE(
    SUM('Fact_RFM'[Monetary]),
    'Fact_RFM'[Customer_Segment] = "At Risk"
)
RETURN AtRiskRev * SELECTEDVALUE('Retention Rate'[Retention Rate])
```

#### What-If Revenue Recovery Simulator

- **Retention Rate parameter:** Numeric range 0 → 1, increment 0.05
- **Projected Recovery card:** Updates in real time as slider moves
- **Business use:** A marketing manager sets the slider to 0.10 (10% retention)
  and instantly sees the projected revenue recovery from At-Risk customers —
  enabling data-driven budget decisions for retention campaigns

#### Dashboard Visuals

| Visual | Fields | Purpose |
|--------|--------|---------|
| KPI Card | Total Revenue | Headline revenue at a glance — 8.91M total across 4,338 customers |
| KPI Card | Customer Count | Total segmented customers — 4,338 unique |
| Treemap | Category = Customer_Segment, Values = Count of CustomerID | Who are my customers and how large is each segment? |
| Bar Chart | Y = Customer_Segment, X = Total Revenue | Which segments drive the most revenue? |
| Scatter Chart | X = Recency, Y = Frequency, Size = Monetary, Legend = Customer_Segment | All three RFM dimensions in one visual — reveals behavioural patterns per segment |
| Line Chart | X = InvoiceDate, Y = Sum of Monetary | How is revenue trending over time? |
| Slicer | Customer_Segment | Interactive cross-filter — clicking any segment updates all visuals simultaneously |
| Slider + Card | Retention Rate + Projected Recovery | Revenue recovery simulator |

---

## Key Findings

- **4,338** unique customers analysed across Dec 2010 – Dec 2011
- **Champions** generate ~£5.8M of the £8.9M total revenue despite being a small % of the customer base — Pareto 80/20 pattern confirmed
- **Hibernating** segment has the largest customer count — the highest churn risk and largest win-back opportunity
- **Scatter chart insight:** Champions cluster top-left (low recency, high frequency) — they buy often and recently. Hibernating customers spread bottom-right (high recency days, low frequency) — they have stopped returning.
- At-Risk Revenue card (locked via `ALL()`) shows revenue at stake regardless of active slicer state
- Cross-filter interactivity: clicking any segment across Treemap, Bar Chart, or Scatter Chart updates all other visuals simultaneously

---

## Data Quality Decisions

| Issue | Action | Reasoning |
|-------|--------|-----------|
| Null CustomerID (~25% of rows) | Dropped | Cannot perform customer-level RFM on anonymous guest checkouts |
| Negative Quantity (product returns) | Dropped | Returns would understate a loyal customer's true spend history |
| Zero UnitPrice rows | Dropped | Data entry errors — not real revenue-generating transactions |
| Float CustomerID in Python export | Cast to INT via T-SQL after staging | Enables Primary Key enforcement and optimises Power BI joins |
| DateTime parsing errors (234,047 rows) | Removed via Power Query + disabled Auto Date/Time | Unparseable rows had no valid date — cannot be plotted on time axis |
| Blank CustomerID in data table | Filtered from visuals (Filters pane) | Guest checkout rows in raw data have no RFM segment — excluded from segmentation visuals |

---

## Technical Challenges Solved

- **Python-to-SQL type mismatch:** Pandas GroupBy outputs CustomerID as float index. Resolved by explicitly reconstructing a clean DataFrame with `.astype(int)` and `index=False` export.
- **Power BI connection error (Error 40):** Resolved by separating server name and database name into distinct fields and enabling SQL Server Browser service.
- **DateTime locale error:** UK date format (DD/MM/YYYY) misread by Power BI. Resolved using Power Query → Change Type → Using Locale (English UK).
- **Disconnected parameter table:** Retention Rate table had no relationship line to Fact_RFM. Resolved by using `SELECTEDVALUE('Retention Rate'[Retention Rate])` inside the DAX measure — a DAX bridge rather than a physical relationship.
- **Flat trend line:** Power BI Date Hierarchy was collapsing all months. Resolved by disabling Auto Date/Time globally and plotting raw InvoiceDate.
- **Scatter chart visual type conflict:** Power BI blocked CustomerID in Values field alongside Legend due to DataViewMappingError_ScatterGroupingValues. Resolved by removing the Values field — Fact_RFM grain is already one row per customer, so X/Y/Legend/Size fields alone produce correct per-customer dot plotting.

---

## Repository Structure

```text
Customer_Churn_and_Retention_Analytics-RFM_Model/
├── README.md
├── notebooks/
│   └── churn_rfm_eda.ipynb            ← Python EDA and RFM aggregation
├── sql/
│   └── rfm_schema_hardening.sql       ← Database creation + schema hardening scripts
├── dashboard/
│   └── Ecommerce_RFM_Analytics_v1.pbix ← Power BI dashboard file
└── screenshots/
    ├── dashboard_preview.jpg           ← Full dashboard (unfiltered)
    ├── champions.jpg                   ← Champions segment cross-filter
    ├── loyal_customers.jpg             ← Loyal Customers cross-filter
    ├── hibernating.jpg                 ← Hibernating cross-filter
    ├── needs_attention.jpg             ← Needs Attention cross-filter
    └── at_risk.jpg                     ← At Risk cross-filter
```

---

## How to Reproduce

1. Download dataset: [Kaggle — E-Commerce Data by carrie1](https://www.kaggle.com/datasets/carrie1/ecommerce-data)
2. Rename file to `data.csv`, place in project folder
3. Run `churn_rfm_eda.ipynb` in Jupyter — produces `rfm_data.csv`
4. In SSMS: create `Ecommerce_Analytics` database, import `rfm_data.csv` as `dbo.Fact_RFM`, then run `rfm_schema_hardening.sql`
5. Open `Ecommerce_RFM_Analytics_v1.pbix` in Power BI Desktop
6. Update data source: Home → Transform Data → Data Source Settings → change server to your local instance name

---

## Screenshots

<p align="center">
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/e2dfe43e-7002-4242-a153-8b8c91270b98" />
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/a96efcd5-7f16-45e6-9bbc-acb07c3acabe" />
</p>

<p align="center">
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/cccdbb36-4023-4a85-9291-fd4c54724d3e" />
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/836a892b-afb1-4446-a529-9dfc497ded5b" />
</p>

<p align="center">
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/5ec9fe37-9975-499a-83ca-6d226411c65f" />
  <img width="400" alt="image" src="https://github.com/user-attachments/assets/97ff48a4-2b81-4ef6-95c0-2c7ea3b5d19a" />
</p>

---

## Loom Walkthrough

[WILL BE UPDATED SOON...]

---

## About

**Tools:** Python (Pandas) | SQL Server | Power BI Desktop | Power Query | DAX

**Skills demonstrated:** EDA & Data Cleaning | Data Warehouse Design | Schema Hardening | RFM Modelling | RANKX Quintile Scoring | DAX Calculated Columns & Measures | What-If Parameter | Cross-filter Interactivity | Scatter Chart Behavioural Analysis | Tooltip Pages
