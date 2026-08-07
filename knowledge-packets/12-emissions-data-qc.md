# Packet 12: Emissions Data Quality Control and Disclosure Tooling

> **Résumé line:** "Developed data-driven tools analysing climate impact, exposure, and alignment of
> corporate lending portfolios, and ran quality control on external emissions data feeding Firmwide
> disclosures and EU regulatory reporting."

**Repo:** `emissions-qc` | **Est. 20 hours** | **Tier A. On 6 résumés. Pairs naturally with Packet 11.**

---

## 1. The idea

A bank buys emissions data from vendors, feeds it into regulatory disclosures, and is legally
accountable for what comes out. The vendor data is wrong in specific, detectable ways. The QC layer
is the thing between the vendor feed and the disclosure.

Build the detector: a tool that ingests emissions data for a set of companies from multiple public
sources and flags every discrepancy, implausibility, and unexplained restatement.

This is a smaller, sharper project than most in the plan, and it maps directly onto what regulated
disclosure work actually feels like day to day.

---

## 2. What goes wrong in emissions data, specifically

These are the checks. Each is a detectable pattern.

1. **Unit errors.** tCO2e vs ktCO2e vs MtCO2e. Off-by-1000 errors are common and produce values that
   are individually plausible but wrong. Detect via sector-relative outlier screening.
2. **Scope confusion.** A vendor reports a scope 1 figure that is actually scope 1+2 combined, or a
   market-based scope 2 in a location-based field. Detect by comparing the ratio of scope 2 to scope
   1 against sector norms and by cross-source comparison.
3. **Boundary changes.** The company divested a division, so this year's figure is not comparable to
   last year's. Look for large step changes without a corresponding revenue step change.
4. **Silent restatements.** The vendor changes a prior-year value without flagging it. This is the
   most damaging one for disclosure, because your published prior-year figure no longer matches your
   source. Detect by snapshotting every vintage and diffing.
5. **Estimated values presented as reported.** Some vendors fill gaps with models and do not always
   flag which is which. Cross-check against a source you know is reported (GHGRP, the company's own
   sustainability report).
6. **Stale carry-forward.** The same value repeated across years because the vendor has not updated.
   Detect by looking for identical values in consecutive periods.
7. **Intensity-vs-absolute confusion.** A figure that is actually tCO2e per million of revenue
   placed in an absolute field.
8. **Coverage gaps that move.** Company enters and leaves the dataset between vintages, so a
   portfolio total changes for compositional rather than real reasons. Always decompose year-on-year
   change into real change, coverage change, and methodology change.
9. **Currency and fiscal year misalignment.** Emissions on a fiscal year, financials on a calendar
   year, revenue in the wrong currency for an intensity denominator.

---

## 3. Data

Use three independent public sources for the same companies so that cross-source comparison is
possible:

| Source | Nature |
|--------|--------|
| **EPA GHGRP** | Regulator-collected, facility level, US only, high confidence |
| **Company sustainability reports / 10-K climate disclosure** | Self-reported, parsed from PDFs or from SEC filings |
| **Climate TRACE** | Independently estimated from sensors and satellites |
| **CDP (free subset) or EU CSRD filings** | Self-reported to a different framework |

The disagreement between GHGRP (regulator, facility, US) and self-reported global figures is itself
informative: a US-heavy company should roughly reconcile; a global one should not, and the gap should
be explainable by international operations.

**On EU regulatory reporting:** the relevant frameworks are **CSRD/ESRS** (corporate sustainability
reporting), **SFDR** (fund-level disclosure, with the Principal Adverse Impact indicators including
GHG intensity), the **EU Taxonomy** (alignment percentages), and **Pillar 3 ESG** for banks. Know
what each one asks for, because "EU regulatory reporting" on a résumé invites the question of which.

---

## 4. Method

1. **Vintage-aware storage.** Every ingest is stored as a dated snapshot, never overwritten. This is
   the precondition for detecting silent restatements, and it is the same discipline as Packet 07.
2. **Rule engine.** Each check from section 2 as an independent rule returning findings with
   severity, entity, field, and an explanation. Same `Finding` pattern as Packet 10.
3. **Sector-relative outlier screening.** Emissions intensity (tCO2e per unit revenue) by sector has
   a known distribution. Flag values beyond a robust z-score threshold using median and MAD rather
   than mean and standard deviation, because the distribution has heavy tails.
4. **Reconciliation report.** For each company, a side-by-side of every source with the delta and a
   plausibility verdict.
5. **Year-on-year decomposition.** Split portfolio change into real, coverage, and methodology
   components. This is the single most useful output for anyone who has to explain a disclosure
   movement to a regulator or a committee.
6. **Sign-off workflow.** Findings must be dispositioned: accepted, corrected, or waived with a
   reason. Nothing reaches the output unless every error-severity finding is dispositioned. That is
   the control, and it is the thing to describe in an interview.

---

## 5. Interview questions and how to answer

**"What kinds of errors did you find in vendor emissions data?"**
Give three concrete ones with mechanisms, not a general statement about data quality. Unit errors of
a factor of 1000, scope 2 reported on the wrong basis, and silent restatements of prior years. The
third is the one that hurts, because your published figure and your source no longer agree and you
cannot reconstruct why without vintage snapshots.

**"How do you catch a silent restatement?"**
You cannot, unless you kept the old vintage. Snapshot every ingest, diff every field across vintages,
and treat any change to a historical value as a finding requiring disposition.

**"What is the difference between location-based and market-based scope 2?"**
Location-based uses the grid average factor where consumption occurs. Market-based reflects
contractual instruments, PPAs and RECs. Companies typically report both, and the market-based number
is usually much lower. Mixing the two across a portfolio makes the total meaningless.

**"How did you decompose year-on-year change?"**
Real change, coverage change, methodology change, held separately. Without that split, a disclosure
movement is uninterpretable, and the first question any reviewer asks is whether the number moved
because the world changed or because the data did.

---

## 6. Further reading

- ESRS E1 (climate) of the CSRD, and the SFDR Principal Adverse Impact indicator list.
- GHG Protocol Scope 2 Guidance, for the location vs market distinction.
- Busch et al., "Corporate carbon and financial data: a critical review," for the empirical work on
  vendor disagreement.
