---
name: delegate-to-github-ai
description: Use when the user wants to hand a coding task off to a GitHub AI assignee — the Copilot coding agent (GitHub) or the Claude coding agent (Anthropic) — by creating an issue and assigning it to the bot. Triggers include "let copilot do this", "assign it to claude", "delegate this to the AI", "make it an issue for copilot", or any request to offload work that will run in GitHub's sandboxed Actions runner. Covers the pre-flight check (is the feature enabled on this repo), the decision of whether the task is even delegatable (sandbox can't reach internal networks, private feeds, deploy targets, or a real browser against a deployed env), picking Copilot vs. Claude, the issue body template the bots behave best on, the exact `gh issue create --assignee` invocation, and follow-up etiquette on the draft PR the bot opens. Skip this skill for tasks the user will execute locally themselves.
---

# Delegate to a GitHub AI assignee

Hand a task off to **Copilot (GitHub)** or **Claude (Anthropic)** as a GitHub issue. Both bots watch for assignment events on enabled repos, open a draft PR, and push commits in a sandboxed GitHub-hosted Actions runner.

The hard part is **not** the `gh` command — it's the *brief* you write. AI assignees behave like a junior contractor who has never seen the codebase: they need a self-contained spec or they ship the wrong thing.

## When NOT to delegate

The runner is network-isolated and ephemeral. **Keep the task local** if it needs any of:

- Internal package feeds (private NuGet, internal npm registry, GitLab/Artifactory mirrors)
- VPN-only services (intranet APIs, on-prem databases, corporate auth)
- Deployment targets (IIS hosts, internal Docker registries, on-prem Kubernetes)
- Filesystem state outside the repo (local config, secrets in a password manager, dev certificates)
- A real browser against a deployed environment (the runner has Playwright but only against what it can spin up itself)
- Manual verification only the user can do (visual design review, hardware-in-the-loop testing)

Litmus test: *"Could a fresh open-source contributor with only the public repo build and verify this?"* If no → keep it local.

## Pre-flight check

Verify the AI bots are actually assignable on this repo before promising the user it'll work. Don't assume from the org plan — repo-level policy can disable it.

```bash
gh api repos/<owner>/<repo>/assignees --jq '.[].login' | grep -iE '^(copilot|claude)$'
```

- Output contains `Copilot` → Copilot coding agent is enabled.
- Output contains `Claude` → Anthropic's Claude GitHub app is installed and enabled.
- Empty → feature is off. Stop and tell the user where to enable it: **Settings → Code & automation → Copilot** for Copilot, and **Settings → GitHub Apps → Claude** (or install from the GitHub Marketplace) for Claude.

The exact assignee handle (`Copilot`, `Claude`, or something else) is whatever `gh api` returns — use that string verbatim in `--assignee`.

## Choose the assignee

Both bots can fail silently. The assignment is a starting point, not a guarantee. Pick based on shape of the task:

| Use **Copilot** when | Use **Claude** when |
|---|---|
| Small, well-scoped change in 1–3 files | Open-ended spec needing judgment |
| Mechanical refactor with a clear pattern | Multi-file reasoning across modules |
| Adding tests for existing code | New feature touching design + code + tests |
| Fixing a narrow bug with a reproducer | Investigation where root cause is unclear |
| The acceptance criteria are mechanical | The acceptance criteria require taste |

If you genuinely can't tell, prefer Claude for design-heavy work and Copilot for grind work. **Do not assign both bots to the same issue** — they don't coordinate; you'll get two divergent draft PRs.

## Issue body template

Use this exact structure. Skip nothing — every section is load-bearing for the bot.

````markdown
## Goal
<one sentence — what should be true when this is done>

## Acceptance criteria
- [ ] <observable, testable outcome>
- [ ] <observable, testable outcome>

## In scope (files / areas)
- `path/to/file.ext` — <why>
- `path/to/dir/` — <why>

## Out of scope
- <thing the bot should NOT touch even if it looks related>

## How to verify
```bash
<exact commands the bot can run in CI / its sandbox>
```

## Constraints
- Follow conventions in `CLAUDE.md` / `.github/copilot-instructions.md`
- <other repo-specific rules: no new packages, no breaking API changes, no schema migrations, etc.>

## References
- <links to related code, prior PRs, design docs, issues>
````

Why each section matters:

- **Goal** in one sentence forces clarity. If you can't compress it, the task isn't ready.
- **Acceptance criteria** must be observable — the bot uses these to decide it's done. "Improve performance" → bot picks an arbitrary metric. "P95 latency on `/api/v1/orders` < 200ms measured by `tests/perf/orders.bench`" → bot has a target.
- **In-scope files** prevent the bot from "helpfully" rewriting unrelated code.
- **Out-of-scope** is where you list the obvious-but-wrong tangent the bot would otherwise take.
- **How to verify** is the bot's done-condition. Without it, the bot will assert success based on its own reading of its own diff.
- **Constraints** carry the repo's invisible rules (style, package policy, compatibility windows).

## Create & assign

Draft the body in a file first so it's reviewable before it goes out:

```bash
$EDITOR /tmp/issue.md
```

Then create the issue and assign **one** bot:

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "<conventional commit-style title>" \
  --body-file /tmp/issue.md \
  --assignee Copilot       # or: --assignee Claude
```

Capture the URL from `gh issue create` output and surface it to the user — they'll want to watch the draft PR appear under it.

## Follow-up etiquette

- **Watch the draft PR, not the issue.** The bot opens a draft PR linked to the issue within ~1–5 minutes. All iteration happens on that PR.
- **Comment on the PR to drive the next iteration.** Comments on the issue itself don't reliably trigger another bot pass.
- **One coherent change per review round.** "Fix X and also refactor Y and rename Z" derails the bot. Land X, then ask for Y.
- **Two-rounds rule.** If after two review rounds the bot is still off, take the work over locally — pulling the bot's branch and finishing it yourself is faster than a third round.
- **Watch for stalls.** Draft PR with no commits for >24h usually means the bot abandoned. Close the issue manually and either retry with a tighter brief or do it yourself.

## Common mistakes

- **Skipping the pre-flight check** and telling the user "it's assigned" when the feature isn't enabled on the repo — wastes their time waiting for a PR that will never appear.
- **Vague goal.** "Improve the upload code" → bot picks something arbitrary. State the observable outcome.
- **No verification command.** Bot can't tell if it's done; will guess from its own diff.
- **Implicit scope.** "Refactor the orders module" without a file list → bot edits the wrong things.
- **Delegating tasks that need internal infra.** Sandbox can't reach your VPN; the bot will fail in confusing ways. Re-read the "When NOT to delegate" list before assigning.
- **Assigning both bots.** They don't coordinate; you get two divergent draft PRs and have to pick one to abandon.
- **Treating the bot as fire-and-forget.** Without follow-up review on the draft PR, the bot's first guess ships. Always review.

## Quick reference

```bash
# 1. Pre-flight
gh api repos/<owner>/<repo>/assignees --jq '.[].login' | grep -iE '^(copilot|claude)$'

# 2. Draft brief
$EDITOR /tmp/issue.md   # use the template above

# 3. Create & assign (one bot)
gh issue create --repo <owner>/<repo> --title "..." --body-file /tmp/issue.md --assignee Copilot

# 4. Watch the draft PR that appears under the new issue
gh pr list --repo <owner>/<repo> --draft --author app/copilot-swe-agent
```
