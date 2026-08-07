# Packet 05: Energy Applications of AI Weather Forecasts

> **Résumé line:** "Applied those models to energy supply and demand: peak load prediction under
> extreme temperatures, heat-wave and cold-wave detection, cold conditions relevant to gas freeze-off
> risk, and renewable generation forecasting from three-dimensional weather fields."

**Repo:** `ai-weather-bench` (same repo as Packet 04) | **Est. 25 hours** | **Depends on P4**

---

## 1. The idea

A global RMSE number is not a business case. This project takes the same forecast archives and asks
four energy questions where the answer has a dollar value attached, and where the *tails* matter far
more than the average. That is the pivot from meteorology to markets, and it is where the AI models
look worse than their headline scores suggest.

Four sub-projects, each small, each with its own chart.

---

## 2. Sub-project A: peak load under extreme temperature

**Why peak, not average.** Grids are built to survive their peak. Scarcity pricing means the top 1%
of hours can carry a double-digit share of annual wholesale value. In ERCOT the price cap has been
as high as $9,000/MWh (now $5,000 with ORDC adders layered on top), so a handful of hours dominate.
Predicting the average load well and the peak badly is a commercially useless model.

**The load-temperature relationship.** It is a hockey stick, or more precisely a V with a flat
bottom. Below roughly 60 F, load rises with heating demand. Between about 60 and 68 F, load is flat
at its comfort minimum. Above roughly 68 F, load rises steeply with cooling demand, and the slope
*steepens* at the top because air conditioners lose efficiency as ambient temperature rises and
because thermostat setpoints get overrun. Fit this with a spline or piecewise linear model, never a
single linear term.

**Thermal mass and saturation.** Buildings integrate temperature over days. The third consecutive
100 F day produces higher load than the first, because building envelopes and interior mass have
heated through. Include lagged temperature or a moving average. Conversely, at extreme heat load can
saturate: once every AC unit in the territory is running flat out, further temperature increase adds
less than the slope predicts.

**Method.** Train the load model on ERA5 historical temperature and EIA-930 demand. Then drive it
with each forecast model's temperature field at each lead time, and score *peak-hour* error and
*daily-maximum* error separately from mean error. Expect the AI models' blurring problem to show up
here as systematic peak underprediction at longer lead times.

---

## 3. Sub-project B: heat-wave and cold-wave detection

**Definition matters and there is no single standard.** Options: fixed threshold (three consecutive
days above 95 F), percentile threshold (daily max above the local 95th percentile for that calendar
day, for three or more days), or an impact-based definition tied to load. Use the percentile version
as primary because it travels across climates, and say why. A 95 F day in Phoenix is routine; in
Seattle it is a grid emergency.

**Metrics.** This is a detection problem, so use detection metrics, not RMSE:
- **Hit rate / probability of detection.** Of real events, how many did the model flag?
- **False alarm ratio.** Of flagged events, how many were not real?
- **Critical Success Index** or **Equitable Threat Score**, which combine both.
- **Lead time to detection.** The commercially important one: how many days ahead does the model
  first flag the event and keep flagging it? Flagging at day 9, dropping it at day 6, then re-flagging
  at day 3 is not a usable signal, so measure *consistent* lead time, not first mention.

**The finding to look for.** AI models tend to detect events earlier but with damped amplitude. So
they may win on lead time and lose on intensity. Report both axes.

---

## 4. Sub-project C: gas freeze-off risk

**The physics, because this is your best interview story.** Natural gas leaves the wellhead carrying
entrained water vapor and light hydrocarbon liquids. In extreme cold, three things happen: liquid
water freezes in wellhead equipment, valves, and gathering lines; **hydrates** form, which are
ice-like crystalline solids of water and methane that plug pipes at temperatures *above* freezing
under pressure; and compressor stations and electric-driven equipment fail or lose power.

Production drops precisely when heating and power demand peak. Supply and demand move in opposite
directions at the same moment. That is the whole reason winter gas and power markets have such
violent tails.

**Winter Storm Uri, February 2021.** ERCOT lost roughly 30 to 35 GW of generation. Gas production in
Texas fell by around half at the worst. Prices hit the $9,000/MWh cap for days. Spot gas at Waha and
the Oklahoma hubs traded in the hundreds of dollars per MMBtu against a normal $3. At least 246
deaths were attributed by the state, with independent estimates considerably higher. ERCOT came
within minutes of an uncontrolled cascading blackout that would have taken weeks to months to
recover. Winter Storm Elliott in December 2022 and Winter Storm Heather in January 2024 repeated
smaller versions of the same pattern.

