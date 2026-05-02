# AKS Patterns

**Load this reference when detected:** AKS, Azure Kubernetes Service, Microsoft Entra Workload ID, Azure CNI, Azure CNI Overlay, kubenet, Application Gateway Ingress Controller, AGIC, Azure Disk CSI, Azure Files CSI, Azure Blob CSI, or Azure Policy for AKS.

## Why this matters

AKS has Azure-specific identity, networking, ingress, and storage behavior. Generic Kubernetes YAML often deploys but fails to authenticate, route, or mount volumes correctly. Do not load this file for non-Azure clusters.

## Identity

Prefer Microsoft Entra Workload ID for pod access to Azure resources.

- Enable OIDC issuer and workload identity at the cluster level before relying on pod identity.
- Add `azure.workload.identity/use: "true"` to pods that require workload identity.
- Annotate the Kubernetes ServiceAccount with `azure.workload.identity/client-id`.
- Restart pods after ServiceAccount identity annotation changes.
- Do not recommend the deprecated Microsoft Entra pod-managed identity path for new work.
- Never place Azure client secrets in Kubernetes Secrets unless the user explicitly accepts that risk and there is no workload-identity option.

## Networking

Capture AKS network plugin and outbound path before generating network-sensitive manifests.

- Azure CNI Overlay is the strategic path for many new clusters and for kubenet migration.
- kubenet is scheduled for AKS retirement on March 31, 2028; do not recommend it for new long-lived clusters.
- NetworkPolicy behavior depends on the selected policy engine and network plugin.
- For private clusters, verify DNS, egress, and private endpoint assumptions before recommending public endpoints.

## Ingress and Load Balancing

Choose the controller deliberately.

- AGIC and Application Gateway for Containers are Azure-specific; do not use nginx annotations with them.
- If using AGIC, verify Application Gateway SKU, managed identity permissions, subnet placement, and controller add-on status.
- Use Service type `LoadBalancer` for L4 exposure, but include internal/public load balancer annotations only when the requirement is explicit.
- Prefer Ingress or Gateway patterns for HTTP routing rather than exposing every workload through a public LoadBalancer.

## Storage

Choose Azure storage by access pattern.

- Azure Disk CSI: block storage for RWO-style workloads.
- Azure Files CSI: shared SMB/NFS file storage for RWX workloads.
- Azure Blob CSI: object-backed mount use cases; do not treat it as a generic database volume.
- Validate StorageClass names from the cluster instead of inventing them.

## Validation

- `kubectl apply --dry-run=server -f <manifest>`
- `kubectl describe pod <name>` for workload identity webhook injection and projected token issues
- `kubectl get ingress,svc -A` and controller logs for AGIC/Application Gateway issues
- `kubectl get storageclass` before selecting Azure Disk/File/Blob classes
- `az aks show --name <cluster> --resource-group <rg>` when identity, OIDC issuer, or network plugin is unknown

## LLM Mistake Checklist

- Using deprecated pod-managed identity for new AKS work.
- Missing the required workload identity pod label.
- Forgetting that ServiceAccount annotation changes require pod restart.
- Recommending kubenet for new long-lived clusters.
- Mixing nginx annotations into AGIC-managed Ingress resources.
- Treating Azure Disk as RWX storage.
- Assuming StorageClass names without checking the cluster.

## Grounding Sources

- Microsoft Entra Workload ID for AKS: https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
- Deploy Workload ID on AKS: https://learn.microsoft.com/en-us/azure/aks/workload-identity-deploy-cluster
- AKS kubenet retirement notice: https://learn.microsoft.com/en-us/azure/aks/configure-kubenet
- AKS CSI storage drivers: https://learn.microsoft.com/en-us/azure/aks/azure-blob-csi
