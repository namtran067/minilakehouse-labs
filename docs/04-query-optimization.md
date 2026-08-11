# 4 — Query Optimization: Predicate Pushdown

How Iceberg turns your `WHERE` clauses into file-skipping decisions —
**before any data is read**.

## 4.1 The three levels of pushdown

```
Your query (with WHERE)
   │
   ▼
1. Partition pruning  ── manifest level ── skip whole manifests
   │
   ▼
2. Metric pushdown    ── file level ──── skip files using min/max
   │
   ▼
3. Read the surviving data files
```

| Level | Triggered by | What gets skipped |
|-------|--------------|-------------------|
| **Full scan** | No filter | Nothing |
| **Partition pruning** | Filter on a partitioned column | Whole manifests / partitions |
| **Metric pushdown** | Filter on any column with stats | Individual files whose min/max don't overlap |

## 4.2 Partition pruning

A filter on the partition column is matched against the **partition summaries**
stored in the manifest list. Whole manifests that cannot contain matching
rows are dropped without being opened.

```sql
-- trips_by_month is partitioned by months(tpep_pickup_datetime)
SELECT COUNT(*) FROM polaris.taxi.trips_by_month
WHERE tpep_pickup_datetime >= '2023-08-01'
  AND tpep_pickup_datetime < '2023-09-01';
```

Even though the query never mentions the partition, Iceberg recognizes the
predicate is equivalent to partition value `{643}` (August 2023) and skips
the other 10 partitions.

> **Hidden partitioning in action:** you wrote a normal timestamp filter;
> Iceberg did the transform matching for you.

## 4.3 Metric pushdown

A filter on a **non-partitioned** column can still skip files, using the
per-file `lower_bound` / `upper_bound` statistics stored in the manifest.

```sql
-- trips_sorted is partitioned by month and SORTED by PULocationID
SELECT COUNT(*) FROM polaris.taxi.trips_sorted
WHERE PULocationID IN (132, 138, 161, 230, 237);
```

Because the data is sorted by `PULocationID`, each file covers a tight,
non-overlapping range of location IDs. Files whose `[lower, upper]` range
doesn't include any of the requested IDs are skipped entirely.

## 4.4 Combined pushdown (the best case)

Filters on **both** a partition column and a sorted column compound:

```sql
SELECT COUNT(*) FROM polaris.taxi.trips_sorted
WHERE tpep_pickup_datetime >= '2023-08-01'
  AND tpep_pickup_datetime < '2023-09-01'    -- partition pruning → only August
  AND PULocationID IN (132, 138, 161, 230, 237);  -- metric pushdown → only matching files
```

Result: the smallest possible scan.

## 4.5 Reading the Spark UI

After running a query, open **http://localhost:4040/SQL/**, find the query,
click **Description → showString** (the last operation), and look for the
`BatchScan` relation. Iceberg reports these statistics:

| Statistic | Meaning |
|-----------|---------|
| `number of result data files` | Files actually read |
| `number of skipped data files` | Files pruned by metrics |
| `number of skipped data manifests` | Manifests pruned by partition |
| `total data file size (bytes)` | Bytes scanned |
| `total planning duration (ms)` | Time Iceberg spent planning |

For a well-pruned query you'll see `skipped data files` close to the table's
total file count.

## 4.6 The `compare_partitioning_strategies` helper

The notebook defines a function that runs the same query across all four
tables and prints timing + speedup vs the unpartitioned baseline:

```python
compare_partitioning_strategies(
    where_clause="tpep_pickup_datetime >= '2023-08-14' AND tpep_pickup_datetime < '2023-08-17' AND PULocationID = 237",
    description="3 days in mid-August, pickup location 237"
)
```

### Expected winners by query shape

| Query pattern | Likely winner | Why |
|---------------|---------------|-----|
| 3 days + 1 location | Daily or Sorted | Strong partition + metric pruning |
| Full month | Monthly / Sorted | Monthly partition prunes to one month |
| Single day + 3 locations | Daily | Most selective partition pruning |
| Location only (no time) | **Sorted** | Only sort order exploits `PULocationID` min/max |

The last row is the clearest demonstration: **without a time filter the
partition does nothing** — only the sort order saves you.

## 4.7 Performance caveats

- **Run each query 2–3 times.** The first run pays warmup costs (loading
  classes, caching metadata). Use the second/third run for comparison.
- **Small files hurt full scans.** A daily-partitioned table can be slower
  than unpartitioned when there is no filter, due to per-file open overhead.
- **Reading the same bytes is not the same cost.** Two queries scanning the
  same data volume can differ a lot if one reads many small files.

## Key takeaways

- Pushdown happens in **planning**, before any data read.
- **Partition pruning** uses manifest partition summaries.
- **Metric pushdown** uses per-file min/max from `readable_metrics`.
- **Sort order** makes metric pushdown effective on non-partition columns.
- The Spark UI's `BatchScan` statistics make skipping **visible**.

Next: [05-snapshots-and-acid.md](05-snapshots-and-acid.md)
