---
layout: article
title: Kubernetes Explained: What Really Happens Under the Hood
date: 2025-02-11
---

<header class="article-header">
  <h1 class="article-title">Kubernetes Explained: What Really Happens Under the Hood</h1>
  <p class="article-subtitle">A complete guide to understanding Kubernetes — from the basics to production-ready patterns</p>
  <div class="article-meta">
    <div class="article-author">
      <div class="author-avatar">DN</div>
      <div class="author-info">
        <span class="author-name">DevOps Notes</span>
        <span class="article-date">February 11, 2025</span>
      </div>
    </div>
  </div>
</header>

<div class="article-content">

If you've spent any time deploying services at scale, you've probably heard of Kubernetes. Maybe you've even cursed at it once or twice — don't worry, you're in good company. Kubernetes has become the de facto standard for container orchestration, and for good reason: it's powerful, versatile, and (once you get the hang of it) surprisingly elegant. But mastering it? That's a whole other story.

Kubernetes is packed with features and an architecture that feels simple on the surface but gets complex real quick. If you've ever struggled with crashing pods, mysterious networking issues, or YAML that just won't behave, you know exactly what I mean.

This guide is your complete roadmap to understanding Kubernetes — from the basics to production-ready patterns.

---

## Part 1: The Architecture - Understanding the Control Plane

Before we deploy anything, we need to understand what's actually running under the hood. Kubernetes isn't a monolithic application — it's a distributed system with distinct components that work together.

### The Mental Model

Think of Kubernetes as an operating system for your datacenter. Just as your laptop's OS manages processes, networking, and storage on a single machine, Kubernetes manages these across an entire fleet of servers.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                              │
│                                                                       │
│  ┌────────────────── Control Plane ──────────────────┐                 │
│  │                                                  │                 │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  │   API       │  │   Scheduler  │  │   Controller │         │
│  │  │   Server    │◄─┤              │  │   Manager    │         │
│  │  └──────┬──────┘  └──────────────┘  └───────┬──────┘         │
│  │         │                                   │                   │
│  │         │              ┌──────────────┐        │                   │
│  │         └─────────────┤   etcd      │◄───────┘                   │
│  │                        (key-value store)                          │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                │                                     │
│                    ┌───────────┴───────────┐                         │
│                    │       Watch/Notify      │                         │
│                    ▼                       ▼                         │
│  ┌────────────────┐              ┌────────────────┐                    │
│  │    Node 1      │              │    Node 2      │     ...           │
│  │  ┌──────────┐ │              │  ┌──────────┐ │                    │
│  │  │ Kubelet  │◄├──────────────├──┤ Kubelet  │◄───┐              │
│  │  └────┬─────┘ │              │  └────┬─────┘ │    │              │
│  │       │       │              │       │       │    │              │
│  │  ┌────┴─────┐ │              │  ┌────┴─────┐ │    │              │
│  │  │kube-proxy│ │              │  │kube-proxy│ │    │              │
│  │  └──────────┘ │              │  └──────────┘ │    │              │
│  │  ┌──────────┐ │              │  ┌──────────┐ │    │              │
│  │  │Container │ │              │  │Container │ │    │              │
│  │  │Runtime   │ │              │  │Runtime   │ │    │              │
│  │  └──────────┘ │              │  └──────────┘ │    │              │
│  │  Pod  Pod  Pod│              │  Pod  Pod  Pod│    │              │
│  └────────────────┘              └────────────────┘    │              │
│                                                   │              │
│                                                   └── Network ────┘
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Control Plane Components

#### API Server - The Front Door

Every interaction with Kubernetes goes through the API Server. It's the only component that talks to etcd directly. When you run `kubectl get pods`, you're actually making an HTTP request to the API Server.

```bash
# What kubectl really does under the hood
kubectl get pods
# ↓
# GET /api/v1/namespaces/default/pods
# Host: kubernetes.default.svc
# Authorization: Bearer <service-account-token>
```

The API Server validates, authenticates, and then stores your request in etcd. It's stateless — you can run multiple instances behind a load balancer.

#### etcd - The Source of Truth

etcd is a distributed key-value store. Every Kubernetes object — every pod, service, config map — exists as a key-value pair in etcd.

```
/registry/pods/default/nginx-abc123 → {
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": { "name": "nginx-abc123", ... },
  "spec": { "containers": [...], ... }
}
```

> 💡 **Why etcd matters**: When a node fails and all its pods die, the Controller Manager watches etcd for "missing" pods and schedules replacements. The entire self-healing mechanism depends on etcd's consistency guarantees.

#### Scheduler - The Placement Engine

When you create a pod, it starts in `Pending` state. The Scheduler's job: find a node where it can run.

The scoring algorithm considers:

