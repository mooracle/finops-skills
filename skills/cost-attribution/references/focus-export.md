# Phase 1 — FOCUS exports

Goal: FOCUS 1.2 billing data landing where your engine can read it (a bucket for
Athena in the reference stack; BigQuery for Google Cloud's own export) (article Part 2). Turn exports on as early as possible, even before the rest of the build:
history is the part you can't always get back later.

## Contents

- What FOCUS buys you · AWS · An export that already exists · Azure · GCP · Vendors
  that bill you directly · No access yet, or practice first · Gate

## What FOCUS buys you

One schema for every provider; the adapters read the columns `providers.md` lists.
Azure's `SubAccountId` is the full `/subscriptions/<GUID>`, not the bare GUID: a bare
GUID in a rule, `vendor_registry` or `k8s_pools` never matches.

**Version.** As of November 2026 AWS, Azure, Google Cloud and Oracle export FOCUS 1.2.
SaaS vendors that export FOCUS are at 1.0, 1.2 or 1.3 (Databricks, MongoDB, Snowflake and
Vercel at 1.3), and the spec is at 1.4. Build on 1.2: it's what every major cloud
exports. Check
https://focus.finops.org/docs/implementation/get-started/get-started-with-focus-datasets/
for each provider's latest version. 1.2 is
not backward-compatible with 1.0 (row counts, nullability, categories): if a 1.0 export
exists, run 1.2 beside it and switch when totals match. From 1.3 the seller is
`ServiceProviderName`: `PublisherName`, which `first_party`, D6 and the vendor registry
read, is deprecated in 1.3 and removed in 1.4.

## AWS

Billing and Cost Management → **Data Exports** → Create → **Standard data export**,
table **FOCUS with AWS columns**, version **1.2**, Parquet, your bucket with prefix
`raw/focus/AWS`. In Terraform it's one `aws_bcmdataexports_export`, and these are the
parts that matter:

- `export.name`, such as `focus-1-2`; it becomes a folder in the path below.
- `data_query.query_statement` = `SELECT <columns> FROM FOCUS_1_2_AWS`. Data Exports
  rejects `SELECT *`: list every column (60 for FOCUS 1.2), the `x_*` extensions
  included. `aws bcm-data-exports get-table --table-name FOCUS_1_2_AWS` prints the
  table's columns; use that list, not one from memory.
- `data_query.table_configurations` = `{ "FOCUS_1_2_AWS" = { "TIME_GRANULARITY" =
  "DAILY" } }`.
- `destination_configurations.s3_destination`: `s3_bucket`, `s3_prefix =
  "raw/focus/AWS"`, `s3_region`, and `s3_output_configurations` with `overwrite =
  "OVERWRITE_REPORT"` (one current snapshot per month), `format = "PARQUET"`,
  `compression = "PARQUET"`, `output_type = "CUSTOM"`.
- `refresh_cadence.frequency = "SYNCHRONOUS"`.
- The export lives in the **payer (management) account**. If the pipeline runs in a
  tooling account, deliver the export straight to a bucket there (`s3_bucket_owner` = that
  account), or grant it read on the payer's bucket.
- The Data Exports API is in `us-east-1` only: give the resource a provider pinned
  there. The bucket policy must let `bcm-data-exports.amazonaws.com` put objects in the
  bucket, conditioned on `aws:SourceAccount` = the payer and `aws:SourceArn` =
  `arn:aws:bcm-data-exports:us-east-1:<payer>:export/*` (also when another account owns
  the bucket), or the export fails to deliver.
- A tag reaches the export only once it's activated as a **cost allocation tag** in the
  payer account, and only from then on, unless you request a backfill (Cost allocation
  tags → **Backfill tags**, up to 12 months back, for tags the resources carried then).
  Activate the routing keys before discovery, or D3 and D7 come back empty.
- Each delivery is the **whole month to date**, refreshed at least daily; the first lands
  within a day (AWS allows three). AWS may still restate a month in the two weeks after
  it ends. With overwrite on, the month's folder holds one current snapshot:

```
s3://<bucket>/raw/focus/AWS/focus-1-2/
  ├── data/billing_period=2026-04/*.snappy.parquet
  └── metadata/billing_period=2026-04/focus-1-2-Manifest.json   ← trigger on this
```

The month folder's case (`billing_period=` here, `BILLING_PERIOD=` on other exports)
is the export's choice, not yours: list the bucket after the first delivery and match
it in the projection and the copy job's prefix filter.

`Manifest.json` lands only after every data file of the delivery and lists them. It is
the list to copy: when a delivery needs fewer chunks than the last one, overwrite mode
leaves the extra chunks behind as empty files, so a listing of `data/` is wrong. The
copy Lambda stages exactly the manifest's files and the ETL publishes them
(`pipeline.md`, "Delivery").

**History:** open a support case from the payer account; AWS can backfill up to 14
months.

## An export that already exists

Someone may have turned one on before you. `aws bcm-data-exports list-exports` names
them and `get-export` shows each one's shape. Three settings decide what to do:

- **Version.** `FOCUS_1_0_AWS` in the query: keep it, and add the 1.2 Parquet export
  above beside it (the version note under "What FOCUS buys you"). Never edit an existing
  export's version or mode in place: its folders are the history you can't get back.
- **Format.** CSV gzip: discovery reads it as it is (`Tags` is a JSON string,
  costs and timestamps are text; `discovery.md`), and the 1.2 export brings Parquet.
- **Overwrite.** `CREATE_NEW_REPORT` instead of `OVERWRITE_REPORT` (`delivery: new_folder`):
  keep it and add the 1.2 overwrite export above beside it, as for a 1.0 one. In this mode
  every delivery is a new folder, `<timestamp>-<execution id>/`, under the month in both `data/` and
  `metadata/`, and nothing is ever removed, so a listing or a projection over the month
  counts each delivery again. The month-level manifest
  (`metadata/billing_period=YYYY-MM/<export>-Manifest.json`) is rewritten with each
  delivery, carries the delivery's `executionId`, and its `dataFiles` list the newest
  folder's files: the copy job stages exactly those, as in overwrite mode. Each delivery
  also writes a copy of the manifest inside its own folder, so an `ObjectCreated` rule
  on the manifest suffix fires twice per delivery; both copies list the same files, so the
  second event at most stages and publishes those files again (`pipeline.md`, "Delivery",
  keys a delivery on the manifest object's ETag).

A prefix is an opaque string and can hold characters the shell treats specially, quotes
and dots included. Quote it in the shell and copy it exactly as `get-export` prints it.

## Azure

Cost Management → **Exports** → a FOCUS export to a storage account, Parquet where
offered, **Overwrite data** on so each month keeps only its latest snapshot. Backfill:
the export's *Export selected dates* command, one month per run, up to 13 months back.
Older months: the Exports REST API's *Execute* with a custom time period, one calendar
month per call, up to 7 years back. FOCUS exports need an EA or MCA billing account: a
classic pay-as-you-go (MOSP) subscription can't export FOCUS, but pay-as-you-go on an
MCA can. The version picker says 1.2-preview.

Each run writes a folder, `<directory>/<export>/<YYYYMMDD-YYYYMMDD>/<run id>/`, holding
the data files and a `manifest.json`: `blobs` (each with `byteCount` and
`dataRowCount`), `dataRowCount` for the run, and `runInfo` (`runId`,
`submittedTime`). With overwrite on, a month keeps one run folder. In the first five days
of a month, a daily export also reruns the previous month, so a closed month's numbers
can still change then.

If the pipeline runs on AWS, an hourly poller (read-only SAS token in SSM SecureString)
stages each month's newest complete run, meaning one whose manifest is there and whose
blobs all match it, and the ETL publishes it into `raw/azure/focus/billing_period=…/`
(`pipeline.md`, "Delivery": the cursor is the published run, not the last one seen).
Note the SAS token's expiry date in DECISIONS.md and a calendar.

## GCP

The adapter, the sync's shape (labels merged into one map, the invoice month as the
billing period) and the currency choice are in `providers.md`; add `gcp` to the provider
enum (`stack.md`).

Billing export → BigQuery → enable the **FOCUS** export. No backfill: in a US or EU
multi-region dataset the data starts at the beginning of the previous month; in a
regional dataset, the day you enable it. Shape and sync from BigQuery on a schedule;
the billing month follows Pacific time.

## Vendors that bill you directly

Some publish FOCUS data of their own (the version note above links the list): check for
it before building rows from the billing API. Either way, their shim writes each invoice
as FOCUS rows to
`raw/saas/focus/` (the `saas_focus` table, provider `other`; `saas.md`).

## No access yet, or practice first

- The FinOps Foundation's 10k-row FOCUS 1.0 sample:
  https://raw.githubusercontent.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS-Sample-Data/main/FOCUS-1.0/focus_sample_10000.csv

The sample is 1.0: 1.2-only columns won't appear in it. To see the whole build work on
a sample month with a known answer, run the trial instead (`trial.md`).

## Gate

1. A full month is present (D0): the export's listing shows the month's
   `billing_period=` folder, in the export's case, and the D0 query returns rows.
2. Record in DECISIONS.md: bucket, prefix, export name, version, format, overwrite
   mode, the month folder's case, months available, whether a legacy export coexists,
   the backfill requested.
