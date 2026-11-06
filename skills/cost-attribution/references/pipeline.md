# Pipeline — the order things run in

The tables are in `data-model.md`. This file is how a delivery becomes rows in them, in
what order, and what happens when something fails. Write the user's pipeline in their
orchestrator; the order and the failure rules below are what must hold.

## Contents

- The flow
- Triggers
- Delivery: from the export to the raw partition
- One run: `(billing_period, provider)`
- Idempotency
- Failure handling
- Backfill and schema changes
- Running it on AWS (Athena inside a deadline · clearing a partition on S3)

## The flow

```
export delivery ─┐                                     vendor API ─► shim ─► raw/vendors, raw/saas
 (manifest)      ├─► stage the manifest's files ─┐     Prometheus or Datadog ─► metrics sync ─► raw/k8s
Azure poll ──────┘   raw/<p>/staging/…           │                        │
                                                 ▼                        ▼
                         "data ready" {period, provider, delivery}  ◄─────┘ (they notify too)
                                                 │
        ETL (concurrency 1): publish the delivery into raw/<p>/focus, then
                             the rebuild ──► split_weights, base_monthly, base_daily, checks;
                             the close, the invoice check, the views check: each on its event
 rules repo ─► CI expand ─► rules/staged ─► merge Lambda ─► rules/lookup ─┐
                                                                         ▼
                                                 services_daily / services_monthly (views)
```

## Triggers

| Source | Trigger | Then |
|---|---|---|
| AWS export | S3 `ObjectCreated` on the month's manifest | a copy Lambda stages the files the manifest lists and sends `{period, provider: aws, delivery}` ("Delivery", below) |
| Azure export | a scheduled poller | stages the newest complete run of each month whose published delivery differs, and sends one event per month |
| Directly billed vendor | its shim's schedule | once the month's invoice exists, writes it as FOCUS rows to `raw/saas/focus/` (one object per vendor and month, replaced whole) and sends `(period, other)`. Nothing before: a month with no invoice yet sends no event, so the close never sees a vendor month that hasn't been billed. Independent of the breakdown, so missing usage data never hides the bill |
| Vendor breakdown | its shim's schedule | writes `vendor_lines` and `vendor_fetch`; sends `(period, provider the vendor bills on)` |
| K8s metrics | the sync's schedule | writes the day to `k8s_metrics`; sends `(period, provider)` |
| Rules | merge to main | CI stages the lookup, the merge Lambda publishes it; nothing rebuilds. The published lookup's storage event (S3 `ObjectCreated` on `rules/lookup/lookup.csv`) sends `{"views": "all"}` to the ETL function, which runs only the views check, over every month (`reconciliation-ops.md`) |
| Monthly close, if finance freezes months | the org's schedule (Q-F1), repeated until written | one event per provider **to the ETL function** (`{"close": "YYYY-MM", "provider": "aws"}`; `reconciliation-ops.md`) |
| Invoice check, unless the org enters no invoice totals (Q-F5) | from the day the month's invoices usually arrive (Q-F5), on each scheduled run until that month's check passes | one event per provider to the ETL function (`{"invoice_check": "YYYY-MM", "provider": "aws"}`; `reconciliation-ops.md`) |

Over-triggering is cheap: rebuilds are idempotent and serialized. Under-triggering is the
bug: a vendor's late data that sends nothing sits unused until the next export.

## Delivery: from the export to the raw partition

The totals check compares the warehouse with raw, so a raw month that lost a file or
kept an old one passes every check. Raw proves itself against the provider's manifest
instead.

1. **The manifest is the only list of files.** AWS writes `Manifest.json` after every
   data file of a delivery has landed and lists them; in overwrite mode, chunks a
   smaller delivery no longer needs stay behind as empty files, so a folder listing is
   wrong. Azure writes a manifest into each run's folder (`blobs` with `byteCount` and
   `dataRowCount` each, `runInfo.runId` and `submittedTime`); a run folder without one
   is still being written, and is skipped. Never copy what a listing finds.
2. **Copy into staging, then verify.** Copy exactly the manifest's files to
   `raw/<provider>/staging/billing_period=YYYY-MM/<delivery>/`. The delivery id is the
   run id on Azure and the manifest object's ETag on AWS. Verify that every listed file
   arrived, that nothing else is there, and that byte counts match where the manifest
   gives them. On AWS, also check the manifest is still the same object after the copy:
   a refresh that landed mid-copy restarts it. Write the manifest itself into the
   folder last, with the delivery's time (the manifest's `LastModified` on AWS,
   `submittedTime` on Azure): ETags and run ids say which delivery, not which is newer.
