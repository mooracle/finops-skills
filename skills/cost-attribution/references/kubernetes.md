# Track B — Kubernetes namespaces

Run this track per cluster the checker marks READY (`k8s:<id>/tags`, and
`k8s:<id>/shared` when shared or GPU pools exist; `<id>` is the cluster's profile id). The bill stops at the resource (a VM, a
disk, a load balancer) and teams own namespaces. This track writes the namespace a
row's cost served into `x_namespace`, written `cluster/namespace`, and the rules route
namespaces to teams like any other key (article Part 6).

Aim for about 90% of cluster spend on real namespaces (`kubernetes.coverage_target`; the
article gives no number, only "fix the biggest bucket first").
The rest stays in visible buckets owned by `kubernetes.shared_owner`; it isn't worth a
finer model.

## Contents

- How a row gets its namespace
- Metrics: requests and capacity
- Weights: the dominant resource
- GPU pools
- Coverage
- Other distributions
- Gate and failure modes

## How a row gets its namespace

The adapter reads three things from a row's tags (`k8s_cluster`, `k8s_pool`,
`k8s_namespace`) and `bill_v` decides, first match wins:

| Row | Signal | `x_namespace` | How |
|---|---|---|---|
| Disk from a PVC, load balancer from an Ingress or Service | a namespace tag **and** a cluster | `cluster/<ns>` | whole, by tag |
| Load balancer of a **shared ingress controller** | its namespace is the profile's `shared_ingress`; the adapter gives it the controller's pool instead | `cluster/<ns>`, one per namespace on that pool | the pool's weights |
| VM (and its boot disk) in a **dedicated** pool | pool tag; `k8s_pools` names the namespace | `cluster/<ns>` | weight 1.0 |
| VM (and its boot disk) in a **shared** pool | pool tag; metrics | `cluster/<ns>`, one per namespace | its weight |
| VM in a **GPU** pool | pool tag; GPU metrics | `cluster/<ns>`, one per namespace | its GPU weight |
| The idle part of a shared or GPU pool whose `idle` is `bucket` | as above | `cluster/idle:<pool>` | what the namespaces' weights leave |
| A pool with no weights (no metrics, nothing requested, not in `k8s_pools`) | pool tag | `cluster/pool:<pool>` | whole |
| Anything else with the cluster tag (the control plane, say) | cluster tag only | `cluster/cluster:shared` | whole |

Network cost follows the same table. AWS bills cross-AZ traffic on the instances at both
ends, each on its own row with its own id, so each half lands with its node and its pool.
A NAT gateway
carries no Kubernetes tag: tagged with the cluster by hand it lands in
`cluster/cluster:shared`; untagged, it's an ordinary account row for the rules.

Every row needs a cluster to count as Kubernetes at all: a namespace tag alone says
which namespace, not which cluster, and stays unattributed. The adapter reads the
cluster's forms in the order `providers.md` gives (EBS CSI volumes carry
`KubernetesCluster` only when the driver runs with its cluster id set). D7 shows which
of these the cluster's resources actually carry; add whatever else it finds.

**EKS Auto Mode** has no managed node groups: the pool is a Karpenter `NodePool`, and
AWS tags what it launches itself. The cluster is `eks:eks-cluster-name`, the pool
`eks:kubernetes-node-pool-name`, and its load balancer controller writes
`ingress.eks.amazonaws.com/stack` or `service.eks.amazonaws.com/stack` as
`<namespace>/<name>`, which is the namespace tag for those rows. None of them is active
by default (D7: `tags.activated`); until the user activates them and requests the
backfill, the cluster's nodes are plain EC2 rows for the rules.

What one namespace asked for (a disk, a load balancer) is tag-based wherever it runs,
even in a shared pool. A shared ingress controller is the one exception. Its
load balancer carries the controller's own namespace tag, but it serves every tenant of
the pool the controller runs on, so it splits with that pool (article Part 6). The
profile names it (`shared_ingress`, Q-K5), and the adapter applies it: on rows of that
cluster whose namespace is the controller's, `k8s_namespace` becomes NULL and `k8s_pool`
the controller's pool, so `bill_v` gives them the pool's weight key like any node in it.
Add a case for it next to the adapter's. Ingress objects that each get a load balancer
of their own stay tag-based.

