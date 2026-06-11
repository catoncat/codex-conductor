# Configuration

Codex Conductor is project-agnostic. It should learn local rules from profiles
instead of hardcoding them into the skill.

## Configuration Locations

Project profile, committed with a repository:

```text
.codex-conductor/
  project.md
  verification.md
  worktree.md
  lifecycle.md
```

Host profile, private to one machine or organization:

```text
~/.config/codex-conductor/host.md
```

Set `CODEX_CONDUCTOR_HOME` to use a different host-profile directory.

## Read Order

Use this order when building a `Project Constraints Capsule`:

1. Current user instruction.
2. Existing workflow state.
3. `.codex-conductor/*.md`.
4. Repo instructions such as `AGENTS.md`, `CONTRIBUTING.md`, or local docs.
5. Host profile.
6. Cheap live detection.

Higher entries override lower entries. Repository profiles override host
profiles inside that repository.

## Project Profile Template

Use `.codex-conductor/project.md` when one short file is enough:

```markdown
# Codex Conductor Project Profile

## Purpose
Short description of the repository and how conductor workflows should operate.

## Intake And Brief
- Required scope questions:
- Required deliverable questions:
- Required authorization questions:
- Acceptance criteria defaults:
- Assumptions allowed without asking:
- Stop and ask when:

## Package Manager
- Use:
- Do not use:

## Verification Tiers
- docs-only:
  - use_when:
  - commands:
  - skip:
- script-or-guard:
  - use_when:
  - commands:
  - skip:
- unit-or-component:
  - use_when:
  - commands:
  - skip:
- integration-runtime:
  - use_when:
  - bootstrap:
  - commands:
  - skip:
- schema-or-migration:
  - use_when:
  - bootstrap:
  - commands:
  - forbidden:
- production-or-ops:
  - use_when:
  - read_only_first:
  - requires_user_authorization:

## Worktree Environment
- New worktrees require:
- Safe to reuse from main checkout:
- Must be isolated:
- Never do:

## Lifecycle
- Identity and naming:
  - session_title:
  - branch_naming:
  - worktree_naming:
  - issue_pr_title:
  - label_policy:
  - max_lengths:
  - illegal_character_replacement:
- Commit policy:
- Push policy:
- Issue/PR policy:
- Merge policy:

## Worker Prompt Requirements
- Include:
- Stop and escalate when:

## Forbidden Commands Or Actions
- ...
```

Split the file only when the profile grows. Use:

- `verification.md` for verification tiers and expensive checks.
- `worktree.md` for dependency, env, database, generated files, or setup rules.
- `lifecycle.md` for branch, commit, PR, issue, release, and deployment rules.

Identity and naming overrides may change display templates, length limits,
illegal character replacement, and issue/PR/branch naming policy. They must not
change the conductor role boundary, require new artifact classes, or make
labels, sessions, worktrees, issues, or PRs mandatory when the chosen workflow
shape does not need them.

## Host Profile Template

Use `~/.config/codex-conductor/host.md` for private machine capabilities:

```markdown
# Codex Conductor Host Profile

## Available Capabilities
- Thread launcher:
- Goal naming and lifecycle conventions:
- GitHub CLI:
- Issue tracker:
- Local memory/search:
- Repository lifecycle tools:

## Capability Rules
- Use this tool when:
- Do not use this tool when:
- Fallback:

## Private Boundaries
- Do not expose:
- Do not commit:
```

Do not commit host profiles to public repositories.

## Capsule Example

After reading the relevant profiles, the controller writes a short capsule into
the workflow state and worker prompts:

```markdown
## Project Constraints Capsule
source_files:
- .codex-conductor/project.md
- AGENTS.md
capability_mode: managed
package_manager: pnpm
verification_tiers:
- tier: docs-only
  commands: git diff --check
  skip: runtime bootstrap, database checks
- tier: integration-runtime
  bootstrap: install dependencies if missing, generate local clients if needed
  commands: focused integration test, targeted verify command
  skip: full suite until controller integration
worktree_bootstrap:
- new worktrees must confirm env and dependency state before tests
- schema or migration work must use isolated data stores
control_plane_publication:
- default: keep controller state local/private or on an orchestration branch
- delivery: cherry-pick verified product commits from the intended base
forbidden_commands:
- do not run destructive data commands without explicit authorization
```

Keep capsules short. They are launch instructions, not project documentation.

## Identity Example

When a durable workflow creates sessions or worktrees, record identity inside
the existing `workflow-state.md` rather than creating a new identity file:

```markdown
## Identity
workflow_slug: auth-rbac-audit
workflow_label: Auth RBAC Audit
run_id: r7f2
project_label: billing-api
naming_overrides: .codex-conductor/project.md
```

Use these fields to keep titles and paths recoverable. `session-registry.md`
remains the canonical map from visible names to thread ids, worktrees, branches,
handoffs, proof, and status.
