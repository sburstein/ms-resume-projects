# Packet 08: Utility to Publicly Traded Parent Crosswalk

> **Résumé line:** "Mapped utility service territories to publicly traded parent companies, making a
> physical and regulatory dataset usable for company-level analysis."

**Repo:** `utility-affordability` (same repo as Packets 06 and 07) | **Est. 12 hours** |
**Depends on P6. On 13 of 20 résumés, and the cheapest bullet to cover.**

---

## 1. The idea

You have a geographic and regulatory dataset. An investor cannot use it, because the entity that
files with EIA is almost never the entity that trades on an exchange. Kentucky Power files with EIA;
American Electric Power trades on Nasdaq. Southern California Edison files; Edison International
trades. Pacific Gas and Electric Company files; PG&E Corporation trades.

The crosswalk is the step that turns a physical dataset into a financial one. It is not intellectually
deep, and that is fine: its value is that it is fiddly, nobody publishes it cleanly, and having built
it is why your dataset can answer an investment question.

---

## 2. What you need to understand

**Holding company structure.** After PUHCA-era restructuring and the 2005 repeal, the dominant form
is a holding company that owns several regulated operating utilities in different states, plus
usually an unregulated generation or services arm. Exelon owns ComEd, PECO, BGE, Pepco, DPL, and ACE.
Dominion, Duke, AEP, and Xcel are all structured the same way. So:

- One ticker maps to many EIA utility IDs.
- The regulated subsidiaries are the ones whose rates a state PUC sets, and whose earnings are the
  stable part of the parent's story.
- The subsidiary is the credit issuer in many cases too, which is why first-mortgage bonds are issued
  at the opco level while the holdco issues unsecured debt. Worth knowing if you talk to credit people.

**Why it is messy:**
- **Mergers and renames.** Utilities are acquired constantly. A crosswalk built in 2020 is wrong by
  2026. Any mapping needs an effective-date dimension, not a single snapshot.
- **Joint ownership.** Some operating utilities are jointly owned. Some plants are jointly owned by
  utilities with different parents.
- **Take-privates.** Several large IOUs are now owned by infrastructure funds or foreign utilities and
  have no US-listed ticker: Oncor (majority Sempra), Puget Sound Energy (Macquarie-led consortium),
  Cleco, El Paso Electric (IIF), Duquesne Light. Your mapping must have a "no listed parent" category,
  and the size of that category is itself a finding.
- **Foreign parents.** National Grid, Iberdrola (Avangrid), Algonquin, Fortis, Emera. Some have ADRs,
  some list only abroad.
- **Name matching is hard.** "Consolidated Edison Co of New York Inc" vs "Consolidated Edison, Inc."
  are a subsidiary and its parent, and the string distance between them is small. Naive fuzzy
  matching will confidently produce wrong answers here.

---

## 3. Data

| Source | What | Access | Gotcha |
|--------|------|--------|--------|
| **EIA-861** ownership fields | Utility ID, name, ownership type | Free | Does not give the parent holding company reliably |
| **EIA-860** | Plant-level operator and owner, including percent ownership | Free | The owner fields are the best public hint at corporate structure |
| **SEC EDGAR `company_tickers.json`** | CIK to ticker to name for every registrant | Free, `sec.gov/files/company_tickers.json` | Includes subsidiaries that file separately, which is useful: many opcos are themselves SEC registrants because they issue public debt |
| **SEC EDGAR full-text search / submissions API** | Filings, former names, business address | Free JSON API | Former-name history is how you catch renames |
| **SEC Exhibit 21** | "Subsidiaries of the Registrant," filed with every 10-K | Free, in the filing index | **This is the key source.** It literally lists each parent's subsidiaries. Parse it. |
| **EIA-861 Service Territory** | Counties served | Free | Lets you sanity check that a mapped parent's footprint matches the geography |
| **LEI / GLEIF** | Legal entity identifiers with parent relationships | Free bulk download | Contains reported direct and ultimate parent relationships. Underused and genuinely helpful here. |

---

## 4. Method, in steps

1. **Start from Exhibit 21, not from fuzzy matching.** Pull the 10-K Exhibit 21 for every US-listed
   utility holding company (start from the S&P 500 Utilities sector plus the S&P Utilities Select
   universe plus known mid-caps). Parse the subsidiary lists. This gives you a high-confidence
   parent-to-subsidiary edge list straight from a legal filing.
