# Packet 01: ENSO Long/Short Equity Basket

> **Résumé line:** "Constructed a long/short equity basket expressing El Niño exposure, translating
> probabilistic seasonal forecasts into sector and single-name positioning across soft commodities,
> metals, EM rates, and consumer sectors."

**Repo:** `enso-basket` | **Est. 25 hours** | **Build last (compliance gate)**

---

## 1. The idea

A seasonal climate forecast is a probability distribution over a physical state. A trade is a
position with a size. This project is the translation layer between them: given a 63% chance of El
Niño rising to an 81% chance of a very strong event, what do you own and what do you short, and how
big?

The whole point of making it long *and* short is to isolate the weather bet. If you are only long
fertilizer names, you are mostly long the equity market with a fertilizer tilt. Pair each long
against a short in an economically linked but oppositely exposed name, neutralize the residual
market beta, and what remains is closer to a pure expression of "El Niño happens and it matters."

---

## 2. The science you need to have cold

**The mechanism.** Under normal conditions, trade winds push warm surface water west across the
Pacific, piling it up near Indonesia and allowing cold water to upwell off South America. In an El
Niño, the trade winds weaken or reverse. Warm water sloshes east. The atmospheric convection center
that normally sits over the Maritime Continent shifts thousands of kilometers east into the central
Pacific.

That relocation of the tropical heat source is what drives everything else. It changes where
atmospheric Rossby waves are generated, which changes the position of the subtropical jet, which
changes storm tracks across North America and the strength of the Indian and Australian monsoons.
This is why a temperature anomaly in one ocean basin reorganizes rainfall on four continents.

**The canonical impacts** (know these, they are the trade rationale):

| Region | El Niño effect | Economic transmission |
|--------|---------------|----------------------|
| Australia, Indonesia | Drought | Palm oil, wheat, coal supply, nickel |
| Brazil (south) | Wet | Sugarcane harvest delays, lower TRS yield, shift to ethanol |
| Brazil (north), Colombia | Dry | Coffee, hydro generation, Panama Canal water levels |
| Peru, Ecuador | Heavy rain, flooding | Fishmeal (anchoveta collapse), infrastructure |
| Chile | Variable, often dry | Copper (mining is water-intensive) |
| India | Weak monsoon | Food inflation, rural consumption, rate policy |
| West Africa | Drier Harmattan | Cocoa |
| Northern US | Milder winter | Lower heating demand, natural gas |
| US Gulf, Southeast | Wetter, cooler | Suppressed Atlantic hurricane season (vertical wind shear) |

**Why hurricanes are suppressed** is worth knowing because it surprises people: El Niño increases
upper-level westerly winds over the tropical Atlantic, which increases vertical wind shear, which
tears apart the vertical structure a tropical cyclone needs. Fewer and weaker Atlantic storms. That
is a directly tradeable insurance and reinsurance view.

**Probabilistic forecasts.** The IRI/CPC ENSO plume publishes a probability distribution across El
Niño, neutral, and La Niña for each overlapping three-month season out about nine months. The
**spring predictability barrier** is the thing to mention: forecasts issued in boreal spring
(March to May) for the following winter are markedly less skillful, because that is when the
ocean-atmosphere coupling is weakest and the system is most sensitive to noise. Anyone who has
actually worked with these forecasts knows this. It is a good signal to drop.

---

## 3. Data

| Source | What | Access | Gotcha |
|--------|------|--------|--------|
| NOAA CPC ONI | Monthly Oceanic Niño Index, 1950 to present | Free text file, `origin.cpc.ncep.noaa.gov` | ONI is revised when the climatology base period shifts every 5 years. Use the current version consistently. |
| NOAA CPC Niño 3.4 weekly | Weekly SST anomaly | Free | Noisier than ONI, better for event timing |
| IRI ENSO plume | Probabilistic forecast archive | IRI website, some archives as CSV | Only recent years are cleanly machine-readable. Older plumes are images. |
| Yahoo Finance via `yfinance` | Daily equity prices | Free | Survivorship bias: delisted names are missing. Note this limitation explicitly. |
| Stooq | Daily prices, alternative source | Free CSV | Better coverage of some non-US listings |
| Nasdaq Data Link (free tier) | Some commodity series | Free with key | Free tier coverage of futures is thin |
| ETF proxies | CANE (sugar), DBA (ag), COPX (copper miners), JJC, WEAT, CORN | `yfinance` | Roll yield contaminates commodity ETF returns. Say so. |

