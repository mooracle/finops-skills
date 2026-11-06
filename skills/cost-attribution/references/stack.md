# Phase 2 — The stack

Goal: the engine can query the raw FOCUS data cheaply, and the storage layout is fixed for
good.

## Contents

- What any stack must provide
- The reference stack: AWS
- S3 layout: decide once, never move
- Athena workgroup
- Glue tables
- Secrets and Terraform state
- Gate

## What any stack must provide

Whatever the engine (`runtime.engine`, intake Q-R0), Phase 2 is done when it has:

- **One raw table per provider**, partitioned by billing month, holding that month's latest
  export snapshot. Never one unified raw table: the `x_*` extensions differ.
- **Warehouse tables partitioned by (billing month, provider)**: `split_weights`,
  `base_monthly`, `base_daily`. A run replaces one partition and touches nothing else.
- **A way to replace one partition** (delete its files, `DELETE … WHERE`, `INSERT
  OVERWRITE`, `MERGE`) and a way to know a statement failed.
- **A scan cap** per query or workgroup, so a missing partition filter fails loudly
  instead of scanning all of history.
- **Small reference tables** edited in Git (`vendor_registry`, `k8s_pools`,
  `commitments`, `invoice_totals`, `fx_rates` if needed) and the inputs the syncs write
  (`vendor_lines`, `vendor_fetch`, `k8s_metrics`), all present even when empty.
- **The lookup** (`rules_lookup`) where the rules engine publishes it.

The rest of this file is the reference stack.

## The reference stack: AWS

Eight primitives, all serverless: S3, Parquet, Lambda, Athena,
EventBridge, SSM Parameter Store, Glue Data Catalog, plus whatever dashboard tool. Total
run cost ~$8–10/month at multi-provider scale. Generate the Terraform for the user; the
settings below are the parts that are easy to get wrong.

## S3 layout — decide once, never move

The full layout is in `data-model.md`. The rule that matters here: **the folder layout is
the partition scheme.** `billing_period=YYYY-MM/` (and `provider=<p>/` in the warehouse)
in the path lets partition projection prune to one month with no catalog updates, so a
query's cost stays flat as history grows.

Lifecycle: expire `athena-results/` after 7 days and `raw/*/staging/` after 30; with
versioning on, expire noncurrent versions after ~30 days and remove expired delete
markers (every partition clear leaves one per object); abort incomplete multipart
uploads after 7 days. Raw and warehouse data stay: a year of grouped Parquet is
megabytes. Raw months a rebuild rarely reads can transition to a tier Athena reads
without a restore (S3 Glacier Instant Retrieval); never to Flexible Retrieval or Deep
Archive, where a backfill would first need a restore.

## Athena workgroup

Non-negotiable settings, in the `configuration` block of `aws_athena_workgroup`:

- `bytes_scanned_cutoff_per_query = 1073741824` (1 GB): the scan cap.
- `enforce_workgroup_configuration = true`, so no client can override the cap or the
  results location.
- `publish_cloudwatch_metrics_enabled = true`. The metrics show Athena's part of the run
  cost at the end of the build.
- `result_configuration { output_location = "s3://<bucket>/athena-results/" }`, the
  prefix the 7-day lifecycle rule expires.

## Glue tables

Define as `aws_glue_catalog_table` resources:

- `aws_focus` — Parquet, columns mirroring the export. Tags column is
  `map<string,string>`.
- `azure_focus` — CSV (OpenCSVSerde) or Parquet depending on export choice; tags is a
  JSON `string`.
- `gcp_focus` — Parquet from the BigQuery sync, labels merged into one `tags` map
  (`providers.md`).
- `base_daily`, `base_monthly` — Parquet, partitioned by `(billing_period, provider)`;
  columns come from the profile's dimensions, so create these *after* discovery.
  Partition keys come last in the column list.
