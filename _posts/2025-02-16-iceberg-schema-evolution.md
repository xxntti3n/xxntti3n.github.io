---
layout: article
title: "Iceberg Schema Evolution — Deep Dive"
date: 2025-02-16
author: Tien Nguyen
excerpt: "How Iceberg evolves table schema without rewriting data: field IDs, add/drop/rename column, type widening, and reorder. With examples and limitations."
reading_time: 12
categories: [iceberg]
permalink: /iceberg/schema-evolution/
---

Schema evolution in Iceberg means changing the table structure—adding or dropping columns, renaming, changing types—**without rewriting data**. This is possible because Iceberg tracks columns by **field IDs** in metadata; data files stay as-is and are interpreted using the current schema.

---

## 1. Why schema evolution matters

In traditional systems, changing schema often means:

- Rewriting the entire table (expensive, slow)
- Or maintaining separate “versioned” tables and complex ETL

In Iceberg:

- You change **metadata** (schema versions); data files are untouched.
- Reads use the **current** schema; old data is projected correctly using field IDs.
- Evolution is **fast** (seconds) and **safe** (no accidental wrong column mapping).

```
Traditional:  Change schema → Rewrite entire table → Hours ✗
Iceberg:      Change schema → Update metadata     → Seconds ✓
```

---

## 2. The key: immutable field IDs

Every column has a **field ID** that never changes for that logical column. Data files store columns by position/name implied by the schema at write time; the schema says which field ID each column has. When you rename a column, only the **name** in the schema changes; the field ID stays the same, so existing data still lines up.

```
Schema v1:   id (ID 1), name (ID 2), amount (ID 3)
Schema v2:   id (ID 1), customer_name (ID 2), amount (ID 3)   ← rename
             Old data still references ID 2 → now displayed as customer_name ✓
```

*Suggested image: diagram showing schema v1 → v2 with same field IDs, different names.*

---

## 3. Supported schema changes (with examples)

### 3.1 Add column

```sql
-- Optional column (no default needed)
ALTER TABLE db.orders ADD COLUMN region STRING;

-- Required column (must have default)
ALTER TABLE db.orders ADD COLUMN priority INT NOT NULL DEFAULT 0;
```

- New column gets the next available field ID.
- Existing data files don’t contain the column; readers see `NULL` (or the default for required columns).
- No data rewrite.

### 3.2 Drop column

```sql
ALTER TABLE db.orders DROP COLUMN deprecated_flag;
```

- Column is removed from the current schema; its field ID is not reused.
- You **cannot** drop partition columns or identifier (primary key) columns this way without changing partitioning/identifiers first.

### 3.3 Rename column

```sql
ALTER TABLE db.orders RENAME COLUMN name TO customer_name;
```

- Only the name in the schema changes; **field ID is preserved**.
- No data rewrite; old files still “have” that column under the same ID.

### 3.4 Change column type (widening only)

Iceberg allows **safe** type promotions so existing values still fit:

| From       | To        | Allowed |
|-----------|-----------|--------|
| `int`     | `long`    | ✅     |
| `float`   | `double`  | ✅     |
| `decimal(9,2)` | `decimal(18,2)` | ✅ (more precision) |
| `string`  | `int`     | ❌     |
| `int`     | `string`  | ❌     |

```sql
ALTER TABLE db.orders ALTER COLUMN quantity TYPE BIGINT;
```

### 3.5 Make column optional or required

```sql
ALTER TABLE db.orders ALTER COLUMN comment DROP NOT NULL;
ALTER TABLE db.orders ALTER COLUMN id SET NOT NULL;  -- only if default exists
```

- Required → optional: always allowed.
- Optional → required: only if the column has a default (so existing rows get that value).

### 3.6 Reorder columns

```sql
ALTER TABLE db.orders ALTER COLUMN created_at FIRST;
ALTER TABLE db.orders ALTER COLUMN status AFTER order_id;
```

- Only affects display/order in the current schema; field IDs unchanged.
- No data rewrite.

---

## 4. How it looks in metadata

The table metadata stores **all** schema versions. Each snapshot references a **schema-id**. Example (conceptual):

```json
{
  "current-schema-id": 2,
  "schemas": [
    { "id": 0, "fields": [{"id": 1, "name": "id", "type": "long"}, {"id": 2, "name": "name", "type": "string"}] },
    { "id": 1, "fields": [{"id": 1, "name": "id", "type": "long"}, {"id": 2, "name": "name", "type": "string"}, {"id": 3, "name": "amount", "type": "decimal(10,2)"}] },
    { "id": 2, "fields": [{"id": 1, "name": "id", "type": "long"}, {"id": 2, "name": "customer_name", "type": "string"}, {"id": 3, "name": "amount", "type": "decimal(10,2)"}] }
  ]
}
```

- Schema 0: `id`, `name`
- Schema 1: add `amount` (field ID 3)
- Schema 2: rename `name` → `customer_name` (still ID 2)

Old snapshots keep pointing to older schema-ids; time travel still works with the right schema for that snapshot.

---

## 5. What to avoid

- **Don’t** change type in a way that can’t be interpreted from existing data (e.g. string → int).
- **Don’t** drop partition columns or identifier columns without updating table properties/specs.
- **Don’t** reuse field IDs (the engine and metadata handle IDs; you only do DDL).

---

## 6. Summary

| Operation        | Data rewrite? | Key point                    |
|-----------------|---------------|------------------------------|
| Add column      | No            | New field ID; NULL/default for old data |
| Drop column     | No            | Removed from schema only     |
| Rename column   | No            | Same field ID, new name      |
| Widen type      | No            | Only safe promotions         |
| Reorder         | No            | Metadata only                |

Schema evolution is one of Iceberg’s strengths for lakehouse and data engineering: you can adapt tables over time without massive rewrites. For how the rest of the table is laid out, see [Table Metadata and File Layout](/iceberg/table-metadata-file-layout/); for changing partition layout over time, see [Partitioning](/iceberg/partitioning/).
