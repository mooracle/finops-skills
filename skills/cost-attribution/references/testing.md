# Testing — what to test, and how, in any stack

The pipeline's failures are quiet: a dropped filter, a doubled row, weights that stopped
adding up to 1.0, a day count off by one. Every one of them produces numbers that look
fine. So every build ships with tests that prove the invariants, and every run ships with
checks (`reconciliation-ops.md`) that prove them again on real data (article Part 2,
"Rebuild the whole month and check it against the invoice").

## Contents

- The golden cases
- Porting them
- What a port must test beyond the cases
- Break it on purpose
- Test data is synthetic
- What a local engine can't prove
- CI

## The golden cases

`cases/` is the behavior as data: input rows, what must come out. They sit at the
**adapter contract** (`providers.md`), so they hold for any provider and any engine.
Two exceptions: `adapters.yaml` feeds one set of raw shapes (AWS tags as a map, Azure
CSV, the GCP sync's merged map) and one profile's choices (tax out, the `support` id,
three CostCenter spellings), so write the user's adapter cases from their raw shape and
their profile, keeping the same intents; and `rules.yaml` is a different format, one
`kind` per case (pattern, decide, expand, merge, review, key, check), each explained in
its header.

| File | Covers |
|---|---|
| `adapters.yaml` | raw provider rows → contract rows: charge categories, tags, currency variants and bad rates, every form of Kubernetes cluster identity, first-party, unused commitments |
| `load.yaml` | what enters the base tables, which day it lands on, totals per cost column, today cap, partitions, names |
| `share.yaml` | fair-share by first-party spend, per payer, positive only, never lose cost, unlisted shared ids |
| `k8s.yaml` | the article's worked example, idle spread or in a bucket, daily weights (a day's spike, a day without metrics) and the monthly option, tags first, GPU pools, buckets, per account and region, coverage (a shared ingress controller is the user's adapter case: `kubernetes.md`) |
| `vendor.yaml` | shares, whole-month data, unmapped share, untrusted breakdowns line by line, identities, registry overlaps, spanning charges, a prepaid deal spread on the bill side |
| `checks.yaml` | what stops a run: unloadable rows, currencies, a lookup key published twice (the views check); the invoice check's report, `invoice_check` (an empty month is in `load.yaml`) |
| `views.yaml` | categories at query time on one key, NULL and case, classification from `split_by`, the worklist per currency |
| `rules.yaml` | patterns, specificity and conflicts, one winner's categories, expansion, the never-remove merge, the PR review per currency and its kept rows, the key recipe, the fields a rule may match |

A case states only what its point needs; `defaults.yaml` fills in an unremarkable rest.
The pipeline cases (every file but `rules.yaml`) look like this:

```
name: what the case proves
given:   {table: [rows]}            rows to insert; `bill` is the adapter contract
run:     {period, provider, today, shared_costs, tolerance} | false
expect:
  stops: {check: weights | totals | views, problems: [names]}   else the run must pass
         (`includes: [names]` instead of `problems`: at least these)
  <table>:                          base_monthly, base_daily, split_weights,
    where: {column: value}          services_monthly, services_daily, or adapter:<provider>
    sum: 130 | {column: 130}        effectivecost unless columns are named
    by: column | [columns]
    sums: {group: amount}           effectivecost per group, exactly these groups ("a / b" for two columns)
    rows: [{column: value}]         exactly these rows, compared on these columns
    count: 3
  report: {name, params, rows}      run the named report (`params` default to
                                    BILLING_PERIOD 2026-04) and compare its rows
  unchanged: [tables]               in a `then` step: exactly as before the step
then: [{given, run, expect}, …]     later steps in the same warehouse
bad_input: true                     the case feeds bad money on purpose
tolerance: 0.01                     for amounts (default 1e-6)
variants: {provider: usd | fx}      adapter cases: the USD columns or rate (`usd`), or `fx_rates` (`fx`)
```

What the runner does with each:

- **Rows in.** Each `given` row is the table's row from `defaults.yaml` with the case's
  columns over it. A `bill` row goes into a table shaped like the adapter contract, and
  every provider's adapter in the test warehouse is a stand-in that selects that
  provider's rows from it; adapter cases (`adapter:<provider>` expectations or
  `variants`) use the real adapters over the raw tables instead. A `rules_lookup` row
  gets its `lookup_key` computed from its key fields with the rules engine's recipe, and
  its NULLs load as `''`, as the CSV table reads them.
- **The run.** `run` merges over `defaults.yaml`'s; `run: false` runs nothing (adapter
  and view cases). A `then` step without `run` uses `defaults.yaml`'s run, not the
  previous step's.
- **Out.** A step with `stops` must fail at that check, with exactly those problem names
  (or at least them, with `includes`); a step without must pass. `rows` matches as a
  multiset, ignoring order and unnamed columns; `sum`, `sums` and `count` apply after
  `where`; numbers compare within the tolerance.

## Porting them

1. Copy `cases/` into the user's repo.
2. Write a runner in their test setup. For each case: an empty warehouse, the `given`
   rows inserted (defaults applied), the run with the case's parameters, then the
   expectations. Expect about 150 lines; most of it is comparing rows.
3. Start with the files for what's built, phase by phase (`SKILL.md`, phase map). Two
   `checks.yaml` cases wait, each marked with a comment: the views case for Phase 4 (it
   needs `services_monthly`), the `invoice_check` case for Phase 5.
