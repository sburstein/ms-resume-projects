# PRD 04: AI Weather Model Benchmark

| | |
|---|---|
| **Repo** | `ai-weather-bench` |
| **Bullets covered** | `aiwx`, `aiwxBias` (12 of 20 résumés) |
| **Packet** | [`04-ai-weather-benchmark.md`](../knowledge-packets/04-ai-weather-benchmark.md) |
| **Est.** | 25 human-hours, 6 to 9 agent session hours |
| **Priority** | 4 of 17. Fastest credibility win in the plan. |
| **Model** | Sonnet throughout. Opus for M3 (metric correctness) if latitude weighting or regridding misbehaves. |
| **Prereqs** | Read `00-CONVENTIONS.md`. No GPU required. |

---

## 1. Objective

Score GraphCast, Pangu-Weather, FourCastNet, and IFS HRES against ERA5 out to 14 days, on a
protocol matching WeatherBench 2 so the numbers are comparable to published results, and identify
where the AI advantage crosses over.

**Success test:** one figure, RMSE versus lead time for Z500 and T850 with all models on the same
axes, plus a variance-retention (activity) diagnostic showing AI blurring, plus an extremes-only
re-score.

**Explicit non-goal:** running model inference. You are building a scoring harness over precomputed
forecast archives. Do not install GPU dependencies or download model weights.

---

## 2. Scope

**In**
- Models: GraphCast, Pangu, FourCastNet, IFS HRES (deterministic physics baseline), plus climatology
  and persistence as reference baselines.
- Variables: `geopotential@500` (Z500), `temperature@850` (T850), `2m_temperature`,
  `10m_wind_speed`, `mean_sea_level_pressure`.
- Lead times: 6h to 336h (14 days) at 6-hourly or 12-hourly steps.
- Evaluation year: 2020 (the WeatherBench 2 standard held-out year).
- Metrics: RMSE, ACC, bias, activity, all latitude-weighted.
- Regions: global, CONUS, WECC footprint.

**Out**
- Ensembles and probabilistic scoring (CRPS, spread-skill). Note as future work; GenCast and IFS ENS
  would require a different harness.
- Model inference and fine-tuning.
- Precipitation, which has its own verification literature and would double the scope.

---

## 3. Data contracts

| id | Source | Access | Notes |
|---|---|---|---|
| `wb2_forecasts` | WeatherBench 2 forecast archives | Public GCS: `gs://weatherbench2/datasets/` | Zarr. Read with `xarray` + `gcsfs`, anonymous access. No auth, no egress cost within reason. |
| `wb2_era5` | ERA5 on the WB2 grid | `gs://weatherbench2/datasets/era5/` | Ground truth. Already regridded and aligned; **use this rather than raw CDS ERA5** to avoid a regridding step. |
| `wb2_climatology` | WB2 climatology | same bucket | Reference baseline, needed for ACC |
| `ecmwf_opendata` | Live AIFS and HRES | `ecmwf-opendata` package | **Optional stretch only.** Recent forecasts only, for a live demo. |
| `noaa_isd` | Station observations | NOAA ISD via `meteostat` or direct | For the independent-truth cross-check in M5 |

**Read the WeatherBench 2 paper before writing code.** Adopting their exact protocol is the point;
inventing your own makes the numbers incomparable.

---

## 4. Milestones

### M0. Scaffold and data access
Repo per conventions. Verify anonymous read from the WB2 bucket and print the available forecast
datasets, variables, and date ranges. Record the exact zarr paths in `sources.yaml`.
**Accept:** a script lists available models and their init dates, and lazily opens one dataset
without downloading it. `xr.open_zarr` with `chunks={}` and no eager compute.

### M1. Alignment layer
`data/align.py`. Given a model name, return an xarray Dataset aligned to truth on: init times, lead
times, variables, vertical levels, grid, and units.

Handle explicitly:
- Different models publish different init cadences (00/12 UTC vs all four synoptic hours). Intersect.
- Some publish lead time as a timedelta, some as an integer step index.
- Units: geopotential (m2/s2) vs geopotential height (m). Assert and convert.
- Longitude convention: 0 to 360 vs -180 to 180.

