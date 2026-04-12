# Multi-Tenancy

**Directive:** When designing shared clusters, namespace isolation, or tenant boundaries, ALWAYS apply defense-in-depth: ResourceQuota, LimitRange, NetworkPolicy, RBAC, and Pod Security Admission per namespace. A namespace without quotas and network policies is an open door. Default security posture is PSS "restricted" profile.

## When to use

Consult this reference whenever the task involves:
- Designing namespace structure for a shared cluster
- Isolating teams, environments, or tenants within a single cluster
- Configuring resource quotas, limit ranges, or RBAC per namespace
- Deciding between namespace isolation and separate clusters
- Implementing hierarchical namespace patterns

---

## Namespace as the Primary Isolation Boundary

Namespaces are the fundamental unit of multi-tenancy in Kubernetes. Every isolation mechanism -- RBAC, NetworkPolicy, ResourceQuota, Pod Security Admission -- is scoped to namespaces. A well-configured namespace provides:

- **Resource isolation** via ResourceQuota and LimitRange
- **Network isolation** via default-deny NetworkPolicy
- **Security isolation** via Pod Security Admission labels
- **Access isolation** via namespace-scoped RBAC

---

## ResourceQuota per Namespace

Every tenant namespace MUST have a ResourceQuota. Without it, one tenant can consume all cluster resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-alpha-quota
  namespace: tenant-alpha
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "50"
    configmaps: "50"
    services.loadbalancers: "2"
    services.nodeports: "0"         # disallow NodePort in shared clusters
```

When a ResourceQuota exists in a namespace, every Pod in that namespace MUST specify resource `requests` and `limits`, or admission is rejected. Use LimitRange to provide defaults.

---

## LimitRange per Namespace

LimitRange sets defaults and bounds so that individual pods cannot claim disproportionate resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-alpha-limits
  namespace: tenant-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
      min:
        cpu: 50m
        memory: 64Mi
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
      min:
        storage: 1Gi
```

---

## NetworkPolicy for Inter-Namespace Isolation

Apply a default-deny ingress and egress policy to every tenant namespace. Then selectively allow required traffic:

```yaml
# Default deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-alpha
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow DNS resolution (required for almost all workloads)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: tenant-alpha
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# Allow intra-namespace communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: tenant-alpha
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
```

See **network-exposure.md** for detailed NetworkPolicy patterns.

---

## RBAC Scoping

Use namespace-scoped `Role` and `RoleBinding` over `ClusterRole` and `ClusterRoleBinding`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-alpha-developer
  namespace: tenant-alpha
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["deployments", "services", "pods", "jobs", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]             # read-only for secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-alpha-developers
  namespace: tenant-alpha
subjects:
  - kind: Group
    name: team-alpha
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: tenant-alpha-developer
  apiGroup: rbac.authorization.k8s.io
```

See **privilege-sprawl.md** for detailed RBAC patterns and anti-patterns.

---

## Pod Security Admission per Namespace

Every tenant namespace MUST have PSA labels. See **insecure-workload-defaults.md** for the full security context requirements:

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/audit: restricted
  pod-security.kubernetes.io/warn: restricted
```

---

## Hierarchical Namespaces (HNC)

For organizations with team-of-teams structures, the Hierarchical Namespace Controller propagates policies from parent to child namespaces:

```yaml
# Parent namespace defines shared policies
apiVersion: hnc.x-k8s.io/v1alpha2
kind: HierarchyConfiguration
metadata:
  name: hierarchy
  namespace: platform-team
spec:
  children:
    - tenant-alpha
    - tenant-beta
```

NetworkPolicies, ResourceQuotas, and RBAC Roles in the parent namespace are inherited by children. This avoids duplicating boilerplate across dozens of tenant namespaces.

---

## Service Account Isolation

Each namespace should have dedicated service accounts. Never share service accounts across namespaces:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: tenant-alpha
automountServiceAccountToken: false   # opt-in, not opt-out
```

Workloads that need API access should use Bound Service Account Token Volumes with audience and expiry, not legacy static tokens.

---

## Naming Conventions

| Pattern | Example | Use when |
|---|---|---|
| `{env}-{service}` | `prod-payments`, `staging-orders` | Environment-based isolation |
| `{team}-{service}` | `platform-monitoring`, `alpha-api` | Team-based multi-tenancy |
| `{tenant}-{env}` | `acme-prod`, `acme-staging` | External multi-tenancy (SaaS) |

Consistency matters more than the specific pattern. Pick one and enforce it with admission webhooks.

---

## What Namespaces Do NOT Isolate

Namespaces are a soft boundary. They do NOT provide:

- **Node-level isolation:** Pods from different namespaces share the same node kernel, CPU, memory, and disk. A noisy neighbor or kernel exploit affects all tenants on that node. Use taints/tolerations or node pools for hard isolation.
- **Cluster-scoped resources:** ClusterRoles, ClusterRoleBindings, CustomResourceDefinitions, PersistentVolumes, and Nodes are visible cluster-wide.
- **Kernel and container runtime:** A container escape reaches the host regardless of namespace. Sandboxed runtimes (gVisor, Kata Containers) provide stronger boundaries.
- **Network without NetworkPolicy:** By default, all pods in all namespaces can communicate freely. NetworkPolicy is not applied until you create one.

---

## When to Use Separate Clusters vs Namespaces

| Criteria | Namespaces | Separate clusters |
|---|---|---|
| Blast radius tolerance | Acceptable shared risk | Zero tolerance for cross-tenant impact |
| Compliance requirements | Same compliance domain | Different regulatory requirements (PCI vs non-PCI) |
| Kubernetes version needs | Same version acceptable | Tenants need different versions |
| Cost sensitivity | Lower cost (shared control plane) | Higher cost, stronger isolation |
| Noisy neighbor risk | Acceptable with quotas | Unacceptable (latency-sensitive workloads) |

Rule of thumb: use namespaces for internal teams in the same trust domain. Use separate clusters when tenants are external customers or have different compliance requirements.

---

## GOOD: Complete Tenant Namespace Setup

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-alpha
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    tenant: alpha
    cost-center: eng-alpha
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: tenant-alpha
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.memory: 32Gi
    pods: "40"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: tenant-alpha
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: tenant-alpha
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

---

## LLM Mistake Checklist

Before finalizing any multi-tenant namespace configuration, verify each item:

- [ ] **ResourceQuota** is present in the namespace -- a namespace without quotas is unbounded.
- [ ] **LimitRange** provides default requests/limits so pods without explicit resources are not rejected by quota enforcement.
- [ ] **Default-deny NetworkPolicy** exists -- namespaces without NetworkPolicy allow all traffic by default.
- [ ] **DNS egress is allowed** in the NetworkPolicy -- forgetting this breaks all service discovery.
- [ ] **RBAC uses namespace-scoped Role**, not ClusterRole, unless cluster-wide access is explicitly needed.
- [ ] **PSA labels are set** on the namespace with all three modes (enforce, audit, warn).
- [ ] **Service accounts are per-namespace** with `automountServiceAccountToken: false` as default.
- [ ] **NodePort services are restricted** via ResourceQuota (`services.nodeports: "0"`) in shared clusters to prevent port conflicts.
