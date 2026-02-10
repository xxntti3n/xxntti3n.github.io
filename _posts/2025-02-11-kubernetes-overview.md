---
layout: post
title: "Kubernetes Overview: What is K8s?"
date: 2025-02-11
tags: [kubernetes, basics, overview, architecture]
author: DevOps Engineer
---

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications.

### Why K8s?

<div class="concept-box">
<strong>The Problem:</strong> You have 100 containers running across multiple servers. One dies. How do you know? How do you replace it? How do you scale when traffic spikes?<br><br>
<strong>The Solution:</strong> Kubernetes handles all of this automatically.
</div>

---

## K8s Architecture

<div class="diagram-container">
```mermaid
graph TB
    subgraph "Control Plane (The Brain)"
        API[API Server]
        Scheduler[Scheduler]
        Controller[Controller Manager]
        Etcd[(etcd)]
    end

    subgraph "Worker Nodes"
        Node1[Node 1]
        Node2[Node 2]
        Node3[Node 3]
    end

    subgraph "Pods on Node 1"
        Pod1A[Pod A]
        Pod1B[Pod B]
    end

    subgraph "Pods on Node 2"
        Pod2A[Pod C]
        Pod2B[Pod D]
    end

    subgraph "Pods on Node 3"
        Pod3A[Pod E]
    end

    API -->|manage| Node1
    API -->|manage| Node2
    API -->|manage| Node3

    Node1 --> Pod1A
    Node1 --> Pod1B
    Node2 --> Pod2A
    Node2 --> Pod2B
    Node3 --> Pod3A

    Etcd --> API
    Scheduler --> API
    Controller --> API

    style API fill:#e74c3c,stroke:#c0392b,color:#fff
    style Etcd fill:#f39c12,stroke:#e67e22,color:#fff
    style Scheduler fill:#3498db,stroke:#2980b9,color:#fff
    style Controller fill:#9b59b6,stroke:#8e44ad,color:#fff
```
</div>

---

## Control Plane Components

| Component | Responsibility |
|-----------|----------------|
| **API Server** | Frontend to K8s, validates config |
| **etcd** | Key-value store, cluster state |
| **Scheduler** | Decides which node runs pods |
| **Controller Manager** | Maintains desired state |
| **Cloud Controller** | Integrates with cloud providers |

---

## Worker Node Components

| Component | Responsibility |
|-----------|----------------|
| **kubelet** | Agent on each node, talks to API server |
| **kube-proxy** | Network rules, service discovery |
| **Container Runtime** | Runs containers (Docker, containerd) |

---

## K8s Does For You

- ✅ **Self-healing**: Restarts failed containers
- ✅ **Auto-scaling**: Adds/removes pods based on load
- ✅ **Load balancing**: Distributes traffic across pods
- ✅ **Rolling updates**: Zero-downtime deployments
- ✅ **Rollback**: Revert to previous version if something breaks
- ✅ **Secret management**: Securely stores passwords/keys
- ✅ **Storage orchestration**: Mounts storage volumes

---

## Next Steps

Now that you understand the architecture, let's dive into specific components:

1. [Pod](#) - The smallest unit
2. [Deployment](#) - Managing replicas
3. [Service](#) - Networking basics
