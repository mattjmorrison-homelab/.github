# GitHub org access control: proposal

Three identities requested for `mattjmorrison-homelab`:

1. **Org owner** — full access, everything. This already exists (your own
   account). No new work.
2. **PR bot** — can pull any repo, can open a PR on any repo, cannot push
   to `main` on any repo, no other org access. This is the identity that
   would push branches and open PRs on your behalf (e.g. Claude Code,
   once controls like this exist).
3. **Repo-admin bot** — can create repos, rename repos, and manage repo
   settings (branch protection, status checks), no other access. This is
   the identity `admin-github`'s CI already uses.

## The one structural problem

GitHub has no per-account "can create repos, nothing else" permission.
Repository creation is gated by a single org-wide toggle (**Settings →
Member privileges → Repository creation**) that applies to *all* members
equally — there's no way to allow it for one specific member while
blocking it for others. An Owner can always create repos regardless of
that toggle; a Member can only create repos if the toggle allows it for
everyone.

This means a plain Member account can't be scoped to "create repos" alone
without either being an Owner (too broad — also grants billing, SSO,
member management, deleting the org) or the org allowing *all* members to
create repos (also too broad, if there are ever other members).

**Resolution: use a GitHub App, not a user account, for the repo-admin
identity.** GitHub Apps have their own fine-grained permission model,
independent of the member-privilege toggle — an App can be granted the
**Administration** organization permission (which includes repo creation)
without needing Owner and without touching the org-wide member setting at
all. This is also just a better fit generally: Apps authenticate with
short-lived, auto-rotating installation tokens instead of a long-lived PAT
sitting in OpenBao, don't consume a seat, don't need a fake email/2FA
setup, and show up as a distinct bot identity in the audit log.

The PR bot doesn't hit this problem — pull/push-branch/open-PR are all
ordinary repository permissions a Member can hold. It could be a GitHub
App too, but a plain user account + fine-grained PAT is simpler to use
from an interactive tool like Claude Code (a PAT drops straight into
`GH_TOKEN`/`gh auth login --with-token`; an App requires generating
installation tokens, more natural for CI than for ad-hoc local use).
Recommendation: **user account for the PR bot, GitHub App for the
repo-admin bot.**

## 1. PR bot

**Type:** dedicated GitHub user account (e.g. `mattjmorrison-bot` or
similar — needs its own email address; enable 2FA on it).

**Org membership:** Member, not Owner.

**Repository access:** Write, on every repo in the org. Write is the
minimum role that allows pushing a branch and opening a PR — there's no
GitHub role between "can't push branches at all" (Read) and "can push any
branch" (Write). **"Cannot push to `main`" is not expressed by the role at
all — it's enforced entirely by `admin-github`'s branch protection**
(`enforce_admins = true`, PR required, no force pushes), which already
applies to every repo regardless of who's pushing. Write + that protection
together give exactly the intended behavior; Write alone would not.

Grant Write via a Team (e.g. `automation-write`) containing this account,
rather than per-repo collaborator invites — 38 repos and growing.
Recommendation: manage the account's org membership, team membership, and
the team's repo access all in `admin-github` itself (`github_membership`,
`github_team_membership`, `github_team_repository`, the last looping over
the same `local.repos` set already used for `github_repository`/
`github_branch_protection`), so all of it stays in sync automatically as
repos are added — no separate manual step to add this user to the org.

**Repo creation:** blocked structurally — leave **Member privileges →
Repository creation** off (or Owners-only). This account being a plain
Member with no repo-creation privilege is what satisfies "no other
permission to any other parts of the org."

**Token:** fine-grained PAT, resource owner = `mattjmorrison-homelab`, all
repositories, permissions: **Contents: Read and write**, **Pull requests:
Read and write**, **Metadata: Read**. No `Administration`. This is a
second, narrower layer on top of the account's own Write role — even if
the token leaked its own scope wouldn't reach repo settings.

## 2. Repo-admin bot

**Type:** GitHub App, installed on `mattjmorrison-homelab`.

