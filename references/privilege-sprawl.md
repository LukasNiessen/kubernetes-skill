# Privilege Sprawl

**Directive:** When generating RBAC resources, ServiceAccounts, or secret references, ALWAYS apply least-privilege principles. Default security posture is PSS "restricted" profile. Never grant permissions broader than the workload requires.

## When to use

Consult this reference whenever the task involves:
- Creating or modifying Roles, ClusterRoles, RoleBindings, or ClusterRoleBindings
- Creating ServiceAccounts or referencing them in pod specs
- Mounting or referencing Kubernetes Secrets
- Designing access patterns for controllers, operators, or application workloads

## Symptoms of privilege sprawl

| Symptom | Risk |
|---|---|
| ClusterRoleBinding with `cluster-admin` attached to a workload SA | Full cluster takeover if pod is compromised |
| Rules containing `verbs: ["*"]` or `resources: ["*"]` | Unrestricted access far beyond what the workload needs |
| Pods running with the `default` ServiceAccount | Every pod in the namespace shares the same identity |
| `automountServiceAccountToken: true` (the default) on pods that never call the API | Leaked token exposes unnecessary attack surface |
| Secrets injected as environment variables | Visible in `kubectl describe pod`, process listings, crash dumps |
| Team assumes base64-encoded Secrets are encrypted | Secrets stored in plaintext in etcd unless encryption-at-rest is configured |

## Root causes

1. Copy-pasting cluster-admin bindings from quickstart guides.
2. Using wildcards to "get it working" and never scoping down.
3. Not creating dedicated ServiceAccounts per workload.
4. Misunderstanding that Kubernetes Secrets are base64-encoded, NOT encrypted.
5. Injecting secrets via `env:` instead of volume mounts or external operators.

## Prevention rules

### RBAC least privilege

- **Role** is namespace-scoped. **ClusterRole** is cluster-scoped. Prefer Role unless access truly spans namespaces.
- **RoleBinding** binds a Role (or ClusterRole) within a single namespace. **ClusterRoleBinding** grants access cluster-wide.
- Never bind `cluster-admin` to any workload ServiceAccount. Reserve it for break-glass human access only.
- List specific verbs: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`. Never use `"*"`.
- List specific resources: `pods`, `deployments`, `configmaps`, etc. Never use `"*"`.
- Always specify `apiGroups` explicitly. An empty string `""` means core API group, not "all groups."

### ServiceAccount hardening

- Create a dedicated ServiceAccount for every workload that needs API access.
- Set `automountServiceAccountToken: false` on the ServiceAccount AND the Pod spec for workloads that do not call the Kubernetes API.
- Use projected token volumes with audience and expiration for workloads that do need API access.

### Secret management

- Kubernetes Secrets are base64-encoded, NOT encrypted. Anyone with `get secrets` RBAC in the namespace can read them.
- Enable etcd encryption at rest via `EncryptionConfiguration` as a baseline.
- Prefer external secret management: `external-secrets-operator` syncing from AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault.
- `sealed-secrets` is an alternative: encrypt secrets client-side so they are safe to commit to git.
- Mount secrets as files (`volumeMounts`), not environment variables. File-mounted secrets can be rotated without pod restart and are not exposed in `kubectl describe`.

## Patterns and examples

### GOOD: Scoped RBAC + dedicated ServiceAccount + external secrets

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-processor
  namespace: orders
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/order-processor
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-processor-role
  namespace: orders
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "watch"]
    resourceNames: ["order-config"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-processor-binding
  namespace: orders
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: order-processor-role
subjects:
  - kind: ServiceAccount
    name: order-processor
    namespace: orders
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-db-creds
  namespace: orders
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: order-db-creds
  data:
    - secretKey: password
      remoteRef:
        key: prod/orders/db-password
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
  namespace: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-processor
  template:
    metadata:
      labels:
        app: order-processor
    spec:
      serviceAccountName: order-processor
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: processor
          image: registry.example.com/order-processor:v2.4.1
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true
          volumeMounts:
            - name: db-creds
              mountPath: /etc/secrets/db
              readOnly: true
      volumes:
        - name: db-creds
          secret:
            secretName: order-db-creds
```

