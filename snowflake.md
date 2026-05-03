# Snowflake Essentials for IICS Developers
### Topics: Data Load | Data Cleaning | Data Validation | Data Analysis

> **Scope:** This guide covers only what an IICS developer truly needs when Snowflake is the final/target table — from loading data through IICS, cleaning and validating it in Snowflake, and doing basic data analysis via Snowsight or SQL.

---

## 1. Table Types — Permanent, Transient & Temporary

### What It Is
Snowflake has three types of tables that behave differently in terms of storage cost, data retention, and use cases.

| Type | Fail-Safe | Time Travel | Cost | Use Case |
|---|---|---|---|---|
| **Permanent** | 7 days | 0–90 days | Higher | Final/target tables |
| **Transient** | None | 0–1 day | Lower | Staging/intermediate tables |
| **Temporary** | None | 0–1 day | Lowest | Session-scoped scratch tables |

### How to Use It
```sql
-- Final target table (permanent)
CREATE TABLE sales_db.public.orders (
    order_id     NUMBER,
    customer_id  NUMBER,
    order_date   DATE,
    amount       DECIMAL(10,2),
    load_dt      TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Staging/intermediate table (transient — cheaper, no fail-safe)
CREATE TRANSIENT TABLE sales_db.public.orders_stage (
    order_id     NUMBER,
    customer_id  NUMBER,
    order_date   DATE,
    amount       DECIMAL(10,2)
);

-- Session-scoped scratch table (temporary — disappears when session ends)
CREATE TEMPORARY TABLE temp_dedup AS
SELECT DISTINCT * FROM orders_stage;
```

### How to Answer in Interview
> **Q: What is the difference between Permanent, Transient, and Temporary tables in Snowflake?**

*"Permanent tables are standard tables with full Time Travel (up to 90 days) and Fail-safe (7 days), making them ideal for final target tables where data recovery is critical. Transient tables skip Fail-safe and have limited Time Travel, which makes them cheaper — we use them for staging or intermediate tables in IICS pipelines where the data can be reloaded if needed. Temporary tables exist only for the duration of the session and are used for scratch work like deduplication within a session."*

---

## 2. Data Loading Patterns — INSERT, TRUNCATE + INSERT, MERGE

### What It Is
IICS uses different write strategies depending on the load type. As an IICS developer, you need to understand what happens in Snowflake when IICS runs each pattern.

### Pattern 1: Full Load (Truncate + Insert)
Used when the entire table is refreshed every run.

```sql
-- IICS does this under the hood for full load
TRUNCATE TABLE sales_db.public.orders;

INSERT INTO sales_db.public.orders (order_id, customer_id, order_date, amount)
SELECT order_id, customer_id, order_date, amount
FROM orders_stage;
```

> **TRUNCATE vs DELETE:**
> - `TRUNCATE` — Instantly removes all rows, no transaction log, cannot be rolled back, much faster.
> - `DELETE` — Logged operation, can be rolled back, slow on large tables.
> - Always prefer `TRUNCATE` for full reloads in IICS.

### Pattern 2: Incremental Load (INSERT only new records)
```sql
-- Insert only records that don't already exist in target
INSERT INTO orders (order_id, customer_id, order_date, amount)
SELECT s.order_id, s.customer_id, s.order_date, s.amount
FROM orders_stage s
WHERE NOT EXISTS (
    SELECT 1 FROM orders t WHERE t.order_id = s.order_id
);
```

### Pattern 3: Upsert / MERGE (Insert new + Update existing)
This is the most common pattern in IICS for maintaining final tables.

```sql
MERGE INTO orders AS target
USING orders_stage AS source
ON target.order_id = source.order_id

WHEN MATCHED THEN
    UPDATE SET
        target.customer_id = source.customer_id,
        target.order_date   = source.order_date,
        target.amount       = source.amount,
        target.load_dt      = CURRENT_TIMESTAMP()

WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, order_date, amount, load_dt)
    VALUES (source.order_id, source.customer_id, source.order_date,
            source.amount, CURRENT_TIMESTAMP());
```

### How to Answer in Interview
> **Q: How does IICS load data into Snowflake? What write strategies do you use?**

*"In IICS, we configure the target write strategy based on the business need. For full loads, IICS truncates the target table first and then inserts all records — this is the fastest and cleanest approach for daily snapshots. For incremental loads, we either insert only new records using a key lookup, or we use MERGE which handles both inserts and updates in a single statement. MERGE is the most common pattern I use for final tables where records can change over time, like order status updates or customer records."*

---

## 3. Staging — Internal & External Stages

