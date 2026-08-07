# Packet 03: Weather-Driver Sensitivity on Net Load

> **Résumé line:** "Quantified weather-driver sensitivity on net load, isolating cloud cover as a
> driver of roughly 10% of net-load variance, and built forecast trajectories at the utility level."

**Repo:** `netload-baseline` (same repo as Packet 02) | **Est. 15 hours** | **Depends on P2**

---

## 1. The idea

Once you have a net-load model, the natural question is: of everything that makes net load bounce
around, how much traces to each weather driver? Temperature is the obvious one. The interesting
answer is that cloud cover, which almost no traditional load model carries as a variable, accounts
for a material share, and it is also the hardest variable to forecast well.

That combination is what makes it worth flagging. A driver that is both influential and poorly
forecast is exactly where forecast error concentrates, and therefore where money is made or lost.

---

## 2. What you need to understand

**Why cloud cover matters more for net load than for gross load.** Clouds barely change human
electricity consumption. A cloudy 85 F afternoon and a sunny 85 F afternoon feel similar indoors.
But clouds slash solar output, both BTM and utility-scale, so they hit net load hard. The cloud
sensitivity of net load is almost entirely a generation effect wearing a demand costume. Saying this
clearly is the single best signal that you understand the system rather than the regression.

**Why cloud cover is hard to forecast.** Temperature is a smooth, large-scale, well-constrained
field, and NWP models handle it well. Clouds form through subgrid-scale processes: convection,
microphysics, aerosol interaction, boundary-layer turbulence. Models parameterize these rather than
resolving them. Cloud fraction and cloud radiative properties remain among the largest sources of
error in both weather and climate models. So a variable that drives, say, a tenth of net-load
variance carries a disproportionate share of net-load *forecast error*.

**Variance vs error.** Keep these distinct or you will misstate your own result.
- *Share of variance* is how much of the observed spread in net load is associated with a driver.
- *Share of forecast error* is how much of your prediction error traces to mispredicting that driver.
The second is what a trader cares about. Compute both, and be explicit about which number you are
quoting. This distinction is a very likely follow-up question.

**Interaction effects.** Temperature and cloud interact: cloud cover suppresses daytime heating,
which lowers cooling demand, which partially offsets the solar loss. A naive one-at-a-time
sensitivity misses this. Sobol indices separate first-order from total effects, and the gap between
them is the interaction. Report both.

---

## 3. Data

Same as Packet 02, plus:

| Source | What | Note |
|--------|------|------|
| ERA5 total cloud cover (`tcc`) | Hourly fraction, 0 to 1 | Reanalysis cloud is itself model output, not observation. State this. |
| ERA5 surface solar radiation downwards (`ssrd`) | Hourly J/m2 | Often a better regressor than cloud fraction, since it is what the panel sees |
| NSRDB clear-sky GHI | Theoretical clear-sky irradiance | The ratio `GHI / clearsky_GHI` is the **clear-sky index**, the cleanest single cloud proxy for energy work. Use it. |
| ISD / ASOS station observations | Human and automated sky cover | Ground truth for validating reanalysis cloud |

**Recommendation:** use clear-sky index rather than raw cloud fraction. It is bounded, physically
meaningful, and directly proportional to what solar generation loses.

---

## 4. Method, in steps

1. **Fit the net-load model** from Packet 02. Keep it differentiable or at least cheap to evaluate,
   because sensitivity analysis needs thousands of evaluations.
2. **Define driver ranges empirically.** For each driver, take its observed distribution conditional
   on month and hour. Do not sample uniformly over an implausible range; a 10 C January afternoon in
   Fresno with zero cloud is not a scenario worth weighting.
3. **Sobol variance decomposition.** Use `SALib`. Generate Saltelli samples over the joint driver
   space, evaluate the model, compute first-order (S1) and total-order (ST) indices. S1 is the
   variance explained by that driver alone; ST includes all its interactions. Report both, per
   driver, and note that Sobol assumes independent inputs, which temperature and humidity are not.
   Handle that either by decorrelating first or by using a Shapley-effects variant, and say which
   you did.
4. **Cross-check with Shapley values.** Run SHAP on the fitted model against the actual historical
   input distribution. This respects the real correlation structure. Two methods agreeing is worth
   far more than one method reported confidently.
5. **One-at-a-time sweep for the narrative chart.** Hold everything at climatology, vary cloud from
   clear to overcast, plot the resulting net-load curve family. This is the chart people remember.
6. **Utility-level trajectories.** Repeat per service territory. A cloud over Sacramento does nothing
   for San Diego demand, and the BTM penetration differs sharply between territories, so the cloud
   share of variance should differ visibly between PG&E, SCE, and SDG&E. That heterogeneity is a
   result, not noise.
7. **Fan charts.** For a given forecast day, sample driver uncertainty from the historical
   distribution of NWP forecast error at that lead time, propagate through the model, and plot the
   10th, 50th, and 90th percentile net-load paths. This is what "forecast trajectories" means
   operationally.

---

## 5. What a good result looks like

You are not trying to reproduce 10%. You are trying to produce a defensible number with a stated
method and a stated uncertainty. Expect something in the range of 5 to 20% depending on territory,
season, and whether you quote first-order or total-order indices. Spring in a high-BTM territory
will be at the top of that range; winter in a low-solar territory at the bottom.

Write the seasonal breakdown. "Cloud accounts for roughly X% of net-load variance annually but
closer to Y% in March through May, when solar penetration relative to load is highest" is a far
better sentence than a single annual figure.

---

## 6. Interview questions and how to answer

**"How did you isolate cloud cover specifically?"**
This is the question the explainer flagged, and it is the likeliest one. "Two ways, because I did
not want to trust one. A Sobol variance decomposition on the fitted model, which gives first-order
and total-order indices so I can see interaction with temperature separately. And SHAP values against
the real historical input distribution, which handles the fact that temperature and cloud are
correlated. I reported the range across both rather than a single point estimate."

**"Is that variance or forecast error?"**
"Variance, at the point I quote it. Forecast error attribution is the more useful number and it is
higher, because cloud is much harder to predict than temperature. I computed both and they are not
the same figure."

**"Why should I believe a reanalysis cloud field?"**
"I should not, entirely. ERA5 cloud is model output with observations assimilated, not a direct
measurement, and cloud is exactly where models are weakest. That is why I used the satellite-derived
clear-sky index from NSRDB as the primary regressor and validated against ASOS sky-cover
observations at a sample of stations."

**"Which utility was most cloud-sensitive and why?"**
Give a specific answer with a mechanism: the territory with the highest ratio of solar capacity
(BTM plus utility scale) to peak load, moderated by how much of its load is weather-driven cooling.

---

## 7. Further reading

- Saltelli et al., *Global Sensitivity Analysis: The Primer*. Chapter 4 for Sobol.
- Owen, "Sobol' indices and Shapley value," SIAM/ASA JUQ 2014. Explains when the two diverge.
- Bony et al., "Clouds, circulation and climate sensitivity," Nature Geoscience 2015, for why cloud
  remains the hard part of atmospheric modeling.
