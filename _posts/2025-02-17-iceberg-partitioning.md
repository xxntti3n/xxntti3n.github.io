---
layout: article
title: "Iceberg Partitioning — Hidden Partitioning and Evolution"
date: 2025-02-17
author: Tien Nguyen
excerpt: "How Iceberg partitioning works: hidden partitioning, partition specs, transforms, and partition evolution. With SQL and Spark examples."
reading_time: 11
categories: [iceberg]
permalink: /iceberg/partitioning/
---

Iceberg partitioning speeds up queries by grouping rows into files by partition value. Unlike Hive, Iceberg uses **hidden partitioning**: you define transforms (e.g. day(ts)) in the table spec; writers and readers don’t manage partition columns by hand, and you can **evolve** the partition spec without rewriting the whole table.

---

## 1. Why “hidden” partitioning?

In Hive-style tables you must:

- Add an explicit partition column (e.g. `event_date`) and populate it correctly on write.
- Remember to filter on that column in every query; otherwise you get full scans.

In Iceberg:

- You partition by **transforms** on columns (e.g. `day(event_time)`). There is no separate partition column in the row.
- The engine derives partition values from the data and records them in metadata.
- The engine **translates** predicates (e.g. `event_time BETWEEN ...`) into partition filters and skips files automatically.

So: no manual partition column, no silent bugs from wrong format or missing filters, and partition layout can change over time (partition evolution).

---

## 2. Partition transforms

Common transforms:

| Transform     | Source type   | Example partition value | Use case        |
|---------------|---------------|--------------------------|-----------------|
| `identity`    | any           | same as column           | Category, region |
| `year(ts)`    | timestamp/date | `2024`                   | Coarse time     |
| `month(ts)`   | timestamp/date | `2024-01`                | Monthly rollups |
| `day(ts)`     | timestamp/date | `2024-01-15`             | Daily tables    |
| `hour(ts)`    | timestamp     | `2024-01-15T10`          | Event streams   |
| `bucket(n, col)` | any        | `0..n-1`                 | Skew reduction |
| `truncate(l, col)` | string/int  | truncated value          | Prefix grouping |

Example: partition by day of `event_time` and by `level`:

```sql
CREATE TABLE logs (
  id BIGINT,
  event_time TIMESTAMP,
  level STRING,
  message STRING
)
USING iceberg
PARTITIONED BY (days(event_time), level);
```

Writes only need `event_time` and `level`; partition values are derived. Queries like `WHERE event_time BETWEEN '2024-01-01' AND '2024-01-07'` are used to prune partitions automatically.

*Suggested image: diagram of table layout with data/date=2024-01-01/level=ERROR/ and files inside.*

---

## 3. Partition spec in metadata

The table’s partition spec is stored in metadata (with a partition-spec-id). Each data file’s manifest entry includes partition values for that spec. When you **evolve** the partition spec (e.g. add a new partition field), new writes use the new spec; old data keeps its old spec. No full table rewrite required.

---

## 4. Partition evolution example

Start with daily partitioning:

```sql
-- Initial: partition by day
CREATE TABLE events (id BIGINT, ts TIMESTAMP, name STRING)
USING iceberg
PARTITIONED BY (days(ts));
```

Later, add monthly partitioning (e.g. for a new rollup). In Spark 3.x with Iceberg:

```sql
ALTER TABLE events ADD PARTITION FIELD months(ts);
```

New data will be written with both `day(ts)` and `month(ts)`; existing data stays day-only. Queries can still filter on `ts`; the engine uses whatever partition info exists per file.

---

## 5. Best practices

- **Time-based**: Prefer `days(ts)` or `hours(ts)` for event/log tables so range queries prune well.
- **High cardinality**: Add `bucket(n, id)` (or similar) to avoid too many small partitions.
- **Evolution**: Prefer adding new partition fields over changing existing ones so old data remains valid.

---

## 6. Summary

| Concept            | Iceberg behavior                                      |
|--------------------|--------------------------------------------------------|
| Partition value    | Derived by transform; not a physical column            |
| Predicate pushdown | Engine maps column predicates to partition pruning    |
| Evolution          | New spec for new writes; old data unchanged            |

For how these partitions are stored in manifests and data paths, see [Table Metadata and File Layout](/iceberg/table-metadata-file-layout/). For versioning and rollback of table state, see [Snapshots and Time Travel](/iceberg/snapshots-time-travel/).
