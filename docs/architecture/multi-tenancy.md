# Multi-Tenancy

Running multiple teams, environments, or customers in a single Kubernetes cluster requires defense-in-depth isolation. Namespaces are the primary boundary, but a namespace without quotas, network policies, RBAC scoping, and Pod Security Admission is an open door. This guide covers the five layers of namespace isolation and when to use separate clusters instead.

## Namespace as the Isolation Unit

Every Kubernetes isolation mechanism is scoped to namespaces: RBAC, NetworkPolicy, ResourceQuota, LimitRange, and Pod Security Admission. A well-configured tenant namespace enforces all five simultaneously. An unconfigured namespace provides none of them.

## Layer 1: ResourceQuota

Every tenant namespace must have a ResourceQuota. Without it, a single tenant can consume all cluster CPU, memory, and storage, starving other tenants.

ResourceQuota sets aggregate caps on the namespace: total CPU requests, memory limits, pod count, PVC count, and service types. When a ResourceQuota exists, every pod in the namespace must specify resource `requests` and `limits` or admission is rejected. This enforces resource discipline across all workloads.

Key settings for shared clusters:
- `services.nodeports: "0"` prevents tenants from claiming node ports that conflict across namespaces.
- `services.loadbalancers` limits the number of cloud load balancers a tenant can provision.
- `persistentvolumeclaims` caps storage consumption.

## Layer 2: LimitRange

LimitRange complements ResourceQuota by setting per-container defaults and bounds. Without LimitRange, a pod that omits resource specifications is rejected by the quota (since the quota requires explicit resources). LimitRange provides sensible defaults so that "lazy" deployments still get resource boundaries.

LimitRange also sets min/max bounds per container, preventing a single container from requesting disproportionate resources (e.g., 32Gi memory in a namespace with a 40Gi quota).

## Layer 3: NetworkPolicy

By default, pods in different namespaces can communicate freely. Default-deny NetworkPolicy is the minimum viable network isolation for multi-tenancy.

A complete namespace network baseline consists of three policies:
1. **Default deny all** -- blocks all ingress and egress for every pod in the namespace.
2. **Allow DNS** -- permits egress to kube-system on port 53 so service discovery works.
3. **Allow intra-namespace** -- permits pods within the same namespace to communicate.

Additional policies are added as needed for cross-namespace communication (e.g., allowing the ingress controller namespace to reach application pods).

The AND/OR semantics of NetworkPolicy rules are critical for multi-tenancy: a `namespaceSelector` and `podSelector` in the same `from` entry are AND-ed (both must match). Separate `from` entries are OR-ed. Getting this wrong can either block legitimate traffic or open traffic to the entire cluster.

## Layer 4: RBAC Scoping

Use namespace-scoped `Role` and `RoleBinding` for tenant access. `ClusterRole` and `ClusterRoleBinding` grant access across all namespaces and should be reserved for platform administrators.

Tenant RBAC should follow least privilege:
- Developers: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete` on workload resources (Deployments, Services, ConfigMaps, Jobs). Read-only on Secrets.
- CI/CD pipelines: `create`, `update`, `patch` on Deployments and ConfigMaps. No access to Secrets (use external secret management).
- Monitoring: `get`, `list`, `watch` on pods, events, and metrics endpoints.

Never use wildcards (`verbs: ["*"]`, `resources: ["*"]`) in tenant roles. See the [Privilege Sprawl](../failure-modes/privilege-sprawl.md) deep dive for details.

## Layer 5: Pod Security Admission

Every tenant namespace must have PSA labels enforcing at minimum the `baseline` profile, and preferably `restricted`:

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/audit: restricted
  pod-security.kubernetes.io/warn: restricted
```

Set all three modes (enforce, audit, warn). `enforce` blocks non-compliant pods. `audit` logs violations. `warn` shows warnings to users during `kubectl apply`. Using all three provides defense-in-depth and visibility into violations that audit mode catches but enforce mode has not yet been enabled for.

## Naming Conventions

Consistent namespace naming enables policy automation and cost attribution:
- **Environment-based:** `prod-payments`, `staging-orders`.
- **Team-based:** `platform-monitoring`, `alpha-api`.
- **Tenant-based (SaaS):** `acme-prod`, `acme-staging`.

Pick one pattern and enforce it with admission webhooks. Inconsistent naming makes RBAC, cost allocation, and policy application error-prone.

## What Namespaces Do Not Isolate

Namespaces are a soft boundary. They do not provide:
- **Node-level isolation.** Pods from different namespaces share the same node kernel. A container escape or noisy neighbor affects all tenants on that node. Use taints/tolerations and dedicated node pools for hard isolation.
- **Cluster-scoped resources.** ClusterRoles, CRDs, PersistentVolumes, and Nodes are visible cluster-wide.
- **Network without NetworkPolicy.** Namespaces without NetworkPolicy allow all traffic by default.
- **Container runtime isolation.** A kernel exploit reaches the host regardless of namespace. Use sandboxed runtimes (gVisor, Kata Containers) for untrusted workloads.

## When to Use Separate Clusters

| Factor | Namespaces | Separate clusters |
|---|---|---|
| Blast radius tolerance | Shared risk acceptable | Zero cross-tenant impact required |
| Compliance | Same regulatory domain | Different requirements (PCI vs non-PCI) |
| Kubernetes version | Same version for all tenants | Tenants need different versions |
| Cost | Lower (shared control plane) | Higher but stronger isolation |
| Noisy neighbor risk | Acceptable with quotas | Unacceptable (latency-sensitive) |

Rule of thumb: use namespaces for internal teams in the same trust domain. Use separate clusters when tenants are external customers, have different compliance requirements, or when the blast radius of a cluster-level failure is unacceptable.

## Further Reading

- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [KubeShark Privilege Sprawl](../failure-modes/privilege-sprawl.md)
- [KubeShark Network Exposure](../failure-modes/network-exposure.md)
