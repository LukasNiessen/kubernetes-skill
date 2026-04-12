# Security Hardening

Defense-in-depth security for Kubernetes clusters, covering supply chain, admission, runtime, and network layers. KubeShark defaults to the PSS restricted profile for all generated workloads. For full configuration examples and the LLM mistake checklist, see [references/security-hardening.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/security-hardening.md).

## NSA/CISA Kubernetes Hardening Guide

Key control areas from the NSA/CISA guidance:

- **Pod security** -- use PSS restricted profile, non-root containers, read-only filesystems, drop all capabilities
- **Network separation** -- default-deny NetworkPolicy per namespace, encrypt traffic with service mesh mTLS
- **Authentication** -- disable anonymous auth, use short-lived tokens, integrate OIDC for human users
- **Authorization** -- RBAC with least privilege, no `cluster-admin` for workloads, regular RoleBinding audits
- **Audit logging** -- API server audit logging at Metadata level minimum, ship logs off-cluster
- **Threat detection** -- runtime monitoring with Falco or Tetragon for syscall and network anomaly detection
- **Upgrading** -- keep cluster and nodes within one minor version of latest, patch CVEs promptly

## OWASP Kubernetes Top 10

KubeShark maps each OWASP K8s risk to a specific reference file: insecure workload configurations (K01), supply chain vulnerabilities (K02), overly permissive RBAC (K03), lack of centralized policy enforcement (K04), inadequate logging (K05), broken authentication (K06), missing network segmentation (K07), secrets management failures (K08), misconfigured cluster components (K09), and outdated components (K10). See the full mapping in the reference file.

## CIS Kubernetes Benchmark

Critical checks organized by component:

- **Control plane** -- API server flags: `--anonymous-auth=false`, `--authorization-mode=RBAC,Node`, `--audit-log-path` set
- **etcd** -- client cert auth enabled, peer TLS enabled, access limited to API server only
- **Worker nodes** -- kubelet: `--anonymous-auth=false`, `--authorization-mode=Webhook`, `--read-only-port=0`
- **Policies** -- PSA enforced, NetworkPolicies present, ResourceQuotas applied

## Pod Security Admission (PSA)

Label every namespace with `enforce`, `audit`, and `warn` modes set to `restricted`. Using all three modes together catches violations at different stages. For gradual migration, enforce `baseline` while auditing and warning on `restricted`, then promote once compliant.

## Image Security and Supply Chain

- **Registry restrictions** -- use an admission webhook (Kyverno or Gatekeeper) to restrict image sources to approved registries
- **Vulnerability scanning** -- scan images in CI with Trivy before pushing, fail on CRITICAL and HIGH severity
- **Supply chain attestation** -- generate SBOMs with `syft` or `trivy sbom`, sign images with `cosign`, attach SLSA provenance

## Runtime Security

- **Falco** -- watches syscalls at the kernel level; create rules for shell spawns, sensitive file reads, and unexpected network connections
- **API server audit policy** -- log at `Metadata` level for secrets and configmaps, `RequestResponse` level for exec and attach operations

## etcd Encryption at Rest

Configure `EncryptionConfiguration` with `aescbc` or `secretbox` providers (never `identity`, which is plaintext). Pass `--encryption-provider-config` to the API server. After applying, re-encrypt existing Secrets with `kubectl get secrets -A -o json | kubectl replace -f -`.

## Network-Level Controls Beyond NetworkPolicy

NetworkPolicy provides segmentation but does not encrypt traffic. For in-transit encryption:

- **Service mesh mTLS** (Istio, Linkerd) -- encrypts all pod-to-pod traffic and provides identity-based authorization
- **DNS policies** -- restrict external DNS resolution to prevent data exfiltration
- **Egress gateways** -- force all outbound traffic through a controlled proxy for inspection and allowlisting

## Common LLM Mistakes

Key security errors LLMs produce include: setting only `enforce` without `audit` and `warn` PSA labels, using `identity` encryption instead of `aescbc`, omitting audit logging for secrets and exec operations, using cluster-scoped RBAC bindings when namespace-scoped ones suffice, auto-mounting service account tokens on pods that do not call the API, and relying on NetworkPolicy for encryption when mTLS is needed. See the full checklist in the [reference file](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/security-hardening.md#llm-mistake-checklist).
