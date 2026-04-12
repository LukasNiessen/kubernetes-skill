# Philosophy

This page describes the design rationale behind KubeShark. For the full treatment, see [PHILOSOPHY.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/PHILOSOPHY.md) in the repository root.

---

## Failure-Mode-First vs. Reference Manuals

The core insight: telling an LLM *what good Kubernetes looks like* is less effective than telling it *how to think about Kubernetes problems*.

A static reference manual gives the model information but no diagnostic process. There is no risk assessment step, no structured output, and no way to verify that the right concerns were addressed. The model reads the reference and generates whatever it thinks fits.

KubeShark takes the opposite approach. The core `SKILL.md` is an operational workflow, not a knowledge dump. It forces a diagnostic sequence: capture context, identify failure modes, load only relevant references, propose fixes with risk controls, validate, and deliver a structured output contract. The model diagnoses before it generates.

---

## Why Kubernetes Needs This More Than Terraform

Terraform fails explicitly. A misconfiguration surfaces at `terraform plan` or `terraform apply` with a clear error message. Kubernetes is different in three critical ways:

**Silent failures are common.** A Service with the wrong selector deploys successfully but routes to nothing. A NetworkPolicy with a mistyped label silently does nothing. A probe pointing to the wrong port passes creation but fails at runtime. The cluster accepts the manifest without complaint -- failures surface only when traffic arrives.

**Runtime is continuous.** Terraform is plan-and-apply. Kubernetes is a continuous reconciliation loop. A misconfigured liveness probe does not just fail once -- it restarts the pod every 30 seconds forever. A missing PodDisruptionBudget does not just affect one deploy -- it allows every future rolling update to take down all replicas simultaneously.

**The blast radius is multi-dimensional.** Terraform operates at infrastructure provisioning time. Kubernetes operates across provisioning, deployment, runtime, networking, scheduling, and security simultaneously. An LLM must reason about all these dimensions for every resource it generates.

These properties make a diagnostic workflow essential. Without one, the LLM produces syntactically valid but operationally dangerous manifests -- and the cluster silently accepts them.

---

## Token Efficiency as Design Constraint

Context window space is a finite resource. Every token spent on skill content is a token unavailable for the user's actual manifests, conversation history, and tool results.

KubeShark is designed for minimal activation cost:

- **SKILL.md is ~85 lines (~650 tokens).** It contains no YAML examples, no inline manifests, and no tutorial material. It is purely procedural.
- **20 granular reference files.** The model loads only the 1-2 files relevant to the diagnosed failure mode per query.
- **No duplication.** A query about probe configuration never loads the RBAC guidance. A query about Helm chart structure never loads the NetworkPolicy patterns.

A single large reference file would force the model to process thousands of irrelevant tokens. Twenty small files let it load precisely what it needs.

---

## LLM-Aware Guardrails

Every reference file that covers a risk domain includes an **LLM mistake checklist** -- a list of specific errors that language models make when generating Kubernetes configurations:

- Omitting `securityContext` entirely, producing manifests that run as root
- Setting liveness probes that check external dependencies, causing cascading restarts
- Using `apiVersion: extensions/v1beta1` for Ingress (removed in 1.22)
- Generating RBAC with wildcard verbs and resources on ClusterRoleBindings
- Omitting resource requests and limits, or using arbitrary round numbers
- Using `:latest` image tags without `imagePullPolicy` override
- Creating Services with selectors that do not match any pod labels

These checklists exist because the model needs to know *what it gets wrong*, not just *what is correct*. A reference that only shows the right pattern still allows the model to hallucinate the wrong one. A reference that explicitly names the hallucination pattern reduces it.

---

## Output Contracts for Auditability

Every KubeShark response ends with a structured output contract: assumptions, failure modes addressed, remediation choices and tradeoffs, validation plan, and rollback notes.

This is a deliberate design choice. Kubernetes manifests applied to a cluster have real operational consequences. The output contract makes every response auditable -- a reviewer can check whether the model's assumptions matched reality, whether the right risks were identified, and whether the rollback path is viable, all before applying anything.

Without an output contract, the user receives a manifest and must independently assess whether it is safe. The contract shifts that burden: the model states what it assumed and what it did not account for.

---

## Default Security Posture

KubeShark defaults to the Pod Security Standards **restricted** profile. Every generated workload includes:

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `readOnlyRootFilesystem: true`
- `capabilities: { drop: ["ALL"] }`
- `seccompProfile: { type: RuntimeDefault }`

The restricted profile prevents the largest class of container escape vulnerabilities. Deviations are allowed only when the user explicitly requests them, and the deviation is documented in the output contract with justification.

This is a secure-by-default posture. It is easier to relax security with documented justification than to retroactively harden manifests that were generated permissively.
