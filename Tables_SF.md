# Snowflake Table Types in IICS (Interview Guide + How to Implement)

---

# 🧠 Overview

In Snowflake + IICS ETL development, choosing the right table type is critical for:

* Performance
* Cost optimization
* Data lifecycle management
* Incremental processing

This guide covers:

* Temporary Tables
* Transient Tables
* Streams
* Dynamic Tables
  With **how to create**, **IICS flow usage**, and **interview-ready answers**

---

# 🔹 1. Temporary Tables (TEMP)

## ✅ What it is

* Session-based table
* Automatically dropped after session ends

## 🛠️ How to Create

```sql
CREATE TEMPORARY TABLE temp_sales (
    id INT,
    amount NUMBER
);
```

## 🔄 How to Use in IICS Flow

1. Source → Read data (flat file / DB)
2. Expression / Filter / Join transformations
3. Target → Write into TEMP table (pre-SQL step or target table)
4. Use TEMP table for further transformation in same mapping

## 🎯 Real Use Case

* Intermediate transformation
* Deduplication before final load

## 🧪 Interview Answer

> “I use temporary tables in IICS when I need short-lived staging within a single mapping execution. For example, during complex joins or deduplication, I store intermediate results in a temp table and process further before loading into the final target.”

## ⚠️ Limitations

* Not reusable across sessions
* Data lost after execution

---

# 🔹 2. Transient Tables

## ✅ What it is

* Persistent table
* No Fail-safe (cost-saving)

## 🛠️ How to Create

```sql
CREATE TRANSIENT TABLE stg_sales (
    id INT,
    amount NUMBER
);
```

## 🔄 How to Use in IICS Flow

1. Source → Extract data
2. Target → Load into TRANSIENT table (staging layer)
3. Next mapping → Read from TRANSIENT
4. Apply transformations → Load into final table

## 🎯 Real Use Case

* Daily staging tables
* Reusable intermediate storage

## 🧪 Interview Answer

> “I use transient tables as staging tables in IICS pipelines. They persist across sessions but reduce storage cost since they don’t have fail-safe. This is ideal for non-critical intermediate data before loading into fact or dimension tables.”

## ⚠️ Limitations

* No fail-safe recovery
* Not suitable for critical data

---

# 🔹 3. Streams (CDC)

## ✅ What it is

* Tracks INSERT, UPDATE, DELETE changes

## 🛠️ How to Create

```sql
CREATE OR REPLACE STREAM sales_stream 
ON TABLE sales;
```

## 🔄 How to Use in IICS Flow

1. Create Stream on source table
2. IICS mapping → Source = STREAM
3. Read only changed records
4. Apply logic (UPSERT / MERGE)
5. Load into target table

## 🎯 Real Use Case

* Incremental loads
* CDC pipelines

## 🧪 Interview Answer

> “I use streams for CDC in Snowflake with IICS to capture incremental changes instead of full loads. This improves performance and reduces cost. I typically combine streams with merge logic to update target tables efficiently.”

## ⚠️ Key Considerations

* Must consume stream regularly
* Handles change tracking, not transformation

---

# 🔹 4. Dynamic Tables

## ✅ What it is

* Auto-refreshing table
* Replaces scheduled ETL jobs in some cases

## 🛠️ How to Create

```sql
CREATE OR REPLACE DYNAMIC TABLE sales_summary
TARGET_LAG = '5 minutes'
WAREHOUSE = compute_wh
AS
SELECT region, SUM(amount) AS total_sales
FROM sales
GROUP BY region;
```

## 🔄 How to Use in IICS Flow

Option 1 (Replace ETL):

* No IICS job needed
* Snowflake auto-refreshes data

Option 2 (Hybrid):

1. IICS loads base table
2. Dynamic table auto-aggregates

## 🎯 Real Use Case

* Near real-time dashboards
* Aggregation layers

## 🧪 Interview Answer

> “I use dynamic tables when I want to reduce ETL dependency and enable near real-time data processing. Instead of scheduling jobs in IICS, I let Snowflake automatically refresh transformed data.”

## ⚠️ Limitations

* Less control than ETL workflows
* Not ideal for complex transformations

---

# ⚔️ Comparison Table

| Feature     | Temp           | Transient        | Streams          | Dynamic Tables |
| ----------- | -------------- | ---------------- | ---------------- | -------------- |
| Persistence | No             | Yes              | Yes              | Yes            |
| Cost        | Low            | Lower            | Low              | Medium         |
| Use Case    | Intermediate   | Staging          | CDC              | Auto pipeline  |
| IICS Role   | Within mapping | Between mappings | Incremental load | Replace ETL    |
| Recovery    | No             | No fail-safe     | N/A              | Yes            |

---

# 🧠 Final Interview Answer (Best Version)

> “In Snowflake with IICS, I select table types based on data lifecycle. For short-lived transformations within a mapping, I use temporary tables. For reusable staging with cost optimization, I use transient tables. For incremental processing, I use streams to capture CDC and avoid full loads. And for reducing ETL complexity or enabling near real-time pipelines, I use dynamic tables instead of scheduled IICS workflows.”

---

# 🔥 Pro-Level Answer (Standout)

> “In my projects, I typically combine streams and transient tables to build efficient CDC pipelines. Streams capture incremental changes, and transient tables act as staging before merging into final fact tables. This improves both performance and cost efficiency.”

---

# 🚫 Common Mistakes (Avoid in Interview)

* Saying “Dynamic tables replace all ETL” ❌
* Using transient tables for critical data ❌
* Not explaining how streams are consumed ❌
* Giving only definitions without real ETL scenarios ❌

---

# ✅ Quick Memory Trick

* TEMP → Session only
* TRANSIENT → Staging (cheap)
* STREAM → Incremental (CDC)
* DYNAMIC → Auto ETL

---

If you want next:

* Mock interview questions on this topic
* Real project-based answers (Fidelity-style)
* Diagram of full ETL architecture

Just ask 👍