- **Resource availability**: Does the node have enough CPU/memory?
- **Affinity rules**: Does this pod prefer certain nodes?
- **Taints and tolerations**: Is this node allowed to run this pod?
- **Pod disruption budget**: Would scheduling here violate availability constraints?

```yaml
# Example: How scheduler thinks
apiVersion: v1
kind: Pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "database"
      effect: "NoSchedule"
```

#### Controller Manager - The Reconciler

Kubernetes works on a **reconciliation loop** model. The Controller Manager continuously watches the desired state (your YAML) and the actual state (etcd), and makes them match.

```
while true:
    desired_state = get_from_etcd()
    actual_state = query_cluster()
    if desired_state != actual_state:
        reconcile(desired_state, actual_state)
```

Common controllers:

- **ReplicaSet Controller**: Ensures N copies of each pod are running
- **Deployment Controller**: Rolls out new ReplicaSets for updates
- **Namespace Controller**: Cleans up resources when namespaces delete
- **Service Account Controller**: Creates default service accounts

---

## Part 2: The Pod - Kubernetes's Atomic Unit

A pod is NOT a container. A pod is a group of containers that share:

- Network namespace (same IP, same localhost)
- Storage volumes (shared filesystem)
- Lifecycle (start together, die together)

### Single Container Pod (99% of cases)

```
┌─────────────────────────────────────────────────────────┐
│                      Pod                             │
│  IP: 10.244.1.5                                  │
│                                                     │
│  ┌─────────────────────────────────────────────────┐  │
│  │                                                 │
│  │   ┌───────────────────────────────────────┐    │  │
│  │   │                                       │    │  │
│  │   │        Container (nginx)                │    │  │
│  │   │        - Process ID 1                 │    │  │
│  │   │        - Filesytem rootfs            │    │  │
│  │   │                                       │    │  │
│  │   └───────────────────────────────────────┘    │  │
│  │                                                 │
│  │   Volume mounts                                 │
│  └─────────────────────────────────────────────────┘  │
│                                                     │
│  Pause container (network namespace holder)              │
└─────────────────────────────────────────────────────────┘
```

The `pause` container is the secret sauce. It holds the network namespace so your app container can restart without losing its IP.

### Multi-Container Pod (Sidecar Pattern)

This is where Kubernetes gets interesting. Sidecars let you extend functionality without modifying the main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logging
spec:
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

    - name: log-shipper
      image: fluentd:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true

  volumes:
    - name: logs
      emptyDir: {}
```

**Why this matters**: Your app writes logs to `/var/log/app`, and fluentd ships them to Elasticsearch — all without touching your app code.

---

## Part 3: Deployment - Managing Pods at Scale

You rarely create pods directly. Instead, you use Deployments — which manage ReplicaSets — which manage Pods. It's controllers all the way down.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### The Rolling Update Dance

When you update the image from `nginx:1.25` to `nginx:1.26`:

```
Old ReplicaSet (nginx:1.25)        New ReplicaSet (nginx:1.26)
┌──────────────┐                   ┌──────────────┐
│ Pod-A        │ ───┐              │              │
│ Pod-B        │ ───┤── 25% ──────▶│ Pod-D        │
│ Pod-C        │ ───┘              │              │
└──────────────┘                   │              │
       │                          │ Pod-E        │ ─── 50%
       ▼                          │              │
  (terminating)                   │ Pod-F        │ ─── 75%
                                  │              │
                                  └──────────────┘
                                       │
                                       ▼
                                    (100% - old deleted)
```

The Deployment controller gradually shifts traffic from old pods to new pods. If health checks fail, it pauses — giving you time to rollback.

### Health Checks Are Critical

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

> 💡 **The difference**: `livenessProbe` restarts containers that hang. `readinessProbe` temporarily removes pods from service rotation if they're not ready to handle traffic.

---

## Part 4: Services - Stable Networking

Pods are ephemeral. Their IPs change when they restart. Services provide stable endpoints.

### ClusterIP - The Default

```
        Service (stable IP: 10.96.0.100)
                 │
        ┌────────┴────────┐
        │                 │
    Pod-A (10.244.1.5)  Pod-B (10.244.2.8)
    IP changes...       IP changes...
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

### NodePort - For Development

```
External Traffic (Node IP:30234)
         │
    ┌────┴────┐
    │ Node 1  │ ──▶ Pod (if exists)
    │ Node 2  │ ──▶ Pod (if exists)
    │ Node 3  │ ──▶ Pod (if exists)
    └─────────┘
```

### LoadBalancer - For Production

Provisions a cloud load balancer (GCP, AWS, Azure) that forwards to all nodes.

---

## Part 5: ConfigMap and Secret - Configuration

Never hardcode configuration. Use ConfigMaps for non-sensitive data, Secrets for sensitive data.

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db:5432/mydb"
  cache_ttl: "300"
