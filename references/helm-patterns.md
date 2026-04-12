# Helm Chart Patterns

> When creating or reviewing Helm charts, templating Kubernetes manifests, or
> managing chart dependencies, follow these patterns. Default security posture
> is PSS "restricted" profile.

---

## Chart.yaml Required Fields

Every chart must include these top-level fields:

```yaml
apiVersion: v2
name: my-app
version: 0.1.0          # Chart version - bump on every change
appVersion: "1.0.0"     # Application version being deployed
type: application        # "application" or "library"
description: "Short description of what this chart deploys"
```

- `apiVersion: v2` is mandatory for Helm 3.
- `version` follows SemVer and must change on every chart modification.
- `appVersion` tracks the application release independently of the chart.

## values.yaml Structure

Group by resource type, use clear defaults, document every section:

```yaml
# -- Number of replicas
replicaCount: 1

image:
  # -- Container image repository
  repository: ghcr.io/org/app
  # -- Image tag (defaults to chart appVersion)
  tag: ""
  pullPolicy: IfNotPresent

securityContext:
  runAsNonRoot: true
  runAsUser: 65534
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi

probes:
  liveness:
    path: /healthz
    port: http
    initialDelaySeconds: 10
  readiness:
    path: /readyz
    port: http
    initialDelaySeconds: 5

ingress:
  enabled: false
  className: ""
  hosts: []
  tls: []

serviceAccount:
  create: true
  name: ""
  annotations: {}
```

## Template Best Practices

- Use `include` and `_helpers.tpl` for all reusable snippets.
- Never inline label sets; always call a named template.
- Use `{{- ... -}}` whitespace trimming to avoid blank lines in output.
- Always wrap string values with `{{ .Values.foo | quote }}`.

### Required Template Helpers (_helpers.tpl)

```yaml
{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "mychart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mychart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Conditional Resources

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  ...
{{- end }}
```

## Deployment Template Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "mychart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.securityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: {{ .Values.probes.liveness.port }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: {{ .Values.probes.readiness.port }}
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## Dependency Management

- Declare sub-charts in `Chart.yaml` under `dependencies`.
- Run `helm dependency update` to generate `Chart.lock`.
- Use `condition` or `tags` to make sub-charts optional.
- Commit both `Chart.yaml` and `Chart.lock` to version control.

## Testing

Run these in order during development and CI:

1. `helm lint ./chart` -- catch syntax and structural errors.
2. `helm template release-name ./chart -f values-prod.yaml` -- render manifests locally.
3. `kubeconform -kubernetes-version 1.29.0 -strict` on rendered output -- validate against schemas.
4. `helm test release-name` (post-install) -- run in-cluster test pods.

## LLM Mistake Checklist

1. **Missing `{{-` whitespace control** -- produces blank lines that break multi-document YAML.
2. **Forgot `| nindent N`** -- YAML indentation wrong in rendered output, causes parse failures.
3. **Used `{{ .Values.foo }}` without `quote`** -- numeric or special-char values break YAML.
4. **Hardcoded labels instead of `include`** -- selector/label mismatch on override.
5. **No default for `.Values.image.tag`** -- empty tag produces `repository:` with trailing colon.
6. **`toYaml` without `nindent`** -- nested objects render at column 0.
7. **Chart version not bumped** -- Helm repo serves stale version from cache.
8. **Missing `required` for mandatory values** -- chart installs with nil values, pods crash at runtime.
