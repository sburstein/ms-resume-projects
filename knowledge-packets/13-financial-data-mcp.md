# Packet 13: Financial-Data MCP Server (the Kensho / Capital IQ analogue)

> **Résumé line:** "Built an Excel health-check agent for the reporting suite and a Kensho API
> integration linking Claude directly to S&P Capital IQ financial data."

**Repo:** `edgar-mcp` (the Excel half lives in Packet 10) | **Est. 15 hours** | **Tier B: public
substitute. On 5 résumés.**

---

## 1. The idea

Kensho and Capital IQ are licensed and inaccessible outside the firm. The architecture is what
matters, and it transfers exactly: a model that can query structured financial data directly instead
of a human clicking through a terminal and pasting into a spreadsheet.

Build the same thing over **SEC EDGAR's XBRL API**, which is free, requires only a descriptive
user-agent header, and contains every financial fact every US registrant has reported.

You likely have most of this already in the 23-company AI equity research platform on your résumé.
Wire it to MCP, document it, and it covers this bullet.

---

## 2. What you need to understand

**XBRL and the `companyfacts` API.** Every 10-K and 10-Q is filed with machine-readable XBRL tags.
SEC exposes it at:

- `data.sec.gov/api/xbrl/companyfacts/CIK##########.json` for every fact a company has ever reported.
- `data.sec.gov/api/xbrl/companyconcept/CIK##########/us-gaap/Revenues.json` for one concept over
  time.
- `data.sec.gov/api/xbrl/frames/us-gaap/Assets/USD/CY2025Q1I.json` for the same concept across every
  company in a period. This one is underused and it is how you build peer comparisons cheaply.
- `sec.gov/files/company_tickers.json` maps ticker to CIK.

**The gotchas that make this a real project rather than an API wrapper:**

1. **Tag inconsistency.** Companies choose different us-gaap tags for the same economic concept.
   Revenue may be `Revenues`, `RevenueFromContractWithCustomerExcludingAssessedTax`,
   `SalesRevenueNet`, or a company-specific extension tag. A naive fetch of `Revenues` returns
   nothing for a large share of registrants. Build a concept resolver with a priority list per
   concept and record which tag was used.
2. **Duplicate and restated facts.** The same period appears multiple times with different `accn`
   (accession numbers) and `fy`/`fp` values as it is re-reported in later filings. You must pick a
   convention: latest filed wins, or as-originally-reported. Both are legitimate and they answer
   different questions. Expose it as a parameter.
3. **Instant vs duration facts.** Balance sheet items are instantaneous (`end` only); income and cash
   flow items are durations (`start` and `end`). Mixing them silently produces nonsense.
4. **Fiscal year alignment.** `fy` and `fp` fields are the filer's fiscal labels, not calendar
   periods. Companies with non-December year ends will misalign in a peer comparison unless you
   normalize on the actual `end` date.
5. **Rate limits and etiquette.** SEC asks for a descriptive User-Agent with contact info and
   roughly 10 requests per second. Respect it, cache aggressively, and say in the README that you do.

---

## 3. Build

**MCP tools to expose:**

| Tool | Purpose |
|------|---------|
| `resolve_company(query)` | Ticker or name to CIK, with fuzzy fallback |
| `get_concept(ticker, concept, periods, basis)` | One normalized concept as a time series, with the resolved tag reported |
| `get_financials(ticker, statement, periods)` | A normalized income statement, balance sheet, or cash flow |
| `compare_peers(tickers, concept, period)` | Cross-sectional comparison, built on the frames endpoint |
| `list_filings(ticker, form, since)` | Filing index with URLs |
| `get_filing_section(accession, item)` | Pull a specific 10-K item, e.g. Item 1A risk factors |

**Design principles that make it good rather than adequate:**

- **Always return provenance.** Every value comes back with the tag used, the accession number, the
  filing date, and the form type. A model that can cite where a number came from is usable in
  regulated work; one that cannot is not.
- **Never silently interpolate.** If a period is missing, return null and say so. A financial data
  tool that fills gaps invisibly will produce a confident wrong answer.
- **Normalize but expose the raw.** Return the normalized concept and the underlying tag, so a user
  can audit the normalization.
- **Cache on disk** keyed by CIK and last-filing-date so repeated calls are free.

**Then build one thing on top of it** so it is a product rather than plumbing. The obvious candidate,
given your existing platform: a monitoring job over your 23 companies that pulls each new filing,
extracts the concepts you track, computes changes, and drafts a summary. That is the "linking Claude
directly to financial data" claim made concrete.

---

## 4. Interview questions and how to answer

**"What is hard about pulling financial data programmatically?"**
Not the HTTP call. Tag heterogeneity and restatement. Companies use different us-gaap tags and
company-specific extensions for the same concept, so a naive concept fetch silently returns nothing
for a large slice of the universe. And the same period is re-reported across filings, so you have to
choose between latest-filed and as-originally-reported, which are different answers to different
questions.

**"Why MCP rather than just a Python library?"**
Because the consumer is a model, not a script. MCP gives the model a typed tool surface with
descriptions it can reason about, so the analyst asks a question in English and the model composes
the calls. The library is still there underneath; MCP is the interface layer that makes it
conversational.

**"How do you stop the model from making numbers up?"**
Provenance on every returned value, nulls instead of interpolation, and a rule that the model must
cite the accession number for any figure it reports. If the tool cannot return a number, the correct
behavior is to say so, and that has to be designed in at the tool layer rather than requested in a
prompt.

---

## 5. Further reading

- SEC EDGAR APIs documentation, `sec.gov/search-filings/edgar-application-programming-interfaces`.
- XBRL US GAAP taxonomy, for the concept hierarchy.
- Model Context Protocol specification, `modelcontextprotocol.io`.
- See Packet 10 for the Excel health-check half of this bullet.