**The realistic constraint:** you will not have clean free futures history. Use ETF and equity
proxies, state the limitation in `FINDINGS.md`, and move on. An interviewer respects a stated
limitation far more than a hidden one.

---

## 4. Method, in steps

1. **Define events.** Use ONI. An El Niño event is five consecutive overlapping three-month seasons
   at or above +0.5 C. Classify by peak ONI: weak (0.5 to 0.9), moderate (1.0 to 1.4), strong (1.5
   to 1.9), very strong (2.0+). Since 1990 you get roughly eight events, of which three are very
   strong (1997-98, 2015-16, 2023-24). Write down the event table before you look at any prices.
2. **Define the window.** Anchor on the month ONI first crosses +0.5, and measure returns over the
   following 3, 6, and 12 months. Do not anchor on the peak; you would not have known the peak in
   advance. This is the main lookahead trap in the whole project.
3. **Build the exposure map.** For each theme, list the long leg and the short leg with a written
   one-line rationale. Do this before running any returns. Freeze the map in a YAML file so it is
   auditable and so you cannot quietly tune it later.
4. **Construct the basket.** Equal-weight within leg, then scale legs to dollar-neutral. Then
   estimate each name's beta to SPY over the prior 252 days and rescale to beta-neutral. Report both
   versions; they will differ and the difference is informative.
5. **Size by probability.** This is the piece that makes it a *probabilistic* translation rather
   than a binary bet. Scale gross exposure by the forecast probability of the event, or by
   probability times expected intensity. Show the equity curve with and without probability scaling.
6. **Backtest.** Event-window returns, hit rate, average and median, and a t-stat across events.
7. **Null test.** Sample 10,000 random windows of identical length from the same price history and
   build the null distribution of basket returns. Your event-window result is meaningful only
   relative to that distribution. With eight events, expect a p-value that is suggestive, not
   conclusive.
8. **Attribution.** Decompose basket return into theme contributions. Which leg actually carried it?
   Often one or two names dominate, and saying so is more credible than presenting a smooth curve.

---

## 5. Numbers to know cold

- NOAA El Niño threshold: **+0.5 C ONI** sustained over five overlapping seasons.
- Very strong threshold: **+2.0 C**.
- 2023-24 event peaked around **+2.0 C**; 2015-16 peaked around **+2.6 C**; 1997-98 around **+2.4 C**.
- ENSO recurs roughly every **2 to 7 years**, and events typically peak in **boreal winter**
  (November to January).
- Roughly **eight El Niño events since 1990**. That is your effective sample size and it is small.
- The spring predictability barrier degrades skill for forecasts issued **March to May**.

---

## 6. Interview questions and how to answer

**"Walk me through how you sized the positions."**
Start with the probability, not the position. "The forecast gave me a distribution, not an event, so
gross exposure scaled with the probability of the event occurring times its expected intensity.
Within that, each theme got equal risk weight rather than equal dollars, because copper and sugar
have very different volatilities."

**"How do you know this is not just a commodity beta trade?"**
"That is exactly why it is long/short. I dollar-neutralized and then beta-neutralized against the
market. I also ran the basket against a commodity index factor to check residual loading. If the
residual is small, what is left is the weather view."

**"Eight events is not a lot. Is this significant?"**
Do not oversell. "No, and I would not claim it is. The direction is consistent across events and the
mechanism is physical rather than data-mined, which is why I believe it. But with eight events the
t-stat is weak, and I built a null distribution from random windows specifically so I would not fool
myself. I would size this as one input among several, not as a standalone strategy."

**"What is the spring predictability barrier?"**
Answer as in section 2. This is a strong differentiator; most finance candidates have never heard
of it.

**"Which is the highest-conviction leg?"**
Sugar and palm oil, because the physical transmission is short and direct: rainfall in a specific
growing region in a specific quarter maps to harvest yield in the same crop year. Copper is weaker
because Chilean mining has water storage and the correlation is diluted by demand-side factors that
swamp supply.

---

## 7. Further reading

- NOAA CPC ENSO Diagnostic Discussion, issued monthly. Read three of them.
- L'Heureux et al., "Observing and Predicting the 2015-16 El Niño," BAMS 2017.
- Cashin, Mohaddes and Raissi, "Fair Weather or Foul? The Macroeconomic Effects of El Niño,"
  IMF Working Paper 2015. The standard economics reference; gives you country-level GDP elasticities.
- Brunner, "El Niño and World Primary Commodity Prices," Review of Economics and Statistics 2002.
