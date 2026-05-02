# SARANYA FIDELITY INTERVIEW — MASTER REFERENCE
_Saved: May 2026 | Do NOT delete — used for interview coaching_

---

## 👤 CANDIDATE
- **Name:** Saranya Arumugam
- **Experience:** 6+ years ETL / Data Engineering (Cognizant)
- **Key Skills:** IICS (Informatica Intelligent Cloud Services), Python, SQL, IBM Cognos, Azure AI
- **Current Program:** NPower Canada (Apr–Jul 2026)
- **Certifications:** IBM Data Analyst, Azure AI-900 (in progress)
- **Education:** B.E. ECE 2015
- **Domain:** Insurance / BFSI
- **Primary Stack:** Oracle → Snowflake (IICS as ETL layer)

---

## 📄 JD FOCUS
- Role: IICS Developer
- Key technologies: IICS / IDMC, Snowflake, Oracle, JSON, Flat Files
- Domain: Insurance / Finance / BFSI

---

## 🗂️ FILES CREATED (all in outputs folder)

| File | Content |
|---|---|
| `Saranya_IICS_Interview_Prep.html` | 3 complex IICS mappings (GL domain, Oracle+CSV→Snowflake) + Snowflake post-load + Q&A |
| `Saranya_IICS_2Mappings_Insurance.html` | 2 new mappings: JSON+Oracle→Snowflake (Claims) and Oracle+Oracle→Snowflake (Agent Premium) |
| `Saranya_IICS_Production_Issues.html` | 6 production issues with root cause, fix, and interview answer |

---

## 🏗️ MAPPING 1 — Heterogeneous GL Transaction Load (Oracle + CSV → Snowflake)
**Domain:** Finance / General Ledger
**Sources:** Oracle GL_TRANSACTIONS + CSV Chart of Accounts (flat file)
**Target:** Snowflake FACT_GL_TRANSACTIONS

