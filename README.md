# agent-skills

Public Claude Code plugin marketplace by [freaxnx01](https://github.com/freaxnx01) — sharable, non-personal skills.

Marketplace name inside `.claude-plugin/marketplace.json`: **`agent-skills`** — use this suffix when installing or updating plugins.

## Plugins

| Plugin | Description |
|---|---|
| [`sync-ai-instructions`](./plugins/sync-ai-instructions) | Initialize or update a project's AI agent instruction files (`CLAUDE.md`, Copilot, `SKILL.md`, `.ai/*`) from [`freaxnx01/ai-instructions`](https://github.com/freaxnx01/ai-instructions). Idempotent — safe for first-time setup and for re-runs to keep files in sync. |

## Install

```bash
/plugin marketplace add freaxnx01/agent-skills
/plugin install sync-ai-instructions@agent-skills
/reload-plugins
```

## Update

```bash
/plugin marketplace update agent-skills
/plugin update sync-ai-instructions@agent-skills
/reload-plugins
```

## Why separate from `freaxnx01/claude-code-plugins`?

`freaxnx01/claude-code-plugins` is private — it holds personal plugins (homelab routing, NAS sync, espanso expansions, etc.) that are specific to one setup and not useful to others. This repo (`agent-skills`) is the public sibling for plugins that anyone can benefit from, so teammates and the wider community can install without needing collaborator access to a private repo.
