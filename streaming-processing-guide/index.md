---
layout: article
title: "End-to-End Streaming Processing: MySQL CDC → Flink → Iceberg → Trino"
date: 2026-02-14
author: Tien Nguyen
excerpt: "A production-style streaming framework: read CDC from MySQL, join and aggregate two tables in Flink, write one result table to Iceberg, query with Trino, and chart with Superset."
reading_time: 25
permalink: /streaming-processing-guide/
---

If you're building a lakehouse or real-time analytics pipeline, you've probably juggled CDC, stream processing, and table formats. Getting them to work together—with joins and aggregates on the fly—is where it gets interesting. This guide walks through an end-to-end streaming framework: **MySQL CDC → Apache Flink (join + aggregate) → Apache Iceberg → Trino → Superset**.

You get a single pipeline that reads two source tables, joins them, aggregates by key, and writes one result table to Iceberg. Then you query and chart that table with Trino and Superset. No Kafka in the middle for this setup—Flink reads binlog directly and writes to Iceberg.

---

## What We're Building

- **Source**: MySQL with `products` and `sales`; row-based binlog for CDC.
- **Processing**: One Flink SQL job that:
  - Reads `products` and `sales` via MySQL CDC.
  - Joins `sales` with `products` on `product_id`.
  - Aggregates by product: `total_qty`, `total_revenue`, `sale_count`, `last_sale_ts`.
  - Writes the result to a single Iceberg table: `demo.sales_by_product` (upsert by `product_id`).
- **Storage**: Iceberg catalog in MySQL (JDBC), data in MinIO (S3-compatible).
- **Query**: Trino over Iceberg.
- **BI**: Superset connected to Trino, charting Iceberg data.

```
┌─────────────┐     ┌──────────────────────────────────────────┐     ┌─────────────┐
│   MySQL     │     │  Flink (CDC → join → aggregate)          │     │   Iceberg   │
│  products   │────▶│  sales ⟗ products → GROUP BY product_id  │────▶│ sales_by_   │
│  sales      │     │  → upsert demo.sales_by_product           │     │ product     │
└─────────────┘     └──────────────────────────────────────────┘     └──────┬──────┘
                                                                              │
       ┌─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│   Trino     │────▶│  Superset   │
│  (query)    │     │  (charts)    │
└─────────────┘     └─────────────┘
```

---

## Part 1: Architecture

### Components

| Component | Role |
|-----------|------|
| **MySQL** | OLTP source; binlog_format=ROW, binlog_row_image=FULL. Databases: `appdb` (products, sales), `iceberg_catalog` (Iceberg JDBC catalog metadata). |
| **MinIO** | S3-compatible object store for Iceberg data (warehouse path `s3://iceberg/warehouse`). |
| **Flink** | JobManager + TaskManagers. Runs one streaming job: MySQL CDC sources → join → group aggregate → Iceberg sink (upsert). |
| **Iceberg** | Table format. Catalog: JDBC (MySQL). Tables created by Flink; Trino and Flink share the same catalog. |
| **Trino** | SQL engine to query Iceberg tables (e.g. `SELECT * FROM iceberg.demo.sales_by_product`). |
| **Superset** | BI; connects to Trino, uses pre-configured "Trino Iceberg" database to build datasets and charts from Iceberg. |

### Data Flow

1. **CDC**: Flink MySQL CDC connector does initial snapshot then streams binlog events for `products` and `sales`.
2. **Join**: Stream–stream join on `sales.product_id = products.id`. Both sides are changelog streams.
3. **Aggregate**: Non-windowed group by `(product_id, product_sku, product_name)` with `SUM(qty)`, `SUM(qty*price)`, `COUNT(*)`, `MAX(sale_ts)`.
4. **Sink**: Iceberg table with primary key `product_id` and `write.upsert.enabled=true`, so each key is updated in place.
5. **Query**: Trino reads Iceberg via the same JDBC catalog and S3 (MinIO); Superset runs SQL via Trino and renders charts.

---

## Part 2: Flink SQL Job

The pipeline is defined in a single Flink SQL file (one statement per line to satisfy the client):

