# Codex Skills

Personal Codex skills that can be installed on any machine.

Each top-level folder in this repository is one installable skill. A skill is a
self-contained directory with a required `SKILL.md` file and optional resources
such as `agents/`, `references/`, `scripts/`, or `assets/`.

## Available Skills

| Skill | Purpose |
| --- | --- |
| `powershell-safety` | Safer Windows PowerShell command composition, with guidance for paths, quoting, `rg`, git pathspecs, environment variables, and destructive file operations. |

## Install

Install a single skill from this repository:

```bash
install-skill-from-github.py --repo ywenhao/skills --path powershell-safety
```

Or ask Codex:

```text
Install the powershell-safety skill from github.com/ywenhao/skills.
```

Installed skills are copied into the local Codex skills directory:

```text
~/.codex/skills
```

On Windows, this is usually:

```text
C:\Users\<username>\.codex\skills
```

Restart Codex after installing or updating skills so the new metadata is loaded.

## Update

If a skill is already installed, remove the installed copy first and reinstall it
from this repository:

```powershell
Remove-Item -LiteralPath "$env:USERPROFILE\.codex\skills\powershell-safety" -Recurse -Force
install-skill-from-github.py --repo ywenhao/skills --path powershell-safety
```

For private repositories or machines without an existing GitHub session, use SSH
credentials or set `GITHUB_TOKEN` / `GH_TOKEN` before installing.

## Repository Layout

```text
skill-name/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── references/
├── scripts/
└── assets/
```

Guidelines:

- Keep `SKILL.md` concise and focused on instructions Codex needs at runtime.
- Put long documentation in `references/`.
- Put repeatable helper code in `scripts/`.
- Put templates, images, and other reusable output files in `assets/`.
- Use lowercase hyphenated folder names, such as `powershell-safety`.

