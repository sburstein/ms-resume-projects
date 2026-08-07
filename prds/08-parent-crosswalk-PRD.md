# PRD 08: Utility to Publicly Traded Parent Crosswalk

| | |
|---|---|
| **Repo** | `utility-affordability` (same repo as PRDs 06 and 07) |
| **Bullets covered** | `ticker` (13 of 20 résumés) |
| **Packet** | [`08-utility-parent-crosswalk.md`](../knowledge-packets/08-utility-parent-crosswalk.md) |
| **Est.** | 12 human-hours, 3 to 5 agent session hours |
| **Priority** | 3 of 17. Cheapest bullet in the plan. |
| **Model** | Sonnet throughout. Escalate only if Exhibit 21 parsing proves adversarial. |
| **Prereqs** | PRD 06 complete. Read `00-CONVENTIONS.md`. |

---

## 1. Objective

Build a dated, confidence-scored mapping from EIA utility IDs to publicly traded parent companies,
then use it to aggregate the PRD 06 energy burden dataset to parent-company level, converting a
geographic dataset into an investable one.

**Success test:** `outputs/tables/utility_parent_crosswalk.csv` maps utilities covering a stated
percentage of IOU residential customers to a ticker, with a confidence tier on every row and the
unmatched list published rather than hidden.

---

## 2. Scope

**In:** US investor-owned utilities. Parent identification via SEC Exhibit 21 as the primary source.
Effective-dated mappings. Coverage measured by customers, not by row count.

**Out:** municipal and cooperative utilities (they have no parent by construction; assign
`no_listed_parent` with reason `public_power`). Non-US utilities. Ownership percentages below the
control threshold, beyond flagging joint ownership.

---

## 3. Data contracts

| id | Source | Endpoint | Notes |
|---|---|---|---|
| `sec_tickers` | SEC ticker to CIK map | `sec.gov/files/company_tickers.json` | Small, fetch fresh each run |
| `sec_submissions` | Filing history per CIK | `data.sec.gov/submissions/CIK##########.json` | Includes `formerNames`, which is how renames get caught |
| `sec_exhibit21` | Subsidiaries of the Registrant | 10-K filing index, exhibit type `EX-21` | **Primary source.** HTML or text, format varies wildly by filer. |
| `eia_861` | From PRD 06 | | Utility names, IDs, states, customers |
| `eia_860` | EIA Form 860 | `eia.gov/electricity/data/eia860/` | Plant owner and operator fields, corroborating evidence |
| `gleif_lei` | GLEIF LEI level 2 relationships | `gleif.org` bulk download | Reported direct and ultimate parent relationships. Free, underused, good tiebreaker. |

**SEC etiquette:** descriptive `User-Agent` with contact email, max 10 requests/second, cache
everything. Violating this gets the IP blocked.

---

## 4. Milestones

### M0. Utility universe
From PRD 06's `utilities.parquet`, produce the target list: every IOU with residential customers,
with its states and customer count. Sort descending by customers; the top 50 get manual verification
later, so make the ordering explicit.
**Accept:** universe table exists, customer counts sum to the national IOU residential total from
PRD 06.

### M1. Candidate parent universe
Fetch `sec_tickers`. Filter to SIC codes 4911, 4931, 4932, 4939, 4991 (electric and combination
utilities), plus a manual additions list in `config/parent_overrides.yaml` for holding companies
classified elsewhere.
**Accept:** between 40 and 80 candidate parents. Print the list; eyeball it for obvious absentees
(Duke, AEP, Exelon, Southern, Dominion, NextEra, Xcel, Entergy, PSEG, Sempra, Edison International,
PG&E, Consolidated Edison, WEC, DTE, Ameren, CMS, CenterPoint, Alliant, Evergy, PPL, FirstEnergy,
Eversource, Avangrid, Pinnacle West, IDACORP, Portland General, NorthWestern, Black Hills, ALLETE,
OGE, NiSource, Unitil, MGE, Avista, Hawaiian Electric).

### M2. Exhibit 21 harvest
`ingest/exhibit21.py`. For each candidate parent, locate the most recent 10-K, find the EX-21
exhibit, download, and parse subsidiary names.

