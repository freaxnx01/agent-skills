# Changelog

All notable changes to `propose-ai-instructions` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] — 2026-07-29

### Added

- `SKILL.md` Step 5 — transitive dependency check. After per-recipe classification, the skill now walks the recipe dependency graph (justfile native prerequisites and explicit `just <name>` invocations in recipe bodies). A `promote` recipe that depends — directly or transitively — on an `exclude` recipe is downgraded to `review` with reason `transitive dep on excluded <name>`, instead of being promoted upstream with a body that has no meaningful generic recipe underneath it.

## [0.2.0] — 2026-05-23

### Changed

- **Breaking:** category renamed from `makefile` to `justfile`. Upstream `freaxnx01/ai-instructions` migrated from GNU Make to [casey/just](https://github.com/casey/just). The skill now parses a repo-root `justfile` instead of `Makefile`. Projects that still ship a `Makefile` are asked to migrate first; the skill stops and reports rather than attempting Makefile parsing.
- `stacks/dotnet.md` heuristics now target `.ai/stacks/_partials/dotnet-core.md` (the canonical source — flat `dotnet-blazor.md` / `dotnet-webapi.md` overlays are regenerated from it). Section heading promotes to `## Essential just Recipes`.
- `SKILL.md` Step 1 now strips a `-<flavour>` suffix from the detected stack name when resolving the heuristics file, so `dotnet-blazor` and `dotnet-webapi` both fall back to `stacks/dotnet.md` (the two flavours share one heuristics file).
- `SKILL.md` Step 3 rewritten for justfile syntax: recognizes `# <doc>` comments above recipe headers, parses recipe attributes (`[unix]`, `[windows]`, `[group(...)]`, `[private]`, …), and collapses OS-split recipe pairs (`[unix] foo` + `[windows] foo`) into one logical promotable unit.
- `project_markers` for `pwsh` / `powershell.exe` / `msedge` / `xdg-open` / `open -a Safari` are now **context-sensitive** — they no longer disqualify when the recipe carries an `[unix]`/`[windows]`/`[macos]`/`[linux]` attribute. Outside an OS-attributed recipe they still disqualify.
- `intent_promotions` notes rewritten to describe the recommended `[unix]`/`[windows]` recipe-pair pattern instead of apologizing for Make's host-specific limitations (which justfile resolves natively).

### Added

- `windows_tools` conditional allowlist in `stacks/dotnet.md` — PowerShell built-ins (`Get-Content`, `Set-Content`, `Start-Process`, `[xml]`, `[version]`, `$ErrorActionPreference`, …) are allowed inside `[windows]`-attributed recipes.
- `just` added to the `generic_tools` allowlist (recipes can invoke other recipes via `just <name>`, e.g. inside the canonical `release` recipe).

### Removed

- `$(MAKE)` from `generic_tools` (justfile uses native recipe dependencies, e.g. `release-auto: bump-auto release`).
- `help` from `skip_targets` (now `skip_recipes`); the justfile convention is `default: @just --list`, so `default` is added to the skip list.

## [0.1.0] — 2026-04-23

### Added

- Initial release. Reverse complement to `sync-ai-instructions`: scans the current project for stack-specific conventions and proposes which entries are generic enough to be promoted upstream to `freaxnx01/ai-instructions`.
- Stack-agnostic architecture: per-stack heuristics are pluggable and live alongside the `SKILL.md` under `skills/propose-ai-instructions/stacks/<stack>.md`.
- v1 ships heuristics for the `dotnet` stack and the `makefile` category only. The skill stops with a clear message for any other stack until heuristics for that stack are added (planned: `flutter`).
- Conservative classification — biases toward `exclude` when unclear; a separate `intent_promotions` escape hatch promotes targets by **canonical name** when the recipe legitimately varies per host/tool/project (e.g. `run-edge`, `package`, `release-notes`).
- Output is a proposal file at `docs/ai-notes/upstream-proposal-<YYYY-MM-DD>.md` with a classification table, a ready-to-paste section for `stacks/<stack>.md`, an excluded list with reasons, and step-by-step PR instructions. Never commits or pushes upstream.
