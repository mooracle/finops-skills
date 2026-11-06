# Trial — see it work on a sample bill first

For someone who wants to see the build before giving an agent billing access: a
synthetic month of an AWS bill, shaped like a real one, run end to end on the laptop
with no credentials, and checked against an answer key written below. It takes one
short session. The trial is a demonstration, not Phase 0: nothing it writes or decides
carries into a real build.

## Contents

- When to offer it
- What it needs
- The sample org
- The sample month
- Discovery should show
- Trial answers
- What to build
- The checks the user sees
- The answer key
- Switching to a real bill

## When to offer it

- The user asks to try it first, see a demo, or run it on sample or fake data.
- The user won't, or can't yet, give the agent access to billing data.
- No export exists yet (the bootstrap path, `SKILL.md`): offer the trial while the
  first delivery lands.

Say what it is up front: a made-up org called Sample Org, one month, AWS only, on DuckDB.
The only data the model sees is the sample. Then run it without stopping for questions:
every answer the interview would need is in "Trial answers".

## What it needs

Python 3 with DuckDB and PyYAML, in a throwaway environment. Network only to install them.
Everything goes in `finops-trial/` in the current directory, never in `finops/`:

```
finops-trial/
├── make_sample.py                 writes the sample month from "The sample month"
├── raw/focus/AWS/focus-1-2/data/BILLING_PERIOD=2026-09/sample.parquet
├── trial.duckdb                   the warehouse
├── profile.yaml, DECISIONS.md, STATUS.md   Sample Org's, from the template and the trial answers
├── rules/*.json                   Sample Org's rules
├── run.py (or SQL files)          adapter, weights, loads, checks, expansion, views
└── …                              whatever else the build needs (the inventory, the lookup)
```

The trial's "today" is fixed at `2026-10-05`, so September is closed and every run gives
the same numbers. No query reads the clock. Date the DECISIONS entries with it too, and
leave STATUS's SHA column as `—`: the trial never commits. The tolerance is 0.01.

The queries in the specs are Trino. `testing.md` ("What a local engine can't prove")
lists what differs in DuckDB; the three that bite here are `regexp_replace`, which needs
the `'g'` option for D3's fold, the partition column, which reads as
`BILLING_PERIOD`, and the session time zone: `SET TimeZone = 'UTC'` before reading the
sample, or its timestamps shift into the laptop's zone (`discovery.md`, "No engine yet").

## The sample org

One payer, `100000000000` (`org-payer`), and six accounts. Names follow
`<team>-<app>-<env>`, as the regex `^[a-z]+-[a-z]+-(prod|dev)$`, except two:

| `SubAccountId` | `SubAccountName` |
|---|---|
| `100000000000` | `org-payer` |
| `111111111111` | `payments-checkout-prod` |
| `111111111112` | `payments-ledger-dev` |
| `222222222221` | `search-indexer-prod` |
| `333333333331` | `data-lake-prod` |
| `444444444441` | `sandbox` |

## The sample month

Write `make_sample.py` from this section and nothing else, so the answer key stays
independent of the build. One Parquet file, FOCUS 1.2 column names, 365 rows.

**Daily usage**: one row per resource per day, 2026-09-01 to 2026-09-30.
`ChargePeriodStart` is the day at `00:00:00` and `ChargePeriodEnd` the next day at
`00:00:00`. `ChargeCategory` `Usage`, `ChargeFrequency` `Usage-Based`,
`PublisherName` `Amazon Web Services, Inc.`. Amounts are per resource per day.