Pool kinds come from the profile, published as `k8s_pools` (`data-model.md`). Never infer "GPU"
or "dedicated" from a pool's name in SQL. A pool is identified with its account and
region: EKS cluster names are unique only per account and region, so two accounts (or two
regions) can both run a `prod` cluster with a `general` pool, and their tenants must not
share weights. The metrics sync writes the same three ids on every row. In the profile,
each cluster has its own `id` and a `name`: the cluster tag's value, which is what
`k8s_pools.cluster` and `x_namespace` use. A name repeated in two regions of one account
needs `regionid` in `routing.key` (Q-G3); the checker blocks Phase 3 until it's there.

Check the boot disks carry the pool tag (D7 shows untagged cluster spend). An instance
having the tag doesn't mean its volumes do (Karpenter tags them by default; on an EKS node
group the launch template has to propagate tags to volumes), and an untagged boot disk
lands in `cluster:shared` instead of the pool.

Each row is classified by its own tags. A resource re-tagged mid-month splits by its old
pool on the days before and its new pool after, which is what happened.

## Metrics: requests and capacity

A daily sync asks Prometheus, per cluster and pool, for two things over the last
24 hours and writes them to `k8s_metrics`, one CSV per cluster and day. **Its pools are
the `shared` and `gpu` rows of `k8s_pools`**, read from that table (or generated from the
same profile entries that write it), never a list of its own: a second list drifts, and
a GPU pool left out of it gets no weights and sits whole in its `pool:` bucket.

- **requests**: CPU, memory and GPUs requested by each namespace's running pods on the
  pool's nodes, one row per namespace;
- **capacity**: the pool's allocatable CPU, memory and GPUs, one row with an empty
  namespace.

Both come from kube-state-metrics, which doesn't know about pools: pick the pool's nodes
by the node-pool label (Q-K2; on EKS `eks.amazonaws.com/nodegroup` for managed node groups, `karpenter.sh/nodepool` for Karpenter and Auto Mode, exported only once
you allow it). Node names won't do. `kube_node_labels` writes a camelCase label name in
snake_case: `eks.amazonaws.com/capacityType` is `label_eks_amazonaws_com_capacity_type`.

**One cluster, under the bill's name.** The sync writes `cluster` as the profile
cluster's `name` (the cluster tag's value), never a name read from the metrics, so its
weights reach the bill's rows and the namespaces land as `<cluster>/<namespace>`. It
reads only that cluster's series: where one endpoint serves several clusters (Amazon
Managed Service for Prometheus, Mimir, Grafana Cloud, Thanos, Google's or Azure's
managed Prometheus), every selector in every query carries the cluster's
`metrics.cluster_selector` (Q-K2), such as `cluster="eks-prod-eu"`. Without it the
clusters' namespaces mix into one split, and two clusters with a pool, or a node, of the
same name add up. The label is whatever the setup writes, and its value is often not the
bill's name: `cluster` is common, Grafana's chart also writes `k8s_cluster_name`, AWS's
examples set `clusterArn` by hand, and a Prometheus queried directly shows no external
labels at all. An endpoint that serves this cluster alone takes an empty selector. The
endpoint is `metrics.endpoint`; its credentials live with the other secrets (`stack.md`,
"Secrets"), and its calls follow `saas.md`, "Calling the vendor's API".

The `namespace`, `pod` and `container` on these series must be kube-state-metrics' own.
A scrape that sets target labels of those names moves kube-state-metrics' ones to
`exported_namespace`, `exported_pod` and `exported_container`: a ServiceMonitor or
PodMonitor without `honorLabels` does, and so does Google's managed Prometheus unless
kube-state-metrics is scraped by a `ClusterPodMonitoring` whose `targetLabels.metadata`
leaves them out. The pool then lands on kube-state-metrics' own namespace, and the
Running filter matches its pod instead of the tenants'. Fix the scrape, or read every
`exported_*` label in place of its plain one.

