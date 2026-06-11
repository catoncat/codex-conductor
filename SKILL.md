---
name: codex-conductor
description: Use in Codex App when a complex task needs coordinated Codex Sessions, long-running milestones, cross-session state, worktrees, verification sessions, heartbeat continuation, project-specific constraints, or durable workflow artifacts. Assumes Codex Goal support; optional lifecycle capabilities and project profiles are discovered at runtime.
---

# Codex Conductor

Use this skill to turn a complex task into a managed network of Codex Sessions.
A Codex Session is a conversation/thread, not a subagent worker.

North star: optimize for recoverable, evidence-driven progress with minimal
coordination cost. The outcome is not "many sessions"; it is a workflow where a
fresh controller can answer, from compact artifacts, what is done, what is
blocked, what proof exists, and what must happen next.

Core rule: every long-running session owns a `Goal`. The orchestrator owns the
overall goal; each execution session owns its assigned slice goal.

Core role boundary: when this skill is invoked because the user asks for an
orchestrator, controller, session orchestration, or multi-session workflow, the
current thread is the controller by default.

The controller owns planning, Goals, workflow state, task prompts, registry,
handoff reconciliation, proof verification, lifecycle decisions, and final
closeout. The controller does not own product implementation.

The controller must not edit product code, tests, runtime config, schema,
package/dependency files, repository implementation files, or production state
by default. Write-capable implementation belongs to execution sessions or to a
thread explicitly assigned an implementation-slice role.

Second core rule: the orchestrator thread is disposable. Durable workflow files
are the source of truth; the controller conversation must not become the
archive. Long controller threads should checkpoint to files and roll over to a
fresh controller before they become the largest token consumer.

## When To Use

Use this for complex work that needs durable coordination, for example:

- broad audit, migration, refactor, incident follow-up, launch plan, or system redesign
- work that spans multiple Codex Sessions or days
- tasks where evidence, state, and handoffs must survive context compaction
- user asks to create sessions, manage sessions, use Goal, use heartbeat, or build a workflow

Do not use it for single-turn fixes, small scoped code changes, or ordinary
parallel subagent fanout inside one session.

## Capability And Project Profile Discovery

Before choosing a workflow shape or writing worker prompts, discover the local
runtime profile. The conductor runs in Codex App and assumes Goal support, but
it must not assume Mainline, GitHub, a specific package manager, a database, a
thread launcher, or any host-local lifecycle tool. It should use optional
capabilities when they are present and fall back to durable files plus explicit
prompts when they are absent.

Read only the files that exist and are relevant:

1. User instructions in the current conversation.
2. Active workflow artifacts, especially any existing `workflow-state.md`.
3. Project conductor profile:
   - `.codex-conductor/project.md`
   - `.codex-conductor/verification.md`
   - `.codex-conductor/worktree.md`
   - `.codex-conductor/lifecycle.md`
4. Repository agent or contributor instructions, such as `AGENTS.md`,
   `CLAUDE.md`, `GEMINI.md`, `CONTRIBUTING.md`, or local workflow docs.
5. Host conductor profile:
   - `$CODEX_CONDUCTOR_HOME/host.md`
   - `~/.config/codex-conductor/host.md`
6. Cheap live detection: lockfiles, package scripts, `.git`, worktree status,
   available CLIs, issue tracker remotes, and obvious test scripts.

The project profile is the right place for project-specific rules such as
package manager, worktree environment, database ownership, verification tiers,
forbidden commands, branch policy, issue labels, or PR conventions. Do not put
project-specific rules into this public skill. The skill owns the protocol; the
project profile owns the local facts.

If a profile file contradicts the current user instruction, the user
instruction wins. If a project profile contradicts a host profile, the project
profile wins inside that repository.

## Clarify The Assignment First

Before creating a durable workflow, optimize the user's initial prompt into a
clear orchestration brief. Do not turn an ambiguous request into sessions,
worktrees, branches, issues, PRs, or handoffs just because the user mentioned
coordination.

First resolve discoverable facts from the current environment: active workflow
state, repo instructions, project profile, branch/worktree status, obvious
issue/PR links, and relevant docs. Ask the user only for decisions that cannot
be discovered locally and would materially change scope, delivery, risk, or
authorization.

The controller should be able to restate the brief before choosing a harness:

```markdown
## Orchestration Brief
goal:
deliverables:
in_scope:
out_of_scope:
acceptance_criteria:
constraints:
known_artifacts:
open_questions:
recommended_first_harness:
```

If important ambiguity remains, ask one to three focused questions before
launching sessions. Good questions choose between meaningful tradeoffs: final
deliverable shape, read-only audit versus implementation, allowed write
surface, verification depth, deadline/budget, production or GitHub mutation
authorization, and whether unresolved work should become follow-up backlog.

Do not ask questions already answered by the repository or current workflow
files. Do not ask merely to delay. If the user explicitly asks to continue
nonstop, make reasonable assumptions, record them in the brief, and continue
until a real blocker or authorization boundary appears.

