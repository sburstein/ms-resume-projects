# Build Plan: Every Project on the Morgan Stanley Résumé Section

**Owner:** Scott Burstein
**Created:** 6 August 2026
**Source résumé:** `~/Desktop/Applications_2026/01_ModoEnergy_MarketAnalyst_RESUME.docx` (latest, 6 Aug 2026 16:51)
**Companion explainer:** `~/Downloads/Resume_Line_By_Line_Explainer.md`

---

## Why this exists

Every bullet in the Morgan Stanley section describes real work you did behind a bank firewall. You
cannot show any of it. An interviewer at Modo, Tyba, Ascend, or Yes Energy can only take your word
for it, and the follow-up question ("walk me through how you isolated cloud cover") is where the
claim either holds or collapses.

This plan rebuilds all ten projects from public data, in public repos, on your own machine. When
you finish, each résumé line has a corresponding artifact you can link, demo, or screen-share. The
work also forces you to re-derive methods you did once and have not touched in two years.

### Hard compliance boundary, read before writing a line of code

1. **No MS code, data, output, or screenshots move into these repos.** Rebuild from public sources
   only. If you cannot source a number publicly, drop it rather than approximate from memory.
2. **`~/Downloads/el_nino_research_handoff.md` is entitlement-gated MS Research.** It stays local
   and private. It is useful as a sanity check on your own conclusions. It is not a citation, and
   nothing derived from it goes in a public repo.
3. **Check the outside-activity and personal-publication policy before you publish anything
   market-facing.** A bank employee posting a public long/short equity backtest is exactly the
   category that needs pre-clearance. Project 1 is the one to route through compliance first; if it
   comes back restricted, build it privately and demo it live in interviews instead of publishing.
4. Numbers on the résumé (10% of net-load variance, 80 to 97% correlation, 150 utilities) describe
   the MS work. Your public rebuild will produce *different* numbers. That is fine and expected.
   Never retrofit a public result to match a private one.

---

## Scope note, added after the full bullet inventory

This plan originally covered the 11 bullets on the Modo résumé. The full inventory across all 20
résumés is **25 distinct bullets, 186 placements**. The expanded plan is **17 projects across 8
repos**, roughly 370 hours.

**Read [`BUILDABILITY.md`](BUILDABILITY.md) first.** It triages all 25 bullets into fully buildable
(19), buildable with a public proxy (4), and not buildable as an artifact (2), and gives a
recommended cut if you do not want to build all of it. You almost certainly should not build all of
it.

## The seventeen projects

| # | Project | Bullet(s) | Résumés | Repo | Hours |
|---|---------|-----------|---------|------|-------|
| 1 | ENSO long/short equity basket | `elnino` | 11 | `enso-basket` | 25 |
| 2 | Western US net-load baseline with BTM solar | `btm`, `btmShort` | 12 | `netload-baseline` | 35 |
| 3 | Weather-driver sensitivity on net load | `cloud` | 11 | `netload-baseline` | 15 |
| 4 | AI weather model benchmark | `aiwx`, `aiwxBias` | 12 | `ai-weather-bench` | 25 |
| 5 | Energy application layer on AI forecasts | `supply`, `supplyShort` | 12 | `ai-weather-bench` | 25 |
| 6 | National utility rate and affordability dataset | `dataset` | 19 | `utility-affordability` | 35 |
| 7 | Nowcast and productionized monthly release | `nowcast` | 17 | `utility-affordability` | 20 |
| 8 | Utility to public parent crosswalk | `ticker` | 13 | `utility-affordability` | 12 |
| 9 | Cat and climate risk model divergence | `vendors`, `vendorsShort` | 15 | `climate-risk-divergence` | 25 |
| 10 | Workbook health-check agent | `agent` | 9 | `workbook-doctor` | 18 |
| 11 | Financed emissions estimator | `ghg` | 8 | `financed-emissions` | 30 |
| 12 | Emissions data QC and disclosure tooling | `disclosure` | 6 | `emissions-qc` | 20 |
| 13 | Financial-data MCP server | `kensho` | 5 | `edgar-mcp` | 15 |
| 14 | Data center siting and resource risk | `datacenter` | 4 | `datacenter-siting` | 22 |
| 15 | Sector climate pathways and target setting | `frameworks` | 4 | `sector-pathways` | 18 |
| 16 | Multi-hazard geospatial analytics | `hazard` | 3 | `multihazard-analytics` | 28 |
| 17 | Corporate asset location database | `geo` | 4 | `asset-locations` | 25 |
| n/a | No artifact possible | `ambassador`, `audit` | 19 | see BUILDABILITY.md Tier C | n/a |

