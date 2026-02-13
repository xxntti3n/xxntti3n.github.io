---
layout: article
title: "Kubernetes RBAC & Service Accounts Deep Dive"
date: 2025-02-22
author: Tien Nguyen
excerpt: "ServiceAccounts, Roles, ClusterRoles, bindings, and least-privilege patterns. With YAML examples and how to audit and debug access."
reading_time: 16
categories: [kubernetes]
permalink: /kubernetes/rbac-security-deep-dive/
---

RBAC (Role-Based Access Control) in Kubernetes controls who can do what on which resources. This deep dive covers ServiceAccounts, Roles and ClusterRoles, bindings, and practical least-privilege patterns with YAML and commands.

---

## 1. RBAC building blocks

- **Who** — User, group, or **ServiceAccount** (for Pods and in-cluster automation).
- **What** — **Role** or **ClusterRole** (list of API resources + verbs: get, list, create, update, delete, patch, watch).
- **Binding** — **RoleBinding** (grants a Role in one namespace) or **ClusterRoleBinding** (grants a ClusterRole cluster-wide).

```
  ServiceAccount "app-sa" (in namespace prod)
        │
        │  RoleBinding "app-sa-reader"
        ▼
  Role "pod-reader" (in namespace prod)
        │
        └──► rules: pods get, list, watch
```

---

## 2. ServiceAccount

Every Pod has a ServiceAccount (default: `default` in the same namespace). The token is mounted so processes in the Pod can call the API server.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: prod
```

Use a **dedicated** ServiceAccount per app or team so you can scope RBAC to that identity.

```yaml
# In Deployment/Pod
spec:
  serviceAccountName: app-sa
  # optional: automountServiceAccountToken: false
```

---

## 3. Role (namespaced)

A **Role** applies to a single namespace: it lists resources and verbs allowed in that namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: prod
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get"]
```

- **apiGroups: [""]** — Core group (pods, services, configmaps, secrets, etc.).
- **resources** — Plural resource name.
- **resourceNames** — Optional; restrict to specific resource names.
- **verbs** — get, list, create, update, patch, delete, watch, deletecollection.

---

## 4. RoleBinding

A **RoleBinding** grants a Role (or ClusterRole) to subjects (user, group, or ServiceAccount) **in one namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-reader
  namespace: prod
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: prod
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Now Pods using `serviceAccountName: app-sa` in `prod` can get/list/watch Pods and get the ConfigMap `app-config` in `prod` only.

---

## 5. ClusterRole and ClusterRoleBinding

**ClusterRole** — Can define permissions for:

- Namespaced resources (in any or all namespaces), or
- Cluster-scoped resources (nodes, PVs, namespaces, etc.).

**ClusterRoleBinding** — Grants a ClusterRole to subjects **cluster-wide**.

Example: read-only access to Pods in all namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-global
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-pods-global
subjects:
- kind: ServiceAccount
  name: monitoring
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: pod-reader-global
  apiGroup: rbac.authorization.k8s.io
```

**Reusing a ClusterRole in one namespace** — Create a **RoleBinding** (not ClusterRoleBinding) and set `roleRef` to the ClusterRole. The ClusterRole is then applied only in the binding’s namespace. Built-in roles like `view`, `edit`, `admin` are ClusterRoles; you bind them per namespace with a RoleBinding.

---

## 6. Least-privilege examples

**CI/CD deployer (one namespace):**

```yaml
# ClusterRole: deploy in a namespace (deployments, pods, services, etc.)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# RoleBinding in namespace "prod"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer
  namespace: prod
subjects:
- kind: ServiceAccount
  name: cicd
  namespace: cicd
roleRef:
  kind: ClusterRole
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**App that only reads its own ConfigMap/Secret:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-config-reader
  namespace: prod
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"]
  verbs: ["get"]
```

---

## 7. Checking and debugging

```bash
# What can a ServiceAccount do in a namespace?
kubectl auth can-i --list --as=system:serviceaccount:prod:app-sa -n prod

# Single check
kubectl auth can-i create pods --as=system:serviceaccount:prod:app-sa -n prod

# Why can't a Pod do something? Check SA, RoleBinding, Role/ClusterRole
kubectl get rolebinding,clusterrolebinding -A -o wide | grep app-sa
kubectl describe rolebinding <name> -n prod
kubectl get role <role-name> -n prod -o yaml
```

---

## 8. Audit and security tips

- Enable **audit logging** on the API server to log who did what (create/delete/patch).
- Prefer **dedicated ServiceAccounts** per app; avoid using `default` for sensitive workloads.
- Prefer **Role + RoleBinding** (namespace-scoped) over ClusterRoleBinding when possible.
- Use **resourceNames** to restrict access to specific ConfigMaps/Secrets.
- Review **ClusterRoleBindings** to `cluster-admin` and other powerful ClusterRoles; limit to a few identities.

---

## Summary

| Concept | Use |
|--------|-----|
| ServiceAccount | Identity for Pods and in-cluster clients |
| Role | Permissions in one namespace |
| ClusterRole | Reusable permission set (namespaced or cluster resources) |
| RoleBinding | Grant Role/ClusterRole in one namespace |
| ClusterRoleBinding | Grant ClusterRole cluster-wide |
| Least privilege | Narrow resources and verbs; use resourceNames where possible |

For how the API server enforces RBAC, see [Control Plane Deep Dive](/kubernetes/control-plane-deep-dive/). For storing credentials used by Pods, see the main [Kubernetes guide](/kubernetes/kubernetes-guide/) (ConfigMap & Secret section).
