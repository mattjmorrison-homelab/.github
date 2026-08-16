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
- **`k8s-`** — deploys to the k3s cluster via ArgoCD. Covers everything
  that ends up as a live Kubernetes resource regardless of *how* it gets
  there — a full Helm chart, a values-only override of a vendored chart, a
  CRD-only install, or a config-only repo layered on a controller
  installed elsewhere. The common thread is "the destination is the
  cluster," not any specific mechanism.
- **`pi-`** — a plain Raspberry Pi, outside both Nix and Kubernetes.
  Provisioning tooling or standalone scripts that just run directly on Pi
  OS.
- **`graph-`** — source code + CI for a GraphQL subgraph backend. Not a
  deployment target itself — this is where the backend code lives before
  it gets built into an image and deployed (to a `k8s-` repo) by CI. (Was
  going to be a `-graph-api` suffix; the prefix alone now carries that
  meaning, so no suffix needed.)
- **`ui-`** — source code + CI for a frontend. Same relationship to a
  `k8s-` repo as `graph-` has — this is where the frontend source lives,
  not where it runs.
- **`ai-`** — AI tooling configuration, not tied to any deployment
  destination at all. Example: `ai-claude` (was `homelab-claude`) — Claude
  Code's own global agents/skills/hooks config.

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

## Full rename mapping (draft)

| Current name | New name | Notes |
|---|---|---|
| `homelab` | `nix-control-plane` | Named after its primary purpose (this Nix-managed host is the k3s control-plane node, matches the `control.morrisons.site` DNS record), not just `nix`, in case more Nix-managed hosts get added later |
| `homelab-alertmanager` | `k8s-alertmanager` | |
| `homelab-apps` | `k8s-apps` | App-of-apps meta repo |
| `homelab-argocd` | `k8s-argocd` | |
| `homelab-argocd-image-updater` | `k8s-argocd-image-updater` | |
| `homelab-cert-manager` | `k8s-cert-manager` | vendored chart, values-only |
| `homelab-cert-manager-config` | `k8s-cert-manager-config` | |
| `homelab-cert-manager-crds` | `k8s-cert-manager-crds` | |
| `homelab-cloudflare` | `k8s-cloudflare` | |
| `homelab-coredns` | `k8s-coredns` | |
| `homelab-external-secrets` | `k8s-external-secrets` | vendored chart, values-only |
| `homelab-external-secrets-crds` | `k8s-external-secrets-crds` | |
| `homelab-grafana` | `k8s-grafana` | |
| `homelab-graphql-router` | `k8s-graphql-router` | deploys the vendored Apollo Router binary + config — not custom source, so `k8s-` not `graph-` |
| `homelab-hdmi-switch` | `graph-hdmi-switch` | app source + CI |
| `homelab-hdmi-switch-k8s` | `k8s-hdmi-switch` | manifests |
| `homelab-hdmi-switch-ui` | `ui-hdmi-switch` | currently empty |
| `homelab-home-assistant` | `k8s-home-assistant` | |
| `homelab-homepage` | `k8s-homepage` | |
| `homelab-kube-state-metrics` | `k8s-kube-state-metrics` | |
| `homelab-metallb` | `k8s-metallb` | |
| `homelab-minecraft` | `k8s-minecraft` | |
| `homelab-node-exporter` | `k8s-node-exporter` | |
| `homelab-openbao` | `k8s-openbao` | vendored chart, values-only |
| `homelab-pihole` | `k8s-pihole` | |
| `homelab-prometheus` | `k8s-prometheus` | |
| `homelab-raspberrypi` | *(retiring)* | Plan: split into `pi-provisioning` (flashing/cloud-init tooling) and a separate repo for whatever apps actually get deployed to the Pi(s), e.g. the planned heartbeat/uptime script. `homelab-raspberrypi` goes away once that split happens. |
| `homelab-speedtest-exporter` | `k8s-speedtest-exporter` | |
| `homelab-traefik` | `k8s-traefik` | HelmChartConfig only, no Deployment of its own |
| `homelab-woodpecker` | `k8s-woodpecker` | |
| `homelab-zot` | `k8s-zot` | |
| `homelab-claude` | `ai-claude` | Claude Code's own global config (agents/skills/hooks) — not tied to any deployment destination |
| `.github` | *(no change)* | Required literal name for GitHub's org-profile README feature; not part of this scheme |

Not yet created, planned:
- `graph-health`, `k8s-health`, and (if it gets a frontend) `ui-health` —
  the health-check subgraph service discussed today

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

## Open questions

- When (if ever) to actually execute the renames — one at a time, like the
  verify-script rollout, or batched?
- Does the `graph-`/`ui-`/`k8s-` split apply retroactively to
  `homelab-graphql-router` too, or does it stay `k8s-` since it has no
  custom source of its own (just vendored binary + config)? Current
  thinking: stays `k8s-`.
