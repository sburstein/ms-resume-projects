# Buildability Triage: All 25 Morgan Stanley GSF Bullets

**Question:** which of these can actually be built out with public data, on your own machine?

**Answer in one line:** 19 of 25 are fully buildable, 4 need a public proxy for a walled data source,
and 2 are not artifacts at all.

Ranked within each tier by leverage, defined as (résumés the bullet appears on) x (how much an
artifact helps the claim survive follow-up questions).

---

## Tier A: fully buildable from public data (19 bullets, 13 projects)

Nothing walled. You can rebuild the substance end to end and publish it.

| Bullet | Résumés | Project | Public data spine | Hours |
|--------|---------|---------|-------------------|-------|
| `dataset` | 19 | P6 utility affordability | EIA-861 + ACS tract income + HIFLD territories | 35 |
| `nowcast` | 17 | P7 nowcast pipeline | EIA-861M vs EIA-861 annual | 20 |
| `ambassador` see Tier C | 14 | n/a | n/a | n/a |
| `ticker` | 13 | P8 parent crosswalk | EIA-861 ownership + SEC EDGAR | 12 |
| `btm` + `btmShort` | 12 | P2 net-load baseline | EIA-930 + CA DG Stats + NSRDB + pvlib | 35 |
| `cloud` | 11 | P3 sensitivity | same repo + SALib/SHAP | 15 |
| `elnino` | 11 | P1 ENSO basket | NOAA ONI + equity prices | 25 |
| `agent` | 9 | P10 workbook doctor | openpyxl + MCP | 18 |
| `aiwx` + `aiwxBias` | 12 | P4 AI weather bench | WeatherBench 2 archives | 25 |
| `ghg` | 8 | P11 financed emissions | EPA GHGRP + PCAF method + USEEIO factors | 30 |
| `supply` + `supplyShort` | 12 | P5 energy applications | P4 forecasts + EIA-930 | 25 |
| `disclosure` | 6 | P12 emissions data QC | GHGRP vs CDP vs 10-K reported | 20 |
| `kensho` | 5 | P13 financial-data MCP | SEC EDGAR XBRL companyfacts | 15 |
| `datacenter` | 4 | P14 data center siting risk | Data Center Map/OSM + WRI Aqueduct + HIFLD | 22 |
| `frameworks` | 4 | P15 sector pathways | SBTi SDA + IEA NZE + NGFS | 18 |
| `hazard` | 3 | P16 multi-hazard analytics | FEMA NRI + Copernicus EMS + Sentinel | 28 |

**The two highest-leverage builds, by a wide margin:** `dataset` (on 19 of 20 résumés) and `nowcast`
(17). If you build nothing else, build those. `ticker` is a cheap add-on to the same repo and takes
the count to 3 bullets covered for about 67 hours total.

**Two of these you have already started.** `~/projects/ngfs-scenario-explorer` is most of P15.
Your 23-company AI equity research platform is adjacent to P13. Finish and document those rather
than starting fresh.

---

## Tier B: buildable only with a public proxy (4 bullets, 3 projects)

The substance is reproducible. One input is behind a paywall or a bank entitlement, so you rebuild
with an open substitute and say so plainly in the README. Stating the substitution is not a
weakness; hiding it is.

| Bullet | Résumés | What is walled | Public substitute | Project |
|--------|---------|----------------|-------------------|---------|
| `vendors` + `vendorsShort` | 15 | Verisk, Moody's RMS, Jupiter, First Street, XDI, Munich Re commercial licences | FEMA NRI as the historical-cat layer; NASA NEX-GDDP-CMIP6, LOCA2, and raw CMIP6 as the forward-looking layers | P9 climate risk divergence |
| `geo` | 4 | The 9.5M-site corporate asset database (bought, not built) | Build a smaller one honestly: S&P 100 facility locations from EPA FRS, Global Energy Monitor, OSM, and 10-K property tables. Report coverage achieved rather than claiming 99%. | P17 asset location DB |
| `elnino` note | 11 | Clean futures history, MS Research | ETF and equity proxies, NOAA ONI | see P1, Tier A |

**On `vendors` specifically:** you cannot license six commercial cat models. But the *finding* is
reproducible, because the divergence is caused by downscaling and normalization choices, and those
choices are visible in open models. Holding the GCM fixed while varying downscaling method, then
the reverse, reproduces the mechanism exactly. That is arguably a better artifact than the original,
because it isolates cause rather than just reporting disagreement.

**On `geo`:** do not try to hit 9.5M sites. A credible 3,000-facility database for the S&P 100 with
a stated coverage percentage and a documented match-confidence column demonstrates the same skill and
is achievable in a fortnight.

---

## Tier C: not buildable as an artifact (2 bullets)

These are role and process claims. No repo will ever substantiate them, and trying to fake an
artifact would look worse than having none.

| Bullet | Résumés | Why | What to do instead |
|--------|---------|-----|-------------------|
| `ambassador` | 14 | "Serve as the group's AI implementation ambassador" is a role, an internal appointment, and a track record of adoption. Adoption is the substance and it happened inside the firm. | Substantiate it sideways. The `workbook-doctor` repo, the MCP server, the multi-agent book pipeline, and this very build plan are all evidence you do this. Write a short public playbook post: "how I got a sceptical analytics team to actually use generative AI." That is the artifact, and it is a writing artifact, not a code one. |
| `audit` | 5 | Coordinating work-paper preparation with Deloitte across overrides, variance, and scope 3 testing is confidential engagement work. | Nothing to build. Prepare a clean verbal walkthrough of the control structure and what human-in-the-loop review meant in practice. The `workbook-doctor` project is the closest adjacent evidence. |

---

## What this changes about the plan

The original `PLAN.md` covered 10 projects for the 11 bullets on the Modo résumé. The full inventory
is 25 bullets, so the plan expands to **17 projects across 8 repos**, roughly 370 hours.

That is too much to do all of it. Recommended cut, in priority order:

**Must build (covers the 4 most-used bullets, 3 repos, ~92 hours):**
1. P6 utility affordability dataset (`dataset`, 19 résumés)
2. P7 nowcast and production pipeline (`nowcast`, 17)
3. P8 parent crosswalk (`ticker`, 13)
4. P4 AI weather benchmark (`aiwx` and `aiwxBias`, 12)

**Should build (the power-market roles you are actively applying to, ~75 hours):**
5. P2 net-load baseline with BTM solar (`btm`, 12)
6. P3 weather-driver sensitivity (`cloud`, 11)
7. P5 energy applications (`supply`, 12)

**Nice to have, high signal per hour (~55 hours):**
8. P10 workbook doctor (`agent`, 9) plus P13 financial-data MCP (`kensho`, 5)
9. P14 data center siting risk (`datacenter`, 4). Small, topical, and interviewers love it right now.

**Only if the role demands it:**
- P11 financed emissions and P15 sector pathways for the ESG and climate-finance roles (Blue Owl,
  CarbonPlan, Galvanize).
- P9 climate risk divergence and P16 multi-hazard for climate-risk roles (HASI risk, Muon Space,
  Neural Earth).
- P1 ENSO basket, gated on compliance pre-clearance, and worth it mainly for the commodities and
  cross-asset angle.
- P17 asset location DB and P12 emissions QC, lowest leverage of the set.
