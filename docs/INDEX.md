# Homelab Docs

This homelab runs a k3s Kubernetes cluster across several machines, all managed as code. NixOS configures the hosts, ArgoCD manages all deployments, and each service lives in its own repo.

## Architecture

```text
NixOS (homelab) → k3s cluster → ArgoCD (homelab-argocd) → services (homelab-apps → homelab-*)
```

Secrets flow: OpenBao stores secrets → External Secrets Operator syncs them into Kubernetes Secrets → apps consume them.

## Docs

- [Cluster](cluster.md) — node inventory, Pi 5 agent node setup plan
- [Networking](networking.md) — current network layout and planned architecture
- [TODOs](todos.md) — general homelab improvements to revisit

## Repos

| Repo | Description |
| --- | --- |
| [homelab](https://github.com/mattjmorrison/homelab) | NixOS modules and integration tests for cluster hosts |
| [homelab-apps](https://github.com/mattjmorrison/homelab-apps) | ArgoCD App of Apps — one Application manifest per deployed service |
| [homelab-argocd](https://github.com/mattjmorrison/homelab-argocd) | Bootstrap manifests to install ArgoCD; run once |
| [homelab-prometheus](https://github.com/mattjmorrison/homelab-prometheus) | Prometheus — scrapes and stores cluster metrics |
| [homelab-alertmanager](https://github.com/mattjmorrison/homelab-alertmanager) | Alertmanager — routes alerts from Prometheus to Discord |
| [homelab-grafana](https://github.com/mattjmorrison/homelab-grafana) | Grafana — dashboards for cluster metrics |
| [homelab-node-exporter](https://github.com/mattjmorrison/homelab-node-exporter) | Node Exporter — exports host-level metrics to Prometheus |
| [homelab-kube-state-metrics](https://github.com/mattjmorrison/homelab-kube-state-metrics) | Kube State Metrics — exports Kubernetes object state to Prometheus |
| [homelab-openbao](https://github.com/mattjmorrison/homelab-openbao) | OpenBao (open-source Vault) — stores all cluster secrets |
| [homelab-external-secrets](https://github.com/mattjmorrison/homelab-external-secrets) | External Secrets Operator — syncs OpenBao secrets into Kubernetes Secrets |
| [homelab-external-secrets-crds](https://github.com/mattjmorrison/homelab-external-secrets-crds) | External Secrets Operator CRDs — installed separately due to size |
| [homelab-home-assistant](https://github.com/mattjmorrison/homelab-home-assistant) | Home Assistant — home automation platform with local device discovery |
