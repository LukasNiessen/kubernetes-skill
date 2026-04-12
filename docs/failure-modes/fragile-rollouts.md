# FM5: Fragile Rollouts

A bad rollout is worse than no rollout. Misconfigured probes, mutable image tags, and missing graceful shutdown logic turn routine deployments into outages. Fragile rollouts are the failure mode most likely to cause user-facing downtime because they activate during the exact moment the system is changing.

## The Three Probe Types

Kubernetes provides three probes, each with a distinct purpose. Confusing them is the leading cause of cascading failures:

- **Liveness probe:** "Is the process alive?" If it fails, the kubelet kills and restarts the container. This probe must check only the process itself -- never external dependencies.
- **Readiness probe:** "Can this pod serve traffic?" If it fails, the pod is removed from Service endpoints. This is where dependency checks belong -- if the database is down, the pod should stop receiving requests but should not be killed.
- **Startup probe:** "Has initialization finished?" Disables liveness and readiness checks until it succeeds. Required for applications with slow startup (JVM warmup, ML model loading, large cache priming).

## Cascading Failure From Liveness Probes

The single most dangerous rollout misconfiguration is a liveness probe that checks an external dependency. When the database goes down:

1. The liveness probe fails on all pods simultaneously.
2. The kubelet restarts all pods.
3. Pods restart, database is still down, liveness fails again.
4. The entire service enters `CrashLoopBackOff` with exponential backoff.
5. When the database recovers, the service takes minutes to recover because of the backoff timer.

If the liveness probe only checked "is the main thread responsive?", the pods would have stayed up and resumed serving immediately when the database returned. The readiness probe would have removed them from traffic in the meantime.

## The `:latest` Tag Trap

Using `:latest` as an image tag introduces three problems:

1. **Nondeterminism:** Different nodes may cache different image layers. After a rollout, some pods run version A and others run version B, depending on which nodes had cached layers.
2. **Impossible rollbacks:** `kubectl rollout undo` re-deploys the same `:latest` tag, which may now point to a newer (broken) image.
3. **Silent drift:** No change is detected by the Deployment controller because the tag has not changed, even though the image content has.

With `imagePullPolicy: IfNotPresent` (the default for non-`:latest` tags), nodes use cached images. With `:latest`, the default policy is `Always`, but some environments override this, creating inconsistent behavior.

The fix: always use immutable tags -- semantic versions (`v2.4.1`), git SHAs, or digests (`@sha256:...`).

## Rolling Update Strategy

The `strategy.rollingUpdate` fields control how many pods are replaced simultaneously:

- **`maxSurge`**: How many extra pods above the desired count during the update. Higher values speed up rollouts but consume more resources.
- **`maxUnavailable`**: How many pods can be unavailable during the update. Set to `0` for zero-downtime deployments (requires `maxSurge >= 1`).
- **`minReadySeconds`**: How long a new pod must be Ready before it counts as Available. Catches pods that start successfully but crash shortly after (e.g., failing to connect to a dependency after initialization).

For critical services, use `maxSurge: 1, maxUnavailable: 0`. This ensures capacity never drops below the desired count during a rollout.

## Graceful Shutdown

When Kubernetes terminates a pod, two things happen in parallel:
1. The pod is removed from Service endpoints (asynchronous).
2. The container receives `SIGTERM`.

Because endpoint removal is asynchronous, the pod may still receive traffic for several seconds after `SIGTERM`. Without a `preStop` hook, the application begins shutting down while requests are still arriving, causing dropped connections and 502 errors.

The fix is a `preStop` sleep of 3-5 seconds to allow endpoint propagation before the application begins its shutdown sequence:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

Set `terminationGracePeriodSeconds` to a value that exceeds the preStop sleep plus the application's drain time. The default of 30 seconds is often insufficient for applications with long-lived connections.

## Init Containers for Dependency Waiting

Dependencies should be waited on in init containers, not liveness probes. An init container blocks pod startup until the dependency is available, then exits. This keeps the probe system focused on runtime health, not startup prerequisites:

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ["sh", "-c", "until nc -z postgres 5432; do sleep 2; done"]
```

## What LLMs Get Wrong

1. **Liveness probe checking database connectivity.** The number one cause of cascading outages. Liveness should check only process health.
2. **Same endpoint for liveness and readiness.** These probes have different purposes and should hit different endpoints (`/healthz` for liveness, `/ready` for readiness).
3. **No startup probe for slow applications.** JVM apps, Python ML services, and applications loading large datasets need 60-120+ seconds to start. Without a startup probe, the liveness probe kills them during initialization.
4. **`failureThreshold: 1` on liveness.** A single blip (GC pause, network hiccup) kills the pod. Use at least 3.
5. **`:latest` tag with no registry prefix.** `image: myapp:latest` with no registry means the kubelet looks in the default registry, which varies by runtime configuration.
6. **Missing `preStop` hook.** Traffic arrives after SIGTERM, causing dropped connections.
7. **`maxUnavailable` too high.** With 3 replicas and `maxUnavailable: 2`, only 1 pod serves traffic during rollout -- a single failure causes a complete outage.

## Real-World Impact

- **Cloudflare outage (2019):** A misconfigured health check caused a cascading restart of edge proxies across multiple data centers, resulting in a global 30-minute outage.
- **GitLab incident (2021):** A canary deployment with no readiness probe sent traffic to pods still loading their configuration, causing elevated error rates for 45 minutes.
- **Shopify Black Friday (2020):** Aggressive liveness probes combined with database latency caused pod restarts during peak traffic, requiring manual intervention to stabilize.

Rollout fragility is entirely preventable. Every field -- probes, strategy, shutdown hooks, image tags -- has a correct configuration that eliminates the corresponding failure mode.

## Further Reading

- [Configure Liveness, Readiness, and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [KubeShark Good Patterns](../examples/good-patterns.md)
- [KubeShark Bad Patterns](../examples/bad-patterns.md)
