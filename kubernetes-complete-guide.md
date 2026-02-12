---
layout: default
title: Kubernetes Explained: What Really Happens Under the Hood
---

# Kubernetes Explained: What Really Happens Under the Hood

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
│  │   Volume mounts                                 │  │
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
      # This container reads what the first one writes
```

Common sidecar patterns:
- **Log shipping**: Fluentd/Filebeat tail logs while app writes them
- **Proxies**: Envoy/Istio sidecar handles TLS, mTLS, observability
- **Adapters**: Transform logs/metrics before sending to monitoring
- **Ambassador**: Connect to external services (databases, APIs)

### The Pod Lifecycle - What Actually Happens

```
kubectl run nginx --image=nginx
│
├─► API Server receives request, stores in etcd
│   Status: Pending
│
├─► Scheduler sees pending pod, evaluates nodes
│   Scores nodes, picks best match
│   Updates pod spec with nodeName
│
├─► Kubelet on target node watches for pods assigned to it
│   Sees new pod, calls CRI (container runtime)
│   Pulls image (if not cached)
│   Creates pause container
│   Creates app container(s)
│
├─► Kube-proxy updates iptables/IPVS rules
│   Adds Service endpoints for this pod
│
└─► Pod is Running
    Status: Running
    Ready: true (after readiness probe passes)
```

> 💡 **Why readiness probes matter**: A pod's IP is added to Service endpoints only AFTER the readiness probe succeeds. Without this, traffic hits pods that aren't ready to serve requests.

---

## Part 3: Deployments - Managing State at Scale

A Deployment is a declarative way to manage ReplicaSets, which manage Pods. It's three levels of indirection that enable rolling updates and rollbacks.

```
Deployment  ──manages──►  ReplicaSet  ──manages──►  Pods
    │                             │                       │
    │  Creates new ReplicaSet    │  Creates N pods       │
    │  for updates               │                       │
    │                             │                       │
    ▼                             ▼                       ▼
"3 replicas of              "Pods with label         actual containers
nginx:1.19"               app=nginx, v=1.19"      with processes
```

### Rolling Update - What Really Happens

When you change a Deployment's image:

```bash
kubectl set image deployment/nginx nginx=nginx:1.20
```

Here's the sequence:

```
Time 0:  ReplicaSet-A (nginx:1.19) has 3 pods
        ┌─────────┬─────────┬─────────┐
        │ Pod-A1  │ Pod-A2  │ Pod-A3  │  (v1.19)
        └─────────┴─────────┴─────────┘

Time 1:  Create ReplicaSet-B (nginx:1.20), scale to 1
        ┌─────────┬─────────┬─────────┐
        │ Pod-A1  │ Pod-A2  │ Pod-A3  │  (v1.19)
        └─────────┴─────────┴─────────┘
        ┌─────────┐
        │ Pod-B1  │  (v1.20) ← new pod
        └─────────┘

Time 2:  Pod-B1 is ready, scale ReplicaSet-B to 2, scale A to 2
        ┌─────────┬─────────┬─────────┐
        │ Pod-A1  │ Pod-A2  │         │  (v1.19)
        └─────────┴─────────┴─────────┘
        ┌─────────┬─────────┐
        │ Pod-B1  │ Pod-B2  │  (v1.20)
        └─────────┴─────────┘

Time 3:  Continue until all 3 are v1.20
        ┌─────────┬─────────┬─────────┐
        │         │         │         │
        └─────────┴─────────┴─────────┘
        ┌─────────┬─────────┬─────────┐
        │ Pod-B1  │ Pod-B2  │ Pod-B3  │  (v1.20)
        └─────────┴─────────┴─────────┘

Result:  ReplicaSet-A scaled to 0 (kept for rollback)
         ReplicaSet-B has 3 pods
```

The default behavior:
- `maxSurge: 25%` - Can create 25% more pods than desired
- `maxUnavailable: 25%` - Can have 25% fewer pods than desired

### The Deployment Status Dance

```bash
kubectl rollout status deployment/nginx
```

This command blocks until:
1. All new replicas are created
2. All new replicas are Ready
3. All old replicas are terminated

If anything fails, the rollout pauses:

```
Progress deadline exceeded
→ The deployment timed out waiting for new replicas to become ready
→ Old ReplicaSet is still running (safety mechanism)
→ You can: rollback, adjust, or continue
```

### Rollback - Instant Revert

```bash
kubectl rollout undo deployment/nginx
```

This doesn't "undo" anything — it simply scales up the old ReplicaSet and scales down the new one. That's why Kubernetes keeps old ReplicaSets around by default (default: 10 revisions).

---

## Part 4: Services - Making Pods Findable

Pods are ephemeral. Their IPs change when they restart. Services provide stable endpoints for a dynamic set of pods.

### How Services Work Under the Hood

```
Service creates stable IP/DNS
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                  Service (ClusterIP)                  │
│  IP: 10.96.0.10 (stable)                         │
│  Selector: app=nginx                                │
└─────────────────────────────────────────────────────────┘
         │
         │ kube-proxy programs:
         │ - iptables rules OR
         │ - IPVS load balancing
         ▼
