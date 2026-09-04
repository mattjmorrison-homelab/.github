# Required status checks: proposal

Goal: every repo's PRs require a passing CI check before merge, with what
"passing" means varying by repo category.

## Confirmed mechanism, and the naming scheme

A flat `.woodpecker.yml` posts one GitHub status context per repo
covering the whole pipeline — confirmed directly against `graph-router`'s
commit statuses: `ci/woodpecker/push/woodpecker`. Woodpecker also supports
multiple pipeline files under a `.woodpecker/` directory, each posting its
**own** context named after the file.

Standardize on **`.woodpecker/check.yaml`** as the PR-gating entrypoint,
same filename in every repo regardless of category — not `build`: nothing
in this list actually produces a deployable artifact. `k8s-` repos have no
build step at all (ArgoCD reads the chart straight from git; `helm
template` there just proves it renders, it doesn't produce an output).
`graph-`/`ui-` do have a real build (`build-release`, via kaniko), but
that's already correctly separate — gated `when: branch: main`, so it
never runs on a PR and was never a candidate for this anyway. Every
category's PR check is pure verification, so the name should say that:
`check` matches GitHub's own term ("required status **checks**") and the
verb `nix flake check` already uses.

This makes the context string identical everywhere —
`ci/woodpecker/pull_request/check` — so `admin-github`'s
`github_branch_protection.main[each].required_status_checks.contexts` is
just `["ci/woodpecker/pull_request/check"]` uniformly, no per-category
special-casing in Terraform. What `check` actually runs stays
category-specific (below); the name and the Terraform reference never
vary.

`graph-`/`ui-`/`admin-github` currently use a flat `.woodpecker.yml` (context
`woodpecker`, not `check`) — for full uniformity these three need their
existing PR-relevant steps (`build-test`+`test` for `graph-`/`ui-`,
`tofu-plan` for `admin-github`) moved into `.woodpecker/check.yaml`, leaving
the deploy-only steps (`build-release`, `verify-deploy`, `tofu-apply`)
in a separate file that keeps running on push to `main` only. Small
relocation, not a rewrite.

## What "passing" means per category

- **`k8s-`** — `helm lint`, `helm template` (proves it renders to valid
  YAML), and `kubectl apply --dry-run=server` against the real cluster
  (catches things client-side dry-run can't, like a field a CRD's
  admission webhook actually rejects). Needs a **new**
  `.woodpecker/check.yaml` in every `k8s-` repo — none of them have any
  CI today, since verification currently happens post-deploy via each
  repo's own PostSync `verify` Job, not pre-merge.
- **`graph-`/`ui-`** — mostly covered already. `build-test`+`test` move
  into `.woodpecker/check.yaml`; `build-release`/`verify-deploy` stay
  behind in a separate deploy-only file.
- **`admin-github`** — mostly covered already. `tofu-plan` moves into
  `.woodpecker/check.yaml`; `tofu-apply` stays in a deploy-only file.
- **`nix-`** (`nix-control-plane`) — `nix flake check`. New pipeline.
- **`pi-`/`steamos-`** (provisioning scripts) — `shellcheck` on every
  `.sh` file. New pipeline, lightweight.
- **`ai-`, `.github`** — no obvious automated check (agent config, docs).
  Recommend leaving these without a required status check rather than
  inventing one that doesn't actually verify anything.

## The real cost: cluster RBAC for k8s- repos

`--dry-run=server` (as opposed to `--dry-run=client`) submits through the
real API server's admission chain, which means the credential running it
needs the same create/update RBAC it would need to actually apply — dry
running skips only the final etcd write, not authorization. This is a
meaningfully larger grant than anything else CI does in this homelab: a
credential with write access to real cluster resources, usable from every
PR.

Recommendation: **one ServiceAccount + Role per repo's target namespace**,
`create`/`update` only (no `delete`), rather than one broad ClusterRole
shared across all `k8s-` repos — same OpenBao-secret pattern as
everything else here (`kv/homelab/<repo>` → a scoped kubeconfig or token),
so a compromised credential for one repo's CI can't touch another
namespace. ~20 repos means ~20 of these to set up — real, repetitive
work, not a one-time cost.

## `admin-github` changes needed

`branch_protection.tf` currently applies identical protection to every
repo via one `for_each` over `local.repos`. This needs a repos-with-CI
subset (or a per-repo boolean/map) so `required_status_checks` only gets
added where a `.woodpecker.yml` actually exists to satisfy it — requiring
a check that can never post would make every PR permanently unmergeable.

## Sequencing

Given ~20 `k8s-` repos each need a new pipeline **and** new per-namespace
RBAC/OpenBao wiring, this should go one repo at a time, not a batch —
same reasoning as the existing sequential-rollout precedent for repo work
in this homelab (a past all-at-once pass caused a real outage).
Recommendation: pilot the whole mechanism (pipeline, RBAC, OpenBao secret,
required-check wiring) on one `k8s-` repo first — `k8s-garage`, since it's
the most recently built and easiest to verify against — before rolling to
the rest.
