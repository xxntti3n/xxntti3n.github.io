---
layout: post
title: "Kubernetes Network Policy: Traffic Control"
date: 2025-02-11
tags: [kubernetes, network-policy, security, networking]
author: DevOps Engineer
---

## Network Policy

**Network Policy** controls traffic flow between pods at the IP address level. It's like a firewall within Kubernetes.

---

## Why Network Policy?

<div class="diagram-container">
```mermaid
graph TB
    subgraph "Without Network Policy"
        Any[Pod A<br/>Can talk to<br/>ANYONE]
        Any -.->.all. All[Pod B, Pod C, Pod D<br/>Internet, Database<br/>Everything reachable]
    end

    subgraph "With Network Policy"
        A[Pod A<br/>Frontend]
        B[Pod B<br/>Backend]
        DB[(Database)]
        Ext[External API]

        A -->|"Allowed"| B
        B -->|"Allowed"| DB
        A -.->.blocked. X1[Database - Blocked]
        B -.->.blocked. X2[External API - Blocked]
    end

    style All fill:#e74c3c,stroke:#c0392b,color:#fff
    style X1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style X2 fill:#e74c3c,stroke:#c0392b,color:#fff
```
</div>

<div class="concept-box">
<strong>Default behavior:</strong> All pods can talk to all pods (and external).<br><br>
<strong>With NetworkPolicy:</strong> Default deny, explicit allow.
</div>

---

## Policy Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
spec:
  podSelector:              # Which pods this applies to
    matchLabels:
      app: backend
  policyTypes:
  - Ingress                 # Incoming traffic rules
  - Egress                  # Outgoing traffic rules
```

---

## Ingress Policy (Incoming Traffic)

### Allow from specific pods

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
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend           # Only frontend can connect
    ports:
    - protocol: TCP
      port: 8080
```

### Allow from specific namespace

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: production        # Only pods in production namespace
  - podSelector:
      matchLabels:
        app: frontend
```

### Allow from specific IP block

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8         # Only from 10.0.0.0/8
  ports:
  - protocol: TCP
    port: 443
```

---

## Egress Policy (Outgoing Traffic)

### Allow DNS only

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-only
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          k8s-app: kube-dns     # CoreDNS
    ports:
    - protocol: UDP
      port: 53
```

### Allow specific external API

```yaml
egress:
- to:
  - ipBlock:
      cidr: 52.1.2.3/32        # Specific external IP
  ports:
  - protocol: TCP
    port: 443
```

### Deny specific external IP

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0         # All IPs
      except:
      - 192.168.1.100/32     # Except this one
  ports:
  - protocol: TCP
    port: 80
```

---

## Full Example: Frontend-Backend-DB

<div class="diagram-container">
```mermaid
graph TB
    FE[Frontend Pods<br/>app=frontend]
    BE[Backend Pods<br/>app=backend]
    DB[(Database Pods<br/>app=database)]

    FE -->|"Allowed"| BE
    BE -->|"Allowed"| DB

    FE -.->.blocked. X1[Database - Blocked]
    DB -.->.blocked. X2[Internet - Blocked]

    style X1 fill:#e74c3c,stroke:#c0392b,color:#fff
    style X2 fill:#e74c3c,stroke:#c0392b,color:#fff
```
</div>

```yaml
---
# Backend policy
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
          app: frontend           # Only frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:                           # Allow to database
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:                           # Allow DNS
    - namespaceSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# Database policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
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
          app: backend           # Only backend
    ports:
    - protocol: TCP
      port: 5432
  # No egress = no external access
```

---

## Policy Types

| Policy Type | Description | Default |
|-------------|-------------|---------|
| **Ingress** | Controls incoming traffic | All allowed |
| **Egress** | Controls outgoing traffic | All allowed |

```yaml
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress     # Only control incoming
  # Egress not specified = all egress allowed
```

---

## Combining Rules

```yaml
ingress:
# Rule 1: Allow from frontend
- from:
  - podSelector:
      matchLabels:
        app: frontend
  ports:
  - port: 8080

# Rule 2: Allow from monitoring
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring
  ports:
  - port: 9090

# Rules are OR'd: match ANY rule = allowed
```

---

## Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}              # Matches all pods
  policyTypes:
  - Ingress                    # No ingress rules = deny all

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}              # Matches all pods
  policyTypes:
  - Egress                     # No egress rules = deny all
```

---

## Port Ranges

```yaml
ingress:
- ports:
  - protocol: TCP
    port: 8080                 # Single port
  - protocol: TCP
    port: 9090                 # Another single port
  - protocol: TCP
    endPort: 10000             # Port range
```

---

## Commands

```bash
# List network policies
kubectl get networkpolicy

# Get policy details
kubectl describe networkpolicy backend-policy

# Get policy YAML
kubectl get networkpolicy backend-policy -o yaml

# Test connectivity between pods
kubectl run test-pod --image=nicolaka/netshoot -it --rm --restart=Never -- \
  wget -O- http://backend-service:8080
```

---

## CNI Requirements

<div class="concept-box">
<strong>Important:</strong> Network Policy requires a CNI plugin that supports it.<br><br>
✅ Calico, Cilium, Weave Net, Romana, Kube-router<br>
❌ Flannel (default) - needs additional setup
</div>

---

## Best Practices

1. **Start with deny-all** - then explicitly allow
2. **Use namespace labels** - for multi-tenant isolation
3. **Document policies** - use clear names and comments
4. **Test thoroughly** - verify connectivity after applying
5. **Monitor hits** - use logs to see policy in action
6. **Use labels effectively** - for pod selection
7. **Separate ingress/egress** - for clarity

---

## Troubleshooting

```bash
# Check if CNI supports network policy
kubectl get pods -n kube-system -l k8s-app=calico-node

# Verify policy is applied
kubectl get networkpolicy -o yaml

# Test from a pod
kubectl exec -it test-pod -- wget -O- http://service:8080

# Check pod labels (policy uses these)
kubectl get pods --show-labels
```

---

## Summary

```
┌─────────────────────────────────────────────┐
│  No NetworkPolicy  =  Allow all traffic     │
│  NetworkPolicy      =  Deny all, allow some │
└─────────────────────────────────────────────┘
```

| Direction | Controls |
|-----------|----------|
| **Ingress** | Who CAN talk to these pods |
| **Egress** | Where these pods CAN talk to |

---

## Next: [RBAC](#) - Access control
