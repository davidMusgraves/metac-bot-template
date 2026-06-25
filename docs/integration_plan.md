# Metaculus Bot — Run It, Then Improve It With Our Toolkit

This is the working plan for the new task: **(A)** get the forked Metaculus bot
running in the FutureEval / AI Forecasting Benchmark tournament, then **(B)** layer
in components we've already built and tested in the `Superforecaster` and
`KalshiPaperTrader` repos to make it more calibrated and measurable.

## Context — three repos

- **`metac-bot-template`** (this fork) — the tournament bot. `main.py` defines
  `SummerTemplateBot2026(ForecastBot)` from the `forecasting-tools` package. Runs
  via GitHub Actions (every 20 min on the live tournament + MiniBench).
- **`Superforecaster`** — our calibrated-forecaster repo: a ported Bayesian
  **calibrator** (per-category Beta-Binomial), a **prior-update logistic model**,
  **scoring** (Brier + log score + calibration curves), a **GDELT/RSS news client**,
  and an **LLM headline classifier**. These are the parts that plug into the bot.
- **`KalshiPaperTrader`** — origin of the above; reference only.

## How the template works (injection points)

Flow per question (from `main.py`): load question → `run_research()` ×
`research_reports_per_question` → a `_run_forecast_on_<type>()` ×
`predictions_per_research_report` → **aggregate predictions** → **submit**. Only
`run_research` and the forecast methods must be implemented; everything else can be
overridden. LLMs are configured via the `llms=` param (`default`, `researcher`,
`summarizer`, `parser`).

Three natural seams for our toolkit:
1. **`run_research()`** — inject a base-rate/outside-view prior and structured news
   evidence into the research text the forecaster sees.
2. **the forecast methods / aggregation** — wrap the final aggregated probability
   through our **calibrator** before it's submitted.
3. **offline** — score prompt/model/calibration changes on *resolved* Metaculus
   questions before deploying (our scoring + no-look-ahead harness).

---

## Part A — Get the bot running (do first)

1. **Secrets** (GitHub → Settings → Secrets and variables → Actions): add
   `METACULUS_TOKEN` (from the bot account you created) and an LLM key —
   `OPENROUTER_API_KEY` (or `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`). Optional search:
   `ASKNEWS_SECRET`, `EXA_API_KEY`, `PERPLEXITY_API_KEY`.
2. **Enable Actions** in the fork.
3. **Smoke test:** Actions → `Test Bot` → Run workflow. It forecasts on the
   `bot-testing-area` tournament. Confirm forecasts appear on your bot's Metaculus
   profile (~3–5 min).
4. **Local dev (optional, for fast iteration):** `poetry install`, copy
   `.env.template` → `.env`, fill keys, `poetry run python main.py` (defaults to
   tournament mode; use the test tournament while iterating).
5. The `Forecast on new AI tournament questions` workflow is already enabled and
   runs every 20 min. **Pause it** while developing so you don't post half-baked
   forecasts: Actions → that workflow → Disable.

**Done when:** a test-tournament forecast posts end-to-end from your fork.

---

## Part B — Improve the bot with our toolkit

### B0. The honest framing (read first)

- The AIB scores you with a **proper score** against a baseline/peer — **calibration
  is the highest-leverage thing**, which is exactly what we measured and built for.
- **No crowd-peeking on live questions.** Our "market/crowd-as-prior" idea was for
  trading; you must *not* anchor live tournament forecasts to the community
  prediction. For the bot, the prior comes from **base rates + reasoning**, not the
  crowd. The crowd/community prediction is only used **offline, as the backtest
  benchmark** on already-resolved questions.
- Same discipline as before: **no look-ahead** in backtests, **regularize** for
  small samples, **don't overfit** prompts to a handful of questions, and treat
  "no improvement" as a real result.

### B1. Calibration layer — *first improvement, highest value, lowest risk*

The bot's raw probabilities are likely miscalibrated (we measured exactly this in
weather: predicted 0.97, won 0.64). Fix it with the ported per-category
Beta-Binomial calibrator.

- **Collect the track record:** pull your bot's resolved predictions from the
  Metaculus API (your forecasts + the resolutions).
