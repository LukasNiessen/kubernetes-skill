# Changelog

All notable changes to the Kubernetes Skill (KubeShark) are documented here. This project uses [Semantic Versioning](https://semver.org/).

For the repository-level changelog, see [CHANGELOG.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/CHANGELOG.md).

---

## v1.0.0

Initial release of KubeShark.

### Failure Modes
- 6 primary failure modes: insecure workload defaults, resource starvation, network exposure, privilege sprawl, fragile rollouts, API drift
- 7-step failure-mode-first diagnostic workflow (diagnose before generate)

### Reference Files
- 20 granular reference files covering failure modes, workload patterns, cross-cutting concerns, tooling, and examples
- LLM mistake checklists in every reference file that covers a risk domain

### Pattern Banks
- 8 production-ready good examples with annotated YAML
- 8 common anti-pattern bad examples with explanations
- Do/Don't checklist spanning 9 categories

### Tooling
- Helm chart pattern guidance with template conventions
- Kustomize overlay and patch patterns
- Validation and policy enforcement (kubeconform, Kyverno, OPA/Gatekeeper, Polaris)

### Infrastructure
- HonKit documentation site
- GitHub Actions CI validation and docs deployment
- Conventional commits and semantic versioning
