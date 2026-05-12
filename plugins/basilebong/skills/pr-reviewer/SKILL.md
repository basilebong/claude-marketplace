---
name: pr-reviewer
description: Spawn specialized sub-agents (Security, Logic, UX, Conventions, Architecture) to review a PR from GitHub or GitLab, then run a separate scoring pass on a 0-100 confidence scale, filter to high-confidence findings, and close with a bullet list of the most important problems and suggested fixes.
---

# PR Reviewer — Multi-Agent Code Review

Detect the hosting platform (GitHub or GitLab), fetch PR changes, read any local uncommitted diff, then spawn five specialized reviewer sub-agents in parallel to **find** issues. Once findings are collected, a single Haiku scoring agent assigns a 0–100 confidence score to every finding. The orchestrator filters to high-confidence findings (≥ 80), presents the per-reviewer breakdown, and closes the report with a bullet list of the most important problems and suggested fixes.

Reviewers and scorers are intentionally separated. Reviewers are biased toward what they care about (a security reviewer wants to find security issues). Asking the same agent to also judge confidence creates an obvious conflict of interest. A separate scorer that only sees findings + context can apply the rubric more dispassionately, and a small model is enough for that judgment call.

---

## Phase 1: Gather Context

### Step 1 — Detect platform

Run these commands and check the exit codes:

```bash
git remote get-url origin
```

Inspect the remote URL:

- If it contains `github.com` → **GitHub**
- If it contains `gitlab` → **GitLab**
- Otherwise → ask the user via `AskUserQuestion`

### Step 2 — Fetch PR diff

**GitHub:**

```bash
# Get current branch name
branch=$(git rev-parse --abbrev-ref HEAD)

# Get the PR diff for the current branch
gh pr diff "$branch"
```

If `gh pr diff` fails (no open PR), fall back to diffing against the default branch:

```bash
default_branch=$(gh repo view --json defaultBranchRef -q '.defaultBranchRef.name')
git diff "$default_branch"...HEAD
```

**GitLab:**

```bash
branch=$(git rev-parse --abbrev-ref HEAD)

# Get MR diff for the current branch
glab mr diff
```

If `glab mr diff` fails, fall back:

```bash
default_branch=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
git diff "$default_branch"...HEAD
```

Store the result as `$PR_DIFF`.

### Step 3 — Fetch PR metadata

**GitHub:**

```bash
gh pr view "$branch" --json title,body,comments,reviews
```

**GitLab:**

```bash
glab mr view
```

Store as `$PR_META` (title, description, existing review comments).

### Step 4 — Local uncommitted changes

```bash
git diff HEAD
```

Store as `$LOCAL_DIFF`. If non-empty, it will be provided to reviewers alongside the PR diff so they can review pending changes too.

### Step 5 — Read CLAUDE.md files

Find all relevant CLAUDE.md files: the root CLAUDE.md (if one exists), plus any CLAUDE.md files in directories whose files the PR modified. Read the **full contents** of each. Store as `$CLAUDE_MD_CONTENTS` (a map of path → content).

### Step 6 — Summarize the PR (Haiku)

Launch a **Haiku** agent and pass it both `$PR_META` (especially the PR description / body) and `$PR_DIFF`. Ask it to return a short, plain-language summary of what this PR does and why — the kind of thing a busy reviewer can skim in five seconds and get the gist.

The summary should:

- Lead with the PR description's own framing if it's clear (the author usually knows best). Pull from the diff only to fill gaps or correct obvious mismatches.
- Be **3–5 sentences max.** Plain English. No jargon dump, no file lists, no bullet-by-bullet enumeration of changes.
- Answer two questions: *what changed* and *why*. Skip the "how".

Store as `$PR_SUMMARY`. This appears at the very top of the final report.

### Step 7 — Read modified files

From `$PR_DIFF`, extract the list of modified files. Read the full current content of each file so reviewers have complete context, not just hunks.

---

## Phase 2: Spawn Reviewer Sub-Agents (find only, no scoring)

Spawn all five reviewers **in parallel** using five separate `Agent` tool calls in the **same message**:

