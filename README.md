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

Everything self-bootstraps through PreSync hooks, in wave order:

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

All of Garage's own core resources (Deployment, Service, PVC, ConfigMap)
are also tagged as PreSync hooks with earlier waves than the bootstrap Job
that depends on them — required so they actually exist and are Healthy
before that Job runs; a plain (non-hook) resource wouldn't be applied
until the main Sync phase, which happens *after* PreSync completes.

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
