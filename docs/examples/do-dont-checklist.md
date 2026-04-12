# Do/Don't Quick Reference Checklist

A terse, actionable checklist of Kubernetes best practices organized by category. Each line is a standalone rule. The default security posture is the PSS restricted profile.

For the full checklist with every rule, see [references/do-dont-patterns.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/do-dont-patterns.md).

## Categories Covered

The checklist spans eight categories that map directly to KubeShark's failure modes:

| Category | Key concern |
|---|---|
| **Security Contexts** | runAsNonRoot, capabilities, seccomp, read-only filesystem |
| **RBAC** | Namespace-scoped roles, least-privilege verbs, no wildcards |
| **Resource Management** | Requests/limits, ResourceQuota, LimitRange, QoS class |
| **Networking** | Default-deny NetworkPolicy, DNS egress, ingressClassName |
| **Probes and Rollouts** | Readiness/liveness separation, revision history, zero-downtime |
| **Image Management** | Immutable tags, imagePullPolicy, private registry secrets |
| **Storage** | Access mode vs storage class, volumeClaimTemplates, no hostPath |
| **Configuration** | Secrets not ConfigMaps, ExternalSecrets, hash-based naming |
| **Namespaces and Isolation** | PSA labels, ResourceQuota per namespace, trust boundaries |

## How to Use

Use this checklist as a final review pass before applying any manifest to a cluster. Each DO/DON'T rule is self-contained -- you can check them individually without reading the surrounding context. The checklist is designed for both human review and LLM self-verification during manifest generation.

## Relationship to Failure Modes

The categories map directly to KubeShark's six named failure modes:

- **Security Contexts, RBAC** -- insecure workload defaults, privilege sprawl
- **Resource Management** -- resource starvation
- **Networking** -- network exposure
- **Probes and Rollouts, Image Management** -- fragile rollouts
- **Storage, Configuration, Namespaces** -- cross-cutting concerns that affect multiple failure modes

Every rule in the checklist exists because it prevents a specific, observed failure pattern. No generic advice is included unless it maps to a real failure mode.
