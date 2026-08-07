# Packet 10: Workbook Health-Check Agent and AI Workflows

> **Résumé lines:** "Deployed AI-assisted workflows into external audit preparation with
> human-in-the-loop review, and built an agent that health-checks reporting workbooks for
> data-integrity issues." Also supports `ambassador` (14 résumés) and `kensho` (5).

**Repo:** `workbook-doctor` | **Est. 18 hours** | **Independent of everything else. Good filler
week, and the fastest project in the plan to finish.**

---

## 1. The idea

Large financial workbooks rot. Formulas get overwritten with hardcoded numbers, links to closed files
break, a copied row misses a column, a named range points at a deleted sheet. Errors propagate
silently and in a regulated report that is a material problem, because the number that reaches a
disclosure has to be traceable and correct.

Build a tool that opens an `.xlsx` and reports every integrity issue it can detect, exposes the same
capability as an MCP server so a language model can call it, and offers a fix mode that proposes
patches but never writes without confirmation.

This is the most demo-able project you have. It takes ninety seconds to show and the audience
immediately understands the value.

---

## 2. What to detect

Ordered by how often it bites in real workbooks:

1. **Error values.** `#REF!`, `#VALUE!`, `#DIV/0!`, `#N/A`, `#NAME?`, `#NULL!`, `#NUM!`. Report both
   the cell holding the error and, by walking dependencies, the *originating* cell. Reporting 400
   downstream `#REF!`s when one deleted column caused all of them is noise; reporting the root cause
   is the product.
2. **Hardcoded overrides in formula ranges.** A column of 200 formulas with three literal constants
   in the middle. This is the single most dangerous pattern in financial modeling because it is
   invisible and it survives recalculation. Detect by scanning contiguous ranges for formula
   consistency (relative R1C1 equivalence) and flagging cells that break the pattern.
3. **Inconsistent formulas.** Same as above, but the odd cell holds a *different formula* rather than
   a constant. Compare R1C1 representations across a range.
4. **External links.** Formulas referencing other workbooks, plus stale cached values from links that
   no longer resolve. Extract from `xl/externalLinks/` in the zip and from formula strings.
5. **Circular references.** Build the dependency graph and detect cycles. Excel permits them with
   iterative calculation enabled, which is almost always a mistake in a reporting workbook.
6. **Orphaned and broken named ranges.** Names pointing at `#REF!`, or defined and never used.
7. **Volatile functions.** `NOW()`, `TODAY()`, `RAND()`, `OFFSET()`, `INDIRECT()`. These make a
   workbook non-reproducible: the same file produces different numbers on different days. In a
   reporting pack that is a control failure. `INDIRECT` also defeats dependency tracing, which is
   worth flagging separately.
8. **Precision and type issues.** Numbers stored as text, mixed types in a column, dates stored as
   strings, values with more displayed precision than stored.
9. **Structural risks.** Merged cells inside data ranges, hidden rows or columns inside a summed
   range (a classic way to hide a number from a reviewer without hiding it from the total), very
   long formulas, deeply nested IFs, and sheets with no formulas at all where you would expect them.
10. **Cross-sheet consistency.** Totals on a summary sheet that do not match the sum of their source
    range. This requires evaluation, not just parsing, and it is the highest-value check.

---

## 3. Build

**Parsing.** `openpyxl` reads formulas with `data_only=False` and cached values with
`data_only=True`. You need both, so open the file twice or use the read-only iterator. For a
dependency graph, `formulas` or `pycel` will parse and evaluate Excel formulas in Python; both are
imperfect on exotic functions, so degrade gracefully and report what you could not parse rather than
crashing or silently skipping.

An `.xlsx` is a zip of XML. When a library fails you, go to the XML directly: `xl/workbook.xml` for
defined names, `xl/externalLinks/` for links, `xl/worksheets/sheetN.xml` for cells. Being able to say
you dropped to the XML when the library was insufficient is a good detail.

**Architecture.**

```
workbook_doctor/
  checks/          one module per check, each returning Finding objects
  graph.py         dependency graph construction
  report.py        JSON and Markdown renderers
  fix.py           proposes patches, never applies without confirm
  mcp_server.py    exposes check_workbook, explain_finding, propose_fix
  cli.py
tests/
  fixtures/        deliberately broken workbooks you build
```