3. **Tell the ETL, and check it heard.** Send `{period, provider, delivery}`.
   EventBridge's `PutEvents` answers 200 even when an entry failed: read
   `FailedEntryCount`.
4. **The ETL publishes, one run at a time, and only forward.** Events arrive late,
   twice and out of order. An event for a delivery older than the published one is
   skipped with a warning (publishing it would roll the month back, and every check
   would still pass); one for the delivery already published skips the swap and only
   rebuilds. Otherwise its first step clears the live partition (`raw/<provider>/focus/billing_period=YYYY-MM/`, with
   the same clear as the warehouse), copies the staged files in (server side), checks
   the live listing is exactly the manifest's files, and writes
   `raw/<provider>/published/YYYY-MM.json` (delivery, its time, files, and the row count
   where the manifest has one) last. It runs inside the ETL, serialized with every rebuild and
   the close, so nothing ever reads a partition halfway through being replaced. A crash
   mid-swap leaves the old marker, and the retry swaps again from staging.
5. **Every rebuild checks raw first.** The live listing must equal the published
   marker's files, and where the marker has a row count (Azure), `COUNT(*)` of the raw
   month must equal it (one exact-month read). A mismatch stops the run before anything
   is cleared. Directly billed invoices (`other`) have no manifest and no marker: the
   shim is their only writer, each vendor's month is one object replaced whole in a
   single write, and their proof is the invoice check
   (`reconciliation-ops.md`), where the org enters invoice totals (Q-F5); without them,
   nothing proves them.
6. **The cursor is what was published, not what was sent.** A poller compares each
   month's newest complete run with its published marker. A run that differs gets
   staged and announced; one already staged but never published (an event lost on the
   way) gets announced again. Nothing records "sent", so a failed notification is
   retried on the next poll instead of losing the delivery. AWS's copy runs on the
   manifest's own event, and a failed copy lands in its DLQ.
7. **Staging is kept until a newer delivery is published**, then deleted (or expired
   by a 30-day lifecycle rule on `raw/*/staging/`).

## One run: `(billing_period, provider)`

```
0  validate: period ^\d{4}-\d{2}$, provider in the known list, TODAY fixed for the run;
   publish the event's delivery, if any; raw matches its published marker
1  split_weights   stop if the bill carries a shared_cost id the profile doesn't list
                   for this provider, warn if it lists one the bill doesn't carry;
                   clear the partition; run every producer (account_weights once per
                   shared cost, namespace_weights, vendor_shares)
2  weights check                                          nothing in base_* cleared yet
3  base_monthly    clear the partition; load
4  base_daily      clear the partition; load
5  totals check                                           any failure clears 3 and 4
6  views check     the run's month, once the views exist  a failure clears nothing
```

Steps 1 to 6 are one call (a `rebuild` of one provider-month) that returns its warnings
and raises on a failed check, naming the check and its problem rows.

## Idempotency

- **One run at a time.** Two deliveries landing together would otherwise interleave
  clear and insert on the same partition (on Lambda: reserved concurrency 1).
- **Late data is just another run.** A copy or a shim that lands while a run is going
  sends its own event, so the month rebuilds again right after; nothing tracks deltas.
- **Cancel what times out.** If the runner gives up on a query, it cancels it and waits
  for a terminal state before clearing or retrying (on Athena, `StopQueryExecution`). A
  query left running can land its INSERT after the retry rebuilt the partition, and book
  the cost twice.
- **Leave time for cleanup.** Each query waits only as long as the run can afford: what
  is left (a Lambda's 15 minutes, a task's timeout) minus a cancel and two partition
  clears. And a run starts nothing while an earlier run's write is still in flight: a
  killed function never got to cancel it ("Running it on AWS", below).
- **Raise on `FAILED` and `CANCELLED`.** A cleared partition plus a silently failed
  INSERT is a half-empty month that looks fine.
