---
name: propose-ai-instructions
description: Reverse of /sync-ai-instructions. Analyze the current project for stack-specific conventions (currently: justfile recipes for the dotnet stack) and propose which entries are generic enough to be promoted upstream to github.com/freaxnx01/ai-instructions. Writes a reviewable proposal file with a classified diff; never commits or pushes upstream. Use when the user wants to harvest local conventions that have proven useful and could benefit other projects using the same stack overlay. Triggers include "propose ai-instructions", "upstream my justfile", "what should go upstream", or running /propose-ai-instructions. Run from the target project's working directory.
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

**Target:** $ARGUMENTS  (optional: category to scan — currently only `justfile` is supported; defaults to `justfile`)

---

## Scope & extensibility

**Architecture is stack-agnostic. v1 ships heuristics for one stack only.**

- **Stack resolution:** read `.ai/stacks/*.md` and take the single basename (same rule as `/sync-ai-instructions`).
- **Per-stack heuristics:** each supported stack has its own rules file bundled *inside this plugin*, under `skills/propose-ai-instructions/stacks/<stack>.md`. The rules describe what counts as a generic tool, what markers make a recipe project-specific, and what sections of the upstream stack overlay the proposal targets.
- **Supported stacks (v1):** `dotnet` only. The same `stacks/dotnet.md` heuristics apply to all `dotnet-*` flavours (e.g. `dotnet-blazor`, `dotnet-webapi`) — see Step 1 for the lookup rule.
- **Planned stacks:** `flutter` (harvest `pubspec.yaml` conventions, `melos` scripts, `fvm` usage, standard `flutter`/`dart`/`melos` command wrappers in a repo-level script runner). Heuristics file lands in the plugin when the Flutter overlay itself ships a canonical justfile upstream.
- **If the detected stack has no heuristics file in the plugin → stop and report.** Do not guess heuristics from another stack.
- **Category (v1):** `justfile` only. Future categories (scripts, CI workflows, `Directory.Build.props` / `pubspec.yaml`, `docker-compose*.yml`) follow the same pattern but are out of scope for v1.

---

## Steps

### Step 1 — Detect stack and load heuristics

1. Read `.ai/stacks/*.md` in the target project.
   - Zero or more than one file → stop and report (same rule as `/sync-ai-instructions`).
2. Resolve heuristics file in this plugin:
   1. Try `skills/propose-ai-instructions/stacks/<stack>.md` (exact match).
   2. If not found and the stack name contains a `-`, strip the suffix and try `skills/propose-ai-instructions/stacks/<base>.md` (e.g. `dotnet-blazor` → `dotnet.md`, `dotnet-webapi` → `dotnet.md`). The dotnet flavours share one heuristics file.
   3. Still not found → stop and report: "Heuristics for stack `<stack>` are not yet implemented in `propose-ai-instructions`. v1 supports: dotnet (incl. dotnet-blazor, dotnet-webapi)."
3. Load the stack's heuristics (generic-tools allowlist, project-specific markers, upstream target section name, intent-promotion overrides).

### Step 2 — Fetch the current upstream stack overlay

The heuristics file's frontmatter `upstream_file` field declares the path to fetch. For the dotnet stack this is `.ai/stacks/_partials/dotnet-core.md` (the canonical source — flat `dotnet-blazor.md` / `dotnet-webapi.md` are regenerated from the partial and a layer file).

Fetch from `main`:

- `https://raw.githubusercontent.com/freaxnx01/ai-instructions/main/<upstream_file>`

Also record the upstream commit SHA for the report:

```
gh api repos/freaxnx01/ai-instructions/commits/main --jq .sha
```

If the fetch fails, stop — do not operate on stale copies.

### Step 3 — Parse the artifact (v1: justfile)

Dispatch by category. For `justfile`:

- Require a `justfile` (or `Justfile`) at the repo root; stop if missing.
  - If only a legacy `Makefile` is present, stop and report: "This project still uses GNU Make. Upstream `freaxnx01/ai-instructions` migrated to `just` — migrate this project's Makefile to a justfile (see `.ai/examples/dotnet/justfile` upstream) before running this skill."
