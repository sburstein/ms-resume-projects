# AI weather benchmark drills

Answer aloud before opening the answer.

## 1. Did GraphCast replace physics-based weather forecasting?

No. It replaced the forecast integration step in this comparison. It still
starts from an analysis created through a physics-based data assimilation system.

## 2. Why is ERA5 a favorable verifier for IFS HRES?

Both come from ECMWF's modeling system. Shared modeling assumptions can create a
home-field advantage for HRES, so an independent station check is valuable.

## 3. How can GraphCast have lower RMSE and worse activity?

Smoothing toward the conditional mean can lower squared error while reducing the
variance and amplitude of forecast anomalies.

## 4. What is the easiest silent alignment error?

Verifying against truth at initialization instead of truth at initialization
plus lead time.

## 5. Why did the scoring loop read all leads together?

The archive stores several leads in one chunk. Reading one lead repeatedly
downloads the same chunk and discards most of it.

## 6. What result contradicted the hypothesis?

The neural models' temperature error relative to HRES improved in the tails
instead of deteriorating.

## 7. What does `aiwx validate` refuse to validate?

It does not encode an expected winner. It checks completeness and consistency,
not whether the conclusion matches a hypothesis.
