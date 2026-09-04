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

# TODO: revisit ArgoCD Application namespace/destination scoping

Found a stale `namespace` field on `k8s-apps`' `k8s-ci-rbac` entry
(pointed at `garage` instead of `monitoring`) while working through CI
RBAC setup. Wasn't actually breaking anything — the chart hardcodes its
own object namespaces — but the field was wrong and nobody would've
noticed. Worth a pass over how these fields get set/scoped generally.

Related idea, GitHub side: there's no real grouping mechanism for repos
on GitHub (flat namespace per org, no folders/subgroups) -- Teams,
Topics, and paid-tier custom properties are the only tools. Would be
nice if a k8s namespace like `monitoring` (spanning `k8s-prometheus`,
`k8s-alertmanager`, `k8s-grafana`, `k8s-ci-rbac`'s monitoring-scoped
consumers) had some visible equivalent on the GitHub side -- e.g. a
`monitoring` Team those repos live under, or a shared Topic tag. Not
scoped/decided yet, just captured as worth exploring alongside the
namespace-scoping pass above.
