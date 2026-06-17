---
name: pr-reviewer
description: Review the current branch's GitHub or GitLab PR with a single Opus sub-agent. Groups findings into High, Medium, and Minor, then gives a clear Blocked or Approved verdict. Short, scannable output. Use whenever the user wants a PR or MR reviewed.
---

# PR Reviewer

Review the current branch's PR. Gather the context, hand it to **one** Opus reviewer, then report the findings by severity with a clear verdict.

One reviewer, not five. The report is short on purpose — the author should read it in under a minute.

## Phase 1 — Gather context

Detect the platform from the remote:

```bash
git remote get-url origin
```

- Contains `github.com` → **GitHub**
- Contains `gitlab` → **GitLab**
- Neither → ask the user.

Get the diff. **GitHub:**

```bash
branch=$(git rev-parse --abbrev-ref HEAD)
gh pr diff "$branch"
```

**GitLab:**

```bash
glab mr diff
```

If that fails (no open PR/MR), diff against the default branch instead:

```bash
# GitHub
default_branch=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
# GitLab
default_branch=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')

git diff "$default_branch"...HEAD
```

Then collect:

- **Metadata** — `gh pr view "$branch" --json title,body` or `glab mr view`. Title and description.
- **Uncommitted changes** — `git diff HEAD`. If non-empty, include it, labeled as not yet part of the PR.
- **CLAUDE.md** — the root file plus any in directories the PR touches. Read them in full.
- **Modified files** — read the full current content of each changed file, so the reviewer sees more than hunks.

If `gh` / `glab` is missing or not authenticated, tell the user what to run and stop.

## Phase 2 — Review (one Opus agent)

Spawn **one** `Agent`, model `opus`. Put all the context inline in its prompt — diff, metadata, uncommitted changes, CLAUDE.md contents, and the full modified files. Tell it not to re-fetch anything; it may read extra files only to trace an import or check a signature.

Reviewer prompt:

> You are a senior engineer reviewing this PR. All context is inline: the diff, the modified files, the relevant CLAUDE.md rules, and any uncommitted changes. Don't run git/gh/glab. Read extra files only to trace an import or a signature.
>
> Look across the whole change: correctness bugs, security holes, missing edge cases, broken async, race conditions, convention violations (CLAUDE.md is the source of truth), UX gaps, and design smells — was there a simpler or safer way?
>
> Only flag what a senior engineer would actually raise. Skip nitpicks a linter catches. Skip pre-existing issues. Skip anything on lines the author didn't touch.
>
> Rate each finding:
> - **High** — bug, security hole, data loss, or breaking change. Must fix before merge.
> - **Medium** — a real problem that should be fixed, but not a merge blocker.
> - **Minor** — a nit or suggestion.
>
> For each finding give: `file:line`, one sentence on the problem, and one concrete fix — what to actually change, not "consider refactoring". The fix is optional for Minor.
>
> Also return a one-line summary of what the PR does. If the PR is clean, say so and return no findings.

## Phase 3 — Report

Pick the verdict:

- **🔴 Blocked** — at least one High finding.
- **🟢 Approved** — no High findings. (Medium and Minor are advice, not blockers.)

Render it short. Drop any empty severity section. Keep `file:line` on every finding. Order findings High → Medium → Minor.

```markdown
## PR Review: <title>

**🔴 Blocked** — <one line: the blocker>   ← use 🟢 Approved when there are no High findings

<one-line summary of what the PR does>

### 🔴 High
- `file:line` — problem. **Fix:** concrete change.

### 🟡 Medium
- `file:line` — problem. **Fix:** concrete change.

### ⚪ Minor
- `file:line` — problem.
```

If nothing was found:

```markdown
## PR Review: <title>

**🟢 Approved** — no issues found.

<one-line summary of what the PR does>
```

## Rules

- **One sub-agent.** A single Opus reviewer — nothing else.
- **Keep it short.** Plain words, one line per finding. No diff dumps, no filler.
- **Every finding has a `file:line`.** Every High and Medium has a concrete fix.
- **Blocked needs a High finding.** Nothing else blocks the merge.
- **Review only.** Never edit code.
- **Label uncommitted findings** with `[uncommitted]` so the author knows they aren't part of the PR yet.