**Organization permissions:** `Administration: Read and write` (covers
creating and renaming repos), `Members: Read and write` (covers org/team
membership — needed so `admin-github` can also manage the PR bot's org
membership and team assignment, via `github_membership` /
`github_team_membership` / `github_team_repository`, rather than that
being a separate manual step).

**Repository permissions:** `Administration: Read and write` (covers
branch protection, rulesets, required status checks, repo settings, on
whichever repos the App is installed against — install on all
repositories, or all current + a policy of adding new ones as they're
created).

**No other permissions** — no `Contents`, no `Pull requests`, no
`Metadata` beyond what's implicitly required. This App can reshape repo
settings and org/team membership but can't read or write code, matching
"no other access."

**Usage:** this is what `admin-github`'s Woodpecker pipeline should authenticate
as — replaces the personal fine-grained PAT currently sitting in
`kv/homelab/gh-org`. Generate a short-lived installation access token at
the start of each CI run (a few lines of `curl`/`jq` against GitHub's
App-auth endpoints, using a JWT signed with the App's private key) instead
of a long-lived static token. The App's private key is the one long-lived
secret to protect — same OpenBao pattern as everything else here.

**Worth calling out explicitly:** this identity can rewrite the branch
protection that constrains the PR bot. If its credential (today: the
personal PAT; recommended: the App's private key) were ever compromised,
an attacker could disable `main` protection org-wide and push anything,
anywhere. Of the three identities, this one's credential deserves the
tightest handling — least people/processes with access to the OpenBao
path holding it, and worth considering shorter-lived credentials or extra
review on anything that touches it.

## Other recommendations

- **`admin-github` stays at 1 required reviewer, same as everywhere else** —
  there's one human (you), and the PR bot is the one opening PRs, never
  reviewing them. A 2-reviewer requirement isn't a stricter policy here,
  it's a lock with no second key: nothing could ever merge. `admin-github`
  defining branch protection for every other repo, including itself, is a
  real asymmetric risk (a bad approval there weakens protection
  everywhere at once) — but the mitigation is being deliberate about what
  you approve in that specific repo, not a reviewer count that doesn't fit
  a one-human org.
- **2FA required org-wide.** Organization settings → require two-factor
  authentication for everyone with access, not just the bot accounts.
- **Audit log retention/export.** GitHub's audit log for the org (repo
  creation, permission changes, protection changes) is worth periodically
  exporting somewhere durable if you want to investigate anything after
  the fact — GitHub's own retention window isn't indefinite on the plans
  used here.
- **Rotate the PR bot's PAT on a schedule**, since (unlike the App's
  installation tokens) it's long-lived by design. A calendar reminder is
  enough for a homelab; no need to build tooling for this.
- **CODEOWNERS for `admin-github`**, once there's more than one human reviewer,
  so changes to org-wide policy always route to whoever should be
  weighing in — not required today with a single admin, but cheap to add
  later and worth remembering.

## Setup checklist

1. Org Settings → Member privileges → Repository creation → off.
2. Org Settings → Require two-factor authentication → on.
3. Register the GitHub App for repo-admin, scoped to `Administration:
   Read and write` and `Members: Read and write` (org + repo as noted
   above), install on `mattjmorrison-homelab`.
4. Store the App's private key in OpenBao; update `admin-github`'s Woodpecker
   pipeline to mint an installation token from it instead of using the
   current personal PAT — this is what lets the remaining steps run
   through `admin-github` instead of by hand.
5. Sign up the PR bot's GitHub user account (own email, 2FA enabled) —
   the one step that can't be Terraform-managed, since nothing can create
   a GitHub user account via API, only manage an existing one's org/team
   membership.
6. Add `github_membership`, `github_team_membership` (an
   `automation-write` team), and `github_team_repository` (Write, looping
   over `local.repos`) to `admin-github` for that account, and apply.
7. Generate the PR bot's fine-grained PAT (Contents RW, Pull requests RW,
   Metadata R) and store it in OpenBao.
8. Revoke the personal fine-grained PAT created earlier once step 4 is
   confirmed working — it was only ever meant to unblock `admin-github` before
   this proposal existed.
