# Incident Harness

Use for runtime incidents, production-like failures, or urgent regressions.

## Shape

- Evidence session captures live truth: current behavior, logs, env/config,
  health checks, request/response readback, and timeline.
- Fix session is launched only if the user and workflow authorize writes.
- Verifier session or compact evidence packet performs readback on the actual
  target environment.
- Orchestrator keeps a short timeline and avoids broad unrelated exploration.

## Task File Fields

```markdown
mode: evidence-session | implementation-slice
objective:
project_constraints_capsule:
verification_tier:
runtime_target:
readback_commands:
allowed_writes:
forbidden:
bootstrap:
rollback_or_stop_line:
verification:
skip_checks:
budget:
stop_when:
handoff_path:
```

## Evidence Checklist

- exact target and timestamp
- observed failure and reproduction path
- config/env/log facts, with secrets redacted
- suspected cause versus confirmed cause
- fix boundary, if any
- post-fix readback commands and results

## Completion Gate

The orchestrator may call the incident closed only after:

- live readback proves the user-visible symptom is gone or clearly deferred
- any production action was explicitly authorized
- remaining risk and follow-up are recorded
- heartbeat or continuation jobs are stopped
