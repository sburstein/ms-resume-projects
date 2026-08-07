# Packet 09: Catastrophe and Climate Risk Model Divergence

> **Résumé line:** "Ran a systematic technical evaluation of 30+ catastrophe and climate risk data
> vendors, showing that historical catastrophe models correlate 80-97% while forward-looking climate
> models show low to negative agreement, and tracing the divergence to downscaling and normalization
> choices."
>
> **Short variant:** "...isolating methodological differences from real signal by comparing model
> output across an identical asset portfolio."

**Repo:** `climate-risk-divergence` | **Est. 25 hours** | **Tier B: public proxy for walled vendors**

---

## 1. The idea

You cannot license Verisk, Moody's RMS, Jupiter, XDI, First Street, and two dozen others. But the
*finding* does not depend on those specific vendors. The divergence is caused by methodological
choices that are visible in open models, so you can reproduce the mechanism directly, and in one
respect do better than the original study: with open models you can hold one choice fixed and vary
the other, which a vendor comparison can never do because vendors will not tell you what they did.

That reframing is the pitch. "I could not buy the vendors, so I rebuilt the experiment in a setting
where I could isolate the cause."

---

## 2. What you need to understand

**Catastrophe models.** Born in the insurance industry in the late 1980s (AIR, now Verisk, and RMS).
Structure is always the same four modules:
1. **Stochastic event set.** Tens of thousands of synthetic events, generated to match the historical
   frequency-severity distribution.
2. **Hazard module.** For each event, the physical intensity at each location (wind speed, flood
   depth, ground acceleration).
3. **Vulnerability module.** Damage functions mapping intensity to loss ratio, by construction type,
   occupancy, age, and height.
4. **Financial module.** Applies deductibles, limits, and reinsurance structures to get loss.

They agree with each other reasonably well because they are calibrated to the *same* historical loss
record. That shared calibration target is the reason for the high correlation, and it is the point to
make: their agreement is evidence of common calibration, not independent confirmation.

**Climate risk models.** Forward-looking, much newer, and calibrated to nothing, because the future
has not happened. They chain: emissions scenario, to global climate model, to downscaling, to hazard
translation, to vulnerability, to score. Every link introduces divergence, and the links compound.

**Where the divergence actually comes from, in rough order of magnitude:**

1. **Scenario choice.** SSP1-2.6 vs SSP5-8.5 produce very different late-century answers. If two
   vendors use different scenarios, nothing else matters.
2. **GCM choice / model spread.** CMIP6 has dozens of models. They disagree substantially on regional
   precipitation, and some (the "hot models") have equilibrium climate sensitivity well above the
   assessed likely range. Selecting or weighting models changes the answer.
3. **Downscaling method.** Global models run at 50 to 250 km. A building needs metre-scale. Methods:
   BCSD (bias-corrected spatial disaggregation), BCCA, LOCA, quantile delta mapping, and dynamical
   downscaling with a regional model. They preserve different properties. Bias correction that
   preserves the mean can distort the tail, and the tail is what risk cares about.
4. **Hazard translation.** Turning climate variables into flood depth requires a hydrological and
   hydraulic model, terrain, drainage, and defence assumptions. Two vendors with identical climate
   inputs will disagree here enormously, particularly on whether flood defences are represented.
5. **Vulnerability.** Damage functions are proprietary and vary widely.
6. **Normalization.** The final 0 to 100 score. Min-max scaling, percentile ranking, and z-scoring of
   the same underlying losses produce different orderings once you bucket into deciles or letter
   grades. This one is pure presentation and it still reshuffles rankings.

**Why negative correlation is possible and not just noise.** If vendor A scores relative-to-peers
(percentile within the portfolio) and vendor B scores absolute (expected annual loss in dollars),
then a portfolio concentrated in high-absolute-risk areas will produce systematically opposite
orderings. Different normalization conventions alone can flip the sign. That is a devastating finding
for anyone who bought one vendor and trusted it, and it is fully reproducible with open data.

---

## 3. Data

| Source | What | Access | Note |
|--------|------|--------|------|
| **FEMA National Risk Index** | County and tract level, 18 hazards, expected annual loss, social vulnerability, community resilience | Free CSV and API | Your historical, cat-style layer. Well documented. |
| **NASA NEX-GDDP-CMIP6** | Statistically downscaled CMIP6, 0.25 deg, ~35 models, multiple SSPs, daily | Free on AWS Open Data (S3) | Uses BCSD. One downscaling method, many GCMs. |
| **LOCA2** | Statistically downscaled CMIP6 over CONUS, 6 km, different method | Free, `loca.ucsd.edu` and AWS | Different downscaling, overlapping GCMs. **This pairing with NEX-GDDP is the experiment.** |
| **STAR-ESDM / MACA** | Further downscaling methods | Free | Optional third method for a stronger design |
| **Raw CMIP6** | Undownscaled GCM output | ESGF, or the Pangeo CMIP6 zarr catalog on GCS | For the "how much does downscaling change it at all" comparison |
| **First Street Foundation** | Property-level flood, fire, heat | Limited free lookup; bulk data is paid | Do not build a dependency on it. Cite as context only. |
| **Climate Impact Lab / Climate Toolbox** | Derived indices | Free | Useful cross-check |
| **NOAA Billion-Dollar Disasters** | Historical loss record | Free | Validation for the historical layer |

