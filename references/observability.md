# Observability

**Directive:** When generating or reviewing any production workload, ALWAYS include metrics exposure, structured logging, and health probes. Observability is not optional -- if you cannot measure it, you cannot operate it. Default security posture is PSS "restricted" profile.

## When to use

Consult this reference whenever the task involves:
- Deploying any workload to a production or staging cluster
- Setting up monitoring, alerting, or dashboards
- Investigating incidents or performing post-mortems
- Capacity planning or performance analysis
- Configuring log aggregation or distributed tracing

---

## Probes as the Foundation

Liveness, readiness, and startup probes are the most basic form of observability -- they tell Kubernetes whether your application is alive, ready, and initialized. See **fragile-rollouts.md** for detailed probe configuration rules. Without correct probes, no amount of metrics or logging will prevent cascading failures.

---

## Prometheus Metrics Exposure

### Annotations pattern (works without prometheus-operator)

Add annotations to the Pod template so Prometheus discovers and scrapes the target:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10000
        runAsGroup: 10000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/order-service:v1.8.3
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090
          securityContext:
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
```

### ServiceMonitor (prometheus-operator)

When using prometheus-operator, prefer ServiceMonitor CRDs over annotations for type-safe configuration:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: orders
  labels:
    release: kube-prometheus-stack    # must match Prometheus operator selector
spec:
  selector:
    matchLabels:
      app: order-service
  endpoints:
    - port: metrics                   # must match Service port name
      interval: 30s
      path: /metrics
```

PodMonitor follows the same pattern but targets pods directly (useful when no Service exists, e.g., CronJobs with metrics).

---

## Key Metrics -- the RED Method

Every service should expose at minimum:

| Signal | Metric | Example |
|---|---|---|
| **R**ate | Request throughput | `http_requests_total` (counter) |
| **E**rrors | Failed request count | `http_requests_total{status=~"5.."}` or a dedicated error counter |
| **D**uration | Request latency | `http_request_duration_seconds` (histogram with buckets) |

For resource-oriented services (queues, databases), add **saturation** metrics: queue depth, connection pool usage, disk I/O utilization.

Use histogram buckets aligned to your SLOs:

```
http_request_duration_seconds_bucket{le="0.05"}   # 50ms - fast API
http_request_duration_seconds_bucket{le="0.1"}
http_request_duration_seconds_bucket{le="0.25"}
http_request_duration_seconds_bucket{le="0.5"}
http_request_duration_seconds_bucket{le="1.0"}
http_request_duration_seconds_bucket{le="2.5"}
http_request_duration_seconds_bucket{le="5.0"}
http_request_duration_seconds_bucket{le="10.0"}
http_request_duration_seconds_bucket{le="+Inf"}
```

---

## Logging

### Structured JSON to stdout

Applications MUST log structured JSON to stdout/stderr. Never log to files inside the container -- it defeats node-level collection and fills the writable layer.

```json
{"timestamp":"2025-03-15T10:23:45Z","level":"error","msg":"payment failed","trace_id":"abc123","order_id":"ord-789","error":"timeout after 5s"}
```

Rules:
- Use `timestamp`, `level`, `msg` as standard fields.
- Include `trace_id` and `span_id` for correlation with distributed traces.
- Never log secrets, tokens, PII, or full request bodies.
- Use `stderr` for error-level logs and `stdout` for everything else (some collectors distinguish).

### Log aggregation -- DaemonSet pattern

Fluent Bit runs as a DaemonSet on every node, reads container logs from `/var/log/containers/`, and forwards to a sink:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.0
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              memory: 128Mi
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
```

Node-level collection (DaemonSet) is preferred over sidecar collection for most workloads. Use sidecars only when you need per-pod log transformation or the application cannot log to stdout.

---

## Distributed Tracing -- OpenTelemetry

### Auto-instrumentation with the OTel Operator

The OpenTelemetry Operator can inject instrumentation sidecars via annotation:

```yaml
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-java: "true"    # or inject-python, inject-nodejs
```

### OTel Collector sidecar pattern

For fine-grained control, run the OTel Collector as a sidecar:

```yaml
- name: otel-collector
  image: otel/opentelemetry-collector-contrib:0.98.0
  args: ["--config=/etc/otel/config.yaml"]
  ports:
    - containerPort: 4317       # gRPC OTLP receiver
    - containerPort: 4318       # HTTP OTLP receiver
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop: ["ALL"]
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      memory: 128Mi
  volumeMounts:
    - name: otel-config
      mountPath: /etc/otel
```

Propagate trace context (`traceparent` header / W3C Trace Context) across all service boundaries. Without propagation, traces are fragmented and useless.

---

## Alerting -- PrometheusRule

Write symptom-based alerts (what the user experiences), not cause-based alerts (what broke internally):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
  namespace: orders
spec:
  groups:
    - name: order-service.rules
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{job="order-service",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{job="order-service"}[5m])) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Order service error rate above 5%"
            runbook_url: "https://wiki.example.com/runbooks/order-service-errors"
        - alert: HighLatencyP99
          expr: |
            histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le)) > 2.0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Order service p99 latency above 2s"
```

Every alert MUST have a `runbook_url` annotation pointing to actionable remediation steps.

---

## Deployment Annotations for Grafana

Annotate deployments in Grafana to correlate metric changes with releases:

```bash
curl -s -X POST http://grafana.monitoring.svc:3000/api/annotations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d "{\"text\":\"Deployed order-service v1.8.3\",\"tags\":[\"deployment\",\"orders\"]}"
```

Integrate this into your CI/CD pipeline as a post-deploy step.

---

## LLM Mistake Checklist

Before finalizing any workload manifest or observability configuration, verify each item:

- [ ] **Prometheus annotations** are on the Pod template `metadata.annotations`, not on the Deployment metadata.
- [ ] **Metrics port** is declared in the container `ports` list and matches the annotation value.
- [ ] **Logs are structured JSON to stdout** -- no file-based logging, no unstructured text.
- [ ] **Trace context propagation** is configured -- auto-instrumentation annotation or SDK integration present.
- [ ] **Alerts are symptom-based** (error rate, latency) not cause-based (pod restarted, CPU high).
- [ ] **Every alert has a `runbook_url`** annotation -- alerts without runbooks are noise.
- [ ] **Histogram buckets** are aligned to SLO thresholds, not arbitrary defaults.
- [ ] **Resource requests and limits** are set on all sidecar containers (OTel Collector, Fluent Bit) to prevent them from starving the main workload.
