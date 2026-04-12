# Kubernetes Skill for Claude Code — KubeShark

KubeShark is a failure-mode-first Kubernetes skill for Claude Code and Codex. It prevents common LLM hallucinations in Kubernetes manifest generation by diagnosing risks before writing YAML.

## Why use it

- **Prevents hallucinations** -- 6 named failure modes with targeted reference files
- **Token-efficient** -- ~650 token activation cost, granular references loaded on demand
- **Production-ready defaults** -- Pod Security Standards restricted profile, proper resource management, cross-resource validation
- **20 reference files** -- covering security, networking, RBAC, probes, storage, Helm, Kustomize, and more

## Key features

- Failure-mode-first diagnostic workflow (diagnose before generate)
- Output contracts with assumptions, tradeoffs, and rollback notes
- LLM mistake checklists in every reference file
- Cross-resource consistency validation (label/selector/port alignment)
- Helm and Kustomize pattern guidance
- Policy engine integration (Kyverno, OPA/Gatekeeper)

## Quick install

```bash
git clone https://github.com/LukasNiessen/kubernetes-skill.git ~/.claude/skills/kubernetes-skill
```

## License

MIT -- see [LICENSE](https://github.com/LukasNiessen/kubernetes-skill/blob/main/LICENSE).
