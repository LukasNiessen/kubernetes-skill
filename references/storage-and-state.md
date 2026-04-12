# Storage and State

**Directive:** When generating or reviewing any workload that persists data, ALWAYS configure StorageClass, PVC sizing, access modes, and reclaim policies correctly. Data loss from misconfigured storage is irreversible. Default security posture is PSS "restricted" profile.

## When to use

Consult this reference whenever the task involves:
- Any workload with persistent data (databases, file storage, caches)
- Creating or modifying PersistentVolumeClaims or StorageClasses
- Configuring StatefulSet volumeClaimTemplates
- Volume snapshots, backup/restore, or data migration
- Choosing between ephemeral and persistent storage

---

## PersistentVolume and PersistentVolumeClaim Model

- **PersistentVolume (PV):** A piece of storage provisioned in the cluster, either manually or dynamically.
- **PersistentVolumeClaim (PVC):** A request for storage by a workload. Binds to a PV that satisfies its requirements.
- **Dynamic provisioning** is the default and preferred approach. Manual PV creation is only needed for pre-existing storage (NFS shares, existing cloud disks).

The binding flow: PVC specifies `storageClassName`, size, and access mode. The provisioner for that StorageClass creates a PV automatically and binds it to the PVC.

---

## StorageClass Configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-retain
provisioner: ebs.csi.aws.com          # or pd.csi.storage.gke.io, disk.csi.azure.com
parameters:
  type: gp3                            # cloud-specific volume type
  encrypted: "true"
reclaimPolicy: Retain                  # CRITICAL for production data
volumeBindingMode: WaitForFirstConsumer  # bind PV only when a pod needs it
allowVolumeExpansion: true             # allow PVC resize without recreation
mountOptions:
  - noatime                            # reduce unnecessary metadata writes
```

Key fields:

| Field | Production value | Why |
|---|---|---|
| `reclaimPolicy` | `Retain` | `Delete` (the default!) destroys the underlying volume when the PVC is deleted. Use `Retain` for any data you care about. |
| `volumeBindingMode` | `WaitForFirstConsumer` | `Immediate` (the default) provisions the volume before a pod is scheduled, which can place the volume in a different availability zone than the pod. `WaitForFirstConsumer` provisions in the same zone as the pod. |
| `allowVolumeExpansion` | `true` | Without this, you must delete and recreate the PVC to resize -- causing data loss if `reclaimPolicy` is `Delete`. |

---

## Access Modes

| Mode | Abbreviation | Meaning | Typical support |
|---|---|---|---|
| `ReadWriteOnce` | RWO | One node can mount read-write | All block storage (EBS, PD, Azure Disk) |
| `ReadOnlyMany` | ROX | Many nodes can mount read-only | NFS, CephFS, cloud file storage |
| `ReadWriteMany` | RWX | Many nodes can mount read-write | NFS, CephFS, EFS, Azure Files -- NOT block storage |
| `ReadWriteOncePod` | RWOP | Exactly one pod can mount read-write (k8s 1.29+) | CSI drivers that support it |

Common mistake: requesting `ReadWriteMany` with a block storage provisioner (EBS, GCE PD). Block storage is physically attached to one node -- it cannot be RWX. Use a file storage solution for shared access.

---

## Volume Expansion

To expand a PVC, the StorageClass must have `allowVolumeExpansion: true`. Then patch the PVC:

```bash
kubectl patch pvc data-postgres-0 -n databases \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

For file systems, expansion happens online. For block storage, some CSI drivers require the pod to be restarted. Always check your CSI driver documentation.

---

## VolumeSnapshot for Backup and Restore

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-2025-03-15
  namespace: databases
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: data-postgres-0
---
# Restore from snapshot into a new PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-restored
  namespace: databases
spec:
  storageClassName: fast-retain
  dataSource:
    name: postgres-snapshot-2025-03-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

**Rule:** Always take a VolumeSnapshot before any destructive operation -- PVC deletion, StorageClass migration, or major application upgrade.

---

## Ephemeral Storage: emptyDir

`emptyDir` volumes are tied to the pod lifecycle -- they are deleted when the pod is removed. Use them for scratch space, caches, and temporary files:

```yaml
volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 100Mi        # ALWAYS set sizeLimit
  - name: cache
    emptyDir:
      medium: Memory           # backed by RAM (tmpfs), counts against memory limits
      sizeLimit: 256Mi
```

