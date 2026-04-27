# Changelog

All notable changes to `delegate-to-github-ai` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] — 2026-04-27

### Added

- Initial release. Single skill that helps a Claude Code session draft a GitHub issue and assign it to the **Copilot (GitHub)** or **Claude (Anthropic)** coding agent on a repo where those AI assignees are enabled.
- Pre-flight check (`gh api .../assignees`) so the agent verifies the bots are assignable before promising the user it'll work.
- "Keep it local vs. delegate" decision aid — flags tasks that need internal package feeds, VPN-only services, deploy targets, or a real browser, since the GitHub-hosted runner can't reach those.
- Copilot-vs-Claude heuristics for picking the assignee.
- Issue-body template with the sections AI assignees behave best on (Goal, Acceptance criteria, In-scope files, Out-of-scope, How to verify, Constraints, References).
- Follow-up etiquette: comment on the **draft PR** (not the issue), one coherent change per round, take it over locally after two unproductive rounds.