### What It Is
When IICS loads data in bulk, it often uses Snowflake Stages — temporary holding areas for data files before they are loaded into tables via the `COPY INTO` command.

### Types of Stages
- **Internal Stage** — Files stored inside Snowflake's own storage.
- **External Stage** — Files stored in S3, Azure Blob, or GCS, referenced from Snowflake.

### How to Use It
```sql
-- Create a named internal stage
CREATE OR REPLACE STAGE my_internal_stage
    FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);

-- List files in the stage (use this to verify IICS staged the file correctly)
LIST @my_internal_stage;

-- Load from stage into table
COPY INTO orders
FROM @my_internal_stage/orders_20240101.csv
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE';  -- skip bad rows and continue

-- Check how many rows were loaded vs rejected
SELECT * FROM TABLE(VALIDATE(orders, JOB_ID => '_last'));
```

### How to Answer in Interview
> **Q: What is a Snowflake Stage and how is it used with IICS?**

*"A Snowflake Stage is a location where data files are temporarily held before being loaded into a table. IICS uses Stages internally when performing bulk loads — it writes the data to a stage first and then runs a COPY INTO command to load it into the target table. I use the LIST command to verify whether the file was staged correctly, and after a load I check the load history or use VALIDATE to confirm how many rows were loaded versus skipped. For external stages, the files sit in S3 or Azure Blob Storage, and Snowflake reads from them directly."*

---

## 4. Data Cleaning in Snowflake

### What It Is
After IICS loads raw data into Snowflake, data cleaning transforms it into usable, consistent, accurate records. Cleaning is done using SQL in Snowflake — either via a second IICS mapping or directly in a post-load SQL task.

### Common Cleaning Operations

#### 4a. Handling NULLs
```sql
-- Replace NULLs with default values
SELECT
    order_id,
    COALESCE(customer_id, -1)           AS customer_id,
    COALESCE(order_date, '1900-01-01')  AS order_date,
    COALESCE(amount, 0)                 AS amount
FROM orders_stage;

-- Filter out rows where key columns are NULL
DELETE FROM orders WHERE order_id IS NULL;

-- Count NULLs per column (quick data quality scan)
SELECT
    COUNT(*) - COUNT(order_id)     AS null_order_id,
    COUNT(*) - COUNT(customer_id)  AS null_customer_id,
    COUNT(*) - COUNT(amount)       AS null_amount
FROM orders;
```

#### 4b. Removing Duplicates
```sql
-- Identify duplicates
SELECT order_id, COUNT(*) AS cnt
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;

-- Remove duplicates — keep the latest record
DELETE FROM orders
WHERE order_id IN (
    SELECT order_id
    FROM (
        SELECT order_id,
               ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY load_dt DESC) AS rn
        FROM orders
    )
    WHERE rn > 1
);
```

#### 4c. Standardizing Strings
```sql
-- Trim whitespace and standardize case
UPDATE orders
SET customer_name = TRIM(UPPER(customer_name));

-- Replace special characters
SELECT REGEXP_REPLACE(phone_number, '[^0-9]', '') AS clean_phone
FROM customers;

-- Standardize date formats from string
SELECT TO_DATE(order_date_str, 'MM/DD/YYYY') AS order_date
FROM orders_raw;
```

#### 4d. Type Casting & Conversion
```sql
-- Cast string to number safely
SELECT TRY_CAST(amount_str AS DECIMAL(10,2)) AS amount
FROM orders_raw;

-- TRY_CAST returns NULL on failure instead of throwing an error
-- Use COALESCE to replace NULL with a default
SELECT COALESCE(TRY_CAST(amount_str AS DECIMAL(10,2)), 0) AS amount
FROM orders_raw;
```

### How to Answer in Interview
> **Q: How do you handle data cleaning in Snowflake as an IICS developer?**

*"After IICS loads raw data into the staging table, I apply cleaning logic in Snowflake using SQL — either through a second IICS mapping that reads the staging table and writes to the final table, or via pre/post SQL tasks in IICS. The most common cleaning steps I do are: handling NULLs with COALESCE, removing duplicates using ROW_NUMBER with a PARTITION BY on the business key, standardizing strings by trimming whitespace and fixing casing, and using TRY_CAST for safe type conversions so bad data doesn't fail the entire load."*

---

## 5. Data Validation in Snowflake

### What It Is
Validation confirms that the data loaded by IICS is correct — right row counts, expected values, no unexpected NULLs, and referential integrity. This is done in Snowsight worksheets or called as post-load SQL in IICS.

### Essential Validation Queries

