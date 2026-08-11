# 1 — The Big Picture: Why an "Open Lakehouse"

## 1.1 The three storage eras

| Era | Example | Pros | Cons |
|-----|---------|------|------|
| **Data Warehouse** | PostgreSQL, BigQuery, Snowflake | Structured, ACID, SQL-ready | Expensive, vendor lock-in, hard with unstructured data |
| **Data Lake** | Raw S3 / GCS folders | Cheap, open, holds anything | No tables, no transactions, becomes a "data swamp" |
| **Open Lakehouse** | Iceberg + S3 | Cheap **and** open **and** structured **and** ACID | Requires learning a table format |

The lakehouse combines the strengths of both: **store cheap files on object
storage, but describe them as real tables with transactions.**

## 1.2 What Apache Iceberg actually is

> Iceberg is **NOT a database, NOT a storage system, NOT a query engine.**
> It is an **open table format** — a set of rules plus a bundle of metadata
> files that lets many engines read and write the same set of data files
> without conflict.

Think of it as the "card catalog + classification system" of a library:
the books (Parquet files) live on shelves (S3), and anyone who knows the
system (any engine) can find and read them.

```
Apache Iceberg = the "bridge"
  - Format: open spec
  - Storage: Parquet files on object storage (cheap)
  - Readers/writers: Spark, Trino, Flink, Snowflake, Dremio...
```

## 1.3 The 5-component architecture in this project

```
Jupyter Notebook ──(PySpark / SQL)──> Spark ─┐
                  ──(trino client)──> Trino ──┤
                                              │  ask: "where is table X?"
                                              ▼
                                        Polaris (catalog)
                                              │  returns file locations
                                              ▼
                                        MinIO (S3-compatible)
                                        - Parquet data files
                                        - Iceberg metadata files
```

| Component | Role | Analogy | Port |
|-----------|------|---------|------|
| **MinIO** | S3-compatible object storage | The book shelves | `9000` API / `9001` console (admin/password) |
| **Polaris** | Iceberg REST Catalog | The card catalog / lookup desk | `8181` (root:s3cr3t) |
| **Spark** | Distributed processing engine | Heavy-lifting warehouse worker | `4040` UI / `15002` Connect |
| **Trino** | Distributed SQL engine for analytics | Fast query advisor | `8080` |
| **Jupyter** | Interactive notebook IDE | Your desk | `8888` |

## 1.4 Why is the catalog a separate service?

In a traditional database, the list of tables lives **inside** the database
server — so only that one engine can use it.

Iceberg **splits the catalog out** into its own service (Polaris here).
Every engine talks to the same catalog via a REST API, so they all share
the **same table definitions**.

> **This is why Spark can CREATE a table and Trino can immediately SELECT
> from it** — both engines ask Polaris "where is this table?" and get the
> same answer. With a traditional warehouse this would be impossible.

## 1.5 The "a-ha" moment: engine interoperability

In notebook E1.1, a table is created with **Spark** and then read back with
**Trino** — a completely separate engine that knows nothing about Spark.

This works because:
1. Both engines ask **Polaris** for the table's metadata.
2. Both read the **same open Parquet files** on MinIO.
3. Iceberg's open metadata means there is no proprietary lock-in.

The data belongs to **you**, not to whichever engine wrote it.

## 1.6 How Docker ties it together

`docker compose up -d` starts 6 services in dependency order:

1. `check-resources` — verifies the VM has enough RAM
2. `minio` — the S3-compatible store
3. `setup_bucket` — creates the `warehouse` bucket
4. `polaris` — the catalog server
5. `polaris-setup` — bootstraps the catalog with storage config
6. `jupyter` — JupyterLab + a single **Spark Connect** server (one JVM shared
   by all notebooks as thin clients)
7. `trino` — the SQL engine, wired to Polaris

> Note the **Spark Connect** architecture: one Spark server runs in the
> background and notebooks connect to it as lightweight clients. If that
> server dies, every `spark.sql(...)` cell hangs forever (see
> [09-troubleshooting.md](09-troubleshooting.md)).

## Key takeaways

- A **lakehouse** = cheap open storage + real tables + ACID.
- **Iceberg** is the open table format that makes it work — not an engine.
- The **catalog** (Polaris) is separated so multiple engines share tables.
- **Interoperability** (Spark writes, Trino reads) is the headline benefit.

Next: [02-metadata-tree.md](02-metadata-tree.md)
