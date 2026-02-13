---
layout: article
title: "Understanding Apache Iceberg Catalogs"
date: 2025-02-15
author: Tien Nguyen
excerpt: "What an Iceberg catalog is, how it fits in the table lifecycle, and the main catalog types: Hive, Hadoop, REST, Glue, JDBC, and custom implementations."
reading_time: 12
categories: [iceberg]
permalink: /iceberg/iceberg-catalog/
---

In Apache Iceberg, the **catalog** is the system that tracks *where* table metadata lives and provides a consistent view of table identity (namespace + table name) to engines like Spark, Flink, and Trino. This article explains what a catalog does, how it fits in the Iceberg stack, and the main catalog types you’ll see in practice.

---

## What Is an Iceberg Catalog?

An Iceberg table’s data and metadata (snapshots, schemas, partition specs) are stored in object storage or HDFS. The **catalog** does not store that data; it stores **table names and the location of each table’s current metadata file**.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHAT THE CATALOG STORES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Catalog holds:                                                             │
│  • Logical name:  namespace + table name  (e.g. prod.orders)               │
│  • Metadata location: path to current metadata.json                        │
│                                                                             │
│  Catalog does NOT hold:                                                     │
│  • Table data files                                                        │
│  • Snapshots, manifests, schema history (those live in metadata + storage) │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

So when you run `LOAD TABLE prod.orders`, the engine asks the catalog for the metadata location of `prod.orders`, then reads metadata (and thus snapshots, manifests, data files) from that location. The catalog is the **index** from (namespace, table name) → metadata location.

---

## Where the Catalog Fits: End-to-End Picture

![Iceberg stack: Engine to Catalog to Metadata to Data](/assets/images/iceberg-catalog-stack.svg)

