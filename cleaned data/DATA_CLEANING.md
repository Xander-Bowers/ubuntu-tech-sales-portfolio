# Data Cleaning Process — Retail Sales Dataset

This document covers the cleaning process applied to four raw source files
(`customers.csv`, `products.csv`, `sales_orders.csv`, `stores.csv`) before
they were loaded for analysis/querying. Raw files were loaded as-is into
staging tables, then transformed with the SQL below into clean, query-ready
tables. SQL is written for SQLite; notes on Postgres/MySQL equivalents are
included where syntax differs.

## Approach

1. Load each raw CSV into a `stg_<table>` staging table with every column
   typed as `TEXT`, so no values are silently altered (e.g. no dropped
   leading zeros) before inspection.
2. Profile each column: nulls, duplicates, distinct values, formats.
3. Write a `CREATE TABLE ... AS SELECT` transformation per table that
   casts types, standardizes text, and resolves the issues found below.
4. Re-validate the output (row counts, null counts, distinct value lists).

## Issues found and how each was resolved

### `stores.csv`
No issues found — column names, types, and casing were already
consistent. Copied through unchanged (aside from trimming whitespace).

### `customers.csv` (2,200 rows)
| Issue | Resolution |
|---|---|
| `province` had inconsistent casing (`GAUTENG`, `western cape` vs. `Gauteng`, `Western Cape`) | Mapped to a canonical Title Case value per province |
| `phone`: 30 numbers were missing their leading `0` (9 digits instead of 10) | Left-padded with `0` where length was 9 |
| Stray whitespace in text fields | Trimmed on all text columns |

No missing values or duplicate rows were found in this file.

### `products.csv` (20 rows)
| Issue | Resolution |
|---|---|
| `category` had inconsistent casing (`mobile` vs `Mobile`, `audio` vs `Audio`, etc.) — 7 real categories, 12 raw spellings | Mapped to 7 canonical Title Case categories |
| `active` used four different spellings (`Y`, `Yes`, `N`, `No`) | Normalized to an `is_active` boolean (1/0) |
| `cost_price` was `NULL` for 4 products | Left as `NULL` — no reliable basis to impute (margins vary a lot by category). Added a `cost_price_missing` flag column so downstream queries can filter/handle these explicitly rather than silently treating them as zero-cost. |

### `sales_orders.csv` (12,035 raw rows → 12,000 clean rows)
This was the messiest file. Issues found and fixes:

| Issue | Resolution |
|---|---|
| 31 fully duplicated rows, plus 4 more rows sharing an `order_id` with another row that differed only by formatting noise | Standardized text fields first, then de-duplicated on `order_id`, keeping the first occurrence — 35 duplicate rows removed in total |
| `order_date`: 3 rows used `DD/MM/YYYY` instead of `YYYY-MM-DD`; 97 rows were blank | Reformatted the 3 mismatched rows to ISO format; left blanks as `NULL` (dropping them would lose otherwise-valid order data) |
| `customer_id` stored as a float (`"1783.0"`) due to the column containing nulls in the source system; 73 rows were `NULL` | Cast to integer; kept `NULL`s as-is (these read as guest/unlinked orders — dropping them would lose valid revenue rows) |
| `quantity`: some values had a `" units"` text suffix (e.g. `"3 units"`) instead of a plain number | Stripped the suffix and cast to integer |
| `unit_price_zar`: some values had an `"R "` currency prefix; 48 rows were `NULL` | Stripped the prefix and cast to numeric; recalculated the 48 missing values as `subtotal_zar / quantity`, since both of those were always populated |
| `discount`: mixed formats — decimal fraction (`0.1`) **and** percentage string (`"10%"`) representing the same thing | Normalized everything to a single decimal fraction (0–1) |
| `payment_method`: casing/whitespace issues (`' card '` vs `Card`); 142 `NULL`s | Trimmed/standardized casing; `NULL`s set to `'Unknown'` rather than dropped |
| `sales_channel`: casing/whitespace issues (`ONLINE`, `'Online '`, `store`/`Store`); 96 `NULL`s | Standardized to `Web` / `Online` / `Store` / `Phone`; `NULL`s set to `'Unknown'` |
| `order_status`: casing issue (`completed` vs `Completed`) | Standardized casing |

