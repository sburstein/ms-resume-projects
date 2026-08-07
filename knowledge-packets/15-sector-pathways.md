# Packet 15: Sector Climate Pathways and Target Setting

> **Résumé line:** "Conducted quantitative analysis supporting net-zero financed-emissions targets,
> including sector-level climate pathways and target setting under SBTi, PCAF, and the GHG Protocol."

**Repo:** `sector-pathways` | **Est. 18 hours** | **Tier A. On 4 résumés. You already have
`~/projects/ngfs-scenario-explorer`, which is most of this. Finish and document it rather than
starting over.**

---

## 1. The idea

A net-zero target is not a number, it is a trajectory. Setting one requires picking a scenario,
extracting the sector-level decarbonization path implied by it, and deciding how to allocate the
remaining carbon budget across a portfolio of companies with different starting intensities.

Build the tool that does that: scenario in, sector pathway out, portfolio alignment and required
reduction rate computed against it.

---

## 2. What you need to understand

**The scenario families, and why they are not interchangeable.**

| Family | Source | Purpose | Note |
|--------|--------|---------|------|
| **IPCC AR6 / SSP** | IPCC | Physical climate forcing | SSP1-1.9, SSP1-2.6, SSP2-4.5, SSP3-7.0, SSP5-8.5. The physical input to everything else. |
| **NGFS** | Network for Greening the Financial System | Financial risk assessment by central banks and supervisors | Orderly, Disorderly, Hot House World, Too Little Too Late. Includes transition *and* physical risk, and macro-financial variables. The one supervisors expect. |
| **IEA** | International Energy Agency | Energy system | STEPS (stated policies), APS (announced pledges), NZE (net zero by 2050). NZE is the normative one used for target setting. |
| **SBTi sectoral pathways** | Science Based Targets initiative | Corporate target validation | Built on IEA and IPCC underneath, expressed as company-level intensity trajectories |

**The two allocation approaches, and this is the core concept.**

- **SDA (Sectoral Decarbonization Approach).** Used for homogeneous, emissions-intensive sectors
  where a physical intensity metric is meaningful: power (tCO2/MWh), cement (tCO2/tonne), steel,
  aviation (gCO2/RTK). Every company in the sector *converges* to the same intensity by the target
  year. Consequence: a company starting dirtier must decarbonize faster in percentage terms. This is
  the fair-sharing logic and it is what makes SDA different from a flat cut.
- **Absolute contraction.** Used where a physical intensity metric does not exist or is not
  comparable. Every company reduces absolute emissions at the same annual rate, currently around
  **4.2% per year** linear for 1.5 C aligned targets. Simple, and it penalizes growing companies.

**Convergence vs contraction is the single most likely conceptual question.** Have the one-line
distinction ready: convergence equalizes intensity, contraction equalizes rate of reduction.

**Carbon budgets.** The remaining budget for a 50% chance of limiting warming to 1.5 C was assessed
at roughly **500 GtCO2 from the start of 2020** in AR6, and subsequent updates have revised it down
substantially given continued emissions of roughly 40 GtCO2 per year. Quote it as a range with a
probability attached, never as a single number, because the probability qualifier is the whole point.

**Alignment metrics for a portfolio.**
- **Implied Temperature Rise (ITR).** Translates a portfolio's aggregate trajectory into a warming
  outcome. Intuitive and widely criticized, because the mapping from a corporate emissions path to a
  global temperature depends on assumptions about everyone else.
- **Portfolio warming potential**, **alignment gap** (distance from the benchmark pathway), and
  **share of AUM covered by validated SBTi targets** are the more defensible alternatives.
- Report an alignment gap rather than a single temperature. The gap is measurable; the temperature
  is an extrapolation.

---

## 3. Data

| Source | What | Access |
|--------|------|--------|
| **NGFS Scenario Explorer** (IIASA) | Full scenario database, all variables, all models (REMIND, GCAM, MESSAGE) | Free with registration, API available |
| **IIASA Scenario Explorer / `pyam`** | Python library purpose-built for scenario data in IAMC format | `pip install pyam-iamc` |
| **IEA NZE data** | Sector trajectories | Some free, some paid; key figures published |
| **SBTi** | Target validation database, sector guidance, SDA tool | Free downloads |
| **IPCC AR6 WGIII scenario database** | The underlying scenario ensemble | Free, IIASA hosted |
| **Company emissions** | From Packet 11 | |

