# PRD 09: Catastrophe and Climate Risk Model Divergence

| | |
|---|---|
| **Repo** | `climate-risk-divergence` |
| **Bullets covered** | `vendors`, `vendorsShort` (15 of 20 résumés) |
| **Packet** | [`09-climate-risk-divergence.md`](../knowledge-packets/09-climate-risk-divergence.md) |
| **Est.** | 25 human-hours, 6 to 9 agent session hours |
| **Priority** | 10 of 17. Tier B (public proxy for licensed vendors). |
| **Model** | Opus for M3 (factorial design and variance attribution). Sonnet elsewhere. |
| **Prereqs** | Read `00-CONVENTIONS.md`. Read Lafferty and Sriver (2023) before M3. |

---

## 1. Objective

Reproduce the vendor-divergence finding using open models, and go one step further than a vendor
comparison can: **attribute the disagreement to its cause** by holding one methodological choice
fixed while varying another.

**Success test:** a variance-attribution table showing what share of county-level ranking disagreement
is driven by GCM choice, downscaling method, scenario, and normalization respectively, plus rank
correlation heatmaps.

---

## 2. Scope

**In:** CONUS counties (~3,100) as the fixed "identical asset portfolio". Historical layer: FEMA NRI.
Forward-looking layers: NEX-GDDP-CMIP6 (BCSD downscaling) and LOCA2 (different downscaling), plus raw
CMIP6. Hazards: extreme heat and extreme precipitation, which are directly derivable from climate
model output. Scenarios: SSP2-4.5 and SSP5-8.5. Horizons: 2050 and 2080.

**Out**
- Commercial vendor data. Explicitly out, and the README says so plainly.
- Flood depth and wildfire modelling, which require hydrological and fire-spread models beyond scope.
  Note as the reason the analysis focuses on heat and precipitation.
- Property-level resolution. County level is the unit.
- Vulnerability and damage functions. This is a hazard-divergence study, not a loss study.

---

## 3. Data contracts

| id | Source | Access | Size note |
|---|---|---|---|
| `fema_nri` | FEMA National Risk Index, county level | Free CSV | Small |
| `nexgddp` | NASA NEX-GDDP-CMIP6 | AWS Open Data S3, netCDF/zarr | **Large.** Read lazily, subset to CONUS and to the needed variables and years before computing. |
| `loca2` | LOCA2 downscaled CMIP6, CONUS, 6 km | AWS / UCSD | Large, same discipline |
| `cmip6_raw` | Raw CMIP6 | Pangeo zarr catalog on GCS | Large |
| `noaa_billion` | NOAA Billion-Dollar Disasters | Free | Small, validation |
| `tiger_counties` | County boundaries | Census | Small |

**Compute discipline is the main engineering risk.** Never download full global fields. Use
`xarray.open_zarr` lazily, subset spatially and temporally first, then compute reduced indices.
Cache the reduced indices, not the raw data.

---

## 4. Milestones

### M0. Scaffold and portfolio definition
Repo per conventions. Define the fixed portfolio: all CONUS counties with FIPS, geometry, population,
and a housing-unit count for optional weighting. Freeze it in `data/processed/portfolio.parquet`.
**Accept:** ~3,100 counties, geometry valid, no duplicates.

### M1. Historical layer
Ingest FEMA NRI. Extract expected annual loss and hazard-specific scores. Compute pairwise Spearman
correlations across NRI hazard types and against the NOAA billion-dollar disaster county footprint.
**Accept:** the historical layers correlate strongly (this is the expected result). **The README must
state the interpretation: their agreement reflects shared calibration to the same loss history, not
independent confirmation.** That framing is the point of the milestone.

### M2. Index derivation
`transform/indices.py`. From each climate dataset, derive two county-level indices per model,
scenario, and horizon:
- **Heat:** days per year above the local historical 99th percentile of daily max temperature.
- **Precipitation:** annual maximum 1-day precipitation, and days above the local historical 99th
  percentile.

Area-weighted conservative aggregation from grid to county using `xesmf` or equivalent. **Regrid
once, up front, identically for every dataset**, because regridding differences can manufacture
apparent disagreement.
**Accept:** a test asserts every dataset is on the identical county aggregation after processing.
Indices are physically plausible (a county cannot have 400 days above a threshold).

### M3. Factorial experiment  **← Opus. The core of the project.**
Four factors, each varied while the others are held fixed:

| Factor | Levels |
|---|---|
| GCM | 8 to 12 models present in **both** NEX-GDDP and LOCA2 |
| Downscaling | NEX-GDDP (BCSD) vs LOCA2 |
| Scenario | SSP2-4.5 vs SSP5-8.5 |
| Normalization | min-max, percentile rank, z-score, log-then-min-max |

For each cell, produce a county ranking. Then:
1. Pairwise Spearman and Kendall between every pair of rankings.
2. **Top-decile Jaccard overlap** between pairs, which is what a user acting on the top decile
   actually experiences.
3. **Variance attribution:** an ANOVA-style decomposition of rank variance across the four factors.

**Accept, all four:**
- The GCM-only comparison uses **the same downscaling** for every model. A test asserts this; mixing
  them silently is the way this experiment gets ruined.
- The downscaling-only comparison uses **the same GCM**. Same assertion.
- Variance attribution shares sum to 1 within tolerance, with an interaction residual reported.
- The normalization experiment operates on **one identical set of underlying values**, so any
  disagreement is purely presentational. Assert this.

### M4. Sign-flip investigation
Identify factor pairs producing near-zero or negative rank correlation. For each, diagnose the cause
and write it up.
**Accept:** at least one concrete, explained case, or an explicit statement that no negative
correlations were found under this design and why that differs from a vendor comparison (vendors mix
relative and absolute scoring conventions, which this design does not).

### M5. Figures and write-up
1. **Rank correlation heatmap**, all pairs, historical block clearly separated from forward-looking.
2. **Variance attribution bar chart** by factor, split by horizon (near-term vs 2080) and by hazard.
3. **Top-decile churn**: which counties enter and leave the riskiest decile under each normalization,
   as a map.
4. `FINDINGS.md` including the buyer's checklist: the five methodological disclosures a climate risk
   vendor must provide (scenario, GCM ensemble, downscaling method, whether defences are represented,
   relative vs absolute scoring).

**Accept:** conventions section 5, plus a README paragraph stating plainly that commercial vendors
were not tested and why, and what the open-model substitution buys instead.

---

## 5. Golden-number regression tests

- Median pairwise Spearman among historical NRI layers, tolerance 0.03.
- Median pairwise Spearman among forward-looking rankings at 2080, tolerance 0.05.
- Variance attribution shares, tolerance 5pp each.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Regridding artifacts manufacture disagreement** | Disagreement that vanishes when regridding changes | Regrid once, identically, up front. Test asserts identical county aggregation. |
| Mixing factors in a "controlled" comparison | Attribution is meaningless | M3 assertions on held-fixed factors. |
| Data volume overwhelms local compute | Hours of download, disk exhaustion | Lazy reads, subset before compute, cache reduced indices only. Set a disk budget and check it. |
| GCM sets do not overlap between downscaling products | Cannot hold GCM fixed | Determine the intersection in M2 and report it. If the intersection is under 5 models, say so and reduce the claim accordingly. |
| Overclaiming a replication of the MS study | Credibility risk | README states the substitution explicitly, up front, in the first paragraph. |

---

## 7. Publication note

This is the project most worth writing up as a public blog post. The finding is genuinely useful to
practitioners and the open-model design is a real contribution rather than a portfolio exercise.
Route it through the outside-activity policy first, as with all public output here.
