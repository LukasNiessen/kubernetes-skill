# DaemonSet and Operator Patterns -- Node-Level and Custom Controllers

**Load this reference when generating:** DaemonSet, PriorityClass, CRDs, or any workload that must run on every (or a targeted subset of) node(s).

## When to Use a DaemonSet
Exactly one pod per qualifying node: log collectors (Fluent Bit, Vector), monitoring agents (node-exporter, Datadog), network plugins (CNI, kube-proxy, Cilium), CSI node drivers, security agents (Falco). If you need multiple replicas per node or the workload is not node-scoped, use a Deployment.

## Update Strategies
| Strategy | Behavior | Use when |
|---|---|---|
| `RollingUpdate` | Replaces pods node-by-node; `maxUnavailable` controls pace | Normal updates |
| `OnDelete` | Pods replaced only when manually deleted | Critical infra (CNI, kube-proxy) needing manual control |

For large clusters, set `maxUnavailable` to a percentage (e.g., `"10%"`) to speed rollouts.

## Node Selectors and Tolerations
**Targeting**: use `nodeSelector` for simple matching, `nodeAffinity` for richer expressions:
```yaml
nodeSelector:
  node.kubernetes.io/os: linux
  kubernetes.io/arch: amd64
```

**Tolerations**: DaemonSets often must run on tainted nodes (control-plane, GPU pools). Add only the tolerations you need:
```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
  - key: node.kubernetes.io/not-ready
    operator: Exists
    effect: NoExecute
```
Never use `operator: Exists` without a `key` (tolerates everything) unless the DaemonSet truly belongs on every node.

## Resource Management
DaemonSet pods run on **every node**. 200m CPU x 100 nodes = 20 cores cluster-wide. Be conservative:
- `requests` = steady-state consumption. `limits` = burst cap.
- Monitor actual usage and right-size iteratively.

## Priority Classes
Prevent preemption of system DaemonSets with a custom PriorityClass:
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-node-agent
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Node-level DaemonSet agents (logging, monitoring)."
```
Built-in `system-cluster-critical` and `system-node-critical` are reserved for core components. Use a custom class in the 100000-10000000 range.

## Operator Pattern Overview
An operator is a custom controller that watches CRs and reconciles cluster state. Use when:
- Complex operational logic (failover, backup, scaling) exceeds built-in controllers.
- Users need a simple declarative API for a complex system (database, queue).
- Manual runbooks are error-prone and should be codified.

Do NOT build an operator when Helm, Kustomize, or a Job suffices. Operators carry significant maintenance burden.

## CRD Basics
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.db.example.com
spec:
  group: db.example.com
  scope: Namespaced
  names: { plural: postgresclusters, singular: postgrescluster, kind: PostgresCluster, shortNames: ["pgc"] }
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["replicas", "version"]
              properties:
                replicas: { type: integer, minimum: 1, maximum: 10 }
                version:  { type: string, enum: ["15", "16"] }
                storage:
                  type: object
                  properties:
                    size: { type: string, pattern: "^[0-9]+Gi$" }
```
Always include `openAPIV3Schema` with validation. CRDs without it accept arbitrary YAML, causing runtime errors.

## Operator Frameworks
- **kubebuilder**: upstream Go framework. Generates scaffolding, RBAC, CRD manifests, webhooks. Preferred for Go teams.
- **operator-sdk**: extends kubebuilder; adds Ansible and Helm operator support for non-Go teams.

Both produce the same runtime pattern: a manager running reconciliation loops.

## Example: Log Collector DaemonSet
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  labels: { app.kubernetes.io/name: fluent-bit, app.kubernetes.io/component: log-collector }
spec:
  selector:
    matchLabels: { app.kubernetes.io/name: fluent-bit, app.kubernetes.io/component: log-collector }
  updateStrategy: { type: RollingUpdate, rollingUpdate: { maxUnavailable: "10%" } }
  template:
    metadata:
      labels: { app.kubernetes.io/name: fluent-bit, app.kubernetes.io/component: log-collector }
    spec:
      priorityClassName: system-node-agent
      serviceAccountName: fluent-bit
      nodeSelector: { node.kubernetes.io/os: linux }
      tolerations:
        - { key: node-role.kubernetes.io/control-plane, operator: Exists, effect: NoSchedule }
        - { key: node.kubernetes.io/not-ready, operator: Exists, effect: NoExecute, tolerationSeconds: 60 }
      securityContext:
        runAsNonRoot: true
        runAsUser: 10000
        runAsGroup: 10000
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.0.4
          ports: [{ containerPort: 2020, name: metrics, protocol: TCP }]
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits:   { cpu: 200m, memory: 128Mi }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: varlog, mountPath: /var/log, readOnly: true }
            - { name: config, mountPath: /fluent-bit/etc, readOnly: true }
            - { name: buffer, mountPath: /fluent-bit/buffer }
      volumes:
        - { name: varlog, hostPath: { path: /var/log, type: Directory } }
        - { name: config, configMap: { name: fluent-bit-config } }
        - { name: buffer, emptyDir: { sizeLimit: 256Mi } }
```

## LLM Mistake Checklist
1. **DaemonSet with `replicas` field.** DaemonSets have no `replicas`. The scheduler places one pod per qualifying node. Including it is an API error.
2. **Missing tolerations for tainted nodes.** Without explicit tolerations, pods stay Pending on control-plane or special-purpose nodes.
3. **Overly generous resource requests.** Multiplied across every node, small over-requests waste enormous capacity. Keep requests minimal.
4. **Using system-node-critical without justification.** Reserved for core components. Use a custom PriorityClass for application agents.
5. **hostPath without `type`.** Always set `hostPath.type` (Directory, Socket, File) to catch mount errors at startup, not runtime.
6. **CRD without openAPIV3Schema.** No validation = any YAML accepted = inscrutable controller errors. Always define a strict schema.
7. **Blanket toleration (no key).** `operator: Exists` with no `key` tolerates every taint including NoExecute eviction taints. Only tolerate specific, known taints.
8. **Forgetting serviceAccountName.** DaemonSets accessing host paths or the API need a dedicated ServiceAccount with minimal RBAC.
