# PRD 13: Financial-Data MCP Server

| | |
|---|---|
| **Repo** | `edgar-mcp` |
| **Bullets covered** | `kensho` (5 of 20 résumés), supports `ambassador` (14) |
| **Packet** | [`13-financial-data-mcp.md`](../knowledge-packets/13-financial-data-mcp.md) |
| **Est.** | 15 human-hours, 3 to 5 agent session hours |
| **Priority** | 12 of 17. Small. Check your existing 23-company research platform first; much of this may exist. |
| **Model** | Sonnet throughout. |
| **Prereqs** | Read `00-CONVENTIONS.md`. Build PRD 10 first; MCP patterns transfer. |

---

## 1. Objective

An MCP server exposing SEC EDGAR XBRL financial data to a language model, with provenance on every
returned value, normalized concepts across tag heterogeneity, and no silent interpolation.

**Success test:** in Claude Code, ask "compare operating cash flow for DUK, SO, and AEP over the last
five fiscal years" and get a correct table where every figure carries an accession number.

---

## 2. Scope

**In:** US registrants filing XBRL. `companyfacts`, `companyconcept`, `frames`, and `submissions`
endpoints. Concept normalization. Statement assembly. Filing-section retrieval. Disk cache.

**Out:** International filers without XBRL. Real-time market data (no free reliable source, and it is
a different problem). Analyst estimates. Anything requiring a paid licence.

---

## 3. Data contracts

| id | Endpoint | Notes |
|---|---|---|
| `sec_tickers` | `sec.gov/files/company_tickers.json` | Ticker to CIK |
| `sec_companyfacts` | `data.sec.gov/api/xbrl/companyfacts/CIK##########.json` | Every fact ever reported. Can be several MB per company. |
| `sec_companyconcept` | `data.sec.gov/api/xbrl/companyconcept/CIK##########/us-gaap/{tag}.json` | One concept over time |
| `sec_frames` | `data.sec.gov/api/xbrl/frames/us-gaap/{tag}/USD/CY2025Q1I.json` | Cross-sectional, all companies. Underused; the cheap path to peer comparison. |
| `sec_submissions` | `data.sec.gov/submissions/CIK##########.json` | Filing history, `formerNames` |

**Etiquette, enforced in code:** descriptive `User-Agent` with a contact email from `.env`, a token
bucket limiting to 10 requests/second, exponential backoff on 429 and 403, and a disk cache keyed by
CIK plus latest filing date.

---

## 4. Milestones

### M0. Scaffold and client
Repo per conventions. `edgar/client.py` with rate limiting, retry, and disk cache.
**Accept:** a test asserts the rate limiter blocks an 11th request within one second. A test asserts
a missing `SEC_USER_AGENT` env var raises a clear error rather than sending a default UA.

### M1. Concept resolver
`edgar/concepts.py`. A YAML-driven priority list per normalized concept.

```yaml
concepts:
  revenue:
    tags:
      - RevenueFromContractWithCustomerExcludingAssessedTax
      - Revenues
      - SalesRevenueNet
      - RevenueFromContractWithCustomerIncludingAssessedTax
    type: duration
    unit: USD
  total_assets:
    tags: [Assets]
    type: instant
    unit: USD
```

Resolution returns the value **and the tag that produced it**.
**Accept:** for a test set of 30 companies across 6 sectors, the resolver returns a non-null revenue
for at least 28. Report the failures by ticker rather than silently returning null. A test asserts
that a company using a non-standard tag still resolves.

### M2. Fact selection semantics
Handle the three things that make this non-trivial:
1. **Duplicates and restatements.** Same period, multiple `accn`. Expose a `basis` parameter:
   `latest_filed` (default) or `as_originally_reported`.
2. **Instant vs duration.** Enforce per concept from the YAML `type`. Mixing them raises.
3. **Fiscal alignment.** Normalize on actual `start`/`end` dates, not the filer's `fy`/`fp` labels,
   so non-December year ends align correctly in peer comparisons.
**Accept:** a test with a company that restated a prior year returns different values under the two
bases. A test with a non-December fiscal year end aligns correctly against a December filer.

### M3. MCP tools
| Tool | Signature | Returns |
|---|---|---|
| `resolve_company` | `(query: str)` | CIK, ticker, name, former names |
| `get_concept` | `(ticker, concept, periods, basis)` | Series with value, period, unit, **tag used, accession, filed date** |
| `get_financials` | `(ticker, statement, periods, basis)` | Normalized statement |
| `compare_peers` | `(tickers, concept, period, basis)` | Cross-sectional table |
| `list_filings` | `(ticker, form, since)` | Filing index with URLs |
| `get_filing_section` | `(accession, item)` | Text of a 10-K item, e.g. `1A` |

**Accept:** every returned numeric value carries `tag`, `accn`, and `filed`. A test asserts no
response path can return a value without provenance. **Missing data returns null with a `reason`
field; never interpolated, never estimated.** Test this explicitly.

### M4. Integration and demo
Config snippets for Claude Code and Claude Desktop in the README. A recorded demo transcript showing
a multi-step peer comparison with citations.
**Accept:** the server starts from a clean checkout following only the README, and the demo query
returns correct figures spot-checked against the actual filings.

### M5. Monitoring job (the product layer)
A scheduled job over a watchlist (start with your existing 23 companies) that detects new filings,
pulls tracked concepts, computes period-over-period changes, and drafts a summary. Output to Markdown.
**Accept:** runs on a schedule, produces a dated digest, and every figure in the digest carries an
accession number. This milestone is what turns plumbing into the claim on the résumé.

---

## 5. Golden-number regression tests

- Revenue resolution rate across the 30-company test set, tolerance 1 company.
- A known value for a known company and period, exact against the filing.
- Provenance completeness: 100% of numeric returns carry tag, accn, and filed date.

---

## 6. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **Tag heterogeneity silently returns nulls** | Sparse peer tables | Priority list per concept, resolution rate measured in M1, failures reported by name. |
| SEC blocks the IP | 403s | Enforced UA and rate limit in M0, tested. |
| Model fabricates a number the tool could not return | Wrong figures presented confidently | Nulls with a `reason` field, provenance mandatory, README instructs that the model must cite `accn` for any figure. |
| `companyfacts` payload size | Slow, memory heavy | Prefer `companyconcept` for single-concept queries; cache `companyfacts` to disk when fetched. |
| Restatement confusion | Numbers change between runs | Explicit `basis` parameter, defaulted and documented. |

---

## 7. Reuse check before starting

Inspect the existing 23-company AI equity research platform first. If it already has an EDGAR client,
a concept resolver, or a monitoring loop, extend it and wire MCP on top rather than rebuilding. Report
what was reused.
