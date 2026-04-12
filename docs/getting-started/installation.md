# Installation

KubeShark can be installed in three ways depending on your environment: direct clone (recommended), marketplace install, or per-project setup for Codex.

---

## Option 1: Direct Clone (Recommended)

Clone the repository into your Claude Code skills directory. Claude Code auto-discovers skills in `~/.claude/skills/` -- no restart or configuration needed.

### macOS / Linux

```bash
git clone https://github.com/LukasNiessen/kubernetes-skill.git ~/.claude/skills/kubernetes-skill
```

### Windows (PowerShell)

```powershell
git clone https://github.com/LukasNiessen/kubernetes-skill.git "$env:USERPROFILE\.claude\skills\kubernetes-skill"
```

### Windows (Command Prompt)

```cmd
git clone https://github.com/LukasNiessen/kubernetes-skill.git "%USERPROFILE%\.claude\skills\kubernetes-skill"
```

After cloning, the skill is active immediately. Claude Code reads `SKILL.md` on the next Kubernetes-related prompt.

---

## Option 2: Marketplace Install

Claude Code includes a built-in plugin system with marketplace support. This avoids manual cloning.

**Add the marketplace source and install:**

```
/plugin marketplace add LukasNiessen/kubernetes-skill
/plugin install kubernetes-skill
```

**Or use the interactive manager:**

1. Run `/plugin` in Claude Code.
2. Switch to the **Discover** tab.
3. Find KubeShark and install.

The marketplace reads `.claude-plugin/marketplace.json` in the repository to register KubeShark as an installable plugin.

---

## Option 3: Codex Per-Project Setup

Codex has no global skill system. Setup is per-project: clone the skill into your repository and reference it from `AGENTS.md`.

**Step 1 -- Clone into your project:**

```bash
git clone https://github.com/LukasNiessen/kubernetes-skill.git .kubernetes-skill
```

**Step 2 -- Reference in AGENTS.md:**

Create or edit `AGENTS.md` in your repository root and add:

```markdown
## Kubernetes

When working with Kubernetes manifests, Helm charts, or Kustomize overlays,
follow the workflow in `.kubernetes-skill/SKILL.md`.
Load references from `.kubernetes-skill/references/` as needed.
```

Codex will follow the workflow whenever it encounters Kubernetes tasks in the project.

---

## Updating

KubeShark is a plain Git repository. Pull the latest changes to update:

**macOS / Linux:**

```bash
cd ~/.claude/skills/kubernetes-skill && git pull
```

**Windows (PowerShell):**

```powershell
cd "$env:USERPROFILE\.claude\skills\kubernetes-skill"; git pull
```

**Codex projects:**

```bash
cd .kubernetes-skill && git pull
```

---

## Uninstalling

Remove the cloned directory to uninstall.

**macOS / Linux:**

```bash
rm -rf ~/.claude/skills/kubernetes-skill
```

**Windows (PowerShell):**

```powershell
Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\kubernetes-skill"
```

**Codex projects:**

```bash
rm -rf .kubernetes-skill
```

Also remove the corresponding section from `AGENTS.md` if you added one.

**Marketplace installs:**

```
/plugin uninstall kubernetes-skill
```

---

## Verifying Installation

Confirm the skill is installed correctly by checking that `SKILL.md` exists:

**macOS / Linux:**

```bash
ls ~/.claude/skills/kubernetes-skill/SKILL.md
```

**Windows (PowerShell):**

```powershell
Test-Path "$env:USERPROFILE\.claude\skills\kubernetes-skill\SKILL.md"
```

If the file exists, KubeShark is ready. You can also verify by asking Claude Code a Kubernetes question -- the response should follow the 7-step workflow and include an output contract with assumptions, failure modes, and rollback notes.
