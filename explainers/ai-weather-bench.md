# AI weather benchmark, in plain language

## The one concept everything hangs on

Neural weather models replaced the **forecast integration** step. They did not
replace the **data assimilation** step.

Data assimilation turns satellites, weather balloons, aircraft readings, radar,
and surface observations into one internally consistent estimate of the current
atmosphere. GraphCast and Pangu begin from that estimate. In this project, the
estimate comes from ECMWF's modeling system. If that starting analysis vanished,
the neural forecast would have no initial state to advance.

So the honest question is not, "Did AI replace physics?" It is, "Given a
physics-assisted starting state, can a learned forecast operator advance it more
accurately and cheaply than the traditional forecast model?"

## Vocabulary you need

**Analysis:** The best estimate of the atmosphere at one moment. It combines
observations with a model. It is the starting point for a forecast.

**Forecast integration:** The repeated calculation that moves the atmospheric
state forward through time. IFS HRES solves physical equations. GraphCast and
Pangu apply learned transformations.

**ERA5:** ECMWF's historical reanalysis. It is used as truth here, with the
warning that it shares institutional and modeling ancestry with IFS HRES.

**Lead time:** How far the forecast is from its starting time. Day 5 is a
120-hour lead. Day 10 is a 240-hour lead.

**Z500:** Geopotential at 500 hPa. It describes the shape of large-scale weather
patterns in the middle atmosphere. Lower error means the ridges and troughs are
placed more accurately.

**RMSE:** Root mean squared error. It punishes large misses more than small ones.
Lower is better.

**ACC:** Anomaly correlation coefficient. It asks whether the forecast places
departures from normal in the right pattern. Higher is better.

**Activity:** The standard deviation of forecast anomalies. It measures how much
variation the model preserves. If activity falls below truth as lead grows, the
forecast is becoming too smooth.

## What the project actually does

The project reads precomputed WeatherBench 2 archives for GraphCast,
Pangu-Weather, and IFS HRES. It does not train a model and does not run inference.
That choice makes the benchmark reproducible on a laptop.

It intersects the models' available start times and lead times, then scores the
same 732 initializations from 2020. Every forecast is verified at its valid time,
which is start time plus lead time. That addition sounds trivial, but using the
start time by mistake produces smooth, plausible, wrong charts.

The scoring loop follows the storage layout. GraphCast's archive chunks several
lead times and all pressure levels together. Asking for one lead at one level
still transfers the whole chunk. Reading all leads in a block reduced repeated
downloads by about 9 times.

The output table contains 7,200 rows:

- 5 forecast or baseline series
- 3 variables
- 2 regions
- 40 lead times
- 6 metrics

The new `aiwx validate` command checks that this rectangular structure is
complete and internally consistent before figures are trusted.

## What the numbers say

At day 5, global Z500 RMSE is 274 for GraphCast, 294 for Pangu, and 305 for IFS
HRES. GraphCast has the lowest average error.

At day 10, the ordering remains GraphCast at 732, Pangu at 778, and IFS HRES at
803. The neural models still win the average-error score.

Activity tells a second story. At day 10, ERA5 Z500 activity is 814. GraphCast
produces 756, Pangu 799, and HRES 803. GraphCast is best on RMSE and worst on
preserving variance. Those findings can coexist because predicting closer to the
mean can reduce squared error as uncertainty grows.

## The result that refused to cooperate

The initial hypothesis said neural models would lose more of their advantage in
the hottest and coldest 1% of 2 m temperature outcomes. The test found the
opposite. GraphCast's error relative to HRES was lower in the tails than in the
full sample at every tested lead.

The project leaves that result intact. It does not tune the tail threshold until
the hypothesis wins. The open question is why lower global activity does not
translate into a larger tail-conditioned error here.

## What each code piece means

`config/sources.yaml` is the permission boundary. Remote data can only come from
declared hosts and the declared Google Cloud bucket.

`align.py` makes model, truth, variable, level, lead, and grid coordinates mean
the same thing before any arithmetic begins.

`metrics.py` defines latitude weighting and each score.

`score.py` is the streaming engine. It accumulates weighted sums instead of
holding the full forecast cube in memory.

`extremes.py` re-scores only observations whose truth lies in the selected tails.

`validation.py` checks the published score table for missing metrics, duplicate
cells, mismatched lead grids, non-finite values, and inconsistent truth rows.

`viz.py` turns tidy score rows into charts without changing the underlying data.

## What you can say in an interview

"I benchmarked forecast archives, not model inference. The most important design
choice was separating initialization from forecasting. Both neural models depend
on an ECMWF analysis, and I verified them against ECMWF's ERA5, so I disclose the
home-field advantage. GraphCast won average error through day 10 but lost more
variance. My preregistered extremes hypothesis failed, and I kept the result."

## Questions this project invites

How much of GraphCast's advantage survives against station observations that are
independent of ECMWF?

Does the variance loss matter more for energy demand, wind ramps, precipitation,
or other operational targets than it does for global Z500?

Would an ensemble model resolve the tension by representing uncertainty instead
of predicting one conditional mean?

Why does the tail-conditioned temperature result improve even while activity
falls?
