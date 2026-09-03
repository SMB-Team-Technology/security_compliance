# gcp-probo-infra

Provisions the GCP infrastructure `probod` needs: a regional GKE cluster
(standard, not Autopilot — see the node disk note below), a Cloud SQL
Postgres instance (private IP, peered onto the cluster's VPC), a GCS bucket
for file storage, a service account + HMAC key pair so `probod` can talk to
that bucket over its S3-compatible API, and two Secret Manager secrets (the
generated DB password and the HMAC secret key) that
[`deploy-gcp.yaml`](../../../.github/workflows/deploy-gcp.yaml) reads at
deploy time.

`deploy-gcp.yaml` applies this module itself on every run (`terraform apply`
is idempotent — a no-op once everything exists), so nothing here needs to be
run by hand before a deploy except the one-time bootstrap below.
[`provision-gcp-infra.yaml`](../../../.github/workflows/provision-gcp-infra.yaml)
is a separate manual (`workflow_dispatch`) entry point kept around for
previewing a change (`action: plan`) or an ad hoc apply outside the deploy
pipeline — it is not a required step.

## One-time bootstrap (before the first run of either workflow)

Terraform can't create its own state bucket, and the APIs this module
enables can't be enabled by a plan that needs them already enabled. Run once,
by hand, against the target project:

```bash
gcloud config set project YOUR_PROJECT_ID

gcloud services enable \
  sqladmin.googleapis.com \
  storage.googleapis.com \
  secretmanager.googleapis.com \
  servicenetworking.googleapis.com \
  compute.googleapis.com \
  container.googleapis.com

# Terraform state bucket — name it, and set TF_STATE_BUCKET to it.
gcloud storage buckets create gs://YOUR_TF_STATE_BUCKET --location=YOUR_REGION
```

Then add these repository secrets (in addition to the ones `deploy-gcp.yaml`
already needs — `GCP_SA_KEY`, `GCP_PROJECT_ID`, `GKE_CLUSTER`,
`GKE_CLUSTER_LOCATION`, which must equal `GCP_REGION`):

| Secret | Example |
|---|---|
| `TF_STATE_BUCKET` | `probo-tfstate` |
| `GCP_REGION` | `us-central1` |
| `GCS_BUCKET_NAME` | `probo-production-files` (must be globally unique) |

The `GCP_SA_KEY` service account needs enough to apply everything in this
module on the first run — Compute, Cloud SQL, Storage, Secret Manager, GKE,
and Project IAM admin (it grants itself the narrower runtime roles it needs
day-to-day as part of that same apply). Once the resources exist, subsequent
applies need less, but there's no reason to narrow it back down.

## Why node disks are `pd-standard`, and why the cluster isn't Autopilot

GKE Autopilot's default node disks are `pd-balanced`, which draws from the
same regional `SSD_TOTAL_GB` Compute Engine quota as Cloud SQL. Running both
against a default (often modest) quota can exhaust it before either
finishes provisioning. This module uses a standard cluster with explicit
`pd-standard` node disks (`var.gke_node_disk_size_gb`, default 50 GB) to
avoid that quota entirely — `probod` doesn't need fast local disk since its
state lives in Postgres and GCS, not on the node. If you still hit a quota
error, check current usage (`gcloud compute regions describe REGION
--format="table(quotas.metric,quotas.limit,quotas.usage)"`) and request an
increase, or lower `gke_node_count`.

## What it doesn't do

- Doesn't create the VPC — `var.network` (default `"default"`) must name a
  VPC that already exists.
- Doesn't rotate the DB password or HMAC key. Re-running `apply` after a
  manual rotation in the console will fight the state; rotate through
  Terraform (`terraform taint random_password.db`, or the HMAC key
  resource) instead.
- `deletion_protection = true` on the Cloud SQL instance is a guard against
  `terraform destroy`, not a backup strategy — point-in-time recovery is
  enabled, but test a real restore before you need one.
- Doesn't adopt a cluster, instance, or bucket you created by hand outside
  Terraform under the same name — that will surface as an "already exists"
  error on apply. Either delete the hand-created resource first, or
  `terraform import` it into this module's state.