- **A clear is all or an error.** A storage delete that reports per-object failures
  (S3's `delete_objects`) is checked object by object ("Running it on AWS").
- **Refuse unfilled placeholders.** The one function that fills `{{NAME}}` placeholders,
  used by the handler and the tests alike, raises on a placeholder with no value, on any
  `{{` or `}}` left after filling (a lower-case or spaced name), on an empty value, and
  on a value with anything but letters, digits, space and `_:./-` (values go into SQL
  text, so a quote must never get in). A check with an empty provider compares 0 with 0
  and passes; an empty period scans nothing and succeeds.

## Failure handling

- **Any failure after a base clear clears both base partitions, then raises.**
  The rebuild does it for a failed check and for a failed statement alike, and
  tries the second clear even if the first one fails. The error that goes up is the
  original one; a failed cleanup is logged as its own alarm, never in its place. The
  views check (step 6) is the exception: it raises and clears nothing, since the totals
  check has proved the base tables and the fault is in the lookup.
- **A rebuild is visible while it runs.** Between the clear and the last INSERT (a few
  minutes) a dashboard can read a partial month. For most orgs that window is fine. If
  it isn't, write to `warehouse/_staging/<run-id>/` and move the objects into place only
  after the totals check passes.
- Alarms and dead-letter queues: `reconciliation-ops.md`, "Alarms".

## Backfill and schema changes

- **Backfill = invoke per month**: `{"period": "2025-08", "provider": "aws"}`, oldest
  first. Nothing else is needed.
- **After adding a routing column**, re-run every month: old Parquet partitions read the
  new column as NULL, and the rules join silently misses them.
- **After changing a split's logic**, re-run every month it can touch, then compare
  per-team totals before and after. A change meant to move cost between teams must leave
  every provider total unchanged to the cent.

## Running it on AWS

The rebuild needs a warehouse with three methods: `clear(table, period, provider)`,
`execute(sql)` and `query(sql)` returning rows; the tests pass a local engine with the
same three. On AWS, `clear` deletes the partition's S3 objects, `execute` runs one statement in Athena to completion, and
`query` reads every row back. The ETL handler does this, per event, in this order:

1. Bind an Athena client to this run's deadline (a Lambda's remaining time).
2. Stop if the workgroup still has an INSERT queued or running (below); the next
   trigger retries. A lookup event raises instead, so its retry or DLQ keeps it.
3. A close event goes to the close, an invoice-check event to the invoice check, and the
   lookup's event (exactly `{"views": "all"}`, no period) to the views check over every
   month (`reconciliation-ops.md`). This comes before step 4, which would refuse it.
4. Validate the period (`YYYY-MM`) and the provider (the profile's list), and refuse
   anything else.
5. Publish the event's delivery, if it carries one ("Delivery", above).
6. The rebuild, with the month, today's UTC date, the shared-cost ids the profile lists
   for that provider and the tolerance in the month's currency. It returns warnings:
   log them where a metric filter can count them; they never fail the run.

### Running Athena inside a deadline

- **Raise on `FAILED` and `CANCELLED`,** with the state-change reason.
- **Read every page.** Use the results paginator: the first row of the first page is the
  header, every other row is data, and a NULL cell has no `VarCharValue`.
- **Each query's wait fits the run.** Wait at most the smaller of the query timeout and
  the time the run has left, minus the time a cancel may take and the time the two
  partition clears of a failure need (a minute and two make sense for a 15-minute
  Lambda). With nothing left, don't start the query.
- **A query that runs over is stopped and waited for.** `StopQueryExecution`, then poll
  until it reaches a terminal state. If it hasn't within the cancel allowance, raise an
  error of its own, distinct from a timeout, and alarm on it: that query may still land
  its INSERT. The failed run's own clear still runs (it can't make things worse); what
  must wait is the next rebuild of that month, and the next rule holds it back.
- **Nothing starts while a write is in flight.** A Lambda killed outright (out of
  memory, a hard timeout) never gets to cancel. So each run first lists the workgroup's
  recent queries (`ListQueryExecutions`, then `BatchGetQueryExecution`) and stops if an
  INSERT is `QUEUED` or `RUNNING`. Give the ETL a workgroup of its own, so the listing
  holds only its statements.
- **Test it against a fake** that runs a query past its budget, takes its time to
  cancel or never does, answers in pages, and has an INSERT in flight (`testing.md`).

### Clearing a partition on S3

Athena's external tables have no `DELETE`: clearing a partition is deleting its
objects. List them with the paginator, delete them with `delete_objects`, and read
every response's `Errors`: S3 answers 200 even when some keys failed, and a key left
behind double counts the month once the next INSERT lands. Raise on the first error,
naming the key and its code.

Refuse any prefix that isn't exactly one partition (it ends with `/` and contains
`billing_period=`): the same function, handed `""` by a bug, would empty the bucket.
