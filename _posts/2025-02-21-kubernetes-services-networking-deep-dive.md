---
layout: article
title: "Kubernetes Services & Cluster Networking Deep Dive"
date: 2025-02-21
author: Tien Nguyen
excerpt: "ClusterIP, NodePort, LoadBalancer, kube-proxy, CoreDNS, and service discovery. With YAML, debugging, and production patterns."
reading_time: 17
categories: [kubernetes]
permalink: /kubernetes/services-networking-deep-dive/
---

Services give Pods a stable name and IP and load-balance across backend Pods. This deep dive covers Service types, kube-proxy modes, CoreDNS, and how to debug and tune cluster networking.

---

## 1. Why Services?

Pods are ephemeral: IPs change when Pods are recreated. A **Service** is a stable endpoint (cluster-internal IP or node port) that forwards traffic to a set of Pods selected by labels.

```
   Client (Pod or external)
        │
        ▼
   Service (ClusterIP 10.96.x.x or NodePort 3xxxx)
        │
        ├──► Pod 1 (backend)
        ├──► Pod 2 (backend)
        └──► Pod 3 (backend)
```

---

## 2. Service types

| Type | Use case | How to reach |
|------|----------|--------------|
| **ClusterIP** | In-cluster traffic only | `<service>.<ns>.svc.cluster.local` or cluster IP |
| **NodePort** | Expose on every node’s IP at a fixed port (30000–32767) | `<NodeIP>:<NodePort>` |
| **LoadBalancer** | Cloud LB in front of Service (e.g. AWS NLB, GCP Network LB) | LB’s external IP/hostname |

### ClusterIP (default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: prod
spec:
  selector:
    app: api
    tier: backend
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP
```

- **port** — Port the Service listens on (e.g. 80).
- **targetPort** — Port on the Pod (e.g. 8080). Omit to use the same as `port`.
- **selector** — Only Pods with these labels are in the endpoints.

DNS: `api.prod.svc.cluster.local` (or `api` from the same namespace).

### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
  type: NodePort
```

Traffic to **any node’s IP:30080** is forwarded to a backend Pod. NodePort is in range 30000–32767 unless you set it explicitly (and the cluster allows it).

### LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

On AWS/GCP/Azure, the cloud controller creates a load balancer and sets `status.loadBalancer.ingress[0].ip` (or hostname). Traffic goes LB → NodePort/ClusterIP → Pods.

---

## 3. Endpoints and EndpointSlices

The control plane creates **Endpoints** (and optionally **EndpointSlices**) that list the Pod IPs and ports matching the Service selector. kube-proxy (and the cloud LB) use this list to forward traffic.

```bash
kubectl get endpoints <service> -n <ns>
kubectl get endpointslices -n <ns>
```

If **Endpoints** is empty, no Pods match the selector—check labels and Pod readiness.

---

## 4. kube-proxy modes

**kube-proxy** runs on every node and programs **rules** so that traffic to the Service IP or NodePort is forwarded to a backend Pod.

- **iptables** (default in many clusters) — Rules in iptables NAT table; random backend choice.
- **ipvs** — More scalable; supports multiple scheduling algorithms (rr, lc, dh, etc.).

```bash
# See which mode (often in kube-proxy Pod args or ConfigMap)
kubectl get pod -n kube-system -l k8s-app=kube-proxy -o yaml | grep -i mode
```

---

## 5. CoreDNS and service discovery

CoreDNS resolves names like `api.prod.svc.cluster.local` to the Service’s ClusterIP. Short names work within the same namespace: `api` → `api.prod.svc.cluster.local`.

```bash
# Test DNS from a Pod
kubectl run -it --rm debug --image=busybox:1.36 --restart=Never -- nslookup api.prod
```

**Pod DNS:** `<pod-ip>.<namespace>.pod.cluster.local` (for headless Services or StatefulSet Pods).

---

## 6. Headless Service

For a **headless** Service, you set `clusterIP: None`. No virtual IP is allocated; DNS returns the **Pod IPs** of the backing Pods (or EndpointSlices). Used for StatefulSets or when you need to talk to specific Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 5432
```

DNS for `db.namespace.svc.cluster.local` can return multiple A records (one per Pod).

---

## 7. Debugging

| Symptom | Check |
|---------|--------|
| No connectivity to Service | `kubectl get endpoints` — are backends listed? Readiness probes passing? |
| DNS not resolving | CoreDNS Pods running? `nslookup` from a test Pod. |
| NodePort not reachable | Security groups / firewall; `kubectl get svc` for nodePort. |
| LoadBalancer stuck Pending | Cloud provider integration; events on the Service. |

```bash
kubectl get svc -n <ns>
kubectl describe svc <name> -n <ns>
kubectl get endpoints <name> -n <ns>
```

---

## 8. Summary

- **ClusterIP** — Stable in-cluster name and IP; use for app-to-app traffic.
- **NodePort** — Fixed port on every node; useful for dev or behind your own LB.
- **LoadBalancer** — Cloud LB; standard for exposing HTTP/HTTPS or TCP in production.
- **Endpoints** — Built from selector; must be non-empty for traffic to reach Pods.
- **CoreDNS** — Resolves `svc.ns.svc.cluster.local` and (for headless) Pod IPs.

For exposing HTTP/HTTPS with TLS and path routing, see [Ingress](/kubernetes/kubernetes-guide/#ingress) in the main guide. For how Pods are selected by Services, see [Pods Deep Dive](/kubernetes/pods-deep-dive/).
