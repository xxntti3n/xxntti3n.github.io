# Prompt 1: Combine All Kubernetes Posts into Single Guide

**Role:** Senior Frontend Engineer / Content Architect

**Context:**
The site currently has 13 separate Kubernetes blog posts. We want to combine them into a single comprehensive Substack-style guide (like https://luminousmen.substack.com/p/spark-caching-explained-what-really).

---

## Task: Create Combined Kubernetes Guide

### Input Files to Read and Combine:
```
/Users/tien.nguyen6/Desktop/Cake/nttien/xxntti3n.github.io/_posts/
├── 2025-02-11-kubernetes-overview.md
├── 2025-02-11-kubernetes-pod.md
├── 2025-02-11-kubernetes-deployment.md
├── 2025-02-11-kubernetes-service.md
├── 2025-02-11-kubernetes-configmap-secret.md
├── 2025-02-11-kubernetes-ingress.md
├── 2025-02-11-kubernetes-statefulset-daemonset.md
├── 2025-02-11-kubernetes-job-cronjob.md
├── 2025-02-11-kubernetes-network-policy.md
├── 2025-02-11-kubernetes-rbac.md
├── 2025-02-11-kubernetes-hpa.md
├── 2025-02-11-kubernetes-storage.md
└── 2025-02-11-kubernetes-taints-tolerations.md
```

### Output File to Create:
```
/Users/tien.nguyen6/Desktop/Cake/nttien/xxntti3n.github.io/_posts/kubernetes-complete-guide.md
```

---

## Content Structure to Create:

### Front Matter
```yaml
---
layout: guide
title: "Kubernetes Explained: What Really Happens Under The Hood"
date: 2025-02-12
tags: [kubernetes, tutorial, guide, complete]
author: Tien Nguyen
excerpt: "A complete guide to understanding Kubernetes — from basics to production-ready patterns. Learn about control plane, pods, services, networking, security, and more in this comprehensive 15,000-word guide."
reading_time: 75
---
```

### Article Structure (in order):

```markdown
# [HERO CONTENT - Introduction]

## Why This Guide?

[Brief intro paragraph explaining this is a complete K8s guide]

---

## Table of Contents

1. [Kubernetes Overview](#overview) - Architecture and Components
2. [Pod](#pod) - The Smallest Deployable Unit
3. [Deployment](#deployment) - Managing Pods at Scale
4. [Service](#service) - Networking and Service Discovery
5. [ConfigMap & Secret](#config) - Configuration Management
6. [Ingress](#ingress) - HTTP/HTTPS Routing
7. [StatefulSet & DaemonSet](#statefulset) - Specialized Workloads
8. [Job & CronJob](#job) - Batch and Scheduled Tasks
9. [Network Policy](#network-policy) - Traffic Control
10. [RBAC](#rbac) - Access Control and Security
11. [HPA](#hpa) - Horizontal Pod Autoscaling
12. [Storage](#storage) - Persistent Volumes and Claims
13. [Taints & Tolerations](#taints) - Pod Scheduling

---

<a id="overview"></a>
# 1. Kubernetes Overview

[PASTE CONTENT from kubernetes-overview.md]

---

<a id="pod"></a>
# 2. Pod: The Smallest Unit

[PASTE CONTENT from kubernetes-pod.md]
**REMOVE the "Next: [Deployment](#)" link at the end**

---

<a id="deployment"></a>
# 3. Deployment: Managing Pods at Scale

[PASTE CONTENT from kubernetes-deployment.md]
**REMOVE the "Next" link at the end**

[... continue pattern for ALL remaining sections ...]

---

## Final Section: What's Next?

Add a conclusion section with:

### You've Learned:
- [ ] Kubernetes architecture and control plane
- [ ] How to deploy and manage pods
- [ ] Service networking and ingress
- [ ] Configuration with ConfigMaps and Secrets
- [ ] Security with RBAC and Network Policies
- [ ] Storage, autoscaling, and advanced scheduling

### Continue Learning:
- Set up a local cluster: minikube, kind, or k3d
- Deploy a real application to production
- Explore Service Mesh (Istio, Linkerd)
- Learn Operators for complex applications

---

## Subscribe Section

[Include subscribe box CTA]

```

---

## Content Transformation Rules:

### DO:
- ✅ Keep ALL content from each post
- ✅ Preserve all Mermaid diagrams
- ✅ Preserve all ASCII art diagrams
- ✅ Keep all YAML code examples
- ✅ Keep all bash command examples
- ✅ Keep all comparison tables
- ✅ Preserve concept boxes and callouts

### DON'T:
- ❌ Remove any content (combine everything)
- ❌ Summarize sections (use full content)
- ❌ Remove code examples
- ❌ Remove diagrams
- ❌ Keep "Next: [Topic](#)" links

### CLEANUP:
- Remove: "## Next: [Deployment](#)" type endings
- Remove: Duplicate front matter from each post
- Fix: Any broken section references
- Add: Section anchors for TOC navigation

---

## Quality Checklist:

After creating the file, verify:

- [ ] File size is approximately 15,000+ words
- [ ] All 13 topics are covered
- [ ] Table of contents has 13 entries
- [ ] Each section has proper anchor ID
- [ ] All Mermaid diagrams are enclosed in ```mermaid blocks
- [ ] All YAML examples are in ```yaml blocks
- [ ] All bash commands are in ```bash blocks
- [ ] No "Next: ..." links remain
- [ ] Front matter is properly formatted
- [ ] Author name is "Tien Nguyen" (not "DevOps Engineer")

---

## Next Steps After This Task:

1. Run: `bundle exec jekyll serve`
2. Navigate to: http://localhost:4000/kubernetes-complete-guide.html
3. Verify all content renders correctly
4. Check all diagrams display properly
5. Test on mobile viewport

---

**Expected Output:** A single ~500+ line markdown file that combines all Kubernetes learning into one comprehensive guide.
