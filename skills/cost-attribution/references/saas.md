# Track A — SaaS bills

Run this track per vendor the checker marks READY. The bill doesn't say which of your
teams used a vendor; the vendor's API does (article Part 5).

## Contents

- Pick the shape
- Onboard a vendor: the discovery gate
- What a vendor adds (data, not SQL)
- The shim
- How the split works
- The vendor report
- Prepaid deals and annual commits (optional)
- Gate and failure modes

## Pick the shape

Simplest first. Most vendors stop at the first that fits.

| Shape | When | What you build |
|---|---|---|
| **One team** | one team uses it | nothing: a rule routes the charge to that team |
| **Shares** | many teams; the vendor bills on your cloud bill | a registry row and a shim that writes the vendor's breakdown |
| **Directly billed** | the vendor invoices you; no cloud row exists | a shim that reads the vendor's charges from its billing API and writes them as FOCUS rows into `saas_focus` (provider `other`), one per charge line, dated by the period each covers, so the cost lands even before any split; their total is checked against the vendor's invoice where the org enters invoice totals (the invoice check, `reconciliation-ops.md`; Q-F5); then the shares shape on top if many teams use it. For a marketplace vendor the cloud row is the reference total and the API supplies only the breakdown; for a direct vendor the API supplies both |

## Onboard a vendor: the discovery gate

Before any vendor code, write this report from **evidence** and end it with one
decision. Evidence is a query result, an API response, a contract line, or a named
person's answer recorded as a `D-NNN` entry. Never fixtures, examples or guessed values.

| Fact | Question | Probe | Lands in |
|---|---|---|---|
| Where it bills | Which bill rows are this vendor's, on which account? | D6, then the rows for `lower(publishername) LIKE '<pattern>'` | `vendor_registry.publisher_like`, `subaccountid` |
| Billing identities | One subscription or account, or several, possibly under different payers? | `COUNT(DISTINCT subaccountid)` and `COUNT(DISTINCT billingaccountid)` over those rows | one vendor id per identity |
| Invoice shape | Daily usage rows, or one monthly installment? | days billed per month in D6 | which charge the shares split (one running past its month stays whole) |
| Unit and owner | What unit do teams own in the vendor (deployment, project, workspace, child org)? Who maps units to teams? | the vendor's console or API list of units | `vendor_lines.workspace`; the rules |
| Breakdown | Which API field gives cost per unit? Does it cover what the vendor bills that no unit owns (support, overhead, failed jobs)? | one month's API response next to the console | `vendor_lines.amount` |
| Completeness | How far does the API lag the bill? Does it restate past days? How far back does it go? Pagination, rate limits? | the same call on two consecutive days; the docs | `vendor_fetch.through_day`, the shim's schedule, the native capture |

The decision:

- **READY**: every row verified. Record the report as a `D-NNN` entry and set the profile's
  `facts`.
