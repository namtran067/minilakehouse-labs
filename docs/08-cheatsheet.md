# 8 — Cheat Sheet: SQL and Docker Commands

Copy-paste reference for everyday work.

## 8.1 Catalog / namespace / table DDL

```sql
-- Namespace (= schema)
CREATE NAMESPACE IF NOT EXISTS polaris.demo;

-- Unpartitioned table (baseline)
CREATE TABLE IF NOT EXISTS polaris.demo.t1 (
    id INT,
    name STRING,
    ts  TIMESTAMP
) USING iceberg
TBLPROPERTIES ('format-version' = '3');

-- Partitioned table (hidden partitioning via transform)
CREATE TABLE IF NOT EXISTS polaris.demo.t2 (
    id INT,
    name STRING,
    ts  TIMESTAMP
) USING iceberg
PARTITIONED BY (months(ts))
TBLPROPERTIES (
    'format-version' = '3',
    'write.target-file-size-bytes' = '536870912'   -- 512 MB, sane default
);

-- Add a sort order (before any data is written)
ALTER TABLE polaris.demo.t2 WRITE ORDERED BY name;
```

> Set `write.target-file-size-bytes` to a sane value (e.g. 512 MB). The
> notebook uses 16 MB only to make partitioning effects visible on small data.

## 8.2 Loading data

```sql
-- CTAS from existing Parquet files (creates table + inserts in one shot)
CREATE TABLE polaris.demo.t3 USING iceberg
PARTITIONED BY (months(ts))
AS SELECT * FROM parquet.`s3a://warehouse/raw/`;

-- Append into an existing table
INSERT INTO polaris.demo.t3 SELECT * FROM parquet.`s3a://warehouse/raw/`;

-- Create empty then populate (needed to set sort order before write)
CREATE TABLE polaris.demo.t4 USING iceberg AS
SELECT * FROM parquet.`s3a://warehouse/raw/` WHERE 1 = 0;
ALTER TABLE polaris.demo.t4 WRITE ORDERED BY name;
INSERT INTO polaris.demo.t4 SELECT * FROM parquet.`s3a://warehouse/raw/`;
```

## 8.3 Querying

```sql
-- Normal query (filter on the ORIGINAL column; Iceberg matches the partition)
SELECT COUNT(*) FROM polaris.demo.t2
WHERE ts >= '2023-08-01' AND ts < '2023-09-01';

-- Time travel by snapshot id
SELECT * FROM polaris.demo.t2 VERSION AS OF 1;

-- Time travel by timestamp
SELECT * FROM polaris.demo.t2 TIMESTAMP AS OF '2026-08-10 10:16:02';
```

## 8.4 Inspecting metadata tables

```sql
-- Files with sizes and min/max bounds
SELECT file_path, partition,
       file_size_in_bytes / 1024 / 1024 AS size_mb,
       record_count,
       readable_metrics.total_amount.lower_bound AS amount_min,
       readable_metrics.total_amount.upper_bound AS amount_max
FROM polaris.demo.t2.files;

-- Rows + file counts per partition (use .entries, not .partitions)
SELECT data_file.partition,
       SUM(data_file.record_count) AS records,
       COUNT(*) AS files
FROM polaris.demo.t2.entries
GROUP BY data_file.partition
ORDER BY records DESC;

-- Snapshot history
SELECT committed_at, snapshot_id, operation
FROM polaris.demo.t2.snapshots;

-- Total rows (cross-check across strategies)
SELECT SUM(data_file.record_count) FROM polaris.demo.t2.entries;
```

## 8.5 Resetting tables

```sql
-- Drop before re-running a CTAS that uses IF NOT EXISTS
DROP TABLE IF EXISTS polaris.taxi.trips_unpartitioned;
DROP TABLE IF EXISTS polaris.taxi.trips_by_month;
DROP TABLE IF EXISTS polaris.taxi.trips_by_day;
DROP TABLE IF EXISTS polaris.taxi.trips_sorted;
```

## 8.6 Docker commands

```bash
# Start / stop the stack
docker compose up -d
docker compose down

# Restart just the Jupyter + Spark container
docker compose restart jupyter

# View logs
docker compose logs -f jupyter          # follow
docker logs jupyter-spark               # one-shot

# Check which containers are running and healthy
docker ps --format "table {{.Names}}\t{{.Status}}"

# Is Spark Connect alive?
docker exec jupyter-spark sh -c \
  'ps aux | grep SparkConnectServer | grep -v grep'

# Live CPU/RAM of the Spark container
docker stats jupyter-spark

# Open a Trino SQL shell
docker exec -it trino trino
```

## 8.7 Colima (macOS Docker runtime)

This project runs on Colima, not Docker Desktop. The socket is at
`~/.colima/docker.sock`.

```bash
# Check Colima resources (the README recommends 16 GB)
cat ~/.colima/_lima/colima/lima.yaml | grep -E "cpus|memory|disk"

# Resize Colima (restarts the VM)
colima stop
colima start --cpu 4 --memory 16

# If docker isn't on PATH in a non-interactive shell, set the host explicitly:
export DOCKER_HOST=unix:///Users/$USER/.colima/docker.sock
/opt/homebrew/bin/docker ps
```

## 8.8 URLs for the UIs

| Service | URL | Credentials |
|---------|-----|-------------|
| JupyterLab | http://localhost:8888 | (none) |
| MinIO Console | http://localhost:9001 | admin / password |
| Trino UI | http://localhost:8080 | admin |
| Spark UI | http://localhost:4040 | (none) |
| Polaris API | http://localhost:8181 | root / s3cr3t |

## 8.9 Decoding `months()` partition values

```
year  = 1970 + (value ÷ 12)
month = (value mod 12) + 1

Examples:
  641 → 2023-06      (53 years, 6th month)
  642 → 2023-07
  395 → 2002-12
```

## Key takeaways

- Prefer `.entries` over `.partitions` for aggregations.
- Use a sane `write.target-file-size-bytes` (512 MB) in real work.
- Colima needs at least 12–16 GB for this stack.
- The notebook's `IF NOT EXISTS` CTAS is a no-op on re-run — drop first to
  reset.

Next: [09-troubleshooting.md](09-troubleshooting.md)