| Account | `ServiceName` | `ServiceCategory` | Resources (`ResourceId`) | Billed | Effective | List | `PricingCategory` | Tags |
|---|---|---|---|---:|---:|---:|---|---|
| `payments-checkout-prod` | Amazon Elastic Compute Cloud | Compute | `i-checkout-1` … `i-checkout-4` | 0 | 75 | 100 | `Committed` (`CommitmentDiscountType` `Reservation`) | `Application=checkout`, `Environment=prod` |
| `payments-checkout-prod` | Amazon Relational Database Service | Databases | `db-checkout` | 100 | 100 | 100 | `Standard` | `Application=checkout`, `Environment=prod` |
| `payments-ledger-dev` | Amazon Elastic Compute Cloud | Compute | `i-ledger-1` | 50 | 50 | 50 | `Standard` | `Application=ledger`, `Environment=dev` |
| `search-indexer-prod` | Amazon OpenSearch Service | Analytics | `search-a`, `search-b` | 100 | 100 | 100 | `Standard` | `Application=indexer`, `environment=prod` (lower-case key) |
| `data-lake-prod` | Amazon Simple Storage Service | Storage | `data-lake` | 200 | 200 | 200 | `Standard` | `Application=lake`, `Environment=prod` |
| `data-lake-prod` | AWS Glue | Analytics | `glue-lake` | 100 | 100 | 100 | `Standard` | `Application=lake` |
| `sandbox` | Amazon Elastic Compute Cloud | Compute | `i-sandbox-1` | 20 | 20 | 20 | `Standard` | none |
| `org-payer` | AWS CloudTrail | Management and Governance | `trail-org` | 30 | 30 | 30 | `Standard` | none |

**Five more rows**, none with tags:

| Account | `ServiceName` | `PublisherName` | `ChargeCategory` | `ChargeFrequency` | Period (start → end) | Billed | Effective | List |
|---|---|---|---|---|---|---:|---:|---:|
| `payments-checkout-prod` | Amazon Elastic Compute Cloud | AWS | `Purchase` | `Recurring` | 09-01 00:00:00 → 10-01 00:00:00 | 9,000 | 0 | 9,000 |
| `org-payer` | AWS Support (Enterprise) | AWS | `Usage` | `Recurring` | 09-01 00:00:00 → 10-01 00:00:00 | 6,000 | 6,000 | 6,000 |
| `data-lake-prod` | Metrics Co Observability | `Metrics Co` | `Usage` | `Recurring` | 09-01 00:00:00 → **09-30 23:59:59** | 1,500 | 1,500 | 1,500 |
| `search-indexer-prod` | Amazon OpenSearch Service | AWS | `Credit` | `One-Time` | 09-15 00:00:00 → 09-16 00:00:00 | −500 | −500 | −500 |
| `org-payer` | AWS CloudTrail | AWS | `Tax` | `Recurring` | 09-01 00:00:00 → 10-01 00:00:00 | 1,000 | 1,000 | 1,000 |

The purchase row is the reservation's monthly fee (`PricingCategory` `Standard`,
`CommitmentDiscountType` `Reservation`, `ResourceId` `ri-checkout`, `ServiceCategory`
`Compute`). The other four have no `ResourceId`; Support and Metrics Co are `Standard`
and the credit and tax have no `PricingCategory`; the `ServiceCategory` of Support
and Metrics Co is `Other`, and the others take their service's. "AWS" means `Amazon Web Services, Inc.`. The
Metrics Co row ends at `23:59:59` on purpose: without the one-second trick
(`splitters.md`) either it or the rows ending at midnight get a day count one off.

**Every row also carries:** `ProviderName` `AWS`, `BillingAccountId` `100000000000`,
`BillingAccountName` `org-payer`, `BillingPeriodStart` 2026-09-01, `BillingPeriodEnd`
2026-10-01, `BillingCurrency` `USD`, `RegionId` `eu-west-1`, `InvoiceId`
`INV-2026-09-001`, `InvoiceIssuerName` `Amazon Web Services EMEA SARL`, and `Tags` as
`MAP(VARCHAR, VARCHAR)` (an empty map where there are none). `CommitmentDiscountType`
is NULL except on the reservation's rows, and `CommitmentDiscountStatus` is `Used` on
its usage rows and NULL everywhere else: the sample has no unused hours. Timestamps are UTC. The
`BILLING_PERIOD=2026-09` folder is the partition, in the case CUR 2.0 writes (a FOCUS 1.2 export writes
`billing_period=`), so the build must read the case from the folder, as on a real bill
(`focus-export.md`).

The sample's invoice, `INV-2026-09-001`, says **38,000.00**.