When the user answers the clarifying questions and confirms the scope, the
clarification gate is closed. Do not create a second "ask the user to confirm
launch" gate unless the next action crosses a new boundary that was not
covered by the clarified brief, such as product-code mutation, production
mutation, push/PR/merge/deploy, a larger session budget, or access to sensitive
data. If the next step is already named, unblocked, inside the complexity
budget, and within the clarified boundary, write the control plane if needed
and immediately continue it according to ownership: execute controller-owned
control-plane, read-only, or verification work in the controller; launch or
delegate write-capable implementation, review-fix, schema, runtime, production,
or other non-controller work. Treat stopping after only producing a plan as a
process miss unless the user explicitly asked for plan only, the budget is
exhausted, or the next action needs fresh authorization.

### Project Constraints Capsule

For every durable workflow, write a compact `Project Constraints Capsule` into
`workflow-state.md`. For every worker session, copy only the relevant subset
into the worker task and prompt. The capsule should include:

```markdown
## Project Constraints Capsule
source_files:
- .codex-conductor/project.md
- AGENTS.md
capability_mode: managed | portable | files-only
package_manager:
verification_tiers:
- tier:
  use_when:
  commands:
  skip:
worktree_bootstrap:
- use_when:
  steps:
  forbidden:
lifecycle:
- commits:
- branches:
- issues_prs:
forbidden_commands:
- ...
worker_prompt_must_include:
- ...
```

Use the capsule to prevent repeated environment rediscovery. If a worker hits a
setup failure that is likely to recur, the controller should update the
workflow capsule or propose a project-profile update instead of letting every
worker rediscover the same failure.

Read `references/configuration.md` when setting up a new project profile,
debugging repeated worker environment failures, or preparing this skill for use
by another repository.

### Worktree Bootstrap Tiers

Classify verification before launching write-capable workers:

- `docs-only`: documentation or control-plane edits. Do not bootstrap runtime
  dependencies unless a direct docs check requires them.
- `script-or-guard`: scripts, guards, or local tooling checks. Run the direct
  script and targeted verification. Do not bootstrap databases unless the
  script requires one.
- `unit-or-component`: code with isolated unit/component proof. Install
  dependencies only when missing and required by the proof.
- `integration-runtime`: backend, database, browser, or runtime integration
  proof. Bootstrap the declared environment before running focused tests.
- `schema-or-migration`: schema, migration, generated-client, or persistent
  data changes. Use isolated data stores unless the project profile explicitly
  allows sharing.
- `production-or-ops`: live systems. Stay read-only until the user and project
  rules authorize mutation.

Worker prompts must state the tier, required bootstrap, checks to run, and
checks to skip. Skipping irrelevant checks is part of correctness; broad
verification can be deferred to the integration controller when the project
profile says so.

## Choose A Task Mode

Pick a mode before writing task prompts. The mode determines expected proof,
write scope, worktree policy, and closeout rules.

- `audit-track`: read-only evidence collection with owned output files.
- `decision-packet`: read-only recommendation for a narrow decision.
- `inventory-packet`: read-only inventory or contract matrix for later slicing.
- `implementation-slice`: write-capable work in a bounded component.
- `evidence-session`: runtime/dogfood/verification evidence only; no commit.
- `review-session`: read-only PR/diff/design review; findings only.
- `review-fix-session`: bounded fixes after review findings.
- `commit-prep`: prepare a commit handoff without mutating lifecycle.

Default actor by mode:

| Mode | Default actor |
| --- | --- |
| `audit-track` | subagent/parallel tool for lightweight read-only work; worker session only when durable state is needed |
| `decision-packet` | subagent/parallel tool for narrow decisions; worker session only when the decision needs durable review history |
| `inventory-packet` | subagent/parallel tool for bounded inventory; worker session only when the packet is large or long-running |
| `implementation-slice` | execution session, not controller |
| `evidence-session` | verifier/evidence session |
| `review-session` | subagent for fast review; review session only for high-stakes or durable review packets |
| `review-fix-session` | fix execution session |
| `commit-prep` | controller only for lifecycle handoff; no product edits |

For long-running audits, treat the first pass as draft until counterevidence has
been checked and unsupported findings are downgraded to open questions.

## Dynamic Harness First

Build a task-specific harness, then launch sessions. Do not start from a fixed
fanout shape. This skill is a coordination framework, not a runbook.

Use existing capabilities as building blocks:

- direct controller execution only as an explicit exception, described below
- subagents or parallel tools inside the controller session for lightweight,
  read-only exploration, fast review, file/diff inspection, and draft packets
- exploratory or audit sessions when independent evidence can be gathered in
  parallel and the result needs durable thread/worktree/handoff state
- implementation sessions in isolated worktrees when write scopes are separable
- verifier or review sessions when the result is high-stakes or easy to
  overclaim
- heartbeat continuation when a specific thread should resume later
- controller rollover when durable state is ready and controller context is
  becoming the largest cost
- direct transcript reads only for missing state, contradiction, or process
  forensics

1. Classify the request by task mode, risk, proof surface, and whether work can
   proceed in parallel.
2. Pick the smallest harness that gives a real benefit. Direct execution is
   better for narrow tasks.
3. Add independent verification when the result is high-stakes, expensive to
   unwind, or prone to premature closure.
4. Add a complexity budget: max active sessions, required control-plane files,
   expected proof, gates that must stay separate, gates that may be joined,
   allowed elapsed time, and stop conditions.
5. Save the harness as workflow artifacts so a later session can resume without
   reconstructing intent from chat history.

Pick the smallest shape that preserves correctness, then resize it as evidence
changes. Do not launch sessions, create worktrees, add a verifier, or roll over
the controller just because the template contains that capability.