#### 5a. Row Count Validation
```sql
-- Simple count
SELECT COUNT(*) AS total_rows FROM orders;

-- Count source vs target (run after IICS load)
SELECT 'Source' AS layer, COUNT(*) AS cnt FROM orders_stage
UNION ALL
SELECT 'Target', COUNT(*) FROM orders;
```

#### 5b. Latest Load Check
```sql
-- Verify the most recent data arrived
SELECT
    MAX(load_dt)   AS last_load_time,
    COUNT(*)       AS total_records,
    MIN(order_date) AS min_order_date,
    MAX(order_date) AS max_order_date
FROM orders;
```

#### 5c. Duplicate Check
```sql
SELECT order_id, COUNT(*) AS cnt
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1
ORDER BY cnt DESC;
```

#### 5d. NULL Check on Key Columns
```sql
SELECT
    SUM(CASE WHEN order_id IS NULL     THEN 1 ELSE 0 END) AS null_order_id,
    SUM(CASE WHEN customer_id IS NULL  THEN 1 ELSE 0 END) AS null_customer_id,
    SUM(CASE WHEN amount IS NULL       THEN 1 ELSE 0 END) AS null_amount
FROM orders;
```

#### 5e. Range / Business Rule Validation
```sql
-- Check for negative amounts (business rule: amount must be > 0)
SELECT COUNT(*) AS invalid_amount_rows
FROM orders
WHERE amount <= 0;

-- Check future dates (order_date should not be in the future)
SELECT COUNT(*) AS future_date_rows
FROM orders
WHERE order_date > CURRENT_DATE();
```

#### 5f. Referential Integrity Check
```sql
-- Orders without a valid customer
SELECT COUNT(*) AS orphan_orders
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL;
```

### How to Answer in Interview
> **Q: How do you validate data after an IICS load into Snowflake?**

*"After every IICS load, I run a set of standard validation queries in Snowflake. First, I compare row counts between the source staging table and the final target table to confirm no records were dropped. Then I check the MAX load date to verify the latest data actually arrived. I also run a duplicate check on the business key and a NULL check on mandatory columns. For business rule validations, I check things like negative amounts or future-dated records. These queries are either run manually in Snowsight during development, or wired into IICS as post-session SQL so they run automatically and fail the mapping if the checks don't pass."*

---

## 6. Time Travel — Data Recovery & Historical Validation

### What It Is
Snowflake Time Travel allows you to query data as it existed at a past point in time. This is extremely useful when an IICS load overwrites data incorrectly — you can recover the previous state without involving a DBA.

### How to Use It
```sql
-- Query data as it was 1 hour ago (offset in seconds)
SELECT * FROM orders AT (OFFSET => -3600);

-- Query data as it was at a specific timestamp
SELECT * FROM orders AT (TIMESTAMP => '2024-06-01 08:00:00'::TIMESTAMP);

-- Compare current data vs data from before the IICS load
SELECT COUNT(*) FROM orders;                                        -- current
SELECT COUNT(*) FROM orders AT (OFFSET => -60 * 30);               -- 30 mins ago

-- Restore accidentally overwritten table
CREATE OR REPLACE TABLE orders AS
SELECT * FROM orders AT (OFFSET => -3600);  -- restore to 1 hour ago

-- Undrop a dropped table
UNDROP TABLE orders;
```

### Time Travel Retention Periods
| Table Type | Default | Max |
|---|---|---|
| Permanent | 1 day | 90 days (Enterprise+) |
| Transient | 0 | 1 day |
| Temporary | 0 | 1 day |

> Set retention explicitly: `ALTER TABLE orders SET DATA_RETENTION_TIME_IN_DAYS = 7;`

### How to Answer in Interview
> **Q: Have you ever needed to recover data in Snowflake after a bad load?**

*"Yes. Snowflake's Time Travel feature is a lifesaver when an IICS load goes wrong — for example, if a full-load mapping accidentally truncated the table and the source data has an issue. I can query the table as it was before the load using the AT (OFFSET) or AT (TIMESTAMP) syntax, verify the data looks right, and then restore it by doing a CREATE OR REPLACE TABLE AS SELECT from the historical snapshot. I've also used UNDROP TABLE when a table was accidentally dropped. Time Travel on permanent tables defaults to 1 day but can be extended to 90 days on Enterprise edition, so we set it to at least 7 days on critical final tables."*

---

## 7. Query History & Debugging in Snowsight

### What It Is
Snowsight's Query History shows every SQL statement that ran against your Snowflake account — including all queries that IICS executed. This is the first place to go when an IICS mapping fails or produces unexpected results.

