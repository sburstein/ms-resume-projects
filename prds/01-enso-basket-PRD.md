# PRD 01: ENSO Long/Short Equity Basket

| | |
|---|---|
| **Repo** | `enso-basket` |
| **Bullets covered** | `elnino` (11 of 20 résumés) |
| **Packet** | [`01-enso-equity-basket.md`](../knowledge-packets/01-enso-equity-basket.md) |
| **Est.** | 25 human-hours, 6 to 9 agent session hours |
| **Priority** | 11 of 17. **Compliance-gated: do not start until pre-clearance is confirmed.** |
| **Model** | Opus for M3 (backtest design, where lookahead bias hides). Sonnet elsewhere. |
| **Prereqs** | **Outside-activity / personal-publication pre-clearance.** Read `00-CONVENTIONS.md`. |

---

## 0. Gate before any work begins

This is the one project that publishes a market view. Before the agent writes code:

1. Confirm with the user that outside-activity and personal-publication pre-clearance has been
   obtained, or that the repo will remain **private** and be demoed live rather than published.
2. The repo carries a prominent disclaimer: research and educational purposes, not investment advice,
   no recommendation, past performance is not indicative, the author's employer is not involved and
   this does not represent its views.
3. No live trading, no broker integration, no position sizing for real capital. Backtest only.

**If the agent cannot confirm item 1, it stops and asks. It does not proceed on an assumption.**

---

## 1. Objective

Translate the historical ENSO record into a market-neutral long/short equity basket, backtest it
across every El Niño event since 1990, and honestly characterize the statistical strength of the
result given a sample of roughly eight events.

**Success test:** an event-study chart with a null distribution built from random windows, and a
`FINDINGS.md` that states the significance honestly rather than overselling it.

---

## 2. Scope

**In:** ONI-defined event windows since 1990. Equity and ETF proxies. Dollar-neutral and beta-neutral
construction. Probability-scaled sizing. Event study with a permutation null.

**Out:** Futures (no free clean history; use ETF proxies and state the roll-yield contamination).
Options. Intraday. Live or paper trading. Any claim of a deployable strategy.

---

## 3. Data contracts

| id | Source | Access | Gotcha |
|---|---|---|---|
| `noaa_oni` | NOAA CPC Oceanic Niño Index, monthly, 1950 to present | Free text file | ONI is revised when the climatology base shifts every 5 years. Pin one version; record which. |
| `noaa_nino34` | Weekly Niño 3.4 SST anomaly | Free | Event timing |
| `iri_plume` | IRI/CPC ENSO probability forecasts | IRI site; recent years machine-readable, older are images | Only recent history is usable. State the limitation; the probability-scaling analysis covers a shorter sample than the event study. |
| `prices` | Daily equity and ETF prices | `yfinance` primary, Stooq fallback | **Survivorship bias: delisted names are absent.** Document it. Do not pretend it is handled. |

---

## 4. Milestones

### M0. Scaffold and disclaimer
Repo per conventions, plus `DISCLAIMER.md` and the disclaimer reproduced at the top of `README.md`.
**Accept:** disclaimer present; repo visibility matches what was cleared in section 0.

### M1. Event definition
`ingest/enso.py`. Parse ONI. Implement the NOAA definition: five consecutive overlapping three-month
seasons at or above +0.5 C. Classify by peak ONI into weak / moderate / strong / very strong.

**Anchor every window on the month ONI first crosses +0.5, never on the peak.** Anchoring on the peak
is lookahead: you would not have known the peak at the time.
**Accept:** the event table is produced and committed **before any price data is touched**. It should
contain roughly 8 events since 1990, with 1997-98, 2015-16, and 2023-24 classified very strong. A
test asserts every window start precedes its own peak.

### M2. Exposure map
`config/exposure_map.yaml`, written and committed **before any returns are computed**. For each
theme: long leg, short leg, and a one-sentence physical rationale.

Themes to cover: soft commodities (sugar, palm oil, cocoa, coffee), grains, metals with a water-use
channel (copper), EM rate sensitivity, consumer staples input costs, Atlantic hurricane suppression
(insurance and reinsurance), and North American winter heating demand (natural gas).
**Accept:** the file is committed in its own commit, timestamped before the backtest commit. Every
position has a written mechanism, not just a ticker. A test asserts the file hash is unchanged
between the backtest run and the reported results, so it cannot be quietly tuned later.

### M3. Construction and backtest  **← Opus**
`model/basket.py`:
1. Equal risk weight within each leg (inverse trailing volatility), not equal dollars.
2. Scale legs to dollar-neutral.
3. Estimate each name's beta to SPY on the prior 252 days and rescale to beta-neutral. **Betas
   estimated on trailing data only.** Produce both dollar-neutral and beta-neutral variants.
4. Probability scaling: gross exposure proportional to forecast event probability times expected
   intensity, using only information available at the rebalance date.

Backtest over 3, 6, and 12 month windows from each event anchor. Report per-event returns, hit rate,
mean, median, and a t-stat across events.
**Accept:** a runtime guard asserts no data with a timestamp after the decision date enters any
calculation. Test it deliberately by feeding future data and confirming the guard fires.

### M4. Null distribution
Sample 10,000 random windows of identical length from the same price history and compute the basket
return in each. Report the event-window result's percentile against this null.
**Accept:** the null distribution is computed and plotted. **The headline result is reported as a
percentile against the null, never as a raw return in isolation.**

### M5. Attribution and robustness
1. Decompose basket return by theme. Report which legs carried it and whether one or two names
   dominate.
2. Residual factor loading: regress basket returns on SPY, a commodity index, and a dollar index. If
   residual loadings are large, the basket is not isolating the weather view. Report the loadings.
3. Sensitivity: results with and without probability scaling; dollar-neutral vs beta-neutral;
   excluding the single largest contributor.
**Accept:** all three reported. If one name drives the result, say so in the first paragraph of
FINDINGS.md.

### M6. Figures and write-up
1. **Event study**: cumulative basket return in event time, one line per event plus the average, with
   the null distribution band shaded.
2. **Theme attribution** bar chart.
3. **ONI overlay**: ONI series with event windows shaded and basket cumulative return beneath.

`FINDINGS.md` must state, in plain language: the sample is roughly eight events, statistical power is
low, the direction is consistent and the mechanism is physical rather than data-mined, and the
appropriate use is as one input among several rather than as a standalone strategy.

---

## 5. Golden-number regression tests

- Event count and classification, exact.
- Exposure map file hash, exact.
- Mean 6-month event-window return, tolerance 0.5pp.
- Null-distribution percentile of the headline result, tolerance 3 percentiles.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Lookahead via peak anchoring** | Implausibly strong results | M1 anchors on first threshold crossing; test asserts it. |
| **Post-hoc tuning of the exposure map** | Results improve suspiciously after "reviewing" the map | Map committed before backtest, hash asserted at run time. |
| Survivorship bias | Overstated returns | Documented explicitly. Do not claim it is corrected. |
| ETF roll yield contaminating commodity proxies | Return attribution wrong | Documented. Where possible, compare an ETF proxy against a spot index to size the contamination. |
| Overselling n=8 | Credibility damage in interview | FINDINGS.md language requirement in M6, and the null distribution as the headline framing. |
| Publishing without clearance | Real professional risk | Section 0 gate. Agent stops if unconfirmed. |

---

## 7. Interview usage

Even if this stays private, the packet's interview answers stand on their own. The spring
predictability barrier, the market-neutral rationale, and the honest treatment of a small sample are
the three things that make this a strong conversation rather than a weak backtest.
