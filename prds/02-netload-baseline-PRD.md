# PRD 02: Western US Net-Load Baseline with Behind-the-Meter Solar

| | |
|---|---|
| **Repo** | `netload-baseline` |
| **Bullets covered** | `btm`, `btmShort` (12 of 20 résumés) |
| **Packet** | [`02-netload-btm-solar.md`](../knowledge-packets/02-netload-btm-solar.md) |
| **Est.** | 35 human-hours, 9 to 14 agent session hours |
| **Priority** | 5 of 17. Highest-value project for power-market roles. |
| **Model** | Sonnet for M0 to M2. **Opus for M3 (PV simulation) and M5 (load model).** |
| **Prereqs** | Read `00-CONVENTIONS.md`. PRD 04's ERA5 loader can be reused but is not required. |

---

## 1. Objective

Reconstruct the invisible behind-the-meter solar layer for CAISO, recover true gross load, fit a
load model on gross load where the weather relationship is physically clean, and project net load by
subtracting simulated renewables. Then prove the BTM-aware model beats a naive model fit directly on
metered load, specifically at midday and on the evening ramp.

**Success test:** a duck-curve figure showing metered load, reconstructed gross load, and simulated
BTM for a clear and a cloudy spring day, plus an error table where the BTM-aware model wins on
midday MAPE and on three-hour ramp error.

---

## 2. Scope

**In**
- CAISO as the primary balancing authority. Two additional WECC BAs (suggest AZPS and PACE) in M7 to
  demonstrate generalization and to expose how much harder it is without California's public
  interconnection data.
- Hourly resolution, at least 3 full years, ending at the most recent complete month.
- Residential and small commercial BTM PV. Not BTM storage (note as a growing confound).

**Out**
- Sub-hourly / 5-minute modelling.
- Utility-scale solar simulation (use EIA-930 reported generation directly).
- Price modelling. Load and net load only.
- BTM battery dispatch, which increasingly distorts the evening ramp. Flag as a known limitation and
  quantify roughly using CA storage interconnection counts.

---

## 3. Data contracts

| id | Source | Endpoint | Notes |
|---|---|---|---|
| `eia930` | EIA-930 hourly demand and generation by BA | EIA API v2, `electricity/rto/region-data` | Use the **adjusted** demand series. Early years have known quality issues. |
| `caiso_oasis` | CAISO actual load, solar, wind, curtailment | OASIS SingleZip API | Awkward SOAP-ish zip responses, rate limited. Write a thin client with backoff and cache every response. |
| `ca_dg_stats` | California Distributed Generation Statistics (NEM interconnection) | `californiadgstats.ca.gov` bulk CSV | Per-application records. **Filter to energized status.** Fields include capacity, ZIP, install date, and sometimes tilt/azimuth. |
| `nsrdb` | NREL National Solar Radiation Database | NREL API, needs free key | GHI/DNI/DHI half-hourly, 4 km. Per-location API, so sample representative points, not every ZIP. |
| `era5` | ERA5 2m temperature, dewpoint, wind, cloud | Copernicus CDS | Queue can be slow. Request once, cache to local zarr. |
| `eia860` | Plant locations and capacities | EIA-860 | For utility-scale context |

**Licence check:** all public. CAISO OASIS terms permit programmatic access; respect the documented
rate limits.

---

## 4. Milestones

### M0. Scaffold and load data
Repo per conventions. `ingest/eia930.py` and `ingest/caiso.py`. Hourly load series for CAISO,
timezone-correct.

**Timezone is a real risk here.** EIA-930 publishes in UTC. CAISO operates on Pacific Prevailing
Time with DST. Solar position depends on true local solar time. Store everything in UTC internally,
convert only at the presentation layer, and write a test asserting that the daily solar maximum
occurs near local solar noon after conversion.
**Accept:** continuous hourly series with gaps explicitly identified and counted, not interpolated
away. A test asserts no duplicate timestamps across the DST transition.

### M1. Interconnection fleet reconstruction
`transform/fleet.py`. From `ca_dg_stats`, build installed AC capacity by ZIP and by month via
cumulative sum over energized install dates.
**Accept:** statewide cumulative capacity for the most recent month is within 15% of the CPUC or CEC
published BTM solar total. If it is not, the status filter or the deduplication is wrong. Report the
number you get against the published figure in FINDINGS.md.

### M2. Geographic and configuration assignment
Assign each ZIP to a CAISO sub-area and a climate zone. Assign a tilt and azimuth distribution:
empirical where the filings report it, otherwise a documented prior centred near 20 to 25 degrees
tilt with azimuth weighted toward south but with real east and west mass.
**Accept:** `config/pv_config.yaml` records the distribution and its justification. A sensitivity
switch exists to swap in a south-only fleet for comparison in M6.

