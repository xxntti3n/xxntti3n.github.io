---
layout: article
title: "Kubernetes Storage Deep Dive — PV, PVC, StorageClass, CSI"
date: 2025-02-22
author: Tien Nguyen
excerpt: "PersistentVolume and PersistentVolumeClaim lifecycle, StorageClass, provisioning, and CSI drivers. With YAML and stateful workload patterns."
reading_time: 16
categories: [kubernetes]
permalink: /kubernetes/storage-pv-pvc-deep-dive/
---

Kubernetes storage gives Pods durable volumes that survive Pod restarts. This deep dive covers PersistentVolume (PV), PersistentVolumeClaim (PVC), StorageClass, dynamic provisioning, and how to use them for stateful workloads.

---

## 1. Why PV and PVC?

Containers and Pods are ephemeral; **emptyDir** is lost when the Pod is removed. For databases, logs, or any data that must persist, you use **PersistentVolume (PV)** and **PersistentVolumeClaim (PVC)**.

- **PV** — Cluster resource representing a piece of storage (e.g. a cloud disk or NFS export).
- **PVC** — A request for storage (size, access mode, optional StorageClass). Pods use a PVC by name in `volumes` and `volumeMounts`.
- **Binding** — The control plane binds a PVC to a PV that meets the request (capacity, access mode, StorageClass). One PVC ↔ one PV (1:1).

```
  Pod
   │  volumes: [ persistentVolumeClaim: claimName: data ]
   ▼
  PVC "data" (request: 10Gi, RWO, storageClassName: fast)
   │
   │  bound to
   ▼
  PV (10Gi, RWO, node affinity, CSI volume)
   │
   ▼
  Actual storage (e.g. cloud disk, NFS)
```

---

## 2. Access modes

| Mode | Meaning |
|------|--------|
| **ReadWriteOnce (RWO)** | Single node read-write (typical for block volumes). |
| **ReadOnlyMany (ROX)** | Multiple nodes read-only (e.g. NFS, read-only PVC). |
| **ReadWriteMany (RWX)** | Multiple nodes read-write (e.g. NFS, some filers). |

Many cloud block volumes support only RWO. Check your CSI driver or storage backend.

---

## 3. Static provisioning: admin creates PV

An admin creates a PV that points to existing storage (e.g. NFS server path or cloud volume ID). A user creates a PVC; the control plane binds the PVC to a matching PV.

**PersistentVolume (admin):**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-1
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-server.default.svc.cluster.local
    path: /exports/data1
```

**PersistentVolumeClaim (user/app):**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: prod
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  # no storageClassName = default StorageClass or match PVs with no class
```

**Pod using the PVC:**

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data
```

---

## 4. Dynamic provisioning: StorageClass and CSI

With **dynamic provisioning**, you don’t create PVs by hand. You create a **StorageClass** that describes a *type* of storage (e.g. SSD, cloud provider, CSI driver). When a PVC references that StorageClass (or the default), a **provisioner** (usually a CSI driver) creates the underlying volume and a PV, then binds the PVC to that PV.

**StorageClass example (generic):**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

- **provisioner** — CSI driver or in-tree provisioner (e.g. `pd.csi.storage.gke.io`, `ebs.csi.aws.amazon.com`).
- **volumeBindingMode: WaitForFirstConsumer** — Create the volume only when a Pod using the PVC is scheduled; then the volume is typically created in the same zone as the node.
- **reclaimPolicy** — Delete (delete volume when PVC is deleted) or Retain (keep volume; admin can reuse or delete later).

**PVC that triggers provisioning:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: prod
spec:
  storageClassName: fast
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

No PV is created by the user; the CSI driver creates the disk and the PV, and the PVC is bound automatically.

---

## 5. Reclaim policy

When a **PVC is deleted**:

- **Retain** — PV remains; status becomes `Released`. Data is still on the volume. Admin can manually delete the PV and/or the underlying storage.
- **Delete** — The provisioner deletes the underlying storage and the PV (typical for dynamic provisioning).

---

## 6. StatefulSet and storage

**StatefulSets** give Pods stable names (`pod-0`, `pod-1`, …) and optional **volumeClaimTemplates**. Each replica gets its own PVC (and thus its own PV), so each Pod has its own durable volume.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: fast
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

Result: `db-0` → PVC `data-db-0`, `db-1` → `data-db-1`, etc. Each PVC is bound to its own PV.

---

## 7. Inspecting and debugging

```bash
kubectl get pv
kubectl get pvc -n prod
kubectl describe pvc data -n prod
kubectl describe pv <pv-name>
```

- **PVC Pending** — No PV matches (capacity, access mode, StorageClass) or provisioner failed; check StorageClass and CSI driver logs.
- **PV Released** — PVC was deleted; reclaim policy Retain. Reuse by deleting the PV and creating a new PVC, or re-import the PV with a new PVC (advanced).

---

## 8. Summary

| Concept | Purpose |
|--------|----------|
| PV | Cluster resource for a piece of storage |
| PVC | Request for storage; Pods reference the PVC by name |
| StorageClass | Type of storage + provisioner; enables dynamic provisioning |
| CSI | Standard for volume drivers; most clusters use CSI for cloud volumes |
| volumeClaimTemplate | In StatefulSet: one PVC per replica |
| reclaimPolicy | Retain or Delete when PVC is removed |

For how the control plane binds PVCs to PVs, see [Control Plane Deep Dive](/kubernetes/control-plane-deep-dive/). For the full Kubernetes guide, see [Kubernetes guide](/kubernetes/kubernetes-guide/).
