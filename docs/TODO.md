# TODO: hdmi-switch commit-hash verification

For most repos, ArgoCD's own `Synced`/`Healthy` status already guarantees
the right version is running — it can't report Synced unless the live
image digest matches git. That guarantee is weaker for
`homelab-hdmi-switch` / `homelab-hdmi-switch-ui`, because Argo CD Image
Updater's write-back bypasses the normal human-commits-then-ArgoCD-diffs
flow.

Idea: have those two expose a `/version` (or similar) endpoint returning
the deployed commit SHA (baked in at build time via Woodpecker), and have
`verify.sh` curl it and assert it matches the expected commit. Needs
investigation into exactly how the Image Updater write-back works first,
plus real app-code and Woodpecker pipeline changes — bigger than a
manifests-only change. Every other repo keeps the plain 200-check
(post-deploy verify scripts are done and live everywhere else).

---

# TODO: OpenBao continuous seal-status monitoring

Prompted by a real incident: a pod restart left OpenBao sealed, which broke
unrelated things before anyone noticed. The `PostSync` verify check only
runs at deploy time (and a plain pod restart isn't a deploy), so it
wouldn't have caught this — need real continuous monitoring instead.

Plan: reuse the existing Prometheus + Alertmanager → Discord pipeline
(already has 13 alert rules wired up in `homelab-prometheus`) rather than
building a separate mechanism.

- [ ] `homelab-openbao` — enable the `telemetry` stanza in
      `server.standalone.config` (exposes `vault_core_unsealed` — the
      standard Vault/OpenBao gauge metric, 1=unsealed/0=sealed — via
      Prometheus format at `/v1/sys/metrics`)
- [ ] `homelab-prometheus` — add a scrape job for OpenBao in
      `prometheus-config` (same `static_configs` shape as the existing
      `kube-state-metrics`/`argocd` jobs)
- [ ] `homelab-prometheus` — add an alert rule in `prometheus-alerts`:
      `vault_core_unsealed == 0`, same shape as the existing 13 rules

---

# TODO: check homelab-traefik's /ping endpoint

`homelab-traefik`'s verify check is currently TCP-only (`traefik.kube-system.svc:443`).
Traefik has a built-in `/ping` endpoint (returns 200 when healthy) if enabled
in its static config — worth checking whether it's turned on here, and if so
switching the verify check to a real HTTP 200 assertion instead of TCP-only.

- [ ] While rolling this out to `homelab-cloudflare` and `homelab-woodpecker`,
      also pin their existing bootstrap Jobs' `alpine:3.20` (currently a
      mutable tag) to a digest, same as `verify.image`.

---

# TODO: convert admin-network to a Helm library chart

`admin-network`'s `local-hosts.yaml`/`ingress-hosts.yaml` currently reach
`k8s-pihole`/`k8s-coredns` via ArgoCD multi-source Applications (a second
`source` in `k8s-apps`, referenced by `ref: network` and pulled in via
`$network/local-hosts.yaml` in `helm.valueFiles`). Decided this should
become a real Helm library chart dependency instead, for one reason:
consistency. `k8s-lib-ci-rbac` already established "shared code between
charts is a helm-lib" as the pattern for this homelab: one way to do
cross-chart sharing, not two different mechanisms doing the same job.

Known cost, not free: unlike the current approach (plain git, no
registry), a chart dependency needs OCI publishing to
`registry.morrisons.site` plus a private-registry pull credential for
every consumer's own CI to run `helm dependency build` -- the same
OpenBao/`k8s-zot` service-consumer chain that's still unresolved for
`k8s-garage`'s `k8s-lib-ci-rbac` dependency. Worth sorting that out
first, or accepting the same block here.

- [ ] Resolve (or accept) the registry-auth blocker before converting,
      since this would hit it too.
- [ ] Convert `admin-network` into a values-only chart dependency,
      consumed by `k8s-pihole`/`k8s-coredns` via `Chart.yaml`
      `dependencies`, not ArgoCD multi-source.
- [ ] Remove the `ref: network` / `$network/...` multi-source wiring from
      both Applications in `k8s-apps` once converted.
