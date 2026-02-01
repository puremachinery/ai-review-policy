# AI Code Review System Setup

This file contains everything needed to set up a structured AI code review system in a repository. An AI agent should parse this file and create the described file structure.

## Instructions for Setup Agent

1. Create `REVIEW_POLICY.yaml` in the repo root (this file IS checked into git)
2. Create `.review_state.yaml` in the repo root (this file is NOT checked into git)
3. Create `.review_history/` directory structure (this directory is NOT checked into git)
4. Append the gitignore entries to the repo's `.gitignore`
5. If the user enables the `feature_verification` flag, create `docs/feature-status.yaml` and `docs/feature-evidence.yaml` (both checked into git)
6. Inform the user which `project_flags` they should uncomment based on project characteristics

---

## File: REVIEW_POLICY.yaml

**Location:** `{repo_root}/REVIEW_POLICY.yaml`
**Git:** CHECK IN (team-agreed configuration)

```yaml
# REVIEW_POLICY.yaml
# Drop this file in your repo root to guide AI agents on targeted code reviews.
# Instead of "review this", agents should perform each enabled review type separately.

version: "1.1"

# Global defaults (can be overridden per review type)
defaults:
  enabled: true
  frequency: "every_pr"  # every_pr | daily | weekly | monthly | quarterly | manual
  severity: "medium"     # low | medium | high | critical

# ============================================================================
# CONDITION LOGIC
# ============================================================================
# skip_if:  List of project flags. Review is SKIPPED if ANY flag matches (OR).
# only_if:  List of project flags. Review RUNS only if ANY flag matches (OR).
#
# For AND semantics (all flags must match), use:
#   skip_if_all: [...]   - skip only when ALL listed flags are set
#   only_if_all: [...]   - run only when ALL listed flags are set
#
# change_triggers: List of file glob patterns. For PR reviews, the review is
#   skipped if no changed files match any pattern. Ignored for periodic reviews.
#
# Evaluation order:
#   1. enabled: false → skip (hard override, nothing else checked)
#   2. postponements → skip if current date < until
#   3. skip_if / skip_if_all → skip if condition met
#   4. only_if / only_if_all → skip if condition NOT met
#   5. change_triggers → skip if PR review and no changed files match
#   6. frequency → skip if not yet due based on last_run

# ============================================================================
# SCOPE
# ============================================================================
# Controls what files the agent examines depending on review context.
scope:
  pr_review: "diff_and_imports"     # Changed files + files that directly import/depend on them
  periodic_review: "full"           # Entire codebase
  max_files_per_pass: 50            # Safety limit to bound token usage per review pass

review_types:

  # ============================================================================
  # CORRECTNESS / LOGIC
  # ============================================================================
  correctness:
    logic:
      description: "Does the code do what it claims? Are algorithms implemented correctly?"
      enabled: true
      frequency: "every_pr"
      severity: "critical"

    edge_cases:
      description: "Boundary conditions, empty/null inputs, overflow, off-by-one errors"
      enabled: true
      frequency: "every_pr"
      severity: "high"

    error_handling:
      description: "Failure modes, exception paths, error propagation, recovery logic"
      enabled: true
      frequency: "every_pr"
      severity: "high"

    concurrency:
      description: "Race conditions, deadlocks, atomicity, thread safety"
      enabled: true
      frequency: "every_pr"
      severity: "critical"
      skip_if:
        - "no_concurrency"
        - "single_threaded"

  # ============================================================================
  # SECURITY
  # ============================================================================
  security:
    input_validation:
      description: "Injection attacks, sanitization, encoding, untrusted input handling"
      enabled: true
      frequency: "every_pr"
      severity: "critical"

    authn_authz:
      description: "Authentication, authorization, access control, privilege escalation"
      enabled: true
      frequency: "every_pr"
      severity: "critical"
      skip_if:
        - "no_auth"
        - "internal_tool"

    secrets:
      description: "Hardcoded credentials, API keys, key management, .env handling"
      enabled: true
      frequency: "every_pr"
      severity: "critical"

    crypto:
      description: "Key derivation, signature verification, HMAC usage, encryption correctness, timing safety"
      enabled: true
      frequency: "every_pr"
      severity: "critical"
      only_if:
        - "has_crypto"
      change_triggers:
        - "**/integrity*"
        - "**/signature*"
        - "**/secrets*"
        - "**/encrypt*"
        - "**/crypto*"
        - "**/hmac*"
        - "**/hash*"
        - "**/key*"

    plugin_security:
      description: "Plugin/extension sandbox enforcement, capability policy, signature trust, supply chain"
      enabled: true
      frequency: "every_pr"
      severity: "critical"
      only_if:
        - "has_plugins"
      change_triggers:
        - "**/plugins/**"
        - "**/extensions/**"
        - "**/sandbox*"
        - "**/signature*"
        - "**/loader*"
        - "**/runtime*"

    dependencies:
      description: "CVEs in dependencies, supply chain risks, outdated packages"
      enabled: true
      frequency: "weekly"
      severity: "high"

  # ============================================================================
  # ARCHITECTURE / DESIGN
  # ============================================================================
  architecture:
    api_design:
      description: "API contracts, versioning, backwards compatibility, REST/GraphQL conventions"
      enabled: true
      frequency: "every_pr"
      severity: "medium"
      only_if:
        - "has_api"
      change_triggers:
        - "src/api/**"
        - "api/**"
        - "**/routes.*"
        - "**/router.*"
        - "**/endpoints/**"
        - "openapi.*"
        - "swagger.*"

    abstractions:
      description: "Coupling, cohesion, separation of concerns, DRY violations, layering"
      enabled: true
      frequency: "every_pr"
      severity: "medium"

    scalability:
      description: "Bottlenecks, resource growth patterns, horizontal vs vertical scaling"
      enabled: true
      frequency: "monthly"
      severity: "medium"
      skip_if:
        - "prototype"
        - "small_scale"

    data_model:
      description: "Schema design, migrations, consistency, normalization, indexing"
      enabled: true
      frequency: "every_pr"
      severity: "high"
      only_if:
        - "has_database"
        - "has_file_store"
      change_triggers:
        - "**/migrations/**"
        - "**/models/**"
        - "**/schema*"
        - "**/entities/**"
        - "prisma/**"
        - "alembic/**"
        - "db/**"

    data_flow:
      description: "Data lifecycle: ingestion, transformation, storage, transmission, deletion, retention"
      enabled: true
      frequency: "monthly"
      severity: "high"
      only_if:
        - "has_database"
        - "has_file_store"
        - "has_api"
        - "handles_pii"

  # ============================================================================
  # QUALITY / MAINTAINABILITY
  # ============================================================================
  quality:
    readability:
      description: "Naming, comments, complexity (cyclomatic, cognitive), code organization"
      enabled: true
      frequency: "every_pr"
      severity: "medium"

    test_coverage:
      description: "Unit tests, integration tests, property-based tests, test quality"
      enabled: true
      frequency: "every_pr"
      severity: "high"

    documentation:
      description: "README, docstrings, inline comments, ADRs, API docs"
      enabled: true
      frequency: "weekly"
      severity: "low"

    style:
      description: "Formatting, linting, language conventions, consistency"
      enabled: false  # Usually handled by automated tooling
      frequency: "every_pr"
      severity: "low"
      note: "Prefer automated linters (prettier, scalafmt, black, etc.)"

  # ============================================================================
  # PERFORMANCE
  # ============================================================================
  performance:
    algorithmic:
      description: "Big-O complexity, hot paths, unnecessary computation"
      enabled: true
      frequency: "every_pr"
      severity: "medium"

    memory:
      description: "Allocations, leaks, GC pressure, large object handling"
      enabled: true
      frequency: "every_pr"
      severity: "medium"

    io:
      description: "N+1 queries, batching, caching, connection pooling, network calls"
      enabled: true
      frequency: "every_pr"
      severity: "high"

  # ============================================================================
  # OPERATIONAL
  # ============================================================================
  operational:
    observability:
      description: "Logging quality, metrics, tracing, alerting hooks"
      enabled: true
      frequency: "weekly"
      severity: "medium"
      skip_if:
        - "prototype"
        - "local_only"

    configuration:
      description: "Env vars, feature flags, defaults, config validation"
      enabled: true
      frequency: "every_pr"
      severity: "medium"

    deployment:
      description: "Rollback safety, migration ordering, blue-green compatibility"
      enabled: true
      frequency: "every_pr"
      severity: "high"
      skip_if:
        - "library"
        - "no_deploy"

    failure_modes:
      description: "Graceful degradation, circuit breakers, retry logic, timeouts"
      enabled: true
      frequency: "monthly"
      severity: "high"

  # ============================================================================
  # COMPLIANCE / PROCESS
  # ============================================================================
  compliance:
    licenses:
      description: "Dependency licenses, compatibility, attribution requirements"
      enabled: true
      frequency: "quarterly"
      severity: "medium"
      skip_if:
        - "internal_only"
        - "prototype"

    regulatory:
      description: "GDPR, PCI, SOC2, HIPAA implications"
      enabled: false  # Enable based on your compliance requirements
      frequency: "quarterly"
      severity: "critical"
      only_if:
        - "handles_pii"
        - "handles_payments"
        - "healthcare"

    breaking_changes:
      description: "Semver compliance, deprecation policy, changelog updates"
      enabled: true
      frequency: "every_pr"
      severity: "medium"
      only_if:
        - "library"
        - "public_api"

  # ============================================================================
  # FEATURE VERIFICATION
  # ============================================================================
  feature:
    status_verification:
      description: "Verify claimed feature completeness against feature-status.yaml and runtime wiring"
      enabled: true
      frequency: "every_pr"
      severity: "high"
      only_if:
        - "feature_verification"
      change_triggers:
        - "docs/feature-status.yaml"
        - "docs/feature-evidence.yaml"

# ============================================================================
# PROJECT FLAGS
# Set these to control which reviews are skipped via skip_if / only_if
# ============================================================================
project_flags:
  - "has_api"
  - "has_database"
  # Uncomment flags that apply to this project:
  # - "has_file_store"       # File-based storage (sessions, config, etc.)
  # - "has_crypto"           # Cryptographic operations (HMAC, signatures, encryption)
  # - "has_plugins"          # Plugin/extension system (WASM, dynamic loading, etc.)
  # - "feature_verification" # Enforce feature status review from feature-status.yaml
  # - "prototype"
  # - "small_scale"
  # - "internal_only"
  # - "internal_tool"
  # - "no_auth"
  # - "no_concurrency"
  # - "single_threaded"
  # - "library"
  # - "public_api"
  # - "handles_pii"
  # - "handles_payments"
  # - "healthcare"
  # - "local_only"
  # - "no_deploy"

# ============================================================================
# POSTPONEMENTS
# Temporarily skip specific reviews until a date
# ============================================================================
postponements:
  # Example:
  # - review: "operational.observability"
  #   until: "2025-03-01"
  #   reason: "Deferring until post-MVP launch"
  []

# ============================================================================
# STATE TRACKING & HISTORY
# ============================================================================
state_file: ".review_state.yaml"      # Gitignored, tracks last run per review type
history_dir: ".review_history/"       # Gitignored, stores full review outputs

history_retention:
  max_per_type: 5                     # Keep N most recent reviews per type
  max_age_days: 90                    # Delete reviews older than N days
  # Delete if count exceeds max_per_type OR age exceeds max_age_days

# ============================================================================
# AGENT INSTRUCTIONS
# ============================================================================
agent_instructions: |
  When asked to "review" this codebase or a PR:

  1. Parse this REVIEW_POLICY.yaml file first
  2. Load .review_state.yaml to check last run times and current git ref
  3. Evaluate which reviews are due using this order for each review type:
     a. enabled: false → skip immediately
     b. Check postponements → skip if current date < until
     c. Check skip_if (OR: skip if ANY flag matches) / skip_if_all (AND: skip if ALL match)
     d. Check only_if (OR: run if ANY flag matches) / only_if_all (AND: run if ALL match)
     e. Check change_triggers (PR reviews only): skip if no changed files match any glob
     f. Check frequency: compare last_run against current time
        - "every_pr" reviews always run on PR/diff reviews or when code changed since last_ref
  4. Check project_flags to determine which skip_if/only_if conditions apply
  5. Check postponements for temporarily disabled reviews
  6. Determine scope based on review context:
     - PR reviews: examine changed files and their direct dependents (importers/callers)
       per the scope.pr_review setting
     - Periodic reviews (daily/weekly/monthly/quarterly): scan the full codebase
       per the scope.periodic_review setting
     - Respect scope.max_files_per_pass to bound token usage
  7. Batch reviews by category:
     - Run all due subcategories within a category (e.g. all correctness.* checks)
       in a SINGLE pass over the relevant files
     - Report and track findings SEPARATELY per subcategory
     - Do NOT combine reviews across different categories into one pass
     - For feature.status_verification, use docs/feature-status.yaml as the feature inventory and
       require evidence for "complete" in docs/feature-evidence.yaml: runtime wiring, config surface, and tests
  8. Prioritize categories by highest severity among their due subcategories:
     critical > high > medium > low
  9. After completing reviews:
     a. UPDATE .review_state.yaml with entries for each completed review:
        - last_run: current ISO timestamp
        - last_ref: current git HEAD short SHA
        - status: pass | warn | fail | skipped
        - findings_count: number of issues found
        - critical_count: number of critical/blocking issues
        - notes: brief summary if useful
     b. WRITE full output to .review_history/{review_type}/{timestamp}_{short_sha}.md
     c. PRUNE old history files per history_retention settings

  On-demand invocation:
  - To run ALL due reviews: "Review this codebase using REVIEW_POLICY.yaml"
  - To run a specific review: "Review security.secrets using REVIEW_POLICY.yaml"
  - To run a category: "Review all security reviews using REVIEW_POLICY.yaml"
  - On-demand requests bypass frequency checks (always run) but still respect
    enabled, skip_if, only_if, and change_triggers

  History file format (.review_history/{category}.{subcategory}/{timestamp}_{sha}.md):
    ```
    # {category}.{subcategory} Review

    **Date:** {ISO timestamp}
    **Git Ref:** {full sha}
    **Status:** PASS | WARN | FAIL
    **Findings:** {count} ({critical_count} critical)

    ## Summary
    {brief summary}

    ## Findings
    - [{SEVERITY}] `file:line` - description
    - [{SEVERITY}] `file:line` - description

    ## Files Reviewed
    - path/to/file.ext
    ```

    Severity labels for individual findings: [CRITICAL], [HIGH], [MEDIUM], [LOW]
    Use the review type's configured severity as the default, but override per
    finding when the actual impact differs.

  History pruning (run after each review):
    1. List files in .review_history/{review_type}/
    2. Sort by timestamp in filename to determine age/order
    3. Delete files where count exceeds max_per_type (keep newest N)
    4. Delete remaining files where age exceeds max_age_days

  Frequency interpretation:
  - every_pr: Run on any PR review or when code changed since last_ref
  - daily: Run if last_run > 24 hours ago
  - weekly: Run if last_run > 7 days ago
  - monthly: Run if last_run > 30 days ago
  - quarterly: Run if last_run > 90 days ago
  - manual: Only run when explicitly requested by name

  Consulting history:
  - When reviewing, you may read recent history files to see patterns
  - Recurring issues across multiple reviews may warrant higher severity
  - Resolved issues from prior reviews can be noted as fixed

  change_triggers evaluation:
  - Only applies to PR / diff reviews (not periodic)
  - Glob patterns are matched against the list of changed files in the PR
  - If change_triggers is present and NO changed files match, skip the review
  - If change_triggers is absent, the review is not filtered by file paths
  - Patterns follow standard glob syntax (**, *, ?)
```

