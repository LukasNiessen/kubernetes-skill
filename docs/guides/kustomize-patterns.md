# Kustomize Patterns

Kustomize provides template-free customization of Kubernetes manifests using overlays and patches. KubeShark follows these conventions when generating or reviewing Kustomize configurations. For full YAML examples and the LLM mistake checklist, see [references/kustomize-patterns.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/kustomize-patterns.md).

## Base/Overlay Structure

Organize manifests in a standard directory layout:

- **base/** -- contains the core `kustomization.yaml`, Deployment, Service, and Namespace manifests shared across all environments
- **overlays/dev/**, **overlays/staging/**, **overlays/production/** -- environment-specific customizations that reference the base
- **components/** -- reusable cross-cutting features (e.g., monitoring, network policies) that any overlay can include

Every `kustomization.yaml` must declare `apiVersion: kustomize.config.k8s.io/v1beta1`, `kind: Kustomization`, and a `resources` list. Use `resources` (not the deprecated `bases` field) for base references.

## Patches

**Strategic Merge Patch** -- merge into an existing resource structure. Best for adding or overriding specific fields like replica count or resource limits. The patch must include `metadata.name` to match the target resource.

**JSON Patch** -- add, remove, or replace at a specific path. Required for array element manipulation. Use `/-` to append to arrays, explicit indices to target known positions. Applied via inline `patch` blocks with a `target` selector.

## Generators

`configMapGenerator` and `secretGenerator` create ConfigMaps and Secrets with an automatic content hash appended to the name. This hash-based naming triggers rolling updates when configuration changes -- a significant advantage over manually managed ConfigMaps.

When overriding a base generator in an overlay, use `behavior: merge` to extend existing values rather than creating a duplicate resource.

## Components

Components use `apiVersion: kustomize.config.k8s.io/v1alpha1` and `kind: Component`. They package reusable features (ServiceMonitor resources, Prometheus scrape annotations, sidecar injections) that any overlay can opt into via the `components` field.

## Common Transformers

Kustomize provides several built-in transformers for cross-cutting modifications:

- **`namePrefix` / `nameSuffix`** -- add prefixes or suffixes to all resource names
- **`commonLabels`** -- add labels to all resources and their selectors (use with caution on mutable resources; see LLM mistakes)
- **`commonAnnotations`** -- add annotations to all resources
- **`namespace`** -- set the namespace on all resources in the kustomization

## Image Transformer

Override image references without patching the Deployment directly:

- Use `newTag` for tag overrides during development
- Use `digest` for immutable production references
- The image transformer matches on the `name` field in container image references, so the name must match exactly

## When to Use Kustomize vs Helm

| Scenario | Recommended |
|---|---|
| Environment-specific overlays on static manifests | Kustomize |
| Complex parameterization with many configuration knobs | Helm |
| Third-party chart consumption | Helm (required) |
| CRDs and operator-managed resources | Either |
| Simple internal services with 2-3 environments | Kustomize |
| Shared library of templates across teams | Helm (library charts) |

## Production Overlay Pattern

A typical production overlay references the base, sets the namespace, applies production labels, patches resource limits, overrides ConfigMap values with `behavior: merge`, pins image tags, and adds production-only resources like HPAs. See the reference file for a complete example.

## Common LLM Mistakes

The most frequent Kustomize-specific errors LLMs produce include: using the deprecated `bases` field, omitting `metadata.name` in strategic merge patches, applying `commonLabels` to resources with immutable selectors, forgetting content hashes in resource references, wrong array indices in JSON patches, and using the wrong `apiVersion` for components. See the full checklist in the [reference file](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/kustomize-patterns.md#llm-mistake-checklist).
