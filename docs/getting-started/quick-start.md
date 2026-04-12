# Quick Start

Get KubeShark running in under two minutes.

---

## 1. Install

```bash
git clone https://github.com/LukasNiessen/kubernetes-skill.git ~/.claude/skills/kubernetes-skill
```

See the [Installation guide](installation.md) for Windows commands and alternative install methods.

---

## 2. Use It

### Explicit invocation

Prefix your prompt with `/kubernetes-skill` to invoke the skill directly:

```
/kubernetes-skill Create a production-ready Deployment for a Node.js API with autoscaling
```

```
/kubernetes-skill Review my StatefulSet for security and reliability issues
```

### Automatic activation

KubeShark activates automatically when Claude Code detects a Kubernetes-related task. No prefix needed:

```
Create a Helm chart for a PostgreSQL StatefulSet with backup CronJobs
```

```
Review my deployment.yaml for security issues
```

Both invocation methods produce the same structured output.

---

## 3. What to Expect

Every KubeShark response follows a **7-step workflow**:

| Step | What happens |
|------|-------------|
| 1. Capture context | Records cluster version, distribution, namespace, environment, workload type |
| 2. Diagnose failure modes | Identifies which of the 6 failure modes apply to your task |
| 3. Load references | Pulls 1-2 targeted reference files (not the entire knowledge base) |
| 4. Propose fix path | Recommends a solution with risk controls and runtime behavior notes |
| 5. Generate artifacts | Produces YAML manifests, Helm charts, Kustomize overlays, or policies |
| 6. Validate | Provides dry-run commands, schema validation, and consistency checks |
| 7. Output contract | States assumptions, tradeoffs, validation plan, and rollback notes |

The output contract at the end is the key differentiator. It makes every response auditable -- you can verify assumptions and check the rollback path before applying anything to your cluster.

---

## 4. Example Tasks

KubeShark handles a wide range of Kubernetes work. Here are common task types to try:

### Deployment creation

```
/kubernetes-skill Create a production Deployment for a Python Flask API with 3 replicas, resource limits, and an Ingress
```

### Security review

```
/kubernetes-skill Review this Deployment for security issues and harden it with proper security contexts, NetworkPolicies, and RBAC
```

### Helm chart generation

```
/kubernetes-skill Create a Helm chart for a Redis cluster with configurable replicas and persistent storage
```

### Kustomize overlay

```
/kubernetes-skill Build a Kustomize overlay structure with base, staging, and production variants for my microservice
```

### RBAC setup

```
/kubernetes-skill Create least-privilege RBAC for a monitoring service that needs read access to pods and metrics across all namespaces
```

### Troubleshooting

```
/kubernetes-skill My pods are stuck in CrashLoopBackOff with OOMKilled status. Here is my manifest -- diagnose and fix it.
```

### Probe configuration

```
/kubernetes-skill Add proper liveness, readiness, and startup probes for a Java Spring Boot app that takes 90 seconds to start
```

### CI pipeline validation

```
/kubernetes-skill Create a CI pipeline step that validates all manifests with kubeconform and checks for policy violations with Kyverno
```
