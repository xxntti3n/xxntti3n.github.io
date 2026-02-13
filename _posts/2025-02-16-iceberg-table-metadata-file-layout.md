---
layout: article
title: "Iceberg Table Metadata and File Layout — Deep Dive"
date: 2025-02-16
author: Tien Nguyen
excerpt: "How Iceberg organizes metadata.json, manifest lists, manifests, and data files. End-to-end read and write flow with examples and diagrams."
reading_time: 14
categories: [iceberg]
permalink: /iceberg/table-metadata-file-layout/
---

Every Iceberg table is a chain of files: one metadata file points to a snapshot, the snapshot to a manifest list, the list to manifests, and manifests to data (and delete) files. This deep dive walks through each layer with examples and diagrams so you can reason about layout, debugging, and performance.

---

## 1. Overview: The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│              ICEBERG TABLE STRUCTURE: BIG PICTURE                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Table Location (e.g., s3://bucket/table/)                          │
│  │                                                                   │
│  │  metadata.json  ────────────────┐                               │
│  │       │                          │                               │
│  │       ▼                          │                               │
│  │   Current Snapshot ID             │                               │
│  │       │                          │                               │
│  │       ▼                          │                               │
│  │   Snapshot ────────────┐         │                               │
│  │       │                  │         │                               │
│  │       ▼                  │         │                               │
│  │   Manifest List (avro)  │         │                               │
│  │       │                  │         │                               │
│  │       ├──────────────────┤         │                               │
│  │       ▼                  │         │                               │
│  │   Manifests (avro)      │         │                               │
│  │       │                  │         │                               │
│  │       ▼                  │         │                               │
│  │   Data/Delete Files     │         │                               │
│  │                          │         │                               │
│  └──────────────────────────────────┘                               │
│                                                                     │
│  Chain: metadata.json → Snapshot → Manifest List → Manifests → Files │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

*Suggested image: diagram of table folder with metadata/, data/, and file types.*

---

## 2. File Types in Detail

### 2.1 Metadata files

The catalog stores the path to the **current** metadata file (e.g. `metadata/00001-abc123.metadata.json`). That file holds:

- **current-schema-id** — which schema version to use
- **schemas** — all schema versions (for evolution)
- **current-snapshot-id** — latest snapshot
- **snapshots** — all snapshots (snapshot id, manifest list path, schema id, etc.)
- **partition-specs** — partitioning definitions
- **refs** — branches and tags

Example (conceptual):

```json
{
  "format-version": 2,
  "table-uuid": "...",
  "location": "s3://bucket/table/",
  "current-schema-id": 1,
  "current-snapshot-id": 1234567890123456789,
  "snapshots": [
    {
      "snapshot-id": 1234567890123456789,
      "manifest-list": "s3://bucket/table/metadata/snap-123.avro",
      "schema-id": 1
    }
  ],
  "schemas": [...],
  "partition-specs": [...]
}
```

Each commit produces a **new** metadata file; the catalog is updated to point to it. Old metadata files remain for history until cleaned up by maintenance.

### 2.2 Manifest list (per snapshot)

Each snapshot has one manifest list file (Avro). It lives under the table’s metadata path, e.g. `metadata/snapshots/1234567890123456789-0.avro`.

It contains:

- Paths to **manifest files**
- Content type (DATA vs DELETES)
- Partition spec id, sequence number
- Optional partition bounds / stats used for pruning

So: **one snapshot → one manifest list file → many manifest file paths.**

### 2.3 Manifest files

Manifests (Avro) live under the table, e.g. `metadata/abc-def.avro`. Each manifest holds **entries** for data or delete files:

- **Data manifest**: path, format (Parquet/ORC/Avro), partition, record count, file size, column stats (min/max, null counts), etc.
- **Delete manifest**: same idea for position-delete or equality-delete files.

Each entry has a status: ADDED, EXISTING, or DELETED. New writes add entries with ADDED; compaction can merge and then mark old files DELETED.

### 2.4 Data files and delete files

- **Data files**: Parquet (recommended), ORC, or Avro under e.g. `data/` (often with partition subpaths like `data/date=2024-01-15/part-00000.parquet`).
- **Delete files**: position deletes (file path + row position) or equality deletes (key columns). Stored as Parquet (or Puffin for deletion vectors). Referenced by delete manifests.

---

## 3. Query flow: from catalog to rows

1. **Resolve table** — Engine asks catalog for metadata location of the table.
2. **Read metadata** — Load current `metadata.json` (from that location).
3. **Pick snapshot** — Use `current-snapshot-id` (or the one for time travel / branch).
4. **Load manifest list** — Read the snapshot’s manifest list Avro.
5. **Prune manifests** — Use partition stats and predicate to skip manifests that cannot contain matching data.
6. **Read manifests** — Open only the needed manifest files and get data (and delete) file paths and stats.
7. **Prune data files** — Use file-level stats (min/max, null counts) to skip files.
8. **Read data** — Read Parquet (or ORC/Avro) files, apply delete files (merge-on-read), and return rows.

```
┌─────────────────────────────────────────────────────────────────────┐
│  READ PATH: Catalog → metadata → snapshot → manifest list → manifests │
│  → data files (+ apply deletes) → rows                                │
└─────────────────────────────────────────────────────────────────────┘
```

*Suggested image: flow diagram of read path with pruning steps.*

---

## 4. Write flow: how a commit creates the chain

1. **Write new data** — Job writes new data files (and optionally delete files).
2. **Manifest(s)** — Create new manifest file(s) that list these files (ADDED).
3. **Manifest list** — Create a new manifest list pointing to the new manifest(s) and optionally existing ones (for unchanged data).
4. **Snapshot** — New snapshot record references this manifest list and schema/spec ids.
5. **Metadata** — New metadata file with new `current-snapshot-id` and updated `snapshots` (and schemas/specs if changed).
6. **Commit** — Catalog is updated to point to this new metadata file (atomic commit).

---

## 5. Example: listing files for a table

If the table lives at `s3://my-bucket/warehouse/db/orders/`, you might see:

```
orders/
├── metadata/
│   ├── 00000-abc.metadata.json      # old version
│   ├── 00001-def.metadata.json      # current (catalog points here)
│   └── snapshots/
│       ├── 123-0.avro               # manifest list for snapshot 123
│       └── 456-0.avro               # manifest list for snapshot 456
├── data/
│   ├── date=2024-01-01/
│   │   └── part-00000.parquet
│   └── date=2024-01-02/
│       └── part-00000.parquet
└── (manifest files referenced in manifest lists)
```

Using Spark or the Iceberg API you can inspect metadata and list files without parsing JSON/Avro by hand.

---

## 6. What to remember

| Layer           | What it is                          | Purpose                                      |
|----------------|-------------------------------------|----------------------------------------------|
| Metadata file  | JSON (current table state)          | Schema, snapshots, partition specs, refs     |
| Manifest list  | Avro (per snapshot)                 | List of manifest file paths + metadata       |
| Manifest       | Avro (many per snapshot)            | List of data/delete files + stats            |
| Data/delete    | Parquet/ORC/Avro (and Puffin)       | Actual rows and delete records               |

Understanding this chain helps with debugging (which file is slow, which snapshot a query uses), tuning (compaction, expiration), and designing partitioning and schema evolution. For row-level deletes and compaction, see the [Row-Level Updates and Deletes](/iceberg/row-level-updates-deletes/) and [Compaction and Maintenance](/iceberg/compaction-and-maintenance/) deep dives.