- Extract every recipe header of the form `^[a-zA-Z_-]+(\s+[a-zA-Z_]+(="[^"]*")?)*:` (recipe name, then zero or more positional parameters with optional defaults).
- Capture the doc comment: the `# <description>` line(s) immediately above the recipe header (justfile convention — note the single `#`, unlike Make's `##`).
- Capture recipe attributes appearing on the lines above the header: `[unix]`, `[windows]`, `[macos]`, `[linux]`, `[group('...')]`, `[private]`, `[no-cd]`, `[confirm(...)]`, etc.
- Capture the recipe body: the indented lines until the next blank line, the next attribute block, or the next recipe header.
- Skip:
  - Variable assignments (`name := "value"`)
  - `set <option>` directives
  - The `default` recipe (justfile convention — usually wraps `just --list`)
  - Recipes with the `[private]` attribute
- **Collapse OS variants of the same recipe name into one logical entry.** A recipe shipped as `[unix] foo:` + `[windows] foo:` is one promotable unit (the canonical recipe `foo`); record both bodies on the same entry for classification purposes. The proposal lists `foo` once.

Future categories (`script`, `ci`, `compose`, etc.) parse their own artifact formats. Same pipeline from Step 4 onwards.

### Step 4 — Classify each recipe

For each recipe, decide **promote** (worth upstreaming) / **exclude** (stay local) / **review** (borderline, user decides).

The per-stack heuristics file loaded in Step 1 provides:

- `generic_tools`: allowlist of commands that, when used alone, don't disqualify a recipe (e.g. dotnet: `dotnet`, `docker`, `docker compose`, `git`, `gh`, `git-cliff`, `curl`, `just`, POSIX basics; plus PowerShell built-ins when the recipe carries a `[windows]` attribute).
- `project_markers`: patterns that disqualify a recipe (hardcoded repo names, company-internal URLs, host-specific glue paths, DB engines not in the upstream stack overlay, etc.). Some markers are **context-sensitive** — e.g. `pwsh` / `msedge` are legitimate inside a `[windows]`-attributed recipe but disqualifying outside one.
- `intent_promotions`: escape hatch — recipes promoted by **canonical name and intent** even when their body contains project-specific tooling. Each entry carries an `upstream_note` rendered as italics under the proposal bullet.
- `upstream_section_name`: where the proposed patch should land (e.g. `## Essential just Recipes` in the upstream partial).

Apply in order:

1. Recipe name appears in `intent_promotions` → **promote** with `upstream_note`. Do not inspect the body further.
2. Recipe body uses only commands in `generic_tools` (respecting OS-attribute context for the conditional entries) and contains no `project_markers` → **promote**.
3. Recipe body contains any `project_markers` (in a disqualifying context) → **exclude** with the matching reason.
4. Recipe body uses tools outside the allowlist but all such references could be parametrized (`{{var}}`) cleanly → **review**.

For OS-split recipes (`[unix]` + `[windows]` of the same name), evaluate each body, then take the **more permissive** verdict:
- both promote → promote
- one promote + one exclude → review (the user decides whether the offending OS variant should be excluded or rewritten)
- both exclude → exclude

**Bias:** when unclear, **exclude**. A false-positive promotion pollutes every downstream project on the same stack; a false-negative exclusion is a one-line fix next run.

For each decision, record: **recipe name**, **verdict**, **one-line reason**, **source line number(s)** (one per OS variant if applicable).

### Step 5 — Transitive dependency check

Step 4 classifies each recipe from its own body alone, which misses recipes that only *compose* other recipes. Build the recipe dependency graph before finalizing verdicts:

- **Native prerequisites** — the recipe header's dependency list, e.g. `run: stop` (recipe `run` depends on `stop`).
- **Explicit invocations** — `just <name>` calls inside a recipe body.

For every recipe classified `promote` in Step 4, walk its dependencies (direct and transitive). Track visited recipes while walking to avoid infinite loops on cyclic references (e.g. `A -> B -> A`). If it depends on any recipe classified `exclude`, **downgrade the verdict to `review`** with reason `transitive dep on excluded <name>` (`<name>` is the nearest excluded dependency found). Leave `exclude` and `review` verdicts from Step 4 untouched.

Record downgrades the same way as any other verdict (recipe name, verdict, reason, source line number(s)).

### Step 6 — Build the proposal, don't just diff

The proposal is **a whole new section** (e.g. `## Essential just Recipes`) that standardizes recipe names for the stack, not a cherry-picked list of "commands upstream is missing". Keep every `promote` recipe in the proposal even if the underlying command is already listed in the upstream `## Essential Commands` — the value is the standardized recipe name, not the novelty of the command.

Only drop a recipe from the proposal if:

- Its body is a literal shell alias for a single already-documented command AND
- The recipe name provides no added convention (e.g. a recipe called `pwd-where-am-i` wrapping `pwd`).

In practice, almost everything in the `promote` set survives.

### Step 7 — Write the proposal

Write `docs/ai-notes/upstream-proposal-<YYYY-MM-DD>.md` in the target project with:

1. **Header** — upstream SHA fetched, stack, date, repo name, branch.
2. **Summary table** — recipe, verdict, reason, source justfile line number(s).
3. **Proposed patch** — a ready-to-paste Markdown section titled per the stack heuristics' `upstream_section_name` (e.g. `## Essential just Recipes` for `dotnet`) for the path declared in `upstream_file`, listing each promoted recipe with its doc-comment string, grouped by theme (build/run, test, release, docker, security, versioning) per the heuristics' `promotion_groups`. Use the same tone and formatting as the existing `## Essential Commands` section so it fits next to it. For intent-promoted recipes, render the `upstream_note` as italics under the bullet.
4. **Excluded items** — full list with reason, so the user can override any misclassification.
5. **How to land this** — step-by-step: clone `freaxnx01/ai-instructions`, create branch `feat/<stack>-just-recipes` (or equivalent for the category), paste the proposed section into the `upstream_file` path under `## Essential Commands`, run `./scripts/build-stacks.sh` if the file is a `_partials/*.md` source, commit with Conventional Commits, open PR.

### Step 8 — Report

Print:

- Stack detected
- justfile recipes scanned (count) / promoted (count) / excluded (count) / review (count)
- Path to the proposal file
- Upstream SHA and the exact PR command (`gh pr create ...`) pre-filled

Do **not** commit the proposal file automatically — leave that to the user.

---

## Rules

- Never modify files outside the current working directory.
- Never push, commit, or open a PR against `ai-instructions` — the skill produces a proposal only.
- Never overwrite an existing proposal file without showing a diff first.
- If the justfile is missing, the stack isn't supported, or the upstream fetch fails — stop and report. Do not fall back to partial output. If only a legacy `Makefile` is present, instruct the user to migrate first; do not attempt to translate Makefile syntax.
- Bias toward **exclude** in classification. A false-positive promotion pollutes every downstream consumer; a false-negative exclusion is a one-line fix in the next run.
- Keep the proposed patch formatted to match the existing upstream stack overlay style — same heading levels, same code-fence language tags, same tone.

---

## Future extensions (out of scope for v1)

- **Stack `flutter`** — add `skills/propose-ai-instructions/stacks/flutter.md` describing: generic tools (`flutter`, `dart`, `fvm`, `melos`, `just`, `git`, `gh`, `git-cliff`); project markers (hardcoded flavor names, company plugin pins, specific `pubspec.yaml` dependency URLs); upstream section name (`## Essential just Recipes` or similar). Activate once `stacks/flutter.md` upstream documents a canonical justfile (the overlay currently references `apk` / `windows` recipes — heuristics file can land alongside).
- **Category `scripts`** — harvest `scripts/*.sh|.ps1|.psm1` conventions.
- **Category `ci`** — harvest `.github/workflows/*.yml` patterns.
- **Category `compose`** — harvest `docker-compose*.yml` patterns (healthchecks, non-root users, named volumes).
- **Category `editor`** — harvest `.editorconfig` / `Directory.Build.props` / `analysis_options.yaml` settings that diverge from upstream defaults.
- **Auto-PR mode** — after user approval, open the PR to `ai-instructions` via `gh` from a local clone if one exists.
