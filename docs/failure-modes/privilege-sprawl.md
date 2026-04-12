# FM4: Privilege Sprawl

Privilege sprawl occurs when workloads accumulate more Kubernetes API access than they need. It compounds the impact of every other failure mode -- a compromised container with `cluster-admin` permissions turns a single vulnerability into a full cluster takeover. RBAC misconfigurations are silent, hard to audit, and rarely reviewed after initial setup.

## RBAC Fundamentals

Kubernetes RBAC has four resource types:

- **Role**: grants permissions within a single namespace.
- **ClusterRole**: grants permissions cluster-wide or across all namespaces.
- **RoleBinding**: binds a Role (or ClusterRole) to subjects within one namespace.
- **ClusterRoleBinding**: binds a ClusterRole to subjects across the entire cluster.

The principle of least privilege means using namespace-scoped Roles unless the workload genuinely needs cluster-wide access. Most application workloads need zero Kubernetes API access at all.

## Wildcard Permissions

Rules containing `verbs: ["*"]`, `resources: ["*"]`, or `apiGroups: ["*"]` grant unrestricted access. A single wildcard rule can negate every other security control in the cluster. Wildcards appear frequently in quickstart guides and Helm chart defaults because they "just work" -- but they grant far more access than any workload needs.

Always enumerate specific verbs (`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`), specific resources (`pods`, `configmaps`, `deployments`), and specific API groups (`""`, `apps`, `batch`). Use `resourceNames` to restrict access to specific named resources when possible.

## The Default ServiceAccount Problem

Every namespace has a `default` ServiceAccount. Every pod that does not specify `serviceAccountName` uses it. Every pod that uses it shares the same identity. This means:

- A single RoleBinding granting permissions to the `default` SA affects every pod in the namespace.
- If any pod in the namespace is compromised, the attacker inherits whatever permissions the `default` SA has.
- RBAC audit trails cannot distinguish between workloads using the same SA.

The fix: create a dedicated ServiceAccount for every workload. Set `automountServiceAccountToken: false` on both the ServiceAccount and the Pod spec for workloads that never call the Kubernetes API (which is most of them).

## automountServiceAccountToken

By default, Kubernetes mounts a service account token into every pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token grants whatever permissions the SA has. For workloads that never call the Kubernetes API (web servers, batch processors, data pipelines), this token is pure attack surface.

Setting `automountServiceAccountToken: false` on the pod spec removes the token mount entirely. For workloads that do need API access, use projected token volumes with explicit audience and expiration instead of the legacy static token.

## Secrets Are Not Encrypted

The most dangerous misconception about Kubernetes Secrets is that they are secure. They are not:

- **Base64 is not encryption.** `kubectl get secret -o yaml` shows the value. `echo <value> | base64 -d` decodes it. Any user or ServiceAccount with `get secrets` RBAC in the namespace can read every secret.
- **etcd stores secrets in plaintext by default.** Without explicit `EncryptionConfiguration`, secrets are stored unencrypted in the cluster's backing store.
- **Environment variable injection exposes secrets.** Secrets injected via `env.valueFrom.secretKeyRef` are visible in `kubectl describe pod`, process listings (`/proc/<pid>/environ`), and crash dumps.

The hardened approach:
1. Mount secrets as files via `volumeMounts`, not environment variables.
2. Enable etcd encryption at rest as a baseline.
3. Use external secret management (External Secrets Operator with AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault) for production secrets.
4. Use Sealed Secrets for secrets that must be stored in Git.

## Token Projection for API Access

When a workload genuinely needs to call the Kubernetes API, use bound service account token volumes instead of the default mount:

```yaml
volumes:
  - name: kube-api-token
    projected:
      sources:
        - serviceAccountToken:
            audience: "https://kubernetes.default.svc"
            expirationSeconds: 3600
            path: token
```

Projected tokens are short-lived and audience-scoped, limiting the damage if the token is leaked.

## What LLMs Get Wrong

1. **Binding `cluster-admin` to workload ServiceAccounts.** The most dangerous mistake. Appears in quickstart-style outputs when the LLM does not know the specific permissions needed.
2. **Using wildcards for convenience.** `verbs: ["*"]` and `resources: ["*"]` appear frequently because they avoid enumeration.
3. **Omitting `serviceAccountName`.** The pod silently uses the `default` SA, sharing identity with every other pod in the namespace.
4. **Leaving `automountServiceAccountToken` at default.** The token is mounted even when the workload never calls the API.
5. **Injecting secrets as environment variables.** Using `env.valueFrom.secretKeyRef` instead of volume mounts.
6. **Hardcoding secret values in manifests.** Plaintext passwords in `env.value` fields, committed to version control.
7. **Treating base64 as encryption.** Generating a Secret resource and assuming the data is protected.

## Real-World Impact

- **Shopify Kubernetes bug bounty:** An attacker gained access to a pod with excessive RBAC permissions, then used `kubectl` from inside the pod to read secrets from other namespaces.
- **Kubernetes CVE-2018-1002105:** A privilege escalation vulnerability in the API server. Clusters where workloads already had broad RBAC permissions experienced full compromise; clusters with least-privilege RBAC contained the blast radius.
- **Uber breach (2022):** While not Kubernetes-specific, the pattern -- hardcoded credentials in source code -- is identical to the secrets-in-env antipattern that LLMs reproduce.

Privilege sprawl is cumulative and invisible until exploitation. Every unnecessary permission is an expansion of the attack surface that persists indefinitely unless explicitly revoked.

## Further Reading

- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [KubeShark Security Hardening Guide](../guides/security-hardening.md)
- [KubeShark Do/Don't Checklist](../examples/do-dont-checklist.md)
