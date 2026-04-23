---
name: propose-ai-instructions
description: Reverse of /sync-ai-instructions. Analyze the current project for stack-specific conventions (currently: Makefile targets for the dotnet stack) and propose which entries are generic enough to be promoted upstream to github.com/freaxnx01/ai-instructions. Writes a reviewable proposal file with a classified diff; never commits or pushes upstream. Use when the user wants to harvest local conventions that have proven useful and could benefit other projects using the same stack overlay. Triggers include "propose ai-instructions", "upstream my Makefile", "what should go upstream", or running /propose-ai-instructions. Run from the target project's working directory.
---

# Propose AI Instructions — Harvest local conventions for upstream promotion

Scan this project for stack-specific conventions that aren't yet documented in the canonical `github.com/freaxnx01/ai-instructions` repo, classify each finding as **generic** (worth promoting) or **project-specific** (stay local), and produce a reviewable patch proposal.

This is the **reverse** of `/sync-ai-instructions`:

```
/sync-ai-instructions     upstream  →  project   (pull)
/propose-ai-instructions  project   →  upstream  (propose — never push)
```

The skill only produces a proposal. The user reviews it, then opens a PR to `ai-instructions` manually (or approves a follow-up action).

**Run this skill from the target project's working directory**, not from the `ai-instructions` source repo itself.

**Target:** $ARGUMENTS  (optional: category to scan — currently only `makefile` is supported; defaults to `makefile`)

---

## Scope & extensibility

**Architecture is stack-agnostic. v1 ships heuristics for one stack only.**

- **Stack resolution:** read `.ai/stacks/*.md` and take the single basename (same rule as `/sync-ai-instructions`).
- **Per-stack heuristics:** each supported stack has its own rules file bundled *inside this plugin*, under `skills/propose-ai-instructions/stacks/<stack>.md`. The rules describe what counts as a generic tool, what markers make a target project-specific, and what sections of the upstream stack overlay the proposal targets.
- **Supported stacks (v1):** `dotnet` only.
- **Planned stacks:** `flutter` (harvest `pubspec.yaml` conventions, `melos` scripts, `fvm` usage, standard `flutter`/`dart`/`melos` command wrappers in a repo-level script runner). Heuristics file lands in the plugin when the Flutter overlay itself exists upstream.
- **If the detected stack has no heuristics file in the plugin → stop and report.** Do not guess heuristics from another stack.
- **Category (v1):** `makefile` only. Future categories (scripts, CI workflows, `Directory.Build.props` / `pubspec.yaml`, `docker-compose*.yml`) follow the same pattern but are out of scope for v1.

---

## Steps

### Step 1 — Detect stack and load heuristics

1. Read `.ai/stacks/*.md` in the target project.
   - Zero or more than one file → stop and report (same rule as `/sync-ai-instructions`).
2. Look up `skills/propose-ai-instructions/stacks/<stack>.md` inside this plugin.
   - Not found → stop and report: "Heuristics for stack `<stack>` are not yet implemented in `propose-ai-instructions`. v1 supports: dotnet."
3. Load the stack's heuristics (generic-tools allowlist, project-specific markers, upstream target section name, intent-promotion overrides).

### Step 2 — Fetch the current upstream stack overlay

Fetch from `main`:

- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/.ai/stacks/<stack>.md`

Also record the upstream commit SHA for the report:

```
gh api repos/freaxnx01/ai-instructions/commits/main --jq .sha
```

If the fetch fails, stop — do not operate on stale copies.

### Step 3 — Parse the artifact (v1: Makefile)

Dispatch by category. For `makefile`:

- Require a `Makefile` at the repo root; stop if missing.
- Extract every target of the form `^[a-zA-Z_-]+:` and its inline help comment (`## ...`), plus the recipe body (the indented lines until the next blank line or next target).
- Skip `.PHONY`, `.DEFAULT_GOAL`, and variables.
- Ignore help/meta targets (`help`, `show`, `open`, anything whose recipe only prints or opens files) — not worth promoting.

Future categories (`script`, `ci`, `compose`, etc.) parse their own artifact formats. Same pipeline from Step 4 onwards.

### Step 4 — Classify each target

For each target, decide **promote** (worth upstreaming) / **exclude** (stay local) / **review** (borderline, user decides).

The per-stack heuristics file loaded in Step 1 provides:

