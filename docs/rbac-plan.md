# Convert k8s-ci-rbac into a reusable Helm library chart, and retire Woodpecker

## Repos touched

| Repo (before) | Repo (after) | What changes |
| --- | --- | --- |
| `k8s-ci-rbac` | `k8s-lib-ci-rbac` | Renamed. Converted to a Helm library chart; gains a GitHub Actions publish-on-merge workflow |
| `actions-helm` | *(no rename)* | `check.sh`/`action.yml` gain a dependency-build step + optional `service-account` input |
| `homelab-argocd` | *(no rename)* | New OCI repo-creds `Secret`/`ExternalSecret` for `argocd-repo-server` |
| `homelab-apps` | *(no rename)* | Old `k8s-ci-rbac` Application entry removed; `homelab-woodpecker` Application entry removed at the end |
| `k8s-garage` | *(no rename)* | New `dependencies:` entry + `templates/ci-rbac.yaml` |
| `k8s-github-runner` | *(no rename)* | New `dependencies:` entry + `templates/ci-rbac.yaml` |
| `homelab-homepage` | *(no rename)* | New `dependencies:` entry + `templates/ci-rbac.yaml` |
| `k8s-prometheus` | *(no rename)* | New `dependencies:` entry + `templates/ci-rbac.yaml`; `check.yml` gets `service-account: prometheus-ci` |
| `homelab-alertmanager` | *(no rename)* | New `dependencies:` entry + `templates/ci-rbac.yaml`; `check.yml` gets `service-account: alertmanager-ci` |
| `graph-router` | *(no rename)* | `.woodpecker.yml` replaced with a GitHub Actions publish workflow |
| `ui-hdmi-switch` | *(no rename)* | `.woodpecker.yml` replaced with a GitHub Actions publish workflow |
| `graph-hdmi-switch` | *(no rename)* | `.woodpecker.yml` replaced with a GitHub Actions publish workflow |
| `homelab-woodpecker` | *(retired)* | Decommissioned once nothing depends on it — Application entry removed, repo archived |

`k8s-ci-rbac`/`k8s-lib-ci-rbac` is the only repo being renamed.
`homelab-woodpecker` is the only one being retired outright — everything
else keeps its current name.

## Context

`k8s-ci-rbac` currently generates CI dry-run RBAC (ServiceAccount + Role +
RoleBinding, per namespace) for 4 targets in one shared Helm chart, deployed
by a single ArgoCD Application that lands resources across 4 different
namespaces (`garage`, `monitoring`, `github-runner`, `homepage`). ArgoCD is
silently dropping 3 of the 4 `*-ci-token-issuer` RoleBindings from its own
computed sync-target list — confirmed not to be a template, cache, admission
webhook, AppProject, or `argocd-cm` config issue (all ruled out), and a
content-uniqueness test (adding a distinguishing label) didn't fix it either.
With no further read-only way to pin the exact ArgoCD-internal cause, the
better fix is architectural: stop having one Application span multiple
namespaces at all, which is also the "one repo = one namespace" convention
every other repo in this homelab already follows.

The fix: turn `k8s-ci-rbac` into a Helm **library chart** — the RBAC
template lives in one place for reuse, but each owning repo's own chart
calls it and renders its own CI RBAC as part of its own single-namespace
Application. This also surfaced a real second issue: `monitoring` has two
independent owners (`k8s-prometheus` and `homelab-alertmanager`), not one.
Per the decision made when planning this, they get **separated** — each
gets its own CI identity (`prometheus-ci` / `alertmanager-ci`) rather than
sharing `monitoring-ci`, so there's no shared-resource ownership conflict
between two ArgoCD Applications.

**Second, unrelated decision folded into this same plan**: while designing
`k8s-ci-rbac`'s new publish-on-merge pipeline, it became clear Woodpecker is
being retired — so the new pipeline goes straight to GitHub Actions instead
of `.woodpecker.yml`, and while touching this, the remaining Woodpecker
consumers (`graph-router`, `ui-hdmi-switch`, `graph-hdmi-switch`) get
migrated off it too, ending with `homelab-woodpecker` itself decommissioned.

## Repo rename: `k8s-ci-rbac` → `k8s-lib-ci-rbac`

`.github/docs/naming.md` already reserves a `k8s-lib-` prefix for exactly
this case — "a Helm library chart (`type: library`)... never deployed or
synced on its own... reserved for if/when one's needed" — and notes no repo
has used it yet. This conversion is that first case, so the rename should
happen as part of this work, not deferred to the separate org-wide renaming
initiative tracked in `TODO.md`/`naming.md` (which covers the *existing*
`homelab-*` fleet and hasn't been greenlit to execute yet — this is
different: a repo taking on new semantics, not a retrofit).

