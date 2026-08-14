# Utility affordability drills

## 1. What are the three entity levels?

Operating utility, geography, and legal parent.

## 2. Why is customer-weighted coverage stronger than row coverage?

Missing one large utility matters more than missing many tiny utilities.

## 3. Why does Exhibit 21 run before fuzzy matching?

It is filed evidence of ownership. String similarity is only a clue.

## 4. What made "AES Electric Ltd" dangerous?

Generic words such as electric created high similarity scores across unrelated
utilities. The matcher now requires a distinctive shared token.

## 5. Why are state effects the nowcast model?

The monthly sample and annual universe differ by a persistent amount determined
by each state's utility mix.

## 6. What prevents lookahead?

Each historical replay fits only on annual data that would have been published
by that simulated date, and a runtime assertion checks the cutoff.

## 7. Why is county burden a proxy?

EIA identifies counties served but not the exact households or tract boundaries
inside each utility territory.