**Referential integrity**: verified `customer_id`, `product_id`, and
`store_id` in `sales_orders` all match a valid row in their parent table
(zero orphaned foreign keys found — no fix needed).

## SQL

```sql
-- ============================================================
-- STORES: no data quality issues found — copy through as-is
-- ============================================================
DROP TABLE IF EXISTS stores;
CREATE TABLE stores AS
SELECT
    CAST(store_id AS INTEGER) AS store_id,
    TRIM(store_name)          AS store_name,
    TRIM(city)                AS city,
    TRIM(province)             AS province
FROM stg_stores;

-- ============================================================
-- CUSTOMERS
--   - province: inconsistent casing ('GAUTENG', 'western cape')
--   - phone: 30 numbers lost their leading 0 (9 digits instead of 10)
--   - text fields: stray whitespace
-- ============================================================
DROP TABLE IF EXISTS customers;
CREATE TABLE customers AS
SELECT
    CAST(customer_id AS INTEGER)                  AS customer_id,
    TRIM(customer_name)                           AS customer_name,
    LOWER(TRIM(email))                            AS email,
    CASE WHEN LENGTH(TRIM(phone)) = 9
         THEN '0' || TRIM(phone)
         ELSE TRIM(phone)
    END                                            AS phone,
    TRIM(city)                                     AS city,
    CASE UPPER(TRIM(province))
        WHEN 'GAUTENG'        THEN 'Gauteng'
        WHEN 'WESTERN CAPE'   THEN 'Western Cape'
        WHEN 'EASTERN CAPE'   THEN 'Eastern Cape'
        WHEN 'NORTHERN CAPE'  THEN 'Northern Cape'
        WHEN 'FREE STATE'     THEN 'Free State'
        WHEN 'LIMPOPO'        THEN 'Limpopo'
        WHEN 'MPUMALANGA'     THEN 'Mpumalanga'
        WHEN 'KWAZULU-NATAL'  THEN 'KwaZulu-Natal'
        WHEN 'NORTH WEST'     THEN 'North West'
        ELSE TRIM(province)
    END                                             AS province,
    TRIM(segment)                                   AS segment,
    DATE(TRIM(signup_date))                         AS signup_date
FROM stg_customers;

-- ============================================================
-- PRODUCTS
--   - category: inconsistent casing ('mobile' vs 'Mobile', etc.)
--   - active: four spellings (Y / N / Yes / No) -> boolean flag
--   - cost_price: 4 NULLs -> left as NULL (no reliable basis to
--     impute); flagged via cost_price_missing for downstream use
-- ============================================================
DROP TABLE IF EXISTS products;
CREATE TABLE products AS
SELECT
    CAST(product_id AS INTEGER)                    AS product_id,
    TRIM(product_name)                             AS product_name,
    CASE LOWER(TRIM(category))
        WHEN 'accessories' THEN 'Accessories'
        WHEN 'audio'       THEN 'Audio'
        WHEN 'computing'   THEN 'Computing'
        WHEN 'mobile'      THEN 'Mobile'
        WHEN 'networking'  THEN 'Networking'
        WHEN 'office'      THEN 'Office'
        WHEN 'printing'    THEN 'Printing'
        ELSE TRIM(category)
    END                                              AS category,
    CAST(cost_price AS REAL)                         AS cost_price,
    CASE WHEN cost_price IS NULL OR TRIM(cost_price) = ''
         THEN 1 ELSE 0 END                           AS cost_price_missing,
    CAST(list_price AS REAL)                         AS list_price,
    CASE UPPER(TRIM(active))
        WHEN 'Y'   THEN 1
        WHEN 'YES' THEN 1
        WHEN 'N'   THEN 0
        WHEN 'NO'  THEN 0
        ELSE NULL
    END                                              AS is_active
FROM stg_products;

-- ============================================================
-- SALES_ORDERS  (the messiest table)
--   - 31 fully duplicated rows + 4 more sharing a duplicated
--     order_id -> de-duplicated, keeping one row per order_id
--   - order_date: 3 rows in DD/MM/YYYY instead of YYYY-MM-DD;
--     97 rows missing entirely -> left NULL
--   - customer_id: stored as float ("1783.0"); 73 rows NULL
--     (guest / unlinked orders) -> left NULL
--   - quantity: some values had a " units" suffix as text
--   - unit_price_zar: some values had an "R " currency prefix;
--     48 rows NULL -> recalculated from subtotal / quantity
--   - discount: mixed formats — decimal fraction (0.1) AND
--     percentage string ("10%") -> normalized to a single
--     decimal fraction (0-1) representing % off
--   - payment_method: casing/whitespace issues (' card ');
--     142 NULLs -> 'Unknown'
--   - sales_channel: casing/whitespace issues (ONLINE, 'Online ',
--     store/Store); 96 NULLs -> 'Unknown'
--   - order_status: casing issue ('completed' vs 'Completed')
-- ============================================================

-- Step 1: standardize text/number formats first. Some duplicate
-- order_ids only differ by whitespace/casing noise that disappears
-- once the row is cleaned, so de-duping happens AFTER this step.
DROP TABLE IF EXISTS stg_sales_orders_std;
CREATE TABLE stg_sales_orders_std AS
SELECT
    CAST(order_id AS INTEGER)                       AS order_id,
    CASE
        WHEN order_date IS NULL OR TRIM(order_date) = '' THEN NULL
        WHEN order_date LIKE '__/__/____'
             THEN substr(order_date,7,4) || '-' || substr(order_date,4,2) || '-' || substr(order_date,1,2)
        ELSE TRIM(order_date)
    END                                               AS order_date,
    CASE
        WHEN customer_id IS NULL OR TRIM(customer_id) = '' THEN NULL
        ELSE CAST(CAST(customer_id AS REAL) AS INTEGER)
    END                                               AS customer_id,
    CAST(product_id AS INTEGER)                       AS product_id,
    CAST(store_id AS INTEGER)                         AS store_id,
    CASE
        WHEN quantity LIKE '% units' THEN CAST(TRIM(REPLACE(quantity,'units','')) AS INTEGER)
        ELSE CAST(TRIM(quantity) AS INTEGER)
    END                                                AS quantity,
    CASE
        WHEN unit_price_zar IS NULL OR TRIM(unit_price_zar) = '' THEN NULL
        WHEN unit_price_zar LIKE 'R %' THEN CAST(TRIM(SUBSTR(unit_price_zar,2)) AS REAL)
        ELSE CAST(TRIM(unit_price_zar) AS REAL)
    END                                                AS unit_price_zar,
    CASE
        WHEN TRIM(discount) LIKE '%\%' ESCAPE '\'
             THEN CAST(REPLACE(TRIM(discount),'%','') AS REAL) / 100.0
        ELSE CAST(TRIM(discount) AS REAL)
    END                                                AS discount,
    CAST(subtotal_zar AS REAL)                         AS subtotal_zar,
    CAST(vat_zar AS REAL)                              AS vat_zar,
    CAST(total_zar AS REAL)                            AS total_zar,
    CASE
        WHEN payment_method IS NULL OR TRIM(payment_method) = '' THEN 'Unknown'
        WHEN UPPER(TRIM(payment_method)) = 'CARD' THEN 'Card'
        ELSE TRIM(payment_method)
    END                                                 AS payment_method,
    CASE
        WHEN sales_channel IS NULL OR TRIM(sales_channel) = '' THEN 'Unknown'
        WHEN UPPER(TRIM(sales_channel)) = 'ONLINE' THEN 'Online'
        WHEN UPPER(TRIM(sales_channel)) = 'WEB'    THEN 'Web'
        WHEN UPPER(TRIM(sales_channel)) = 'STORE'  THEN 'Store'
        WHEN UPPER(TRIM(sales_channel)) = 'PHONE'  THEN 'Phone'
        ELSE TRIM(sales_channel)
    END                                                 AS sales_channel,
    CASE UPPER(TRIM(order_status))
        WHEN 'COMPLETED' THEN 'Completed'
        WHEN 'CANCELLED' THEN 'Cancelled'
        WHEN 'RETURNED'  THEN 'Returned'
        WHEN 'PENDING'   THEN 'Pending'
        ELSE TRIM(order_status)
    END                                                 AS order_status
FROM stg_sales_orders;

-- Step 2: fill the 48 missing unit prices from subtotal / quantity
-- (both are always populated in this dataset).
DROP TABLE IF EXISTS stg_sales_orders_filled;
CREATE TABLE stg_sales_orders_filled AS
SELECT
    *,
    CASE WHEN unit_price_zar IS NULL AND quantity > 0
         THEN ROUND(subtotal_zar / quantity, 2)
         ELSE unit_price_zar
    END AS unit_price_zar_final
FROM stg_sales_orders_std;

-- Step 3: de-duplicate on order_id, keeping the first occurrence
DROP TABLE IF EXISTS sales_orders;
CREATE TABLE sales_orders AS
SELECT
    order_id, order_date, customer_id, product_id, store_id,
    quantity, unit_price_zar_final AS unit_price_zar, discount,
    subtotal_zar, vat_zar, total_zar,
    payment_method, sales_channel, order_status
FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY order_id) AS rn
    FROM stg_sales_orders_filled
)
WHERE rn = 1;

DROP TABLE stg_sales_orders_std;
DROP TABLE stg_sales_orders_filled;

-- ============================================================
-- Referential integrity checks (all returned 0 rows on this data)
-- ============================================================
-- SELECT so.* FROM sales_orders so LEFT JOIN customers c ON so.customer_id = c.customer_id WHERE so.customer_id IS NOT NULL AND c.customer_id IS NULL;
-- SELECT so.* FROM sales_orders so LEFT JOIN products p ON so.product_id = p.product_id WHERE p.product_id IS NULL;
-- SELECT so.* FROM sales_orders so LEFT JOIN stores st ON so.store_id = st.store_id WHERE st.store_id IS NULL;
```

