# Changelog

All notable changes to `sync-ai-instructions` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.1] — 2026-05-01

### Fixed

- The available-stacks listing (used in the no-argument flow and to validate `$ARGUMENTS`) now filters out directory entries and underscore-prefixed files. Previously, after the `ai-instructions` repo gained `.ai/stacks/_partials/` and `.ai/stacks/_layers/` for stack composition, those directories appeared in the list as if they were selectable stacks. The new `jq` filter keeps only `*.md` files whose names don't start with `_`, and strips the `.md` suffix so output is the bare stack name (e.g. `dotnet-webapi`).

## [0.2.0] — 2026-04-20

### Changed

- **Renamed from `init-instructions` to `sync-ai-instructions`.** The old name (`init-*`) implied "first-time only" (cf. `git init`, `npm init`), but this skill is idempotent and is meant to be re-run to keep a project's AI instruction files aligned with the source-of-truth template. `sync-*` better reflects the create-or-update semantics.
- **Moved from private `freaxnx01/claude-code-plugins` marketplace to new public `freaxnx01/agent-skills` marketplace.** The plugin only consumes the already-public `freaxnx01/ai-instructions` template repo, so nothing personal leaks by making it public. Teammates and the wider community can now install it without needing collaborator access to a private repo.

### Migration

If you had the old plugin installed:

```
/plugin uninstall init-instructions@freax-claude-code-plugins
/plugin marketplace add freaxnx01/agent-skills
/plugin install sync-ai-instructions@freax-agent-skills
/reload-plugins
```

The slash command trigger changes from `/init-instructions` → `/sync-ai-instructions`.

## [0.1.0] — 2026-04-19

### Added

- Initial release (as `init-instructions` in `freaxnx01/claude-code-plugins`). Promoted from `freaxnx01/ai-instructions` (`.claude/commands/init-instructions.md` + `.ai/skills/init-instructions.md`) into the marketplace as a standalone plugin. Pulls `base-instructions.md` + one `stacks/<stack>.md` overlay + shared `skills/*.md` from the source repo and assembles `CLAUDE.md`, `.github/copilot-instructions.md`, `SKILL.md` in the target project.

### Changed

- Target projects no longer receive a copy of `init-instructions.md` in `.claude/commands/` — the plugin is the single source of truth and is available globally once installed from the marketplace.

### Removed

- Original `.claude/commands/init-instructions.md` and `.ai/skills/init-instructions.md` deleted from `freaxnx01/ai-instructions`; that repo now only holds the base + stacks + shared skills that the plugin consumes.
- Stopped fetching `release-notes.md` into target projects. `release-notes` is its own plugin; target projects no longer need a local copy. `sync-ai-instructions` cleans up any stale `.ai/skills/release-notes.md` / `.claude/commands/release-notes.md` it finds on an update run.
