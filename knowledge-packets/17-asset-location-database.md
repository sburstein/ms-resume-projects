# Packet 17: Corporate Asset Location Database

> **Résumé line:** "Contributed to a corporate asset location database covering 9.5M+ sites with 99%
> coverage of the S&P 500 and 92% of MSCI World, used as the join key for asset-level physical risk
> analysis."

**Repo:** `asset-locations` | **Est. 25 hours** | **Tier B: public proxy. On 4 résumés. Lowest
leverage in the plan; build it only if you are chasing a geospatial or physical-risk role and have
already built Packet 16.**

---

## 1. Be honest about the scale

The MS database was licensed and assembled at a scale you cannot replicate. Do not try. A public
rebuild targeting the **S&P 100** with a stated, measured coverage percentage demonstrates exactly
the same skill and is achievable in a fortnight. Claiming 9.5M sites in a public repo you built alone
would be the one thing that damages you.

The framing to use: "the production system was licensed and much larger; I rebuilt the entity
resolution and coverage measurement problem at S&P 100 scale from public sources, because that is
where the actual difficulty lives."

---

## 2. Why this is hard, which is the real content

The naive view is that you geocode a list of addresses. The actual problems:

1. **Corporate structure.** A company's assets are held by subsidiaries, joint ventures, and
   partially-owned affiliates, often under names with no textual relationship to the parent. Coverage
   requires resolving ownership, and ownership requires Exhibit 21 (see Packet 08) and often more.
2. **What counts as an asset?** Owned versus leased. Operated versus owned. A retailer's 4,000 leased
   stores, a miner's three owned mines, and a bank's headquarters are all "sites" with wildly
   different materiality. A raw site count is a nearly meaningless metric unless you weight by value,
   revenue, or capacity. This is the first thing to say when someone quotes a site count at you.
3. **Entity resolution.** The same facility appears in EPA FRS, in OSM, in a state permit database,
   and in a 10-K property table, under four name variants and three coordinate pairs. Deduplication
   is the core engineering task.
4. **Geocoding quality.** Rooftop versus parcel centroid versus street interpolation versus ZIP
   centroid. For physical risk this difference is decisive, since flood and wildfire hazard vary over
   metres. Every coordinate needs a quality tier attached and carried downstream.
5. **Coverage measurement.** "99% of the S&P 500" needs a denominator definition. 99% of companies
   having at least one located asset is a trivial claim. 99% of revenue or of PP&E attributable to
   located assets is a hard one. Always state which. Asking this question back is itself a good
   interview move.

---

## 3. Data

| Source | What | Access | Note |
|--------|------|--------|------|
| **EPA FRS** | US regulated facilities with coordinates and parent linkage | Free bulk and API | The best single free US source. Covers anything with an environmental permit. |
| **EPA GHGRP / TRI / eGRID** | Emitting and generating facilities | Free | Overlaps FRS, adds capacity and emissions attributes |
| **SEC 10-K Item 2 (Properties)** | The company's own description of principal properties | Free via EDGAR | Unstructured text, but authoritative on materiality. Good LLM extraction target. |
| **SEC Exhibit 21** | Subsidiary list | Free | Required for the ownership layer, same as Packet 08 |
| **OpenStreetMap** | Buildings, industrial sites, plants, retail | Free, Overpass | Global, inconsistent, free of licensing complications |
| **Global Energy Monitor** | Plant-level trackers for coal, gas, oil and gas, steel, cement, LNG | Free | Best-in-class for heavy industry globally |
| **OpenCorporates** | Entity registry data | Partly free | Useful for the subsidiary layer outside the US |
| **Overture Maps / Foursquare OS Places** | Open POI and building datasets | Free | Newer, large, worth evaluating for retail footprints |
| **US Census / national statistical geocoders** | Free geocoding | Free | Census geocoder is free and decent for US addresses; avoid paid geocoders for a public repo |

---

## 4. Method

1. **Define the universe and the coverage denominator up front.** S&P 100 constituents. Denominator:
   share of consolidated PP&E, or share of revenue, attributable to located sites. Write this down
   before you start so you cannot move the goalposts later.
2. **Build the ownership layer** from Exhibit 21, producing parent to subsidiary edges with effective
   dates. Reuse Packet 08's code.
3. **Harvest sites** from each source, keyed by the subsidiary names in the ownership layer.
4. **Resolve entities.** Blocking on state and industry, then similarity on normalized name plus
   spatial proximity. Two records within 250 m with high name similarity are the same site. Keep a
   cluster id and every contributing source record. Never delete a source record.
5. **Extract from 10-K Item 2** with an LLM, into a structured schema (site type, location, owned or
   leased, approximate size). Then verify a sample by hand and report extraction accuracy. This is
   the honest way to use an LLM in a data pipeline: extract, then measure the extraction error rate,
   and publish it.
6. **Assign geocode quality tiers** and carry them into every downstream product.
7. **Weight sites** by an available materiality proxy: permitted capacity, emissions, employee count,
   or building footprint area from OSM.
8. **Measure and publish coverage** against the denominator you defined in step 1, per company and in
   aggregate. Publish the misses.
9. **Join to Packet 16** hazard layers to produce the asset-level physical risk output that is the
   whole reason the database exists.

---

## 5. Interview questions and how to answer

**"How do you get to 99% coverage?"**
Turn it into a question about the denominator, because that is the honest answer. "It depends
entirely on what the 99% is measured against. Ninety-nine percent of companies having at least one
located asset is easy and close to meaningless. Ninety-nine percent of PP&E or revenue attributable
to located sites is hard, and for asset-level risk work it is the only version that matters. In my
public rebuild I defined it as revenue-weighted and reported the number I actually achieved, which
was lower."

**"What is the hardest part?"**
Entity resolution across the ownership layer. The facility is registered to a subsidiary whose name
shares no tokens with the parent, so before you can attribute a site to a company you have to know
the corporate tree, and the corporate tree changes with every acquisition.

**"How would you validate it?"**
Sample and hand-verify, stratified by geocode quality tier and by source. Report precision and recall
per tier. And cross-check company totals against the 10-K Item 2 narrative, which is the company's
own statement about its principal properties.

**"You used an LLM to extract from filings. How do you know it is right?"**
Because I measured it rather than assumed it. Hand-verified a stratified sample and published the
extraction error rate alongside the data. An extraction pipeline without a measured error rate is not
a data product.

---

## 6. Further reading

- EPA Facility Registry Service documentation and the FRS data model.
- Global Energy Monitor tracker methodologies.
- The record-linkage literature: Fellegi-Sunter is the classical framework; `splink` and
  `dedupe` are the practical Python tools.
- Overture Maps Foundation schema documentation.
