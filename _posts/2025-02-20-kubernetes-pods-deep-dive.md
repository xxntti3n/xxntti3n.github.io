---
layout: article
title: "Kubernetes Pods Deep Dive — Lifecycle, Probes, Init Containers, Sidecars"
date: 2025-02-20
author: Tien Nguyen
excerpt: "Pod lifecycle, liveness/readiness/startup probes, init containers, sidecar pattern, resource limits, and security context. With YAML and kubectl examples."
reading_time: 16
categories: [kubernetes]
permalink: /kubernetes/pods-deep-dive/
---

Pods are the smallest deployable unit in Kubernetes. This deep dive covers the pod lifecycle, health probes, init and sidecar containers, resources, and security context—with concrete YAML and commands.

---

## 1. Pod lifecycle (states and conditions)

A Pod moves through well-defined states:

```
  Created
     │
     ▼
  Pending ──► Running ──► Succeeded / Failed
     │            │
     │            └──► Unknown (node lost)
     │
     └──► (stuck: scheduling / image pull / init)
```

- **Pending** — Accepted by the API server; not yet scheduled or not yet running (e.g. pulling image, running init containers).
- **Running** — At least one container is still running.
- **Succeeded** — All containers exited with code 0.
- **Failed** — At least one container exited non-zero or was terminated by the system.
- **Unknown** — Kubelet could not report status (e.g. node lost).

Conditions (in `status.conditions`) include `PodScheduled`, `Initialized`, `ContainersReady`, `Ready`. Use them to see why a Pod isn’t ready.

```bash
kubectl get pod <pod> -o jsonpath='{.status.conditions}' | jq .
kubectl describe pod <pod>   # Events + conditions
```

---

## 2. Probes: liveness, readiness, startup

Probes tell Kubernetes whether to **restart** the container (liveness), **send traffic** to it (readiness), and how to handle **slow starters** (startup).

| Probe | Fails when… | Effect |
|-------|-------------|--------|
| **Liveness** | Probe fails | Container is restarted |
| **Readiness** | Probe fails | Pod removed from Service endpoints (no new traffic) |
| **Startup** | Probe fails | Container is restarted; while startup is running, liveness is disabled |

Example: HTTP liveness and readiness, and a startup probe for slow app init.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-probes
  labels:
    app: web
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 30
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 2
```

- **startupProbe** — Give the app up to 30×5s = 150s to become ready; until it succeeds, liveness is not run.
- **livenessProbe** — If `/health/live` fails 3 times in a row, the container is restarted.
- **readinessProbe** — If it fails, the Pod is taken out of Service until it passes again.

Use **exec**, **httpGet**, or **tcpSocket** depending on what your app supports. Avoid using the same endpoint for both liveness and readiness if one is heavier than the other.

---

## 3. Init containers

Init containers run to completion **in order** before the main containers start. Typical uses: wait for a dependency, migrate DB, or copy files into a shared volume.

```yaml
spec:
  initContainers:
  - name: wait-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  - name: migrate
    image: myapp:migrate
    command: ['rake', 'db:migrate']
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

If any init container fails, the Pod is retried (according to `restartPolicy`). Init containers always run to completion before the main containers start.

---

## 4. Sidecar pattern (multi-container Pod)

Containers in the same Pod share the same network namespace (same IP, localhost), and can share volumes. A common pattern is a **main app** plus a **sidecar** (e.g. log shipper, proxy, or metrics agent).

```
┌─────────────────────────────────────────────────────────┐
│  Pod (one IP, shared volumes)                            │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │  Main app       │  │  Log shipper     │              │
│  │  writes to      │  │  reads from      │              │
│  │  /var/log/app   │  │  /var/log/app    │              │
│  └────────┬────────┘  └────────┬────────┘              │
│           │                    │                         │
│           └────── volume ─────┘                         │
└─────────────────────────────────────────────────────────┘
```

Example: app plus a sidecar that tails logs and sends them to a log backend.

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  - name: log-sidecar
    image: log-shipper:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
```

The sidecar starts and stops with the Pod; use readiness on the main container so traffic only goes to the Pod when the app is ready, not the sidecar.

---

## 5. Resources: requests and limits

- **requests** — Used for **scheduling**. The scheduler only places the Pod on nodes that have at least that much free CPU/memory.
- **limits** — **Enforced on the node**. CPU is throttled when exceeded; memory over limit can get the container OOMKilled.

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
```

Always set **requests** (and limits where you want hard caps). Without requests, the Pod can be scheduled on overloaded nodes; without limits, one Pod can starve others.

---

## 6. Security context

You can set security options at **Pod** or **container** level. Container settings override Pod-level for that container.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

- **runAsNonRoot / runAsUser** — Don’t run as root; use a specific UID.
- **readOnlyRootFilesystem** — Container root filesystem is read-only (use emptyDir or volumes for writable paths).
- **capabilities.drop: ALL** — Drop all Linux capabilities (good default for app containers).

---

## 7. Useful commands

```bash
# Pod status and reasons
kubectl get pods -o wide
kubectl describe pod <pod> -n <ns>

# Logs (main container)
kubectl logs <pod> -n <ns>

# Logs (specific container in a multi-container Pod)
kubectl logs <pod> -c <container> -n <ns>

# Previous container log (after crash/restart)
kubectl logs <pod> --previous -n <ns>

# Execute in Pod
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl exec <pod> -c sidecar -n <ns> -- /bin/sh
```

---

## Summary

| Topic | Takeaway |
|-------|----------|
| Lifecycle | Pending → Running → Succeeded/Failed; use `describe` and `conditions` when stuck |
| Probes | Liveness = restart; readiness = remove from Service; startup = protect slow start |
| Init | Run before main containers; use for dependencies or setup |
| Sidecar | Same Pod = same network + shared volumes; good for logs, proxy, metrics |
| Resources | Set requests (scheduling) and limits (enforcement) |
| Security | Prefer runAsNonRoot, readOnlyRootFilesystem, drop capabilities |

For how Pods are created and updated by controllers, see [Deployments & Rolling Updates](/kubernetes/deployments-rolling-updates/). For how they get traffic, see [Services & Cluster Networking](/kubernetes/services-networking-deep-dive/).
