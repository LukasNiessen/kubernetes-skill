# Workload Patterns

Kubernetes provides five workload resource types, each designed for a specific execution model. Choosing the wrong type forces workarounds that break update semantics, storage management, and scaling behavior. This guide provides a decision framework for selecting the right workload type.

## Decision Matrix

| Workload Type | Execution Model | Pod Identity | Storage | Scaling |
|---|---|---|---|---|
| **Deployment** | Long-running, stateless | Interchangeable (random suffix) | Shared or none | HPA, manual replicas |
| **StatefulSet** | Long-running, stateful | Stable ordinal (0, 1, 2...) | Per-pod PVC via volumeClaimTemplates | Manual or custom |
| **DaemonSet** | One pod per node | Per-node | hostPath or emptyDir | Automatic (node count) |
| **Job** | Run-to-completion | Disposable | Temporary | completions + parallelism |
| **CronJob** | Scheduled run-to-completion | Disposable | Temporary | schedule-driven |

## Deployment

**Use when:** Pods are interchangeable and need no stable identity or persistent local storage. Web servers, REST/gRPC APIs, microservices, frontend proxies, stateless queue workers.

**Key considerations:**
- Always set `replicas >= 2` for production with a PodDisruptionBudget.
- Use `topologySpreadConstraints` to distribute across zones and nodes.
- Pair with HPA for elastic scaling. Set `scaleDown.stabilizationWindowSeconds` to prevent flapping.
- Never put `app.kubernetes.io/version` in `selector.matchLabels` -- selectors are immutable and this breaks upgrades.

**Common mistake:** Using a Deployment with a RWO PersistentVolumeClaim and `replicas > 1`. Only one pod can mount a RWO volume at a time. The second replica stays Pending. Use a StatefulSet with per-pod volumes or switch to RWX storage.

## StatefulSet

**Use when:** Pods need stable network identity (predictable DNS per pod), stable per-pod storage (PVC follows the pod across reschedules), or ordered deployment. Databases (PostgreSQL, MySQL), message brokers (Kafka, RabbitMQ), consensus systems (etcd, ZooKeeper).

**Key considerations:**
- Requires a headless Service (`clusterIP: None`) for per-pod DNS: `<pod>.<service>.<ns>.svc.cluster.local`.
- `volumeClaimTemplates` create one PVC per pod. PVCs are never auto-deleted on scale-down to protect data.
- `podManagementPolicy: OrderedReady` (default) creates pods sequentially. Use `Parallel` when pods initialize independently.
- Set `terminationGracePeriodSeconds` to 60-120 seconds for databases. The default 30 seconds is insufficient for clean shutdown.

**Common mistake:** Using a StatefulSet when a Deployment with a single PVC or an external database would suffice. If you only need storage (not per-pod identity), a Deployment is simpler. StatefulSets add operational complexity for ordered rollouts, scale-down behavior, and PVC lifecycle management.

## DaemonSet

**Use when:** Exactly one pod must run on every qualifying node. Log collectors (Fluent Bit, Vector), monitoring agents (node-exporter, Datadog), network plugins (Cilium), CSI node drivers, security agents (Falco).

**Key considerations:**
- DaemonSets have no `replicas` field. The scheduler places one pod per qualifying node automatically.
- Resources are multiplied across every node. 100m CPU x 200 nodes = 20 CPU cores cluster-wide. Be conservative with requests.
- Use `nodeSelector` or `nodeAffinity` to target specific node pools. Add tolerations for tainted nodes (control-plane, GPU).
- Use a custom PriorityClass (not `system-node-critical`) for application-level agents.

**Common mistake:** Specifying a `replicas` field. DaemonSets do not support it -- the API rejects the manifest.

## Job

**Use when:** Work runs to completion and then stops. Database migrations, data exports, ETL pipelines, one-time scripts, ML training runs.

**Key considerations:**
- `restartPolicy` must be `Never` or `OnFailure`. The default `Always` is rejected by the API for Jobs.
- Always set `activeDeadlineSeconds` to prevent runaway jobs.
- Always set `ttlSecondsAfterFinished` to auto-clean completed Jobs and their pods.
- Jobs may retry on failure. Every Job must be idempotent -- assume at-least-once execution.
- Use `podFailurePolicy` (1.26+) to distinguish retryable from fatal errors.

**Common mistake:** Using `restartPolicy: Always`, which is the default for pods but invalid for Jobs. LLMs frequently omit `restartPolicy` in Job specs, relying on the default that the API rejects.

## CronJob

**Use when:** Work runs on a recurring schedule. Report generation, cache warming, log rotation, periodic health checks, certificate renewal.

**Key considerations:**
- Set `concurrencyPolicy: Forbid` by default. Overlapping runs cause resource exhaustion and data corruption.
- Set `startingDeadlineSeconds` to skip runs that are too late (prevents burst of overdue jobs after controller downtime).
- Set `timeZone` explicitly. Without it, the schedule uses the controller's clock (typically UTC).
- CronJobs have three label levels (CronJob, jobTemplate, pod template). All three need consistent labels.

**Common mistake:** Leaving `concurrencyPolicy` at the default `Allow`, which permits overlapping runs. A CronJob that takes 10 minutes, scheduled every 5 minutes, will accumulate concurrent instances until the cluster runs out of resources.

## Anti-Patterns

- **StatefulSet for stateless workloads.** Adds unnecessary complexity. Use a Deployment.
- **Deployment for one-shot tasks.** The pod restarts forever after completion. Use a Job.
- **DaemonSet when only some nodes need the workload.** Use `nodeSelector` to target the correct subset, not a blanket DaemonSet with no selector.
- **CronJob for long-running daemons.** If the workload should run continuously, use a Deployment with HPA.

## Further Reading

- [Workloads](https://kubernetes.io/docs/concepts/workloads/)
- [KubeShark Good Patterns](../examples/good-patterns.md)
- [KubeShark Bad Patterns](../examples/bad-patterns.md)
