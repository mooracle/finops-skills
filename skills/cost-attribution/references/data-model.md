# Data model — where the data lives and what shape it has

Decide this once, in Phase 2, and don't move it: the S3 layout is the partition scheme,
and every template, view and test assumes these shapes. The column lists are the core;
the profile adds a `tag_<key>` column per routed tag on top (`x_resourcegroupname` is
core: NULL outside Azure).
`pipeline.md` covers how data moves between the tables; this file covers what they hold.

## Contents

- S3 layout
- Raw layer
- Weights: one table, three producers
- Warehouse: `base_monthly`, `base_daily`
- Rules and views
- Finance outputs
- Provider quirks
- Rules of thumb (cardinality, schema changes, CSV)

## S3 layout

```
s3://<prefix>-data-<account>/
├── raw/                                     what arrived; only the ETL's publish step replaces a month
│   ├── focus/AWS/focus-1-2/…                  where the AWS export itself writes (focus-export.md)
│   ├── aws/staging/billing_period=YYYY-MM/<delivery>/   a delivery's files and its manifest, verified
│   ├── aws/published/YYYY-MM.json             which delivery the live partition holds, written last
│   ├── aws/focus/billing_period=YYYY-MM/      the live month: the latest published delivery
│   ├── azure/focus/billing_period=YYYY-MM/
│   ├── gcp/focus/billing_period=YYYY-MM/      synced from BigQuery (providers.md)
│   ├── saas/focus/billing_period=YYYY-MM/     invoices of directly billed vendors (FOCUS)
│   ├── vendors/lines/billing_period=YYYY-MM/<vendor>.csv    the vendor's breakdown
│   ├── vendors/fetch/billing_period=YYYY-MM/<vendor>.csv    how far the breakdown reaches
│   ├── vendors/<vendor>/native/…              verbatim API payloads, append-only
│   └── k8s/metrics/billing_period=YYYY-MM/cluster_id=<id>/<day>.csv
├── reference/                               small, human-owned, edited in Git
│   ├── vendor_registry/registry.csv
│   ├── k8s_pools/pools.csv
│   ├── commitments/commitments.csv          prepaid deals spread on the bill side (saas.md)
│   └── invoice_totals/invoice_totals.csv    the invoice totals the org enters (Q-F5)
├── warehouse/                               rebuilt, never edited
│   ├── split_weights/billing_period=YYYY-MM/provider=<p>/
│   ├── base_monthly/billing_period=YYYY-MM/provider=<p>/
│   └── base_daily/billing_period=YYYY-MM/provider=<p>/
├── rules/
│   ├── staged/lookup.csv                    CI writes the fresh expansion here
│   └── lookup/lookup.csv                    the merge Lambda writes; = rules_lookup
├── close/<provider>/YYYY-MM.csv             frozen monthly close, written once
├── invoice_check/<provider>/YYYY-MM.csv     the month's invoice check, replaced by each check
└── athena-results/                          7-day lifecycle expiry
```

