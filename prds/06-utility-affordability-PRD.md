# PRD 06: National Utility Rate and Affordability Dataset

| | |
|---|---|
| **Repo** | `utility-affordability` |
| **Bullets covered** | `dataset` (19 of 20 résumés) |
| **Packet** | [`06-utility-affordability.md`](../knowledge-packets/06-utility-affordability.md) |
| **Est.** | 35 human-hours, 9 to 14 agent session hours |
| **Priority** | **1 of 17. Build this first.** |
| **Model** | Sonnet for M0 to M2 and M6. Opus for M3 (apportionment) and M5 (validation). |
| **Prereqs** | Read [`00-CONVENTIONS.md`](00-CONVENTIONS.md) |

---

## 1. Objective

Produce a public, reproducible dataset of **residential electricity energy burden** for every
investor-owned utility in the United States, by resolving the spatial mismatch between utility
service territories and census geography, and quantify how much the answer depends on the
apportionment method chosen.

**One-sentence success test:** a stranger can `git clone`, run `make all`, and get a per-utility
energy burden table plus a figure showing how much area-weighted and population-weighted
apportionment disagree.

---

## 2. Scope

**In scope**
- All US investor-owned utilities, at the utility-state pair level.
- Residential class only.
- Latest complete EIA-861 year, plus a 5-year history for trend.
- Two apportionment methods, compared.

**Out of scope**
- Municipal utilities and cooperatives (note the coverage gap; do not model them).
- Commercial and industrial classes.
- Gas, delivered fuels, or total home energy. This is electricity-only burden and the README must
  say so explicitly, since the conventional 6% threshold covers all home energy.
- Tariff-level modelling. Average revenue per kWh is a blended effective rate, not a tariff.

---

## 3. Data contracts

Declare all of these in `config/sources.yaml` before writing any fetch code.

| id | Source | Endpoint / file | Notes |
|---|---|---|---|
| `eia_861` | EIA Form 861 annual | `eia.gov/electricity/data/eia861/` ZIP per year | Sales, revenue, customers by class. Schema differs by vintage; write a per-vintage column mapper. |
| `eia_861_territory` | EIA-861 Service Territory file | in the same ZIP | Authoritative utility-to-county list. Cross-check layer. |
| `eia_861m` | EIA-861M monthly | `eia.gov/electricity/data/eia861m/` | Needed by PRD 07, ingest now. |
| `census_acs` | ACS 5-year, tract level | `api.census.gov/data/{year}/acs/acs5` | Variables: `B19013_001E` median household income, `B11001_001E` households, `B01003_001E` population, plus MOE fields. Needs a free API key. |
| `tiger_tracts` | TIGER/Line tract shapefiles | `www2.census.gov/geo/tiger/TIGER{year}/TRACT/` | **Vintage must match the ACS vintage.** |
| `tiger_blocks` | TIGER/Line block shapefiles + block population | same host | For population weighting. Large: fetch per state. |
| `hifld_territories` | Electric Retail Service Territories | HIFLD ArcGIS hub, GeoJSON or shapefile | Weakest layer in the chain. Quality varies by state. |

**Licence check:** all of the above are US Government works or public domain. HIFLD terms permit
redistribution of derived products; note the attribution requirement in `README.md`.

---

## 4. Milestones

### M0. Scaffold
Repo per conventions section 3. `pyproject.toml`, Makefile targets `install`, `fetch`, `build`,
`figures`, `test`, `all`. `sources.yaml` fully populated. `.env.example` with `CENSUS_API_KEY`.
**Accept:** `make install && make test` passes with one trivial test.

### M1. EIA-861 ingest
`ingest/eia861.py` downloading, caching, and parsing the residential sales table for the target year
plus 4 prior years. Per-vintage column mapping in `config/eia861_schema.yaml`.

Output `data/interim/utilities.parquet` with pandera schema:

| column | type | constraint |
|---|---|---|
| `utility_id` | int | not null |
| `utility_name` | str | not null |
| `state` | str | 2-char, valid USPS |
| `year` | int | 2015 to current |
| `ownership_type` | category | EIA codes |
| `res_sales_mwh` | float | >= 0 |
| `res_revenue_kusd` | float | >= 0 |
| `res_customers` | int | > 0 |
| `avg_rate_cents_kwh` | float | 0 < x < 100, derived |
| `avg_annual_bill_usd` | float | 0 < x < 10000, derived |

**Accept:** row count within 5% of the prior year's; IOU count between 140 and 200; national
weighted-average residential rate within 1 cent of EIA's own published national figure. Assert all
three in tests. Log and report the IOU count you actually get.

