# Claude Code Marketplace

A collection of Claude Code skills for software development workflows. All skills are bundled into a single `basilebong` plugin and invoked via the `basilebong:` namespace.

## Skills

### pm-spec
Acts as a Product Manager — explores the codebase, gathers requirements via interactive discovery, and produces a concise feature spec with lean user stories, edge cases, and file references. Runs a panel of reviewer sub-agents (UX, Security, Architecture, Business) before finalizing.

**Usage:** `/basilebong:pm-spec` or describe a feature you want to spec out.

### pr-reviewer
Spawns specialized sub-agents (Security, Logic, UX, Conventions, Architecture) to review a PR from GitHub or GitLab. The Architecture reviewer steps back and asks "was there a cleaner way?". A separate Haiku scoring pass rates each finding on a 0-100 confidence scale; the orchestrator filters to high-confidence findings and closes the report with a bullet list of the most important problems and suggested fixes.

**Usage:** `/basilebong:pr-reviewer` with a PR URL or number.

## Installation

Add this repository as a Claude Code plugin source:

```
claude plugin add /path/to/basile-cc-marketplace
```

## Author

Basile Bong
