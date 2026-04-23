---
stack: dotnet
category: makefile
upstream_file: .ai/stacks/dotnet.md
upstream_section_name: "## Essential Make Targets"
insert_after_section: "## Essential Commands"
insert_before_section: "## Docker"
---

# propose-ai-instructions — heuristics for `dotnet` / `makefile`

Machine-readable rules consumed by `SKILL.md` Step 4. All pattern fields are regex.

---

## generic_tools

Commands whose presence in a recipe does **not** disqualify a target. A target whose entire recipe consists only of these is a promotion candidate.

```yaml
generic_tools:
  # .NET
  - dotnet
  - dotnet ef
  - dotnet format
  - dotnet watch
  - dotnet test
  - dotnet build
  - dotnet run
  - dotnet restore
  - dotnet list
  - dotnet publish

  # Containers
  - docker
  - docker compose        # v2 plugin form
  - docker-compose        # legacy binary form

  # Git ecosystem
  - git
  - gh
  - git-cliff

  # Networking / probing
  - curl
  - wget

  # Shell built-ins and POSIX basics (no disqualification)
  - echo
  - printf
  - mkdir
  - rm
  - cp
  - mv
  - find
  - grep
  - sed
  - awk
  - cut
  - sort
  - uniq
  - head
  - tail
  - cat
  - sleep
  - test
  - pkill
  - until
  - if
  - for
  - while
  - case
  - $(MAKE)
```

`$(MAKE)` is allowed so composite targets (`release-auto: bump-auto release`) can be promoted when their dependencies are also promoted.

---

## project_markers

Any recipe containing **any** of these patterns is **excluded** with the matching reason.

```yaml
project_markers:
  # Host/OS specific
  - pattern: '\bpowershell\.exe\b'
    reason: Windows PowerShell invocation — host-specific
  - pattern: '\bpwsh\b'
    reason: PowerShell invocation — host-specific
  - pattern: '\bcmd\.exe\b'
    reason: Windows cmd invocation — host-specific
  - pattern: '/mnt/[a-z]/'
    reason: WSL Windows mount path
  - pattern: '\\\\wsl\$'
    reason: WSL UNC path
  - pattern: '\bmsedge\b|\bchrome\.exe\b|\bfirefox\.exe\b'
    reason: host-specific browser invocation

  # Databases not in the canonical .NET stack overlay (PostgreSQL, SQLite)
  - pattern: '(?i)\boracle\b'
    reason: Oracle DB — not in canonical .NET stack overlay
  - pattern: '(?i)\bmssql\b|\bsqlserver\b'
    reason: SQL Server — not in canonical .NET stack overlay
  - pattern: '(?i)\bmysql\b|\bmariadb\b'
    reason: MySQL/MariaDB — not in canonical .NET stack overlay

  # Registry / org / company specifics
  - pattern: '\bghcr\.io/[^/\s${}()]+/'
    reason: hardcoded GHCR owner — org-specific (use $(OWNER) variable instead)
  - pattern: '\bdocker\.io/[^/\s${}()]+/'
    reason: hardcoded Docker Hub owner — org-specific
  - pattern: '\bazurecr\.io'
    reason: hardcoded Azure Container Registry

  # Tooling that isn't universally available
  - pattern: '\bclaude\b'
    reason: Claude Code CLI invocation — depends on a specific agent installation
  - pattern: '\bcopilot\b'
    reason: Copilot CLI invocation — depends on a specific agent installation

  # Hardcoded endpoints / ports in critical paths (probing is fine when parametrized)
  - pattern: 'https?://(?!localhost|127\.0\.0\.1|0\.0\.0\.0)[a-z0-9.-]+\.(com|ch|net|io|dev|internal)'
    reason: hardcoded non-localhost URL
```

### Patterns that do NOT disqualify (explicit allowlist)

