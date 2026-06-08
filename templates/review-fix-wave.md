# Review/Fix Wave Harness

Use when review and remediation must stay separate.

## Shape

- Review sessions are read-only and produce findings with proof.
- The orchestrator deduplicates findings and assigns bounded fix sessions.
- Fix sessions implement only assigned findings and record verification.
- A final verification session or compact evidence packet checks that the
  original finding is resolved and no adjacent behavior regressed.

## Review Finding Schema

```markdown
id:
severity:
claim:
evidence:
affected_paths:
minimal_fix_scope:
counterevidence_or_uncertainty:
```

## Fix Task Fields

```markdown
mode: review-fix-session
finding_ids:
project_constraints_capsule:
verification_tier:
allowed_writes:
forbidden:
bootstrap:
verification:
skip_checks:
commit_policy:
budget:
stop_when:
handoff_path:
```

## Completion Gate

The orchestrator closes this wave only after:

- every actionable finding is fixed, deferred with reason, or rejected with
  counterevidence
- each fix has focused verification
- each fix followed the declared project constraints and verification tier
- the final verifier or evidence packet confirms the finding-to-fix mapping
- residual risk is recorded in the final handoff

Closing this wave is not closing the workflow. After closeout, update the
program backlog in `workflow-state.md` or `milestone-plan.md`. If follow-up
review, fix, or verification waves remain unblocked and inside budget, launch
the next one or roll over with an exact next-wave launch instruction.
