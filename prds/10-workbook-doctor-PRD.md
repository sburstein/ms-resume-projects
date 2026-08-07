# PRD 10: Workbook Health-Check Agent

| | |
|---|---|
| **Repo** | `workbook-doctor` |
| **Bullets covered** | `agent` (9 of 20 résumés), supports `ambassador` (14) |
| **Packet** | [`10-workbook-doctor.md`](../knowledge-packets/10-workbook-doctor.md) |
| **Est.** | 18 human-hours, 4 to 7 agent session hours |
| **Priority** | 8 of 17. Independent of everything else, fastest to finish, best live demo. |
| **Model** | Sonnet throughout. Opus only for the dependency-graph evaluator if it fights back. |
| **Prereqs** | Read `00-CONVENTIONS.md`. |

---

## 1. Objective

A Python package, CLI, and MCP server that audits `.xlsx` workbooks for data-integrity defects,
reports root causes rather than symptom floods, and offers a fix mode that proposes patches without
ever mutating an input file.

**Success test:** `workbook-doctor check tests/fixtures/broken_model.xlsx` returns structured
findings covering every seeded defect, with zero findings on the matched clean fixture, and a demo
where Claude audits a workbook conversationally through MCP.

---

## 2. Scope

**In:** `.xlsx` and `.xlsm`. Ten check families (section 4). JSON and Markdown reports. MCP server.
Propose-only fix mode with a separate apply step.

**Out:** `.xls` (legacy binary). Google Sheets. VBA macro analysis beyond noting presence. Automatic
correction without human approval, ever.

---

## 3. Architecture

```
workbook_doctor/
  model.py          Finding, Severity, Location dataclasses
  loader.py         dual open (formulas + cached values), XML fallback
  graph.py          dependency graph, cycle detection, root-cause attribution
  checks/
    errors.py       C01 error values
    hardcodes.py    C02 constants in formula ranges
    consistency.py  C03 inconsistent formulas
    external.py     C04 external links
    circular.py     C05 circular references
    names.py        C06 orphaned/broken named ranges
    volatile.py     C07 volatile functions
    types.py        C08 numbers as text, mixed types
    structure.py    C09 merged cells, hidden rows in ranges
    crosssheet.py   C10 totals not matching source ranges
  report.py
  fix.py
  mcp_server.py
  cli.py
```

**`Finding` schema** (stable public contract, version it):

```python
@dataclass(frozen=True)
class Finding:
    check_id: str            # "C02"
    severity: Severity       # ERROR | WARNING | INFO
    sheet: str
    cell: str                # "B47"
    message: str             # human-readable, one sentence
    detail: str              # the formula or value involved
    root_cause: Location | None
    affected_count: int      # for collapsed findings
    suggested_fix: str | None
```

---

## 4. Check specification

| id | Check | Severity | Detection |
|---|---|---|---|
| C01 | Error values | ERROR | Scan cached values for `#REF!`, `#VALUE!`, `#DIV/0!`, `#N/A`, `#NAME?`, `#NULL!`, `#NUM!`. **Walk the dependency graph to the originating cell and collapse downstream instances into `affected_count`.** |
| C02 | Hardcoded override in formula range | ERROR | In a contiguous range where >= 80% of cells share an R1C1-equivalent formula, flag literal constants. **The highest-value check.** |
| C03 | Inconsistent formula | WARNING | Same detection, but the odd cell holds a different formula rather than a constant |
| C04 | External links | WARNING | Parse `xl/externalLinks/` and formula strings for `[workbook]` references. ERROR if the link target is unresolvable. |
| C05 | Circular reference | ERROR | Cycle detection on the dependency graph |
| C06 | Broken or orphaned named range | WARNING for broken, INFO for unused | Parse `definedNames` in `xl/workbook.xml`; flag `#REF!` targets and names with zero references |
| C07 | Volatile functions | WARNING | Regex for `NOW`, `TODAY`, `RAND`, `RANDBETWEEN`, `OFFSET`, `INDIRECT`. Flag `INDIRECT` separately at ERROR because it also defeats dependency tracing. |
| C08 | Type issues | WARNING | Numeric-looking strings, mixed types within a column, dates as strings |
| C09 | Structural risk | INFO, or WARNING if inside a summed range | Merged cells in data ranges, hidden rows or columns inside a range referenced by an aggregate |
| C10 | Cross-sheet total mismatch | ERROR | Evaluate summary cells against the sum of their referenced ranges; flag mismatches beyond a float tolerance |