---

## File: .review_state.yaml

**Location:** `{repo_root}/.review_state.yaml`
**Git:** DO NOT CHECK IN (add to .gitignore)

```yaml
# .review_state.yaml
# AUTO-GENERATED by review agent - Do not edit manually
# Tracks last execution and results of each review type
# Agents: create/update entries dynamically based on REVIEW_POLICY.yaml review_types

schema_version: "1.1"

# Git ref at time of last full review (helps agent know if code changed)
last_reviewed_ref: null  # e.g., "abc123f"
last_reviewed_at: null   # e.g., "2025-01-28T14:30:00Z"

# Reviews are keyed as "{category}.{subcategory}" matching REVIEW_POLICY.yaml.
# Agents should create entries as reviews are executed — do not pre-populate.
# Entry format:
#
#   {category}.{subcategory}:
#     last_run: null       # ISO timestamp of last execution
#     last_ref: null       # git short SHA at time of last review
#     status: null         # pass | warn | fail | skipped
#     findings_count: 0    # total issues found
#     critical_count: 0    # critical/blocking issues found
#     notes: null          # optional brief summary
#
reviews: {}
```

---

## Directory: .review_history/

**Location:** `{repo_root}/.review_history/`
**Git:** DO NOT CHECK IN (add to .gitignore)