Default budget: one write-capable worker plus at most one independent reviewer
or verifier. Exceed it only when work has independent write surfaces,
independent proof, and a cheap join condition.

Treat the budget as a hard guard until fresh evidence changes it. Before adding
a worker, gate, template, checklist, runbook, or durable artifact, answer:

- what distinct risk or proof it covers that existing state cannot cover
- why it must be durable now instead of a note in an existing file
- what it replaces, merges, unblocks, or prevents

### Autonomous Progress And Backlog

The controller is not a launch-and-stop actor. When it creates or discovers a
multi-step workflow, it must keep driving the program until the orchestrator
Goal is complete, genuinely blocked, out of budget, or waiting on an explicit
authorization boundary.

Use the Codex Goal as the workflow-level contract, not as a per-wave checklist.
Do not mark the orchestrator Goal complete just because the current wave,
milestone, or batch is integrated. The Goal can complete only when the workflow
backlog is empty or every remaining item is explicitly deferred with evidence.

If the controller splits work into waves, milestones, phases, or follow-up
passes, record the backlog in existing state, normally `workflow-state.md` or
`milestone-plan.md`:

```markdown
## Program Backlog
current:
completed:
pending:
blocked:
next:
launch_condition:
```

Closing a wave is not closing the workflow. After every wave closeout,
checkpoint state, check the backlog, and continue. If the next item is known,
unblocked, and inside the complexity budget, choose the next action by
ownership:

- execute controller-owned control-plane, read-only evidence, reconciliation,
  verification, task-writing, or session-launch work in the controller
- launch or delegate write-capable implementation, review-fix, schema, runtime,
  production, or other non-controller work to the right execution session
- use direct controller execution only under the explicit exception rule

If controller context is too large, roll over with an exact next action such as
"launch wave 2 from `<task-file>`"; do not stop with a status report that leaves
the next wave implicit.

Stop only when one of these is true:

- the orchestrator Goal is complete under the workflow completion rules
- every remaining backlog item is blocked, deferred, or requires user approval
- the complexity budget is exhausted and the next step would exceed it
- no autonomous next action exists after checking workflow state and registry
- rollover is required, and the outgoing handoff names the exact next action

### Identity And Recovery

No entity without necessity. Identity is a recovery index, not a workflow
harness and not a reason to create sessions, worktrees, branches, issues, PRs,
labels, files, or schemas.

When the chosen shape already requires durable objects, give them names that
map back to the same workflow state and registry row. Reuse existing artifacts:
write identity fields into `workflow-state.md`, exact object references into
`session-registry.md`, and assigned names into task prompts and handoffs. Do not
create `identity.md`, `naming.md`, a second registry, or a separate identity
schema by default.

Use the smallest identity surface that lets a later controller recover from any
visible object:

```markdown
## Identity
workflow_slug:
workflow_label:
run_id:
project_label: optional
naming_overrides: default | .codex-conductor/project.md | host profile
```

- `workflow_slug` is the compact machine-searchable slug used in paths and
  branch names.
- `workflow_label` is the short human-readable label used in session, issue,
  and PR titles.
- `run_id` is a short stable discriminator for repeated runs of the same
  workflow; use it in branch/worktree paths when collisions are likely.
- `project_label` is optional and only needed when Codex conversation titles
  would be ambiguous outside the repository.

For visible Codex session titles, prefer a readable compact shape:

```text
<Workflow Label>: <Role> - <Task Label> [<task-id>]
```

Add `[<project>]` only when the Codex conversation list is cross-project and
ambiguous. Add strong temporary status words such as `BLOCKED` or `NEEDS USER`
only when they change where the user should look; keep ordinary status in
`session-registry.md`.

For branches and worktrees, include enough identity to recover the registry row
without enforcing one global template. A good default includes
`workflow_slug`, short `run_id`, and `task_id`. Handoffs should stay
`handoffs/<task-id>.md`; expand to `handoffs/<task-id>-<role>-<attempt>.md`
only when retries or multiple roles would collide.

Do not create GitHub labels by default. If a project already uses labels for
workflow recovery, keep them low-noise, such as `cc:<workflow-slug>` or
`cc-run:<short-id>`. Role, wave, milestone, or phase labels are project-profile
overrides, not public-skill defaults.

### Shrink Mode

Enter shrink mode when the user pauses, asks for closeout, duplicate worker
noise appears, the controller notices coordination cost growing faster than
task progress, or the remaining work has become a single serial proof chain.

In shrink mode:

- do not launch new sessions or worktrees unless the existing actor is
  `stalled`/`invalid` and the next proof is truly blocked
- do not create new artifact classes, templates, runbooks, or checklists; fold
  state into `workflow-state.md`, `session-registry.md`, `milestone-plan.md`, or
  the latest handoff
- merge adjacent gates unless they protect different independence boundaries
- run the next proof or write the controller checkpoint, then check the program
  backlog before stopping. If the next backlog item is unblocked and within the
  existing budget, continue without asking. Ask only when continuing would
  require a new entity, a larger budget, or an authorization boundary.

Separate review and verification gates only when independence matters: for
example a semantic review can change the verification set, verification needs a
different runtime/install environment, or lifecycle/release proof must be kept
apart. If a review only confirms "no unresolved blocker" on the same checkout,
and verification is a focused command/readback set, use one compact evidence
packet instead of launching another durable gate.

