# Failure Modes

KubeShark organizes Kubernetes risks into six named failure modes. Every piece of guidance in the skill maps to at least one of these. Content that does not reduce the probability of any failure mode is excluded.

These are not arbitrary categories. They represent the six most common ways LLM-generated Kubernetes manifests cause real damage in production.

---

## 1. Insecure Workload Defaults

Containers running with overly permissive security settings because no explicit security context was provided.

**Symptoms:**
- Containers running as root (UID 0)
- Pods admitted without any `securityContext`
- Linux capabilities not dropped (`CAP_NET_RAW`, `CAP_SYS_ADMIN` still present)
- `hostPath` volumes mounted into workload pods
- Privileged containers that can escape to the node
- PodSecurity admission rejecting manifests at deploy time

**Common causes:**
- Upstream example manifests and Helm chart defaults rarely include security contexts
- LLMs train on those permissive examples and reproduce them verbatim
- `securityContext` has both pod-level and container-level fields; omitting either leaves gaps
- Confusion between PSS levels (privileged, baseline, restricted)

**Risk pattern:** A Deployment without a security context deploys successfully, runs as root, and becomes a container escape vector when a CVE is exploited. The cluster accepts it without complaint.

---

## 2. Resource Starvation

Workloads deployed without proper resource requests and limits, leading to scheduling failures, evictions, and cascading outages.

**Symptoms:**
- OOMKilled containers exceeding memory limits
- Pods stuck in Pending because the scheduler cannot find a node
- Node pressure evictions killing BestEffort pods
- CPU throttling causing invisible latency spikes
- Noisy neighbors starving co-located pods
- HPA flapping between replica counts

**Common causes:**
- Missing requests and limits entirely (BestEffort QoS, first to be evicted)
- Arbitrary round numbers (`cpu: 1`, `memory: 1Gi`) without profiling
- No PodDisruptionBudget -- voluntary disruptions take down all replicas
- CPU limits set too close to requests, causing constant CFS throttling
- No LimitRange to catch misconfigured pods at admission

**Risk pattern:** A pod without resource requests gets scheduled on an overcommitted node. Under load, the kubelet evicts it. The replacement pod lands on another overcommitted node. The cycle continues until the workload is effectively unavailable.

---

## 3. Network Exposure

Cluster networking left in the default open state, exposing all pods to all other pods and potentially to the internet.

**Symptoms:**
- All pods can reach all pods (Kubernetes default)
- Unexpected external exposure via `NodePort` or `LoadBalancer` Services
- DNS resolution failures from wrong Service names or missing namespace qualifiers
- Silent routing to nothing when Service selectors do not match pod labels
- Lateral movement after compromise because no NetworkPolicy exists
- Ingress 404s or 502s from path/backend mismatches

**Common causes:**
- Kubernetes has no network segmentation by default -- every pod can reach every other pod
- LLMs generate `NodePort` and `LoadBalancer` Services when `ClusterIP` is sufficient
- Service selectors silently fail when labels do not match (zero errors, zero traffic)
- No policy means allow-all, not deny-all
- Egress policies are forgotten -- ingress-only policies still allow unrestricted outbound

**Risk pattern:** A compromised pod in one namespace freely connects to the database in another namespace. No NetworkPolicy exists, so every service in the cluster is reachable. The blast radius of a single vulnerability is the entire cluster.

---

## 4. Privilege Sprawl

RBAC permissions, ServiceAccount tokens, and secret access granted far beyond what workloads actually require.

**Symptoms:**
- ClusterRoleBinding with `cluster-admin` attached to a workload ServiceAccount
- Rules containing `verbs: ["*"]` or `resources: ["*"]`
- Pods running with the `default` ServiceAccount (shared identity across the namespace)
- `automountServiceAccountToken: true` on pods that never call the Kubernetes API
- Secrets injected as environment variables (visible in `kubectl describe pod` and crash dumps)

**Common causes:**
- Copy-pasting `cluster-admin` bindings from quickstart guides
- Using wildcards to "get it working" and never scoping down
- Not creating dedicated ServiceAccounts per workload
- Misunderstanding that Kubernetes Secrets are base64-encoded, not encrypted
- Injecting secrets via `env` instead of volume mounts or external operators

**Risk pattern:** A web application pod runs with the default ServiceAccount, which has a ClusterRoleBinding to `cluster-admin` left over from initial setup. An SSRF vulnerability in the application allows an attacker to read the mounted token and take full control of the cluster.

---

## 5. Fragile Rollouts

Deployments that break during updates due to misconfigured probes, mutable image tags, or missing graceful shutdown handling.

**Symptoms:**
- Cascading restarts across all pods (liveness probe checks an external dependency)
- Dropped connections and 502s during deploys (readiness probe passes too early)
- All replicas unavailable simultaneously (`maxUnavailable` too high)
- Version drift across pods (`:latest` tag with cached layers)
- Pods killed before finishing in-flight requests (no preStop hook)
- Slow-starting apps killed in restart loops (no startup probe)

**Common causes:**
- Misunderstanding the difference between liveness and readiness probes
- Checking external dependencies (databases, APIs) in liveness probes
- Using `:latest` tags, which are mutable and nondeterministic
- Not setting `terminationGracePeriodSeconds` or preStop hooks
- `maxUnavailable` and `maxSurge` left at defaults without considering replica count

**Risk pattern:** A Deployment with a liveness probe that checks database connectivity deploys successfully. The database has a brief network blip. Every pod fails its liveness check simultaneously. Kubernetes restarts all pods at once, causing a full outage that outlasts the original database blip.

---

## 6. API Drift

Manifests using wrong, deprecated, or removed API versions that fail silently or break on cluster upgrades.

**Symptoms:**
- `no matches for kind "Ingress" in version "extensions/v1beta1"` (removed API)
- `Warning: policy/v1beta1 PodDisruptionBudget is deprecated` (deprecated, not yet removed)
- Fields silently ignored after upgrade (existed in beta, removed in stable)
- Helm templates render valid YAML but `kubectl apply` fails
- `kubeconform` reports schema violations

**Common causes:**
- LLM training data contains outdated manifests from blog posts and Stack Overflow
- Copy-paste from tutorials written for the Kubernetes 1.18-1.21 era
- Helm charts pinned to old API versions without `Capabilities` checks
- Not running schema validation against the target cluster version
- Confusing "deprecated" (still works, prints warning) with "removed" (hard failure)

**Risk pattern:** An LLM generates a manifest with `apiVersion: extensions/v1beta1` for an Ingress resource. This was removed in Kubernetes 1.22. The manifest looks correct, passes YAML linting, but fails on any modern cluster. The correct version is `networking.k8s.io/v1`.

---

## How Failure Modes Are Used

Failure modes drive the entire KubeShark workflow:

1. **Step 2 (Diagnose)** selects the relevant failure modes based on the task.
2. **Step 3 (Load references)** pulls the reference files that correspond to the diagnosed failure modes.
3. **Step 4 (Propose)** structures recommendations around preventing the specific risks identified.
4. **Step 7 (Output contract)** lists which failure modes were addressed, making the response auditable.

Most tasks involve multiple failure modes. A Deployment creation task typically triggers insecure workload defaults, resource starvation, and fragile rollouts at minimum. The workflow ensures none are overlooked.
