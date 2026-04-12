# Validation and Policy Enforcement

> When validating Kubernetes manifests, enforcing policies, or integrating checks
> into CI pipelines, follow these patterns. Default security posture is PSS
> "restricted" profile.

---

## Validation Layers

Apply in order, each catches different classes of errors:

1. **Client-side schema validation** (kubeconform/kubeval) -- catches structural YAML errors, unknown fields, wrong types.
2. **Policy enforcement** (Kyverno/OPA Gatekeeper) -- catches organizational rule violations.
3. **Server-side dry-run** (kubectl --dry-run=server) -- catches admission webhook rejections, quota violations, naming conflicts.

## kubeconform

Fast, offline schema validation against specific Kubernetes versions.

```bash
# Validate all manifests against K8s 1.29
kubeconform \
  -kubernetes-version 1.29.0 \
  -strict \
  -summary \
  -output json \
  manifests/

# With CRD schema support (e.g., for Prometheus Operator)
kubeconform \
  -kubernetes-version 1.29.0 \
  -strict \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  manifests/

# Validate Helm rendered output
helm template my-release ./chart -f values-prod.yaml | \
  kubeconform -kubernetes-version 1.29.0 -strict

# Validate Kustomize rendered output
kustomize build overlays/production | \
  kubeconform -kubernetes-version 1.29.0 -strict
```

- Always use `-strict` to reject unknown fields.
- Pin `-kubernetes-version` to the target cluster version.
- Use CRD schema registries for custom resources; without them, CRDs are silently skipped.

## kubectl Dry-Run

```bash
# Client-side: basic YAML parsing, no server contact
kubectl apply -f manifest.yaml --dry-run=client

# Server-side: full admission chain minus persistence
kubectl apply -f manifest.yaml --dry-run=server
```

- `--dry-run=client` catches only syntax errors. It does not validate against the cluster schema.
- `--dry-run=server` runs through all admission webhooks and validations. Requires cluster access.
- Server-side dry-run is the final gate before actual apply.

## Kyverno

YAML-native policy engine. Policies are Kubernetes resources.

### Require Resource Limits

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
  annotations:
    policies.kyverno.io/title: Require Resource Limits
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "All containers must have memory and cpu limits."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

### Require Standard Labels

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-labels
      match:
        any:
          - resources:
              kinds:
                - Deployment
                - StatefulSet
                - DaemonSet
      validate:
        message: "Must include app.kubernetes.io/name and app.kubernetes.io/version labels."
        pattern:
          metadata:
            labels:
              app.kubernetes.io/name: "?*"
              app.kubernetes.io/version: "?*"
```

- `ClusterPolicy` applies cluster-wide; `Policy` is namespace-scoped.
- `validationFailureAction: Enforce` blocks non-compliant resources; `Audit` only logs.
- Kyverno supports validate, mutate, generate, and verifyImages rule types.

## OPA Gatekeeper

Policy engine using Rego. Uses a two-object model: ConstraintTemplate defines the logic, Constraint applies it.

### Disallow Privileged Containers

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowprivileged
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container '%v' must not be privileged", [container.name])
        }
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Init container '%v' must not be privileged", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowPrivileged
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters: {}
```

- ConstraintTemplate defines reusable policy logic in Rego.
- Constraint instances apply the template with specific match criteria and parameters.
- Always check both `containers` and `initContainers` in Rego rules.

## Polaris

Score-based configuration auditing. Good for baseline posture assessment.

```bash
# CLI audit against manifests
polaris audit --audit-path manifests/ --format pretty

# Generate a score for CI gating
polaris audit --audit-path manifests/ --format score
# Fails CI if score < threshold (default 0)
polaris audit --audit-path manifests/ --set-exit-code-on-danger
```

## CI Pipeline Integration

Run validations in this order:

```
validate (kubeconform) -> lint (helm lint / kustomize build) -> policy-check (kyverno/polaris) -> dry-run (server)
```

### GitHub Actions Example

```yaml
name: Validate Kubernetes Manifests
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install tools
        run: |
          curl -sL https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz
          sudo mv kubeconform /usr/local/bin/

      - name: Render manifests
        run: |
          helm template my-release ./chart -f values-prod.yaml > rendered.yaml

      - name: Schema validation
        run: |
          kubeconform -kubernetes-version 1.29.0 -strict -summary rendered.yaml

      - name: Policy check
        uses: kyverno/action-install-cli@v0.2
        with:
          release: "v1.12.0"
      - run: |
          kyverno apply policies/ --resource rendered.yaml
```

## LLM Mistake Checklist

1. **Used `--dry-run` without `=client` or `=server`** -- bare `--dry-run` is deprecated and defaults to client; always be explicit.
2. **Forgot CRD schemas in kubeconform** -- custom resources pass validation silently with no schema, hiding errors.
3. **Kyverno `validationFailureAction: Audit` in production** -- logs violations but does not block them; use `Enforce`.
4. **Gatekeeper ConstraintTemplate missing `initContainers` check** -- privileged init containers bypass the policy.
5. **Policy match on `Pod` only** -- misses workloads created by Deployments; match the controller kind or use Kyverno auto-gen.
6. **kubeconform without `-strict`** -- unknown/misspelled fields pass validation silently.
7. **Skipped server-side dry-run in CI** -- client-side validation cannot catch webhook rejections or quota violations.
8. **Policy tested only on `apply`, not on `create`** -- some admission policies behave differently on update vs create operations.
