# The plan — components, verdicts, and the checker

The profile decides what gets built. In Phase 0, write a small checker into the user's
repo (`finops/check_plan.py`, or their language) from this file, with a test per rule,
and run it whenever the profile changes and before every phase. It reads
`finops/profile.yaml` and `finops/DECISIONS.md` and prints one verdict per component:

- **READY**: every fact the component needs is known, with evidence.
- **BLOCKED**: a fact is missing. The reason names the question (`Q-…`) or the
  discovery query (`D…`) that unblocks it.
- **NOT_A_FIT**: building it would cost more than it returns. The reason names the
  simpler alternative.
- **SKIPPED**: the user declined it.

A component also carries notes: things worth saying that don't block (a weak naming
convention, FOCUS 1.0). Show them with the verdict.

## Contents

- A valid profile
- The components
- Readiness and entering a phase
- Output and exit codes

## A valid profile

A profile that fails any of these is malformed: the checker stops with exit code 2 and
the reason, before any verdict.

- `schema_version` is 1.
- `providers`, `shared_costs`, `kubernetes.clusters` and `saas.vendors` are lists (or
  absent), and every entry has an `id`, unique within its list. Ids compare as exact
  text; provider ids, which are words, in any case.
- `org.why_build` uses only `multi_cloud`, `saas_bills`, `kubernetes`, `rules_in_git`,
  `own_categories`, `own_dashboards`.
- `categories.outputs`, when set, includes `team`.
- Every `evidence` value and every `assumptions[].decision` anywhere in the profile is one
  `D-NNN` id (three or more digits; a list of ids is malformed)
  and exists as a `## D-NNN` heading in DECISIONS.md. A profile that cites evidence with
  no DECISIONS.md is malformed. The check proves the record exists, not that it's true.
- A value of a shape the template doesn't show is malformed too: a list where it shows a
  mapping, a word where it shows a number, a word other than the ones it names (`none`)
  where it shows a mapping. A list item that is null is malformed. A `null` field's
  comment shows its shapes. Keys the template doesn't show are ignored where it names
  the keys; in `match`, `pools.dedicated` and the per-provider maps the keys are the
  user's. Exit 2, never a crash that reads as "not ready". `finance.frozen_close` is the
  exception: its paragraph below gives every value a verdict.
- An account id or `focus_version` written as a YAML number counts as text.
- `kubernetes.coverage_target` is a fraction in [0, 1].

## The components

In build order, each vendor's and each cluster's components together. **Core**
components make up the main path; the rest are optional tracks, one per shared charge,
vendor or cluster. `needs` is what must be ready first; `export:*` means at least one
export.

| Id | Phase | Core | Needs |
|---|---|---|---|
| `org` | 0 | yes | |
| `export:<provider id>` | 1 | yes | `org` |
| `stack` | 2 | yes | `org` |
| `base-tables` | 3 | yes | `export:*`, `stack` |
| `proration` | 3 | yes | `base-tables` |
| `shared:<id>` | 3 | | `base-tables`, `export:<its provider>` |
| `rules` | 4 | yes | `base-tables` |
| `saas:<id>/ingest` (directly billed only) | A | | `base-tables` |
| `saas:<id>/split` | A | | `base-tables`, then its ingest (direct), `export:<its provider>` (marketplace), or `export:*` while `billed` or the provider is unknown |
| `k8s:<id>/tags` | B | | `base-tables`, `export:<its provider>` |
| `k8s:<id>/shared` | B | | `k8s:<id>/tags` |
| `reconciliation` | 5 | yes | `base-tables` |
| `frozen-close` | 5 | | `reconciliation` |

When `org` is NOT_A_FIT, it's the only component: nothing else is worth checking.

Three rules that hold for every component:

- When several rules fire on one component, NOT_A_FIT wins over SKIPPED, SKIPPED over
  BLOCKED, and every reason is kept and shown.
- `stack` is always READY: its facts are assume-class (Q-R0 … Q-R4, `intake.md`) and
  have defaults.
- A component's spec file is the file its paragraph names; `frozen-close` reads
  `reconciliation-ops.md`, `stack` reads `stack.md`, and `base-tables` names two.

