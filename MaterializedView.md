# Snowflake Materialized Views — Complete Guide (Beginner to Expert)

---

## 1. What is a Materialized View?

A **Materialized View (MV)** is a **pre-computed, physically stored** result of a SQL query.

Think of it this way:

| Type | What It Is | Analogy |
|------|-----------|---------|
| **Regular View** | A saved SQL query. Re-runs every time you access it. | A recipe — you cook fresh each time |
| **Materialized View** | A saved SQL query + **stored results**. Only re-computes when base data changes. | A meal-prepped dish — ready to serve instantly |
| **Table** | Raw stored data. Never auto-updates from another source. | Raw ingredients sitting in the fridge |

> **One-liner:** MV = Speed of a table + Freshness of a view

---

## 2. How It Works (Step by Step)

```
STEP 1: You write  →  CREATE MATERIALIZED VIEW ... AS SELECT ...
STEP 2: Snowflake  →  Runs the query and STORES the result (like a table)
STEP 3: You query  →  Snowflake reads from STORED data (no re-computation)
STEP 4: Base table changes (INSERT/UPDATE/DELETE)
        →  Background service AUTO-REFRESHES the MV
STEP 5: If MV is slightly behind, Snowflake combines
        MV data + fresh base data = always CORRECT results
```

**You never manually refresh it.** Snowflake handles everything automatically.

---

## 3. Syntax

### Create
```sql
CREATE OR REPLACE MATERIALIZED VIEW <database>.<schema>.<mv_name>
AS
SELECT <columns>
FROM <base_table>
WHERE <optional_filter>;
```

### With Aggregation
```sql
CREATE OR REPLACE MATERIALIZED VIEW mv_claim_summary
AS
SELECT
    POLICY_ID,
    COUNT(*) AS TOTAL_CLAIMS,
    SUM(CLAIM_AMOUNT) AS TOTAL_AMOUNT
FROM CLAIM
GROUP BY POLICY_ID;
```

### With Clustering (for large data)
```sql
CREATE OR REPLACE MATERIALIZED VIEW mv_claims_by_status
  CLUSTER BY (STATUS)
AS
SELECT CLAIM_ID, POLICY_ID, CLAIM_DATE, STATUS
FROM CLAIM
WHERE STATUS IN ('PENDING', 'APPROVED');
```

### Manage
```sql
-- View all MVs
SHOW MATERIALIZED VIEWS IN SCHEMA <db>.<schema>;

-- Pause auto-refresh (saves credits)
ALTER MATERIALIZED VIEW <mv_name> SUSPEND;

-- Resume auto-refresh
ALTER MATERIALIZED VIEW <mv_name> RESUME;

-- Delete
DROP MATERIALIZED VIEW <mv_name>;

-- Check refresh cost
SELECT * FROM TABLE(INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY());
```

---

## 4. Working Examples (Using Insurance Tables)

### Example 1: Filter — Active Policies Only
**Problem:** Every mapping scans full POLICY table just to get active ones.

```sql
CREATE OR REPLACE MATERIALIZED VIEW DEV_CL_DB.APP_GS_WORK.MV_ACTIVE_POLICIES
AS
SELECT POLICY_ID, CUSTOMER_ID, POLICY_NUMBER, POLICY_TYPE,
       START_DATE, END_DATE, PREMIUM_AMOUNT
FROM DEV_CL_DB.APP_GS_WORK.POLICY
WHERE STATUS = 'ACTIVE';
```

### Example 2: Aggregation — Claim Summary Per Policy
**Problem:** Multiple reports need total/max/min claims per policy.

```sql
CREATE OR REPLACE MATERIALIZED VIEW DEV_CL_DB.APP_GS_WORK.MV_CLAIM_SUMMARY
AS
SELECT POLICY_ID,
       COUNT(*) AS TOTAL_CLAIMS,
       SUM(CLAIM_AMOUNT) AS TOTAL_CLAIM_AMOUNT,
       MAX(CLAIM_AMOUNT) AS MAX_CLAIM_AMOUNT,
       MIN(CLAIM_DATE) AS FIRST_CLAIM_DATE,
       MAX(CLAIM_DATE) AS LAST_CLAIM_DATE
FROM DEV_CL_DB.APP_GS_WORK.CLAIM
GROUP BY POLICY_ID;
```

