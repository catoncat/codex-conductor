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
allowed_writes:
forbidden:
verification:
commit_policy:
budget:
stop_when:
handoff_path:
```

## Completion Gate

The orchestrator closes the wave only after:

- every actionable finding is fixed, deferred with reason, or rejected with
  counterevidence
- each fix has focused verification
- the final verifier or evidence packet confirms the finding-to-fix mapping
- residual risk is recorded in the final handoff