The sequencing below front-loads the projects that matter most for the power-market roles you are
actively applying to. Projects 11 through 17 are role-conditional: build them only when a specific
application calls for them.

---

## Build order and dependencies

```
Phase A (weeks 1-4)    P4 AI weather bench  ──┐
                       P6 utility dataset     │
                                              ▼
Phase B (weeks 5-9)    P2 net-load baseline ◄─┘   (P2 reuses P4's ERA5 loader)
                       P7 nowcast pipeline   (needs P6)
                       P3 sensitivity        (needs P2)

Phase C (weeks 10-13)  P5 energy application (needs P4)
                       P8 parent crosswalk   (needs P6)
                       P10 workbook agent    (independent, good filler week)

Phase D (weeks 14-16)  P1 ENSO basket        (compliance-gated, do last)
                       P9 climate divergence
```

Rationale for the order:

- **P4 first** because it is the most self-contained, needs no GPU if you score precomputed
  forecasts from WeatherBench 2, and produces a chart (error vs lead time, AI vs physics) that is
  the single most demo-able artifact in the set.
- **P6 second** because the geospatial join is the longest slog and everything in that repo depends
  on it. Starting it early means P7 and P8 become short add-ons rather than new projects.
- **P2 third** because it is the highest-value project for Modo-type roles and it inherits the
  weather data plumbing from P4.
- **P1 last** because of the compliance question, and because it is the one you can talk about
  fluently without an artifact if it comes to that.

---

## Project briefs

### P1. ENSO long/short equity basket
Turn the NOAA ONI / Niño3.4 record into a market-neutral basket and backtest it across every ENSO
event since 1990.

- **Data:** NOAA CPC ONI and Niño3.4 (monthly, free); IRI ENSO probability plume for the
  probabilistic side; daily equity prices via `yfinance` or Stooq; commodity proxies via ETFs
  (CANE, DBA, COPX, JJC) since clean futures history is not free.
- **Method:** define event windows from ONI thresholds; build a sector and single-name exposure map
  (long fertilizer and dry-weather beneficiaries, short sugar and palm-oil-exposed consumer names,
  short EM rate-sensitive names); dollar-neutralize and beta-hedge; compute event-window returns,
  hit rate, and a t-stat across events; run the null test of random windows of the same length.
- **The honest finding you should expect:** the effect is small, the sample is about eight events,
  and statistical significance will be weak. Say that. "I found the signal is real in direction but
  underpowered in sample" is a stronger interview answer than a fake Sharpe ratio.
- **Deliverable:** notebook + a one-page tearsheet PDF with the event-study chart.

### P2. Western US net-load baseline with behind-the-meter solar
Reconstruct hourly net load for CAISO and two other WECC balancing authorities, then build the BTM
solar adjustment that traditional load models are missing.

- **Data:** EIA-930 hourly demand and generation by balancing authority; CAISO OASIS actual load,
  solar, wind, curtailment; California Distributed Generation Statistics (interconnection dataset)
  for rooftop capacity by ZIP and vintage; NREL NSRDB for irradiance; NREL PVWatts API to simulate
  output from installed capacity.
- **Method:** estimate BTM fleet capacity by geography and month from interconnection filings;
  simulate hourly BTM generation with PVWatts using NSRDB irradiance and a tilt/azimuth
  distribution; add it back to metered load to recover *gross* load; fit the load model on gross
  load with weather regressors; subtract simulated BTM and utility-scale renewables to project net
  load. Validate against held-out months.
- **Deliverable:** a duck-curve chart showing metered load vs your reconstructed gross load vs
  simulated BTM, plus a MAPE table vs a naive weather-only baseline.

### P3. Weather-driver sensitivity on net load
Sits in the same repo as P2. Decompose net-load variance by weather driver and produce utility-level
forecast trajectories.

- **Method:** fit the load model, then run (a) a variance decomposition via Sobol or Shapley values
  on the fitted model, and (b) a one-at-a-time sensitivity sweep holding other drivers at
  climatology. Report the share of net-load variance attributable to cloud cover, temperature,
  humidity, and wind separately. Repeat per utility service territory rather than per BA.
- **Deliverable:** a stacked variance-share bar chart per utility, and a fan chart of forecast
  trajectories.

### P4. AI weather model benchmark
Score GraphCast, Pangu-Weather, FourCastNet, and the ECMWF AIFS against physics baselines out to 14
days.

- **Shortcut that saves you a GPU:** WeatherBench 2 publishes precomputed forecast archives for all
  of these on Google Cloud Storage, plus ERA5 as ground truth. You are writing a scoring harness,
  not running inference. If you want live forecasts, ECMWF open data serves AIFS and HRES free.
