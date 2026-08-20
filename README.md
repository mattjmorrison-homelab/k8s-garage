# k8s-garage

[Garage](https://garagehq.deuxfleurs.fr/) — self-hosted S3-compatible
object storage, single-node here (`replication_factor = 1`).

## Why

Started as a state backend for `gh-org`'s OpenTofu state (see that repo's
README — the `kubernetes` state backend was the earlier plan; a real S3
bucket is the more standard and more portable choice now that this exists).
Also the eventual home for hosting `ui-hdmi-switch`'s static build from a
public bucket instead of an nginx Deployment — **not built yet**, deferred
until this is confirmed working for state. Garage's website-hosting mode
supports that directly when the time comes; it just needs an Ingress in
front for TLS (Garage doesn't terminate TLS itself), same pattern as every
other service here — Traefik + the existing wildcard `*.morrisons.site`
cert.

## What's here

Everything self-bootstraps through plain `sync-wave` ordering (no
`argocd.argoproj.io/hook` annotations at all):

1. `garage-bootstrap-secrets` (wave -4) — mints `rpc_secret`/`admin_token`/
   `metrics_token` if not already in `kv/homelab/garage`, same
   check-then-mint idiom as `homelab-woodpecker`'s bootstrap Job.
2. Garage itself deploys (wave -1) once those secrets exist.
3. `garage-bootstrap-tofu-state` (wave 0, runs after Garage is Healthy) —
   assigns this node to the cluster layout if unassigned, then creates a
   `tofu-state` bucket + an access key scoped to just that bucket (via
   Garage's admin API — `/v2/CreateBucket`, `/v2/CreateKey`,
   `/v2/AllowBucketKey`), and writes the resulting
   `TOFU_STATE_ACCESS_KEY_ID`/`TOFU_STATE_SECRET_ACCESS_KEY` into
   `kv/homelab/gh-org` (merged in alongside the existing `github_token`,
   not overwriting it).

**Deliberately not using PreSync hooks**, despite the ordering need looking
like the same shape as `homelab-woodpecker`'s bootstrap Job. First attempt
did tag every resource as a PreSync hook (reasoning: the tofu-state Job
depends on Garage being Healthy, so everything it depends on transitively
needs to run before the main Sync phase too) — that broke the deploy
entirely: when *every* resource in a chart is a hook, ArgoCD's normal
Sync-phase diff compares an empty target set against an empty live set,
reports `Synced`/`Healthy` immediately, and never triggers a sync
operation at all — so the hooks (which only run *during* a sync operation)
never fire either. Confirmed live: the namespace never even got created,
despite the Application showing `Synced`/`Healthy`. Plain `sync-wave`
ordering doesn't have this problem — ArgoCD applies wave N, waits for it
to become Healthy (a Job counts Healthy once `Succeeded`), then moves to
wave N+1, all within the normal Sync phase — which is really all the
ordering here ever needed.

One tradeoff from dropping hooks: no `hook-delete-policy`, so if either
bootstrap Job's *first* run fails outright (not just "nothing to do"),
the failed Job object sticks around and needs a manual
`kubectl delete job -n garage <name>` before a fixed retry can create a
new one — Jobs are immutable, so ArgoCD can't just reapply over it.

One deliberate simplification: a single ServiceAccount/Vault role
(`garage`) is shared by the SecretStore and both bootstrap Jobs, rather
than a least-privilege role per Job like `homelab-woodpecker` uses. Fine
for a single-admin homelab; tighten later if it matters.

## Manual OpenBao setup (can't be done from here)

A Kubernetes auth role named `garage`, bound to the `garage` ServiceAccount
in the `garage` namespace, with a policy granting:

- read + write on `kv/data/homelab/garage` (and `kv/metadata/homelab/garage`)
  — where its own rpc/admin/metrics secrets get minted and stored
- read + write on `kv/data/homelab/gh-org` — **a cross-app grant**: this is
  `gh-org`'s own kv path (already holds `github_token`), and the
  tofu-state bootstrap Job needs to write the two `TOFU_STATE_*` keys into
  it alongside that

No manual secret *values* need seeding — unlike `gh-org`'s GitHub token,
everything here mints itself once the role/policy exist.

## Not done yet

- `gh-org`'s `backend "s3" { ... }` block pointing at this bucket, and the
  `tofu init -migrate-state` to actually move state here. Left out
  deliberately — didn't want to wire up a backend pointing at
  infrastructure that doesn't exist yet and isn't deployed/verified. Next
  step once this is live.
- Public bucket + website-hosting Ingress for `ui-hdmi-switch`.
