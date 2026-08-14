# EDGAR MCP drills

## 1. Why is there no universal revenue field?

Companies choose different valid XBRL tags based on their disclosures and
industry.

## 2. Why can the first tag with data be wrong?

A company may have abandoned that tag years ago while old facts remain in the
API.

## 3. Why filter annual facts inside the tag ladder?

The first tag may contain only quarterly facts. Filtering later would return
missing instead of trying the next tag with valid annual data.

## 4. What is the difference between instant and duration facts?

An instant fact is measured on one date. A duration fact covers a start and end
date.

## 5. What is the difference between restatement basis and filing cutoff?

Basis chooses among available filings for one period. The cutoff decides which
filings were available at all by a historical date.

## 6. Why return null for missing capex?

A smaller standard tag could look plausible while materially understating the
company's construction spending.

## 7. What makes a value traceable?

Tag, unit, period, accession number, filing date, and direct filing URL.
