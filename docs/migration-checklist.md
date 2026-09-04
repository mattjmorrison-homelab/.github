# Migration checklist

Three separate, previously-scoped efforts, being executed together,
one repo at a time, rather than as three separate sweeps:

1. **Rename** — apply the new `k8s-`/`admin-`/`graph-`/`ui-`/etc. prefix
   scheme from `naming.md`.
2. **CI migration** — replace `.woodpecker.yml` with
   `.github/workflows/*.yml`, running on the new self-hosted runners
   (`k8s-github-runner`). Only applies to repos that actually have CI
   today.
3. **Secrets migration** — move the repo's KV secrets in OpenBao from
   their old combined-per-app path to the new one-path-per-key layout
   under `admin-openbao`, and drop the old-path grant from that role's
   policy once verified.

Not every repo needs all three — most only need the rename. Each repo's
entry below lists only the steps that actually apply to it.

## Per-repo steps, in order

### 1. `homelab-openbao` → `k8s-openbao`, and `admin-openbao`

**Up next.**

- [x] Rename `homelab-openbao` → `k8s-openbao` on GitHub
- [x] Rename the ArgoCD `Application` itself (`homelab-openbao` →
      `k8s-openbao`) — decided: full rename, not just a `repoURL` swap.
      Since `Application.metadata.name` has no in-place rename, did this
      as a safe adopt-then-retire sequence rather than delete-then-create,
      so the live OpenBao pod was never treated as orphaned:
      1. [x] Added the new `k8s-openbao` entry to `homelab-apps`'
         `values.yaml` (same destination/namespace, new `repoURL`), left
         the old `homelab-openbao` entry in place initially. Gotcha found
         here: without `helm.releaseName: homelab-openbao` pinned on the
         new entry's chart source, ArgoCD defaults the Helm release name
         to the Application name, which would have rendered a brand-new,
         empty `k8s-openbao` StatefulSet/PVC instead of adopting the
         existing one — pin this on every later repo in this same
         situation.
      2. [x] Pushed, `k8s-openbao` synced — adopted the existing live
         resources by updating their ArgoCD tracking annotation; metadata-
         only, didn't restart the OpenBao pod (confirmed 0 restarts,
         unchanged pod age).
      3. [x] Confirmed `k8s-openbao` showed `Synced`/`Healthy` and the
         StatefulSet's tracking annotation read `k8s-openbao:...` before
         touching the old entry.
      4. [x] Removed the old `homelab-openbao` entry from `values.yaml` —
         non-cascading as expected, old Application gone, resources
         untouched.
      Resources on disk are still literally named `homelab-openbao`
      (release name pinned) — the underlying rename (PVC pre-seed +
      `helm.releaseName` flip to `k8s-openbao`) is a separate follow-up,
      tracked below.
