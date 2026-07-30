# agent-skills

Public Claude Code plugin marketplace by [freaxnx01](https://github.com/freaxnx01) — sharable, non-personal skills. Separate from personal slash commands (see [`config`](https://github.com/freaxnx01/config) + [`agent-pipeline`](https://github.com/freaxnx01/agent-pipeline)).

Marketplace name inside `.claude-plugin/marketplace.json`: **`freax-agent-skills`** — use this suffix when installing or updating plugins. (The plain name `agent-skills` is reserved by Anthropic for their official marketplaces.)

## Not the same as personal commands

This repo ships **Claude Code plugins**, installed per-machine via `/plugin marketplace add` + `/plugin install` (below). That's a different mechanism from the personal `~/.claude/commands/` slash commands, which come from `freaxnx01/config` and `freaxnx01/agent-pipeline` and are synced by symlinking, not by the plugin system — see those repos' `setup/` scripts (`01-claude-commands.sh`, `link-commands.sh`).

## Plugins

| Plugin | Description |
|---|---|
| [`sync-ai-instructions`](./plugins/sync-ai-instructions) | Initialize or update a project's AI agent instruction files (`CLAUDE.md`, Copilot, `SKILL.md`, `.ai/*`) from [`freaxnx01/ai-instructions`](https://github.com/freaxnx01/ai-instructions). Idempotent — safe for first-time setup and for re-runs to keep files in sync. |
| [`propose-ai-instructions`](./plugins/propose-ai-instructions) | Reverse direction. Scans the current project (v1: `justfile` recipes for the `dotnet` stack), classifies entries as generic-enough-to-promote vs. project-specific, and writes a reviewable proposal patch for [`freaxnx01/ai-instructions`](https://github.com/freaxnx01/ai-instructions). Never commits or pushes upstream. |
| [`release-notes`](./plugins/release-notes) | Generate user-friendly `RELEASENOTES.md` entries from git tags and commit history, grouped by "New Features" / "Improvements" / "Bug Fixes". Append-only; distinct from the developer-facing, Conventional-Commit-generated `CHANGELOG.md`. |
| [`hermes-tweet`](./plugins/hermes-tweet) | Guide Claude Code users operating Hermes Agent through read-first X/Twitter research and approval-gated actions with Hermes Tweet. |

## Install

```bash
/plugin marketplace add freaxnx01/agent-skills
/plugin install sync-ai-instructions@freax-agent-skills
/plugin install propose-ai-instructions@freax-agent-skills
/plugin install release-notes@freax-agent-skills
/plugin install hermes-tweet@freax-agent-skills
/reload-plugins
```

## Update

```bash
/plugin marketplace update freax-agent-skills
/plugin update sync-ai-instructions@freax-agent-skills
/plugin update propose-ai-instructions@freax-agent-skills
/plugin update release-notes@freax-agent-skills
/plugin update hermes-tweet@freax-agent-skills
/reload-plugins
```

## Why separate from `freaxnx01/claude-code-plugins`?

`freaxnx01/claude-code-plugins` is private — it holds personal plugins (homelab routing, NAS sync, espanso expansions, etc.) that are specific to one setup and not useful to others. This repo (`agent-skills`) is the public sibling for plugins that anyone can benefit from, so teammates and the wider community can install without needing collaborator access to a private repo.
