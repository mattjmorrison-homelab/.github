# The homelab team

"The homelab team" is the set of Claude Code agents (defined in `~/.claude/agents/`) that make every code change across this homelab's repos. No repo gets edited by hand or by an ad-hoc prompt — every change goes through this pipeline, regardless of language or framework.

## Pipeline

- **`homelab-planner`** — researches and writes a plan document for anything spanning more than a trivial edit (new services, cross-repo migrations, new infrastructure). Never implements.
- **`plan-implementer`** — executes a `homelab-planner` plan: derives dependency-gated phases from its rollout order, dispatches `orchestrator` per repo per phase (parallel within a phase, sequential across phases), and produces the final merge runbook once all phases have dispatched.
- **`orchestrator`** — drives one repo through a strict Red→Green→Refactor loop, one task at a time. Discovers each repo's own check command (its `Makefile` `check`/`test` target, or its CI config) rather than assuming a stack. Delegates all actual work; never writes code or tests itself.
- **`tester`** / **`implementer`** — write the single smallest failing test, then the minimum code to pass it. Match whatever test/code conventions already exist in the target repo.
- **`verifier`** — runs the discovered check command and judges minimalism at each phase. Never edits files.
- **`refactorer`** — one behavior-preserving cleanup pass after green, verified against the same check command.
- **`doc-writer`** — updates documentation to match what changed, following the repo's own existing documentation convention.
- **`architect`** — adversarial review of the complete diff before it's committed: checks it against this homelab's own documented standards (`CLAUDE.md`, `naming.md`, `style.md`, `TODO.md`) and scans other repos for downstream impact (a secret that should exist in OpenBao, duplication that belongs in a shared lib, a manual step that should be automated). Sends findings back into the pipeline until clean; never edits files itself.
- **`pr-committer`** — the only agent permitted to commit, push, or open a PR. Never merges — a human always reviews and merges.

## What it can and can't do

Every step above stops at "PR open." Nothing in this pipeline merges a PR, and nothing runs a cluster-mutating command (`kubectl apply`/`delete`/`patch`, `helm upgrade --install`, etc.) — check commands are restricted to a repo's literal `check`/`test` target, never inferred from another target name. Deployment only happens when a human merges a PR and ArgoCD's own sync picks it up.

## Trust, but verify the pipeline itself

Real defects found in this pipeline's own machinery, not just in the tasks it was given (2026-08-30/31): a `for_each`-by-name rename trap in `admin-openbao`'s Terraform (a value change in a for_each key looks like destroy+create, not rename); the identical trap independently rediscovered in `admin-github`; an agent trying to relay its result via `SendMessage` instead of returning it, silently stalling the parent orchestrator for multiple rounds; a `create_before_destroy` fix that would have broken `tofu apply` outright, caught only by manually reading a live plan's output rather than trusting a green check. None of these were the underlying task being hard — they were the pipeline's own design being subtly wrong.

Default posture when something looks sketchy: suspect the pipeline before assuming the task was actually done correctly, especially for anything touching live infrastructure state (Terraform, DNS, RBAC). A green check or an "APPROVED" from `architect` is evidence, not proof — reading the actual diff or plan output yourself remains worth doing for anything consequential.