┌─────────────────────────────────────────────────────────┐
│                   Endpoints                          │
│  10.244.1.5:80   (pod A)                        │
│  10.244.1.6:80   (pod B)                        │
│  10.244.2.7:80   (pod C)                        │
└─────────────────────────────────────────────────────────┘
```

Every Service gets:
- **ClusterIP**: Internal virtual IP (only reachable within cluster)
- **DNS name**: `nginx.default.svc.cluster.local`
- **Endpoints list**: Updated automatically as pods come/go

### Service Types - When to Use What

| Type | Use Case | External Access | Notes |
|------|-----------|-----------------|-------|
| **ClusterIP** | Internal microservices | No (cluster only) | Default, most common |
| **NodePort** | Testing, local dev | Yes (NodeIP:Port) | Port 30000-32767 |
| **LoadBalancer** | Production, cloud LB | Yes (cloud LB IP) | Requires cloud controller |
| **Headless** | StatefulSet, DNS directly | No | Returns pod IPs, not proxy |

### Headless Services - When You Need Direct Access

For StatefulSets or when you need to know which pod you're talking to:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  clusterIP: None  # ← Makes it headless
  selector:
    app: mongodb
  ports:
    - port: 27017
```

Result: DNS returns A records for each pod directly:
```
mongo-0.mongo.default.svc.cluster.local → 10.244.1.5
mongo-1.mongo.default.svc.cluster.local → 10.244.1.6
mongo-2.mongo.default.svc.cluster.local → 10.244.2.7
```

---

## Part 5: ConfigMap and Secret - Configuration Management

Never hardcode configuration in container images. Images should be immutable; configuration should be external.

### ConfigMap - Non-Sensitive Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_host: "postgres.default.svc.cluster.local"
  cache_ttl: "300"
  features.json: |
    {
      "new_ui": true,
      "beta_features": false
    }
```

**Ways to use ConfigMaps:**

1. **Environment variables**:
```yaml
envFrom:
  - configMapRef:
      name: app-config
```

2. **Single value as env var**:
```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: database_host
```

3. **Mounted as files**:
```yaml
volumeMounts:
  - name: config
    mountPath: /etc/config
volumes:
  - name: config
    configMap:
      name: app-config
# Results in /etc/config/database_host, /etc/config/cache_ttl, /etc/config/features.json
```

### Secret - Sensitive Data

Secrets are similar to ConfigMaps but:
- Base64 encoded (not encrypted by default!)
- Separate RBAC permissions
- Can use external secret stores (Vault, AWS Secrets Manager)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64("password123")
  api-key: YWJjZGVmZ2hpams=
```

> ⚠️ **Critical**: Base64 is encoding, not encryption. Anyone with API access can decode secrets. Use:
> - External secrets operator (for Vault, cloud secret managers)
> - Encryption at rest (etcd encryption config)
> - RBAC to limit secret access

### The Immutable Pattern (Best Practice)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-v2
immutable: true  # Can't be updated after creation
data:
  # ...
```

When you update config:
1. Create new ConfigMap (app-v2)
2. Update Deployment to reference app-v2
3. Rollout restarts pods with new config
4. Old ConfigMap (app-v1) can be deleted after rollout

This prevents accidentally changing config and causing undefined behavior.

---

## Part 6: Ingress - HTTP/HTTPS Routing

Services handle L4 (TCP) load balancing. Ingress handles L7 (HTTP) routing — path-based routing, TLS termination, host-based routing.

### Why Ingress?

Without Ingress:
```
External LoadBalancer (expensive!)
    ├── Service A (NodePort 30001)
    ├── Service B (NodePort 30002)
    └── Service C (NodePort 30003)
```

With Ingress:
```
External LoadBalancer (one!)
    └── Ingress Controller (Nginx/Traefik/Contour)
        ├── /api/v1   → Service A
        ├── /api/v2   → Service B
        └── /static   → Service C
