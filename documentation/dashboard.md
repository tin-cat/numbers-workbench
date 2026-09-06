# The dashboard

The home page once logged in. Panels of current figures, yearly trends, and period-over-period
comparisons. Decided 2026-09-06, nothing implemented.

## Who sees it

Gated on `statistics.view`. `admin` holds it, like everything else.

A user without it does not see an empty dashboard: they land on the first section they do have
permission for. A page that exists only to say "you cannot see this" is worse than not routing there
at all.

## What it shows

Three bands, coarse to fine.

**Current position.** Invoiced this month, this quarter, this fiscal year. Expenses over the same
periods. Net. The VAT position for the open quarter, which is the number that actually gets paid.

**Yearly trends.** Income and expenses by month across the current year, with the previous year
behind it. Fifteen years of history is available, so a multi-year view is cheap and genuinely
informative.

**Period comparisons.** This month against the same month last year, this quarter against the same
quarter last year. See the warning about partial periods below, which is the thing that makes these
either useful or actively misleading.

Two operational panels earn their place next to the financial ones:

- **The AEAT submission backlog**, which is the main operational signal this system produces. It
  also lives in the top bar. See [[interface#Layout]].
- **Anything refused**, meaning invoices that could not be issued and submissions permanently
  failed. A count of zero is the normal state and should be visibly zero rather than absent.

## Five ways to get this wrong

These are the traps specific to this data, and all five are easier to design out now than to notice
later in a chart nobody quite trusts.

**Rectificativas must net, not add.** A refund is a rectificativa, and a naive sum of invoices would
count it as more activity rather than less. A month with a large refund must show reduced income,
not an inflated invoice count. Every aggregate nets corrections against what they correct.

**Currencies cannot be summed.** There are EUR and USD invoices. Aggregate in **EUR, converting each
invoice at the rate stored on that invoice**, never at today's rate. This is exactly what the
stored-rate decision in [[money#The exchange rate is part of the fiscal record]] exists to make
possible, and it means a chart drawn in 2030 shows the same figures it showed in 2026.

**Partial periods must compare like with like.** On 6 September, "this month against the same month
last year" comparing six days against thirty is not a comparison, it is a false alarm. Compare the
same day-of-month range and label it as such ("1 to 6 September, against 1 to 6 September 2025").
This matters here more than usual: the business has real seasonality, so a badly framed comparison
looks like a trend.

**Money must not become a float on the way to a chart.** Serialise aggregates as minor units with a
scale, or as decimal strings, and format at render. A JSON number becomes a double in the browser,
which is the whole thing [[money]] exists to prevent. It would be a shame for the one place amounts
turn into floats to be the page whose job is showing them.

**Aggregate in SQL.** 58,807 invoices is nothing for the database and a lot for the ORM. No
hydration, no summing in PHP.

## Retention does not break the history

Worth stating because it is reassuring and not obvious: minimization and anonymization strip
**identity**, not amounts. The dates, totals, tax figures and currency all survive every retention
step, so trends and comparisons remain correct and complete across the whole fifteen years.

Only per-customer breakdowns degrade over time, and they degrade to "a customer", never to a gap in
the totals.

## Charts

**ApexCharts.** Tabler's own components and examples are built around it, so charts look native to
the theme instead of bolted on, and it vendors cleanly through AssetMapper with no Node build. See
[[interface#Assets]].

Chart.js is the reasonable alternative if ApexCharts ever proves awkward; the choice here is theme
consistency, not capability.

## Related

- [[access-control]] for the permission
- [[interface]] for the layout it sits in
- [[money]] for why the numbers must not be floats
