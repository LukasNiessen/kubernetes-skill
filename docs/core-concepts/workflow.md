# Workflow

KubeShark operates through a 7-step workflow defined in `SKILL.md`. The workflow runs top to bottom on every Kubernetes task. This page explains what each step does and why it exists.

---

## Step 1: Capture Execution Context

Before writing any YAML, KubeShark records the environment it is operating in. This prevents the most common LLM failure: generating manifests that assume a generic cluster and ignore the user's actual setup.

**Context captured:**

| Dimension | Examples | Why it matters |
|-----------|----------|----------------|
| Cluster version | 1.29, 1.30, 1.31 | API availability differs across versions; deprecated APIs cause hard failures |
| Distribution | EKS, GKE, AKS, k3s, vanilla | Each has distribution-specific defaults, storage classes, and networking behaviors |
| Namespace | `default`, `production`, `monitoring` | Determines resource quotas, network policies, and RBAC scope |
| Environment | dev, staging, prod | Controls security strictness, resource sizing, and validation rigor |
| Workload type | Deployment, StatefulSet, Job, CronJob, DaemonSet | Different workload types have different failure patterns and configuration requirements |
| Deployment method | Raw YAML, Helm, Kustomize, operator-managed | Determines output format and which tooling references to load |
| Policy enforcement | Pod Security Admission, Kyverno, OPA/Gatekeeper | Affects what security controls are required versus optional |
| Cloud provider and CNI | AWS/VPC CNI, GCP/Calico, Azure/Azure CNI | Impacts networking, storage classes, load balancer annotations, and service mesh compatibility |

When any dimension is unknown, KubeShark states the assumption explicitly rather than guessing silently. These assumptions appear in the output contract (Step 7) so the user can verify them.

---

## Step 2: Diagnose Failure Modes

This is the step that distinguishes KubeShark from a reference manual. Before generating anything, the workflow identifies which of the six failure modes are relevant to the task.

**The six failure modes:**

1. **Insecure workload defaults** -- missing security contexts, PSS violations, host access, excessive capabilities
2. **Resource starvation** -- missing requests/limits, no QoS strategy, absent PodDisruptionBudgets, scheduling chaos
3. **Network exposure** -- flat networking, missing NetworkPolicies, wrong Service types, DNS misconfigurations
4. **Privilege sprawl** -- overly permissive RBAC, leaked secrets, unscoped ServiceAccount tokens
5. **Fragile rollouts** -- misconfigured probes, mutable image tags, unsafe update strategies, missing graceful shutdown
6. **API drift** -- wrong apiVersion, deprecated APIs, schema violations, tool-specific structural errors

Most tasks trigger multiple failure modes. A "create a Deployment with an Ingress" request involves at least insecure workload defaults, network exposure, and fragile rollouts. The diagnosis step ensures none of these are overlooked.

See [Failure Modes](failure-modes.md) for a detailed breakdown of each.

---

## Step 3: Load Targeted References

KubeShark includes 20 reference files, but only 1-2 are loaded per query. This is a deliberate token efficiency decision: loading all references would burn thousands of tokens on irrelevant guidance.

**Reference selection logic:**

- A probe configuration question loads `fragile-rollouts.md` -- it never touches `privilege-sprawl.md` or `network-exposure.md`.
- A Helm chart task loads `helm-patterns.md` and the failure-mode reference for the workload being charted.
- A security review loads `insecure-workload-defaults.md` and `security-hardening.md`.

**Reference categories:**

| Category | Files | Loaded when |
|----------|-------|-------------|
| Primary failure modes | 6 files (one per failure mode) | The corresponding failure mode is diagnosed in Step 2 |
| Workload patterns | Deployment, StatefulSet, Job, DaemonSet patterns | Generating a specific workload type |
| Cross-cutting concerns | Security hardening, observability, multi-tenancy, storage | The task spans multiple domains |
| Tooling | Helm patterns, Kustomize patterns, validation and policy | Using a specific deployment tool |
| Pattern banks | Good examples, bad examples, do/don't checklist | Reviewing code or learning patterns |

Each reference file is self-contained. No file depends on another being loaded simultaneously.

---

## Step 4: Propose Fix Path

For every recommendation, KubeShark provides three things:

1. **Why this addresses the failure mode** -- the causal link between the fix and the diagnosed risk.
2. **What could still go wrong** -- runtime behavior, edge cases, and deployment-time risks that remain even after the fix.
3. **Guardrails** -- validation commands, policy checks, and rollback paths that protect against the remaining risks.

This structure prevents a common LLM pattern: recommending a fix without acknowledging its limitations. A liveness probe fix that does not mention the risk of checking external dependencies is incomplete. A NetworkPolicy recommendation that does not mention egress is incomplete.

---

## Step 5: Generate Artifacts

When the task calls for implementation, KubeShark produces the appropriate artifacts:

- **Kubernetes manifests** -- YAML with security contexts, resource limits, proper labels, and annotations
- **Helm values and templates** -- chart structure following Helm best practices
- **Kustomize overlays** -- base/overlay structure with proper patch formats
- **NetworkPolicies** -- default-deny with explicit allow rules
- **RBAC resources** -- least-privilege Roles and RoleBindings with dedicated ServiceAccounts
- **PodDisruptionBudgets** -- tuned to workload replica count and availability requirements
- **Policy rules** -- Kyverno ClusterPolicies or OPA/Gatekeeper ConstraintTemplates

All generated manifests default to the Pod Security Standards restricted profile: `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `drop: ["ALL"]` capabilities, and `RuntimeDefault` seccomp profile.

---

## Step 6: Validate

KubeShark never recommends applying directly to production without validation. Every response includes validation steps matched to the deployment method and risk level:

- **`kubectl apply --dry-run=server`** or **`kubectl diff`** -- catches API-level errors without making changes
- **`kubeconform`** -- schema validation against the target cluster version to catch API drift
- **Cross-resource consistency checks** -- verifies that labels, selectors, ports, and names align across Deployments, Services, Ingress, PDBs, HPAs, and NetworkPolicies
- **Policy scan** -- PSS profile compliance check, Kyverno audit, or OPA/Gatekeeper dry-run

Cross-resource consistency is especially important because Kubernetes silently accepts mismatched selectors. A Service with a selector that matches no pods deploys without error -- the failure only surfaces when traffic arrives.

---

## Step 7: Output Contract

Every KubeShark response ends with a structured output contract containing five sections:

| Section | Purpose |
|---------|---------|
| **Assumptions and cluster version floor** | States what was assumed about the cluster, distribution, and environment so the user can verify |
| **Selected failure modes** | Lists which of the 6 failure modes were diagnosed as relevant |
| **Chosen remediation and tradeoffs** | Explains what was recommended and what was explicitly traded off |
| **Validation/test plan** | Provides the specific commands and checks to verify the output |
| **Rollback/recovery notes** | Describes how to undo the changes if something goes wrong -- `kubectl rollout undo`, revision history, data safety considerations |

The output contract makes every response auditable. A reviewer can check whether the assumptions match reality, whether the right failure modes were identified, and whether the rollback path is viable -- all before applying anything to the cluster.
