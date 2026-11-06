# Phase 4 — The rules engine

Goal: every warehouse row gets the org's categories (team, app, environment, …) at
**query time**, through a lookup generated from rules in Git and never edited by hand
(article Part 4). A rule change re-categorizes all of history the moment the lookup
updates; nothing in `base_*` rebuilds.

## Contents

- The pieces
- What rules can match
- The key: three places that must agree
- Rule semantics
- Expansion, history, and undoing a rule
- The views
- Pull requests: the dispute process
- Writing the first rules
- The weekly worklist
- Running it on AWS
- Gate and failure modes

## The pieces

Write the engine in
the user's rules repo, in a language their CI runs. Its core has no cloud in it: one
command, `rules`, with five subcommands. The key fields come from the profile's
`routing.key`, the categories from `categories.outputs`; the stack wiring (the
inventory's runner, the merge, CI) is theirs, written to the guarantees in "Running it
on AWS" below.

| Command | Reads | Does | Exits 1 when |
|---|---|---|---|
| `rules check` | `rules/*.json`, the rule-file schema | lists every problem: a file that isn't JSON, a schema error, and the checks in "What rules can match" and "Rule semantics" (a field that is neither a key field nor an account name, a regex that doesn't compile, an unanchored regex). No rule files at all is a problem too | any problem |
| `rules inventory-sql` | the key | prints the inventory query in the engine's SQL ("Expansion, history, and undoing a rule") | never |
| `rules expand` | `rules/`, `resources.json` | writes `lookup.csv` and prints how many rules, resources and matched resources | a conflict, printed with the tuple and each rule's file and number (`teams.json rule 2`) |
| `rules review --published <file>` | the published lookup, the fresh `lookup.csv`, `resources.json`, `rules/` (for the expansion warnings) | markdown for the PR summary ("Pull requests"); an explicit flag instead of `--published` when CI proved nothing is published yet | a kept row, or an unreadable input; never on a move |
| `rules unmatched [--min-cost N]` | `rules/`, `resources.json` | the resources no rule matches with `recent_cost` at or above N, biggest first, each amount with its currency | never |

Every command exits non-zero on a bad argument or an unreadable input; "never" means
never on what it finds.

**The rule-file schema.** Ship it as a JSON Schema next to the rules, outside `rules/`
(the commands read every `rules/*.json` as a rule file). A file is
`{"rules": [...]}` with at least one rule. A rule has `match` (at least one field) and
`team`, optionally any other category in `categories.outputs` (`app` and `environment`
in the article) and `why` (for the reviewer, never read by the engine), and nothing
else. So the schema's category properties come from the profile. A match field is a column name or `Tag:<Key>`; its value is
one non-empty pattern or a non-empty list of them. A category is a non-empty string, and
`Unallocated` is allowed: it's how a rule sends resources back after a wrong one.

## What rules can match

A rule can combine any fields in its `match`, but rules never see raw FOCUS. Expansion
runs over the inventory, and the inventory is `base_monthly` grouped by the key: a
column or tag the loads didn't group by is gone after Phase 3. So the fields rules can
match (profile `routing.key`, Q-G3) are decided before the first load, not when the
first rules are written.

Each field has to be carried through five places:

| Where | What it does with the field |
|---|---|
| the adapters (`providers.md`) | emit it: a FOCUS column, or `tag_<key>` with every D3 spelling folded in |
| the loads (`base_monthly`, `base_daily`) | keep it in the `GROUP BY` |
| the engine's key | name it, in key order |
| the inventory and the views | group and join on it (next section) |
| `rules check` | accept it in a `match`; any other field is refused |

Without the last one, a rule on `RegionId`, or on a tag nobody grouped, reads `''` on
every resource, matches nothing, and leaves its cost Unallocated with no error.
`cases/rules.yaml` (kind `check`) pins the refusal. Add two tests of your own: the loads
group by every key field, and the views join on them in key order.

Choosing the key:

