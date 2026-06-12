# Codex Conductor

`codex-conductor` is a Codex App skill for coordinating complex work across
multiple Codex Sessions.

Use it when one chat is no longer enough: broad audits, implementation waves,
incident follow-up, migrations, multi-PR cleanup, or any task where another
Codex session should be able to resume from durable state instead of reading a
long conversation.

The conductor separates:

- the controller, which plans, tracks state, launches workers, verifies proof,
  and reconciles handoffs
- execution sessions, which do bounded implementation, review, evidence
  gathering, or verification work
- project configuration, which lives in your repository instead of inside the
  public skill

It assumes Codex Goal support. It does not require Mainline, GitHub, a local
multi-agent skill, or any private tooling. If optional lifecycle and subagent
capabilities are available, the conductor can use them. If not, it falls back
to workflow files, task prompts, compact handoffs, and explicit proof.

The controller uses a Codex Goal for the whole workflow. If it splits work into
waves, milestones, phases, or follow-up passes, it keeps a small backlog in
`workflow-state.md` or `milestone-plan.md` and continues through the next
unblocked item instead of treating the first wave as the whole job.

Before launching sessions, the controller clarifies the assignment into an
orchestration brief: goal, deliverables, scope, acceptance criteria,
constraints, known artifacts, open questions, and the recommended first
harness. It asks focused questions only when the answer cannot be discovered
from the repo or current workflow files and would materially change the plan.

## Install

Install from GitHub:

```bash
npx skills add https://github.com/catoncat/codex-conductor -g -a codex -y
```

Install from a local checkout:

```bash
git clone git@github.com:catoncat/codex-conductor.git
cd codex-conductor
npx skills add . -g -a codex -y
```

After installation, ask Codex to use the skill:

```text
Use $codex-conductor to coordinate this migration across multiple Codex Sessions.
```

## When To Use It

Good fits:

- a feature or refactor that needs several isolated worktrees
- an audit where findings need evidence and counterevidence
- an incident where runtime facts, fixes, and readback must stay separate
- a review/fix wave where review findings should not be mixed with fixes
- a long-running task that needs durable checkpoints and compact handoffs

Poor fits:

- a small one-file fix
- a quick answer or explanation
- ordinary parallel file reads that fit in one session; use subagents directly
  for that when available instead of creating a durable conductor workflow
- work where the user wants a single agent to directly edit and finish

## Subagent Capability And Host Tuning

`codex-conductor` is a skill. Subagents are a Codex capability exposed by the
current runtime when available. The conductor must not assume a user-local skill
with a specific name exists.

Subagents are useful for context isolation: a short-lived explorer can read a
large worker transcript, test log, or status trail and return a compact result
so the controller does not carry raw intermediate output.

Recommended optional host tuning:

```toml
[features]
multi_agent = true
child_agents_md = true # optional / under development if supported
enable_fanout = true   # optional / under development if supported

[agents]
max_threads = 6
max_depth = 1
```

Treat these as tuning, not install requirements. If subagent tools are missing,
check `codex features list` and the tool surface available in the current
session. Do not enable `multi_agent_v2` unless the current Codex install
explicitly supports it and the user asks for it. Raise `agents.max_threads` only
when more parallel workers are intentionally useful; keep `max_depth = 1` by
default to avoid recursive fanout.

## Add Project Configuration

For best results, add a project profile to repositories where you expect to use
the conductor repeatedly:

```text
.codex-conductor/
  project.md
```

Start from the bundled template:

```bash
mkdir -p .codex-conductor
cp ~/.agents/skills/codex-conductor/templates/project-profile.md .codex-conductor/project.md
```

Then fill in the parts that matter for your project:

```markdown
# Codex Conductor Project Profile

## Package Manager
- Use: pnpm
- Do not use: npm

## Verification Tiers
### docs-only
- Commands: git diff --check
- Skip: dependency install, database setup, full test suite

### integration-runtime
- Bootstrap: install dependencies if missing; ensure test env exists
- Commands: focused integration test, targeted verify command
- Skip: full suite until controller integration

## Worktree Environment
- New worktrees require dependency install before checks.
- Schema or migration work must use isolated databases.
- If workers repeatedly fail on environment, generated files, or verification
  setup, add a project-specific recovery rule here instead of making every
  worker rediscover it.

## Lifecycle
- Identity and naming:
  - Session title: <Workflow Label>: <Role> - <Task Label> [<task-id>]
  - Branch naming: include workflow slug, short run id, and task id
  - Label policy: do not create labels unless the controller asks
- Commit policy: commit verified slices only.
- PR policy: workers open PRs only when explicitly assigned.

## Forbidden Commands Or Actions
- Do not run production mutations without explicit authorization.
```

Split the profile only if it grows:

```text
.codex-conductor/
  project.md       # short overview and defaults
  verification.md  # expensive checks, tiers, skip rules
  worktree.md      # env, dependencies, databases, generated files
  lifecycle.md     # branch, commit, issue, PR, release policy
```

The conductor also reads normal repo instructions such as `AGENTS.md`,
`CONTRIBUTING.md`, and workflow docs. The `.codex-conductor/` profile exists so
conductor-specific rules are easy to find and do not get buried in product
documentation.

## Identity And Recovery

When the chosen workflow already needs durable sessions, worktrees, branches,
issues, PRs, or handoffs, the conductor gives those objects names that map back
to the same `workflow-state.md` and `session-registry.md`. Identity is only a
recovery index; it is not a reason to create extra objects.

The controller records a short identity section in `workflow-state.md`:

