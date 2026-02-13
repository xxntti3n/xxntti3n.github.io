---
layout: article
title: "Iceberg Branches and Tags — Deep Dive"
date: 2025-02-17
author: Tien Nguyen
excerpt: "Git-like branches and tags in Iceberg: mutable vs immutable refs, WAP, parallel development, and how they are stored in metadata."
reading_time: 10
categories: [iceberg]
permalink: /iceberg/branches-and-tags/
---

Iceberg supports **branches** and **tags** as named references to snapshots, similar to Git. Branches are mutable (they can move to newer snapshots); tags are immutable (they always point to the same snapshot). This enables parallel development, release pinning, and patterns like Write-Audit-Publish (WAP).

---

## 1. Branches vs tags at a glance

```
┌─────────────────────────────────────────────────────────────────────┐
│  BRANCH: mutable pointer                                            │
│  • Points to a snapshot now; can be updated to a newer snapshot      │
│  • Has optional retention (e.g. keep min snapshots, max age)        │
│  • Use: main, feature-x, audit-branch                               │
├─────────────────────────────────────────────────────────────────────┤
│  TAG: immutable pointer                                             │
│  • Always points to the same snapshot                                │
│  • Use: v1.0.0, release-2024-01-01                                  │
└─────────────────────────────────────────────────────────────────────┘
```

*Suggested image: snapshot timeline with branch “main” moving and tag “v1.0” fixed.*

---

## 2. How they are stored

References live in the table metadata under **refs**:

```json
"refs": {
  "main": { "snapshot-id": 106, "type": "branch", "min-snapshots-to-keep": 1 },
  "v1.0.0": { "snapshot-id": 102, "type": "tag" }
}
```

The catalog (or the table’s metadata location) is the source of truth; engines resolve a branch or tag name to a snapshot-id, then load that snapshot’s manifest list and data.

---

## 3. Typical use cases

- **main branch**: Default branch; most queries and jobs read from `main`.
- **Feature / dev branch**: Write and validate changes without affecting `main`; merge or promote when ready.
- **Tags**: Mark releases or critical points (e.g. `release-2024-01-15`) for auditing or rollback.
- **WAP (Write-Audit-Publish)**: Write to a branch, run checks, then update `main` to that snapshot (publish).

---

## 4. Operations (conceptual)

- **Create branch**: Register a new ref pointing to a snapshot (or current).
- **Create tag**: Register an immutable ref pointing to a snapshot.
- **Use branch for reads**: Query table `FOR BRANCH branch_name` (or engine-specific option).
- **Use branch for writes**: Some catalogs support committing to a branch so the branch pointer advances.
- **Update main to a snapshot**: Effectively “publish” by moving `main` to the snapshot that was written on another branch.

Syntax is engine- and catalog-specific (e.g. Spark procedures, Flink, or REST catalog). Check your engine’s docs for `CREATE BRANCH`, `CREATE TAG`, and how to read/write by branch.

---

## 5. Retention (branches)

Branch refs can specify retention so that when the branch moves forward, old snapshots can be expired (e.g. keep at least N snapshots, or expire older than T). That keeps metadata and storage under control while still allowing time travel on the branch within the retention window.

---

## 6. Summary

| Ref type | Mutable? | Typical use                    |
|----------|----------|---------------------------------|
| Branch   | Yes      | main, feature branches, WAP    |
| Tag      | No       | Releases, audit points         |

For how snapshots and time travel work under the hood, see [Snapshots and Time Travel](/iceberg/snapshots-time-travel/). For the role of the catalog in resolving table and refs, see [Understanding Apache Iceberg Catalogs](/iceberg/iceberg-catalog/).
