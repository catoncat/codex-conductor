# Codex Conductor Project Profile

Copy this template to `.codex-conductor/project.md` in a repository that needs
project-specific orchestration rules.

## Purpose

Describe the repository and the kinds of work Codex Conductor should coordinate.

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

- Branch naming:
- Commit policy:
- Push policy:
- Issue/PR policy:
- Merge/release/deploy policy:

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
