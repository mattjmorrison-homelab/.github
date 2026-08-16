# Home Lab Plan

## Hardware / Nodes

- **iMac** (repurposed, 32GB RAM, 1TB disk) — currently Ubuntu, to be reinstalled with **NixOS**. Will run the k3s **control plane** plus act as a worker node.
- **Raspberry Pis** (various: 16GB, 8GB, 2GB, and an original low-memory model) — worker nodes. Considering NixOS across the board, TBD if that's practical on all Pi models.
- **Mini PC** (currently SteamOS/Arch, ~32GB RAM, used by daughter for Minecraft/Roblox) — potential additional worker node when idle.
- One Raspberry Pi is physically wired via serial port to a media device (for an app that must run specifically on that Pi).
Will use node labels + node affinity/selectors so that app is pinned there, while still allowing other workloads to use spare capacity on that Pi.

## Orchestration

- **k3s** (lightweight Kubernetes) chosen over full Kubernetes/Docker Compose given multiple physical machines.
- Control plane runs on the iMac; Pis and mini PC join as agent nodes.
- k3s auto-discovers node resources (CPU/RAM) and schedules workloads across the pool.
- Node failures: workloads reschedule automatically to healthy nodes. Control plane failure is a bigger risk (single point of failure) — acceptable tradeoff for a home lab.

## Planned Applications

- Pihole (ad-blocking DNS)
- Minecraft server
- Media controller (serial-port app, pinned to specific Pi)
- Home Assistant
- A Python-based application

## Deployment Model

- **Preference:** each application owns its own repo, containing its own manifest(s)/resource definitions, rather than one big manifest repo.
- **ArgoCD** to stitch it together: watches multiple app repos and syncs their manifests to the k3s cluster (GitOps model) — no manual `kubectl apply` needed once set up.
- ArgoCD itself runs as a workload inside the k3s cluster.

## CI/CD

- Want to avoid GitHub Actions usage/cost — prefer self-hosted CI running inside the k3s cluster.
- Candidates discussed: **GitLab Runner** (self-hosted), **Tekton** (Kubernetes-native), Woodpecker, Gitea CI.
- CI pipeline responsibility: build Docker image → push to internal registry → update manifest version/tag → ArgoCD detects and deploys.

## Container Registry

- Run a **private Docker image registry** inside the cluster (e.g., Harbor or a simple registry container) instead of relying on Docker Hub.

## Source Control

- Keep using **GitHub**, but with a **private personal GitHub organization** (free for private repos) — not shared publicly.
- Repo separation: NixOS baseline config repo (bootstraps machine + installs k3s) kept separate from per-application repos.

## Monitoring & Alerting

- **Prometheus** — scrapes/collects metrics from apps and infrastructure (pull-based).
- **Alertmanager** — handles alerting rules/notifications based on Prometheus data.
- **Grafana** — dashboards/visualization, queries Prometheus as a data source.
- Pihole metrics via a Pihole Prometheus exporter (pre-built Grafana dashboards available).
- All of the above (Prometheus, Alertmanager, Grafana) run as workloads inside the k3s cluster.

## Open Questions / To Revisit

- Whether NixOS is practical/worth it on lower-memory Raspberry Pi models.
- Final choice of self-hosted CI tool (GitLab Runner vs. Tekton vs. others).
- Ingress controller setup for routing internal DNS names to the right in-cluster services.
