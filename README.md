# Codex Conductor

`codex-conductor` is a portable Codex skill for coordinating complex work across
multiple Codex Sessions.

It keeps long-running work recoverable by separating the controller role from
execution sessions, recording workflow state in files, requiring compact
handoffs, and verifying evidence before lifecycle claims.

## Use It For

- multi-session audits, migrations, incidents, or implementation waves
- tasks that need Goals, worktrees, durable handoffs, or heartbeat continuation
- workflows where a fresh controller should be able to resume from artifacts

It is intentionally project-agnostic: no Mainline, private repo, or host-local
tooling is required.

## Install

From a local checkout:

```bash
npx skills add . -g -a codex -y
```

Or install from GitHub:

```bash
npx skills add https://github.com/catoncat/codex-conductor -g -a codex -y
```

## Contents

- `SKILL.md`: the main controller/session workflow
- `references/harness-patterns.md`: orchestration pattern guide
- `templates/`: starter shapes for audits, incidents, implementation waves,
  review-fix waves, and design tournaments
- `agents/openai.yaml`: skill metadata for OpenAI/Codex installs
