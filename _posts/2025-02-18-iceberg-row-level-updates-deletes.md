---
layout: article
title: "Iceberg Row-Level Updates and Deletes — Deep Dive"
date: 2025-02-18
author: Tien Nguyen
excerpt: "How Iceberg implements UPDATE and DELETE: copy-on-write vs merge-on-read, position deletes vs equality deletes, and when to use each."
reading_time: 15
categories: [iceberg]
permalink: /iceberg/row-level-updates-deletes/
---

Apache Iceberg supports row-level UPDATE and DELETE. Under the hood, deletes are represented by **delete files** (position deletes or equality deletes), and the engine can either **rewrite data files** (copy-on-write) or **apply deletes at read time** (merge-on-read). This deep dive explains both strategies and the delete file types, with examples.

---

## 1. Two strategies: copy-on-write vs merge-on-read

```
┌─────────────────────────────────────────────────────────────────────┐
│  COPY-ON-WRITE (CoW)                                                 │
│  • Rewrite entire data files with deletes applied                     │
│  • No delete files left; reads are simple                             │
│  • Higher write I/O, lower read cost                                  │
├─────────────────────────────────────────────────────────────────────┤
│  MERGE-ON-READ (MoR)                                                 │
│  • Write delete files (position or equality); apply at read time     │
│  • Lower write I/O, higher read cost (merge pass)                     │
│  • Compaction can later rewrite files to remove delete files          │
└─────────────────────────────────────────────────────────────────────┘
```

*Suggested image: CoW = one data file in, one new data file out; MoR = data file + delete file, reader merges.*

---

## 2. Delete file types

### 2.1 Position deletes

Each record is a **(file path, row position)**. “Remove row at position N in file F.”

- **Pros**: Simple, works for any row; no need to know key columns.
- **Cons**: Tied to a specific data file; if the file is compacted or rewritten, positions change (so position deletes are usually rewritten too).

Format (conceptual): e.g. Parquet with `file_path`, `pos`, and optionally full row.

### 2.2 Equality deletes

Each record is a **row key** (one or more column values). “Remove any row where (col1, col2) = (v1, v2).”

- **Pros**: Independent of file layout; good for key-based deletes (e.g. primary key).
- **Cons**: Reader must apply by joining/key lookup; need to choose the right columns (often identifier columns).

Format: Parquet with the same columns as the equality delete spec (e.g. `id`).

---

## 3. When to use which strategy

| Scenario                    | Prefer           | Reason |
|----------------------------|------------------|--------|
| Delete &lt; 20–30% of rows  | Merge-on-read    | Avoid rewriting whole files |
| Delete &gt; 20–30% of rows  | Copy-on-write    | Rewrite cost similar to long-term read cost |
| Read-heavy, few updates     | Copy-on-write    | No read-time merge |
| Write-heavy / streaming     | Merge-on-read    | Lower write latency; compact later |
| Key-based DELETE/UPDATE     | Equality deletes | Natural fit for identifier columns |

Engines choose position vs equality (and CoW vs MoR) based on table properties and the kind of operation (e.g. “DELETE WHERE id = 5” → equality delete on `id` if supported).

---

## 4. How UPDATE is implemented

UPDATE is **DELETE + INSERT**: mark matching rows as deleted (via delete files or CoW rewrite), then write new rows. So the same delete semantics (position or equality) and strategies (CoW vs MoR) apply.

---

## 5. Example: SQL and Spark

**DELETE (conceptual):**

```sql
DELETE FROM db.orders WHERE status = 'cancelled';
-- Engine may produce equality deletes on status or position deletes
```

**UPDATE (conceptual):**

```sql
UPDATE db.orders SET status = 'shipped' WHERE order_id = 12345;
-- Engine: delete row(s) for order_id 12345, insert new row(s)
```

In Spark you can tune behavior via table properties (e.g. prefer copy-on-write vs merge-on-read, or delete file format).

---

## 6. Read path with deletes

1. Read manifest list and manifests for the current snapshot.
2. Collect data files and delete files (position and equality) from manifests.
3. For each data file:
   - Apply **position deletes**: skip rows whose (file_path, pos) is in a position-delete file.
   - Apply **equality deletes**: skip rows whose key columns match any row in an equality-delete file.
4. Return remaining rows.

Compaction (rewrite) can merge data files and delete files into new data files with no delete files, simplifying future reads.

---

## 7. Summary

| Concept           | Description                                              |
|------------------|----------------------------------------------------------|
| Copy-on-write     | Rewrite data files; no delete files; simple reads        |
| Merge-on-read     | Keep delete files; apply at read; compact later         |
| Position delete   | (file_path, row position)                                |
| Equality delete   | Key column values; good for primary-key deletes        |
| UPDATE            | DELETE + INSERT                                         |

For cleaning up delete files and optimizing layout, see [Compaction and Maintenance](/iceberg/compaction-and-maintenance/). For how delete files are stored in the metadata and manifest layer, see [Table Metadata and File Layout](/iceberg/table-metadata-file-layout/).