| Name           | Model    | Focus                                                  |
| -------------- | -------- | ------------------------------------------------------ |
| `security`     | `opus`   | Security                                               |
| `logic`        | `opus`   | Logic & Correctness                                    |
| `ux`           | `sonnet` | UX & Accessibility                                     |
| `conventions`  | `sonnet` | Conventions & Quality                                  |
| `architecture` | `opus`   | Overall design — was there a cleaner, safer, or easier-to-hand-off way to solve this? |

Embed **all context inline** in each agent's prompt so they do not need to fetch anything. Pass **each** reviewer:

- `$PR_DIFF` (the full PR diff)
- `$LOCAL_DIFF` (uncommitted changes, if any — clearly labeled as "uncommitted, not yet part of the PR")
- `$PR_META` (title, description, existing comments)
- `$PR_SUMMARY` (the summary from Phase 1)
- `$CLAUDE_MD_CONTENTS` (the full contents of all relevant CLAUDE.md files, labeled by path)
- The full content of all modified files (labeled by path)
- Their specific review prompt (see Reviewer Prompts below)
- **"All context you need is provided above. Do NOT run git commands, gh/glab commands, or re-fetch the diff. You MAY read additional files if you need to trace an import, check a function signature, or understand a dependency — but do not re-read files already provided."**

Reviewers **find and explain** issues. They do **not** score them — scoring happens in Phase 3 by an independent agent.

Do **not** use `TeamCreate` or `SendMessage`. Spawn each reviewer as a standalone `Agent` call (not `run_in_background`) so it returns its findings directly and terminates immediately when done.

Collect all findings from all five reviewers into a single list, preserving which reviewer produced each one. Call this `$FINDINGS`.

---

## Phase 3: Score (single Haiku agent)

Spawn **one** Haiku agent to score every finding in `$FINDINGS`. A single scoring agent is enough — the job is mechanical rubric application, not deep reasoning, and using one agent keeps scores comparable across reviewers (no cross-reviewer drift in interpretation of the rubric).

Pass the scoring agent:

- `$FINDINGS` — the full list, each item tagged with its reviewer category, file:line, and description+justification
- `$PR_DIFF` so it can confirm a finding lands on lines the author actually modified
- `$CLAUDE_MD_CONTENTS` so it can verify any cited convention rule actually exists
- The scoring rubric (see Confidence Scoring Rubric below) verbatim

The agent's task is to read each finding and assign a 0–100 score plus one short sentence of justification for the score itself (e.g. "verified the SQL injection sink, user input flows in unfiltered" or "CLAUDE.md doesn't actually call out this pattern, downgrading").

The scoring agent returns a list with each finding annotated: `{reviewer, file:line, description, score, score_justification}`. Store as `$SCORED_FINDINGS`.

---

## Phase 4: Filter

Discard all findings scoring **below 80**. Keep the discarded ones in a separate list so they can appear in the "Filtered Out" section of the report.

---

## Phase 5: Render the Report

Produce the final report. Layout, in order:

