# Good Patterns -- Production-Ready Examples

These eight patterns demonstrate production-ready Kubernetes manifests that follow the PSS restricted profile, include proper labels, and set explicit resource constraints. Each pattern is annotated with key points explaining why specific choices were made.

For the full annotated YAML of every pattern below, see [references/examples-good.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/examples-good.md).

## 1. Minimal Production Deployment

A complete Deployment with full security context (pod-level and container-level), resource bounds, liveness and readiness probes, topology spread constraints, and standard `app.kubernetes.io/*` labels. Demonstrates the `readOnlyRootFilesystem` pattern with an emptyDir `/tmp` mount.

**Key takeaway:** Both pod-level and container-level `securityContext` are required. Topology spread prevents all replicas landing on one node.

## 2. Default-Deny NetworkPolicy

A two-resource pattern: a blanket deny-all policy (empty `podSelector`) followed by a targeted allow policy. Demonstrates allowing specific ingress from an ingress controller, scoped egress to a database, and mandatory DNS egress to kube-dns.

**Key takeaway:** Always allow DNS egress (UDP/TCP 53 to kube-dns) or name resolution breaks silently.

## 3. Scoped RBAC for CI Deployer

Namespace-scoped Role and RoleBinding for a CI pipeline ServiceAccount. Only grants the specific verbs and resources needed for deployment -- no `delete`, no `cluster-admin`, no ClusterRoleBinding.

## 4. CronJob with Lifecycle Controls

A CronJob with `concurrencyPolicy: Forbid`, `startingDeadlineSeconds`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`, history limits, and proper security context. Demonstrates safe scheduled job configuration that prevents overlapping runs and auto-cleans completed pods.

**Key takeaway:** `activeDeadlineSeconds` kills jobs that hang; `ttlSecondsAfterFinished` auto-cleans completed pods.

## 5. Ingress with TLS and Path-Based Routing

Uses the current `networking.k8s.io/v1` API with `ingressClassName` (not the deprecated annotation), TLS configuration, and path-based routing with explicit `pathType`. More specific paths listed first.

## 6. HPA with Scale-Down Stabilization

An HPA using `autoscaling/v2` with separate scale-up and scale-down behaviors. Scale-down is conservative (300s stabilization window, 25% per minute limit) while scale-up is aggressive. Targets both CPU and memory utilization.

## 7. Namespace with Quota, LimitRange, and PSA Labels

A complete namespace setup: PSA labels enforcing the restricted profile, a ResourceQuota capping total resource consumption, and a LimitRange providing defaults and bounds for containers that omit resource specs.

## 8. ExternalSecret for Vault Integration

Namespace-scoped SecretStore with Vault backend using Kubernetes auth, and an ExternalSecret that syncs credentials with a refresh interval. Demonstrates `deletionPolicy: Retain` to prevent accidental secret loss.

**Key takeaway:** Use namespace-scoped SecretStore (not ClusterSecretStore) unless multiple namespaces genuinely share the same Vault path.

---

Each of these patterns addresses one or more of KubeShark's six named failure modes. Use them as starting points and adapt to your cluster's specific requirements.