### Transformations (in order):
1. **SQ_ORACLE_GL** (Source Qualifier) — SQL override filter on Oracle, only POSTED entries
2. **SRC_CSV_COA** (Flat File Source) — reads chart_of_accounts.csv
3. **EXP_CLEAN_COA** (Expression) — trim/UPPER CSV data before join
4. **JNR_GL_COA** (Joiner) — LEFT OUTER join Oracle + CSV on account_code (heterogeneous — can't do in DB)
5. **FIL_VALID_RECORDS** (Filter) — remove zero-amount and closed account rows
6. **LKP_DIM_DATE / LKP_DIM_CURRENCY / LKP_DIM_COSTCENTER** (3 Lookups) — fetch surrogate keys from Snowflake dimensions
7. **EXP_DERIVE_AUDIT** (Expression) — net_amount, load_date, source_system, record_hash
8. **TGT_FACT_GL_TRANSACTIONS** (Snowflake Target) — INSERT only (GL is immutable)

**Key interview point:** Joiner for heterogeneous sources (Oracle ≠ CSV, can't SQL JOIN). Lookup for dimension tables (small, cached, dictionary lookup). Left Outer Join so no GL transaction is lost even if COA is missing.

---

## 🏗️ MAPPING 2 — Incremental Load with Validation & Error Routing (Oracle AP → Snowflake)
**Domain:** Accounts Payable / Finance
**Source:** Oracle AP_INVOICES (delta via $$LAST_RUN_DATE parameter)
**Target:** Snowflake FACT_AP_INVOICES + REJECT_AP_LOG

### Transformations:
1. **SQ_AP_INVOICES** — SQL override `WHERE last_updated_date > $$LAST_RUN_DATE`
2. **EXP_VALIDATE** — 4 business rules: amount>0, date valid, account not null, vendor not null → v_is_valid flag
3. **RTR_VALID_INVALID** (Router) — VALID → fact table path, INVALID → reject log path
4. **LKP_DIM_VENDOR / LKP_DIM_DATE / LKP_DIM_ACCT** — surrogate key lookups (valid path)
5. **EXP_REJECT_REASON** — builds human-readable error message (invalid path)
6. **TGT_FACT_AP_INVOICES** + **TGT_REJECT_AP_LOG** — two targets

**Key interview point:** Router vs Filter — Filter drops records silently. Router keeps both paths alive — valid to fact, invalid to reject log. Insurance/Finance regulatory requirement: never silently drop records.

---

## 🏗️ MAPPING 3 — Budget vs Actuals with Normalizer (CSV Budget + Oracle GL → Snowflake)
**Domain:** Finance — CFO Dashboard
**Sources:** CSV budget file (wide format: 12 monthly columns) + Oracle GL_ACTUALS
**Target:** Snowflake FACT_BUDGET_VS_ACTUALS

### Transformations:
1. **SRC_BUDGET_CSV** — flat file source (wide format)
2. **NORM_BUDGET** (Normalizer, occurrence=12) — UNPIVOTS 12 monthly columns into 12 rows, outputs GCID (1–12)
3. **EXP_DERIVE_PERIOD** — GCID 1→202601, GCID 12→202612
4. **SQ_GL_ACTUALS** — Oracle actuals for same period
5. **JNR_BUDGET_ACTUALS** — LEFT OUTER join on cost_center + period_id
6. **FIL_CURRENT_YEAR** — only current fiscal year records
7. **EXP_VARIANCE** — variance = actual - budget, variance_pct, ALERT if >10%
8. **AGG_QUARTERLY** (Aggregator) — quarterly roll-up by cost_center
9. **LKP_DIM_COSTCENTER** — surrogate key
10. **TGT_FACT_BUDGET_VS_ACTUALS** — Snowflake

**Key interview point:** Normalizer is the only IICS transformation that unpivots columnar data to rows. Without it, you'd need 12 separate source branches. GCID is the sequence number output that maps to month.

---

## 🏗️ MAPPING 4 — JSON (Claims App) + Oracle (Policy Master) → Snowflake FACT_CLAIMS
**Domain:** Insurance — Claims Analytics
**Sources:** JSON flat file (mobile claims app) + Oracle POLICY_MASTER
**Target:** Snowflake FACT_CLAIMS + REJECT_CLAIMS_LOG

### Transformations:
1. **HIER_CLAIMS** (Hierarchy Parser) — opens nested JSON, flattens to rows/columns
2. **SQ_POLICY_MASTER** — Oracle, only ACTIVE policies
3. **EXP_CLEAN_JSON** — trim/UPPER/date format JSON fields before join
4. **JNR_CLAIM_POLICY** (Joiner) — LEFT OUTER on policy_number
5. **EXP_VALIDATE_CLAIM** — 3 rules: amount≤sum_insured, date within policy period, policy active
6. **RTR_VALID_INVALID** (Router) — valid→fact, invalid→reject
7. **LKP_DIM_DATE / LKP_DIM_PRODUCT / LKP_DIM_AGENT** — 3 surrogate key lookups
8. **EXP_AUDIT_COLS** — etl_load_dt, source_system, batch_id, record_hash
9. **FACT_CLAIMS** + **REJECT_CLAIMS_LOG** — two Snowflake targets

**Key interview point:** Hierarchy Parser is mandatory for JSON. Expression before Joiner to clean JSON (app data is always dirty). Router because insurance can never silently drop a claim — every rejection must be logged.

---

## 🏗️ MAPPING 5 — Oracle PAS + Oracle AMS → Snowflake FACT_AGENT_PREMIUM
**Domain:** Insurance — Agent Performance Dashboard
**Sources:** Oracle DB1 (Policy Admin System — PREMIUM_TRANSACTIONS) + Oracle DB2 (Agent Management System — AGENT_MASTER)
**Target:** Snowflake FACT_AGENT_PREMIUM

### Key point: Two SEPARATE Oracle databases — different servers, different schemas — cannot SQL JOIN across them. IICS is the only place the join can happen.

### Transformations:
1. **SQ_PREMIUM_TXN** — Oracle DB1, incremental `WHERE transaction_date > $$LAST_RUN_DT`
2. **SQ_AGENT_MASTER** — Oracle DB2, `WHERE agent_status = 'ACTIVE'` (separate connection object)
3. **EXP_CLEAN_PAS** — standardize agent_id (UPPER, trim) from PAS
4. **EXP_CLEAN_AMS** — standardize agent_id from AMS (two systems, different conventions)
5. **JNR_PREMIUM_AGENT** (Joiner) — LEFT OUTER on agent_id, Master=PAS, Detail=AMS
6. **FIL_VALID_TRANSACTIONS** — amount>0 AND paid_date NOT NULL AND payment_mode in valid list
7. **EXP_CALCULATE** — commission_amt, days_overdue, overdue_flag, net_premium
8. **AGG_AGENT_MONTHLY** (Aggregator) — GROUP BY agent+month, SUM premium/commission, COUNT policies
9. **LKP_DIM_AGENT / LKP_DIM_DATE** — surrogate key lookups
10. **EXP_AUDIT_COLS** — audit metadata
11. **FACT_AGENT_PREMIUM** — Snowflake

**Key interview point:** Two Source Qualifiers because two separate Oracle DBs. Expression on BOTH sides before Joiner to standardize join keys. Aggregator to pre-summarize agent+month — fact table granularity.

---

## ❄️ SNOWFLAKE POST-LOAD STEPS
1. **MERGE** — SCD Type 2 for dimension tables (update old, insert new version)
2. **Reconciliation** — COUNT(*) and SUM check vs source
3. **Update ETL Control Table** — set last_run_dt, rows_loaded, status=SUCCESS
4. **Refresh Dynamic Tables / Materialized Views** — downstream aggregates refresh
5. **Column Masking Policies** — sensitive finance/insurance data security
6. **Clustering Keys** — on date column for fast BI query pruning
7. **Trigger BI Refresh** — IBM Cognos / Power BI notified data is ready
8. **Snowflake Tasks + Streams** — for CDC within Snowflake (advanced)

---

## 🚨 6 PRODUCTION ISSUES

### Issue 1 — Source-Target Row Count Mismatch
- **Symptom:** 1,878 premium records missing in Snowflake daily
- **Root Cause:** Filter `NOT ISNULL(paid_date)` was dropping backdated payments that have NULL paid_date by design (business rule change not communicated to ETL team)
- **Fix:** Updated filter to allow `payment_mode = 'BACKDATED'` even with NULL paid_date. Added post-load count reconciliation in taskflow.
- **Interview line:** "Root cause was a business rule change not communicated to ETL team. Fix was filter condition update + automated reconciliation."

### Issue 2 — Mapping Performance Degraded (90 min → 8 hours)
- **Symptom:** Nightly claims load grew from 90 min to 8 hours after campaign volume spike
- **Root Cause (3 causes):** Joiner unsorted → disk spill; Oracle index on claim_date gone UNUSABLE after partition rebuild; Pushdown optimization disabled
- **Fix:** Sorted input on Joiner; DBA rebuilt index; enabled full pushdown optimization
- **Result:** 8 hours → 45 minutes
- **Interview line:** "Three compounding issues — sorted input, index rebuild, pushdown optimization."

### Issue 3 — Lookup Cache Returning Stale Data
- **Symptom:** Agent region showing SOUTH in reports when agent moved to NORTH
- **Root Cause:** Persistent lookup cache rebuilding only Monday. Agent transferred Tuesday. Rest of week used Monday's stale cache.
- **Fix:** Dynamic cache (rebuild every session) for slow-changing dimensions. Static dimensions keep persistent cache. Fact table stores only surrogate key, not the attribute.
- **Interview line:** "Cache strategy must match dimension change frequency."

### Issue 4 — Duplicate Records (Premium Doubled)
- **Symptom:** FACT_AGENT_PREMIUM showing 2× premium. COUNT=104,680, COUNT(DISTINCT)=52,340.
- **Root Cause:** Mapping failed at 98% on Tuesday (Snowflake timeout). 51,800 rows already written. Re-run Wednesday without clearing partial data → duplicates.
- **Fix:** Pre-load DELETE by batch_id in taskflow. Stage-then-MERGE pattern. Idempotent design.
- **Interview line:** "No idempotency. Fixed with pre-load cleanup + MERGE pattern."

### Issue 5 — Secure Agent Down (All 23 Jobs Failed)
- **Symptom:** Monday morning — all 23 mappings failed, dashboards blank
- **Root Cause:** Server restarted after OS patch. Secure Agent service startup = MANUAL, not auto-start. Offline for 48 hours undetected.
- **Fix:** `systemctl enable infaagent`. Cron health check every 5 min. Change management process for server patches.
- **Interview line:** "Agent not set to auto-start after OS patch. Fixed with systemctl enable + monitoring cron."

### Issue 6 — File Not Found + Wrong Column Count
- **Incident A:** Upstream team changed file delivery path `/data/inbound/` → `/data/feeds/claims/` without notice
- **Incident B:** Upstream added 2 new columns to CSV without notice → column count mismatch
- **Root Cause (both):** No File Interface Change Management process
- **Fix:** Pre-mapping Command Task validates file existence and column count. File Interface Specification document. Formal change request required for any upstream file change.
- **Interview line:** "Same root cause both times — no interface change management. Fixed with pre-validation script + formal change process."

---

## 🎯 KEY CONCEPTS TO REMEMBER

### Joiner vs Lookup
| | Joiner | Lookup |
|---|---|---|
| Use for | Two large/equal-weight datasets, heterogeneous sources | Small reference/dimension tables |
| Memory | Builds hash table on smaller side | Caches entire lookup table once |
| Use case | Oracle + CSV, Oracle DB1 + Oracle DB2 | DIM_DATE, DIM_AGENT, DIM_PRODUCT |
| Analogy | Stapling two files together | Checking a dictionary |

### Router vs Filter
- **Filter** = silently drops records that don't match → use when discarded rows have NO business value
- **Router** = splits data into multiple paths → use when rejected records must be LOGGED and REVIEWED (insurance, finance)

### Normalizer
- Used to UNPIVOT wide/columnar data into rows
- occurrence = number of columns to unpivot (e.g., 12 for months)
- GCID = generated sequence number output (1, 2, 3... 12)
- Use case: Budget files with monthly columns

### Persistent vs Dynamic Cache (Lookups)
- Static dimensions (DIM_DATE) → Persistent cache, weekly rebuild OK
- Slow-changing (DIM_AGENT) → Dynamic cache, rebuild every session
- Fast-changing (status tables) → No cache, query live

### Incremental Load Pattern
- `$$LAST_RUN_DATE` mapping parameter
- SQL: `WHERE last_updated_date > $$LAST_RUN_DATE`
- After load: update control table with new timestamp
- Re-run safety: always DELETE by batch_id before INSERT (idempotency)

---

---

## 🏗️ MAPPING 6 — 3 STG Tables (Snowflake) → FACT_POLICY_SUMMARY (Snowflake)
**Domain:** Insurance — Policy Analytics
**Architecture:** Mix — IICS Taskflow + Snowflake Stored Procedure
**STG Sources (all in Snowflake, loaded by earlier mappings):**
- STG_POLICY (from Oracle Policy Admin System)
- STG_CLAIMS (from S3 CSV — approved claims)
- STG_RATE_GRADE (from XML rate/grade config)

**Target:** Snowflake FACT_POLICY_SUMMARY (one row per policy per reporting year, SCD Type 2)

### Why 3 STG Tables → IICS handles business logic (not source load)
Phase 1 (Mappings 1–3): Load raw source data into STG tables — no business logic.
Phase 2 (Mapping 4): Read from STG tables, apply business logic, load into FACT table.

### Transformation Chain (in order):
1. **SQ_STG_POLICY** — `WHERE reporting_year = $$REPORT_YEAR AND policy_status = 'ACTIVE'`
2. **SQ_STG_CLAIMS** — `WHERE reporting_year = $$REPORT_YEAR AND claim_status = 'APPROVED'`
3. **SQ_STG_RATE_GRADE** — `WHERE CURRENT_DATE() BETWEEN eff_from_dt AND eff_to_dt`
4. **AGG_CLAIMS_BY_POLICY** (Aggregator) — BEFORE JOIN: GROUP BY policy_number + reporting_year → SUM(claim_amount), COUNT(claim_id). WHY: prevent row multiplication (1 policy × 5 claims = 5 rows without AGG; with AGG = 1 claim row per policy)
5. **JNR_POLICY_CLAIMS** (Joiner) — LEFT OUTER on policy_number + reporting_year (Policy = Master, AGG_CLAIMS = Detail). WHY: keep all policies even if no claims
6. **JNR_POLICY_GRADE** (Joiner) — LEFT OUTER on product_type (brings ALL grade rows: A, B, C, D per policy). WHY: Lookup returns only 1 row; Joiner returns all 4 so Expression can evaluate which grade applies
7. **EXP_CALCULATE** (Expression):
   - `claim_ratio = total_claim_amt / sum_insured * 100`
   - `v_grade_match = IIF(claim_ratio BETWEEN min_claim_ratio AND max_claim_ratio, 'Y', 'N')`
   - `calculated_premium = sum_insured * rate_pct / 100`
   - `v_is_valid = IIF(total_claim_amt <= sum_insured, 'VALID', 'INVALID')`
   - `record_hash = MD5(policy_number || reporting_year || grade_code)`
8. **FIL_GRADE_MATCH** (Filter) — `v_grade_match = 'Y'` — collapses 4 grade rows down to 1 correct grade row per policy
9. **RTR_VALID_INVALID** (Router) — VALID → WORK_POLICY_SUMMARY; INVALID → REJECT_POLICY_LOG
10. **Command Task** — calls `CALL SP_MERGE_POLICY_SUMMARY('$$BATCH_ID')` in Snowflake
11. **FACT_POLICY_SUMMARY** — populated atomically by Snowflake SP (SCD Type 2)

**Key interview point:** AGG before JOIN to prevent row multiplication. Second Joiner (not Lookup) for rate/grade because multiple rows per product_type. FIL after EXP to collapse to 1 grade row per policy. SP handles SCD2 MERGE atomically.

---

## ⚡ PUSHDOWN OPTIMIZATION

### Without Pushdown:
Data travels: Snowflake → network → IICS Server RAM → network → Snowflake (slow, uses IICS memory)

### With Full Pushdown (STG → FACT, both Snowflake):
IICS generates ONE SQL statement → sends to Snowflake → Snowflake executes using MPP internally (zero network transfer, fastest possible)

### 3 Types:
| Type | When | What IICS does |
|---|---|---|
| Full | Source + Target = same DB (Snowflake→Snowflake) | Converts entire mapping to SQL, runs in DB |
| Partial | Some transformations pushable, some not | Splits: pushes DB parts to DB, runs rest in IICS |
| None | Heterogeneous sources (Oracle + CSV) | All data moves to IICS RAM for processing |

### How to Enable:
Mapping Properties → Advanced tab → Pushdown Optimization → **Full**
(IICS auto-generates INSERT…SELECT SQL from visual mapping — developer does NOT write SQL manually)

### Generated SQL Example:
```sql
INSERT INTO FACT_POLICY_SUMMARY
SELECT p.policy_key, p.policy_number, p.product_type,
       COALESCE(c.total_claim_amt, 0), r.grade_code, r.rate_pct,
       p.sum_insured * r.rate_pct / 100 AS calculated_premium,
       CURRENT_TIMESTAMP(), '$$BATCH_ID'
FROM STG_POLICY p
LEFT JOIN (SELECT policy_number, SUM(claim_amount) AS total_claim_amt FROM STG_CLAIMS GROUP BY policy_number) c
  ON p.policy_number = c.policy_number
JOIN STG_RATE_GRADE r ON p.product_type = r.product_type
WHERE p.policy_status = 'ACTIVE'
AND CURRENT_DATE() BETWEEN r.eff_from_dt AND r.eff_to_dt;
```

---

## 📅 SCHEDULING IICS JOBS

### IICS Built-in Scheduler:
Administrator → Schedules → New Schedule → attach to Mapping Task
Supports: Once, Hourly, Daily, Weekly, Monthly, Cron expression (e.g., `0 2 * * 1-5` = Mon–Fri 2 AM)

### IICS Taskflow (with dependencies):
```
Phase 1 (PARALLEL):
  Mapping Task 1 (STG_POLICY load)  ─┐
  Mapping Task 2 (STG_CLAIMS load)  ─┼─► ALL must succeed ─► Mapping Task 4 (Business Logic)
  Mapping Task 3 (STG_RATE load)    ─┘                           → Command Task (SP)
                                                                  → Notification Task
```
Phase 1 runs simultaneously; Phase 2 starts only after ALL Phase 1 tasks succeed.

### External Schedulers (BFSI standard):
- **Control-M** — most common in BFSI/insurance; enterprise job scheduling
- **Apache Airflow** — Python DAG for cloud-native pipelines
- **Snowflake Tasks** — for pure Snowflake-to-Snowflake transformations (no IICS needed)
- **IICS REST API** — trigger via `POST /api/v2/job/start` (for integration with CI/CD)

---

## ✅ 9-STEP POST-LOAD DEVELOPER CHECKLIST

1. **Reconciliation** — COUNT(*) and SUM(premium) in FACT vs STG source; alert if >0.1% variance
2. **Update ETL Control Table** — set last_run_dt = CURRENT_TIMESTAMP(), rows_loaded, status = 'SUCCESS'
3. **Refresh Dynamic Tables / Materialized Views** — `ALTER DYNAMIC TABLE REFRESH;` — downstream BI aggregates update
4. **Truncate WORK table** — `TRUNCATE TABLE WORK_POLICY_SUMMARY;` — clean staging; keep STG 7 days for reprocessing
5. **SCD2 Verification** — `SELECT policy_number, COUNT(*) FROM FACT WHERE is_current='Y' GROUP BY 1 HAVING COUNT(*)>1` → must return 0 rows
6. **Apply/verify masking + row access policies** — column masking on sensitive fields; row access by business unit
7. **Cluster key maintenance** — `SYSTEM$CLUSTERING_INFORMATION('FACT_POLICY_SUMMARY')` — if DML_RATIO > 0.5, trigger manual reclustering
8. **Send load completion notification** — email + trigger BI tool refresh (Cognos / Power BI reload)
9. **Review REJECT_POLICY_LOG** — daily developer responsibility; escalate business rule exceptions; clear after review

---

## 🗄️ SNOWFLAKE SP — SCD TYPE 2 MERGE
```sql
CREATE OR REPLACE PROCEDURE SP_MERGE_POLICY_SUMMARY(p_batch_id STRING)
RETURNS STRING LANGUAGE SQL AS $
BEGIN
  -- Step 1: Expire changed records
  UPDATE FACT_POLICY_SUMMARY tgt
  SET eff_end_dt = CURRENT_DATE() - 1, is_current = 'N'
  WHERE tgt.is_current = 'Y'
  AND EXISTS (SELECT 1 FROM WORK_POLICY_SUMMARY src
    WHERE src.policy_number = tgt.policy_number
    AND src.reporting_year = tgt.reporting_year
    AND src.record_hash <> tgt.record_hash);
  -- Step 2: Insert new/changed records
  INSERT INTO FACT_POLICY_SUMMARY
  SELECT POLICY_KEY_SEQ.NEXTVAL, src.*, CURRENT_DATE(), '9999-12-31', 'Y', CURRENT_TIMESTAMP()
  FROM WORK_POLICY_SUMMARY src
  WHERE NOT EXISTS (SELECT 1 FROM FACT_POLICY_SUMMARY tgt
    WHERE tgt.policy_number = src.policy_number
    AND tgt.record_hash = src.record_hash AND tgt.is_current = 'Y');
  -- Step 3: Update control table
  UPDATE ETL_CONTROL_TABLE SET last_run_dt = CURRENT_TIMESTAMP(), status = 'SUCCESS'
  WHERE job_name = 'FACT_POLICY_SUMMARY_LOAD';
  RETURN 'MERGE COMPLETE';
END $;
```

---

## 📁 ALL REFERENCE FILES

| File | Content |
|---|---|
| `Saranya_IICS_Interview_Prep.html` | 3 complex IICS mappings (GL domain, Oracle+CSV→Snowflake) + Snowflake post-load + Q&A |
| `Saranya_IICS_2Mappings_Insurance.html` | 2 insurance mappings: JSON+Oracle→Snowflake (Claims) and Oracle+Oracle→Snowflake (Agent Premium) |
| `Saranya_IICS_Production_Issues.html` | 6 production issues with root cause, fix, and interview answer |
| `Saranya_IICS_STG_To_Target_Documentation.html` | Mapping 4: 3 STG tables→FACT_POLICY_SUMMARY + Pushdown + Scheduling + Post-load 9-step checklist |

---

_Reference document for Saranya's Fidelity IICS interview. All content from May 2026 interview prep session._
