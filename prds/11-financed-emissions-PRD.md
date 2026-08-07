# PRD 11: Financed Emissions Estimator

| | |
|---|---|
| **Repo** | `financed-emissions` |
| **Bullets covered** | `ghg` (8 of 20 résumés) |
| **Packet** | [`11-financed-emissions.md`](../knowledge-packets/11-financed-emissions.md) |
| **Est.** | 30 human-hours, 7 to 11 agent session hours |
| **Priority** | 13 of 17. Role-conditional: build for ESG, climate-finance, and credit applications. |
| **Model** | Opus for M4 (estimator validation design). Sonnet elsewhere. |
| **Prereqs** | Read `00-CONVENTIONS.md`. PRD 13's EDGAR client is reusable for the financial denominators. |

---

## 1. Objective

A PCAF-compliant financed emissions estimator over a synthetic portfolio built from public data, with
an explicit estimation ladder, a PCAF data-quality score on every position, a **measured error rate**
for the revenue-based estimates, and a demonstration of the EVIC denominator artifact.

**Success test:** a portfolio-level financed emissions figure with a data-quality distribution, plus
two charts: the estimator error distribution against known reported values, and financed emissions
varying with equity market level while the loan book is held fixed.

---

## 2. Scope

**In:** listed equity and corporate bonds (EVIC denominator) and business loans (equity + debt
denominator). Scope 1 and 2 fully; scope 3 for the sectors where it dominates (oil and gas, autos,
utilities) with an explicit statement of unreliability. A synthetic portfolio of 150 to 300 US listed
companies with stated weights.

**Out:** mortgages, motor vehicle loans, sovereign debt, project finance. Non-US companies. Any real
portfolio data. The portfolio is synthetic and the README says so in the first paragraph.

---

## 3. Data contracts

| id | Source | Notes |
|---|---|---|
| `epa_ghgrp` | EPA Greenhouse Gas Reporting Program facility emissions | Free API. Threshold 25,000 tCO2e, US only, direct emissions. Highest-confidence layer. |
| `epa_frs` | Facility Registry Service | Facility-to-parent linkage |
| `sec_xbrl` | EDGAR XBRL | Revenue, market cap components, total debt, cash for EVIC. Reuse PRD 13. |
| `climate_trace` | Climate TRACE asset-level independent estimates | Free. The independent cross-check; the most interesting part of the project. |
| `useeio` | EPA USEEIO environmentally extended input-output model | Free. tCO2e per dollar of output by NAICS. The engine for score-5 estimates. |
| `egrid` | EPA eGRID subregion factors | For location-based scope 2 |
| `gem_trackers` | Global Energy Monitor plant trackers | Physical activity data for utilities and heavy industry |

---

## 4. Milestones

### M0. Scaffold and synthetic portfolio
Repo per conventions. Build `data/processed/portfolio.parquet`: 150 to 300 US listed companies across
at least 8 GICS sectors, with synthetic exposure amounts and an asset class per position. Weights
documented as illustrative.
**Accept:** portfolio exists, sums to a stated notional, spans sectors including at least 20 high-
emitting names (utilities, energy, materials, industrials).

### M1. Emissions data layers
Ingest GHGRP (facility level, roll up to parent via FRS), Climate TRACE, and any company-reported
figures extractable from filings.
**Accept:** for each portfolio company, the layers available are recorded. Report coverage: what share
of the portfolio has GHGRP data, Climate TRACE data, both, or neither.

### M2. Estimation ladder
`model/ladder.py`. In priority order, with the PCAF score recorded:

| Rung | Method | PCAF score |
|---|---|---|
| 1 | Verified reported | 1 |
| 2 | Unreported but disclosed | 2 |
| 3 | Physical activity (MWh generated, tonnes produced) x emission factor | 3 |
| 4 | Revenue x company-specific factor | 4 |
| 5 | Revenue x sector-average USEEIO factor | 5 |

**Accept:** every company gets exactly one score with the rung recorded. A test asserts the ladder is
applied in order and never skips a higher-confidence rung when its data is available.

### M3. Attribution and roll-up
Implement PCAF attribution factors:
- Listed equity / bonds: `outstanding / EVIC`, where EVIC = market cap + total debt + minority
  interest + cash and equivalents.
- Business loans / unlisted equity: `outstanding / (total equity + total debt)`.

Roll up by sector, by asset class, and by PCAF score.
**Accept:** attribution factors are all in (0, 1] or flagged with a reason. Portfolio total is
reported **alongside its data-quality distribution**, never alone. A test asserts the report cannot
be generated without the distribution.

### M4. Estimator validation  **← Opus. This is what makes it a real project.**
Take companies that *do* report and have GHGRP coverage. Hide the reported value. Run the score-4 and
score-5 estimators. Measure the error distribution: median absolute percentage error, the fraction
within a factor of 2, and the fraction within a factor of 10, broken out by sector.
**Accept:** the error distribution is computed and reported. Expect revenue-based scope 1 estimation
for heavy industry to be wrong by a large factor; **publish that**, because quantifying the error is
more valuable than the estimate itself. A test asserts the validation set is disjoint from any
calibration used in the estimator.

### M5. Independent cross-check
Compare self-reported and GHGRP figures against Climate TRACE for the companies where all three
exist. Report the divergence distribution and investigate the largest gaps.
**Accept:** divergence reported by sector, with at least two specific cases examined and explained.

### M6. EVIC artifact demonstration
Hold the portfolio positions fixed. Recompute financed emissions using EVIC at each month-end over
five years, so only market levels vary. Plot reported financed emissions against the S&P 500 level.
**Accept:** the figure shows the metric moving materially on market moves alone. Quantify: what
percentage swing in reported financed emissions occurred with zero change in the book or in real
emissions? This is the headline critique and it should be a single memorable number.

### M7. Figures and docs
1. **Estimator error distribution** by sector and PCAF rung.
2. **EVIC artifact** chart.
3. **Portfolio composition** by PCAF score, stacked.
4. **Attribution waterfall**: portfolio financed emissions by sector.

---

## 5. Golden-number regression tests

- Median absolute percentage error of the score-5 estimator, tolerance 10% relative.
- Share of portfolio at PCAF score 4 or 5, tolerance 3pp.
- EVIC-driven swing in reported financed emissions, tolerance 3pp.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| Presenting a portfolio total without the data-quality distribution | False precision | M3 test blocks report generation without it. |
| GHGRP covers only US direct emissions | Systematic understatement for global companies | Explicit coverage statement; use Climate TRACE for the global cross-check. |
| Double counting across scopes | Inflated totals | Report scopes separately; never sum scope 1, 2, and 3 into one headline number without labelling. |
| Validation set contaminated | Error rate understated | M4 test asserts disjointness. |
| Reading as advocacy rather than analysis | Credibility | Report the methodology's flaws (EVIC, data quality) as prominently as its outputs. |

---

## 7. Companion

PRD 15 (`sector-pathways`) consumes this output for portfolio alignment. PRD 12 (`emissions-qc`)
shares the emissions data layers; consider building them in adjacent repos with a shared ingest
package if both are in scope.
