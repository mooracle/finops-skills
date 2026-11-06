# Intake — the questions data can't answer

Run discovery first (`discovery.md`). Ask only for intent, ownership and facts that live
outside the bill.

## Contents

- How to ask
- The questions: Org (O) · Runtime (R) · Categories (C) · Routing (G) · Shared costs (S)
  · Kubernetes (K) · Vendors (V) · Finance (F)
- After the last round

## How to ask

- **Rounds of at most five questions, one topic per round.** Show the discovery output
  the round is about, then ask. Factual lists (which vendors, which clusters) can be one
  question answered in shorthand.
- **Every question carries one line on why it matters, and its `Default` as the
  recommended answer.** A `—` default means there is none to give: "you decide" there
  counts as unknown (`On unknown`). On a question with a default, "you decide" means:
  give it, say what it changes, and get a yes.
- **Write answers down as you go**: the value into `finops/profile.yaml`, the exchange as
  a `D-NNN` entry in `finops/DECISIONS.md`. The profile field's `evidence` is that id.
- **Stop or assume** — the column `On unknown` below decides:
  - **block**: the answer moves money or ownership. Leave the field `null`; the
    component it gates stays BLOCKED. Never fill it with a plausible guess.
  - **assume**: names, locations, tools. Use the default, add an `assumptions:` entry in
    the profile with the reason, and move on. The user can overturn it later.
- Skip a question whose `When` condition is false.

## Org

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-O1 | Which team runs this after the build (a couple of hours a month)? | A tool without an owner rots. "Nobody" means buy. Ask who, never what they know: the code is generated and reviewed like any other. | — | block | `org.owner` (`none` = nobody) |
| Q-O2 | Is there someone in finance who should shape this and review the numbers? | Optional, and it depends on the org: a startup often has nobody yet. Where there is one, they answer the Finance round and the Phase 5 walk-through is with them; where there isn't, the owner (Q-O1) does both. | — ("none" is an answer) | assume none | `org.finance_partner` (`none` = nobody) |
| Q-O3 | Which of these do you need: a second cloud, SaaS bills in the same view, Kubernetes namespaces, rules reviewed in Git, categories your cloud console can't express, cost by team in the dashboards your teams already watch? | If none, the free option (AWS Cost Categories / Azure cost allocation rules) is enough. Say so. | — | block | `org.why_build`, in the question's order: `multi_cloud`, `saas_bills`, `kubernetes`, `rules_in_git`, `own_categories`, `own_dashboards` (`[]` = none) |

## Runtime

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-R0 | Which engine, storage and orchestrator should it run on? (Athena + S3 + Lambda, BigQuery + Cloud Run or Composer, Snowflake + tasks, Databricks + jobs, dbt on any of them…) | Everything is built in it; the reference is Athena + S3 + Lambda. | the warehouse they already run, next to the cloud that bills most | assume | `runtime.engine`, `runtime.storage`, `runtime.orchestrator` |
| Q-R1 | Which account (or project) and region does the pipeline run in? | Cross-account export access, data residency. | a tooling account, the export's region | assume | `runtime.account`, `runtime.region` |
| Q-R2 | Resource name prefix? | Everything is named from it. | `finops` | assume | `runtime.prefix` |
| Q-R3 | IaC tool? | Generated infra code. | Terraform | assume | `runtime.iac` |
| Q-R4 | Dashboard tool? | Only that it can read the engine. | whatever they have | assume | `runtime.dashboard` |

## Categories

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-C1 | What does finance report spend by? | These become the rule outputs and the view's columns. | `team` + `environment` | block | `categories.outputs` |
| Q-C2 | Is there a hierarchy above team (division → cost center → team), and who keeps it? | Goes in a separate table joined in the view: a team moving to another division is a one-row edit there; a team that splits or merges is a rule change. | none yet | assume | `categories.hierarchy` (`owner`: who keeps it) |
| Q-C3 | Environment values? | Rules can only assign these; enforce them with an enum on `environment` in the rules schema if the list is fixed. | Production, Development, Test, Sandbox | assume | `categories.environment_values` |

## Routing

*When:* always, after D2 and D3 (and D8 on Azure). Q-G3 fixes what the Phase 3 loads group by, so ask it before anything is built.

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-G1 | (Show the D2 coverage.) Is this naming convention real policy, and who provisions accounts? | A real convention means one regex rule covers future accounts. | — | assume "not policy" | `routing.account_convention.policy` |
| Q-G2 | Which accounts or subscriptions do several teams share, and what separates them there: resource group, tag or namespace? | Those accounts need a second key; everything else routes on the account. | — | block (the rules phase) | `routing.accounts_shared_by_teams` |
| Q-G3 | (Show the default key and the D3 tags.) Besides account, service, namespace and those tags, does anything else decide who owns a cost: a region, a resource group, another tag? | Rules only see the base tables, never raw FOCUS: whatever a rule must match is grouped in Phase 3 or is out of reach. Adding a field later rebuilds every month and migrates the lookup. | account, service, resource group, namespace, plus each tag D3 kept | assume | `routing.key` |

