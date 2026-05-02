# GitOps Controllers

**Load this reference when detected:** Argo CD, Application, ApplicationSet, sync waves, Flux, GitRepository, Kustomization, HelmRepository, HelmRelease, OCIRepository, GitOps, Config Sync, OpenShift GitOps, or pull-based deployment.

## Why this matters

GitOps controllers continuously reconcile desired state. A manifest that is safe with manual `kubectl apply` can become unsafe when pruning, self-healing, sync ordering, generated Applications, or Helm remediation are enabled. Do not load this file for ordinary one-off YAML unless a GitOps controller is involved.

## Shared GitOps Rules

- Treat Git as the source of truth; avoid manual `kubectl edit` remediation except as an emergency action followed by a Git fix.
- Separate application source code from environment configuration when auditability matters.
- Pin chart versions, OCI artifact digests, or Git revisions for production.
- Keep cluster-scoped resources, CRDs, namespaces, and policy baselines in clearly owned bootstrap layers.
- Use narrow controller credentials; the controller should not have cluster-admin by default.
- Pruning and self-healing are powerful; enable them only with rollback and ownership boundaries.

## Argo CD

When generating Argo CD resources:

- Use `argoproj.io` API versions that match the installed Argo CD version.
- Use sync waves for resource ordering within a sync operation; do not assume they order unrelated independent Applications.
- Use hooks only for idempotent Jobs or lifecycle actions with deletion policies.
- Keep `ignoreDifferences` narrow and documented; never hide broad drift to make sync look green.
- For ApplicationSet, verify generator inputs and destination namespaces before enabling automated sync.
- Avoid auto-prune for production bootstrap unless ownership is explicit and reviewed.

## Flux

When generating Flux resources:

- Distinguish Flux `Kustomization` CRs from `kustomization.yaml` files.
- Use `dependsOn` for explicit ordering between Flux Kustomizations or HelmReleases.
- Configure remediation for Helm install/upgrade failures instead of leaving infinite broken retries.
- Keep `interval`, `timeout`, `retryInterval`, and `prune` deliberate per environment.
- Use SOPS or an approved external secret flow for encrypted secrets in Git.
- Validate source references: `GitRepository`, `OCIRepository`, `HelmRepository`, and chart names.

## Rollout and Drift Controls

- For CRD upgrades, apply CRDs before custom resources and avoid deleting CRDs while CRs exist.
- For generated namespaces, verify ownership before pruning.
- For multi-cluster GitOps, make cluster selection explicit and review generator filters.
- For Helm under GitOps, render locally and validate the rendered manifests before relying on controller reconciliation.

## Validation

- Argo CD: `argocd app diff <app>` and `argocd app get <app>`
- Argo CD in-cluster: `kubectl get applications,applicationsets -A`
- Flux: `flux diff kustomization <name> --path <path>` where available
- Flux: `flux reconcile kustomization <name> --with-source` for controlled reconciliation
- Generic: render Helm/Kustomize output and run `kubectl apply --dry-run=server`

## LLM Mistake Checklist

- Enabling automated prune/self-heal without ownership boundaries.
- Assuming sync waves order separate Applications or separate controllers.
- Creating hooks that are not idempotent.
- Using broad `ignoreDifferences` to mask real drift.
- Confusing Flux `Kustomization` CRs with Kustomize files.
- Omitting `dependsOn` for Flux resources that require ordering.
- Putting plaintext secrets in Git because GitOps needs declarative state.

## Grounding Sources

- Argo CD best practices: https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/
- Argo CD sync phases and waves: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/
- Argo CD ApplicationSet progressive syncs: https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Progressive-Syncs/
- Flux concepts: https://fluxcd.io/flux/concepts/
- Flux Helm controller: https://fluxcd.io/docs/components/helm/
