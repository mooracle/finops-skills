# Discovery — scan the bill before asking anything

Run these before the interview. Each query answers a question you would otherwise
ask, and each answer lands in a named field of `finops/profile.yaml`. Show the
user the result of every query, then ask only what the data can't tell you
(`intake.md`).

Rules for every query here:

- One closed month (`{{M}}` = last full month, `YYYY-MM`). Exact-match partition
  filter, named columns, the engine's scan cap on (a workgroup byte cap on Athena). A discovery query that scans more
  than its month's files is missing its partition filter.
- Rank by **cost**, never by row count. Coverage shares use `Usage` and `Purchase`
  only: credits are negative and would make a share exceed 100%.
- Record what you ran and what it showed as a `D-NNN` entry in `finops/DECISIONS.md`
  (query id, month, the two or three numbers that mattered). That entry is the
  `evidence` the profile field points to.
- Each query below is described for `aws_focus`; write it in the user's engine. Azure
  notes follow where the shape differs, and the function map below covers the usual
  translations.

## Contents

- Before anything is built: a table to discover with
- Translating to another engine
- D0 Is there data? (export inventory)
- D1 Candidate dimensions (cardinality sweep)
- D2 Accounts and the naming convention
- D3 Tag keys by cost coverage
- D4 Period charges (does proration matter?)
- D5 Shared-cost candidates
- D6 Third-party publishers (SaaS candidates)
- D7 Kubernetes footprint
- D8 Azure resource groups
- D9 Commitments
- Refresh mode

## Before anything is built: a table to discover with

Discovery runs before the stack and the copy job (both Phase 2) exist, so query the
export where it already lands:

- **AWS**: a table over the export's own folder (`<prefix>/<export-name>/data/`,
  partitioned by its `billing_period=YYYY-MM` folders; `focus-export.md` shows the
  layout). List the folder first (D0): projection must match the folder's case
  (`billing_period=` or `BILLING_PERIOD=`), and in create-new-report mode the table reads
  only the folder the month's manifest names, never the month, or each delivery counts
  again. Name it `aws_focus` for now; Phase 2 points the real one at the mirror.
- **No engine yet** (no Athena workgroup, no Glue database, and no wish to create them
  before Phase 2): copy one month's delivery, the files its manifest lists, to a
  scratch folder and run the queries on DuckDB, after `SET TimeZone = 'UTC'`: DuckDB reads
  FOCUS timestamps (CSV text ending in `Z`, or Parquet's UTC-adjusted ones) as
  `TIMESTAMPTZ`, and a cast to `TIMESTAMP` or `DATE` moves them into the laptop's zone,
  shifting days without an error. Say in each `D-NNN` entry which engine ran the query;
  Phase 3's gate reproduces D0's total in the real engine.
- **Azure**: the export's CSV or Parquet files in the storage account, read from where
  the engine can reach them (synced once by hand, or through the engine's external
  table support).
- **Google Cloud**: the export is already a BigQuery table; run the queries there.

## Translating to another engine

Written in Trino (Athena), these queries need a few functions whose names differ by
engine:

| Trino (Athena) | BigQuery | Snowflake | Databricks (Spark SQL) | DuckDB (discovery with no engine, the trial) |
|---|---|---|---|---|
| `element_at(tags, 'k')` (map) | labels are `ARRAY<STRUCT<key, value>>`: `(SELECT value FROM UNNEST(labels) WHERE key = 'k')` | `tags:"k"::string` (VARIANT) | `tags['k']` or `element_at(tags, 'k')` | CSV export: `Tags` is a JSON string, `json_extract_string(tags, '$."k"')` (quote the key: tag keys carry `.` and `/`); Parquet: `tags` is a map, index it and check whether your version returns the value or a one-element list |
| `CROSS JOIN UNNEST(map) AS t(k, v)` | `CROSS JOIN UNNEST(labels) AS l` | `, LATERAL FLATTEN(input => tags) f` (`f.key`, `f.value`) | `, LATERAL explode(tags) AS t(k, v)` | JSON: `CROSS JOIN unnest(json_keys(tags)) AS t(k)`, then `json_extract_string(tags, '$."' || k || '"')`; map: `unnest(map_entries(tags))` |
| `regexp_like(s, p)` | `REGEXP_CONTAINS(s, p)` | `REGEXP_LIKE(s, p)` (whole string) | `s RLIKE p` | `regexp_matches(s, p)` |

On the CSV export, cast `effectivecost` and `billedcost` to `DOUBLE` and the
`chargeperiod*` columns to `TIMESTAMP` once, in a view, and run every query on the view.

## D0 — Is there data?

First list the export's folder (`<prefix>/<export-name>/data/` on AWS). Each month has one
`billing_period=YYYY-MM/` folder, in whichever case this export writes. In overwrite
mode it holds the month's current snapshot. In create-new-report mode it holds one
timestamped folder per delivery, and the month's manifest
(`metadata/billing_period=YYYY-MM/<export>-Manifest.json`, rewritten with each
delivery) lists the files of the newest: read exactly those, never the month. A folder
that isn't a `billing_period=` of this export (one written by another export or an
earlier run, say) belongs to no month and stays out of the table. Record the mode and the
case: Phase 1 and the copy job depend on both. Then, for the month, group the rows by
`billingcurrency` and `chargecategory`. Per group, count the line items and sum
`billedcost` and `effectivecost`, largest effective cost first.

