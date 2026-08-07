# Packet 14: Data Center Siting and Resource Risk

> **Résumé line:** "Analysed roughly 11,000 global data center locations against water stress and
> utility service-area layers to quantify siting and resource risk."

**Repo:** `datacenter-siting` | **Est. 22 hours** | **Tier A. On 4 résumés, and the single most
topical project in the plan right now. Small, fast, and interviewers will want to talk about it.**

---

## 1. The idea

Data center load growth is the dominant story in US power markets in 2026. Every utility IRP, every
interconnection queue, every capacity auction is being reshaped by it. A dataset that puts data
center locations against utility service territories, water stress, and grid carbon intensity answers
questions that people are actively paying for right now.

It is also a natural bridge project: it uses the utility territory work from Packet 06, and it lands
squarely on the power-market roles you are applying to.

---

## 2. What you need to understand

**Why data centers strain the grid differently from other load.**
- **Density.** A hyperscale campus can draw 100 to 1000+ MW at a single point of interconnection.
  That is a mid-sized city arriving at one substation.
- **Flat load shape.** Near-constant 24/7 draw with a very high load factor (often 80 to 90%+),
  versus roughly 50 to 60% for a typical utility system. High load factor is good for utility
  economics but it means the load does not shrink when the system is tight.
- **Speed.** A campus is built in 18 to 36 months. Transmission takes 7 to 12 years. That mismatch is
  the entire interconnection crisis in one sentence.
- **Siting mobility, up to a point.** Data centers can locate where power is available, which is why
  Northern Virginia, central Ohio, Georgia, Texas, and Phoenix are the clusters. But AI training
  clusters are less latency-constrained than inference and colocation, so the siting logic differs by
  workload. Mentioning that distinction signals current knowledge.

**Water.** Cooling is the water story and it is more nuanced than the headlines.
- **Direct water use** comes mostly from evaporative cooling. A large facility can consume on the
  order of millions of gallons per day at peak.
- **WUE (water usage effectiveness)**, litres per kWh of IT load, is the standard metric. Ranges from
  near zero for closed-loop and air-cooled designs to well over 1 L/kWh for evaporative.
- **Indirect water** is the water consumed generating the electricity, and for a thermal-heavy grid
  it can exceed direct on-site use. A facility that switches to air cooling to save water but draws
  from a water-intensive grid may not have helped.
- The tradeoff is real: evaporative cooling saves energy and costs water; air cooling saves water and
  costs energy. Say this rather than treating water use as a straightforward negative.
- **PUE (power usage effectiveness)**, total facility power over IT power, is the energy analogue.
  Hyperscale runs roughly 1.1 to 1.2; older enterprise facilities 1.5 to 2.0.

**Liquid cooling changes the picture.** AI racks at 40 to 130 kW cannot be air cooled effectively, so
direct-to-chip and immersion cooling are becoming standard. Closed-loop liquid systems typically
reduce water consumption relative to evaporative while improving thermal efficiency. Any water-risk
analysis of the current build-out has to account for the design shift, not just the location.

---

## 3. Data

| Source | What | Access | Gotcha |
|--------|------|--------|--------|
| **Data Center Map / Baxtel / Cloudscene** | Facility locations, operators, sometimes capacity | Web, scrapable in part; some paid | Coverage is best for colocation, worst for owner-operated hyperscale. Check terms of service before scraping. |
| **OpenStreetMap** | `telecom=data_center` and `building=data_center` tags | Free, Overpass API | Incomplete and inconsistently tagged, but free and legally clean. Start here. |
| **EPA FRS / state permits** | Air permits for backup generators reveal facility locations and often capacity | Free | Underused and surprisingly effective: nearly every data center permits diesel backup |
| **Utility IRPs and interconnection queues** | Announced large-load requests by location | Free, PDFs | The forward-looking layer. Dominion, AEP Ohio, Georgia Power, and Oncor filings are the interesting ones. |
| **WRI Aqueduct 4.0** | Baseline water stress, depletion, drought risk, future projections, at basin level | Free, downloadable | The standard water-stress layer. Understand what "baseline water stress" measures: withdrawal over available renewable supply. |
| **HIFLD electric retail service territories** | Utility territory polygons | Free | From Packet 06 |
| **EPA eGRID** | Grid carbon and water intensity by subregion | Free | For the indirect footprint |
| **EIA-930** | Hourly BA-level demand, for context on load growth | Free | Shows the growth signal directly in some BAs |
| **NREL / EIA-860** | Nearby generation capacity | Free | For a "can the local grid serve this" screen |

