# FM6: API Drift

Kubernetes follows a strict API deprecation lifecycle: beta APIs are introduced, stable APIs replace them, and beta APIs are eventually removed. LLMs hallucinate removed API versions more than any other type of Kubernetes error because their training data contains years of blog posts, tutorials, and Stack Overflow answers written for older cluster versions.

## The Deprecation Lifecycle

Every API migration follows the same pattern:

1. **Beta API introduced** -- a new resource or feature enters as `v1beta1` under an API group.
2. **Stable API introduced** -- the resource graduates to `v1`. The beta version is deprecated in the same release or shortly after.
3. **Beta API removed** -- typically 2-3 minor versions after deprecation, per the Kubernetes deprecation policy. From this point, the API server rejects manifests using the old version with a hard error.

"Deprecated" means the API still works but prints a warning. "Removed" means it fails. LLMs do not distinguish between these states.

## Major Migrations LLMs Get Wrong

### Ingress: `extensions/v1beta1` to `networking.k8s.io/v1`

Removed in Kubernetes 1.22. This is the most frequently hallucinated API version because Ingress existed as a beta for years (1.1 through 1.21) and generated enormous amounts of training data.

The structural changes in v1 are not just a version swap:
- `spec.backend` renamed to `spec.defaultBackend`.
- `serviceName` and `servicePort` (flat fields) replaced by `service.name` and `service.port.number` (nested).
- `pathType` is required on every path -- it was optional in beta.
- `ingressClassName` replaces the `kubernetes.io/ingress.class` annotation.

An LLM that generates `extensions/v1beta1` will also use the old field structure, compounding the error.

### PodDisruptionBudget: `policy/v1beta1` to `policy/v1`

Removed in Kubernetes 1.25. The v1 API makes `spec.selector` immutable after creation and adds `spec.unhealthyPodEvictionPolicy`. LLMs frequently generate `policy/v1beta1` because PDB examples in training data predate 1.25.

### HorizontalPodAutoscaler: `autoscaling/v2beta1` and `v2beta2` to `autoscaling/v2`

`v2beta1` removed in 1.25, `v2beta2` removed in 1.26. The key structural change: `targetAverageUtilization` (a top-level field in beta) moves to `target.averageUtilization` (nested under `target` in v2). LLMs mix beta and stable field structures unpredictably.

### Other Removed APIs

| Resource | Old API | Stable API | Removed in |
|---|---|---|---|
| CronJob | `batch/v1beta1` | `batch/v1` | 1.25 |
| EndpointSlice | `discovery.k8s.io/v1beta1` | `discovery.k8s.io/v1` | 1.25 |
| CSIDriver | `storage.k8s.io/v1beta1` | `storage.k8s.io/v1` | 1.22 |
| FlowSchema | `flowcontrol.apiserver.k8s.io/v1beta1` | `v1` | 1.26 |

## Schema Validation

There are two levels of manifest validity, and LLM-generated manifests can fail at either:

- **Structural validity:** Does the YAML conform to the schema for this API version? Caught by `kubeconform` or `--dry-run=server`. Wrong field names, wrong nesting, unknown fields.
- **Semantic validity:** Does the manifest make sense in context? Does the referenced Service exist? Is the port correct? Caught only at apply time or with policy tools.

`kubeconform` validates manifests against the OpenAPI schema for a specific Kubernetes version. Always pin the version to match your target cluster:

```bash
kubeconform -kubernetes-version 1.30.0 -strict manifests/
```

The `-strict` flag rejects unknown fields, which catches the common case where an LLM generates fields from one API version in a manifest tagged with a different version.

## Helm-Specific Drift

Helm templates can produce syntactically valid YAML that uses the wrong API version. The template renders without error, but `kubectl apply` fails on the cluster. Use `Capabilities.APIVersions` to branch on cluster version:

{% raw %}
```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
```
{% endraw %}

Another common Helm drift error: broken Go template expressions that fail silently. {% raw %}`{{ .Values.replicas }}`{% endraw %} evaluates to empty (not an error) if `replicas` is not defined in `values.yaml`. Always use defaults: {% raw %}`{{ .Values.replicas | default 3 }}`{% endraw %}.

## Kustomize-Specific Drift

Kustomize strategic merge patches specify a `target` with `group`, `version`, and `kind`. If the API group in the patch does not match the resource, the patch silently fails to apply -- no error, no warning, just unpatched output.

## What LLMs Get Wrong

1. **`extensions/v1beta1` for Ingress.** Removed since 1.22, but still the most common LLM-generated Ingress API version.
2. **Beta HPA API versions.** Mixing `autoscaling/v2beta1` field structures with `autoscaling/v2` API version, or vice versa.
3. **Flat Ingress backend fields.** Using `serviceName`/`servicePort` instead of the nested `service.name`/`service.port.number` structure.
4. **Missing `pathType` on Ingress paths.** Required in `networking.k8s.io/v1` but optional in beta. LLMs trained on beta examples omit it.
5. **`batch/v1beta1` for CronJob.** Removed since 1.25, but CronJob tutorials from the beta era are abundant in training data.
6. **No schema validation in the workflow.** LLMs generate manifests without suggesting validation, so errors are discovered only at deploy time.

## Prevention

The most effective defense against API drift is automated validation in the CI pipeline:

1. **`kubeconform`** with `-strict` and `-kubernetes-version` matching the target cluster.
2. **`pluto`** scans manifests, Helm charts, and running clusters for deprecated and removed APIs.
3. **`--dry-run=server`** validates against the live API server schema, catching CRD and admission webhook issues that offline tools miss.

Run all three: kubeconform in CI, pluto as a pre-commit check, and dry-run=server in the deployment pipeline.

## Further Reading

- [Kubernetes Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
- [API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [KubeShark Validation and Policy Guide](../guides/validation-and-policy.md)
