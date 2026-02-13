---
layout: category
title: Iceberg
description: Apache Iceberg — open table format for large analytic datasets. Catalogs, schema evolution, and lakehouse patterns.
category: iceberg
icon: "🧊"
---

Guides and articles on **Apache Iceberg**: catalogs, table format, schema evolution, and integrating with Spark, Flink, and data lakes.

**Components & topics** (each has a deep-dive page with examples and diagrams):

| Component | Description |
|-----------|--------------|
| [Iceberg Catalog](/iceberg/iceberg-catalog/) | What the catalog stores, catalog types (Hive, REST, JDBC, etc.), and how engines use it |
| [Table Metadata & File Layout](/iceberg/table-metadata-file-layout/) | metadata.json, manifest list, manifests, data/delete files, read and write flow |
| [Schema Evolution](/iceberg/schema-evolution/) | Add/drop/rename columns, type widening, reorder; field IDs and no-rewrite evolution |
| [Partitioning](/iceberg/partitioning/) | Hidden partitioning, transforms (day, month, bucket), partition evolution |
| [Snapshots & Time Travel](/iceberg/snapshots-time-travel/) | Snapshots, query past version, rollback, snapshot expiration |
| [Branches & Tags](/iceberg/branches-and-tags/) | Git-like refs, WAP, parallel development, release tags |
| [Row-Level Updates & Deletes](/iceberg/row-level-updates-deletes/) | Copy-on-write vs merge-on-read, position vs equality deletes |
| [Compaction & Maintenance](/iceberg/compaction-and-maintenance/) | Expire snapshots, rewrite manifests/data files, orphan file removal |