### How to Use It in Snowsight
1. Open Snowsight → **Activity > Query History**
2. Filter by: **User** (your IICS service account), **Status** (Failed), **Time range**
3. Click any query to see: full SQL text, execution time, rows returned, error message

### Useful SQL for Audit & Debugging
```sql
-- Check all queries run by IICS service account in the last 24 hours
SELECT
    query_id,
    query_text,
    start_time,
    end_time,
    execution_status,
    rows_produced,
    error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE user_name = 'IICS_SERVICE_USER'
  AND start_time >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;

-- Find failed queries
SELECT query_id, query_text, error_message, start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE execution_status = 'FAILED'
  AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;

-- Check load history for a specific table
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.LOAD_HISTORY
WHERE table_name = 'ORDERS'
ORDER BY last_load_time DESC
LIMIT 20;
```

### How to Answer in Interview
> **Q: How do you debug a failed IICS mapping that loads data into Snowflake?**

*"My first step is checking the IICS session log for the error message. If the error points to Snowflake, I go to Snowsight → Query History and filter by the IICS service account and the time window of the failed run. This shows me the exact SQL that IICS generated and the error Snowflake returned — for example, a data type mismatch or a permissions error. For load failures involving staged files, I also check the LOAD_HISTORY view in ACCOUNT_USAGE to see how many rows were loaded vs rejected and what error caused rejections. This combination of IICS logs and Snowflake Query History gives me everything I need to pinpoint the issue."*

---

## 8. Data Analysis Queries — Aggregations, Window Functions & Profiling

### What It Is
As an IICS developer, you often need to profile the data in your final Snowflake tables — either to validate it, share metrics with the business, or investigate data quality issues.

### Essential Analysis Queries

#### 8a. Basic Aggregations
```sql
-- Summary stats for a numeric column
SELECT
    COUNT(*)            AS total_rows,
    COUNT(amount)       AS non_null_amount,
    MIN(amount)         AS min_amount,
    MAX(amount)         AS max_amount,
    AVG(amount)         AS avg_amount,
    SUM(amount)         AS total_amount,
    STDDEV(amount)      AS stddev_amount
FROM orders;
```

#### 8b. Group-by Analysis
```sql
-- Orders and revenue by customer
SELECT
    customer_id,
    COUNT(*)        AS order_count,
    SUM(amount)     AS total_revenue,
    AVG(amount)     AS avg_order_value
FROM orders
GROUP BY customer_id
ORDER BY total_revenue DESC
LIMIT 10;
```

#### 8c. Window Functions (Ranking & Running Totals)
```sql
-- Rank customers by revenue
SELECT
    customer_id,
    SUM(amount) AS total_revenue,
    RANK() OVER (ORDER BY SUM(amount) DESC) AS revenue_rank
FROM orders
GROUP BY customer_id;

-- Running total of revenue by date
SELECT
    order_date,
    SUM(amount)                                          AS daily_revenue,
    SUM(SUM(amount)) OVER (ORDER BY order_date)         AS running_total
FROM orders
GROUP BY order_date
ORDER BY order_date;

-- Find most recent order per customer
SELECT *
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
WHERE rn = 1;
```

#### 8d. Data Profiling (Column-Level)
```sql
-- Profile every key column at once
SELECT
    COUNT(*)                                    AS total_rows,
    COUNT(DISTINCT order_id)                    AS unique_orders,
    COUNT(DISTINCT customer_id)                 AS unique_customers,
    MIN(order_date)                             AS earliest_order,
    MAX(order_date)                             AS latest_order,
    ROUND(AVG(amount), 2)                       AS avg_amount,
    SUM(CASE WHEN amount IS NULL THEN 1 END)    AS null_amounts
FROM orders;
```

#### 8e. Trend Analysis by Period
```sql
-- Monthly revenue trend
SELECT
    DATE_TRUNC('month', order_date)  AS month,
    COUNT(*)                          AS order_count,
    ROUND(SUM(amount), 2)             AS monthly_revenue
FROM orders
GROUP BY 1
ORDER BY 1;
```

### How to Answer in Interview
> **Q: How do you use SQL in Snowflake for data analysis as an IICS developer?**

*"Once the data is in Snowflake, I use SQL in Snowsight worksheets to profile and analyze it. For basic profiling, I run aggregations — COUNT, MIN, MAX, AVG, SUM — to understand the data range and check for anomalies. For more advanced analysis, I use window functions like ROW_NUMBER for deduplication and RANK or SUM OVER for running totals and rankings. DATE_TRUNC is very useful for grouping data by month or week to spot trends. All of this helps me validate that the IICS load produced the correct output and gives business teams quick metrics without needing a BI tool."*