- `$(VAR)` — any Make variable reference. Variables are how projects parametrize; their presence is fine, their *hardcoded values* in the variable definitions are not our concern (we only inspect recipes).
- `localhost` / `127.0.0.1` / `0.0.0.0` in URLs — local-only, every project has some of these.
- Environment variable names that look project-specific (`InternalSettings__Foo`) — the name alone doesn't disqualify, but if it appears hardcoded in a recipe body (not a `$(VAR)`) it does. Flagged via `project_env_prefix` below.

```yaml
project_env_prefix:
  # Env-var names that, when literal in a recipe, mark the target as project-specific
  - pattern: '\bInternalSettings__'
    reason: project-specific app-internal configuration env var
  - pattern: '\bConnectionStrings__Default='
    reason: hardcoded connection string env — project-specific; if parametrized via $(VAR) this is fine
```

---

## skip_targets

Meta/help targets to skip without classifying.

```yaml
skip_targets:
  - help
  - show
  - open
  - list
  - print-%
```

---

## promotion_groups

How to group promoted targets in the output section. Order matters — this is the order that ends up in the upstream patch.

```yaml
promotion_groups:
  - name: "Build & run"
    targets: [build, watch, run]
  - name: "Testing"
    targets: [test, test-unit, test-coverage]
  - name: "Docker (Compose)"
    targets: [docker-run, up, down, logs, rebuild]
  - name: "Quality"
    targets: [lint, outdated, vuln]
  - name: "Versioning"
    targets: [version, version-set, bump-major, bump-minor, bump-patch, bump-auto]
    note: "single source of truth — Directory.Build.props → <Version>"
  - name: "Release"
    targets: [changelog, release, release-auto, push-release]
  - name: "Cleanup"
    targets: [clean]
```

Targets not in any group that are promoted anyway go into a trailing "Other" group.

---

## disqualification_priority

When a recipe matches both a `generic_tools` whitelist entry AND a `project_markers` pattern, `project_markers` wins. (Otherwise `docker compose -f $(ORACLE_DIR)/docker-compose.yml` would pass on "docker compose" and miss "oracle".)

---

## intent_promotions

Targets promoted by **name and intent**, not by recipe content. Upstream documents the canonical target name; the recipe body is left to each project (host-specific, tool-specific, or destination-specific).

This is the escape hatch for targets that every .NET project benefits from having — but whose implementation legitimately varies. Without it, every recipe with a `powershell.exe` or `claude` invocation would be excluded, and the canonical Makefile would be missing genuinely standard verbs.

```yaml
intent_promotions:
  - target: run-edge
    intent: "Start the frontend and open it in the developer's default/preferred browser (host-specific implementation)"
    group: "Build & run"
    upstream_note: "Recipe is host-specific (Windows/WSL: powershell + msedge; macOS: open -a Safari; Linux: xdg-open). Standardize the target name; leave the recipe to each project."

  - target: package
    intent: "Build a distributable deployment artifact (ZIP, tarball, container image, etc.) and deliver it to the project's drop location"
    group: "Release"
    upstream_note: "Recipe is project-specific (artifact format, drop location, signing). Standardize the target name; leave the body to each project."

  - target: release-notes
    intent: "Generate user-friendly release notes for the current version using an AI/LLM tool"
    group: "Release"
    upstream_note: "Recipe is tool-specific (Claude Code, Copilot CLI, llm CLI, OpenAI, hand-rolled). Standardize the target name; leave the body to each project."
```

The skill emits these in the proposal under their assigned group with the `upstream_note` rendered as italics under the bullet, so reviewers see *why* the recipe isn't prescribed.

---

## notes for maintainers

- Heuristics are deliberately conservative. If you find the skill excluding something that should promote, **widen `generic_tools`** rather than narrowing `project_markers` — a looser marker risks polluting every downstream project.
- Patterns are anchored with `\b` word boundaries where possible to avoid substring false positives (`oracle` vs. `oracles`).
- `i` inline flag (`(?i)`) for case-insensitive when the match space is text inside descriptions, not command names.
