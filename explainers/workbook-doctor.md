# Workbook Doctor, in plain language

## The one concept everything hangs on

The dangerous spreadsheet error is the quiet one.

A `#REF!` is loud. Someone notices it. A number typed over a formula in row 87
can recalculate, print, and reconcile without complaint. Excel stores what a cell
contains. It does not store what the cell was supposed to contain.

The tool infers intent from repeated structure. If neighboring rows share one
formula shape and a literal appears inside that block, the pattern is evidence
that something changed.

## Vocabulary you need

**Hardcoded override:** A literal value in a place where a formula pattern says
a formula normally belongs. It may be intentional, so the finding still needs
human review.

**Formula shape:** A formula rewritten in relative terms. `=A10*B10` and
`=A11*B11` have the same shape even though their row numbers differ.

**Dependency graph:** A map from each formula cell to the cells it references.

**Root cause:** The first broken cell in a chain. Downstream error cells are
symptoms.

**Cached value:** The last result Excel calculated and stored in the file.
Libraries can read it, but they do not calculate formulas themselves.

**Volatile function:** A function such as `TODAY()` or `RAND()` that can change
without an input cell changing.

**Shared formula:** An xlsx storage optimization that records a repeated formula
once and points neighboring cells back to it.

## What the tool actually does

The loader opens a workbook twice. One view preserves formulas. The other reads
cached values. Checks need both: a hardcode is visible in the formula view, while
an evaluated error may only appear in the cached-value view.

The loader also reads xlsx XML directly for facts openpyxl does not expose
reliably, including shared formulas and external link targets.

The analyzer builds a dependency graph, runs ten check families, sorts findings
by severity, and reports coverage limits. Analysis never saves a workbook.

The fix path is separate. It creates a JSON patch for mechanical repairs, shows
a diff, and writes a new workbook only after a separate apply command. It refuses
to overwrite the source.

## The ten checks in human terms

**C01:** Find formula error values and collapse a cascade to its origin.

**C02:** Find a number typed into a formula column.

**C03:** Find a formula shaped differently from its neighbors.

**C04:** Find links to other workbooks.

**C05:** Find circular calculations.

**C06:** Find broken or unused named ranges.

**C07:** Find volatile functions and functions that defeat tracing.

**C08:** Find numbers stored as text in numeric columns.

**C09:** Find hidden rows inside totals and merged cells in data regions.

**C10:** Find a total whose range stops before the formula block ends.

## Why root-cause collapsing matters

One deleted column can turn hundreds of downstream formulas into `#REF!`. A tool
that reports 400 findings teaches users to ignore it. The graph walks upstream
through error cells and reports the origin once, with the number of affected
cells.

The limitation is important: downstream errors are only visible when Excel has
calculated and saved cached values. A workbook created by openpyxl contains
formulas but no calculated results. In that case, the origin can still be found,
but the full casualty count cannot.

## The formula-text bug

Formula detectors used to rely on uppercasing or simple quoted-string regular
expressions. That can confuse displayed text with executable formula syntax.

For example, `="Explain #REF! and TODAY()"` contains an error token and a
volatile function name, but neither executes. Excel also escapes quotes by
doubling them, which defeats the usual `"[^"]*"` shortcut.

The project now tokenizes quoted Excel strings before scanning. Display text
does not create an error, external link, volatile function, or dependency.
Shared-formula translation also leaves quoted `A1` text unchanged while shifting
real references.

## What each code piece means

`loader.py` creates the formula and value views and recovers xlsx-only metadata.

`formula.py` separates executable syntax from quoted display text.

`graph.py` extracts dependencies, finds cycles, and traces error origins.

`checks/core.py` implements the ten check families.

`analyze.py` runs the checks and records coverage gaps.

`fix.py` proposes reviewable patches and protects the source file.

`mcp_server.py` exposes the same read-only analysis to a language model.

## What you can say in an interview

"I treated false positives as a product failure, not a cosmetic issue. The clean
fixture must produce zero findings. I used local formula patterns to detect quiet
overrides, a dependency graph to collapse cascades, and a two-step patch workflow
that cannot overwrite the source. I also report what cannot be traced, including
INDIRECT, OFFSET, VBA, and missing cached values."

## Questions this project invites

How should deliberate overrides be documented so C02 can distinguish them from
accidents?

Could Excel's calculation engine be used in a sandbox to test arithmetic while
preserving the no-mutation guarantee?

How should the tool represent confidence when a formula block has several valid
patterns, such as monthly rows and quarterly subtotal rows?

Which checks should block CI, and which should only warn?