- **Method:** RMSE, ACC, and bias by lead time for Z500, T850, and 2m temperature; find the lead
  time where the AI advantage crosses over; add a spread-skill check.
- **Deliverable:** the error-vs-lead-time chart. This is the chart from the slide you already
  presented at MS, rebuilt on public data, and it is the fastest credibility win in the whole plan.

### P5. Energy application layer
Take P4's forecasts and answer energy questions with them.

- Peak-load prediction under extreme temperature (train on EIA-930 demand vs ERA5 temperature,
  score peak-hour error specifically, not average error).
- Heat-wave and cold-wave detection: percentile-threshold event definition, then measure lead time
  and false-alarm rate of each model's detection.
- Gas freeze-off risk: build a composite index from temperature, wind chill, and duration below
  freezing; validate against the Uri period and against EIA weekly gas production dips.
- Renewable generation forecasting from 3D fields: extrapolate 100m hub-height wind from the model's
  pressure levels, apply a power curve, compare to EIA-930 wind generation.

### P6. National utility rate and affordability dataset
The geospatial join. This is the project that proves you can build data infrastructure.

- **Data:** EIA-861 (annual utility sales, revenue, customers) and EIA-861M (monthly); ACS 5-year
  median household income at census tract via the Census API; HIFLD Electric Retail Service
  Territories shapefile; TIGER/Line tract shapefiles.
- **The hard part:** apportioning tracts to service territories. Territories cross county and state
  lines and split cities. Use area-weighted overlay as the baseline, then upgrade to
  population-weighted apportionment using tract-level population, and report how much the answer
  moves between the two. That comparison *is* the interesting result.
- **Deliverable:** a tidy table of energy burden (annual residential bill / median household income)
  for every IOU, plus a choropleth map and a documented methodology note.

### P7. Nowcast and monthly production pipeline
- Close the reporting lag: EIA-861 annual data lands roughly a year late. EIA-861M monthly lands in
  about two months. Build a bridge model that projects the annual series forward using the monthly
  series, then validate it by backtesting: hide the last available year, nowcast it, measure error.
- Productionize: a scheduled job, schema validation on ingest, a data dictionary, a source
  whitelist file, changelog, and versioned outputs. Fail loudly on schema drift.
- **Deliverable:** a repo that runs `make monthly` and emits a dated, versioned release with a
  methodology PDF.

### P8. Utility to publicly traded parent crosswalk
- **Data:** EIA-861 ownership fields, SEC EDGAR company tickers JSON, EIA-860 operator/owner tables.
- **Method:** fuzzy-match utility names to EDGAR registrants, then hand-verify. There is no clean
  public crosswalk, which is exactly why building one is worth showing. Track match confidence and
  publish the unmatched list rather than hiding it.
- **Deliverable:** `utility_id → CIK → ticker` CSV with a confidence column and a coverage stat
  (what share of US residential customers you mapped).

### P9. Catastrophe and climate risk divergence study
Rebuild the vendor-comparison finding using open models, since the commercial vendors are not
accessible to you outside the bank.

- **Data:** FEMA National Risk Index (historical, county and tract level); NASA NEX-GDDP-CMIP6 and
  LOCA2 for statistically downscaled climate projections; raw CMIP6 for the undownscaled
  comparison.
- **Method:** at county level, correlate hazard rankings across (a) historical cat-style layers,
  which should agree strongly, and (b) forward-looking downscaled projections across different
  models and downscaling methods, which should agree much less. Then isolate *why*: hold the GCM
  fixed and vary the downscaling method, then hold downscaling fixed and vary the GCM. Show which
  choice drives more rank churn. Add a normalization experiment: score the same underlying data with
  min-max, quantile, and z-score scaling and measure how the top-decile membership changes.
- **Deliverable:** rank-correlation heatmaps and a short write-up. This is publishable as a blog
  post and it is genuinely useful to the field.

### P10. Workbook health-check agent and AI workflow artifacts
- **Build:** a Python CLI plus an MCP server that opens an `.xlsx`, and reports formula errors
  (`#REF!`, `#VALUE!`, `#DIV/0!`), external links, hardcoded overrides of formula cells,
  inconsistent formulas within a row or column range, circular references, orphaned named ranges,
  and stale volatile functions. Output as JSON and as a human-readable report.
- **Add:** a `--fix` mode that proposes patches but never writes without confirmation. That is your
  human-in-the-loop story, made concrete.
- **Deliverable:** installable package, MCP server config, and a demo on a deliberately broken
  workbook you build as a fixture.

### P11 through P17: the role-conditional projects

Added after the full 25-bullet inventory. Full briefs live in their packets; one line each here.

