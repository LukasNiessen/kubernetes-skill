# FM1: Insecure Workload Defaults

Kubernetes does not ship with secure defaults. Pods created without explicit security contexts run as root, retain all Linux capabilities, and have writable root filesystems. This is the single most impactful failure mode for LLM-generated manifests because training data overwhelmingly consists of insecure examples.

## Why This Matters

OWASP Kubernetes Top Ten ranks "Insecure Workload Configurations" as **K01** -- the number one risk. A compromised container running as root with full capabilities can escape to the host node, access the cloud metadata service, pivot to other workloads, and exfiltrate secrets. Every missing security control compounds the blast radius.

## Security Context: Pod-Level vs Container-Level

Kubernetes splits security settings across two scopes, and both must be configured:

- **Pod-level** (`spec.securityContext`): applies to all containers including init containers. This is where `runAsNonRoot`, `runAsUser`, `runAsGroup`, `fsGroup`, and `seccompProfile` belong.
- **Container-level** (`spec.containers[].securityContext`): per-container overrides. This is where `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, and `capabilities` belong.

Omitting either level leaves gaps. A pod-level `runAsNonRoot: true` without container-level `capabilities.drop: [ALL]` still retains dangerous capabilities like `CAP_NET_RAW` (used for ARP spoofing and network sniffing within the cluster).

## Pod Security Standards

Kubernetes enforces security through Pod Security Admission (PSA), which evaluates pods against three profiles:

| Profile | Purpose | Typical use |
|---|---|---|
| **Restricted** | Full hardening: non-root, drop all caps, read-only FS, seccomp required | All application workloads (the KubeShark default) |
| **Baseline** | Prevents known privilege escalations but allows running as root | Legacy apps that cannot run as non-root |
| **Privileged** | No restrictions at all | CNI plugins, CSI drivers, node-level agents only |

PSA is enforced via namespace labels. A namespace without these labels has no enforcement -- pods run with whatever the manifest specifies, including fully privileged.

## Capabilities and Privilege Escalation

Linux capabilities grant fine-grained privileges. The default Docker/containerd capability set includes `CAP_NET_RAW`, `CAP_SETUID`, `CAP_SETGID`, and others that attackers exploit for container escapes. The hardened baseline is:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

If a workload genuinely needs a specific capability (e.g., `NET_BIND_SERVICE` to bind port 443), add only that one capability back. Never leave the default set in place.

The `allowPrivilegeEscalation: false` field is equally critical. Without it, a process inside the container can gain more privileges than its parent process through setuid binaries or other escalation vectors. This field must be set at the container level -- setting it at the pod level has no effect.

## Host Namespace Access

Setting `hostNetwork`, `hostPID`, or `hostIPC` to `true` breaks the container isolation boundary entirely. `hostNetwork` exposes the pod to the node's network stack and bypasses all NetworkPolicy enforcement. `hostPID` lets the container see and signal every process on the node. These fields must be `false` (the default) for all application workloads.

## AppArmor and Seccomp

Seccomp restricts which system calls a container can make. The `RuntimeDefault` profile blocks dangerous syscalls like `ptrace` and `mount` while allowing normal application behavior. Under PSS restricted, `seccompProfile.type: RuntimeDefault` is mandatory at the pod level.

AppArmor provides mandatory access control on top of seccomp. As of Kubernetes 1.30, AppArmor has graduated to a first-class field (`securityContext.appArmorProfile`), replacing the older annotation-based approach (`container.apparmor.security.beta.kubernetes.io/<name>`). For clusters running 1.30+, use the native field. For older clusters, use the annotation. LLMs frequently mix these two approaches in the same manifest.

Custom seccomp profiles (`type: Localhost`) can further restrict syscall access beyond `RuntimeDefault`, but require the profile to be available on every node. Use `RuntimeDefault` as the starting point unless specific workload requirements demand a custom profile.

## What LLMs Get Wrong

LLMs reproduce patterns from their training data, which is dominated by quickstart guides and blog posts without security hardening. The most frequent errors:

1. **Omitting security context entirely.** The most common mistake. The generated manifest has no `securityContext` at either level.
2. **Setting `runAsNonRoot` but not `runAsUser`.** The kubelet checks the image metadata at runtime -- if the image specifies `USER root`, the pod fails to start with a confusing error.
3. **Dropping capabilities partially.** Dropping `SYS_ADMIN` but not all capabilities still leaves `NET_RAW`, `SETUID`, and others.
4. **Forgetting init containers.** Security context on main containers but not init containers leaves a privilege escalation window during pod startup.
5. **Confusing pod-level and container-level fields.** Putting `allowPrivilegeEscalation` at the pod level (where it is ignored) instead of the container level.
6. **Missing `readOnlyRootFilesystem`.** Without it, an attacker can write binaries into the container filesystem. Combine with `emptyDir` mounts for `/tmp` and any other write paths.

## Real-World Impact

- **Tesla cryptojacking (2018):** Kubernetes dashboard exposed without authentication, pods deployed with no security context, cryptominers ran as root on GPU nodes.
- **Shopify bug bounty (2020):** A container escape via `CAP_SYS_ADMIN` in a pod that did not drop capabilities, granting access to the underlying node.
- **Capital One breach (2019):** While not Kubernetes-specific, the pattern is identical -- overly permissive workload identity plus missing runtime restrictions enabled lateral movement from a single SSRF to full S3 access.

The common thread: every breach was amplified by workloads running with more privileges than they needed. Secure defaults are not optional -- they are the primary defense against turning a single vulnerability into a cluster-wide compromise.

## Further Reading

- [OWASP Kubernetes Top Ten - K01](https://owasp.org/www-project-kubernetes-top-ten/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [KubeShark Security Hardening Guide](../guides/security-hardening.md)
