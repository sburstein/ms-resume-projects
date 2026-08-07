# Packet 02: Western US Net-Load Baseline with Behind-the-Meter Solar

> **Résumé line:** "Defined the solar-weather inputs and behind-the-meter facility-data approach used
> to set the Western US load-forecast baseline, addressing a modeling gap where distributed rooftop
> solar is invisible to traditional load models but materially reshapes daytime net load."

**Repo:** `netload-baseline` | **Est. 35 hours** | **Build third (Phase B)**

---

## 1. The idea

Traditional load forecasting predicts metered demand from temperature, day of week, and season.
Those models were designed when all generation was large and grid-side, so they have no variable for
"how much invisible rooftop solar is running right now."

In California that omission is now enormous. When a cloud bank moves over the Central Valley,
metered load can jump several gigawatts in twenty minutes with no change whatsoever in what people
are actually consuming. A temperature-driven model sees an unexplained demand spike. A model that
knows the BTM fleet sees exactly what happened.

This project reconstructs the invisible layer, recovers true gross load, fits the load model on
gross load where the physics is clean, and then projects net load by subtracting simulated
renewables. That ordering is the whole insight.

---

## 2. The physics and market structure you need

**Why BTM solar is invisible.** The utility meter sits between the house and the grid. It measures
net flow. If the house consumes 3 kW and the roof produces 4 kW, the meter reads minus 1 kW. The
utility knows the net; it never observes either component. Under net energy metering the customer is
credited for exports, which is why interconnection filings exist at all and why we have any handle
on installed capacity.

**Why it matters more than the megawatts suggest.** California has roughly 17 to 19 GW of BTM solar
across about 2 million installations. Peak CAISO demand is roughly 45 to 50 GW. So on a clear spring
afternoon, BTM solar is displacing something like a third of what demand would otherwise be. The
midday net-load trough on a mild spring Sunday has dropped below 10 GW, and CAISO has recorded
negative wholesale prices for hours at a stretch.

**The duck curve, precisely.** Plot net load across a spring day. Morning demand rises normally.
From roughly 09:00 solar ramps hard and net load collapses into a midday belly. From roughly 16:00
solar falls off while human demand rises toward the evening peak, producing a ramp that in CAISO now
exceeds 13 to 15 GW over three hours. That ramp is the single most valuable feature of the curve for
storage, and it is entirely a solar artifact.

**Solar output physics you need for the simulation:**

- **GHI, DNI, DHI.** Global horizontal irradiance is total on a flat surface. Direct normal is the
  beam component perpendicular to the sun. Diffuse horizontal is scattered sky light. A tilted panel
  sees a transposition of all three, which is why you cannot just multiply GHI by capacity.
- **Plane-of-array irradiance.** What the tilted panel actually receives. Computed from GHI/DNI/DHI
  plus solar position plus tilt and azimuth. The Perez transposition model is the standard.
- **Temperature derating.** Silicon PV loses roughly 0.35 to 0.45% of output per degree C above
  25 C cell temperature. On a 40 C day, cell temperature can hit 60 C, so you lose 12 to 15%. Hot
  and sunny is not the same as maximally productive.
- **Soiling, shading, inverter clipping, and DC-to-AC ratio.** Residential systems typically run a
  DC/AC ratio around 1.15 to 1.25, so peak output clips. Total system losses in PVWatts default to
  about 14%.
- **Tilt and azimuth distribution.** Residential roofs are not all south-facing at optimal tilt. A
  west-facing fleet produces less total energy but shifts output later, which changes the evening
  ramp materially. Model this as a distribution, not a single value. This detail is the kind of
  thing that signals you have actually done the work.

---

## 3. Data

| Source | What | Access | Gotcha |
|--------|------|--------|--------|
| EIA-930 | Hourly demand, net generation by fuel, interchange, per balancing authority, 2015 to present | Free API and bulk CSV | Demand is *metered*, so BTM is already netted out. Early years have known data-quality issues; EIA publishes an adjusted series, use it. |
| CAISO OASIS | 5-minute actual load, solar, wind, curtailment | Free, awkward SOAP/zip API | Rate-limited. Cache aggressively. `pyiso`-style wrappers are mostly stale; write your own thin client. |
| CAISO "Today's Outlook" data | Daily net demand and curtailment CSVs | Free | Easiest path to a clean duck curve dataset |
| California Distributed Generation Statistics | NEM interconnection records: capacity, ZIP, install date, tilt where reported | Free CSV, `californiadgstats.ca.gov` | Large files. Records are per-application; some never energize. Filter on status. |
| NREL NSRDB | Satellite-derived GHI/DNI/DHI, half-hourly, 4 km grid, 1998 to present | Free API with key | API is per-location. For a fleet, sample representative points per climate zone rather than per ZIP. |
| NREL PVWatts v8 API | Hourly AC output given capacity, tilt, azimuth, location | Free API with key | Rate limited to 1000 requests/hour. Simulate once per representative configuration, then scale. |
| `pvlib-python` | Full PV modeling stack, runs locally | `pip install pvlib` | Strictly better than the API for a fleet simulation. Use this. |
| ERA5 | Temperature, cloud cover, wind, humidity | Copernicus CDS, free with account | CDS queue can be slow. Request once, cache to local zarr/netCDF. |

---

## 4. Method, in steps

