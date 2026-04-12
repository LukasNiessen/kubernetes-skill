# Network Exposure

**USE THIS GUIDE** when generating or reviewing any Kubernetes networking resource:
Services, Ingress, Gateway, NetworkPolicy, or any manifest involving cross-pod communication.
Default posture: **deny all traffic** and explicitly allow only what is required.

---

## Symptoms

- **All pods can reach all pods**: default Kubernetes networking is flat and open.
- **Unexpected external exposure**: `NodePort` or `LoadBalancer` Service created without intent.
- **DNS resolution failures**: wrong Service name, missing namespace qualifier, ndots misconfiguration.
- **Silent routing to nothing**: Service selector does not match any pod labels; no error, just no backends.
- **Lateral movement after compromise**: attacker pivots freely between namespaces because no NetworkPolicy exists.
- **Ingress 404s or 502s**: path matching, backend Service name, or port mismatch.
- **Slow DNS**: excessive search domain lookups from default `ndots: 5` setting.

---

## Root Causes

1. Kubernetes has **no network segmentation by default** -- every pod can reach every other pod on any port.
2. LLMs generate `NodePort` and `LoadBalancer` Services when `ClusterIP` is sufficient.
3. Service `selector` labels silently fail when they do not match pod `labels` -- zero errors, zero traffic.
4. NetworkPolicies are additive (union of all policies), but **no policy means allow-all**, not deny-all.
5. Egress policies are forgotten -- ingress-only policies still allow unrestricted outbound traffic.
6. DNS resolution requires the full `<svc>.<ns>.svc.cluster.local` form for cross-namespace calls.
7. Ingress path matching semantics differ between `Exact`, `Prefix`, and regex-based controllers.

---

## Prevention Rules

### Default-Deny NetworkPolicy

Apply to every namespace before deploying any workload. Without this, all traffic is permitted.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Matches ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = deny everything
```

After applying default-deny, explicitly allow required traffic with additional policies.

### Allowing Specific Ingress Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        # Allow from pods in the same namespace with specific label
        - podSelector:
            matchLabels:
              role: frontend
        # Allow from pods in another namespace
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 8080
```

Important: `namespaceSelector` and `podSelector` in the same `from` entry are AND-ed.
Separate `from` entries are OR-ed. This is the most common NetworkPolicy mistake:

```yaml
# AND logic -- pods matching BOTH conditions:
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: staging
        podSelector:             # Same list item = AND
          matchLabels:
            app: client

# OR logic -- pods matching EITHER condition:
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: staging
      - podSelector:             # Separate list item = OR
          matchLabels:
            app: client
```

### Egress Policies: DNS, External APIs, Cross-Namespace

Always allow DNS (port 53) in egress policies or all name resolution breaks:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Egress
  egress:
    # Allow DNS resolution (kube-dns / CoreDNS)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow traffic to the database in the same namespace
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow HTTPS to external APIs (CIDR-based)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### Service Types

| Type           | Exposure          | When to use                                      |
|----------------|-------------------|--------------------------------------------------|
| `ClusterIP`    | Internal only     | **Default.** All inter-service communication.    |
| `NodePort`     | Every node IP     | Avoid in production. Debugging only.             |
| `LoadBalancer`  | External via LB   | Only when direct external access is required.   |
| `ExternalName` | DNS CNAME alias   | Bridging to external services. No proxying.      |

Always explicitly set `type: ClusterIP` rather than relying on the default -- it documents intent.

### Service Selector Matching: The Silent Failure

The number one networking debugging issue. Service `selector` must exactly match pod `labels`:

```yaml
# Deployment labels
template:
  metadata:
    labels:
      app: api-server        # <-- This label
      version: v2

# Service selector -- MUST match
spec:
  selector:
    app: api-server          # <-- Must be identical
  # Do NOT include 'version: v2' unless you want to select only v2 pods
```

If the selector matches zero pods, the Service gets zero Endpoints. There is no error, no warning,
no log entry. Traffic simply vanishes. Always verify with `kubectl get endpoints <svc-name>`.

### Ingress and IngressClass

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    # Controller-specific annotations (nginx example)
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx          # Required -- do not omit
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert     # Must exist as a TLS Secret in the same namespace
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix       # Prefix, Exact, or ImplementationSpecific
            backend:
              service:
                name: api-server   # Must match a Service in the same namespace
                port:
                  number: 80       # Must match a port on the Service (not the pod)
```

### Gateway API (Modern Alternative to Ingress)

Gateway API provides richer routing, better role separation, and is the future direction:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: production
spec:
  parentRefs:
    - name: production-gateway
      namespace: gateway-infra
  hostnames:
    - api.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - name: api-server
          port: 80
```

### DNS Considerations

**ndots setting**: Kubernetes default is `ndots: 5`, meaning any name with fewer than 5 dots
gets the search domains appended first. For external names like `api.stripe.com` (2 dots),
the resolver tries `api.stripe.com.production.svc.cluster.local` and several others before
resolving the real address. Fix with a trailing dot or lower ndots:

