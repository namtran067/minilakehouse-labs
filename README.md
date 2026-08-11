# Mini Lakehouse — Apache Iceberg Lab

A self-contained Docker environment for learning **Apache Iceberg** with a
production-like lakehouse stack: a REST catalog (**Polaris**), object storage
(**MinIO**, S3-compatible), two query engines (**Spark** and **Trino**) sharing
the same tables, and a Jupyter notebook front end.

## Architecture

```
┌───────────────────────────┐                  ┌───────────────────────────┐
│    Spark Connect + Jupyter│                  │    Trino (port 8080)      │
│                           │                  │                           │
│ - Connect server (:15002) │                  │ - SQL Query Engine        │
│ - Jupyter Lab (:8888)     │                  │                           │
│                           │                  │                           │
└─────────────┬─────────────┘                  └─────────────┬─────────────┘
              │                                              │
              │\                                            /│
              │ \                                          / │
              │  \          ┌─────────────────┐          /  │
              │   \         │     Polaris     │         /   │
              │    \        │   REST Catalog  │        /    │
              │     └──────▶│   (port 8181)   │◀──────┘     │
              │             └────────┬────────┘             │
              │                      │                      │
              └──────────────────────┼──────────────────────┘
                                     │
                                     ▼
              ┌───────────────────────────────────────┐
              │         MinIO (S3-compatible)         │
              │         (ports 9000/9001)             │
              │      - Bucket: warehouse              │
              │      - Parquet data files             │
              │      - Metadata files                 │
              └───────────────────────────────────────┘
```

- Table data lives in MinIO at `s3://warehouse/`
- All metadata is managed by the Polaris REST catalog (centralized)
- A single Spark Connect server runs inside the Jupyter container; notebooks
  are thin clients

### How the services connect (system design)

Three principles make this stack work:

1. **Catalog (Polaris) is separate from storage (MinIO).** Polaris only stores
   metadata — "table `taxi` has this schema, this is its newest snapshot, its
   files live at `s3://warehouse/...`". The Parquet data itself lives in MinIO.
   This separation lets you swap storage (MinIO → real S3) without touching the
   catalog.

2. **Two engines share one set of files.** Spark writes a table → metadata
   updates in Polaris → Trino reads that metadata → reads the correct Parquet
   files. This is the core promise of Iceberg: an *open* format where every
   engine follows the same rules.

3. **Polaris vends credentials.** Spark/Trino don't need MinIO access keys
   hardcoded — Polaris hands out credentials scoped to each table's path (see
   `X-Iceberg-Access-Delegation=vended-credentials` in the Spark config). In
   this lab the creds are static, but the mechanism mirrors production (STS/role).

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

### Two network worlds

| World | How to reach it | Example |
|-------|-----------------|---------|
| **Container ↔ container** (internal) | service name + internal port | `http://minio:9000`, `http://polaris:8181` |
| **Host/laptop → container** | `localhost:MAPPED_PORT` | `http://localhost:8888` (Jupyter) |

Notebooks run *inside* the container, so their code uses service names
(`http://minio:9000`), not `localhost`.

### Startup order (dependency chain)

```
check-resources ──▶ minio (healthy) ──▶ polaris (healthy) ──▶ polaris-setup ──▶ trino
                                          ──▶ setup_bucket
```

Jupyter only waits on the memory check. Trino waits for `polaris-setup` because
it needs the catalog already registered.

## Pinned versions

All versions are managed centrally in [`.env`](.env):

| Component          | Version  |
| ------------------ | -------- |
| Apache Iceberg     | 1.10.0   |
| Spark (Scala 2.13) | 4.0.1    |
| Polaris            | `latest` |
| Trino              | 465      |
| AWS SDK v2         | 2.20.18  |

To change versions, edit `.env` and rebuild: `docker compose up -d --build`.

## Prerequisites

- Docker Desktop running
- **16 GB RAM** allocated to Docker (the stack runs Spark + Polaris + Trino + MinIO; 8 GB will OOM on the bigger notebooks)
- ~10 GB free disk space

## Quick start

```bash
docker compose up -d            # start all services
docker compose logs -f          # follow logs until ready (1-2 min), then Ctrl+C
```

Then open:

| Service              | URL                   | Credentials                     |
| -------------------- | --------------------- | ------------------------------- |
| Jupyter (start here) | http://localhost:8888 | none                            |
| MinIO Console        | http://localhost:9001 | `admin` / `password`            |
| Trino UI             | http://localhost:8080 | `admin` / none                  |
| Polaris API          | http://localhost:8181 | client `root` / secret `s3cr3t` |