- **BLOCKED**: name each missing fact and the exact read-only probe or question that
  would answer it, and who can run it ("an admin with billing read on the vendor
  console runs `GET /usage?month=…` once"). Scaffold nothing.
- **NOT_A_FIT**: name the simpler alternative: one team (a rule), too small to matter
  (leave it where it was billed), or no per-unit signal at all (leave it whole and say
  so in the report).

A breakdown in usage units (gigabytes, events) stands in for dollars only when one flat
rate prices every unit, since only proportions are used; that rate is evidence too.

## What a vendor adds (data, not SQL)

Every vendor feeds the same tables (`vendor_registry`, `vendor_lines`, `vendor_fetch`,
and `saas_focus` when it bills directly; shapes in `data-model.md`) and the same SQL;
adding one never edits a template.

The shared SQL (the vendor-shares step, the loads, the checks) reads those tables and
nothing vendor-specific. Give it a test that fails when a string literal in it names a
vendor, a cluster or an account: those belong in the registry or an adapter.

## The shim

One per vendor, with only that vendor's quirks in it. Auth, pagination and the vendor's
idea of a SKU are most of the work. Share a few FOCUS and storage helpers between shims,
never base classes or hooks.

One run, for one month, in this order:

1. Read the month's usage from the vendor's API (this vendor's auth, paging and
   retries).
2. Keep the response verbatim, append-only, before anything transforms it. Some APIs
   only serve the last year.
3. Turn it into lines: this vendor's units and SKUs become `workspace, servicename,
   amount`.
4. Write the lines to `raw/vendors/lines/billing_period=<month>/<vendor>.csv`, replacing
   that vendor's file for the month.
5. Write one row, `vendor, through_day, fetched_at`, to
   `raw/vendors/fetch/billing_period=<month>/<vendor>.csv`.
6. Ask the ETL to rebuild that month for the provider the vendor bills on.

- **One line per workspace and service line.** `servicename` says what the vendor
  billed for (compute, storage, data transfer): the split writes it into the base row,
  so an Elastic deployment comes out as one row per service line.
- **Unmapped cost is a line too**, with an empty workspace: support, overhead, anything
  the vendor bills that no unit owns. Its share stays where the charge was billed.
- **`through_day` is the honest answer to "is the month complete?"**: the last day the
  API's numbers are final for. A shim that can't tell should write yesterday minus the
  vendor's documented lag.
- **Money only from the bill.** The vendor's numbers set proportions, never amounts, so
  the split always adds up to what the bill says. Counts stand in for dollars only at one
  flat rate ("Onboard a vendor"). One exception: a prepaid deal's spread takes its timing
  from the lines; its total is still `total_paid` from the bill ("Prepaid deals").

### Calling the vendor's API

The shim holds a credential that can read your vendor account, and it runs unattended.
Whatever HTTP client it uses, these hold:

- **The credential goes to one origin.** Send it only to the vendor's API origin
  (scheme, host and port), over HTTPS. Don't follow redirects with it attached: turn
  automatic redirects off, or follow one only to the same origin. Some clients forward an
  `Authorization` header to whatever host a redirect names; Python's `urllib` does.
- **Pagination stays on that origin too.** A `next` link that points elsewhere is an
  error, not a page.
- **Every request has a timeout**, and the run has a deadline. Retry network errors,
  429 and 5xx with backoff, within the deadline. Honour `Retry-After` in both its forms,
  seconds or an HTTP date. Never retry a 4xx that isn't 429: it won't get better.
- **Secrets stay out of everything you keep.** Not in logs, exception messages, the
  verbatim capture (keep bodies, not request headers) or the URL you log (strip query
  strings that carry a key). Read the credential from the secret store at run time.
- **A test for each rule,** against a local server or a fake transport: a cross-origin
  redirect refused with no credential sent, a foreign `next` link refused, a dated
  `Retry-After` honoured, a 401 not retried.

## How the split works

The vendor-shares step turns a vendor's lines into weights under `vendor:<id>`, and the
base loads fan each of the vendor's bill rows out by them (`data-model.md`):

- **Which rows are the vendor's.** A registry row matches a bill row when the provider
  is the same, the lowercased `publishername` is `LIKE` its `publisher_like`, and its
  `subaccountid` is empty or equal to the row's. Read the registry `DISTINCT`, so a row
  pasted twice can't load a charge twice. In `bill_v` a matched row carries
  `split_by = 'vendor:<id>'` whether or not it splits; the loads keep it only on rows that
  split (`data-model.md`).
- **Only charges inside their month.** A row splits only if it starts on or after the
  month's first day and ends on or before the next month's first day. An annual fee
  matches the vendor but has no weight key, so it stays whole where it was billed.
- **Wait for the data.** A vendor splits only once `vendor_fetch` covers the whole month:
  the `through_day` of its newest fetch row for the month (by `fetched_at`; on a tie, the
  earlier `through_day`) is on or after the month's last day. Until then its charge stays
  where it was billed, in both tables (article Part 5).
- **Shares.** Group the vendor's lines for the month by workspace (empty is no
  workspace) and service line; each group's weight is its amount over the vendor's
  total. A group with amount 0 gets no weight row. A workspace's weight sends its share
  to `x_namespace = <workspace>` with the line's `servicename`. Only vendors registered
  for the run's provider are split.
- **Keep what doesn't map.** A line with no workspace keeps its share on the billed row,
  with the row's own `ServiceName`. If 90% maps to workspaces, 10% stays.
- **All or nothing.** A vendor with one line that is empty, negative or not a finite
  number, or lines that don't add up to more than zero, isn't split at all. Each line
  is checked as the shim wrote it, before anything is summed: a refund can't hide inside
  one workspace's total, and an empty amount can't hand the whole bill to the other
  workspaces. Refunds belong on the bill, not in the proportions.
- **Per identity.** A vendor id is one billing identity, so two payers buying the same
  vendor never share proportions.
- **Credits too.** Every row the registry matches splits by the same shares, a credit on
  the vendor's charge included, so the vendor's net cost lands where its usage was.

Two registry rows that match the same bill row would load it twice: the weights check
stops the run before anything is cleared, naming both entries.

## The vendor report

For each registered vendor: was this month's charge split, and if not, why.

`vendor_status` takes one parameter, the billing period, and returns one row per
distinct `(vendor, provider)` in `vendor_registry`, biggest `on_bill` first, vendors
not on the bill last:

| Column | What |
|---|---|
| `vendor`, `provider` | the registry entry |
| `on_bill` | the month's `effectivecost` on the bill rows carrying `split_by = 'vendor:<id>'` for that provider (the stacked adapters, before any split) |
| `kept_whole` | the part of `on_bill` on rows with no weight key: charges running past their month |
| `through_day` | the `through_day` of the fetch row the split reads (the month's newest by `fetched_at`; on a tie, the earlier `through_day`), or empty |
| `status` | the first that applies, in this order |

1. `not_on_bill`: no bill row of that provider carries the vendor this month (`on_bill`
   and `kept_whole` are 0).
2. `kept_whole`: no bill row of the vendor this month has a weight key; every row stayed
   whole by design.
3. `split`: `split_weights` holds weights under `vendor:<id>` for that provider and
   month.
4. `no_lines`: the vendor has no line at all this month. A line counts even when its
   amount is empty: a vendor whose lines all lack an amount is `unusable`, not
   `no_lines`.
5. `waiting`: no fetch row for the month, or a `through_day` before the month's last
   day.
6. `unusable`: anything else. The lines broke the all-or-nothing rule.

Lines and fetch rows are read by vendor id and month, whatever the provider.

## Prepaid deals and annual commits (optional)

A charge that runs past its month stays whole where it was billed (above), and the
purchase shows as a spike, like the annual licence in article Part 3. That is the
default: nothing to build, and finance may amortize in its own books. If the user wants
the deal spread in the views, build it as data on the bill side, never by editing raw, so
every check holds as it is, the totals check on every cost column included:

- a `commitments` file in Git, one row per deal: `vendor`, the buyer's
  `billingaccountid` and `subaccountid`, `publishername` and `servicename` as the
  purchase row shows them, `total_paid`, `total_credits` (empty for a
  flat fee), `start_period`, `end_period`, `currency`;
- the adapter of the provider the vendor bills on reads it and makes exactly these
  changes, no other:
  1. the deal's purchase row (the vendor's `Purchase` row on the buyer's account, in
     `start_period`) carries `EffectiveCost` 0, as FOCUS does for a commitment paid up
     front; its `BilledCost`, `ListCost`, dates and account stay as billed, so
     `BilledCost` still reconciles the invoice;
  2. each month of the deal gets one row of its own: the file's columns, `ChargeCategory`
     `Usage`, `PricingCategory` `Committed`, the rest NULL (`first_party` false), dated
     the whole month, with `BilledCost` and `ListCost` 0 and `EffectiveCost` the month's
     part of `total_paid`: the credits the month's lines (all of them, mapped or not) say
     were used, at `total_paid / total_credits`, or an equal part per month for a flat
     fee. The deal vendor's line `amount` is in the unit of `total_credits`; a month's
     credits are its finite line amounts summed, never below 0. The month rows never
     take the deal past `total_paid`: an overage month gets what's left. A month the
     provider's raw table doesn't hold yet gets no row. Inside its month and matched by
     the registry, it splits by that month's shares like any other charge;
  3. in the deal's last month, once the provider's raw table holds it, the credits never
     used get a row like it, dated the deal's whole term: it runs past its month, so it
     stays whole on the buying account, and the deal's `EffectiveCost` over its life
     equals `total_paid`.

Record it as a `D-NNN` entry. No check needs an exception: the bill side of the totals
check reads the adapter, so these rows are on both sides of it.

## Gate and failure modes

Per vendor, on a closed month:

1. The vendor report says `split`.
2. The totals check returns zero rows for the provider the vendor bills on.
3. The split's per-workspace proportions look right to whoever owns the vendor account,
   against the vendor's own console.

Read `vendor_status` weekly and fix the top unsplit vendor; an unsplit vendor is a
warning, never a failed run.

| Status / symptom | Likely cause |
|---|---|
| `waiting` | the month isn't covered yet; it splits on the first run after the shim catches up |
| `kept_whole` | every row stayed whole by design: it runs past its month (an annual fee, say) |
| `no_lines` | the shim ran but wrote nothing: check its transform |
| `unusable` | a line that is empty, negative or not finite, or a zero total: look at the month's lines |
| `not_on_bill` | the registry pattern matches nothing: wrong publisher name, or the vendor billed late |
| Run stops with `registry_overlap` | two entries match the same rows: narrow one by account |
| Proportions drift from the console | the API field leaves out something the vendor bills (overhead, failed jobs); use the field that includes it |