- **Start from the default:** `provider, subaccountid, x_resourcegroupname, x_namespace,
  servicename`, plus one `tag_<key>` for each tag D3 kept. `x_resourcegroupname` is NULL
  outside Azure and costs nothing there; `x_namespace` comes from the Kubernetes and
  vendor splits.
- **Add a field when ownership depends on it.** Whatever Q-G2 says separates teams in a
  shared account goes in the key (the checker blocks Phase 3 until it is). So does
  `regionid` when a team owns "everything in `us-east-2`", or when one account runs a
  cluster name in two regions: `x_namespace` is `<cluster>/<namespace>`, so without the
  region both clusters' `default` is one resource with one owner (the checker blocks
  this too).
- **Reporting columns don't need to be key fields.** `regionid`, `servicecategory` and
  `chargecategory` stay in `base_*` for dashboards either way. Keying on them grows the
  lookup without routing anything.
- **Keep it at org-chart cardinality:** never a resource id or a per-deploy tag.
- **Key on ids, match on names.** A field rules match without keying must have one value
  per key, the way an account id has one latest name. Anything that can differ within a
  key belongs in the key.
- **Give a new field a place in the specificity order** ("Rule semantics"). With no
  place, a rule on it ranks as nothing and loses to a rule on `Provider` alone. Test that
  every field a rule can match has a place.
- **Rules see rows after the splits.** A split row takes its target's account and
  namespace, and a vendor row the vendor's service line; every other column, tags
  included, comes from the charge that was split. A tag rule won't catch the fair-share
  rows of a support charge; an account rule will.

## The key: three places that must agree

The lookup is keyed on a tuple of raw attributes. The tuple is defined once in the
engine (the key fields in order, and the function that builds the key) and used in
three places:

1. the inventory query groups `base_monthly` by those fields;
2. expansion and the merge write and compare `lookup_key`;
3. the services views build the same `lookup_key` in SQL and join on it.

`cases/views.yaml` feeds both recipes the same rows (NULLs, case, tags) and
`cases/rules.yaml` pins the key's exact string; test also that the views name the same
fields in the same order.

The recipe: the key fields in order, NULL read as `''`, joined with `chr(31)`, ASCII
letters folded to lowercase. Fold with a translation of the 26 upper-case ASCII letters
to the 26 lower-case ones, on both sides, never with `lower()`: Python and Trino
disagree on some non-ASCII letters (`İ`), and the key must come out the same in both. Account **names** are matched by rules and
carried in the lookup for people reading it, but never keyed: names change, ids don't.

**Adding a key field is a migration.** Published rows were keyed without the new field,
so none of them would match again and their history would fall to Unallocated. The merge
refuses with a migration error naming the new fields. Instead: add it to `routing.key`,
the adapters and `base_*` (rebuild history, `pipeline.md`), to the engine's key and the
views; expand against all history; compare
Unallocated per month before and after; then publish that expansion over the old lookup
on purpose (a replace, recorded as a `D-NNN` entry), and write rules for anything that lost
its owner.

## Rule semantics

One JSON file per team in `rules/`.

| Pattern | Means |
|---|---|
| `payments-ledger-prod` | exactly this value (case never matters, in any form) |
| `payments-*`, `*-prod`, `*ledger*`, `prod-*-eu`, `acct-?` | a wildcard over the whole value |
| `^payments-.*-prod$`, `^payments-`, `-dev$` | a regular expression: a leading `^` or a trailing `$` makes it one. Searched as written, ignoring case, so anchor what you mean. Compile it as written: lowercasing the pattern turns `\D` into `\d` |
| `["a", "b-*"]` | any one of these |

`rules check` rejects a regex that doesn't compile, and a pattern without an anchor that
contains `.*`, `.+`, `\d`, `\w`, `\s` or `|`: it reads like a regex but would be taken
as a wildcard or an exact value (`payments-.*` would match something else). Parentheses
alone don't make a regex: `AWS Support (Enterprise)` is matched exactly. Fields in one
`match` must all match, and a field the resource lacks reads as `''`. A match field
names its column in lower case (`ServiceName` → `servicename`); `Tag:<Key>` reads the
folded tag column, the key lowercased with `-`, `_` and spaces removed (`Tag:Cost-Center`
→ `tag_costcenter`), the same fold the adapters apply to a tag's spellings. A category a
rule leaves out is `Unallocated`.

