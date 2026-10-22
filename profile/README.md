# mattjmorrison-homelab

A personal homelab running everything as code: a [k3s](https://k3s.io)
Kubernetes cluster, deployed and kept in sync entirely through
[ArgoCD](https://argo-cd.readthedocs.io) GitOps, with the base machines
managed declaratively via [NixOS](https://nixos.org).

## How it's organized

One repo per service, no shared "everything" manifest repo. A central
`k8s-apps` repo (ArgoCD's "app of apps") holds one `Application`
manifest per service — add a repo, register it there, ArgoCD picks it up
automatically.

## Stack

- **Cluster**: k3s, ArgoCD, Helm
- **Ingress/networking**: Traefik, Cloudflare Tunnel, Pi-hole + CoreDNS
- **Secrets**: OpenBao + External Secrets Operator
- **CI/CD**: GitHub Actions (self-hosted runners via `k8s-github-runner`), zot
- **Observability**: Prometheus, Grafana, Alertmanager
- **Security**: cert-manager (wildcard TLS via Let's Encrypt)

Every deployed service also runs a post-deploy verification check
(`PostSync` ArgoCD hook) that hits the live app after each sync and fails
the deploy fast if it doesn't come back healthy.

## Repositories

Organized by what each repo actually is, per the `admin-`/`ai-`/`app-`/
`graph-`/`k8s-`/`k8s-lib-`/`nix-`/`pi-`/`steamos-`/`ui-` naming scheme
documented in this repo's own `docs/naming.md`.

### Cluster

- **[homelab](https://github.com/mattjmorrison-homelab/homelab)** — NixOS configuration and modules (e.g. `k3s-control-plane`) for the homelab's host machines, tested via NixOS VM integration tests. Still named `homelab`, not yet renamed to `nix-control-plane` under the current scheme.
- **[k8s-apps](https://github.com/mattjmorrison-homelab/k8s-apps)** — ArgoCD "app of apps" repo holding one `Application` manifest per deployed service so ArgoCD can sync everything automatically.
- **[k8s-argocd](https://github.com/mattjmorrison-homelab/k8s-argocd)** — Bootstraps and upgrades ArgoCD itself into the k3s cluster as the tool that deploys everything else.
- **[k8s-argocd-image-updater](https://github.com/mattjmorrison-homelab/k8s-argocd-image-updater)** — Argo CD Image Updater install that watches the HDMI-switch Application and bumps its image tag via parameter overrides, with no git commits.

### Ingress/networking

- **[k8s-traefik](https://github.com/mattjmorrison-homelab/k8s-traefik)** — `HelmChartConfig` overrides for k3s's built-in Traefik that redirect all plain HTTP traffic to HTTPS.
- **[k8s-cloudflare](https://github.com/mattjmorrison-homelab/k8s-cloudflare)** — Runs `cloudflared` (Cloudflare Tunnel) for public ingress to the cluster; DNS records for the tunnel are now managed declaratively by `admin-cloudflare` instead of this repo's own bootstrap script.
- **[k8s-pihole](https://github.com/mattjmorrison-homelab/k8s-pihole)** — Pi-hole deployment providing DNS/ad-blocking and local DNS records for `*.morrisons.site` hostnames and a handful of `.local` LAN devices.
- **[k8s-coredns](https://github.com/mattjmorrison-homelab/k8s-coredns)** — Custom CoreDNS config that forwards the `morrisons.site` zone to Pi-hole so pods resolve it the same way LAN clients do.

### Secrets

- **[k8s-openbao](https://github.com/mattjmorrison-homelab/k8s-openbao)** — OpenBao secrets manager deployment; the source of truth for every secret consumed elsewhere in the cluster via External Secrets Operator.
- **[k8s-external-secrets](https://github.com/mattjmorrison-homelab/k8s-external-secrets)** — Deploys External Secrets Operator, which bridges OpenBao secrets into Kubernetes Secrets for other apps.
- **[k8s-external-secrets-crds](https://github.com/mattjmorrison-homelab/k8s-external-secrets-crds)** — External Secrets Operator's CRDs, split into their own Application because they're too large for a standard Helm install.

### CI/CD

- **[k8s-github-runner](https://github.com/mattjmorrison-homelab/k8s-github-runner)** — Self-hosted GitHub Actions runners for the org — plain per-architecture Deployments that register natively, not Actions Runner Controller (ARC's runner registration didn't work reliably in this cluster).
- **[k8s-zot](https://github.com/mattjmorrison-homelab/k8s-zot)** — Self-hosted OCI registry (`registry.morrisons.site`) for images and Helm charts built by CI, with a per-consumer-scoped credential for every publishing repo.
- **[k8s-garage](https://github.com/mattjmorrison-homelab/k8s-garage)** — Self-hosted S3-compatible object storage (Garage), built first as the state backend for `admin-github`'s OpenTofu state; planned (not yet built) to also host `ui-hdmi-switch`'s static build from a public bucket.

### Shared CI (`actions-*`)

Reusable GitHub Actions composite actions, called from other repos' own
workflow files rather than run directly themselves.

- **[actions-bash](https://github.com/mattjmorrison-homelab/actions-bash)** — Installs pinned `actionlint`, `shellcheck`, and `bats` for any repo's bash-script CI checks.
- **[actions-helm](https://github.com/mattjmorrison-homelab/actions-helm)** — `helm lint`/`helm template` plus a server-side `kubectl apply --dry-run=server`, used by every `k8s-` repo's CI.
- **[actions-k8s](https://github.com/mattjmorrison-homelab/actions-k8s)** — Mints a scoped token and runs a real in-cluster Kubernetes `Job` from CI (image builds via kaniko, in-cluster tests), since the runners themselves have no Docker socket.
- **[actions-openbao](https://github.com/mattjmorrison-homelab/actions-openbao)** — Logs into OpenBao via the runner pod's own Kubernetes identity and reads or writes one KV secret for the rest of the job.
- **[actions-tofu](https://github.com/mattjmorrison-homelab/actions-tofu)** — Shared `tofu plan`/`apply` composite actions for `admin-github` and `admin-openbao`'s near-identical OpenTofu CI.

### Observability

- **[k8s-prometheus](https://github.com/mattjmorrison-homelab/k8s-prometheus)** — Scrapes and stores cluster metrics, firing alerts to Alertmanager based on configured rules.
- **[k8s-grafana](https://github.com/mattjmorrison-homelab/k8s-grafana)** — Grafana deployment providing dashboards for cluster and host metrics sourced from Prometheus.
- **[k8s-alertmanager](https://github.com/mattjmorrison-homelab/k8s-alertmanager)** — Routes alerts fired by Prometheus to Discord webhooks (general alerts, and node-down alerts separately).
- **[k8s-kube-state-metrics](https://github.com/mattjmorrison-homelab/k8s-kube-state-metrics)** — Exports Kubernetes object state (pods, deployments, nodes) as Prometheus metrics.
- **[k8s-node-exporter](https://github.com/mattjmorrison-homelab/k8s-node-exporter)** — Exports host-level metrics (CPU, memory, disk, network) from each node to Prometheus.
- **[k8s-speedtest-exporter](https://github.com/mattjmorrison-homelab/k8s-speedtest-exporter)** — Runs periodic Ookla speed tests and exposes the results as Prometheus metrics, visualized in Grafana.

### Security

- **[k8s-cert-manager](https://github.com/mattjmorrison-homelab/k8s-cert-manager)** — Helm chart install of cert-manager, kept separate from its CRDs and its issuer/certificate configuration.
- **[k8s-cert-manager-config](https://github.com/mattjmorrison-homelab/k8s-cert-manager-config)** — cert-manager configuration: Let's Encrypt `ClusterIssuer`s via Cloudflare DNS-01 and the `*.morrisons.site` wildcard certificate.
- **[k8s-cert-manager-crds](https://github.com/mattjmorrison-homelab/k8s-cert-manager-crds)** — cert-manager's CustomResourceDefinitions, applied with server-side apply since they're too large for a normal client-side apply.
- **[k8s-ci-rbac](https://github.com/mattjmorrison-homelab/k8s-ci-rbac)** — Deploys the live ServiceAccount/Role/RoleBinding each consumer's CI dry-run check actually authenticates as, independently of any consumer's own chart, so a consumer's first PR isn't blocked on RBAC that PR's own unmerged chart would otherwise have to create.
- **[k8s-lib-ci-rbac](https://github.com/mattjmorrison-homelab/k8s-lib-ci-rbac)** — A Helm *library* chart (`type: library`) that generates that same per-namespace CI RBAC as a template other charts pull in via their own `dependencies:` block, rather than deploying it as a standalone Application. See note below — this is not simply the renamed form of `k8s-ci-rbac`; both repos exist today with distinct purposes.
- **[k8s-host-rbac](https://github.com/mattjmorrison-homelab/k8s-host-rbac)** — Planned, not yet built: a ServiceAccount + ClusterRoleBinding and a long-lived token Secret giving the NixOS control-plane host its own read-only kubeconfig.

> **Note:** `docs/naming.md`'s rename-mapping table currently records `k8s-ci-rbac` → `k8s-lib-ci-rbac` as a completed rename. In reality both repos exist independently today, with different purposes (listed separately above) — a known documentation inconsistency, not resolved as part of this update.

### Apps & services

- **[k8s-home-assistant](https://github.com/mattjmorrison-homelab/k8s-home-assistant)** — Home Assistant deployment (`hostNetwork` for mDNS device discovery) for home automation, at `home-assistant.morrisons.site`.
- **[k8s-matter-server](https://github.com/mattjmorrison-homelab/k8s-matter-server)** — [matter.js](https://github.com/matter-js/matterjs-server) Matter server giving Home Assistant's Matter integration something to talk to over WebSocket; pod-affinity-pinned to run on the same node as Home Assistant's own `hostNetwork` pod.
- **[k8s-homepage](https://github.com/mattjmorrison-homelab/k8s-homepage)** — Homepage dashboard deployment linking out to the cluster's other services, at `homelab.morrisons.site`.
- **[graph-router](https://github.com/mattjmorrison-homelab/graph-router)** — Builds the Apollo Router image for the federated GraphQL gateway, composing the supergraph schema at build time from each live subgraph.
- **[k8s-graphql-router](https://github.com/mattjmorrison-homelab/k8s-graphql-router)** — Deploys the Apollo Router image built by `graph-router`.
- **[graph-hdmi-switch](https://github.com/mattjmorrison-homelab/graph-hdmi-switch)** — Python/Flask + GraphQL app source, tests, and CI pipeline for controlling a TESMart HDMI switch over serial.
- **[k8s-hdmi-switch](https://github.com/mattjmorrison-homelab/k8s-hdmi-switch)** — Helm chart and manifests deploying `graph-hdmi-switch`'s built image; image bumps land here via Argo CD Image Updater, with no git commits.
- **[ui-hdmi-switch](https://github.com/mattjmorrison-homelab/ui-hdmi-switch)** — React/TypeScript PWA remote control for the HDMI switch (install-to-home-screen), talking to its GraphQL API with deliberately no offline caching of switch state.
- **[app-backstage](https://github.com/mattjmorrison-homelab/app-backstage)** — Backstage app source (Yarn workspace, scaffolded `packages/app` + `packages/backend`) — the dependency/relationship catalog being built for this homelab.
- **[docker-backstage](https://github.com/mattjmorrison-homelab/docker-backstage)** — Shared dependency base image (`prod`/`test` targets) that `app-backstage`'s build and CI run on top of, instead of repeating a ~750MB, 1700+-package `yarn install` on every PR.
- **[k8s-backstage](https://github.com/mattjmorrison-homelab/k8s-backstage)** — Helm chart deploying Backstage: namespace, Deployment/Service, a hand-rolled Postgres StatefulSet, OpenBao-backed ExternalSecrets, and Ingress. Bare-app pass only so far — no catalog discovery, CI validation, or plugins yet.
- **[graph-health](https://github.com/mattjmorrison-homelab/graph-health)** — Planned health-check GraphQL subgraph; still empty scaffolding.
- **[k8s-health](https://github.com/mattjmorrison-homelab/k8s-health)** — Planned cluster health-check deployment; still empty scaffolding.

### Admin / IaC

OpenTofu/Terraform repos administering another system's own policy or
access-control layer — not a workload deployed into the cluster or onto
a machine.

- **[admin-github](https://github.com/mattjmorrison-homelab/admin-github)** — OpenTofu config managing the `mattjmorrison-homelab` GitHub org itself (every repo, branch protection on `main`) via the `integrations/github` provider.
- **[admin-openbao](https://github.com/mattjmorrison-homelab/admin-openbao)** — OpenTofu config managing OpenBao's own policies, Kubernetes auth roles, and KV secret paths, so its whole access-control layer can be rebuilt from code if OpenBao's data is ever lost.
- **[admin-cloudflare](https://github.com/mattjmorrison-homelab/admin-cloudflare)** — Terraform config managing the homelab's Cloudflare zone and DNS records declaratively, replacing the old ad hoc `curl`-based record creation that used to live in `k8s-cloudflare`'s bootstrap script.
- **[admin-discord](https://github.com/mattjmorrison-homelab/admin-discord)** — Terraform config managing the homelab's Discord channels and webhooks, so each webhook's purpose/channel is recorded declaratively instead of tracked by memory.
- **[admin-network](https://github.com/mattjmorrison-homelab/admin-network)** — Shared network config (LAN hostname→IP map, ingress hostnames) consumed as an extra Helm values source by `k8s-pihole` and `k8s-coredns`; not a deployed workload of its own.

### Provisioning

- **[pi-provision](https://github.com/mattjmorrison-homelab/pi-provision)** — Tooling for provisioning and managing the Raspberry Pi fleet: flashing microSD cards, joining k3s nodes, and orchestrating updates via GitHub Actions.
- **[pi-health](https://github.com/mattjmorrison-homelab/pi-health)** — Standalone watchdog running on a Pi outside the cluster, checking that Prometheus is reachable — the other half of a mutual watchdog with Prometheus's own `PiNodeExporterDown` alert.
- **[steamos-provision](https://github.com/mattjmorrison-homelab/steamos-provision)** — Bootstrapping tooling for SteamOS mini-PCs; installs `node_exporter` as a per-user systemd service so it survives SteamOS's wholesale root-filesystem image swaps.

### AI tooling

- **[ai-claude](https://github.com/mattjmorrison-homelab/ai-claude)** — Claude Code's own global agents/skills/hooks configuration, not tied to any deployment destination.
