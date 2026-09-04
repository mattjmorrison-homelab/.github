# TODO: renaming repos

Full plan and rename mapping live in [naming.md](naming.md) — not
duplicating the table here. Summary: drop the redundant `homelab-` prefix
(the GitHub org already says that) and replace it with a destination
prefix instead — `nix-` for NixOS hosts, `k8s-` for anything deployed to
the cluster via ArgoCD, `pi-` for standalone Raspberry Pi tooling,
`graph-`/`ui-` for app source+CI repos (as opposed to their `k8s-`
manifests repo). `.github` is the one exception, kept as-is since GitHub
requires that literal name for the org-profile README feature.

Rename is a real operation per repo (GitHub rename, `repoURL` in every
`homelab-apps` Application manifest, Woodpecker resync for app-source
repos) — go one repo at a time, same cadence as the verify-script
rollout, not a single batch.

Open questions (see naming.md for full detail):

- [ ] Do ArgoCD `Application` object names also adopt the `k8s-` prefix,
      or only the underlying git repos? Application `metadata.name` is a
      real identity (unlike the repo URL), so this decides whether each
      rename is a cheap URL edit or a full Application recreate.
- [ ] Does `homelab-graphql-router` become `k8s-graphql-router` (current
      thinking) or `graph-graphql-router`, given it's a vendored binary
      with no custom source of its own?
- [ ] Sequencing: one repo at a time as work touches it, or a dedicated
      batch pass?

---

# TODO: OpenBao continuous seal-status monitoring

