# 2 — The Metadata Tree (the heart of Iceberg)

If you understand only one thing about Iceberg, understand this tree.

## 2.1 The 4-layer structure

Every Iceberg table stores its data as files, but the **metadata** forms a
tree with four layers:

```
Snapshot                          (the table's state at one point in time)
   │
   ▼
Manifest List  (snap-*.avro)      (groups manifests + partition summaries)
   │
   ▼
Manifest        (*-m0.avro)       (lists data files + per-column min/max)
   │
   ▼
Data files      (*.parquet)       (the actual row data, columnar)
```

## 2.2 Why four layers instead of one?

Each layer lets the engine **skip files at a different granularity** during
query planning — without ever opening the data files.

| Layer | Contains | Enables skipping of... |
|-------|----------|------------------------|
| **Snapshot** | Pointer to one manifest list | Nothing (it's the entry point) |
| **Manifest list** | Many manifests + partition summaries | Whole groups of files at once |
| **Manifest** | Many data files + partition values + min/max stats | Individual files |
| **Data file** | Rows (Parquet) | — |

A query reads the metadata top-down and prunes aggressively:

```
Snapshot
  └─ Manifest list          (skip entire manifests via partition summaries)
       └─ Manifest          (skip files via partition values)
            └─ Data file    (skip row groups via min/max, then read)
```

With thousands of files, this layered skipping is what makes Iceberg fast.

## 2.3 What you actually see on disk

For the table `polaris.taxi.trips_unpartitioned`, the folder layout on MinIO is:

```
warehouse/taxi/trips_unpartitioned/
├── data/
│   ├── 00000-23-...-00001.parquet       ← data files (Parquet)
│   ├── 00000-23-...-00002.parquet
│   └── ... (17 files total)
└── metadata/
    ├── 00000-...metadata.json           ← table metadata (schema, location, current snapshot)
    ├── fbc37b50-...-m0.avro             ← manifest
    └── snap-1242...avro                 ← manifest list (the current snapshot points here)
```

### What each metadata file holds

- **`*.metadata.json`** — the table's "brain": schema, partition spec,
  properties, snapshot log, and a pointer to the **current** snapshot.
- **`*-m0.avro`** (manifest) — a list of data files, each with its
  **partition value** and **per-column statistics** (lower/upper bound,
  null count, NaN count, record count).
- **`snap-*.avro`** (manifest list) — groups manifests together with
  partition summaries so the engine can skip whole manifests at once.

## 2.4 The `readable_metrics` column — Iceberg's "gold"

Each data file entry in a manifest stores **statistics per column**:

```
readable_metrics.<column>.lower_bound   ← minimum value of that column in this file
readable_metrics.<column>.upper_bound   ← maximum value
readable_metrics.<column>.null_count    ← number of nulls
readable_metrics.<column>.nan_count     ← number of NaNs
readable_metrics.<column>.record_count  ← number of rows
```

This is exactly what powers **metric pushdown** (see
[04-query-optimization.md](04-query-optimization.md)): a query
`WHERE total_amount > 50000` opens only files whose `upper_bound > 50000`.

## 2.5 File naming convention

```
00000-23-8bfd1b03-2e9c-4a4e-bef9-dce21f805c1b-0-00001.parquet
└─┬─┘└┬┘└─────────────────┬──────────────────────┘└┬┘└─┬─┘
  │   │                    UUID of the write job       │   │
  │   Task ID                                         │   File sequence
  │   (one task reads one input file,                 Attempt number
  │    writes several output files)                   
  Partition ID (= 0 when the table is unpartitioned)
```

Observed pattern on the NYC Taxi load: 5 different task IDs (one per input
parquet month), each producing 3–4 output files of ~16 MB.

## 2.6 Partition directories on MinIO are cosmetic

When a table is partitioned, you'll see folders like:

```
warehouse/taxi/trips_by_month/data/
└── tpep_pickup_datetime_month=2023-06/
    └── 00000-...-00001.parquet
```

> **Important:** these directory names exist for **human browsing only**.
> Iceberg does **not** scan directories during query planning. The real
> partition tracking lives in the **manifest files**. This is a critical
> difference from Hive, which relies on directory listing (slow and fragile).

Practical consequence: **never rename or delete partition directories by
hand** — you'd desync them from the manifest and corrupt the table. All
changes must go through SQL.

## Key takeaways

- Iceberg's metadata is a **4-layer tree**: snapshot → manifest list →
  manifest → data files.
- Each layer enables file **skipping at a different scale**, which is what
  makes Iceberg fast.
- `readable_metrics` (min/max/null per column per file) is the fuel for
  metric pushdown.
- Partition folders are **cosmetic**; the manifest is the source of truth.

Next: [03-partitioning.md](03-partitioning.md)
