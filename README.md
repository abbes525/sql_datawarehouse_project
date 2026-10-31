# SQL Server Data Warehouse | CRM & ERP Integration

A hands-on SQL Server portfolio project that brings data from six CRM and ERP CSV files into a three-layer warehouse: **Bronze → Silver → Gold**.

 My goal was to understand how raw data becomes a reliable analytical model: ingestion, cleaning, integration, dimensional modeling, and validation.

**Author:** [Mohamed Ben Abbes](https://github.com/abbes525)

## Project objective

Prepare customer, product, and sales data for analytical queries by integrating two source systems. The final model supports questions such as sales by product category, customer, or country.

The current scope is **data preparation and warehouse modeling**. A separate EDA project, business findings, and a Power BI dashboard are not included yet.

## Architecture

- **Bronze — raw ingestion:** Six source-aligned tables loaded from CSV files using `BULK INSERT`. The loader prints row counts and elapsed times.
- **Silver — cleaning and standardization:** Six tables populated from Bronze through a stored procedure. Transformations address customer duplicates, text formatting, coded values, identifiers, dates, and sales fields. A `dwh_create_date` column records insertion time.
- **Gold — analytical model:** Three SQL views expose customer and product dimensions and a sales fact. Gold reads Silver directly; it has no separate data-loading procedure.

Bronze and Silver use **full refreshes with `TRUNCATE` and reload**. The project does not preserve successive ingestion snapshots or implement incremental loading.

## Data sources

The pipeline expects these files from the tutorial dataset:

- `source_crm/cust_info.csv` — customer information.
- `source_crm/prd_info.csv` — product information and validity dates.
- `source_crm/sales_details.csv` — sales records.
- `source_erp/CUST_AZ12.csv` — additional customer details.
- `source_erp/LOC_A101.csv` — customer locations.
- `source_erp/PX_CAT_G1V2.csv` — product classifications.


## Transformations implemented

### Customers

- Exclude records with missing customer IDs.
- Keep the latest customer record by creation date using `ROW_NUMBER()`.
- Trim first and last names.
- Map marital-status and gender codes to readable labels.
- Align ERP customer identifiers with CRM identifiers.
- Replace future birthdates with `NULL`.
- Standardize selected country codes and handle blank countries.

### Products

- Extract category identifiers and product numbers from source keys.
- Replace missing product costs with zero, following the tutorial rule.
- Translate product-line codes into descriptive labels.
- Derive product end dates with `LEAD()`.
- Expose only current product records in the Gold product dimension.

### Sales

- Convert source integer date fields to SQL dates, treating zero or incorrect-length values as missing.
- Recalculate missing, nonpositive, or inconsistent sales amounts using the implemented quantity/price rule.
- Derive invalid prices from sales and quantity where possible, using `NULLIF()` to avoid division by zero.

These are dataset-specific learning rules. Business owners would need to validate them before use with operational data; the scripts do not guarantee that every possible input anomaly is repaired.

## Gold data model

- **`gold.dim_customers`** combines CRM customer data with ERP demographics and locations. CRM is prioritized for known gender values, with ERP as a fallback.
- **`gold.dim_products`** combines current products with category, subcategory, and maintenance information.
- **`gold.fact_sales`** exposes source sales-detail rows with customer and product keys, order information, dates, sales amounts, quantities, and prices.

The intended analytical grain is a **sales-detail line**, rather than one row per order or customer. The source line identifier is not exposed, so uniqueness should be checked rather than inferred from the order number alone.

The fact view joins the dimensions through `customer_key` and `product_key`. These are logical relationships; foreign-key constraints are not enforced on the views.

Dimension keys are generated with `ROW_NUMBER()`. They are suitable for this full-refresh learning project but are not durable identifiers across changing datasets.

## Repository structure

```text
sql_datawarehouse_project/
├── datasets/                                                    
├── scripts/
│   ├── init_database.sql
│   ├── bronze/
│   │   ├── ddl_script.sql
│   │   └── proc_load_bronze.sql
│   ├── silver/
│   │   ├── ddl_silver.sql
│   │   └── proc_load_silver.sql
│   └── gold/
│       └── ddl_gold.sql
├── tests/
│   ├── quality_checks_silver.sql
│   └── quality_checks_gold.sql
├── LICENSE
└── README.md
```

## Run the project



Run these files in order:

1. `scripts/init_database.sql`
2. `scripts/bronze/ddl_script.sql`
3. `scripts/silver/ddl_silver.sql`
4. `scripts/bronze/proc_load_bronze.sql`
5. `scripts/silver/proc_load_silver.sql`

For steps 2 onward, select `DataWarehouse` as the active database. Creating a procedure defines it; it does not load the data.

### 3. Execute the loaders

```sql
USE DataWarehouse;
GO
EXEC bronze.load_bronze;
GO
EXEC silver.load_silver;
GO
```

Review the output from each loader before proceeding. The procedures print caught errors; they do not rethrow them or provide an all-or-nothing transaction across the complete load.

### 4. Create Gold views

Run `scripts/gold/ddl_gold.sql` in `DataWarehouse`. Then inspect the results:

```sql
SELECT TOP (10) * FROM gold.dim_customers;
SELECT TOP (10) * FROM gold.dim_products;
SELECT TOP (10) * FROM gold.fact_sales;
```

For subsequent refreshes, rerun Bronze followed by Silver. Recreating Gold views is only needed when their definitions change.

### 5. Run quality checks

Execute:

- [`tests/quality_checks_silver.sql`](tests/quality_checks_silver.sql)
- [`tests/quality_checks_gold.sql`](tests/quality_checks_gold.sql)

The scripts inspect duplicates or missing identifiers, text formatting, standardized values, date consistency, sales arithmetic, and fact-to-dimension connectivity. Some queries list distinct values for manual inspection; anomaly queries generally expect no rows. One date diagnostic intentionally reads Bronze.

These are manually reviewed SQL diagnostics, not an automated pass/fail test framework. No execution results or row-count benchmarks are included in this repository.

## What I learned

- How Bronze, Silver, and Gold separate ingestion, preparation, and consumption.
- How data-quality decisions affect the reliability of analysis.
- How to combine CRM and ERP data through aligned identifiers.
- How fact grain, dimensions, and surrogate keys shape a star schema.
- How stored procedures and validation queries support repeatable data preparation.

## Next steps


- Build SQL exploratory analysis on the Gold views.
- Connect the model to Power BI for reporting.
- Improve robustness with safer date conversion, explicit error propagation, and additional join-cardinality checks.

## Acknowledgments

This is a **guided learning implementation**, based on [Data with Baraa’s tutorial](https://www.youtube.com/watch?v=9GVqKuTVANE) and [original project materials](https://github.com/DataWithBaraa/sql-data-warehouse-project). Credit for the tutorial, dataset, and original project design belongs to Data with Baraa. This repository documents my implementation and learning.

## License

See [LICENSE](LICENSE).
