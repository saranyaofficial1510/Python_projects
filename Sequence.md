# IICS — Sequence Generator: Complete Guide

## What is a Sequence Generator?

A **Sequence Generator** is an IICS transformation that produces unique, auto-incrementing numbers (1, 2, 3...). It is used to create **Surrogate Keys (SK)** — artificial IDs that don't come from the source data.

---

## When to Use Sequence Generator in IICS

| Scenario | Need Sequence? | Reason |
|----------|---------------|--------|
| Target has a **Surrogate Key** column | **YES** | Source business keys may not be unique across systems |
| Loading **Dimension** tables (SCD Type 1/2) | **YES** | Each new/changed row needs a unique SK |
| Loading **Fact** tables (SK column) | **YES** | Fact table needs its own unique row ID |
| Loading **Fact** tables (FK columns) | **NO** | FK values come from Dimension Lookup, not sequences |
| **Multiple sources** (Oracle + XML + Flat File) merging into one target | **YES** | Business keys can clash (Oracle ID=1001, XML ID=1001) |
| **XML source** with no reliable unique key | **YES** | XML records often lack unique identifiers |
| Simple **staging/landing** table load | **NO** | Raw dump, no SK needed |
| Source already has a **globally unique ID** | **NO** | Reuse it directly |

---

## Where in the Mapping?

The Sequence Generator goes **AFTER Lookup** (to check if row exists) and **BEFORE Target** (to assign the SK).

```
SQ_SOURCE
    │
    ▼
LKP_CHECK_EXISTS ← (Does this row already exist in target?)
    │
    ├── NEW ROW ──→ SEQ_GEN ──→ Assigns sk_id = 1, 2, 3...
    │                                │
    ├── EXISTING ROW ──────────→ Keep old sk_id from Lookup
    │                                │
    └────────────────┬───────────────┘
                     ▼
                TGT_TABLE
```

---

## Real-World Example: Oracle + XML → Snowflake

### Problem
- Oracle `customer_id = 1001` (John)
- XML `order_id = 1001` (Order for John)
- Same number `1001` but **different meaning** — cannot use as target key!

### Solution with Sequence Generator

**Mapping 1: Load Dimension (Customers from Oracle)**

```
SQ_ORACLE (customers)
    │
    ▼
LKP_DIM_CUSTOMER (check if customer exists in target)
    │
    ├── NEW → SEQ_CUSTOMER_SK → generates sk_customer_id = 1, 2, 3...
    │
    ├── EXISTING → keep old sk_customer_id from LKP
    │
    └──→ TGT_DIM_CUSTOMER
```

**Result in target:**

| sk_customer_id (from SEQ) | customer_id (from Oracle) | name |
|---------------------------|---------------------------|------|
| 1 | 1001 | John |
| 2 | 1002 | Jane |

**Mapping 2: Load Fact (Orders from XML)**

```
SQ_XML (orders)
    │
    ▼
LKP_DIM_CUSTOMER (get sk_customer_id for this customer name)
    │
    ▼
SEQ_ORDER_SK → generates sk_order_id = 1, 2, 3...
    │
    ▼
TGT_FACT_ORDERS
```

**Result in target:**

| sk_order_id (from SEQ) | order_id (from XML) | fk_customer_id (from LKP) | amount |
|------------------------|---------------------|---------------------------|--------|
| 1 | 1001 | 1 (John's SK) | 5000 |
| 2 | 1002 | 2 (Jane's SK) | 3000 |

---

## Sequence Generator — Transformation Properties in IICS

| Property | Description | Typical Value |
|----------|-------------|---------------|
| **Start Value** | First number to generate | `1` or `MAX(sk_id)+1` from target |
| **Increment By** | Gap between numbers | `1` |
| **End Value** | Maximum number (optional) | Leave blank for unlimited |
| **Reset** | When to restart counting | Per mapping run or continuous |
| **Cycle** | Restart from Start Value after reaching End Value | Usually NO |
| **Output Port** | The port name that carries the generated number | `NEXTVAL` |

### How to Configure in IICS Mapping Designer

1. Drag **Sequence Generator** transformation into mapping
2. Set **Start Value** = 1, **Increment** = 1
3. Connect the `NEXTVAL` output port to the target's SK column
4. For incremental loads: set Start Value = `MAX(sk_id) + 1` (use a pre-session SQL or parameter)

---

## Common Patterns in ETL Flows

### Pattern 1: Initial Load (Full Load)

```
SQ_SOURCE → SEQ_GEN (start=1) → TGT_DIM
```

All rows are new. Sequence starts at 1.

### Pattern 2: Incremental Load (Daily)

```
SQ_SOURCE → LKP_TARGET → RTR (Router)
                              │
                    ┌─────────┴─────────┐
                    NEW                 EXISTING
                    │                   │
              SEQ_GEN (start=           UPD (update
              MAX(sk)+1)                existing row)
                    │                   │
                    └─────────┬─────────┘
                              ▼
                         TGT_DIM
```

### Pattern 3: SCD Type 2 (History Tracking)

```
SQ_SOURCE → LKP_TARGET → RTR (Router)
                              │
              ┌───────────────┼───────────────┐
              NEW             CHANGED          UNCHANGED
              │               │                │
         SEQ_GEN         SEQ_GEN (new row)    (skip)
              │          + UPD (expire old)
              │               │
              └───────┬───────┘
                      ▼
                 TGT_DIM (INSERT new version)
```

For SCD2, even CHANGED rows get a **new sequence value** because a new row version is inserted.

### Pattern 4: Multiple Sources into One Target

```
SQ_ORACLE ──┐
             ├──→ UNION ──→ SEQ_GEN ──→ TGT_COMBINED
SQ_XML ─────┘
```

Sequence ensures every row from every source gets a unique SK regardless of source.

---

## Sequence in SQL (Snowflake) vs IICS

| Concept | Snowflake SQL | IICS |
|---------|--------------|------|
| Create sequence | `CREATE SEQUENCE seq1` | Sequence Generator transformation |
| Get next value | `seq1.NEXTVAL` | `NEXTVAL` output port |
| Same value in 2 columns | Subquery: `SELECT a, a FROM (SELECT seq.NEXTVAL a)` | Connect same `NEXTVAL` port to multiple target columns |
| Reset numbering | `ALTER SEQUENCE seq1 SET START = 1` | Set Start Value property |
| Use in mapping | SQL Override in SQ | Drag-and-drop transformation |

---

## Interview-Ready Answer (2 Lines)

> **"A Sequence Generator in IICS creates unique auto-incrementing surrogate keys for dimension and fact tables, ensuring every row has a unique identifier independent of source business keys."**

> **"It's placed after the Lookup (to check if row is new) and before the Target — only NEW rows get the next sequence value; existing rows keep their old SK from the Lookup."**

---

## Quick Reference: Do I Need a Sequence Here?

```
Loading dimension table?          → YES (SK for each row)
Loading fact table SK?            → YES (unique fact row ID)
Loading fact table FK?            → NO  (comes from dimension LKP)
Merging Oracle + XML + files?     → YES (avoid key clashes)
Simple staging load?              → NO  (raw dump)
Source has global unique ID?      → NO  (reuse it)
SCD Type 2 new version?          → YES (new row = new SK)
```
