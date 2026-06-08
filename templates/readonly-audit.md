# Read-Only Audit Harness

Use for `audit-track`, `inventory-packet`, and high-risk `decision-packet`
work.

## Shape

- Orchestrator creates the control plane and a shared evidence schema.
- Evidence sessions own non-overlapping areas.
- A verifier session or compact counterevidence packet checks the strongest
  findings before they become final.
- No session edits product code or lifecycle state.

## Task File Fields

```markdown
mode: audit-track
objective:
project_constraints_capsule:
verification_tier: docs-only | script-or-guard | production-or-ops
read_paths:
allowed_writes: <workflow-dir>/outputs/<task-id>/ only
forbidden:
evidence_schema:
counterevidence_required:
budget:
stop_when:
handoff_path:
```

## Handoff Schema

```markdown
## Conclusion
## Evidence
## Counterevidence Checked
## Unsupported Or Open Questions
## Files Read
## Noise Events
## Efficiency Notes
## Tool Fit
```

## Completion Gate

The orchestrator may call the audit complete only after:

- every owned area has a handoff
- the verifier or counterevidence packet checked the strongest findings
- unsupported claims are downgraded
- registry status matches handoff/proof
- the workflow backlog has no pending unblocked audit passes or follow-up
  packets