When choosing a shape, read only the relevant files:

- `references/configuration.md` for project profiles, host profiles,
  capability discovery, and Project Constraints Capsule examples.
- `references/harness-patterns.md` for orchestration patterns and failure modes.
- `templates/readonly-audit.md` for evidence-first audits.
- `templates/implementation-wave.md` for parallel implementation slices.
- `templates/review-fix-wave.md` for review followed by bounded fixes.
- `templates/incident.md` for runtime incidents and recovery.
- `templates/design-tournament.md` for multiple competing designs or proposals.

### Session Vs Subagent Selection

Codex Sessions and subagents solve different coordination problems.

Use a subagent or parallel tool when the task is short-lived, read-only, and the
controller only needs a compact answer:

- quick diff or file review
- risk scan for one implementation idea
- small evidence lookup
- fast decision-packet draft
- parallel file reads or API/doc checks
- work that can be rerun cheaply if the answer is incomplete

Use a Codex Session when the task needs durable lifecycle state:

- independent `Goal`
- independent worktree, branch, commit, release, install, or verification trail
- write-capable implementation or review-fix work
- long-running verification or runtime evidence
- formal handoff tracked in `session-registry.md`
- cross-wave work that a future controller must recover without this chat

Default to this ladder:

1. Controller does control-plane work and small fact checks.
2. Subagents or parallel tools handle lightweight read-only side quests.
3. Codex Sessions handle durable workers, implementation, verification, release,
   install, and high-stakes review.

Do not turn every read-only packet into a Codex Session. For early truth
reconciliation or fast review, use subagents first unless the workflow needs a
thread id, independent worktree, or durable handoff.

### Parallel Vs Serial Planning

Before creating subagents, Codex Sessions, worktrees, or wave tasks, classify
the dependency shape:

- `parallel`: tasks are independent, have clear owners, no overlapping writes,
  independent proof, and compact joinable outputs.
- `serial`: tasks share a write set, schema, public contract, dependency, root
  config, CI/release/install boundary, production state, or a semantic decision
  that must be settled first.
- `weak-dependency`: side evidence may help, but the controller can keep moving
  on the critical path and join the evidence only when it becomes relevant.

Parallelize independence, not uncertainty. Use fanout when the output contract
and join condition are clear. Serialize when one result can redefine the next
task, when workers would touch shared contracts, or when external mutable state
requires ordered operations.

For each fanout, state the owner, allowed scope, expected compact output,
verification requirement, and join condition. If the controller cannot describe
a cheap join condition, keep the work serial or use one small probe first.

### Direct Controller Execution Exception

Direct controller execution is an exception, not the default.

It is allowed only when all of these are true:

- no durable multi-session workflow is active or needed
- the user did not explicitly frame the current thread as the controller,
  orchestrator, or coordinator
- the write scope is tiny, local, and lower risk than launching a worker
- no separate execution session would materially improve isolation, review, or
  proof quality
- before mutation, the controller says it is switching into direct execution and
  why delegation is being skipped

If the user explicitly asks this thread to be the orchestrator or controller,
prefer launching, continuing, or handing off to an execution session. A phrase
such as "no need to ask me for approval" removes permission pauses; it does not
remove the controller/worker role boundary.

When the workflow is nontrivial, tell the user briefly:

- the chosen shape and why it fits this task
- what state/proof will be canonical
- when the controller will escalate, read transcripts, or roll over

## Controller Mutation Gate

Before the controller mutates any file, stages, commits, changes configuration,
or runs a write-capable command, it must classify the target:

1. Is the target a control-plane artifact owned by the orchestrator, such as
   workflow-state, registry, task prompts, operating rules, or handoffs?
   - If yes, proceed only inside the workflow scope.
2. Is this thread explicitly assigned as the execution session for this slice?
   - If yes, follow the implementation-slice rules.
3. Otherwise, stop and delegate to an execution session, or record an explicit
   direct-execution role switch before mutating.

Product code, tests, schema, configs, package/dependency files, and repository
implementation files are not controller-owned artifacts.

## Durable Workspace

Create a neutral workflow artifact directory. Do not default to a review name.
When sessions run in isolated worktrees, designate one canonical control-plane
directory owned by the orchestrator. `session-registry.md` and
`workflow-state.md` are authoritative only there. Execution worktrees may write
their assigned handoffs or local evidence, but the orchestrator must reconcile
status back into the canonical control plane.

Default:

```text
docs/workflows/<yyyy-mm-dd>-<short-slug>/
```

If the project already has a better current-task location, use it. Use
`docs/reviews/...` only when the task is actually a review/audit and the project
already treats that as the right place.

Minimal starting set for most workflows:

- `workflow-state.md`
- `session-registry.md` when sessions exist
- `tasks/<task-id>.md` and `prompts/<task-id>.md` only for launchable sessions
- compact handoffs

Common optional files:

```text
README.md
operating-rules.md
workflow-state.md
session-registry.md
milestone-plan.md
tasks/<task-id>.md
prompts/<task-id>.md
handoffs/<task-id>.md
handoffs/controller-*.md
```

Do not create completion audits, verification runbooks, acceptance checklists,
or templates by default. Use them only when the workflow has multiple externally
visible layers, repeated proof packets, or a high-stakes handoff whose
acceptance criteria would otherwise be easy to overclaim.