**Use `pyam`.** Scenario data comes in IAMC long format (model, scenario, region, variable, unit,
year, value) and `pyam` handles filtering, aggregation, and unit conversion natively. Writing your
own parser is a waste of a day.

---

## 4. Method

1. **Ingest scenarios** via `pyam` from the NGFS explorer. Filter to the variables you need: sector
   emissions, sector activity (generation, production), and derived intensity.
2. **Derive sector intensity pathways.** For power: `Emissions|CO2|Energy|Supply|Electricity` divided
   by `Secondary Energy|Electricity`, converted to tCO2/MWh, by region and year. Do this for two or
   three scenarios so the user can compare.
3. **Implement SDA.** Given a company's current intensity, the sector's current intensity, the
   sector's target-year intensity, and the company's activity growth assumption, compute the required
   intensity path and the implied absolute emissions path. The SBTi publishes the exact formula; use
   it rather than approximating.
4. **Implement absolute contraction** at the linear annual rate for comparison.
5. **Portfolio alignment.** For each holding, compute the gap between its actual trajectory (from
   history and stated targets) and its required pathway. Aggregate weighted by financed emissions
   from Packet 11.
6. **Scenario sensitivity.** Run the same portfolio under NGFS Orderly, Disorderly, and Hot House
   World and show how the alignment verdict changes. The instability of the verdict across scenarios
   is the honest finding, and it is more interesting than any single alignment number.
7. **Chart:** sector pathway with the portfolio's weighted trajectory overlaid, plus a company
   scatter of current intensity versus required reduction rate. That scatter immediately shows who has
   the hardest job, which is the practical output.

---

## 5. Numbers to know cold

- Linear absolute contraction rate for 1.5 C alignment: **~4.2% per year**.
- Remaining carbon budget for 1.5 C at 50% probability: roughly **500 GtCO2 from 2020** per AR6,
  revised down since; annual global emissions roughly **40 GtCO2**.
- NGFS scenario families: **Orderly, Disorderly, Hot House World, Too Little Too Late**.
- IEA scenarios: **STEPS, APS, NZE**.
- Power sector intensity: global average roughly **0.4 to 0.45 tCO2/MWh** today; NZE requires
  advanced economies to reach near zero by around **2035** and globally by around **2040**.
- SBTi requires near-term targets covering **5 to 10 years** and, for net zero, at least **90%**
  absolute reduction with residual neutralization.

---

## 6. Interview questions and how to answer

**"What is the difference between the SDA and absolute contraction?"**
Convergence versus contraction, in one line, then the consequence: under SDA a company that starts
dirtier must decarbonize faster, which is why intensity-based targets are used in homogeneous
industrial sectors and absolute targets elsewhere.

**"Which scenario would you use and why?"**
Depends on the question. NGFS for financial risk assessment and supervisory work, because it carries
macro-financial variables and it is what regulators expect. IEA NZE for corporate target setting,
because it is normative and sector-detailed. Never one scenario alone for a decision, because the
alignment verdict flips across scenarios more often than people admit.

**"What is wrong with Implied Temperature Rise?"**
It converts a company-level trajectory into a global outcome, which requires an assumption about
everybody else's behavior. Two providers with the same underlying data can produce different ITRs
from different attribution assumptions. I would report an alignment gap against a named pathway,
which is measurable and auditable, rather than a temperature that implies more precision than exists.

---

## 7. Further reading

- SBTi, "Sectoral Decarbonization Approach" methodology paper, and the current Corporate Net-Zero
  Standard.
- NGFS Scenarios technical documentation, latest vintage.
- IEA, "Net Zero Roadmap: A Global Pathway to Keep the 1.5 C Goal in Reach."
- IPCC AR6 WGIII, Chapter 3, on scenario families and carbon budgets.
- `pyam` documentation and tutorials.