Open the demo notebook directly:
http://localhost:8888/lab/tree/work/E1.1%20-%20OpenLakehouse.ipynb

## What you'll learn (Iceberg concepts)

The notebooks teach these concepts. Read this section top to bottom the first
time; it mirrors how the exercises build on each other.

### The metadata tree — the heart of Iceberg

Iceberg never scans files blindly. Every table has a layered metadata structure
that lets engines *skip* files without opening them:

```
Snapshot ──────────── full table state at a point in time
   │                  (points to a manifest list)
   ▼
Manifest list ─────── groups manifests + partition summaries
   │                  (skip whole groups of files)
   ▼
Manifest ──────────── Avro index listing data files with
   │                  partition values + per-column min/max stats
   ▼
Data files ────────── the actual Parquet files
```

This tree is why partition pruning and metric pushdown work — engines consult
metadata, not data, to decide what to read. Inspect it via the
[metadata tables](#metadata-tables-reference).

### Format versions

| Version | Adds | Status |
|---------|------|--------|
| **V1** | original spec | still widely used |
| **V2** | row-level deletes (COW/MOR) | most production |
| **V3** | delete vectors, initial defaults, `Variant`/`Geo` types | recommended for new tables |

Upgrading is a **one-way metadata change** — no data rewrite. Notebooks create
tables with `'format-version' = '3'`.

### Hidden partitioning & transforms

You never write the partition value in a query. Iceberg applies a **transform
function** to an existing column and matches it automatically:

```sql
PARTITIONED BY (months(tpep_pickup_datetime))   -- query still filters the raw column
```

Available transforms: `years`, `months`, `days`, `hours`, `bucket(N, col)`,
`truncate(N, col)`. The partition value stored internally is an encoded integer
(e.g. `641` = June 2023, months since epoch) — engines translate this for you.

**Best practice:** target **100 MB – 1 GB per partition**. Over-partitioning on
high-cardinality columns creates many tiny files; the per-file open/read/close
overhead dominates when files are a few MB.

### Query optimization: pushdowns

Iceberg evaluates predicates against metadata at two levels:

1. **Manifest level** — partition predicates compared to manifest summaries →
   skip whole manifests (critical at scale).
2. **File level** — surviving manifests opened; each data file's partition value
   + column stats (min/max, null/NaN counts) compared to predicates.

Add a **sort order** (`WRITE ORDERED BY col`) to tighten per-file min/max bounds
on a column → metric pushdown skips files whose range doesn't overlap the filter.
Cost: slower writes, and Iceberg does *not* re-sort on later inserts — needs
maintenance. Best for frequent range `WHERE` columns.

### Schema evolution (metadata-only, no data rewrite)

| Operation | Behavior |
|-----------|----------|
| `ADD COLUMN` | instant; old rows return `NULL` (or an *initial default* under format-version 3) |
| `DROP COLUMN` | instant; data stays on disk but is hidden |
| `RENAME COLUMN` | instant; only the name mapping moves |

**Field IDs, not names.** Iceberg tracks columns by internal numeric IDs. Drop a
column and re-add one with the same name → it gets a **new** ID, so old data
does **not** "resurrect". This prevents a subtle data-corruption bug.

### Partition evolution

Change partition strategy **without rewriting existing data**. Iceberg records
which spec each file was written under; old files keep their layout, new files
use the new spec, engines reconcile at read time. Specs are tracked by
definition, so reverting to an earlier spec **reuses** the old spec ID (no
duplication).

### Versioning: snapshots, time travel, branching, WAP

- **Snapshots** — every write creates an immutable snapshot. Cheap (metadata
  pointers), but their data files can't be garbage-collected until expired.
- **Time travel** — `SELECT ... VERSION AS OF <snapshot_id>` (or a timestamp /
  tag name) queries any past state.
- **Tags** — name an important snapshot (`CREATE TAG`). Every table has at least
  a `main` branch (see the `.refs` metadata table).
- **Rollback** — `rollback_to_snapshot` (backward only, safe undo) vs
  `set_current_snapshot` (jump to *any* snapshot, forward or backward). History
  is never erased until `expire_snapshots`.
- **Branches** — Git-like metadata pointers, not data copies. `fast_forward`
  merges a branch into main, but only if the branch is a **direct descendant**
  of main's current snapshot. Keep branches short-lived.
- **Write-Audit-Publish (WAP)** — stage a write invisibly (via `spark.wap.id`),
  audit it via `VERSION AS OF`, then `publish_changes` to make it visible or
  `expire_snapshots` to discard. Like a transaction for immutable files.

### Writing patterns

- **Batch vs streaming** — batch = large infrequent commits; streaming = small
  frequent commits (more snapshots, more small files → more maintenance).
- **Copy-on-Write (COW)** vs **Merge-on-Read (MOR)**:
  - **COW** rewrites the whole data file on update. Slower writes, faster reads.
  - **MOR** writes small delete files alongside originals. Faster writes, slower
    reads (merge at read time). Wins at low update %; loses at high %.
- **Row-level ops** — `DELETE` / `UPDATE` / `MERGE INTO` all work through COW or
  MOR. When *every row in a file* matches the predicate, Iceberg drops the whole
  file from metadata — **no read, no rewrite** (metadata-only delete).
- **Distribution modes** (Spark-specific) — controls the pre-write shuffle:
  - `hash` (default, no sort) — shuffle by partition value; clean uniform files.
  - `range` (default with sort) — also sorts within partition.
  - `none` — no shuffle. Fast if data is already organized by the partition key;
    otherwise causes a small-file problem.

### Concurrency

Iceberg uses **optimistic concurrency**: writers proceed independently, conflicts
checked only at commit. Detection is **file-level**, using the same column
min/max stats.

- Two writes touching **different files** → both succeed (no conflict).
- Two writes touching the **same file** → first wins, second gets
  `CommitFailedException`. No corruption; loser retries.
- **Put file-scoping predicates in the `ON` clause of MERGE / `WHERE` of
  UPDATE**, not in `WHEN MATCHED`. Conflict detection happens *before* `WHEN`
  evaluation.
- `commit.retry.num-retries` (default 4) handles catalog-level races for
  appends. COW update/delete failures are `ValidationException` (data conflict),
  requiring full re-execution.

### Migration strategies (moving existing data into Iceberg)

| Strategy | Rewrites data? | When to use |
|----------|----------------|-------------|
| **CTAS** (`CREATE TABLE AS SELECT`) | yes (full reserialize) | any source, want transform/repartition |
| **`snapshot` procedure** | no (metadata-only over existing Parquet) | fastest; source is already Parquet |
| **`add_files` procedure** | no (register files incrementally) | incremental adds; copy file into table location first |

> **Polaris vending requirement:** all of a table's files must live under its
> location path. `snapshot`/`add_files` do **not** move data — copy the Parquet
> file into the table's location first, then register it.

### Maintenance procedures

Traditional DBs run maintenance automatically. Iceberg separates storage from
compute, so there's no always-on process — schedule these as periodic jobs:

| Procedure | Fixes | Effect |
|-----------|-------|--------|
| `rewrite_manifests` | too many manifests | faster planning (fewer opens) |
| `rewrite_data_files` | too many small files | faster scans; `strategy => 'sort'` also re-sorts to restore non-overlapping ranges |
| `expire_snapshots` | old history hoarding files | physically deletes unreferenced files (kills time travel to them) |
| `remove_orphan_files` | files from failed writes | reclaims storage (`older_than` ≥ 24h required in production) |

Run them in that order. Note: compaction writes *new* files but old ones stay on
disk until the snapshots referencing them are expired.

### Sort order degradation

Each INSERT/MERGE creates its own independently-sorted files covering the
**full** value range. After many writes, files overlap → min/max pruning
degrades. Fix with `rewrite_data_files` using `strategy => 'sort'` (default
`binpack` only merges small files, doesn't re-sort).

### Metadata tables reference

Inspect physical layout without scanning data. In Spark: `catalog.namespace.table.<metadata_table>`.

| Table | Shows | Scope |
|-------|-------|-------|
| `.files` | data files + size + record count + `readable_metrics` (per-column min/max) | current snapshot |
| `.all_data_files` | same, across all live snapshots | total storage footprint |
| `.entries` | manifest-level entries (parallelizes better than `.files` in Spark) | — |
| `.manifests` | manifests + size | current snapshot |
| `.all_manifests` | manifests across all snapshots | — |
| `.snapshots` | commits + `operation` + `summary` map (`added-records`, `deleted-records`, …) | non-expired |
| `.refs` | branches + tags + the snapshot each points to | — |
| `.partitions` | partition stats | current snapshot |

Healthy table: `.files` count ≈ `.all_data_files` count (no replaced files
hoarded), few manifests.

## Notebooks

Eight exercises grouped into three modules. Run them in order the first time;
later modules assume catalogs and tables created by earlier ones.

| Module | Notebook                                   | Concepts covered                                                        |
| ------ | ------------------------------------------ | ------------------------------------------------------------------------ |
| 1      | `E1.1 - OpenLakehouse.ipynb`               | Environment setup, first table, Spark ↔ Trino interop, metadata tree     |
| 1      | `E1.2 - DataModeling.ipynb`                | Partitioning strategies on NYC Taxi data, predicate/metric pushdown      |
| 2      | `E2.1 - MovingExistingTables.ipynb`        | CTAS, `snapshot`, `add_files` migration + benchmarks                     |
| 2      | `E2.2 - BranchingAndTagging.ipynb`         | Time travel, tags, rollback, WAP, branching/merging                      |
| 2      | `E2.3 - SchemaAndPartitionEvolution.ipynb` | Schema/partition evolution, field IDs, PyIceberg API                     |
| 3      | `E3.1 - Ingestion.ipynb`                   | Batch/streaming, COW vs MOR, row-level ops, concurrency & conflicts      |
| 3      | `E3.2 - MaintenanceProcedures.ipynb`       | Manifest/data compaction, snapshot expiry, orphan removal, health report |
| 3      | `E3.3 - TableModelingAndIngestion.ipynb`   | Distribution modes, sort-order decay, `rewrite_data_files` sort strategy  |

## Study notes

Written notes are in [`docs/`](docs/README.md), organized top to bottom:

| #   | File                                                      | Covers                                                    |
| --- | --------------------------------------------------------- | --------------------------------------------------------- |
| 1   | [01-big-picture.md](docs/01-big-picture.md)               | Lakehouse vs warehouse/lake, the 5-component architecture |
| 2   | [02-metadata-tree.md](docs/02-metadata-tree.md)           | The 4-layer metadata tree (the heart of Iceberg)          |
| 3   | [03-partitioning.md](docs/03-partitioning.md)             | Hidden partitioning, transforms, strategies               |
| 4   | [04-query-optimization.md](docs/04-query-optimization.md) | Predicate pushdown, sort order, metric pushdown           |
| 5   | [05-snapshots-and-acid.md](docs/05-snapshots-and-acid.md) | Snapshots, ACID, time travel, idempotency gotchas         |
| 6   | [06-inspecting-tables.md](docs/06-inspecting-tables.md)   | Metadata tables reference (`.files`, `.snapshots`, …)     |
| 7   | [07-data-quality.md](docs/07-data-quality.md)             | Partitioning as an EDA tool (NYC Taxi lessons)            |
| 8   | [08-cheatsheet.md](docs/08-cheatsheet.md)                 | Copy-paste SQL + Docker snippets                          |
| 9   | [09-troubleshooting.md](docs/09-troubleshooting.md)       | OOM kills, Colima tuning, hanging cells                   |

## Services in detail

### MinIO (S3-compatible storage)

- API `9000`, Console `9001`, credentials `admin` / `password`
- Bucket `warehouse` is created automatically on startup
- Host data dir: `./data/minio`

### Polaris (Iceberg REST catalog)

- Port `8181` (management/health on internal `8182`, debug `5005`)
- Bootstrap credentials: `POLARIS,root,s3cr3t` (defined in `docker-compose.yml`)
- Uses **in-memory persistence** — catalog entries are lost when the container
  is removed (`docker compose down -v`). Fine for a lab; not for production.
- `polaris-setup` runs [`polaris/bootstrap-catalog.sh`](polaris/bootstrap-catalog.sh)
  on startup to register the `quickstart_catalog` catalog and storage config.

### Trino

- Port `8080`, username `admin` (no password)
- Catalog `polaris` connects to Polaris (config in [`trino/catalog/polaris.properties`](trino/catalog/polaris.properties) — the filename *is* the catalog name; the connector type is `iceberg`)
- Connect via CLI: `docker exec -it trino trino`

Example:

```sql
SHOW CATALOGS;
CREATE SCHEMA polaris.demo;
CREATE TABLE polaris.demo.test (id BIGINT, name VARCHAR) WITH (format = 'PARQUET');
INSERT INTO polaris.demo.test VALUES (1, 'Alice'), (2, 'Bob');
SELECT * FROM polaris.demo.test;
```

### Jupyter + PySpark (Spark Connect)

- Port `8888` (Lab UI), `4040` (Spark UI), `15002` (Spark Connect server)
- Pre-installed: PySpark, PyIceberg, Trino client, pandas, matplotlib, seaborn
- Notebooks are mounted from `./notebooks` → `/home/jovyan/work`
- The default catalog is `polaris` (see Spark config below)

## How Spark gets configured

Spark config is generated at container startup, not hard-coded. Three files work
together:

### 1. [`spark-defaults.conf.template`](spark-defaults.conf.template) — the static skeleton

Declares Spark's *desired* config with `${ENV_VAR}` placeholders: which JARs to
load, the Iceberg extension, Polaris catalog wiring (URI, OAuth, warehouse),
and MinIO/S3 storage settings. Edit versions in `.env`, not here.

### 2. [`generate-spark-config.py`](generate-spark-config.py) — runs in the container at boot

Reads the template, substitutes env vars from `.env`, and writes the real
`spark-defaults.conf`. It also implements **dual-mode JAR loading**:

- If `./jars/*.jar` exist (pre-downloaded) → uses `spark.jars=<local paths>`
  (offline mode).
- Otherwise → keeps `spark.jars.packages=...` and Spark downloads from Maven
  Central at startup.

### 3. [`manual-download-dependencies.sh`](manual-download-dependencies.sh) — offline fallback, run on the host

Pre-downloads 6 JARs (Iceberg runtime, AWS bundle, hadoop-aws, AWS SDK v1/v2)
from Maven Central into `./jars/` so the container can use them without network.
Needed when a corporate proxy blocks HTTPS to Maven Central.

> ⚠️ **Version gotcha:** the AWS SDK v2 JAR must be `2.33.0` (hardcoded in the
> script), *not* the `AWS_SDK_VERSION` from `.env` (2.20.18). `iceberg-aws-bundle`
> is compiled against the newer version; loading an older one causes
> `NoSuchMethodError`.

```
HOST:  manual-download-dependencies.sh ──▶ ./jars/*.jar (optional)
                                              │ mount read-only
                                              ▼
CONTAINER boot:
  spark-defaults.conf.template ──▶ generate-spark-config.py ──▶ spark-defaults.conf (real)
                                         │
                                   substitute ${ENV} + detect JAR mode
```

## Common commands

```bash
docker compose up -d            # start
docker compose down             # stop (keeps data)
docker compose down -v          # stop and wipe volumes
docker compose logs -f jupyter  # follow one service
docker compose restart jupyter  # restart one service
docker compose up -d --build    # rebuild after config changes
```

## Data persistence

All runtime data is stored under `./data/` (git-ignored):

| Path           | Contents                                        |
| -------------- | ----------------------------------------------- |
| `data/minio`   | Iceberg table data (Parquet + metadata files)   |
| `data/polaris` | Polaris catalog metadata (in-memory, ephemeral) |
| `data/trino`   | Trino working data                              |
| `data/jupyter` | Jupyter user data                               |

## Troubleshooting

### "Connection refused" / Spark Session won't start

The Spark Connect server may have failed. Check `docker logs jupyter-spark`,
look for the line `Spark Connect server is ready`. Common causes: insufficient
Docker RAM, or port 15002 already in use. Restart with
`docker compose restart jupyter`.

### Port already in use

```bash
lsof -i :8888   # Jupyter
lsof -i :8080   # Trino
lsof -i :9000   # MinIO API
lsof -i :9001   # MinIO Console
lsof -i :8181   # Polaris
```

### SSL / corporate proxy errors downloading JARs

If Spark fails to fetch its dependency JARs (SSL certificate errors in
`docker logs jupyter-spark`), pre-download them on the host and let the
container load them locally:

```bash
./manual-download-dependencies.sh --insecure   # skips SSL verification
docker compose down && docker compose up -d
```

You should see `Using pre-downloaded JARs from /opt/spark-jars` in the Jupyter
logs.

### Permission errors on data dirs

```bash
chmod -R 755 data/
```

### Reset everything

```bash
docker compose down -v
rm -rf data/*
docker compose up -d --build
```

## Notes

- All services communicate over the `iceberg-net` Docker bridge network.
- Jupyter has no password — fine for local use, not for production.
- Polaris persistence is in-memory — restart-safe only with `docker compose down`
  (no `-v`). Removing volumes wipes the catalog.
- The default Polaris client secret `s3cr3t` is committed in `.env` for lab
  convenience; rotate it for any non-local deployment.

## Additional resources

- [Apache Iceberg documentation](https://iceberg.apache.org/)
- [Polaris catalog](https://github.com/apache/polaris)
- [Trino Iceberg connector](https://trino.io/docs/current/connector/iceberg.html)
- [Spark + Iceberg getting started](https://iceberg.apache.org/docs/latest/spark-getting-started/)

## License

Licensed under the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0).