```yaml
spec:
  dnsConfig:
    options:
      - name: ndots
        value: "2"           # Reduces unnecessary search domain lookups
```

**Cross-namespace DNS**: always use the full form `<service>.<namespace>.svc.cluster.local`
or at minimum `<service>.<namespace>`. Never rely on short names across namespaces.

**Headless Services for StatefulSets**: required for stable per-pod DNS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  clusterIP: None              # Headless -- returns pod IPs directly
  selector:
    app: postgres
  ports:
    - port: 5432
```

Each pod gets a DNS record: `postgres-0.postgres.production.svc.cluster.local`.

---

## Patterns

### GOOD: Full Stack with Network Segmentation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
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
          image: registry.example.com/api-server:v2.4.1@sha256:abc123...
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-server
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api-server               # Matches pod label exactly
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-server
                port:
                  number: 80
```

### BAD: NodePort Service with No Network Policy

```yaml
# INSECURE - DO NOT USE
apiVersion: v1
kind: Service
metadata:
  name: api-server
spec:
  type: NodePort                  # Exposed on every node's IP
  selector:
    app: api                      # Does this match the pod labels? Who knows.
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080             # Hardcoded, conflicts with other services
  # No NetworkPolicy -- every pod in the cluster can reach this
  # No Ingress -- no TLS termination, no host-based routing
  # No namespace -- lands wherever current context points
```

Problems with the bad example:
1. `NodePort` exposes the service on every node at port 30080 -- no access control.
2. Selector says `app: api` but pod label might be `app: api-server` -- silent mismatch.
3. No NetworkPolicy -- all pods in the cluster can reach this service.
4. No TLS termination -- traffic is unencrypted.
5. No namespace specified.
6. Hardcoded `nodePort` value -- port conflicts are discovered only at apply time.
7. No Ingress -- direct node IP access bypasses all edge security.

---

## LLM Mistake Checklist

Before emitting any networking manifest, verify every item:

- [ ] **Default-deny NetworkPolicy exists in the target namespace** -- or is included in the output.
- [ ] **Both `policyTypes: [Ingress, Egress]` specified** -- ingress-only policies still allow all egress.
- [ ] **DNS egress (port 53 UDP+TCP to kube-system) explicitly allowed** -- or all name resolution breaks.
- [ ] **Service type is `ClusterIP`** unless external access is explicitly required and justified.
- [ ] **Service `selector` exactly matches pod template `labels`** -- verify spelling, casing, key names.
- [ ] **Service `targetPort` matches the container `containerPort`** -- not the Service `port`.
- [ ] **Ingress specifies `ingressClassName`** -- omitting it relies on a default class that may not exist.
- [ ] **Ingress TLS block includes both `hosts` and `secretName`** -- and the Secret exists.
- [ ] **Ingress backend `service.port.number` matches the Service `port`** -- not the pod `targetPort`.
- [ ] **Cross-namespace DNS uses full form** `<svc>.<ns>.svc.cluster.local` -- short names do not resolve.
- [ ] **NetworkPolicy `namespaceSelector` + `podSelector` AND/OR logic is correct** -- same item = AND, separate items = OR.
- [ ] **No `hostNetwork: true`** unless explicitly required -- it bypasses all NetworkPolicy enforcement.

---

## Verification Commands

```bash
# Check if any NetworkPolicy exists in the namespace
kubectl get networkpolicy -n production

# Verify Service has endpoints (non-zero)
kubectl get endpoints api-server -n production
# If ENDPOINTS column shows <none>, selector does not match any pods

# Compare Service selector with pod labels
kubectl get svc api-server -n production -o jsonpath='{.spec.selector}' | jq .
kubectl get pods -n production -l app=api-server -o name

# Test DNS resolution from inside a pod
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.36 -- nslookup api-server.production.svc.cluster.local

# Test connectivity between pods (with NetworkPolicy)
kubectl run nettest --rm -it --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://api-server.production:80/healthz

# List all Services of type NodePort or LoadBalancer (potential exposure)
kubectl get svc -A -o json | jq -r '.items[] | select(.spec.type == "NodePort" or .spec.type == "LoadBalancer") | "\(.metadata.namespace)/\(.metadata.name): \(.spec.type)"'

# Check Ingress status and assigned addresses
kubectl get ingress -n production -o wide

# Verify TLS Secret exists and is valid
kubectl get secret api-tls-cert -n production -o jsonpath='{.type}'
# Should output: kubernetes.io/tls

# Inspect NetworkPolicy rules for a specific pod
kubectl get networkpolicy -n production -o json | jq '.items[] | select(.spec.podSelector.matchLabels.app == "api-server")'

# Check for pods using hostNetwork (bypasses NetworkPolicy)
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.hostNetwork == true) | "\(.metadata.namespace)/\(.metadata.name)"'

# Validate manifests
kubeconform -strict -kubernetes-version 1.30.0 manifest.yaml
```