- **Fit** a `CalibratorSet` (from `Superforecaster/forecaster/models/calibrator.py`)
  keyed by category (binary first; numeric/MC later).
- **Apply** the calibration map to the aggregated probability *before submit* — wrap
  it in the binary forecast path (or override the aggregation step).
- **Prove it helps** with the **leave-one-out Brier** we already built — only deploy
  if LOO improves. Re-fit periodically as more questions resolve.

Start binary-only; it's where the calibrator maps cleanly.

### B2. Offline backtest / scoring harness — *iterate without burning reputation*

Before any prompt or model change goes live, score it on **resolved** Metaculus
questions:
- Pull resolved questions + the community prediction (the benchmark).
- Run the bot's research+forecast on them with **strict no-look-ahead** (only use
  information dated ≤ the question's forecast window).
- Score with `Superforecaster/forecaster/score/scoring.py` — **Brier + log score +
  calibration curve**, vs the **community baseline** and a base-rate baseline.
- Adopt a change only if it beats the incumbent out-of-sample.

This is the single most important habit; everything else plugs into it.

### B3. Base-rate / outside-view priors in `run_research`

Prepend an explicit **reference-class base rate** to the research the forecaster
reads ("Outside view: base rate for <reference class> ≈ X%, because…"). Use the
`Superforecaster` base-rate scaffold + an LLM outside-view estimate with *explicit
reference-class reasoning* (not a vibe). Anchoring to a base rate is one of the
best-documented ways to improve calibration.

### B4. Structured news evidence

Augment `run_research` with our **GDELT/RSS news client** + **LLM headline
classifier** (`Superforecaster/forecaster/evidence/`) to add a structured,
timestamped evidence summary alongside whatever search provider (AskNews/Exa) the
template uses. Keep strict timestamp discipline for backtests.

### B5. Ensemble / aggregation tuning

The template already aggregates multiple predictions. Experiment with
`research_reports_per_question` / `predictions_per_research_report`, multiple models
in `llms=`, and a smarter aggregator (e.g., weight by per-model track record), then
**calibrate the aggregate** (B1). Measure every change via B2.

### B6. Meta-layer — target where you're failing

Reuse the per-category calibration + Brier tracking to build a small report: which
categories/horizons is the bot worst-calibrated or worst-scoring on? Then focus
prompts/priors/models there, or **abstain-to-prior** where you have no edge. This is
the closed-loop "find the weak slice and fix it" idea from the design doc.

### Component → injection map

| Our component (`Superforecaster/forecaster/…`) | Where it plugs into the bot |
|---|---|
| `models/calibrator.py` (`CalibratorSet`) | wrap aggregated prob before submit (B1) |
| `score/scoring.py` (Brier, log score, calibration) | offline backtest harness (B2) |
| `priors/base_rate.py` + LLM outside view | prepend to `run_research()` (B3) |
| `evidence/news_client.py`, `evidence/llm_features.py` | augment `run_research()` (B4) |
| per-category calibration tracking | meta-report (B6) |

---

## Suggested order & first step

1. **Part A** — get a test forecast posting (today).
2. **B2 skeleton** — wire the offline scorer on resolved questions (so every later
   change is measurable).
3. **B1** — calibration layer (first real score improvement).
4. **B3/B4** — priors + structured evidence.
5. **B5/B6** — ensemble tuning + meta-layer.

**First concrete step:** finish Part A (smoke-test forecast posts), then stand up
the B2 offline scorer against a batch of resolved questions with the community
prediction as the baseline — that's the yardstick everything else is judged by.

## Risks / honest notes

- Calibration needs a **track record**; early on you have few resolved questions, so
  lean on the prior (the calibrator already shrinks to identity when data is thin).
- **Don't overfit** prompts to a small question set; the LOO/backtest guards this.
- **Reproducibility:** GitHub Actions uses repo secrets and `poetry.lock`; pin
  models and keep changes small and measured.
- **Rate limits / cost:** more research reports × predictions × models = more API
  spend and rate-limit pressure; scale deliberately.
- The bot must stay within tournament rules (no crowd-peeking on live questions;
  see the [resources page](https://www.metaculus.com/notebooks/38928/futureeval-resources-page/)).