Create the following empty directory structure:

```
.review_history/
├── correctness.logic/
├── correctness.edge_cases/
├── correctness.error_handling/
├── correctness.concurrency/
├── security.input_validation/
├── security.authn_authz/
├── security.secrets/
├── security.crypto/
├── security.plugin_security/
├── security.dependencies/
├── architecture.api_design/
├── architecture.abstractions/
├── architecture.scalability/
├── architecture.data_model/
├── architecture.data_flow/
├── quality.readability/
├── quality.test_coverage/
├── quality.documentation/
├── quality.style/
├── performance.algorithmic/
├── performance.memory/
├── performance.io/
├── operational.observability/
├── operational.configuration/
├── operational.deployment/
├── operational.failure_modes/
├── compliance.licenses/
├── compliance.regulatory/
├── compliance.breaking_changes/
└── feature.status_verification/
```

History files are named: `{ISO_timestamp}_{short_sha}.md`
Example: `2025-01-28T14-30-00Z_abc123d.md`

---

## Gitignore Entries

**Action:** Append these lines to `{repo_root}/.gitignore`

```
# AI Review System
.review_state.yaml
.review_history/
```

---

## Sample History File

**Purpose:** Reference format for agents writing history files

**Location example:** `.review_history/security.input_validation/2025-01-28T14-30-00Z_abc123d.md`

