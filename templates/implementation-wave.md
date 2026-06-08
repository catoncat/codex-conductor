# Implementation Wave Harness

Use for `implementation-slice` work where multiple bounded components can move
in parallel.

## Shape

- Orchestrator defines shared objective, stop lines, and verification commands.
- Each write session owns exact paths or a non-overlapping component.
- Sessions confirm `pwd`, repo root, branch, and `git status` before editing.
- Each coherent slice gets focused verification and a scoped commit only if
  authorized by the task.
- Integrator reviews diffs, resolves conflicts, and runs final focused checks.

## Task File Fields

```markdown
mode: implementation-slice
objective:
project_constraints_capsule:
verification_tier:
owned_paths:
allowed_writes:
forbidden:
shared_contracts:
bootstrap:
verification:
skip_checks:
commit_policy:
budget:
stop_when:
handoff_path:
```

## Stop Conditions

Stop and report instead of expanding scope when:

- the fix requires shared schema, root config, dependency, CI, migration, or
  production changes not listed in the task
- another session has touched the same owned path
- verification fails for a reason outside the owned scope
- dirty files do not belong to this slice

## Completion Gate

The orchestrator may integrate and close this wave only after:

- each write session produced proof or a blocker
- each worker followed the declared verification tier and bootstrap rules
- registry status and actual files agree
- no owned paths overlap unexpectedly
- final focused verification ran in the integration workspace

Closing this wave is not closing the workflow. After integration, update the
program backlog in `workflow-state.md` or `milestone-plan.md`. If the next wave
is known, unblocked, and inside the complexity budget, launch it or roll over
with an exact next-wave launch instruction.