---

## 9. Information Schema — Table & Column Metadata

### What It Is
Snowflake's INFORMATION_SCHEMA is a built-in set of views that describe your database objects — tables, columns, stages, and more. IICS developers use this to inspect target table structures, check column data types, and automate documentation.

### How to Use It
```sql
-- List all tables in a schema
SELECT table_name, table_type, row_count, created
FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'PUBLIC'
ORDER BY created DESC;

-- List all columns for a specific table
SELECT column_name, data_type, is_nullable, column_default
FROM INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'ORDERS'
  AND table_schema = 'PUBLIC'
ORDER BY ordinal_position;

-- Check all stages in the schema
SELECT stage_name, stage_type, stage_url
FROM INFORMATION_SCHEMA.STAGES;
```

### How to Answer in Interview
> **Q: How do you check the structure of a Snowflake table without opening Snowsight?**

*"I use the INFORMATION_SCHEMA views. INFORMATION_SCHEMA.COLUMNS gives me the column names, data types, and whether they allow NULLs — which I compare against the IICS mapping to spot any mismatches before running a load. INFORMATION_SCHEMA.TABLES gives me row counts and creation timestamps, which is useful for confirming when a table was last rebuilt. These are faster than navigating through the Snowsight UI, especially when scripting pre-load checks."*

---

## 10. Roles & Grants — IICS Service Account Setup

### What It Is
IICS connects to Snowflake using a service account (a dedicated user). That service account needs exactly the right permissions — no more, no less. As an IICS developer, you need to understand what grants are required so you can work with your Snowflake admin to set them up correctly.

### Minimum Required Grants for IICS Service Account
```sql
-- Create a dedicated role for IICS
CREATE ROLE iics_role;

-- Warehouse access (required to run queries)
GRANT USAGE ON WAREHOUSE iics_warehouse TO ROLE iics_role;

-- Database access
GRANT USAGE ON DATABASE sales_db TO ROLE iics_role;

-- Schema access
GRANT USAGE ON SCHEMA sales_db.public TO ROLE iics_role;

-- Table-level privileges
GRANT SELECT, INSERT, UPDATE, DELETE, TRUNCATE
    ON ALL TABLES IN SCHEMA sales_db.public TO ROLE iics_role;

-- Stage access (for bulk loading)
GRANT READ, WRITE ON STAGE sales_db.public.my_stage TO ROLE iics_role;

-- File format access
GRANT USAGE ON FILE FORMAT sales_db.public.my_csv_format TO ROLE iics_role;

-- Assign role to IICS user
GRANT ROLE iics_role TO USER iics_service_user;
ALTER USER iics_service_user SET DEFAULT_ROLE = iics_role;
```

### How to Answer in Interview
> **Q: What permissions does the IICS service account need in Snowflake?**

*"The IICS service account needs a dedicated role with the minimum required privileges. At the warehouse level, it needs USAGE so it can execute queries. At the database and schema level, it needs USAGE for access. On the target tables, it needs SELECT, INSERT, UPDATE, DELETE, and TRUNCATE — especially TRUNCATE for full-load patterns. If the mappings use staged file loading, the role also needs READ and WRITE on the stage and USAGE on the file format. I always work with the Snowflake admin to set up a least-privilege role specifically for IICS rather than using SYSADMIN, which is a security risk."*

---

## Quick Reference Card

| Task | SQL / Location |
|---|---|
| Full reload | `TRUNCATE TABLE` → `INSERT INTO` |
| Upsert | `MERGE INTO target USING source ON key` |
| Check load | `SELECT COUNT(*), MAX(load_dt) FROM table` |
| Find duplicates | `GROUP BY key HAVING COUNT(*) > 1` |
| Recover bad load | `SELECT * FROM table AT (OFFSET => -3600)` |
| Restore table | `CREATE OR REPLACE TABLE t AS SELECT * FROM t AT (...)` |
| Undrop table | `UNDROP TABLE orders` |
| Debug IICS queries | Snowsight → Activity → Query History |
| Check failed loads | `SNOWFLAKE.ACCOUNT_USAGE.LOAD_HISTORY` |
| Inspect columns | `INFORMATION_SCHEMA.COLUMNS` |
| Profile data | `COUNT, MIN, MAX, AVG, STDDEV, COUNT(DISTINCT)` |
| Trend analysis | `DATE_TRUNC('month', date_col)` + `GROUP BY` |
| Dedup per group | `ROW_NUMBER() OVER (PARTITION BY key ORDER BY date DESC)` |

---

*Guide prepared for IICS Developers working with Snowflake as the final target layer.*