```markdown
# security.input_validation Review

**Date:** 2025-01-28T14:30:00Z
**Git Ref:** abc123def456789
**Status:** WARN
**Findings:** 2 (0 critical)

## Summary
Minor input validation gaps in API endpoints. No critical issues but recommend addressing before next release.

## Findings
- [HIGH] `src/api/users.py:47` - User ID parameter not validated as integer before DB query
- [MEDIUM] `src/api/search.py:23` - Search query length not bounded, potential DoS vector

## Files Reviewed
- src/api/users.py
- src/api/search.py
- src/api/auth.py
- src/utils/validators.py
```

---

## File: docs/feature-status.yaml (optional)

**Location:** `{repo_root}/docs/feature-status.yaml`
**Git:** CHECK IN (feature inventory is part of the project record)
**When:** Only create if `feature_verification` project flag is enabled

```yaml
# Feature status inventory for {project_name}.
# Single-marker status scheme.
version: "1.0"
source: "single-marker verification scheme"

status_legend:
  "[ ]": "Not sure — believed done, not yet verified"
  "[~]": "Partial — incomplete or missing behavior"
  "[s]": "Stub — placeholder response only"
  "[-]": "Verified missing — confirmed absent"
  "[x]": "Verified done — confirmed working"

feature_inventory_markdown: |
  ## Feature Inventory

  Comprehensive list of every feature in the system, grouped by module.
  Markers follow the status_legend.

  ### Module Name (`src/module/`)

  - [ ] **Feature name** — brief description (`implementation_file.ext`)
  - [x] **Another feature** — brief description (`file.ext`)
  - [~] **Partial feature** — what works, what doesn't (`file.ext`)
  - [-] **Missing feature** — confirmed not implemented
  - [s] **Stubbed feature** — placeholder response only (`file.ext`)
```

