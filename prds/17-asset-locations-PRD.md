# PRD 17: Corporate Asset Location Database

| | |
|---|---|
| **Repo** | `asset-locations` |
| **Bullets covered** | `geo` (4 of 20 résumés) |
| **Packet** | [`17-asset-location-database.md`](../knowledge-packets/17-asset-location-database.md) |
| **Est.** | 25 human-hours, 6 to 9 agent session hours |
| **Priority** | 17 of 17. Lowest leverage. Build only if chasing a geospatial role and PRD 16 is done. |
| **Model** | Sonnet throughout. Opus for M4 (LLM extraction validation design). |
| **Prereqs** | Read `00-CONVENTIONS.md`. PRD 08's Exhibit 21 parser is directly reusable. |

---

## 0. Scale honesty gate

The résumé bullet describes a licensed 9.5M-site database with 99% S&P 500 coverage. **Do not attempt
to match it, and do not imply that you have.** The public rebuild targets the **S&P 100** with a
measured, honestly reported coverage figure.

The README's first paragraph must state: the production system was licensed and far larger; this
rebuild reproduces the entity resolution and coverage measurement problem at S&P 100 scale from
public sources, because that is where the difficulty lives.

**The agent must not inflate coverage claims, and must define the denominator before measuring.**

---

## 1. Objective

Build a facility-level location database for S&P 100 constituents from public sources, with an
ownership layer, entity resolution across sources, geocode quality tiers, materiality weighting, and
a coverage figure measured against a denominator defined in advance.

**Success test:** a facility table with cluster IDs, quality tiers, and source provenance, plus a
coverage report stating what share of revenue or PP&E is attributable to located sites, with the
misses published.

---

## 2. Scope

**In:** S&P 100 constituents. US facilities primarily; international where public sources reach.
Owned and leased distinguished where determinable. Ownership resolution via Exhibit 21.

**Out:** the full S&P 500 or MSCI World. Paid geocoders. Paid facility databases. Any source whose
terms prohibit redistribution.

---

## 3. Data contracts

| id | Source | Access | Licence |
|---|---|---|---|
| `epa_frs` | EPA Facility Registry Service | Free bulk and API | US Gov work. **Best single free US source.** |
| `epa_ghgrp` / `epa_tri` | Emitting facilities | Free | US Gov work |
| `sec_exhibit21` | Subsidiaries of the Registrant | EDGAR | Public record. Parser reusable from PRD 08. |
| `sec_item2` | 10-K Item 2, Properties | EDGAR | Public record. Unstructured; LLM extraction target. |
| `osm` | Buildings, industrial sites, retail | Overpass | ODbL, attribution required |
| `gem_trackers` | Global Energy Monitor plant trackers | Free | Best public heavy-industry coverage |
| `overture` | Overture Maps places and buildings | Free | Permissive; evaluate for retail footprints |
| `census_geocoder` | US address geocoding | Free | US Gov work. Use this, not a paid geocoder. |

---

## 4. Milestones

### M0. Universe and denominator  **← do this before anything else**
S&P 100 constituent list with CIK, ticker, sector, revenue, and PP&E from EDGAR XBRL.

**Define the coverage denominator explicitly in `METHODOLOGY.md` before any harvesting.** Recommended:
share of consolidated PP&E attributable to located sites, with revenue-weighted as a secondary
measure. Commit this file before M1.
**Accept:** 100 companies with financials. Denominator definition committed, with its own commit
timestamp preceding the harvest commits.

### M1. Ownership layer
Reuse PRD 08's Exhibit 21 parser. Parent-to-subsidiary edges with effective dates for all 100.
**Accept:** at least 85% of companies yield a subsidiary list. Failures reported by name.

### M2. Source harvest
Query each source keyed by parent and subsidiary names. Store raw source records with full
provenance; never mutate or delete a source record.
**Accept:** per-source facility counts reported. EPA FRS should dominate for industrials and energy;
OSM for retail and offices. If a major industrial company yields zero FRS facilities, the name
matching is broken.

### M3. Entity resolution
Blocking on state and industry, similarity on normalized name, plus spatial proximity. Two records
within 250 m with compatible names are one site. Emit `cluster_id` with every contributing source
record retained.
**Accept:** hand-verified test set of 25 known facilities clusters correctly. Report the dedupe
reduction ratio and the count of ambiguous clusters queued for review.

### M4. Item 2 extraction with measured accuracy  **← Opus for the validation design**
LLM extraction from 10-K Item 2 into a schema: `(site_type, location_text, owned_or_leased,
approximate_size, segment)`.

**Then measure it.** Hand-verify a stratified random sample of at least 50 extractions across
sectors. Compute precision and recall per field.
**Accept:** the extraction error rate is computed and published in `FINDINGS.md`. **An extraction
pipeline without a measured error rate is not a data product**, and the agent must not proceed to M5
without this number. Do not tune the prompt against the validation sample; hold it out.

### M5. Geocoding and quality tiers
Geocode `location_text` via the Census geocoder. Assign tiers: `rooftop`, `parcel`, `street`,
`city_centroid`, `zip_centroid`, `state_only`. Carry the tier into every downstream record.
**Accept:** tier distribution reported. A test asserts no record lacks a tier.

### M6. Materiality weighting
Weight sites by the best available proxy: permitted capacity, reported emissions, OSM building
footprint area, or an equal weight within site type as a fallback. Record which proxy was used.
**Accept:** every site has a weight and a proxy label. The README states plainly that raw site counts
are nearly meaningless without weighting, and shows the difference between count-based and
weight-based coverage.

### M7. Coverage report
Against the M0 denominator: coverage per company and in aggregate, plus the published miss list.
**Accept:** coverage stated against the pre-registered denominator. **Report the number achieved,
whatever it is.** A 62% PP&E-weighted coverage honestly measured is a stronger artifact than a
claimed 99% with an undefined denominator, and the README should make that argument explicitly.

### M8. Join to hazard
Join the located sites to PRD 16's hazard layers, respecting the geocode-quality suppression rule
(no fine-grained hazard on `zip_centroid` locations). This join is the reason the database exists.
**Accept:** asset-level physical risk output produced, with the share of sites too coarsely located
for fine-grained hazard reported prominently.

---

## 5. Golden-number regression tests

- Facility count after clustering, tolerance 5%.
- LLM extraction precision and recall on the held-out sample, tolerance 5pp.
- PP&E-weighted coverage, tolerance 3pp.
- Geocode tier distribution, tolerance 3pp per tier.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Overstating coverage** | Claimed figure without a stated denominator | M0 pre-registers the denominator with its own commit. |
| Unmeasured LLM extraction error | Confident wrong data | M4 gate: no progression without a measured error rate on a held-out sample. |
| Prompt tuned against the validation set | Error rate understated | Hold out the sample; do not iterate against it. State this in METHODOLOGY. |
| Ownership layer misses subsidiaries | Under-coverage that looks like real absence | Report Exhibit 21 parse failures by name; distinguish "no facilities found" from "could not resolve ownership". |
| Deleting source records during dedupe | Cannot audit or reprocess | Cluster IDs with all contributing records retained; test asserts no source record is dropped. |
| Paid geocoder creeps in | Licence and reproducibility problem | Census geocoder only; whitelist enforces it. |

---

## 7. What this project actually demonstrates

Not scale. Entity resolution across an ownership tree, honest coverage measurement against a
pre-registered denominator, and a measured error rate on an LLM extraction step. Those three things
are the transferable skill, and they are exactly what the licensed version required at larger scale.