**The `Finding` object** should carry: check id, severity, sheet, cell, a human-readable message, the
formula or value involved, the root-cause cell where applicable, and a suggested fix. Severity in
three tiers: `error` (the number is wrong), `warning` (the number may be wrong or is not
reproducible), `info` (style and maintainability).

**Fixtures matter.** Build a set of deliberately broken workbooks as test fixtures, generated by a
script so they are reproducible. Every check gets a fixture that triggers it and a fixture that must
not trigger it. This is also your demo file.

**MCP server.** Expose three tools: `check_workbook(path)` returning structured findings,
`explain_finding(id)` returning the reasoning and consequence in plain language, and
`propose_fix(id)` returning a patch. Register it in Claude Code and Claude Desktop. Now a language
model can audit a workbook conversationally, which is the actual "AI-assisted workflow" story rather
than a claim about one.

**Human in the loop, made concrete.** `--fix` writes nothing. It emits a patch file and a diff. A
separate `--apply` command takes the reviewed patch and writes a *new* file, never overwriting the
original, and records what it changed in a sidecar log. Design it so that no path through the code
mutates an input file. That design constraint is the interview answer.

---

## 4. The adjacent bullets

**`kensho` (5 résumés): "a Kensho API integration linking Claude directly to S&P Capital IQ financial
data."** You cannot access Kensho or Capital IQ outside the firm. Build the direct analogue: an MCP
server over **SEC EDGAR's XBRL `companyfacts` API**, which is free, requires only a user-agent
header, and returns every reported financial fact for any registrant by CIK. Tools:
`get_company_facts(ticker)`, `get_concept(ticker, concept, period)`, `compare_peers(tickers,
concept)`. Same architectural story, same demonstration of connecting a model to structured financial
data, zero licensing. Your existing 23-company equity research platform likely already contains half
of this; wire it to MCP and it counts.

**`ambassador` (14 résumés): the AI implementation role.** No repo substantiates a role. What
substantiates it is a body of work plus a written account of adoption. Concretely:
- Publish `workbook-doctor` and the EDGAR MCP server with real READMEs.
- Write one post: how you got a sceptical analytics team to adopt generative AI. The content that
  matters is the failure modes: what people tried that did not work, where trust broke, what
  guardrail made it acceptable to a risk function. Nobody writes this honestly and it reads as
  authority.
- The multi-agent book pipeline and this build plan are both artifacts of the same habit.

---

## 5. Interview questions and how to answer

**"What does human-in-the-loop actually mean in your implementation?"**
Point at the design constraint, not the policy. "No code path mutates an input file. The fix mode
emits a patch and a diff; applying it is a separate command that writes a new file and logs what
changed. The control is structural, so it cannot be forgotten under deadline pressure, which is the
failure mode a policy alone does not prevent."

**"Which check catches the most real errors?"**
Hardcoded constants inside formula ranges. Error values are loud and someone usually notices. A
literal typed over a formula in row 87 of a 300-row schedule recalculates cleanly, prints cleanly,
reconciles cleanly, and is wrong.

**"Why volatile functions?"**
Reproducibility. A workbook containing `TODAY()` or `RAND()` produces different numbers on different
days from the same inputs, so you cannot re-derive a figure that was reported last quarter.
`INDIRECT` additionally breaks dependency tracing, so no tool, including Excel's own audit features,
can tell you what a cell depends on.

**"How do you keep the tool from being annoying?"**
Root-cause collapsing and severity tiers. One deleted column producing 400 downstream errors should
be one finding with a count, not 400 findings. A tool that reports everything gets muted in a week.

---

## 6. Further reading

- Panko, "What we know about spreadsheet errors," Journal of End User Computing. The empirical base
  rate literature; the error rates in production spreadsheets are startling and worth quoting.
- EuSpRIG (European Spreadsheet Risks Interest Group) horror stories archive. Includes JPMorgan's
  London Whale VaR model and the Reinhart-Rogoff error. Good interview color.
- `openpyxl`, `formulas`, and `pycel` documentation.
- Model Context Protocol specification, `modelcontextprotocol.io`.
- SEC EDGAR XBRL frames and companyfacts API documentation.
