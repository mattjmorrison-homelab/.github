# TODOs

General homelab improvements to revisit.

## Credentials / Secrets

- [ ] **Centralized credentials via OpenBao** — store generated passwords for all services (Grafana, Home Assistant, ArgoCD, Pihole, etc.) in OpenBao. Use ESO to sync them into Kubernetes Secrets and seed them at deploy time via init containers. Services that support it (Grafana, ArgoCD) read credentials from env vars. Services that don't (Home Assistant, Pihole) get credentials seeded into their config/storage files. Goal: one place to look up all service passwords, no manually set credentials in manifests.
- [ ] **Vaultwarden** — deploy self-hosted Bitwarden-compatible server so the Bitwarden browser extension can be pointed at the homelab instance. Complements OpenBao: OpenBao holds secrets consumed by apps/pods; Vaultwarden holds passwords used by humans in browsers. Can be seeded with service passwords from OpenBao.

## Networking

- [ ] **Pihole** — deploy DNS ad-blocking once networking is settled on a single router. See [networking.md](networking.md) for context.
- [ ] **MetalLB** — set up MetalLB as part of the networking migration. Deferred until after the Google Wifi → Deco + OPNsense migration so the IP pool only needs to be configured once. See [networking.md](networking.md) for the planned pool range.
- [ ] **Network migration** — migrate from Google Wifi to Deco + OPNsense + Protectli. See [networking.md](networking.md) for full plan.
