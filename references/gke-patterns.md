# GKE Patterns

**Load this reference when detected:** GKE, Google Kubernetes Engine, Autopilot, Standard, Workload Identity Federation for GKE, GKE Dataplane V2, GCE Ingress, Cloud Load Balancing, Filestore CSI, Persistent Disk CSI, or Config Sync.

## Why this matters

GKE guidance changes depending on Standard versus Autopilot, identity mode, dataplane, and Google Cloud load-balancing integration. Do not load this file for non-Google clusters.

## Cluster Mode

Capture whether the cluster is Standard or Autopilot.

- Autopilot enforces stronger platform constraints and may reject or mutate unsupported pod settings.
- Avoid host access, privileged workloads, and node-level assumptions for Autopilot unless the user explicitly confirms support.
- In Standard clusters, node pools, taints, and workload placement are user-managed; include scheduling and upgrade safety controls.
- Do not generate DaemonSet or privileged-agent patterns for Autopilot without checking compatibility.

## Workload Identity

Prefer Workload Identity Federation for GKE over service account JSON keys.

- Bind Kubernetes service accounts to Google Cloud IAM identities using the project and namespace/service account boundary.
- Never mount service account key files into pods for normal cloud API access.
- When NetworkPolicy is used with GKE Dataplane V2 and the workload needs Google Cloud auth, ensure egress to the metadata server is allowed.
- Keep ServiceAccount names stable because IAM bindings and manifests depend on them.

## Networking and Ingress

Controller-specific behavior matters.

- Do not copy nginx, AWS ALB, or AGIC annotations into GCE Ingress resources.
- For Google Cloud Load Balancing, verify Service, backend, health check, and NEG expectations.
- Prefer Gateway API only when the target cluster has the required GKE Gateway controller and CRDs.
- For Dataplane V2, validate NetworkPolicy behavior against GKE documentation rather than assuming another CNI's semantics.

## Storage

Choose storage by access pattern.

- Persistent Disk CSI: block storage for RWO-style workloads; account for zone or regional topology.
- Filestore CSI: shared file storage for RWX workloads.
- Do not assume volume snapshots are available until snapshot CRDs and the relevant CSI driver support are present.
- For StatefulSets, combine storage with topology spread and disruption controls.

## Config Sync and Fleet Policy

When Config Sync or Anthos/Fleet policy is detected:

- Treat Git as the source of truth for managed resources.
- Avoid imperative `kubectl edit` or manual drift fixes in generated runbooks.
- Keep namespace and cluster-scoped resources in the repository structure expected by the platform team.

## Validation

- `kubectl apply --dry-run=server -f <manifest>`
- `kubectl describe ingress <name>` for Google load balancer events
- `kubectl describe networkpolicy <name>` plus connectivity tests for Dataplane V2 behavior
- `kubectl get storageclass` before choosing PD or Filestore classes
- `gcloud container clusters describe <cluster> --region <region>` when cluster mode or Workload Identity status is unknown

## LLM Mistake Checklist

- Recommending service account JSON keys instead of Workload Identity Federation.
- Generating privileged/host-level workloads for Autopilot without compatibility checks.
- Mixing nginx or AWS ALB annotations into GCE Ingress.
- Forgetting metadata-server egress when restrictive NetworkPolicies and GCP auth are both present.
- Treating zonal Persistent Disks as freely movable across zones.
- Assuming Gateway API support without confirming installed controller/CRDs.

## Grounding Sources

- Workload Identity Federation for GKE: https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity
- GKE Dataplane V2: https://cloud.google.com/kubernetes-engine/docs/how-to/dataplane-v2
- Config Sync GitOps best practices: https://docs.cloud.google.com/kubernetes-engine/config-sync/docs/concepts/gitops-best-practices
