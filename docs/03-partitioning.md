# 3 — Partitioning and Data Modeling

The single most important rule in table design:

> **The fastest query is the one that never reads data off disk.**
> Partitioning is how you arrange files so engines can *exclude* them
> without opening them.

## 3.1 Hidden partitioning vs the Hive way

### Old approach (Hive)

You add **synthetic partition columns** to your data and manage them yourself:

```sql
CREATE TABLE trips (..., pickup_ts TIMESTAMP)
PARTITIONED BY (year STRING, month STRING);   -- fake columns

INSERT ... VALUES (..., '2023', '08');        -- you compute these
SELECT ... WHERE year='2023' AND month='08';  -- you must remember the fake columns
```

Painful and error-prone.

### Iceberg way (hidden partitioning)

You declare a **transform function** on an existing column:

```sql
CREATE TABLE trips (...) USING iceberg
PARTITIONED BY (months(pickup_ts));
```

Iceberg **automatically** applies `months()` to compute the partition value.
You:

- Never add fake columns.
- Never change your queries — you still filter on the **original column**:

```sql
WHERE pickup_ts >= '2023-08-01' AND pickup_ts < '2023-09-01'
```

Iceberg recognizes the predicate matches the `months()` transform and prunes
the other months automatically.

## 3.2 Transform functions

| Transform | Output | Use case |
|-----------|--------|----------|
| `years(col)` | Year number | Multi-year datasets |
| `months(col)` | **Month offset from 1970-01** | Monthly data |
| `days(col)` | Day offset | Daily logs |
| `hours(col)` | Hour offset | High-frequency logs |
| `bucket(N, col)` | Hash into N buckets | High-cardinality IDs (user_id, order_id) |
| `truncate(W, col)` | Truncate to width W | Strings, URLs |

### Decoding `months()` output

`months()` returns the **number of months since January 1970** (epoch):

```
year  = 1970 + (value ÷ 12)
month = (value mod 12) + 1
```

Examples from the NYC Taxi run:

| Value | Decodes to | Meaning |
|-------|-----------|---------|
| `641` | 2023-06 | June 2023 (real data) |
| `642` | 2023-07 | July 2023 |
| `643` | 2023-08 | August 2023 |
| `395` | 2002-12 | **bad data** (timestamp error) |
| `467` | 2008-12 | **bad data** |

These integers are **internal only** — you never type them in queries.

## 3.3 The 4 strategies compared

| Strategy | Partition spec | Sort | Typical file size | Best for |
|----------|----------------|------|-------------------|----------|
| **Unpartitioned** | none | none | ~16 MB | Full-table scans |
| **Monthly** | `months(ts)` | none | ~16 MB | Time-range queries |
| **Daily** | `days(ts)` | none | ~1.8 MB ⚠️ | Single-day queries |
| **Sorted** | `months(ts)` | `PULocationID` | ~16 MB | Location queries |

All four contain **the exact same 15,407,558 rows** — only the physical
layout differs.

### Why daily is a trap (the small files problem)

Daily partitioning of 5 months produces ~150 files of ~1.8 MB each. For a
single-day query this is great, but for a **full scan** it is *slower than
unpartitioned*, because the per-file overhead (open, read metadata, close)
dominates the tiny data read.

**Rule of thumb:** aim for **100 MB – 1 GB per partition**. Below a few MB
hurts performance.

## 3.4 Sort order (the second tool)

Partitioning decides **which group** a row belongs to. Sort order decides
**how rows are laid out inside each file**:

```sql
ALTER TABLE polaris.taxi.trips_sorted WRITE ORDERED BY PULocationID
```

When a file is sorted by column X, its min/max bounds for X become tight:

```
File A: PULocationID range 1–100
File B: PULocationID range 101–200
File C: PULocationID range 201–265
```

A query `WHERE PULocationID = 237` then opens **only File C**. This is
**metric pushdown** made effective by sorting.

### Cost

- Sorting costs extra CPU at write time.
- Iceberg does **not** re-sort on subsequent inserts → the sort degrades
  over time. Periodic maintenance (rewrite) is needed (covered in E3.2).

### Spark SQL quirk

Spark SQL cannot set a sort order inside a `CREATE TABLE AS SELECT`, so the
notebook uses a 3-step workaround:

```sql
-- 1. Create empty table
CREATE TABLE ... AS SELECT * FROM ... WHERE 1 = 0;
-- 2. Set the sort order BEFORE any data is written
ALTER TABLE ... WRITE ORDERED BY PULocationID;
-- 3. Insert the data (it gets sorted as it's written)
INSERT INTO ... SELECT * FROM ...;
```

## 3.5 Best practices for a Data Analyst designing tables

1. **Partition on the column you filter most** (usually a timestamp).
2. **Avoid high-cardinality columns** (user_id, order_id) — use `bucket()`
   instead of partitioning directly.
3. **Target 100 MB – 1 GB per partition** — don't over-partition.
4. **Sort on a column you filter but shouldn't partition** (category,
   location, status).
5. **Rely on hidden partitioning** — never add synthetic columns; always
   filter on the original column.
6. **Inspect metadata before and after changes** to measure impact (see
   [06-inspecting-tables.md](06-inspecting-tables.md)).
7. **There is no one perfect partition scheme** — it's always a trade-off
   tied to your query patterns.

## Key takeaways

- **Hidden partitioning** uses transforms (`months`, `days`, `bucket`...) —
  no fake columns, query the original column, Iceberg handles the rest.
- `months()` returns an **offset from 1970-01** (641 = June 2023).
- Pick partition grain by **expected partition size**, not by "more = better".
- **Sort order** enables metric pushdown on non-partition columns.
- Daily partitioning of small datasets causes the **small files problem**.

Next: [04-query-optimization.md](04-query-optimization.md)
