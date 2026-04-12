# Token Efficiency

How KubeShark minimizes context window consumption while maximizing manifest generation quality.

## The Problem

Context window space is a finite resource. Every token spent on skill content is a token unavailable for the user's actual manifests, conversation history, and tool results. A monolithic skill file that dumps thousands of lines of Kubernetes guidance wastes context on information irrelevant to the current task. This is not just inefficient -- it degrades output quality by forcing the model to process noise alongside signal.

## KubeShark's Approach

KubeShark is designed around three principles:

### Lean Activation

The core SKILL.md is approximately 85 lines (~650 tokens). It contains no YAML examples, no inline manifests, no tutorial material. It is purely procedural: a 7-step workflow the model follows. This means the skill activates with minimal context cost regardless of the task.

### Granular References

Depth lives in 20 separate reference files organized by concern:

- **6 failure mode files** -- insecure workload defaults, resource starvation, network exposure, privilege sprawl, fragile rollouts, API drift
- **4 workload pattern files** -- Deployments, StatefulSets, Jobs/CronJobs, DaemonSets and operators
- **4 cross-cutting concern files** -- security hardening, observability, multi-tenancy, storage and state
- **3 tooling files** -- Helm patterns, Kustomize patterns, validation and policy
- **3 pattern bank files** -- good examples, bad examples, do/don't checklist

The model loads only the 1-2 files relevant to the diagnosed failure mode. A query about probe configuration never loads the RBAC guidance. A query about Helm chart structure never loads the NetworkPolicy patterns.

### Selective Loading

Step 3 of the workflow explicitly instructs the model to load only the relevant references. This is not a suggestion -- it is a structural constraint built into the diagnostic flow.

## Content Inclusion Rules

Content enters KubeShark only when at least one condition is met:

- It materially lowers the probability of insecure, unreliable, or invalid manifest generation
- It prevents common deploy-time or runtime surprises (probe cascades, selector mismatches, OOMKills)
- It encodes operational guardrails that general model knowledge cannot reliably infer

Content is excluded when:

- It is generic Kubernetes knowledge with low failure impact
- It is cloud-provider-specific deep configuration that belongs in project docs
- It duplicates an existing rule without adding a new decision signal

## What Models Need Help With

LLMs have strong general Kubernetes knowledge but consistently fail on specific operational details:

- **Security contexts** -- models frequently omit them entirely, producing root-running containers
- **Cross-resource consistency** -- label/selector/port alignment across Deployment, Service, Ingress, HPA, PDB
- **API version currency** -- models generate removed APIs from training data (e.g., `extensions/v1beta1`)
- **Provider-specific constraints** -- storage class capabilities, CNI behavior, load balancer semantics
- **Probe design** -- liveness probes that check external dependencies, causing cascading failures

Models generally do not need help with basic YAML syntax, resource kind selection, or standard field names. KubeShark avoids restating what models already know reliably.

## Core Principle

High signal density. Every line in every reference file must earn its token cost by reducing the probability of a specific, named failure mode.
