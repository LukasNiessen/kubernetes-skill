# Helm Chart Best Practices

Helm is the standard package manager for Kubernetes. KubeShark follows these conventions when generating or reviewing Helm charts. For full YAML examples and the LLM mistake checklist, see [references/helm-patterns.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/helm-patterns.md).

## Chart.yaml

Every chart must declare `apiVersion: v2` (mandatory for Helm 3), a SemVer `version` that bumps on every chart change, and an independent `appVersion` tracking the application release. The `type` field should be `application` or `library`. Include a concise `description` field.

Key rules:

- `version` follows SemVer and must change on every chart modification -- Helm repositories serve stale versions from cache if the version is not bumped
- `appVersion` tracks the application release independently of the chart version
- Declare sub-chart dependencies in `Chart.yaml` under `dependencies`, not in a separate `requirements.yaml`

## values.yaml Structure

Group values by resource type. Provide secure defaults that match the PSS restricted profile out of the box:

- **image** -- repository, tag (defaults to `appVersion`), pullPolicy
- **securityContext** -- `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: ["ALL"]`
- **resources** -- explicit requests and memory limits
- **probes** -- liveness and readiness paths, ports, and initial delays
- **ingress** -- disabled by default with `enabled: false`
- **serviceAccount** -- `create: true`, blank name, empty annotations

Document every section with `# --` comments so `helm-docs` can auto-generate documentation.

## Template Helpers (_helpers.tpl)

Define reusable named templates for `fullname`, `labels`, `selectorLabels`, and `serviceAccountName`. All templates should:

- Truncate names to 63 characters (Kubernetes DNS label limit)
- Support `nameOverride` and `fullnameOverride` values
- Use `include` (not `template`) so output can be piped to `nindent`

## Template Conventions

- Use {% raw %}`{{- ... -}}`{% endraw %} whitespace trimming to prevent blank lines in rendered output.
- Always pipe string values through {% raw %}`{{ .Values.foo | quote }}`{% endraw %}.
- Use {% raw %}`{{ toYaml .Values.resources | nindent N }}`{% endraw %} for nested objects -- never render at column 0.
- Wrap optional resources in {% raw %}`{{- if .Values.ingress.enabled }}`{% endraw %} conditionals.
- Use {% raw %}`{{ required "message" .Values.key }}`{% endraw %} for values that must be supplied by the user.

## Dependency Management

Declare sub-charts in `Chart.yaml` under `dependencies`. Run `helm dependency update` to generate `Chart.lock`. Use `condition` or `tags` to make sub-charts optional. Commit both `Chart.yaml` and `Chart.lock` to version control.

## Security Defaults

Charts should ship with secure defaults out of the box. Users who need to relax security (e.g., for a CNI plugin that requires host networking) can override values explicitly, but the default path should produce a PSS-restricted-compliant workload.

Key defaults to include in every chart's `values.yaml`:

- Pod-level `securityContext` with `runAsNonRoot: true` and `seccompProfile: RuntimeDefault`
- Container-level `securityContext` with `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, and `capabilities.drop: ["ALL"]`
- `automountServiceAccountToken: false` unless the workload calls the Kubernetes API

## Testing Pipeline

Run these checks in order during development and CI:

1. **`helm lint ./chart`** -- catch syntax and structural errors
2. **`helm template release-name ./chart -f values-prod.yaml`** -- render manifests locally
3. **`kubeconform -kubernetes-version X.Y.0 -strict`** -- validate rendered output against target cluster schemas
4. **`helm test release-name`** -- run in-cluster test pods post-install

Integrate these steps into your CI pipeline so every chart change is validated before merge. The schema validation step (kubeconform) is especially important because `helm lint` does not validate against the Kubernetes API schema.

## Common LLM Mistakes

The most frequent Helm-specific errors LLMs produce include: missing {% raw %}`{{-`{% endraw %} whitespace control, omitting `| nindent N` on `toYaml` calls, forgetting to `quote` string values, hardcoding labels instead of using `include` helpers, not providing defaults for image tags, and not bumping the chart version. See the full checklist in the [reference file](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/helm-patterns.md#llm-mistake-checklist).
