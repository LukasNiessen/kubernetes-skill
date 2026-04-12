# Contributing

Thanks for contributing to Kubernetes Skill (KubeShark). This is a condensed guide. For the full version, see [CONTRIBUTING.md](https://github.com/LukasNiessen/kubernetes-skill/blob/main/CONTRIBUTING.md).

## Core Principle

Every change must map to a failure mode. Before submitting, answer three questions:

1. Which failure mode does this prevent?
2. What measurable quality gain does it provide?
3. Is the token cost justified?

## Development Flow

1. **Branch** -- create a feature or fix branch from `main`
2. **Change** -- make focused changes; keep PRs small and single-purpose
3. **Check** -- run local checks (see below)
4. **PR** -- open a pull request using the PR template

## Local Checks

```bash
# Verify no placeholder text remains
rg -n "FIXME|placeholder-text" README.md SKILL.md references/*.md

# Verify required files exist
python - <<'PY'
from pathlib import Path
assert Path('SKILL.md').exists()
assert Path('README.md').exists()
for p in [
  'references/insecure-workload-defaults.md',
  'references/resource-starvation.md',
  'references/network-exposure.md',
  'references/privilege-sprawl.md',
  'references/fragile-rollouts.md',
  'references/api-drift.md',
]:
  assert Path(p).exists(), f'missing {p}'
print('basic structure OK')
PY
```

## Content Rules

- Keep examples original and clearly distinct
- Prefer failure-mode framing over generic best-practice text
- Avoid cloud-provider-specific deep dives unless they directly reduce a known LLM failure mode
- Keep claims precise; avoid vague "always" language when tradeoffs exist
- Default to the PSS restricted profile in all examples

## Required for PR Approval

- Clear mapping to one or more failure modes
- No contradictory guidance across references
- Updated links and indexes if files were moved or renamed
- Validation workflow passing (`.github/workflows/validate.yml`)

## Security

- Never commit credentials, tokens, or secret values
- Do not paste real cluster state or kubeconfig data
- Do not include real IP addresses, hostnames, or cloud account identifiers

## Reporting Issues

Open an issue with: the observed hallucination or failure pattern, a minimal reproducible prompt/context, and the expected behavior.
