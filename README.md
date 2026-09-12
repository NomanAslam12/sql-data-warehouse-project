# Data Warehouse and Analytics Project

A SQL Server data warehouse built on the Medallion Architecture (Bronze, Silver, Gold), consolidating CRM and ERP data into a star schema for sales analytics. Built with SSMS and T-SQL, following Data With Baraa's Data Warehouse course as a structured way to practice the full pipeline end to end — extraction, cleansing, integration, and dimensional modeling.

---

## Data Architecture

![Data Architecture](/docs/data_architecture.drawio.png)

- **Bronze Layer**: Raw data loaded as-is from CRM and ERP source systems. Six source tables — `crm_cust_info`, `crm_prd_info`, `crm_sales_details`, `erp_cust_az12`, `erp_loc_a101`, `erp_px_cat_g1v2` — loaded via `BULK INSERT` from CSV files, with each table truncated and fully reloaded on every run (batch processing, no historization).
- **Silver Layer**: Cleansed and standardized versions of the same six tables. Includes deduplication, trimming, type casting of integer-encoded dates (`YYYYMMDD` → `DATE`), gender and country code normalization (e.g. `F`/`Female` → `Female`, `DE` → `Germany`), stripped `NAS` prefixes and hyphens from customer/location IDs, and recalculation of `sls_sales` and `sls_price` where the source values were missing or internally inconsistent (`sales != quantity * price`).
- **Gold Layer**: Three business-ready views forming a star schema — `gold.dim_customers`, `gold.dim_products`, `gold.fact_sales` — built on top of the Silver layer with no further loading, only joins and surrogate key generation via `ROW_NUMBER()`.

---

## Data Integration

![Data Integration](/docs/Integration_model.drawio.png)

CRM is the primary source for customer and product identity; ERP supplies supplementary attributes:

- `crm_sales_details` (transactional sales/orders) links to `crm_prd_info` on `prd_key` and to `crm_cust_info` on `cst_id`.
- `crm_prd_info` links to `erp_px_cat_g1v2` on the product category ID, for category/subcategory/maintenance flag.
- `crm_cust_info` links to `erp_cust_az12` (birthdate, gender) and `erp_loc_a101` (country) on customer ID, after stripping the `NAS` prefix and hyphens ERP appends to the key.

---

## Data Flow

![Data Flow](/docs/Data_flow_diagram.drawio.png)

Each source table flows Bronze → Silver unchanged in shape (cleansed, not reshaped), then converges into the three Gold objects: `crm_sales_details` becomes `fact_sales`; `crm_cust_info`, `erp_cust_az12`, and `erp_loc_a101` converge into `dim_customers`; `crm_prd_info` and `erp_px_cat_g1v2` converge into `dim_products`.

---

## Data Mart (Star Schema)

![Data Mart](/docs/data_model.drawio.png)

| Table | Type | Key columns |
|---|---|---|
| `gold.dim_customers` | Dimension | `customer_key` (PK), `customer_id`, `customer_number`, name, country, marital status, gender, birthdate |
| `gold.dim_products` | Dimension | `product_key` (PK), `product_id`, `product_number`, name, category, subcategory, maintenance, cost, product line, start date |
| `gold.fact_sales` | Fact | `order_number`, `product_key` (FK), `customer_key` (FK), order/ship/due dates, `sales_amount`, `quantity`, `price` |

`gender` in `dim_customers` prefers the CRM value and falls back to the ERP value where CRM has `n/a`. `dim_products` filters out historical product records (`prd_end_dt IS NOT NULL`), so the view holds only current products. `sales_amount = quantity * price`, enforced during the Silver-layer load rather than left to the source data.

---

## Project Requirements

### Data Warehouse (Data Engineering)

Consolidate CRM and ERP CSV exports into a single SQL Server data warehouse for analytical querying:

- Two source systems, CSV format, latest snapshot only — no historization.
- Data quality issues (invalid dates, inconsistent sales math, inconsistent gender/country coding) resolved in the Silver layer, not left for reporting to work around.
- Single integrated model across both sources, documented for both business and analytics use.

### BI: Analytics & Reporting (Data Analysis)

SQL-based analysis on top of the Gold views, covering customer behavior, product performance, and sales trends.spec.

---

## Repository Structure

```
data-warehouse-project/
│
├── datasets/                           # Raw CSV exports (CRM and ERP)
│
├── docs/                               # Documentation and diagrams
│   ├── data_architecture_drawio.png    # Bronze/Silver/Gold architecture
│   ├── data_flow_diagram_drawio.png    # Source-to-Gold data lineage
│   ├── data_model_drawio.png           # Star schema (Data Mart)
│   ├── integration_model_drawio.png    # CRM/ERP table relationships
│   ├── data_catalog.md                 # Field-level catalog of Gold objects
│   ├── naming-conventions.md           # Table/column/file naming rules
│
├── scripts/
│   ├── bronze/                         # DDL + load_bronze stored procedure
│   ├── silver/                         # DDL + load_silver stored procedure
│   ├── gold/                           # View definitions (dim/fact)
│
├── tests/                              # Data quality checks
│
├── README.md
├── LICENSE
└── requirements.txt
```

---

## Running It

1. Run the bronze DDL script to create the `DataWarehouse` database and `bronze`/`silver`/`gold` schemas.
2. Run the bronze and silver table DDL scripts.
3. Update the file paths in `bronze.load_bronze` (currently `C:\sql\dwh_project\datasets\...`) to match your local `datasets/` location.
4. Execute `EXEC bronze.load_bronze;` then `EXEC silver.load_silver;`.
5. Run the gold view scripts. Query `gold.fact_sales`, `gold.dim_customers`, `gold.dim_products` directly for reporting.

---

## What I Learned

This was my first full data warehousing build, and I used Notion to track the project stage by stage so nothing between source extraction and the Gold views got missed. A few specific takeaways:

- **Translating logic into SQL.** I already knew SQL syntax going in, but hadn't had to convert a business rule ("recalculate sales if it doesn't match quantity × price") into a working `CASE` statement before. This project is where that clicked.
- **Stored procedures as a system, not a script.** Writing `bronze.load_bronze` and `silver.load_silver` as procedures — with `TRY`/`CATCH` blocks and `PRINT` statements timing each table load — showed me how to track where a pipeline actually is while it's running, instead of just executing a script and hoping it worked.
- **Views as the integration layer.** Building the Gold layer as views instead of materialized tables made the star schema easy to change without touching the data underneath.

**Next**: keep building on SQL — window functions, query performance/execution plans, and incremental loading instead of full truncate-and-reload — and start applying the same Bronze/Silver/Gold structure to a dataset I choose myself rather than a course dataset.

---

## License

Licensed under the [MIT License](LICENSE).

## Connect

[Connect with me on LinkedIn](https://www.linkedin.com/in/muhammad-noman-aslam-30125b125/)

