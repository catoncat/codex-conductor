# Harness Patterns

Use this reference after choosing a task mode. Pick the smallest pattern that
materially improves correctness, speed, or context isolation.

## Failure Modes To Guard Against

- Premature closure: one session declares the work done before counterevidence
  or runtime proof exists.
- Goal drift: a session optimizes for an adjacent task because the slice prompt
  lacks hard boundaries.
- State drift: registry, handoff, and actual thread status disagree.
- Control-plane overgrowth: artifact and gate creation grows faster than
  task proof.
- Handoff-latency replacement: a controller launches a duplicate worker because
  a compact handoff is late but the original worker is still progressing.
- Tool friction: repeated failed command shapes, broad searches, or wrapper
  paths that do not produce proof.
- Synthesis loss: fanout outputs cannot be compared because they use different
  evidence formats.

## Patterns

### Classify And Act

Use for mixed or ambiguous requests. First classify task mode, required proof,
write boundary, complexity budget, and whether parallelism is valuable. Use a
controller fact check or subagent first unless durable session state changes
correctness.

### Fan Out And Synthesize

Use when slices are independent. Each worker has an owned scope and identical
handoff schema. The orchestrator waits at a barrier: handoff written, proof
present, registry updated. Synthesize only after the barrier.

### Adversarial Verification

Use for high-risk audits, security boundaries, schema/API contracts, production
readback, or claims that are easy to overstate. One actor produces findings;
another checks counterexamples and downgrades unsupported claims. The verifier
can be a compact evidence packet when a durable session is not needed.

### Generate And Filter

Use for proposal, prompt, design, or test-case generation. Several sessions
produce candidates against the same rubric; a separate filter session selects
or merges.

### Tournament

Use when multiple plausible architectures or plans exist. Each contender argues
for a plan with cost, risk, proof, and migration path. A judge session compares
them against the user goal.

### Loop Until Done

Use for long-running implementation, incident recovery, or flaky verification.
Each loop performs the next smallest safe step, records proof, updates state,
and stops when `stop_when` is met.

## Budget Heuristics

- Small direct task: no orchestration.
- Narrow two-path decision: 1 controller/subagent classifier, or 2 candidate
  sessions plus 1 judge only when proposals need durable records.
- Audit: start with 1-2 evidence actors plus a counterevidence check; expand to
  3-4 only when areas are independent and joinable.
- Implementation wave: start with 1 write session; expand to 2-5 only when
  paths are non-overlapping and each slice has independent proof.
- Incident: 1 runtime evidence actor, 1 fix session if authorized, and 1
  readback verifier only when the target environment or risk justifies it.

If a pattern needs more sessions than the likely value justifies, shrink the
scope or run the first wave only.
