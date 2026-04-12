# FM3: Network Exposure

Kubernetes networking is flat by default. Every pod can reach every other pod on any port, across all namespaces. There is no firewall, no segmentation, no access control until you explicitly create NetworkPolicy resources. This default-open posture means a single compromised container can reach databases, internal APIs, and cloud metadata endpoints without restriction.

## The Default-Open Problem

Unlike traditional networks where firewalls deny traffic by default, Kubernetes starts with full connectivity. Installing a CNI plugin that supports NetworkPolicy (Calico, Cilium, Antrea) is necessary but not sufficient -- the plugin only enforces policies that exist. A namespace with zero NetworkPolicy objects allows all traffic regardless of the CNI plugin.

The correct baseline is a default-deny policy in every namespace, followed by explicit allow rules for required communication paths.

## Service Types and Exposure Risk

| Type | Exposure | Risk level |
|---|---|---|
| `ClusterIP` | Internal only | Low -- reachable only within the cluster |
| `NodePort` | Every node IP on a high port | High -- bypasses Ingress, no TLS, no auth |
| `LoadBalancer` | Public IP via cloud provider | Critical -- directly internet-facing |
| `ExternalName` | DNS alias to external service | Low -- no proxying, but DNS rebinding possible |

LLMs frequently generate `LoadBalancer` or `NodePort` Services when `ClusterIP` is sufficient. Always default to `ClusterIP` and expose externally only through an Ingress controller with TLS termination.

## The Silent Selector Mismatch

The most frustrating Kubernetes networking bug produces no error, no warning, and no log entry. When a Service `selector` does not match any pod labels, the Service gets zero Endpoints. Traffic sent to the Service simply vanishes -- connections time out or receive connection refused errors.

This happens because:
- The pod label says `app: api-server` but the Service selector says `app: api` (typo).
- The selector includes a version label that changes on deploy (e.g., `version: v2` in the selector, but the new pods have `version: v3`).
- Labels are case-sensitive: `App: api-server` does not match `app: api-server`.

Always verify with `kubectl get endpoints <service-name>` after any Service or Deployment change.

## NetworkPolicy AND/OR Logic

The most common NetworkPolicy mistake is confusing AND and OR semantics in `from`/`to` rules:

- **Same list item = AND:** A `namespaceSelector` and `podSelector` in the same `from` entry must both match.
- **Separate list items = OR:** Two separate `from` entries are unioned -- traffic matching either rule is allowed.

Getting this wrong can either block legitimate traffic or open traffic to the entire cluster. A single misplaced hyphen in YAML changes the behavior completely.

## Egress Policies and DNS

A default-deny egress policy blocks all outbound traffic including DNS resolution. If you forget to allow DNS (port 53 UDP and TCP to kube-system), every service lookup fails and the application appears to have network connectivity issues when it actually has a policy misconfiguration.

Always include a DNS egress rule when writing egress policies:

```yaml
egress:
  - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

## DNS Performance and ndots

Kubernetes defaults to `ndots: 5`, meaning any hostname with fewer than 5 dots triggers search domain expansion. For a call to `api.stripe.com` (2 dots), the resolver first tries `api.stripe.com.production.svc.cluster.local`, then `api.stripe.com.svc.cluster.local`, then `api.stripe.com.cluster.local`, and finally the actual address. This multiplies DNS queries by 4-5x for every external call.

Fix with `dnsConfig.options: [{name: ndots, value: "2"}]` or append a trailing dot to external hostnames (`api.stripe.com.`).

## Lateral Movement After Compromise

Without NetworkPolicy, an attacker who compromises a single pod can:
1. Scan the entire cluster network to discover services.
2. Access databases directly (bypassing application-level auth).
3. Reach the cloud metadata endpoint (169.254.169.254) to steal IAM credentials.
4. Pivot to other namespaces to access higher-privilege workloads.
5. Exfiltrate data to external endpoints without restriction.

NetworkPolicy is the primary control against lateral movement. It reduces the blast radius of any single compromise from "the entire cluster" to "the pods this workload is explicitly allowed to reach."

## What LLMs Get Wrong

1. **No NetworkPolicy at all.** The most common error. The generated manifests include Deployments and Services but no network segmentation.
2. **Ingress-only policies.** Writing a policy with only ingress rules still allows unrestricted egress. Always specify both `policyTypes: [Ingress, Egress]`.
3. **Forgetting DNS egress.** Blocking all egress without a DNS exception breaks all service discovery.
4. **NodePort as default.** Generating `type: NodePort` when `ClusterIP` would suffice, exposing the service on every node.
5. **Missing `ingressClassName`.** Omitting it in an Ingress resource relies on a default IngressClass that may not exist, causing silent 404s.
6. **Wrong port mapping.** Confusing the Service `port`, `targetPort`, and Ingress backend `port.number`. The Ingress backend references the Service port, not the container port.
7. **`hostNetwork: true` without justification.** Bypasses all NetworkPolicy enforcement entirely.

## Real-World Impact

Lateral movement is the primary attack vector in Kubernetes breaches. The 2022 Sysdig threat report found that 87% of container images contained a high or critical vulnerability, and the average time from initial compromise to lateral movement was under 10 minutes in clusters without NetworkPolicy.

Network segmentation is not optional security hardening -- it is the minimum viable defense for any multi-service deployment.

## Further Reading

- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [KubeShark Security Hardening Guide](../guides/security-hardening.md)
- [KubeShark Do/Don't Checklist](../examples/do-dont-checklist.md)