2. **Normalize names on both sides.** Strip corporate suffixes (Inc, Co, Corp, LLC, LP, Company),
   expand abbreviations (Elec to Electric, Pwr to Power, Co-op to Cooperative), uppercase, collapse
   whitespace and punctuation. Keep the raw name alongside the normalized one always.
3. **Match EIA utility names to the Exhibit 21 subsidiary list.** Use token-set ratio (RapidFuzz)
   rather than plain Levenshtein, because word order varies. Accept high-confidence matches
   automatically, queue the middle band for review.
4. **Add state as a blocking key.** Only compare EIA utilities against subsidiaries plausibly
   operating in that state. This kills most false positives at essentially no cost.
5. **Hand-verify the top 50 by customer count.** The distribution is heavily skewed: the largest 50
   IOUs cover the large majority of IOU customers. An hour of manual verification on those 50 buys
   more accuracy than any amount of algorithm tuning on the tail.
6. **Record a confidence tier on every row:** `exhibit21_exact`, `fuzzy_high`, `fuzzy_reviewed`,
   `manual`, `no_listed_parent`, `unmatched`. Never collapse these into a single boolean.
7. **Add effective dates.** `valid_from` and `valid_to` per mapping, so a historical analysis uses
   the ownership structure that existed at the time. Populate from merger announcement and close
   dates.
8. **Publish coverage honestly.** The headline stat should be "mapped X% of IOU residential customers
   to a listed parent; Y% are held by private or foreign owners with no US listing; Z% unmatched."
   Publishing the unmatched list is a feature, not an admission.
9. **Join back to Packet 06.** Now you can produce parent-company-level exposure: weighted average
   energy burden across each parent's territories, share of customers in high-burden tracts,
   regulatory-risk proxies. That aggregation is the actual deliverable, not the crosswalk itself.

---

## 5. Why the aggregation matters commercially

Energy burden is a regulatory risk signal. A utility whose customers spend an unusually high share of
income on power faces harder rate cases, more political attention, higher bad-debt and disconnection
rates, and more pressure for bill assistance programs that shift costs. Rolled up to the parent, that
becomes a comparable metric across listed names, which is something an equity or credit analyst can
actually use.

Say this in interviews. It is what converts "I did a data engineering task" into "I built a signal."

---

## 6. Numbers to know cold

- Roughly **170 to 190** IOUs, held by roughly **45 to 55** parent groups.
- The **largest 50** IOUs cover the large majority of IOU customers, so verification effort should
  be weighted accordingly.
- Notable no-listed-parent cases: **Oncor, Puget Sound Energy, Cleco, El Paso Electric, Duquesne
  Light**.
- Notable foreign parents: **Avangrid (Iberdrola), National Grid, Fortis, Emera, Algonquin**.
- Exhibit 21 is filed as part of every **10-K**.

---

## 7. Interview questions and how to answer

**"How did you build the mapping?"**
Lead with Exhibit 21, because it shows you went to the legal source rather than reaching for fuzzy
matching first. "Every 10-K includes Exhibit 21, a list of subsidiaries of the registrant. I parsed
those to get a high-confidence edge list from a filed document, then used fuzzy matching only to
join EIA's utility names onto that list, blocked by state, with a confidence tier on every row and
manual verification on the largest fifty by customer count."

**"What is your error rate?"**
Have a real answer. "Verified exact on the top fifty by customers, which covers most of the IOU
customer base. Below that I report a confidence tier per row and publish the unmatched list. I would
rather ship a mapping with a stated coverage of, say, 94% of customers than one that claims 100% and
quietly guesses."

**"What happens when a utility gets acquired?"**
The mapping is dated, with `valid_from` and `valid_to`, so a historical analysis uses the structure
that existed at the time. A single-snapshot crosswalk silently corrupts any time series that spans a
merger.

**"Why does anyone care?"**
Give the section 5 answer. Do not describe the crosswalk as the product.

---

## 8. Further reading

- SEC EDGAR full-text search, and any utility 10-K Exhibit 21. Read three.
- EEI (Edison Electric Institute) member list and their annual financial review, for the parent
  universe.
- S&P Global Market Intelligence's public write-ups on utility M&A, for the merger history.
