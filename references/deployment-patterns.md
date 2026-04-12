# Deployment Patterns -- Stateless Workloads

**Load this reference when generating:** Deployment, Service, HPA, PDB, Ingress, or any stateless application manifest.

## When to Use a Deployment
Any workload that is stateless: web apps, REST/gRPC APIs, microservices, frontend proxies, queue-consuming workers. If pods are interchangeable and need no stable identity or persistent local storage, use a Deployment.

## Minimum Production Checklist
1. `replicas` >= 2 -- never ship a single replica to production.
2. `resources.requests` AND `resources.limits` on every container (cpu + memory).
3. Pod-level `securityContext` satisfying PSS **restricted** profile.
4. `readinessProbe` (gates traffic) and `livenessProbe` (restarts stuck pods) on separate endpoints.
5. `topologySpreadConstraints` or pod anti-affinity across failure domains.
6. An accompanying `PodDisruptionBudget`.

## Label Strategy
```yaml
labels:
  app.kubernetes.io/name: order-service      # app identity -- use in selectors
  app.kubernetes.io/version: "1.4.2"         # NEVER put in selector.matchLabels
  app.kubernetes.io/component: api           # role: api | worker | cache
  app.kubernetes.io/part-of: ecommerce       # higher-level system
  app.kubernetes.io/managed-by: helm         # tooling
```

## Service Wiring
Default: **ClusterIP + Ingress**. ClusterIP for in-cluster traffic; Ingress terminates TLS and routes externally. Avoid LoadBalancer Services unless no Ingress controller exists or the workload needs raw TCP/UDP.

## Config Mounting
- **Prefer volume mounts** for file-based config -- enables atomic updates on ConfigMap rotation.
- Use `env`/`envFrom` only for simple key-value pairs.
- Set `immutable: true` on Secrets that should never change in place.

## Environment-Specific Configuration
- **Kustomize overlays**: `base/` + `overlays/{dev,staging,prod}/` for per-env patching (replicas, resources, images).
- **Helm values**: `values-prod.yaml` per environment when conditionals or loops are needed.

## Example: Production Deployment + Service + HPA
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels: &labels
    app.kubernetes.io/name: order-service
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: ecommerce
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels: { app.kubernetes.io/name: order-service, app.kubernetes.io/component: api }
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  template:
    metadata:
      labels: { <<: *labels, app.kubernetes.io/version: "1.4.2" }
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10000
        runAsGroup: 10000
        fsGroup: 10000
        seccompProfile: { type: RuntimeDefault }
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: { app.kubernetes.io/name: order-service }
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.4.2
          ports: [{ containerPort: 8080, protocol: TCP }]
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { cpu: "1", memory: 512Mi }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          readinessProbe:
            httpGet: { path: /healthz/ready, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz/live, port: 8080 }
            initialDelaySeconds: 15
            periodSeconds: 20
          volumeMounts:
            - { name: config, mountPath: /etc/order-service, readOnly: true }
            - { name: tmp, mountPath: /tmp }
      volumes:
        - { name: config, configMap: { name: order-service-config } }
        - { name: tmp, emptyDir: {} }
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  selector: { app.kubernetes.io/name: order-service, app.kubernetes.io/component: api }
  ports: [{ port: 80, targetPort: 8080, protocol: TCP }]
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: order-service }
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
    - type: Pods
      pods:
        metric: { name: http_requests_per_second }
        target: { type: AverageValue, averageValue: "1000" }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies: [{ type: Percent, value: 25, periodSeconds: 60 }]
    scaleUp:
      stabilizationWindowSeconds: 30
      policies: [{ type: Percent, value: 50, periodSeconds: 60 }]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service
spec:
  minAvailable: 2
  selector:
    matchLabels: { app.kubernetes.io/name: order-service, app.kubernetes.io/component: api }
```

## LLM Mistake Checklist
1. **Version label in selector.** Never put `app.kubernetes.io/version` in `selector.matchLabels` -- selectors are immutable; this breaks upgrades.
2. **Missing readOnlyRootFilesystem.** PSS restricted requires it. Mount an `emptyDir` at `/tmp` if the app writes temp files.
3. **Omitting resource limits.** Both `requests` and `limits` are required. Without them the pod is BestEffort QoS and evicted first.
4. **Single replica in production.** Always `replicas >= 2` with a PDB. One replica = zero availability during node drains.
5. **HPA without scaleDown stabilization.** Default scale-down is aggressive. Set `stabilizationWindowSeconds: 300` to prevent thrashing.
6. **Probes hitting the main API path.** Use dedicated `/healthz/*` endpoints to avoid cascading failures under load.
7. **Forgetting /tmp emptyDir.** With `readOnlyRootFilesystem: true`, processes writing to `/tmp` crash without this volume.
8. **LoadBalancer Service by default.** Each provisions a cloud LB -- use ClusterIP + Ingress instead.