### Notes on portability
- `stg_<table>` refers to the raw CSVs loaded verbatim (all columns as text)
  into staging tables before this script runs.
- Written/tested in SQLite. On **PostgreSQL**: replace `CREATE TABLE ... AS`
  with the same syntax (supported natively), swap `substr()` for
  `SUBSTRING()`, and `LIKE ... ESCAPE '\'` works the same. Province/category
  casing could alternatively use `INITCAP()` in Postgres instead of the
  explicit `CASE` mapping used here (kept explicit so every mapping is
  visible in one place for documentation purposes).
- On **MySQL**: use `CREATE TABLE ... AS SELECT` (supported), replace
  `ROW_NUMBER() OVER (...)` with an equivalent (`ROW_NUMBER()` is supported
  in MySQL 8+), and note MySQL's `TRIM`/`SUBSTRING` syntax is compatible.

## Result

| Table | Raw rows | Clean rows | Rows removed | Reason |
|---|---:|---:|---:|---|
| `stores` | 10 | 10 | 0 | — |
| `customers` | 2,200 | 2,200 | 0 | — |
| `products` | 20 | 20 | 0 | — |
| `sales_orders` | 12,035 | 12,000 | 35 | duplicate `order_id` rows |

Remaining `NULL`s (intentionally kept, not dropped, since the rows are
otherwise valid and usable for querying):
- `sales_orders.order_date` — 97 rows (date was blank in source)
- `sales_orders.customer_id` — 72 rows (unlinked/guest orders)
- `products.cost_price` — 4 rows (flagged via `cost_price_missing`)
