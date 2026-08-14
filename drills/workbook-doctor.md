# Workbook Doctor drills

## 1. Why is a hardcoded override more dangerous than `#REF!`?

It can calculate and print normally. Nothing in Excel announces that a formula
was expected there.

## 2. How does the tool infer intent?

It compares relative formula shapes across neighboring cells and looks for
breaks inside a dominant pattern.

## 3. Why open the workbook twice?

One view preserves formulas. The other reads cached calculated values.

## 4. Why collapse error cascades?

Hundreds of downstream errors can have one origin. Reporting the origin is
actionable and reduces alert fatigue.

## 5. Why can INDIRECT not be traced reliably?

It constructs a reference at calculation time, often from text. Static analysis
cannot know the target in general.

## 6. What is the mutation guarantee?

Analysis and proposals never save a workbook. Applying a reviewed patch writes a
new file and refuses to overwrite the source.

## 7. Why tokenize formula strings?

Displayed text can contain tokens such as `#REF!`, `TODAY()`, or `A1` without
executing them. Detectors must inspect syntax only.