- [x] Rename the underlying resources (`homelab-openbao` →
      `k8s-openbao` for the StatefulSet/PVC/Service themselves, not just
      the Application object). Pre-seed the new PVC with the data so the
      new pod boots on real state instead of empty:
      1. [x] Fresh backup of `/openbao/data` (tarball off this machine) —
         `openbao-backup-2026-08-22-3.tar.gz`, taken 16:24, close to the
         actual cutover.
      2. [x] Manually create PVC `data-k8s-openbao-0` in `openbao`,
         storageClass `local-path`, size `10Gi` (matches the existing
         `data-homelab-openbao-0` PVC's spec) — created, `Pending` as
         expected (WaitForFirstConsumer, binds once step 3's pod mounts
         it).
      3. [x] Populate it: throwaway pod mounts the new PVC, `kubectl cp`
         the fresh backup in, `chown -R 1000:100` to match current live
         file ownership, delete the throwaway pod. Confirmed `1000:100`
         (busybox's `/etc/group` shows gid 100 as `users`).
      4. [x] Update the hardcoded health-check URL in
         `k8s-openbao/verify/scripts/verify.sh`
         (`http://homelab-openbao.openbao.svc:8200/v1/sys/health`) to
         `http://k8s-openbao.openbao.svc:8200/v1/sys/health` — the
         PostSync verify hook will fail otherwise once the Service is
         renamed.
      5. [x] Flip `helm.releaseName` in `homelab-apps/values.yaml` from
         `homelab-openbao` to `k8s-openbao`, push. Chart rendered
         everything as `k8s-openbao`; its volumeClaimTemplate bound the
         pre-seeded PVC from step 2 instead of creating an empty one.
         (Sync briefly retried against the pre-flip commit since the
         verify-script push landed first — self-resolved once that
         operation exhausted its retry limit and picked up the newer
         commit.)
      6. [x] Unseal via `openbao.morrisons.site` with the existing unseal
         key.
      7. [x] Verify secrets are all present and correct — all 16
         `ExternalSecret`s cluster-wide still show `SecretSynced`.
      8. [x] Remove the now-orphaned old StatefulSet + PVC
         (`data-homelab-openbao-0`) manually — StatefulSet was already
         pruned by ArgoCD; PVC deleted manually and confirmed gone.
      Gotcha hit here, relevant to every remaining rename in this
      checklist: renaming the Service broke every other repo that
      hardcoded `homelab-openbao.openbao.svc` — 19 files across 12 repos
      (`SecretStore`s, a couple of bootstrap Jobs, one live Deployment env
      var, one Prometheus scrape target). Not caught until an
      `ExternalSecret` visibly failed after the fact. Grep every other
      repo for a service's old hostname *before* renaming it, next time.
- [ ] `admin-openbao` CI: replace `.woodpecker.yml` with a GitHub Actions
      workflow (`tofu plan` on PR/push, `tofu apply` on push to `main`),
      running on `k8s-github-runner`'s `k8s-amd64` scale set.
      `VAULT_TOKEN` needs to move from Woodpecker's native secret store to
      a GitHub Actions secret. Workflow (`tofu.yml`) is written and pushed,
      actions pinned to SHAs; currently blocked on a bootstrap deadlock,
      found 2026-08-23: the workflow fetches the tofu-state S3 credentials
      via a Vault Kubernetes-auth login (role `github-actions-runner` in
      `locals.tf`), bound to a dedicated `github-runner-workload`
      ServiceAccount (added this session, replacing a stale ARC-era SA
      reference). That role binding only takes effect in Vault once
      `tofu apply` actually runs — but the only way to run it is through
      this same CI pipeline, which can't authenticate to fetch its own
      state-backend credentials until the apply has already happened.
      Needs one manual, out-of-band `tofu apply` (local Vault/AWS creds,
      not through CI) to break the deadlock; after that, CI's own logins
      should work going forward.
- [x] `admin-openbao` bootstrap apply: pushed the full `locals.secrets` +
      `locals.roles` target state (all 14 apps) in one shot rather than
      the originally-planned comment-out-to-minimal approach — confirmed
      safe first, since `secrets.tf` only ever creates *new* one-path-
      per-key entries (old paths aren't in this resource block at all,
      can't be touched) and every policy in `auth.tf` grants the old path
      *plus* the new one, so nothing currently working loses access.
      Needed a manual one-time step first: `vault_root_token` added to
      Woodpecker's Settings → Secrets for this repo (documented in the
      repo's own README, can't be automated — it's the credential that
      unlocks OpenBao, so it can't come from OpenBao itself). Ran
      successfully in Woodpecker.
- [ ] Per-app secrets cutover (this is the part that's still genuinely
      one-at-a-time, and can't be skipped): for each of the 14 apps —
      copy its real secret value(s) from the old combined path into the
      new blank one-path-per-key path, repoint/verify that app's
      `ExternalSecret`(s) read correctly from the new path, then drop
      that role's old-path grant from `locals.tf` and `tofu apply` again.
- [ ] No secrets of OpenBao's own to migrate — this app manages
      everyone else's, has none itself in `locals.secrets`.

### 2. `homelab-woodpecker` → `k8s-woodpecker`

- [ ] Rename
- [ ] Secrets: `github-client`, `github-secret`, `agent-secret`,
      `vault-token`, `prometheus-auth-token`, `zot-ci-password` — old
      path `kv/homelab/woodpecker`, new `kv/homelab/k8s-woodpecker/*`
- [ ] No CI migration — Woodpecker is the thing being retired, not a
      consumer of it

### 3. `admin-github` (already correctly named)

- [ ] CI: replace `.woodpecker.yml` with a GitHub Actions workflow
      (`tofu plan`/`apply`, same shape as `admin-openbao`'s)
- [ ] Secrets: `github-token`, `tofu-state-access-key-id`,
      `tofu-state-secret-access-key` — old path `kv/homelab/gh-org`, new
      `kv/homelab/admin-github/*`

### 4. `k8s-garage` (already correctly named)

- [ ] Secrets: `rpc-secret`, `admin-token`, `metrics-token` — old path
      `kv/homelab/garage`, new `kv/homelab/k8s-garage/*` (also holds a
      cross-repo grant into `admin-github`'s path for the Tofu state
      bucket credentials — check that still resolves correctly after
      `admin-github`'s own secrets move)
- [ ] No CI, no rename needed

### 5. `k8s-github-runner` (already correctly named, brand new)

- [ ] Nothing to migrate — built directly against the target
      conventions (kebab-case secrets, `k8s-` prefix) from the start

### 6. `homelab-hdmi-switch` / `homelab-hdmi-switch-k8s` / `homelab-hdmi-switch-ui`

- [ ] Rename: `homelab-hdmi-switch` → `graph-hdmi-switch`,
      `homelab-hdmi-switch-k8s` → `k8s-hdmi-switch`,
      `homelab-hdmi-switch-ui` → `ui-hdmi-switch`
- [ ] CI (both `graph-hdmi-switch` and `ui-hdmi-switch` have
      `.woodpecker.yml` today): migrate both to GitHub Actions
- [ ] Secrets: `zot-ci-password` — old path `kv/homelab/hdmi-switch`, new
      `kv/homelab/k8s-hdmi-switch/*`
- [ ] Woodpecker's repo list needs a manual resync after the rename to
      pick up the new repo names

### 7. `homelab-graphql-router` → `k8s-graphql-router`

- [ ] Rename
- [ ] Secrets: `zot-ci-password` — old path `kv/homelab/graphql-router`,
      new `kv/homelab/k8s-graphql-router/*`
- [ ] No CI of its own found locally — confirm before skipping

### 8. `graph-router` (already correctly named)

- [ ] CI: has `.woodpecker.yml` today — migrate to GitHub Actions

### 9. `homelab-cloudflare` → `k8s-cloudflare`

- [ ] Rename
- [ ] Secrets: `account-tag`, `tunnel-id`, `tunnel-secret`,
      `cloudflare-api-token`, `cf-account-id` — consolidates **two** old
      paths (`kv/homelab/cloudflare`, `kv/homelab/tunnel`) into
      `kv/homelab/k8s-cloudflare/*`
- [ ] No CI

### 10. `homelab-argocd` → `k8s-argocd`

- [ ] Rename
- [ ] Secrets: `discord-webhook-url`, `github-webhook-secret` —
      consolidates the webhook role's old path (`kv/homelab/argocd`) and
      the notifications role's old path
      (`kv/homelab/argocd-notifications`) into `kv/homelab/k8s-argocd/*`
- [ ] No CI

### 11. `homelab-argocd-image-updater` → `k8s-argocd-image-updater`

- [ ] Rename
- [ ] Secrets: `zot-ci-password` — old path
      `kv/homelab/argocd-image-updater`, new
      `kv/homelab/k8s-argocd-image-updater/*`
- [ ] No CI

### 12. `homelab-cert-manager-config` → `k8s-cert-manager-config`

- [ ] Rename
- [ ] Secrets: `cloudflare-api-token` — old path
      `kv/homelab/certmanager`, new
      `kv/homelab/k8s-cert-manager-config/*`
- [ ] No CI

### 13. `homelab-prometheus` → `k8s-prometheus`

- [ ] Rename
- [ ] Secrets: `woodpecker-prometheus-auth-token` — old path
      `kv/homelab/prometheus`, new `kv/homelab/k8s-prometheus/*`
- [ ] No CI

### 14. `homelab-alertmanager` → `k8s-alertmanager`

- [ ] Rename
- [ ] Secrets: `discord-webhook-url` — old path
      `kv/homelab/alertmanager`, new `kv/homelab/k8s-alertmanager/*`
- [ ] No CI

### 15. `homelab-zot` → `k8s-zot`

- [x] Rename the ArgoCD `Application` itself (`homelab-zot` →
      `k8s-zot`) using the adopt-then-retire sequence:
      1. [x] Added the new `k8s-zot` entry to `homelab-apps`'
         `values.yaml` (same destination/namespace, new `repoURL`), left
         the old `homelab-zot` entry in place initially.
      2. [ ] Push and sync — `k8s-zot` will adopt the existing live
         resources by updating their ArgoCD tracking annotation.
      3. [ ] Confirm `k8s-zot` shows `Synced`/`Healthy` before touching
         the old entry.
      4. [ ] Remove the old `homelab-zot` entry from `values.yaml` —
         non-cascading as expected, old Application gone, resources
         untouched.
- [ ] Secrets: `htpasswd` — old path `kv/homelab/zot`, new
      `kv/homelab/k8s-zot/*`
- [ ] No CI

### 16. Rename-only, no secrets, no CI

Everything else in `naming.md`'s table that isn't listed above — just the
GitHub rename, the `repoURL` update in `homelab-apps`, and (for the one
NixOS host) the local flake path:

- [ ] `homelab` → `nix-control-plane`
- [ ] `homelab-apps` → `k8s-apps`
- [ ] `homelab-cert-manager` → `k8s-cert-manager`
- [ ] `homelab-cert-manager-crds` → `k8s-cert-manager-crds`
- [ ] `homelab-coredns` → `k8s-coredns`
- [ ] `homelab-external-secrets` → `k8s-external-secrets`
- [ ] `homelab-external-secrets-crds` → `k8s-external-secrets-crds`
- [ ] `homelab-grafana` → `k8s-grafana`
- [ ] `homelab-home-assistant` → `k8s-home-assistant`
- [ ] `homelab-homepage` → `k8s-homepage`
- [ ] `homelab-kube-state-metrics` → `k8s-kube-state-metrics`
- [ ] `homelab-metallb` → `k8s-metallb`
- [ ] `homelab-minecraft` → `k8s-minecraft`
- [ ] `homelab-node-exporter` → `k8s-node-exporter`
- [ ] `homelab-pihole` → `k8s-pihole`
- [ ] `homelab-speedtest-exporter` → `k8s-speedtest-exporter`
- [ ] `homelab-traefik` → `k8s-traefik`
- [ ] `homelab-claude` → `ai-claude`
- [ ] `homelab-raspberrypi` → split into `pi-provision` + a separate
      apps repo (see `naming.md` for the split plan)

## Decision carried over from `naming.md`'s open question

`homelab-apps`' `Application` object names adopt the new prefix scheme
too, not just the `repoURL` — full rename, every repo, using the
adopt-then-retire sequence spelled out under repo #1 above rather than a
delete-then-create.