---

## File: docs/feature-evidence.yaml (optional)

**Location:** `{repo_root}/docs/feature-evidence.yaml`
**Git:** CHECK IN (evidence records are part of the project record)
**When:** Only create if `feature_verification` project flag is enabled

```yaml
# Evidence for verified feature statuses (done/partial/missing).
# Keep this focused on audited entries in docs/feature-status.yaml.

version: "1.0"
updated_at: null  # ISO date of last update

features:
  # Each entry documents evidence for a feature's claimed status.
  # Required fields: feature, status
  # Optional fields: runtime_wiring, tests, notes
  #
  # - feature: "descriptive feature name"
  #   status: "verified_done"    # verified_done | partial | verified_missing
  #   runtime_wiring:
  #     - "src/path/to/file.ext (description of how feature is wired)"
  #   tests:
  #     - "src/path/to/test.ext::test_name"
  #   notes:
  #     - "Optional notes about caveats, limitations, or missing pieces"
```

---

## Final Directory Structure

After setup, the repo should contain:

```
{repo_root}/
├── REVIEW_POLICY.yaml              # checked into git
├── .gitignore                      # checked in (with review entries appended)
├── .review_state.yaml              # gitignored
├── .review_history/                # gitignored
│   ├── correctness.logic/
│   ├── correctness.edge_cases/
│   ├── ... (all review type subdirs)
│   └── feature.status_verification/
└── docs/                           # optional, if feature_verification enabled
    ├── feature-status.yaml         # checked into git
    └── feature-evidence.yaml       # checked into git
```

