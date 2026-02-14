# Iceberg Category — Components & Content Index

As a senior data engineer and blogger, these are the **Iceberg components and content** that each have a dedicated deep-dive page on the site. Each page includes examples, diagrams (ASCII in post; add SVG/PNG under `assets/images/` as needed), and cross-links.

---

## 1. [Iceberg Catalog](/iceberg/iceberg-catalog/)

- **What it is**: The system that maps (namespace, table name) → metadata location.
- **Content**: What the catalog stores vs what lives in metadata; Hive, Hadoop, REST, JDBC, Glue, custom catalogs; how engines resolve tables.
- **Page**: `/iceberg/iceberg-catalog/`

---

## 2. [Table Metadata & File Layout](/iceberg/table-metadata-file-layout/)

- **What it is**: The chain of metadata.json → snapshot → manifest list → manifests → data/delete files.
- **Content**: File types, hierarchy, read path, write path, example directory layout.
- **Page**: `/iceberg/table-metadata-file-layout/`

---

## 3. [Schema Evolution](/iceberg/schema-evolution/)

- **What it is**: Changing table schema without rewriting data, using field IDs.
- **Content**: Add/drop/rename column, type widening, reorder; how it’s stored in metadata; limitations.
- **Page**: `/iceberg/schema-evolution/`

---

## 4. [Partitioning](/iceberg/partitioning/)

- **What it is**: Hidden partitioning with transforms (day, month, bucket, etc.) and partition evolution.
- **Content**: Transforms, predicate pushdown, partition spec in metadata, evolution example.
- **Page**: `/iceberg/partitioning/`

---

## 5. [Snapshots & Time Travel](/iceberg/snapshots-time-travel/)

- **What it is**: Immutable table versions and querying/rolling back to a snapshot.
- **Content**: What a snapshot is, time travel by snapshot-id and timestamp, rollback, snapshot expiration.
- **Page**: `/iceberg/snapshots-time-travel/`

---

## 6. [Branches & Tags](/iceberg/branches-and-tags/)

- **What it is**: Named refs to snapshots (branch = mutable, tag = immutable).
- **Content**: Use cases (main, feature branches, WAP, release tags), how refs are stored, retention.
- **Page**: `/iceberg/branches-and-tags/`

---

## 7. [Row-Level Updates & Deletes](/iceberg/row-level-updates-deletes/)

- **What it is**: UPDATE/DELETE via copy-on-write or merge-on-read and delete files.
- **Content**: Position vs equality deletes, when to use CoW vs MoR, read path with deletes.
- **Page**: `/iceberg/row-level-updates-deletes/`

---

## 8. [Compaction & Maintenance](/iceberg/compaction-and-maintenance/)

- **What it is**: Keeping tables healthy (expire snapshots, rewrite manifests/data, remove orphans).
- **Content**: Expire snapshots, rewrite manifests, rewrite data files, delete orphan files; suggested schedule.
- **Page**: `/iceberg/compaction-and-maintenance/`

---

## Adding more pages

- **Engine-specific**: e.g. “Spark with Iceberg” (DDL, reads, writes, streaming), “Flink with Iceberg”, “REST Catalog”.
- **Platform**: e.g. “Iceberg on AWS/S3”, “Iceberg with Trino”.
- **Operations**: e.g. “Migration from Hive/Delta”, “Multi-engine consistency”.

Create a new post in `_posts/` with `categories: [iceberg]` and `permalink: /iceberg/your-slug/`, then add it to the table in `iceberg/index.md` and to this list.