**Building the index.** A freeze-off is not a single-temperature event. Construct a composite from:
minimum temperature, duration of continuous hours below 20 F and below 0 F, wind speed (wind chill
drives equipment failure and heat loss from uninsulated lines), and whether precipitation preceded
the cold (wet then freezing is far worse than dry cold). Weight by basin, because the Permian and
Appalachia are winterized very differently: Texas equipment is built for heat, Appalachian equipment
for cold.

**Validation data.** EIA weekly natural gas production estimates and the EIA Natural Gas Monthly show
the dips. ERCOT and SPP post event reports. FERC/NERC published a joint inquiry into Uri that is
worth reading for the causal chain.

**Method.** Compute the index from each forecast model at each lead time, and score how early and how
reliably each model would have flagged Uri, Elliott, and Heather. A model that misses the tail is
useless here even if its annual RMSE is excellent.

---

## 5. Sub-project D: renewable generation from 3D weather fields

**Why three-dimensional matters.** Modern turbine hub heights are 90 to 150 m, with rotor tips above
200 m. Wind at 10 m, which is the standard surface variable, is a poor proxy. The wind profile
follows roughly a power law or log law, and the exponent depends on **atmospheric stability**: in a
stable nocturnal boundary layer, wind shear is strong and hub-height wind can be more than double the
10 m value; in an unstable convective afternoon boundary layer, the profile is nearly uniform. Using
a fixed shear exponent introduces a diurnal bias that is systematic and therefore expensive.

Use the model's pressure-level winds (850 hPa and below) or, better, the 100 m wind field that ERA5
and the AI models provide directly.

**Power curve.** Turbine output is roughly cubic in wind speed between cut-in (about 3 m/s) and rated
(about 12 to 14 m/s), flat at rated power up to cut-out (about 25 m/s), then zero. The cubic region
is the problem: a 10% wind speed error becomes a roughly 33% power error. This is why wind forecasting
error is so much larger than temperature forecasting error in percentage terms, and it is a good
thing to be able to state.

**Aggregation.** Do not apply a single turbine power curve to a regional average wind speed. Jensen's
inequality bites: the average of the cubes is not the cube of the average. Use a smoothed
regional power curve, or simulate at the plant level and sum. EIA-860 gives plant locations and
capacities.

**Solar.** Reuse the pvlib pipeline from Packet 02. The forecast input is model-derived irradiance
or cloud cover converted to irradiance. AI weather models generally do not output irradiance
directly, so you will derive it from cloud fields, and that conversion is itself a source of error
worth quantifying.

**Validation.** EIA-930 reports hourly wind and solar generation by balancing authority. Score
against it.

---

## 6. Numbers to know cold

- ERCOT offer cap: **$5,000/MWh** (reduced from $9,000 after Uri).
- Uri: roughly **30 to 35 GW** of ERCOT generation lost; Texas gas production down roughly **half**;
  **246** deaths per the state's official count.
- Turbine cut-in **~3 m/s**, rated **~12 to 14 m/s**, cut-out **~25 m/s**; power is cubic in between.
- A **10%** wind speed error becomes roughly a **33%** power error in the cubic region.
- Load-temperature comfort minimum: roughly **60 to 68 F**.
- Hub heights: **90 to 150 m** for modern onshore turbines.
- Methane hydrates can form **above 32 F** under pipeline pressure.

---

## 7. Interview questions and how to answer

**"Why would a forecast model that scores well globally fail for energy?"**
Because energy value is concentrated in the tails and in specific regions, and global RMSE measures
neither. An MSE-trained model hedges toward the conditional mean, which is exactly the wrong
behavior when you are pricing the hottest afternoon of the year. I scored peak-hour error and
extreme-event detection separately for that reason, and the ranking is not the same as the RMSE
ranking.

**"Explain gas freeze-offs."**
Use section 4. Lead with the counterintuitive part: supply falls at the moment demand peaks. Then the
hydrate detail, then Uri as the case study, then the point that basin winterization differs.

**"How would you forecast wind generation?"**
Hub-height wind from the 3D field, not 10 m wind, because shear depends on stability and using a
fixed exponent bakes in a diurnal bias. Then a plant-level power curve rather than a regional
average, because the cubic response means aggregation before the power curve is biased. Then validate
against EIA-930.

---

## 8. Further reading

- FERC/NERC, "The February 2021 Cold Weather Outages in Texas and the South Central United States,"
  November 2021. The definitive Uri post-mortem.
- Hong, Pinson et al., "Global Energy Forecasting Competition" papers, for load and wind forecasting
  benchmark method.
- Draxl et al., "The Wind Integration National Dataset (WIND) Toolkit," Applied Energy 2015.
- Sengupta et al., "The National Solar Radiation Data Base (NSRDB)," Renewable and Sustainable Energy
  Reviews 2018.
