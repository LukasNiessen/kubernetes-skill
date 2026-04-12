# Do / Don't Quick Reference

> Terse checklist of Kubernetes best practices organized by category. Each line is
> a standalone rule. Default security posture is PSS "restricted" profile.

---

## Security Contexts

- DO set `runAsNonRoot: true` and explicit `runAsUser`/`runAsGroup` on every pod.
- DO set `allowPrivilegeEscalation: false` on every container.
- DO set `readOnlyRootFilesystem: true` and mount writable paths as emptyDir.
- DO set `capabilities.drop: ["ALL"]` and only add back specific caps if required.
- DO set `seccompProfile.type: RuntimeDefault` at the pod level.
- DON'T set `privileged: true` unless the workload genuinely requires it (CNI plugins, node agents).
- DON'T omit the security context and rely on cluster defaults.

## RBAC

- DO use namespace-scoped Role + RoleBinding for workloads that operate in one namespace.
- DO grant only the specific verbs, API groups, and resources needed.
- DO use `resourceNames` to scope access to specific objects when possible.
- DON'T bind to `cluster-admin` for application workloads.
- DON'T use ClusterRoleBinding when RoleBinding is sufficient.
- DON'T grant `*` (wildcard) verbs or resources.
- DON'T leave `automountServiceAccountToken: true` on pods that do not call the Kubernetes API.

## Resource Management

- DO set `requests` for both CPU and memory on every container.
- DO set `limits.memory` to prevent OOM from killing other workloads.
- DO set ResourceQuota and LimitRange on every namespace.
- DO leave CPU limits unset or generous to avoid CPU throttling.
- DON'T omit resource requests -- the scheduler cannot bin-pack without them.
- DON'T set requests equal to limits unless you need Guaranteed QoS class intentionally.

## Networking

- DO create a default-deny NetworkPolicy in every namespace.
- DO allow DNS egress (UDP/TCP 53 to kube-dns) in every allow-list policy.
- DO use `ingressClassName` instead of the deprecated `kubernetes.io/ingress.class` annotation.
- DO use `networking.k8s.io/v1` for Ingress and NetworkPolicy resources.
- DON'T expose Services as `type: LoadBalancer` without understanding cost and security implications.
- DON'T use `type: NodePort` in production without firewall rules.

## Probes and Rollouts

- DO set a readiness probe on every container that serves traffic.
- DO set a liveness probe that checks only the process's own health.
- DO set `initialDelaySeconds` to account for application startup time.
- DO set `revisionHistoryLimit` to a small number (3-5) to reduce etcd storage.
- DO use `maxUnavailable: 0` with `maxSurge: 1` for zero-downtime rolling updates.
- DON'T point liveness probes at external dependencies.
- DON'T set liveness and readiness probes to the same endpoint and thresholds without understanding the difference.
- DON'T set `failureThreshold: 1` on liveness probes -- one slow response kills the pod.

## Image Management

- DO use immutable image tags (`v1.2.3`) or digests (`@sha256:...`).
- DO set `imagePullPolicy: IfNotPresent` with immutable tags.
- DO reference images from a private registry with `imagePullSecrets`.
- DON'T use `:latest` -- it is mutable, breaks rollback, and causes inconsistent replicas.
- DON'T omit the image tag entirely -- it implicitly defaults to `:latest`.

## Storage

- DO verify the storage class supports the requested access mode before creating a PVC.
- DO use `ReadWriteOnce` for block storage (EBS, Persistent Disk).
- DO use StatefulSet with `volumeClaimTemplates` for per-replica storage.
- DON'T request `ReadWriteMany` with block storage classes (gp3, pd-ssd).
- DON'T use `hostPath` volumes in production workloads.

## Configuration

- DO store credentials in Secrets, not ConfigMaps.
- DO use ExternalSecrets or Sealed Secrets so plain-text credentials never enter version control.
- DO use ConfigMap/Secret hash-based naming (Kustomize generator, Helm sha annotation) to trigger rolling updates on config change.
- DON'T embed passwords in connection string environment variables inside ConfigMaps.
- DON'T commit raw Secret manifests to Git.

## Namespaces and Isolation

- DO apply PSA labels (`pod-security.kubernetes.io/enforce: restricted`) to every namespace.
- DO create ResourceQuota in every namespace to prevent noisy-neighbor resource exhaustion.
- DO use separate namespaces for separate trust boundaries.
- DON'T deploy application workloads in `default`, `kube-system`, or `kube-public`.
- DON'T assume namespace isolation provides network isolation -- it does not without NetworkPolicies.