Any kube-state-metrics series can arrive twice (a second replica, a recreated pod), and a
duplicate breaks a one-to-one match and doubles a sum. Take each series' `max` by its own
labels before any match or sum: namespace, pod, container and resource for requests;
namespace and pod for the phase; node and resource for allocatable; node and the pool
label for node labels.

Build each number the same way for CPU (`resource="cpu"`), memory
(`resource="memory"`) and GPUs (`resource="nvidia_com_gpu"`):

- **Requests**: `kube_pod_container_resource_requests` and the native sidecars' series
  (below), joined per container with `or`, kept only for pods whose
  `kube_pod_status_phase{phase="Running"}` is 1 (match on namespace and pod). Sum by
  namespace and node, then keep the pool's nodes: match on node against
  `kube_node_labels` where the pool label equals the pool's `pool` in `k8s_pools` (the
  pool tag's value; a different value in the metrics is a labelling fault, not something
  the sync maps). Sum by namespace.
- **Capacity**: `kube_node_status_allocatable` on the same nodes, summed over all of them.
- **Hours**: sample each series at the pool's `step` over the 24 hours (a `[24h:1h]`
  subquery, `[24h:5m]` at `5m`, evaluated at `23:59:59` UTC, the bill's day: at midnight, every version takes the next day's 00:00 as
  the last sample, and Prometheus 2 keeps the day's own 00:00 too, 25 samples in all at `1h`) and add the samples up. Each sample stands for one step, so the sum, divided by 12 at `5m`, is core-hours,
  GiB-hours or GPU-hours. Requests and capacity take the same step ("The step", below).
- **Rows**: per pool and day, one row per namespace and one capacity row with an empty
  namespace, each with the pool's `provider`, `subaccountid`, `regionid` and `cluster`
  (`k8s_metrics` in `data-model.md`).

Memory comes back in bytes; write GiB-hours. One day per run, a cursor, and an event
with a date for backfill; Prometheus retention bounds how far back it can go. A day the
endpoint can't answer (nothing matches the selector, or it's past retention) writes
nothing for that cluster and is logged; the cursor moves only past days that were written.

**The step.** An hourly sample sees a job of twenty minutes as a full hour or not at
all, and one that starts and ends between two samples reserves for free. So the step is
per pool (Q-K8): `step` on the pool's `k8s_pools` row is `5m` for a pool in the
cluster's `pools.short_jobs` (batch, CI runners, cron), `1h` for every other. Five
minutes costs twelve times the samples per query, and a job shorter than that can still
slip between two. The Q-K8 probe: for each shared or GPU pool, build one recent day's
weights at both steps and show them side by side. Where a namespace's weight moves by a
percentage point or more, the hourly samples miss or overcount its pods: recommend that
pool for `short_jobs`. The threshold is this skill's recommendation, shown with the
figures, not a rule: the user answers Q-K8. Test the sync with a fake endpoint that
evaluates the step: a job requesting one core from 02:03 to 02:23 writes a third of a
core-hour on a `5m` pool and nothing on a `1h` one. Datadog needs no step: its sync
reads by minute (below).

**Native sidecars.** An init container with `restartPolicy: Always` runs beside the app
for the pod's whole life, and the scheduler reserves its requests like an app
container's. Meshes inject their proxies this way (Istio by default from 1.27 when every
node runs Kubernetes 1.33 or later, Linkerd from 2.20), and
`kube_pod_container_resource_requests` leaves them out. Add
`kube_pod_init_container_resource_requests`, kept only for the containers
`kube_pod_init_container_info{restart_policy="Always"}` names (match on namespace, pod
and container), and join it to the app containers' series with `or`, per container,
before any sum: their `container` labels differ, so `+` matches nothing. Leave ordinary
init containers out; they finish before the app starts. Both series need a recent
kube-state-metrics, and an older one fails without an error: before 2.3 the
init-container requests have no `node` label, so the pool match drops them; before 2.11
the info series has no `restart_policy`, so nothing passes the filter.

