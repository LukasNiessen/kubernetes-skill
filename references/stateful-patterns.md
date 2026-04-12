# StatefulSet Patterns -- Stateful Workloads

**Load this reference when generating:** StatefulSet, headless Service, PersistentVolumeClaim (stateful apps), VolumeSnapshot, or any workload requiring stable identity or persistent storage.

## When to Use a StatefulSet
When pods need: **stable network identity** (predictable DNS per pod), **stable per-pod storage** (PVC follows the pod across reschedules), or **ordered deployment** (sequential create/delete). Common: PostgreSQL, MySQL, Kafka, RabbitMQ, etcd, ZooKeeper, Redis Sentinel, Cassandra.

## StatefulSet vs Deployment
| Concern | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random suffix, interchangeable | Ordinal index, stable hostname |
| Storage | Shared PVC or none | Per-pod PVC via volumeClaimTemplates |
| Scaling | All pods equal | Ordered creation/deletion |
| DNS | Via Service only | Per-pod DNS via headless Service |

**Anti-pattern:** Using StatefulSet when a Deployment + single PVC (RWX) or external database suffices. If you only need storage (not per-pod identity), a Deployment with a PVC is simpler.

## Stable Network Identity
A headless Service (`clusterIP: None`) is **required**. It creates per-pod DNS: `<pod>.<headless-svc>.<ns>.svc.cluster.local`. Example: `postgres-0.postgres-headless.database.svc.cluster.local`.

## volumeClaimTemplates
Creates one PVC per pod. PVCs are **never auto-deleted** on scale-down (protects data).
- **StorageClass**: verify it matches durability needs. Never rely on the default class in prod.
- **Access mode**: `ReadWriteOnce` for databases. `ReadWriteOncePod` (1.27+ GA) for stricter guarantees.
- **Size**: plan for growth. PVCs can expand (if `allowVolumeExpansion: true`) but never shrink.

## Pod Management Policy
- `OrderedReady` (default): sequential 0, 1, 2... each must be Ready before next starts. Use for consensus systems.
- `Parallel`: all pods launch simultaneously. Use when pods initialize independently (Cassandra).

## Update Strategy
- **RollingUpdate**: reverse ordinal order. Set `partition` for canary -- pods >= partition get the new version.
- **OnDelete**: manual control. Pods update only when you delete them. Use for databases needing careful upgrade sequencing.

## Backup and Restore
- **VolumeSnapshot**: CSI snapshots for point-in-time backups. Automate with CronJobs or Velero.
- **Application-level**: always run logical backups (pg_dump, mysqldump) alongside snapshots -- snapshots alone can be crash-inconsistent.
- Test restores regularly. A backup never restored is not a backup.

## Example: PostgreSQL StatefulSet
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector: { app.kubernetes.io/name: postgres, app.kubernetes.io/component: database }
  ports: [{ port: 5432, targetPort: 5432, protocol: TCP }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  labels: { app.kubernetes.io/name: postgres, app.kubernetes.io/component: database }
spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy: { type: RollingUpdate, rollingUpdate: { partition: 0 } }
  selector:
    matchLabels: { app.kubernetes.io/name: postgres, app.kubernetes.io/component: database }
  template:
    metadata:
      labels: { app.kubernetes.io/name: postgres, app.kubernetes.io/component: database }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        seccompProfile: { type: RuntimeDefault }
      terminationGracePeriodSeconds: 120
      containers:
        - name: postgres
          image: postgres:16.2-bookworm
          ports: [{ containerPort: 5432, protocol: TCP }]
          env:
            - { name: PGDATA, value: /var/lib/postgresql/data/pgdata }
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef: { name: postgres-credentials, key: password }
          resources:
            requests: { cpu: 500m, memory: 1Gi }
            limits:   { cpu: "2", memory: 2Gi }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          readinessProbe:
            exec: { command: ["pg_isready", "-U", "postgres"] }
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            exec: { command: ["pg_isready", "-U", "postgres"] }
            initialDelaySeconds: 30
            periodSeconds: 30
          volumeMounts:
            - { name: data, mountPath: /var/lib/postgresql/data }
            - { name: tmp, mountPath: /tmp }
            - { name: run, mountPath: /var/run/postgresql }
      volumes:
        - { name: tmp, emptyDir: {} }
        - { name: run, emptyDir: {} }
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources: { requests: { storage: 50Gi } }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres
spec:
  maxUnavailable: 1
  selector:
    matchLabels: { app.kubernetes.io/name: postgres, app.kubernetes.io/component: database }
```

## LLM Mistake Checklist
1. **Missing headless Service.** StatefulSet requires `clusterIP: None`. Without it, pods get no stable DNS and `serviceName` validation fails.
2. **Forgetting `serviceName`.** Must match the headless Service name exactly. Omitting it is an API error.
3. **volumeClaimTemplates nested under `template.spec`.** It is a peer of `template`, not inside it.
4. **Expecting PVCs deleted on scale-down.** They are retained intentionally. Delete manually or set `persistentVolumeClaimRetentionPolicy` (1.27+).
5. **ReadWriteMany for single-node databases.** Use `ReadWriteOnce` or `ReadWriteOncePod`. RWX adds complexity and is rarely supported by performant storage.
6. **Low terminationGracePeriodSeconds.** Default 30s is insufficient for databases. Set 60-120s for clean shutdown.
7. **Omitting PGDATA subdirectory.** PostgreSQL needs the data dir as a subdirectory of the mount (e.g., `.../data/pgdata`) because the mount root may contain `lost+found`.
8. **No PodDisruptionBudget.** Stateful workloads are disruption-sensitive. Always create a PDB with `maxUnavailable: 1`.
