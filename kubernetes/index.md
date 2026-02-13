---
layout: category
title: Kubernetes
description: Kubernetes (K8s) — container orchestration, architecture, and production patterns.
category: kubernetes
icon: "🎯"
featured_url: /kubernetes/kubernetes-guide/
featured_title: Kubernetes Explained — What Really Happens Under The Hood
featured_desc: "Complete 75-min guide from basics to production: architecture, pods, services, storage, RBAC, and more."
---

Guides and articles on **Kubernetes**: control plane, pods, deployments, services, and production practices.

**Components & deep dives** (each with examples, YAML, and commands):

| Component | Description |
|-----------|-------------|
| [Kubernetes guide](/kubernetes/kubernetes-guide/) | Full guide: architecture, Pods, Deployments, Services, ConfigMap/Secret, Ingress, RBAC, HPA, storage, taints |
| [Control Plane Deep Dive](/kubernetes/control-plane-deep-dive/) | API server, etcd, scheduler, controller manager — flow, HA, debugging |
| [Pods Deep Dive](/kubernetes/pods-deep-dive/) | Lifecycle, liveness/readiness/startup probes, init containers, sidecars, resources, security context |
| [Services & Networking](/kubernetes/services-networking-deep-dive/) | ClusterIP, NodePort, LoadBalancer, kube-proxy, CoreDNS, headless Services |
| [Deployments & Rolling Updates](/kubernetes/deployments-rolling-updates-deep-dive/) | Strategy, maxSurge/maxUnavailable, rollback, canary, blue-green |
| [RBAC & Security](/kubernetes/rbac-security-deep-dive/) | ServiceAccounts, Roles, ClusterRoles, bindings, least-privilege |
| [Storage (PV, PVC, CSI)](/kubernetes/storage-pv-pvc-deep-dive/) | PersistentVolume, PersistentVolumeClaim, StorageClass, dynamic provisioning, StatefulSet |