---

## 4. Method

1. **Assemble the facility layer.** Union OSM, EPA FRS backup-generator permits, and any
   permissibly-obtained commercial listing. Deduplicate spatially (facilities within ~200 m are
   likely the same campus). Record a source and confidence per record. State your coverage honestly
   rather than claiming a global count you cannot verify.
2. **Estimate capacity where missing.** Backup generator permitted capacity is a decent proxy for IT
   load. Building footprint area from OSM plus a power-density assumption is a cruder second option.
   Flag estimated vs stated.
3. **Spatial joins.** Facility to utility service territory, to Aqueduct basin, to eGRID subregion,
   to county. Equal-area projection, same discipline as Packet 06.
4. **Compute the risk stack per facility:** baseline water stress category, projected 2030 and 2050
   water stress under a scenario, grid carbon intensity, grid water intensity, and a local grid
   headroom proxy (BA demand growth rate versus generation additions).
5. **Aggregate by operator and by utility.** Two rollups, two audiences. By operator: which
   hyperscaler has the most water-stressed footprint. By utility: which utility faces the largest
   concentration of new large load relative to its current peak. The second is the one that gets a
   power-markets interviewer's attention, because it is a load-growth forecast in disguise.
6. **The headline chart.** Data center capacity by utility territory as a percentage of that
   utility's current peak load, coloured by water stress. That one figure carries the whole project.

---

## 5. Numbers to know cold

- US data centers consumed roughly **4 to 5%** of US electricity as of 2024 and are commonly projected
  toward **7 to 12%** by 2030, with wide uncertainty. Quote the range and the uncertainty, not a
  point.
- Northern Virginia (Loudoun County, "Data Center Alley") is the largest cluster globally, with
  multiple GW in operation and more in queue.
- Hyperscale **PUE** roughly **1.1 to 1.2**; older enterprise **1.5 to 2.0**.
- **WUE** ranges from near **0** (closed-loop, air-cooled) to over **1 L/kWh** (evaporative).
- AI rack densities have moved from roughly **10 to 15 kW** to **40 to 130+ kW**, forcing liquid
  cooling.
- Data center load factor roughly **80 to 90%+** versus **50 to 60%** for a typical utility system.
- Transmission build takes **7 to 12 years**; a data center campus takes **18 to 36 months**.

---

## 6. Interview questions and how to answer

**"Is data center water use actually a problem?"**
Resist the headline. "It depends on where and on the cooling design. Evaporative cooling in a
water-stressed basin is a genuine local constraint. Closed-loop liquid cooling in a
water-abundant basin is close to a non-issue directly, but it shifts the burden to electricity, and
if the grid is thermal-heavy the indirect water consumption can exceed the direct. The right analysis
is direct plus indirect, at basin level, conditioned on the cooling technology. That is why I built
the layers separately rather than reporting one number."

**"Which utilities are most exposed to load growth?"**
Answer with the metric, not a list: new large-load requests as a share of current peak. Then name the
obvious cases (Dominion in Virginia, AEP Ohio, Georgia Power) and the reason: they sit where fibre,
land, tax treatment, and historically cheap power coincided.

**"How would this inform a battery or a generation investment?"**
Load growth concentrated at specific nodes, arriving faster than transmission can be built, is the
textbook case for local capacity value. It raises scarcity pricing and capacity prices in constrained
zones and makes co-located generation and storage economic. The nodal detail is what matters, which
is why the analysis is at territory level rather than national.

---

## 7. Further reading

- LBNL, "2024 United States Data Center Energy Usage Report." The authoritative US estimate; read the
  uncertainty discussion, not just the headline.
- IEA, "Electricity 2024" and subsequent editions, for the global data center projections.
- WRI Aqueduct 4.0 technical note, for what baseline water stress actually measures.
- Mytton, "Data centre water consumption," npj Clean Water 2021. The careful treatment of direct vs
  indirect water.
- Dominion Energy Virginia and AEP Ohio recent IRP filings, for how utilities are actually modeling
  this.