### M2. Census ingest
`ingest/census.py` pulling tract-level ACS for every state containing an IOU, with margins of error
retained. `ingest/tiger.py` fetching tract and block geometries.
**Accept:** tract count within 2% of the published national total for that ACS vintage; every tract
in the income table has a matching geometry; report the count of tracts with null or suppressed
median income (this is common in low-population tracts and must be handled, not dropped silently).

### M3. Spatial apportionment  **← Opus**
`transform/apportion.py`.

1. Reproject everything to EPSG:5070 (CONUS Albers). Alaska and Hawaii handled separately with their
   own projections, or excluded with an explicit note.
2. `overlay(tracts, territories, how="intersection")`.
3. `area_weights()`: tract-to-territory weight = intersected area / total tract area.
4. `population_weights()`: allocate via census blocks nested in tracts; weight = population of blocks
   inside the territory / total tract population.
5. Both return a long table `(utility_id, state, tract_geoid, weight, method)` where weights per
   tract sum to <= 1.0 (less than 1 where a tract is partly outside all IOU territories).

**Accept:**
- Weights per (tract, method) sum to <= 1.0 + 1e-6 for every tract. Assert.
- Total apportioned population across all territories is within 15% of the sum of IOU customer counts
  times average household size. Large deviation means the territory layer is wrong; investigate
  before proceeding.
- Overlay of a hand-checked known case (e.g. a tract split by the PG&E boundary) produces a sensible
  weight. Include as a fixture test.

### M4. Energy burden
`model/burden.py`. For each utility-state and each method: territory-weighted median household
income (weight the tract medians; document that a weighted median of medians is an approximation and
state the bias direction), average annual residential bill from M1, and
`burden = bill / weighted_income`.

Emit `outputs/tables/energy_burden.csv` with both methods and their difference, plus a coverage
column (share of the utility's customers represented by matched tracts).
**Accept:** no burden value outside 0.1% to 25%; any outside that range is flagged with the reason,
not dropped. National customer-weighted mean burden lands in a plausible 1.5% to 4% band.

### M5. Validation  **← Opus**
`validate/` comparing against three independent references:
1. EIA's own published state-level average residential price. Per-state weighted average of your
   utility figures should reconcile within 0.5 cents.
2. HIFLD territory assignment vs the EIA-861 county service list. Report disagreement rate per state.
3. Ten hand-picked utilities against their state PUC published average bill.

Write results to `FINDINGS.md`, including the disagreement rate, not just the successes.
**Accept:** all three comparisons run and are documented. Reconciliation failures are explained, not
hidden.

### M6. Figures, docs, release
1. **Choropleth** of energy burden by utility territory.
2. **Scatter** of area-weighted vs population-weighted burden, with the 45-degree line, coloured by
   territory urban share. This is the headline figure.
3. **Bar chart** of the 25 highest-burden IOUs with uncertainty bars from the ACS margins of error.

Plus `README.md`, `METHODOLOGY.md`, `FINDINGS.md`, and `outputs/releases/YYYY-MM/`.
**Accept:** conventions section 5, all seven items.

---

## 5. Golden-number regression tests

Store expected values in `tests/golden.json` with a tolerance. Fail the build if exceeded without a
CHANGELOG entry.
- National customer-weighted mean burden, tolerance 0.1pp.
- IOU count, tolerance 0.
- Median absolute difference between the two apportionment methods, tolerance 0.05pp.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| HIFLD territory polygons wrong or overlapping | Apportioned population far off customer counts; weights summing above 1 | Cross-check against EIA-861 county list. Report per-state confidence. Do not silently pick one. |
| Doing overlay in WGS84 degrees | Areas look wrong, weights skew by latitude | Assert CRS is projected before any area computation. Unit test this. |
| ACS/TIGER vintage mismatch | Unmatched GEOIDs | Assert vintages match in config; fail loudly. |
| Block-level fetch is huge | Slow, memory pressure | Fetch per state, process per state, concatenate. Never load national blocks at once. |
| Weighted median of medians | Silent bias | Document the bias direction in METHODOLOGY. Optionally compute an income-distribution-based alternative for a sample of territories and report the gap. |
| Suppressed tract income | Dropped rows change the answer | Count them, report them, test sensitivity by imputing county median and showing the delta. |

---

## 7. Downstream

PRD 07 (nowcast) and PRD 08 (parent crosswalk) build in this same repo and depend on M1 and M4.
Do not start them until this PRD's definition of done is met.
