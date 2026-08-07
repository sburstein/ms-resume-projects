# Packet 16: Multi-Hazard Geospatial Analytics

> **Résumé line:** "Built hazard-specific geospatial analytics across flood, wildfire, and tropical
> cyclone using FEMA, Copernicus, and Sentinel-derived datasets, including near-real-time event
> tracking against asset footprints."

**Repo:** `multihazard-analytics` | **Est. 28 hours** | **Tier A. On 3 résumés, and the right
project for the remote-sensing and geospatial roles (Muon Space, Neural Earth, CarbonPlan).**

---

## 1. The idea

Two distinct capabilities that share a data spine.

**Static exposure:** for a set of asset locations, what is their flood, wildfire, and tropical
cyclone hazard, from the best available public layers?

**Near-real-time event tracking:** when a hurricane or wildfire is actually happening, which assets
are inside the footprint, right now, automatically?

The second is what makes the project distinctive. Static risk scores are commodity. An operational
pipeline that ingests an active event footprint and returns an affected-asset list within minutes is
what an insurer, a lender, or an operator actually wants at 3am.

---

## 2. Hazard by hazard

### Flood

**Types:** riverine (fluvial), surface water (pluvial), coastal storm surge, and groundwater. They
have different drivers and different data, and conflating them is the standard amateur error. FEMA's
maps are strong on riverine and coastal and essentially silent on pluvial, which is why FEMA
flood-zone status underestimates real flood exposure, particularly in cities.

**The FEMA NFHL caveats you must know:**
- The **100-year floodplain** (Special Flood Hazard Area, Zone A and AE) means a 1% annual chance of
  flooding, not "floods once a century." Say the annual probability, not the return period, and note
  that over a 30-year mortgage the cumulative probability is roughly 26%.
- Maps are **backward-looking**, based on historical hydrology, and many are decades old.
- Maps reflect **current defences**, so a levee-protected area can appear low risk while carrying
  significant residual risk if the levee fails or is overtopped.
- Coverage is incomplete: many rural areas are unmapped.

**Data:** FEMA National Flood Hazard Layer (free, API and downloads), FEMA National Risk Index,
Copernicus Emergency Management Service rapid mapping (free flood delineations for activated events),
Sentinel-1 SAR for flood extent (radar penetrates cloud, which is exactly what you need during a
storm), USGS 3DEP elevation, and the Global Flood Awareness System (GloFAS) for forecast river
discharge.

**Sentinel-1 flood mapping method:** water is specular at C-band, so it reflects radar away from the
sensor and appears very dark. Threshold the VV backscatter, difference against a pre-event baseline
image, and mask permanent water and radar shadow from terrain. This is a well-trodden workflow and it
is genuinely operational.

### Wildfire

**Drivers:** fuel (load, type, continuity), weather (temperature, relative humidity, wind, and
critically antecedent dryness), topography (fire moves fast uphill), and ignition (in the US the
large majority of ignitions are human, but lightning fires burn more acreage in some regions).

**Key indices to know:** ERC (Energy Release Component) and the wider NFDRS family, the Fosberg Fire
Weather Index, and VPD (vapour pressure deficit), which has become the standard atmospheric-dryness
metric in the recent literature and correlates strongly with burned area.

**Data:** USFS Wildfire Risk to Communities (free, national, burn probability and flame length),
LANDFIRE fuels, MTBS burn severity history, NIFC active incident perimeters (free, updated during
events), and satellite active-fire detections from **VIIRS** (375 m, roughly 4 overpasses a day) and
**MODIS** (1 km), both free via NASA FIRMS with a near-real-time feed. FIRMS is the backbone of the
real-time half.

### Tropical cyclone

**Data:** NOAA NHC advisories and the forecast cone (free, machine-readable via the GIS products),
IBTrACS for the historical global track archive, and NOAA's Hurricane Surge product plus SLOSH MOMs
for surge.

**The forecast cone is widely misread and this is a good thing to know:** it contains the historical
track of the *centre* about two thirds of the time. It says nothing about the size of the storm, and
impacts routinely extend far outside it. Never treat the cone as an impact footprint.

**Wind field:** the NHC wind speed probability products and the wind radii in the advisory are what
you intersect assets against, not the cone.

---

## 3. The build

