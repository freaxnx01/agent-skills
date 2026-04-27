# agent-skills

Public Claude Code plugin marketplace by [freaxnx01](https://github.com/freaxnx01) — sharable, non-personal skills.

Marketplace name inside `.claude-plugin/marketplace.json`: **`freax-agent-skills`** — use this suffix when installing or updating plugins. (The plain name `agent-skills` is reserved by Anthropic for their official marketplaces.)

## Plugins

| Plugin | Description |
|---|---|
| [`sync-ai-instructions`](./plugins/sync-ai-instructions) | Initialize or update a project's AI agent instruction files (`CLAUDE.md`, Copilot, `SKILL.md`, `.ai/*`) from [`freaxnx01/ai-instructions`](https://github.com/freaxnx01/ai-instructions). Idempotent — safe for first-time setup and for re-runs to keep files in sync. |
| [`propose-ai-instructions`](./plugins/propose-ai-instructions) | Reverse direction. Scans the current project (v1: `Makefile` for the `dotnet` stack), classifies entries as generic-enough-to-promote vs. project-specific, and writes a reviewable proposal patch for [`freaxnx01/ai-instructions`](https://github.com/freaxnx01/ai-instructions). Never commits or pushes upstream. |
| [`delegate-to-github-ai`](./plugins/delegate-to-github-ai) | Hand a coding task off to a GitHub AI assignee (**Copilot** or **Claude**) by creating a self-contained issue brief and assigning it to the right bot. Pre-flight check, delegate-vs-keep-local decision aid, the issue-body template AI agents behave best on, the exact `gh issue create --assignee` invocation, and follow-up etiquette for the draft PR. |

## Install

```bash
/plugin marketplace add freaxnx01/agent-skills
/plugin install sync-ai-instructions@freax-agent-skills
/plugin install propose-ai-instructions@freax-agent-skills
/plugin install delegate-to-github-ai@freax-agent-skills
/reload-plugins
```

## Update

```bash
/plugin marketplace update freax-agent-skills
/plugin update sync-ai-instructions@freax-agent-skills
/plugin update propose-ai-instructions@freax-agent-skills
/plugin update delegate-to-github-ai@freax-agent-skills
/reload-plugins
```

## Why separate from `freaxnx01/claude-code-plugins`?

`freaxnx01/claude-code-plugins` is private — it holds personal plugins (homelab routing, NAS sync, espanso expansions, etc.) that are specific to one setup and not useful to others. This repo (`agent-skills`) is the public sibling for plugins that anyone can benefit from, so teammates and the wider community can install without needing collaborator access to a private repo.
