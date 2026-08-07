# PRD 07: Nowcast and Productionized Monthly Release

| | |
|---|---|
| **Repo** | `utility-affordability` (same repo as PRD 06) |
| **Bullets covered** | `nowcast` (17 of 20 résumés) |
| **Packet** | [`07-nowcast-productionization.md`](../knowledge-packets/07-nowcast-productionization.md) |
| **Est.** | 20 human-hours, 5 to 8 agent session hours |
| **Priority** | 2 of 17 |
| **Model** | Opus for M2 (backtest design, where lookahead bias hides). Sonnet elsewhere. |
| **Prereqs** | PRD 06 complete through M6. Read `00-CONVENTIONS.md`. |

---

## 1. Objective

Two things, judged separately.

**A. Nowcast.** Close the EIA-861 reporting lag (9 to 12 months) by projecting the annual
utility-level residential rate forward using faster-published indicators, and prove it beats naive
carry-forward in an honest pseudo-real-time backtest.

**B. Productionization.** Turn the whole repo into a monthly release process with validated ingest,
a pinned source whitelist, versioned immutable outputs, and a documented methodology.

**Success test:** `make release MONTH=2026-08` produces a dated, hashed, versioned release; and
`make backtest` produces a chart proving the nowcast beats carry-forward, with the error shrinking
monotonically as the year fills in.

---

## 2. Scope

**In:** annual residential average revenue per kWh, per utility-state. Bridge-equation nowcast.
Pseudo-real-time backtest. Full release machinery.

**Out:** monthly-frequency nowcasts of anything other than the annual target. State-space or MIDAS
models unless the bridge equation demonstrably underperforms, in which case document why before
escalating complexity.

---

## 3. Data contracts

| id | Source | Lag | Role |
|---|---|---|---|
| `eia_861` | Annual, from PRD 06 | 9 to 12 mo | Target |
| `eia_861m` | EIA-861M monthly | ~2 mo | Primary indicator |
| `eia_elec_price` | EIA monthly state average retail price (API v2, series `ELEC.PRICE`) | ~2 mo | Secondary indicator |
| `bls_cpi_elec` | BLS CPI electricity index, series `CUUR0000SEHF01` | 2 to 6 wk | Fastest indicator |

**Vintage requirement.** Every ingest writes to `data/raw/<source>/<fetch_date>/`, never overwriting.
The backtest in M2 is only valid if vintages are available. If historical vintages cannot be
obtained (BLS and EIA do not serve arbitrary past vintages), state this limitation explicitly in
`FINDINGS.md` and note which direction it biases the backtest (optimistically, since revised data is
cleaner than what was available in real time).

---

## 4. Milestones

### M0. Indicator ingest
`ingest/eia861m.py`, `ingest/eia_price.py`, `ingest/bls.py`. Each with a pandera schema and a
recorded publication lag in `sources.yaml`.
**Accept:** three indicator series land in `data/interim/indicators.parquet`, monthly, per state,
with an explicit `published_at` column derived from the source's stated release schedule.

### M1. Bridge model
`model/nowcast.py`.

```python
def fit_bridge(target: pd.DataFrame, indicators: pd.DataFrame) -> BridgeModel: ...
def nowcast(model: BridgeModel, as_of: date) -> pd.DataFrame:
    """Returns utility_id, state, year, point, lower, upper, n_months_observed."""
```

Specification: regress annual target on (a) year-to-date mean of the EIA-861M implied rate, (b)
year-over-year change in the CPI electricity index, (c) prior-year target level, with a state fixed
effect. Estimate on all complete years. Prediction interval from the backtest error distribution,
conditioned on `n_months_observed`.
**Accept:** model fits, coefficients have the expected sign (prior-year level positive, CPI change
positive), and `nowcast()` returns one row per utility-state with a non-degenerate interval.

### M2. Pseudo-real-time backtest  **← Opus. This is the milestone that matters.**
`validate/backtest.py`.

For each month `m` from January of year Y-4 to the present:
1. Truncate every indicator series to what would have been published by `m`, using the recorded
   publication lags. **No exceptions, no peeking.**
2. Truncate the target series to years whose EIA-861 would have been published by `m`.
3. Refit the model on that truncated history. Refitting, not just re-predicting: using coefficients
   estimated on the full sample is a subtle and common lookahead leak.
4. Nowcast the current year. Store the prediction.

Compare against two baselines: last published annual value carried forward, and carried forward
escalated by national CPI-electricity.

**Accept, all four:**
- Error decreases monotonically (allowing minor noise) as `n_months_observed` rises. If it does not,
  the model is unstable; report it rather than tuning until the curve looks right.
- MAE beats plain carry-forward by a margin you state numerically.
- A test asserts that no input row with `published_at > as_of` was used. Implement this as a runtime
  guard in the backtest loop, not just a code review claim.
- `outputs/figures/backtest_error_by_month.png` exists and is legible.

### M3. Production hardening
1. **Schema validation on ingest.** Every ingest module wrapped in pandera. Add
   `tests/fixtures/eia861_corrupted.xlsx` with a renamed column; assert the run aborts with a
   non-zero exit code and a clear message.
2. **Source whitelist enforcement.** `ingest.fetch()` raises on any host absent from `sources.yaml`.
   Test with a disallowed URL.
3. **Idempotency.** Running `make build` twice produces byte-identical outputs. Test by hashing.
4. **Manifest.** `outputs/releases/<YYYY-MM>/manifest.json` recording input file hashes, git commit
   SHA, package versions, and row counts at each stage.
5. **Immutable releases.** `make release MONTH=...` refuses to overwrite an existing release
   directory.

**Accept:** all five have a passing test.

### M4. Docs and distribution
`METHODOLOGY.md` covering the bridge specification, the vintage limitation, and the apportionment
choices inherited from PRD 06. Auto-generate `DATA_DICTIONARY.md` from the pandera schemas so it
cannot drift. `CHANGELOG.md` with the first release entry. GitHub Actions workflow that runs the
full pipeline monthly on a schedule and opens a PR with the new release.
**Accept:** data dictionary is generated, not hand-written. Scheduled workflow file exists and is
syntactically valid.

---

## 5. Golden-number regression tests

- Backtest MAE at 6 months observed, tolerance 5% relative.
- Improvement over carry-forward in percentage points, tolerance 1pp.
- Release manifest contains a hash for every declared source.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Lookahead bias** | Backtest error suspiciously low, or flat across months observed | Runtime guard asserting `published_at <= as_of` on every row entering the fit. This is the primary correctness risk in the whole PRD. |
| Historical vintages unavailable | Cannot reconstruct real-time data | Document the limitation and its optimistic bias in FINDINGS.md. Do not pretend it is not there. |
| EIA-861M sample differs from EIA-861 universe | Systematic bias in the bridge | Include a level-shift term; measure and report the residual bias. |
| EIA restructures its workbooks | Pipeline breaks or, worse, runs green with wrong columns | Schema validation with a corrupted-file test. This is why M3.1 exists. |
| Nowcast does not beat carry-forward | | Report it. A negative result honestly reported is a valid outcome and a better interview answer than a tuned positive one. |

---

## 7. Interview artifact

The single figure to produce: backtest error versus months of the year observed, with the nowcast
line falling below both baselines and converging toward zero. That chart is the entire claim.
