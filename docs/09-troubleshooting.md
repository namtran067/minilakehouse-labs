# 9 — Troubleshooting

Real problems encountered while running the course notebooks, with fixes.

## 9.1 Symptom: a cell shows `[*]` forever and never finishes

**Most likely cause:** the Spark Connect server has died (often an OOM kill),
and the Spark Connect client has no timeout, so the cell hangs silently.

### Diagnose

```bash
# 1. Is the Spark Connect process still running?
docker exec jupyter-spark sh -c \
  'ps aux | grep SparkConnectServer | grep -v grep'
# If this prints nothing, the server is dead.

# 2. Check the container logs for an OOM kill
docker logs jupyter-spark | grep -i killed
# A line like:  bash: line 6: 60 Killed .../spark-submit ...SparkConnectServer
# confirms the Linux OOM killer terminated it.

# 3. Check available RAM inside the VM
docker exec jupyter-spark free -h
```

### Fix (quick)

The root cause is almost always **insufficient RAM**. Free memory by stopping
unrelated containers, then restart Spark and the kernel:

```bash
# Stop anything heavy that isn't part of this project
# (e.g. OpenMetadata's Elasticsearch is a notorious RAM hog)
docker stop openmetadata_server openmetadata_postgresql openmetadata_elasticsearch

# Restart the Spark + Jupyter container (brings Spark Connect back up)
docker compose restart jupyter

# Wait ~45 s, then verify the server is back
docker exec jupyter-spark sh -c 'ps aux | grep SparkConnectServer | grep -v grep'
```

Then in Jupyter: **Kernel → Restart Kernel**, and re-run from the
"Initialize Spark Session" cell.

### Fix (permanent)

Give Colima more memory — the project README recommends 16 GB:

```bash
colima stop
colima start --cpu 4 --memory 16
```

### Before re-running a heavy cell

Quick health check — if this returns instantly, Spark is alive:

```python
spark.sql("SELECT 1 AS test").show()
```

While running large CTAS cells, watch memory live:

```bash
docker stats jupyter-spark
```

If `MEM USAGE` approaches the `LIMIT`, you're at risk of another OOM.

## 9.2 Symptom: `ConnectionRefusedError: [Errno 111] Connection refused`

The Spark Connect server isn't listening on port 15002. Same root cause as
9.1 — restart the container:

```bash
docker compose restart jupyter
```

Check the Spark Connect log for startup errors:

```bash
docker exec jupyter-spark tail -50 /tmp/spark-connect.log
```

## 9.3 Symptom: CTAS seems to "do nothing" on re-run

Expected behavior: `CREATE TABLE IF NOT EXISTS ... AS SELECT` is a **no-op**
if the table already exists. The `print(...)` after it still runs, which is
misleading.

To actually recreate the table, drop it first:

```python
spark.sql("DROP TABLE IF EXISTS polaris.taxi.trips_unpartitioned")
# then re-run the CTAS cell
```

Verify a re-run did nothing by checking snapshot count before/after:

```sql
SELECT COUNT(*) FROM polaris.taxi.trips_unpartitioned.snapshots;
```

See [05-snapshots-and-acid.md](05-snapshots-and-acid.md) for the full
idempotency discussion.

## 9.4 Symptom: `INSERT INTO` doubled the data

`INSERT INTO` is **append-only** — every run adds rows. If you ran it twice,
the table now has duplicates. Fix by dropping and reloading, or by
expiring/removing the unwanted snapshot (advanced; covered in E3.2).

Prevention: prefer `CREATE TABLE IF NOT EXISTS ... AS SELECT` for one-shot
loads, and reserve `INSERT INTO` for true incremental appends.

## 9.5 Symptom: the download cell re-downloads 225 MB every time

This is by design — the cell calls `os.remove(local_path)` after each upload,
so the local cache is always empty on the next run. It does **not** check
MinIO for an existing copy.

You don't need to re-run it: once the 5 monthly Parquet files are in
`s3a://warehouse/raw/`, the CTAS cells read from there. Skip the download
cell on subsequent runs.

## 9.6 Symptom: query results look "stale" after editing a CTAS

Because the CTAS uses `IF NOT EXISTS`, editing the SQL (e.g. changing
`format-version` or target file size) and re-running has **no effect** — the
old table is reused. Drop the table first (see 9.3).

## 9.7 Symptom: permission errors on the data directories

```bash
chmod -R 755 data/
```

## 9.8 Symptom: SSL / proxy errors downloading Spark JARs

On a corporate network, Spark may fail to fetch its dependency JARs. Use the
provided script with `--insecure`:

```bash
./manual-download-dependencies.sh --insecure
docker compose down
docker compose up -d
```

## 9.9 Full reset (start over from scratch)

```bash
docker compose down -v
rm -rf data/*
docker compose up -d --build
```

> Warning: this deletes all tables and data. Re-running every notebook from
> scratch will be required.

## 9.10 Port already in use

```bash
lsof -i :8888   # Jupyter
lsof -i :8080   # Trino
lsof -i :9000   # MinIO API
lsof -i :9001   # MinIO console
lsof -i :8181   # Polaris
lsof -i :15002  # Spark Connect
```

Kill or reconfigure whatever holds the port, then restart the stack.

## Key takeaways

- A **hanging cell** almost always means the **Spark Connect server died**
  (usually OOM) — check with `ps aux` inside the container.
- The fix is: free RAM → `docker compose restart jupyter` → restart the
  kernel → re-run.
- Give **Colima at least 12–16 GB** to prevent recurrence.
- `IF NOT EXISTS` CTAS is a silent no-op on re-run — drop to reset.
- `INSERT INTO` appends every time and can silently duplicate data.

Back to: [README.md](README.md)
