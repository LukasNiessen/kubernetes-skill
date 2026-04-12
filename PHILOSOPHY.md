# Philosophy

This document describes KubeShark's design strategy, architectural decisions, and the reasoning behind its content.

## Failure-Mode-First Architecture

KubeShark is built around a single insight: telling an LLM *what good Kubernetes looks like* is less effective than telling it *how to think about Kubernetes problems*.

The core SKILL.md is not a reference manual. It is a 7-step operational workflow:

1. **Capture execution context** -- cluster version, distribution, namespace, environment, workload type, deployment method, policy enforcement, cloud provider
2. **Diagnose likely failure modes** -- insecure workload defaults, resource starvation, network exposure, privilege sprawl, fragile rollouts, API drift
3. **Load only relevant references** -- pull targeted guidance, not everything
4. **Propose fix path with risk controls** -- why this works, what could go wrong at runtime, guardrails
5. **Generate implementation artifacts** -- YAML, Helm, Kustomize, policies, RBAC
6. **Validate before finalize** -- dry-run, schema validation, cross-resource consistency, policy scan
7. **Output contract** -- assumptions, tradeoffs, validation plan, rollback notes

The model diagnoses before it generates. This prevents the most common failure pattern in LLM-assisted Kubernetes work: producing syntactically valid but operationally dangerous manifests.

## Why Kubernetes Needs This More Than Terraform

Terraform fails explicitly. A misconfiguration surfaces at `terraform plan` or `terraform apply` with a clear error message. Kubernetes is different in three critical ways:

1. **Silent failures are common.** A Service with the wrong selector deploys successfully but routes to nothing. A NetworkPolicy with a mistyped label silently does nothing. A probe pointing to the wrong port passes creation but fails at runtime. The cluster accepts the manifest, the reconciliation loop enforces the bad state, and the failure only appears when traffic arrives.

2. **Runtime is continuous.** Terraform is plan-and-apply. Kubernetes is a continuous reconciliation loop. A misconfigured liveness probe does not just fail once -- it restarts the pod every 30 seconds forever. A missing PodDisruptionBudget does not just affect one deploy -- it allows every future rolling update to take down all replicas simultaneously.

3. **The blast radius is multi-dimensional.** Terraform operates at infrastructure provisioning time. Kubernetes operates across provisioning, deployment, runtime, networking, scheduling, and security simultaneously. An LLM must reason about all these dimensions for every resource.

## Token Efficiency

Context window space is a finite resource. Every token spent on skill content is a token unavailable for the user's actual manifests, conversation history, and tool results.

KubeShark is designed for minimal activation cost. The core SKILL.md is ~85 lines (~650 tokens). It contains no YAML examples, no inline manifests, no tutorial material. It is purely procedural: a workflow the model follows.

Depth lives in 20 granular reference files. The model loads only the 1-2 files relevant to the diagnosed failure mode. A query about probe configuration never loads the RBAC guidance. A query about Helm chart structure never loads the NetworkPolicy patterns.

This granularity matters. A single large reference file forces the model to process thousands of irrelevant tokens. 20 small files let it load precisely what it needs.

## Six Failure Modes

KubeShark names six failure modes explicitly. Six -- not five -- because Kubernetes has a broader failure surface than Terraform. The additional dimension (runtime behavior) demanded its own failure mode.

1. **Insecure workload defaults** -- missing security contexts, Pod Security Standard violations, host access, excessive capabilities
2. **Resource starvation** -- missing requests/limits, no QoS strategy, absent PodDisruptionBudgets, scheduling chaos
3. **Network exposure** -- flat networking, missing NetworkPolicies, wrong Service types, DNS misconfigurations
4. **Privilege sprawl** -- overly permissive RBAC, leaked secrets, unscoped ServiceAccount tokens
5. **Fragile rollouts** -- misconfigured probes, mutable image tags, unsafe update strategies, missing graceful shutdown
6. **API drift** -- wrong apiVersion values, deprecated APIs, schema violations, Helm/Kustomize structural errors

These are not arbitrary categories. They represent the six most common ways LLM-generated Kubernetes manifests cause real damage. Every piece of content in the skill maps to at least one of these failure modes. Content that does not reduce the probability of any failure mode is excluded.

## LLM-Aware Guardrails

