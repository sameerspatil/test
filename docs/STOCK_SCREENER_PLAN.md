# Stock Screener System — Architecture & Implementation Plan

**Status:** Design locked from call (2026-11-19, America/Los_Angeles)  
**Scope:** Technical reversal filters + Druckenmiller-style macro liquidity chain + direction/magnitude prediction + 20-year walk-forward backtest + evals  
**Repo:** Greenfield (`sameerspatil/test`)

---

## 1. Product goal

Build an easy-to-use stock screening system that:

1. Filters equities using **technical reversal indicators**.
2. Biases the universe using a **macro liquidity chain** (Fed / jobs / inflation → overnight rates → liquidity → sectors).
3. Predicts, per stock-day: **direction (up/down)** and **magnitude (% move)** at **1-day, 7-day, and 50-day** horizons.
4. Is validated by a **20-year walk-forward backtest** with automated correctness, statistical, robustness, and operational evals.

**Out of scope for v1:** Robinhood as a data/API backbone; multi-agent news readers inside the backtest loop; company-level fundamental statement screens (P/E, D/E, margins) as the primary “fundamental” layer (macro-only fundamentals for v1).

---

## 2. Decisions locked from the call

| Topic | Decision |
|---|---|
| Data backbone | **Not Robinhood** (no official equities screening API; S&P movers = top 10 only; ToS risk on unofficial APIs) |
| “Duncan Miller” macro | **Likely mishearing of Stanley Druckenmiller**; no named “Duncan Miller style” methodology found |
| Fundamentals (v1) | **Macro-only** liquidity chain, not company financial statements |
| Technicals (v1) | RSI, MACD, MA crossovers, Bollinger, Stochastic (candles/Ichimoku later) |
| Headline MA signal | **50/200 golden & death cross** + shorter pairs (20/50, 9/20) |
| Prediction outputs | Direction + magnitude × horizons 1 / 7 / 50 |
| Model structure | **Separate** direction vs magnitude models; **separate** model per horizon (6 boosters) |
| Model type (v1) | LightGBM (+ rules baseline + logistic baseline) |
| Macro encoding | **Regime labels + changes/surprises**, not raw levels alone |
| News (v1) | Deterministic text→features if/when archive exists; **agents only for live/research**, not official backtest |
| Eval core | Direction hit/miss + magnitude within **τ = 20%** relative error (tunable); separate + combined leaderboards |
| Engine philosophy | **Deterministic core, agent shell** |

---

## 3. System architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                     Ingestion (PIT)                         │
│  Prices/OHLCV │ S&P membership │ FRED macro │ Sector ETFs  │
│  (optional) News archive with first_seen timestamps         │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Feature store  (one row = ticker × date)       │
│  Technicals │ Cross-sectional z │ Macro regimes/Δ │ Gates   │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌──────────────────┐    ┌─────────────────────────────────────┐
│ Deterministic    │    │ Models (per horizon h ∈ {1,7,50})   │
│ macro regime →   │───▶│  LightGBM_dir_h  → P(up)            │
│ sector allow-list│    │  LightGBM_mag_h  → |R̂|_h            │
└──────────────────┘    └──────────────────┬──────────────────┘
                                           ▼
                            Screener / signals + scorecards
                                           ▼
                            Walk-forward backtest + eval gates
```

### Pipeline stages

1. **Universe:** Point-in-time S&P 500 (preferred). Survivorship-biased current list only for smoke tests, clearly labeled.
2. **Technicals:** Consensus indicators + threshold flags.
3. **Macro chain:** Jobs/inflation → Fed bias → overnight/repo path → liquidity regime → sector bias.
4. **Models:** Direction classifier + magnitude regressor per horizon.
5. **Screen:** Combine model scores with optional hard sector gate; emit signals.
6. **Backtest/evals:** Walk-forward; HitDir / HitMag / HitBoth; promote only OOS winners.

---

## 4. Data sources

### 4.1 Price / volume / universe

| Need | Source (practical v1) | Notes |
|---|---|---|
| Daily OHLCV | Yahoo Finance / Polygon | Adjusted closes consistently |
| S&P 500 membership | Point-in-time vendor or historical lists | Avoid current-only membership for official results |
| Sector map | GICS as-of date | Needed for regime sector bias |
| Sector proxies | `SPY`, `XLK`, `XLF`, `XLY`, `XLP`, `XLV`, `XLU`, `XLE`, `XLI`, `XLB`, `XHB`/`ITB` | Relative strength vs SPY |

### 4.2 Macro (FRED)

**Jobs / inflation:** `PAYEMS`, `UNRATE`, `ICSA`, `CPIAUCSL`, `CPILFESL`, `PCEPILFE`, `PPIACO`  
**Policy / repo:** `DFEDTARU`, `EFFR`, `IORB`, `RRPONTSYAWARD`, `SOFR`  
**Liquidity:** `WALCL`, `WRESBAL`, `M2SL`, `T10Y2Y`, `DGS10`, `BAMLH0A0HYM2`, `BAMLC0A0CM`, `NFCI`, `VIXCLS`, `DTWEXBGS`

**Availability:** SOFR/IORB are shorter history — use EFFR + spreads + NFCI fallbacks with an availability mask.

### 4.3 Robinhood (explicit non-choice)

- No official stocks screening API; crypto API only is official.
- Unofficial libraries (`robin_stocks`, `pyhood`) expose quotes, light fundamentals, OHLCV, and **top-10** S&P movers — not full S&P membership or deep fundamentals.
- In-app screeners exist for humans only.
- **Do not** build v1 on Robinhood.

### 4.4 Optional news (phase 2 data)

Timestamped headline archives (Refinitiv/RavenPack/Benzinga/etc.). Free 20y full-text is unrealistic. Prefer `first_seen_at` headlines for PIT safety.

---

## 5. Druckenmiller-style macro liquidity chain

```text
Liquidity crunch detection
        │
