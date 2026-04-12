# Kustomize Patterns

> When applying environment-specific customization or overlay-based configuration
> to Kubernetes manifests, follow these patterns. Default security posture is PSS
> "restricted" profile.

---

## Directory Structure

```
app/
  base/
    kustomization.yaml
    deployment.yaml
    service.yaml
    namespace.yaml
  overlays/
    dev/
      kustomization.yaml
      replica-patch.yaml
    staging/
      kustomization.yaml
    production/
      kustomization.yaml
      resource-patch.yaml
      hpa.yaml
  components/
    monitoring/
      kustomization.yaml
      servicemonitor.yaml
```

## kustomization.yaml Required Fields

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

Every kustomization.yaml must declare `apiVersion`, `kind`, and `resources` (or `bases` in legacy usage, but prefer `resources`).

## Common Transformers

```yaml
# Prefix/suffix all resource names
namePrefix: prod-
nameSuffix: -v2

# Add labels to all resources and their selectors
commonLabels:
  app.kubernetes.io/part-of: my-platform
  environment: production

# Add annotations to all resources
commonAnnotations:
  team: platform-eng

# Set namespace on all resources
namespace: production
```

## Patches

### Strategic Merge Patch

Use when you want to merge into an existing structure. Good for adding or overriding specific fields.

```yaml
# kustomization.yaml
patches:
  - path: resource-patch.yaml

# resource-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: my-app
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              memory: 1Gi
```

### JSON Patch

Use when you need to add, remove, or replace at a specific path. Required for array element manipulation.

```yaml
# kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: LOG_LEVEL
          value: "debug"
      - op: replace
        path: /spec/replicas
        value: 5
```

## ConfigMap and Secret Generators

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - DB_HOST=postgres.default.svc
  - name: app-scripts
    files:
      - scripts/init.sh

secretGenerator:
  - name: db-credentials
    literals:
      - username=admin
      - password=changeme
    type: kubernetes.io/basic-auth
```

Generators append a content hash to the name automatically, enabling rolling updates on config changes.

## Components

Reusable cross-cutting features that can be included in any overlay:

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - servicemonitor.yaml

patches:
  - target:
      kind: Deployment
    patch: |-
      - op: add
        path: /spec/template/metadata/annotations/prometheus.io~1scrape
        value: "true"
```

Include in an overlay:

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
components:
  - ../../components/monitoring
```

## Image Transformer

```yaml
images:
  - name: my-app
    newName: ghcr.io/org/my-app
    newTag: "v1.4.2"
  - name: sidecar
    newName: ghcr.io/org/sidecar
    digest: sha256:abcdef1234567890
```

Prefer `digest` over `newTag` in production for immutable references.

## When to Use Kustomize vs Helm

| Scenario | Kustomize | Helm |
|---|---|---|
| Environment-specific overlays on static manifests | Preferred | Overkill |
| Complex parameterization with many knobs | Awkward | Preferred |
| CRDs and operator-managed resources | Good fit | Good fit |
| Third-party chart consumption | Cannot | Required |
| Simple internal services with 2-3 envs | Preferred | Acceptable |
| Shared library of templates | Not supported | Library charts |

## Production Overlay Example

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - hpa.yaml

namespace: production

commonLabels:
  environment: production

patches:
  - path: resource-patch.yaml

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - ENABLE_DEBUG=false

images:
  - name: my-app
    newName: ghcr.io/org/my-app
    newTag: "v2.1.0"
```

## LLM Mistake Checklist

1. **Used `bases:` instead of `resources:`** -- `bases` is deprecated; always use `resources` for base references.
2. **Strategic merge patch missing `name` in metadata** -- Kustomize cannot match the patch to a resource without it.
3. **commonLabels applied to resources with immutable selectors** -- breaks Deployments on update because `spec.selector.matchLabels` is immutable after creation.
4. **Forgot content hash in Secret/ConfigMap references** -- hardcoded name in Deployment envFrom does not match generated name with hash suffix.
5. **JSON patch with wrong array index** -- use `/-` to append, explicit index to target a known position.
6. **Component declared with wrong apiVersion** -- components use `v1alpha1`, not `v1beta1`.
7. **Relative path wrong in overlay** -- must point to the directory containing kustomization.yaml, not individual files.
8. **Missing `behavior: merge` on generator overlay** -- creates a new ConfigMap instead of merging with the base one.