kube-state-metrics is the default source. Where the scheduler's own
`kube_pod_resource_request` is scraped (`/metrics/resources`), use it instead of both
kube-state-metrics series, never beside them: one series per pod with what the scheduler
reserves (sidecars, the init container that sets the peak, pod overhead). It spells
`resource` and `unit` its own way (`nvidia.com/gpu`, `cores`, `bytes`), and the Running
filter still applies.

**Datadog instead of Prometheus** (`metrics.source: datadog`): a cluster that runs only
the Datadog agent has the same two numbers behind a different API, and the sync's
contract (`k8s_metrics`, the rows above) doesn't change. Requests per container come
from the agent (`kubernetes.cpu.requests`, `kubernetes.memory.requests`, which it
reports for live containers only, native sidecars included from agent 7.53 on, so the
Running filter is built in), allocatable per
node from the cluster agent's kube-state core check
(`kubernetes_state.node.cpu_allocatable`, `kubernetes_state.node.memory_allocatable`).
Check each name and tag in the metrics explorer before writing the sync. Scope every
query to the cluster with its `kube_cluster_name` tag (`cluster_selector`): the agent
lowercases the name and turns `_` into `-`, so it can differ from the bill's. The node-pool
label reaches Datadog only when the agent maps it to a tag
(`kubernetesResourcesLabelsAsTags` under `nodes`, or the legacy `nodeLabelsAsTags`, in
its Helm values; the Q-K2 probe in `intake.md` checks it).
Hours: Datadog rolls each series (a container, a node) up in time before it sums them,
so a one-hour rollup counts a pod that ran ten minutes as a full hour. Query the day by
minute (`.rollup(avg, 60)`: 1,440 points, inside the 1,500-point limit), add the points
and divide by 60. The minute already catches a short job, so this sync ignores the
pool's `step`. GPU requests and capacity come from the same core check
(`kubernetes_state.container.gpu_requested`, `kubernetes_state.node.gpu_allocatable`).
That request has no Running filter: a finished pod counts until it's deleted, so keep
only the pods whose `kubernetes_state.pod.status_phase` carries `pod_phase:running`.
Without these metrics, GPU pools stay in their bucket (Q-K3). The API and application
keys live in SSM like a vendor's (`stack.md`, "Secrets").

## Weights: the dominant resource

`namespace_weights` computes, per pool and day (`kubernetes.weights`, Q-K7; see
"Daily weights" below):

```
cpu share   = namespace's cpu_h / pool's cpu_h
mem share   = namespace's mem_h / pool's mem_h
binding     = the larger of the two          what fills the pool first
weight      = binding / sum of bindings      over the namespaces on the pool
```

The pool's figures are its capacity row. A pool with no capacity row for a day gets
no weights for that day and its rows stay in its bucket: the sync is broken or the
pool is mislabeled, and the bucket shows it. A resource whose capacity is zero gives a
share of 0.

**Requests, not usage.** The scheduler reserves against requests, used or not. A pod
with no requests gets no weight and its namespace pays nothing for it: suggest default
requests per namespace (a `LimitRange`), the cluster team's change. A short job's
request counts only if a sample sees it: hourly samples miss one that starts and ends
between them, so a pool of short jobs takes the `5m` step ("The step").

**Idle capacity: spread, or a bucket (Q-K6).** The bindings rarely add up
to 1.0 (0.72 in the article's example; mixed workloads can pass 1.0, when one namespace
fills the memory and another the CPU). The pool's `idle` (`k8s_pools`, from
`kubernetes.idle`, Q-K6) decides who carries the rest:

- **`spread`**: weight = binding / sum of bindings. The namespaces carry
  the whole pool, idle capacity included: requests decided how big it had to be.
