# xrobot-workflow

Agent skill for building and maintaining [XRobot](https://xrobot.work)/LibXR embedded modules: Python CLI-driven module generation, dependency/configuration setup, generated entry points, and lifecycle-safe C++ module implementations.

## Files

- `SKILL.md` — the skill: operating rules, typical workflow, module argument source of truth, review checklist.
- `references/cli-and-module-contract.md` — exact CLI commands, files, and metadata contract.
- `agents/openai.yaml` — agent interface metadata (display name, default prompt).

## Install

Copy or symlink this directory into any agent's skills directory, keeping the folder name `xrobot-workflow`:

- **OpenCode**: `~/.cache/opencode/skills/`
- **Codex CLI**: `~/.codex/skills/`
- **pi**: `~/.agents/skills/`

Each agent picks up the skill from its `SKILL.md` frontmatter (`name`, `description`).