Keep files terse. They are a control plane, not a diary. If adding a file would
mainly restate another artifact, merge the content into the existing file.

## State Sync And Controller Rollover

The controller/orchestrator thread is the highest risk for token runaway. Worker
sessions are bounded by task scope, but the controller can become a large
context carrier if it repeatedly reads worker transcripts, workflow copies,
large command output, and old design discussion.

Use this rule: worktrees isolate code changes; handoffs and commits synchronize
facts; worker transcripts are exceptional audit material, not routine state.

The controller should normally consume only `session-registry.md` status rows,
compact worker final handoffs, commit/diff/proof-command pointers, and changed
canonical workflow files.

If controller context is already large, do not compensate by reading more worker
context. First checkpoint controller state, then continue from compact workflow
artifacts or roll over to a fresh controller.

For workflows that use worker sessions, each worker final handoff should be
short enough for the controller to read directly, usually 50-100 lines. Use this
compact shape:

```markdown
# Worker Handoff
task_id:
thread_id:
cwd:
branch:
commit:
status: complete | blocked | needs-review
conclusion:
- ...
changed_files:
- path
proof:
- command: ...
  result: pass/fail
  evidence: short decisive line only
controller_updates:
- registry status should become ...
- next task is ...
- risk/blocker is ...
do_not_read_transcript_unless:
- handoff missing
- commit/proof missing
- scope crossed
- result contradicts registry
```

Long logs, full test output, large API JSON, and long diffs should be stored as
artifacts or referenced by command/path. Do not paste them into handoffs or
registry rows.

Direct worker-thread reads are an exception path. Start with the smallest recent
status read the tool supports, preferably one recent turn without tool outputs.
Use full outputs only when the handoff/proof is missing, contradictory, or the
user asks for process forensics. Record that escalation as `noise_events`.

After each wave or major integration in a long workflow, write a compact
controller checkpoint for the next controller:

```markdown
# Controller Checkpoint
current_wave:
completed:
pending:
active:
blocked:
decisions:
proof_verified:
next_actions:
do_not_reopen:
```

Start the next unblocked wave in the current controller unless rollover is
needed for context size, tool limits, or a clean control-plane handoff. A fresh
controller should read only the workflow overview, `workflow-state.md`,
`session-registry.md`, the latest controller checkpoint, and changed handoffs,
then perform the exact next action named in the checkpoint. Do not keep a
controller alive just because it has history, but do not use rollover as a
reason to leave the next wave implicit.

## Orchestrator Setup

1. Build the orchestration brief from the user's prompt, live repo/workflow
   facts, project profile, and any necessary clarifying answers.
   - Restate goal, deliverables, scope, acceptance criteria, constraints, known
     artifacts, open questions, and recommended first harness.
   - Ask focused clarification questions before launching durable objects when
     the brief is not decision-ready.
   - If proceeding with assumptions, write them into `workflow-state.md` and
     worker prompts.
2. Create or align the orchestrator `Goal` from the refined brief.
   - If no active Goal exists, call `create_goal`.
   - If a Goal exists, confirm it matches this workflow before using it.
   - The orchestrator Goal covers the whole workflow backlog, not only the
     current wave.
   - Do not call `update_goal complete` while `workflow-state.md` or
     `milestone-plan.md` has pending unblocked work.
   - If all remaining work is blocked or deferred, keep the Goal active unless
     the platform blocked-goal rule is satisfied or the user explicitly accepts
     the deferral.
3. Decide the workflow shape before creating artifacts:
   - direct controller work
   - controller plus subagents/parallel tools for lightweight read-only probes
   - controller plus read-only evidence sessions
   - parallel implementation sessions
   - implementation plus verifier/reviewer
   - heartbeat continuation
   - controller rollover
4. Set the complexity budget and shrink trigger before creating task prompts or
   optional artifacts.
5. Classify the dependency shape before fanout: `parallel`, `serial`, or
   `weak-dependency`.
6. Choose the workflow directory and create only the files needed for that
   shape.
7. Build the Project Constraints Capsule from profile files and live detection.
8. Write the refined brief, global objective, non-goals, capsule, identity,
   program backlog, Goal completion rule, evidence rules, allowed write scope,
   forbidden actions, verification commands, and stop lines.
9. Write the chosen workflow shape, complexity budget, dependency shape,
   communication rule, state source of truth, worker-handoff rule, shrink
   trigger, controller-rollover rule, next-wave launch condition, and naming
   override source in `workflow-state.md`.
   - A launch condition must describe the next objective condition, not repeat
     a clarification question that the user has already answered.
   - Do not write "wait for user confirmation to start" when the user's
     clarified request already asks the controller to do the work. Use that
     only for a real authorization boundary.
10. Break work into session-sized tasks only where sessions are useful. Before
   creating a Codex Session for read-only work, ask whether a subagent or
   parallel tool can answer it with a compact result. A good task has:
   - task mode
   - verification tier
   - relevant Project Constraints Capsule subset
   - clear objective
   - exact read paths
   - allowed write paths
   - forbidden changes
   - bootstrap steps and checks to skip
   - expected output
   - proof commands or evidence requirement
   - escalation condition
   - commit policy
   - budget and stop condition
   - noise/efficiency reporting requirement
