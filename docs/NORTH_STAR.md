# Codex Conductor North Star

This document is the review standard for the `codex-conductor` skill. Use it
when changing the skill, reviewing a real orchestration run, or turning dogfood
observations into skill improvements.

## Purpose

The orchestrator exists to make complex Codex work recoverable,
evidence-driven, and low-noise across long-running sessions.

It should help a controller:

- keep the user's real goal visible while work spans threads, worktrees, and
  days
- preserve the critical path instead of waiting on non-blocking side work
- pick the smallest actor that can safely produce the needed evidence
- make durable state legible enough for a fresh controller to resume
- prevent implementation workers, reviewers, verifiers, release work, and
  install checks from collapsing into one unreviewable chat

The north-star outcome is not "many sessions". The outcome is a workflow where
the next controller can answer, from compact artifacts, what is done, what is
blocked, what proof exists, and what must happen next.

## Non-Goals

The skill should not optimize for:

- maximum fanout
- formal ceremony for small tasks
- replacing every subagent or parallel tool with a durable Codex Session
- keeping the controller thread alive because it contains old context
- using worker transcripts as the primary state store
- treating "worker launched" as meaningful progress by itself

## Operating Model

Think of the workflow as a small distributed system.

- Controller: the control plane. It owns goals, routing, workflow state,
  handoff reconciliation, proof gates, conflict handling, and closeout.
- Codex Session: a durable actor. It is useful when a slice needs its own
  lifecycle state, worktree, commit, verification trail, or recoverable handoff.
- Subagent or parallel tool: an ephemeral probe. It is useful for short-lived,
  independent, usually read-only work whose output can be compressed into a
  small answer.
- Workflow files: the durable source of truth. They replace controller memory
  and make rollover possible.
- Handoffs: the synchronization primitive. They should be compact, decisive,
  and evidence-linked.

The controller should be disposable. If losing the current chat would lose the
workflow, the skill has failed its purpose.

## Why Handoffs And Rollover Exist

This skill was shaped by a real failure mode: a controller can explode its own
context by repeatedly reading the worker sessions it created. Even when worker
sessions are well-scoped, the controller becomes expensive if it keeps pulling
thread transcripts, large outputs, old design discussion, and repeated status
reads into the main chat.

The handoff protocol is the fix for that failure mode. It intentionally asks
workers to compress their result into a small synchronization packet so the
controller can join results without importing the worker's whole transcript.
Waiting for a handoff is not ceremony; it is the mechanism that prevents the
controller from becoming the archive.

That fix introduced a second risk: if handoff arrival is slow or uncertain, a
controller may create a replacement worker too early and duplicate nearly
finished work. The replacement discipline exists to balance those two risks:

- do not routinely read full transcripts, because that recreates context
  explosion
- do not replace progressing workers just because the compact handoff is late,
  because that creates duplicate work and conflicting state
- use bounded, minimal status reads and one closeout prompt to bridge the gap
  between "no handoff yet" and "worker failed"

Controller rollover is the same idea applied to the main thread. When the
controller itself has accumulated too much context, it should write a compact
controller handoff and continue in a fresh controller thread. The durable
workflow files, not the old controller transcript, carry the truth forward.

## Design Principles

### 1. Critical Path First

Use critical-path thinking: identify the next decision or proof that actually
unblocks the goal, then keep the controller moving on it.

Do not wait for side probes just because they are running. Wait only when the
next controller action depends on their result.

### 2. Smallest Sufficient Actor

Choose the lightest actor that can safely produce the required evidence:

1. controller fact check for tiny control-plane facts
2. subagent or parallel tool for compact independent read-only probes
3. durable Codex Session for write-capable, long-running, high-stakes, or
   lifecycle-bearing work

Promote work from ephemeral to durable only when durability changes correctness,
recoverability, or review quality.

### 3. Parallelize Independence, Serialize Coupling

Parallelism helps only when tasks are independent and their results can be
joined cheaply. Use fanout/fanin deliberately:

- fanout when tasks have separate read or write surfaces, independent proof, no
  sequential decision dependency, and compact outputs
- fanin when the controller has enough handoffs or evidence to make the next
  decision
- serialize when tasks share a write set, shared contract, root config,
  dependency file, migration, release boundary, or semantic decision

More agents can make the workflow slower if coordination cost exceeds work
saved.

### 4. Durable State Over Transcript Memory

Record decisions, worker state, proof, and blockers in workflow artifacts.
Transcripts are audit material for contradictions or missing handoffs, not the
normal data path.

