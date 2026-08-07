# Packet 07: Nowcasting and Production Pipelines

> **Résumé line:** "Engineered forward projections to close a reporting lag that ran up to a year,
> then productionized the pipeline into a monthly release with documented methodology, source
> whitelisting, and distribution to internal users."

**Repo:** `utility-affordability` (same repo as Packet 06) | **Est. 20 hours** | **Depends on P6.
Second-highest leverage: 17 of 20 résumés.**

---

## 1. The idea

Two halves, and interviewers care about them differently.

The **nowcast** half is a forecasting problem with an unusual target: you are predicting the
*present*, because the official measurement of the present will not be published for another year.
Economists do this constantly and call it nowcasting. Energy analysts mostly just shrug and use stale
data.

The **productionization** half is engineering. A notebook that ran once is a document. A pipeline
that runs monthly, validates its inputs, fails loudly, versions its outputs, and documents its
methodology is a product other people can depend on. Most analyst candidates have only ever built the
first kind. The word "productionized" is what job descriptions are reaching for when they ask for
production Python.

---

## 2. What you need to understand

**Why the lag exists.** EIA-861 is an annual form. Utilities file after their fiscal year closes,
EIA processes and validates, and publication lands 9 to 12 months after the reference year ends. So
in August 2026 the newest final annual data may reference 2024. If you want to say anything about what
households are paying now, official data cannot help you.

**The bridge.** EIA-861M is monthly, published with roughly a two-month lag, but covers a sample
rather than the full universe and carries less detail. Other faster indicators: EIA's monthly state
average retail price series, BLS CPI for electricity (published within weeks), state PUC rate case
decisions (available the day they are issued), and utility 10-Q filings. The nowcast problem is
combining a slow, complete, authoritative series with fast, partial, noisy ones.

**Ragged edge.** Different inputs have different lags, so at any moment your input matrix has a
staircase of missing values at the bottom. This is the defining feature of nowcasting and it is why
you cannot just run a regular regression. Standard treatments: a state-space model with a Kalman
filter that naturally handles missing observations, MIDAS regression for mixed frequencies, or a
simpler and perfectly defensible bridge equation where you regress the annual series on aggregated
monthly indicators.

**Start simple.** A bridge equation with two or three indicators, backtested honestly, beats a
Kalman filter you cannot explain. Add sophistication only if the backtest says it helps.

**Vintage discipline.** This is the part people get wrong. To backtest a nowcast honestly you must
know what data *was available on the date you are pretending to stand on*, not what the series looks
like today after revisions. If you backtest a 2023 nowcast using the current, revised 2023 monthly
data, you have cheated. Store input vintages, or at minimum note where revisions could contaminate
the backtest. Being able to articulate this is a strong signal.

**Source whitelisting.** In a regulated bank you cannot pull data from arbitrary URLs into a
production process. Each source is approved, documented, and pinned. Reproduce that discipline: a
`sources.yaml` listing every source with its URL, licence, update cadence, expected schema, and an
approval note. Any fetch from a source not in the file fails the run. This costs an hour and it is
exactly the control an interviewer at a regulated firm is listening for.

---

## 3. Method: the nowcast

1. **Establish the target and the timing.** Target: annual residential average revenue per kWh per
   utility-state. Timing: at month `t`, what is the estimate for the calendar year containing `t`?
2. **Assemble indicators** with their true publication lags recorded: EIA-861M monthly revenue and
   sales, EIA state average price, BLS CPI electricity index, approved rate case outcomes.
3. **Bridge equation.** Regress the annual target on the year-to-date aggregate of the monthly
   indicators plus a state fixed effect plus the prior year's level. Estimate on all complete years.
4. **Backtest by pseudo-real-time replay.** For each historical month, truncate every input series to
   what would have been published by that date, run the nowcast, and compare to the eventual final
   value. Plot error against how far into the year you are. The error should fall monotonically as
   the year fills in. If it does not, your model is unstable.
5. **Compare against the naive alternatives:** carry forward the last published annual value, and
   carry forward with a national CPI-electricity escalation. Your model must beat both or it is not
   worth running. Publish that comparison.
