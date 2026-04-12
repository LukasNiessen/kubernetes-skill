# Security Hardening

**Directive:** When performing security reviews, hardening existing clusters, or preparing for compliance audits, ALWAYS follow defense-in-depth principles across the full stack: supply chain, admission, runtime, and network. Default security posture is PSS "restricted" profile.

## When to use

Consult this reference whenever the task involves:
- Hardening an existing Kubernetes cluster or namespace
- Preparing for a security audit or compliance review (SOC 2, PCI-DSS, HIPAA)
- Reviewing cluster configuration against industry benchmarks
- Implementing image security, supply chain controls, or runtime protection
- Configuring encryption at rest, audit logging, or network-level security

---

## NSA/CISA Kubernetes Hardening Guide -- Key Controls

| Control area | Summary |
|---|---|
| Pod security | Use PSS restricted, non-root containers, read-only filesystems, drop all capabilities |
| Network separation | Default-deny NetworkPolicy per namespace, encrypt traffic with service mesh mTLS |
| Authentication | Disable anonymous auth, use short-lived tokens, integrate OIDC for human users |
| Authorization | RBAC with least privilege, no `cluster-admin` for workloads, audit RoleBindings regularly |
| Audit logging | Enable API server audit logging at Metadata level minimum, ship logs off-cluster |
| Threat detection | Runtime monitoring (Falco, Tetragon), anomaly detection for syscalls and network |
| Upgrading | Keep cluster and nodes within one minor version of latest, patch CVEs promptly |

---

## OWASP Kubernetes Top 10 Mapping

| ID | Risk | Covered by |
|---|---|---|
| K01 | Insecure workload configurations | insecure-workload-defaults.md |
| K02 | Supply chain vulnerabilities | This file (supply chain section) |
| K03 | Overly permissive RBAC | privilege-sprawl.md |
| K04 | Lack of centralized policy enforcement | This file (admission webhooks) |
| K05 | Inadequate logging and monitoring | observability.md |
| K06 | Broken authentication mechanisms | This file (API server auth) |
| K07 | Missing network segmentation | network-exposure.md |
| K08 | Secrets management failures | This file (etcd encryption) |
| K09 | Misconfigured cluster components | This file (CIS benchmark) |
| K10 | Outdated and vulnerable components | This file (image scanning) |

---

## CIS Kubernetes Benchmark -- Key Sections

| Section | Critical checks |
|---|---|
| Control plane | API server: `--anonymous-auth=false`, `--authorization-mode=RBAC,Node`, `--audit-log-path` set |
| etcd | Client cert auth enabled, peer TLS enabled, access limited to API server only |
| Worker nodes | Kubelet: `--anonymous-auth=false`, `--authorization-mode=Webhook`, `--read-only-port=0` |
| Policies | PSA enforced, NetworkPolicies present, ResourceQuotas applied |

---

## Pod Security Admission Configuration

Label every namespace. Use `enforce` + `audit` + `warn` together to catch violations at different stages:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

For gradual migration, enforce `baseline` while auditing and warning on `restricted`, then promote.

---

## Image Security and Supply Chain

### Allowed registries and signing

Use an admission webhook (Kyverno or Gatekeeper) to restrict image sources:

```yaml
# Kyverno ClusterPolicy: restrict image registries
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-registries
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Images must come from registry.example.com."
        pattern:
          spec:
            containers:
              - image: "registry.example.com/*"
            initContainers:
              - image: "registry.example.com/*"
```

### Vulnerability scanning in CI

```yaml
# CI pipeline step -- scan with Trivy before push
- name: scan-image
  run: |
    trivy image --exit-code 1 --severity CRITICAL,HIGH \
      --ignore-unfixed \
      registry.example.com/myapp:${{ github.sha }}
```

### Supply chain attestation

- Generate SBOMs with `syft` or `trivy sbom` at build time.
- Sign images with `cosign sign` and verify in admission with `cosign verify`.
- Attach SLSA provenance using `slsa-verifier` to prove build origin.

---

## Runtime Security

### Falco rule example -- detect shell in container

Falco watches syscalls at the kernel level. Ship alerts to your SIEM:

```yaml
- rule: Terminal shell in container
  desc: A shell was spawned in a container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, dash)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name image=%container.image.repository)
  priority: WARNING
  tags: [container, shell]
```

### API server audit policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]
  - level: Metadata
    omitStages: ["RequestReceived"]
```

---

## etcd Encryption at Rest

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key-2024
              secret: <base64-encoded-32-byte-key>
      - identity: {}       # fallback for reading unencrypted data during migration
```

Pass `--encryption-provider-config` to the API server. After applying, re-encrypt existing Secrets: `kubectl get secrets -A -o json | kubectl replace -f -`.

---

## Network-Level Controls Beyond NetworkPolicy

- **Service mesh mTLS** (Istio, Linkerd): encrypts all pod-to-pod traffic and provides identity-based authz. NetworkPolicy alone does not encrypt traffic.
- **DNS policies**: restrict external DNS resolution to prevent data exfiltration.
- **Egress gateways**: force all outbound traffic through a controlled proxy for inspection and allowlisting.

---

## LLM Mistake Checklist

Before finalizing any security-related manifest or configuration, verify each item:

- [ ] **Pod Security Admission labels** are set on every namespace, not just the workload namespace.
- [ ] **All three PSA modes** (`enforce`, `audit`, `warn`) are configured -- not just `enforce` alone.
- [ ] **Image registry restrictions** are enforced via admission webhook, not just documented as policy.
- [ ] **etcd encryption** uses `aescbc` or `secretbox`, not `identity` (which is plaintext).
- [ ] **Audit logging** is enabled with at least `Metadata` level for secrets and exec operations.
- [ ] **RBAC bindings** are namespace-scoped (`RoleBinding`) not cluster-scoped unless required.
- [ ] **Service account tokens** are not auto-mounted (`automountServiceAccountToken: false` on pods that do not need API access).
- [ ] **No wildcard verbs or resources** in Roles (e.g., `verbs: ["*"]`, `resources: ["*"]`).
- [ ] **Image tags** are immutable (digest or semver), not `:latest`, and images are scanned for CVEs.
- [ ] **Network encryption** is addressed -- NetworkPolicy provides segmentation but not encryption; mTLS or a service mesh is needed for in-transit encryption.
