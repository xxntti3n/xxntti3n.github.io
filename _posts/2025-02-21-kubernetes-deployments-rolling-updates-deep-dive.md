---
layout: article
title: "Kubernetes Deployments & Rolling Updates Deep Dive"
date: 2025-02-21
author: Tien Nguyen
excerpt: "Deployment strategy, maxSurge and maxUnavailable, rollback, canary and blue-green patterns. With YAML and kubectl examples."
reading_time: 15
categories: [kubernetes]
permalink: /kubernetes/deployments-rolling-updates-deep-dive/
---

Deployments manage ReplicaSets and Pods and drive rolling updates and rollbacks. This deep dive covers strategy tuning, rollout commands, and practical canary/blue-green patterns.

---

## 1. Deployment = desired state + rollout

A **Deployment** declares:

- **Replicas** — How many Pods.
- **Pod template** — Image, env, resources, probes (the “what” to run).
- **Strategy** — How to replace old Pods with new ones (rolling update or recreate).

The controller creates/updates **ReplicaSets** so that the current RS matches the desired replicas and template; the ReplicaSet then creates/deletes **Pods**.

```
  Deployment (replicas: 3, image: app:v2)
       │
       ▼
  ReplicaSet (app:v2) ──► Pod, Pod, Pod
       │
       │  (previous ReplicaSet kept for rollback)
       ▼
  ReplicaSet (app:v1) ──► (scaled to 0 after rollout)
```

---

## 2. Rolling update strategy

**strategy.type: RollingUpdate** (default) replaces Pods gradually. Two knobs:

- **maxSurge** — How many *extra* Pods above desired count can exist during the rollout (e.g. 1 or 25%).
- **maxUnavailable** — How many Pods can be *unavailable* during the rollout (e.g. 0 or 25%).

Example: 10 replicas, maxSurge=2, maxUnavailable=0 → at most 12 Pods (10 old + 2 new), and you never go below 10 available.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: prod
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myreg/api:v2
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 5
```

- **maxUnavailable: 0** — Good when you must not drop capacity (e.g. low replica count or strict SLA).
- **maxSurge: 1** — Slower, fewer extra resources; **maxSurge: 25%** — Faster rollout, more temporary Pods.

---

## 3. Recreate strategy

**strategy.type: Recreate** — All old Pods are deleted first, then new ones are created. Downtime between old and new. Use only when you need a full stop (e.g. schema migration) or when the app cannot run two versions at once.

```yaml
spec:
  strategy:
    type: Recreate
```

---

## 4. Rollout and rollback commands

```bash
# Trigger rollout (e.g. after changing image or env)
kubectl set image deployment/api api=myreg/api:v3 -n prod
# or
kubectl apply -f deployment.yaml

# Status
kubectl rollout status deployment/api -n prod

# Pause / resume (e.g. canary: deploy one, test, then resume)
kubectl rollout pause deployment/api -n prod
kubectl rollout resume deployment/api -n prod

# History (ReplicaSets)
kubectl rollout history deployment/api -n prod

# Rollback to previous revision
kubectl rollout undo deployment/api -n prod

# Rollback to specific revision
kubectl rollout undo deployment/api --to-revision=2 -n prod
```

Rollback is implemented by re-applying a previous ReplicaSet template; the Deployment controller does another rolling update “back” to that template.

---

## 5. Canary pattern (manual)

- **Stable Deployment** — e.g. 9 replicas, `app: api`, `version: stable`.
- **Canary Deployment** — e.g. 1 replica, `app: api`, `version: canary`.
- **Service** — Selector `app: api` (no version). Traffic is split 9:1 by replica count.
- Promote canary: scale stable to 0, canary to 10, or make canary the new template and delete canary Deployment.

You can also use two Services (e.g. `api-stable` and `api-canary`) and split traffic at Ingress or mesh level.

---

## 6. Blue-green (manual)

- **Blue Deployment** — e.g. `api-blue`, 10 replicas, image v1.
- **Green Deployment** — e.g. `api-green`, 10 replicas, image v2.
- **Service** — Selector points to one of them (e.g. `version: blue`). Switch selector to `version: green` to cut over.

No gradual rollout; instant switch. Rollback = switch selector back to blue.

---

## 7. Practical tips

- **Always set resource requests/limits** on the Deployment template so scheduling and capacity are predictable.
- **Use readiness probes** so new Pods get traffic only when ready; otherwise rolling update can hit starting Pods.
- **Use rollout status** in CI/CD and consider failing the pipeline if the rollout doesn’t complete in time.
- **Revision history** is limited (default 10); adjust with `spec.revisionHistoryLimit`.

---

## 8. Summary

| Topic | Takeaway |
|-------|----------|
| Strategy | RollingUpdate (maxSurge / maxUnavailable) or Recreate |
| Rollback | `kubectl rollout undo` or `--to-revision=N` |
| Canary | Separate Deployment with fewer replicas, same Service selector |
| Blue-green | Two Deployments; switch Service selector to cut over |

For how Pods and probes behave during a rollout, see [Pods Deep Dive](/kubernetes/pods-deep-dive/). For the control plane that drives Deployments, see [Control Plane Deep Dive](/kubernetes/control-plane-deep-dive/).
