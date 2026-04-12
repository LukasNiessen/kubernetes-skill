# Job and CronJob Patterns -- Batch Processing

**Load this reference when generating:** Job, CronJob, or any one-off / scheduled batch workload (migrations, ETL, reports, cleanup).

## When to Use
- **Job**: finite work that runs to completion. Database migrations, data exports, ML training, one-time scripts.
- **CronJob**: recurring scheduled work. Report generation, cache warming, log rotation, periodic health checks.
- If the workload runs indefinitely, use a Deployment (even for queue workers -- scale with HPA).

## Job Configuration
| Field | Purpose | Guidance |
|---|---|---|
| `completions` | Pods that must succeed | Default 1. Increase for fan-out. |
| `parallelism` | Max concurrent pods | Default 1. Increase for parallelizable work. |
| `backoffLimit` | Retries before failure | Default 6. Lower (2-3) for non-transient errors. |
| `activeDeadlineSeconds` | Hard timeout | **Always set.** Prevents runaway jobs. |
| `ttlSecondsAfterFinished` | Auto-cleanup delay | **Always set.** 3600 is a safe default. |

## Completion Modes
- **NonIndexed** (default): pods are interchangeable. Job succeeds after `completions` pods succeed.
- **Indexed**: each pod gets `JOB_COMPLETION_INDEX` env var (0, 1, 2...). Use for partitioned/sharded work.

## TTL After Finished
Without `ttlSecondsAfterFinished`, finished Jobs and their pods accumulate forever. The TTL controller deletes the Job, pods, and logs -- ship logs externally if you need them longer.

## CronJob Patterns

**Schedule**: standard 5-field cron (`minute hour dom month dow`).
**Timezone** (1.27+): `timeZone: "America/New_York"` pins to a specific tz instead of controller clock.

**Concurrency policies**:
| Policy | Behavior | Default to |
|---|---|---|
| `Allow` | Overlapping runs permitted | Avoid unless jobs are independent |
| `Forbid` | Skip if previous still running | **Use this by default** |
| `Replace` | Cancel running, start new | Only when latest run is all that matters |

**startingDeadlineSeconds**: skip the run if more than N seconds late (prevents burst of overdue jobs after controller downtime).

## Idempotency
Jobs may retry (node failure, preemption). Every Job MUST be idempotent:
- Upserts, not inserts. Check for already-completed work. Write to unique output locations.
- Assume **at least once** execution, never exactly once.

## Pod Failure Policy (1.26+)
Handle different failures differently -- avoid retrying known-fatal errors:
```yaml
spec:
  backoffLimit: 3
  podFailurePolicy:
    rules:
      - action: FailJob
        onExitCodes:
          containerName: worker
          operator: In
          values: [1]               # bad config -- no point retrying
      - action: Ignore
        onPodConditions:
          - type: DisruptionTarget  # node drain -- retry without counting
```
Actions: `FailJob` (fail immediately), `Count` (count toward backoffLimit), `Ignore` (retry free).

## Example: Production CronJob
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  labels: { app.kubernetes.io/name: daily-report, app.kubernetes.io/component: batch }
spec:
  schedule: "30 3 * * *"
  timeZone: "UTC"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    metadata:
      labels: { app.kubernetes.io/name: daily-report, app.kubernetes.io/component: batch }
    spec:
      ttlSecondsAfterFinished: 86400
      activeDeadlineSeconds: 3600
      backoffLimit: 2
      template:
        metadata:
          labels: { app.kubernetes.io/name: daily-report, app.kubernetes.io/component: batch }
        spec:
          restartPolicy: Never
          securityContext:
            runAsNonRoot: true
            runAsUser: 10000
            runAsGroup: 10000
            fsGroup: 10000
            seccompProfile: { type: RuntimeDefault }
          containers:
            - name: report-generator
              image: registry.example.com/daily-report:2.1.0
              args: ["--date", "yesterday", "--output", "s3://reports/daily/"]
              resources:
                requests: { cpu: 500m, memory: 512Mi }
                limits:   { cpu: "2", memory: 2Gi }
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                capabilities: { drop: ["ALL"] }
              volumeMounts:
                - { name: tmp, mountPath: /tmp }
          volumes:
            - { name: tmp, emptyDir: { sizeLimit: 1Gi } }
```

## LLM Mistake Checklist
1. **Wrong restartPolicy.** Job pods MUST use `Never` or `OnFailure`. The default `Always` is rejected by the API.
2. **Missing activeDeadlineSeconds.** A stuck Job without a deadline runs forever. Always set an upper bound.
3. **Omitting ttlSecondsAfterFinished.** Completed Jobs accumulate indefinitely without this. Always set a TTL.
4. **ConcurrencyPolicy defaulting to Allow.** Most CronJobs should use `Forbid`. Overlapping runs cause resource exhaustion and data corruption.
5. **Labels missing on nested templates.** CronJobs have three label levels (CronJob, jobTemplate, pod template). All three need consistent labels.
6. **Indexed Job ignoring JOB_COMPLETION_INDEX.** Setting `completionMode: Indexed` is useless if the container never reads the index env var.
7. **Non-idempotent retried Jobs.** If `backoffLimit > 0`, the Job will retry. Inserts without upsert logic create duplicates.
8. **Schedule without timeZone.** Without `timeZone`, the schedule uses the controller's clock (typically UTC). Set it explicitly if you mean a specific timezone.