1. **PR header** — title, branch, platform, whether local uncommitted changes were included.
2. **Summary** — `$PR_SUMMARY` from Phase 1 Step 6. Short, plain language, skimmable in five seconds.
3. **Issues Found** count (surviving / total candidates).
4. **Per-reviewer sections** — Security, Logic, UX, Conventions. Each lists every surviving issue (score ≥ 80), sorted by confidence descending. Drop a section entirely if it has no surviving findings (don't print empty headers).
5. **Filtered Out** — every finding that scored below 80, one line each, with the reason for the low score from the scorer.
6. **Most important problems & suggested fixes** — your synthesis of the surviving findings, at the very bottom. 3–8 bullets. Each bullet must:
   - State the problem in plain language (what's wrong, where).
   - Suggest a **concrete** fix — what the author should actually change. Specific is good ("wrap the DB call in `select_for_update`", "replace the bare `except:` with `except ValidationError`", "move this `await` outside the loop"). "Consider refactoring" is not a fix.

   Lead with hard blockers (security, correctness) before stylistic ones. If two findings point at the same root cause, fold them into one bullet with one fix. If no issues survived filtering, omit this section entirely — don't print an empty header.

Template:

```markdown
## PR Review Report

**PR:** <title>
**Branch:** <branch>
**Platform:** GitHub | GitLab
**Local uncommitted changes:** Yes (included in review) | No

---

### Summary

<$PR_SUMMARY — 3–5 sentences, plain English, skimmable>

---

### Issues Found: <count> (from <total pre-filter count> candidates)

---

### Security (Opus)

- [<score>] `file:line` — description + justification

---

### Logic & Correctness (Opus)

- [<score>] `file:line` — description + justification

---

### UX & Accessibility (Sonnet)

- [<score>] `file:line` — description + justification

---

### Conventions & Quality (Sonnet)

- [<score>] `file:line` — description + justification

---

### Architecture (Opus)

- [<score>] `file:line (or PR-level)` — what + cleaner alternative + why meaningfully better

---

### Filtered Out (<count> issues below threshold)
- [<score>] <reviewer>: <brief description> — reason for low confidence

---

### Most important problems & suggested fixes

<3–8 bullets, each pairing a plain-language problem with a concrete suggested fix. Blockers first. Omit this section entirely if no issues survived filtering.>
```

If no issues survived filtering:

```markdown
## PR Review Report

**PR:** <title>
**Branch:** <branch>
**Platform:** GitHub | GitLab

---

### Summary

<$PR_SUMMARY>

---

No issues found. Reviewed <count> candidates across Security, Logic, UX, Conventions, and Architecture — all scored below the confidence threshold.
```

---

## Reviewer Prompts

Each reviewer's job is to **find and explain** issues. They do not assign confidence scores — Phase 3 handles that.

### Security Reviewer (`security`)

**Model:** `opus`

You are a security reviewer. All context is provided inline: the PR diff, the full content of modified files, the contents of relevant CLAUDE.md files, and optionally uncommitted local changes. Do NOT run git/gh/glab commands or re-fetch the diff. You MAY read additional files only to trace imports or check dependencies.

Note any security-related rules from the provided CLAUDE.md contents.

Find: XSS, SQL injection, command injection, SSRF, path traversal, hardcoded secrets/credentials, authentication and authorization gaps, missing input validation at system boundaries, insecure data exposure, unsafe deserialization, CSRF vulnerabilities, open redirects.

For each finding, explain the attack vector and impact. Do not assign a score — a separate scoring agent will do that.

Format:

```
### Security

- file:line — description + attack vector + impact
- file:line — description + attack vector + impact
```

Return your findings as your final output.

---

### Logic & Correctness Reviewer (`logic`)

**Model:** `opus`

You are a logic and correctness reviewer. All context is provided inline: the PR diff, the full content of modified files, the contents of relevant CLAUDE.md files, and optionally uncommitted local changes. Do NOT run git/gh/glab commands or re-fetch the diff. You MAY read additional files only to trace imports or check dependencies.

Note any correctness-related rules from the provided CLAUDE.md contents.

Trace every changed function path. Find: wrong boolean conditions, missing null/undefined checks, off-by-one errors, race conditions, broken async/await chains, unhandled promise rejections, stale closures, infinite loops or recursion, incorrect state transitions, unhandled edge cases, type mismatches that slip past the compiler.

For each finding, explain why it's a bug and the expected impact. Do not assign a score — a separate scoring agent will do that.

Format:

```
### Logic & Correctness

- file:line — description + justification
- file:line — description + justification
```

Return your findings as your final output.

---

### UX & Accessibility Reviewer (`ux`)

**Model:** `sonnet`

You are a UX and accessibility reviewer. All context is provided inline: the PR diff, the full content of modified files, the contents of relevant CLAUDE.md files, and optionally uncommitted local changes. Do NOT run git/gh/glab commands or re-fetch the diff. You MAY read additional files only to trace imports or check dependencies.

Note any UX/accessibility-related rules from the provided CLAUDE.md contents.

Evaluate: user-facing flows (are they intuitive?), feedback states (loading, error, success, empty states — are they all handled?), accessibility (keyboard navigation, screen reader support, focus management, ARIA attributes, color contrast), i18n readiness (are all user-facing strings translatable?), responsive behavior, form UX (validation feedback, disabled states, required field indicators).

For each finding, explain the user impact. Do not assign a score — a separate scoring agent will do that.

Format:

```
### UX & Accessibility

- file:line — description + user impact
- file:line — description + user impact
```

Return your findings as your final output.

---

### Conventions & Quality Reviewer (`conventions`)

**Model:** `sonnet`

You are a conventions and code quality reviewer. All context is provided inline: the PR diff, the full content of modified files, the contents of relevant CLAUDE.md files, and optionally uncommitted local changes. Do NOT run git/gh/glab commands or re-fetch the diff. You MAY read additional files only to trace imports or check dependencies.

The provided CLAUDE.md contents are your primary source of truth for project conventions. Also check `.claude/` rules if referenced.

Common things to look for:

- **Frontend:** no `as` type assertions, no `Optional[...]`, correct import order, `Box`/`List`/`Flex`/`FlexItem` from `@moblin/chakra-ui` (not Chakra Stack/VStack/HStack), correct i18n usage (`intl.$t()` / `FormattedMessage`), icons via `@ul/icons` with `<Icon as={...}>`, no `.otherwise()` in ts-pattern matches, proper form patterns with React Hook Form
- **Backend:** `str | None` not `Optional[str]`, `get_val()` in serializer `validate()`, no N+1 queries (check for `select_related`/`prefetch_related`), proper error arrays in ValidationError
- **General:** naming conventions (PascalCase components, camelCase hooks, kebab-case files), dead code, unnecessary complexity, missing error handling at system boundaries, performance issues (queries in loops, missing memoization where clearly needed)

For each finding, cite the convention source (CLAUDE.md rule, code comment, etc.). Do not assign a score — a separate scoring agent will do that.

Format:

```
### Conventions & Quality

- file:line — description + convention reference
- file:line — description + convention reference
```

Return your findings as your final output.

---

### Architecture Reviewer (`architecture`)

**Model:** `opus`

You are a perfectionist senior engineer reviewing the **overall design** of this PR. You are not here to flag specific bugs, missing null checks, or convention violations — other reviewers cover that. You are here to step back and ask three questions about the change as a whole:

1. **Was there another way to solve this problem?** Is the approach taken the most direct one, or did the author route around the obvious solution? Are there off-the-shelf patterns, existing helpers, or framework features that would have made this simpler?
2. **Was there another way to code it?** Within the chosen approach, could the structure be cleaner — fewer layers of indirection, fewer new abstractions, fewer moving parts? Does the PR introduce concepts (a new helper, a new base class, a new module boundary) that don't earn their keep? Or is it the opposite — duplication where a small abstraction would clarify?
3. **If yes, would the alternative have been meaningfully better?** Better across three dimensions, any one of which can justify a finding:
   - **Cleaner / simpler** — easier to read, fewer files touched, less coupling, less indirection.
   - **Safer** — fewer ways for this code to break now or under future changes. Harder to misuse. Fails loudly instead of silently. Smaller blast radius if something goes wrong.
   - **Easier for the next developer to take over** — intent is obvious from the code, not buried in tribal knowledge. Standard patterns over clever tricks. Discoverable through naming and structure. A new hire could modify it without reading five other files first.

   Don't flag alternatives that are merely *different*. Only flag them if they win on at least one of those three. If it's just a matter of taste, stay silent.

All context is provided inline: the PR diff, the full content of modified files, the contents of relevant CLAUDE.md files, and optionally uncommitted local changes. Do NOT run git/gh/glab commands or re-fetch the diff. You MAY read additional files to understand the broader architecture (look at neighbouring modules, sibling files, the call sites of changed code) — this is encouraged for architecture review, since you need to understand the system, not just the diff.

Things that should trip your radar:

- **Over-engineering** — abstractions, generics, base classes, config layers, or hooks added for a single use case
- **Under-engineering** — the same pattern repeated 3+ times where a small helper would fit naturally
- **Misplaced logic** — business logic in serializers, validation in views, data fetching in templates, etc.
- **Leaky abstractions** — internal details bleeding into callers, or callers reaching past the abstraction
- **Wrong layer of the stack** — solving in the frontend what belongs in the backend, or vice versa
- **Reinventing existing infrastructure** — hand-rolling what the framework, the codebase, or a dependency already does
- **Unnecessary coupling** — modules that didn't need to know about each other now do
- **Premature optimization** or **premature flexibility** — code shaped for a future requirement that may never arrive
- **Complicated where simple would do** — multi-step pipelines, indirection chains, or clever tricks where a straight-line implementation is clearer
- **Easy to misuse** — APIs where the wrong call compiles fine but does the wrong thing at runtime; ordering requirements not enforced by types; silent fallbacks that hide bugs
- **Fragile under change** — code where modifying one piece requires remembering to update three other places, with nothing to remind you
- **Requires tribal knowledge to maintain** — relies on context that lives in someone's head, in a Slack thread, or in a closed PR — not in the code or its neighbours. A new hire would not be able to safely modify this without an interview first.

For each finding, explain (a) what the PR does, (b) what the alternative is, (c) which of the three dimensions it wins on — **cleaner**, **safer**, or **easier-to-take-over** (or several), and (d) why concretely. Do not assign a score — a separate scoring agent will do that.

Be a perfectionist, but earn each finding. If the PR is genuinely solid, return "(no findings)" — don't manufacture issues.

Format:

```
### Architecture

- file:line (or PR-level) — what the PR does + alternative + dimension(s) it wins on + why concretely
- file:line (or PR-level) — what the PR does + alternative + dimension(s) it wins on + why concretely
```

Return your findings as your final output.

---

## Confidence Scoring Rubric

Include this rubric **verbatim** in the Phase 3 scoring agent's prompt:

> Score each issue on a scale from 0–100 indicating your confidence that the issue is real and worth fixing:
>
> - **0:** Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
> - **25:** Somewhat confident. This might be a real issue, but may also be a false positive. You weren't able to verify it. If the issue is stylistic, it was not explicitly called out in the relevant CLAUDE.md.
> - **50:** Moderately confident. You verified this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
> - **75:** Highly confident. You double-checked the issue and verified it is very likely real and will be hit in practice. The existing approach in the PR is insufficient. The issue is very important and will directly impact the code's functionality, or it is directly mentioned in the relevant CLAUDE.md.
> - **100:** Absolutely certain. You double-checked the issue and confirmed it is definitely real and will happen frequently in practice. The evidence directly confirms this.
>
> For issues flagged due to CLAUDE.md instructions, double-check that the CLAUDE.md actually calls out that issue specifically. If it doesn't, score no higher than 25.
>
> ### Examples of false positives (score 0–25)
>
> - Pre-existing issues not introduced by the PR
> - Something that looks like a bug but is not actually a bug
> - Pedantic nitpicks that a senior engineer wouldn't call out
> - Issues that a linter, typechecker, or compiler would catch (missing imports, type errors, formatting). Assume CI runs these separately.
> - General code quality issues (lack of test coverage, general security issues, poor documentation), unless explicitly required in CLAUDE.md
> - Issues called out in CLAUDE.md but explicitly silenced in the code (e.g. lint-ignore comment)
> - Changes in functionality that are likely intentional or directly related to the broader change
> - Real issues, but on lines the author did not modify in the PR
>
> For each finding, output: the original finding unchanged, a score (0–100), and one short sentence explaining the score (e.g. "verified the sink, user input flows in unfiltered" or "CLAUDE.md doesn't actually mention this pattern").

---

## Important Rules

- **Preserve file:line references.** The user needs to locate issues quickly.
- **Uncommitted changes are clearly labeled.** If `$LOCAL_DIFF` is non-empty, prefix those findings with `[UNCOMMITTED]` so the user knows they are not yet part of the PR.
- **No auto-fixing.** This skill only reviews — it does not modify code.
- **Fail gracefully.** If `gh`/`glab` is not installed or not authenticated, tell the user what to run and stop. Do not proceed without the PR diff.
- **Scoring is separated.** Reviewers find issues; a single Haiku agent scores them. This avoids the conflict of interest of asking a reviewer to also judge how confident it is in its own findings, and keeps the rubric applied consistently across all four reviewer categories.
- **The orchestrator writes the closing summary.** The "Most important problems & suggested fixes" section sits at the **bottom** of the report and is your synthesis of the surviving findings — not a copy-paste of the scorer. Each bullet must pair a clear problem statement with a concrete suggested fix. If nothing survived filtering, omit the section entirely.
- **The top-of-report summary is plain English.** `$PR_SUMMARY` from Phase 1 Step 6 leans on the PR description (the author's own framing) and stays to 3–5 sentences — readable in five seconds. It's not a diff dump.
- **Show filtered issues.** The report includes a "Filtered Out" section so the user can see what was considered but didn't meet the threshold.
