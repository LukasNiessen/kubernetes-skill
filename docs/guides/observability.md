# Observability

Metrics, logging, tracing, and alerting for Kubernetes workloads. KubeShark treats observability as mandatory for production -- if you cannot measure it, you cannot operate it. For full configuration examples and the LLM mistake checklist, see [references/observability.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/observability.md).

## Probes as the Foundation

Liveness, readiness, and startup probes are the most basic form of observability. They tell Kubernetes whether your application is alive, ready to serve traffic, and initialized. Without correct probes, no amount of metrics or logging prevents cascading failures. See [fragile-rollouts](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/fragile-rollouts.md) for detailed probe rules.

## Prometheus Metrics

### Annotations Pattern

Add `prometheus.io/scrape: "true"`, `prometheus.io/port`, and `prometheus.io/path` annotations to the Pod template metadata (not the Deployment metadata). This enables Prometheus auto-discovery without the prometheus-operator.

### ServiceMonitor Pattern

When using prometheus-operator, prefer ServiceMonitor CRDs for type-safe configuration. The ServiceMonitor `selector.matchLabels` must match the Service labels, and the `release` label must match the Prometheus operator selector.

### RED Method

Every service should expose at minimum:

- **Rate** -- request throughput (`http_requests_total` counter)
- **Errors** -- failed request count (`http_requests_total{status=~"5.."}`)
- **Duration** -- request latency (`http_request_duration_seconds` histogram)

Align histogram buckets to your SLO thresholds, not arbitrary defaults. For resource-oriented services (queues, databases), add saturation metrics like queue depth and connection pool usage.

## Structured Logging

Applications must log structured JSON to stdout/stderr. Rules:

- Use `timestamp`, `level`, `msg` as standard fields
- Include `trace_id` and `span_id` for correlation with distributed traces
- Never log secrets, tokens, PII, or full request bodies
- Never log to files inside the container -- it defeats node-level collection and fills the writable layer

Log aggregation uses a DaemonSet pattern (Fluent Bit on every node reading `/var/log/containers/`). Use sidecars only when per-pod log transformation is required.

## OpenTelemetry Tracing

### Auto-Instrumentation

The OpenTelemetry Operator injects instrumentation via pod annotations (e.g., `instrumentation.opentelemetry.io/inject-java: "true"`).

### Collector Sidecar

For fine-grained control, run the OTel Collector as a sidecar with gRPC (4317) and HTTP (4318) OTLP receivers. Set resource requests and limits on the sidecar to prevent it from starving the main workload.

### Context Propagation

Propagate trace context (`traceparent` header / W3C Trace Context) across all service boundaries. Without propagation, traces are fragmented and useless.

## Alerting Patterns

Write symptom-based alerts (what the user experiences), not cause-based alerts (what broke internally):

- **HighErrorRate** -- error rate above SLO threshold for a sustained period
- **HighLatencyP99** -- p99 latency above target for a sustained period

Every PrometheusRule alert must include a `runbook_url` annotation pointing to actionable remediation steps. Alerts without runbooks are noise.

Use Grafana deployment annotations to correlate metric changes with releases. Integrate annotation creation into your CI/CD pipeline as a post-deploy step.

## Common LLM Mistakes

Key observability errors LLMs produce include: placing Prometheus annotations on the Deployment metadata instead of the Pod template, not declaring the metrics port in the container ports list, generating file-based logging instead of structured JSON to stdout, omitting trace context propagation, writing cause-based alerts instead of symptom-based ones, and omitting resource limits on sidecar containers. See the full checklist in the [reference file](https://github.com/LukasNiessen/kubernetes-skill/blob/main/references/observability.md#llm-mistake-checklist).