Run it per provider. The currency column feeds Q-F4: show the user each provider's
billing currency side by side before asking how finance reports.

No folder → the export isn't on (`export: missing`): the bootstrap path (`SKILL.md`).
Fewer months than the user expects → note the backfill route for that provider
(`focus-export.md`). Profile: `providers[].export`, `months_available`,
`billing_currency`, `focus_version` (the export's FOCUS version, from its name or
manifest; 1.2 preferred) and `delivery` (`overwrite`, or `new_folder` for
create-new-report mode: Phase 1 adds a 1.2 overwrite export beside it,
`focus-export.md`).

## D1 — Candidate dimensions

One row per candidate column the export carries (`servicename`, `regionid` and the
rest): the column's name, its number of distinct values in the month, and up to 8
sample values.

One value everywhere = no signal. One value per resource = `ResourceId` in disguise, and it
would grow the warehouse back toward raw size. 2 to a few thousand, far fewer than
the rows, is the useful band.
Columns that move with one you already keep (`servicecategory` with `servicename`) are
nearly free. Profile: `dimensions.columns`.

## D2 — Accounts and the naming convention

Accounts by cost: per `subaccountid`, one name for it (the month's latest non-empty
`subaccountname`)
and its summed `effectivecost`, largest first.

Read the names with the user. If a convention is visible (`<team>-<app>-<env>`), write
it as a regex (`{{CONVENTION_REGEX}}`; you run discovery by hand, so a regex goes in
as written, but never with a quote in it) and measure how much **cost** follows it. Take
the month's `Usage` and `Purchase` rows and split them in two: the lower-cased
`subaccountname` matches the regex, or it doesn't. For each side, count the distinct
accounts and give its share of the cost, as a percentage of both sides together, to
one decimal. Guard the division: a month with no cost gives NULL, not an error.

Above ~70% by cost: rules can follow the convention and new accounts route themselves.
Below that: rules will be enumerations, and the convention is a conversation with
whoever provisions accounts (record it as a recommendation, not a blocker).
Profile: `routing.account_convention` (regex + `pct_cost`), `routing.accounts_shared_by_teams`.

Azure: same query on `azure_focus`; subscriptions are the accounts, and the convention
often lives in resource-group names instead (D8).

## D3 — Tag keys by cost coverage

Work on the month's `Usage` and `Purchase` rows, one line item at a time:

- **Fold each key.** Take the line's tag keys whose value isn't empty. Fold each key:
  lower case, then remove `-`, `_` and spaces (`regexp_replace(lower(k), '[-_ ]', '')`).
  Drop duplicate folded keys within the line before any cost is summed.
- **Coverage per folded key.** Sum the `effectivecost` of the lines that carry the key.
  Divide by the total `effectivecost` of all the month's `Usage` and `Purchase` lines,
  tagged or not. That's `pct_cost_tagged`, a percentage to one decimal.
- **Per folded key, also:** every original spelling seen (`spellings`), the number of
  distinct non-empty values, the number of distinct resources (non-empty `resourceid`) on
  the lines that carry it, and up to 8 sample values.