- Real GitHub rename (`k8s-ci-rbac` → `k8s-lib-ci-rbac`) — a real operation
  the user runs themselves, done early (before/alongside the library-chart
  conversion) since the new GitHub Actions publish workflow and any repo
  references should target the final name, not get set up then renamed
  again.
- Local clone directory: `~/Projects/k8s-ci-rbac` → `~/Projects/k8s-lib-ci-rbac`.
- `naming.md`: update the `k8s-lib-` bullet to drop "not built yet" and
  point at `k8s-lib-ci-rbac` as the first real example.
- Everywhere else in this plan that says `k8s-ci-rbac`, read as
  `k8s-lib-ci-rbac` (repo name, OCI chart name/path, publish workflow,
  consumers' `dependencies:` entries, etc.) — written as `k8s-ci-rbac`
  below only because that's its name until the rename happens.

## Design

**`k8s-ci-rbac` becomes a library chart:**
- `Chart.yaml`: add `type: library`, bump `version`.
- Delete `values.yaml` (no more `ciTargets` list — nothing to loop over,
  each caller passes its own params).
- Replace `templates/rbac.yaml` with `templates/_rbac.tpl` defining a named
  template `k8s-ci-rbac.rbac`, called via
  `{{ include "k8s-ci-rbac.rbac" (dict "namespace" .Release.Namespace "apiGroups" (list ...) "serviceAccountName" "prometheus-ci") }}`.
  - `serviceAccountName` optional, defaults to `printf "%s-ci" .namespace`
    (preserves today's exact resource names for garage/github-runner/homepage
    — no forced renaming for the 3 single-owner namespaces).
  - Resource names derive from `serviceAccountName` (e.g.
    `{{ $serviceAccountName }}-writer`, `-token-issuer`), not from
    `.namespace` — this is what lets `monitoring`'s two owners coexist.
  - `github-runner-workload`/`github-runner` (the shared runner identity)
    stays hardcoded inside the template, not parameterized — it doesn't
    vary today.
  - Body is a straight port of the existing per-namespace block in the
    current `templates/rbac.yaml` (ServiceAccount, writer Role/RoleBinding,
    token-issuer Role/RoleBinding), just de-looped and parameterized.
- `README.md`: rewrite to describe the library-chart usage pattern
  (replace the "add an entry to values.yaml" instructions with "add a
  dependency + call the template").
- `.github/workflows/check.yml`: this repo no longer deploys itself, so
  drop the `actions-helm` call entirely — replace with a plain `helm lint`
  step (nothing to dry-run against a live namespace anymore).
- **New**: `.github/workflows/publish.yml` — GitHub Actions workflow
  triggered on push to `main`, publishing the chart to the OCI registry:
  `helm registry login registry.morrisons.site -u ci -p <password>` →
  `helm package manifests -d dist --version 0.1.0-<short-sha>` → `helm push
  dist/*.tgz oci://registry.morrisons.site/charts`. Version is derived from
  the commit SHA (mirrors this homelab's existing "pin to a digest, bump
  deliberately" convention for container images) so every push is uniquely
  addressable and consumers pin an exact version. Credential fetch should
  follow whatever mechanism `actions-tofu/fetch-credentials/fetch-credentials.sh`
  already uses (read that script before authoring this — if it pulls
  directly from OpenBao via the runner's in-cluster identity, mirror that
  rather than inventing a separate GitHub-level secret for the zot
  password).

**New infra required (doesn't exist yet anywhere in this homelab):**
1. **ArgoCD OCI repo-creds** — a `Secret` labeled
   `argocd.argoproj.io/secret-type: repo-creds` in the `argocd` namespace
   (`type: helm`, `enableOCI: "true"`, `url: registry.morrisons.site/charts`,
   `username: ci`, password from OpenBao) so `argocd-repo-server` can
   resolve the `k8s-ci-rbac` chart dependency at sync time. Likely lives in
   `homelab-argocd`, sourced via an `ExternalSecret` following the same
   pattern as `homelab-woodpecker`'s `external-secret-zot-pull.yaml` (copy
   the pattern before that repo is decommissioned, reusing the existing
   `ci`/`ZOT_CI_PASSWORD` OpenBao credential).
2. **`actions-helm` dependency-build step** — `check.sh` needs `helm
   dependency build "$CHART_PATH"` added before `helm lint`/`helm template`
   (currently absent entirely; confirmed no repo has ever declared Helm
   chart dependencies). Also add an optional `service-account` input to
   `action.yml`/`check.sh` (env var, defaulting to `${NAMESPACE}-ci` —
   today's exact behavior) so `k8s-prometheus` and `homelab-alertmanager`
   can each tell the dry-run step which CI ServiceAccount to mint a token
   for, since they no longer share `monitoring-ci`. Zero change in
   behavior for the ~24 other repos that don't set it.
   `test/mocks/helm`'s bats mock needs a `dependency)` case added so the
   test suite doesn't break.

**Per-consumer changes (same shape for all 4, apply one at a time):**
- `Chart.yaml`: add
  ```yaml
  dependencies:
    - name: k8s-ci-rbac
      version: "<pinned version>"
      repository: "oci://registry.morrisons.site/charts"
  ```
- New `templates/ci-rbac.yaml` calling `k8s-ci-rbac.rbac` with that repo's
  own `apiGroups` list (same values already in `k8s-ci-rbac`'s current
  `values.yaml` — no need to re-derive):
  - `k8s-garage`: `["", "apps", "batch", "external-secrets.io"]`
  - `k8s-github-runner`: `["", "apps", "external-secrets.io"]`
  - `homelab-homepage`: `["", "apps", "batch", "networking.k8s.io"]`
  - `k8s-prometheus`: `["", "apps", "batch", "external-secrets.io", "networking.k8s.io"]`, `serviceAccountName: "prometheus-ci"`
  - `homelab-alertmanager`: same apiGroups as prometheus (its own manifests
    need the same set — deployment/service/configmap/external-secret/
    ingress/verify-job), `serviceAccountName: "alertmanager-ci"`
- That repo's own `check.yml` gets `service-account: prometheus-ci` /
  `service-account: alertmanager-ci` added (only these two repos need this
  input at all).
- No changes needed to `k8s-garage`, `k8s-github-runner`, or
  `homelab-homepage`'s `check.yml` — default SA-name behavior covers them.

**Rollout order** (one repo at a time, matching the existing
sequential-rollout preference — never all repos in one pass):
1. Ship the new infra first (ArgoCD repo-creds, `actions-helm` changes) —
   nothing depends on it yet, so it's safe to land alone.
2. Convert `k8s-ci-rbac` to a library chart, publish it via the new GitHub
   Actions workflow.
3. Convert **one** consumer (suggest `homelab-homepage` first — it has zero
   existing RBAC of its own today, so it's the lowest-risk proof of
   concept) end-to-end: add the dependency, deploy, confirm the new
   `homepage-ci` SA/Role/RoleBinding show up correctly in-cluster and that
   `homelab-homepage`'s own CI dry-run step can mint a token and pass.
4. Once proven, convert the remaining 3 (`k8s-garage`, `k8s-github-runner`,
   then the `monitoring` pair together since they share a namespace) the
   same way, one at a time.
5. Only after all 4 are confirmed working: remove the old `k8s-ci-rbac`
   entry from `homelab-apps` (this prunes the old shared SA/Role/RoleBinding
   set) — do this last so there's no gap where a not-yet-converted
   consumer's CI loses its RBAC mid-migration.
6. Migrate `graph-router` off Woodpecker to a GitHub Actions publish
   workflow (its `.woodpecker.yml` is the simplest of the three — a single
   kaniko `build-release` step gated on `branch: main` — closest to
   `k8s-ci-rbac`'s new workflow, good second proof of concept for the
   pattern). Verify it still builds and pushes the image correctly.
7. Migrate `ui-hdmi-switch`, then `graph-hdmi-switch` (the fullest pipeline
   — build-test → test → build-release-on-main → verify-deploy →
   notify-discord — read its current `.woodpecker.yml` in full before
   converting; don't drop stages), one at a time.
8. Once all three consumers are confirmed off Woodpecker: remove
   `homelab-woodpecker`'s Application entry from `homelab-apps` and archive
   the repo.

## Files to touch

- `k8s-ci-rbac`: `Chart.yaml`, `values.yaml` (delete), `templates/rbac.yaml`
  → `templates/_rbac.tpl`, `README.md`, `.github/workflows/check.yml`,
  new `.github/workflows/publish.yml`
- `actions-helm`: `check.sh`, `action.yml`, `test/mocks/helm`,
  `test/check.bats` (add dependency-build coverage)
- `homelab-argocd`: new repo-creds `Secret`/`ExternalSecret` template
- `homelab-apps`: remove the `k8s-ci-rbac` Application entry; remove the
  `homelab-woodpecker` Application entry (final step)
- `k8s-garage`, `k8s-github-runner`, `homelab-homepage`, `k8s-prometheus`,
  `homelab-alertmanager`: `Chart.yaml` + new `templates/ci-rbac.yaml` each;
  `k8s-prometheus`/`homelab-alertmanager` also get `check.yml` updated
- `graph-router`, `ui-hdmi-switch`, `graph-hdmi-switch`: `.woodpecker.yml`
  deleted, replaced by a new `.github/workflows/publish.yml` each (exact
  stages ported from each repo's current pipeline — read them first,
  especially `graph-hdmi-switch`'s multi-stage one)
- `homelab-woodpecker`: repo archived once retired (no further code
  changes inside it)

## Manual steps

Everything else in this plan is a code change ArgoCD/GitHub Actions apply
automatically. These don't fall out of a `git push` — someone has to do
them by hand:

- **GitHub repo rename**: `k8s-ci-rbac` → `k8s-lib-ci-rbac` (GitHub UI or
  `gh repo rename`). GitHub redirects the old URL, but do this before
  wiring the new publish workflow so it targets the final name.
- **Local clone rename**: `mv ~/Projects/k8s-ci-rbac ~/Projects/k8s-lib-ci-rbac`.
- **Confirm the credential mechanism for GitHub Actions → registry.morrisons.site**:
  read `actions-tofu/fetch-credentials/fetch-credentials.sh` first — if CI
  runners (self-hosted on `k8s-github-runner`, in-cluster) can already
  reach OpenBao directly, reuse that instead of creating a new GitHub-level
  secret for the zot `ci` password. If a new GitHub secret genuinely is
  needed, add it (org or repo level) manually — never commit it.
- **New OpenBao key for ArgoCD's OCI pull credential**: the existing zot
  `ci` user's password is already in OpenBao, but per this homelab's
  established convention every consumer stores its **own copy** at its own
  `homelab/<app>` path (e.g. `homelab/homelab-woodpecker` →
  `ZOT_CI_PASSWORD`) rather than sharing one path — so ArgoCD needs its own
  copy too. Manually write the existing `ci` zot password into a new
  OpenBao path, suggest `homelab/argocd` → key `ZOT_CI_PASSWORD` (same
  value already used elsewhere, just a new path so `homelab-argocd`'s
  `ExternalSecret` has something to point at). This is a real secret value
  — do it via the OpenBao UI/CLI directly, never commit it.
- **Verify `homelab-argocd`'s `argocd` namespace has an OpenBao
  `SecretStore` already** — needed before the new `ExternalSecret` for the
  repo-creds Secret can resolve anything. Not confirmed one way or the
  other during planning; check first, stand one up if it's missing.
- **First `helm dependency build`**: after adding the `dependencies:` block
  to each consumer's `Chart.yaml`, someone needs to run `helm dependency
  build` locally at least once to generate `Chart.lock` and confirm the OCI
  pull actually works end-to-end before committing — don't rely on CI to
  discover a broken dependency reference for the first time.
- **Ongoing (not one-time)**: every time `k8s-ci-rbac`/`k8s-lib-ci-rbac`
  publishes a new chart version, each consumer's pinned
  `dependencies[].version` needs a manual bump + `helm dependency update`
  to actually pick it up — versions don't float, by design (mirrors the
  existing pin-to-a-digest-and-bump-deliberately convention used for
  container images everywhere else in this homelab).
- **Decommissioning `homelab-woodpecker`**: once all three consumers are
  confirmed migrated, revoke/rotate anything Woodpecker-specific it held
  (its own OpenBao secrets, its GitHub App/webhook installation across the
  org — check every repo it was wired into, not just the four touched
  here), then archive the repo.

## Verification

- Confirm ArgoCD (`v3.4.6`) can actually resolve an OCI Helm chart
  dependency at sync time once repo-creds are wired — this is genuinely
  untested infra in this cluster, verify on the first consumer conversion
  before treating the pattern as proven.
- After each consumer conversion: `kubectl -n <namespace> get
  serviceaccount,role,rolebinding` shows the expected `<name>-ci`,
  `<name>-ci-writer`, `<name>-ci-token-issuer` objects, and that repo's own
  `Check` GitHub Actions run passes (specifically the `apply --dry-run=server`
  step, which requires successfully minting the token).
- After the final RBAC cutover: confirm the old shared SA/Role/RoleBinding
  sets are actually pruned once the old `k8s-ci-rbac` Application is
  removed, and that no repo's CI is still pointing at a now-deleted
  identity.
- After each Woodpecker-consumer migration: confirm the new GitHub Actions
  workflow actually produces and pushes a working image (not just "the
  workflow ran green") — check the image lands in the registry and that
  whatever deploys it (ArgoCD Image Updater, etc.) still picks it up
  correctly.
- Before archiving `homelab-woodpecker`: confirm nothing else in the org
  still references it (grep every repo for `woodpecker`, not just the
  three known consumers).
