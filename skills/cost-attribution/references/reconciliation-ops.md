# Phase 5 — Reconciliation, operations, the frozen close

Goal: the pipeline proves its own numbers on every run and, where the org enters invoice
totals (Q-F5), against the invoice once a month, finance has a month that never moves if
it freezes months (Q-F1), and running it is boring (article Part 2, "Rebuild the whole
month and check it against the invoice", and Part 4, "Old months change on purpose").

## Contents

- The checks, and what each one fails on
- Tolerances
- The invoice check
- Freeze the close
- Alarms
- Failure modes
- Wrap-up checklist

## The checks, and what each one fails on

Every check is SQL that returns **zero rows when healthy**. Blocking checks fail the run
(`pipeline.md`, "One run" and "Failure handling").

| Check | When | Blocks | Catches |
|---|---|---|---|
| weights check | after the weights, before any base clear | yes | a split key whose weights don't add up to 1.0; a weight that is NULL, not finite or outside (0, 1]; `bill_v` carrying more rows than the adapter (a join matched a row twice); which registry entries overlap; a `shared_cost` id the profile doesn't list (`shared_cost_not_configured`, decided in `pipeline.md` step 1 before `split_weights` is cleared, so the previous weights stay; the run raises it as `weights`) |
| totals check | end of every run | yes | bill ≠ monthly per payer and cost column; monthly ≠ daily once the month is over; bill rows the loads can't place or sum right (`unloadable_rows`, whatever their dates: no start, NULL or non-finite money, no currency; a non-finite amount loads as it is, so `bad_money` reports it on the warehouse side too); daily rows outside their month or from today; an empty month (no adapter row for the month, whatever its dates); non-finite warehouse money; mixed currencies |
| views check | end of every run, once the views exist (Phase 4); after each lookup publish, over every month | yes, but clears nothing | `services_monthly` off `base_monthly`, per payer and currency, in a cost total or the row count: a lookup key published twice |
| invoice check (below) | monthly, once the month's invoices are in (Q-F5); never where the org enters no invoice totals | the close only | raw `BilledCost` off the invoice totals the org entered |
| coverage report (`kubernetes.md`) | Phase B gate, monthly | no | a cluster below its coverage target |
| vendor report (`saas.md`) | weekly | no | vendors left unsplit, and why |
| worklist (`rules-engine.md`) | weekly | no | the top tuples no rule reaches |
| close readiness (below) | before each close | the close only | the views check and, where the org enters invoice totals, the invoice check, run again; whatever else the org requires before freezing |

The bill side of the totals check reads the adapter, filtered the way the loads see it,
before any join, with nothing carved out. Per payer, because no split crosses payers and
two payers' errors could cancel out.

The views check compares `services_monthly` with `base_monthly` per payer and currency:
each cost total within the tolerance (`view_off_base`, per cost column; a payer or
currency on one side only, an empty month, or a NULL or NaN difference fails too) and
the row count (`view_rows_off_base`). In a run, an empty month passes when no adapter row
of the month starts before today. A lookup key published twice doubles every row it
joins, and when it doubles a charge and its credit the sums still agree: only the count
shows it. The daily view joins the same lookup on the same key, so the monthly one
answers for both. After a lookup publish it runs over every month at once, an
all-history read of `base_monthly` (invariant 9). A failure fails the run, or the
lookup's event, and clears nothing: the totals check has proved the base tables, and the
fault is in the lookup, which every month reads.

A blocking check returns one row per problem, so the run can stop with all of them at
once: `problem` (the name, as the cases and the failure table below use it), `subject`
(what it's about: the payer, the weight key, the provider, the overlapping registry
entries; NULL for a month-wide count), `cost_column` for the per-column totals, and the
numbers that disagree, as `expected` (the side the check trusts: the bill against
`base_monthly`, `base_monthly` against `base_daily` or the views) and `actual`. The run's
error carries the check's name (`weights`, `totals` or `views`) and these rows.

## Tolerances

- **5 per payer and cost column, in the month's currency** (`TOLERANCE`): the same
  numbers added in a different order across millions of rows. Anything bigger is a
  regression. Scale it for a currency with small units (500 for JPY, say). Tests run at
  0.01.
- **1e-6** on weights summing to 1.0.

## The invoice check

The totals check proves the warehouse kept raw, not that raw is what the org paid
(article Part 2). The invoice check compares raw with the invoice totals the org enters,
once a month, per provider. Only the org has those totals (Q-F5). Where it enters none
(`finance.invoice_totals: none`), the check is SKIPPED: nothing is scheduled or stored,
the Phase 5 gate and close readiness drop their invoice items, nothing blocks, and a
note with the gate says that nothing ties the bill to the invoice.

- **When.** From the day the month's invoices usually arrive (Q-F5,
  `finance.invoice_totals.day`), a scheduled job sends one event per provider to the ETL
  function (`{"invoice_check": "YYYY-MM", "provider": "aws"}`), so it never reads raw
  mid-swap, and sends it again on each scheduled run until that month's check passes.