### BAD: cluster-admin binding + default SA + env var secrets

```yaml
# DO NOT DO THIS
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: order-processor-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin          # grants full cluster control to a workload
subjects:
  - kind: ServiceAccount
    name: default              # shared by every pod in the namespace
    namespace: orders
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
  namespace: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-processor
  template:
    metadata:
      labels:
        app: order-processor
    spec:
      # serviceAccountName omitted -- uses "default"
      # automountServiceAccountToken defaults to true -- token exposed
      containers:
        - name: processor
          image: registry.example.com/order-processor:latest
          env:
            - name: DB_PASSWORD                   # visible in describe, logs, crash dumps
              value: "hunter2"                     # hardcoded plaintext password
            - name: DB_PASSWORD_FROM_SECRET
              valueFrom:
                secretKeyRef:
                  name: db-creds
                  key: password                    # still exposed via env, not file mount
```

### Token projection for workloads that need API access

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-consumer
  namespace: orders
spec:
  serviceAccountName: order-processor
  automountServiceAccountToken: false   # disable the default mount
  containers:
    - name: app
      image: registry.example.com/api-consumer:v1.0.0
      volumeMounts:
        - name: kube-api-token
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: kube-api-token
      projected:
        sources:
          - serviceAccountToken:
              audience: "https://kubernetes.default.svc"
              expirationSeconds: 3600
              path: token
```

## LLM mistake checklist

Before finalizing any RBAC or secret-related manifest, verify each item:

- [ ] No rule uses `verbs: ["*"]` -- every verb is listed explicitly
- [ ] No rule uses `resources: ["*"]` -- every resource is listed explicitly
- [ ] No rule uses `apiGroups: ["*"]` -- each API group is listed explicitly
- [ ] No ClusterRoleBinding references `cluster-admin` for a workload ServiceAccount
- [ ] A dedicated ServiceAccount is created (not relying on `default`)
- [ ] `automountServiceAccountToken: false` is set on pods that do not need API access
- [ ] Secrets are mounted as volumes, not injected as environment variables
- [ ] No hardcoded secret values appear in the manifest (use ExternalSecret, SealedSecret, or at minimum a Secret resource)
- [ ] `resourceNames` is used where possible to restrict access to specific named resources
- [ ] RoleBinding is preferred over ClusterRoleBinding unless cluster-wide scope is required
- [ ] Pod securityContext sets `runAsNonRoot: true`, drops all capabilities, enables seccomp

## Verification commands

```bash
# Check what a specific ServiceAccount can do
kubectl auth can-i --list --as=system:serviceaccount:orders:order-processor -n orders

# Check if a ServiceAccount can perform a specific action
kubectl auth can-i get secrets --as=system:serviceaccount:orders:order-processor -n orders

# Find all ClusterRoleBindings that reference cluster-admin
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " -> " + (.subjects[]? | .kind + "/" + .name)'

# Find RBAC rules with wildcard verbs or resources
kubectl get roles,clusterroles -A -o json | \
  jq -r '.items[] | select(.rules[]? | .verbs[]? == "*" or .resources[]? == "*") | .metadata.namespace + "/" + .metadata.name'

# List all pods using the default ServiceAccount
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.serviceAccountName == "default" or .spec.serviceAccountName == null) | .metadata.namespace + "/" + .metadata.name'

# Check if etcd encryption at rest is enabled (control plane access required)
kubectl get apiserver -o=jsonpath='{.items[0].spec.encryption}'

# Audit secrets exposed as environment variables
kubectl get pods -A -o json | \
  jq -r '.items[] | .metadata.namespace + "/" + .metadata.name as $pod | .spec.containers[]?.env[]? | select(.valueFrom.secretKeyRef != null) | $pod + " env:" + .name'
```