Parsing reality: Exhibit 21 comes as an HTML table, a plain text list, or occasionally prose. Write
three parsers and a dispatcher. Where parsing confidence is low, store the raw text and flag for
review rather than guessing.
**Accept:** at least 85% of candidate parents yield a non-empty subsidiary list. Report the failures
by name; do not silently proceed with a truncated universe.

### M3. Matching
`transform/match.py`.

1. **Normalize** both sides: strip `Inc|Co|Corp|LLC|LP|Company|Incorporated|Corporation`, expand
   `Elec→Electric`, `Pwr→Power`, `Svc→Service`, `Gas & Elec→Gas and Electric`, uppercase, collapse
   punctuation and whitespace. Keep the raw string alongside always.
2. **Block by state.** Only compare a utility against subsidiaries plausibly operating in its state,
   determined from the parent's 10-K or from EIA-860 plant locations.
3. **Score** with `rapidfuzz.fuzz.token_set_ratio`. Auto-accept >= 92, queue 80 to 92 for review,
   reject below 80.
4. **Corroborate** with GLEIF parent relationships and EIA-860 owner fields. A match confirmed by two
   independent sources is promoted a tier.

Emit confidence tier per row: `exhibit21_exact`, `fuzzy_high`, `fuzzy_reviewed`, `manual`,
`no_listed_parent`, `unmatched`.
**Accept:** every utility in the universe has exactly one row with exactly one tier. No nulls.

### M4. Manual verification of the top 50
Present the top 50 by customer count with their proposed parent and confidence, for human
confirmation. Record decisions in `config/manual_verifications.yaml` with a date and a note. The
agent proposes; the human confirms. Do not mark anything `manual` without a recorded human decision.
**Accept:** all 50 dispositioned, file committed.

### M5. Effective dating
Add `valid_from` and `valid_to` per mapping. Populate from SEC `formerNames`, from merger 8-Ks, and
from a hand-maintained `config/mergers.yaml` for the material recent deals. Where no date evidence
exists, set `valid_from` to the earliest EIA-861 year in scope and note the assumption.
**Accept:** a test asserts no overlapping validity windows for the same utility, and that a
historical join for year Y uses the mapping valid at Y.

### M6. Coverage report and parent-level aggregation
1. `outputs/tables/coverage.md`: share of IOU residential customers mapped to a listed parent, share
   with `no_listed_parent` (Oncor, Puget Sound Energy, Cleco, El Paso Electric, Duquesne Light and
   similar), share unmatched. Plus the full unmatched list.
2. `outputs/tables/parent_burden.csv`: for each parent, customer-weighted energy burden across its
   territories, share of customers in the top burden quintile, and the state regulatory footprint.
3. Figure: scatter of parent-level customer-weighted burden against customer count, labelled with
   tickers. This is the artifact that shows a physical dataset became a financial one.

**Accept:** coverage is stated as a percentage of customers, not of rows. Unmatched list is published.

---

## 5. Golden-number regression tests

- Share of IOU residential customers mapped to a listed parent, tolerance 1pp.
- Count of distinct parent tickers, tolerance 2.
- Top 50 manual verifications all present.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| Fuzzy match confuses opco with holdco | "Consolidated Edison Co of New York" matched to "Consolidated Edison, Inc." as if they were unrelated entities | Exhibit 21 is authoritative on which is which. Never rely on string distance for this specific relationship. |
| Exhibit 21 format defeats the parser | Empty subsidiary lists for major filers | Three parsers plus a manual fallback. Report failures by name. |
| Take-private and foreign ownership treated as failures | Coverage understated | `no_listed_parent` is a legitimate outcome with a reason code, not an error. |
| SEC rate limiting | 403s | Cache aggressively, 10 req/s ceiling, exponential backoff. |
| Merger changes structure mid-series | Time series silently corrupted | Effective dating in M5. Test it. |

---

## 7. Interview framing note for the README

State the commercial point, not the engineering point: energy burden aggregated to parent level is a
regulatory-risk signal, because high-burden territories mean harder rate cases, more political
attention, and higher bad-debt and disconnection rates. The crosswalk is the means; the parent-level
metric is the product.