- **`bucket`**: weight = binding / the larger of 1 and the sum of bindings, and the
  remainder, 1 − the namespaces' weights, is one more weight row to
  `<cluster>/idle:<pool>`, written only when it's above zero. In the article's example
  that's 0.42, 0.20, 0.10 and 0.28 idle. A pool whose bindings pass 1.0 has nothing
  left for the bucket, so it splits as in `spread`, even when some CPU or memory is
  still unreserved (CPU/memory shares of 60/5 and 5/60 bind at 1.2, both at 65%): the
  tenants carry that idle. The rules route the idle bucket to `shared_owner`.

Either way a pool's weights add up to 1.0, so the weights check doesn't change.

Worked example (article Part 6): requests of 480/2,016, 240/960 and 120/384
core-hours/GiB-hours on a 1,200/4,800 pool bind at 0.42, 0.20, 0.10, giving weights
0.583, 0.278, 0.139. A day's $1,000 of EC2 becomes $583, $278 and $139, and the pool's
$1,065 comes back to $1,065. `cases/k8s.yaml` ("the article's worked example") uses
exactly these numbers.

System namespaces (`kube-system`, monitoring) are tenants by default: they get weights
like anyone and the rules route them to `shared_owner`. Spreading them over everyone
else instead is a policy choice: drop them from the bindings and record a `D-NNN` entry.

**Daily weights.** Each day's `k8s_metrics` rows give that day's weights, written to
`split_weights` with `dt` set, and both base loads join a cluster row to the weights
of its day (`DATE(chargeperiodstart)`; a row covering several days joins its first
day). A day with no capacity row lands in the pool bucket for that day only. This is
`kubernetes.weights: daily` (Q-K7), written to every `k8s_pools` row. `monthly`
sums the month's metrics per pool into one set of weights and writes it for every day
of the month before today, so the join is the same, and a day with no metrics borrows the month's
mix.

## GPU pools

On a GPU node the card is most of the price, so a GPU pool splits by GPU requests alone,
with no `max()` against CPU and memory (article Part 6). It needs its capacity row too,
with GPUs in it: a GPU pool with no capacity row, or a GPU capacity of zero, gets no
weights. So does one where nothing requested a GPU in the month: it stays in its
`pool:` bucket, visibly.
Never smear it over CPU tenants. If that bucket is big, find out how the workloads reach
the GPUs (some platforms expose the device without a request) and record what you do
about it as a `D-NNN` entry.

## Coverage

How much of each cluster's cost reached a real namespace (article Part 6, "Not
everything is worth splitting"). The rest sits in buckets, whose names contain `:` after
the cluster. An idle bucket (`idle:<pool>`) is placed by choice, not a gap, so it counts
as covered. The Phase B gate reads it against `kubernetes.coverage_target`; it is a
report, never a reason to fail a run.

When coverage is short, or a tenant's split is disputed, the first suggestion is never a
finer model. A noisy tenant in a shared pool (one that hammers logging or moves most of
the traffic) is never split right by requests: suggest a dedicated pool for it, so every
row its nodes bill maps straight, their traffic included. Or its own ingress or egress:
on EKS its load balancer then carries its namespace, and a NAT gateway that only its
traffic leaves through can carry a tag the rules route on. Record the suggestion as a
`D-NNN` entry; it is the cluster team's change to make.

The report (`k8s_coverage`) takes one parameter, the month (`{{BILLING_PERIOD}}`), and
reads `base_monthly` for that month. It keeps the Kubernetes rows by `split_by` starting
with `k8s:`, never by the shape of `x_namespace`. One row per cluster:

| Column | What |
|---|---|
| `provider`, `subaccountid`, `regionid`, `cluster` | the group; `cluster` is the part of `x_namespace` before the first `/` |
| `cluster_cost` | the sum of `effectivecost` |
| `on_namespaces` | the same sum over rows on a real namespace: the part after the first `/` has no `:` |
| `idle` | the same sum over idle buckets: the part after the first `/` starts with `idle:` |
| `coverage` | `(on_namespaces + idle) / cluster_cost`; NULL when the cluster's cost is 0 |

The account and region are part of the group, so two `prod` clusters in two accounts are
two lines. Lowest coverage first, a NULL coverage last.

## Other distributions

