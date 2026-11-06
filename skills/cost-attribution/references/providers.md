# Providers — one adapter each, and nothing provider-specific anywhere else

The **adapter** is the one place that knows a provider's shape (its `x_*` extensions,
tag format, currency, timestamps). It reads the provider's raw table and emits the
contract below; everything downstream reads only the contract.

Write one adapter per provider in the profile, in the user's engine. Below are the
Athena (Trino) rules for each; `cases/adapters.yaml` says what each must produce.

## Contents

- The contract
- Decisions every adapter encodes (charge categories, currency, tags, shared charges,
  Kubernetes)
- AWS
- Azure
- Google Cloud
- Vendors that bill you directly
- Adding a provider

## The contract

One row per line item, these columns, in this order (the stack of adapters is a
`UNION ALL`, which matches by position):

| Column | Type | Meaning |
|---|---|---|
| `provider` | varchar | a literal: `aws`, `azure`, `gcp`, `other` (the partition value) |
| `billing_period` | varchar | `YYYY-MM`, the raw partition |
| `billingaccountid` | varchar | the payer; no split crosses it |
| `subaccountid`, `subaccountname` | varchar | account, subscription, project |
| `regionid` | varchar | |
| `servicename`, `servicecategory` | varchar | NULL where the provider has none |
| `chargecategory`, `pricingcategory` | varchar | FOCUS values |
| `x_resourcegroupname` | varchar | Azure; NULL elsewhere |
| `tag_<key>` | varchar | one column per routed tag key (profile `dimensions.tags`) |
| `chargeperiodstart`, `chargeperiodend` | timestamp | UTC |
| `billedcost`, `effectivecost`, `listcost` | double | in `currency` |
| `currency` | varchar | one currency per provider-month |
| `publishername` | varchar | who sold it |
| `first_party` | boolean | sold by the provider itself, not resold through its marketplace (on AWS, Bedrock models count as AWS's own) |
| `shared_cost` | varchar | the id of the shared charge this row is (profile `shared_costs[].id`), else NULL |
| `k8s_cluster`, `k8s_pool`, `k8s_namespace` | varchar | from the row's own tags; NULL when absent |

Every adapter emits every column, `CAST(NULL AS varchar)` (or the column's type) where the
provider has no value, including each `tag_<key>`: a missing column shifts every column
after it. The adapter also drops the rows that aren't cost at all
(`finance.charge_categories`).

## Decisions every adapter encodes

**Charge categories.** Which `ChargeCategory` values count (profile
`finance.charge_categories`, intake Q-F3). AWS usually `Usage` and `Purchase`, plus
`Credit` where credits flow to the teams (the org's call, article Part 2); tax is the
org's call too (Q-F3). Azure usually `Usage`, `Purchase`: its export has no credit or tax
rows. Google Cloud folds credits into the cost and writes
only `Usage`, `Tax` and `Adjustment`, so its list is usually `Usage`, `Adjustment`.

**Currency.** Each provider bills in a currency (D0); finance reports in
`finance.reporting_currency` (Q-F4): USD, one local currency through the org's monthly
rate table (`finance.fx_rates`), or `per_provider`, converting nothing. Convert only
where billing and reporting differ:

| Provider | Columns | Report in the billing currency | Report in USD |
|---|---|---|---|
| AWS | core cost columns, `BillingCurrency` | as is (usually USD already) | an FX table, if the account bills in another currency |
| Azure | core columns in billing currency, plus `x_*CostInUsd` | as is | the `x_*InUsd` columns, `'USD'` |
| Google Cloud | core columns in billing currency, `x_CurrencyConversionRate` (local per USD) | as is | `cost / x_CurrencyConversionRate`, `'USD'` |
| Directly billed | whatever the invoice says | as is | an FX table |

One provider-month carries one currency (the totals check enforces it), and nothing
sums two currencies: if providers stay in their own billing currencies, dashboards
group by `currency`. Reconcile an invoice against the raw table, in the billing
currency and with every charge category, never against an adapter: the adapter's
`billedcost` is converted and filtered. AWS: one check per `InvoiceId` (a month has
several: usage, upfront purchases, Marketplace, now and then a credit memo; tax sits on
each). Google Cloud has no `InvoiceId`: check per billing account and invoice month,
against the month's invoices plus any credit or debit memos, tax and credits already in.
Azure's export has no tax and no Azure credits: on MCA, compare each invoice's total
before those two. EA has no `InvoiceId` either: check each billing account and month
against Total Charges on the portal's Usage + Charges page. Directly billed vendors that
invoice in different currencies convert through `fx_rates` the same way as AWS.

**Tags.** One `tag_<key>` column per key the rules route on, with every spelling D3
found folded in (`CostCenter`, `costcenter`, `cost-center`); "the first non-NULL" below
means the first non-empty one, since an empty tag value is no tag. The rules engine reads
`Tag:Cost-Center` as `tag_costcenter`.

**Shared charges.** `shared_cost` names the shared charge a row is, with the exact id
the profile lists (`support` for Enterprise Support). The run stops if the bill carries
an id the profile doesn't list, so a typo can't leave a charge unsplit silently.

**Kubernetes.** `k8s_cluster`, `k8s_pool`, `k8s_namespace` from the row's own tags
(`kubernetes.md`). The column recipes below are the full menu and the profile picks
from it: no cluster listed for the provider → the adapter writes NULL for all three;
`dimensions.tags` empty → no `tag_` column at all. A namespace tag only counts next to
a cluster. The cluster comes in several forms (a tag value, a tag key, or only the
resource id; the AWS recipe lists them in order), and D7 shows which forms each kind of
resource carries: read every one it finds.

## AWS

Tags arrive as `map(varchar, varchar)`; read them with `element_at(tags, 'Key')`, which
returns NULL for a missing key (`tags['Key']` fails). Cost is usually USD already.

`focus_aws_v` reads `aws_focus`:

- **Passed through:** `billing_period`, `billingaccountid`, `subaccountid`,
  `subaccountname`, `regionid`, `servicename`, `servicecategory`, `chargecategory`,
  `pricingcategory`, `chargeperiodstart`, `chargeperiodend`, the three money columns and
  `publishername`. `currency` is `billingcurrency`; `provider` is the literal `'aws'`.
- **NULL:** `x_resourcegroupname`.
- **`tag_costcenter`:** the first non-NULL of the keys `CostCenter`, `costcenter`,
  `cost-center`, in that order. Each routed tag key gets a column like it, folding the
  spellings D3 found.
- **`first_party`:** true when `publishername` is NULL, empty, or an AWS seller of record
  (first-party rows carry the payer's, such as `Amazon Web Services EMEA SARL`): in lower
  case it starts with `amazon web services`, `pt amazon web services` (Indonesia) or
  `amazon aws` (Brazil), as of November 2026 (D6 flags a new one). Also true for a model
  on Amazon Bedrock that its maker sells, such as Anthropic's: AWS bills it through the
  Marketplace with the maker as `publishername`, but counts it as Bedrock usage, support
  fee included. Its rows are the ones whose `servicename` contains
  `(amazon bedrock edition)` in lower case or whose `resourceid` starts with
  `arn:aws:bedrock:` (D6 confirms the markers on the user's bill). Anything else is a
  Marketplace resale.
- **`shared_cost`:** `'support'` where `servicename` is exactly
  `AWS Support (Enterprise)`, else NULL. One such condition per shared charge the
  profile lists.
- **Unused commitments:** `servicename` is `Unused commitment` where
  `commitmentdiscountstatus` is `'Unused'`, so the waste is a line of its own. AWS names
  those rows after the commitment (`Savings Plans for AWS Compute usage`), and their cost
  is in `effectivecost` only: `billedcost` is 0, and the fee sits on the `Purchase` rows.
  They keep their `subaccountid`, the account that bought the commitment (D9 checks it),
  so the waste lands on that account's owner, often whoever owns the management account.
- **`k8s_cluster`:** the first non-NULL of, in order:
  1. the tag `aws:eks:cluster-name` (node VMs, however they were launched; AWS-generated);
  2. the tag `eks:cluster-name` (managed node group VMs);
  3. the tag `elbv2.k8s.aws/cluster` (load balancers);
  4. the tag `KubernetesCluster` (EBS CSI volumes);
  5. the cluster's name when the tag `kubernetes.io/cluster/<name>` is `owned` (what EKS
     or the CSI driver created). The name is in the key, so this takes one condition per
     cluster name in the profile. `shared` gives NULL: the resource serves several
     clusters;
  6. the name in an EKS control plane's resource id, the whole id matching
     `arn:aws:eks:<region>:<account>:cluster/<name>`. Any other id gives NULL, never
     `''`: `regexp_extract` returns NULL on no match in Trino but `''` in DuckDB, so wrap
     it in `NULLIF(…, '')`.
- **`k8s_pool`:** the first non-NULL of the tags `eks:nodegroup-name` (managed node groups),
  `karpenter.sh/nodepool` (Karpenter) and `eks:kubernetes-node-pool-name` (EKS Auto Mode);
  D7 shows which one a cluster carries (profile `tags.pool`).
- **`k8s_namespace`:** the first non-NULL of the tag
  `kubernetes.io/created-for/pvc/namespace` (volumes), then the part before the first
  `/` of `ingress.k8s.aws/stack`, then of `service.k8s.aws/stack` (load balancers; the
  value is `namespace/name`). A value with no `/` is an IngressGroup's name, not a
  namespace: it gives NULL.
- **EKS Auto Mode's keys** (`kubernetes.md`): `eks:eks-cluster-name` is one more
  `k8s_cluster` tag, after `eks:cluster-name`; `ingress.eks.amazonaws.com/stack`, then
  `service.eks.amazonaws.com/stack`, are two more `k8s_namespace` stacks after
  `service.k8s.aws/stack`, read the same way.
- **Rows kept:** `chargecategory` in the profile's list (`finance.charge_categories`;
  `Usage`, `Purchase`, `Credit` in the cases).

Reporting in a currency AWS doesn't bill in (EUR, say): convert through the org's
monthly rate table, `fx_rates` (`data-model.md`). Which rate (month-end, monthly
average) is the org's call, recorded as a `D-NNN` entry, and so is the rate that covers
the open month before its own exists (Q-F4). Wherever the bullets below give NULL money,
the totals check stops the run as `unloadable_rows`: a month with no rate never loads. The positive-and-finite
guard is the one nothing else provides: a rate of 0 erases the month and −1 reverses
it, and every totals check still passes, because the bill side reads the same
converted adapter.

The converting adapter is `focus_aws_v` with only its money and one join changed (the
cases report in EUR):

- **The rate.** Reduce `fx_rates` to one row per `(billing_period, from_currency)` for
  the reporting `to_currency` before joining, so a rate pasted twice can't double a cost.
  The rate is usable only when that month and currency have exactly one distinct rate,
  an empty one counted, and it is positive and finite; otherwise it is NULL.
- **The join.** A left join on the row's `billing_period` and `billingcurrency`: a row
  with no rate stays, with NULL money.
- **Money.** Each of `billedcost`, `effectivecost` and `listcost` is the amount as it is
  when `billingcurrency` already is the reporting currency, else the amount times the
  rate (NULL when the rate is).
- **`currency`** is the reporting currency, a literal, on every row.

## Azure

Tags arrive as a JSON string; read them with `json_extract_scalar`. CSV exports carry
timestamps as ISO-8601 text, which Athena's CAST rejects: parse it with `from_iso8601_timestamp`
(a Parquet export is already typed: drop the parse). Tag names can't contain `/`, so
Kubernetes keys differ (the disk tag is `kubernetes.io-created-for-pvc-namespace`); fill
`k8s_*` from what D7 finds.

`focus_azure_v` reads `azure_focus`:

- **Passed through:** as on AWS, plus `x_resourcegroupname`. `currency` is
  `billingcurrency`; `provider` is `'azure'`.
- **`tag_costcenter`:** `json_extract_scalar(tags, '$.CostCenter')`, then
  `'$.costcenter'`.
- **Timestamps:** `CAST(from_iso8601_timestamp(chargeperiodstart) AT TIME ZONE 'UTC' AS
  timestamp)`, likewise `chargeperiodend`: parsed, moved to UTC, then a plain timestamp
  like every other adapter's.
- **`first_party`:** `publishername` NULL, empty, or starting with `microsoft` in lower
  case.
- **Empty fields:** a CSV export writes `''` for an empty field; read text columns
  through `NULLIF(col, '')` so empty and NULL are one.
- **NULL:** `shared_cost` and the three `k8s_*` columns, until the profile names a
  shared charge on Azure or D7 finds clusters there; then fill them as on AWS, with
  Azure's keys.
- **Rows kept:** the profile's charge categories, as on AWS.

Reporting in USD instead: `billedcost`, `effectivecost` and `listcost` come from
`x_billedcostinusd`, `x_effectivecostinusd` and `x_listcostinusd`, and `currency` is
`'USD'`. Nothing else changes.

The portal's "service name" is `x_SkuMeterCategory`, not FOCUS `ServiceName`: group by
it when someone compares against the portal. Unused reservations and support rows carry
unhelpful `ServiceName`s; give them stable names here so rules can target them. Unused
commitment rows get `Unused commitment`, as on AWS: `commitmentdiscountstatus = 'Unused'`,
or, where the export leaves that column empty (Microsoft's 1.2-preview schema says it
may), `Usage` rows whose `resourceid` equals `commitmentdiscountid` in lower case, which
FOCUS requires of unused rows. They name no subscription (`subaccountid` empty,
`subaccountname` `Unassigned`), so where `NULLIF(subaccountid, '')` is NULL they take the
billing account as their account: `subaccountid` from `billingaccountid` and
`subaccountname` from `billingaccountname`, together. Key on the id, never the name.

## Google Cloud

The FOCUS export lands in BigQuery, in a read-only dataset Google creates
(`gcp_billing_immutable_<billing account id>_<location>`, table
`gcp_billing_export_focus_<billing account id>`). Google deletes its rows after two
years. **On BigQuery**, the adapter is a view over a copy you own, refreshed month by
month by a scheduled query, never over the export itself: a rebuild of a month that has
aged out stops on `no_bill_rows` and leaves that month's base tables cleared. Read a
label with
`(SELECT value FROM UNNEST(x_Labels) WHERE key = 'k')` (likewise `x_SystemLabels`,
`x_ProjectLabels`, `x_Tags`) and take the billing month from `BillingPeriodStart`. **On the
reference stack**, a scheduled sync copies each month to
`raw/gcp/focus/billing_period=YYYY-MM/` as Parquet, and does two things the adapter
below relies on:

- **One `tags` map.** Labels arrive as repeated key/value records in three columns:
  `x_Labels` (resource labels, GKE's `goog-k8s-cluster-name`, `goog-k8s-node-pool-name`
  and `k8s-namespace` among them), `x_SystemLabels` (`compute.googleapis.com/machine_spec`,
  …) and `x_ProjectLabels`; Resource Manager tags arrive in a fourth, `x_Tags`. The sync
  merges all four into one `map(varchar, varchar)`, project labels prefixed `project:` and
  Resource Manager tags `rmtag:`, so neither can collide with a resource label of the same
  name. The adapter then reads tags as on AWS.
- **The billing period** is the month of `BillingPeriodStart` (Google's invoice month,
  which follows Pacific time). Rows dated outside it are fine: the daily load lands
  them on the month's first or last day.

With GKE cost allocation turned on, Google splits cluster cost by namespace itself (by
requests, from the day it's enabled, and only for vCPU, RAM, GPU, TPU and
persistent-disk SKUs) and labels the rows `k8s-namespace`: the adapter passes it through
and those rows land whole on their namespace, with no metrics of yours. Four values
aren't namespaces: `kube:unallocated` (capacity nothing requested) and
`kube:system-overhead` (what the nodes hold back) contain `:`, so they read as buckets
for `shared_owner`, and the adapter turns `goog-k8s-unknown` and
`goog-k8s-unsupported-sku` (rows Google couldn't split) into NULL.

`focus_gcp_v` reads `gcp_focus`, the synced copy with its one `tags` map:

- **Passed through:** as on AWS. `servicecategory` and `x_resourcegroupname` are NULL;
  `currency` is `billingcurrency`; `provider` is `'gcp'`.
- **`tag_costcenter`:** the first non-NULL of `costcenter`, `cost-center` and
  `project:costcenter`. The project label comes last, so a resource's own label wins.
- **`first_party`:** `publishername` NULL, empty, or starting with `google` in lower
  case.
- **`shared_cost`:** NULL.
- **Kubernetes:** `k8s_cluster` from `goog-k8s-cluster-name`, `k8s_pool` from
  `goog-k8s-node-pool-name`, `k8s_namespace` from `k8s-namespace`.
- **Rows kept:** the profile's list, usually `Usage` and `Adjustment`. There are no
  `Credit` rows, and `Tax` stays out unless finance counts it.

Reporting in USD instead: divide each of the three money columns by
`x_currencyconversionrate` (local currency per USD), and `currency` is `'USD'`. Divide
only by a rate that is positive and finite. A missing rate, or any other, gives NULL
money, which the totals check reports as `unloadable_rows` instead of loading a wrong
number.

## Vendors that bill you directly

No cloud bill has a row for them, so each vendor's shim writes its invoice as FOCUS rows
into `saas_focus` (provider `other`; `saas.md`). Those rows are the bill, and their
total must match the invoice. The shim sets `publishername` to the vendor's name so the
registry can find the rows.

`focus_other_v` reads `saas_focus` and passes the columns through as on AWS
(`currency` from `billingcurrency`, `provider` `'other'`). `first_party` is always false:
the vendor sold it, not a cloud. `x_resourcegroupname`, every `tag_<key>`, `shared_cost`
and the three `k8s_*` columns are NULL. Rows kept: the profile's charge categories.

## Adding a provider

1. Its raw table, partitioned by `billing_period`, and the sync that fills it.
2. An adapter with the contract's columns in the contract's order.
3. Its branch in the stack of adapters (`bill_v`), and its value in the `provider`
   partition enum (`stack.md`).
4. Its cases: a few raw rows and the contract rows they must become
   (`cases/adapters.yaml` has the pattern), run in the user's test runner.