Metrics files sit one folder per cluster (`cluster_id=` is the profile's unique id), so
two clusters never overwrite each other's day. Not `cluster=`: an engine that infers
partitions from `key=value` folders (Spark, DuckDB's hive partitioning)
would replace the file's `cluster` column, the cluster tag's value that the weights
join on, with the profile id, and every pool would lose its weights.

Each export-fed provider has the same `staging/` and `published/` beside its `focus/`
(`pipeline.md`, "Delivery"). Directly billed invoices (`raw/saas/focus/`) have neither:
one object per vendor and month, replaced whole by its shim.

Partition projection, and what replaces it on another engine: `stack.md`.

## Raw layer

| Table | Grain | Notes |
|---|---|---|
| `aws_focus`, `azure_focus`, `gcp_focus` | one FOCUS line item | one table per provider; the adapters unify them |
| `saas_focus` | one invoice line, FOCUS columns | written by a directly billed vendor's shim (`saas.md`); provider `other`; its total must match the invoice |
| `vendor_lines` | vendor × month × workspace × service line | `billing_period, vendor, workspace, servicename, amount`; `workspace` empty = cost the vendor can't map to one |
| `vendor_fetch` | vendor × fetch | `billing_period, vendor, through_day, fetched_at`: the last day the lines are complete for |
| `k8s_metrics` | day × cluster × pool × namespace | `billing_period, dt, provider, subaccountid, regionid, cluster, pool, namespace, cpu_h, mem_h, gpu_h`: requested core-hours, GiB-hours, GPU-hours. The row with an empty namespace is the pool's allocatable capacity; a resource the pool doesn't have is 0 |

The adapters (`providers.md`) turn each raw FOCUS table into one shared column list.

`bill_v` is the adapters stacked (`UNION ALL`, one branch per provider) plus three
columns, decided once per row. A row that qualifies twice is, in this order, a shared
charge, then a vendor's charge, then a Kubernetes row:

| Column | Value |
|---|---|
| `weight_key` | the weights that apply, if any: `share:<id>:<payer>` for a row with `shared_cost`; `vendor:<id>` for a row the registry matches (same provider, lower-cased `publishername` `LIKE` the entry's `publisher_like`, and the entry's account when it names one; registry rows read `DISTINCT`; a row with `shared_cost` never joins the registry), only when the charge lies inside its billing month; `k8s:<account>/<region>/<cluster>/<pool>` for a cluster row with no namespace of its own |
| `split_by` | which splitter owns the row, whether or not it splits: `share:<id>`, `vendor:<id>`, `k8s:<cluster>` |
| `k8s_home` | where a Kubernetes row lands without weights: `<cluster>/<namespace>` when its tags name a namespace, `<cluster>/pool:<pool>` when they name only a pool, `<cluster>/cluster:shared` otherwise (the control plane). NULL on shared-charge, vendor and non-cluster rows. `:` can't occur in a namespace name, so a bucket never collides with one |

The loads set `x_namespace` to the weight's `to_namespace`, else `k8s_home`, and keep
`split_by` only on a row that was split or has a Kubernetes home: an unsplit vendor or
shared charge loads with `split_by` NULL, like any plain row.

## Weights: one table, three producers

Every split's weights live in one table (invariant 6):

| `split_weights` column | Meaning |
|---|---|
| `weight_key` | which rows these weights split: `share:<id>:<payer>`, `vendor:<id>`, `k8s:<account>/<region>/<cluster>/<pool>` |
| `to_subaccountid`, `to_subaccountname` | the account the share lands on (fair-share). They move as a pair: a row with a `to_subaccountid` takes `to_subaccountname` even when it's NULL, never the source row's name; a NULL `to_subaccountid` keeps the row's own id and name |
| `to_namespace` | what lands in `x_namespace`: `<cluster>/<namespace>` (K8s) or the workspace (vendor); NULL keeps the row's own |
| `to_servicename` | the vendor's service line; NULL keeps the row's own |
| `weight` | the share, in (0, 1]; a key's weights sum to 1.0 per `dt` |
| `dt` | the day the weights apply to: set on `k8s:` keys (one set per pool and day, never today or later; `kubernetes.md` "Daily weights"), NULL on `share:` and `vendor:` keys, which hold for the month. The loads join a row to its key's weights where `dt` is NULL or equals the row's day (`DATE(chargeperiodstart)`) |
| `billing_period`, `provider` | partition |

| Producer | Writes | Inputs |
|---|---|---|
| `account_weights` | `share:<id>:<payer>` | `bill_v`: first-party spend per account, per payer (`splitters.md`) |
| `namespace_weights` | `k8s:<account>/<region>/<cluster>/<pool>` | `k8s_pools`, `k8s_metrics` (`kubernetes.md`) |
| `vendor_shares` | `vendor:<id>` | `vendor_registry`, `vendor_lines`, `vendor_fetch` (`saas.md`) |

`split_weights` holds what the last run computed (`pipeline.md`, "One run"): after a
good run, the weights the base tables were built with; after a failed weights check,
the rejected weights, while the base tables still hold the previous build.

Reference tables, edited in Git and copied to storage:

| Table | Columns |
|---|---|
| `vendor_registry` | `vendor, provider, publisher_like, subaccountid`: one row per way a vendor shows up on a bill. `publisher_like` is a lowercase LIKE pattern on `PublisherName`; `subaccountid` narrows it to the account that buys (NULL = any). A vendor id is one billing identity: two payers buying the same vendor separately are two ids. A row without `vendor`, `provider` or `publisher_like` matches nothing |
| `k8s_pools` | `provider, subaccountid, regionid, cluster, pool, mode, namespace, idle, weights, step`: `mode` is `dedicated` (then `namespace` is the owner), `shared` or `gpu`; `idle` is `spread` (also when empty) or `bucket`, and `weights` is `daily` (also when empty) or `monthly`, both read on `shared` and `gpu` pools; `step` is the Prometheus sync's sample step on those pools, `1h` (also when empty) or `5m` (`kubernetes.md`) |
| `commitments` | `vendor, billingaccountid, subaccountid, publishername, servicename, total_paid, total_credits, start_period, end_period, currency`: one row per prepaid deal the user wants spread, read by the adapter of the provider the vendor bills on (`saas.md`, "Prepaid deals") |
| `invoice_totals` | `provider, billing_period, invoice, currency, amount`: one row per thing the invoice check compares, entered by whoever Q-F5 names (`reconciliation-ops.md`, "The invoice check") |
| `fx_rates` | `billing_period, from_currency, to_currency, rate`, kept by whoever Q-F4 names: an amount in `from_currency` times `rate` is the amount in `to_currency`. Only when a provider bills in a currency other than the reporting one and its export can't convert (`providers.md`) |

## Warehouse: `base_monthly`, `base_daily`

One table each for all providers, partitioned `(billing_period, provider)`. Every row is
a group of bill rows with the same routing columns, costs summed, every split applied.
`base_daily` adds `dt`. Rules only ever see these columns, never raw FOCUS: the ones
they can match (profile `routing.key`) are fixed here (`rules-engine.md`, "What rules
can match").

| Column | Type | From | Notes |
|---|---|---|---|
| `dt` | date | charge period | daily only; never today or later |
| `billingaccountid` | varchar | FOCUS | the payer; no split crosses it |
| `subaccountid`, `subaccountname` | varchar | FOCUS, or the fair-share target | the name is the latest non-empty one in the month; a NULL name never wins |
| `regionid` | varchar | FOCUS | |
| `servicename`, `servicecategory` | varchar | FOCUS; `servicename` is the vendor's service line on a vendor split | |
| `chargecategory`, `pricingcategory` | varchar | FOCUS | |
| `x_resourcegroupname` | varchar | Azure | NULL elsewhere |
| `tag_<key>` | varchar | tags, spellings folded | one column per routed tag key |
| `x_namespace` | varchar | splits | K8s `cluster/namespace` (or a `cluster/pool:<pool>`, `cluster/idle:<pool>` or `cluster/cluster:shared` bucket), a vendor workspace; NULL on plain rows |
| `split_by` | varchar | splits | `share:<id>`, `vendor:<id>`, `k8s:<cluster>`; NULL on plain rows |
| `billedcost`, `effectivecost`, `listcost` | double | FOCUS × weight | every column split by the same weight |
| `currency` | varchar | FOCUS | one currency per partition (the totals check enforces it) |
| `billing_period`, `provider` | varchar | partition | `provider` is the bill the cost sits on |

## Rules and views

`rules_lookup`, `services_daily`, `services_monthly` and the optional `org_hierarchy`:
`rules-engine.md`. Only matched tuples have lookup rows, and the merge never removes one.

## Finance outputs

The frozen close (`close/<provider>/YYYY-MM.csv`, only if finance freezes months, Q-F1),
the invoice check's results (`invoice_check/<provider>/YYYY-MM.csv`, where the org enters
invoice totals, Q-F5) and budgets: `reconciliation-ops.md`.

## Provider quirks

Each provider's quirks (currency, tags, Azure's timestamps and portal names) are
handled in its adapter: `providers.md`.

## Rules of thumb

- **Cost follows cardinality, not bill size.** Each routing column multiplies the grouped
  rows; `ResourceId` makes every resource its own row and the warehouse grows back to raw
  size. Keep columns at org-chart level.
- **Adding a column means rebuilding history** (`pipeline.md`, "Backfill and schema
  changes"); a new key field also migrates the lookup (`rules-engine.md`).
- **CSV tables return `''`, not NULL**, for empty fields. Read them through a view with
  `NULLIF(col, '')` and a `TRY_CAST` per column, so a shim never breaks a type: text that
  isn't a value of the type reads as empty. Strip
  newlines from vendor free text before writing, or a row splits in two.
