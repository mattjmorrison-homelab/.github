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
