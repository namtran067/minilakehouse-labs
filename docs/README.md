# Apache Iceberg — Study Notes (from the ground up)

Personal notes from working through the `apache-iceberg-from-zero` course
(notebooks E1.1 and E1.2), written for someone with a **Data Analyst background
and basic Python only** — no prior data engineering assumed.

## How to use these notes

Read top to bottom the first time (1 → 9). After that, jump straight to the
file you need. Every file is self-contained and cross-links to related topics.

## Table of contents

| # | File | What it covers |
|---|------|----------------|
| 1 | [01-big-picture.md](01-big-picture.md) | Lakehouse vs warehouse/lake, what Iceberg actually is, the 5-component architecture |
| 2 | [02-metadata-tree.md](02-metadata-tree.md) | The 4-layer metadata tree (snapshot → manifest list → manifest → data files) — the heart of Iceberg |
| 3 | [03-partitioning.md](03-partitioning.md) | Hidden partitioning, transform functions, the 4 strategies, best practices, small files problem |
| 4 | [04-query-optimization.md](04-query-optimization.md) | Predicate pushdown (3 levels), sort order, metric pushdown, reading the Spark UI |
| 5 | [05-snapshots-and-acid.md](05-snapshots-and-acid.md) | Snapshots, ACID, time travel, idempotency gotchas |
| 6 | [06-inspecting-tables.md](06-inspecting-tables.md) | Metadata tables reference: `.files`, `.entries`, `.snapshots`, `.partitions` |
| 7 | [07-data-quality.md](07-data-quality.md) | Using partitioning as an EDA tool to surface anomalies (NYC Taxi lessons) |
| 8 | [08-cheatsheet.md](08-cheatsheet.md) | Copy-paste SQL snippets and Docker commands |
| 9 | [09-troubleshooting.md](09-troubleshooting.md) | OOM kills, Colima tuning, hanging cells, idempotency traps |

## Mental model in one diagram

```
You (Jupyter) ──> Spark / Trino (engines)
                        │
                        ▼
                Polaris (catalog: "where is table X?")
                        │
                        ▼
               MinIO / S3 (stores Parquet files + metadata)
                        ▲
                        │
               Apache Iceberg = the open rules + metadata
               that let many engines share the same files
```

## Source notebooks

- `notebooks/E1.1 - OpenLakehouse.ipynb` — environment setup, first table,
  engine interoperability demo.
- `notebooks/E1.2 - DataModeling.ipynb` — partitioning strategies on NYC Taxi
  data, predicate pushdown, performance comparison.
