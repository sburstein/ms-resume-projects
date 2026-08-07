# Packet 00: Cross-Cutting Glossary

Terms that appear across multiple projects. Project-specific vocabulary lives in each packet.

## Power system

- **Gross load** the total electricity consumers actually use in an hour.
- **Net load** gross load minus non-dispatchable renewable generation. This is what the grid
  operator must supply from dispatchable resources. Batteries earn from the *shape* of net load,
  not its level.
- **Duck curve** the daily net-load shape in high-solar regions: moderate morning, deep midday
  trough as solar floods in, steep evening ramp as the sun sets. First named by CAISO in 2013.
- **Ramp** the rate of change of net load, in MW per hour. The CAISO evening ramp now exceeds
  15 GW in three hours on some spring days. Ramp, not peak, is the modern reliability constraint.
- **Behind the meter (BTM)** on the customer side of the utility meter. Rooftop solar is BTM. The
  utility never measures it; it only sees reduced purchases.
- **Front of meter (FOM)** grid-side, directly metered generation.
- **Balancing authority (BA)** the entity responsible for matching supply and demand in a control
  area. CAISO, ERCOT, BPAT, PACE, and about 60 others in the US. EIA-930 reports at BA level.
- **ISO / RTO** Independent System Operator / Regional Transmission Organization. Non-profit market
  operators: CAISO, ERCOT, PJM, MISO, NYISO, ISO-NE, SPP. Roughly two thirds of US load.
- **FERC** Federal Energy Regulatory Commission. Federal regulator for interstate wholesale power.
  State PUCs regulate retail rates.
- **LMP** Locational Marginal Price. The price of an additional MW at a specific grid node.
  Decomposes into energy, congestion, and loss components. ERCOT and PJM are nodal; CAISO is nodal
  in wholesale but settles load zonally.
- **BESS** Battery Energy Storage System.
- **Revenue stack** the several income streams a battery earns simultaneously: energy arbitrage,
  capacity payments, and ancillary services (frequency regulation, spinning reserve).
- **Curtailment** deliberately throttling renewable output when generation exceeds what the grid can
  absorb or transmit. CAISO curtailed over 3 TWh of solar and wind in 2024.
- **Dispatch model** an optimization that decides which units run each hour to meet load at minimum
  cost, subject to capacity, ramp, and transmission constraints. Usually mixed-integer linear
  programming (unit commitment) followed by linear programming (economic dispatch).
- **IOU / muni / co-op** investor-owned utility (shareholder-owned, publicly traded parent),
  municipal utility (city-owned), rural electric cooperative (member-owned). Roughly 70% of US
  customers are served by IOUs.

## Weather and climate

- **ERA5** ECMWF's fifth-generation reanalysis. Hourly global weather from 1940 to near present on a
  0.25 degree grid. The de facto ground truth for model verification. Free from the Copernicus
  Climate Data Store.
- **Reanalysis** a retrospective "best estimate" of past atmospheric state, produced by running a
  fixed modern forecast model and assimilating all historical observations. Not observation, not
  pure model. A blend.
- **NWP** Numerical Weather Prediction. Physics-based forecasting by integrating the primitive
  equations forward on a 3D grid.
- **Z500** geopotential height at the 500 hPa pressure level, roughly 5.5 km up. The standard
  headline verification variable because it captures large-scale flow.
- **T850** temperature at 850 hPa, roughly 1.5 km up. Above the boundary layer, so it is a cleaner
  measure of airmass than surface temperature.
- **RMSE** root mean square error. Penalizes large errors quadratically.
- **ACC** anomaly correlation coefficient. Correlation between forecast anomaly and observed anomaly
  relative to climatology. The convention is that a forecast is "useful" above 0.6 ACC for Z500.
- **Lead time / horizon** how far ahead the forecast is valid. Day 1 through day 14 here.
- **Spread-skill relationship** in a good ensemble, the spread between members should match the RMSE
  of the ensemble mean. Underdispersive ensembles are overconfident.
- **Climatology** the long-run average state for a location and time of year. The baseline every
  forecast must beat to be worth anything.
- **Persistence** the naive forecast that tomorrow equals today. The other baseline to beat.
- **Downscaling** converting coarse climate-model output (50 to 100 km grid cells) to local scale.
  Statistical downscaling maps model output onto observed fine-scale patterns. Dynamical
  downscaling nests a high-resolution regional model inside the global one.
- **ENSO** El Niño Southern Oscillation. The coupled ocean-atmosphere oscillation in the tropical
  Pacific. El Niño is the warm phase, La Niña the cool phase, and the neutral state sits between.
- **ONI** Oceanic Niño Index. Three-month running mean of sea surface temperature anomaly in the
  Niño 3.4 region. NOAA declares an El Niño at +0.5 C sustained for five overlapping seasons.
  "Very strong" is +2.0 C or above.
- **Teleconnection** a statistical linkage between weather in widely separated regions, mediated by
  atmospheric wave propagation. ENSO's effect on North American winter is the classic case.

## Statistics and modeling

- **Backtest** evaluating a model on historical data as if you did not know the outcome. The
  cardinal sin is lookahead bias: letting the model see information unavailable at the time.
- **Nowcasting** estimating the present or very recent past for a quantity whose official
  measurement is published with a lag. Standard in macroeconomics, underused in energy.
- **MAPE / MAE / RMSE** mean absolute percentage error, mean absolute error, root mean square
  error. Load forecasting convention is MAPE; it breaks near zero, which matters for net load.
- **Variance decomposition** attributing the variance of an output to its inputs. Sobol indices for
  a nonlinear model, Shapley values for a fitted ML model, simple partial R-squared for a linear one.
- **Rank correlation** Spearman or Kendall. Measures agreement in ordering rather than magnitude.
  The right tool when comparing risk *scores* that use different scales.
- **Market neutral** a portfolio construction where long and short exposures offset so the position
  is not a bet on market direction. Dollar-neutral matches notional; beta-neutral matches market
  sensitivity, which is the stricter and more correct version.
- **Event study** measuring returns in a defined window around a recurring event, averaged across
  occurrences, with a null distribution built from random windows.

## AI and workflow

- **MCP** Model Context Protocol. An open standard for connecting language models to external tools
  and data sources. A server exposes tools; a client (Claude Code, Claude Desktop) calls them.
- **Human in the loop** the model proposes, a person approves before anything takes effect. Not
  optional in regulated finance.
- **Agent** a model given tools and a loop, able to take multiple actions toward a goal rather than
  producing one response.