---

## 4. Method: the experiment design

This is the part that makes the project good. Use a factorial design.

1. **Fix the portfolio.** Define an identical set of locations. Use all ~3,100 US counties, or a
   synthetic 1,000-asset portfolio sampled to resemble a corporate footprint. Same locations for every
   model. This mirrors the "identical asset portfolio" framing in the short bullet.
2. **Historical layer agreement.** Correlate FEMA NRI hazard rankings against each other across
   hazards and against the NOAA billion-dollar disaster record. Expect high agreement, and note that
   the reason is shared calibration to the same loss history, not independent validation.
3. **Factorial run.** Compute a forward-looking heat and precipitation risk metric under:
   - **Vary GCM, hold downscaling fixed.** Use NEX-GDDP across N models. Spearman correlation of
     county rankings between every model pair. Report the distribution of pairwise correlations.
   - **Vary downscaling, hold GCM fixed.** Take the GCMs present in both NEX-GDDP (BCSD) and LOCA2,
     and compare rankings for the same GCM under the two methods.
   - **Vary scenario.** SSP2-4.5 vs SSP5-8.5 for the same GCM and downscaling.
   - **Vary normalization.** Take one single set of underlying values and score it by min-max,
     percentile rank, z-score, and log-then-min-max. Measure top-decile membership churn: what
     fraction of the "riskiest 10% of counties" changes identity purely from the scaling choice.
4. **Attribute the variance.** With a factorial design you can run an ANOVA-style decomposition:
   what share of ranking variance is attributable to GCM, to downscaling, to scenario, to
   normalization? This is the headline result and it is genuinely useful to the field. My expectation,
   which you should test rather than assume: scenario and GCM dominate at long horizons, downscaling
   dominates at short horizons and for precipitation-driven hazards, and normalization is small in
   correlation terms but large in decile-membership terms.
5. **Regrid carefully.** All comparisons must be on a common grid with area-weighted aggregation to
   county. Regridding artifacts can manufacture disagreement, so do the regridding once, up front,
   with `xesmf` conservative remapping, and document it.
6. **Rank correlation, not Pearson.** Scores use different scales. Spearman and Kendall tau are the
   right tools. Also report top-decile Jaccard overlap, which is what a user who acts on the top
   decile actually experiences.

---

## 5. Numbers to know cold

- CMIP6 has roughly **50+ modeling groups' models**; NEX-GDDP-CMIP6 downscales about **35**.
- Global climate models run at roughly **50 to 250 km**; NEX-GDDP is **0.25 degrees** (~25 km);
  LOCA2 is **6 km**; property-level risk needs metres. That is five orders of magnitude of gap being
  bridged by statistics.
- CMIP6 equilibrium climate sensitivity spans roughly **1.8 to 5.6 C**; IPCC AR6 assessed *likely*
  range is **2.5 to 4.0 C**, which is narrower than the raw model spread. The "hot model problem."
- FEMA NRI covers **18 hazards** at census tract and county level.
- Catastrophe model stochastic event sets typically contain **tens of thousands** of synthetic events.

---

## 6. Interview questions and how to answer

**"Why do catastrophe models agree and climate models disagree?"**
The one-sentence version: cat models are all calibrated against the same historical loss record, so
their agreement measures shared calibration rather than independent validation; climate models are
forward-looking with no calibration target, so every methodological choice propagates unchecked.

**"You said negative correlation. How is that possible?"**
Normalization convention. A relative score (percentile within portfolio) and an absolute score
(expected annual loss) can order the same portfolio in opposite directions when the portfolio is
concentrated. It is not that one model thinks a building is safe and another thinks it is dangerous;
it is that they are answering different questions and both are labelled "risk score."

**"Which choice matters most?"**
Give your measured answer from the ANOVA, then the caveat: it depends on horizon and hazard.
Scenario dominates late century. Downscaling dominates for precipitation-driven hazards at
near-term horizons because that is where local process representation matters and scenario spread has
not yet opened up.

**"What would you tell someone about to buy a climate risk vendor?"**
Buy two and compare, or at minimum demand the methodological disclosure: scenario, GCM ensemble,
downscaling method, whether flood defences are represented, and whether the score is relative or
absolute. If a vendor will not tell you those five things, the number is not usable for a decision
you have to defend.

**"You did not actually test the commercial vendors."**
Say it before they do, in the README and out loud. "Correct. I could not licence them outside the
bank, so I rebuilt the experiment with open models, which let me do something the vendor study could
not: hold one methodological choice fixed and vary another, and attribute the disagreement to a
cause rather than just measuring it."

---

## 7. Further reading

- Fiedler et al., "Business risk and the emergence of climate analytics," Nature Climate Change 2021.
  The best critical survey of the vendor landscape.
- Condon, "Climate Services: The Business of Physical Risk," Arizona State Law Journal 2023. Blunt on
  vendor opacity.
- Hausfather et al., "Climate simulations: recognize the hot model problem," Nature 2022.
- Lafferty and Sriver, "Downscaling and bias-correction contribute considerable uncertainty to local
  climate projections," npj Climate and Atmospheric Science 2023. This paper is essentially your
  project, done by climate scientists. Read it before you start and cite it.
- FEMA National Risk Index Technical Documentation.
