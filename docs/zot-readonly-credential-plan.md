# Zot read-only credential + host-auth precursor

Status: planned, not started. Written by `homelab-planner`, 2026-08-30.

## Scope note

This is explicitly a **precursor step**, not the full fix. The full fix —
a genuinely distinct Zot user *per publishing repo* — is tracked
separately in `TODO.md` under "real per-repo isolation for Zot publish
credentials." This plan only adds one shared read-only tier (`ci-readonly`)
alongside the existing admin `ci` user, and wires up the machinery to get
a scoped credential onto `~/Projects/homelab`'s host automatically. Do not
conflate the two — a future pass replaces `ci-readonly` with per-repo
users using the same OpenBao/AppRole plumbing this plan builds.

While researching this, the requirements grew to include a related but
separable hardening: the host's existing kubeconfig
(`/etc/rancher/k3s/k3s.yaml`) is world-readable and full-admin today. This
plan also locks that down, because the original "can the host just use
`kubectl get secret`?" question turned out to be a bad idea for security
reasons (see Design) — closing that path off is what pushed the actual
Zot-credential-fetch mechanism toward OpenBao AppRole. Both changes are
bundled here because the AppRole design specifically depends on the
kubeconfig no longer being able to read Secrets.

## 1. Repos touched

| Repo (before) | Repo (after) | What changes |
| --- | --- | --- |
| `k8s-zot` | *(no rename)* | Add `ci-readonly` user: htpasswd entry + whole-registry read-only `accessControl` policy. First-ever CI workflow for this repo (none exists today). |
| `admin-openbao` | *(no rename)* | New `approle.tf`: enable AppRole auth backend, one policy + role scoped to exactly one new KV path. New blank KV path in `secrets.tf`/`locals.tf`. |
| `k8s-host-rbac` | *(new repo)* | ServiceAccount + ClusterRoleBinding (to built-in `view`) + a static long-lived SA-token Secret, for the host's new read-only kubeconfig. |
| `homelab` | *(no rename — see Assumptions)* | Tighten `k3s-control-plane.nix`'s kubeconfig mode to `0600`. New `zot-credential-sync.nix` module: dedicated system user + systemd timer that logs into OpenBao via AppRole and refreshes the Zot pull credential. |
| `dotfiles` | *(no rename)* | `hosts/imac/default.nix` imports the new `zot-credential-sync` module. |
| `admin-github` | *(no rename)* | Add `k8s-host-rbac` to `branch_protection.tf`'s `repos` set — this is how repos actually get created in this homelab (`tofu apply`), never a manual GitHub action; see `admin-github/README.md`'s "Adding a new repo" section. |
| `k8s-apps` | *(no rename)* | New `Application` entry registering `k8s-host-rbac`, same pattern as every other consumer this session (e.g. the `k8s-homepage`/`k8s-argocd` Application additions) — a normal orchestrator-dispatched PR, not a manual step either. |

`~/Projects/homelab`'s own rename to `nix-control-plane` (per `naming.md`'s
opportunistic-rename policy, since this plan touches it) is **deferred**,
not executed here — see Assumptions. All OpenBao paths below use the
literal current repo name `homelab`.

## 2. Context

`k8s-zot` has exactly one user, `ci`, configured as a blanket
`adminPolicy` with read/create/update/delete across the whole registry
(`k8s-zot/manifests/templates/configmap.yaml`). Every consumer — every
publish workflow, every pull (`homelab-woodpecker`'s
`external-secret-zot-pull.yaml`, every `k8s-*` chart's image pull
secret) — uses this same admin credential. There is no read-only tier at
all today; "pull" secrets across the homelab are just the admin password
re-labeled.

This came to a head because a developer workstation
(`~/Projects/homelab`, the NixOS host that is also the k3s control-plane
node) needs to run `helm dependency build`/`helm template` against
`oci://registry.morrisons.site/charts/...` locally, for the homelab
team's TDD pipeline (see `homelab-team.md`) — and the only credential
available for that is full-admin.

Every existing credential-fetch pattern in this homelab
(`actions-tofu/fetch-credentials`, every publish workflow, e.g.
`k8s-lib-ci-rbac/.github/workflows/publish.yml`) authenticates to OpenBao
via Kubernetes auth — a pod presents its own ServiceAccount JWT. The host
is not a pod and has no JWT to present, so that pattern doesn't transfer.

