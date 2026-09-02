# mattjmorrison-homelab

A personal homelab running everything as code: a [k3s](https://k3s.io)
Kubernetes cluster, deployed and kept in sync entirely through
[ArgoCD](https://argo-cd.readthedocs.io) GitOps, with the base machines
managed declaratively via [NixOS](https://nixos.org).

## How it's organized

One repo per service, no shared "everything" manifest repo. A central
`homelab-apps` repo (ArgoCD's "app of apps") holds one `Application`
manifest per service — add a repo, register it there, ArgoCD picks it up
automatically.

## Stack

- **Cluster**: k3s, ArgoCD, Helm
- **Ingress/networking**: Traefik, Cloudflare Tunnel, Pi-hole + CoreDNS
- **Secrets**: OpenBao + External Secrets Operator
- **CI/CD**: Woodpecker, zot
- **Observability**: Prometheus, Grafana, Alertmanager
- **Security**: cert-manager (wildcard TLS via Let's Encrypt)

Every deployed service also runs a post-deploy verification check
(`PostSync` ArgoCD hook) that hits the live app after each sync and fails
the deploy fast if it doesn't come back healthy.

## Repositories

### Cluster

- **[homelab](https://github.com/mattjmorrison-homelab/homelab)** — NixOS configuration and modules (e.g. `k3s-control-plane`) for the homelab's host machines, tested via NixOS VM integration tests.
- **[homelab-apps](https://github.com/mattjmorrison-homelab/homelab-apps)** — ArgoCD "app of apps" repo holding one `Application` manifest per deployed service so ArgoCD can sync everything automatically.
- **[homelab-argocd](https://github.com/mattjmorrison-homelab/homelab-argocd)** — Bootstraps and upgrades ArgoCD itself into the k3s cluster as the tool that deploys everything else.
- **[homelab-argocd-image-updater](https://github.com/mattjmorrison-homelab/homelab-argocd-image-updater)** — Argo CD Image Updater install that watches the HDMI-switch Application and bumps its image tag via parameter overrides, with no git commits.

### Ingress/networking

- **[homelab-traefik](https://github.com/mattjmorrison-homelab/homelab-traefik)** — `HelmChartConfig` overrides for k3s's built-in Traefik that redirect all plain HTTP traffic to HTTPS.
- **[homelab-cloudflare](https://github.com/mattjmorrison-homelab/homelab-cloudflare)** — Runs `cloudflared` (Cloudflare Tunnel) to expose cluster services to the internet without port forwarding.
- **[homelab-pihole](https://github.com/mattjmorrison-homelab/homelab-pihole)** — Pi-hole deployment providing DNS/ad-blocking and local DNS records for the LAN.
- **[homelab-coredns](https://github.com/mattjmorrison-homelab/homelab-coredns)** — Custom CoreDNS config that forwards the `morrisons.site` zone to Pi-hole so pods resolve it the same way LAN clients do.

### Secrets

- **[homelab-openbao](https://github.com/mattjmorrison-homelab/homelab-openbao)** — OpenBao secrets manager deployment; the source of truth for every secret consumed elsewhere in the cluster via External Secrets Operator.
- **[homelab-external-secrets](https://github.com/mattjmorrison-homelab/homelab-external-secrets)** — Deploys External Secrets Operator, which bridges OpenBao secrets into Kubernetes Secrets for other apps.
- **[homelab-external-secrets-crds](https://github.com/mattjmorrison-homelab/homelab-external-secrets-crds)** — External Secrets Operator's CRDs, split into their own Application because they're too large for a standard Helm install.

### CI/CD

- **[homelab-woodpecker](https://github.com/mattjmorrison-homelab/homelab-woodpecker)** — CI build server (`woodpecker.morrisons.site`), integrated with OpenBao for secrets.
- **[homelab-zot](https://github.com/mattjmorrison-homelab/homelab-zot)** — Container image registry (`registry.morrisons.site`) for images built by CI.

### Observability

- **[homelab-prometheus](https://github.com/mattjmorrison-homelab/homelab-prometheus)** — Scrapes and stores cluster metrics, firing alerts to Alertmanager based on configured rules.
- **[homelab-grafana](https://github.com/mattjmorrison-homelab/homelab-grafana)** — Grafana deployment providing dashboards for cluster and host metrics sourced from Prometheus.
- **[homelab-alertmanager](https://github.com/mattjmorrison-homelab/homelab-alertmanager)** — Routes alerts fired by Prometheus to notification targets, currently a Discord webhook.
- **[homelab-kube-state-metrics](https://github.com/mattjmorrison-homelab/homelab-kube-state-metrics)** — Exports Kubernetes object state (pods, deployments, nodes) as Prometheus metrics.
- **[homelab-node-exporter](https://github.com/mattjmorrison-homelab/homelab-node-exporter)** — Exports host-level metrics (CPU, memory, disk, network) from each node to Prometheus.
- **[homelab-speedtest-exporter](https://github.com/mattjmorrison-homelab/homelab-speedtest-exporter)** — Runs periodic Ookla speed tests and exposes the results as Prometheus metrics, visualized in Grafana.

### Security

- **[homelab-cert-manager](https://github.com/mattjmorrison-homelab/homelab-cert-manager)** — Helm chart install of cert-manager, kept separate from its CRDs and its issuer/certificate configuration.
- **[homelab-cert-manager-config](https://github.com/mattjmorrison-homelab/homelab-cert-manager-config)** — cert-manager configuration: Let's Encrypt `ClusterIssuer`s via Cloudflare DNS-01 and the `*.morrisons.site` wildcard certificate.
- **[homelab-cert-manager-crds](https://github.com/mattjmorrison-homelab/homelab-cert-manager-crds)** — cert-manager's CustomResourceDefinitions, applied with server-side apply since they're too large for a normal client-side apply.

### Apps & services

- **[homelab-home-assistant](https://github.com/mattjmorrison-homelab/homelab-home-assistant)** — Home Assistant deployment (`hostNetwork` for mDNS device discovery) for home automation, at `home-assistant.morrisons.site`.
- **[homelab-homepage](https://github.com/mattjmorrison-homelab/homelab-homepage)** — Homepage dashboard deployment linking out to the cluster's other services, at `homelab.morrisons.site`.
- **[homelab-graphql-router](https://github.com/mattjmorrison-homelab/homelab-graphql-router)** — Apollo Router deployment that composes a GraphQL supergraph from subgraphs (currently the HDMI-switch service).
- **[homelab-hdmi-switch](https://github.com/mattjmorrison-homelab/homelab-hdmi-switch)** — Python/Flask + GraphQL app source, tests, and CI pipeline for controlling a TESMart HDMI switch over serial.
- **[homelab-hdmi-switch-k8s](https://github.com/mattjmorrison-homelab/homelab-hdmi-switch-k8s)** — Helm chart and manifests that deploy `homelab-hdmi-switch`, kept separate so image updates land here without git commits.
- **[homelab-hdmi-switch-ui](https://github.com/mattjmorrison-homelab/homelab-hdmi-switch-ui)** — Planned frontend for the HDMI switch service; not yet built (empty repo).
- **[k8s-matter-server](https://github.com/mattjmorrison-homelab/k8s-matter-server)** — Named for a planned Matter smart-home server deployment; not yet built (empty scaffolding).
- **[graph-health](https://github.com/mattjmorrison-homelab/graph-health)** — Named for a planned health-check GraphQL subgraph; not yet built (empty scaffolding).
- **[k8s-health](https://github.com/mattjmorrison-homelab/k8s-health)** — Named for a planned cluster health-check service; not yet built (empty scaffolding).

### Tooling

- **[pi-provision](https://github.com/mattjmorrison-homelab/pi-provision)** — Scripts to flash Raspberry Pi OS onto microSD cards and join new Pi nodes to the k3s cluster over SSH.