11. Register delegated or durable tasks in `session-registry.md`.
12. Create Codex Sessions with the thread tool when available. If no Codex
   thread tool is available, write starter prompts and tell the user they must
   launch them manually.
13. After creating each session, immediately update the registry with the actual
   title, thread id, cwd/worktree path, branch, task mode, role, allowed
   writes, budget, and next proof checkpoint. Do not rely on planned paths
   after launch.

## Per-Session Goal Protocol

Every starter prompt must require:

```text
1. First call `create_goal` with the assigned task objective.
2. Read the workflow files listed below.
3. Work only inside the assigned scope.
4. Update the Goal to complete only when the task is verified and no required
   work remains.
5. Mark the Goal blocked only after the same blocking condition repeats under
   the platform's blocked-goal rules.
6. Write a compact handoff to the requested location or return it in the exact
   requested format.
```

Goal names should include the workflow slug and task id:

```text
<workflow-slug>: <task-id> - <objective>
```

The orchestrator should read session status through the thread tool when
available, but it should not treat a chat summary as proof. Proof comes from
commands, files, runtime evidence, or an explicit handoff.

When a thread id is known, read it directly with the smallest accepted argument
set first. Do not search by thread id unless direct read fails. If thread
tooling is noisy or unavailable, fall back to durable handoffs and command/file
proof. Do not expand into broad transcript or filesystem searches unless the
task explicitly requires that evidence.

## Worker Progress And Replacement

Do not replace an active worker merely because its handoff has not landed yet.
Missing handoff is a sync gap, not by itself a worker failure.

Before launching any replacement worker, classify the existing worker state:

- `progressing`: recent thread status shows substantive evidence discovery,
  analysis progress, implementation progress, or a near-complete conclusion.
- `closing`: the worker has enough evidence and is only slow to write the
  compact handoff.
- `stalled`: no recent useful progress, repeated tool/runtime failure, or no
  recoverable state.
- `invalid`: the worker crossed scope, writes outside its allowed boundary,
  contradicts known evidence, or cannot produce required proof.

If the worker is `progressing` or `closing`, keep it as the primary worker. The
controller may send one compact closeout prompt asking it to stop expanding,
write the required handoff from existing evidence, and return final. Then wait
a reasonable window for the task type instead of launching a replacement.

Replacement preflight is mandatory. Before launching a replacement, the
controller must have:

- an actual thread id, or evidence that the launcher never created a recoverable
  thread; an unresolved pending worktree id alone is launcher uncertainty, not
  worker failure
- one minimal direct status read unless the worker is already known invalid
- one closeout prompt unless the worker crossed scope or would risk bad writes
- a written reason why the next step is blocked and existing evidence cannot be
  recovered
- a stop plan in case the original worker later appears

Skipping this preflight is a process miss and must be recorded as
`noise_events`.

Replacement is allowed only when one of these is true:

- the worker is `stalled` after a direct status read and one closeout prompt;
- the worker is `invalid`;
- the next workflow step is blocked and the worker has no recoverable evidence;
- the handoff/proof remains missing after the closeout window and the thread
  status does not show fresh progress;
- the result is contradictory and cannot be reconciled from existing proof.

If a replacement is launched and the original worker later appears with useful
progress, choose one primary worker immediately, instruct the other to stop
without writing repo files, and record the duplicate launch as `noise_events`.

## Worktree And Commit Policy

Large workflows should not accumulate one huge dirty worktree. Small verified
checkpoints reduce conflict cost and make recovery easier.

Default for write-capable execution sessions:

- use an isolated git worktree when the task may overlap with other sessions,
  run for a long time, or touch more than one bounded component
- commit each coherent, verified slice when the task policy allows commits
- keep commits scoped to the assigned files and objective
- prefer multiple small commits over one large commit when each commit has its
  own proof
- stop before push, PR, merge, release, deploy, or production action unless the
  user and repository workflow explicitly allow that boundary

When using a thread launcher, do not pre-fill task files with manual worktree
paths unless those paths are actually usable by the launcher. After launch, the
execution session must confirm `pwd`, repository root, branch, and `git status`
before edits; the orchestrator then records the actual values.

For read-only sessions, do not create commits; write the handoff and update the
Goal instead.

If checks block commit, record the blocker in the task file and handoff instead
of leaving unexplained dirty work.

## Noise And Efficiency

Every execution handoff should include:

- `noise_events`: failed command paths, wrong tool routes, repeated status
  polling, stale registry, path/cwd mistakes, provider/API friction, or any
  detour not materially needed for task proof or safety.
- `efficiency_notes`: elapsed time, token usage if available, expensive
  commands, retries, whether parallel work saved or cost time, and whether the
  controller had to read this session directly instead of using a compact
  handoff.
- `tool_fit`: which tools fit the project and which created friction.

Noise is not worktree parallelism. Noise is duplicated or stale state, avoidable
command/tool failure, unclear lifecycle, unnecessary polling, or proof that is
hard to reconcile later. Controller context bloat is also noise: if the main
orchestrator had to keep old wave details in chat instead of files, record the
missing checkpoint or rollover rule.

If the controller directly performs work that should have belonged to an
execution session, record it as a process miss in `noise_events` and final
closeout:

- why delegation was skipped
- what mutation happened in the controller
- whether a review or verifier session is now required
- how the workflow should avoid repeating it

## Heartbeat Protocol

