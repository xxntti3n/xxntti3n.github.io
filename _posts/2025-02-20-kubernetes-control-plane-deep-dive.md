---
layout: article
title: "Kubernetes Control Plane Deep Dive — API Server, etcd, Scheduler, Controllers"
date: 2025-02-20
author: Tien Nguyen
excerpt: "How the control plane really works: API server request flow, etcd storage, scheduler placement, and controller reconciliation. With flags, HA, and debugging commands."
reading_time: 18
categories: [kubernetes]
permalink: /kubernetes/control-plane-deep-dive/
---

The Kubernetes control plane is the brain of the cluster. This deep dive walks through each component—API server, etcd, scheduler, and controller manager—with how they interact, how to inspect them, and what to tune in production.

---

## Control plane at a glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE COMPONENTS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   kubectl / kubelet / controllers                                           │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────┐     ┌─────────┐     ┌──────────────────────────────┐ │
│   │   API Server    │────►│  etcd   │     │  Scheduler                     │ │
│   │   (kube-apiserver)│     │ (state) │     │  (assigns Pods to Nodes)      │ │
│   └────────┬────────┘     └─────────┘     └──────────────────────────────┘ │
│            │                                                                 │
│            ▼                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │   Controller Manager (deployment, replica, namespace, PV, ...)       │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

All cluster changes go through the **API server**. It validates requests, persists state in **etcd**, and triggers the **scheduler** and **controllers** to reconcile the desired state.

---

## 1. API server (kube-apiserver)

The API server is the only component that talks to etcd. Every `kubectl` command, kubelet watch, or controller action is an HTTP request to the API server.

### Request flow

1. **Authentication** — Who are you? (client cert, bearer token, or service account)
2. **Authorization** — Are you allowed to do this? (RBAC, Node, ABAC)
3. **Admission** — Mutating and validating webhooks (e.g. inject sidecar, enforce policy)
4. **Schema validation** — Object matches the API version schema
5. **Persistence** — Write to etcd (for create/update/delete) or read from etcd (for get/list/watch)

### Inspecting the API server

```bash
# API server is typically a static Pod or systemd service
kubectl get pods -n kube-system -l component=kube-apiserver

# Check which admission plugins are enabled (often visible in manifest or flags)
kubectl get pod -n kube-system -l component=kube-apiserver -o yaml | grep -A 50 "admission"

# Health
kubectl get --raw /livez
kubectl get --raw /readyz
```

### Common flags (for ops)

| Flag | Purpose |
|------|---------|
| `--etcd-servers` | etcd client URLs |
| `--service-cluster-ip-range` | CIDR for Service ClusterIPs |
| `--enable-admission-plugins` | Which admission controllers to run |
| `--audit-log-path` | Audit log file for security/compliance |

---

## 2. etcd

etcd is the key-value store holding all cluster state: Pods, Services, ConfigMaps, Secrets, Deployments, and so on.

### Data layout

Keys are hierarchical, for example:

- `/registry/pods/<namespace>/<name>`
- `/registry/services/<namespace>/<name>`
- `/registry/deployments/<namespace>/<name>`

Values are serialized API objects (often protobuf).

### Operations you might run (with etcdctl)

```bash
# If you have etcdctl and certs (e.g. on a control-plane node)
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# List keys under registry (read-only inspection)
etcdctl get /registry --prefix --keys-only | head -20

# Check etcd health
etcdctl endpoint health
```

### HA and backup

- Run 3 or 5 etcd members (odd for quorum).
- Back up etcd regularly (e.g. `etcdctl snapshot save`) and test restore.
- API server uses etcd for all reads/writes; if etcd is down or lost, the cluster cannot function.

---

## 3. Scheduler (kube-scheduler)

The scheduler assigns **Pods** that have no `nodeName` to **Nodes**. It does not start containers; it only writes `nodeName` into the Pod spec. The kubelet on that node then runs the Pod.

### Scheduling pipeline (simplified)

1. **Filter (predicates)** — Which nodes *can* run the Pod? (resources, node selector, taints/tolerations, affinity, PV topology)
2. **Score (priorities)** — Rank feasible nodes (e.g. spread across zones, prefer less loaded nodes)
3. **Bind** — Write the chosen node into the Pod’s `spec.nodeName` via the API server

### Viewing scheduler decisions

```bash
# Why a Pod is pending
kubectl describe pod <pod> -n <ns>
# Events will show "Scheduled" or "FailedScheduling" and often the reason (e.g. insufficient CPU)

# Scheduler metrics (if exposed)
# scheduler_binding_duration_seconds, scheduler_scheduling_attempt_duration_seconds, etc.
```

### Customizing behavior

- **Node affinity / anti-affinity** — Prefer or require certain nodes.
- **Taints and tolerations** — Reserve nodes for specific workloads; only Pods with matching tolerations get scheduled.
- **Pod affinity / anti-affinity** — Co-locate or spread Pods relative to other Pods.

---

## 4. Controller manager (kube-controller-manager)

Controllers watch the API server for changes and try to make the **current state** match the **desired state**. Examples:

| Controller | Watches | Acts |
|------------|--------|------|
| Deployment | Deployment, ReplicaSet | Create/update ReplicaSets to match replicas and strategy |
| ReplicaSet | ReplicaSet, Pod | Create/delete Pods to match replicas |
| Node | Node | Set conditions, evict Pods when node not ready |
| PersistentVolume | PV, PVC | Bind PVC to PV; delete PV when PVC is gone (if reclaim policy allows) |
| Namespace | Namespace | Delete all objects in namespace when namespace is deleted |

They all follow the same pattern: **watch → diff desired vs current → call API server to create/update/delete**.

### Where it runs

On many installs, a single process `kube-controller-manager` runs multiple controllers in one binary. You can also run controllers out-of-tree (e.g. in separate Pods) as long as they have RBAC and talk to the API server.

---

## 5. Putting it together: create a Deployment

1. You run `kubectl apply -f deployment.yaml`.
2. **kubectl** sends a POST to the API server: create this Deployment.
3. **API server** validates, runs admission, writes the Deployment to **etcd**.
4. **Deployment controller** (in controller manager) sees the new Deployment, creates a ReplicaSet.
5. **ReplicaSet controller** sees the new ReplicaSet, creates Pod specs (no `nodeName` yet).
6. **Scheduler** sees Pods with no `nodeName`, picks nodes, patches Pods with `nodeName`.
7. **Kubelet** on each node sees Pods assigned to it, pulls images and starts containers.

So: **API server + etcd** for state; **scheduler** for placement; **controllers** for creating and updating child objects.

---

## 6. High availability (HA)

- **API server** — Run multiple replicas behind a load balancer; they are stateless and all talk to the same etcd.
- **etcd** — Run 3 or 5 members with a separate cluster; API server points to the etcd client endpoints.
- **Scheduler / controller manager** — Run multiple instances with **leader election**; only the leader runs the scheduling/control loops.

---

## 7. Debugging tips

- **API server down** — `kubectl` and all control-plane traffic fail. Check API server Pods and LB.
- **etcd down or degraded** — API server can’t read/write; cluster is effectively down. Check etcd health and quorum.
- **Scheduler not scheduling** — Check `kubectl describe pod` events; often resource limits, taints, or affinity.
- **Controllers not reconciling** — Check controller logs (e.g. `kube-controller-manager`); often RBAC or API errors.

For day-to-day workload authoring, see [Pods Deep Dive](/kubernetes/pods-deep-dive/) and [Deployments & Rolling Updates](/kubernetes/deployments-rolling-updates/). For the big picture, see the [Kubernetes guide](/kubernetes/kubernetes-guide/).