Prompted by a real incident: a pod restart left OpenBao sealed, which broke
unrelated things before anyone noticed. The `PostSync` verify check only
runs at deploy time (and a plain pod restart isn't a deploy), so it
wouldn't have caught this — need real continuous monitoring instead.

Plan: reuse the existing Prometheus + Alertmanager → Discord pipeline
(already has 13 alert rules wired up in `homelab-prometheus`) rather than
building a separate mechanism.

- [x] `homelab-openbao` — enabled the `telemetry` stanza in
      `server.standalone.config` (`unauthenticated_metrics_access = true`
      on the listener + a top-level `telemetry` block), exposing
      `vault_core_unsealed` via Prometheus format at `/v1/sys/metrics`.
      Verified by pulling the real vendored `openbao-helm` chart and
      rendering it with these values — the HCL config block comes out
      exactly as intended.
- [x] `homelab-prometheus` — added the `openbao` scrape job in
      `prometheus-config` (same `static_configs` shape as the existing
      jobs, hitting `homelab-openbao.openbao.svc:8200/v1/sys/metrics`).
- [x] `homelab-prometheus` — added the `OpenBaoSealed` alert rule in
      `prometheus-alerts`: `vault_core_unsealed == 0`, `for: 1m`,
      `severity: critical` (fast-fire, matching `PodFailed`/`NodeNotReady`
      rather than the slower resource-threshold alerts, since a sealed
      vault is immediately impactful). Not committed — ready for review.

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

# TODO: Home Assistant setup, as IaC as possible

`homelab-home-assistant` currently just deploys the container; all actual
HA config (integrations, plugins, automations) lives in mutable state on
the PVC at `/config`, untouched by ArgoCD. Want to push as much of the
setup as possible into source control instead of clicking through the UI.

Needs investigation into how much HA actually supports this:

- `configuration.yaml` (and split-out `automations.yaml`/etc.) can be
  managed as a ConfigMap/git-tracked file for anything that supports
  YAML config.
- HACS (community plugin store) and specific integrations — check which
  ones can be pre-installed/configured via file-based config vs. which
  require the UI (many integrations need an OAuth flow or device pairing
  that can't be scripted).
- Whatever's left un-automatable stays a manual one-time step, documented
  rather than silently left out.

---

# TODO: backup strategy for state that lives outside git

Two things flagged as risky if lost, neither backed up anywhere today —
only the deployment scaffolding is in git, not the actual data:

- [x] OpenBao secret data — superseded by the "IaC for OpenBao's own
      policies/roles" TODO item further down (Terraform-managed policies/roles/
      paths, one secret per path via write-only arguments, real values
      never touched after initial creation). Same goal as the
      `bao kv put -cas=0` Makefile idea originally here, more complete.
- [ ] Home Assistant's `.storage/` — entity/device registries, Lutron
      cert private keys, long-lived auth tokens. Can't go in the public
      `homelab-home-assistant` repo as-is (unlike the plain YAML files
      such as `configuration.yaml`). Look at HA's native backup feature
      (full tarball snapshot) shipped to private storage on a schedule,
      separate from git.

---

# DONE: SteamOS/Arch mini PC metrics

Simpler than the original plan below: not joined to k3s at all, just
`node_exporter` running standalone on the machine, scraped as a static
Prometheus target (`steamos-node-exporter` job in `homelab-prometheus`'s
config, targeting `basementpc.local:9100`/`nevespc.local:9100`). No k3s
agent, no taint, no toleration needed — "metrics-only" achieved by never
making it a cluster node in the first place.

Original plan considered (not what was actually done, kept for context):
join as a k3s agent, taint `dedicated=metrics-only:NoSchedule`, add a
matching toleration to `homelab-node-exporter`'s DaemonSet. Moot now.

---

# TODO: invert the dotfiles/homelab flake relationship

Right now `~/dotfiles` is the top-level flake that owns everything,
including the control-plane host's NixOS config — it pulls in homelab as a
module. Matt wants this flipped: `~/Projects/homelab` becomes the flake
that owns the control-plane NixOS configuration directly, and pulls in
`~/dotfiles` as an input instead (not the other way around).

Why: cluster/control-plane setup doesn't really belong nested inside a
general dotfiles repo, even as a separate module.

Not yet scoped — just captured, nothing investigated yet:

- [ ] Look at the current dotfiles flake's actual inputs/outputs and how
      homelab is wired in today, before planning the flip.
- [ ] Decide what (if anything) homelab still needs from dotfiles after
      the inversion — e.g. shell/user-level home-manager config — versus
      what's genuinely control-plane-only and shouldn't move.
- [ ] This is a structural change to how the control-plane NixOS config is
      assembled — decide whether it goes through the orchestrator's
      TDD pipeline like other `homelab` module changes, or is a one-time
      by-hand restructuring since it's flake plumbing, not a module.

---

# TODO: move this TODO list to a GitHub Project

Matt wants to shift tracking off this markdown file and onto GitHub
Projects, since it spans ~30 repos and a flat file doesn't scale great.

Recommendation discussed: an **org-level Project (v2)**, not repo-level
(repo-level wouldn't capture most items, which span multiple repos). Table
view over Board, with a `Status` field (Backlog/In Progress/Done) — Table
suits these entries better since they carry a lot of narrative context
(why, plan, open questions) that reads better as a linked Issue body than
a terse card title.

Open question: Issues have to live in *some* repo. Repo-specific TODOs
(e.g. the Traefik `/ping` check) go in that repo; cross-cutting ones
(repo renaming, the SteamOS node, the dotfiles/homelab flip, this item
itself) don't have an obvious home — leaning `.github` since it's already
the meta/org-wide doc repo.

- [ ] Set up the actual GitHub Project (a GitHub UI/org action, not
      something doable from here).
- [ ] Once it exists, migrate: one Issue per current TODO.md entry,
      checklist items carried over as task-list checkboxes.

---

# TODO: more central way to route things to the Pi nodes

Prompted by the `graph-hdmi-switch` arm64 saga: getting one service onto a
Pi node currently means hard-coding the same `nodeSelector`/toleration
pair in *two* unrelated places — the deploy-side `k8s-hdmi-switch`
Deployment, and separately in `graph-hdmi-switch`'s `.woodpecker.yml` (on
every step that touches the arm64 image, plus the clone step, since all
steps in a pipeline share one node-pinned workspace volume). Nothing
connects the two; each has to be kept in sync by hand.

- [ ] Look for a way to define "this service belongs on the Pi" once and
      have both the CI (Woodpecker build arch) and the deploy
      (Kubernetes nodeSelector/toleration) pick it up from a single
      source, rather than duplicating the same taint/arch info in both
      `.woodpecker.yml` and the manifests repo.

---

# TODO: update network topology doc for the 2nd Woodpecker build node

`.github/docs/network-topology.md` describes Woodpecker's Kubernetes
backend but predates `graph-hdmi-switch`'s arm64 pinning work — build jobs
can now land on a Pi node (arm64), not just the original node, via
per-step `nodeSelector`/tolerations. Update the doc to reflect that.

---

# TODO: remove duplication of the GraphQL schema across repos

No repo should have a checked-in *copy* of the schema — full stop, not
just "reduce it." Currently at least `ui-hdmi-switch`'s `schema.graphql`
(hand-maintained, feeds GraphQL Codegen) is a hardcoded, manually-kept-in-sync
copy of the backend's schema, same category of problem already fixed for
`graph-router` (which composes live via `subgraph_url` introspection
instead of a checked-in subgraph file).

- [ ] Audit every repo for a checked-in schema copy, not just
      `ui-hdmi-switch`.
- [ ] For `ui-hdmi-switch` specifically: GraphQL Codegen supports loading
      the schema directly from a live URL (confirmed via its own source —
      `UrlLoader` is a registered schema loader), so `codegen.ts`'s
      `schema: 'schema.graphql'` could point at the live router endpoint
      instead, removing the checked-in file entirely.
- [ ] Deploy and confirm current state is actually working first — this
      is follow-up cleanup, not urgent.

---

# TODO: host ui-hdmi-switch from a public Garage bucket instead of nginx

`k8s-garage` (self-hosted S3-compatible object storage, built for
`admin-github`'s OpenTofu state bucket) supports website-hosting mode on public
buckets natively. Once the state-bucket use case is confirmed working,
worth replacing `ui-hdmi-switch`'s current nginx-Deployment hosting with
a public bucket + Ingress pointed at Garage's website endpoint instead —
Garage doesn't terminate TLS itself, so this still goes through Traefik

- the existing wildcard `*.morrisons.site` cert, same as everything else.

- [ ] Not started — confirm the state-bucket setup works first.

---

# TODO: IaC for OpenBao's own policies/roles

Every app's OpenBao Kubernetes auth role and policy has been set up by
hand via OpenBao's own UI/CLI so far — `homelab-openbao` only deploys the
vendored chart, nothing configures policies/roles for the apps that read
from it. This has been a recurring "manual, out-of-band" step for every
app added this session (`homelab-woodpecker`, `k8s-garage`, `admin-github`,
`homelab-prometheus` all needed one).

The actual motivation: disaster recovery. Today, if OpenBao's data were
lost, reconstructing every policy/role/path correctly means
reverse-engineering them from application code and manifests — nothing in
source control lists the complete set. The goal is a rebuild story of
"nuke OpenBao, `tofu apply`, retype the actual secret values, done" —
not tighter security boundaries as the primary driver (that's a
possible side effect, not the point).

**Design, converged:**

- Terraform/OpenTofu's `vault` provider (Vault-API-compatible, works
  against OpenBao) manages policies, Kubernetes auth roles, and which KV
  *paths* exist — structure, not secret values.
- **KV layout changes to one secret per path**: `kv/homelab/<app>/<key>`
  instead of today's `kv/homelab/<app>` holding several keys in one JSON
  document. Necessary because `vault_kv_secret_v2`'s write-only argument
  (`data_json_wo`, Terraform 1.11+) — the mechanism that lets Terraform
  scaffold a path once and never touch its value again on subsequent
  applies — is a whole-document write, not a patch. One key per path
  means adding a new secret is a new resource, never a mutation of a
  document holding other real values.
- Bootstrap credential: the OpenBao root token already saved during
  initial unseal (per `homelab-openbao`'s README) — no new credential to
  mint. Reused for every `tofu apply`, local or eventually from CI.
  Store a copy in Vaultwarden (or wherever) as the durable human-facing
  copy; a separate copy goes into Woodpecker's own native secret store if
  this ever runs from CI, since it can't come from OpenBao itself.
- Migration is incremental, app by app: add new per-key paths alongside
  the existing combined ones (nothing breaks), copy real values over for
  one app, repoint that app's `ExternalSecret`s at the new paths, deploy,
  verify, move to the next app. Old paths/policies stay until every
  consumer is migrated, then get deleted.

Known tradeoffs, accepted: temporary complexity increase (old and new
policies coexist until cleanup actually happens), more individual secret
entries to browse in OpenBao's UI, and a real drift risk if a value gets
rotated at the new path before its app has been repointed to read from
it — mitigated by moving through the migration promptly rather than
leaving apps half-migrated indefinitely.

- [x] `admin-openbao` built — `vault_policy` +
      `vault_kubernetes_auth_backend_role` for every existing SecretStore
      and bootstrap-script role (14 total, captured directly from every
      repo's manifests), `vault_kv_secret_v2` scaffolding every individual
      secret key (26 total) at its new one-path-per-key location via
      write-only arguments. `tofu validate` clean; not applied yet —
      needs the `vault_root_token` secret created in Woodpecker's own
      Settings → Secrets before CI can run it, and the actual app-by-app
      `ExternalSecret` migration to the new paths hasn't started.

---

# TODO: external heartbeat/dead-man's-switch monitoring

Prompted by a real incident: `prometheus-0` was stuck `ContainerCreating`
for ~2 hours (a required Secret volume that didn't exist yet — see
`homelab-prometheus`'s fix), and nothing alerted. Root cause isn't fixable
from inside the cluster's own monitoring stack: Prometheus is both the
thing being watched and the thing doing the watching, so when it's down
there's nothing left to evaluate alert rules or notice its own outage.
Alertmanager only fires on alerts Prometheus sends it — a dead Prometheus
sends nothing.

Needs something outside the cluster entirely. Two shapes discussed:

- **Dead-man's-switch**: an always-firing Prometheus alert (a
  `Watchdog`-style rule, always true) routed to an external service that
  pages when the heartbeat *stops* arriving, rather than when a specific
  condition fires (e.g. Healthchecks.io, or self-hosted equivalent).
  Still depends on Prometheus being the thing sending the heartbeat, so
  it only catches "Prometheus stopped functioning," not "Prometheus is
  up but can't reach the network."
- **External prober**: something running on a standalone, non-cluster Pi
  (`pi1.local`/`pizero.local`) actively hitting Prometheus (or the
  cluster generally) from outside and alerting on its own if it can't
  reach it — doesn't depend on the cluster's own health at all.

`pi-health` already exists as a scaffolded, empty repo — likely the
intended home for whichever approach gets picked (name suggests it was
meant for exactly this, distinct from `graph-health`/`k8s-health`, which
are an in-cluster health-check *service*, not an external watchdog).

- [ ] Not started — approach not chosen yet.

---

# TODO: CI for the `actions-*` repos themselves

`actions-tofu` and `actions-helm` (shared composite actions/reusable
workflows, called from every `admin-*`/`k8s-*` repo's own CI) have no CI
of their own — their actual shell scripts
(`actions-tofu/fetch-credentials/fetch-credentials.sh`,
`actions-tofu/find-check-run/find-check-run.sh`,
`actions-helm/check.sh`) are untested and unlinted, only ever exercised
indirectly by whichever repo happens to call them.

Plan: add `bats` tests and/or `shellcheck` linting for these scripts,
gated on PR the same way as every other repo's `check` job.

- [ ] Not started — approach not chosen yet (bats, shellcheck, both).

---

# TODO: per-repo dependency health-check endpoint + post-deploy verification

Every repo should identify its own runtime external dependencies —
databases, file systems, APIs, whatever it actually talks to at
runtime — and expose a health-check URL that actively "touches" each
one: confirms it can authenticate, confirms it's actually reachable,
not just a static 200. Then the existing post-deploy verification step
(the `PostSync` verify Job pattern already used by several repos, e.g.
`homelab-traefik`'s TCP check, `homelab-openbao`'s seal-status check)
should call that health endpoint as part of deploy verification for
every repo, not just the ones that happen to have one today.

Goal: catch "deployed fine, but can't actually reach its database/API/
whatever" at deploy time, rather than discovering it later via a user
report or an unrelated incident (same category of gap as the OpenBao
seal-status TODO above — deploy-time success isn't the same as
runtime health).

- [ ] Not started — needs: (1) a convention for what the health-check
      endpoint looks like/where it lives per repo's stack, (2) an audit
      of every repo's actual runtime dependencies, (3) rollout, one
      repo at a time like the naming migration and other cross-repo
      rollouts.

---

# TODO: kcov coverage enforcement for bats suites

Bats tests exist across most `k8s-*`/`admin-*`/`actions-*` repos now (via
the TDD pipeline), but nothing measures how much of each script's actual
logic they exercise — a passing suite could still be leaving whole
branches or error paths untouched.

Plan: wire `kcov` into each repo's `check` job to instrument the bats run,
enforcing 100% coverage as a real gate, not just a report.

- [ ] Not started — needs: a shared pattern rather than duplicated
      per-repo setup (likely belongs in `actions-helm`/a new shared
      action, same shape as `actions-tofu`), and a decision on what
      "100%" means for shell scripts with genuinely untestable branches
      (`set -e` traps, defensive code that should be unreachable).

---

# TODO: real per-repo isolation for Zot publish credentials

Discovered while wiring up `k8s-lib-ci-rbac`'s chart-publish workflow:
every repo that publishes to `registry.morrisons.site` (`k8s-zot`) shares
the exact same Zot user (`ci`), configured as a blanket `adminPolicy` with
`read`/`create`/`update`/`delete` on the entire registry. Each repo
storing its own copy of that password at its own `homelab/<app>` OpenBao
path isolates where the credential is *stored*, not what it can actually
*do* — a leaked copy from any one repo's CI can push, overwrite, or delete
any other repo's images or charts, not just its own.

Zot supports real per-repository scoping (a `repositories` block under
`accessControl` binding named policies to specific repo path patterns),
but using it means moving away from one shared `ci` user toward a genuinely
distinct Zot user per publishing repo, each scoped to only its own path
with only the actions it actually needs (`create`+`read` to publish, not
`update`/`delete`).

The actual ask: automate all three pieces together, not just the Zot
policy — `k8s-zot`'s `accessControl` config, the corresponding per-repo
OpenBao secret, and the consuming repo's own GitHub Actions publish job
should all be driven from one place/process, so adding a new publisher
doesn't mean hand-editing three separate systems that can silently drift
out of sync with each other.

- [ ] Not started — needs: a design for what "one place" actually is
      (likely OpenTofu, same direction as the `admin-openbao` IaC-for-
      policies TODO above, extended to cover Zot's own access-control
      config too, not just OpenBao's), and an audit of every repo
      currently publishing to `registry.morrisons.site` under the shared
      `ci` user before picking an order to migrate them in.

# TODO - I think I accidentially renamed a bunch of github repos by running the plan in admin-github

# TODO - create a new doc prefix doc- and have one for like doc-network that shows network topology diagrams, doc-todo for the documented things that need to be done - along with this I'd like to minimize the amount of stuff in the .github repo since it's special

# TODO - I want to know if there is a way that I can set up home assistant (post adding the IaC version of home assistant setup stuff) so that when one of my aqara door sensors is tripped for "door open" that it only alerts if that door isn't closed within 10 seconds or something instead of right now I get an alert when it opens and again when it closes

# TODO - I need to get all of my stuff set up in Home Assistant, currently everything is in Homekit, but not home assistant

# TODO - I want to research running local models - what hardware needs do I have, what 2nd hand hardware can I get, etc

# TODO - I need to work on the Opnsense setup work with maybe ansible? - this could result in re-assinging a lot of IP addresses

# TODO - I need to do a lot of work with the physical cables - it's getting to be a mess

# TODO - i think our docs todo repo can be organized by category - some are physical things (cable management), some are future (planning and shopping for hardware), some are future software/network changes, opnsense, some are just adding to what's already there (home assistant), etc - there are probably other categories

# TODO - should I create a 2nd claude subscription and sign up for a month and swap between the subscripts when one maxes out? That would probably get me by and be $40 / month instead of $100

# TODO - I have a bunch of old macbooks that I can probably install nixos on and add to the cluster
---
# TODO: revisit ArgoCD Application namespace/destination scoping

Found a stale `namespace` field on `k8s-apps`' `k8s-ci-rbac` entry
(pointed at `garage` instead of `monitoring`) while working through CI
RBAC setup. Wasn't actually breaking anything — the chart hardcodes its
own object namespaces — but the field was wrong and nobody would've
noticed. Worth a pass over how these fields get set/scoped generally.