## Shared costs

*When:* D5 returned candidates.

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-S1 | (For each D5 candidate.) Should this be spread across teams by their spend? | Each shared charge gets its own weights in `split_weights`; spreading it, or leaving it on the account that pays, is the decision. | spread by each account's first-party spend in the same month | block (per cost) | `shared_costs[]` |

## Kubernetes

*When:* D7 found cluster spend worth splitting (more than a rounding error of the bill).

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-K1 | (Show D7.) Which pools are dedicated to one namespace and which are shared? | Dedicated pools route by tag; shared pools need metrics. | infer from pool names, confirm | block (that cluster) | `kubernetes.clusters[].pools` |
| Q-K2 | Is there a Prometheus-compatible endpoint, or Datadog with the cluster agent, that the pipeline can query, and does the node-pool label reach it (`kube_node_labels`, or a Datadog tag)? Does it serve other clusters too, and what picks this one? | Without the pool label, requests can't be tied to a pool; without the cluster's selector, a shared endpoint mixes clusters. | — | block (shared pools) | `kubernetes.clusters[].metrics` (`source: prometheus` for any Prometheus-compatible endpoint, `datadog`, `none` when there is neither) |
| Q-K3 | (Show D7's pools and their cost.) Which pools are GPU pools? | They split by GPU requests alone; getting it wrong moves the most expensive nodes to the wrong teams. | ask whoever runs the cluster | block (shared split) | `kubernetes.clusters[].pools.gpu` (`[]` = none) |
| Q-K4 | Who carries system namespaces and the buckets metrics can't place (`<cluster>/pool:<pool>`, `<cluster>/cluster:shared`)? And how much of cluster spend should reach a namespace? | Someone pays for what metrics can't place; that's an ownership decision. | — for the owner; 90% | block the owner (that cluster); assume 90% | `kubernetes.shared_owner`, `kubernetes.coverage_target` |
| Q-K5 | (Show D7's load balancers by namespace.) Is there a shared ingress controller (one load balancer in front of many namespaces)? Which namespace runs it, on which pool? | Its load balancer is tagged with the controller's namespace but serves the pool's tenants; it splits with the pool (article Part 6). | none unless D7 shows one | assume | `kubernetes.clusters[].shared_ingress` |
| Q-K6 | Who carries the idle capacity of shared pools: the tenants who reserved the pool (spread over them by their weights), or the cluster's owner (an `<cluster>/idle:<pool>` bucket)? | Requests leave part of every pool unreserved; spreading it raises every tenant's number, a bucket makes it one visible line for `shared_owner`. | spread | block (shared split) | `kubernetes.idle` |
| Q-K7 | Should a namespace's share follow each day's requests (daily weights), or the month's mix (monthly)? | Daily weights show a spike the morning after and never move a finished day; monthly ones smooth it over the month and restate every day of the month until it ends. | daily | block (shared split) | `kubernetes.weights` |
| Q-K8 | (Prometheus only. Show the step probe: `kubernetes.md`, "The step".) Which shared or GPU pools run short jobs (batch, CI runners, cron) that often start and end within an hour? | An hourly sample sees such a job as a full hour or not at all, and one that starts and ends between two samples reserves for free. These pools are sampled every 5 minutes, at twelve times the samples per query. | recommend none, except a pool where the probe moves a namespace's weight by a percentage point or more (the skill's threshold, not a rule) | block (shared split) | `kubernetes.clusters[].pools.short_jobs` (`[]` = none) |

Probe for Q-K2 (read-only, give it to whoever has cluster access): count the nodes in
`kube_node_labels` by the candidate label, keeping only nodes where it's set. The right
label has one value per pool, and every node carries it.

Read the values, not just the count: one row per pool name you expect. An empty result
means kube-state-metrics isn't exporting the label and needs
`--metric-labels-allowlist=nodes=[<label>]`. That is the BLOCKED reason, verbatim.
(Without the `!=""` filter, a missing label still returns one unlabeled row and looks
like success.)

Run the probe with the cluster's selector. To find it, list the label names on
`kube_node_labels` (`/api/v1/labels?match[]=kube_node_labels`), then the values of any
cluster label among them, such as `cluster`, `k8s_cluster_name` or `clusterArn`
(`/api/v1/label/<name>/values?match[]=kube_node_labels`). Tie a value to the profile
cluster by its nodes, never by how the name looks: the node names under that value must
be the cluster's own (`kubectl get nodes`, or ask whoever runs it). That value is the
`cluster_selector`, such as `cluster="eks-prod-eu"`. No cluster label, and the user
confirms the endpoint serves only this cluster: `""`. Several clusters and no label that
tells them apart: BLOCKED, "add a cluster external label to each cluster's remote write".
Last, read a few `namespace` and `pod` values on `kube_pod_container_resource_requests`:
workload namespaces and pods are right; kube-state-metrics' own on every series means the
scrape moved them (`kubernetes.md`, "One cluster, under the bill's name"). Where the
cluster runs a mesh or other native sidecars, check that
`kube_pod_init_container_info{restart_policy="Always"}` returns series: none means
kube-state-metrics is older than 2.11, or a keep-list drops it.

On Datadog the probe is a metrics query: the node allocatable metric grouped by the
candidate tag, over the last hour. One series per pool means the label is mapped. One
series with no tag value means the agent isn't mapping node labels to tags
(`kubernetesResourcesLabelsAsTags` under `nodes`, or the legacy `nodeLabelsAsTags`, in its Helm values); that is the BLOCKED reason (`kubernetes.md`,
"Datadog instead of Prometheus"). Group it by `kube_cluster_name` too: this cluster's
value is its `cluster_selector`, written `kube_cluster_name:<value>`.

## Vendors

*When:* always.

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-V1 | Which vendors invoice you directly (not through a cloud marketplace), and should their cost appear in the same views? | They have no row on the cloud bill: without an ingest shim their cost is simply absent. | — | block | `saas.vendors[].billed = direct` |
| Q-V2 | (Per vendor.) How many teams use it? | One team → a rule routes the whole row: no split and no breakdown shim (a directly billed vendor still needs a shim that loads its charges from the API, so the cost lands at all). | — | block (that vendor) | `saas.vendors[].shape`, `used_by_teams` |
| Q-V3 | (Per vendor, if split.) The six discovery facts in `saas.md` | READY / BLOCKED / NOT_A_FIT comes from them. | — | block (that vendor) | `saas.vendors[].facts` |

## Finance

Ask these of the finance partner where there is one (Q-O2), else of the owner (Q-O1).

| Id | Question | Why it matters | Default | On unknown | Profile |
|---|---|---|---|---|---|
| Q-F1 | How does finance use these numbers today: a showback report, chargeback entries per team or cost center, budgets against actuals, an allocation it already runs (headcount, fixed internal rates)? On what calendar? Once a month is reported, must it stay frozen? If so: when, by which deadline, at what grain, what else must be true first (vendors split, someone's sign-off), and who may freeze a month despite a known problem? | The views serve any of these. A freeze always waits for the views check, and for the invoice check too where the org enters invoice totals (Q-F5; article Part 4). The rest of what's built for finance (a frozen file, its grain and day, what else the close waits for) follows the org's process; a default would quietly become the process. | — (show the options; "no frozen month" is a valid answer, and so is "nothing else" for what must be true first) | block (the close only) | `finance.uses`, `finance.frozen_close` |
| Q-F2 | Unallocated alert threshold per month, in the reporting currency? | Mapping the long tail is endless; alert on an amount, not a percentage. | ~1% of monthly spend, rounded; one per provider when providers stay in their own currencies | assume | `finance.unallocated_alert` |
| Q-F3 | Should tax and adjustments be attributed? | Changes every total and what reconciles to the invoice. | recommend each provider's usual list in `providers.md` ("Charge categories"), tax out and no side on `Credit`: whether credits flow to the teams is the org's call (article Part 2); show each charge category the bill carries | block (base tables) | `finance.charge_categories` |
| Q-F4 | (Show each provider's billing currency from D0.) Which currency does finance report in: **USD for everything**, **one local currency** (convert what bills in USD, like AWS, through a monthly rate table), or **each provider in its own** (nothing converts; dashboards group by currency)? Who keeps the rate table, if one is needed, and which rate covers the month that is still open? | Every number finance sees is in this currency; converting at the wrong place, or summing two currencies, makes every total wrong. Azure and Google Cloud exports can convert to USD themselves; anything else needs a rate table. | if every provider bills in one currency: that one, nothing converts; the open month's rate: — | block (base tables) | `finance.reporting_currency`, `finance.fx_rates` |
| Q-F5 | Who enters each month's invoice totals, or pulls them from the provider's invoice API where there is one? Roughly when do the month's invoices arrive? | The invoice check compares the bill with these totals: it's what ties the bill to what the org paid (article Part 2). Without them there is no invoice check, and nothing blocks. | the owner (Q-O1); the check alarms from the day they say the invoices arrive, and until then runs once the month's rows are in `invoice_totals` ("none" is an answer: no invoice check) | assume | `finance.invoice_totals` (`{by, day}`; `none` = no invoice check) |

## After the last round

Then check and agree the plan (`SKILL.md`, step 3). BLOCKED optional tracks don't
stop the core path.
