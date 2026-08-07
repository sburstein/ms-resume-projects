# Packet 06: National Utility Rate and Affordability Dataset

> **Résumé line:** "Built a national electric utility rate and affordability dataset from the ground
> up, joining residential bill data to census-tract income across roughly 150 investor-owned
> utilities and resolving the spatial mismatch between service territories, state, and census
> geography."

**Repo:** `utility-affordability` | **Est. 35 hours** | **Build second (Phase A). Highest leverage:
this bullet appears on 19 of your 20 résumés.**

---

## 1. The idea

Two datasets exist. Electricity bills are reported per utility. Income is reported per census tract.
Nobody can answer "what share of income do households in this utility's territory spend on power?"
because the two are reported on incompatible geographies, and no public crosswalk resolves it.

The project is the crosswalk, and then the metric it unlocks: **energy burden**, annual residential
electricity spend divided by household income.

The reason this bullet is on nearly every résumé is that it demonstrates the full stack: sourcing,
geospatial engineering, a defensible methodological choice, and a usable output. Build it first
among the data-product projects.

---

## 2. What you need to understand

**US utility structure.** Roughly 3,000 electric distribution utilities. About 170 to 190 are
investor-owned (IOUs), and they serve roughly 70% of customers. The rest are municipal utilities and
rural electric cooperatives. IOUs matter for finance because they have publicly traded parents, and
they matter for rates because they are rate-regulated by state public utility commissions through
formal rate cases.

**How a residential rate is actually set.** A utility files a rate case with its state PUC.
The commission approves a **revenue requirement**: allowed operating costs, plus depreciation, plus
taxes, plus a return on **rate base** (the depreciated value of used-and-useful capital) at an
authorized **return on equity**, typically 9 to 10.5% these days. That revenue requirement is then
allocated across customer classes and converted to rates. Two structural consequences worth knowing:

1. The utility earns on *capital deployed*, not energy sold. That is why utilities like building
   things, and why the "utility death spiral" story about rooftop solar is really about who pays for
   fixed network costs, not about lost energy margin.
2. Residential rates typically split into a fixed customer charge plus a volumetric charge, sometimes
   tiered or time-of-use. The average revenue per kWh you compute from EIA-861 is a blended
   effective rate, not any tariff a customer actually sees. Say so in the methodology.

**Energy burden, defined.** Annual household energy spend divided by annual household income. The
conventional affordability threshold is **6%**, with anything above that treated as high burden and
above 10% as severe. Note two caveats you should state: the 6% figure covers *all* home energy
(electric plus gas plus delivered fuels), and the national median household energy burden is roughly
3%, while low-income households frequently exceed 10%. If you compute electricity-only burden against
median tract income, you are computing something narrower than the standard definition. Define your
metric precisely and do not let it drift.

**The spatial mismatch, precisely.** Utility service territories were drawn by a century of franchise
grants, mergers, and geography. They cross county lines, split cities, and ignore state boundaries
(Pacific Power serves parts of Oregon, Washington, and California). Census tracts follow population
density and are redrawn each decade. Neither nests inside the other. So any join requires
apportionment, and apportionment requires a choice.

---

## 3. Data

| Source | What | Access | Gotcha |
|--------|------|--------|--------|
| **EIA-861** | Annual utility-level sales (MWh), revenue ($), and customer counts by class | Free ZIP of Excel files, `eia.gov/electricity/data/eia861` | Published with roughly a 9 to 12 month lag. Schema changes between years. Utility IDs are stable; names are not. |
| **EIA-861M** | Monthly version, smaller sample, faster | Free | The bridge for the nowcast in Packet 07 |
| **EIA API v2** | Programmatic access to retail sales and price series | Free with API key | Cleaner than the ZIP files for time series; less complete for the full form detail |
| **Census ACS 5-year** | Median household income, households, population, at tract level | Free API, `api.census.gov` | 5-year estimates are centered, so ACS 2019-2023 is not "2023". Margins of error at tract level can be large; carry them. |
| **TIGER/Line** | Census tract boundary shapefiles | Free | Boundaries change with each decennial redistricting. Match ACS vintage to the correct TIGER vintage or your join silently misaligns. |
| **HIFLD Electric Retail Service Territories** | Utility territory polygons | Free, `hifld-geoplatform.hub.arcgis.com` | Quality varies by state. Some territories are approximations. Some overlap. Municipal coverage is patchy. This is the weakest link in the chain and you must say so. |
| **EIA-861 Service Territory file** | County-level list of counties served by each utility | Free, part of EIA-861 | A coarse but *authoritative* alternative to HIFLD polygons. Use it as a cross-check: if HIFLD says a utility serves a county that EIA-861 does not list, investigate. |
| **Census LODES / block-level population** | For population weighting | Free | Needed for the population-weighted apportionment upgrade |

---

## 4. Method, in steps