Heartbeats are for waking the same Codex thread later. They are not a task
queue, and they can run too long if the stop rule is vague.

Use this conservative rule:

- one active heartbeat per Codex Session/thread by convention
- multiple Codex Sessions may each have their own heartbeat
- the platform may allow multiple heartbeat automations on one thread, but this
  workflow treats duplicates as unsafe unless the user explicitly asks for them
- if a heartbeat already exists for the thread/workflow, update it instead of
  creating another
- do not create a heartbeat without an active Goal and a written stop condition
- store heartbeat state in `workflow-state.md` or the task file:
  `heartbeat_status`, `last_wakeup`, `wakeups_used`, `wakeups_max`,
  `max_steps_per_wakeup`, `stop_when`
- define the per-wakeup budget before scheduling the heartbeat. Default to one
  controller checkpoint-level pass per wakeup; allow multiple steps only when
  `max_steps_per_wakeup` or an equivalent wakeup budget is explicitly written.
- when the Goal is complete/blocked, or `wakeups_used >= wakeups_max`, pause or
  delete the heartbeat

Heartbeat prompt shape:

```text
Continue the current Codex Session for <workflow-slug>/<task-id>.

Before doing work:
- Read the active Goal.
- Read <workflow-state.md>, <milestone-plan.md> if present, registry, and this
  task file.
- If the Goal is complete/blocked, or there is no autonomous next step, pause or
  delete this heartbeat and report why.
- If wakeups_used has reached wakeups_max, pause or delete this heartbeat and
  report the remaining handoff.
- Read max_steps_per_wakeup or wakeup_budget. If absent, use 1 checkpoint-level
  controller pass.

Then run a bounded wakeup pass: do the next smallest safe checkpoint-level
controller action, update state, check the program backlog, and stop unless a
written per-wakeup budget explicitly allows another step. Heartbeat wakeups may
continue the workflow across wakeups; they must not become an unbounded task
queue inside a single wakeup. Stop on the written stop condition, wakeup budget,
blocker, or authorization boundary.
```

Prefer short-lived heartbeats for recovery or continuation. For detached,
scheduled workspace jobs, use cron automation instead.

## Creating Codex Sessions

For each task, create a prompt that is self-contained but small:

```text
You are executing <workflow-slug>/<task-id>.

Use the codex-conductor skill.

Identity:
- workflow_slug: <workflow-slug>
- workflow_label: <human-readable workflow label>
- run_id: <short stable run id, or "none">
- project_label: <optional project label, or "none">
- task_id: <task-id>
- task_label: <human-readable task label>
- role: <controller | integration | worker | verifier | reviewer>
- assigned_session_title: <title to use when a title tool is available>
- assigned_handoff: <workflow-dir>/handoffs/<task-id>.md
- assigned_branch: <branch name if already created, or "confirm actual branch">
- assigned_worktree: <path if already created, or "confirm actual cwd">

Goal:
<assigned objective>

First action:
Call create_goal with this objective.

Read:
- <workflow-dir>/README.md
- <workflow-dir>/operating-rules.md
- <workflow-dir>/milestone-plan.md
- <workflow-dir>/tasks/<task-id>.md
- Any project profile files named by the task capsule

Task mode:
<audit-track | decision-packet | inventory-packet | implementation-slice |
evidence-session | review-session | review-fix-session | commit-prep>

Project Constraints Capsule:
<only the package manager, verification tier, bootstrap, lifecycle, and
forbidden-action rules relevant to this task>

Verification tier:
<docs-only | script-or-guard | unit-or-component | integration-runtime |
schema-or-migration | production-or-ops>

Allowed writes:
<exact paths or "read-only">

Forbidden:
<schema/root config/production/other sessions' files/etc.>

Bootstrap:
<steps to perform before verification, or "none">

Required proof:
<commands, file evidence, runtime readback, or "read-only evidence only">

Skip:
<irrelevant checks that should not be run for this tier>

Budget:
<max sessions/time/token note/retry limit/stop condition>

Worktree/commit:
- If this task can write files and may overlap with other sessions, use an
  isolated worktree.
- Confirm the actual cwd, repository root, branch, and `git status` before
  edits; do not invent replacement names when assigned names are missing or
  unusable.
- After each coherent verified slice, create a scoped commit if authorized.
- Do not push, open PRs, merge, deploy, or touch production unless explicitly
  authorized.

Heartbeat:
If heartbeat tooling is available and this task will continue unattended, create
or update one heartbeat for this thread with the stop rules in the task file.

Handoff:
Write or return:
- task_id
- thread_id
- session_title
- cwd
- branch
- commit, or `none` with reason
- status: `complete`, `blocked`, or `needs-review`
- conclusion
- changed_files
- proof commands and short decisive results
- controller_updates
- blockers or decisions needed
- noise_events
- efficiency_notes
- tool_fit
- do_not_read_transcript_unless
- remaining_or_followup_work_if_seen
```

Use the thread-title tool when available to title each session with the assigned
session title from the task identity. Pin only the orchestrator and currently
active sessions. If the assigned title is missing, use the compact default
`<Workflow Label>: <Role> - <Task Label> [<task-id>]` and record the exact title
in `session-registry.md`.

For shell and remote-runtime tasks, prefer robust command shapes:

- quote URLs and shell-sensitive arguments (`?`, `&`, `#`, spaces)
- for nontrivial remote scripts, write a temporary script file and run it rather
  than sending a very long one-liner or `data:` import