6. **Publish uncertainty, not just a point.** Report the backtest error distribution as a prediction
   interval on every nowcast value. A nowcast without an interval invites false confidence.

---

## 4. Method: productionization

Build these seven things. Each is small; together they are the difference between a script and a
product.

1. **Deterministic entry point.** `make monthly` or `python -m pipeline run --month 2026-08`. One
   command, no manual steps, no notebook cells run out of order.
2. **Schema validation on ingest.** Use `pandera` or `pydantic`. Assert column names, dtypes, ranges,
   uniqueness of keys, and expected row-count bounds. EIA changes its Excel layouts between years, so
   the pipeline must fail loudly on schema drift rather than silently producing garbage. Write a test
   that feeds a deliberately corrupted file and asserts the run aborts.
3. **Source whitelist.** `sources.yaml` as described above, enforced in the fetch layer.
4. **Idempotency and caching.** Re-running the same month produces byte-identical output and does not
   re-download unchanged sources. Hash the raw inputs and store the hash with the output.
5. **Versioned, dated releases.** `releases/2026-08/` containing the data, a `manifest.json` with
   input hashes and code version, and a changelog entry. Never overwrite a prior release. If a number
   changes, it changes in a new release with an explanation.
6. **Data dictionary and methodology note.** Every column defined, every unit stated, every
   assumption listed. Generate the dictionary from the schema definitions so it cannot drift.
7. **Distribution.** In the bank this was internal users. Publicly: publish the release to GitHub
   Releases, plus a small static site or a `datasette` instance. Add a `CHANGELOG.md` that a
   downstream user can subscribe to.

**Add regression tests on the headline numbers.** A test that asserts national median energy burden
is within a tolerance of the last release, and fails the build if it moves more than a threshold
without a changelog entry. This catches silent upstream data changes, which is the most common way
production data products go wrong.

---

## 5. Numbers to know cold

- EIA-861 annual lag: **9 to 12 months**. EIA-861M lag: roughly **2 months**.
- BLS CPI for electricity: published **mid-month for the prior month**, so roughly a 2 to 6 week lag.
- Your nowcast should beat carry-forward. Quantify by how much and at which point in the year.

---

## 6. Interview questions and how to answer

**"What does productionized mean to you?"**
Have a crisp list ready, not a vibe. "One command runs it. Inputs are schema-validated and fail
loudly. Sources are pinned and whitelisted. Outputs are versioned and never overwritten, with a
manifest of input hashes so any number can be traced to the exact inputs that produced it. There is a
regression test on the headline figures so an upstream change cannot silently move them. And there is
a methodology document that a user who was not in the room can read."

**"How did you validate a forecast of something you cannot observe yet?"**
Pseudo-real-time replay: truncate every input to what was actually published as of the historical
date, run the model, compare to the eventual final value, and plot error against how much of the year
had elapsed. And I compared against the naive carry-forward baseline, because if you cannot beat
carry-forward the whole exercise is theatre.

**"What breaks a monthly pipeline?"**
Upstream schema drift, silently. EIA restructures its Excel workbooks between vintages, adds columns,
renames fields. Without ingest validation you get a pipeline that runs green and produces wrong
numbers, which is far worse than one that crashes. That is why validation fails the run rather than
warning.

**"Why does source whitelisting matter?"**
Because in a regulated environment provenance is a control, not a nicety. Every number that reaches a
client or a disclosure has to be traceable to an approved source, and "I found it on a website" does
not survive audit. It also happens to be good engineering, because a pinned source is a reproducible
source.

---

## 7. Further reading

- Bańbura, Giannone and Reichlin, "Nowcasting," Oxford Handbook of Economic Forecasting, 2011. The
  standard survey.
- Giannone, Reichlin and Small, "Nowcasting: The real-time informational content of macroeconomic
  data," JME 2008.
- Federal Reserve Bank of New York Staff Nowcast methodology documentation, for a worked real-world
  example with a ragged edge.
- `pandera` and `dbt` docs on data testing patterns, for the productionization half.
