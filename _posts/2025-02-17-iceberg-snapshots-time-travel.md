---
layout: article
title: "Iceberg Snapshots and Time Travel — Deep Dive"
date: 2025-02-17
author: Tien Nguyen
excerpt: "How Iceberg snapshots work, how to query past versions (time travel), and how to roll back. With SQL examples and snapshot lifecycle."
reading_time: 10
categories: [iceberg]
permalink: /iceberg/snapshots-time-travel/
---

Every write to an Iceberg table creates a **snapshot**: a consistent view of the table at a point in time. Snapshots enable **time travel** (query an older version) and **rollback** (make the table look like a previous snapshot). This post covers how snapshots are stored, how to use them, and how to manage their lifecycle.

---

## 1. What is a snapshot?

A snapshot is an immutable pointer to:

- A **manifest list** (which lists all manifest files for that version)
- **Schema id** and **partition spec id** for that version
- **Metadata**: snapshot id, timestamp, summary (e.g. added/deleted file counts)

The table’s current metadata file holds `current-snapshot-id` and a list of all snapshots. Each snapshot points to one manifest list; that list points to manifests; manifests point to data (and delete) files. So “snapshot” = one consistent set of files for the table at that time.

```
metadata.json
  current-snapshot-id: 456
  snapshots:
    - snapshot-id: 123, timestamp-ms: ..., manifest-list: snap-123.avro
    - snapshot-id: 456, timestamp-ms: ..., manifest-list: snap-456.avro
```

*Suggested image: timeline of snapshots S1 → S2 → S3 with manifest lists and data files.*

---

## 2. Time travel: query a past version

You can read the table as it was at a given snapshot or time.

**By snapshot id (exact version):**

```sql
SELECT * FROM db.orders FOR VERSION AS OF 1234567890123456789;
```

**By timestamp (first snapshot at or after that time):**

```sql
SELECT * FROM db.orders FOR TIMESTAMP AS OF '2024-01-15 10:00:00';
```

**In Spark (Dataset/DataFrame):**

```scala
spark.read
  .option("snapshot-id", "1234567890123456789")
  .format("iceberg")
  .load("db.orders")

// or by time
spark.read
  .option("as-of-timestamp", "1705312800000")  // ms
  .format("iceberg")
  .load("db.orders")
```

Use cases: debugging (“what did the table look like yesterday?”), reproducible reports, auditing.

---

## 3. Rollback: reset table to a snapshot

Rollback makes the table’s **current** state equal to a chosen snapshot. It does not delete data files; it creates a new snapshot that reuses the same manifest list as the target snapshot (or equivalent), so the table “looks like” that version.

**Spark SQL (example):**

```sql
CALL catalog_name.system.rollback_to_snapshot('db.orders', 1234567890123456789);
```

After rollback, `current-snapshot-id` points to that snapshot (or a snapshot that references the same data). Snapshots after the target remain in history until you expire them; they are no longer “current.”

---

## 4. Snapshot lifecycle and expiration

Snapshots accumulate until you **expire** them. Expiring snapshots:

- Removes them from metadata (so they are no longer available for time travel or rollback).
- Allows physical deletion of data files that are **only** referenced by expired snapshots (e.g. via maintenance jobs).

Example (Java API style; Spark has similar procedures):

```java
table.expireSnapshots()
     .expireOlderThan(System.currentTimeMillis() - 7 * 24 * 60 * 60 * 1000L)  // 7 days
     .commit();
```

Best practice: run expiration regularly (e.g. daily) with a retention window that matches your time-travel and compliance needs.

---

## 5. Summary

| Concept      | Description                                              |
|-------------|----------------------------------------------------------|
| Snapshot    | Immutable table state: one manifest list + schema/spec   |
| Time travel | Read table as of snapshot-id or timestamp               |
| Rollback    | Set current table state to a previous snapshot           |
| Expiration  | Remove old snapshots and free unreferenced data files    |

For branches and tags (named references to snapshots), see [Branches and Tags](/iceberg/branches-and-tags/). For cleaning up metadata and files after expiration, see [Compaction and Maintenance](/iceberg/compaction-and-maintenance/).