4. Use a local engine when the real one is slow or remote: DuckDB stands in for Athena
   and BigQuery, Snowflake and Databricks have local or emulated options of varying
   fidelity. Shim with macros the functions the local engine lacks or reads differently
   (`element_at`, which returns a list on a DuckDB map; `json_extract_scalar`, `is_finite`,
   `from_iso8601_timestamp`); never fork the SQL for
   the tests. Make the stand-in stricter where it's laxer than the real engine: run it
   in UTC; give DuckDB a `sequence` (both the two- and three-argument forms, over
   `generate_series`) that raises when start is after stop, as Trino's three-argument form does (its two-argument form counts down instead), rather than
   returning nothing so the row vanishes; shim `last_day_of_month` too; and after every statement, fail the test if the
   base tables or `split_weights` hold NULL or non-finite numbers (a division by zero
   gives `inf` or NaN on a DOUBLE rather than an error on either engine, while an integer or decimal one raises on Trino, so
   guard every division and let the tests catch what slips), except in `bad_input` cases.
   Shadow `max_by` with one that keeps a NULL value, as Trino does (DuckDB's
   `arg_max_null`); otherwise a missing NULL guard passes locally. A function that takes
   a lambda (`transform`, `map_filter`) can't be a macro: rewrite the call and its lambda to
   DuckDB's form in the test runner only.
5. Every adaptation for the user's bill gets its own case in their copy: a second shared
   charge, a tag with three spellings, a vendor whose API reports counts. Name it after
   the rule it protects.

## What a port must test beyond the cases

Write these for the user's code too:

- **A broken split is caught.** Break the code the way a bug would (drop the weight from
  the fan-out, drop a normalization) for one test, and expect a check to stop the run.
  Assert the edit applied, so a later rename can't make the test pass by missing.
- **Tampered tables are reported.** Delete a day, shift money between two payers, change
  one cost column alone, date a row outside its month: the totals check names each.
- **No half-built month.** A statement that fails mid-load or a failed totals check:
  both base partitions end up cleared. A failed weights check stops before any base clear, and
  a failed views check after the totals check passed: neither clears a base partition. A clear
  that fails: the other partition is still cleared, and the failed clear raises its own
  alarm.
- **The wiring.** Fakes, not mocks, for S3, the query engine's client and HTTP; assert on
  what got written, never on which calls were made. Make the default cloud client raise
  in tests, so nothing reaches a real account by accident. The fakes must fail the way
  the cloud does, or the tests prove only the happy path:
  - storage lists in pages and answers a batch delete with per-key errors;
  - the query engine runs past a deadline, and a cancel that takes its time or never
    lands;
  - an HTTP server that redirects to another origin, sends a foreign `next` link,
    answers 429 with a dated `Retry-After`;
  - an event bus that accepts the call and fails the entry;
  - an export run folder without its manifest, and a manifest that changes mid-copy.
- **Deliveries.** A file the manifest lists but staging lacks stops the publish; a file
  staging has that the manifest doesn't list stops it too; a lost notification is sent
  again on the next poll; a crash between the clear and the marker is repaired by the
  retry; an older delivery's event arriving after a newer one is published leaves the
  newer one live, and the same event twice swaps once (`pipeline.md`, "Delivery").
- **Conventions.** The loads group by every key field; the view's key fields equal the
  rules engine's, in order; every field a rule can match has a place in the specificity
  order; shared SQL names no vendor, cluster or account; every query parameter is listed
  and filled; no query reads the clock; the metrics sync's pools are `k8s_pools`' own
  rows, every selector in its queries carries the cluster's `cluster_selector`, and the
  `cluster` it writes is the profile's `name`.

## Break it on purpose

For each invariant, break the code the way the bug would and watch a test fail
(mutation testing, by hand or with a tool), and trust the suite once every mutant is
caught by a case or a test.

When you mutate Python in a loop, turn bytecode caching off
(`PYTHONDONTWRITEBYTECODE=1`): a mutant the same size as the original, restored within
the same second, can leave a stale `.pyc` that the next run loads.

## Test data is synthetic

Cases, fixtures and recorded API responses never carry the org's real identifiers or
amounts, even in a private repo: tests get copied into issues, gists and other repos.

- Ids, teams and vendors obviously made up (`acct-1`, `payer-1`, `Metrics Co`); amounts
  that make the arithmetic readable (600 / 300 / 100).
- Service names are short stand-ins (`Amazon EC2`, `Amazon S3`). AWS's FOCUS rows carry
  the full product name (`Amazon Elastic Compute Cloud`), and that's what rules match on
  a real bill.
- Neutral regions and currencies unless the case is about them.
- A recorded vendor response is redacted before it's committed: ids, names, emails,
  tokens and amounts replaced, structure kept.

## What a local engine can't prove

A local stand-in is close to the real engine, not identical. Before a query's first
production run:

1. Run it once on the real engine against one closed month (or `EXPLAIN` the INSERTs) to
   catch syntax and type coercion the local engine accepted.
2. Watch for the known differences (DuckDB vs Trino here): `GREATEST`/`LEAST` with a NULL
   (Trino returns NULL), map subscripts on a missing key (use `element_at`), ISO
   timestamps with a `T` (use `from_iso8601_timestamp`), integer division, `max_by` with
   a NULL value (Trino returns it; order by a time that's NULL whenever the value is), NaN
   comparisons, `regexp_extract` with no match (DuckDB returns `''`, Trino NULL: wrap it
   in `NULLIF(…, '')`), `regexp_replace` (DuckDB replaces the first match unless given
   the `'g'` option; Trino replaces all), and `AT TIME ZONE 'UTC'` on a parsed timestamp (a
   test run in UTC passes with the conversion missing: give one case an offset like `+02:00` and
   run it on the real engine).

## CI

- On every PR: the ported cases and the user's own tests, for the pipeline and for the
  rules engine; for a rules change, also `rules check`, `rules expand` and `rules
  review` (`rules-engine.md`).
- On main: the same, then deploy and stage the lookup.
- Keep the suite under a minute with no credentials, so nobody skips it.
