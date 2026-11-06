---
name: cost-attribution
description: Guided build of a custom FinOps cost-attribution (cost allocation) system on FOCUS billing exports from AWS, Azure or Google Cloud, shaped to one organization's accounts, tags, clusters and SaaS vendors, in the user's own stack (the reference stack is S3, Athena and Lambda). Scans the user's bill first, interviews only for what the data can't show, records the answers in a validated profile, decides READY / BLOCKED / NOT_A_FIT per component, then builds phase by phase from specs and golden test cases, with a verification gate after each phase. Use when the user wants chargeback or showback, to allocate or split cloud spend by team, app or cost center, to replace or avoid a commercial FinOps or cost tool, to work with FOCUS exports, to split shared costs such as Enterprise Support, to attribute Kubernetes namespaces or SaaS vendor bills, to resume or refresh a build this skill started, or to try it on a sample bill before giving it billing access.
---

# Build cost attribution shaped like the org

You are guiding one person (and, where the org has one, a finance partner) through a
build that is mostly glue between a standard schema (FOCUS) and a serverless runtime.
The patterns are fixed; the shape is theirs. Your job is to find their shape in their
data, write it down, and apply the patterns to it — never to install a generic product.

The article explains why each pattern is right:
<https://mooracle.io/blog/build-custom-finops-tool/> (Parts 1–7). When the user asks
"why", point at the part; keep the session on the how.

## Invariants

These never vary by profile. Check generated code against them before showing it.

1. **Every split keeps its total.** `SUM(out) == SUM(in)` per cost column. Splitters move
   money; they never create or destroy it.
2. **Allocate at query time.** Org categories attach in a view, through a lookup. Never
   write `team` into a warehouse row.
3. **`EffectiveCost` for chargeback.** `BilledCost` reconciles invoices, `ListCost`
   measures savings. Never blend them.
4. **Rebuild the whole month.** Each `(billing_period, provider)` partition is cleared and
   rewritten from raw on every delivery. No deltas, no merges.
5. **Cap at yesterday.** Nothing dated today or later enters the warehouse.
6. **Every split is one fan-out.** A row that splits joins its weights inside the regular
   load and becomes one row per weight; a key's weights add up to 1.0. Fair-share,
   Kubernetes and vendor splits all work this way, through one `split_weights` table.
   Nothing is excluded from the load and added back later, so there is no exclusion to
   forget, and nothing gets counted twice.
7. **Every run proves its numbers.** Raw holds exactly the files the provider's
   manifest lists, and a blocking totals check compares the bill with both base tables
   and fails the run (and clears what it wrote) when they disagree. Once the views exist,
   every run and every close also check that `services_monthly` serves exactly
   `base_monthly` (the views check, which fails without clearing anything).
8. **Never lose cost; routing may fail.** When a split can't be done safely, the cost stays
   whole where it was billed. A skipped split is visible; a lost cost is silent.
9. **Raw reads are exact-month.** `WHERE billing_period = 'YYYY-MM'`, with the engine's
   scan cap on (a workgroup byte cap on Athena). All-history reads are allowed only over
   `base_monthly` (megabytes).
10. **Classify a shared column once, in a view.** `x_namespace` holds Kubernetes and SaaS
    workspaces alike; `split_by` says which. Never infer from the value's shape.

## Files you keep in the user's repo

Create `finops/` in Phase 0, or wherever the user wants it. Record the path as the first
`D-001`; below, `finops/` means that path.

- `finops/profile.yaml` — the org's shape. Start from `assets/profile.template.yaml`.
  Every phase reads it; nothing else holds org-specific decisions.
