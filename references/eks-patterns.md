# EKS Patterns

**Load this reference when detected:** EKS, AWS, IRSA, EKS Pod Identity, AWS Load Balancer Controller, AWS VPC CNI, EBS CSI, EFS CSI, Karpenter, or EKS add-ons.

## Why this matters

EKS looks like upstream Kubernetes until identity, load balancing, pod networking, storage, and node provisioning enter the design. Those surfaces are AWS-integrated and high-risk for LLM drift. Do not load this file for non-AWS clusters.

## Identity

Prefer short-lived pod identity over static AWS keys.

- Use one IAM role per workload or controller responsibility.
- Prefer EKS Pod Identity where the cluster and organization support it; otherwise use IRSA.
- For IRSA, annotate the Kubernetes ServiceAccount with `eks.amazonaws.com/role-arn`.
- For EKS Pod Identity, keep the ServiceAccount name and namespace stable because the pod identity association is bound to them.
- Set `automountServiceAccountToken: false` for workloads that do not call AWS or Kubernetes APIs.
- Never put AWS access keys in Secrets, ConfigMaps, Helm values, or CI artifacts.

## Load Balancing

Choose the controller by traffic type.

- HTTP/HTTPS: use Ingress or Gateway resources managed by the AWS Load Balancer Controller.
- L4 TCP/UDP: use `Service` type `LoadBalancer` with NLB-specific annotations only when required.
- Do not copy nginx, GCE, or AGIC annotations into AWS resources.
- Verify subnet tags and security group rules when a load balancer is requested but not provisioned.
- Treat controller annotations as version-sensitive; check the installed controller version before generating advanced annotations.

## Storage

Use the CSI driver that matches access semantics.

- EBS CSI: block storage, normally ReadWriteOnce, tied to zone scheduling.
- EFS CSI: shared file storage for ReadWriteMany workloads.
- For StatefulSets using EBS, include topology-aware scheduling expectations and do not assume a pod can move across zones without volume implications.
- Use `VolumeSnapshot` only when the snapshot CRDs and driver support are installed.

## Networking

AWS VPC CNI assigns pod IPs from the VPC address space.

- Watch subnet/IP exhaustion before increasing replicas or max pods.
- NetworkPolicy requires a compatible implementation; do not assume policy enforcement solely because the cluster is EKS.
- Security Groups for Pods change the boundary from node-level to pod-level security; use only when enabled and needed.
- Private clusters need VPC endpoints for controllers that call AWS APIs.

## Karpenter and Node Provisioning

When Karpenter is detected:

- Use current Karpenter APIs for `NodePool` and provider-specific node classes.
- Keep workload scheduling constraints explicit: requests, tolerations, node selectors, topology spread, and disruption sensitivity.
- Set consolidation/disruption behavior deliberately for stateful or latency-sensitive workloads.
- Do not let Karpenter compensate for missing resource requests; bad requests produce bad capacity decisions.

## Validation

- `kubectl apply --dry-run=server -f <manifest>`
- `kubectl describe service <name>` or `kubectl describe ingress <name>` for load balancer events
- `kubectl describe sa <name> -n <namespace>` for IRSA annotation checks
- `kubectl get pods -o wide` to verify zone/node placement for EBS-backed StatefulSets
- Check AWS controller logs for IAM denial, subnet discovery, or security group errors

## LLM Mistake Checklist

- Recommending static AWS keys in Kubernetes Secrets.
- Mixing IRSA annotations with EKS Pod Identity assumptions without naming which mechanism is used.
- Generating nginx or GCE Ingress annotations for AWS Load Balancer Controller.
- Treating EBS as ReadWriteMany storage.
- Omitting resource requests while also recommending Karpenter.
- Assuming NetworkPolicy is enforced without confirming the CNI/policy engine.
- Forgetting that ServiceAccount namespace/name changes can break identity bindings.

## Grounding Sources

- AWS EKS identity best practices: https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html
- EKS Pod Identity: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- EKS Karpenter best practices: https://docs.aws.amazon.com/eks/latest/best-practices/karpenter.html
