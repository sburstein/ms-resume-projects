# PRD 16: Multi-Hazard Geospatial Analytics

| | |
|---|---|
| **Repo** | `multihazard-analytics` |
| **Bullets covered** | `hazard` (3 of 20 résumés) |
| **Packet** | [`16-multihazard-geospatial.md`](../knowledge-packets/16-multihazard-geospatial.md) |
| **Est.** | 28 human-hours, 7 to 10 agent session hours |
| **Priority** | 16 of 17. Role-conditional: geospatial and remote-sensing roles. |
| **Model** | Sonnet for M0 to M3 and M6. Opus for M4 (SAR flood detection) if thresholds misbehave. |
| **Prereqs** | Read `00-CONVENTIONS.md`. PRD 17 supplies the asset layer if built; otherwise use a synthetic one. |

---

## 1. Objective

Two capabilities on one data spine: **static hazard exposure** for a set of asset locations, and a
**near-real-time event tracking pipeline** that intersects live event footprints with assets and
alerts on new exposure, with every footprint version persisted so the state of knowledge at any point
during an event can be reconstructed.

**Success test:** the NRT pipeline, run against a historical event replay, produces a correct,
deduplicated, timestamped sequence of affected-asset alerts.

---

## 2. Scope

**In:** flood, wildfire, tropical cyclone. CONUS. Static exposure from FEMA NFHL, FEMA NRI, USFS
Wildfire Risk to Communities, and NOAA surge products. NRT from NASA FIRMS, NIFC perimeters, NHC
advisories, and Copernicus EMS. One Sentinel-1 SAR flood-extent case study.

**Out:** building a flood or fire-spread model. Global coverage. Loss estimation (no vulnerability
functions). Earthquake and severe convective storm.

---

## 3. Data contracts

| id | Source | Access | Notes |
|---|---|---|---|
| `fema_nfhl` | National Flood Hazard Layer | ArcGIS REST / bulk | Zone A/AE = 1% annual chance. Backward-looking, defence-dependent, no pluvial. |
| `fema_nri` | National Risk Index | Free CSV | Tract and county, 18 hazards |
| `usfs_wrc` | Wildfire Risk to Communities | Free raster | Burn probability, flame length |
| `landfire` | LANDFIRE fuels | Free raster | Large |
| `mtbs` | Monitoring Trends in Burn Severity | Free | Historical fire perimeters |
| `nifc_perimeters` | NIFC active incident perimeters | Free ArcGIS feature service | **NRT source**, updates during events |
| `nasa_firms` | VIIRS and MODIS active fire detections | Free API with key | **NRT source.** VIIRS 375 m, ~3h latency. |
| `nhc_gis` | NHC advisories, forecast cone, wind radii | Free GIS products | **NRT source.** Use wind radii for impact, never the cone. |
| `copernicus_ems` | EMS rapid mapping flood delineations | Free | Activated events only |
| `sentinel1` | Sentinel-1 GRD SAR | Microsoft Planetary Computer STAC or AWS | For the flood-extent case study |
| `usgs_3dep` | Elevation | Free | Terrain masking for SAR |

---

## 4. Milestones

### M0. Scaffold and asset layer
Repo per conventions. Asset layer: PRD 17 output if available, otherwise a synthetic 2,000-asset
CONUS set weighted toward realistic industrial and commercial locations. **Every asset carries a
`geocode_quality` tier** (`rooftop`, `parcel`, `street`, `zip_centroid`).
**Accept:** asset layer exists with quality tiers populated. A test asserts no asset lacks a tier.

### M1. Projection discipline
`geo/crs.py` with an `assert_projected(gdf)` guard called before every area, distance, or buffer
operation. Standard CRS: EPSG:5070 for CONUS analysis, EPSG:4326 only for storage and display.
**Accept:** a test asserts the guard raises on a WGS84 GeoDataFrame. **This is the single most
important correctness control in the repo**; a 1 km buffer in degrees is a different distance at
every latitude.