**Accept:** for each model, a test asserts identical coordinates against truth after alignment, and
that the intersected init-time count is at least 300 for the evaluation year. Report the count per
model; a model with far fewer inits is not comparable and must be flagged.

### M2. Metrics  **← correctness-critical**
`metrics/core.py`:

```python
def lat_weights(lat: xr.DataArray) -> xr.DataArray:
    """cos(lat), normalized to mean 1."""

def rmse(fc, truth, weights) -> xr.DataArray: ...
def acc(fc, truth, clim, weights) -> xr.DataArray: ...
def bias(fc, truth, weights) -> xr.DataArray: ...
def activity(fc, clim, weights) -> xr.DataArray:
    """Std dev of forecast anomaly. Reveals blurring."""
```

All return arrays dimensioned by lead time.

**Accept, all four:**
- A test verifies `lat_weights` integrates correctly and that an unweighted vs weighted RMSE differ
  by a non-trivial amount (proving the weighting is actually applied).
- Reproduce a published WeatherBench 2 number for one model, one variable, one lead time, within 2%.
  **This is the key acceptance gate for the whole project.** If you cannot reproduce a published
  number, the harness is wrong and everything downstream is worthless. Do not proceed past M2 until
  this passes.
- A synthetic test where forecast equals truth yields RMSE 0, ACC 1, bias 0.
- A synthetic test where forecast equals climatology yields ACC ≈ 0.

### M3. Scoring run
Score all models, all variables, all lead times, global. Cache results to
`outputs/tables/scores.parquet` so figures do not re-compute.
**Accept:** scores exist for every (model, variable, lead) cell, or the gap is explained. RMSE
increases monotonically with lead time for every model (a violation means an alignment bug).

### M4. Regional and extremes slices
1. Regional masks for CONUS and WECC; re-score.
2. Extremes: restrict to grid-point-times where truth is in the top or bottom 1% of the local
   climatological distribution for that variable and calendar period; re-score.
**Accept:** extremes scoring produces a different model ranking than the headline, or, if it does
not, that null result is explicitly stated in FINDINGS.md rather than omitted.

### M5. Independent truth cross-check
Re-score 2m temperature against NOAA ISD station observations at roughly 200 stations, nearest-grid-
point matched, rather than against ERA5.
**Accept:** results reported alongside the ERA5-based ranking, with a note on whether the ranking
changed. ERA5 is an ECMWF product, so IFS has a structural home-field advantage that this check
exposes.

### M6. Figures and docs
1. **RMSE vs lead time**, Z500 and T850, all models plus climatology and persistence. The headline.
2. **Activity vs lead time** against ERA5's, showing AI variance loss. The sophisticated one.
3. **Skill difference** panel: each AI model minus IFS HRES, with the crossover point annotated.
4. **Extremes bar chart**: RMSE ratio (AI / IFS) for all data vs top-1% data.
**Accept:** conventions section 5.

---

## 5. Golden-number regression tests

- Reproduced WeatherBench 2 reference value, tolerance 2%.
- Day-5 Z500 RMSE for each model, tolerance 3%.
- Crossover lead time for Z500, tolerance 1 day.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Missing latitude weighting** | Metrics look wrong, Arctic dominates | M2 test asserts weighted and unweighted differ. Most common error in this field. |
| Init-time mismatch across models | One model looks anomalously good or bad | M1 intersects inits and asserts a minimum count. Report per-model counts in the README. |
| Unit confusion (geopotential vs height) | Z500 RMSE off by ~9.8x | Assert units in M1. Test with a known magnitude. |
| Downloading rather than lazily reading | Hours of transfer, disk full | `open_zarr` with lazy chunks. Compute only reduced metrics. Never `.load()` a full dataset. |
| Verifying against ERA5 only | IFS advantage baked in | M5 station cross-check. State the caveat in the README regardless. |
| Scope creep into ensembles | Timeline doubles | Explicitly out of scope. Note as future work. |

---

## 7. Downstream

PRD 05 (energy applications) builds in this repo and depends on M1 (alignment layer) and M3
(scores). The alignment layer is the reusable asset; design it as a clean public interface.
