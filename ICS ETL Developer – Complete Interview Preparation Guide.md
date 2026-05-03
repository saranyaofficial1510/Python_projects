# IICS ETL Developer – Complete Interview Preparation Guide
> MCQ + Scenario-Based Questions with Answers | Round 1 & Round 2 Ready

---

## TABLE OF CONTENTS

1. [IICS Platform & Architecture](#1-iics-platform--architecture)
2. [Mappings & Transformations](#2-mappings--transformations)
3. [Lookup Transformation](#3-lookup-transformation)
4. [Expression Transformation](#4-expression-transformation)
5. [Aggregator Transformation](#5-aggregator-transformation)
6. [Joiner Transformation](#6-joiner-transformation)
7. [Filter & Router Transformation](#7-filter--router-transformation)
8. [Update Strategy & Error Handling](#8-update-strategy--error-handling)
9. [Slowly Changing Dimensions (SCD)](#9-slowly-changing-dimensions-scd)
10. [Sequence Generator & Sorter](#10-sequence-generator--sorter)
11. [Taskflows & Workflows](#11-taskflows--workflows)
12. [Performance Tuning](#12-performance-tuning)
13. [Data Quality & Validation](#13-data-quality--validation)
14. [SQL & Source/Target Concepts](#14-sql--sourcetarget-concepts)
15. [Scheduling, Monitoring & Logging](#15-scheduling-monitoring--logging)
16. [Cloud & Connector Concepts](#16-cloud--connector-concepts)

---

## 1. IICS Platform & Architecture

### Q1. What is IICS and what are its main service components?
**Answer:**  
IICS (Informatica Intelligent Cloud Services) is a cloud-native data integration platform. Its main components are:
- **Cloud Data Integration (CDI)** – ETL/ELT mappings and tasks
- **Cloud Data Quality (CDQ)** – Data profiling and cleansing
- **Cloud Mass Ingestion** – Bulk data movement from databases/files
- **Application Integration (CAI)** – API and real-time integration
- **Cloud Data Governance & Catalog** – Metadata and lineage management

> **Key Point:** All services run through the **Informatica Intelligent Cloud Services (IICS) org** and communicate via the **Secure Agent**.

---

### Q2. What is a Secure Agent and why is it required?
**Answer:**  
A Secure Agent is a lightweight program installed on-premises or in a cloud VM. It:
- Acts as a bridge between IICS cloud and local/on-prem data sources
- Executes mapping jobs locally (data never leaves the network)
- Enables connectivity to databases, file systems, and APIs behind firewalls

> **Trick MCQ:** "Which component executes the actual data processing in IICS?" → **Secure Agent**, NOT the IICS cloud server itself.

---

### Q3. What is the difference between a Mapping and a Mapping Task in IICS?
**Answer:**  

| Concept | Description |
|---|---|
| **Mapping** | The design-time canvas that defines source, transformations, and target logic |
| **Mapping Task** | The runtime configuration that binds a mapping to actual connections and schedules |

> A mapping alone cannot run — you need a **Mapping Task** to execute it. One mapping can have **multiple mapping tasks** pointing to different environments (Dev, QA, Prod).

---

### Q4. What are the deployment modes available in IICS mappings?
**Answer:**  
- **Advanced Mode** – Runs on the Spark engine (supports complex transformations, pushdown optimization)
- **Standard Mode** – Runs on the Secure Agent directly (simpler mappings, row-by-row processing)

> **MCQ Trap:** Advanced mode supports features like **Mapplets**, **Spark SQL**, and **schema drift** — Standard mode does NOT support all of these.

---

## 2. Mappings & Transformations

### Q5. In what order should transformations be placed to implement SCD Type 2 in an IICS mapping?
**Answer (Correct Order):**

```
Source → Lookup (check existing record) → Expression (flag: INSERT/UPDATE/History)
       → Router (route to New / Changed / Unchanged)
       → Update Strategy (DD_INSERT for new, DD_UPDATE for expire old)
       → Target (Dimension Table)
```

**Detailed Steps:**
1. **Source Qualifier** – Read incoming data
2. **Lookup** – Look up existing record in dimension table using business key
3. **Expression** – Compare incoming vs existing values, generate `EFF_START_DATE`, `EFF_END_DATE`, `CURRENT_FLAG`, `SURROGATE_KEY`
4. **Router** – Route rows: New record / Changed record / No change
5. **Update Strategy** – Mark old row as `DD_UPDATE` (expire it), new row as `DD_INSERT`
6. **Target** – Write to dimension table

---

### Q6. Which mapping scenario is valid and will produce correct output?
**Example MCQ:**  
> Which of the following is a valid IICS mapping flow?  
> A) Source → Target (no transformation)  
> B) Source → Aggregator → Lookup → Target  
> C) Source → Expression → Filter → Target  
> D) Source → Joiner (with no second source) → Target  

**Answer: C**  
- **A** – Valid but no transformation
- **B** – Invalid: Aggregator after Lookup is problematic; Aggregator should come before Lookup
- **C** ✅ Valid: Expression then Filter is a standard pattern
- **D** – Invalid: Joiner requires TWO input streams

---

### Q7. How many rows will come out of an Aggregator transformation?
**Answer:**  
The Aggregator outputs **one row per group**. If no GROUP BY port is defined, it outputs **exactly 1 row** (a grand total).

> **Example:**  
> Input: 100 rows with 5 unique departments → Aggregator (GROUP BY dept) → Output: **5 rows**

---

## 3. Lookup Transformation

### Q8. How do you configure a Lookup transformation to return multiple rows?
**Answer:**  
By default, Lookup returns only **one row** (the first match). To return multiple rows:
- Set **"Return All Rows"** option = **Yes** (also called "Lookup policy on multiple match" = **Return All Values**)
- This converts the Lookup into an **Unconnected Lookup** behavior or requires using it in **Advanced mode**

> **In Standard Mode:** You cannot return multiple rows from a connected lookup. Use a **Joiner** transformation instead to get all matching rows.

> **MCQ Answer:** To return multiple matching rows → Set **"Return All Rows" = Yes** OR use a **Joiner** transformation.

---

### Q9. Which option gives the best performance in a Lookup transformation?
**Answer: Cache the Lookup**

| Option | Performance |
|---|---|
| **Lookup Cache = Yes (Static Cache)** | ✅ Best – data loaded to memory once |
| **Lookup Cache = No (No Cache)** | ❌ Worst – DB hit for every input row |
| **Dynamic Cache** | Medium – updates cache as rows process |
| **Persistent Cache** | Good – reuses cache across runs |

> **Rule of Thumb:** Always use **Static Cache** when the lookup table doesn't change during the mapping run. This avoids repeated database calls.

---

### Q10. What is the correct SQL override syntax for a Lookup transformation to filter only active records?
**Answer:**  
```sql
SELECT CUST_ID, CUST_NAME, STATUS
FROM CUSTOMER
WHERE STATUS = 'ACTIVE'
```

> **Important Rules for Lookup SQL Override:**
> - Must include all **lookup ports** used in the mapping
> - Must include the **lookup condition column** (e.g., CUST_ID)
> - Cannot use `ORDER BY` in standard lookup SQL override
> - The WHERE clause filters what gets **cached** (not filtered row-by-row)

**Wrong syntax (MCQ trap):**
```sql
-- WRONG: Missing lookup port column
SELECT CUST_NAME FROM CUSTOMER WHERE STATUS = 'ACTIVE'
-- This will error because CUST_ID (the lookup key) is missing
```

---

### Q11. What is the difference between Connected and Unconnected Lookup?

| Feature | Connected Lookup | Unconnected Lookup |
|---|---|---|
| Called by | Data flow (pipeline) | `:LKP.lookup_name()` expression |
| Returns | Multiple output ports | Only ONE return port |
| Multiple matches | Can configure policy | Returns first match only |
| Usage | Standard ETL flow | Reusable across expressions |

> **MCQ:** "Which lookup type can be called from an Expression transformation?" → **Unconnected Lookup**

---

## 4. Expression Transformation

### Q12. Choose the valid expression syntax in an Informatica Expression transformation
**Example MCQ Options:**  
> A) `IIF(SALARY > 5000, 'HIGH', 'LOW')`  
> B) `IF SALARY > 5000 THEN 'HIGH' ELSE 'LOW'`  
> C) `CASE WHEN SALARY > 5000 THEN 'HIGH'`  
> D) `IIF(SALARY > 5000 'HIGH' 'LOW')`  

**Answer: A**  
- **A** ✅ Correct IICS expression syntax using `IIF`
- **B** ❌ SQL/procedural syntax – not valid in IICS expression editor
- **C** ❌ Incomplete CASE expression (missing ELSE and END)
- **D** ❌ Missing commas between arguments

**More valid expression examples:**
```
-- String concat
CONCAT(FIRST_NAME, ' ', LAST_NAME)

-- Null check
IIF(ISNULL(PHONE), 'N/A', PHONE)

-- Date format
TO_CHAR(SYSDATE, 'YYYY-MM-DD')

-- String to Number
TO_INTEGER(EMP_ID)

-- Substring
SUBSTR(FULL_NAME, 1, 5)
```

---

### Q13. Which built-in functions are available in IICS Expression transformation?
**Answer (Key Categories):**

| Category | Functions |
|---|---|
| String | `CONCAT`, `SUBSTR`, `LTRIM`, `RTRIM`, `UPPER`, `LOWER`, `LENGTH`, `INSTR`, `REPLACE` |
| Numeric | `ROUND`, `TRUNC`, `MOD`, `ABS`, `CEIL`, `FLOOR`, `POWER`, `SQRT` |
| Date | `SYSDATE`, `ADD_TO_DATE`, `DATE_DIFF`, `TO_DATE`, `TO_CHAR`, `LAST_DAY` |
| Conditional | `IIF`, `DECODE`, `NVL`, `ISNULL`, `IS_NUMBER`, `IS_DATE` |
| Conversion | `TO_CHAR`, `TO_INTEGER`, `TO_DECIMAL`, `TO_FLOAT`, `TO_DATE` |

---

## 5. Aggregator Transformation

### Q14. What happens if you use an Aggregator without a Group By port?
**Answer:**  
The Aggregator treats **all input rows as one group** and returns a **single output row** with the aggregate result (e.g., total SUM, MAX, COUNT of all rows).

---

### Q15. What is the difference between SUM in Expression vs Aggregator?
**Answer:**  

| Transformation | SUM Behavior |
|---|---|
| **Expression** | No SUM function – row-by-row only |
| **Aggregator** | SUM across multiple rows – group-level |

> **MCQ:** "Which transformation would you use to calculate total salary per department?" → **Aggregator**

---

### Q16. Can you use a Filter transformation before or after Aggregator? Which is better?
**Answer:**  
**Before Aggregator is BETTER** for performance.

- **Filter → Aggregator**: Fewer rows processed in aggregation → **faster**
- **Aggregator → Filter**: All rows aggregated first, then filtered → **slower, more memory used**

---

## 6. Joiner Transformation

### Q17. What is the difference between Joiner and Lookup transformation?

| Feature | Joiner | Lookup |
|---|---|---|
| Join type | Inner, Left Outer, Right Outer, Full Outer | Equi-join (left outer by default) |
| Data sources | Two separate pipelines | One main + one lookup table |
| Returns multiple rows | Yes | Only with "Return All Rows" |
| Performance | Slower (sorts both datasets) | Faster with caching |
| Use case | Two active data streams | Dimension/reference table lookup |

---

### Q18. Which source should be the "Master" and which should be the "Detail" in a Joiner?
**Answer:**  
- **Master Source** = The **smaller** dataset (loaded into memory/cache)
- **Detail Source** = The **larger** dataset (streamed row by row)

> **Performance Rule:** Always set the smaller table as Master to minimize memory usage.

---

## 7. Filter & Router Transformation

### Q19. What is the difference between Filter and Router transformation?

| Feature | Filter | Router |
|---|---|---|
| Groups | One condition – passes or rejects | Multiple groups with conditions |
| Default group | No | Yes – catches unmatched rows |
| Output ports | Single stream | Multiple output streams |
| Use case | Simple include/exclude | Route to different targets |

---

### Q20. In a Router transformation, what happens to rows that don't match any group condition?
**Answer:**  
They are routed to the **Default Group**. If no transformation/target is connected to the Default group, those rows are **dropped (discarded)**.

> **MCQ:** "Where do unmatched rows go in a Router?" → **Default Group**

---

## 8. Update Strategy & Error Handling

### Q21. How do you implement custom error handling in Update Strategy transformation?
**Answer:**  

The Update Strategy transformation uses these flags:

| Flag | Value | Action |
|---|---|---|
| `DD_INSERT` | 0 | Insert row |
| `DD_UPDATE` | 1 | Update row |
| `DD_DELETE` | 2 | Delete row |
| `DD_REJECT` | 3 | Reject row (send to reject file) |

**Custom Error Handling Example:**
```
IIF(ISNULL(EMP_ID), DD_REJECT,
    IIF(FLAG = 'NEW', DD_INSERT,
        IIF(FLAG = 'UPD', DD_UPDATE, DD_REJECT)))
```

> **To capture rejected rows:**  
> Enable **"Reject File"** on the Target and connect the reject output. Rejected rows go to a flat file for review.

---

### Q22. What must be enabled on the Target for Update Strategy to work?
**Answer:**  
On the Target transformation, set:
- **"Update Strategy" = Insert/Update/Delete** (choose appropriate)
- For UPDATE: Enable **"Update else Insert"** if you want upsert behavior
- The **primary key** column must be mapped for UPDATE/DELETE to identify the row

---

## 9. Slowly Changing Dimensions (SCD)

### Q23. What is the correct order of transformations to implement SCD Type 1?
**Answer:**  
```
Source → Lookup (find existing record by business key)
       → Expression (compare old vs new values)
       → Filter (only process changed records)
       → Update Strategy (DD_UPDATE)
       → Target (overwrite old value)
```

> SCD Type 1 = **Overwrite** – no history kept. Just update the record.

---

### Q24. What is the correct order for SCD Type 2?
**Answer:**  
```
Source → Lookup (match by business key)
       → Expression (generate: IS_CHANGED flag, EFF_START_DATE, EFF_END_DATE, CURRENT_FLAG, SURROGATE_KEY via Sequence Generator)
       → Router:
           Group 1 (New records)     → Update Strategy (DD_INSERT) → Target
           Group 2 (Changed records) → Update Strategy (DD_UPDATE to expire old row) → Target
                                     → Update Strategy (DD_INSERT for new version) → Target
           Group 3 (No change)       → Discard
```

**Key SCD Type 2 columns:**

| Column | New Record | Expired Record |
|---|---|---|
| `EFF_START_DATE` | Today's date | Unchanged |
| `EFF_END_DATE` | `9999-12-31` | Yesterday's date |
| `CURRENT_FLAG` | `Y` | `N` |
| `SURROGATE_KEY` | New sequence | Unchanged |

---

### Q25. What is SCD Type 3 and when is it used?
**Answer:**  
SCD Type 3 keeps **limited history** by adding a column for the previous value.

| Column | Description |
|---|---|
| `CURRENT_CITY` | Current value |
| `PREVIOUS_CITY` | One version back |
| `CHANGE_DATE` | When it changed |

> **Limitation:** Only tracks **one** previous value. Not suitable for multiple historical changes.

---

## 10. Sequence Generator & Sorter

### Q26. What is the Sequence Generator transformation used for?
**Answer:**  
Used to generate **unique numeric values** (surrogate keys). Key properties:

| Property | Description |
|---|---|
| `Start Value` | First number to generate |
| `Increment By` | Step size (default: 1) |
| `End Value` | Maximum value |
| `Cycle` | Restart from Start Value after End Value |
| `Reset` | Reset each run (useful for batch IDs) |

> **Output Ports:** `NEXTVAL` (next value) and `CURRVAL` (current value)  
> **MCQ:** "Which port of Sequence Generator gives the actual value to use?" → **NEXTVAL**

---

### Q27. What does the Sorter transformation do and when is it required?
**Answer:**  
Sorter sorts rows by specified columns in ascending or descending order. It is **required before:**
- **Aggregator** (if "Sorted Input" = Yes for performance)
- **Joiner** (sorted input optimization)
- Any transformation requiring sorted data

> **"Sorted Input" in Aggregator:** If enabled, Aggregator expects data pre-sorted by Group By keys → much faster, uses less memory.

---

## 11. Taskflows & Workflows

### Q28. What is the difference between a Mapping Task and a Taskflow?
**Answer:**  

| Component | Description |
|---|---|
| **Mapping Task** | Executes a single mapping |
| **Taskflow** | Orchestrates multiple tasks with conditions, loops, and branching |

> Taskflow is like a **workflow** – it can contain Mapping Tasks, Command Tasks, Notification Tasks, and conditional branching (If/Else, Loops).

---

### Q29. How do you pass parameters between tasks in a Taskflow?
**Answer:**  
Using **Taskflow Variables**:
1. Define output variable in Task 1 (e.g., `ROW_COUNT`)
2. Map it to input variable of Task 2
3. Use `$taskflow.VAR_NAME` syntax to reference

---

### Q30. What types of tasks can be included in a Taskflow?
**Answer:**  
- **Mapping Task** – Run an ETL mapping
- **Command Task** – Execute a shell command or script
- **Notification Task** – Send email alerts
- **Data Transfer Task** – Move files (SFTP, S3)
- **Sub-taskflow** – Nest one taskflow inside another
- **Decision/Condition** – If/Else branching based on variable values

---

## 12. Performance Tuning

### Q31. Which of the following improves Lookup transformation performance the most?
**MCQ Options:**  
> A) Increasing source fetch size  
> B) Enabling Lookup Cache  
> C) Adding more output ports  
> D) Using a Router before Lookup  

**Answer: B – Enabling Lookup Cache**

> Cache loads the entire lookup table into memory once. Without cache, every input row triggers a **database query** → extremely slow for large datasets.

---

### Q32. What is Pushdown Optimization (PDO) in IICS?
**Answer:**  
PDO pushes the transformation logic down to the **source database** engine (using SQL) instead of pulling data to the Secure Agent for processing.

| Type | Description |
|---|---|
| **Source PDO** | Pushes Source Qualifier logic to source DB |
| **Target PDO** | Pushes transformation to target DB |
| **Full PDO** | Entire mapping runs in DB (no Secure Agent processing) |

> **Benefit:** Less data movement, leverages DB engine power.  
> **Limitation:** Not all transformations support PDO (e.g., Joiner with heterogeneous sources does not).

---

### Q33. How do you improve performance of a mapping that processes millions of rows?
**Answer (Best Practices):**
1. **Enable caching** on Lookup transformations
2. Use **Sorted Input** on Aggregator (pre-sort with Sorter)
3. Use **Pushdown Optimization** where possible
4. **Filter early** – reduce rows before heavy transformations
5. Set **Master as smaller dataset** in Joiner
6. Partition the source read (parallel processing)
7. Use **bulk loading** on target (disable constraints temporarily)
8. Avoid **SELECT *** – only pull required columns

---

## 13. Data Quality & Validation

### Q34. How do you handle duplicate records in IICS mappings?
**Answer:**  
- Use **Aggregator** with GROUP BY on all key columns → `COUNT(*)` to detect duplicates
- Use **Sorter** + **Expression** (compare current row with previous row using `PREV_KEY` variable)
- Use **Router** to route duplicates to a separate reject target

---

### Q35. How do you validate that a column value is not null before loading?
**Answer:**  
Use **Filter transformation:**
```
NOT ISNULL(EMP_ID) AND NOT ISNULL(EMP_NAME)
```
Or use **Update Strategy** with `DD_REJECT` for null records:
```
IIF(ISNULL(EMP_ID), DD_REJECT, DD_INSERT)
```

---

## 14. SQL & Source/Target Concepts

### Q36. What is a Source Qualifier and what can it do?
**Answer:**  
Source Qualifier represents the SQL query that reads data from a relational source. It can:
- Add **WHERE clause** to filter source rows
- Add **ORDER BY** to sort
- Write a **custom SQL override** to join multiple tables at source
- Define **Distinct** to avoid duplicates

**Custom SQL Override Example:**
```sql
SELECT E.EMP_ID, E.EMP_NAME, D.DEPT_NAME
FROM EMPLOYEE E
JOIN DEPARTMENT D ON E.DEPT_ID = D.DEPT_ID
WHERE E.STATUS = 'ACTIVE'
ORDER BY E.EMP_ID
```

---

### Q37. What is the correct SQL override for a Lookup to return only active records?
**Answer:**
```sql
SELECT CUST_ID, CUST_NAME, EMAIL
FROM CUSTOMER
WHERE IS_ACTIVE = 'Y'
```
> **Rules:**
> - All ports used in the Lookup must appear in SELECT
> - No ORDER BY allowed in Lookup SQL override
> - WHERE clause filters the **cached data**, not row-by-row

---

### Q38. How do you implement an incremental load (delta load) in IICS?
**Answer:**  
1. Add a **watermark column** (e.g., `LAST_MODIFIED_DATE`) to source table
2. Store last successful run timestamp in a **parameter file** or control table
3. In Source Qualifier, add WHERE clause:
   ```sql
   WHERE LAST_MODIFIED_DATE > TO_DATE('$$LAST_RUN_DATE', 'YYYY-MM-DD HH24:MI:SS')
   ```
4. After successful run, **update the watermark** in control table

---

## 15. Scheduling, Monitoring & Logging

### Q39. How do you schedule a Mapping Task in IICS?
**Answer:**  
1. Open the **Mapping Task** in IICS
2. Go to **Schedule** tab
3. Set Frequency: Hourly / Daily / Weekly / Monthly / Custom (Cron)
4. Set Start Date/Time and Time Zone
5. Optionally configure **email notifications** on Success/Failure

> **Cron Example:** `0 2 * * *` = Run at 2:00 AM every day

---

### Q40. How do you monitor a running job in IICS?
**Answer:**  
Go to **Monitor → My Jobs** (or **Activity Monitor**):
- View **status**: Running / Success / Failed / Stopped
- See **row statistics**: rows read, written, rejected, affected
- View **session logs** for error details
- Check **error rows** in reject file

---

### Q41. What is the purpose of the Session Log and how do you access it?
**Answer:**  
The Session Log captures:
- Row counts per transformation
- Errors and warnings
- SQL queries generated (if verbose)
- Start/End time and duration

**Access:** Monitor → Select job → **View Session Log**

---

## 16. Cloud & Connector Concepts

### Q42. What connectors are commonly used in IICS?
**Answer:**

| Connector | Use Case |
|---|---|
| **Salesforce** | CRM data integration |
| **SAP** | ERP data extraction |
| **Snowflake** | Cloud data warehouse |
| **Amazon S3** | File-based cloud storage |
| **Azure Data Lake** | Cloud storage for analytics |
| **REST/SOAP API** | Web service integration |
| **Oracle / SQL Server** | On-prem relational DB |
| **Flat File** | CSV, fixed-width file processing |

---

### Q43. What is the difference between Cloud Data Integration (CDI) and Cloud Mass Ingestion?
**Answer:**  

| Feature | CDI | Mass Ingestion |
|---|---|---|
| Purpose | ETL/ELT with transformation logic | Bulk replication of tables (no transformation) |
| Transformation | Full transformation support | Minimal – column mapping only |
| Use case | Complex data integration | DB-to-DB full/incremental replication |
| Speed | Moderate | Very fast (bulk operations) |

---

### Q44. How do you handle schema drift in IICS Advanced Mode mappings?
**Answer:**  
Enable **Schema Handling** options:
- **Allow Schema Drift** – Mapping does not fail if source adds new columns
- **New Fields Behavior** – Ignore, pass through, or map dynamically
- Use **Dynamic Mapping** with field rules to auto-map new columns

---

## QUICK REVISION – KEY MCQ FACTS

| Question | Answer |
|---|---|
| Best performance in Lookup | **Enable Static Cache** |
| Return multiple rows in Lookup | **Return All Rows = Yes** or use Joiner |
| Correct lookup SQL override | SELECT must include lookup key column |
| SCD Type 2 transformation order | Source → Lookup → Expression → Router → Update Strategy → Target |
| Update Strategy: Reject rows | `DD_REJECT` (value = 3) |
| How many rows from Aggregator (no GROUP BY) | **1 row** |
| Rows from Aggregator (5 groups) | **5 rows** |
| Connected vs Unconnected Lookup | Unconnected called via `:LKP.name()` |
| Joiner Master side | **Smaller** dataset |
| Filter vs Router | Filter = one stream; Router = multiple streams |
| Sequence Generator port to use | **NEXTVAL** |
| Best for incremental load | Watermark / High-water mark pattern |
| Valid expression syntax | `IIF(condition, true_val, false_val)` |
| Pushdown Optimization benefit | Reduces data movement, uses DB engine |
| Secure Agent role | Executes jobs, bridges IICS cloud to on-prem data |

---

*Prepared for IICS ETL Developer Interview – Round 1 (MCQ) and Round 2 (Scenario-Based)*
