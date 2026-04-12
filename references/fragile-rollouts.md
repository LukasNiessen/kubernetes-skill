# Fragile Rollouts

**Directive:** When generating Deployments, StatefulSets, or any workload with rolling updates, ALWAYS configure probes correctly, use immutable image tags, and design for graceful shutdown. A bad rollout is worse than no rollout. Default security posture is PSS "restricted" profile.

## When to use

Consult this reference whenever the task involves:
- Creating or modifying Deployments, StatefulSets, or DaemonSets
- Configuring liveness, readiness, or startup probes
- Setting image tags or pull policies
- Defining rolling update strategy parameters
- Implementing graceful shutdown or preStop hooks
- Adding init containers for dependency readiness

## Symptoms of fragile rollouts

| Symptom | Likely cause |
|---|---|
| Cascading restarts across all pods simultaneously | Liveness probe checks an external dependency (DB, cache) that went down |
| Dropped connections / 502s during deploy | No readiness probe, or readiness probe passes before app is truly ready |
| All replicas unavailable at the same time | `maxUnavailable` too high or `minReadySeconds` not set |
| Version drift -- some pods on old image, some on new | `:latest` tag with `imagePullPolicy: IfNotPresent` and cached layers |
| Pods killed before finishing in-flight requests | No preStop hook, `terminationGracePeriodSeconds` too short |
| Slow-starting apps killed in a restart loop | No startup probe, liveness probe fires before app initializes |

## Root causes

1. Misunderstanding the difference between liveness and readiness probes.
2. Checking external dependencies (databases, APIs) in liveness probes.
3. Using `:latest` tags, which are mutable and nondeterministic.
4. Not accounting for graceful shutdown and connection draining.
5. Setting probe timings without understanding the application startup profile.

## Prevention rules

### Probe types -- what each does

- **Liveness probe:** "Is the process alive and not deadlocked?" If it fails, kubelet kills and restarts the container. NEVER check external dependencies here. A simple `/healthz` that returns 200 if the event loop or main thread is responsive.
- **Readiness probe:** "Can the pod serve traffic right now?" If it fails, the pod is removed from Service endpoints. This IS the place to check dependencies -- if the database is down, the pod should stop receiving requests but should NOT be killed.
- **Startup probe:** "Has the application finished initializing?" Used for slow-starting apps (JVM warmup, Python loading ML models). While the startup probe is running, liveness and readiness probes are disabled. Once the startup probe succeeds, the other probes take over.

### Critical rule: NEVER check external dependencies in liveness probes

This is the single most common cause of cascading outages. When the database goes down:
- Liveness probe fails on all pods simultaneously
- Kubelet restarts all pods
- Pods come back, database is still down, liveness fails again
- Entire service enters a crash loop while the database recovers
- If the liveness probe only checked "is the process alive?", the pods would have stayed up and recovered when the database returned.

### Probe timing guidelines

```
startupProbe:
  failureThreshold x periodSeconds >= maximum startup time
  Example: JVM app that takes up to 120s to start
           failureThreshold: 30, periodSeconds: 5 = 150s budget

livenessProbe:
  initialDelaySeconds: only if NOT using a startup probe
  periodSeconds: 10-30s (don't hammer the app)
  timeoutSeconds: must be less than periodSeconds
  failureThreshold: 3 (don't kill on a single blip)

readinessProbe:
  periodSeconds: 5-10s (faster than liveness, controls traffic)
  failureThreshold: 1-3 (remove from traffic quickly)
```

### Rolling update strategy

- `maxSurge`: how many extra pods above `replicas` during update. Higher = faster rollout, more resource usage.
- `maxUnavailable`: how many pods can be down during update. Set to 0 for zero-downtime deploys (requires `maxSurge >= 1`).
- `minReadySeconds`: how long a new pod must be Ready before it counts as Available. Catches pods that start and crash shortly after.
- For critical services: `maxSurge: 1, maxUnavailable: 0` ensures capacity never drops.

### Image tagging

- NEVER use `:latest`. It is mutable, nondeterministic, and makes rollbacks impossible.
- Use immutable tags: semantic versions (`v2.4.1`), git SHAs (`abc123def`), or digests (`@sha256:...`).
- `imagePullPolicy: IfNotPresent` is correct for immutable tags. Use `Always` only with mutable tags (which you should not be using).
- `imagePullPolicy: Never` only for local development with pre-loaded images.

### Graceful shutdown sequence

When Kubernetes terminates a pod, the following happens in parallel:
1. Pod is marked `Terminating` and removed from Service endpoints (async).
2. `preStop` hook runs (if defined).
3. `SIGTERM` is sent to PID 1 in the container.
4. Kubelet waits up to `terminationGracePeriodSeconds` (default 30s).
5. `SIGKILL` is sent if the process has not exited.

