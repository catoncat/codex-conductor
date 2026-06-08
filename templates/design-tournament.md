# Design Tournament Harness

Use for architecture, planning, prompt design, product flow, or other decisions
where several plausible approaches exist.

## Shape

- Orchestrator writes a shared rubric: goal, constraints, non-goals, evidence
  expected, and decision deadline.
- Candidate sessions propose distinct approaches.
- Judge session compares proposals against the rubric.
- Optional verifier checks the winning proposal against code, docs, or runtime
  facts before implementation starts.

## Candidate Prompt Fields

```markdown
mode: decision-packet
objective:
project_constraints_capsule:
verification_tier:
rubric:
required_evidence:
forbidden_assumptions:
budget:
handoff_path:
```

## Candidate Handoff Schema

```markdown
## Proposal
## Why This Fits
## Costs
## Risks
## Evidence
## Rejected Alternatives
## First Implementation Slice
## Noise Events
## Efficiency Notes
```

## Completion Gate

The orchestrator chooses a plan only after:

- proposals are comparable under the same rubric
- assumptions are separated from verified facts
- the first implementation slice is small enough to hand to a worker
