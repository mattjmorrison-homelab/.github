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

# TODO: home CI identities in a dedicated namespace, separate from their target namespace

Today `k8s-ci-rbac`'s per-consumer ServiceAccounts (`prometheus-ci`,
`grafana-ci`, `alertmanager-ci`, `argocd-ci`, `garage-ci`, ...) live in
the *same* namespace as the real app they validate (`monitoring`,
`argocd`, `garage`, ...) -- mixed in with production service accounts.

A `RoleBinding`'s subject can reference a ServiceAccount from a
different namespace, so the identity itself could be homed in a
dedicated namespace instead (e.g. `github-runner`, where
`github-runner-workload` itself already lives) while the `Role`/
`RoleBinding` granting it access stays in the target namespace (Roles
are inherently namespace-scoped, that part can't move without
switching to a much broader ClusterRole).

Not a security change -- the granted permission scope is identical
either way. Purely organizational: `kubectl get sa -n monitoring`
would show only real app service accounts, not CI helpers mixed in,
and every CI identity would be auditable/rotatable from one place.

- [ ] Pick a dedicated namespace for CI identities (reuse
      `github-runner`, or a new one).
- [ ] Update `k8s-ci-rbac`'s template to create the ServiceAccount in
      that namespace while keeping the Role/RoleBinding in each
      consumer's own namespace.
- [ ] No changes needed in consumer repos -- `service-account: <name>`
      in each `check.yml` stays the same either way.
