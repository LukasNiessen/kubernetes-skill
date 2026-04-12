# Insecure Workload Defaults

**USE THIS GUIDE** when generating or reviewing any Kubernetes workload manifest
(Deployment, StatefulSet, DaemonSet, Job, CronJob, or bare Pod).
Default security posture: **PSS "restricted" profile** unless the user explicitly
requests otherwise and provides justification.

---

## Symptoms

- Containers running as root (UID 0) inside the cluster.
- Pods admitted without any `securityContext` at all.
- Linux capabilities not dropped, leaving `CAP_NET_RAW`, `CAP_SYS_ADMIN`, etc.
- `hostPath` volumes mounted into workload pods.
- Privileged containers that can escape to the node.
- PodSecurity admission webhook rejecting manifests at deploy time.
- CVE exploitation amplified by overly permissive container runtime settings.

---

## Root Causes

1. Upstream example manifests and Helm chart defaults rarely include security contexts.
2. LLMs train on those same permissive examples and reproduce them verbatim.
3. `securityContext` has both pod-level and container-level fields; omitting either leaves gaps.
4. Teams copy "it works in dev" manifests into production without hardening.
5. Confusion between PSS levels (privileged, baseline, restricted) leads to the wrong choice.

---

## Pod Security Standards Quick Reference

| Level        | When to use                                                        |
|--------------|--------------------------------------------------------------------|
| `restricted` | **Default for all workloads.** Enforces non-root, drops caps, etc. |
| `baseline`   | Minimum acceptable floor. Use only when restricted is impossible.  |
| `privileged` | CNI plugins, storage drivers, node-level agents. Never for apps.   |

Label namespaces to enforce:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Prevention Rules

### Container Security Context Baseline

Every container MUST include this block unless a specific, documented deviation is required:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
  seccompProfile:
    type: RuntimeDefault
```

### Pod-Level vs Container-Level Security Context

Pod-level fields apply to ALL containers including init containers:

```yaml
spec:
  securityContext:            # Pod-level
    runAsNonRoot: true
    runAsUser: 10000
    runAsGroup: 10000
    fsGroup: 10000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:        # Container-level (overrides/supplements pod-level)
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

Key distinctions:
- `runAsUser`, `runAsGroup`, `fsGroup`, `seccompProfile` belong at pod level.
- `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, `capabilities` belong at container level.
- `runAsNonRoot` can be set at either level; pod level is preferred for consistency.

### When Deviations Are Acceptable

Init containers sometimes need narrow capabilities. Always document the reason:

```yaml
initContainers:
  - name: fix-permissions
    # DEVIATION: requires CAP_CHOWN to set volume ownership before app starts.
    # Pod-level runAsNonRoot is still true; this container runs as root briefly.
    securityContext:
      runAsNonRoot: false
      runAsUser: 0
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
        add:
          - CHOWN
```

### Host Namespace Access

These fields MUST be `false` (or omitted, since false is the default) for application workloads:

```yaml
spec:
  hostNetwork: false   # Exposes pod to node network stack
  hostPID: false       # Allows seeing all node processes
  hostIPC: false       # Allows shared memory with node processes
  hostUsers: false     # Maps to host user namespace (k8s 1.28+)
```

### AppArmor and Seccomp

Seccomp `RuntimeDefault` is mandatory under PSS restricted. For additional confinement:

```yaml
metadata:
  annotations:
    # AppArmor (becomes a first-class field in k8s 1.30+)
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault    # Or Localhost with a custom profile
```

---

## Patterns

### GOOD: Production Deployment with Full Security Hardening

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10000
        runAsGroup: 10000
        fsGroup: 10000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: registry.example.com/api-server:v2.4.1@sha256:abc123...
          ports:
            - containerPort: 8080
              protocol: TCP
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /var/cache/app
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
      serviceAccountName: api-server
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
        - name: cache
          emptyDir:
            sizeLimit: 128Mi
```

### BAD: Typical LLM-Generated Deployment (Missing All Controls)

```yaml
# INSECURE - DO NOT USE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: api-server:latest          # No registry, mutable tag, no digest
          ports:
            - containerPort: 8080
          # No securityContext at all
          # No resource requests or limits
          # No readOnlyRootFilesystem
          # Capabilities not dropped
          # automountServiceAccountToken defaults to true
```

Problems with the bad example:
1. No pod-level or container-level `securityContext` -- runs as root.
2. `image: api-server:latest` -- mutable tag, no registry prefix, no digest pinning.
3. No `resources` -- becomes BestEffort QoS, first to be evicted.
4. Default service account token mounted -- unnecessary API access.
5. Missing `namespace` -- deploys to whatever context is active.
6. Single replica -- no availability guarantee.
7. No `readOnlyRootFilesystem` -- writable container filesystem aids attackers.
8. Capabilities not dropped -- container retains default Linux capabilities.

---

## LLM Mistake Checklist

Before emitting any workload manifest, verify every item:

- [ ] **securityContext present at BOTH pod and container level** -- not just one.
- [ ] **`runAsNonRoot: true`** set at pod level (not just assumed from the image).
- [ ] **`allowPrivilegeEscalation: false`** on every container including init containers.
- [ ] **`capabilities.drop: [ALL]`** on every container -- never omitted, never partial.
- [ ] **`readOnlyRootFilesystem: true`** with `emptyDir` mounts for `/tmp` and write paths.
- [ ] **`seccompProfile.type: RuntimeDefault`** at pod level.
- [ ] **`automountServiceAccountToken: false`** unless the workload calls the Kubernetes API.
- [ ] **Image uses a digest or immutable tag** -- never `:latest` or bare image names.
- [ ] **No `hostPath` volumes** unless explicitly requested with justification.
- [ ] **No `hostNetwork`, `hostPID`, `hostIPC`** unless explicitly requested.
- [ ] **No `privileged: true`** unless explicitly requested for infrastructure components.
- [ ] **`runAsUser`/`runAsGroup` set to non-zero values** -- do not leave them unset.

---

## Verification Commands

```bash
# Check if a manifest passes PSS restricted validation (dry-run)
kubectl apply --dry-run=server -f manifest.yaml

# Inspect running pod security contexts
kubectl get pod <pod> -o jsonpath='{.spec.securityContext}' | jq .
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].securityContext}' | jq .

# Validate manifest schema
kubeconform -strict -kubernetes-version 1.30.0 manifest.yaml

# Scan with kubesec (static analysis)
kubesec scan manifest.yaml

# Check namespace PSS enforcement labels
kubectl get namespace <ns> -o jsonpath='{.metadata.labels}' | jq .

# Audit running workloads for security issues
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.runAsNonRoot != true) | .metadata.name'

# Check for privileged containers cluster-wide
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged == true) | "\(.metadata.namespace)/\(.metadata.name)"'

# Verify no hostPath volumes in use
kubectl get pods -A -o json | jq '.items[] | select(.spec.volumes[]?.hostPath != null) | "\(.metadata.namespace)/\(.metadata.name)"'
```