**Most specific first.** When several rules match a resource, the one on the narrowest
field wins, in this order: `x_namespace`, tags, `x_ResourceGroupName`, `ServiceName`,
account, `Provider` (article Part 4).
Each rule's rank is one of three per group (`x_namespace`, any `tag_` column,
`x_resourcegroupname`, `servicename`, the account's id or name, `provider`): absent,
a pattern (wildcard or regex), or an exact value, compared from the narrowest group:
the first group where two rules differ decides, exact above pattern above absent, and
two fields in one group count once (the higher of the two). A list ranks by the best element
that matched. So **an exact value beats a pattern on the same field**:
`payments-ledger-prod` → data wins over `payments-*` → payments, which is how one
account leaves a convention. All patterns rank the same, wildcard or regex. Rules that
tie all the way are a **conflict**, whether or not they assign the same categories
(`rules expand` exits 1).
Two patterns on one field that both match a key are such a tie; the fix is a full name
on one side, a narrower field, or disjoint patterns.

**One rule wins, with every category it sets.** Categories never combine across rules:
when a namespace rule beats an account rule, the account rule's `app` doesn't fill in
what the namespace rule left out; that category is `Unallocated`. So a rule sets every
category it means. Forgetting one is silent, so `rules expand`
warns, per resource, when a rule it beat sets a category the winner leaves out; the PR
summary lists those warnings. A warning, not a failure: leaving `app` open on purpose
is allowed. A naming convention gives you `team` and `environment` from one
regex, but not `app`: patterns don't capture into values. Name apps in narrower rules
(one per app is normal, and reads well in review), or leave `app` `Unallocated` until
someone owns it. A rule that names the app of accounts the convention already covers
must be narrower than the convention rule (a namespace, a service, a second field, or
the exact account name), or the two tie and conflict.

```json
{"rules": [
  {"match": {"SubAccountName": "^payments-.*-prod$"},
   "team": "payments", "environment": "Production"},
  {"match": {"x_namespace": "prod/ledger-*"},
   "team": "payments", "app": "ledger", "environment": "Production"},
  {"match": {"SubAccountName": "payments-reconcile-prod", "ServiceName": "Amazon Simple Storage Service"},
   "team": "payments", "app": "reconciler", "environment": "Production",
   "why": "narrower than the convention: it repeats team and environment"}
]}
```

## Expansion, history, and undoing a rule

- **The inventory reads all history.** `rules inventory-sql` reads every resource
  `base_monthly` has ever held (megabytes; the one all-history read the skill allows). A
  recent window strands backfilled months: their resources never get rows.
- **What the inventory returns.** One row per key tuple, the key fields folded as the key
  folds them (NULL as `''`, ASCII lowercase), so tuples that differ only by case are one
  resource. `subaccountname` is the account's latest non-empty name across all its rows
  (by provider and account id, not by tuple), so a tuple last billed before a rename
  matches by the current name; a NULL name never wins, which on Trino means `max_by` over
  a key that is NULL when the name is. `currency` is the provider's latest one, and
  `recent_cost` sums `effectivecost` over the calendar month `--period` and the two before
  it, in that currency only, so the review and `unmatched` rank by what matters now and never
  add USD to EUR. Check the table name and the period before they go into the SQL.
- **Expansion writes matched resources only.** No row means Unallocated, via the view.
  `resources.json` tuples that differ only by case or by NULL against `''` are one
  resource, their `recent_cost` added. The lookup's columns, in order: `lookup_key`, the
  key fields, the name fields, the categories; empty for NULL, sorted by `lookup_key`.
- **The merge never removes.** The merge writes every fresh row and keeps every published
  row the fresh lookup doesn't have, sorted by `lookup_key`. It refuses an empty file
  (not even a header) on either side, and a fresh lookup missing a lookup column. An
  expansion that matched nothing changes nothing.
