# Bad Examples -- Common LLM Anti-Patterns

> These are manifests that LLMs frequently generate. Each one compiles and appears
> valid but has serious issues in production. Study the annotations to understand
> what is wrong and why.

---

## 1. Deployment Running as Root with No Security Context

```yaml
# BAD -- DO NOT USE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          ports:
            - containerPort: 8080
```

**What is wrong:**
- No `securityContext` at pod or container level -- container runs as root by default.
- Missing `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`.
- Missing `capabilities.drop: ["ALL"]` -- container retains all Linux capabilities.
- No `seccompProfile` -- fails PSS restricted profile.
- No resource requests or limits -- can consume unbounded node resources.
- No probes -- Kubernetes cannot detect if the app is healthy or ready.
- No standard `app.kubernetes.io/*` labels.
- Uses `:latest` tag (see anti-pattern 5).

## 2. Service with Selector That Matches No Pods

```yaml
# BAD -- DO NOT USE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
        version: v2
    spec:
      containers:
        - name: web
          image: ghcr.io/org/web:v2.0.0
---
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
spec:
  selector:
    app: web-frontend
    version: v1            # <-- MISMATCH: pods have version: v2
  ports:
    - port: 80
      targetPort: 8080
```

**What is wrong:**
- Service selector includes `version: v1` but pods have `version: v2`.
- Kubernetes does not warn about selector mismatches -- the Service silently has zero endpoints.
- This is a frequent LLM mistake when updating version labels on the Deployment without updating the Service.
- Debug with `kubectl get endpoints web-frontend` -- it will show an empty subset.

## 3. ClusterRoleBinding with cluster-admin for a Single-Namespace App

```yaml
# BAD -- DO NOT USE
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: my-app
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-app-admin
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: my-app
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

**What is wrong:**
- `cluster-admin` grants unrestricted access to the entire cluster: every namespace, every resource, every verb.
- A single-namespace application needs only a namespace-scoped Role with specific verbs.
- If this service account token is compromised, the attacker owns the entire cluster.
- Use a namespace-scoped `Role` + `RoleBinding` with only the specific API groups, resources, and verbs needed.

## 4. Liveness Probe Checking External Database

```yaml
# BAD -- DO NOT USE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: ghcr.io/org/api:v1.0.0
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - "pg_isready -h postgres.db.svc -p 5432"
            periodSeconds: 10
            failureThreshold: 3
```

**What is wrong:**
- Liveness probe depends on an external database. If the database is briefly unavailable, Kubernetes kills all API pods.
- This causes cascading failure: database blip -> all pods restart -> thundering herd reconnects -> database overloaded further.
- Liveness probes must check only the process's own health (e.g., `/healthz` that returns 200 if the HTTP server is responsive).
- Use readiness probes (not liveness) to check dependency connectivity, so the pod is removed from Service endpoints but not killed.

## 5. Deployment with :latest Tag and No imagePullPolicy

```yaml
# BAD -- DO NOT USE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
        - name: worker
          image: org/worker:latest
```

**What is wrong:**
- `:latest` is a mutable tag. Different nodes may pull different versions, causing inconsistent behavior across replicas.
- When `imagePullPolicy` is not set and the tag is `:latest`, Kubernetes defaults to `Always`. But if the tag is anything else, it defaults to `IfNotPresent`.
- Rollbacks are impossible because every revision points to `:latest`.
- No way to audit which exact image is running.
- Use immutable tags (`v1.2.3`) or digests (`@sha256:abc...`). Set `imagePullPolicy: IfNotPresent` with immutable tags.

## 6. Ingress Using Removed API Version

```yaml
# BAD -- DO NOT USE
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: frontend
              servicePort: 80
```

**What is wrong:**
- `extensions/v1beta1` Ingress was removed in Kubernetes 1.22. This manifest fails on any modern cluster.
- The `kubernetes.io/ingress.class` annotation is deprecated; use `spec.ingressClassName: nginx`.
- The backend syntax (`serviceName`/`servicePort`) is the old format. The `networking.k8s.io/v1` API uses `service.name` and `service.port.number`.
- Missing `pathType` field, which is required in `networking.k8s.io/v1`.
- LLMs frequently generate this because training data contains many examples of the old API.

## 7. Secret Data in a ConfigMap

```yaml
# BAD -- DO NOT USE
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: my-app
data:
  DATABASE_URL: "postgres://admin:s3cretP@ssw0rd@postgres:5432/mydb"
  API_KEY: "sk-live-abc123def456"
  AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  config.yaml: |
    smtp:
      username: noreply@company.com
      password: emailP@ss123
```

**What is wrong:**
- ConfigMaps are stored unencrypted in etcd and are readable by anyone with `get` access to the namespace.
- ConfigMap data appears in plain text in `kubectl describe`, in logs, and in version control if committed.
- Credentials, API keys, and passwords must be in Secrets (which are at least base64-encoded and can be encrypted at rest).
- Better: use ExternalSecrets or Sealed Secrets so credentials never appear in manifests at all.
- Connection strings with embedded passwords are especially dangerous because they are easily overlooked in review.

## 8. PVC with ReadWriteMany on an Unsupported Provider

```yaml
# BAD -- DO NOT USE
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workers
spec:
  replicas: 5
  selector:
    matchLabels:
      app: workers
  template:
    metadata:
      labels:
        app: workers
    spec:
      containers:
        - name: worker
          image: ghcr.io/org/worker:v1.0.0
          volumeMounts:
            - name: shared
              mountPath: /data
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: shared-data
```

**What is wrong:**
- `gp3` (AWS EBS) does not support `ReadWriteMany`. The PVC will be stuck in `Pending` state with no clear error in pod events.
- EBS volumes are `ReadWriteOnce` only -- they can be attached to a single node.
- For RWX access, use EFS (`efs-sc`), NFS, or a distributed storage solution like Longhorn or Rook-Ceph.
- LLMs frequently pair `ReadWriteMany` with block storage classes because they do not track provider-specific storage capabilities.
- If only one pod needs write access, use `ReadWriteOnce` and a StatefulSet instead of a Deployment.