```

### Using it as Environment Variables

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config
```

### Secret - Encrypted at Rest

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 encoded
```

> ⚠️ **Security note**: Base64 is encoding, not encryption. Kubernetes encrypts Secrets at rest in etcd, but you should also enable RBAC and consider external secret management (Vault, AWS Secrets Manager).

---

## Part 6: Ingress - HTTP Routing

Services operate at Layer 4 (TCP/UDP). Ingress operates at Layer 7 (HTTP).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Flow**: User → Cloud Load Balancer → Ingress Controller → Service → Pod

Popular ingress controllers: NGINX, Traefik, HAProxy, AWS ALB Ingress.

---

## Part 7: StatefulSet - For Databases

Deployments give pods random names: `nginx-7d9c8b5f-xabc`. StatefulSets give them ordered names: `mysql-0`, `mysql-1`, `mysql-2`.

This matters for databases because:

- Each pod gets stable storage (PVC stays with the pod)
- Pods are created/deleted in order
- Each pod has a stable network identity

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  template:
    # ... pod template ...
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

> 💡 **Production tip**: Don't run databases in Kubernetes unless you have to. Use managed services (RDS, Cloud SQL) when possible.

---

## Part 8: Storage - PV, PVC, StorageClass

Kubernetes abstracts storage with three layers:

```
StorageClass (provisioner: aws-ebs)
       │
       ▼
PersistentVolumeClaim (request: 10Gi)
       │
       ▼
PersistentVolume (actual disk)
       │
       ▼
   Pod (mounts PVC)
```

### StorageClass - Dynamic Provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
```

### PVC - Requesting Storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

---

## Part 9: HPA - Auto-scaling

Horizontal Pod Autoscaler scales pods based on metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

> 💡 **How it works**: The metrics-server scrapes resource usage from Kubelet. HPA queries the metrics API and adjusts the replicas field on your Deployment.

---

## Part 10: RBAC - Access Control

RBAC is critically important for production clusters.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-read
  namespace: production
subjects:
- kind: User
  name: jane@company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

> ⚠️ **Security tip**: Use `ClusterRole` sparingly. Prefer namespaced `Role` whenever possible. Regularly audit `ClusterRoleBindings`.

---

## Part 11: Network Policy - Traffic Control

By default, Kubernetes allows all traffic. Network policies restrict it.

```yaml
apiVersion: networking.k8s.io/v1
kind:NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

This denies all traffic. Then you allow what you need:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 5432
```

> 💡 **Requires**: A CNI that supports network policies (Calico, Cilium, Weave).

---

## Part 12: Taints and Tolerations - Scheduling Control

Taints repel pods. Tolerations allow pods to be scheduled on tainted nodes.

```yaml
# Taint a node for GPU workloads only
kubectl taint nodes node-1 gpu=true:NoSchedule
```

```yaml
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: cuda-app
    image: cuda-app:latest
```

**Use cases**:

- Dedicated nodes for databases (taint: `dedicated=db`)
- GPU nodes (tolerate: `gpu`)
- Nodes with special hardware
- Temporary exclusion during maintenance

---

## Putting It All Together - A Complete Example

Here's a complete web application stack:

```yaml
---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: "production"
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: myapp:1.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: web-config
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: web-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Best Practices Summary

1. **Always set resource requests/limits** — prevents noisy neighbor problems
2. **Use liveness and readiness probes** — enables self-healing
3. **Prevent privileged containers** — security best practice
4. **Use NetworkPolicies** — zero-trust networking
5. **Limit RBAC permissions** — principle of least privilege
6. **Use labels consistently** — enables organized querying
7. **Set up HPA** — automatic scaling
8. **Use Probes for health checks** — enables self-healing
9. **Secure your etcd** — the source of truth
10. **Monitor everything** — Prometheus + Grafana

---

## Learning Path

I recommend learning these concepts in order:

1. **Pod** — The atomic unit
2. **Deployment** — Managing pods
3. **Service** — Stable networking
4. **ConfigMap/Secret** — Configuration
5. **Ingress** — HTTP routing
6. **StatefulSet** — Stateful apps
7. **Storage** — PV, PVC, StorageClass
8. **HPA** — Auto-scaling
9. **RBAC** — Access control
10. **NetworkPolicy** — Traffic control
11. **Taints/Tolerations** — Scheduling

---

## Final Thoughts

Kubernetes is complex, but it's complexity that pays off at scale. The key is understanding the mental model — controllers reconciling desired state with actual state.

Start simple. Deploy a pod. Then a deployment. Then a service. Add ingress. Then scale. Each concept builds on the previous ones.

Don't try to learn everything at once. Kubernetes is a marathon, not a sprint.

---

*This guide covers the fundamentals. For hands-on practice, I recommend minikube or kind for local development.*

</div>
