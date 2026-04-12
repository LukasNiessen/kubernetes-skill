# Validation and Policy Enforcement

Manifest validation and policy enforcement for Kubernetes, from offline schema checks to admission-time policy engines. For full configuration examples, CI pipeline templates, and the LLM mistake checklist, see [references/validation-and-policy.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/validation-and-policy.md).

## Validation Layers

Apply these three layers in order -- each catches different classes of errors:

1. **Client-side schema validation** (kubeconform) -- catches structural YAML errors, unknown fields, wrong types
2. **Policy enforcement** (Kyverno / OPA Gatekeeper) -- catches organizational rule violations
3. **Server-side dry-run** (`kubectl --dry-run=server`) -- catches admission webhook rejections, quota violations, naming conflicts

## kubeconform

Fast, offline schema validation against specific Kubernetes versions.

- Always use `-strict` to reject unknown or misspelled fields
- Pin `-kubernetes-version` to the target cluster version
- Use CRD schema registries for custom resources; without them, CRDs are silently skipped
- Validate Helm output: `helm template ... | kubeconform -strict`
- Validate Kustomize output: `kustomize build ... | kubeconform -strict`

## kubectl Dry-Run

- **`--dry-run=client`** -- basic YAML parsing only, no server contact, catches syntax errors
- **`--dry-run=server`** -- full admission chain minus persistence, runs through all webhooks and validations

Always use the explicit `=client` or `=server` form. Bare `--dry-run` is deprecated.

## Kyverno

YAML-native policy engine where policies are Kubernetes resources.

- `ClusterPolicy` applies cluster-wide; `Policy` is namespace-scoped
- `validationFailureAction: Enforce` blocks non-compliant resources; `Audit` only logs
- Supports validate, mutate, generate, and verifyImages rule types
- Common policies: require resource limits, require standard labels, restrict image registries

## OPA Gatekeeper

Policy engine using Rego with a two-object model:

- **ConstraintTemplate** -- defines reusable policy logic in Rego
- **Constraint** -- applies the template with specific match criteria and parameters
- Always check both `containers` and `initContainers` in Rego rules to prevent bypasses

## Polaris

Score-based configuration auditing, useful for baseline posture assessment:

- `polaris audit --audit-path manifests/` for local checks
- `polaris audit --set-exit-code-on-danger` for CI gating

## CI Pipeline Integration

Run validations in this order in your CI pipeline:

```
validate (kubeconform) -> lint (helm lint / kustomize build) -> policy-check (kyverno/polaris) -> dry-run (server)
```

A GitHub Actions example that chains these steps is available in the [reference file](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/validation-and-policy.md#github-actions-example).

## Common LLM Mistakes

Key validation and policy errors LLMs produce include: using bare `--dry-run` without `=client` or `=server`, omitting CRD schemas in kubeconform (hiding errors), setting Kyverno to `Audit` instead of `Enforce` in production, missing `initContainers` checks in Gatekeeper rules, matching only `Pod` kind (missing workloads created by Deployments), and skipping server-side dry-run in CI. See the full checklist in the [reference file](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/validation-and-policy.md#llm-mistake-checklist).