Jobs + inflation (Fed reaction function)
        │
Fed adjusts overnight / repo corridor (EFFR, IORB, ON RRP, SOFR)
        │
Market liquidity shifts (spreads, NFCI, M2, reserves, USD, curve)
        │
Sector bias (allow-list / relative strength)
        │
Stock selection (technicals + models inside allowed set)
```

### Regime labels (v1)

- `liquidity_regime ∈ {crunch, tight, neutral, ample}`
- `fed_bias ∈ {hawkish, neutral, dovish}`
- Curve: `inverted | flat | steep`

### Sector mapping (illustrative)

| Regime | Bias |
|---|---|
| Crunch / tightening | Defensives: Health Care, Staples, Utilities; cut high-beta Discretionary/spec growth |
| Tight but stable | Value / Financials if curve steepening & spreads stable |
| Easing / ample | Growth, Tech, Discretionary, cyclicals |
| Inflation-hot + loose | Energy / Materials tilt |
| Labor soft + dovish pivot | Rate-sensitive: homebuilders, Discretionary, REITs |

### Macro feature encoding (important)

Use **regimes + changes/surprises + z-scores**, not raw CPI/M2 levels as the primary signal (avoids era-fitting).

---

## 6. Technical indicators & consensus thresholds

### Starter set (ranked for v1 reversal filter)

1. **RSI (14)** — &lt;30 oversold / &gt;70 overbought (strict variant 20/80)
2. **MACD (12,26,9)** — signal-line cross or histogram sign flip
3. **MA crossovers** — **50/200 golden & death cross**; also 20/50, 9/20
4. **Bollinger (20, 2σ)** — outside band then close back inside
5. **Stochastic (14,3,3)** — &lt;20 / &gt;80 with %K/%D cross

Defer: candlestick patterns, Ichimoku, ADX-as-direction (ADX as exhaustion filter later).

### Good combos

- RSI + MACD (core)
- RSI + Bollinger re-entry
- MA cross + volume expansion
- ADX rollover as optional gate later
- Avoid stacking RSI + Stochastic (redundant oscillators)

---

## 7. Prediction problem design

### Outputs per stock-day

| Horizon | Direction | Magnitude |
|---|---|---|
| 1 trading day | up/down | predicted % move |
| 7 trading days | up/down | predicted % move |
| 50 trading days | up/down | predicted % move |

### Architecture choices (opinionated)

| Question | Choice | Why |
|---|---|---|
| Direction vs magnitude | **Separate models** | Different objectives; joint MSE ignores side |
| Horizons | **Separate models** (shared features) | 1d ≠ 50d noise scale |
| Model type | **LightGBM** (+ rules & logistic baselines) | Tabular, fast, interpretable enough |
| Not v1 | LSTM/TFT, horizon-as-feature, agents scoring returns | Leakage, cost, weak CI |

### Feature matrix

One row = `(ticker, date)` with **strictly point-in-time** features:

- **Technicals:** RSI, MACD, Stoch, BB %B/flags, MAs, crosses, ATR%, volume ratio
- **Cross-sectional:** within-sector z-scores / ranks same day
- **Macro:** regime one-hots, Δ rates/spreads/M2/NFCI, real rate, curve, availability masks
- **Stock×macro:** sector id, sector RS vs SPY, `allowed_by_regime`
- **Controls (diagnosis):** `ret_21`, `ret_63` — used to detect “momentum disguised as alpha”

### Labels

```text
R_h     = close[t+1+h] / close[t+1] - 1     # no same-bar leakage
y_dir_h = 1 if R_h > +ε else 0              # flats |R|<ε excluded from dir denom
y_mag_h = R_h                               # mag model uses |R| / quantile regression
```

Optional parallel target: excess vs SPY / sector to fight market-beta “alpha.”

### Anti-momentum controls

Must beat OOS:

1. Sign of 21d/63d return baseline  
2. Tech-rules-only screen  
3. Tech + momentum-only GBM  

Reject configs whose importance is dominated by `ret_*` alone.

---

## 8. Scoring & leaderboards

### Direction

- **Hit** if `sign(predicted) == sign(actual)` and actual ≠ flat  
- **HitDir_h** = hits / scored signals  

### Magnitude

- Relative error `e_h = |R̂_h - R_h| / max(|R_h|, floor)` with floor ≈ 0.5%  
- **Magnitude hit** if `e_h ≤ τ`  
- **Default τ = 0.20** (tunable: 0.10–0.50)  
- Prefer signed error; require direction hit before counting “useful” magnitude (report both conditional and unconditional)

### Combined (per signal, per horizon)

```text
C_h = 1[dir hit] * max(0, 1 - e_h/τ)     # continuous
HitBoth_h = dir hit AND mag hit          # binary
```

Wrong direction ⇒ combined score 0.

### Aggregation

- Overall HitDir, HitMag, HitBoth per horizon  
- Avg return per direction-correct signal: `mean(s * R_h | dir hit)`  
- Avg return per combined-correct signal  
- Expectancy over all signals  
- Slice by sector / liquidity regime  

### Leaderboards (keep separate)

1. Direction-only  
2. Magnitude-only  
3. Combined (primary promote: **OOS HitBoth_7** with floor on HitDir_7 and positive avg return on dir-correct)

---

## 9. Backtest experiment plan

### Design

- **Warm-up:** first ~5 years for indicator history (no score)  
- **Walk-forward:** e.g. 3y train → 1y test, yearly step; **purged CV** with embargo ≥ horizon  
- **Official metrics:** concatenated OOS folds only  
- **Held-out final year:** never used for tuning  
- Full-period in-sample fit = diagnostic only  

### Parameter search (train folds only)

1. Fix equal tech weights → tune thresholds  
2. Freeze thresholds → tune weights  
3. Add macro hard gate → soft regime multipliers  
4. Light per-fold re-tune vs frozen global params (robustness check)  

Primary optimize: **7d HitDir / HitBoth** with minimum signals/year; constraints on 50d mean excess and drawdown.

### First experiment (`exp001_7d_direction_macro_gate`)

Comparators:

1. Rules-only  
2. LGBM tech-only  
3. LGBM tech + momentum controls  
4. LGBM tech + macro regimes/Δ  
5. Same as 4 with **excess-vs-SPY** labels  

**Success bar:** OOS HitDir_7 beats rules & tech-only by ≥2 pp; excess-label HitDir also ≥ tech-only; train−OOS gap &lt;5 pp; importance not pure momentum.

Then: `exp002` magnitude @ τ=0.20; `exp003` 50d and 1d.

### Python stack

`pandas`, `numpy`, `pyarrow`, `pandas_ta`/`ta`, `fredapi`, price vendor, `lightgbm`, `optuna`, `joblib`  
Storage: parquet feature store; precompute features once; sweeps only re-threshold/re-weight.

---

## 10. Eval checklist

### Correctness (CI-blocking)

| ID | Check |
|---|---|
| C1 | Features at *t* recompute from history ≤ *t* |
| C2 | Labels start at *t+1* (no same-bar leak) |
| C3–C4 | Macro uses release/vintage dates, not period-end peek |
| C5 | Macro asof ≤ price date |
| C6 | Point-in-time universe |
| C7 | Indicator warm-up respected |
| C8 | SOFR/IORB fallbacks masked before availability |
| C9 | Sector map as-of |
| C10 | Deterministic re-run hash |
| C11–C14 | Prediction schema, sign/flat rules, τ logged |

### Statistical (promote gates)

| ID | Check |
|---|---|
| S1a–c | HitDir / HitMag / HitBoth vs null baselines |
| S3 | Train–test gap on each primary metric |
| S4 | Param budget vs OOS signal count |
| S5 | Multiple-testing / held-out year |
| S6 | 7d and 50d both above null (not 1d-only) |
| S7 | Ablation: combined vs tech-only |
| S9–S10 | Calibration; avg return on correct signals |

### Robustness

| ID | Check |
|---|---|
| R1–R2 | Threshold/weight jitter |
| R3–R4 | Bull/bear/sideways/high-vol + crisis windows |
| R5 | Cost stress (5–20 bps) |
| R7 | τ sensitivity curve around 0.20 |

### Operational

| ID | Check |
|---|---|
| O1–O2 | Freshness; failed pulls fail loudly |
| O3–O5 | Schema, NaN policy, calendar holes |
| O6–O8 | Universe size, recon checksums, runtime SLA |
| O9–O10 | Prediction coverage; publish three scorecard JSONs |

### Agents vs deterministic

| Part | Owner |
|---|---|
| Ingest, features, backtest, scoring, promote gates | **Deterministic** |
| Exploring combos, interpreting failures, live news triage | **Agent-assisted** → writes config → deterministic runner |

---

## 11. Unstructured / news strategy

### Priority for this project

| Priority | Source | Actionability |
|---|---|---|
| P0 now | Structured tech + macro only | Full 20y backtest feasible |
| P1 | Options implied sentiment (put/call, unusual activity) | Numeric, PIT-friendly if timestamped |
| P2 | News headlines → frozen NLP → stock-day features | Needs paid/partial archive |
| P3 | 8-K / earnings transcripts | Event-dense; alignment heavy |
| P4 | Analyst targets | Useful but sticky/lagged |
| P5 | Social (X/Reddit/StockTwits) | Noisy; weak for clean 20y PIT |

### News: fine-tune LM features vs multi-agent

| | LM → features (batch) | Multi-agent reader |
|---|---|---|
| Backtest | Feasible | Costly / nondeterministic |
| GBM join | Clean | Only if cached JSON schema |
| PIT safety | Enforceable with first_seen | Fragile without frozen corpus |
| **v1 choice** | **Yes** | Live/research shell only |

**Hybrid:** Official scorecard always uses cached deterministic news features; agents improve labels/prompts offline and assist live watchlists.

---

## 12. Minimal working prototype (implementation order)

### Phase A — Skeleton screener (week 1 engineering target)

1. Ingest daily prices for a fixed liquid universe (smoke: current S&P or top 100).  
2. Compute RSI/MACD/BB/Stoch/MAs; apply consensus thresholds.  
3. CLI or simple table: “pass/fail + why.”  
4. Unit tests for no look-ahead on a synthetic series.

### Phase B — Macro gate

1. Pull FRED series; release-lag join.  
2. Emit `liquidity_regime` + `allowed_sectors`.  
3. Filter Phase A results through the gate.  
4. Correctness tests C3–C5.

### Phase C — Labels + rules backtest

1. Attach R_1 / R_7 / R_50.  
2. Score HitDir for rules-only.  
3. Walk-forward harness (even if one param set).

### Phase D — `exp001` LightGBM direction (7d)

1. Build feature parquet.  
2. Run ablations vs baselines.  
3. Publish scorecards; pass/fail vs success bar.

### Phase E — Magnitude + eval pack

1. Quantile magnitude model; τ=0.20 HitMag/HitBoth.  
2. Implement minimal eval pack: C1,C2,C3,C10,S1,S3,S4,S7,R1,O1,O4.

### Phase F — Hardening

1. Point-in-time S&P membership.  
2. Optional news headline features.  
3. Simple UI (Streamlit/local web) on top of CLI core.

**Default stack recommendation:** Python, parquet feature store, LightGBM, FRED + price vendor, CLI first / thin UI second.

---

## 13. Open items to confirm with stakeholders

1. Price/news vendor budget (Yahoo-only vs Polygon + news archive).  
2. Confirm macro thesis = **Druckenmiller-style** (not a different “Duncan Miller”).  
3. Long-only v1 vs long/short.  
4. UI priority: CLI-only prototype vs web table for “easy to use.”  
5. Whether company fundamentals (P/E, growth, D/E, margins) enter in v2.

---

## 14. Success criteria (v1)

- Deterministic pipeline with frozen feature store and reproducible scorecards.  
- Walk-forward OOS **HitDir_7** and **HitBoth_7** beat rules-only and tech-only baselines under eval gates.  
- Macro regime gate demonstrably changes sector mix and improves (or risk-adjusts) OOS results in ablation.  
- Documented non-use of Robinhood as research data plane.  
- Easy path: one command to refresh data → screen today’s names → show why each passed.

---

*This document consolidates the call thread: Robinhood research, reversal indicators/thresholds, Druckenmiller liquidity chain, backtest design, direction/magnitude evals, model architecture, and news/agent recommendations.*
