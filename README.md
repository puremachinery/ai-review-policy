# ai-review-policy

A structured review policy schema that turns AI agents into systematic code reviewers instead of vague ones.

## The Problem

When you ask an AI agent to "review this code," you get a single pass that tries to check everything at once — security, performance, correctness, style — and does none of them well. The output is a flat list of surface-level observations with no prioritization, no memory of what was checked before, and no way to know what was missed.

This is the equivalent of asking a human reviewer to "just look at everything." No experienced reviewer works that way. They focus on specific concerns, one at a time, with context about what matters for this particular project.

## The Approach

This repo provides a `REVIEW_POLICY.yaml` schema that decomposes "code review" into 27 discrete review types across 7 categories. An AI agent reads the policy file and executes reviews as separate, focused passes — each with its own scope, frequency, severity, and applicability conditions.

The result: instead of one shallow review, you get structured passes like "check all error handling paths" or "audit input validation" that go deeper because the agent isn't trying to do everything at once.

## Why Not Just Prompt Better?

You could write a detailed review prompt every time. The policy file solves the problems that approach doesn't:

**Separation of concerns produces deeper analysis.** A single pass that checks security, correctness, performance, and style simultaneously dilutes attention across all of them. Separate passes with explicit descriptions ("injection attacks, sanitization, encoding, untrusted input handling") focus the agent on one problem class at a time. The descriptions act as checklists — they tell the agent what to look for, not just what category it falls under.

**Not everything needs checking every time.** A correctness review belongs on every PR. A dependency CVE scan is a weekly task. A license compliance audit is quarterly. Encoding this in the policy means agents run the right reviews at the right cadence without you remembering to ask.

**Project context eliminates irrelevant reviews.** A single-threaded CLI tool doesn't need concurrency reviews. A library doesn't need deployment safety checks. Project flags (`skip_if`, `only_if`) prune the review set automatically so agents don't waste passes on inapplicable concerns.

**Change triggers scope PR reviews to relevant files.** The `data_model` review has triggers for `**/migrations/**`, `**/models/**`, `**/schema*`. If a PR only touches CSS, that review is skipped. Without this, every PR gets every review regardless of what changed.

**State tracking prevents redundant work and catches regressions.** The agent records what it reviewed and when. A `weekly` review that ran 3 days ago is skipped. Code that changed since the last review gets re-examined. History files let the agent spot recurring issues across reviews.

**Category batching keeps it practical.** 27 separate passes would be expensive. Reviews within the same category (e.g., all `correctness.*` checks) are batched into a single pass over the relevant files, with findings tracked separately. This gives you 7 focused passes instead of 27 redundant file reads.

**Per-finding severity enables triage.** A review type has a default severity, but individual findings are tagged `[CRITICAL]`, `[HIGH]`, `[MEDIUM]`, or `[LOW]` based on actual impact. A correctness review might surface both a critical algorithm bug and a low-priority simplification opportunity — the severity tag distinguishes them.

## What's in This Repo

| File | Purpose |
|---|---|
| `REVIEW_SYSTEM_SETUP.md` | Full setup instructions for an AI agent to bootstrap the review system into any repo |
| `README.md` | This file — design rationale and overview |
| `LICENSE` | MIT |

The setup document contains the complete `REVIEW_POLICY.yaml` template, `.review_state.yaml` template, `.review_history/` directory structure, `.gitignore` entries, and agent instructions.

## Quick Start

To add the review system to a repository, give an AI agent this instruction:

```
Read REVIEW_SYSTEM_SETUP.md and follow the setup instructions for this repo.
```

The agent will:
1. Create `REVIEW_POLICY.yaml` in the repo root (checked into git — this is your team-agreed config)
2. Create `.review_state.yaml` for tracking review runs (gitignored)
3. Create `.review_history/` directory tree for review outputs (gitignored)
4. Update `.gitignore`
5. Ask which project flags apply to customize the review set

Once set up, trigger reviews with:

```
Review this codebase using REVIEW_POLICY.yaml
```
```
Review PR #123 using REVIEW_POLICY.yaml
```
```
Review security.secrets using REVIEW_POLICY.yaml
```

## Review Categories

| Category | Reviews | Default Frequency |
|---|---|---|
| **Correctness** | logic, edge_cases, error_handling, concurrency | every PR |
| **Security** | input_validation, authn_authz, secrets, dependencies | every PR / weekly |
| **Architecture** | api_design, abstractions, scalability, data_model, data_flow | every PR / monthly |
| **Quality** | readability, test_coverage, documentation, style | every PR / weekly |
| **Performance** | algorithmic, memory, io | every PR |
| **Operational** | observability, configuration, deployment, failure_modes | every PR / weekly / monthly |
| **Compliance** | licenses, regulatory, breaking_changes | every PR / quarterly |

## Schema Design

The policy schema supports:

- **Condition logic** with explicit boolean semantics: `skip_if` / `only_if` (OR — any match), `skip_if_all` / `only_if_all` (AND — all must match)
- **Change triggers** — file glob patterns that scope PR reviews to relevant changes
- **Scope control** — PR reviews focus on diff + dependents; periodic reviews scan the full codebase; `max_files_per_pass` bounds token cost
- **Frequency scheduling** — every_pr, daily, weekly, monthly, quarterly, manual
- **Postponements** — temporarily defer specific reviews until a date with a reason
- **Documented evaluation order** — enabled > postponements > skip_if > only_if > change_triggers > frequency
- **Dynamic state** — entries created as reviews execute, not pre-populated
- **History retention** — auto-pruned by count (default 5 per type) or age (default 90 days)

## License

MIT