---

## Post-Setup: Customize project_flags

After creating the files, the setup agent should ask the user which flags apply to their project and uncomment the relevant ones in `REVIEW_POLICY.yaml` under `project_flags:`.

**Questions to ask:**

1. Does this project expose an API? -> `has_api`
2. Does it use a database? -> `has_database`
3. Does it use file-based storage (sessions, config, etc.)? -> `has_file_store`
4. Does it perform cryptographic operations (HMAC, signatures, encryption)? -> `has_crypto`
5. Does it have a plugin/extension system? -> `has_plugins`
6. Do you want to track and verify feature completeness? -> `feature_verification`
7. Is this a prototype/MVP? -> `prototype`
8. Is it small-scale / personal project? -> `small_scale`
9. Is it internal-only (not public)? -> `internal_only`
10. Is it an internal tool (no external users)? -> `internal_tool`
11. Does it have authentication/authorization? (if no: `no_auth`)
12. Is it single-threaded / no concurrency? -> `single_threaded`, `no_concurrency`
13. Is it a library consumed by others? -> `library`
14. Does it have a public API contract? -> `public_api`
15. Does it handle PII? -> `handles_pii`
16. Does it handle payments? -> `handles_payments`
17. Is it healthcare-related? -> `healthcare`
18. Is it local-only (no deployment)? -> `local_only`, `no_deploy`