- `saas_focus` — the FOCUS rows of directly billed vendors (provider `other`).
- `split_weights` — Parquet, partitioned like the base tables.
- `rules_lookup` — CSV table over `rules/lookup/`.
- `vendor_registry`, `vendor_lines`, `vendor_fetch`, `k8s_pools`, `k8s_metrics`,
  `commitments`, `invoice_totals` — **always**, even without those tracks: `bill_v` (its
  adapters read `commitments`), the weights producers and the invoice check read them,
  and empty they cost nothing (shapes in `data-model.md`). CSV inputs get a casting view.
- `bill_v` stacks every adapter, so every adapter's raw table must exist. For a
  provider the org doesn't have (no Azure, no directly billed vendors), either create the
  empty table or delete that adapter's branch from `bill_v`, its view, and its test
  fixtures together.

Use **partition projection** on every partitioned table — it beats crawlers and
`MSCK REPAIR`: zero maintenance, zero cost, new months just work. These table
`parameters` do it (all values are strings):

- `projection.enabled` = `true`.
- `billing_period` as a date projection: `projection.billing_period.type` = `date`,
  `projection.billing_period.format` = `yyyy-MM`, `projection.billing_period.range` =
  `2024-01,NOW` (from the first month you hold), `projection.billing_period.interval` =
  `1`, `projection.billing_period.interval.unit` = `MONTHS`.
- On the warehouse tables, `provider` as an enum projection:
  `projection.provider.type` = `enum`, `projection.provider.values` = the providers you
  have, comma-separated (`aws,azure,gcp,other`; `other` is the directly billed vendors).
- `storage.location.template` = the table's path with each partition key as a literal
  `${key}`, such as
  `s3://<bucket>/warehouse/base_daily/billing_period=${billing_period}/provider=${provider}/`.
  Athena needs the literal `${…}`, so in Terraform write `$${billing_period}`: `$$`
  escapes interpolation. The template must end with `/`, or Athena rejects the table's
  queries.

## Secrets and Terraform state

The pipeline holds read credentials for other systems: an Azure SAS token, each
vendor's API key. Keep them out of everything Terraform writes down:

- **An ephemeral variable is asked for on every plan and apply**, CI included: give
  the pipeline that applies it the secret from its own store, or apply the parameter
  from a separate root module that only changes when the key rotates.
- **`ignore_changes` doesn't do it.** A parameter created with a placeholder and
  `lifecycle { ignore_changes = [value] }` still lands in state with its real value: the
  next refresh reads it back. Either set the value through a write-only argument
  (Terraform 1.11 and later, never stored in plan or state), or create the parameter
  outside Terraform and give Terraform only its name for IAM.
- **The write-only way:** a `variable` with `sensitive = true` and `ephemeral = true`
  (an ephemeral value never reaches the plan or the state), passed to
  `aws_ssm_parameter` (`type = "SecureString"`, a name per secret such as
  `/finops/vendors/<vendor>/api-key`) as `value_wo`, next to `value_wo_version = 1`.
  Terraform can't see a write-only value change, so rotating means bumping
  `value_wo_version`.

- **Read it at run time,** with IAM on that one parameter (and its KMS key), not on a
  path prefix every function shares.
- **State is remote:** an S3 backend (versioned, encrypted, `use_lockfile = true`), set
  up before the first apply.
- **Pull requests validate without credentials:** `terraform fmt -check`,
  `terraform init -backend=false`, `terraform validate`. Only main applies, with a role
  that can.

## Gate

The raw table reads what the ETL published, so build the delivery path in this phase
too: the copy job or poller, staging, and the ETL's first step, publish
(`pipeline.md`, "Delivery"). The rebuild comes in Phase 3. Then:

1. The month's published marker names the export's latest delivery, and the live
   partition holds exactly the manifest's files.
2. A raw month query scans little: on `aws_focus`, filtered to
   `billing_period = '<current month>'`, the row count and `SUM(BilledCost)` per
   `BillingPeriodStart`.

It must return rows, and the Athena console's "data scanned" must be a small fraction of
that month's Parquet in S3 (two columns of one partition: Parquet and partition pruning
working). If it scans more than the month's files, partition projection is
misconfigured — check the projection range and that the S3 path actually contains
`billing_period=` segments.