The obvious-looking alternative — "the host already has a cluster-admin
kubeconfig (`/etc/rancher/k3s/k3s.yaml`, mode `0644`, confirmed readable
by any local user via this repo's own NixOS test), so just
`kubectl get secret` the credential out of the cluster instead of talking
to OpenBao at all" — was explicitly ruled out. Reading a live `Secret`
object is exactly the pivot this plan needs to close off (a
read-only-looking credential that can walk from "read the cluster" to
"read anything OpenBao ever pushed into it via ExternalSecret"). So the
host needs two genuinely separate, non-overlapping privilege boundaries:
a scoped read-only *Kubernetes* identity that structurally cannot read
Secrets, and a separate, narrowly-scoped *OpenBao* identity — reachable
only via a new AppRole auth path, with its own bootstrap credential that
does not itself come from the (now Secret-blind) kubeconfig.

## 3. Design

### 3.1 `k8s-zot` — the `ci-readonly` user

`manifests/templates/configmap.yaml`'s `accessControl` block gains a
`repositories` policy alongside the untouched `adminPolicy`:

```json
{
  "storage": { "rootDirectory": "/var/lib/registry" },
  "http": {
    "address": "0.0.0.0",
    "port": "5000",
    "auth": { "htpasswd": { "path": "/secret/htpasswd" } },
    "accessControl": {
      "repositories": {
        "**": {
          "policies": [
            {
              "users": ["ci-readonly"],
              "actions": ["read"]
            }
          ],
          "defaultPolicy": []
        }
      },
      "adminPolicy": {
        "users": ["ci"],
        "actions": ["read", "create", "update", "delete"]
      }
    }
  },
  ...
}
```

`"**"` matches every repository path (both `charts/*` and the plain
image paths like `graph-hdmi-switch`, `ui-hdmi-switch`, `graph-router`),
per the confirmed answer: whole-registry read, not `charts/**` only.
`defaultPolicy: []` keeps unauthenticated/other-user access at zero —
only `ci-readonly` gets the new grant, `ci` keeps full admin via the
unchanged `adminPolicy`. **This exact schema (`repositories` +
`policies` + `defaultPolicy`) is asserted from Zot's documented config
format, not verified against a live instance** — this repo has zero
existing precedent for anything beyond `adminPolicy` — see Assumptions.

The htpasswd file itself (bcrypt hashes, mounted from the `zot-htpasswd`
Secret, sourced via `ExternalSecret` from OpenBao path `homelab/zot`
property `HTPASSWD`) is a single manually-maintained blob with no
generation tooling in this repo. Adding `ci-readonly` means manually
regenerating that blob to contain *two* bcrypt lines instead of one:

```bash
htpasswd -nB ci                >> new-htpasswd   # existing line, unchanged hash
htpasswd -nbB ci-readonly '<new password>' >> new-htpasswd
```

then pasting the resulting two-line file content back into OpenBao at
the same existing path (`kv/homelab/zot`, property `HTPASSWD`) — this is
a manual step, detailed in section 9.

### 3.2 `admin-openbao` — new `approle.tf`

No `vault_auth_backend` resource exists in this repo today — Kubernetes
auth is assumed already mounted, never Terraform-managed here. AppRole
needs its own backend mount, its own policy, and its own role, kept
entirely separate from `locals.roles` (which is Kubernetes-auth-shaped:
`namespace`/`service_account` keys that don't apply to a host identity).
New file:

```hcl
# approle.tf
# AppRole auth for the nix-control-plane host -- the one non-pod identity
# in this homelab. Deliberately its own file/locals, not folded into
# locals.roles (Kubernetes-auth-role-shaped, doesn't apply here).
#
# Scoped to exactly one KV path -- this is a narrower grant than every
# other role in this repo (which grant `path` + `path/*`), because this
# credential exists for exactly one purpose and nothing else should ever
# widen it silently.
resource "vault_auth_backend" "approle" {
  type = "approle"
}

resource "vault_policy" "nix_control_plane_zot_readonly" {
  name   = "nix-control-plane-zot-readonly"
  policy = <<-EOT
    path "kv/data/homelab/homelab/zot-readonly-password" {
      capabilities = ["read"]
    }
  EOT
}

resource "vault_approle_auth_backend_role" "nix_control_plane_zot_readonly" {
  backend        = vault_auth_backend.approle.path
  role_name      = "nix-control-plane-zot-readonly"
  token_policies = [vault_policy.nix_control_plane_zot_readonly.name]
  token_ttl      = 300
  token_max_ttl  = 600
  # secret_id never expires and is reusable across every scheduled sync run
  # (this is a long-lived machine identity, not a one-shot CI credential) --
  # rotation is a manual re-issue if the host's copy is ever suspected
  # compromised, same shape as every other bootstrap credential in this repo.
  secret_id_ttl      = 0
  secret_id_num_uses = 0
}

output "nix_control_plane_zot_readonly_role_id" {
  value       = vault_approle_auth_backend_role.nix_control_plane_zot_readonly.role_id
  description = "Vault-generated role_id -- not sensitive on its own (no login without the separately-issued secret_id), safe to bake into homelab's Nix module as a plain literal."
}
```

`secret_id` is deliberately **not** a Terraform resource — the provider
would store it in state, which is exactly what this repo's write-only-
argument pattern (`secrets.tf`) exists to avoid for real secret values.
It's generated once, out-of-band, by a human (section 9).

New blank KV path in `secrets.tf`/`locals.tf`'s `secrets` map, following
the "app prefix = the consuming repo's exact current name" rule (matches
how `zot-ci-password` is scoped per-consumer today, e.g. under
`k8s-hdmi-switch`/`homelab-woodpecker`, never under `zot`/`k8s-zot`
itself):

```hcl
homelab = ["zot-readonly-password"]
```

giving `kv/homelab/homelab/zot-readonly-password` — the doubled
`homelab/homelab` segment looks odd but is correct per this repo's own
stated convention today, since `~/Projects/homelab` hasn't been renamed
yet (deferred, see Assumptions). It becomes
`kv/homelab/nix-control-plane/zot-readonly-password` whenever that
rename happens, as a follow-up.

New test, matching `tests/locals_role_argocd_notifications_test.tftest.hcl`'s
convention:

```hcl
# tests/approle_nix_control_plane_zot_readonly_test.tftest.hcl
run "approle_role_scoped_to_single_zot_readonly_path" {
  command = plan

  assert {
    condition     = strcontains(vault_policy.nix_control_plane_zot_readonly.policy, "kv/data/homelab/homelab/zot-readonly-password")
    error_message = "policy must grant the zot-readonly-password path"
  }

  assert {
    condition     = !strcontains(vault_policy.nix_control_plane_zot_readonly.policy, "/*")
    error_message = "policy must NOT use a wildcard -- this credential is scoped to exactly one secret, not a whole app's KV tree"
  }
}
```

### 3.3 `k8s-host-rbac` (new repo)

A normal deployed chart (`k8s-` prefix — deploys to the cluster via
ArgoCD), not a library chart. Own namespace `host-rbac`, matching the
one-namespace-per-Application convention that avoids the cross-namespace
sync issues `k8s-lib-ci-rbac`'s README describes.

- `manifests/Chart.yaml`
- `manifests/templates/namespace.yaml` — namespace `host-rbac`
- `manifests/templates/serviceaccount.yaml` — ServiceAccount
  `zot-credential-sync` in `host-rbac` (same name as the Nix-side system
  user, deliberately — same conceptual actor, two different platforms)
- `manifests/templates/clusterrolebinding.yaml` — binds that
  ServiceAccount to the **built-in** `ClusterRole/view`:

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: zot-credential-sync-view
  subjects:
    - kind: ServiceAccount
      name: zot-credential-sync
      namespace: host-rbac
  roleRef:
    kind: ClusterRole
    name: view
    apiGroup: rbac.authorization.k8s.io
  ```

  No custom `ClusterRole` needed — `view` is a Kubernetes built-in that
  already excludes `secrets` (and RBAC objects) specifically to prevent
  this exact escalation, confirmed via Kubernetes' own RBAC docs. This is
  a better foundation than a hand-rolled rule set: it's already audited,
  and its exclusions are load-bearing for requirement #2 (kubeconfig
  must not be able to read Secrets at all).

- `manifests/templates/serviceaccount-token-secret.yaml` — a static,
  non-expiring token so the resulting kubeconfig doesn't need periodic
  re-minting:

  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: zot-credential-sync-token
    namespace: host-rbac
    annotations:
      kubernetes.io/service-account-name: zot-credential-sync
  type: kubernetes.io/service-account-token
  ```

  Kubernetes' controller auto-populates `data.token`/`data.ca.crt` once
  this Secret exists (legacy long-lived SA-token mechanism, still
  supported for exactly this use case — **not verified against this
  cluster's exact k3s version**, flagged in Assumptions).

- PostSync verify Job, matching `k8s-zot`/`k8s-openbao`'s convention —
  proves the exclusion holds rather than just trusting `view`'s docs:

  ```sh
  # verify.sh
  set -eu
  CAN_READ_PODS=$(kubectl auth can-i get pods --as=system:serviceaccount:host-rbac:zot-credential-sync)
  CAN_READ_SECRETS=$(kubectl auth can-i get secrets --as=system:serviceaccount:host-rbac:zot-credential-sync)
  [ "$CAN_READ_PODS" = "yes" ] || { echo "FAIL: expected read access to pods"; exit 1; }
  [ "$CAN_READ_SECRETS" = "no" ] || { echo "FAIL: zot-credential-sync can read Secrets -- exclusion broken"; exit 1; }
  echo "PASS: read-only, and Secrets are correctly excluded"
  ```

### 3.4 `homelab` — kubeconfig tightening + new sync module

`modules/k3s-control-plane.nix`: drop the loosening flag entirely
(k3s's own built-in default is already `0600`):

```nix
# before
extraFlags = [ "--write-kubeconfig-mode=0644" ];
# after
extraFlags = [ "--write-kubeconfig-mode=0600" ];
```

(explicit `0600` rather than just deleting the line, so the intent is
self-documenting rather than relying on a reader knowing k3s's default).
Remove the now-stale comment above it about `0644` letting non-root
users run kubectl — that's no longer true by design.

New file `modules/zot-credential-sync.nix`, exposed via `flake.nix` as
`nixosModules.zot-credential-sync` (alongside the existing
`k3s-control-plane` output):

```nix
{ pkgs, ... }: {
  users.groups.zot-credential-sync = {};
  users.users.zot-credential-sync = {
    isSystemUser = true;
    group = "zot-credential-sync";
    home = "/var/lib/zot-credential-sync";
    createHome = true;
  };

  systemd.services.zot-credential-sync = {
    description = "Fetch the Zot read-only pull credential from OpenBao and refresh helm's registry login";
    serviceConfig = {
      Type = "oneshot";
      User = "zot-credential-sync";
      Group = "zot-credential-sync";
      LoadCredential = "secret-id:/var/lib/zot-credential-sync/secret-id";
      Environment = "HOME=/var/lib/zot-credential-sync";
    };
    script = ''
      set -euo pipefail
      VAULT_ADDR=https://openbao.morrisons.site
      ROLE_ID=<literal role_id from admin-openbao's tofu output>
      SECRET_ID=$(cat "$CREDENTIALS_DIRECTORY/secret-id")

      CLIENT_TOKEN=$(${pkgs.curl}/bin/curl -sf -X POST "$VAULT_ADDR/v1/auth/approle/login" \
        -d "{\"role_id\":\"$ROLE_ID\",\"secret_id\":\"$SECRET_ID\"}" \
        | ${pkgs.jq}/bin/jq -r '.auth.client_token')

      PASSWORD=$(${pkgs.curl}/bin/curl -sf -H "X-Vault-Token: $CLIENT_TOKEN" \
        "$VAULT_ADDR/v1/kv/data/homelab/homelab/zot-readonly-password" \
        | ${pkgs.jq}/bin/jq -r '.data.data.value')

      echo "$PASSWORD" | ${pkgs.kubernetes-helm}/bin/helm registry login registry.morrisons.site \
        -u ci-readonly --password-stdin
    '';
  };

  systemd.timers.zot-credential-sync = {
    wantedBy = [ "timers.target" ];
    timerConfig = {
      OnBootSec = "2m";
      OnUnitActiveSec = "1h"; # matches ExternalSecret refreshInterval elsewhere
    };
  };
}
```

`/var/lib/zot-credential-sync/secret-id` (the AppRole bootstrap
credential) and the new read-only kubeconfig file are **not created by
this module** — both are placed manually, owned
`zot-credential-sync:zot-credential-sync` mode `0400`, directly at that
ownership (never written as root then chowned). See section 9 for exact
commands. `helm`'s registry config lands in this user's own
`$HOME/.config/helm/registry/config.json`, not root's or the interactive
user's.

New NixOS test `tests/zot-credential-sync.nix` (VM test, matching the
existing `pkgs.testers.runNixOSTest` convention): asserts the dedicated
user/group exist, the timer and service units are defined, `helm` is on
`PATH` inside the unit. `tests/k3s-control-plane.nix`'s existing subtest
inverts:

```python
with subtest("kubeconfig is NOT readable by non-root"):
    machine.fail("su -s /bin/sh nobody -c 'cat /etc/rancher/k3s/k3s.yaml'")
```

### 3.5 `dotfiles`

`hosts/imac/default.nix`'s `imports` gains
`inputs.homelab.nixosModules.zot-credential-sync`, alongside the existing
`inputs.homelab.nixosModules.k3s-control-plane`. After `homelab`'s PR
merges, `dotfiles/flake.lock` needs `nix flake lock --update-input
homelab` to pick up the new module — a manual step (section 9), since
lockfile updates aren't something CI does unattended in this repo today.

## 4. New infrastructure required

- **OpenBao AppRole auth backend** — no precedent anywhere in this
  homelab; every existing OpenBao consumer uses Kubernetes auth. This is
  the first non-pod OpenBao identity ever created here. Flagged as
  genuinely new and untested until it's actually exercised end-to-end.
- **A manually-placed, out-of-git bootstrap secret on a NixOS host**
  (`/var/lib/zot-credential-sync/secret-id`) — no `sops-nix`/`agenix`
  precedent exists in `~/dotfiles` or `~/Projects/homelab` today. This
  plan uses a plain manually-placed file (matching the *spirit* of every
  other bootstrap credential here — the OpenBao root token, SSH keys —
  but this is the first time that pattern is expressed as a NixOS
  `LoadCredential=` reference rather than a CI secret store entry).
- **A static, non-expiring Kubernetes ServiceAccount-token Secret** used
  to mint a durable kubeconfig — this legacy mechanism is assumed still
  supported by this cluster's k3s version; not verified live.
- **First repo (`k8s-host-rbac`) whose sole purpose is host-level RBAC**,
  not a running workload — no exact precedent, though it's structurally
  similar to `k8s-lib-ci-rbac`'s RBAC-only shape.

## 5. Automated tests (PR-time)

- **`k8s-zot`**: has **zero CI today** — no `.github/workflows` at all.
  This plan adds the first one, matching the shared-check convention
  used by every other `k8s-*` chart repo (`azure/setup-helm` + either
  `actions-helm`'s dry-run check or, if this chart can't dry-run cleanly
  against a real registry secret in CI, `helm lint` alone — same
  reasoning `k8s-lib-ci-rbac`'s `check.yml` gives for using lint instead
  of the shared dry-run action). Plus new bats tests (this repo has no
  `test/` directory yet either) asserting `helm template`'s rendered
  `configmap.yaml` contains the new `repositories`/`ci-readonly`/`read`
  block, that `adminPolicy` for `ci` is unchanged (regression guard), and
  that no other user gets implicit write via `defaultPolicy`.
- **`admin-openbao`**: existing `tofu plan`/test CI (`.github/workflows/tofu.yml`,
  `actions-tofu`'s shared workflow) already gates on `.tftest.hcl` files —
  just needs the new test file from 3.2, no new CI wiring.
- **`k8s-host-rbac`**: new repo, so new `check.yml` from scratch —
  `helm lint` at minimum; the RBAC-generation-circularity note in
  `k8s-lib-ci-rbac`'s own `check.yml` may or may not apply here (this
  repo's own ClusterRoleBinding doesn't generate the dry-run identity
  other repos depend on), so a real dry-run check via `actions-helm` is
  probably viable — orchestrator should verify this rather than assume.
- **`homelab`**: existing `make check` (`nix build .#checks.x86_64-linux.k3s-control-plane`)
  already gates `k3s-control-plane.nix`'s test; the new
  `zot-credential-sync.nix` test needs a matching `checks.x86_64-linux.zot-credential-sync`
  entry in `flake.nix` and a `make check` target that runs both.
- **`dotfiles`**: no existing automated check target found for NixOS host
  configs during this research — orchestrator should confirm on entry
  whether one exists (e.g. `nix flake check`) before assuming none does.
- **`admin-github`**: existing `tofu plan`/test CI already gates
  `branch_protection.tf` changes — no new test file needed, adding one
  name to the existing `repos` set is covered by the existing plan/apply
  gate.
- **`k8s-apps`**: matches this session's established convention for
  Application-entry additions (a bats test asserting `helm template`
  renders the new `k8s-host-rbac` Application with the right
  `repoURL`/`namespace`/`path`).

## 6. Post-deploy verification

`k8s-host-rbac` gets a `PostSync` verify Job per section 3.3 — this is
the right place for it (a live, in-cluster assertion that RBAC actually
behaves as intended, not just that the manifest rendered correctly).
`k8s-zot`'s *existing* verify Job (`registry.morrisons.site/` returns
200) is unaffected and keeps passing regardless of the new user. No new
verify Job is proposed for `admin-openbao` (its own CI already plans/
applies; there's no live resource to PostSync-verify the way a Kubernetes
Deployment has). `homelab`/`dotfiles` have no PostSync convention (NixOS
host config isn't ArgoCD-managed) — verification there is the NixOS test
suite plus the manual verification steps in section 13.

## 7. Notifications/metrics

None needed. This isn't a running service with its own failure mode
worth alerting on independently — if the AppRole login or Zot pull ever
fails, the visible symptom is the *next* thing that depends on it
failing loudly (a `helm dependency build` failing locally, or
`k8s-host-rbac`'s verify Job failing at PostSync, which ArgoCD already
surfaces). If this sync mechanism gets reused for something
higher-stakes later (e.g. an automated CI step depends on it silently
succeeding), that would be the time to add a
`systemd.services.zot-credential-sync.onFailure` hook or similar — not
warranted for a single developer-workstation convenience credential.

## 8. Documentation updates

- `k8s-zot/README.md` — currently just the bare hostname; needs the new
  `ci-readonly` user documented (whole-registry read-only, where its
  password lives in OpenBao, how it differs from `ci`).
- `admin-openbao/README.md` — its "Current state vs. this repo's paths"
  and "Adding a new secret or role" sections should mention the AppRole
  backend now exists alongside Kubernetes auth, since the README's
  framing today ("every SecretStore and bootstrap script's
  Kubernetes-auth role") is no longer completely accurate.
- `k8s-host-rbac/README.md` — new repo, needs one from scratch: why it
  exists, what `view` grants/excludes, how the static token Secret works.
- `homelab/README.md` — document the new module and the two manual
  bootstrap files it depends on (`secret-id`, the read-only kubeconfig).
- `.github/docs/TODO.md` — the "real per-repo isolation for Zot publish
  credentials" entry should get a note that this precursor (shared
  `ci-readonly`, AppRole plumbing) is done, so the follow-up work is
  scoped correctly against what already exists.
- `.github/docs/naming.md` — add `k8s-host-rbac` to the rename-mapping
  table's "not yet created, planned" section is moot once created; just
  make sure it's listed as an existing repo, not still "planned."

## 9. Manual steps

1. **k8s-zot htpasswd regeneration**: run `htpasswd -nB ci` (recover/reuse
   the existing hash, or regenerate if the plaintext isn't at hand) and
   `htpasswd -nbB ci-readonly '<new password>'` locally, concatenate into
   a two-line file, and write it into OpenBao at the existing path
   `kv/homelab/zot`, property `HTPASSWD` (overwrite the current
   single-line value with the two-line one).
2. **Same `ci-readonly` password, second location**: write the identical
   plaintext password (not the hash) into OpenBao at
   `kv/homelab/homelab/zot-readonly-password` (created blank by
   `admin-openbao`'s `tofu apply`) — this is the value the host's AppRole
   role reads.
3. **AppRole secret_id generation**: after `admin-openbao`'s `tofu apply`
   creates the AppRole backend/role, run
   `bao write -f auth/approle/role/nix-control-plane-zot-readonly/secret-id`
   against OpenBao directly (not via Terraform) and capture the
   `secret_id` value.
4. **Place the secret_id on the host**: on `~/Projects/homelab`'s host,
   `install -o zot-credential-sync -g zot-credential-sync -m 0400
   <(echo -n '<secret_id>') /var/lib/zot-credential-sync/secret-id` —
   written directly with that ownership, never as root then chowned.
5. **Mint the read-only kubeconfig's token**: using the *old* admin
   kubeconfig one last time (before/while this same change tightens it),
   after `k8s-host-rbac` deploys, run
   `kubectl get secret -n host-rbac zot-credential-sync-token -o
   jsonpath='{.data.token}' | base64 -d` and build a kubeconfig file
   pointing at the cluster's API server with that token; place it at
   (proposed) `/var/lib/zot-credential-sync/readonly-kubeconfig`, same
   ownership/mode as step 4.
6. **`admin-openbao`'s `role_id`**: after `tofu apply`, run
   `tofu output -raw nix_control_plane_zot_readonly_role_id` and paste
   the literal value into `homelab`'s `zot-credential-sync.nix` (it's
   not sensitive — see 3.2 — so this can be a plain committed literal).
7. **`dotfiles/flake.lock` update**: after `homelab`'s PR merges, run
   `nix flake lock --update-input homelab` in `~/dotfiles` and commit the
   updated lockfile.

(Repo creation and Application registration were previously listed here as
manual GitHub steps — corrected: both are normal orchestrator-dispatched
PRs, see the `admin-github`/`k8s-apps` rows in section 1 and steps 0/3.5
in the Rollout order below.)

## 10. Security/secrets note

`k8s-zot`, `admin-openbao`, `k8s-host-rbac`, and `homelab` are all public.
No real secret value is proposed to be committed anywhere in this plan:
the `ci-readonly` plaintext password, the AppRole `secret_id`, and the
minted SA token all flow through OpenBao/Kubernetes Secrets or a manually
-placed out-of-git host file, never through a manifest or Nix literal.
The one thing that *is* proposed as a plain committed literal is the
AppRole `role_id` — deliberately, because Vault/OpenBao treat it as
non-sensitive by design (a login attempt with only a valid `role_id` and
no `secret_id` fails), matching how this repo already treats other
non-secret identifiers as plain config. The new `k8s-host-rbac` static
SA-token Secret is itself a real credential living in the cluster as a
Kubernetes `Secret` object (that's inherent to how durable SA tokens
work) — it is never rendered into a manifest with a real value; the
token field is populated by the Kubernetes controller after the Secret
is created empty of the token itself, same non-committal pattern
`external-secret.yaml` templates already use elsewhere.

## 11. Rollout order

One repo at a time, per this homelab's own outage-avoidance rule — and
here specifically ordered least-risky/purely-additive first, with the
one behavior-changing step (`k3s-control-plane.nix`'s kubeconfig mode)
deliberately last and isolated:

0. **`admin-github`** — add `k8s-host-rbac` to `branch_protection.tf`'s
   `repos` set, merge, `tofu apply`. Real prerequisite: `orchestrator`
   can't be pointed at a repo that doesn't exist yet. Purely additive to
   this repo's own state.
1. **`admin-openbao`** — proof-of-concept / lowest-risk first move.
   Purely additive: new backend, new policy, new blank secret path.
   Nothing existing is touched or re-read differently.
2. **`k8s-zot`** — additive: new user alongside the untouched admin one.
   Verify existing `ci` logins and the existing verify Job still pass
   before moving on.
3. **`k8s-host-rbac`** — deploys fresh resources only, into the repo step
   0 just created; zero risk to anything already running.
3.5. **`k8s-apps`** — register `k8s-host-rbac` as a new `Application`,
   same pattern as every other Application addition this session. Can
   dispatch as soon as step 3's manifests exist; doesn't block step 4.
4. **`homelab`**, in two parts within the same repo, sequenced
   internally: (a) add `zot-credential-sync.nix` first and confirm the
   AppRole login + Zot pull actually works end-to-end while the *old*
   admin kubeconfig is still world-readable as a safety net; only then
   (b) tighten `k3s-control-plane.nix`'s kubeconfig mode to `0600`, since
   that's the one change that can lock out standing kubectl access if
   something upstream of it isn't actually working yet.
5. **`dotfiles`** — last, gated on `homelab`'s PR merging first (the new
   module doesn't exist as a flake output until then).

## 12. Assumptions/unknowns

- **`~/Projects/homelab`'s rename to `nix-control-plane` is deferred**,
  not decided as "no" — `naming.md`'s own opportunistic-rename policy
  arguably applies here since this plan touches that repo, but folding a
  GitHub rename + flake-input URL change + OpenBao path rename into an
  already-large precursor step risks exactly the scope creep this task
  was explicit about avoiding. Flagging this as a live tension rather
  than a settled decision.
- **Zot's `repositories`/`policies`/`defaultPolicy` config schema** (3.1)
  is taken from Zot's documented config format; this repo has no
  existing example of anything beyond `adminPolicy` to pattern-match
  against, and it hasn't been verified against a running Zot instance.
- **Legacy static ServiceAccount-token Secrets still auto-populate**
  (`data.token`) on this cluster's exact k3s/Kubernetes version — assumed
  based on general Kubernetes behavior, not confirmed against this
  cluster.
- **`view`'s aggregated coverage of CRDs** (ArgoCD `Application`,
  `Certificate`, `ExternalSecret`, etc.) depends on whether each CRD's
  own chart labeled its ClusterRole `aggregate-to-view: "true"` — not
  checked per-CRD here. The read-only kubeconfig may see less than the
  old admin one for anything CRD-shaped, even though it sees strictly
  more for core resources (pods, deployments, services, nodes).
  Confirmed via Kubernetes' documented behavior generally — not verified
  against every CRD-installing chart in this cluster specifically.
- **"Root shouldn't have standing access" is achieved in the sense that
  no root-run process incidentally touches this material** (nothing runs
  `User = root` for it) — not as an absolute guarantee, since root's
  DAC-bypass on Linux means a root user who deliberately goes looking can
  always read any file regardless of ownership.
- **`k8s-host-rbac`'s CI dry-run-vs-lint choice** (section 5) — flagged
  for orchestrator to actually check rather than assume, since this
  repo's relationship to the shared `actions-helm` dry-run action is not
  identical to `k8s-lib-ci-rbac`'s (which has a specific, stated
  circularity reason for using lint instead).
- **AppRole `secret_id_ttl = 0` (never expires)** is a deliberate choice
  favoring operational simplicity over forced rotation, matching this
  homelab's existing bootstrap-credential conventions elsewhere — not
  something the user was asked to confirm explicitly; flagging here in
  case a periodic-rotation policy is actually preferred.

## 13. Verification

- `helm registry login registry.morrisons.site -u ci-readonly
  --password-stdin` succeeds with the OpenBao-stored password, and a
  subsequent write attempt with that same user (e.g. `helm push` against
  any repo path) fails with a permission error from Zot.
- `kubectl auth can-i get secrets --as=system:serviceaccount:host-rbac:zot-credential-sync`
  returns `no`; `kubectl auth can-i get pods` (same identity) returns
  `yes` — both checkable directly, and asserted by `k8s-host-rbac`'s
  PostSync verify Job.
- On the host: `su -s /bin/sh nobody -c 'cat /etc/rancher/k3s/k3s.yaml'`
  fails with permission denied (inverted from today's behavior).
- On the host: `systemctl status zot-credential-sync.timer` shows
  `active (waiting)`; `journalctl -u zot-credential-sync.service` shows a
  successful run within the last hour; `helm dependency build` against
  `oci://registry.morrisons.site/charts/...` succeeds using the
  credential the timer refreshed, without any admin password involved.
- `tofu plan` in `admin-openbao` against the merged `approle.tf` shows no
  diff (confirms the AppRole backend/role/policy match what's live).
- `grep -r "0644" ~/Projects/homelab/modules/k3s-control-plane.nix`
  returns nothing.
