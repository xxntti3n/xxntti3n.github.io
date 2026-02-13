---
layout: article
title: "Iceberg Compaction and Maintenance — Deep Dive"
date: 2025-02-18
author: Tien Nguyen
excerpt: "Expire snapshots, rewrite manifests, rewrite data files, and delete orphan files. Keep metadata and storage healthy with examples."
reading_time: 12
categories: [iceberg]
permalink: /iceberg/compaction-and-maintenance/
---

Keeping Iceberg tables healthy means expiring old snapshots, rewriting manifests and data files when needed, and removing orphan files. This post covers what each operation does and when to run it, with concrete examples.

---

## 1. Why maintenance matters

- **Snapshots** accumulate with every commit; without expiration, metadata grows and time-travel history can become huge.
- **Manifests** can become many small files; rewriting them improves planning and pruning.
- **Data files** can become small and numerous (e.g. after many small writes or merge-on-read); rewriting consolidates them and can apply delete files so future reads are cheaper.
- **Orphan files** (leftover from failed jobs or old snapshots) waste storage and should be removed with care.

---

## 2. Expire snapshots

**What it does**: Removes snapshot entries from metadata older than a given time (or beyond a retention policy). Data files that are **only** referenced by expired snapshots become candidates for physical deletion (e.g. by a separate orphan file cleanup).

**Example (Spark procedure style):**

```sql
CALL catalog.system.expire_snapshots(
  table => 'db.orders',
  older_than => TIMESTAMP '2024-01-01 00:00:00'
);
```

**Example (Java API):**

```java
table.expireSnapshots()
     .expireOlderThan(System.currentTimeMillis() - 7 * 24 * 60 * 60 * 1000L)
     .commit();
```

**Best practice**: Run regularly (e.g. daily) with a retention window that matches your time-travel and compliance needs (e.g. 7 days).

---

## 3. Rewrite manifests

**What it does**: Combines small manifest files into fewer, larger manifests. Improves planning and partition pruning because the engine reads fewer, better-organized manifest files.

**When**: After many small commits (e.g. streaming) when you have a large number of manifest files.

**Example (Spark procedure):**

```sql
CALL catalog.system.rewrite_manifests('db.orders');
```

---

## 4. Rewrite data files (compaction)

**What it does**: Reads a set of data files (and their delete files), merges them into fewer, larger data files, and optionally applies delete files so the new files have no delete files (merge-on-read → copy-on-write for those rows). Reduces small-file count and read-time merge cost.

**When**: After many small writes or when merge-on-read has accumulated many delete files.

**Example (Spark procedure):**

```sql
CALL catalog.system.rewrite_data_files(
  table => 'db.orders',
  where => 'partition = 20240101'
);
```

You can limit scope by partition or run full-table rewrite; the procedure picks files to combine based on size/count.

---

## 5. Remove old metadata files

Each commit creates a new metadata file. Old metadata files can be removed once no longer needed for history. Some catalogs or table properties support “delete previous metadata after commit” (e.g. keep last N). Untracked or very old metadata files can be removed by **orphan file deletion** (see below) if your tool supports it and you’re careful not to delete the current one.

---

## 6. Delete orphan files

**What it does**: Deletes files in the table directory that are not referenced by any valid snapshot or metadata (e.g. leftover from failed writes or expired snapshots).

**When**: Run after expiring snapshots and with a safety horizon (e.g. only delete files older than 24 hours) to avoid removing in-progress writes.

**Example (Spark procedure):**

```sql
CALL catalog.system.remove_orphan_files(
  table => 'db.orders',
  older_than => TIMESTAMP '2024-01-15 00:00:00'
);
```

**Caution**: Ensure no concurrent jobs are still writing; use a conservative `older_than` so you don’t delete active files.

---

## 7. Suggested maintenance schedule

| Operation           | Frequency   | Purpose |
|--------------------|------------|---------|
| Expire snapshots   | Daily      | Limit history and free unreferenced data |
| Rewrite manifests  | Weekly or as needed | Reduce manifest count |
| Rewrite data files | Weekly or as needed | Consolidate small files, apply deletes |
| Remove orphan files | After expire_snapshots | Clean storage |

---

## 8. Summary

| Operation          | Effect |
|--------------------|--------|
| Expire snapshots   | Drop old versions from metadata; allow file cleanup |
| Rewrite manifests  | Fewer, larger manifest files |
| Rewrite data files | Fewer, larger data files; apply delete files |
| Remove orphan files | Delete unreferenced files in table location |

For how snapshots and expiration interact with time travel, see [Snapshots and Time Travel](/iceberg/snapshots-time-travel/). For how row-level deletes and merge-on-read work, see [Row-Level Updates and Deletes](/iceberg/row-level-updates-deletes/).