**Static exposure pipeline.**
1. Asset locations in, with a coordinate quality flag (rooftop geocode versus ZIP centroid matters
   enormously here, and a ZIP centroid can put an asset in the wrong flood zone entirely).
2. Spatial join against each hazard layer in an equal-area projection.
3. Output a per-asset hazard profile with the layer version and vintage recorded for each value.

**Near-real-time pipeline.**
1. **Poll** the event feeds: NIFC perimeters, NASA FIRMS active fire, NHC advisories, Copernicus EMS
   activations, USGS stream gauges.
2. **Normalize** each into a common event footprint schema: geometry, event id, hazard type,
   timestamp, severity, source.
3. **Intersect** with the asset layer using a spatial index (R-tree via `geopandas.sindex`, or
   PostGIS if you want it to be a real service).
4. **Alert** on new intersections, with deduplication so a slowly growing fire perimeter does not
   re-alert every poll.
5. **Persist** every footprint version, so you can reconstruct what was known at any point during the
   event. This is the same vintage discipline as Packets 07 and 12, and it is what makes the system
   auditable after the fact.

**Tech:** `geopandas`, `shapely`, `rasterio`, `xarray` and `rioxarray` for gridded data, `pystac-client`
and `odc-stac` for satellite search, and either **Microsoft Planetary Computer** or **AWS Open Data**
for Sentinel access without a Copernicus account. Google Earth Engine is the alternative if you want
server-side processing and are comfortable with its licence terms.

**The critical technical discipline: projections.** Every area, distance, and buffer computation must
happen in an appropriate projected CRS, not in WGS84 degrees. A 1 km buffer in degrees is a different
distance at different latitudes. This single error invalidates more geospatial analysis than anything
else.

---

## 4. Numbers to know cold

- **100-year floodplain** = **1% annual** chance; roughly **26%** cumulative over 30 years.
- VIIRS active fire resolution **375 m**, MODIS **1 km**; FIRMS near-real-time latency is roughly
  **3 hours**, with an ultra-real-time feed under an hour for some regions.
- Sentinel-1 revisit: **6 to 12 days** depending on constellation status; Sentinel-2 **5 days** at the
  equator with two satellites.
- NHC forecast cone contains the storm centre about **two thirds** of the time.
- Saffir-Simpson categories are **wind only** and say nothing about surge or rainfall, which cause
  the majority of US tropical cyclone deaths.

---

## 5. Interview questions and how to answer

**"Why not just use FEMA flood zones?"**
Three reasons, stated crisply: they are backward-looking and often decades old; they largely exclude
pluvial (surface water) flooding, which is a large share of urban flood loss; and they reflect
current defences, so levee-protected areas look safe while carrying residual failure risk. They are a
regulatory product for insurance purchase requirements, not a risk model, and using them as one is a
category error.

**"How would you detect flooding from satellite?"**
Sentinel-1 SAR, because radar sees through cloud and flooding happens under cloud. Water is specular
at C-band so it appears dark; threshold the backscatter, difference against a pre-event baseline, and
mask permanent water and terrain shadow. Optical (Sentinel-2) is higher confidence when you can get a
clear scene, which during a flood you usually cannot.

**"What is the hardest part of asset-level physical risk?"**
Geocoding quality, by a distance. Hazard varies over metres for flood and wildfire, and a ZIP-centroid
geocode can be kilometres from the actual asset. A precise hazard model on an imprecise location is
false precision, so I carried a coordinate-quality flag through to the output and refused to report
fine-grained hazard for coarse locations.

**"Real-time event tracking sounds simple. What is hard about it?"**
Deduplication and versioning. A fire perimeter updates continuously, so naive alerting fires
constantly. And after the event, someone will ask what you knew and when, so every footprint version
has to be persisted with its timestamp rather than overwritten.

---

## 6. Further reading

- Wing et al., "New insights into US flood vulnerability revealed from flood insurance big data,"
  Nature Communications 2020, and the First Street flood model methodology, on how much FEMA maps
  understate exposure.
- Abatzoglou and Williams, "Impact of anthropogenic climate change on wildfire across western US
  forests," PNAS 2016, for VPD and fuel aridity.
- Copernicus EMS Rapid Mapping portfolio, for worked flood delineation examples.
- NOAA NHC "Cone of Uncertainty" explainer, and the GIS product documentation.
- Microsoft Planetary Computer STAC catalog docs.