Only the signals change, in the adapter and the sync:

- **AKS**: tag keys and JSON tags as in `providers.md` (Azure). Node pools are VM scale
  sets, or plain VMs in a Virtual Machines pool, tagged `aks-managed-poolName`; node
  auto-provisioning tags what it launches `karpenter.sh_nodepool` and
  `karpenter.azure.com_cluster`. Unless node pool tags add it, no other node or disk row
  names the cluster (cluster tags skip node pools, and CSI disks carry PVC tags, not the
  cluster): take it from `x_resourcegroupname`, the node resource group
  (`MC_<group>_<cluster>_<region>` unless renamed at creation), one condition per
  profile cluster, ignoring case.
- **OpenShift** (self-managed or as a managed service): a pool is one role's compute
  machine sets, one per zone (on the managed services, its machine pools); control-plane
  and infra nodes have machine sets of their own, usually routed whole to the platform
  team. No tag names the pool: on AWS the Machine API tags instances and volumes only with
  `kubernetes.io/cluster/<infrastructure id>` = `owned`, `Name` (the machine set's name
  plus a suffix) and the user tags of the machine set and the cluster. Until each machine
  set of a role carries a pool tag (the cluster team's change, recorded as a `D-NNN` entry), its
  nodes land in `cluster:shared`.
- **GKE**: `providers.md` (Google Cloud) has the label routes and GKE cost allocation's
  `k8s-namespace`.

## Gate and failure modes

Per cluster, on a closed month:

1. The weights check returns zero rows (every pool's weights add up to 1.0).
2. The totals check returns zero rows with the cluster's weights in place.
3. The coverage report (above) shows the cluster at or above its coverage target. If not,
   group the bucket rows by service (`base_monthly` where `x_namespace` is
   `<cluster>/pool:…` or `<cluster>/cluster:shared`) to see what to fix: usually untagged
   boot disks, or a pool missing from `k8s_pools`.
4. A team that knows its namespace's footprint looks at its number and finds it
   plausible.

| Symptom | Likely cause |
|---|---|
| A shared pool entirely in `pool:<p>` | no weights: the pool label isn't exported (or, on Datadog, not mapped to a tag), or the pool name in the metrics differs from the tag value |
| A whole cluster as plain EC2 rows, no `x_namespace` | its tag keys are inactive in the payer account (D7, `tags.activated`); activate and backfill |
| Every cluster row in one `<cluster>/default` | one workload namespace: nothing to split (D7, `workload_namespaces`); one namespace per app first |
| Lots of `cluster:shared` | boot disks or load balancers without a pool or namespace tag |
| GPU pool cost on CPU-only tenants | the GPU pool is listed as `shared` in `k8s_pools` |
| A GPU pool entirely in `pool:<p>` | the sync's pool list missed it (it must read `k8s_pools`, GPU rows included), or nothing requested a GPU |
| Two accounts' or regions' tenants share one split | metrics or pools written without the account or region |
| A cluster's split holds another cluster's namespaces, or its capacity is too big | the endpoint serves several clusters and a query lacks `cluster_selector` |
| A shared pool split onto one namespace (kube-state-metrics' own), or finished pods still counted | the scrape replaced kube-state-metrics' `namespace`, `pod` or `container` with its own (the originals are in `exported_*`) |
| Skewed weights, or the node match fails on duplicate series | two kube-state-metrics replicas, and a series not taken by `max` first |
| Meshed namespaces pay less than they reserve | native sidecars not counted: the init-container series left out or added with `+`, kube-state-metrics older than 2.11 (or 2.3), or a Datadog agent older than 7.53 |
| A namespace of short jobs pays little or nothing on a shared pool | the pool is sampled hourly and its jobs start and end between samples: put it in `short_jobs` (Q-K8) |
| One namespace's two regions get one owner | the cluster name repeats in one account and `routing.key` lacks `regionid` |
| Dashboards count a vendor workspace as Kubernetes | a query tested `x_namespace LIKE '%/%'` instead of `split_kind = 'k8s'` |