- **Catalog**: Iceberg JDBC catalog pointing at MySQL and MinIO (S3).
- **Temporary tables**: `mysql_products_stream`, `mysql_sales_stream` (MySQL CDC).
- **Iceberg table**: `lake.demo.sales_by_product` with format-version 2 and upsert enabled.
- **Insert**: Join + aggregate into `lake.demo.sales_by_product`.

Important details:

- No `IF NOT EXISTS` in DDL (Flink SQL parser doesn’t support it in this setup).
- Single-line statements so the SQL client submits one statement at a time.
- State TTL and checkpointing configured so the job can run continuously.

---

## Part 3: Deployment

### Prerequisites

- Docker and Docker Compose (or docker-compose).
- Optional: Trino needs the MySQL JDBC JAR for the Iceberg JDBC catalog; the repo can document downloading it into `trino/lib/`.

### Commands

```bash
# Build and start all services (MySQL, MinIO, Flink, Trino, Superset, mysql-data-inserter)
docker compose up -d --build

# Create MinIO bucket for Iceberg (e.g. iceberg)
docker compose run --rm mc

# Submit the Flink job (after Flink is up)
./submit-flink-job.sh
```

### Verification

- **Flink**: http://localhost:8081 — one running job (insert into `lake.demo.sales_by_product`).
- **Trino**: http://localhost:8080 — run `SELECT * FROM iceberg.demo.sales_by_product`.
- **Superset**: http://localhost:8088 (admin / admin). Trino Iceberg database is pre-added; add dataset from `demo.sales_by_product` and build charts.

Scripts like `verify-stack.sh` can check containers, MySQL CDC settings, MinIO bucket, Flink job, and Trino visibility of the Iceberg table.

---

## Part 4: Superset and Trino

- **Trino database in Superset**: Pre-configured at startup with display name **Trino Iceberg** and SQLAlchemy URI `trino://admin@trino:8080/iceberg`. No need to add it manually.
- **Charts**: In Superset, create a dataset from schema `demo`, table `sales_by_product`. Then create charts (e.g. bar chart of `total_revenue` by `product_sku`). Data is live from Iceberg via Trino.
- **SQL Lab**: Select database **Trino Iceberg** and run ad-hoc SQL (e.g. `SELECT * FROM demo.sales_by_product`).

---

## Part 5: CI/CD and Repo Layout

A minimal CI workflow can:

- **Validate**: Run `docker compose config` (or `docker-compose config`) and shell-check scripts like `submit-flink-job.sh` and `verify-stack.sh`.
- **Build**: Build Flink, Superset, and mysql-data-inserter images (no push) to ensure Dockerfiles and context are valid.

Redundant files to remove for a clean repo:

- Duplicate or one-off scripts (e.g. standalone insert script if the loop script is the one used by the inserter service).
- A second compose file if one main `docker-compose.yml` is enough.

---

## Best Practices Summary

1. **Single pipeline**: One Flink job for CDC + join + aggregate → one Iceberg table keeps the topology simple and avoids multiple writers to the same table.
2. **Upsert by key**: Iceberg table with primary key and `write.upsert.enabled=true` so Flink’s changelog stream updates rows correctly.
3. **Shared catalog**: Same Iceberg JDBC catalog and warehouse for Flink and Trino so both see the same tables and paths.
4. **Pre-add Trino in Superset**: Adding the Trino database at startup avoids permission issues and gives users immediate access to Iceberg data.
5. **Verify end-to-end**: Use a small script to assert containers, CDC, bucket, Flink job, and Trino query so the stack is ready for demos or development.

---

## Putting It Together

This setup is a template for a **streaming lakehouse**: OLTP in MySQL, stream processing in Flink (with join and aggregation), and query/BI on Iceberg via Trino and Superset. You can extend it with more sources, more joins, or different aggregates; the pattern—CDC → Flink → Iceberg → Trino/Superset—stays the same.

For the full code, Docker Compose, Flink SQL job, and scripts, see the [streaming-processing](https://github.com/xxntti3n/streaming-processing) repository (or your fork). Run it locally, point Superset at the pre-configured Trino Iceberg database, and start charting.

---

_This guide describes an end-to-end streaming framework suitable for development and learning. For production, consider managed services (e.g. Flink on K8s or cloud), dedicated catalog and object storage, and proper security and networking._

Share this article
