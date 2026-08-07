# Packet 04: AI Weather Model Benchmark

> **Résumé line:** "Evaluated and back-tested next-generation AI weather models (GraphCast,
> FourCastNet, Pangu) against industry-standard commercial weather vendors, measuring prediction
> error out to a 14-day horizon."

**Repo:** `ai-weather-bench` | **Est. 25 hours** | **Build first (Phase A)**

---

## 1. The idea

Build a scoring harness that takes forecast archives and a truth dataset and produces error-vs-lead-
time curves for every model on the same grid, same variables, same dates, same metrics. Then answer
the only question that matters: at what lead time does each approach win, and where do the curves
cross?

This is the fastest credibility win in the plan. It is self-contained, it needs no GPU if you score
precomputed forecasts, and it ends in a chart that instantly communicates that you have done real
verification work.

---

## 2. The science you need to have cold

**How physics-based NWP works.** Divide the atmosphere into a 3D grid. At each point, track wind,
temperature, pressure, humidity, and condensate. Step the primitive equations (Navier-Stokes on a
rotating sphere, plus thermodynamics, plus continuity) forward in time. Processes smaller than a grid
cell (convection, cloud microphysics, turbulence, radiation) are *parameterized*, meaning
approximated by empirical schemes. ECMWF's IFS runs at roughly 9 km globally and takes on the order
of an hour on thousands of supercomputer cores.

**Data assimilation is the underrated half.** Before every forecast run, the model state is updated
with millions of observations (satellite radiances, radiosondes, aircraft, buoys) via 4D-Var, which
finds the model trajectory most consistent with both observations and the prior forecast. Most of the
improvement in forecast skill over the past forty years came from assimilation and observations, not
from the dynamical core. Worth knowing, because every AI model currently depends on a physics-based
assimilation system to produce its initial conditions. The AI models replaced the forecast step, not
the analysis step. That is the honest framing and it is what separates someone who read a headline
from someone who read the papers.

**The AI models:**

| Model | Origin | Architecture | Resolution | Notes |
|-------|--------|-------------|------------|-------|
| **FourCastNet** | NVIDIA, 2022 | Adaptive Fourier Neural Operator, then a diffusion variant | 0.25 deg | First to show the approach was viable at scale |
| **Pangu-Weather** | Huawei, 2023 | 3D Earth-Specific Transformer | 0.25 deg | Hierarchical temporal aggregation: separate 1h, 3h, 6h, 24h models composed to reduce error accumulation |
| **GraphCast** | Google DeepMind, 2023 | Graph neural network on a multi-mesh icosahedral grid | 0.25 deg | Beat IFS on ~90% of 1380 verification targets. Published in Science. |
| **GenCast** | Google DeepMind, 2024 | Diffusion-based ensemble | 0.25 deg | Probabilistic rather than deterministic. Changes the comparison. |
| **AIFS** | ECMWF, 2024-25 | Graph transformer | ~0.25 deg | Operational at an actual weather center. The most credible AI comparison point. |
| **Aurora** | Microsoft, 2024 | Foundation model, fine-tuned per task | 0.1 deg | Air quality and wave variants |

All are trained on ERA5, typically 1979 to 2017 for training with later years held out. They run a
global forecast in **under a minute on a single GPU** versus hours on a supercomputer. That is a
four to five order of magnitude reduction in compute.

**The known weaknesses, which are the interesting part:**

1. **Blurring.** Models trained on MSE loss produce forecasts that become increasingly smooth with
   lead time, because the conditional mean is the MSE-optimal prediction under uncertainty. They
   score well on RMSE while systematically under-representing extremes. This is why RMSE alone is a
   misleading verification metric for AI models, and why activity or spectral diagnostics matter.
2. **Extremes.** Precisely because of blurring, peak intensity of cyclones, heat waves, and heavy
   precipitation tends to be underestimated. This matters enormously for energy applications, where
   the tails are where the money is.
3. **Physical consistency.** Nothing enforces conservation of mass or energy. Outputs can be
   physically inconsistent in ways a dynamical model cannot be.
4. **Distribution shift.** Trained on the historical climate. Behavior in a genuinely unprecedented
   event is unconstrained.
5. **Dependence on IFS analysis** for initial conditions, as above.

**Verification vocabulary:** RMSE, ACC, bias, CRPS (for probabilistic forecasts), spread-skill
ratio, and activity (the standard deviation of the forecast anomaly field, which reveals blurring
directly). Score against ERA5 as truth, and note that ERA5 is itself an ECMWF product, which gives
IFS a small home-field advantage in verification. Verifying against independent station observations
is the fair-minded cross-check.

---

## 3. Data

| Source | What | Access | Gotcha |
|--------|------|--------|--------|
| **WeatherBench 2** | Precomputed forecast archives for GraphCast, Pangu, FourCastNet, IFS HRES, IFS ENS, plus ERA5 truth, as zarr on Google Cloud Storage | Free, public GCS bucket, no auth | This is the shortcut. Start here. Read the WB2 paper first so you use their exact evaluation protocol. |
| ECMWF open data | Live AIFS and HRES forecasts, 0.25 deg | Free, `ecmwf-opendata` Python package | Only recent forecasts retained. Good for live demo, not for a long backtest. |
| Copernicus CDS | ERA5 | Free with account | Slow queue for large requests. WB2's copy is faster. |
| NOAA GFS on AWS Open Data | Physics baseline, full archive | Free S3 | GRIB2 parsing is tedious. `cfgrib` or `herbie` help. |
| `ai-models` (ECMWF) | Runner for GraphCast, Pangu, FourCastNet | GitHub, open | Needs a GPU and model weights. Only go here if you want live inference. |
| MeteoStat / ISD | Station observations | Free | For independent verification away from ERA5 |

