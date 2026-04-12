# Resource Starvation

**USE THIS GUIDE** when generating any workload manifest, performing capacity planning,
troubleshooting pod scheduling failures, or reviewing cluster reliability posture.
Every workload MUST have explicit resource management -- omitting it is a production incident waiting to happen.

---

## Symptoms

- **OOMKilled**: container exceeds its memory limit and is terminated by the kernel.
- **Pending pods**: scheduler cannot find a node with enough allocatable resources.
- **Node pressure evictions**: kubelet evicts BestEffort and Burstable pods under memory/disk pressure.
- **CPU throttling**: container hits its CPU limit and is throttled by CFS, causing latency spikes.
- **Noisy neighbors**: one pod without limits starves co-located pods of CPU or memory.
- **CrashLoopBackOff from OOM**: container repeatedly killed, backoff timer grows exponentially.
- **HPA flapping**: autoscaler thrashes between replica counts due to poorly tuned thresholds.

---

## Root Causes

1. **Missing requests and limits entirely** -- pod gets BestEffort QoS, first to be evicted.
2. **Arbitrary round numbers** -- `cpu: 1` and `memory: 1Gi` without profiling actual usage.
3. **No QoS strategy** -- mixing Guaranteed and BestEffort pods on the same node unpredictably.
4. **Requests set too low** -- scheduler packs too many pods per node; everything degrades under load.
5. **Limits set too close to requests** -- no room for legitimate burst; constant OOMKills or throttling.
6. **CPU limits causing latency** -- CFS throttling is invisible and worse than queueing in many cases.
7. **No LimitRange** -- a single misconfigured pod can consume an entire node.
8. **No PodDisruptionBudget** -- voluntary disruptions (upgrades, node drain) take down all replicas.

---

## QoS Classes

Kubernetes assigns QoS based on how requests and limits are set:

| QoS Class     | Condition                                        | Eviction priority | Use when                        |
|---------------|--------------------------------------------------|--------------------|---------------------------------|
| `Guaranteed`  | Every container has requests == limits for CPU and memory | Last evicted       | Latency-sensitive, databases    |
| `Burstable`   | At least one container has requests != limits     | Middle             | Most application workloads      |
| `BestEffort`  | No requests or limits set on any container        | First evicted      | **Never in production**         |

---

## Prevention Rules

### Resource Request/Limit Guidelines

**Requests** = expected steady-state usage. The scheduler uses this for placement.
**Limits** = hard ceiling. Exceeding memory limit causes OOMKill; exceeding CPU limit causes throttling.

### CPU: Prefer No Limit in Most Cases

Setting CPU limits causes CFS throttling, which introduces unpredictable latency spikes.
Current best practice for most workloads:

```yaml
resources:
  requests:
    cpu: 250m         # What the app typically uses
    # No CPU limit -- avoids CFS throttling
  limits:
    memory: 512Mi     # Memory limit is always required
```

Set CPU limits only when:
- Running in a multi-tenant cluster where fairness is enforced.
- The workload is batch/background and must not starve interactive pods.
- Guaranteed QoS is required (requests must equal limits).

### Memory: Always Set a Limit

Memory is incompressible. Unlike CPU (which throttles), exceeding memory causes OOMKill.
Always set a memory limit. Set it 25-50% above observed p99 usage to absorb spikes:

```yaml
resources:
  requests:
    memory: 256Mi     # Observed p99 steady-state
  limits:
    memory: 384Mi     # 50% headroom for spikes
```

### LimitRange: Namespace-Level Defaults and Guardrails

Prevents workloads from deploying without resource specs:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:                # Applied when limits are missing
        memory: 256Mi
        cpu: 500m
      defaultRequest:         # Applied when requests are missing
        memory: 128Mi
        cpu: 100m
      max:                    # Hard ceiling per container
        memory: 2Gi
        cpu: "2"
      min:                    # Minimum per container
        memory: 32Mi
        cpu: 10m
```

### ResourceQuota: Namespace-Level Aggregate Cap

Prevents a single namespace from consuming the entire cluster:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
```

### PodDisruptionBudgets

Required for any workload with more than one replica. Without a PDB, a node drain can
terminate all replicas simultaneously.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # Use ONE of minAvailable or maxUnavailable, not both.
  minAvailable: 2              # At least 2 replicas must remain during disruption
  # maxUnavailable: 1          # Alternative: at most 1 replica down at a time
  selector:
    matchLabels:
      app: api-server