```markdown
## Identity
workflow_slug: auth-rbac-audit
workflow_label: Auth RBAC Audit
run_id: r7f2
project_label: billing-api
naming_overrides: .codex-conductor/project.md
```

Visible session titles should stay readable, while exact thread ids, worktree
paths, branches, proof, and handoff paths stay canonical in
`session-registry.md`.

When Codex App thread tools are available, the conductor lets the launcher keep
the account-compatible default model unless the user or a verified host profile
explicitly says otherwise. Thread creation must be followed by live readback:
pending worktree ids and failed launches are not treated as active workers until
an actual thread id and status are known.

## Optional Host Configuration

Machine-specific capabilities belong in a private host profile, not in the
public skill or project repository:

```text
~/.config/codex-conductor/host.md
```

Use it for local tools and policies such as:

- thread launcher availability
- Goal naming and lifecycle conventions
- whether GitHub CLI is configured
- how to use local lifecycle tools
- private safety boundaries

Set `CODEX_CONDUCTOR_HOME` if you want a different host profile directory.

## What The Conductor Creates

For a durable workflow, the controller usually creates:

```text
docs/workflows/<yyyy-mm-dd>-<short-slug>/
  workflow-state.md
  session-registry.md
  tasks/
  prompts/
  handoffs/
```

The exact shape depends on the task. The conductor should create the smallest
control plane that keeps the work recoverable.

These files are recovery state first, not automatically product history. In a
long-running workflow, keep noisy controller updates on the controller/private
side. When product work is ready to publish, assemble a clean delivery branch
from the intended base by cherry-picking verified product commits. If the
workflow needs a public record, prefer one compact milestone archive commit over
many heartbeat or registry commits.

The most important file is `workflow-state.md`. It records:

- refined orchestration brief
- objective and non-goals
- workflow shape and complexity budget
- program backlog and next unblocked item
- Project Constraints Capsule
- active sessions and proof gates
- shrink/stop conditions
- controller checkpoints

## Project Constraints Capsule

Before launching workers, the controller summarizes project-specific rules into
a compact capsule. Worker prompts receive only the relevant subset.

Example:

```markdown
## Project Constraints Capsule
source_files:
- .codex-conductor/project.md
- AGENTS.md
capability_mode: portable
package_manager: pnpm
verification_tiers:
- tier: docs-only
  commands: git diff --check
  skip: runtime bootstrap, database setup
- tier: integration-runtime
  bootstrap: install dependencies if missing; ensure test env exists
  commands: focused integration test
worktree_bootstrap:
- new worktrees must confirm dependency and env state before tests
control_plane_publication:
- default: keep high-frequency controller state out of product main
- delivery: cherry-pick verified product commits from the intended base
forbidden_commands:
- do not mutate production without explicit authorization
```

This is the mechanism that prevents every worker from rediscovering the same
environment rules.

The controller should give a short doctor-style reminder when it discovers a
missing host capability or project rule that would prevent repeated failures. If
the user has granted write access and the fix is a small repo-specific profile
entry, the controller may add it directly. Machine-specific settings belong in
the host profile or user configuration; add them only when current authorization
covers personal host config, otherwise show the recommended setting and continue
with the fallback path.

## Common Prompt Shapes

Start a new workflow:

```text
Use $codex-conductor.

Coordinate the remaining checkout migration. Create a durable workflow, split
the work into independent sessions only where useful, and make sure every worker
gets the project constraints capsule before it starts. Use the controller Goal
to keep going through all planned waves until the backlog is complete, blocked,
or explicitly deferred. For each next item, execute controller-owned control
plane or verification work in the controller, and delegate write-capable work to
execution sessions. Start by restating the real goal, deliverables, scope,
acceptance criteria, and any questions that materially affect the plan.
```

Resume an existing workflow:

```text
Use $codex-conductor.

Resume docs/workflows/2026-06-08-checkout-migration. Read workflow-state.md and
session-registry.md first, reconcile handoffs, then continue the next smallest
safe step.
```

Run a read-only audit:

```text
Use $codex-conductor for a read-only audit of the auth boundary. Keep review and
fixes separate. Findings need file/line evidence and counterevidence checks.
```

Coordinate implementation workers:

```text
Use $codex-conductor to split this into implementation slices. Workers may write
only their assigned paths, must use isolated worktrees, and must produce focused
proof before handoff.
```

Diagnose a stalled worker:

```text
Use $codex-conductor.

Classify the worker as progressing, closing, env-stuck, stalled, invalid, or
contradictory. Use compact handoffs and proof first. If subagent capability is
available, use a read-only explorer to summarize transcript/status evidence
before the controller reads raw transcript. If the worker is invalid or must be
replaced, send STAND DOWN first and let the replacement inherit only compact
facts, proof pointers, and trusted commits/handoffs.
```

## Contents

- `SKILL.md`: the main controller/session workflow.
- `references/configuration.md`: project and host configuration guide.
- `references/harness-patterns.md`: orchestration patterns and failure modes.
- `templates/project-profile.md`: starter `.codex-conductor/project.md`.
- `templates/`: starter shapes for audits, incidents, implementation waves,
  review-fix waves, and design tournaments.
- `agents/openai.yaml`: skill metadata for OpenAI/Codex installs.

## Design Principles

- One controller owns planning and reconciliation.
- Workers own bounded execution.
- Durable files are the source of truth, not the chat transcript.
- Proof comes from commands, files, runtime readback, or explicit handoffs.
- Project-specific behavior belongs in `.codex-conductor/`, not in the public
  skill.
- Use the smallest orchestration shape that preserves correctness.
