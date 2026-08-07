# PRD 14: Data Center Siting and Resource Risk

| | |
|---|---|
| **Repo** | `datacenter-siting` |
| **Bullets covered** | `datacenter` (4 of 20 résumés) |
| **Packet** | [`14-datacenter-siting-risk.md`](../knowledge-packets/14-datacenter-siting-risk.md) |
| **Est.** | 22 human-hours, 5 to 8 agent session hours |
| **Priority** | 9 of 17. Most topical project in the plan; disproportionate interview value per hour. |
| **Model** | Sonnet throughout. Opus for M4 (capacity estimation methodology). |
| **Prereqs** | Read `00-CONVENTIONS.md`. Reuses PRD 06's HIFLD territory layer if available. |

---

## 1. Objective

Assemble a US data center facility layer from public sources, join it to utility service territories,
water stress basins, and grid carbon intensity, and produce the metric that matters to power markets:
**announced and operating data center load as a share of each utility's current peak demand.**

**Success test:** one figure ranking utility territories by data center load as a percentage of peak,
coloured by basin water stress, built entirely from sources whose licences permit redistribution.

---

## 2. Scope

**In:** CONUS facilities. Public sources only. Direct and indirect water. Utility-territory and
operator rollups. Announced large-load requests from utility filings where machine-readable.

**Out:** Global coverage (note it, do not attempt it). Any scraped source whose terms of service
prohibit it. Facility-level power measurement, which is not public. Latency or network modelling.

**Honesty constraint:** the résumé bullet says roughly 11,000 global locations. That was a licensed
dataset. State in the README that the public rebuild is CONUS-only with a measured, lower coverage,
and report the count you actually achieve. Do not attempt to match the licensed count.

---

## 3. Data contracts

| id | Source | Access | Licence check |
|---|---|---|---|
| `osm_datacenters` | OSM `telecom=data_center`, `building=data_center` | Overpass API | ODbL, redistribution permitted with attribution. **Start here.** |
| `epa_frs` | EPA Facility Registry Service | Free bulk and API | US Gov work. Backup-generator air permits reveal most facilities. |
| `epa_permits` | State and federal air permits for emergency generators | EPA ECHO / state portals | US Gov work |
| `wri_aqueduct` | Aqueduct 4.0 baseline water stress, depletion, future projections | Free download | CC BY 4.0, attribution required |
| `hifld_territories` | Electric retail service territories | HIFLD | Same layer as PRD 06 |
| `egrid` | eGRID subregion emission and water intensity factors | EPA | US Gov work |
| `eia930` | Hourly BA demand, for peak load denominators | EIA API | US Gov work |
| `utility_irps` | Announced large-load requests from IRP and rate filings | State PUC dockets, PDFs | Public record. Manual extraction; small n, high value. |

**Explicitly excluded:** Data Center Map, Baxtel, Cloudscene, and similar commercial directories.
Their terms generally prohibit scraping and redistribution. **The agent must not scrape them.** If
coverage from permitted sources proves inadequate, stop and report rather than reaching for a
prohibited source.

---

## 4. Milestones

### M0. Scaffold and licence audit
Repo per conventions. `sources.yaml` with a `licence` and `redistribution_permitted` field on every
entry. A test asserts every source used has `redistribution_permitted: true`.
**Accept:** licence audit test passes. Any source that fails is removed, not worked around.

### M1. OSM harvest
Overpass query for data center tags across CONUS, with a bounding-box tiling strategy to avoid
timeouts. Extract geometry, name, operator tag, and building footprint area.
**Accept:** at least 1,500 candidate facilities. Report the count. Known clusters appear: Loudoun
County VA, central Ohio, Atlanta metro, Dallas and San Antonio, Phoenix, Santa Clara, Umatilla OR,
and northern Nevada. If Loudoun County is not the densest cluster, the query is wrong.

### M2. EPA permit harvest
Query FRS and ECHO for facilities whose NAICS is 518210 (data processing and hosting) or whose
permit description indicates emergency generators at a data processing facility.
**Accept:** permitted generator capacity captured where available. Report match count and the share
of facilities with a capacity figure.

### M3. Entity resolution
Union the layers. Deduplicate spatially: facilities within 250 m with compatible names are one
campus. Keep a `cluster_id` and every contributing source record. Never delete a source record.
**Accept:** a test on a hand-verified set of 20 known campuses confirms correct clustering. Report
the dedupe reduction ratio.

### M4. Capacity estimation  **← Opus**
Three-tier ladder, with the tier recorded per facility:
1. **Stated** capacity from a permit or filing.
2. **Generator-derived**: permitted backup generator capacity as an IT-load proxy, with a documented
   conversion assumption and its justification.
3. **Footprint-derived**: OSM building area times a power-density assumption (W per square foot),
   with a wide stated uncertainty.
**Accept:** every facility has a capacity estimate and a tier. The README reports what share of total
estimated capacity comes from each tier. **Tier 3 estimates are reported with an explicit
uncertainty range, never as point values.**

### M5. Spatial joins
Facility to utility territory, to Aqueduct basin, to eGRID subregion, to county. Equal-area
projection throughout (EPSG:5070), asserted before any area or distance computation.
**Accept:** a test asserts the CRS is projected at every spatial operation. Every facility has all
four joins or an explicit reason for a null.

### M6. Risk stack
Per facility: baseline water stress category, projected 2030 and 2050 water stress under a stated
scenario, grid carbon intensity, grid water intensity, and estimated indirect water consumption
(IT load x PUE x grid water intensity).
**Accept:** direct and indirect water are reported **separately**, and the README states the cooling
technology caveat: evaporative cooling trades energy for water, closed-loop liquid trades water for
energy, so a facility's water profile depends on design as much as location.

### M7. Rollups and the headline figure
1. **By utility territory:** total estimated data center capacity as a percentage of that BA's
   recent peak demand from EIA-930. Add announced-but-not-built capacity from M0's IRP extraction
   as a separate stacked component.
2. **By operator:** where the operator tag is reliable, footprint-weighted water stress exposure.
3. **Headline figure:** horizontal bar chart, top 25 utility territories by data center capacity as a
   share of peak load, bars coloured by area-weighted basin water stress, with announced capacity
   shown as a lighter stacked segment.
**Accept:** the figure is legible and the ranking is defensible. Dominion Virginia should rank at or
near the top; if it does not, investigate before publishing.

---

## 5. Golden-number regression tests

- Facility count after dedupe, tolerance 5%.
- Loudoun County facility count, tolerance 10%.
- Share of total capacity from tier 1 stated sources, tolerance 3pp.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Scraping a prohibited source** | | M0 licence audit test. Agent instructed to stop and ask rather than work around a licence. |
| OSM coverage skew | Hyperscale owner-operated sites missing while colocation is over-represented | Report coverage limitations prominently. Cross-check against known hyperscale campus lists compiled from company press releases (public, citable). |
| Footprint-derived capacity is very uncertain | False precision in the headline | Tier recorded per facility; tier 3 reported as ranges. Show the headline figure with and without tier 3 to demonstrate sensitivity. |
| Announced capacity double-counted with operating | Overstated ranking | Separate stacked components, never summed into one number without labelling. |
| Water framing oversimplified | Reads as a hit piece rather than analysis | Direct and indirect reported separately, cooling-technology caveat stated in the README and in FINDINGS. |

---

## 7. Stretch

If the core lands quickly: overlay interconnection queue positions from ISO queue data (CAISO, ERCOT,
PJM, MISO publish these) to show where new generation is being requested relative to new load. That
turns a siting study into a load-growth-versus-supply study, which is the question every power-markets
interviewer is actually thinking about in 2026.