Every reference file that covers a risk domain includes an **LLM mistake checklist** -- a list of specific errors that language models make when generating Kubernetes configurations. Examples:

- Omitting securityContext entirely, producing manifests that run as root
- Setting liveness probes that check external dependencies, causing cascading restarts
- Using `apiVersion: extensions/v1beta1` for Ingress (removed in 1.22)
- Generating RBAC with wildcard verbs and resources on ClusterRoleBindings
- Omitting resource requests and limits, or using arbitrary round numbers
- Using `:latest` image tags without imagePullPolicy override
- Creating Services with selectors that do not match any pod labels

These checklists exist because the model needs to know *what it gets wrong*, not just *what is correct*. A reference that only shows the right pattern still allows the model to hallucinate the wrong one. A reference that explicitly names the hallucination pattern reduces it.

## Cross-Resource Consistency

Kubernetes has a unique failure pattern that Terraform largely avoids: **cross-resource reference breakage**. A Kubernetes application typically requires 5-10 resources that must be consistent with each other through implicit label/selector/port references:

- Deployment `.spec.selector.matchLabels` must match `.spec.template.metadata.labels`
- Service `.spec.selector` must match Deployment pod template labels
- Ingress backend service name and port must match the actual Service
- PDB `.spec.selector.matchLabels` must match Deployment pod template labels
- HPA `.spec.scaleTargetRef` must reference the correct Deployment name and apiVersion
- NetworkPolicy `.spec.podSelector` must match intended target pods

LLMs frequently break these cross-resource references. KubeShark's validation step (Step 6) explicitly checks for this.

## Output Contracts

Every KubeShark response includes a structured output contract:

- **Assumptions and cluster version floor** -- what the model assumed about the cluster
- **Selected failure modes** -- which risks were diagnosed
- **Chosen remediation and tradeoffs** -- what was recommended and what was traded off
- **Validation/test plan** -- how to verify the output
- **Rollback/recovery notes** -- how to undo if something goes wrong

This makes outputs auditable. A reader can check whether the model's assumptions were correct, whether the right failure modes were identified, and whether the rollback path is viable -- before applying anything.

## Reference Granularity

The 20 reference files are organized by concern, not by Kubernetes concept:

**Primary failure modes** (loaded when the failure mode is diagnosed):
- Insecure workload defaults, resource starvation, network exposure, privilege sprawl, fragile rollouts, API drift

**Workload patterns** (loaded when generating specific resource types):
- Deployment patterns, StatefulSet patterns, Job/CronJob patterns, DaemonSet and operator patterns

**Cross-cutting concerns** (loaded when the task spans multiple domains):
- Security hardening, observability, multi-tenancy, storage and state

**Tooling and deployment** (loaded for tool-specific tasks):
- Helm patterns, Kustomize patterns, validation and policy

**Pattern banks** (loaded for review or teaching):
- Good examples, bad examples, do/don't patterns

Each file is self-contained. No file depends on another file being loaded simultaneously.

## Grounding Sources

When guidance conflicts, KubeShark resolves in this order:

1. **Official Kubernetes documentation** (kubernetes.io)
2. **Pod Security Standards** (restricted profile as default target)
3. **NSA/CISA Kubernetes Hardening Guide**
4. **OWASP Kubernetes Top 10**
5. **CIS Kubernetes Benchmark**

## Content Inclusion Rules

Content enters KubeShark only when at least one condition is met:

1. It materially lowers the probability of insecure, unreliable, or invalid manifest generation
2. It prevents common deploy-time or runtime surprises (probe cascades, selector mismatches, OOMKills)
3. It encodes operational guardrails that general model knowledge cannot infer

Content is excluded when:

1. It is generic Kubernetes knowledge with low failure impact
2. It is cloud-provider-specific deep configuration that belongs in project docs
3. It duplicates an existing rule without adding a new decision signal

If repeated failure patterns emerge, targeted lines are added for that specific failure mode instead of broad expansion.

---

## Default Security Posture

KubeShark defaults to the Pod Security Standards "restricted" profile. Every generated workload includes these security controls unless the user explicitly requests deviation (with justification noted in the output contract):

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true`
- `capabilities: { drop: ["ALL"] }`
- `seccompProfile: { type: RuntimeDefault }`

This is a deliberate design choice. The restricted profile prevents the largest class of container escape vulnerabilities. Deviations should be explicit, justified, and documented.
