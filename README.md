# agent-skills

This repository contains only skills for the libraries Thinktecture.Runtime.Extensions and Thinktecture.EntityFrameworkCore.

## Skills

| Skill | Library |
|-------|---------|
| `thinktecture-runtime-extensions` | [Thinktecture.Runtime.Extensions](https://github.com/PawelGerr/Thinktecture.Runtime.Extensions) |

## Install with the `skills` CLI

The `skills` CLI works with Claude Code, Cursor, and other agents. Install one skill into the current project:

```bash
npx skills@latest add PawelGerr/agent-skills --skill thinktecture-runtime-extensions
```

Add `-g` to install the skill for all your projects. Add `-a <agent>` (for example `-a claude-code`) to target one agent and `-y` to skip prompts. Run the same command again to update the skill.

## Install as a Claude Code plugin

```
/plugin marketplace add PawelGerr/agent-skills
/plugin install thinktecture-runtime-extensions@pawelgerr
```

## Layout

- Each library owns exactly one folder under `skills/` and exactly one entry in `.claude-plugin/marketplace.json`.
- The AI agent of each library repository edits only the folder and the marketplace entry of its own library.
- Both library repositories consume this repository as a git submodule at `agent-skills/`.
