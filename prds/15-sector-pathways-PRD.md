# PRD 15: Sector Climate Pathways and Target Setting

| | |
|---|---|
| **Repo** | `sector-pathways` |
| **Bullets covered** | `frameworks` (4 of 20 résumés) |
| **Packet** | [`15-sector-pathways.md`](../knowledge-packets/15-sector-pathways.md) |
| **Est.** | 18 human-hours, 4 to 6 agent session hours |
| **Priority** | 15 of 17. **Check `~/projects/ngfs-scenario-explorer` first; much of this exists.** |
| **Model** | Sonnet throughout. Opus for M3 (SDA formula correctness) if the SBTi maths gets fiddly. |
| **Prereqs** | Read `00-CONVENTIONS.md`. PRD 11 output for the portfolio alignment layer. |

---

## 0. Reuse gate

Before writing any code, inspect `~/projects/ngfs-scenario-explorer`. Report what exists: scenario
ingest, `pyam` usage, pathway derivation, any UI. **Extend it rather than rebuilding.** If it already
covers M0 to M2, start at M3. State in the README what was reused.

---

## 1. Objective

A tool that ingests NGFS and IEA scenarios, derives sector-level decarbonization pathways, implements
both SBTi allocation approaches (SDA convergence and absolute contraction), and computes portfolio
alignment gaps with an honest demonstration of how much the verdict depends on scenario choice.

**Success test:** a chart showing a sector pathway with a portfolio's weighted trajectory overlaid,
plus a table showing the alignment verdict flipping across NGFS scenarios.

---

## 2. Scope

**In:** power, cement, steel, and aviation (the sectors where SDA applies cleanly), plus absolute
contraction for everything else. NGFS Orderly / Disorderly / Hot House World / Too Little Too Late.
IEA NZE where public figures permit. Horizons 2030 and 2050.

**Out:** running an integrated assessment model. Physical risk (that is PRDs 09 and 16). Validating
targets against SBTi criteria formally. Non-CO2 gases beyond CO2e aggregates.

---

## 3. Data contracts

| id | Source | Access | Notes |
|---|---|---|---|
| `ngfs_scenarios` | NGFS Scenario Explorer (IIASA) | Free with registration; API available | IAMC long format. Use `pyam`. Multiple models (REMIND, GCAM, MESSAGE) per scenario. |
| `ipcc_ar6_db` | IPCC AR6 WGIII scenario database | Free, IIASA hosted | Broader ensemble |
| `iea_nze` | IEA Net Zero Roadmap published figures | Partly free | Where paywalled, use only published headline figures and cite them; do not scrape paid content. |
| `sbti_targets` | SBTi target validation database and sector guidance | Free download | Company targets, SDA tool documentation |
| `portfolio_emissions` | From PRD 11 | | Company emissions and financed-emissions weights |

**Use `pyam` (`pip install pyam-iamc`).** It handles IAMC format, filtering, aggregation, and unit
conversion natively. Writing a custom parser wastes a day.

---

## 4. Milestones

### M0. Reuse audit and scaffold
Per section 0. Then repo scaffolding per conventions for whatever is new.
**Accept:** a written reuse report. Nothing rebuilt that already exists and works.

### M1. Scenario ingest
`ingest/ngfs.py` via `pyam`. Filter to needed variables: sector emissions, sector activity, and
carbon price. Store as a tidy parquet.
**Accept:** at least 3 NGFS scenarios x 2 IAM models load. Variable coverage report printed. Units
recorded and asserted.

### M2. Pathway derivation
`transform/pathways.py`. For power:
`intensity = Emissions|CO2|Energy|Supply|Electricity / Secondary Energy|Electricity`, converted to
tCO2/MWh, by region and year. Analogous derivations for cement (tCO2/t), steel (tCO2/t), aviation
(gCO2/RTK).
**Accept:** derived power intensity for a 1.5 C-aligned scenario reaches near zero in advanced
economies around 2035 to 2040, and today's global value lands near 0.4 to 0.45 tCO2/MWh. **If it does
not, the variable mapping or the unit conversion is wrong.** These two sanity checks are the
milestone gate.

### M3. Allocation methods
`model/allocation.py`:

```python
def sda_pathway(company_intensity_now, sector_intensity_now,
                sector_intensity_target, company_activity_growth,
                sector_activity_growth, years) -> pd.Series:
    """SBTi Sectoral Decarbonization Approach. Convergence to a common
    intensity by the target year. Use the published SBTi formula."""

def absolute_contraction(base_emissions, annual_rate, years) -> pd.Series:
    """Linear annual reduction, default 4.2% for 1.5 C."""
```

**Accept:** three property tests. (1) A company starting above sector intensity must decarbonize
faster in percentage terms than one starting below. (2) All companies converge to the same intensity
at the target year under SDA. (3) Absolute contraction from base year B at rate r reaches
`base * (1 - r*n)` at year B+n. Assert all three.

### M4. Portfolio alignment
For each holding: actual trajectory from history plus stated targets, versus the required pathway.
Compute an alignment gap in tCO2e and in percentage terms. Aggregate weighted by financed emissions
from PRD 11.
**Accept:** gaps computed for every holding or an explicit reason for exclusion. Report the share of
portfolio emissions covered by companies with validated SBTi targets as a separate, more defensible
metric.

### M5. Scenario sensitivity
Recompute the alignment verdict under every NGFS scenario and every IAM model.
**Accept:** a table showing the verdict per scenario. **The instability of the verdict across
scenarios is the finding**, and it should be the headline in FINDINGS.md rather than any single
alignment number. Quantify: how many holdings change classification between the most and least
stringent scenario?

### M6. Figures and docs
1. **Sector pathway** with portfolio weighted trajectory overlaid, one panel per sector.
2. **Company scatter**: current intensity (x) versus required annual reduction rate (y), bubble size
   by financed emissions. Immediately shows who has the hardest job. The most useful chart.
3. **Scenario sensitivity** heatmap: holdings by scenario, coloured by aligned / not aligned.
4. `METHODOLOGY.md` covering the SDA formula, the convergence-versus-contraction distinction, and why
   an Implied Temperature Rise number is not reported.

---

## 5. Golden-number regression tests

- Derived power intensity for a named scenario in 2030, tolerance 5% relative.
- SDA convergence property, exact assertion.
- Count of holdings changing alignment classification across scenarios, tolerance 2.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| Unit conversion errors in pathway derivation | Intensities off by orders of magnitude | M2 sanity gates against known present-day values. |
| Scraping paywalled IEA content | Licence violation | Use only published headline figures with citation. Agent stops and asks if tempted. |
| Reporting an ITR number | Overclaiming precision | Explicitly out of scope; report alignment gaps instead, and explain why in METHODOLOGY. |
| Rebuilding what exists | Wasted effort | Section 0 reuse gate. |
| Single-scenario alignment claim | Misleading | M5 makes multi-scenario reporting mandatory. |

---

## 7. Note on the existing repo

`~/projects/ngfs-scenario-explorer` may already have a UI. If so, extending it with the SDA layer and
the portfolio alignment view is a much better use of time than a new repo, and a working explorer is
a stronger demo than a notebook.
