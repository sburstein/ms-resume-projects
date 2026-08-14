# Utility affordability, in plain language

## The one concept everything hangs on

Three authoritative sources describe three different kinds of thing, and none
shares a key with the others.

EIA bills belong to an operating utility. Census income belongs to a geography.
SEC ownership belongs to a legal entity. Every source can be correct while a
direct join between them is impossible. The project is the set of bridges.

## Vocabulary you need

**Operating utility:** The company that sells or delivers electricity to a
customer. Its name may differ from the listed parent investors recognize.

**Utility-state row:** One utility's activity inside one state. The same utility
can appear in several states.

**Investor-owned utility, or IOU:** A utility owned by private investors rather
than a municipality, cooperative, state, or federal authority.

**Exhibit 21:** A 10-K attachment listing the registrant's subsidiaries. It is
filed evidence of the legal ownership chain.

**Energy burden:** Annual energy spending divided by household income. This
project currently produces a county exposure proxy, not a customer-level burden.

**Nowcast:** An estimate of a value before the slow authoritative release arrives.

**Lookahead:** Using information in a historical test that would not have been
available at the simulated date. It makes a backtest look better without leaving
an obvious error in the output.

## Bridge one: utility bills

EIA-861 reports residential revenue, sales, customers, ownership, and service
territories. The project aggregates filing parts deliberately, then calculates:

`average rate = residential revenue / residential sales`

`average annual bill = residential revenue / residential customers`

The 2023 output has 2,497 utility-state rows. Of those, 190 are investor-owned.
Those IOUs serve 94.7 million residential customers and have a customer-weighted
average rate of 16.10 cents per kWh.

PG&E at 25.96, Southern California Edison at 27.44, Con Edison at 29.75, and
ComEd at 13.15 provide recognizable spot checks. A national aggregate without
those simple checks could be precisely wrong.

## Bridge two: public ownership

The project starts with Exhibit 21, not fuzzy matching. A normalized exact match
to a filed subsidiary is evidence. Fuzzy similarity is only a fallback.

The crosswalk maps 121 of 190 IOU rows, but row coverage is not the important
number. Those matched utilities represent 82.96% of IOU residential customers.
Coverage weighted by customers tells you how much of the market was mapped.

Confidence stays attached to every row:

- 69.64% of customers map through exact Exhibit 21 matches
- 10.47% map through high-confidence fuzzy matches
- 2.86% remain in a review tier
- 17.04% remain unmatched

Unmatched is a published result. Some utilities have no listed parent. Others
sit under foreign filers, private owners, or registrants outside the initial SEC
universe.

## The matching bugs that mattered

Southern Company and Georgia Power file jointly. The same subsidiary exhibit can
appear under both registrants, which creates a false ownership cycle. The fix
prefers the group parent over an operating company and rejects self-ownership.

"AES Electric Ltd" once matched utilities across many states because the word
"electric" looked similar everywhere. The matcher now requires a shared token
that is not industry boilerplate.

"Mississippi Power Co" once matched Entergy Mississippi through fuzzy overlap
even though Southern's Exhibit 21 contained an exact normalized name. Exact
evidence now runs before fuzzy scoring.

## Bridge three: present-day rates

Annual EIA-861 data arrives 9 to 12 months after the year ends. EIA-861M arrives
monthly with a shorter lag. The nowcast learns each state's persistent ratio
between the monthly sample and the final annual universe.

The state effect is the model. A pooled national equation still had about 7%
error after all 12 months were observed. State-specific ratios reduced year-end
MAPE to 1.52%.

In a pseudo-real-time replay, one-month MAPE is 4.10%, six-month MAPE is 2.93%,
and twelve-month MAPE is 1.52%. Carrying the last annual value forward produces
7.65% MAPE. The nowcast beats that baseline at every horizon in the aggregate.

The first month does not beat carry-forward in every individual target year.
That nuance is visible in `nowcast_backtest.csv`; the headline refers to the
mean across replay years.

## Bridge four: bills to income

The Census API now requires a key for all data queries. The project uses the same
official ACS estimates through Census's keyless table-based bulk files.

EIA's service-territory filing maps utilities to counties. The project joins a
utility's annual bill to median household income in each served county and
computes a burden proxy.

This is deliberately labeled a proxy. EIA does not say which households inside
the county belong to which utility. Overlapping utilities can both list the same
county. Tract precision still needs service-territory polygons or customer
allocation data.

The 2023 build has 8,205 utility-county rows with usable residential bills, and
99.5% join to ACS income. Among 3,271 matched IOU-county exposure rows, the
unweighted median burden proxy is 2.20% and the 90th percentile is 3.62%. Those
are row-level exposure statistics, not a customer-weighted national estimate.

## What each code piece means

`fetch.py` is the network gate and byte-level provenance recorder.

`ingest/eia861.py` reconstructs multi-row headers, combines filing parts, and
calculates rates and bills.

`ingest/edgar.py` retrieves and parses subsidiary exhibits.

`transform/crosswalk.py` resolves ultimate parents, matches utilities, and
measures customer-weighted coverage.

`model/nowcast.py` fits state bridge factors inside each historical replay date.

`ingest/acs.py` joins official bulk income estimates to county geography labels.

`model/burden.py` connects utility bills to county income and labels the method.

## What you can say in an interview

"The hard part was entity resolution across utility operations, geography, and
legal ownership. I used filed subsidiary lists as primary evidence, reported
customer-weighted coverage, and published unmatched rows. For timeliness, I
backtested a state-specific monthly-to-annual bridge with a runtime lookahead
guard. For affordability, I built a county proxy and stated why it is not a
customer-level estimate."

## Questions this project invites

Can state commission territory maps replace the missing national polygon layer?

Which unmatched utilities matter most by customers, and which have discoverable
ownership through 20-F filings or private-company sources?

Does affordability stress cluster under specific listed parents after adjusting
for climate, housing stock, and heating fuel?

Would a hierarchical nowcast improve thin-history states without erasing their
persistent level differences?