**org** (`intake.md`, Org):
- `org.owner` is `none` → NOT_A_FIT: nobody will own it; buy a tool or don't start (Q-O1).
- `org.owner` unknown → BLOCKED (Q-O1).
- `org.why_build` unknown → BLOCKED (Q-O3). Answered with an empty list: if the profile
  itself shows a reason (more than one provider, a vendor whose shape isn't `one_team`,
  a cluster with more than one workload namespace, or an unknown count), BLOCKED: re-ask
  Q-O3 and name what the bill shows. If it shows none
  and nothing else blocks, NOT_A_FIT: use the console's own allocation (AWS Cost
  Categories, Azure cost allocation rules, Google Cloud labels in billing reports).

**export:\<id\>** (`focus-export.md`): `export: missing` → BLOCKED: turn the export on,
and meanwhile run the non-data intake rounds and build the stack against the expected
layout (D0). Anything but `enabled` → BLOCKED: run D0. `focus_version` 1.0 → a note: it
works, plan the switch to 1.2. `delivery: new_folder` → a note: every delivery is a
new folder, so Phase 1 adds a 1.2 overwrite export beside it and the copy job reads
the manifest's files, never the month (`focus-export.md`).

**base-tables** (`data-model.md`, `providers.md`). BLOCKED when:
- no provider is listed (run D0);
- `dimensions.columns` lacks `subaccountid` or `servicename` (the rules join needs both),
  or `dimensions.evidence` is missing (choose them from D1 and D3 with the user);
- `finance.charge_categories` is unknown or empty (Q-F3);
- the currency plan has a gap (Q-F4): no `finance.reporting_currency`; or a provider
  with no `billing_currency` (D0); or a provider that bills in another currency than
  finance reports in, when neither its export converts (USD reporting on Azure, with
  the `x_*InUsd` columns, or Google Cloud, dividing by `x_CurrencyConversionRate`) nor
  `finance.fx_rates` names the org's rate table. `per_provider` converts nothing: a
  note that dashboards group by currency. Each conversion that will happen is a note.
- the routing key has a problem (`rules-engine.md`, "What rules can match"). The key is
  `routing.key`, or, unset, `provider, subaccountid, x_resourcegroupname, x_namespace,
  servicename` with a note (Q-G3). Fields name base-table columns the way rules do:
  lower-case, and `Tag:Cost-Center` is `tag_costcenter` (the `tag:` prefix becomes
  `tag_`, and `-`, `_` and spaces drop out of the key). It blocks when the key lacks
  `provider` or `subaccountid`; when it holds `subaccountname` (names change, key on the
  id; rules still match the name); when a field isn't grouped into the base tables (the
  grouped columns are `dimensions.columns`, a `tag_` column per `dimensions.tags[].key`,
  and the built-in `provider`, `billingaccountid`, `x_resourcegroupname`, `x_namespace`); when an
  `routing.accounts_shared_by_teams` entry splits by a field the key doesn't carry
  (Q-G2, Q-G3); and when one account runs clusters of the same name in two regions and
  the key lacks `regionid` (both write `<name>/<namespace>`, so rules can't tell their
  namespaces apart; D7). A cluster's name is its `name`, or its `id` when it has none. A
routed tag that isn't in the key is a note: no rule can match
  it, so route on it or drop it, since each grouped column multiplies the base rows.

**proration** (`splitters.md`): always READY once the base tables are. With
`period_charges.material: false`, a note that it's a correctness detail here.