**On "commercial weather vendors":** you cannot access DTN, Atmospheric G2, Vaisala, or Speedwell
outside the bank. Substitute the operational physics models (IFS HRES, GFS) as the industry baseline
and say plainly in the README that the commercial vendor comparison is the one piece you rebuilt with
a public proxy. Honesty here costs nothing and protects you.

---

## 4. Method, in steps

1. **Read the WeatherBench 2 paper** and adopt its protocol: same evaluation dates (2020 is the
   standard held-out year), same variables, same regridding, latitude-weighted metrics. Do not
   invent your own protocol; matching a published one makes your numbers comparable and defensible.
2. **Latitude weighting matters.** Grid cells shrink toward the poles. Every metric must be weighted
   by cos(latitude) or you will overweight the Arctic. Getting this wrong is the classic beginner
   error in this field.
3. **Score the headline variables:** Z500, T850, T2m, 10m wind speed, and MSLP. Compute RMSE, ACC,
   and bias by lead time from 6 hours to 14 days.
4. **Plot the crossover.** The expected shape: AI models lead at short and medium range, the gap
   narrows with lead time, and by days 10 to 14 everything converges toward climatology. Where the
   curves cross, and for which variable, is your finding.
5. **Add the activity diagnostic.** Plot forecast anomaly standard deviation vs lead time against
   ERA5's. AI models will visibly lose variance with lead time. Physics models hold it. This is the
   blurring result made visual, and it is the single most sophisticated thing in the project.
6. **Extremes verification.** Restrict to the top and bottom 1% of the truth distribution and re-score.
   AI advantage typically shrinks or reverses. Report it.
7. **Regional slice.** Score over CONUS and over the WECC footprint separately from the global
   number. Global skill is not what an energy trader buys.
8. **Independent truth check.** Re-score T2m against ISD station observations rather than ERA5, at
   least for a sample. If model rankings change, that is a real finding about verification-dataset
   bias.

---

## 5. Numbers to know cold

- GraphCast beat IFS HRES on roughly **90% of 1380** verification targets (Science, 2023).
- AI forecast runtime: **under one minute on a single GPU**, vs roughly an hour on a supercomputer
  for IFS.
- Training data: **ERA5, 1979 onward**, 0.25 degree, 37 pressure levels.
- Useful-skill convention: **ACC 0.6 for Z500** is the traditional cutoff, reached around day 7 to
  10 for modern models.
- ERA5 grid: **0.25 degrees**, roughly 31 km at the equator, hourly, 1940 to present.
- Modern forecast skill improves by roughly **one day per decade**: today's day-6 forecast is about
  as good as a day-5 forecast ten years ago.

---

## 6. Interview questions and how to answer

**"Which model was better?"**
Never pick a single winner. "It depends on the horizon and the variable, and the crossover is the
actual answer. The AI models led at short and medium range on the standard upper-air variables. The
physics models held up better on extremes and on variance retention at long lead. If I were buying
one for peak-load work, I would weight extreme-event performance much more heavily than headline
RMSE, and that changes the ranking."

**"Why do AI models score well on RMSE but disappoint on extremes?"**
The blurring explanation from section 2, stated crisply: MSE-optimal prediction under uncertainty is
the conditional mean, so the model hedges toward the mean and loses amplitude. It is a property of
the loss function, not a bug in the architecture, which is exactly why the field moved to diffusion
ensembles like GenCast.

**"Do these replace physics models?"**
"Not yet, and not in the way the headlines suggest. They replaced the forecast integration step, not
the data assimilation step. Every one of them takes initial conditions from a physics-based analysis
system, so if ECMWF turned off 4D-Var tomorrow, GraphCast would have nothing to start from. The
interesting recent work is end-to-end learned assimilation, and that is the thing to watch."

**"How did you back-test?"**
"Held-out year, matching the WeatherBench 2 protocol so my numbers are comparable to published ones.
Latitude-weighted metrics, ERA5 as truth, plus an independent station-based check because ERA5 is an
ECMWF product and I did not want to hand IFS a home-field advantage without noting it."

---

## 7. Further reading

- Lam et al., "Learning skillful medium-range global weather forecasting," Science 2023 (GraphCast).
- Bi et al., "Accurate medium-range global weather forecasting with 3D neural networks," Nature 2023
  (Pangu).
- Pathak et al., "FourCastNet," arXiv 2022.
- Price et al., "Probabilistic weather forecasting with machine learning," Nature 2024 (GenCast).
- Rasp et al., "WeatherBench 2," JAMES 2024. Read this one twice; it is the evaluation bible.
- Ben Bouallègue et al., "The rise of data-driven weather forecasting," BAMS 2024. ECMWF's own
  assessment, notably even-handed about limitations.
- You already have `~/nvidia-earth2-learning` and a book on AI weather modeling. Mine both.
