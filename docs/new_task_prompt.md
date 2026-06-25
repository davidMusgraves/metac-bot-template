# Orientation prompt for the new task

Paste the block below to start the new task in the `metac-bot-template` folder.

---

I'm working on my forked Metaculus tournament bot in this repo
(`metac-bot-template`), a fork of `Metaculus/metac-bot-template`. It's a
`forecasting-tools` `ForecastBot` subclass (`SummerTemplateBot2026` in `main.py`)
that runs in the FutureEval / AI Forecasting Benchmark tournament via GitHub
Actions. I've created a Metaculus bot account and have a `METACULUS_TOKEN`.

I have two sibling repos with components I've already built and tested that I want
to use to improve this bot:
- `Superforecaster` — a per-category Bayesian **calibrator**
  (`forecaster/models/calibrator.py`), a **prior-update logistic model**, **scoring**
  (Brier + log score + calibration curves, `forecaster/score/scoring.py`), a
  **GDELT/RSS news client** and an **LLM headline classifier**
  (`forecaster/evidence/`).
- `KalshiPaperTrader` — where those came from (reference only).

**Read `docs/integration_plan.md` in this repo first** — it has the full plan, the
template's injection points, and a component→bot mapping.

Goals, in order:
1. **Get the bot running:** help me set up secrets, run the `Test Bot` workflow,
   and confirm a forecast posts to my bot's Metaculus profile from the
   `bot-testing-area` tournament. Pause the live 20-min workflow while we develop.
2. **Stand up an offline backtest/scorer** on *resolved* Metaculus questions using
   the `Superforecaster` scoring module — Brier + log score + calibration vs the
   community-prediction baseline, with strict no-look-ahead. This is the yardstick
   for every later change.
3. **Add the calibration layer:** collect my bot's resolved track record, fit the
   per-category calibrator, and wrap the aggregated probability before submit —
   deploy only if leave-one-out Brier improves.
4. Then iterate on base-rate priors, structured news evidence, ensemble/aggregation,
   and a per-category meta-report — each change measured offline before going live.

Important constraints: this is the AI Benchmark, so **calibration is the highest
lever** and I must **never anchor live forecasts to the community prediction** (the
crowd is only the offline backtest benchmark). Keep changes small and measured;
treat "no improvement" as a real result; no look-ahead in backtests.

Start by checking the current state of the repo and walking me through finishing
Part A (getting a test forecast to post). Use a task list.
