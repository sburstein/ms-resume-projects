# Packet 11: Financed Emissions and GHG Modeling

> **Résumé line:** "Developed quantitative models estimating GHG emissions for relationship and
> event-driven lending, supporting the Firm's net-zero financed-emissions targets and scenario
> analysis tooling."

**Repo:** `financed-emissions` | **Est. 30 hours** | **Tier A. On 8 résumés. Build only for the ESG,
climate-finance, and credit roles.**

---

## 1. The idea

A bank's own emissions are trivial: offices, travel, data centers. Its real climate exposure is
**financed emissions**, the emissions of the companies it lends to and invests in. The measurement
problem is that most borrowers do not report emissions, so you have to estimate them, and how you
estimate them determines the answer by a wide margin.

Build the estimator: a model that takes a borrower (ticker or private company with revenue and
sector), returns estimated scope 1, 2, and 3 emissions with an explicit data-quality score, and rolls
up to a portfolio with attribution.

---

## 2. What you need to understand

**Scopes.**
- **Scope 1:** direct emissions from owned or controlled sources. A utility's power plants.
- **Scope 2:** indirect emissions from purchased electricity, steam, heat, and cooling. Reported two
  ways: **location-based** (grid average emission factor where you consume) and **market-based**
  (reflecting contractual instruments like PPAs and RECs). The gap between them is where most
  corporate renewable claims live, and asking which basis a number uses is a good diagnostic question.
- **Scope 3:** everything else in the value chain, 15 categories. Category 11 (use of sold products)
  dominates for oil and gas and for automakers. Scope 3 is where the emissions actually are for most
  sectors, and it is the least reliable number on any disclosure.

**PCAF.** The Partnership for Carbon Accounting Financials publishes the standard method. The core
idea is the **attribution factor**: the financier is allocated a share of the borrower's emissions
equal to its share of the borrower's financing.

- Listed equity and corporate bonds: `outstanding amount / (enterprise value including cash)`.
- Business loans and unlisted equity: `outstanding amount / (total equity + debt)`.
- Project finance: `outstanding / total project value`.
- Motor vehicle and mortgage: their own formulas.

`Financed emissions = attribution factor x borrower emissions`, summed across the portfolio.

**The EVIC denominator problem.** For listed equity the denominator is enterprise value including
cash, which moves with the share price. So a bank's reported financed emissions fall when markets
rally and rise when they fall, with no change in the underlying real-world emissions or in the loan
book. This is a genuine, well-documented flaw in the standard. Knowing it and being able to explain
it is a strong signal that you did the work rather than read the summary.

**PCAF data quality score.** Every estimate carries a score from **1 (best) to 5 (worst)**:
1. Audited/verified reported emissions.
2. Unverified reported emissions.
3. Emissions estimated from physical activity data (MWh generated, tonnes of cement).
4. Emissions estimated from economic activity data (revenue) with company-specific factors.
5. Emissions estimated from revenue with sector-average factors.

Most real portfolios sit at 4 and 5. Reporting the score distribution alongside the tonnage is
mandatory in the standard and it is the honest part of the disclosure. A portfolio number with an
average data quality of 4.6 should not be quoted to three significant figures, and saying so is the
right instinct.

**Sector pathways and target setting** are Packet 15.

---

## 3. Data

| Source | What | Access | Note |
|--------|------|--------|------|
| **EPA GHGRP (FLIGHT)** | Facility-level reported emissions for large US emitters, 2010 to present | Free API and bulk | Direct emissions only, and only facilities above 25,000 tCO2e. Excellent ground truth for the sectors it covers. |
| **EPA FRS** | Facility registry, parent company linkage | Free | Joins facilities to corporate parents |
| **SEC EDGAR** | Revenue, financials for the denominator and for revenue-based estimation | Free XBRL API | Pairs with Packet 13's MCP server |
| **CDP** | Self-reported corporate emissions | Some data free, full data paid | Coverage is skewed toward large listed firms |
| **Global Energy Monitor** | Plant-level trackers for coal, gas, oil, steel, cement | Free | Best public physical-activity data for scope 3 category 11 and utility scope 1 |
| **Climate TRACE** | Independently estimated, asset-level global emissions from satellite and sensor data | Free | The most interesting recent public dataset in this space. Use it as an independent cross-check against self-reported figures. |
| **USEEIO / EXIOBASE** | Environmentally extended input-output tables: emissions per dollar of output by sector | Free | The engine for PCAF score 5 revenue-based estimates |
| **EPA eGRID** | US grid emission factors by subregion | Free | For location-based scope 2 |
| **IEA emission factors** | International grid factors | Partly paid | Free alternatives exist per country |

