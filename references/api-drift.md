# API Drift

**Directive:** When generating any Kubernetes manifest, Helm chart, or Kustomize overlay, ALWAYS use the current stable API version for the target cluster version. Never assume an API version is correct from training data -- verify it. LLMs hallucinate deprecated and removed API versions frequently.

## When to use

Consult this reference whenever the task involves:
- Generating any Kubernetes manifest from scratch
- Upgrading manifests for a newer cluster version
- Writing or reviewing Helm templates
- Writing or reviewing Kustomize overlays and patches
- Validating manifests before applying to a cluster

## Symptoms of API drift

| Symptom | Cause |
|---|---|
| `error: unable to recognize: no matches for kind "Ingress" in version "extensions/v1beta1"` | Using a removed apiVersion |
| `Warning: policy/v1beta1 PodDisruptionBudget is deprecated` | Using deprecated but not yet removed apiVersion |
| Fields silently ignored after upgrade | Field existed in beta, removed or renamed in stable |
| `unknown field "spec.hard"` in `kubectl apply` | Structural schema validation rejects unknown fields in stable APIs |
| Helm template renders but `kubectl apply` fails | Template produces syntactically valid YAML with wrong apiVersion |

## Root causes

1. LLM training data contains outdated manifests from blog posts, Stack Overflow, and old documentation.
2. Copy-paste from tutorials written for Kubernetes 1.18-1.21 era.
3. Helm charts pinned to old API versions without `Capabilities` checks.
4. Not running schema validation against the target cluster version.
5. Confusing "deprecated" (still works, prints warning) with "removed" (hard failure).

## The API deprecation lifecycle

Kubernetes follows a predictable pattern:

1. **Beta API introduced** (e.g., `extensions/v1beta1 Ingress` in 1.1)
2. **Stable API introduced** (e.g., `networking.k8s.io/v1 Ingress` in 1.19)
3. **Beta API deprecated** (same release as stable introduction, or one release later)
4. **Beta API removed** (usually 2-3 minor versions after deprecation, per policy)

Once removed, the API server rejects manifests using that version. There is no graceful fallback.

## Major API migrations LLMs frequently get wrong

### Ingress: extensions/v1beta1 and networking.k8s.io/v1beta1 -> networking.k8s.io/v1

- Removed in: **Kubernetes 1.22**
- Key structural changes in v1:
  - `spec.backend` renamed to `spec.defaultBackend`
  - `serviceName` and `servicePort` replaced with `service.name` and `service.port.number` (or `service.port.name`)
  - `pathType` is now **required** on every path (was optional in beta)
  - `ingressClassName` replaces the `kubernetes.io/ingress.class` annotation

### PodDisruptionBudget: policy/v1beta1 -> policy/v1

- Removed in: **Kubernetes 1.25**
- Key changes in v1:
  - `spec.selector` is now immutable after creation
  - Unhealthy pod eviction policy field added (`spec.unhealthyPodEvictionPolicy`)

### HorizontalPodAutoscaler: autoscaling/v2beta1 and v2beta2 -> autoscaling/v2

- v2beta1 removed in: **Kubernetes 1.25**
- v2beta2 removed in: **Kubernetes 1.26**
- Key changes in v2:
  - `targetAverageUtilization` moved under `target.averageUtilization`
  - `metrics[].type` uses `ContainerResource` for per-container scaling
  - `behavior` field for scale-up/scale-down policies is stable

### FlowSchema/PriorityLevelConfiguration: flowcontrol.apiserver.k8s.io/v1beta1 -> v1beta3 -> v1

- v1beta1 removed in: **Kubernetes 1.26**
- v1beta2 removed in: **Kubernetes 1.29**
- v1beta3 removed in: **Kubernetes 1.32**

### Other common migrations

| Resource | Old API | Current Stable API | Removed in |
|---|---|---|---|
| CronJob | batch/v1beta1 | batch/v1 | 1.25 |
| EndpointSlice | discovery.k8s.io/v1beta1 | discovery.k8s.io/v1 | 1.25 |
| CSIDriver, CSINode | storage.k8s.io/v1beta1 | storage.k8s.io/v1 | 1.22 |
| CertificateSigningRequest | certificates.k8s.io/v1beta1 | certificates.k8s.io/v1 | 1.22 |
| TokenReview | authentication.k8s.io/v1beta1 | authentication.k8s.io/v1 | 1.22 |

## API version quick reference (current stable)

| Resource | apiVersion |
|---|---|
| Deployment, ReplicaSet, StatefulSet, DaemonSet | apps/v1 |
| Service, ConfigMap, Secret, Pod, Namespace | v1 |
| Ingress | networking.k8s.io/v1 |
| NetworkPolicy | networking.k8s.io/v1 |
| HorizontalPodAutoscaler | autoscaling/v2 |
| PodDisruptionBudget | policy/v1 |
| CronJob, Job | batch/v1 |
| ServiceAccount | v1 |
| Role, ClusterRole, RoleBinding, ClusterRoleBinding | rbac.authorization.k8s.io/v1 |
| PersistentVolumeClaim, PersistentVolume | v1 |
| StorageClass | storage.k8s.io/v1 |
| IngressClass | networking.k8s.io/v1 |
| EndpointSlice | discovery.k8s.io/v1 |
| ValidatingWebhookConfiguration | admissionregistration.k8s.io/v1 |