```

- `minAvailable` -- use when you know the minimum replica count for correctness (e.g., quorum).
- `maxUnavailable` -- use for most stateless services; scales naturally with replica count.
- Never set `minAvailable` equal to `replicas` -- it blocks all voluntary disruptions including upgrades.

### HPA Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Target 70% of CPU request
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # Prevent flapping on scale-down
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
```

### Topology Spread and Pod Anti-Affinity

Distribute replicas across failure domains to survive node and zone failures:

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: api-server
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway     # Soft constraint for node spread
      labelSelector:
        matchLabels:
          app: api-server
```

---

## Patterns

### GOOD: Deployment with Proper Resource Management

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-server
      containers:
        - name: api
          image: registry.example.com/api-server:v2.4.1@sha256:abc123...
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              memory: 384Mi       # No CPU limit -- avoid CFS throttling
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            failureThreshold: 3
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: api-server
```

### BAD: Deployment with No Resource Management

```yaml
# UNRELIABLE - DO NOT USE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: api-server:latest
          ports:
            - containerPort: 8080
          # No resources -- BestEffort QoS, evicted first under pressure
          # No probes -- kubelet cannot detect unhealthy state
          # No topology spread -- all 3 replicas may land on same node
      # No PDB -- node drain kills all replicas simultaneously
```

Problems with the bad example:
1. No `resources` block -- BestEffort QoS, first to be evicted under node pressure.
2. No readiness probe -- traffic routed before app is ready; errors during startup.
3. No liveness probe -- hung process never restarted.
4. No topology spread -- all replicas may schedule to the same node or zone.
5. No PDB -- voluntary disruptions can take down 100% of replicas.
6. No namespace -- deploys wherever the current context points.
7. Mutable `:latest` tag -- different nodes may pull different versions.
8. 3 replicas with no anti-affinity is false redundancy.

---

## LLM Mistake Checklist

Before emitting any workload manifest, verify every item:

- [ ] **`resources.requests` set on every container** -- never omit; BestEffort is unacceptable.
- [ ] **`resources.limits.memory` set on every container** -- OOMKill is always worse than throttling.
- [ ] **CPU limits deliberately chosen or deliberately omitted** -- do not cargo-cult `cpu: 1`.
- [ ] **Requests reflect measured or estimated usage** -- not round numbers pulled from thin air.
- [ ] **Memory limit has headroom above request** -- at least 25% margin for GC spikes and bursts.
- [ ] **Readiness probe defined** -- without it, traffic arrives before the app can serve.
- [ ] **Liveness probe defined with conservative thresholds** -- avoid aggressive `failureThreshold: 1`.
- [ ] **PDB exists for any workload with replicas > 1** -- `maxUnavailable: 1` as a sensible default.
- [ ] **Topology spread or pod anti-affinity configured** -- replicas on one node is not HA.
- [ ] **LimitRange exists in the target namespace** -- catches pods that slip through without resources.
- [ ] **HPA `minReplicas` >= PDB `minAvailable`** -- otherwise scale-down can violate the disruption budget.
- [ ] **HPA target utilization is 60-80%** -- not 90% (no headroom) or 30% (wasteful scaling).

---

## Verification Commands

```bash
# Check QoS class of running pods
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.qosClass}{"\n"}{end}'

# Find pods with no resource requests (BestEffort candidates)
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].resources.requests == null) | "\(.metadata.namespace)/\(.metadata.name)"'

# Check for OOMKilled containers
kubectl get pods -A -o json | jq -r '.items[].status.containerStatuses[]? | select(.lastState.terminated.reason == "OOMKilled") | "\(.name): OOMKilled"'

# View actual resource usage vs requests (requires metrics-server)
kubectl top pods -n production --containers

# Check if PDB exists for a deployment
kubectl get pdb -n production -o wide

# Validate PDB is not blocking all disruptions
kubectl get pdb -n production -o jsonpath='{range .items[*]}{.metadata.name}{"\t allowed disruptions: "}{.status.disruptionsAllowed}{"\n"}{end}'

# Check node resource pressure conditions
kubectl describe nodes | grep -A5 "Conditions:" | grep -E "MemoryPressure|DiskPressure|PIDPressure"

# View HPA status and current metrics
kubectl get hpa -n production -o wide

# Find pods without topology spread constraints
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.topologySpreadConstraints == null) | "\(.metadata.namespace)/\(.metadata.name)"'

# Check LimitRange in namespace
kubectl get limitrange -n production -o yaml

# Check ResourceQuota usage
kubectl describe resourcequota -n production
```