---

## 5. Milestones

### M0. Scaffold and fixtures
Repo per conventions. **Write `tests/fixtures/make_fixtures.py` first**, generating with `openpyxl`:
- `clean_model.xlsx`: a realistic 3-sheet financial model with no defects.
- `broken_model.xlsx`: the same model with one seeded instance of every check C01 to C10, with the
  expected findings recorded in `tests/fixtures/expected.json`.
**Accept:** both fixtures generate deterministically. `expected.json` is the spec the checks must
satisfy. This is test-first by design; the fixtures define correctness.

### M1. Loader
Dual open (formulas and cached values). Fall through to raw XML in the zip when `openpyxl` cannot
answer (external links, defined names). Graceful on password-protected and corrupt files: a clear
error, never a stack trace.
**Accept:** loads both fixtures. Handles a deliberately truncated file with a clean error message.

### M2. Dependency graph
Parse formula references into a directed graph. Support A1 and R1C1, absolute and relative,
cross-sheet, and ranges. Where a formula cannot be parsed (exotic functions, `INDIRECT`), record it
as unparsed and **report the count** rather than silently omitting it.
**Accept:** cycle detection works on a seeded circular reference. Unparsed-formula count is reported
in the run summary. A test asserts root-cause attribution: one seeded `#REF!` producing 20 downstream
errors collapses to 1 finding with `affected_count == 20`.

### M3. Checks C01 to C07
Implement each as an independent module with its own test against the fixtures.
**Accept:** every seeded defect in `broken_model.xlsx` is found; **zero findings on
`clean_model.xlsx`.** The false-positive requirement is as important as the true-positive one; a
noisy tool gets muted.

### M4. Checks C08 to C10
C10 needs evaluation, so this is the hardest. Use `formulas` or `pycel` for evaluation, degrade
gracefully where evaluation fails, and report what could not be evaluated.
**Accept:** as M3, plus a documented list of formula constructs the evaluator cannot handle.

### M5. Reporting
JSON (machine, schema-versioned) and Markdown (human, grouped by severity then by sheet, with
root-cause findings first). Exit codes: 0 clean, 1 warnings only, 2 any error-severity finding.
**Accept:** exit codes tested. Markdown report is legible at a glance for the broken fixture.

### M6. Fix mode
`fix.py` proposing patches as a JSON patch file plus a human-readable diff. `--apply` is a **separate
command** that reads a reviewed patch and writes a **new** file, never overwriting the input, and
logs every change to a sidecar `.changes.json`.
**Accept:** a test asserts that no code path in the package opens an input file in write mode.
Implement this as an actual test (monkeypatch `open` and `Workbook.save`), not a code-review claim.
This test is the concrete expression of human-in-the-loop.

### M7. MCP server
Tools: `check_workbook(path)`, `explain_finding(check_id, finding_id)`, `propose_fix(finding_id)`,
`list_checks()`. Config snippet for Claude Code and Claude Desktop in the README.
**Accept:** server starts, tools are discoverable, and a manual round-trip through Claude produces a
sensible audit conversation. Record a short terminal capture for the README.

### M8. Package and docs
`pipx`-installable. README with a 90-second demo transcript. `CHECKS.md` documenting every check,
its rationale, and its false-positive profile.

---

## 6. Golden-number regression tests

- Findings count on `broken_model.xlsx` per check id, exact.
- Findings count on `clean_model.xlsx`, exactly 0.
- Root-cause collapse ratio for the seeded `#REF!` cascade, exact.

---

## 7. Risks and mitigations

| Risk | Signal | Mitigation |
|---|---|---|
| **False positives make it unusable** | Findings on the clean fixture | Zero-findings requirement on `clean_model.xlsx` is a hard gate in M3 and M4. |
| Symptom floods | 400 findings from one deleted column | Root-cause collapsing in M2, tested. |
| Formula evaluation is incomplete | C10 silently misses mismatches | Report unevaluable constructs explicitly; never treat "could not evaluate" as "passed". |
| Accidentally mutating a user's workbook | Catastrophic and unrecoverable | M6 test asserting no write-mode opens anywhere in the package. |
| `openpyxl` limitations | Missing external links, defined names | XML fallback in M1. |

---

## 8. Companion scope

PRD 13 (`edgar-mcp`) covers the second half of the `kensho` bullet. Build it after this one; the MCP
server patterns transfer directly.