- for timing/profiling, avoid noisy logging unless log output itself is the
  evidence target
- if a wrapper command fails once on a known flaky path, pivot to the reliable
  direct readback surface and record the wrapper failure as noise

## Orchestrator Loop

Use this loop only when the chosen workflow has durable state or multiple
sessions. Treat it as a control-plane checklist, not a script. On each pass,
reconsider whether to keep the current shape, shrink to direct controller work,
add a subagent/parallel probe, promote work to a Codex Session, roll over, or
stop.

1. Read `workflow-state.md`, `session-registry.md`, and active task files.
2. Reconfirm the current workflow shape and tell the user if it changed.
3. Check the complexity budget and shrink trigger before launching anything.
4. Check whether the controller should roll over before reading more history.
5. Read changed handoffs, commits, and proof files before reading active Codex
   Sessions.
6. Read active Codex Sessions only when handoff/proof is missing, stale, or
   contradictory. Start with the smallest recent status read when the tool
   supports it. Classify the worker as `progressing`, `closing`, `stalled`, or
   `invalid` before changing the wave shape. If it is `progressing` or
   `closing`, do not replace it; send at most one closeout prompt and wait. Use
   full tool outputs only for forensics or unresolved proof gaps, and record
   that escalation as noise.
7. Update registry statuses:
   - `planned`
   - `launched`
   - `active`
   - `needs-check`
   - `blocked`
   - `verified`
   - `integrated`
   - `closed`
8. Verify completed work yourself. Do not accept "done" without evidence.
9. Integrate non-conflicting results.
10. Write a compact controller checkpoint after each wave or major integration.
11. Archive or unpin sessions that are closed.
12. Check `workflow-state.md` or `milestone-plan.md` for pending backlog before
    stopping.
    If the only pending item is gated by a generic "user confirmation to
    launch" condition, re-evaluate it against the current conversation. If the
    user has already clarified scope and authorized the workflow, remove or
    reinterpret that gate and continue.
13. If the next planned wave or task is unblocked and inside the complexity
    budget, continue it by ownership. Execute controller-owned control-plane,
    read-only, reconciliation, verification, task-writing, and session-launch
    work in the controller. Launch or delegate write-capable implementation,
    review-fix, schema, runtime, production, or other non-controller work to an
    execution session unless the direct-controller-execution exception is
    explicitly invoked and recorded. If rollover is needed, write the exact
    next-wave launch instruction and continue in the fresh controller instead
    of reporting workflow completion.

On every loop, check registry drift:

- known thread id exists but registry says `planned` or `not started`
- handoff exists but registry proof is pending
- workflow copies exist in multiple worktrees and disagree
- Goal complete but canonical state is stale

If drift exists, mark the registry stale, reconcile from handoff/proof, and only
then synthesize or launch the next wave.

After a wave closes, never infer workflow completion from `session-registry.md`
alone. The registry only knows launched sessions; the workflow backlog or
milestone plan owns not-yet-launched waves. A closed wave with pending unblocked
backlog means continue, not final closeout.

If a session crosses scope, changes shared contracts unexpectedly, or cannot
produce proof after a closeout prompt and reasonable wait, mark it `blocked` and
stop expanding that branch. If recent status shows useful progress, treat slow
handoff writing as `closing`, not as proof failure.

For review/fix workflows, preserve the orchestrator role: delegate review or fix
details to bounded sessions. The orchestrator may read worker status, handoffs,
and final findings, but should not independently redo the review/fix unless it
explicitly reassumes that task.

When rolling over the controller, the outgoing controller handoff should include:

- current objective and non-goals
- active thread ids, task statuses, and next proof checkpoint
- decisions made since the previous checkpoint
- files changed or proof already verified by the controller
- blockers, risks, and deferred questions
- exact next action for the fresh controller
- controller noise/efficiency notes, including token or rate-limit evidence if
  available

## Status Table Template

Use a compact table in `session-registry.md`:

```markdown
| Task | Role | Run | Title | Thread | Worktree | Branch | Commit | Status | Proof | Handoff | Next |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A1 | worker | r7f2 | Auth Audit: Worker - Matrix [A1] | <thread-id> | /path | branch | abc123 | verified | pass | handoffs/A1.md | integrate |
```

## Completion

A workflow is complete only when:

- the orchestrator Goal matches the whole workflow objective
- `workflow-state.md` or `milestone-plan.md` has no pending unblocked waves,
  milestones, phases, follow-up passes, or next actions
- all required task Goals are complete or explicitly deferred
- heartbeat automations are paused/deleted
- session registry is current
- latest controller checkpoint is written if the controller changed or ran long
- proof commands/readbacks are recorded
- write-capable sessions have either committed their verified slices or
  documented why they could not
- orchestrator Goal is updated to complete
- final handoff says what is done, what remains, and which sessions were closed
- noise/efficiency notes have been captured for future workflow improvement
- any controller-side direct execution has been explicitly called out as either
  an allowed exception or a process miss

Do not push, open PRs, merge, deploy, or touch production unless the current
repository workflow and the user's boundary allow it.

If any pending backlog remains, do not call the workflow complete. Continue the
next unblocked item according to ownership, or roll over with a handoff whose
first next action is the exact backlog item to launch, execute as controller
work, or verify.