1. **Build the BTM fleet by geography and vintage.** From interconnection records, produce installed
   AC capacity by ZIP (or by CAISO sub-LAP) and by month. Cumulative sum by install date, filtered
   to energized systems. Sanity check the statewide total against CPUC published figures.
2. **Assign a configuration distribution.** For each geography, assume a tilt distribution centered
   near 20 to 25 degrees and an azimuth distribution weighted toward south but with meaningful west
   and east mass. Where the filings report actual tilt/azimuth, use the empirical distribution.
3. **Simulate hourly BTM generation.** With `pvlib`: NSRDB irradiance to plane-of-array via Perez,
   cell temperature via the SAPM or PVsyst thermal model, DC output, inverter model, losses. Run per
   representative configuration per climate zone, then weight by installed capacity.
4. **Recover gross load.** `gross_load = metered_load + simulated_BTM`. This is the central step.
   Everything downstream depends on this reconstruction being defensible.
5. **Validate the reconstruction.** Gross load should be a *smoother, more weather-explicable*
   series than metered load. Test this: fit the same temperature-and-calendar model to both, and
   show R-squared improves and the midday residual bias disappears. If it does not, your BTM
   simulation is wrong. This is your acceptance test.
6. **Fit the load model on gross load.** Regressors: heating and cooling degree hours (with a
   nonlinear or spline temperature response, since load vs temperature is a hockey stick with a
   comfort minimum around 65 F), humidity, hour of day, day of week, holidays, a trend term for
   population and electrification, and lagged temperature to capture building thermal mass. Start
   with a GAM or gradient boosting; do not start with a neural net.
7. **Project net load.** `net_load = predicted_gross - simulated_BTM - forecast_utility_scale_solar
   - forecast_wind`. Now the model has an explicit, inspectable solar term instead of an implicit,
   confounded one.
8. **Benchmark.** Compare against a naive model fit directly on metered load with the same weather
   regressors. Report MAPE overall, MAPE in the 10:00 to 15:00 window, and error on the daily
   minimum net load and the maximum three-hour ramp. The naive model should be visibly worse midday
   and on ramps, which is the entire point.
9. **Extend beyond CAISO.** Repeat for two more WECC BAs (Arizona Public Service and PacifiCorp East
   are good choices) to show it generalizes and to expose how much harder it is where
   interconnection data is less public than California's.

---

## 5. Deliverables

- Duck-curve figure: metered load, reconstructed gross load, and simulated BTM, on the same axes,
  for one clear spring day and one cloudy spring day.
- Error table: naive vs BTM-aware model, overall and midday and on ramp.
- A short note on how much of the residual is BTM simulation error vs load-model error.

---

## 6. Numbers to know cold

- California BTM solar: roughly **17 to 19 GW** across about **2 million** systems.
- CAISO peak demand: roughly **45 to 52 GW**; all-time peak **52.1 GW**, September 2022.
- CAISO minimum net load has dropped below **10 GW** on mild spring days.
- CAISO three-hour evening ramp now exceeds **13 to 15 GW**.
- CAISO renewable curtailment: over **3 TWh** in 2024, mostly spring midday solar.
- PV temperature coefficient: roughly **minus 0.35 to 0.45 % per degree C** above 25 C.
- PVWatts default total system losses: **14%**.
- Typical residential DC/AC ratio: **1.15 to 1.25**.

---

## 7. Interview questions and how to answer

**"How would you estimate behind-the-meter solar if you cannot measure it?"**
Three approaches, and say all three: (1) bottom-up simulation from interconnection capacity plus
irradiance, which is what I built; (2) top-down statistical disaggregation, comparing metered load on
clear vs overcast days with matched temperature to back out the solar signature; (3) sampled AMI
data or utility-published estimates as a validation target. The bottom-up version is auditable,
which matters when someone asks why the number changed.

**"What breaks your simulation?"**
Cloud timing. Irradiance products are satellite-derived at 4 km, and a broken cumulus field is not
resolved. Aggregate fleet output across a large territory is smoothed by geographic diversity, so
the error partly cancels, but on a partly cloudy day in a single utility territory the hourly error
can be large. Also: systems that were approved and never energized, and undocumented curtailment or
inverter-level export limits.

**"Why fit on gross load rather than net?"**
Because the relationship between weather and human electricity consumption is stable and physical,
and the relationship between weather and metered load is contaminated by an unmeasured generation
term that has been growing 15% a year. Fitting on metered load means your temperature coefficients
are silently absorbing solar, so the model is unstable out of sample and will mis-forecast the moment
the fleet grows or a cloud pattern differs from the training distribution.

**"Why does a battery operator care about your baseline?"**
Because arbitrage revenue comes from the spread between the midday trough and the evening peak, and
the value of a fast-responding asset comes from the ramp. Both are properties of the *shape* of net
load, and the shape is set almost entirely by solar. A load forecast that gets the daily average
right and the shape wrong is worthless to them.

---

## 8. Further reading

- CAISO, "What the duck curve tells us about managing a green grid" (2016) and the annual Renewables
  Curtailment reports.
- Holmgren, Hansen and Mikofski, "pvlib python: a python package for modeling solar energy systems,"
  JOSS 2018. Then read the pvlib docs on transposition and thermal models.
- Hong and Fan, "Probabilistic electric load forecasting: A tutorial review," IJF 2016. The standard
  entry point to load forecasting method.
- Lawrence Berkeley National Lab, "Tracking the Sun" annual report, for fleet characteristics.
