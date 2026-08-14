# EDGAR MCP, in plain language

## The one concept everything hangs on

There is no universal field called revenue.

XBRL is a filing language, not a normalized analyst database. Companies choose
tags that fit their disclosures. Four utilities can report the same economic
idea under three valid tags. Asking EDGAR for one tag can return a clean answer
for three companies and silently omit the fourth.

The server resolves a human concept across a ladder of tags and returns the tag,
period, accession number, filing date, and filing link behind every value.

## Vocabulary you need

**XBRL:** Structured financial-reporting data embedded in SEC filings.

**Taxonomy:** The dictionary of allowed tags. `us-gaap` is the standard US GAAP
taxonomy. Companies can also define extensions.

**Tag:** The machine-readable label attached to a reported fact.

**Concept ladder:** An ordered list of tags that can represent one normalized
analyst concept, such as revenue.

**Duration fact:** A value measured over a period, such as annual revenue. It
needs a start date and an end date.

**Instant fact:** A value measured on one date, such as total assets. It needs an
end date but no start date.

**Accession number:** The SEC's identifier for a filing. It makes the source
traceable.

**Restatement basis:** Whether to use the first filing for a period or the latest
filing that later revised it.

**Point-in-time cutoff:** The latest filing date the query is allowed to see.
It prevents historical analysis from using future information.

## Why one tag fails

AEP's `NetIncomeLoss` history stops in 2013, while current consolidated net
income appears under `ProfitLoss`. A resolver that takes the first tag with any
data returns a stale number as if it were current.

Xcel reports current revenue under
`RegulatedAndUnregulatedOperatingRevenue`. Its generic revenue tags stop years
earlier. A generic-only ladder silently drops Xcel.

Some tags contain quarterly segment facts but not annual consolidated facts.
The annual filter has to run inside the ladder. Filtering only after choosing a
tag can turn valid annual data under the next tag into a false missing result.

## The worked utility example

For fiscal 2025, the server resolves:

| Company | Revenue | Tag |
|---|---:|---|
| AEP | $21.70B | `RevenueFromContractWithCustomerExcludingAssessedTax` |
| Duke | $31.74B | `RevenueFromContractWithCustomerIncludingAssessedTax` |
| Southern | $28.94B | `RevenueFromContractWithCustomerIncludingAssessedTax` |
| Xcel | $14.67B | `RegulatedAndUnregulatedOperatingRevenue` |

The values become comparable only after the server records how each one was
found. The tags are related but not always identical in scope, so the caller can
see when a comparison needs accounting judgment.

## Restatement and point-in-time are different choices

`latest_filed` asks what the company says now about a historical period.

`as_originally_reported` asks what appeared in the first filing for that period.

`filed_on_or_before` asks what information existed by a historical date.

These controls solve different problems. A current research note may want the
latest restated history. A backtest needs a filing cutoff first, then a basis
within the filings available by that date.

The new cutoff is applied before tag selection and restatement selection. Later
periods and later revisions never enter the candidate set.

## Why missing stays missing

Utility capital expenditure is a good example. Some filers use custom extension
tags for construction spending. A smaller standard property, plant, and
equipment tag may exist, but substituting it would produce a plausible
understatement.

The server returns null, lists the tags tried, and explains that a custom
extension may be required. A missing value with a reason is safer than an
invented comparable.

## What each code piece means

`config/concepts.yaml` is the analyst judgment layer. It defines normalized
concepts, units, instant or duration type, and ordered tag ladders.

`client.py` handles SEC identification, rate limiting, caching, company
resolution, and the network allowlist.

`concepts.py` filters forms and periods, applies filing cutoffs, resolves
restatements, prefers current tags, and returns provenance-bearing facts.

`server.py` exposes the logic as MCP tools and turns failures into structured
responses. Each observation now includes a direct SEC filing URL.

## What you can say in an interview

"The main problem was semantic normalization, not HTTP access. I built tag
ladders, filtered annual facts inside the ladder, preferred tags still in use,
separated instant from duration periods, and made restatement basis explicit.
For historical work I added a filing-date cutoff so later facts cannot leak into
a point-in-time query. Missing data stays null with the tags tried."

## Questions this project invites

How should company-specific extension tags be discovered without weakening the
meaning of a normalized concept?

Can concept ladders be versioned by taxonomy year and industry?

How should peer comparisons flag different fiscal year ends or different tag
scope?

Which derived metrics can preserve provenance for both numerator and denominator?