### Example 3: Lookup — Coverage Totals
**Problem:** Lookup transformation needs coverage data per policy.

```sql
CREATE OR REPLACE MATERIALIZED VIEW DEV_CL_DB.APP_GS_WORK.MV_COVERAGE_LOOKUP
AS
SELECT POLICY_ID,
       MAX(COVERAGE_LIMIT) AS MAX_COVERAGE,
       SUM(DEDUCTIBLE) AS TOTAL_DEDUCTIBLE,
       COUNT(*) AS NUM_COVERAGES
FROM DEV_CL_DB.APP_GS_WORK.COVERAGE
GROUP BY POLICY_ID;
```

---

## 5. Full Comparison Table

| Feature | Regular View | Materialized View | Table | Dynamic Table |
|---------|-------------|-------------------|-------|---------------|
| **Stores Data?** | No | Yes | Yes | Yes |
| **Auto-Refresh?** | N/A (always live) | Yes (automatic) | No | Yes (scheduled) |
| **Query Speed** | Slow (re-runs query) | Fast (reads cache) | Fast | Fast |
| **Storage Cost** | None | Yes | Yes | Yes |
| **Compute Cost** | On every query | On refresh only | None | On refresh only |
| **Always Current?** | Yes | Yes (auto-synced) | No (stale) | Near real-time |
| **Supports JOINs?** | Yes | **No** | N/A | Yes |
| **Supports Window Functions?** | Yes | **No** | N/A | Yes |
| **DML (INSERT/UPDATE)?** | No | No | Yes | No |
| **Enterprise Edition?** | No | **Yes** | No | No |

---

## 6. IICS ETL Concept Mapping

| Materialized View Feature | IICS Equivalent |
|--------------------------|-----------------|
| Pre-filtered rows stored | **Source Qualifier** with pushdown filter |
| Pre-aggregated totals | **Aggregator Transformation** output cached |
| Fast data retrieval | **Persistent Lookup Cache** (connected/unconnected) |
| Shared across mappings | **Reusable Transformation** / Mapplet |
| Auto-refresh on change | **Event-based Task** triggering re-processing |

### Real-World IICS Scenarios

| Scenario | Without MV | With MV |
|----------|-----------|---------|
| 5 mappings read POLICY WHERE STATUS='ACTIVE' | Each scans full table (5x cost) | All read MV_ACTIVE_POLICIES (pre-filtered) |
| Lookup needs SUM(CLAIM_AMOUNT) per policy | Aggregation runs every mapping execution | MV_CLAIM_SUMMARY already has totals |
| Delta load needs only PENDING claims | Full table scan + filter each cycle | MV_CLAIMS_BY_STATUS has only needed rows |
| Dashboard queries every 5 minutes | Warehouse computes aggregation each time | Pre-computed, near-instant, saves credits |
| Multiple targets use same filtered source | Source Qualifier filter runs N times | MV stores filtered data once, read N times |

---

## 7. Auto-Refresh Behavior

| Base Table Action | What Happens to MV |
|------------------|-------------------|
| `INSERT` new rows | MV adds matching rows automatically |
| `DELETE` rows | MV removes those rows (compaction) |
| `UPDATE` rows | Treated as DELETE + INSERT internally |
| No changes | MV sits idle (no cost) |

**Key Points:**
- Refresh runs in the **background** (serverless compute, no warehouse needed)
- You are **billed** for the compute used by refresh
- If MV is slightly behind, Snowflake **merges** MV + base table data on-the-fly (always correct)
- Check freshness: `SHOW MATERIALIZED VIEWS` → look at `BEHIND_BY` column

---

## 8. Query Optimizer — Automatic Rewrite

Even if you query the **base table**, Snowflake may automatically use the MV:

```
You write:   SELECT * FROM POLICY WHERE STATUS = 'ACTIVE';
Snowflake sees MV_ACTIVE_POLICIES has this exact data.
Snowflake rewrites to: SELECT * FROM MV_ACTIVE_POLICIES;
Result: Faster — without changing your SQL!
```

