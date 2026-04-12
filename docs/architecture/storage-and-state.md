# Storage and State

Misconfigured storage is the only Kubernetes failure mode that can cause irreversible data loss. Unlike compute issues (which resolve by restarting pods) or network issues (which resolve by fixing policies), a deleted PersistentVolume with `reclaimPolicy: Delete` destroys the underlying disk permanently. Every storage decision must account for data durability.

## The PV/PVC Model

Kubernetes abstracts storage through three resources:

- **PersistentVolume (PV):** Represents a piece of provisioned storage -- a cloud disk, an NFS share, or a local SSD. PVs are cluster-scoped, not namespaced.
- **PersistentVolumeClaim (PVC):** A namespaced request for storage. Specifies size, access mode, and StorageClass. The control plane binds the PVC to a PV that satisfies its requirements.
- **StorageClass:** Defines how PVs are dynamically provisioned. Specifies the CSI driver, parameters (disk type, encryption), reclaim policy, and binding mode.

Dynamic provisioning is the default workflow: a PVC references a StorageClass, the CSI driver provisions a volume, and the control plane creates a PV and binds it to the PVC automatically.

## StorageClass: Critical Fields

Two StorageClass defaults are dangerous for production data:

**`reclaimPolicy: Delete`** (the default) destroys the underlying volume when the PVC is deleted. A single `kubectl delete pvc` command permanently deletes the data. Production StorageClasses must use `Retain`, which preserves the volume for manual recovery.

**`volumeBindingMode: Immediate`** (the default) provisions the volume before a pod is scheduled. This can place the volume in a different availability zone than the pod, causing the pod to stay Pending indefinitely. `WaitForFirstConsumer` provisions the volume in the same zone as the pod.

Always set `allowVolumeExpansion: true` so PVCs can be resized without recreation. PVCs can be expanded but never shrunk.

## Access Modes

| Mode | Abbreviation | Meaning | Supported by |
|---|---|---|---|
| `ReadWriteOnce` | RWO | One node mounts read-write | All block storage (EBS, PD, Azure Disk) |
| `ReadOnlyMany` | ROX | Many nodes mount read-only | NFS, CephFS, cloud file storage |
| `ReadWriteMany` | RWX | Many nodes mount read-write | NFS, CephFS, EFS, Azure Files |
| `ReadWriteOncePod` | RWOP | Exactly one pod mounts read-write | CSI drivers supporting RWOP (1.29+ GA) |

The most common mistake: requesting `ReadWriteMany` with a block storage provisioner. Block storage is physically attached to a single node and cannot support RWX. The PVC stays in `Pending` state with no clear error message. Use a file storage solution (EFS, Filestore, Azure Files) for shared access.

For databases, prefer `ReadWriteOncePod` over `ReadWriteOnce`. RWO allows multiple pods on the same node to mount the volume, which can cause data corruption. RWOP restricts access to exactly one pod.

## Dynamic Provisioning and CSI Drivers

Each cloud provider and storage platform has a CSI driver:

| Environment | Block storage CSI | File storage CSI |
|---|---|---|
| AWS EKS | `ebs.csi.aws.com` | `efs.csi.aws.com` |
| GKE | `pd.csi.storage.gke.io` | `filestore.csi.storage.gke.io` |
| Azure AKS | `disk.csi.azure.com` | `file.csi.azure.com` |
| Bare metal | Longhorn, Rook-Ceph, OpenEBS | Rook-CephFS, NFS provisioner |

All major CSI drivers support snapshots, volume expansion, and encryption. Always enable encryption (`parameters.encrypted: "true"`) for production StorageClasses.

## VolumeSnapshot for Backup and Restore

VolumeSnapshots provide point-in-time copies of PVCs. They are the primary mechanism for data protection before destructive operations:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot-2025-04-12
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: data-postgres-0
```

To restore, create a new PVC with `dataSource` referencing the snapshot. The CSI driver provisions a new volume from the snapshot data.

Critical rules for snapshots:
- Always snapshot before PVC deletion, StorageClass migration, or major upgrades.
- Snapshots may be crash-consistent, not application-consistent. For databases, run logical backups (pg_dump, mysqldump) alongside snapshots.
- Test restore procedures regularly. A backup never restored is not a backup.

## Ephemeral Storage: emptyDir

`emptyDir` volumes are tied to the pod lifecycle -- deleted when the pod is removed. Use them for scratch space, caches, and temporary files required by `readOnlyRootFilesystem: true`.

Always set `sizeLimit` on emptyDir volumes. Without it, a runaway process can fill the node's disk and trigger eviction of every pod on that node. For in-memory emptyDirs (`medium: Memory`), the size counts against the container's memory limit.

## StatefulSet volumeClaimTemplates

StatefulSets create one PVC per replica automatically. PVCs created by `volumeClaimTemplates` are intentionally not deleted when the StatefulSet is deleted or scaled down -- this protects data. To reclaim storage, delete the PVCs manually after verifying the data is no longer needed.

The `persistentVolumeClaimRetentionPolicy` field (1.27+) can configure automatic PVC deletion on scale-down or StatefulSet deletion, but use it with extreme caution in production.

## fsGroup and Permissions

When running containers as non-root with `readOnlyRootFilesystem: true`, mounted PVCs may not be writable because the volume's filesystem ownership does not match the container's user. Set `fsGroup` in the pod security context to ensure the mounted volume is writable by the pod's group:

```yaml
securityContext:
  runAsUser: 10000
  runAsGroup: 10000
  fsGroup: 10000
```

Without `fsGroup`, the pod mounts the volume but cannot write to it, causing application errors that appear to be permission issues inside the container.

## Further Reading

- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [KubeShark Resource Starvation](../failure-modes/resource-starvation.md)