## Discovery should show

Run discovery (`discovery.md`) on the file where it lands, as on a real bill: D0 to D6
and D9 (D7 and D8 find nothing here). Show each result. Before anything is built,
check the sample against these; a mismatch is a bug in `make_sample.py`, so fix it there:

| Query | Shows |
|---|---|
| D0 | 365 rows, all USD. `Usage` 362 rows, billed 28,500, effective 37,500; `Purchase` 1, 9,000 / 0; `Tax` 1, 1,000 / 1,000; `Credit` 1, −500 / −500 |
| D2 | by cost: `payments-checkout-prod` 12,000, `data-lake-prod` 10,500, `org-payer` 7,900, `search-indexer-prod` 5,500, `payments-ledger-dev` 1,500, `sandbox` 600. The convention covers 4 accounts and 80.0% of `Usage` and `Purchase` cost |
| D3 | `Application` on 76.0% of cost, `Environment` on 68.0% (the Glue row carries only `Application`). `Environment` has two spellings, folded into one key, and two values |
| D4 | four rows over 30 days, all ending in the month (the purchase, Support, Metrics Co, tax) |
| D5 | on `org-payer`: Support 6,000 and CloudTrail 1,900 (its tax included). Only Support is shared (Q-S1) |
| D6 | Metrics Co on `data-lake-prod`: one row, billed 1,500 |
| D9 | the purchase row billed 9,000 and effective 0; the covered EC2 usage billed 0, effective 9,000, list 12,000 |

## Trial answers

Show the interview as it would run (`intake.md`), one round per topic, with each
question's trial answer filled in. These are Sample Org's answers, made up for the
sample. They are never defaults, and never go into a real profile.

