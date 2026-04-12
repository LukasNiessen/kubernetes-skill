# Bad Patterns -- Common LLM Anti-Patterns

These eight anti-patterns represent manifests that LLMs frequently generate. Each one compiles and appears valid but has serious issues in production. The danger of these patterns is that Kubernetes accepts them without error -- the failure only surfaces at runtime or under load.

For full annotated YAML with detailed explanations of what is wrong in each case, see [references/examples-bad.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/examples-bad.md).

## 1. Deployment Running as Root with No Security Context

No `securityContext` at pod or container level -- container runs as root by default. Missing `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, capabilities drop, seccomp profile. Also lacks resource requests, probes, and standard labels. Uses `:latest` tag.

**Failure modes:** Insecure workload defaults, resource starvation, fragile rollouts.

## 2. Service with Selector That Matches No Pods

Service selector includes `version: v1` but pods have `version: v2`. Kubernetes does not warn about selector mismatches -- the Service silently has zero endpoints. A frequent LLM mistake when updating version labels on the Deployment without updating the Service.

## 3. ClusterRoleBinding with cluster-admin for a Single-Namespace App

Binds a single-namespace application ServiceAccount to `cluster-admin`, granting unrestricted access to the entire cluster. If the service account token is compromised, the attacker owns every namespace, every resource, every verb.

**Failure mode:** Privilege sprawl.

## 4. Liveness Probe Checking External Database

Liveness probe depends on `pg_isready` against an external database. If the database is briefly unavailable, Kubernetes kills all API pods, causing cascading failure: database blip leads to thundering herd reconnects and further overload.

## 5. Deployment with :latest Tag and No imagePullPolicy

Uses the mutable `:latest` tag. Different nodes may pull different versions, causing inconsistent behavior across replicas. Rollbacks are impossible because every revision points to the same tag.

## 6. Ingress Using Removed API Version

Uses `extensions/v1beta1` (removed in Kubernetes 1.22) with the deprecated `kubernetes.io/ingress.class` annotation, old backend syntax (`serviceName`/`servicePort`), and missing `pathType`. LLMs frequently generate this because training data contains many examples of the old API.

**Failure mode:** API drift.

## 7. Secret Data in a ConfigMap

Stores database passwords, API keys, and AWS credentials in a ConfigMap instead of a Secret. ConfigMaps are stored unencrypted in etcd and appear in plain text in `kubectl describe`, logs, and version control.

## 8. PVC with ReadWriteMany on an Unsupported Provider

Requests `ReadWriteMany` access mode with a `gp3` (AWS EBS) storage class. EBS volumes only support `ReadWriteOnce`. The PVC will be stuck in `Pending` state with no clear error. LLMs frequently pair RWX with block storage classes because they do not track provider-specific storage capabilities.

---

Each anti-pattern maps to one or more of KubeShark's six failure modes. The reference file includes the exact broken YAML so you can study the specific mistakes and understand why Kubernetes does not catch them at admission time.
