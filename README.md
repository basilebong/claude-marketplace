# Claude Code Marketplace

A collection of Claude Code skills for software development workflows. All skills are bundled into a single `basilebong` plugin and invoked via the `basilebong:` namespace.

## Skills

### pm-spec
Acts as a Product Manager — explores the codebase, gathers requirements via interactive discovery, and produces a concise feature spec with lean user stories, edge cases, and file references. Runs a panel of reviewer sub-agents (UX, Security, Architecture, Business) before finalizing.

**Usage:** `/basilebong:pm-spec` or describe a feature you want to spec out.

### pr-reviewer
Reviews the current branch's GitHub or GitLab PR using a single Opus sub-agent. Groups findings into High, Medium, and Minor, and closes with a clear Blocked or Approved verdict. Output is short and scannable — readable in under a minute.

**Usage:** `/basilebong:pr-reviewer` (reviews the current branch's PR/MR).

## Installation

Add this repository as a Claude Code plugin source:

```
claude plugin add /path/to/basile-cc-marketplace
```

## Author

Basile Bong
