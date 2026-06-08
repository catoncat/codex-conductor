# Codex Conductor Project Profile

Copy this template to `.codex-conductor/project.md` in a repository that needs
project-specific orchestration rules.

## Purpose

Describe the repository and the kinds of work Codex Conductor should coordinate.

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

### docs-only

- Use when:
- Commands:
- Skip:

### script-or-guard

- Use when:
- Commands:
- Skip:

### unit-or-component

- Use when:
- Bootstrap:
- Commands:
- Skip:

### integration-runtime

- Use when:
- Bootstrap:
- Commands:
- Skip:

### schema-or-migration

- Use when:
- Bootstrap:
- Commands:
- Forbidden:

### production-or-ops

- Use when:
- Read-only evidence:
- Requires explicit authorization:
- Stop lines:

## Worktree Environment

- New worktrees require:
- Safe to reuse from main checkout:
- Must be isolated:
- Never do:

## Lifecycle

- Identity and naming:
  - Session title:
  - Branch naming:
  - Worktree naming:
  - Issue/PR title:
  - Label policy:
  - Max lengths:
  - Illegal character replacement:
- Commit policy:
- Push policy:
- Issue/PR policy:
- Merge/release/deploy policy:

Identity and naming overrides may change display templates, length limits,
illegal character replacement, and issue/PR/branch naming policy. They must not
change the conductor role boundary, require new artifact classes, or make
labels/sessions/worktrees mandatory when the chosen workflow shape does not
need them.

## Worker Prompt Requirements

Every worker prompt must include:

- Task mode:
- Verification tier:
- Allowed writes:
- Forbidden actions:
- Bootstrap steps:
- Required proof:
- Checks to skip:
- Escalation condition:

## Forbidden Commands Or Actions

- ...
