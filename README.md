# SQL Server Data Warehouse (Medallion Architecture)

A data warehouse built in SQL Server that combines sales data from two source systems (CRM and ERP) into one clean, analysis-ready star schema.

Data moves through three layers: **Bronze** (raw), **Silver** (cleaned) and **Gold** (star schema for reporting). All ETL is written in T-SQL stored procedures and views.

> **Credit:** This is a guided portfolio project. I followed the free Data With Baraa course for the structure and datasets, and wrote, ran and tested the scripts myself in SQL Server.

---

## Architecture

| Layer | Purpose | How it is built |
|-------|---------|-----------------|
| **Bronze** | Raw data, loaded as-is | `bronze.load_bronze` stored procedure (`TRUNCATE` + `BULK INSERT` from CSV) |
| **Silver** | Cleaned and standardized data | `silver.load_silver` stored procedure (transformations in T-SQL) |
| **Gold** | Business-ready star schema | Three views: `dim_customers`, `dim_products`, `fact_sales` |

## Source Data

| Source | File | Rows | Content |
|--------|------|------|---------|
| CRM | `cust_info.csv` | 18,494 | Customer details |
| CRM | `prd_info.csv` | 397 | Product details with start and end dates |
| CRM | `sales_details.csv` | 60,398 | Sales transactions |
| ERP | `CUST_AZ12.csv` | 18,484 | Birth date and gender |
| ERP | `LOC_A101.csv` | 18,484 | Customer country |
| ERP | `PX_CAT_G1V2.csv` | 37 | Product categories |
| | **Total** | **116,294** | |

---

## Bronze Layer

- Creates 6 tables, one per source file, with the same columns as the CSVs.
- `bronze.load_bronze` truncates each table, then loads the CSV with `BULK INSERT`.
- Prints load time per table and for the whole batch, and wraps the load in `TRY...CATCH`.

## Silver Layer: Data Cleaning

`silver.load_silver` truncates and reloads all 6 Silver tables, adding a `dwh_create_date` audit column.

| Table | What was cleaned |
|-------|------------------|
| `crm_cust_info` | Removed rows with no customer ID; kept the latest record per customer using `ROW_NUMBER()` (removes duplicates); trimmed names; mapped `M/S` to Married/Single and `M/F` to Male/Female (`n/a` if unknown) |
| `crm_prd_info` | Split the product key into `cat_id` and `prd_key`; replaced missing cost with 0; mapped product line codes (R, M, S, T) to Road, Mountain, Other Sales, Touring; rebuilt `prd_end_dt` with `LEAD()` as one day before the next version's start date |
| `crm_sales_details` | Converted integer dates (YYYYMMDD) to `DATE` and set invalid ones to NULL; recalculated `sales` when missing, zero, negative or not equal to quantity x price; derived `price` when missing or invalid |
| `erp_cust_az12` | Removed the `NAS` prefix from customer IDs; set future birth dates to NULL; standardized gender values |
| `erp_loc_a101` | Removed hyphens from customer IDs so keys match the CRM; standardized countries (`DE` to Germany, `US` and `USA` to United States); blank values set to `n/a` |
| `erp_px_cat_g1v2` | Loaded without changes |

### Data quality issues found in the raw files

- 4 customer rows with no ID, and 5 customer IDs that appear more than once
- About 4,578 customers with missing gender in the CRM
- All 200 product end dates earlier than their start dates (wrong in the source)
- 19 invalid order dates
- About 35 sales rows with missing, zero, negative or inconsistent sales or price values
- 16 birth dates in the future
- Mixed country spellings (USA, US, United States, DE, Germany) and blanks
- CRM and ERP customer keys in different formats (`NAS` prefix, hyphens)

## Gold Layer: Star Schema

| View | Type | Description |
|------|------|-------------|
| `gold.dim_customers` | Dimension | Joins CRM customers with ERP birth date, gender and country. Gender comes from the CRM, with the ERP as a fallback when the CRM value is `n/a`. Adds a surrogate `customer_key`. |
| `gold.dim_products` | Dimension | Current products only (`prd_end_dt IS NULL`), joined with ERP category, subcategory and maintenance flag. Adds a surrogate `product_key`. |
| `gold.fact_sales` | Fact | Sales transactions linked to the customer and product keys, with order, shipping and due dates, sales amount, quantity and price. |

```
dim_customers --+
                +-- fact_sales
dim_products ---+
```

## Results

| Output | Rows |
|--------|------|
| Raw source rows | 116,294 |
| `gold.dim_customers` | 18,484 |
| `gold.dim_products` | 295 |
| `gold.fact_sales` | 60,398 |

- Every sales row matches a customer and a current product, so the fact table has no orphan keys.
- 4,554 customers had missing gender in the CRM and received it from the ERP source.

---

## Repository Structure

```
sql_data_warehouse_project/
├── datasets/    # Raw CRM and ERP CSV files
├── scripts/
│   ├── bronze/  # Create tables + load procedure
│   ├── silver/  # Create tables + load procedure
│   └── gold/    # Dimension and fact views
├── tests/       # Test and quality check scripts
├── LICENSE
└── README.md
```

## How to Run

1. Install SQL Server and SSMS, then create a database with the schemas `bronze`, `silver` and `gold`.
2. Run the Bronze table script, then the Silver table script.
3. Open `bronze.load_bronze` and **change the `BULK INSERT` file paths** (currently `C:\sql\dwh_project\datasets\...`) to the folder where you saved the CSVs.
4. Run the procedures:
   ```sql
   EXEC bronze.load_bronze;
   EXEC silver.load_silver;
   ```
5. Run the Gold views script, then query the views:
   ```sql
   SELECT COUNT(*) FROM gold.fact_sales;
   ```

## Known Limitations

- Full reload only: tables are truncated and reloaded each run (no incremental loading or history tracking).
- File paths in the Bronze procedure are hardcoded to one machine.
- `CATCH` blocks print the error but do not stop or re-raise it, so a failed load can look finished.
- 15 customers still have gender `n/a` after combining CRM and ERP data.
- Birth dates in the past that look unrealistic (before 1924) are not cleaned; only future dates are set to NULL.

## Tools

SQL Server, T-SQL (stored procedures, views, window functions), SSMS, Git

## License

MIT License