- **The invoices.** `invoice_totals` (`provider, billing_period, invoice, currency,
  amount`): a reference table edited in Git, never read from the export. One row per
  thing the check compares, as `providers.md` ("Currency") says per provider: in
  `invoice`, the invoice id where the export has `InvoiceId`, else the billing account
  (Google Cloud, EA) or the vendor's `subaccountid` (`other`); in `amount`, the total
  `providers.md` names (MCA's before tax and Azure credits, EA's Total Charges, Google
  Cloud's invoices plus memos). Whoever Q-F5 names enters each month's rows, or pulls
  them from the provider's invoice API where there is one.
- **What it compares.** One exact-month read of the provider's raw table, never the
  adapter (its `billedcost` is converted and filtered): `BilledCost` in the billing
  currency, every charge category, summed per `invoice` as above, against
  `invoice_totals`, within the tolerance.
- **What it returns.** The report (`invoice_check`) takes the month
  (`{{BILLING_PERIOD}}`), the provider (`{{PROVIDER}}`) and `TOLERANCE`. One row per
  problem, `problem`, `subject` (the `invoice`), `currency`, `billed` and `invoiced`:
  `invoice_off` (the two differ by more than the tolerance, or the difference is NULL
  or NaN), `invoice_missing` (raw carries an `invoice` that `invoice_totals` lacks),
  `invoice_not_on_bill` (the reverse).
- **Stored with the month.** `invoice_check/<provider>/<YYYY-MM>.csv`: every `invoice`
  with both sums and its problem (empty when it matched), the published delivery it read
  (empty for `other`) and when. Each check of the month replaces it.
- **On a problem.** An alarm ("Alarms"); whoever enters the totals (Q-F5) looks into it.
  Nothing is cleared: the base tables still match raw. The month doesn't close until it
  passes ("Freeze the close").

## Freeze the close

The pipeline restates history on purpose: corrections arrive, rules improve, a re-run
changes March. If finance has reported March, it may need March to stay put. Whether it
does, when, at what grain, and what must be true first depends on how the org charges
teams: a showback report, chargeback entries per cost center, budgets against actuals,
an allocation it already runs. Ask (Q-F1); never assume a calendar or a process. When
the answer is a frozen month:

- **A scheduled job writes `close/<provider>/<YYYY-MM>.csv`** on the org's schedule,
  from `services_monthly`, at the grain finance reports on. Retry on each scheduled run
  until written, one event per provider, so one provider's late data never holds up
  another's close. Alarm when the org's deadline passes without it.
- **Only from a month that is ready, never mid-rebuild.** Each event goes to the ETL
  function itself, so the close runs serialized with rebuilds (concurrency 1). For its
  provider it checks, right then:
  1. raw holds the export's latest delivery: no newer one is staged or waiting
     (`pipeline.md`, "Delivery");
  2. the totals check returns nothing: a check from an earlier run proves nothing
     about a partition a later rebuild is halfway through;
  3. the views check returns nothing for the month;
  4. where the org enters invoice totals (Q-F5), the invoice check returns nothing for
     the month, run again right then: a later delivery may have moved raw since it
     passed;
  5. whatever else the org's process requires before freezing (Q-F1): for example
     vendor charges split (a vendor-keyed `bill_v` row of the month with no weights in
     `split_weights` hasn't), a person's sign-off. Check only what the org asked for.
- **Not ready is not an error.** Write nothing, log why, and let the next scheduled
  run try again.
- **Freezing despite a known problem is a decision** by whoever the org named (Q-F1),
  recorded as a `D-NNN` entry, through whatever override the build gives them.
- **Write once.** If the file exists, the job does nothing: a retry or a later run never
  overwrites a close. Republishing a month is a deliberate human act (delete, re-run,
  with a `D-NNN` entry saying why).
- A close that fails (an error, not "not ready") lands in the ETL's DLQ.
- **Frozen and live drift apart on purpose.** AWS may restate a month for two weeks
  after it ends, and Azure reruns the previous month in the first five days.

**Budgets**, if the org wants them next to actuals: whoever keeps them edits a wide sheet (one
column per month) in Git; publish it long (`billing_period, team, currency,
budget_amount`: the reporting currency, or one per currency under `per_provider`). A
missing budget is NULL, never 0. Join budgets to aggregated actuals, never through the
rules lookup (it multiplies them).

## Alarms

- **Scheduled pollers** (the Azure poller, the GCP sync, vendor shims, metrics syncs) rebuild their
  window on the next run: an **error alarm**, no DLQ.
- **Event-driven functions** (the copy Lambda, the ETL, which also runs the close): a
  **DLQ** plus an alarm on its depth.
- **Unallocated** above `finance.unallocated_alert` for the last month, in its currency.
  Missing data counts as breaching: a metric that stopped arriving is not good news.
- **A close still missing** past the org's deadline (Q-F1).
- **An invoice check that doesn't pass**, on each scheduled run until it does. Whoever
  enters the totals (Q-F5) looks into it.
- Every alarm goes to a channel the owner (`org.owner`) reads.

## Failure modes

| Symptom | Likely cause | Where to look |
|---|---|---|
| `monthly_off_bill` after a deploy | a load filters rows the adapter keeps (or the reverse) | the loads' `WHERE`; what counts as cost belongs in the adapter only |
| Warehouse above the bill | weights that don't add up to 1.0 got past their check, or a load lost its `COALESCE(w.weight, 1.0)` | the weights check, `splitters.md` |
| `bill_rows_added` | a join in `bill_v` matched a row twice | `registry_overlap` names the entries |
| `daily_off_monthly` on a closed month | the daily load filters or splits differently from the monthly one | diff the two loads' filters and fan-out (`bill_v` joined to `split_weights`); they must match line for line |
| `weights_sum` or `weight_range` | a producer lost its normalization, or wrote a zero or negative weight | the producer named by the key's prefix (`share:`, `k8s:`, `vendor:`) |
| `shared_cost_not_configured` | the adapter marks a shared charge whose id the profile doesn't list (a typo, or a new shared cost) | `shared_costs[].id` against the adapter's `shared_cost` |
| Warning `shared_cost_not_on_bill` | the profile lists a shared cost no row carries this month: the adapter's predicate has a typo, or the charge didn't come. A warning, because some months legitimately have none | the adapter's `shared_cost` line against D5 |
| `no_bill_rows` | a wrong period, or the export or copy hasn't landed | the provider's raw partition for the month |
| `bad_money` | a load produced NULL or non-finite money (a division by zero, NaN in raw) | the load, then `unloadable_rows` for the same run |
| Warehouse below the bill by about one day of one charge | proration day count | the one-second trick |
| `unloadable_rows` | rows with no start date or NULL money: an export without the USD columns the adapter reads, or a month missing from the FX rate table | the adapter, `fx_rates` |
| `day_outside` | a charge dated outside its billing period escaped the clamp, or one dated today or later escaped the today cap | the daily load's clamp to the month's first or last day (`splitters.md`, Proration), and its today cap |
| `currencies` | an adapter stopped normalizing | the adapter's cost columns |
| Numbers "wrong" against the Azure portal | comparing FOCUS `ServiceName` with the portal's `x_SkuMeterCategory` | `providers.md`, "Azure" |
| Today's number keeps growing | a new query without the today cap | `chargeperiodstart < DATE '{{TODAY}}'` |
| A month looks half empty | read mid-rebuild, or a failed INSERT that didn't raise | the runner must raise on `FAILED` (then the rebuild clears both partitions); consider staging + swap |
| Unallocated creeping up | new accounts outside the naming convention | fix the convention, not just the rule |
| A month short, and every check green | a copy into raw lost a file or kept an old one: the totals check compares the warehouse with raw, not with the export | the delivery's manifest against the live partition (`pipeline.md`, "Delivery") |
| `view_off_base` or `view_rows_off_base`: a run or a lookup publish fails, and no close is written | the lookup holds a key twice (a stray file under `rules/lookup/`, a hand edit) | `SELECT lookup_key, COUNT(*) FROM rules_lookup GROUP BY 1 HAVING COUNT(*) > 1` |
| Close not written: a vendor hasn't split | the vendor's data is waiting, missing or unusable | the vendor report (`saas.md`); the org's override if it won't come |
| `invoice_off` | the total entered isn't the one `providers.md` says to compare for that provider, raw doesn't hold the export's latest delivery yet, or the month was restated after the invoice | `providers.md`, "Currency"; the month's published marker |
| `invoice_missing` or `invoice_not_on_bill` | an invoice not entered yet, or entered under another id, month or currency | `invoice_totals` against the month's invoices in raw |

## Wrap-up checklist

- [ ] Every blocking check wired into the run. Prove it on a dev copy: break a template
      the way a bug would (drop the weight from the fan-out) and watch the run fail
      and clear both partitions. Editing a warehouse row by hand
      doesn't prove it: the next run rebuilds the row; run the totals check alone to
      see that drift reported
- [ ] The invoice check scheduled for every provider, and a closed month passed it: each
      invoice's raw `BilledCost` next to its total, the result stored with the month
      (part of the Phase 5 gate). Where the org enters no invoice totals (Q-F5): a
      `D-NNN` that says nothing ties the bill to the invoice
- [ ] Unallocated alert at the agreed threshold, to a channel someone reads
- [ ] Dead-letter queues (or the engine's equivalent) on event-driven runs and the close
      job; error alarms on pollers
- [ ] The engine's query costs reviewed (on Athena: workgroup scan metrics; the whole
      reference stack runs about $10 a month at 15 million bill lines a month)
- [ ] If the org freezes months: the first close written from a ready month; whoever
      reports from it has it, and knows who may freeze a month despite a known problem, and how
- [ ] `STATUS.md` shows every phase passed with a SHA; `DECISIONS.md` names the owner,
      the finance partner if there is one, and every negative decision
- [ ] The finance partner, or the owner where there is none, walked through
      `services_monthly` and whatever cross-check the org's process uses (part of the
      Phase 5 gate)
