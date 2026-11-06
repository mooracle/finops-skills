# Vendor notes

What onboarding a SaaS vendor taught someone, so the next person knows where to probe
instead of starting from a blank page. One file per vendor: `<vendor>.md`, copied from
[`_template.md`](_template.md).

The notes sit outside the skill folder, so installing the skill doesn't copy them and your
agent won't find them on its own: when you onboard a vendor that has a note, point your
agent at it. A note says where to probe; the discovery gate still takes its evidence from
your own bill and the vendor's API.

A note is the discovery report from `skills/cost-attribution/references/saas.md` as you
verified it, plus the quirks that cost you time. It is **not** a sync implementation:
auth, pagination and SKU handling belong in your own repo, shaped by your codebase.
A `to_lines()` sketch of up to 30 lines is welcome; a framework is not.

Before opening a PR:

- Nothing org-identifying: no account ids, org names, amounts, contract terms, hostnames
  or credentials.
- Fill in the `Verified:` line. Vendors change; an undated note misleads.
- If something didn't work, say so. "The usage API doesn't include support fees, so
  shares drift ~3% from the invoice" is the most useful sentence a note can have.