The following diagram shows how the catalog sits between the engine and the table’s metadata and data.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│              ICEBERG STACK: CATALOG IN CONTEXT                                     │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│   Engine (Spark / Flink / Trino)                                                  │
│        │                                                                          │
│        │  "Load table prod.orders"                                                │
│        ▼                                                                          │
│   ┌─────────────┐                                                                 │
│   │  CATALOG    │  ← Returns: metadata location (e.g. s3://warehouse/orders/     │
│   │             │              metadata/00001-xxx.metadata.json)                  │
│   └──────┬──────┘                                                                 │
│          │                                                                        │
│          │  metadata location                                                     │
│          ▼                                                                        │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  TABLE METADATA (in object storage / HDFS)                               │   │
│   │  • metadata.json  → current snapshot, schema, partition spec, refs      │   │
│   │  • snapshots/*.avro → manifest lists                                     │   │
│   │  • manifests/*.avro → data/delete file paths                             │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│          │                                                                        │
│          │  file paths from manifests                                             │
│          ▼                                                                        │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  DATA FILES (Parquet / ORC / Avro)                                       │   │
│   │  data/partition=value/file.parquet                                        │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                   │
│   Flow: Catalog → metadata location → metadata → manifests → data files          │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

So: **catalog = name → metadata location**. Everything else (snapshots, manifests, data) is read from the table’s metadata and storage, not from the catalog.

---

## What the Catalog Is Responsible For

Conceptually, the catalog implements a small set of operations:

| Operation | What the catalog does |
|----------|------------------------|
| **Create table** | Choose a metadata location, write initial metadata, register (name → metadata location). |
| **Load table** | Resolve name → metadata location and return it so the engine can load metadata. |
| **Drop table** | Remove the mapping; optionally trigger or rely on external cleanup of data/metadata. |
| **Rename table** | Update the mapping from old name to new name (and possibly move metadata). |
| **List tables / namespaces** | Enumerate names (and sometimes metadata locations) the catalog knows about. |

The catalog must also support **atomic commit semantics**: when a commit writes a new metadata file and updates the catalog to point to it, that update must be atomic so concurrent readers never see a half-updated state.

---

## Main Catalog Types

Iceberg supports several built-in and common catalog implementations. The table below summarizes them; details depend on the engine (Spark, Flink, etc.) and environment.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              COMMON ICEBERG CATALOG TYPES                                         │
├──────────────────┬──────────────────────────────────────────────────────────────┤
│  Catalog         │  Use case / storage of "name → metadata location"             │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│  HiveCatalog     │  Hive Metastore (HMS). Good for existing Hadoop/Hive shops.   │
│  HadoopCatalog   │  No central service; paths under a warehouse directory.       │
│  RESTCatalog     │  REST API (OpenAPI). Cross-language, used by many SaaS.        │
│  GlueCatalog     │  AWS Glue Data Catalog. Managed, serverless on AWS.           │
│  JdbcCatalog     │  Any JDBC DB (e.g. MySQL, Postgres) for table registry.       │
│  NessieCatalog   │  Nessie (Git-like branches/tags) for table metadata.         │
│  Custom          │  Your own implementation (e.g. internal metadata store).      │
└──────────────────┴──────────────────────────────────────────────────────────────┘
```

- **HiveCatalog** — Uses Hive Metastore to store table name → metadata location. Fits well when you already run Hive/Hadoop and want to share the same metastore.
- **HadoopCatalog** — No separate service. The “catalog” is a warehouse root; table identity is effectively path-based (e.g. `warehouse/db/table/metadata.json`). Simple for single-cluster or file-based setups.
- **RESTCatalog** — Implements the [Iceberg REST Catalog API](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml). Any engine or language can talk to the same catalog over HTTP. Common in managed/SaaS offerings.
- **GlueCatalog** — Uses AWS Glue as the registry. Fully managed, works with Spark, Flink, Athena, etc., on AWS.
- **JdbcCatalog** — Stores namespace/table and metadata location in a relational database via JDBC. Good when you want a simple, DB-backed catalog.
- **NessieCatalog** — Uses Project Nessie for versioning; adds Git-like branches and tags for table metadata.

You choose one (or several) based on your environment (on-prem vs cloud, existing HMS vs Glue vs nothing) and whether you need REST, branching, or a simple path-based catalog.

---

## How the Catalog Is Used at Read and Write Time

**Read path:** The engine gets the table name (e.g. from SQL). It calls the catalog to resolve the name to a metadata location, then loads the table metadata from that location and follows snapshot → manifest list → manifests → data files as in the [Iceberg spec](https://iceberg.apache.org/spec/).

**Write path (commit):** The engine writes new metadata (new snapshot, manifest list, manifests) to the table’s metadata path, then tells the catalog to **commit** by updating the stored metadata location from the previous file to the new file. That update must be atomic so that the “current” pointer always points to a valid, complete metadata file.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              COMMIT FLOW (SIMPLIFIED)                                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. Engine writes new metadata file:  .../metadata/00002-yyy.metadata.json       │
│  2. Engine asks catalog: "Set metadata location for prod.orders to 00002-yyy"    │
│  3. Catalog atomically updates:  prod.orders → 00002-yyy.metadata.json          │
│  4. New readers see new snapshot; old readers can still use previous metadata.   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

So the catalog’s main job at write time is to **atomically switch** the metadata location for a table after a successful commit.

---

## Custom Catalogs

If you need a catalog backed by your own system (e.g. internal DB or API), you implement the Iceberg `Catalog` interface (and related pieces such as `TableOperations`). The catalog is responsible for:

- **Resolving** table identifier → metadata location (for load).
- **Committing** by atomically updating that mapping to a new metadata file (for writes).
- **Create / drop / rename / list** as required by your use case.

Engines (Spark, Flink, etc.) typically load the custom catalog via a catalog property (e.g. `catalog-impl=com.example.MyCatalog`). For more detail, see the Iceberg docs on [custom catalog implementation](https://iceberg.apache.org/docs/latest/custom-catalog/).

---

## Summary

- The **catalog** maps **logical table name (namespace + table)** to the **current metadata file location**. It does not store table data or full metadata history.
- **Metadata and data** live in object storage/HDFS; the catalog only stores the pointer to the current metadata file.
- **Built-in options** include Hive, Hadoop, REST, Glue, JDBC, and Nessie; you can also implement a **custom catalog**.
- **Read path:** resolve name → metadata location in catalog, then load metadata and data as per the Iceberg spec.
- **Write path:** write new metadata, then ask the catalog to **atomically** update the metadata location for the table.

Understanding the catalog’s role makes it easier to reason about multi-engine setups, migrations, and where to plug in your own catalog or REST/Glue/JDBC.