---

## 4. Method, in steps

1. **Build the estimation ladder.** For a given company, try in order: reported and verified (GHGRP
   or CDP), reported unverified, physical-activity-based (generation, production volume x emission
   factor), revenue-based with a company-specific factor, revenue-based with a sector average from
   USEEIO. Record which rung you landed on as the PCAF data quality score. The ladder *is* the model.
2. **Sector emission factors.** From USEEIO, compute tCO2e per dollar of revenue by NAICS. Map
   companies to NAICS via SIC codes in EDGAR, with a manual override table for the misclassified.
3. **Attribution.** Implement the PCAF formulas with the right denominator per asset class. Pull
   EVIC components (market cap, total debt, minority interest, cash) from EDGAR XBRL.
4. **Portfolio roll-up** with attribution by sector, by geography, and by data quality tier.
5. **Validate the estimator.** This is the part that makes it a real project. Take companies that
   *do* report and are in GHGRP, hide the reported number, run the revenue-based estimator, and
   measure the error distribution. Report it honestly: revenue-based estimation of scope 1 for a
   heavy industrial is often wrong by a factor of several, and quantifying that error is far more
   valuable than the estimate itself.
6. **Cross-check against Climate TRACE.** Where self-reported and independently-sensed estimates
   diverge, that divergence is a finding worth writing up.
7. **Demonstrate the EVIC artifact.** Hold the loan book fixed, vary equity market levels across
   history, and plot reported financed emissions. Showing that the metric moves 20 or 30% on market
   moves alone is a memorable, defensible critique.

---

## 5. Numbers to know cold

- PCAF data quality scale: **1 (best) to 5 (worst)**.
- EPA GHGRP reporting threshold: **25,000 tCO2e per year**.
- Scope 3 has **15 categories**; category 11 (use of sold products) dominates for oil, gas, and autos.
- For a typical bank, financed emissions exceed operational emissions by a factor of **hundreds to
  thousands**.
- US grid average emission factor: roughly **0.37 to 0.40 tCO2e/MWh** nationally, ranging from under
  0.05 in the Pacific Northwest to over 0.6 in parts of the Midwest. Always cite the eGRID subregion.

---

## 6. Interview questions and how to answer

**"What is wrong with the PCAF methodology?"**
Lead with EVIC. "Financed emissions for listed exposures scale inversely with market value, so a bank
can report a reduction because equity markets rallied, with no change to the loan book or to
real-world emissions. There is also a double-counting question across the value chain, and the
data-quality distribution means the headline tonnage often has an implied uncertainty larger than the
year-on-year change being reported."

**"How would you estimate emissions for a private borrower with no disclosure?"**
Walk the ladder, and be explicit that each rung down costs accuracy. Then give the error magnitude
from your validation exercise. That number is what separates you from someone who has only read the
standard.

**"How confident are you in any of this?"**
"Confident in the direction and in the relative ranking of sectors. Not confident in the absolute
tonnage to better than a factor of two for anything at PCAF score 4 or 5, which is most of a real
portfolio. I would report the data-quality distribution alongside every number, which the standard
requires and which most disclosures bury."

---

## 7. Further reading

- PCAF, "The Global GHG Accounting and Reporting Standard for the Financial Industry," Part A.
- GHG Protocol Corporate Standard and Scope 3 Standard.
- Climate TRACE methodology documentation.
- Kölbel et al. and the broader literature on the reliability of corporate carbon disclosure.
