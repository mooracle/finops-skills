# <Vendor>

Verified: <YYYY-MM> · vendor API <version> · bill: <cloud> FOCUS <version>
Billed: marketplace | direct

## Where it bills

How the vendor's charge appears: `PublisherName` value (as a lowercase LIKE pattern),
`ServiceName`, `ChargeCategory`, one row per day or one per month, which account it lands on.
Direct-billed: what the invoice API returns and how it maps to FOCUS columns.

## Billing identities

What one vendor id corresponds to (a marketplace subscription, an org, an invoiced
account) and how to tell two apart.

## Unit

The unit teams own in this vendor (deployment, project, workspace, child org), how to
list them, and what goes in `vendor_lines.workspace`.

## Breakdown

Endpoint and field that give cost per unit and service line. What it includes and
excludes compared with the invoice (support, overhead, failed jobs, data transfer), and
what goes on the line with no workspace. Counts instead of dollars work only when one
flat rate prices every unit, since only the proportions are used.

## Completeness

How far the API lags the bill (what goes in `vendor_fetch.through_day`) · restatement
window · history the API keeps · pagination and rate limits · least-privilege scope
that's enough.

## Transform sketch (optional, ≤30 lines)

```python
def to_lines(raw) -> list[dict]:     # -> workspace, servicename, amount
    ...
```

## Quirks

What surprised you, and what you did about it.