**shared:\<id\>** (`splitters.md`): BLOCKED when its provider isn't one of
`providers` (D5), without a `match` predicate (missing or empty; D5), without
`weights` (spread it, or leave it where it's billed? Q-S1), or without `evidence`
(not confirmed as shared, Q-S1).

**rules** (`rules-engine.md`): BLOCKED without both `categories.outputs` and
`categories.evidence` (Q-C1), without `routing.account_convention` (run D2), or when
`routing.accounts_shared_by_teams` is unknown (Q-G2; an empty list means no account is
shared), or when an entry has no `account` or no `split_by` (Q-G2). A convention of `none`,
or one covering under 70% of cost, is a note: rules will be enumerations; raise a
naming convention with whoever provisions accounts.

**saas:\<id\>/ingest**, for a vendor with `billed: direct` (`saas.md`): BLOCKED until
`billing_identities`, `invoice_shape` and `completeness` each have a value and evidence
(Q-V3). Always a note: no cloud row to split; the shim writes FOCUS rows, reconciled to
each invoice where the org enters invoice totals (Q-F5).

**saas:\<id\>/split** (`saas.md`): `shape: one_team` or `used_by_teams` under 2 →
NOT_A_FIT: a rule routes the whole cost, no split. Otherwise BLOCKED while `billed` is
unknown (Q-V1), and when a marketplace vendor's `provider` is unknown or isn't one of
`providers` (D6). A shape other than `shares` → BLOCKED: how many teams use it? (Q-V2).
Then BLOCKED until all six discovery facts
(`where_billed`, `billing_identities`, `invoice_shape`, `unit`, `breakdown`,
`completeness`) each have a value and evidence (Q-V3). A `monthly_cost` under the
Unallocated alert (`finance.unallocated_alert`, one number or one per provider) is a
note: ask whether a split is worth it.

**k8s:\<id\>/tags** (`kubernetes.md`): `workload_namespaces` under 2 → NOT_A_FIT: every
row would read `<cluster>/<that namespace>`, which a rule on the account already
routes; one namespace per app first (D7). BLOCKED when `workload_namespaces` is unknown
(D7), when the cluster's provider isn't one of `providers` (D7), when the cluster or
pool tag key is unknown (D7), on a `provider: aws` cluster when `tags.activated` isn't
`true` (false: the keys exist but aren't cost allocation tags, so the user activates them
in the payer account and requests the backfill; null or missing: run D7, which always
answers it on AWS), when `pools` or `pools.shared` is unknown (Q-K1; `shared: []` =
none), when the cluster's account or region is unknown (D7: the `subaccountid` and
`regionid` of its rows), or when `kubernetes.shared_owner` is unknown (Q-K4). On other
providers `tags.activated` doesn't apply and never blocks: Azure tags and Google Cloud
labels reach the export with no activation step. An untagged share of cluster spend
above the coverage gap (`1 − kubernetes.coverage_target`, default target 0.9; the target
is a fraction, `untagged_pct` a percent) is a note: tag boot disks before tuning
weights.

**k8s:\<id\>/shared** exists only when the cluster has shared pools, or its GPU pools
aren't known to be none, and never when its tags component is NOT_A_FIT.
`metrics.source: none` → NOT_A_FIT: shared pools stay in
`<cluster>/pool:<pool>`, routed to the platform owner by a rule. A missing `metrics`
block is unknown, not none. Otherwise BLOCKED
unless the source is `prometheus` or `datadog` with a `pool_label` (Q-K2) confirmed
where the sync will read it (`pool_label_exported: true`: in `kube_node_labels`, or
as a Datadog tag; the Q-K2 probe). BLOCKED while `cluster_selector` is missing or null
(Q-K2; `""` means the endpoint serves this cluster alone), and BLOCKED while GPU
pools are unknown (Q-K3), or `kubernetes.idle` or `kubernetes.weights` is (Q-K6, Q-K7). On `prometheus`, BLOCKED while `pools.short_jobs` is missing or null (run the Q-K8 probe and ask), or names a pool that isn't one of its shared or GPU pools (Q-K8; `[]` = none). GPU pools present → a note: they split by GPU requests alone.

**reconciliation** (`reconciliation-ops.md`): READY; no profile field blocks it,
`org.finance_partner` and `finance.invoice_totals` included. `org.finance_partner`
missing, null or `none` → a note: the owner answers the Finance round and walks through
the numbers at the Phase 5 gate (Q-O2). `finance.invoice_totals: none` → a note: no
invoice check, so nothing ties the bill to the invoice; the Phase 5 gate and close
readiness drop their invoice items (Q-F5).

**frozen-close**: `finance.frozen_close: false` or `none` → SKIPPED: no month is frozen. Unknown, or anything but a
mapping with `schedule`, `deadline`, `grain` and `ready_when` all set (an empty
`ready_when` is unset; `none` = nothing beyond the checks the close always runs) →
BLOCKED: are reported months frozen, when, at what grain, and what else must hold first?
(Q-F1; the org's process, no default).

## Readiness and entering a phase

A component is **effectively ready** when it's READY and everything it needs is
effectively ready. **Ready to build** lists the effectively ready components outside
Phase 0, in build order. A BLOCKED optional track (one vendor, one cluster) never holds
up the core path. Whether a component was already built is STATUS.md's business, not
the checker's.

Entering a component means it's effectively ready. Entering a phase means all its core
components are; a phase with no core component (A, B) needs at least one ready
component.

## Output and exit codes

A table (component, phase, core, spec file, verdict, reasons and notes), then the
ready-to-build list; `--json` prints the same as data. Exit 2 on a malformed, missing or
unparsable profile. With `--entering <phase or component id>`, exit 1 when it isn't
ready to enter, else 0. A phase with no component, or a target that names nothing,
isn't ready to enter.
