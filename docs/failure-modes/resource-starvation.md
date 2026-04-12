# FM2: Resource Starvation

Every container in a Kubernetes cluster shares finite CPU, memory, and disk. Without explicit resource requests and limits, workloads compete unpredictably -- a single runaway process can starve an entire node. Resource starvation is the most common cause of production instability in Kubernetes and the hardest to diagnose after the fact.

## QoS Classes and Eviction Order

Kubernetes assigns a Quality of Service class to every pod based on how its resource fields are configured. This class determines eviction priority when a node runs out of resources:

| QoS Class | Condition | Eviction order |
|---|---|---|
| **Guaranteed** | Every container has `requests == limits` for both CPU and memory | Last evicted |
| **Burstable** | At least one container has `requests != limits` | Middle |
| **BestEffort** | No requests or limits on any container | First evicted |

A pod with no `resources` block at all is BestEffort. Under node memory pressure, the kubelet kills BestEffort pods first, then Burstable pods exceeding their requests, and Guaranteed pods only as a last resort. Running BestEffort in production is never acceptable.

## CPU Throttling: The Invisible Latency Killer

CPU is a compressible resource -- when a container hits its CPU limit, the kernel's Completely Fair Scheduler (CFS) throttles it rather than killing it. This causes latency spikes that are invisible in standard metrics. A container with a 250m CPU limit that needs 300m for a request will pause mid-execution, adding unpredictable delays.

Current best practice for most application workloads: set CPU requests but omit CPU limits. This allows bursting to available capacity without CFS throttling. Set CPU limits only when running in multi-tenant clusters that require hard fairness guarantees, or when Guaranteed QoS is specifically needed.

## Memory: Always Set a Limit

Memory is incompressible. When a container exceeds its memory limit, the kernel OOM-kills the process immediately. There is no throttling, no warning -- the process is terminated and the container restarts. Set memory limits 25-50% above observed p99 usage to absorb garbage collection spikes and temporary allocations.

## LimitRange and ResourceQuota

Namespace-level guardrails catch workloads that slip through without resource specifications:

- **LimitRange** sets default requests/limits for containers that omit them, and enforces min/max bounds per container. Without a LimitRange, a single container can request all available resources on a node.
- **ResourceQuota** caps aggregate resource consumption per namespace: total CPU, memory, pod count, PVC count. When a ResourceQuota exists, every pod must specify resources or admission is rejected.

Both should be present in every production namespace. LimitRange provides sensible defaults; ResourceQuota prevents a single namespace from starving the rest of the cluster.

## OOMKill Cascades

A particularly dangerous pattern occurs when OOMKills cascade. Pod A exceeds its memory limit and is killed. Its traffic shifts to pods B and C, which now handle more load, consume more memory, and also get OOMKilled. Within seconds, the entire service is in `CrashLoopBackOff`. This is especially common with JVM workloads where heap sizing does not account for off-heap memory, native threads, and container overhead.

## PodDisruptionBudgets

Without a PDB, voluntary disruptions (node upgrades, autoscaler scale-downs, `kubectl drain`) can terminate all replicas simultaneously. A PDB with `maxUnavailable: 1` ensures at least N-1 replicas remain running during planned disruptions. Critical rules:

- Never set `minAvailable` equal to `replicas` -- it blocks all voluntary disruptions including cluster upgrades.
- The PDB selector must exactly match the pod labels. A mismatched selector silently protects nothing.
- PDBs only protect against voluntary disruptions. Node crashes and OOMKills bypass PDB constraints.

## HPA Pitfalls

The Horizontal Pod Autoscaler scales replicas based on metrics, but misconfiguration causes more problems than it solves:

- **Target utilization too high (90%):** No headroom for traffic spikes. By the time new pods start, the existing pods are overwhelmed.
- **Target utilization too low (30%):** Wasteful. The cluster runs 3x the needed capacity.
- **No scale-down stabilization:** HPA scales down aggressively by default. A brief traffic dip removes pods, then the next spike overwhelms the reduced fleet. Set `scaleDown.stabilizationWindowSeconds: 300`.
- **HPA `minReplicas` below PDB `minAvailable`:** HPA scales down to a count that violates the disruption budget, causing node drains to block indefinitely.

## Topology Spread and Anti-Affinity

Three replicas on the same node provide zero high availability. A single node failure takes all of them down. Use `topologySpreadConstraints` to distribute pods across zones and nodes:

- Zone-level spread with `whenUnsatisfiable: DoNotSchedule` prevents all replicas from landing in one availability zone.
- Node-level spread with `whenUnsatisfiable: ScheduleAnyway` provides a soft preference that does not block scheduling in small clusters.

## What LLMs Get Wrong

1. **Omitting resources entirely.** The most common error. The pod becomes BestEffort and is evicted first under any pressure.
2. **Round-number guessing.** `cpu: 1` and `memory: 1Gi` without profiling. Requests should reflect measured steady-state usage, not arbitrary values.
3. **Setting CPU limits by default.** CFS throttling causes latency spikes. Omit CPU limits unless multi-tenancy or Guaranteed QoS requires them.
4. **Memory limit equal to request.** Zero headroom means any spike triggers OOMKill. Allow 25-50% margin.
5. **Forgetting PDB.** Multiple replicas without a PDB is false redundancy -- a node drain kills them all.
6. **Topology spread missing.** Three replicas with no spread constraints may all schedule to the same node.

## Real Incidents

- **Datadog outage (2023):** A cascading OOMKill across monitoring agents caused loss of observability during a separate infrastructure incident, delaying diagnosis by hours.
- **GitHub rate limiting regression:** CPU throttling on API servers caused p99 latency to spike from 50ms to 2s. Removing CPU limits restored performance immediately.
- **Zalando postmortem:** A missing PDB allowed a cluster upgrade to drain all pods of a critical payment service simultaneously, causing a 15-minute outage.

Resource management is not optimization -- it is correctness. A manifest without resource configuration is incomplete.

## Further Reading

- [Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pod Quality of Service Classes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [KubeShark Good Patterns](../examples/good-patterns.md)