- `generic_tools`: allowlist of commands that, when used alone, don't disqualify a target (e.g. dotnet: `dotnet`, `docker`, `docker compose`, `git`, `gh`, `git-cliff`, `curl`, POSIX basics).
- `project_markers`: patterns that disqualify a target (hardcoded repo names, company-internal URLs, host-specific tooling, DB engines not in the upstream stack overlay, Windows/UNC paths, etc.).
- `intent_promotions`: escape hatch — targets promoted by **canonical name and intent** even when their recipe contains project-specific tooling. Each entry carries an `upstream_note` rendered as italics under the proposal bullet.
- `upstream_section_name`: where the proposed patch should land (e.g. `## Essential Make Targets` in `stacks/dotnet.md`).

Apply in order:

1. Target name appears in `intent_promotions` → **promote** with `upstream_note`. Do not inspect the recipe further.
2. Recipe uses only commands in `generic_tools` and contains no `project_markers` → **promote**.
3. Recipe contains any `project_markers` → **exclude** with the matching reason.
4. Recipe uses tools outside the allowlist but all such references could be parametrized (`$(VAR)`) cleanly → **review**.

**Bias:** when unclear, **exclude**. A false-positive promotion pollutes every downstream project on the same stack; a false-negative exclusion is a one-line fix next run.

For each decision, record: **target name**, **verdict**, **one-line reason**, **source line number**.

### Step 5 — Build the proposal, don't just diff

The proposal is **a whole new section** (e.g. `## Essential Make Targets`) that standardizes target names for the stack, not a cherry-picked list of "commands upstream is missing". Keep every `promote` target in the proposal even if the underlying command is already listed in the upstream `## Essential Commands` — the value is the standardized target name, not the novelty of the command.

Only drop a target from the proposal if:

- Its recipe is a literal shell alias for a single already-documented command AND
- The target name provides no added convention (e.g. a target called `pwd-where-am-i` wrapping `pwd`).

In practice, almost everything in the `promote` set survives.

### Step 6 — Write the proposal

Write `docs/ai-notes/upstream-proposal-<YYYY-MM-DD>.md` in the target project with:

1. **Header** — upstream SHA fetched, stack, date, repo name, branch.
2. **Summary table** — target, verdict, reason, source Makefile line number.
3. **Proposed patch** — a ready-to-paste Markdown section titled per the stack heuristics' `upstream_section_name` (e.g. `## Essential Make Targets` for `dotnet`) for `stacks/<stack>.md`, listing each promoted target with its help string, grouped by theme (build/run, test, release, docker, security, versioning) per the heuristics' `promotion_groups`. Use the same tone and formatting as the existing `## Essential Commands` section so it fits next to it. For intent-promoted targets, render the `upstream_note` as italics under the bullet.
4. **Excluded items** — full list with reason, so the user can override any misclassification.
5. **How to land this** — step-by-step: clone `freaxnx01/ai-instructions`, create branch `feat/<stack>-make-targets` (or equivalent for the category), paste the proposed section into `.ai/stacks/<stack>.md` under `## Essential Commands`, commit with Conventional Commits, open PR.

### Step 7 — Report

Print:

- Stack detected
- Makefile targets scanned (count) / promoted (count) / excluded (count) / review (count)
- Path to the proposal file
- Upstream SHA and the exact PR command (`gh pr create ...`) pre-filled

Do **not** commit the proposal file automatically — leave that to the user.

---

## Rules

- Never modify files outside the current working directory.
- Never push, commit, or open a PR against `ai-instructions` — the skill produces a proposal only.
- Never overwrite an existing proposal file without showing a diff first.
- If the Makefile is missing, the stack isn't supported, or the upstream fetch fails — stop and report. Do not fall back to partial output.
- Bias toward **exclude** in classification. A false-positive promotion pollutes every downstream consumer; a false-negative exclusion is a one-line fix in the next run.
- Keep the proposed patch formatted to match the existing `stacks/<stack>.md` style — same heading levels, same code-fence language tags, same tone.

---

## Future extensions (out of scope for v1)

- **Stack `flutter`** — add `skills/propose-ai-instructions/stacks/flutter.md` describing: generic tools (`flutter`, `dart`, `fvm`, `melos`, `git`, `gh`, `git-cliff`); project markers (hardcoded flavor names, company plugin pins, specific `pubspec.yaml` dependency URLs); upstream section name (`## Essential Melos Scripts` or similar). Only activate once `stacks/flutter.md` exists in the ai-instructions repo.
- **Category `scripts`** — harvest `scripts/*.sh|.ps1|.psm1` conventions.
- **Category `ci`** — harvest `.github/workflows/*.yml` patterns.
- **Category `compose`** — harvest `docker-compose*.yml` patterns (healthchecks, non-root users, named volumes).
- **Category `editor`** — harvest `.editorconfig` / `Directory.Build.props` / `analysis_options.yaml` settings that diverge from upstream defaults.
- **Auto-PR mode** — after user approval, open the PR to `ai-instructions` via `gh` from a local clone if one exists.