### M2. Static exposure
Join assets to each hazard layer. Output a per-asset hazard profile recording, for every value, the
source layer and its vintage.
**Accept:** every asset has a value or an explicit null with a reason for each hazard. **Fine-grained
hazard values are suppressed for assets with `zip_centroid` geocode quality**, replaced by a
coarse-resolution value with a flag. A precise hazard on an imprecise location is false precision and
the pipeline must refuse to produce it.

### M3. Event footprint normalization
`nrt/footprints.py`. Normalize every NRT source into a common schema:
`(event_id, hazard_type, geometry, observed_at, severity, source, footprint_version)`.

Per source:
- NIFC: perimeter polygon, severity from acreage.
- FIRMS: point detections, clustered with DBSCAN and buffered into an approximate footprint.
- NHC: **wind radii polygons, not the forecast cone.** The cone contains the storm centre about two
  thirds of the time and says nothing about impact extent; a test should assert the cone is not used
  as a footprint.
- Copernicus EMS: delineation polygons.
**Accept:** all four sources normalize. Cone-misuse test passes.

### M4. Sentinel-1 flood case study  **← Opus if thresholds misbehave**
Pick one historical flood with EMS coverage. Search Sentinel-1 GRD via STAC for pre-event and
post-event scenes. Threshold VV backscatter, difference against the pre-event baseline, mask
permanent water (from a water occurrence layer) and radar shadow using 3DEP terrain.
**Accept:** derived flood extent overlaps the Copernicus EMS delineation with an IoU above 0.5.
Report the actual IoU. If below, diagnose (threshold, speckle filtering, terrain masking) rather than
tuning until it passes.

### M5. NRT pipeline
1. **Poll** each source on a schedule.
2. **Persist every footprint version.** Never overwrite. This is what makes post-event
   reconstruction possible.
3. **Intersect** with assets using a spatial index (`gdf.sindex`).
4. **Deduplicate alerts**: an asset already alerted for an event does not re-alert unless severity
   crosses a new threshold. A growing fire perimeter must not produce an alert every poll.
5. **Emit** alerts as structured records with the footprint version that triggered them.

**Accept, all three:** a replay test against a historical event (a named 2023 to 2025 wildfire with
NIFC perimeter history) produces a correct alert sequence; the dedupe test confirms a growing
perimeter produces one alert per asset per threshold crossing, not per poll; and the version store
allows reconstructing the affected-asset list as of any timestamp during the event.

### M6. Figures and docs
1. **Sentinel-1 flood extent** vs EMS delineation, side by side with IoU annotated.
2. **Event replay timeline**: perimeter growth with cumulative affected assets over time.
3. **Static exposure summary** by hazard and geocode quality tier, showing how much of the portfolio
   has hazard values too coarse to act on.
4. `LIMITATIONS.md` covering the FEMA NFHL caveats (backward-looking, no pluvial, defence-dependent)
   and the geocode-precision constraint.

---

## 5. Golden-number regression tests

- Sentinel-1 flood extent IoU against EMS, tolerance 0.05.
- Alert count in the event replay, exact.
- Share of assets with suppressed fine-grained hazard due to geocode quality, tolerance 1pp.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Unprojected geometry operations** | Silently wrong areas and buffers | M1 guard, tested. Highest-impact control in the repo. |
| Using the NHC cone as an impact footprint | Wrong assets flagged | M3 test asserts wind radii are used. |
| Alert storms from growing perimeters | Tool gets muted | M5 dedupe requirement, tested. |
| Overwriting footprints | Cannot reconstruct what was known when | Versioned persistence in M5, and it is a hard requirement not an optimization. |
| False precision on coarse geocodes | Misleading output | M2 suppression rule. |
| Sentinel-1 data volume | Slow, disk heavy | STAC search, subset to AOI before download, one case study only. |

---

## 7. Interview artifact

The event replay timeline. Showing perimeter growth alongside cumulative affected assets, with the
ability to answer "what did the system know at 04:00 on day 2", is the thing that distinguishes an
operational pipeline from a static risk score.