```

### Ingress Resource Definition

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - example.com
      secretName: tls-cert-secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

### How Ingress Controllers Work

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Ingress Controller Pod                        │
│  (Nginx/Traefik/Contour running inside your cluster)           │
│                                                                │
│  1. Watches Ingress resources via API Server                    │
│  2. Generates config dynamically (nginx.conf)                     │
│  3. Reloads config gracefully                                  │
│  4. Terminates TLS                                            │
│  5. Proxies to Services via ClusterDNS                         │
└─────────────────────────────────────────────────────────────────────┘
                          │
                          │ HTTP/HTTPS (Layer 7)
                          ▼
                   ┌─────────────┐
                   │   Service   │
                   └─────────────┘
```

Popular Ingress Controllers:
- **Nginx Ingress**: Battle-tested, feature-rich
- **Traefik**: Auto-config, great dashboard
- **Contour**: Envoy-based, HTTP/2, gRPC support
- **AWS ALB Ingress**: Uses AWS ALB directly

---

## Part 7: StatefulSet - For Databases and Stateful Apps

Deployments create pods with random names. StatefulSets create numbered, ordered pods with stable identities.

### When to Use StatefulSet?

- Databases (PostgreSQL, MySQL, MongoDB)
- Distributed systems (ZooKeeper, etcd)
- Applications needing stable storage
- Ordered deployment and scaling

### StatefulSet vs Deployment

```
Deployment:
    nginx-deployment-7b8c9x5-k4abc  (random)
    nginx-deployment-7b8c9x5-p9def  (random)
    nginx-deployment-7b8c9x5-m2ghi  (random)
    → No ordering, no stable identity

StatefulSet:
    postgres-0  (always 0, starts first)
    postgres-1  (always 1, starts after 0)
    postgres-2  (always 2, starts after 1)
    → Ordered deployment, stable network identity, stable storage
```

### Example: PostgreSQL Cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless for StatefulSets
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # Must match headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_REPLICA_MODE
              value: "slave"
            - name: POSTGRES_MASTER
              value: "postgres-0.postgres"  # Uses stable DNS!
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # Each pod gets unique PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

Result:
- `postgres-0.postgres.default.svc.cluster.local` → master
- `postgres-1.postgres.default.svc.cluster.local` → replica 1
- `postgres-2.postgres.default.svc.cluster.local` → replica 2
- Each has its own persistent volume

---

## Part 8: DaemonSet - One Pod Per Node

DaemonSets ensure a pod runs on every (or selected) node. Use cases:
- Logging agents (Fluentd, Filebeat)
- Monitoring agents (Prometheus node-exporter)
- Network plugins (Calico, Cilium)
- Storage agents (Ceph, GlusterFS)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        # Run on master nodes too
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluentd:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

---

## Part 9: Job and CronJob - Batch Processing

Not everything runs continuously. Jobs run to completion, then exit.

### Job - One-Time or Parallel Tasks

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-import
spec:
  completions: 10  # Need 10 successful completions
  parallelism: 3    # Run 3 in parallel
  backoffLimit: 6    # Retry failed pods up to 6 times
  template:
    spec:
      restartPolicy: Never  # Jobs use Never or OnFailure
      containers:
        - name: importer
          image: data-importer:latest
          args: ["--batch-size=1000"]
```

Use cases:
- Database migrations
- Batch data processing
- Backup jobs
- One-time initialization

### CronJob - Scheduled Recurring Tasks

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: cleanup-script:latest
          restartPolicy: OnFailure
```

> ⚠️ **CronJob gotcha**: Kubernetes doesn't guarantee exact execution time. If your cluster is busy, the job might start late. The `startingDeadlineSeconds` parameter controls how late is too late.

---

## Part 10: Network Policy - Traffic Control

By default, Kubernetes allows all pod-to-pod traffic. NetworkPolicy lets you define rules like "frontend can talk to backend, but not to database."

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}  # Matches all pods
  policyTypes:
    - Ingress
    - Egress
```

### Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53  # DNS
```

Requires a CNI that supports NetworkPolicy:
- Calico (most feature-complete)
- Cilium (eBPF-based, very fast)
- Weave Net
- Romana

---

## Part 11: RBAC - Access Control

Kubernetes has a powerful RBAC system. Every request is: `User → Group → ServiceAccount` trying to do `Action` on `Resource`.

### The Model

