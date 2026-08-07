# PRD 12: Emissions Data Quality Control

| | |
|---|---|
| **Repo** | `emissions-qc` |
| **Bullets covered** | `disclosure` (6 of 20 résumés) |
| **Packet** | [`12-emissions-data-qc.md`](../knowledge-packets/12-emissions-data-qc.md) |
| **Est.** | 20 human-hours, 5 to 7 agent session hours |
| **Priority** | 14 of 17. Role-conditional. Pairs with PRD 11. |
| **Model** | Sonnet throughout. |
| **Prereqs** | Read `00-CONVENTIONS.md`. Shares data layers with PRD 11. |

---

## 1. Objective

A vintage-aware QC harness that ingests emissions data for a company set from multiple independent
public sources and flags every discrepancy, implausibility, and silent restatement, with a
disposition workflow that blocks output until every error-severity finding is resolved.

**Success test:** running the harness across three sources for 200 companies produces a
reconciliation report and a year-on-year change decomposition (real / coverage / methodology), and a
seeded restatement in a test fixture is detected.

---

## 2. Scope

**In:** nine check families (section 4). Vintage snapshots. Cross-source reconciliation. Year-on-year
decomposition. Disposition workflow. Scope 1 and 2; scope 3 where reported.

**Out:** correcting the data (the harness flags, humans decide). Building a new emissions estimate
(that is PRD 11). Non-US companies beyond what the sources cover.

---

## 3. Data contracts

Three independent sources minimum, so cross-source comparison is possible:

| id | Source | Nature | Role |
|---|---|---|---|
| `epa_ghgrp` | EPA GHGRP | Regulator-collected, facility, US, direct only | Highest confidence, narrowest scope |
| `company_reported` | Sustainability reports and SEC climate disclosure | Self-reported, global | Broadest scope, lowest independence |
| `climate_trace` | Climate TRACE | Independently sensed | Independent check |
| `eu_csrd` | CSRD/ESRS filings where available | Self-reported to a different framework | Optional fourth |

---

## 4. Check specification

| id | Check | Severity | Detection |
|---|---|---|---|
| Q01 | Unit error (factor of 1000) | ERROR | Sector-relative outlier screen using median and MAD; a value ~1000x off the sector intensity norm |
| Q02 | Scope confusion | ERROR | Scope 2 to scope 1 ratio outside sector norms; cross-source disagreement in a specific pattern |
| Q03 | Boundary change | WARNING | Large step change in emissions without a corresponding revenue step |
| Q04 | **Silent restatement** | ERROR | Diff a historical value across ingest vintages. **Undetectable without vintage snapshots.** |
| Q05 | Estimated presented as reported | WARNING | Value absent from a known-reported source but present in a vendor-style feed with a reported flag |
| Q06 | Stale carry-forward | WARNING | Identical value in consecutive periods to full precision |
| Q07 | Intensity in an absolute field | ERROR | Magnitude implausible for absolute emissions given company size |
| Q08 | Coverage churn | WARNING | Company enters or leaves the dataset between vintages |
| Q09 | Period misalignment | WARNING | Emissions fiscal period differs from the financial period used as an intensity denominator |

---

## 5. Milestones

### M0. Scaffold and vintage store
Repo per conventions. **Vintage-aware ingest is the foundation:** every fetch writes to
`data/raw/<source>/<fetch_date>/` and is never overwritten. A `vintages.parquet` index records what
was fetched when.
**Accept:** two successive ingests of the same source produce two vintage directories, and a test
asserts the earlier one is unmodified.

### M1. Source ingest
One module per source with a pandera schema. Normalize to a common long schema:
`(entity_id, ticker, source, scope, basis, period_start, period_end, value_tco2e, reported_flag, vintage)`.
Where scope 2 basis (location vs market) is available, it must be captured; mixing bases is Q02.
**Accept:** all three sources land in the common schema. Report per-source company coverage and the
overlap.

### M2. Rule engine
Each check from section 4 as an independent module returning `Finding` objects (same pattern as
PRD 10). Sector-relative screening uses **median and MAD**, not mean and standard deviation, because
emissions intensity distributions have heavy tails.
**Accept:** each check has a test fixture that triggers it and a fixture that must not. **Zero
findings on the clean fixture** is a hard gate, as in PRD 10.

### M3. Restatement detection
Q04 specifically: build a fixture with two vintages of the same source where a historical value
changed. Assert detection.
**Accept:** the seeded restatement is found, the report shows old value, new value, and both vintage
dates.

### M4. Reconciliation report
Per company: side-by-side of every source with deltas and a plausibility verdict. Aggregate: the
distribution of cross-source disagreement by sector.
**Accept:** the report explains expected divergence (a global company should not reconcile to
US-only GHGRP) rather than flagging it as an error. This distinction is the analytical content of the
milestone.

### M5. Year-on-year decomposition
Split portfolio-level change into **real change** (same companies, changed emissions), **coverage
change** (companies entering or leaving), and **methodology change** (restatements and basis
switches).
**Accept:** the three components sum to the total change within tolerance. Assert it. This is the
single most useful output in the repo.

### M6. Disposition workflow
Every finding must be dispositioned as `accepted`, `corrected`, or `waived` with a reason and a date,
recorded in `dispositions.yaml`. The output release is blocked while any ERROR-severity finding is
undispositioned.
**Accept:** a test asserts release generation fails with an undispositioned error finding and
succeeds once dispositioned.

### M7. Figures and docs
1. **Cross-source divergence** by sector, box plot.
2. **YoY decomposition** waterfall.
3. **Findings by check type** over vintages, showing whether data quality is improving.
Plus a `FRAMEWORKS.md` briefly mapping CSRD/ESRS E1, SFDR PAI indicators, EU Taxonomy, and Pillar 3
ESG to what each asks for, since the résumé bullet references EU regulatory reporting.

---

## 6. Golden-number regression tests

- Findings count per check on the broken fixture, exact.
- Findings on the clean fixture, exactly 0.
- YoY decomposition components sum to total change, exact within float tolerance.

---

## 7. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **No vintage snapshots** | Q04 cannot work at all | M0 is the foundation milestone; do not proceed without it. |
| Flagging expected divergence as error | Noise, tool gets ignored | M4 encodes expected-divergence logic (scope and geography mismatch). |
| Mean-based outlier screening | Heavy tails produce false positives and misses | Median and MAD, specified in M2. |
| Treating self-reported as truth | Circular validation | Climate TRACE is the independent leg; keep it in the design. |

---

## 8. Interview artifact

The year-on-year decomposition. Being able to say "the portfolio number moved 8%, of which 3 points
were real, 4 were coverage, and 1 was a restatement" is the answer that distinguishes someone who ran
a disclosure process from someone who read about one.