The problem: step 1 is async. The pod may still receive traffic for a few seconds after SIGTERM. The fix: add a `preStop` sleep to allow endpoint propagation before the app begins shutdown.

### Init containers for dependency waiting

Use init containers to wait for dependencies, NOT liveness probes. Init containers run before the main container starts and block until they succeed.

## Patterns and examples

### GOOD: Deployment with proper probes, rolling update, graceful shutdown

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: payments
spec:
  replicas: 4
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
        version: v3.2.0
    spec:
      serviceAccountName: payment-api
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 60
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      initContainers:
        - name: wait-for-db
          image: registry.example.com/toolbox:v1.0.0
          command: ["sh", "-c"]
          args:
            - |
              until pg_isready -h postgres.payments.svc -p 5432; do
                echo "Waiting for database..."
                sleep 2
              done
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true
      containers:
        - name: api
          image: registry.example.com/payment-api:v3.2.0
          ports:
            - containerPort: 8080
              protocol: TCP
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 5
            failureThreshold: 30          # 150s budget for JVM startup
          livenessProbe:
            httpGet:
              path: /healthz              # checks ONLY process health
              port: 8080
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready                # checks process + downstream deps
              port: 8080
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]   # allow endpoint de-registration
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              memory: 512Mi
```

### BAD: Liveness probe checking database, :latest tag, no graceful shutdown

```yaml
# DO NOT DO THIS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: payments
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1             # with 2 replicas, this means 50% down during deploy
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      # no terminationGracePeriodSeconds -- defaults to 30s, may not be enough
      containers:
        - name: api
          image: registry.example.com/payment-api:latest    # mutable tag
          imagePullPolicy: IfNotPresent                      # may use stale cached layer
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 1         # killed on a single failure
            # THIS ENDPOINT CHECKS DATABASE CONNECTIVITY
            # When the DB goes down, ALL pods restart simultaneously
          # no readiness probe -- traffic hits pods before they are ready
          # no startup probe -- slow starts trigger liveness kills
          # no preStop hook -- in-flight requests dropped on termination
          # no resource requests/limits
```

### Pod Disruption Budget for high-availability services

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
  namespace: payments
spec:
  minAvailable: 2                    # or use maxUnavailable: 1
  selector:
    matchLabels:
      app: payment-api
```

## LLM mistake checklist

Before finalizing any Deployment or workload manifest, verify each item:

- [ ] Liveness probe does NOT check any external dependency (database, cache, queue, other service)
- [ ] Readiness probe is defined and separate from liveness probe
- [ ] Startup probe is defined for applications with initialization time > 10 seconds
- [ ] Image tag is immutable (semantic version, git SHA, or digest) -- not `:latest`
- [ ] `imagePullPolicy` is `IfNotPresent` for immutable tags, not `Always`
- [ ] `maxUnavailable: 0` is set if zero-downtime deployment is required
- [ ] `terminationGracePeriodSeconds` exceeds the time the application needs to drain connections
- [ ] `preStop` hook includes a short sleep (3-5s) to allow endpoint de-registration
- [ ] `failureThreshold` for liveness probe is at least 3, not 1
- [ ] Resource `requests` are set (required for scheduling); memory `limits` are set
- [ ] Init containers handle dependency waiting, not liveness probes
- [ ] `minReadySeconds` is set to catch crash-after-start scenarios

## Verification commands

```bash
# Check rollout status
kubectl rollout status deployment/payment-api -n payments

# Watch pods during a rollout
kubectl get pods -n payments -l app=payment-api -w

# Check rollout history and revision details
kubectl rollout history deployment/payment-api -n payments
kubectl rollout history deployment/payment-api -n payments --revision=3

# Rollback to previous revision
kubectl rollout undo deployment/payment-api -n payments

# Verify probe configuration on running pods
kubectl get pods -n payments -l app=payment-api -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  liveness: {.livenessProbe.httpGet.path}{"\n"}  readiness: {.readinessProbe.httpGet.path}{"\n"}  startup: {.startupProbe.httpGet.path}{"\n"}{end}{end}'

# Check for deployments using :latest tag
kubectl get deployments -A -o json | \
  jq -r '.items[] | .metadata.namespace + "/" + .metadata.name as $d | .spec.template.spec.containers[] | select(.image | endswith(":latest") or (contains(":") | not)) | $d + " -> " + .image'

# Check for pods without readiness probes
kubectl get pods -A -o json | \
  jq -r '.items[] | .metadata.namespace + "/" + .metadata.name as $pod | .spec.containers[] | select(.readinessProbe == null) | $pod + " container:" + .name + " has no readiness probe"'

# Verify PodDisruptionBudget coverage
kubectl get pdb -n payments -o wide

# Check events for probe failures
kubectl get events -n payments --field-selector reason=Unhealthy --sort-by='.lastTimestamp'

# Inspect endpoint changes during rollout
kubectl get endpoints payment-api -n payments -w
```
