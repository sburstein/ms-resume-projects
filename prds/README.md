# PRDs: Agent-Executable Build Specs

Seventeen PRDs, one per project, written for an agent to execute milestone by milestone. Each has
numbered milestones with **binary acceptance criteria**, a data-contract table with real endpoints, a
risk table, and golden-number regression tests.

**Read [`00-CONVENTIONS.md`](00-CONVENTIONS.md) first.** Every PRD assumes it: guardrails, stack,
repo layout, the `sources.yaml` contract, definition of done, and the agent execution protocol.

---

## How to start a build

Open a fresh Claude Code session in an empty directory and paste:

```
Read ~/ms-resume-projects/prds/00-CONVENTIONS.md and ~/ms-resume-projects/prds/06-utility-affordability-PRD.md.
Then read ~/ms-resume-projects/knowledge-packets/06-utility-affordability.md for domain background.
Execute milestone M0 only. Run its acceptance check. Commit. Then stop and report.
```

One milestone per session for the large projects. Two or three per session for the small ones.
`/clear` between milestones. That scoping is the main control on both wall-clock time and token
consumption.

---

## Build order

| Order | PRD | Repo | Bullets | Résumés | Hours |
|---|---|---|---|---|---|
| 1 | [06 Utility affordability](06-utility-affordability-PRD.md) | `utility-affordability` | `dataset` | 19 | 35 |
| 2 | [07 Nowcast pipeline](07-nowcast-pipeline-PRD.md) | same | `nowcast` | 17 | 20 |
| 3 | [08 Parent crosswalk](08-parent-crosswalk-PRD.md) | same | `ticker` | 13 | 12 |
| 4 | [04 AI weather benchmark](04-ai-weather-benchmark-PRD.md) | `ai-weather-bench` | `aiwx`, `aiwxBias` | 12 | 25 |
| 5 | [02 Net-load baseline](02-netload-baseline-PRD.md) | `netload-baseline` | `btm`, `btmShort` | 12 | 35 |
| 6 | [03 Weather sensitivity](03-weather-sensitivity-PRD.md) | same | `cloud` | 11 | 15 |
| 7 | [05 Energy applications](05-energy-applications-PRD.md) | `ai-weather-bench` | `supply`, `supplyShort` | 12 | 25 |
| 8 | [10 Workbook doctor](10-workbook-doctor-PRD.md) | `workbook-doctor` | `agent` | 9 | 18 |
| 9 | [14 Data center siting](14-datacenter-siting-PRD.md) | `datacenter-siting` | `datacenter` | 4 | 22 |
| 10 | [09 Climate risk divergence](09-climate-risk-divergence-PRD.md) | `climate-risk-divergence` | `vendors`, `vendorsShort` | 15 | 25 |
| 11 | [01 ENSO basket](01-enso-basket-PRD.md) | `enso-basket` | `elnino` | 11 | 25 |
| 12 | [13 EDGAR MCP](13-edgar-mcp-PRD.md) | `edgar-mcp` | `kensho` | 5 | 15 |
| 13 | [11 Financed emissions](11-financed-emissions-PRD.md) | `financed-emissions` | `ghg` | 8 | 30 |
| 14 | [12 Emissions QC](12-emissions-qc-PRD.md) | `emissions-qc` | `disclosure` | 6 | 20 |
| 15 | [15 Sector pathways](15-sector-pathways-PRD.md) | `sector-pathways` | `frameworks` | 4 | 18 |
| 16 | [16 Multi-hazard](16-multihazard-PRD.md) | `multihazard-analytics` | `hazard` | 3 | 28 |
| 17 | [17 Asset locations](17-asset-locations-PRD.md) | `asset-locations` | `geo` | 4 | 25 |

**Stopping points.** After order 3 you have covered the three most-used bullets (49 résumé
placements) in one repo. After order 7 you have every power-market bullet. Orders 13 to 17 are
role-conditional; build them when an application demands it.

---

## PRDs with a gate before code

Three PRDs open with a blocking gate the agent must resolve before writing anything:

- **[01 ENSO basket](01-enso-basket-PRD.md)**: outside-activity and personal-publication
  pre-clearance, or the repo stays private. The agent stops and asks if unconfirmed.
- **[15 Sector pathways](15-sector-pathways-PRD.md)**: audit `~/projects/ngfs-scenario-explorer` and
  extend it rather than rebuilding.
- **[17 Asset locations](17-asset-locations-PRD.md)**: scale honesty. Do not attempt or imply the
  licensed 9.5M-site scale; pre-register the coverage denominator before harvesting.

Two more have reuse checks: **[13 EDGAR MCP](13-edgar-mcp-PRD.md)** against your existing 23-company
research platform, and **[05 Energy applications](05-energy-applications-PRD.md)** against PRD 04's
alignment layer.

---

## The correctness controls worth knowing about

Each PRD has one or two milestones where getting it wrong invalidates everything downstream. These
are the ones to review personally rather than trusting an acceptance check:

| PRD | Control | Why |
|---|---|---|
| 04 | Reproduce a published WeatherBench 2 number within 2% | If the harness is wrong, every score is wrong |
| 02 | Gross-load reconstruction must improve model fit | If it does not, the BTM simulation is wrong |
| 07 | Runtime guard on `published_at <= as_of` | Lookahead bias is invisible without it |
| 01 | Exposure map hash frozen before backtest | Prevents post-hoc tuning |
| 09 | Assertions that held-fixed factors are actually held fixed | Otherwise the attribution is meaningless |
| 16 | `assert_projected()` before every spatial operation | Degrees are not metres |
| 06 | Weights per tract sum to <= 1.0 | Catches overlay errors |
| 17 | Measured LLM extraction error on a held-out sample | Extraction without an error rate is not data |

---

## Guardrails, restated because they matter

No MS code, data, or output enters any repo. `~/Downloads/el_nino_research_handoff.md` stays private.
Never fabricate data to make a pipeline run. Never retrofit a computed result to match a résumé
figure. When blocked by a licence, a rate limit, or a schema change: stop, write it down, report.