```
Subject (who)
    │
    ├── User: "jane@example.com"     (external auth)
    ├── Group: "devops"              (external auth)
    └── ServiceAccount: "bot"         (Kubernetes native)
         │
         └── gets permissions via RoleBinding/ClusterRoleBinding
              │
              ▼
         Role (namespace-scoped) or ClusterRole (cluster-wide)
              │
              defines: apiGroups, resources, verbs
              │
              ▼
         Resources: pods, deployments, configmaps, secrets, etc.
         Verbs: get, list, watch, create, update, delete, etc.
```

### Example: ReadOnly User

```yaml
# Create a service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: readonly-user
  namespace: default
---
# Create a role with read-only permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: readonly-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
---
# Bind the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: readonly-user
    namespace: default
roleRef:
  kind: Role
  name: readonly-role
  apiGroup: rbac.authorization.k8s.io
```

### Common RBAC Patterns

| Pattern | Role | Use Case |
|---------|-------|----------|
| **Read-only** | get, list, watch | Dashboards, monitoring |
| **Developer** | * on resources except secrets/secrets | Dev team |
| **CI/CD** | create, update, delete on deployments | Pipeline bots |
| **Admin** | * on * | Platform engineers |

> 💡 **Best practice**: Never give admin permissions to applications. Use least privilege. Regularly audit with `kubectl auth can-i --list --all-namespaces`.

---

## Part 12: Horizontal Pod Autoscaler - Auto-scaling

HPA automatically scales deployments based on CPU, memory, or custom metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageAverageValue: 80
```

### How HPA Works

```
1. Metrics Server collects metrics from pods (every 15s)
2. HPA controller calculates desired replicas:
   desiredReplicas = currentReplicas × (currentMetric / targetMetric)
3. If desired != current, scales Deployment
4. Takes ~30 seconds to detect change + 30 seconds to scale
```

> ⚠️ **HPA gotcha**: If you manually scale a Deployment managed by HPA, HPA will revert it. To manually scale, update the HPA's min/max or temporarily delete the HPA.

---

## Part 13: Storage - PV, PVC, StorageClass

Kubernetes separates storage provisioning (StorageClass/PV) from storage consumption (PVC).

### The Model

```
StorageClass (defines storage types)
    ├── "fast-ssd" → provisions SSD disks
    ├── "standard" → provisions HDD disks
    └── "nfs" → provisions NFS shares
         │
         ▼ creates
PersistentVolume (actual storage)
    ├── PV-1: 10Gi SSD, RWO
    ├── PV-2: 50Gi HDD, RWX
    └── PV-3: 5Gi SSD, RWO
         │
         ▼ claimed by
