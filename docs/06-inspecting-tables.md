# 6 — Inspecting Tables: Metadata Tables Reference

Iceberg exposes "virtual tables" that describe a table's physical layout
**without scanning the data**. In Spark, access them with the pattern
`catalog.namespace.table.metadata_table`.

## 6.1 The metadata tables

| Metadata table | One row per | Best for |
|----------------|-------------|----------|
| `.files` | data file | Quick view of files + min/max |
| `.entries` | data file **× manifest** | Parallel-friendly aggregation in Spark |
| `.partitions` | partition | Small tables, partition summaries |
| `.snapshots` | snapshot | Audit commit history |
| `.history` | change event | Debugging table evolution |
| `.properties` | table property | Inspecting configuration |

## 6.2 `.files` vs `.entries`

Same information, different shape:

| | `.files` | `.entries` |
|---|---------|-----------|
| Granularity | 1 row = 1 data file | 1 row = 1 data file × 1 manifest |
| Column shape | flat (`partition`, `record_count`) | nested struct (`data_file.partition`) |
| Spark parallelism | moderate | **better** (reads manifests independently) |
| When to use | quick inspection | aggregations, large tables |

> Rule of thumb: for aggregations over many files, use `.entries` — it
> parallelizes better in Spark.

## 6.3 Common inspection queries

### List files with sizes and bounds

```sql
SELECT
    file_path,
    partition,
    file_size_in_bytes / 1024 / 1024 AS size_mb,
    record_count,
    readable_metrics.total_amount.lower_bound AS amount_min,
    readable_metrics.total_amount.upper_bound AS amount_max
FROM polaris.taxi.trips_unpartitioned.files;
```

### Aggregate rows and file counts per partition

```sql
SELECT
    data_file.partition,
    SUM(data_file.record_count) AS record_count,
    COUNT(*) AS data_files
FROM polaris.taxi.trips_by_month.entries
GROUP BY data_file.partition
ORDER BY data_file.partition;
```

> Note the `data_file.` prefix — in `.entries` the per-file fields are
> nested inside a struct called `data_file`.

### View the snapshot history

```sql
SELECT committed_at, snapshot_id, operation
FROM polaris.taxi.trips_unpartitioned.snapshots;
```

### Verify row counts match across strategies

The same dataset must produce the same total row count regardless of
partition scheme:

```sql
-- unpartitioned
SELECT SUM(data_file.record_count) FROM polaris.taxi.trips_unpartitioned.entries;
-- = 15,407,558

-- by month
SELECT SUM(data_file.record_count) FROM polaris.taxi.trips_by_month.entries;
-- = 15,407,558  (must match)
```

If they don't match, something is wrong (partial load, dropped table, etc.).

## 6.4 The `readable_metrics` struct

Each file entry exposes per-column statistics. Access them with
`readable_metrics.<column>.<stat>`:

```
readable_metrics.total_amount.lower_bound
readable_metrics.total_amount.upper_bound
readable_metrics.total_amount.null_count
readable_metrics.total_amount.nan_count
readable_metrics.total_amount.record_count
```

These are the same numbers the engine uses for **metric pushdown**, so
inspecting them tells you exactly which files a filtered query will skip.

## 6.5 Practical example: spotting a data-quality problem

Partition counts from the monthly table:

```
| partition | record_count | data_files |
|  {395}    |            5 |          1 |   ← 2002-12, only 5 rows!
|  {467}    |           10 |          1 |   ← 2008-12, only 10 rows!
|  {641}    |    3,307,234 |          4 |   ← 2023-06 (real data)
|  {645}    |    3,522,242 |          4 |   ← 2023-10 (real data)
```

A partition with a handful of rows sitting next to partitions with millions
is an instant red flag. See [07-data-quality.md](07-data-quality.md) for the
full story.

## Key takeaways

- Metadata tables let you inspect layout **without scanning data**.
- `.entries` parallelizes better than `.partitions` in Spark — prefer it for
  aggregations.
- In `.entries`, fields are nested under the `data_file` struct.
- `readable_metrics` exposes the exact min/max/null stats used for pushdown.
- Cross-checking totals across partition schemes catches load problems.

Next: [07-data-quality.md](07-data-quality.md)