### M3. BTM generation simulation  **← Opus**
`model/pv.py` using `pvlib`. Per representative configuration per climate zone:
NSRDB irradiance → solar position → Perez transposition to plane-of-array → cell temperature (SAPM
or PVsyst) → DC output → inverter model → AC output, with documented system losses and DC/AC ratio.
Weight by installed capacity and sum to a fleet hourly series.

**Accept, all three:**
- Simulated fleet capacity factor is physically plausible: annual ~18 to 22% for California, with a
  clear-sky summer midday hourly CF approaching 0.7 to 0.85 of nameplate AC.
- Temperature derating is visible: a hot clear day produces lower peak output than a mild clear day
  at the same irradiance. Test this explicitly; it is an easy thing to get silently wrong.
- Simulated output is zero at night for every day of the year. Trivial, and it catches timezone bugs.

### M4. Gross load reconstruction
`gross_load = metered_load + simulated_btm`.

**The validation gate, and the intellectual core of the project:** fit the same
temperature-and-calendar model to metered load and to reconstructed gross load. Gross load must be
*more* explicable: higher R-squared, and the systematic midday residual bias present in the metered
fit must be visibly reduced.
**Accept:** R-squared improves, and a plot of mean residual by hour-of-day shows the midday
distortion shrinking. **If it does not, the BTM simulation is wrong. Stop and fix M3 rather than
proceeding.** State the before and after numbers in FINDINGS.md.

### M5. Load model  **← Opus**
`model/load.py`. Fit on gross load. Regressors: spline or piecewise temperature response (V-shaped
with a flat comfort band roughly 60 to 68 F), lagged and rolling temperature for building thermal
mass, humidity or dewpoint, hour of day, day of week, holidays, month, and a trend term.

Start with a GAM (`pygam`) or gradient boosting (`lightgbm`). Do not start with a neural network.
Time-based train/test split, never random: the last 12 months are held out.
**Accept:** out-of-sample MAPE on gross load below 5%. The fitted temperature response is
V-shaped with a flat bottom, not monotonic. Plot it; if it is monotonic, the specification is wrong.

### M6. Net load projection and benchmark
`net_load = predicted_gross - simulated_btm - reported_utility_solar - reported_wind`.

Benchmark against a naive model fit directly on metered load with identical weather regressors.
Report, on the held-out period:

| metric | naive | BTM-aware |
|---|---|---|
| MAPE, all hours | | |
| MAPE, 10:00 to 15:00 local | | |
| MAE on daily minimum net load | | |
| MAE on maximum 3-hour ramp | | |

Plus the south-only-fleet sensitivity from M2.
**Accept:** the BTM-aware model wins on midday MAPE and on ramp error. If it wins on neither, report
that honestly with a diagnosis; do not tune until it wins.

### M7. Generalization
Repeat M1 to M6 for two more WECC BAs. Where interconnection data is not public, use a documented
top-down estimate and report the resulting accuracy loss.
**Accept:** results for all three BAs, with an explicit statement of how much California's data
transparency was worth in accuracy terms.

### M8. Figures and docs
1. **Duck curve**: metered load, reconstructed gross load, simulated BTM, for one clear and one
   cloudy spring day. The headline.
2. **Residual-by-hour** before and after BTM reconstruction. The proof.
3. **Error table** as a figure.
4. **Ramp distribution**: histogram of maximum daily 3-hour ramps by year, showing growth.

---

## 5. Golden-number regression tests

- Simulated BTM annual capacity factor, tolerance 1pp.
- Out-of-sample gross-load MAPE, tolerance 0.3pp.
- Midday MAPE improvement over naive, tolerance 0.5pp.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Timezone / DST errors** | Solar peak at the wrong hour, duplicate or missing hours in October and March | UTC internally, test asserting solar noon alignment. This is the single most likely source of a silently wrong answer. |
| Interconnection records include non-energized systems | Fleet capacity overstated, BTM over-simulated, gross load too high | Status filter in M1, validated against the published statewide total. |
| NSRDB API rate limits | Slow or failing fetches | Sample representative points per climate zone, not per ZIP. Cache aggressively. |
| Cloud timing error | Large hourly errors on partly cloudy days | Expected and acceptable. Quantify it: report error conditioned on clear-sky index. Geographic aggregation cancels some of it. |
| BTM storage confound | Evening ramp mismatch growing in recent years | Flag as a limitation, quantify roughly from CA storage interconnection counts, exclude from scope. |
| Fitting on metered load out of habit | Unstable coefficients, poor out-of-sample behaviour | M4 gate exists specifically to prevent this. |

---

## 7. Downstream

PRD 03 (weather-driver sensitivity) builds in this repo and depends on M5 and M6. Design
`model/load.py` so the fitted model is cheap to evaluate thousands of times, since Sobol sampling
requires it.