| Question | Sample Org's answer |
|---|---|
| Q-O1 | the platform team runs it |
| Q-O2 | yes: Sample Finance |
| Q-O3 | `rules_in_git` |
| Q-R0 to Q-R4 | DuckDB on local files, no IaC, no dashboard (the trial's, whatever the real ones will be) |
| Q-C1 | `team` and `environment` |
| Q-G1 | the convention is policy; the platform team provisions accounts |
| Q-G2 | no account is shared by teams |
| Q-S1 | Support is shared: spread by first-party spend, same month |
| Q-V1, Q-V2 | no vendor bills directly; Metrics Co is used by the data team alone |
| Q-F1 | a showback report, no frozen month |
| Q-F2 | alert above 400 USD of Unallocated a month |
| Q-F3 | `Usage`, `Purchase`, `Credit`; tax stays out |
| Q-F4 | USD |
| Every other question | its default, logged as an assumption |
| Owners | accounts by the convention (`payments-*`, `search-*`, `data-*`; `-prod` is Production, `-dev` Development); `org-payer` is the platform team's, Production |

Write `profile.yaml` from the template with these (Metrics Co goes in `saas.vendors`:
`billed: marketplace`, `shape: one_team`, `used_by_teams: 1`), a `D-NNN` per answer, and
give the verdicts `plan.md` assigns. Fields no answer names take the sample's own facts:
the provider is `export: enabled`, `focus_version: "1.2"`, `delivery: overwrite`,
`location` the sample folder; `runtime.account` and `runtime.region` are `none` (it
runs on a laptop); `period_charges.material` is what D4 showed; Metrics Co needs only
`facts.where_billed`, since NOT_A_FIT asks for nothing more. Number the `D-NNN` entries
yourself; the trial fixes none. Write no checker; the verdicts come out: `org`, `export:aws`,
`stack`, `base-tables`, `proration`, `shared:support`, `rules` and `reconciliation`
READY; `saas:metrics-co/split` NOT_A_FIT (one team, so a rule routes it); `frozen-close`
SKIPPED.

## What to build

The trial runs Phases 3 and 4 and the checks of Phase 5 from their specs, in DuckDB,
reduced to what the sample needs. Tables are DuckDB tables; clearing a partition is a
`DELETE` of its `(billing_period, provider)`.

1. **Phase 3** (`providers.md`, `data-model.md`, `splitters.md`, `pipeline.md` "One
   run"): `aws_focus` over the Parquet, the AWS adapter, `bill_v`, `split_weights` with
   `account_weights` for `support`, the weights check, `base_monthly`, `base_daily`, the
   totals check. Route on the default key; no tag columns.
2. **Phase 4** (`rules-engine.md`): Sample Org's rules (four following the convention,
   since patterns don't capture: payments prod, payments dev, search prod, data prod;
   and one for `org-payer`; Metrics Co rides on `data-lake-prod`'s rule), the inventory,
   a minimal `rules check` and `rules expand` (patterns, specificity, the key recipe), a
   published lookup to merge into, `services_monthly` and `services_daily`, and the
   worklist.
3. **The loop.** Show the worklist: `sandbox` is the only Unallocated account (two
   tuples, its EC2 and its share of Support). Name who you
   would ask about its owner, then give the trial answer without waiting (the platform
   team, environment `Sandbox`), add the rule, expand again, and show the view move with
   nothing in `base_*` rebuilt.
4. **Phase 5 checks** (`reconciliation-ops.md`): the views check (the view
   serves exactly the base table); the invoice check (raw `BilledCost` for
   `INV-2026-09-001`, every charge category, against the 38,000.00 above, entered as its
   `invoice_totals` row); then break it on purpose: in the fan-out step both loads
   read, stop multiplying by the weight, so each of the six fanned-out rows carries the
   whole 6,000 (that is where 67,000 comes from), run, watch the totals check stop the
   run and clear both partitions, restore it and run again.

Skipped, and said so at the end: the plan checker, porting the golden cases, deliveries
and manifests, infrastructure, CI, alarms, the close, and the SaaS and Kubernetes
tracks. A real build does all of them.

## The checks the user sees

```
Trial: Sample Org, 2026-09, synthetic, no credentials
Gate 3 (base tables), 2026-09, aws: PASSED
  bill 37,000.00 · monthly 37,000.00 · daily 37,000.00 · drift 0.00   (effective)
  billed 37,000.00 and list 49,000.00, same on all three
  support: 1 row → 6 accounts, weights sum 1.000000
Invoice INV-2026-09-001: raw billed 38,000.00 = invoice 38,000.00 (tax 1,000.00 stays out)
View = base: services_monthly 37,000.00, same row count as base_monthly
Broken on purpose (no weight in the support fan-out): totals check STOPPED the run,
  monthly 67,000.00 vs bill 37,000.00; both partitions cleared. Restored: PASSED
```

Then the answer key, expected next to what the view returned, row by row.

## The answer key

`services_monthly`, 2026-09, `effectivecost` summed, in USD. Compare the view with these
numbers as written here, never with a total computed from the build's own tables.

**Before the sandbox rule:**

| `team` | Effective |
|---|---:|
| payments | 16,200.00 |
| data | 12,300.00 |
| search | 6,700.00 |
| platform | 1,080.00 |
| Unallocated | 720.00 |
| **total** | **37,000.00** |

**After it:** platform 1,800.00 and Unallocated gone; the rest unchanged. By
`environment`: Production 34,480.00, Development 1,800.00, Sandbox 720.00.

**Along the way:**

- `split_weights` for `share:support:100000000000`: `payments-checkout-prod` 0.40,
  `data-lake-prod` 0.30, `search-indexer-prod` 0.20, `payments-ledger-dev` 0.05,
  `org-payer` 0.03, `sandbox` 0.02. The Metrics Co charge and the credit earn no weight.
- `base_daily`: 1,250.00 effective every day except 09-15, which is 750.00. The
  reservation's 9,000 billed lands as 300.00 a day and Metrics Co as 50.00 a day.

Every line matches → the trial passed. Any line off → say which, find the cause with the
spec's failure table, and fix the build, not the key.

## Switching to a real bill

Nothing carries over, code included. Delete `finops-trial/`, or keep it away from the
real build. The real build starts at "Discover before asking" (`SKILL.md`) in
`finops/`, against the user's bill and in their engine, and asks every question that
has no default for real.
