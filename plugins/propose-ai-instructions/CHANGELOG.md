# Changelog

All notable changes to `propose-ai-instructions` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-04-23

### Added

- Initial release. Reverse complement to `sync-ai-instructions`: scans the current project for stack-specific conventions and proposes which entries are generic enough to be promoted upstream to `freaxnx01/ai-instructions`.
- Stack-agnostic architecture: per-stack heuristics are pluggable and live alongside the `SKILL.md` under `skills/propose-ai-instructions/stacks/<stack>.md`.
- v1 ships heuristics for the `dotnet` stack and the `makefile` category only. The skill stops with a clear message for any other stack until heuristics for that stack are added (planned: `flutter`).
- Conservative classification — biases toward `exclude` when unclear; a separate `intent_promotions` escape hatch promotes targets by **canonical name** when the recipe legitimately varies per host/tool/project (e.g. `run-edge`, `package`, `release-notes`).
- Output is a proposal file at `docs/ai-notes/upstream-proposal-<YYYY-MM-DD>.md` with a classification table, a ready-to-paste section for `stacks/<stack>.md`, an excluded list with reasons, and step-by-step PR instructions. Never commits or pushes upstream.
