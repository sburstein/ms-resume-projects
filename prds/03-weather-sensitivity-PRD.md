# PRD 03: Weather-Driver Sensitivity on Net Load

| | |
|---|---|
| **Repo** | `netload-baseline` (same repo as PRD 02) |
| **Bullets covered** | `cloud` (11 of 20 résumés) |
| **Packet** | [`03-weather-sensitivity.md`](../knowledge-packets/03-weather-sensitivity.md) |
| **Est.** | 15 human-hours, 4 to 6 agent session hours |
| **Priority** | 6 of 17 |
| **Model** | Opus for M2 (variance decomposition correctness). Sonnet elsewhere. |
| **Prereqs** | PRD 02 complete through M6. Read `00-CONVENTIONS.md`. |

---

## 1. Objective

Decompose net-load variance by weather driver, per utility service territory, and quantify the share
attributable to cloud cover both as *variance* and as *forecast error*. Then produce utility-level
forecast trajectories with honest uncertainty bands.

**Success test:** a per-territory stacked variance-share chart with cloud broken out, computed by two
independent methods that agree, plus a fan chart of net-load trajectories.

---

## 2. Scope

**In:** Sobol indices (first-order and total-order) and SHAP values on the PRD 02 load model. Drivers:
clear-sky index, temperature, humidity, wind speed. Seasonal and per-territory breakdowns. Fan charts
driven by empirical NWP forecast-error distributions.

**Out:** re-fitting the load model (use PRD 02's). Causal inference claims. Anything requiring new
data beyond the clear-sky index derivation.

---

## 3. Data contracts

Inherits PRD 02's sources. Adds:

| id | Source | Notes |
|---|---|---|
| `nsrdb_clearsky` | NSRDB clear-sky GHI | Already fetched in PRD 02. Derive **clear-sky index** = GHI / clearsky_GHI, clipped to [0, 1.2]. **Use this as the cloud regressor, not ERA5 cloud fraction.** It is bounded, satellite-derived, and directly proportional to what solar generation loses. |
| `era5_tcc` | ERA5 total cloud cover | Secondary, for comparison only. Reanalysis cloud is model output, not observation. |
| `noaa_isd_sky` | ASOS/ISD sky cover observations | Validation of the cloud fields at ~30 California stations |
| `gefs_reforecast` | NOAA GEFS reforecast, or forecast-error statistics from PRD 04 | For the empirical forecast-error distributions driving the fan charts |

---

## 4. Milestones

### M0. Clear-sky index and cloud validation
Derive the clear-sky index per territory. Validate ERA5 cloud fraction and the clear-sky index
against ASOS sky-cover observations.
**Accept:** correlation between clear-sky index and station sky cover reported per station. State
which cloud proxy you adopted and why, in METHODOLOGY.md.

### M1. Driver distributions
`sensitivity/distributions.py`. For each driver, the empirical distribution conditional on month and
hour, for each territory. **Do not sample uniformly over an implausible joint range.**
**Accept:** a test asserts sampled scenarios respect physical bounds (clear-sky index in [0, 1.2],
relative humidity in [0, 100], no 10 C July afternoons in Fresno).

### M2. Variance decomposition  **← Opus**
Two independent methods, both required.

1. **Sobol** via `SALib`. Saltelli sampling over the driver space, evaluate the PRD 02 model, compute
   S1 (first-order) and ST (total-order). Sobol assumes independent inputs, which temperature and
   humidity are not. Handle this explicitly: either decorrelate via a copula transform or use a
   Shapley-effects estimator. **Document which you chose and why.**
2. **SHAP** on the fitted model against the real historical input distribution, which respects the
   actual correlation structure.

**Accept, all four:**
- Sobol S1 values sum to <= 1.0; ST >= S1 for every driver. Assert both.
- The two methods agree on the *ranking* of drivers. If they disagree, that is a finding to
  investigate and report, not to average away.
- Cloud share is reported as a range across the two methods, never as a single point.
- Interaction magnitude (ST minus S1) is reported per driver.

### M3. Variance vs forecast error
Two distinct numbers, kept distinct throughout.
- **Variance share:** from M2.
- **Forecast error share:** propagate each driver's empirical NWP forecast error at a given lead time
  through the load model, holding others at truth, and attribute the resulting net-load error.
**Accept:** both reported side by side in a table. The README must state which number any headline
quote refers to. Cloud's forecast-error share should exceed its variance share; if it does not,
explain why.

### M4. Territory and seasonal breakdown
Repeat per utility service territory (PG&E, SCE, SDG&E at minimum) and per season.
**Accept:** cloud share differs measurably across territories, and the difference correlates with
solar-capacity-to-peak-load ratio. Report that correlation; it turns heterogeneity from noise into a
mechanism.

### M5. Forecast trajectories
Fan charts: for a chosen forecast day, sample driver uncertainty from the empirical forecast-error
distribution at each lead time, propagate, and plot P10 / P50 / P90 net-load paths per territory.
**Accept:** intervals widen with lead time. Coverage check: over a held-out period, the realized
value falls inside the P10 to P90 band close to 80% of the time. Report the actual coverage; a badly
calibrated fan chart is worse than none.

### M6. Figures and docs
1. **Stacked variance shares** per territory, cloud broken out, with the method range shown.
2. **One-at-a-time sweep**: net-load curve family as clear-sky index varies from overcast to clear,
   others at climatology. The narrative chart.
3. **Fan chart** for one territory with the realized path overlaid.
4. **Variance vs forecast-error** comparison table as a figure.

---

## 5. Golden-number regression tests

- Cloud first-order Sobol index, annual, CAISO-wide, tolerance 2pp.
- Fan chart P10 to P90 empirical coverage, tolerance 5pp from nominal 80%.
- Sobol S1 sum <= 1.0 and ST >= S1, exact assertions.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| Correlated inputs invalidate Sobol | S1 sum far from 1, or nonsensical indices | Decorrelate or use Shapley effects. Document the choice. |
| Quoting variance share as if it were error share | Overstated claim | M3 exists to keep them separate. Enforce in README wording. |
| Using ERA5 cloud as the primary regressor | Cloud share biased by reanalysis error | Clear-sky index is primary. ERA5 is comparison only. |
| Sobol evaluation cost | Sampling is slow if the model is expensive | PRD 02 M5 requires a cheap-to-evaluate model. If it is not, fit a fast surrogate and report surrogate fidelity. |
| Overfit narrative to hit ~10% | Dishonest result | Whatever the number is, publish it. Expect 5 to 20% depending on territory and season. |

---

## 7. Expected result

Do not target 10%. Expect a range of roughly 5 to 20%, highest in spring in high-BTM territories and
lowest in winter in low-solar territories. The seasonal and territorial breakdown is a better result
than any single annual figure.