The controller should prefer compact handoffs, changed workflow files, commits,
and proof pointers. If it must inspect a worker thread, read the smallest recent
slice first and record the escalation as coordination noise.

### 5. Evidence Before Lifecycle Claims

Do not claim a wave, commit, release, or install state is complete because a
worker says it is done. The controller must reconcile proof: commands, diffs,
runtime readback, installed version checks, registry state, or other decisive
evidence appropriate to the task.

### 6. Replacement Is A Recovery Action

A slow handoff is not failure. Replace a worker only after classifying the
worker state as stalled or invalid, or after a bounded closeout attempt fails
and the next step is truly blocked.

If two workers overlap, choose one primary, stop the duplicate, and record the
duplication as coordination noise.

### 7. WIP Limits Beat Launch Theater

Keep active work small enough that the controller can reason about it. If every
worker requires direct transcript reading, the workflow is over-fanned-out or
under-specified.

### 8. Dogfood Should Review Behavior, Not Just Text

Dogfood examples should test whether the skill caused better orchestration in
real runs. The review target is the actual behavior:

- Did the controller choose the right actor type?
- Did it preserve the critical path?
- Did it parallelize independent work and serialize coupled work?
- Did it avoid replacing progressing workers?
- Did it keep product edits out of the controller?
- Did it consume compact handoffs instead of transcripts?
- Did it verify evidence before closeout?
- Did it record noise and use it to improve the skill?

## Parallel Vs Serial Decision Guide

Use this guide before creating subagents, Codex Sessions, worktrees, or wave
tasks.

Parallelize when all are true:

- each task has a clear owner and output contract
- write scopes are absent or non-overlapping
- the next step does not require task A's answer before task B can start
- each task can produce independent proof
- output can be summarized in a compact handoff
- failed or late output will not corrupt other work

Serialize when any are true:

- tasks touch the same file, schema, public contract, root config, dependency,
  migration, CI pipeline, release channel, or production state
- one task defines vocabulary, architecture, or acceptance criteria for the
  others
- a worker result can change the task decomposition itself
- the external system is mutable or rate-limited and needs ordered operations
- the risk is destructive, security-sensitive, or hard to roll back
- the controller cannot describe a cheap join condition

Use weak-dependency fanout when side evidence may help but should not block the
critical path. Launch compact probes, keep moving, and join only when their
results become decision-relevant.

## Review Rubric

When reviewing a real orchestrator run or a dogfood case, score the run against
these questions:

1. Goal clarity: was the user's actual goal preserved across waves?
2. Actor fit: were sessions used only where durable lifecycle state mattered?
3. Parallel/serial fit: were independent tasks fanned out and coupled tasks
   serialized?
4. Critical path: did the controller keep moving on non-blocked work?
5. State quality: could a fresh controller resume from files and handoffs?
6. Proof quality: were claims backed by decisive evidence?
7. Replacement discipline: were slow but progressing workers preserved?
8. Noise control: were duplicate launches, polling loops, transcript reads, and
   stale registry states recorded and reduced?
9. Context control: did handoffs and controller rollover prevent the main
   thread from becoming the long-term archive?

## Common Anti-Patterns

- Launch-count theater: creating sessions because a workflow feels large, not
  because each slice needs durability.
- Handoff-latency replacement: replacing a worker only because the compact
  handoff has not appeared yet.
- Handoff bypass: repeatedly reading worker transcripts instead of requiring a
  compact result packet.
- Controller-as-worker drift: the controller mutates product code because it is
  already reading context.
- Controller-context hoarding: keeping a bloated controller alive instead of
  checkpointing and rolling over.
- Transcript archaeology: using long transcript reads as routine state sync.
- Parallelizing shared contracts: letting multiple workers change schema,
  public API, dependency, root config, or release state independently.
- Finish-after-launch: treating worker creation as progress without join
  criteria, proof, or reconciliation.

## How This Fits Codex

Codex Sessions are strongest when they carry recoverable context, a worktree,
proof commands, commits, or release/install evidence. Subagents and parallel
tools are strongest when the controller needs bounded independent facts without
durable lifecycle overhead.

The skill should combine both:

- use subagents early for cheap truth reconciliation, review probes, and
  parallel reads
- promote only durable implementation, verification, review-fix, release, and
  install slices to Codex Sessions
- keep controller artifacts short enough that a new Codex controller can resume
  without the old transcript

The result should feel like a disciplined control plane, not a bigger chat.