**Critical rule:** ALWAYS set `sizeLimit` on `emptyDir` volumes. Without it, a runaway process can fill the node's disk and cause eviction of all pods on that node.

---

## CSI Drivers Overview

| Environment | Default CSI driver | Notes |
|---|---|---|
| AWS EKS | `ebs.csi.aws.com` | Block storage (RWO only). Use EFS CSI for RWX. |
| GKE | `pd.csi.storage.gke.io` | Block storage. Use Filestore CSI for RWX. |
| Azure AKS | `disk.csi.azure.com` | Block storage. Use `file.csi.azure.com` for RWX. |
| Bare metal | Longhorn, Rook-Ceph, OpenEBS | Longhorn is simplest. Rook-Ceph for production-grade distributed storage. |

All major cloud CSI drivers support snapshots, volume expansion, and encryption.

---

## Data Protection Rules

1. **Production StorageClass must use `reclaimPolicy: Retain`.** `Delete` is acceptable only for ephemeral environments (CI, preview deploys).
2. **Take VolumeSnapshots before destructive changes.** PVC deletion, resize, migration.
3. **Test restore procedures regularly.** A backup you have never restored is not a backup.
4. **Encrypt volumes at rest.** Use CSI driver `parameters.encrypted: "true"` or cloud provider defaults.
5. **Use `ReadWriteOncePod` for databases.** Prevents accidental multi-attach that corrupts data.

---

## StatefulSet volumeClaimTemplates

StatefulSets create a PVC per replica automatically. See **stateful-patterns.md** for full StatefulSet configuration:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: fast-retain
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
```

This creates PVCs named `data-postgres-0`, `data-postgres-1`, `data-postgres-2`. PVCs created by `volumeClaimTemplates` are NOT deleted when the StatefulSet is deleted -- this is intentional to protect data.

---

## GOOD: StorageClass + PVC + Deployment

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-retain
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: production
spec:
  storageClassName: standard-retain
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-processor
  namespace: production
spec:
  replicas: 1                          # RWO -- single replica only
  selector:
    matchLabels:
      app: file-processor
  template:
    metadata:
      labels:
        app: file-processor
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10000
        runAsGroup: 10000
        fsGroup: 10000                 # ensures mounted volume is writable by this GID
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: processor
          image: registry.example.com/file-processor:v2.1.0
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: data
              mountPath: /data
            - name: tmp
              mountPath: /tmp
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              memory: 512Mi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: app-data
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
```

## BAD: Common Storage Mistakes

```yaml
# PROBLEMATIC - DO NOT USE
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  # no storageClassName -- uses cluster default, which likely has reclaimPolicy: Delete
  accessModes:
    - ReadWriteMany              # block storage CSI does not support RWX -- PVC stays Pending
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myapp:latest
          volumeMounts:
            - name: data
              mountPath: /data
            - name: scratch
              mountPath: /tmp
          # no securityContext, no resources
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: shared-data
        - name: scratch
          emptyDir: {}           # no sizeLimit -- can fill the node disk
```

Problems: no explicit StorageClass (defaults to Delete reclaim), RWX on block storage (will never bind), no `sizeLimit` on `emptyDir`, no `fsGroup` (mounted volume may not be writable by the non-root user), `:latest` image tag.

---

## LLM Mistake Checklist

Before finalizing any storage-related manifest, verify each item:

- [ ] **StorageClass `reclaimPolicy`** is `Retain` for production -- not the default `Delete`.
- [ ] **`volumeBindingMode: WaitForFirstConsumer`** is set to avoid cross-zone volume/pod mismatch.
- [ ] **Access mode matches the CSI driver** -- do not request `ReadWriteMany` from block storage.
- [ ] **`allowVolumeExpansion: true`** is set on the StorageClass to allow future resizing.
- [ ] **`emptyDir` volumes have `sizeLimit`** set -- an unbounded emptyDir can evict all pods on the node.
- [ ] **`fsGroup`** is set in the pod security context so the non-root user can write to mounted volumes.
- [ ] **VolumeSnapshot** is taken before any destructive operation (PVC deletion, migration).
- [ ] **Deployment replicas match access mode** -- do not set `replicas > 1` with `ReadWriteOnce` PVCs unless using `ReadWriteOncePod` or StatefulSet per-replica volumes.
