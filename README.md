# finops-skills

Agent skills for building FinOps tooling shaped like your org instead of buying an
umbrella product. They run in Claude Code or any other agent that reads `SKILL.md` skills.

Companion to the article [*Map Your Cloud Bill to Your Teams: Build Your Own FinOps
Tool*](https://mooracle.io/blog/build-custom-finops-tool/).
The article explains **why** each pattern is the right one; the skill applies the
**how** to your accounts, tags, clusters and vendors.

## Skills

| Skill | What it does |
|---|---|
| [`cost-attribution`](skills/cost-attribution/) | A guided build of cloud cost attribution on FOCUS exports, in your own stack (the article's is S3, Athena and Lambda). It scans your bill before asking anything, interviews you only for what the data can't show, writes your org's shape to a validated profile, decides READY / BLOCKED / NOT_A_FIT for every component (including "don't build this, use your console"), then builds phase by phase from specs and golden test cases, with a verification gate after each phase. Optional tracks for SaaS bills and Kubernetes namespaces. A later session resumes from the profile, the decision log and the status file. |

## What's in the skill

```
skills/cost-attribution/
├── SKILL.md                 the run loop: invariants, phases, gates, resume
├── references/              the specs for every phase and track, answers to pushback,
│                            and the trial on a sample bill
├── cases/                   golden test cases as data (rows in → rows out), for any stack
└── assets/profile.template.yaml   your org's shape, from discovery and the interview
```

The skill builds in your stack: Athena, BigQuery, Snowflake, Databricks or whatever
you run. The specs say what each piece must do and the golden cases what it must produce
on the cases that bite; your agent writes the code in your stack and ports the cases to
your test runner.

## Install

In Claude Code (any agent that reads `SKILL.md` skills takes the same folder in its own
skills directory):

```bash
git clone https://github.com/mooracle/finops-skills
mkdir -p ~/.claude/skills
cp -r finops-skills/skills/cost-attribution ~/.claude/skills/
```

Then, in the repo where your cost platform will live, ask for it ("help me build cost
attribution for our AWS org") or run `/cost-attribution`.

To see it work before giving it any access, ask it to run on sample data. It builds the
core path on a synthetic month in DuckDB, with no credentials, and checks the result
against a known answer (`references/trial.md`). It needs Python 3 with DuckDB and PyYAML.

**An agent that doesn't read skills:** the skill is markdown and YAML, no code. Point it
at `skills/cost-attribution/SKILL.md`.

## Why there's no code

Every piece the skill specifies was rebuilt by a fresh agent from the skill alone and
passed the golden cases that ship with it, or a grader's tests where no case covers it.
Wherever an agent had to guess, the spec got one more line. So the skill ships the
logic, not the code.

## What you end up with

A pipeline in your own account and repo (on S3, Athena and Lambda, about $10 a month at
15 million billing lines a month):

```
FOCUS exports (AWS / Azure / Google Cloud) + vendor invoices
        │ raw, one table per provider
        ▼
split_weights ─► base_monthly / base_daily  (rebuilt per month, checked every run)
(shared costs,         │
 K8s, SaaS)            │ query-time JOIN
                       ▼
rules in Git ─► lookup ─► services_daily / services_monthly ─► your dashboards
```

The schema follows your org chart, a rule change is a pull request whose diff shows who
moves where and how much, and every path through the pipeline adds up to the bill or the
run fails.

## Contributing

Two kinds of contributions help most:

- **A vendor note** for a SaaS vendor you onboarded that has none yet: where its charge
  shows up, the API that splits it, and the quirks you hit. Notes live in
  [`vendors/`](vendors/), outside the skill: the install doesn't copy them, so when you
  onboard a vendor that has one, point your agent at it.
- **An integration that gave you extra pain**: a vendor API that never matches its
  invoice, a cluster the namespace weights don't fit, a shared cost you had to split
  some other way. Open an issue with what broke.

## Requirements

- Billing access to at least one cloud (AWS, Azure or Google Cloud), enough to read its
  FOCUS export or to turn one on; on AWS, also read access to the payer account's list of
  cost allocation tags
- Someone who will own it after the build, a couple of hours a month
- Optionally, someone in finance to shape the finance side (how the numbers are used,
  whether months freeze) and walk through the numbers at the end. Without one, the owner
  does
- Optionally, read-only `kubectl` on the clusters you split: without it, the skill finds
  the namespaces from the bill's tags

## License

MIT — see [LICENSE](LICENSE).