PersistentVolumeClaim (pod's request for storage)
    ├── pvc-1: needs 10Gi, RWO → binds to PV-1
    └── pvc-2: needs 5Gi, RWO → binds to PV-3
         │
         ▼ mounted by
Pod
    volumeMount: pvc-1 → /data
```

### StorageClass - Dynamic Provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # Or other cloud provider
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
reclaimPolicy: Delete  # or Retain
volumeBindingMode: WaitForFirstConsumer
```

### PVC - Claiming Storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce  # RWO, RWX, or ReadOnlyMany
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

### Access Modes

| Mode | Description | Use Case |
|-------|-------------|----------|
| **ReadWriteOnce (RWO)** | Single node read-write | Most databases, stateful apps |
| **ReadOnlyMany (ROX)** | Multiple nodes read-only | Shared configuration, CDNs |
| **ReadWriteMany (RWX)** | Multiple nodes read-write | NFS-based shared storage |

---

## Part 14: Taints and Tolerations - Pod Scheduling Control

Taints repel pods. Tolerations allow pods to be scheduled on tainted nodes.

### Use Cases

- **Dedicated nodes**: Database-only nodes
- **Hardware constraints**: GPU nodes for ML workloads
- **Special networking**: Nodes with direct internet access
- **Node maintenance**: Mark nodes for draining

### Example: Dedicated Database Nodes

```yaml
# Taint the node
kubectl taint nodes node-1 database=dedicated:NoSchedule
```

```yaml
# Pod that CAN run on database nodes
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  tolerations:
    - key: database
      operator: Equal
      value: dedicated
      effect: NoSchedule
  # ... rest of pod spec
```

```yaml
# Deployment that MUST run on database nodes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  template:
    spec:
      tolerations:
        - key: database
          operator: Equal
          value: dedicated
          effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: database
                    operator: In
                    values: ["dedicated"]
```

### Taint Effects

| Effect | Behavior |
|--------|----------|
| **NoSchedule** | Pod won't be scheduled unless it tolerates |
| **PreferNoSchedule** | Scheduler tries to avoid, but allows |
| **NoExecute** | Evicts existing pods that don't tolerate |

---

## Putting It All Together: A Microservice Architecture

Here's how these pieces fit in a real application:

```
                        Ingress Controller
                     (Nginx, Traefik, etc.)
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
              Frontend Svc           Backend Svc
            (Deployment)           (Deployment)
            3 replicas             HPA: 2-10 replicas
                    │                     │
                    │                     │
                    ▼                     ▼
            ┌──────────┐         ┌──────────┐
            │  Pods:   │         │  Pods:   │
            │  web-xxx  │         │ api-xxx   │
            │  web-yyy  │         │ api-yyy   │
            │  web-zzz  │         │ api-zzz   │
            └──────────┘         └──────────┘
                    │                     │
                    │ ConfigMap/Secret     │ ConfigMap/Secret
                    │                     │
                    ▼                     ▼
            ┌──────────┐         ┌──────────┐
            │          │         │          │
            └──────────┘         └──────────┘
                    │                     │
                    └────────┬────────────┘
                             ▼
                    ┌─────────────────┐
                    │  Database Svc   │
                    │  (StatefulSet) │
                    │  postgres-0    │
                    │  postgres-1    │
                    │  postgres-2    │
                    └─────────────────┘
                             │
                        PVC per pod
```

---

## Quick Reference Commands

```bash
# Get all resources in namespace
kubectl get all

# Watch pod status
kubectl get pods -w

# Get pod logs (follow)
kubectl logs -f pod/name

# Get pod logs with previous container (if crashed)
kubectl logs pod/name --previous

# Execute command in pod
kubectl exec -it pod/name -- /bin/sh

# Port forward to local machine
kubectl port-forward pod/name 8080:80

# Apply manifest
kubectl apply -f manifest.yaml

# Edit live resource (not recommended for prod)
kubectl edit deployment/name

# Describe resource for debugging
kubectl describe pod/name

# Check rollout status
kubectl rollout status deployment/name

# Rollback deployment
kubectl rollout undo deployment/name

# Scale deployment
kubectl scale deployment/name --replicas=5

# Get events for namespace
kubectl get events --sort-by='.lastTimestamp'

# Check API resources
kubectl api-resources

# Explain a resource
kubectl explain pod.spec
```

---

## Common Pitfalls and Solutions

| Problem | Cause | Solution |
|----------|--------|----------|
| Pod keeps restarting | App crashes, OOMKilled | Check logs, increase memory limits, fix app |
| ImagePullBackOff | Wrong image name/credentials | Check image name, create imagePullSecret |
| CrashLoopBackOff | App fails healthcheck | Fix readiness/liveness probe or app |
| Pending (Insufficient cpu) | Cluster doesn't have resources | Add nodes or reduce resource requests |
| Service can't reach pod | Wrong selector, wrong port | Check Service selector matches pod labels |
| DNS not working | Missing DNS policy, wrong service name | Use FQDN: `service.namespace.svc.cluster.local` |
| ConfigMap not found | Typo in name, wrong namespace | Check namespace, verify ConfigMap exists |
| Storage provisioning fails | No StorageClass, wrong provisioner | Verify StorageClass exists, check cloud setup |

---

## Learning Checklist

Use this to track your progress:

- [ ] **Architecture**: Understand control plane vs. worker nodes
- [ ] **Pods**: Run a simple nginx pod
- [ ] **Deployments**: Deploy an application with 3 replicas
- [ ] **Services**: Expose deployment via ClusterIP
- [ ] **ConfigMap/Secret**: Externalize configuration
- [ ] **Ingress**: Set up HTTP routing with TLS
- [ ] **StatefulSet**: Deploy a StatefulSet with persistent storage
- [ ] **Job/CronJob**: Run a scheduled batch job
- [ ] **NetworkPolicy**: Restrict pod-to-pod traffic
- [ ] **RBAC**: Create a read-only service account
- [ ] **HPA**: Enable autoscaling based on CPU
- [ ] **Storage**: Use PVCs with StatefulSets
- [ ] **Taints/Tolerations**: Dedicate nodes for specific workloads

---

## External Resources

- [Official Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Interactive Tutorials](https://kubernetes.io/docs/tutorials/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)

---

*Last updated: February 2026*
