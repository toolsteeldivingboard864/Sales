# Sales Analytics Project: Sample Source Data

Currency: GBP. Region: United Kingdom. Orders span 2023-01-01 to 2025-12-31. All keys validated (unique, no orphans).

## Files and suggested source split

| File | Rows | Primary key | Suggested source |
|---|---|---|---|
| fact_sales.csv | 5,022 | sales_id | Azure Storage (ADLS/Blob) |
| dim_customers.csv | 500 | customer_id | Azure Storage |
| dim_products.csv | 60 | product_id | Azure Storage |
| dim_store.csv | 12 | store_id | GitHub |
| dim_location.csv | 27 | location_id | GitHub |
| dim_date.csv | 1,461 | date_key | GitHub |
| dim_subcategories.csv | 15 | subcategory_id | GitHub |
| dim_categories.csv | 5 | category_id | GitHub |

## Join keys

- fact_sales.order_date_key -> dim_date.date_key (role-playing: ship_date_key -> dim_date.date_key)
- fact_sales.customer_id -> dim_customers.customer_id
- fact_sales.product_id -> dim_products.product_id
- fact_sales.store_id -> dim_store.store_id
- fact_sales.location_id -> dim_location.location_id
- dim_store.location_id -> dim_location.location_id
- dim_customers.location_id -> dim_location.location_id
- dim_products.subcategory_id -> dim_subcategories.subcategory_id
- dim_subcategories.category_id -> dim_categories.category_id

Fact grain: one row per order line (order_id + order_line_number).
Measures: quantity, unit_price, discount_pct, sales_amount, cost_amount (margin = sales_amount - cost_amount).

## Store and location rules

- 12 stores: ST001 is the Online Store (about 29% of sales); ST002-ST012 are physical (1 Flagship, 8 High Street, 2 Outlet).
- Physical-store sales: fact_sales.location_id = the store's location, and ship_date_key = order_date_key.
- Online sales: fact_sales.location_id = the customer's ship-to location, and ship_date_key is 1-7 days after the order date.
- dim_store.floor_area_sqft is blank for the Online Store (null handling practice for Silver).

## Built in for SCD Type 1 later

- Stable business keys (customer_id, product_id, store_id, location_id).
- updated_at on every table for incremental loads and change detection.
- In Gold, add surrogate keys (e.g. customer_sk, store_sk) and MERGE on the business key, updating changed attributes in place.
- To test: change a store's store_type, a customer's segment or a product's list_price, bump updated_at, re-run, and confirm the Gold row is overwritten, not duplicated.

## Deliberate data-quality issues for the Silver layer

- dim_customers: 10 emails in UPPERCASE, 10 first_name values with stray spaces and lowercase, about 3% null gender.
- dim_store: null floor_area_sqft for the Online Store.
- Silver tasks: trim, standardise case, handle nulls, cast types, add an ingestion timestamp.

## Suggested Fabric flow

1. GitHub CSVs and Storage files -> Data Factory pipeline (Copy activity) -> Bronze Lakehouse (raw, as-is).
2. Bronze -> Silver Lakehouse (Notebook / Dataflows Gen2: clean, type, dedupe).
3. Silver -> Gold Warehouse (T-SQL: dim_ tables with surrogate keys, fact_sales with key lookups).
4. Gold Warehouse -> Power BI semantic model (Direct Lake / Import) -> report.