## Schema validation

### Structural vs semantic validity

A manifest can be valid YAML and even match the general shape of a Kubernetes resource while still being wrong:
- **Structural validity**: "Does this YAML parse? Do the fields exist in the schema?" -- caught by `kubeconform` or `--dry-run=server`.
- **Semantic validity**: "Does this make sense? Does the referenced Service exist? Is the port correct?" -- only caught at apply time or with policy tools.

### kubeconform usage

```bash
# Validate against a specific Kubernetes version
kubeconform -kubernetes-version 1.29.0 -strict manifests/

# Validate with CRD schemas (e.g., from datreeio/CRDs-catalog)
kubeconform -kubernetes-version 1.29.0 \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  manifests/

# Validate Helm output
helm template my-release ./chart | kubeconform -kubernetes-version 1.29.0 -strict
```

### kubectl dry-run

- `--dry-run=client`: validates locally against the client's built-in schema. Fast but may be outdated.
- `--dry-run=server`: sends the request to the API server for validation without persisting. More accurate -- catches unknown fields, CRD validation, admission webhooks.

```bash
# Server-side dry-run (preferred)
kubectl apply -f manifest.yaml --dry-run=server

# Client-side dry-run (no cluster needed)
kubectl apply -f manifest.yaml --dry-run=client
```

## Helm-specific drift errors

- **Broken Go templates**: `{{ .Values.replicas }}` fails if `replicas` is not defined in `values.yaml`. Always use `{{ .Values.replicas | default 3 }}` or check with `{{ if .Values.replicas }}`.
- **API version in templates**: Use `Capabilities.APIVersions` to branch on cluster version:

```yaml
{{- if .Capabilities.APIVersions.Has "networking.k8s.io/v1" }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}
```

- **Missing Chart.yaml fields**: `apiVersion: v2` is required for Helm 3. `type: application` (default) or `type: library` must be valid.

## Kustomize-specific drift errors

- **Invalid patch target**: the `target` in a strategic merge patch must specify the correct `group`, `version`, `kind`. A wrong API group silently fails to match.
- **Wrong resource in kustomization.yaml**: listing a file with a removed apiVersion causes `kustomize build` to fail.

## Patterns and examples

### GOOD: Manifest with correct current apiVersions

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: frontend
spec:
  ingressClassName: nginx                       # not annotation
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix                    # required in v1
            backend:
              service:
                name: web-frontend
                port:
                  number: 8080                  # nested under service.port
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend-hpa
  namespace: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

### BAD: Manifest mixing deprecated and removed apiVersions

```yaml
# DO NOT DO THIS
apiVersion: extensions/v1beta1           # REMOVED in 1.22
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: nginx   # replaced by spec.ingressClassName
spec:
  backend:                               # renamed to defaultBackend in v1
    serviceName: web-frontend            # flat fields replaced by nested service block
    servicePort: 8080
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
                                         # missing pathType (required in v1)
            backend:
              serviceName: web-frontend
              servicePort: 8080
---
apiVersion: autoscaling/v2beta1          # REMOVED in 1.25
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        targetAverageUtilization: 70     # moved to target.averageUtilization in v2
```

## LLM mistake checklist

Before finalizing any Kubernetes manifest, verify each item:

- [ ] Every `apiVersion` is checked against the quick reference table above, not generated from memory
- [ ] Ingress uses `networking.k8s.io/v1`, NOT `extensions/v1beta1` or `networking.k8s.io/v1beta1`
- [ ] Ingress paths include `pathType` (required in v1)
- [ ] Ingress backend uses nested `service.name` / `service.port.number`, not flat `serviceName` / `servicePort`
- [ ] HPA uses `autoscaling/v2`, NOT `v2beta1` or `v2beta2`
- [ ] PodDisruptionBudget uses `policy/v1`, NOT `policy/v1beta1`
- [ ] CronJob uses `batch/v1`, NOT `batch/v1beta1`
- [ ] No `extensions/v1beta1` appears anywhere in the output
- [ ] If targeting a specific cluster version, all apiVersions are validated against that version
- [ ] Helm templates use `Capabilities.APIVersions` checks when supporting multiple cluster versions
- [ ] `kubeconform` or `--dry-run=server` validation is included in the workflow

## Verification commands

```bash
# Scan for deprecated APIs in manifests using pluto
pluto detect-files -d manifests/
pluto detect-helm -owide

# Scan for deprecated APIs in a running cluster
pluto detect-api-resources --cluster

# Validate manifests against a specific Kubernetes version
kubeconform -kubernetes-version 1.29.0 -strict -summary manifests/

# Validate Helm-rendered output
helm template my-release ./chart -f values.yaml | kubeconform -kubernetes-version 1.29.0 -strict

# Check which API versions the current cluster supports
kubectl api-versions | sort

# Check if a specific API version exists
kubectl api-versions | grep networking.k8s.io

# Server-side dry-run to validate against live cluster schema
kubectl apply -f manifest.yaml --dry-run=server --validate=true

# List resources with deprecated API annotations (if using migration tools)
kubectl get all -A -o json | jq -r '.items[] | select(.apiVersion | test("beta")) | .apiVersion + " " + .kind + " " + .metadata.namespace + "/" + .metadata.name'

# Validate Kustomize output
kustomize build overlays/production | kubeconform -kubernetes-version 1.29.0 -strict
```
