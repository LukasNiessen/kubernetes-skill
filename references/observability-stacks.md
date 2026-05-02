# Observability Stacks

**Load this reference when detected:** Prometheus Operator, kube-prometheus-stack, ServiceMonitor, PodMonitor, PrometheusRule, AlertmanagerConfig, OpenTelemetry Collector, OpenTelemetry Operator, Loki, Grafana, Tempo, metrics, logs, traces, dashboards, or telemetry pipelines.

## Why this matters

Observability add-ons use CRDs, label selectors, generated scrape config, and deployment modes that differ by stack. LLMs frequently generate resources that apply successfully but are never selected, never scraped, or duplicate data. Do not load this file for basic application logging unless an observability stack is involved.

## Prometheus Operator

ServiceMonitor and PodMonitor behavior depends on selectors.

- `ServiceMonitor` selects Services, not Deployments.
- The Service must expose a named port, and the ServiceMonitor endpoint should reference that port name.
- `PodMonitor` selects Pods directly and should be used only when a Service is unnecessary or unavailable.
- Prometheus or PrometheusAgent selects monitors through label and namespace selectors; labels must match both sides.
- `PrometheusRule` must be selected by the relevant Prometheus rule selectors.
- Do not create ServiceMonitor/PodMonitor resources unless the CRDs are installed.

## OpenTelemetry Collector

Choose collector mode by signal source.

- `DaemonSet`: node-local logs, host metrics, kubelet metrics, or per-node collection.
- `Deployment`: centralized OTLP gateway, cluster events, or singleton receivers.
- `StatefulSet`: stable identity or persistent queue/storage requirements.
- Avoid duplicate cluster-wide receivers across multiple replicas unless the receiver supports it.
- Set memory limits and memory limiter processor together.
- Bind receivers as narrowly as practical and expose OTLP only inside the cluster unless explicitly required.

## Loki and Logs

- Choose Loki deployment mode by scale: monolithic for small stacks, scalable or microservices for production/high volume.
- Configure durable object storage for production Loki; do not rely on ephemeral storage.
- Keep log labels low-cardinality. Do not label on request IDs, user IDs, pod UIDs, or raw paths.
- Prefer structured JSON logs to stdout/stderr from applications.
- Separate log collection agents from application pods unless the sidecar is explicitly required.

## Grafana and Dashboards

- Treat dashboards and datasources as configuration owned by the observability platform.
- Avoid embedding secrets in dashboard ConfigMaps or Helm values.
- When using sidecar dashboard discovery, ensure labels match the sidecar selector.
- Keep dashboard ConfigMaps namespace and RBAC aligned with the deployed Grafana chart.

## Alerting

- Alerts should be actionable, routed, and include runbook context.
- Use `for:` durations to reduce flapping.
- Avoid high-cardinality alert labels.
- Separate symptom alerts from cause alerts; do not page on every transient pod restart.
- Validate PromQL against the actual metric names emitted by the stack.

## Validation

- `kubectl get crd | grep -Ei "servicemonitors|podmonitors|prometheusrules"` (or `findstr /i` on Windows)
- `kubectl get servicemonitor,podmonitor,prometheusrule -A`
- Inspect Prometheus target discovery for selected monitors.
- Check generated Prometheus config or operator logs when targets are missing.
- `helm template` observability charts and validate rendered CRDs/resources before applying.
- For OpenTelemetry, inspect Collector logs for invalid pipeline components and dropped data.

## LLM Mistake Checklist

- Creating ServiceMonitor selectors that match Deployment labels but not Service labels.
- Referencing a numeric port when the ServiceMonitor expects a named Service port.
- Forgetting that Prometheus selectors must select the monitor.
- Creating monitoring CRDs when the Prometheus Operator is not installed.
- Running cluster-wide OpenTelemetry receivers in multiple replicas and duplicating data.
- Choosing Loki monolithic mode for high-volume production without durable storage.
- Creating high-cardinality Loki labels or alert labels.
- Shipping dashboards with plaintext datasource credentials.

## Grounding Sources

- Prometheus Operator design: https://prometheus-operator.dev/docs/getting-started/design/
- Prometheus Operator ServiceMonitor and PodMonitor getting started: https://prometheus-operator.dev/docs/developer/getting-started/
- Prometheus Operator troubleshooting: https://prometheus-operator.dev/docs/platform/troubleshooting/
- OpenTelemetry Collector Helm chart: https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/
- Loki Helm installation: https://grafana.com/docs/loki/latest/setup/install/helm/
