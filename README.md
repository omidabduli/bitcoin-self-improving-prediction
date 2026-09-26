# Bitcast

[![Learn & publish](https://github.com/omidabduli/bitcoin-self-improving-prediction/actions/workflows/pipeline.yml/badge.svg)](https://github.com/omidabduli/bitcoin-self-improving-prediction/actions/workflows/pipeline.yml)

**Live: https://omidabduli.github.io/bitcoin-self-improving-prediction/**

Every 15 minutes this project writes down where it thinks the Bitcoin (BTC) price will be in 1 hour, 3 hours and 24 hours. Then it waits, checks, and keeps the score in public.

It is the twin of [ADAptive](https://github.com/omidabduli/cardano-self-improving-prediction), which does the same for Cardano. Same code, same rules, a different coin.

## Why I built this

I love prediction. Nassim Taleb changed how I look at the world. He taught me that most of what happens is more random than we like to admit, and that we are very good at fooling ourselves after the fact. Ray Dalio says you have to be a hyperrealist. I agree with both, and I think prediction is where the two meet.

To predict something you have to be realistic. You have to look at how things actually work, not how you wish they worked. Even then you will be wrong a lot. But every time you check a prediction against what really happened, you learn a bit more about reality. That makes the next prediction a little closer, and it makes you a more realistic person. I think being realistic is directly connected to being happy: you understand the world better, you expect the right things, and you keep wanting to understand more.

The other part is flexibility. If you can't change your mind, you get crushed. So I didn't want a model that I train once and admire. I wanted one that is forced to face every result and adjust itself, every day, in public.

And honestly, there is a joy in it that is hard to describe. When you build a model of the future and the future comes out close to it, it feels like you understood something real. That feeling is the reason for this whole project.

## What it does

Every 15 minutes (at :00, :15, :30 and :45 UTC) it publishes three forecasts: Bitcoin in 1 hour, 3 hours and 24 hours. Each one has a price, an 80% range and a probability that the price will be higher. Every forecast is committed to this repository before the outcome is known, and checked when its time comes.

There is no server. When you open the page, your browser runs the published model on the live Binance feed and computes every forecast and every score up to the current minute. A GitHub Actions job is the notary: every 15 minutes it replays the same minutes with the same code and commits the official record. GitHub's own timer only fires a few times a day, so a free [cron-job.org](https://cron-job.org) job starts it at minute 1, 16, 31 and 46 of every hour. I checked that the browser and the record give the same numbers, to the last digit.

## What to expect (the realistic part)

I built and tested this system on Cardano first, over 300 days of history, walking forward day by day and only ever training on the past:

- **Direction was a coin flip.** Every combination of signals I tried called the 1 h, 3 h and 24 h direction right between 48% and 54% of the time.
- **The price estimate was about as good as "no change".**
- **The range was the part that worked.** The 50%, 80% and 95% ranges held 50%, 80% and 95% of the time.

Then I ran the same thing on Bitcoin for a full year. Same result at first: the ranges were right, the direction was a coin flip (50.5% at 1 hour). Nine small variations of the settings didn't change that.

What did change it was one idea: **train a model only on whether the price went up or down, not on by how much.** A model that learns the size of moves is pushed around by a few huge swings. A model that learns only the direction isn't, and direction is exactly what "right or wrong" measures. I searched for the best version of that idea using only the first eight months (24 Sep 2025 to 24 May 2026) and didn't look at the last four until the choice was made. Then I ran the whole system over the full year, refitting every day on past data only:

| Bitcoin, 25 Sep 2025 to 24 Sep 2026 | 1 hour | 3 hours | 24 hours |
|---|---|---|---|
| Confident calls right (independent) | **54.2%** of 4,527 | **54.0%** of 1,535 | 45.7% of 173 |
| ... in the eight months used for choosing | 53.6% | 53.7% | 45.0% |
| ... in the four untouched months | **55.4%** | **54.5%** | 46.9% |
| All calls right (independent) | 52.5% of 8,759 | 53.4% of 2,919 | 47.0% of 364 |
| 80% range held | 80.0% | 80.0% | 79.8% |
| Typical miss vs. "no change" | 0.31% vs. 0.31% | 0.54% vs. 0.54% | 1.66% vs. 1.66% |

A confident call is one where the direction model's signal is stronger than its usual (the median on its own training data), so about half of all calls. At 1 hour, 54.2% over 4,527 calls is 5.7 standard deviations away from a coin flip. That is very unlikely to be luck, and it held up in the months I didn't use for choosing. At 24 hours there is still no edge, and I'm not pretending there is.

It is still a backtest. Markets change, and a pattern that worked for a year can fade. That's why the live record sits right next to it on the page, scored the same way, and the goal stays public: 54% on confident calls at 1 and 3 hours.

The backtest is on the page, month by month, and every single forecast is in `data/backtest/`. Only forecasts that don't overlap are counted, so a single lucky move isn't counted a hundred times.

If a real pattern shows up, the system is built to find it, and the record will show it. Until then, it is honest about what it doesn't know. This is an experiment, not financial advice.

## How it keeps adjusting

Six "experts" look at the market in different ways:

| Expert | What it looks at |
|---|---|
| The Skeptic | Nothing. It always says "no change". Everyone else has to beat it. |
| Trend Reader | Bitcoin's momentum and reversals from 15 minutes to 3 days, and where the price sits in its recent range |
| Market Watcher | Moves in Ethereum and Solana, and how far Bitcoin lags behind them |
| Crowd Reader | Buying and selling pressure, trading activity and the daily Fear & Greed index |
| Linear Brain | A regularised regression on the signals that evolution picked |
| Boosted Forest | Gradient-boosted trees on the same signals, for non-linear patterns |

Next to them sits the **direction model**: a ridge regression and gradient-boosted trees trained on the sign of the move only (up or down), on all 55 signals and the last 240 days. It decides the up/down call and its probability. The six experts decide the price estimate and the range.

After every result, four things happen:

1. Experts that were closer to reality get more trust, and the others get less (a Hedge ensemble).
2. A range that missed gets wider, and one that held gets narrower, until each holds as often as it promises (adaptive conformal inference).
3. The stated probabilities are recalibrated, so "55%" really means 55%. It only learns how strong the signal is, never an up or down bias, so it can't just follow the recent trend.
4. Once a day at 00:00 UTC, mutated settings challenge the current ones on the last ten days they haven't seen. A challenger only wins if its lead is clear and consistent. In testing on Cardano, my first rule changed settings on 38 of 50 days and did worse than never changing at all. Being flexible doesn't mean reacting to every bit of noise.

A few details that mattered more than I expected:

- 24-hour outcomes that are 15 minutes apart are almost the same outcome. So the longer the horizon, the harder the models are held back (ridge penalty × h/60, and slower, bigger-leaved trees).
- Volatility is estimated from an equal mix of the last 1 hour, 6 hours, 24 hours and 3 days. That gave the sharpest ranges that still held.
- The ranges are centred on what the experts think, never on the drift of the last few weeks. The old version followed that drift, and it made the 24-hour estimate worse than "no change".

Everything is written from scratch in plain JavaScript with no dependencies, including the gradient boosting. The same files in `site/core/` run in the browser and in GitHub Actions.

## The record

Everything lives in `data/` and is committed by the bot:

| File | What's in it |
|---|---|
| `predictions/YYYY-MM-DD.csv` | one row per forecast: the price, and for each horizon the predicted move (bp), P(up), the 80% range (bp) and whether it was a confident call |
| `state.json` | the learning checkpoint: trust weights, range sizes, calibration, forecasts still waiting |
| `model.json` | all six experts, including every tree of the forest |
| `evolution.json` | every daily tournament and the settings that won |
| `daily/YYYY-MM.json` | scores per day and snapshots of the ensemble |
| `fng.json` | the last 30 days of the Fear & Greed index, exactly as the record used it |
| `status.json` | the last run, totals and a file index |
| `backtest.json`, `backtest/YYYY-MM.csv` | the one-year walk-forward backtest: scores per day and every forecast with its outcome |
| `warmup.json` | the 30-day warm-up simulation from launch |

A forecast made at time *t* only uses models trained before *t* and data that was public at *t*. Times are UTC candle open times, so the `14:29` row is the forecast issued when that candle closed, at 14:30. Anyone can replay a checkpoint with `site/core/engine.js` and get the same numbers.

## Run it yourself

You need Node.js 22 or newer. No packages to install.

```bash
npm test                              # unit tests: no look-ahead, model parity, the online learners
node engine/run.mjs --bootstrap       # fetch ~290 days, train, simulate 30 days (about 8 minutes)
node engine/backtest.mjs --days 365   # the one-year backtest (downloads Binance archive files, ~35 minutes)
node engine/run.mjs                   # a normal run: replays everything since the last one
node engine/serve.mjs                 # preview on http://localhost:8787
```

To run your own copy, fork the repo, set **Settings → Pages → Source** to **GitHub Actions**, and enable the workflow.

```
site/            the static site on GitHub Pages
  core/          the shared engine: signals, models, online learning, scoring
  assets/        the page: live feed, chart, app
engine/          Node only: data fetching, training, boosting, evolution, the pipeline
data/            the public record
test/            unit tests
```

## Credits

Market data comes from Binance's public API (`data-api.binance.vision`) and the Crypto Fear & Greed Index from [alternative.me](https://alternative.me/crypto/fear-and-greed-index/). This project isn't affiliated with Binance or alternative.me.

MIT licensed. Made by Omid Abduli.
