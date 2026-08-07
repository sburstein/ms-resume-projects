# PRD 05: Energy Applications of AI Weather Forecasts

| | |
|---|---|
| **Repo** | `ai-weather-bench` (same repo as PRD 04) |
| **Bullets covered** | `supply`, `supplyShort` (12 of 20 résumés) |
| **Packet** | [`05-energy-applications.md`](../knowledge-packets/05-energy-applications.md) |
| **Est.** | 25 human-hours, 6 to 9 agent session hours |
| **Priority** | 7 of 17 |
| **Model** | Sonnet for A and D. Opus for C (freeze-off index design). |
| **Prereqs** | PRD 04 complete through M3. Read `00-CONVENTIONS.md`. |

---

## 1. Objective

Four sub-projects, each answering an energy question with the PRD 04 forecast archives, each with its
own chart. Collectively they demonstrate that headline RMSE is the wrong metric for energy, because
energy value concentrates in the tails.

**Success test:** four figures, and a summary table showing that the model ranking under
energy-relevant metrics differs from the ranking under global RMSE.

---

## 2. Scope

**In:** peak load under extreme temperature; heat and cold wave detection; a gas freeze-off risk
index; wind and solar generation forecasting from 3D fields. CONUS focus with per-BA detail.

**Out:** price forecasting. Dispatch or unit commitment modelling. Anything requiring nodal data.
Storage optimization.

---

## 3. Data contracts

Inherits PRD 04. Adds:

| id | Source | Notes |
|---|---|---|
| `eia930_demand` | Hourly demand by BA | Target for sub-project A |
| `eia930_wind_solar` | Hourly wind and solar generation by BA | Target for sub-project D |
| `eia860_plants` | Plant locations, capacities, hub heights where available | For plant-level power curves |
| `eia_ngas_weekly` | EIA weekly natural gas production and storage | Validation for sub-project C |
| `ercot_reports` | ERCOT event reports and generation outage data | Uri / Elliott / Heather validation |
| `noaa_storm_events` | NOAA Storm Events Database | Independent event catalogue for heat and cold waves |

---

## 4. Sub-project A: peak load under extreme temperature

### A1. Load-temperature model
Train per BA on ERA5 temperature and EIA-930 demand. Spline temperature response with a flat comfort
band, plus lagged and rolling temperature for thermal mass, humidity, calendar terms.
**Accept:** fitted response is V-shaped with a flat bottom, and the top of the range shows a
steepening slope (AC efficiency loss) or a documented saturation. Plot it.

### A2. Forecast-driven scoring
Drive the load model with each forecast model's temperature at each lead time. Score three ways:
mean-hour MAE, **peak-hour MAE**, and **daily-maximum MAE**.
**Accept:** the three metrics produce different model rankings, or the null result is stated. Expect
AI models to underpredict peaks at long lead times because of blurring; verify against the PRD 04 M6
activity diagnostic and cross-reference the two findings.

**Figure:** peak-hour load error vs lead time, by model.

---

## 5. Sub-project B: heat-wave and cold-wave detection

### B1. Event definition
Percentile-based and travel-independent: daily max (or min) beyond the local 95th (or 5th) percentile
for that calendar day, persisting 3+ days. Implement a fixed-threshold alternative for comparison.
**Accept:** detected historical events reconcile with the NOAA Storm Events catalogue at better than
70% overlap. Report the actual overlap.

### B2. Detection metrics
Per model, per lead time: probability of detection, false alarm ratio, Critical Success Index, and
**consistent lead time** (first flag that is maintained through to the event, not first mention).
**Accept:** consistent lead time is implemented as specified. A model that flags, drops, and
re-flags must not be credited with the earliest flag. Test with a synthetic flip-flopping series.

**Figure:** CSI vs lead time and consistent-lead-time distribution, by model.

---

## 6. Sub-project C: gas freeze-off risk  **← Opus**

### C1. Index construction
Composite from: minimum temperature, continuous hours below 20 F and below 0 F, wind speed (wind
chill and equipment exposure), and antecedent precipitation. Basin-specific weights, because Permian
equipment is winterized very differently from Appalachian.
**Accept:** `config/freezeoff.yaml` records the components, weights, and the justification for each.
Weights are declared before fitting, not tuned to hit the events.

### C2. Validation against real events
Score the index against Winter Storm Uri (Feb 2021), Elliott (Dec 2022), and Heather (Jan 2024),
using EIA weekly gas production dips and ERCOT/SPP event reports as truth.
**Accept:** the index ranks all three events in the top percentile of its historical distribution.
If it does not, the construction is wrong. Also report false positives: how many times did the index
exceed the Uri threshold without a production dip?

### C3. Forecast lead time
For each model, how many days ahead did the index cross its warning threshold for each event, and did
it hold?
**Accept:** results reported per model per event. With only three events, state explicitly that this
is anecdotal rather than statistical. Do not compute a p-value on n=3.

**Figure:** index time series through Uri with each model's forecast index overlaid at 3, 5, 7, and
10 day leads.

---

## 7. Sub-project D: renewable generation from 3D fields

### D1. Hub-height wind
Extract 100 m wind directly where the model provides it; otherwise extrapolate from pressure-level
winds using a stability-dependent profile, **not a fixed shear exponent**.
**Accept:** a test demonstrates that a stability-dependent profile and a fixed exponent produce
materially different diurnal patterns, and the diurnal bias of the fixed-exponent version is
quantified. This is the technical point of the sub-project.

### D2. Power conversion
Plant-level power curves from EIA-860 capacities, aggregated to BA. **Do not apply a single power
curve to a regional average wind speed** (Jensen's inequality). Implement both and quantify the bias
from doing it wrong, as a demonstration.
**Accept:** the aggregate-then-convert bias is measured and reported.

### D3. Solar
Reuse `pvlib` from PRD 02 if available; otherwise implement a minimal irradiance-to-AC conversion.
AI models generally do not output irradiance, so derive it from cloud fields and **quantify the error
introduced by that conversion separately** from the forecast error.
**Accept:** conversion error and forecast error reported separately.

### D4. Scoring
Against EIA-930 hourly wind and solar generation by BA, by lead time.
**Figure:** wind generation MAE vs lead time, with a panel showing percentage error magnified
relative to wind-speed error (the cubic amplification).

---

## 8. Golden-number regression tests

- Peak-hour vs mean-hour MAE ratio at day 5, tolerance 5% relative.
- Uri index percentile rank, tolerance 1 percentile.
- Wind speed error to power error amplification factor, tolerance 10% relative.

---

## 9. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| n=3 for freeze-offs | Overclaiming | State explicitly that C is a case study, not a statistical test. |
| Tuning the freeze-off index to the events | Suspiciously clean detection | Declare weights in config before validating. Report false positives. |
| Fixed shear exponent | Diurnal bias in wind forecasts | D1 requires the stability-dependent version and a measurement of the difference. |
| Aggregation before power curve | Systematic wind generation bias | D2 measures it deliberately. |
| Demand data quality in EIA-930 | Noisy load model | Use the adjusted series; document gaps rather than interpolating. |

---

## 10. Summary deliverable

A single table: model ranking under global RMSE (from PRD 04) versus model ranking under peak-hour
error, extreme-event CSI, and wind generation MAE. If the rankings differ, that table is the whole
argument of both PRDs and the strongest thing you can put in front of an energy interviewer.
