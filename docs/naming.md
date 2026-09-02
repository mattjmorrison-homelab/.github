# Repo naming conventions

Living document — brainstorming in progress, not finalized.

## Current scheme

Everything is prefixed `homelab-`. Beyond that, the suffix (or lack of one)
indicates what kind of repo it is:

**Plain `homelab-<name>`** — a repo containing a Helm chart deploying a
single service (vendored image + custom config, no app code of its own).
This is the majority of repos: `homelab-grafana`, `homelab-prometheus`,
`homelab-pihole`, `homelab-alertmanager`, `homelab-zot`, etc.

**`homelab-<name>-crds`** — CRD-only install, no running workload. (CRD =
Custom Resource Definition — a Kubernetes API extension that teaches the
cluster a new resource type, e.g. `Certificate` or `ExternalSecret`. Just
installing the CRDs doesn't deploy anything that acts on them; that's the
job of the corresponding controller, installed separately.)
Example: `homelab-cert-manager-crds`, `homelab-external-secrets-crds`.

**`homelab-<name>-config`** — config/CR resources layered on top of a
vendored controller that lives elsewhere; no Deployment of its own.
Example: `homelab-cert-manager-config` (Certificate + ClusterIssuers +
TLSStore for cert-manager, which is installed separately as a vendored
chart).

**Vendored chart, values-only** — some `homelab-<name>` repos don't
contain a chart at all, just a `values.yaml` consumed by a remote Helm
chart via ArgoCD's multi-source Applications. Example: `homelab-cert-manager`,
`homelab-external-secrets`, `homelab-openbao` (charts come from
`charts.jetstack.io`, `charts.external-secrets.io`,
`openbao.github.io/openbao-helm` respectively).

## Proposed new scheme: destination prefix, drop `homelab-`

All repos live in the `mattjmorrison-homelab` GitHub org. The `homelab-`
prefix on the repo name itself is redundant — the org name already says
that. The prefix should instead name the repo's *destination/kind*:

- **`nix-`** — a NixOS host. The actual machine/OS configuration for a
  physical or dedicated machine, managed declaratively via Nix.
  Destination: the machine itself, outside Kubernetes entirely.
  (e.g. `nix-control-plane`)
- **`k8s-lib-`** — a Helm *library chart* (`type: library` in `Chart.yaml`),
    providing reusable template snippets that other charts pull in via
    their own `dependencies:` block and `{{ include }}` — never deployed
    or synced on its own, no ArgoCD Application of its own. Doesn't fit
    plain `k8s-` (that's for things that end up as a live resource
    themselves); same "included by, not deployed as" relationship
    `actions-` has to other repos' CI, just for Helm instead of GitHub
    Actions. `k8s-mod-` ("module") was the first name considered, but
    `lib`/"library chart" is Helm's own term for this exact mechanism, so
    it's the more legible choice. First (and so far only) example:
    `k8s-lib-ci-rbac`, converted from the old `k8s-ci-rbac` — see
    `rbac-plan.md`.
- **`k8s-`** — deploys to the k3s cluster via ArgoCD. Covers everything
  that ends up as a live Kubernetes resource regardless of *how* it gets
  there — a full Helm chart, a values-only override of a vendored chart, a
  CRD-only install, or a config-only repo layered on a controller
  installed elsewhere. The common thread is "the destination is the
  cluster," not any specific mechanism.
- **`pi-`** — a plain Raspberry Pi, outside both Nix and Kubernetes.
  Provisioning tooling or standalone scripts that just run directly on Pi
  OS.
- **`steamos-`** — a repo specific to bootstrapping steamos minipcs
  Provisioning tooling or standalone scripts that just run directly on
  steamos.
- **`graph-`** — source code + CI for a service that contributes a
  subgraph (schema/resolvers) to the federated graph. Not a deployment
  target itself — this is where the backend code lives before it gets
  built into an image and deployed (to a `k8s-` repo) by CI. Infrastructure
  that merely routes/federates (e.g. the Apollo Router) doesn't qualify —
  see the `graphql-router` note in the rename mapping below. (Was going to
  be a `-graph-api` suffix; the prefix alone now carries that meaning, so
  no suffix needed.)
- **`ui-`** — source code + CI for a frontend. Same relationship to a
  `k8s-` repo as `graph-` has — this is where the frontend source lives,
  not where it runs.
- **`ai-`** — AI tooling configuration, not tied to any deployment
  destination at all. Example: `ai-claude` (was `homelab-claude`) — Claude
  Code's own global agents/skills/hooks config.