- **Filter and rank.** Keep keys with 2 to 5,000 distinct values (D1's band), so a
  two-value key like `Environment` stays and so does a cost center with a few thousand. Rank
  by `pct_cost_tagged` and show the top 40. A key with a value per resource (its distinct
  values close to its distinct resources: `Name`, say) is `ResourceId` in disguise however
  high it ranks: show it, never keep it.

A key under ~30% coverage by cost is a nice-to-have, not a routing axis. Profile:
`dimensions.tags` (key, spellings, `pct_cost`). On AWS, also list the payer account's
inactive keys (`aws ce list-cost-allocation-tags --status Inactive`, as in D7; never in
the trial, which reads only its sample): a routing key there is invisible to this query
until the user activates it and requests the backfill (`focus-export.md`). Re-run D3
once the backfill has filled the month.

Azure: `tags` is a JSON string. Turn it into a map with
`CAST(json_parse(tags) AS map(varchar, varchar))` and run the same steps; reading only the
keys you expect would miss the spellings this query exists to find.

## D4 — Period charges

Find the rows that cover more than one day. Count a row's days from the date of
`chargeperiodstart` to the date one second before `chargeperiodend`, plus one. The
end is exclusive, so a charge ending at midnight belongs to the day before. Mark
whether it ends in the month: the date of `chargeperiodend` is on or before the first
day of the next month. Keep rows covering more than one day and group them by
`servicename`, `chargecategory`, days and ends-in-month. Per group, count the line
items and sum `effectivecost`. Show the top 30 by cost.

Rows that start and end in the month get prorated across their days; any other row stays
whole on one day (`splitters.md`). If this returns nothing material, daily charts won't spike
and proration is a correctness detail, not a priority. Profile: `period_charges`.

## D5 — Shared-cost candidates

Look at the month's rows that might be shared. A row qualifies if any of these hold:

- its lower-cased `servicename` contains `support`, `on ramp` or `unified operations`
  (two of AWS's support plans bill as `AWS Enterprise On Ramp` and `AWS Unified Operations`);
- it's on a payer's own account (`subaccountid = billingaccountid`);
- its lower-cased `subaccountname` matches
  `shared|network|logging|security|platform|transit`.

Group by account name and service, sum `effectivecost`, and show the top 25.

The data shows *candidates*. Whether a cost is shared, and what it should be split by,
is the user's call (intake Q-S1). Profile: `shared_costs[]`.

## D6 — Third-party publishers

Find the month's rows sold by someone other than the provider: `publishername` is
neither NULL nor empty, and its lower-cased value doesn't start with
`amazon web services`, `pt amazon web services` or `amazon aws` (AWS's own sellers of
record, the `first_party` rule in `providers.md`). Group by publisher, account name, service and charge category.
Per group, give `days_billed` (the number of distinct `chargeperiodstart` dates),
summed `billedcost` and summed `effectivecost`, largest billed first.

Each row is a SaaS candidate. `days_billed` already tells you the invoice shape: ~30
means daily usage rows, 1 means a monthly installment or a one-off charge. Profile:
`saas.vendors[]`: `provider`, `monthly_cost`, `facts.where_billed` and
`facts.invoice_shape` (publisher pattern, account).

A publisher whose name has the word `Amazon` or `AWS` in it but none of those prefixes
is probably a seller of record the rule doesn't know yet (AWS adds and renames them):
show it to the user before treating it as a vendor.

A Bedrock model its maker sells (Anthropic's, for one) shows up here under the maker's
name. It's Bedrock usage, not a SaaS vendor: check its rows carry one of the Bedrock
markers in the `first_party` rule (`providers.md`), and if they don't, show the user
before the adapter is written, or the support weights leave that spend out.

Azure: Marketplace charges carry `publishername` too; vendors billed directly (their own
invoice) don't appear on any cloud bill at all, so ask (intake Q-V1).

## D7 — Kubernetes footprint

Tag names differ by distribution, installer and org, so search for them instead of
assuming. Unnest every tag of the month's rows and keep the keys whose lower-cased
name matches `eks|aks-managed|kubernetes|k8s|nodegroup|nodepool|poolname|cluster|karpenter|eksctl`. Per key,
give the number of distinct values, the number of distinct resources (`resourceid`),
the summed `effectivecost` and up to 8 sample values, by cost.

On AWS, run the same pattern over the cost allocation tag keys the payer account knows
but doesn't export: `aws ce list-cost-allocation-tags --status Inactive` (and
`--status Active` for the ones it does). A tag reaches the export only once it is
active, so a cluster whose keys sit on the inactive list is invisible to this query
however well its nodes are tagged: EKS Auto Mode clusters are the usual case, since
AWS writes `eks:eks-cluster-name` and `eks:kubernetes-node-pool-name` on every node it
launches and activates neither. Once the cluster's keys are picked (below), set its
`tags.activated`: `true` when every tag key picked for it is on the active list, else
`false`, and record the inactive ones. D7 always answers it on AWS: the checker blocks
the cluster while it's null or missing. Activating them is the user's change to make, in
the payer account, with the backfill request (`focus-export.md`); a closed month carries
them only once the backfill lands, so re-run D7 then. Azure tags and Google Cloud labels
reach the export with no activation step, so `tags.activated` doesn't apply off AWS.

From the output, pick (with the user) the keys that name the **cluster**, the key that
names the **node pool**, and the keys that name a **namespace** on disks and load
balancers. The cluster comes in several forms, one per kind of resource (the table
below): look for all of them, then split the cluster's spend by pool and by the form
that found it. The query takes the cluster's name (`{{CLUSTER}}`) and the pool key
picked above (`{{POOL_TAG}}`). Give each of the month's rows a `found_by` label: the
first of these forms that names the cluster, checked in this order.

| `found_by` | The row names the cluster when |
|---|---|
| `node tag` | `aws:eks:cluster-name`, `eks:cluster-name` or (Auto Mode) `eks:eks-cluster-name` equals the name |
| `lb tag` | `elbv2.k8s.aws/cluster` equals the name, or (Auto Mode) `ingress.eks.amazonaws.com/stack` or `service.eks.amazonaws.com/stack` names a namespace of it |
| `csi tag` | `KubernetesCluster` equals the name |
| `owned key` | the key `kubernetes.io/cluster/<name>` has the value `owned` |
| `eks arn` | `resourceid` is the control plane's ARN, `arn:aws:eks:<region>:<account>:cluster/<name>` |

Read tags with `element_at`, so a missing key gives NULL rather than an error. Drop the
rows no form names. Group the rest by account, region, `found_by`, pool (the pool
tag's value, NULL when absent) and service. Per group, count distinct resources and sum
`effectivecost`, largest first.

A pool named after one namespace is **dedicated** (tag routing, no metrics). A pool
many namespaces share is **shared** (needs metrics). Cluster spend outside any pool
(boot disks without the pool tag, CSI volumes, load balancers, the control plane) shows
up as `pool IS NULL`: note its size and which forms found it, because the adapter reads
exactly those. A load balancer whose namespace tag belongs to an ingress controller
(`ingress-nginx`, say) is a **shared ingress** (Q-K5).
The account and region columns are each cluster's identity: one cluster name in two
accounts or two regions is two clusters, with pools of their own.

Last, count the namespaces that run workloads: `kubectl get pods -A` read-only, system
and agent namespaces aside, or, without cluster access, the distinct namespace tags on
the cluster's disks and load balancers. One workload namespace means the track has
nothing to split (`plan.md`, `k8s:<id>/tags`): say so before anyone activates tags for
it. Per-app cost needs one namespace per app first, the cluster team's change; record
the suggestion as a `D-NNN` entry.
Profile: `kubernetes.clusters[]` (account, region, tags and whether they're active,
pools, untagged share, `workload_namespaces`).

## D8 — Azure resource groups

On `azure_focus`, per subscription name, count the distinct `x_resourcegroupname`
values and sum `x_effectivecostinusd` as `cost_usd`, largest first.

If `cost_usd` comes back NULL, the rows carry no USD amounts: group by
`billingcurrency` instead and never add amounts across currencies.

Subscriptions with many resource groups and several owners are where the resource
group becomes a routing key. Run the D2 convention test on `x_resourcegroupname` too.
Profile: `routing.azure_resource_groups`.

## D9 — Commitments

Group the month by `chargecategory`, `pricingcategory`, `commitmentdiscounttype` and
`commitmentdiscountstatus`. Per group, sum `billedcost`, `effectivecost` and `listcost`,
largest billed first.

The `Unused` rows are commitment waste, in `effectivecost` only (`billedcost` is 0). The
adapters name their service `Unused commitment` (`providers.md`), and it lands on the
account that bought the commitment or, on Azure, whose unused rows name no subscription,
on the billing account. Whoever owns that account carries it; until a rule covers the
account, it shows in `Unallocated` like any other cost. Check two things on the user's
bill before the adapters are written:

- **AWS:** `subaccountid` on the `Unused` rows is the account in `commitmentdiscountid`
  (the ARN's fifth `:` field). Show the user any row where it isn't.
- **Azure:** an `Unused` group exists. If `commitmentdiscountstatus` is empty on every
  row, confirm the adapter's fallback (`providers.md`) finds the unused rows.

Show the user the gap between `billed` and `effective` on `Purchase` rows: it is the
evidence for chargeback on `EffectiveCost` (invariant 3), and there is nothing to
build. Profile: `finance.chargeback_column` (always `effectivecost`; the evidence is
this query).

## Refresh mode

After a re-org, a new account family, a new vendor or a new cluster, re-run D2, D3, D6
and D7 for the latest month and diff against the profile. Re-ask only the fields that
moved, and append the diff as a new `D-NNN` entry.