Verify with:
```sql
EXPLAIN
SELECT POLICY_ID, CUSTOMER_ID, PREMIUM_AMOUNT
FROM DEV_CL_DB.APP_GS_WORK.POLICY
WHERE STATUS = 'ACTIVE';
-- Check if MV name appears in the plan
```

---

## 9. Cost Control Best Practices

| Strategy | How |
|----------|-----|
| Suspend during off-hours | `ALTER MATERIALIZED VIEW mv_name SUSPEND;` |
| Resume before ETL window | `ALTER MATERIALIZED VIEW mv_name RESUME;` |
| Monitor credit usage | `SELECT * FROM TABLE(INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY());` |
| Avoid on high-DML tables | Frequent INSERT/UPDATE = high refresh cost |
| Use for read-heavy workloads | Best ROI when data is read 10x more than written |

---

## 10. Limitations

| Restriction | Detail |
|------------|--------|
| No JOINs | Single base table only |
| No Window Functions | ROW_NUMBER, RANK, LAG, etc. not allowed |
| No UDFs | User-defined functions not supported |
| No Subqueries | No nested SELECT or CTEs |
| No HAVING / ORDER BY / LIMIT | Not supported in MV definition |
| Limited Aggregates | Only SUM, COUNT, AVG, MIN, MAX, STDDEV, VARIANCE |
| No DML | Cannot INSERT/UPDATE/DELETE into MV |
| No Time Travel | Cannot query MV at a past point in time |
| Enterprise Edition | Required (not available on Standard Edition) |

---

## 11. MV vs Dynamic Table — When to Use Which?

| Criteria | Materialized View | Dynamic Table |
|----------|-------------------|---------------|
| **Query Complexity** | Simple (single table, basic aggregates) | Complex (JOINs, window functions, CTEs) |
| **Refresh** | Automatic (near real-time) | Scheduled (TARGET_LAG) |
| **Best For** | Filtering, simple aggregation, lookups | Multi-table transformations, pipelines |
| **Edition** | Enterprise | Standard or higher |
| **IICS Analogy** | Lookup Cache / Source Qualifier | Mapping with multiple sources + transforms |

**Rule of Thumb:**
- Can do it with **single table + simple aggregates**? → Use **Materialized View**
- Need **JOINs, window functions, complex logic**? → Use **Dynamic Table**

---

## 12. Common Interview Questions

**Q: Can you INSERT/UPDATE/DELETE into a Materialized View?**
> No. Only Snowflake's background service modifies it. You modify the base table.

**Q: What happens if the base table is dropped?**
> MV becomes INVALID/SUSPENDED. You must recreate it.

**Q: What happens if a column is dropped from the base table?**
> MV becomes INVALID. Must recreate with remaining columns.

**Q: Can you create an MV on another MV?**
> No. Only on a base table (not a view, not another MV, not a dynamic table).

**Q: Does MV support Time Travel?**
> No. You cannot query an MV at a past point in time.

**Q: What if the base table changes very frequently?**
> MV maintenance cost will be high. Consider a regular view or dynamic table instead.

**Q: Can the optimizer use MV even if you don't reference it?**
> Yes. Snowflake auto-rewrites queries to use MV if it contains the needed data.

**Q: How is MV different from CTAS (CREATE TABLE AS SELECT)?**
> CTAS = static copy (never auto-updates). MV = live copy (auto-updates when base changes).

**Q: What is the difference between SUSPEND and DROP?**
> SUSPEND = pauses auto-refresh (MV still exists, saves credits). DROP = deletes MV entirely.

---

## 13. Quick Reference Cheat Sheet

```
CREATE   →  CREATE MATERIALIZED VIEW mv_name AS SELECT ...
QUERY    →  SELECT * FROM mv_name;
SUSPEND  →  ALTER MATERIALIZED VIEW mv_name SUSPEND;
RESUME   →  ALTER MATERIALIZED VIEW mv_name RESUME;
DROP     →  DROP MATERIALIZED VIEW mv_name;
CHECK    →  SHOW MATERIALIZED VIEWS IN SCHEMA db.schema;
COST     →  SELECT * FROM TABLE(INFORMATION_SCHEMA.MATERIALIZED_VIEW_REFRESH_HISTORY());
```