- **`admin-`** — IaC administering another system's own policy/
  access-control layer, not deploying a workload into the cluster or
  onto a machine. Ongoing administration (these get re-applied
  repeatedly as things change), not just first-run setup. Examples:
  `admin-github` (was `gh-org`) — OpenTofu config for the
  `mattjmorrison-homelab` org via the `integrations/github` provider;
  `admin-openbao` — OpenTofu config for OpenBao's own
  policies/Kubernetes-auth-roles/KV paths via the `vault` provider.
- **`actions-`** — a reusable GitHub Actions workflow (`workflow_call`),
  called from other repos' own workflow files rather than run directly
  itself. Destination: other repos' CI, not a machine, the cluster, or a
  system's policy layer — doesn't fit `admin-` (that's for IaC managing
  another system's access control, not shared CI logic). Named after the
  actual tool it wraps, same reasoning as `admin-openbao` over
  `admin-vault`: e.g. `actions-tofu` for the shared OpenTofu
  plan/check/apply workflow, not `actions-terraform`.

One true exception outside this whole scheme: **`.github`** keeps its
literal name no matter what — GitHub requires that exact repo name for the
org-profile-README feature to work at all (`.github/profile/README.md`
renders on the org's Overview tab). It can't take a prefix.

Resolves the earlier "does app-source count as a k8s destination"
ambiguity: `graph-`/`ui-` are their own categories independent of where
the built artifact eventually deploys, so no compound suffix
(`-graph-api`/`-graph-k8s`) is needed — just `graph-<subject>` for the
source/CI repo and `k8s-<subject>` for its deployment manifests.

Example, hdmi-switch under the new scheme:

- `homelab-hdmi-switch` → `graph-hdmi-switch` (source + CI)
- `homelab-hdmi-switch-k8s` → `k8s-hdmi-switch` (manifests)
- `homelab-hdmi-switch-ui` → `ui-hdmi-switch` (frontend, currently empty)

Rename is a real operation (GitHub repo rename, `repoURL` in every
`homelab-apps` Application manifest, Woodpecker webhook/repo config for
app-source repos) — going one repo at a time, same as the verify-script
rollout.

## Full rename mapping

Renames happen opportunistically per the Execution policy above. As of
2026-08-31, everything below is **done** except the two rows explicitly
marked otherwise — confirmed via a direct GitHub API audit, not assumed.

| Current name | New name | Status | Notes |
| --- | --- | --- | --- |
| `homelab` | `nix-control-plane` | not done | Named after its primary purpose (this Nix-managed host is the k3s control-plane node, matches the `control.morrisons.site` DNS record), not just `nix`, in case more Nix-managed hosts get added later. Deliberately deferred each time this repo's been touched so far — not yet greenlit to execute. |
| `homelab-alertmanager` | `k8s-alertmanager` | done | |
| `homelab-apps` | `k8s-apps` | done | App-of-apps meta repo |
| `homelab-argocd` | `k8s-argocd` | done | |
| `homelab-argocd-image-updater` | `k8s-argocd-image-updater` | done | |
| `homelab-cert-manager` | `k8s-cert-manager` | done | vendored chart, values-only |
| `homelab-cert-manager-config` | `k8s-cert-manager-config` | done | |
| `homelab-cert-manager-crds` | `k8s-cert-manager-crds` | done | |
| `homelab-cloudflare` | `k8s-cloudflare` | done | |
| `homelab-coredns` | `k8s-coredns` | done | |
| `homelab-external-secrets` | `k8s-external-secrets` | done | vendored chart, values-only |
| `homelab-external-secrets-crds` | `k8s-external-secrets-crds` | done | |
| `homelab-grafana` | `k8s-grafana` | done | |
| `homelab-graphql-router` | `k8s-graphql-router` | done | deploys the vendored Apollo Router binary + config — it routes/federates but contributes no subgraph of its own, so `k8s-` not `graph-` |
| `homelab-hdmi-switch` | `graph-hdmi-switch` | done | app source + CI |
| `homelab-hdmi-switch-k8s` | `k8s-hdmi-switch` | done | manifests |
| `homelab-hdmi-switch-ui` | `ui-hdmi-switch` | done | the old `homelab-hdmi-switch-ui` local clone was an empty, zero-commit duplicate and was deleted, not renamed |
| `homelab-home-assistant` | `k8s-home-assistant` | done | |
| `homelab-homepage` | `k8s-homepage` | done | |
| `homelab-kube-state-metrics` | `k8s-kube-state-metrics` | done | |
| `homelab-metallb` | `k8s-metallb` | not applicable | not yet built, not currently tracked in `admin-github`'s `local.repos` |
| `homelab-minecraft` | `k8s-minecraft` | not applicable | not yet built, not currently tracked in `admin-github`'s `local.repos` |
| `homelab-node-exporter` | `k8s-node-exporter` | done | |
| `homelab-openbao` | `k8s-openbao` | done | vendored chart, values-only |
| `homelab-pihole` | `k8s-pihole` | done | |
| `homelab-prometheus` | `k8s-prometheus` | done | |
| `homelab-raspberrypi` | *(retiring)* | not applicable | Plan: split into `pi-provision` (flashing/cloud-init tooling) and a separate repo for whatever apps actually get deployed to the Pi(s), e.g. the planned heartbeat/uptime script. `homelab-raspberrypi` goes away once that split happens. |
| `homelab-speedtest-exporter` | `k8s-speedtest-exporter` | done | |
| `homelab-traefik` | `k8s-traefik` | done | HelmChartConfig only, no Deployment of its own |
| `homelab-woodpecker` | `k8s-woodpecker` | **contested, not done** | This table says renamed; `.github/docs/rbac-plan.md` (newer, more detailed) says **retired outright instead** — Application entry removed, repo archived, once its 3 remaining CI consumers move off Woodpecker. Whichever is correct, this row and `rbac-plan.md` currently disagree and need reconciling. Not renamed either way as of 2026-08-31. |
| `homelab-zot` | `k8s-zot` | done | |
| `homelab-claude` | `ai-claude` | done | Claude Code's own global config (agents/skills/hooks) — not tied to any deployment destination |
| `gh-org` | `admin-github` | done | |
| `k8s-ci-rbac` | `k8s-lib-ci-rbac` | done | Not part of the `homelab-*`→`k8s-*` scheme — converted to a Helm library chart (`k8s-lib-` prefix, see above), a new-semantics rename, not a retrofit. Full context in `rbac-plan.md`. |
| `.github` | *(no change)* | done | Required literal name for GitHub's org-profile README feature; not part of this scheme |

Not yet created, planned:

- `graph-health`, `k8s-health`, and (if it gets a frontend) `ui-health` —
  the health-check subgraph service discussed today

`k8s-garage` — Garage, self-hosted S3-compatible object storage. Built
for `admin-github`'s OpenTofu state bucket first (now live); a public
bucket for hosting `ui-hdmi-switch`'s static build (replacing its nginx
Deployment) is planned follow-up, not built yet.

`actions-tofu` — reusable GitHub Actions workflow (`check`/`plan`/`apply`)
shared by `admin-openbao` and `admin-github`, the two repos whose
near-identical OpenTofu CI prompted this prefix. Not built yet — the two
repos currently each carry their own near-duplicate workflow file.

## Migration notes

Things to watch out for when actually executing the renames, beyond just
the git repo rename itself:

- **ArgoCD `Application` object names are a separate identity from the git
  repo name**, and today they don't always match 1:1 (e.g. the Application
  is named `homelab-hdmi-switch`, but its `repoURL` points at
  `homelab-hdmi-switch-k8s`). Renaming the *repo* is just a URL change in
  `homelab-apps` — low risk. Renaming the *Application object itself* is
  not: its `metadata.name` is its actual identity in the cluster, so
  changing it means creating a new `Application` and deleting the old one,
  not an in-place rename. Done carelessly, that risks ArgoCD briefly
  treating the old app's live resources as orphaned and pruning them.
  Decide up front whether Application names also adopt the new `k8s-`
  prefix scheme, since that determines how carefully each one needs to be
  sequenced (repo-only rename vs. full Application recreation).
- **Woodpecker's repo list is tied to GitHub via webhook/app integration.**
  GitHub preserves the repo ID through a rename (old URLs redirect), but
  Woodpecker may still need a manual "resync repo list" in its UI to pick
  up the new names for `graph-hdmi-switch` / `ui-hdmi-switch` (and any
  future `graph-`/`ui-` repos).

## Execution policy

Renames happen opportunistically, not as a single batched migration: whenever
any repo is touched for other work and its current name appears in the rename
mapping above, it gets renamed as part of that touch (GitHub rename, local
clone, and any other repo's reference to it — an `homelab-apps` Application
entry, a hardcoded `repoURL`, an `admin-openbao` role/secret path) before the
actual task proceeds. No separate org-wide rename effort is planned; the
mapping above is just the reference other renames are checked against.