- **Undoing a wrong rule.** Deleting it moves the resources another rule matches to
  that rule. The ones no rule matches any more would keep the team they had, since the
  merge never removes a row, and nothing in Git would explain that team. So `rules
  review` lists them as kept and **fails the PR** until a rule says where they go: one
  that assigns them correctly, or, if nobody owns them yet, one that says `"team":
  "Unallocated"`.

## The views

`services_daily` and `services_monthly`: the article's view, `base_*` `LEFT JOIN
rules_lookup` on `lookup_key`, with every category
`COALESCE(…, 'Unallocated')`. One equality, no `LIKE` or regex in the hot path: the
patterns already ran during expansion.

The views also classify once: `split_kind` (`share`, `vendor`,
`k8s`), `unit_scope` (the cluster or vendor) and `unit_name` (the namespace or workspace),
all from `split_by`. Dashboards read those, never `x_namespace LIKE '%/%'`.

**Hierarchy.** When finance wants division → cost center → team, add an
`org_hierarchy (team, cost_center, division)` table, kept by whoever Q-C2 names, and join
it in the views on `team`. A team moving to another division edits that table, and the
rules stay; a team that splits or merges is a rule change.

## Pull requests: the dispute process

"Why is my number carrying the data platform's S3?" is answered with a PR that moves the
pattern. On every PR, CI writes `rules review` to the job summary ("Running it on AWS",
below).

What `rules review` counts, the way the merge will apply the branch: a fresh row is a
move when its categories differ from the published row's, or when there was no published
row and it assigns anything but `Unallocated`. A published row the branch doesn't
produce isn't a move (the merge would keep it); the review lists it as **kept** and
exits non-zero when there is one. Each move costs the resource's `recent_cost`
in its currency. The summary opens with the flows, `from team → to team` per currency
(from `Unallocated` when there was no published row; a move that keeps the team isn't a
flow), largest first within each currency; then a line with how many resources change
categories and how many leave Unallocated; then the moves, largest first, the top 40 and
a count of the rest; then the kept rows with their team and `recent_cost`, largest
first, with the line that the PR fails until a rule claims them. No moves and nothing
kept: one line saying the change moves no cost.

## Writing the first rules

With the user, against real data:

1. Refresh `resources.json` (the inventory query), then `rules unmatched --min-cost
   100` (everything is unmatched at first), or the worklist query for last month.
2. Write rules for the top 5–10 rows. Prefer the naming convention D2 found over lists:
   one regex covers future accounts too.
3. `rules check && rules expand`, then `unmatched` again; repeat until the big dollars
   are covered.
4. Stop. Set the alert (`finance.unallocated_alert`) and let the weekly loop take it
   down. Don't map every cent.

## The weekly worklist

The tuples that reached no team last month, biggest first (article Part 1, "The
loop"). Write rules for the top rows, merge, and watch the total shrink. Alert on the
total (`finance.unallocated_alert`, in the reporting currency; one threshold per
provider when providers stay in their own currencies), never fail a run on it.
The query, `unallocated`, takes the billing period and reads `services_monthly` rows
of that month whose `team` is `Unallocated`:

- **One row per key tuple and currency:** the key fields (`routing.key`; by default `provider`, `subaccountid`,
  `x_resourcegroupname`, `x_namespace`, `servicename`, each routed `tag_<key>`) plus
  `currency`, grouped as the view spells them (no folding: the worklist shows what the
  bill says, and the person writing a rule matches case-insensitively anyway).
- **`subaccountname`:** within the tuple, the name on its costliest row that has one.
- **`effectivecost`:** the tuple's total.
- **`part_of_unallocated`:** that total over the month's Unallocated total in the same
  currency, a window over `currency`, never over all rows. A currency whose Unallocated
  adds up to 0 gives NULL, not a division error.
- **Order:** biggest `effectivecost` first, the top 50. With several currencies that
  ranks unlike amounts against each other; that's fine for a to-do list, but never add
  them up.

## Running it on AWS

Three pieces connect the engine to a stack. Write them for the user's (AWS and GitHub
Actions on the reference stack), keeping these guarantees.

**The inventory runner.** Runs `rules inventory-sql` in the engine and saves every row as
`resources.json` (`{"resources": [...]}`, `recent_cost` as a number next to its
`currency`). On Athena, use the client rules in `pipeline.md` ("Running it on AWS"):
every page read, a failed query raises.

**The merge, where the lookup lives.** A function on the storage event for the staged
lookup (S3 `ObjectCreated` on `rules/staged/lookup.csv`):

- reads the staged file the event names, and the published `rules/lookup/lookup.csv`
  (the `rules_lookup` table's location);
- treats only "no such key" as an empty published lookup (the first publish); any other
  read error fails, or a permissions glitch would publish the staged rows alone and
  drop history. An object that exists but is empty fails too, for the same reason. Its
  role needs list on `rules/lookup/` as well as read: without it, S3 reports a missing
  key as 403, and the first publish fails;
- writes the engine's merge of the published and the staged lookup ("Expansion,
  history, and undoing a rule") back to the published key, as CSV;
- runs one at a time (on Lambda, reserved concurrency 1).

Test it against a fake store: a staged row replaces its published one, a published row
the staged file lacks stays, the first publish writes the staged rows.

**CI.** Every PR shows what it moves; main publishes exactly what CI checked:

- **Triggers:** pull requests, pushes to main, and a daily schedule (resources that
  appeared on the bill since yesterday).
- **Credentials:** short-lived, from the CI's identity provider (GitHub OIDC on AWS).
  Two roles: a read role (Athena on the workgroup to read `base_monthly`, read on the
  published lookup, and list on its prefix: without the list permission S3 answers
  403, not 404, for a lookup never published, and the first PR can't be reviewed), and
  a stage role that can write `rules/staged/` and nothing else, whose trust policy
  accepts only the main branch (on GitHub, `repo:<org>/<repo>:ref:refs/heads/main`), or
  a PR's job could take it and stage its own lookup. The merge function, not CI, writes
  the published lookup. PRs from forks get no cloud credentials; their review can't
  run.
- **Every run:** install, `rules check`, the tests, the inventory runner, `rules
  expand` (exit 1 on a conflict fails the job).
- **On a PR:** fetch the published lookup. Compare against an empty one only when it
  provably doesn't exist (404); any other error fails the job, because an empty
  baseline shows every resource as newly allocated. Write `rules review` to the job
  summary.
- **On main:** a second job, after the first, stages the very `lookup.csv` the first
  job built (passed as an artifact, never re-expanded) into `rules/staged/`.
- **Order:** main runs take turns in commit order, never cancelled midway, so an older
  commit's lookup can't land after a newer one's. PRs run side by side.

## Gate and failure modes

- The ported `cases/rules.yaml` and `cases/views.yaml` green, and the rules engine's own
  tests.
- `SELECT team, currency, SUM(effectivecost) FROM services_monthly WHERE billing_period = '<M>'
  GROUP BY team, currency ORDER BY currency, 3 DESC` returns the user's real teams. Never
  add amounts across currencies: under `per_provider`, USD 100 and EUR 100 aren't 200.
- Unallocated reported as an amount and a percentage, per currency. Under ~20% is a fine
  Phase 4 exit; the steady state is under 1–2%.

| Symptom | Likely cause |
|---|---|
| `rules expand` matches it, but the view says Unallocated | the key recipe differs between the engine and the views (`cases/views.yaml` should have caught it), or the lookup wasn't published yet |
| Backfilled months stay Unallocated | the inventory was windowed |
| Removing a rule didn't move anything | by design: history stays; write the rule that moves it |
| `rules check` refuses a field | the rule names a field that is neither a key field nor an account name: match on a key field, or add it to the key (a migration, "What rules can match") |
| A tag rule misses some rows | split rows carry the split charge's tags, not the target's: route them by account or namespace |
| `rules check` says "looks like a regex" | anchor it with `^` or `$`, or drop the regex characters |
| The merge refuses with a migration error | a key field was added: publish a full expansion on purpose |
