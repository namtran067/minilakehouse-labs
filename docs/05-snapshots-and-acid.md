# 5 — Snapshots and ACID

## 5.1 What is a snapshot?

A **snapshot** is the complete, consistent state of the table at one point
in time. Every successful write creates exactly one new snapshot.

```
Before 17:16  →  table empty
17:16:02      →  Snapshot #1 (append, 17 files)   ← CTAS created the table
Next INSERT   →  Snapshot #2 (append, more files)
Next DELETE   →  Snapshot #3 (delete operation)
```

A snapshot is **not a diff** — it points to a manifest list that fully
describes all data files needed to reconstruct the table at that moment.

## 5.2 Snapshot operations

Each snapshot has an `operation`:

| Operation | When | Effect |
|-----------|------|--------|
| `append` | `INSERT INTO`, CTAS | New files added; old files untouched |
| `overwrite` | `INSERT OVERWRITE`, `UPDATE` | Some files replaced by new ones |
| `replace` | Maintenance (rewrite/compaction) | Files rewritten without changing data |
| `delete` | `DELETE` | Files removed (or delete markers in V2/V3) |

### Example from the NYC Taxi CTAS

```sql
SELECT committed_at, snapshot_id, operation
FROM polaris.taxi.trips_unpartitioned.snapshots;
```
```
+-----------------------+-------------------+---------+
|committed_at           |snapshot_id        |operation|
+-----------------------+-------------------+---------+
|2026-08-10 10:16:02.562|1242379465758984597|append   |
+-----------------------+-------------------+---------+
```

One commit → one snapshot → `append` because CTAS inserts into an empty table.

## 5.3 Why this matters for a Data Analyst — ACID

| Property | What it gives you |
|----------|-------------------|
| **Atomicity** | A query never sees a half-finished write — always a consistent snapshot |
| **Consistency** | The table always satisfies its schema and constraints |
| **Isolation** | Two concurrent writes don't corrupt each other |
| **Durability** | Once committed, the snapshot survives restarts |

You don't have to think about locking or partial reads — Iceberg hands you
a consistent view.

## 5.4 Time travel

Because every state is a snapshot, you can query **past** versions:

```sql
-- By snapshot number
SELECT * FROM polaris.taxi.trips_unpartitioned VERSION AS OF 1;

-- By timestamp
SELECT * FROM polaris.taxi.trips_unpartitioned
TIMESTAMP AS OF '2026-08-10 10:16:02';
```

Use cases for a DA:
- "What did this number look like before yesterday's load?"
- Reproduce a report exactly as it ran last week.
- Compare two snapshots to see what changed.

## 5.5 Idempotency — three patterns you must not confuse

These behave **very differently** on re-run:

| Pattern | Re-run behavior | Snapshot impact |
|---------|-----------------|-----------------|
| `CREATE TABLE IF NOT EXISTS ... AS SELECT` (CTAS) | **Skipped** if table exists | No new snapshot |
| `INSERT INTO ... SELECT` | **Appends** again → data duplicated | New snapshot, rows doubled |
| The E1.2 download cell (`os.remove` after upload) | **Re-downloads & re-uploads** 225 MB | (not a table write) |

### Proving the CTAS no-op

```sql
-- Count snapshots before
SELECT COUNT(*) FROM polaris.taxi.trips_unpartitioned.snapshots;  -- = 1
-- Run the CTAS cell again (IF NOT EXISTS)
-- Count snapshots after
SELECT COUNT(*) FROM polaris.taxi.trips_unpartitioned.snapshots;  -- still = 1
```

If the count doesn't change, the re-run was a no-op.

> **Gotcha:** the cell's `print("Unpartitioned table created!")` runs in
> Python **regardless** of whether Spark actually created the table. The
> print does **not** reflect what the SQL did.

## 5.6 Resetting a table for a clean re-run

The E1.2 notebook does **not** drop `trips_unpartitioned`, `trips_by_month`,
or `trips_by_day` before the CTAS (only `trips_sorted` has a `DROP`). So
re-running the notebook silently reuses the old tables. To reset cleanly:

```python
for t in ['trips_unpartitioned', 'trips_by_month', 'trips_by_day', 'trips_sorted']:
    spark.sql(f"DROP TABLE IF EXISTS polaris.taxi.{t}")
print("All taxi tables dropped — safe to re-run CTAS cells")
```

## Key takeaways

- Every write = **one atomic snapshot** with an `operation` tag.
- Snapshots enable **ACID** guarantees and **time travel**.
- CTAS with `IF NOT EXISTS` is **idempotent**; `INSERT INTO` is **not**.
- Use snapshot counts to prove whether a re-run actually did anything.
- The notebook doesn't drop tables before CTAS — drop manually to reset.

Next: [06-inspecting-tables.md](06-inspecting-tables.md)
