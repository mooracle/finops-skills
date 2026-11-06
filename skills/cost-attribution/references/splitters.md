# Phase 3 — Splitters: when one row should be many

Built with the base tables in Phase 3: a charge covering several days, or one every
team relies on, becomes many rows with different keys and the **same total** (article
Part 3).

| Axis | Rows | Weights | Written by |
|---|---|---|---|
| Time (proration) | charges covering several days | `1 / day_count` | the daily load |
| Accounts (fair-share) | a shared charge on one account | each account's first-party spend | `account_weights` |
| Namespaces | cluster resources | Kubernetes requests | `namespace_weights` (`kubernetes.md`) |
| Workspaces | a vendor's charge | the vendor's breakdown | `vendor_shares` (`saas.md`) |

The last three write one table, `split_weights`, and both base loads apply it with one
`LEFT JOIN` on `weight_key`, and on the row's day where the weights carry one
(`dt`: Kubernetes weights are daily, the others monthly; `data-model.md`). A row with
weights becomes one row per weight; a row without stays as it was billed (invariant 6).

## Contents

- Proration
- Fair-share
- Other shared costs
- Order
- Gate and failure modes

## Proration

Built into the daily load; there is no separate splitter to forget.

- **Spread only what starts and ends inside the month.** One row per day, every cost
  column divided by the day count.
- **Anything else stays whole on one day.** A row dated before or after its month
  lands on the month's first or last day, so it never escapes its partition.
- **Keep the one-second trick.** A FOCUS end is exclusive (the next day's
  `00:00:00`), but a shim or a non-conforming export can end at `23:59:59`;
  `DATE(end - INTERVAL '1' SECOND)` puts both on the last day (article Part 3). A plain
  `date_diff` gets one of the two a day wrong: without `+ 1` the `23:59:59` row comes
  out a day short, with it the midnight row a day over.
- **Never today.** Rows starting today are out, and spread days stop at yesterday (the
  divisor stays the whole period's day count; the days not reached yet aren't written).
  Mid-month a monthly fee has written only its elapsed days, which is why monthly and
  daily are compared only once the month is over: its last day is before the run date.
- **A degenerate period** (zero length, end before start) stays whole on its start day.

`EffectiveCost` already amortizes commitments, so reservations and savings plans need
no splitter of yours: proration is about charge periods, not commitments.

## Fair-share

The adapter marks a shared charge (`shared_cost = 'support'` for Enterprise Support), and
`account_weights` writes the weights for it under
`share:<id>:<payer>`:

- **Weights from the same month's bill, first-party only** (article Part 3). Each
  account's weight is its first-party `effectivecost` over its payer's total. Only
  `Usage` and `Purchase` rows with a finite cost and both a payer and an account count:
  credits, tax and adjustments stay out, and so does every shared charge, this one and
  any other (a network account's charge earns no share of support). So does today.
- **Per payer.** Each billing account's charge splits over that billing account's
  accounts, never across payers.
- **Positive spend only.** Zero and net-refunded accounts drop out: a negative weight
  would hand them a refund of someone else's cost.
- **Never lose cost.** A payer with no account left after the filters gets no weights,
  and its charge stays whole where it was billed, with no `split_by`. It shows up, and
  someone asks.
- **The split rows** carry the target account's id and its latest name in the month;
  everything else (service, category, currency) is the shared charge's own.

The daily table gets the same weights: the support row is spread over its days like any
charge, and each day fans out by account.

## Other shared costs

Same shape, a different marker:

- **A shared networking, logging or security account**: mark its rows in the adapter
  (`shared_cost = 'network'`), add the id to the profile's `shared_costs`, and
  the run computes `account_weights` for it too. The id is the link: the run
  stops if the bill carries a `shared_cost` the profile doesn't list.
- **Weights that aren't spend** are rare; if finance insists, the producer for that id
  reads its own source but keeps the per-payer normalization and the
  `share:<id>:<payer>` key.


## Order

`pipeline.md` ("One run") has the order of a run. The producers write disjoint keys, so
their order among themselves doesn't matter.

## Gate and failure modes

On a closed month, per provider: the totals check returns zero rows (bill = monthly =
daily, per payer and cost column). Show the user the support split: one row in, N
accounts out, weights adding up to 1.0.

| Symptom | Likely cause |
|---|---|
| One account's support share jumps the month a Marketplace prepayment lands | the weights count resold spend: check `first_party` in the adapter |
| Warehouse below the bill by about one day of one charge | the day count is off by one: the one-second trick was dropped |
| Support stays where it was billed | no positive first-party spend under that payer that month: correct, and visible |

Other problem names (`weights_sum`, `bill_rows_added`, …): `reconciliation-ops.md`,
"Failure modes".