1. **Ingest EIA-861 and normalize.** Build a tidy table: `utility_id`, `year`, `state`,
   `ownership_type`, `residential_sales_mwh`, `residential_revenue_000usd`,
   `residential_customers`. Derive `avg_rate_cents_per_kwh = revenue / sales` and
   `avg_annual_bill = revenue / customers`. Handle the multi-state utilities: EIA reports by
   utility-state pair, so a single utility appears in several rows and you must decide whether your
   unit of analysis is the utility or the utility-state pair. Pick utility-state. It is the honest
   unit because rates are set by state commissions.
2. **Filter to IOUs** using the ownership field. Expect roughly 150 to 190 depending on year and
   whether you count holding-company subsidiaries separately. Note the count you get and why.
3. **Pull ACS tract income** for every tract in every state where any IOU operates. Store median
   household income, household count, and population, with margins of error.
4. **Load geometries.** Tract polygons from TIGER, territory polygons from HIFLD. Reproject both to
   an equal-area CRS (US National Atlas Equal Area, EPSG 2163, or Albers EPSG 5070). Doing spatial
   overlay in WGS84 degrees produces wrong areas. This is a common and quietly fatal error.
5. **Overlay.** `geopandas.overlay(tracts, territories, how='intersection')`. For every
   tract-territory pair you now have the intersected geometry and its area.
6. **Apportion, version 1: area weighted.** Weight each tract's contribution to a territory by the
   share of tract area inside it. Simple, fast, and wrong in cities, because population is not
   uniform across a tract.
7. **Apportion, version 2: population weighted.** Use census blocks (which nest inside tracts) to
   compute what share of a tract's *population* falls inside each territory, then weight by that.
   This is the defensible version.
8. **Report the difference between them.** Compute burden both ways and show the distribution of the
   difference. This comparison is the intellectual core of the project. It converts "I did a spatial
   join" into "I quantified how much the apportionment choice moves the answer," which is a
   completely different level of claim.
9. **Compute energy burden** per utility-state: territory-weighted median household income, and
   average annual residential bill from EIA-861. Flag territories where coverage is incomplete
   rather than silently reporting a partial number.
10. **Validate.** Cross-check a sample of 10 utilities against published state PUC average residential
    bill figures and against EIA's own state-level average price table. Where you disagree, explain
    why. Cross-check territory assignment against the EIA-861 county list.
11. **Publish.** A tidy CSV, a methodology note, a choropleth, and a data dictionary.

---

## 5. Numbers to know cold

- Roughly **3,000** US electric distribution utilities; about **170 to 190** IOUs serving about
  **70%** of customers.
- US average residential rate: roughly **16 to 17 cents/kWh** nationally, ranging from roughly
  11 cents (Idaho, Utah, North Dakota) to over 40 cents (Hawaii) and high 20s to 30s (California,
  Massachusetts, Connecticut).
- Average US household consumption: roughly **10,500 kWh/year**, about 870 kWh/month, though Texas
  and the South run far higher and the Pacific Northwest lower.
- Affordability threshold convention: **6%** of income on total home energy; **3%** is roughly the
  national median; low-income households routinely exceed **10%**.
- Census tract: roughly **4,000 people**, about 84,000 tracts nationally.
- Authorized utility ROE in recent rate cases: roughly **9.5 to 10.5%**.
- EIA-861 publication lag: roughly **9 to 12 months**.

---

## 6. Interview questions and how to answer

**"Walk me through the spatial join."**
Do not describe a function call. Describe the *choice*. "Tracts and territories do not nest, so every
join is an apportionment and every apportionment is an assumption. I built it area-weighted first,
then population-weighted using census blocks, and reported how much the answer moved between them.
In dense urban territories the difference was material, which is exactly where the affordability
question matters most."

**"Why does this dataset not already exist?"**
Because the pieces are owned by three different agencies on three different geographies with three
different update cadences, and because the service-territory geometry is the weakest public layer.
Assembling it is unglamorous and the failure modes are silent, so most people use state averages
instead and accept the loss of resolution.

**"What is the weakest part of your dataset?"**
The HIFLD service territory polygons. Quality varies by state, some are approximations, and some
overlap. I cross-checked against the EIA-861 county service list, which is authoritative but coarse,
and I flagged utilities where the two disagreed rather than picking one silently.

**"Average revenue per kWh is not a rate. Does that bother you?"**
It should bother anyone. It is a blended effective rate across tiers, time-of-use periods, and fixed
charges, and it will not match any tariff line a customer sees. It is the right metric for
*affordability*, which is about total spend, and the wrong metric for *tariff analysis*. I documented
that distinction in the methodology note.

---

## 7. Further reading

- ACEEE, "How High Are Household Energy Burdens?" 2020. The standard reference for the metric.
- Lawrence Berkeley National Lab, "Low-Income Energy Affordability Data (LEAD) Tool" methodology,
  from DOE. Closest public analogue to what you built; read how they apportion.
- EIA-861 instructions and form. Boring, and the only way to get the field definitions right.
- Census Bureau, "A Compass for Understanding and Using ACS Data," on margins of error at tract level.
