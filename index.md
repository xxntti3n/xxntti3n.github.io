---
layout: default
title: Kubernetes Basics for DevOps Engineers
---

# Kubernetes Basics - Complete Guide

Welcome to my Kubernetes learning journey! This page contains all the K8s concepts every DevOps engineer should know, with diagrams and examples.

---

## 📚 All Posts

<div class="diagram-container" style="text-align:center; margin: 30px 0;">
```mermaid
mindmap
  root((Kubernetes))
    Core
      Overview
      Pod
    Workloads
      Deployment
      StatefulSet
      DaemonSet
      Job
      CronJob
    Networking
      Service
      Ingress
      NetworkPolicy
    Config
      ConfigMap
      Secret
    Storage
      PV
      PVC
      StorageClass
    Security
      RBAC
    Scaling
      HPA
    Scheduling
      Taints
```
</div>

<ul class="index-list">
  <li>
    <a href="/blog/2025/02/11/kubernetes-overview.html">🚀 Kubernetes Overview: What is K8s?</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, basics, overview, architecture</div>
    <p>Introduction to Kubernetes architecture, control plane components, and what K8s does for you.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-pod.html">📦 Kubernetes Pod: The Smallest Unit</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, pod, workload, basics</div>
    <p>Understanding Pods: single and multi-container pods, lifecycle, and pod vs container.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-deployment.html">🔄 Kubernetes Deployment: Managing Pods at Scale</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, deployment, workload, scaling</div>
    <p>Deployments for rolling updates, scaling, and self-healing applications.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-service.html">🔗 Kubernetes Service: Networking Basics</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, service, networking, basics</div>
    <p>Service types (ClusterIP, NodePort, LoadBalancer), DNS, and endpoints.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-configmap-secret.html">⚙️ ConfigMap and Secret: Configuration in Kubernetes</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, configmap, secret, configuration</div>
    <p>Managing configuration and sensitive data separately from container images.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-ingress.html">🌐 Kubernetes Ingress: HTTP/HTTPS Routing</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, ingress, networking, routing</div>
    <p>Ingress controllers, routing rules, TLS termination, and annotations.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-statefulset-daemonset.html">🗄️ StatefulSet and DaemonSet: Specialized Workloads</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, statefulset, daemonset, workload</div>
    <p>StatefulSets for databases, DaemonSets for node agents.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-job-cronjob.html">⏰ Job and CronJob: Scheduled Workloads</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, job, cronjob, batch</div>
    <p>Batch processing jobs and scheduled recurring tasks.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-network-policy.html">🛡️ Kubernetes Network Policy: Traffic Control</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, network-policy, security, networking</div>
    <p>Controlling traffic flow between pods with network policies.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-rbac.html">🔐 Kubernetes RBAC: Access Control</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, rbac, security, authorization</div>
    <p>Role-Based Access Control: Roles, ClusterRoles, and bindings.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-hpa.html">📈 Horizontal Pod Autoscaler (HPA)</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, hpa, autoscaling, scaling</div>
    <p>Auto-scaling pods based on CPU, memory, and custom metrics.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-storage.html">💾 Kubernetes Storage: PV, PVC, StorageClass</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, storage, pv, pvc, volume</div>
    <p>Persistent volumes, claims, and storage classes for data persistence.</p>
  </li>

  <li>
    <a href="/blog/2025/02/11/kubernetes-taints-tolerations.html">🎯 Taints and Tolerations: Pod Scheduling Control</a>
    <div class="date">February 11, 2025 | Tags: kubernetes, taint, toleration, scheduling</div>
    <p>Controlling which pods can schedule on which nodes.</p>
  </li>
</ul>

---

## 📖 Learning Path

I recommend learning these concepts in order:

<div class="diagram-container">
```mermaid
graph LR
    A[1. Overview] --> B[2. Pod]
    B --> C[3. Deployment]
    C --> D[4. Service]
    D --> E[5. ConfigMap/Secret]
    E --> F[6. Ingress]
    F --> G[7. Advanced Topics]

    G --> H[StatefulSet]
    G --> I[Jobs]
    G --> J[NetworkPolicy]
    G --> K[RBAC]
    G --> L[HPA]
    G --> M[Storage]
    G --> N[Taints]
```
</div>

### Core Concepts (Start Here)
1. **Overview** - Architecture and components
2. **Pod** - The smallest deployable unit
3. **Deployment** - Managing replicas and updates

### Networking & Configuration
4. **Service** - Internal networking
5. **ConfigMap/Secret** - Configuration management
6. **Ingress** - External HTTP/HTTPS access

### Advanced Topics
7. **StatefulSet** - For databases
8. **Job/CronJob** - Batch processing
9. **NetworkPolicy** - Traffic control
10. **RBAC** - Access control
11. **HPA** - Auto-scaling
12. **Storage** - Persistent data
13. **Taints/Tolerations** - Scheduling control

---

## 🛠️ Quick Reference

| Concept | Purpose | Link |
|---------|---------|------|
| **Pod** | Smallest unit | [Read →](/blog/2025/02/11/kubernetes-pod.html) |
| **Deployment** | Stateless apps | [Read →](/blog/2025/02/11/kubernetes-deployment.html) |
| **Service** | Stable endpoint | [Read →](/blog/2025/02/11/kubernetes-service.html) |
| **ConfigMap** | Non-sensitive config | [Read →](/blog/2025/02/11/kubernetes-configmap-secret.html) |
| **Secret** | Sensitive data | [Read →](/blog/2025/02/11/kubernetes-configmap-secret.html) |
| **Ingress** | HTTP routing | [Read →](/blog/2025/02/11/kubernetes-ingress.html) |
| **StatefulSet** | Databases | [Read →](/blog/2025/02/11/kubernetes-statefulset-daemonset.html) |
| **DaemonSet** | Node agents | [Read →](/blog/2025/02/11/kubernetes-statefulset-daemonset.html) |
| **Job** | One-time tasks | [Read →](/blog/2025/02/11/kubernetes-job-cronjob.html) |
| **CronJob** | Scheduled tasks | [Read →](/blog/2025/02/11/kubernetes-job-cronjob.html) |
| **NetworkPolicy** | Traffic rules | [Read →](/blog/2025/02/11/kubernetes-network-policy.html) |
| **RBAC** | Authorization | [Read →](/blog/2025/02/11/kubernetes-rbac.html) |
| **HPA** | Auto-scaling | [Read →](/blog/2025/02/11/kubernetes-hpa.html) |
| **PV/PVC** | Storage | [Read →](/blog/2025/02/11/kubernetes-storage.html) |
| **Taint** | Scheduling | [Read →](/blog/2025/02/11/kubernetes-taints-tolerations.html) |

---

## 🔗 External Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Interactive Tutorials](https://kubernetes.io/docs/tutorials/)

---

## 💡 Tips for Learning

1. **Practice locally** - Use minikube, kind, or k3d
2. **Read the docs** - Official docs are comprehensive
3. **Join the community** - Kubernetes Slack, Discord
4. **Build something** - Deploy a real application
5. **Get certified** - CKA, CKAD are valuable

---

*Last updated: February 2025*