- `finops/DECISIONS.md` — append-only log. Never edit an entry; supersede it.

  ```markdown
  ## D-007 Enterprise Support is shared (2026-10-17)
  Source: D5 on 2026-09; Q-S1 answered by <user>
  Decision: split by first-party spend of the same month, per payer
  Supersedes: —
  ```

  Discovery results, answers, assumptions, approvals, and **negative decisions** ("tried a
  daily weights table; rolled back because the mix doesn't swing within a month; don't
  propose it again") all go here. Profile `evidence` fields point at these ids.
- `finops/STATUS.md` — one line per phase: state, gate result with the numbers, the
  commit SHA the gate passed at, and the profile fields the phase depends on.

  ```markdown
  | Phase | State | Gate | At | Depends on |
  | 3 base tables | passed | 2026-09 bill 120,000.00 = monthly 120,000.00 | a1b2c3d | providers, dimensions, routing.key, finance, period_charges, shared_costs |
  ```

## Run loop

### Entering

- `finops/STATUS.md` exists → **resume**: read STATUS, DECISIONS and the profile. For every
  passed phase, `git diff <its SHA> -- finops/profile.yaml`; if a field it depends on
  changed, reopen that phase **and every phase after it that reads its output** (1 → 2 → 3;
  3 → 4, A, B, 5; 4 → 5). Re-run the last passed gate against current data (data moves
  too). Continue from the first phase not passed.
- The user reports a change (re-org, new account family, new vendor, new cluster) →
  **refresh**: re-run the matching discovery queries, diff against the profile, re-ask
  only what moved, re-check, reopen dependent phases.
- The user wants to see it work first, or won't give billing access yet → the **trial**
  (`references/trial.md`): a synthetic month on DuckDB, no credentials, checked against
  an answer key. Nothing from it carries into a real build.
- Otherwise → **start** below.

### 1. Discover before asking

Read `references/discovery.md` and run its queries on the last closed month, against
the export where it already lands (a table over its files, or DuckDB over one
downloaded delivery when there is no engine), before any of the build exists. Show
every result. Record each as a `D-NNN`. Discovery reads; it changes nothing in the
account, not even a tag activation.

No export yet → the **bootstrap path**: turn the export on (`references/focus-export.md`),
run the non-data intake rounds (Org, Runtime, Categories, Finance), build the stack
against the expected layout, and run discovery in the next session once the first
delivery has landed. Offer the trial (`references/trial.md`) while waiting.

### 2. Interview for the gaps

Read `references/intake.md`. The stop-or-assume rule:

- the answer moves **money or ownership** → leave it `null`; the component stays BLOCKED;
- it's a **name, location or tool** → use the default, add an `assumptions:` entry and a
  `D-NNN`, move on.

Never fill a fact with a plausible guess, a fixture or a value from an example.

### 3. Check and agree the plan

Write the checker `references/plan.md` describes into `finops/` (once, with a test per
rule) and run it on the profile. It prints one verdict per component and what is ready
to build (READY all the way down its dependencies). Show it.
For each BLOCKED line, name who can answer and the exact probe. For each NOT_A_FIT,
say the simpler alternative out loud — including "don't build this" when the org line
says so. Get explicit approval of the plan; record it as a `D-NNN`.

### 4. Build, phase by phase

For each phase in the map below, in order (the optional tracks A and B after Phase 4,
only when READY):

1. **Entry check.** The checker with `--entering <phase>` exits 0, and every phase it
   depends on is passed in STATUS.md.
2. **Read its spec** (`references/<file>`, the phase map's Spec column): the contract,
   the invariants, and the edge cases that bite when improvised.
3. **Write it in their stack.** Build the phase's components in the user's engine and
   language (Athena, BigQuery, Snowflake, Databricks, dbt, …) and in their repo's style,
   shaped by their profile: an adapter per provider from `providers.md`, only the splits
   the profile needs, only the columns it routes on. The specs say what must hold and
   the cases what must come out; the code is yours to write. Take the most care where a
   mistake is silent: the weights, the spread over days, the totals checks, the currency
   guards.
4. **Port the golden cases** (`cases/*.yaml`; how, in `references/testing.md`) that
   apply to what this phase built, and add a case for each org-specific adaptation. A
   phase isn't built until its cases pass.
5. **Deploy and run the gate** against their real data. Gates are SQL that return zero
   rows when healthy. Show the numbers.
6. **Record** the result, the SHA and the dependencies in STATUS.md.

Gate report, every time:

```
Gate 3 (base tables), 2026-09, aws: PASSED
  bill 120,000.00 · monthly 120,000.00 · daily 120,000.00 · drift 0.00
  support: 1 row → 12 accounts, weights sum 1.000000
```

A failed gate stops the build. Debug with the spec's failure table (every check's
problem names are in `reconciliation-ops.md`, "Failure modes"), fix, re-run.

### Phase map

| Phase | Builds | Profile fields | Spec | Cases | Gate |
|---|---|---|---|---|---|
| 0 | discovery, interview, the checker, plan | all | `discovery.md`, `intake.md`, `plan.md` | — | plan approved (`D-NNN`) |
| 1 | FOCUS exports | `providers` | `focus-export.md` | — | a full month queryable (D0) |
| 2 | storage layout, engine, tables, and the deliveries that fill raw (copy or poller, staging, the ETL's publish step) | `runtime`, `providers` | `stack.md`, `data-model.md`, `pipeline.md` ("Delivery") | — | a published month matches its manifest; a raw month query scans a fraction of that month's files |
| 3 | adapters, base tables with proration and shared-cost splits, the ETL's rebuild | `providers`, `dimensions`, `routing.key`, `finance`, `period_charges`, `shared_costs` | `providers.md`, `data-model.md`, `splitters.md`, `pipeline.md` | `adapters`, `load`, `share`, `checks` | bill == monthly == daily per payer on a closed month |
| 4 | rules, lookup, views | `categories`, `routing` | `rules-engine.md` | `rules`, `views` | categories appear; Unallocated reported |
| A | SaaS splits | `saas.vendors` | `saas.md` | `vendor` | the vendor shows `split`; totals hold |
| B | Kubernetes namespaces | `kubernetes` | `kubernetes.md` | `k8s` | weights clean; coverage ≥ target |
| 5 | reconciliation, ops, close | `finance`, `org.finance_partner` | `reconciliation-ops.md`, `testing.md` | `checks` | where the org enters invoice totals (Q-F5), a closed month passes the invoice check for every provider, its result stored with the month; a deliberately broken split fails the run on a dev copy; a walk-through of the numbers with the finance partner, or the owner where there is none |

`references/pipeline.md` has the processing order these pieces run in at runtime.

## Degrees of freedom

- **Fixed** (whatever the engine): the invariants, and what the cases say must come out
  (`adapters.yaml` excepted: it shows one profile's choices on the exports' raw shapes):
  the proration rules, the fan-out with weights that add up to 1.0, the producers' gates
  (per payer, positive spend, whole-month vendor data), the checks and what they stop
  on, the run order and clearing on failure, refusing unfilled query parameters.
- **Theirs** (write freely to match their codebase): the engine and language, file
  layout, infrastructure as code, packaging, scheduling, dashboards, CI, and the names of
  files, functions and infrastructure (the table, column and problem names the cases use
  are fixed).

When you are unsure which side a change is on, it's fixed.

## When the user pushes back

`references/pushback.md` has the usual ones ("allocate in the ETL", "map 100%", "one
framework for every vendor", "per-pod precision") with the answer and where the
article argues it.

## Done means

- Every closed month passes the totals check for every provider, through every split,
  and, where the org enters invoice totals, the invoice check.
- Whoever reports cost (the finance partner, or the owner where there is none) answers
  "what did team X cost last month?" from a view, without new SQL.
- A rule change is a pull request: CI expands, the reviewer sees what moves, merge
  re-categorizes all history with no rebuild.
- Unallocated is alerted on as an amount, in its currency, and trending down.
- The run cost is small and measured (on the reference stack, about $10 a month for S3,
  Athena and Lambda together at 15 million bill lines a month; Athena's share is in the
  workgroup's scan metrics).
- STATUS.md shows every phase the plan builds passed with a SHA; DECISIONS.md explains
  every choice a newcomer would question.
