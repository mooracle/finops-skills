# When the user pushes back

The usual objections, the short answer, and where the article argues it. Give the
answer, offer the alternative the build already has, and record the outcome as a
`D-NNN` entry. If the user overrides an invariant with a reason, record the reason; if they
override it without one, say plainly what will break and when.

| They say | Answer | Article |
|---|---|---|
| "Let's write the team into `base_daily`, queries get simpler." | No (invariant 2). | Part 1 |
| "Let's map 100% of spend." | No: rules for the top of the worklist, an alert on the amount (`rules-engine.md`); for clusters, the coverage target (`kubernetes.md`). | Part 1 ("The loop"), Part 6 |
| "Charge on `BilledCost`, it's what we pay." | No (invariant 3). Show them D9. | Part 2 |
| "Track deltas, rebuilding the month is wasteful." | No (invariant 4). | Part 2 |
| "One raw table for all clouds." | No: one per provider (`stack.md`, "What any stack must provide"). | Part 2 |
| "Add `ResourceId` as a dimension." | No: cost follows cardinality (`data-model.md`, "Rules of thumb"). | Part 2 |
| "A generic framework for every SaaS vendor." | No: one shim per vendor (`saas.md`, "The shim"). | Part 5 |
| "Drop the Marketplace row once the vendor's split covers it." | Never: split it (invariants 6 and 8; `saas.md`, "How the split works"). | Part 3, Part 5 |
| "Spread the annual commit over the year, finance hates the spike." | Whole and visible by default. Spread only as data on the bill side, never by editing raw: the adapter changes only what `saas.md`, "Prepaid deals", lists. | Part 3 |
| "Per-pod, per-second Kubernetes precision." | No: requests and the dominant resource, the rest in visible buckets, aiming at the coverage target. For a noisy tenant, suggest a dedicated pool or its own ingress or egress, never a finer model (`kubernetes.md`, "Coverage"). | Part 6 |
| "If it has a namespace, it's Kubernetes." | No (invariant 10). | Part 2 |
| "Skip the tests, it's just SQL." | No: a phase isn't built until its cases pass (`testing.md`). | Part 7 |
| "Let finance read the live views for the month-end numbers." | Their call. The live views restate history on purpose; if a reported month must stay put, that's the frozen close (Q-F1, their process, no default). | Part 4 |
| "Let's build optimization too: rightsizing, anomalies." | Out of scope. Attribution and optimization don't conflict: build attribution, buy optimization if you need it. | the intro ("Build, buy, or use your cloud's own tools?") |
