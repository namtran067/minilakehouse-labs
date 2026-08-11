# Mini Lakehouse — Apache Iceberg Lab

A self-contained Docker environment for learning **Apache Iceberg** with a
production-like lakehouse stack: a REST catalog (**Polaris**), object storage
(**MinIO**, S3-compatible), two query engines (**Spark** and **Trino**) sharing
the same tables, and a Jupyter notebook front end.

This is a personal fork of the Snowflake "Apache Iceberg from zero" course
setup, extended with my own study notes in [`docs/`](docs/README.md).

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

## Notebooks

Eight exercises grouped into three modules. Run them in order the first time;

later modules assume catalogs and tables created by earlier ones.

| Module | Notebook                                   | Topic                                                        |
| ------ | ------------------------------------------ | ------------------------------------------------------------ |
| 1      | `E1.1 - OpenLakehouse.ipynb`               | Environment setup, first table, Spark ↔ Trino interop        |
| 1      | `E1.2 - DataModeling.ipynb`                | Partitioning strategies on NYC Taxi data, predicate pushdown |
| 2      | `E2.1 - MovingExistingTables.ipynb`        | Migrating existing tables into Iceberg                       |
| 2      | `E2.2 - BranchingAndTagging.ipynb`         | Git-like branching and tagging                               |
| 2      | `E2.3 - SchemaAndPartitionEvolution.ipynb` | Schema and partition evolution                               |
| 3      | `E3.1 - Ingestion.ipynb`                   | Batch and streaming ingestion patterns                       |
| 3      | `E3.2 - MaintenanceProcedures.ipynb`       | Compaction, expiry, snapshot management                      |
| 3      | `E3.3 - TableModelingAndIngestion.ipynb`   | Combined modeling + ingestion exercises                      |

## Study notes

My written notes from working through modules 1–2 are in [`docs/`](docs/README.md),
written for a Data-Analyst background (basic Python, no data engineering assumed).
Read them top to bottom the first time, then jump straight to the file you need.

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

- Port `8181` (management/health on internal `8182`)
- Bootstrap credentials: `POLARIS,root,s3cr3t` (defined in `docker-compose.yml`)
- Uses **in-memory persistence** — catalog entries are lost when the container
  is removed (`docker compose down -v`). Fine for a lab; not for production.
- `polaris-setup` runs [`polaris/bootstrap-catalog.sh`](polaris/bootstrap-catalog.sh)
  on startup to register the `warehouse` catalog and storage config.

### Trino

- Port `8080`, username `admin` (no password)
- Catalog `iceberg` connects to Polaris (config in [`trino/catalog/`](trino/catalog))
- Connect via CLI: `docker exec -it trino trino`

Example:

```sql
SHOW CATALOGS;
CREATE SCHEMA iceberg.demo;
CREATE TABLE iceberg.demo.test (id BIGINT, name VARCHAR) WITH (format = 'PARQUET');
INSERT INTO iceberg.demo.test VALUES (1, 'Alice'), (2, 'Bob');
SELECT * FROM iceberg.demo.test;
```

### Jupyter + PySpark (Spark Connect)

- Port `8888` (Lab UI), `4040` (Spark UI), `15002` (Spark Connect server)
- Pre-installed: PySpark, PyIceberg, Trino client, pandas, matplotlib, seaborn
- Notebooks are mounted from `./notebooks` → `/home/jovyan/work`
- The default catalog is `polaris` (see Spark config below)

## How Spark gets configured

Spark config is generated at container startup, not hard-coded:

1. [`spark-defaults.conf.template`](spark-defaults.conf.template) — the config
   skeleton with `${ENV_VAR}` placeholders for versions and credentials.
2. [`generate-spark-config.py`](generate-spark-config.py) — runs inside the
   Jupyter container on boot, substitutes env vars from `.env`, and writes the
   final `spark-defaults.conf`.
3. **Dual-mode JAR loading** — the generator detects which mode to use:
   - If `./jars/*.jar` exist (pre-downloaded) → uses `spark.jars=<local paths>`
     (offline mode).
   - Otherwise → keeps `spark.jars.packages=...` and Spark downloads from
     Maven Central at startup.

See **Troubleshooting → SSL / proxy errors** below for when to use the offline
mode.

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
logs. See [`manual-download-dependencies.sh`](manual-download-dependencies.sh)
for the full list of JARs and version-pinning rationale.

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

Copyright (c) 2026 Snowflake Inc. Licensed under the
[Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0).