- **P11 financed emissions** (`ghg`, 8 résumés). PCAF attribution over public emissions data, with an
  estimation ladder and a measured error rate on revenue-based estimates. Packet 11.
- **P12 emissions data QC** (`disclosure`, 6). Vintage-aware ingest of three public emissions sources
  with a rule engine that detects unit errors, scope confusion, and silent restatements. Packet 12.
- **P13 financial-data MCP** (`kensho`, 5). The Kensho/Capital IQ analogue built over SEC EDGAR XBRL.
  Your existing 23-company research platform is most of the way there. Packet 13.
- **P14 data center siting risk** (`datacenter`, 4). Facility locations against utility territories,
  WRI Aqueduct water stress, and grid carbon intensity. Small, fast, and the most topical thing you
  could put in front of a power-markets interviewer in 2026. Packet 14.
- **P15 sector pathways** (`frameworks`, 4). NGFS and IEA scenarios to SBTi SDA convergence pathways
  and portfolio alignment gaps. `~/projects/ngfs-scenario-explorer` is most of this already. Packet 15.
- **P16 multi-hazard analytics** (`hazard`, 3). Static exposure plus a near-real-time event tracking
  pipeline over FIRMS, NIFC, NHC, and Sentinel-1. Packet 16.
- **P17 asset location database** (`geo`, 4). Entity resolution and coverage measurement at S&P 100
  scale. Do not attempt the 9.5M-site claim publicly. Packet 17.

---

## Definition of done, per project

Every project ships all six of these or it does not count:

1. `README.md` that states the question, the data, the method, and the finding in under 400 words.
2. Reproducible environment (`pyproject.toml` or `requirements.txt`, pinned).
3. One command that regenerates every figure from raw data.
4. At least one chart good enough to screen-share in an interview.
5. A `FINDINGS.md` with the honest result, including what did not work.
6. A test that fails if the headline number changes unexpectedly.

---

## Knowledge packets

Seventeen packets live in `knowledge-packets/`, one per project, plus a cross-cutting glossary. Each
contains the domain background, the data sources with access notes and gotchas, the method in steps,
the numbers to know cold, and the interview questions you should expect with the shape of a good
answer.

Read the packet before building the project. Re-read it the night before an interview. The packets
are useful on their own even for projects you never build, because the interview questions are the
ones you will actually get.

| Packet | File |
|--------|------|
| 00 Cross-cutting glossary | [`00-glossary.md`](knowledge-packets/00-glossary.md) |
| 01 ENSO basket | [`01-enso-equity-basket.md`](knowledge-packets/01-enso-equity-basket.md) |
| 02 Net-load baseline and BTM solar | [`02-netload-btm-solar.md`](knowledge-packets/02-netload-btm-solar.md) |
| 03 Weather-driver sensitivity | [`03-weather-sensitivity.md`](knowledge-packets/03-weather-sensitivity.md) |
| 04 AI weather model benchmark | [`04-ai-weather-benchmark.md`](knowledge-packets/04-ai-weather-benchmark.md) |
| 05 Energy applications of AI forecasts | [`05-energy-applications.md`](knowledge-packets/05-energy-applications.md) |
| 06 Utility rate and affordability dataset | [`06-utility-affordability.md`](knowledge-packets/06-utility-affordability.md) |
| 07 Nowcasting and production pipelines | [`07-nowcast-productionization.md`](knowledge-packets/07-nowcast-productionization.md) |
| 08 Utility to parent-company crosswalk | [`08-utility-parent-crosswalk.md`](knowledge-packets/08-utility-parent-crosswalk.md) |
| 09 Climate risk model divergence | [`09-climate-risk-divergence.md`](knowledge-packets/09-climate-risk-divergence.md) |
| 10 Workbook health-check agent | [`10-workbook-doctor.md`](knowledge-packets/10-workbook-doctor.md) |
| 11 Financed emissions | [`11-financed-emissions.md`](knowledge-packets/11-financed-emissions.md) |
| 12 Emissions data QC | [`12-emissions-data-qc.md`](knowledge-packets/12-emissions-data-qc.md) |
| 13 Financial-data MCP server | [`13-financial-data-mcp.md`](knowledge-packets/13-financial-data-mcp.md) |
| 14 Data center siting risk | [`14-datacenter-siting-risk.md`](knowledge-packets/14-datacenter-siting-risk.md) |
| 15 Sector climate pathways | [`15-sector-pathways.md`](knowledge-packets/15-sector-pathways.md) |
| 16 Multi-hazard geospatial analytics | [`16-multihazard-geospatial.md`](knowledge-packets/16-multihazard-geospatial.md) |
| 17 Corporate asset location database | [`17-asset-location-database.md`](knowledge-packets/17-asset-location-database.md) |
