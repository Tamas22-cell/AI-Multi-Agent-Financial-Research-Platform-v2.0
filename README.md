# AI Multi-Agent Financial Research Platform v2.0

> Autonomous multi-agent market intelligence: nine specialist agents, an orchestrator with consensus scoring and regime detection, a risk-agent veto layer, local-LLM bull/bear debate, adaptive agent weighting, forward validation and a 9-panel terminal dashboard — all generated from a single run.

![dashboard](docs/ai-multiagent-platform-v2-hero.jpg)

---

## Table of contents

1. [What it does](#what-it-does)
2. [Architecture](#architecture)
3. [Specialist agents](#specialist-agents)
4. [Decision pipeline](#decision-pipeline)
5. [Research coordination & LLM review](#research-coordination--llm-review)
6. [Adaptive weights & leaderboard](#adaptive-weights--leaderboard)
7. [Historical validation (backtesting)](#historical-validation-backtesting)
8. [Portfolio Intelligence](#portfolio-intelligence)
9. [Audit trail](#audit-trail)
10. [Dashboard](#dashboard)
11. [Getting started](#getting-started)
12. [Configuration](#configuration)
13. [Project structure](#project-structure)
14. [Roadmap](#roadmap)
15. [Disclaimer](#disclaimer)

---

## What it does

Every run pulls live market data, lets nine specialist agents assess it independently, aggregates their scores into a **consensus market score** (0–1), classifies the **market regime** (BULLISH / MIXED / BEARISH) and emits a single **BUY / HOLD / AVOID** decision. The decision is not final until a dedicated **Risk Agent** authorizes it — it can veto or downgrade when critical thresholds or data-quality restrictions fire.

Around that core the platform adds what a real research desk needs: agents challenge each other's evidence, a local LLM argues the bull and bear case, agent influence is re-weighted by *measured* forward accuracy, every decision is stored and validated on 1D / 7D / 30D horizons against SPY and BTC-USD, and every message between agents is logged.

Example output (single run):

| Metric | Value |
|---|---|
| Final decision | **HOLD** |
| Market regime | MIXED |
| Consensus score | 0.52 |
| Avg. confidence | 0.82 |
| Votes | 77.8 % neutral · 11.1 % bullish · 11.1 % bearish |
| Risk veto | none |

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                          Orchestrator                              │
│  routes analyze() → collects responses → consensus → regime        │
└────────────┬──────────────────────────────────────────┬────────────┘
             │ analyze / response (message_id, reply_to) │
   ┌─────────▼─────────┐                       ┌────────▼────────┐
   │  9 specialist     │                       │  Research       │
   │  agents           │◄──── explain /────────│  coordinator    │
   │  (score, conf,    │      challenge        │  (rule-based)   │
   │   evidence)       │                       └────────┬────────┘
   └─────────┬─────────┘                                │
             │                                 ┌────────▼────────┐
             │                                 │  Local LLM      │
             │                                 │  bull / bear /  │
             │                                 │  synthesis      │
             │                                 └────────┬────────┘
   ┌─────────▼──────────────────────────────────────────▼────────┐
   │  Adaptive weights  ·  Market memory  ·  Risk Agent (veto)   │
   └──────────────────────────────┬──────────────────────────────┘
                                  │ BUY / HOLD / AVOID
   ┌──────────────────────────────▼──────────────────────────────┐
   │  Forward validation (1D/7D/30D)  ·  Portfolio Intelligence  │
   │  Audit trail  ·  JSON report  ·  HTML dashboard             │
   └─────────────────────────────────────────────────────────────┘
```

**Stack:** Python 3.11+, LangGraph / CrewAI for orchestration, Ollama-hosted local LLM (`qwen3:8b` by default), Pandas / NumPy for analytics, plain HTML + Canvas for the dashboard (no frontend build step).

---

## Specialist agents

Each agent receives the same market snapshot, returns `score ∈ [0,1]`, `confidence ∈ [0,1]`, a verdict and structured evidence entries (`A<n>-positive_evidence-<k>`, `A<n>-risks-<k>`) that downstream stages can reference.

| # | Agent | Inputs | Benchmark |
|---|---|---|---|
| A1 | **Macro** | USD strength, Treasury yields, S&P 500 / NASDAQ momentum (1D, 7D) | SPY |
| A2 | **Stock** | Broad indexes, tech leadership, small-cap participation, relative strength | SPY |
| A3 | **Crypto** | 1D / 7D momentum, volume confirmation, BTC/ETH relative strength, breadth | BTC-USD |
| A4 | **OnChain** | Hashrate, BTC price momentum, network transaction activity | BTC-USD |
| A5 | **Derivatives** | BTC/ETH funding, funding trend, open-interest change, positioning risk | BTC-USD |
| A6 | **Technical** | RSI, EMA structure, MACD + histogram, short-term momentum | SPY |
| A7 | **News** | Weighted bullish/bearish headlines, concentration, uncertainty, severity | SPY |
| A8 | **Geopolitical** | Escalation / de-escalation, sanctions, energy & supply-chain disruption | SPY |
| A9 | **Risk** | Cross-asset volatility, correlation, worst single-asset drawdown | SPY |

Scores ≥ 0.60 read as bullish, ≤ 0.40 as bearish, otherwise neutral.

---

## Decision pipeline

1. **Collect** — the orchestrator sends `analyze` to each agent sequentially and records the `response` (see [Audit trail](#audit-trail)).
2. **Weight** — each score is multiplied by the agent's *adaptive* weight (base weight × performance multiplier, see below).
3. **Consensus** — weighted average → market score; vote shares (bullish / neutral / bearish) by verdict.
4. **Regime** — BULLISH / MIXED / BEARISH from the score band and vote distribution.
5. **Market memory** — the score is compared with a 30-day history (historical average, previous score, score & confidence trend, dominant decision / regime).
6. **Proposed decision** — BUY / HOLD / AVOID.
7. **Risk Agent review** — final authorization: `veto_active`, `veto_type`, `decision_changed`. A missing-data restriction blocks BUY authorization without asserting critical market risk.

---

## Research coordination & LLM review

A rule-based coordinator issues **explain** and **challenge** tasks to the agents whose evidence conflicts most (e.g. OnChain vs Geopolitical), then runs reciprocal **evidence reviews**. Stats per run: messages, research tasks, reruns attempted / accepted, review pairs, request errors, confidence threshold (default 0.6) and the list of uncertain agents.

Three review roles are then executed by the local LLM:

| Role | Output |
|---|---|
| Bullish case | assessment · counter-argument · scope & evidence limitations · referenced evidence ids |
| Bearish challenge | same structure, opposite side |
| Research synthesis | unresolved discrepancies, comparability limits |

These arguments are **advisory only** — they never modify agent scores and cannot authorize the final decision.

---

## Adaptive weights & leaderboard

Agent influence starts from configured base weights and is adjusted from **measured forward performance**:

* `multiplier = f(weighted_accuracy, samples)`, clipped to `max_adjustment` (default ±20 %)
* below `min_samples` (default 10) an agent is **PENDING** (no adjustment) or **PROVISIONAL** (limited adjustment)
* the **Agent Leaderboard** ranks agents by raw / weighted accuracy and score against their benchmark

Example: after one sample, OnChain 1.100 → 1.122 (+2 %), Crypto 1.000 → 0.980 (−2 %), Derivatives 1.100 → 1.078 (−2 %); all others pending.

---

## Historical validation (backtesting)

Real stored decisions are tracked on **1D / 7D / 30D** horizons — no synthetic signal backfill.

* Strategy equity curve from completed 1D observations; SPY and BTC-USD as benchmarks
* Statistics: completed / winning / losing / flat periods, win rate, exposure rate, profit factor, Sharpe-like
* Horizon status, valid / invalid snapshots
* **Decision validation** table: BUY / HOLD / AVOID performance per horizon
* Methodology: BUY = SPY exposure, HOLD = cash, AVOID = cash (never short); 7D and 30D are evaluated independently and are not compounded into the equity curve

---

## Portfolio Intelligence

Combines position weights, historical performance, volatility, drawdown, concentration, cross-asset correlation and target allocation into one health / risk view:

* Health score, risk score, health level (e.g. FAIR / MODERATE RISK)
* Annual return & volatility, max drawdown, Sharpe-like, Sortino, downside volatility, daily historical VaR / CVaR 95 %
* Diversification: top position, effective positions, concentration, average correlation
* Holdings table (ticker, asset class, sector, weight, market value, volatility)
* **Rebalancing Intelligence**: INCREASE / REDUCE / HOLD per position vs target allocation

---

## Audit trail

Every orchestrator ↔ agent exchange is stored with `message_id`, `reply_to`, ISO timestamp and payload. The dashboard derives per-agent round-trip latency from it (e.g. news 5.2 s, geopolitical 4.4 s out of a 20.4 s sequential chain) — useful for spotting slow data sources and for deciding what to parallelize.

---

## Dashboard

The run produces a JSON report and an HTML dashboard. Two layouts ship in `reports/`:

| Template | Layout |
|---|---|
| `terminal.html` (default) | 3 × 3 terminal grid — Global Market Pulse · Live Decision Panel · Signals & Risks · Specialist Scorecards + Leaderboard · Backtesting · Portfolio Intelligence · Research Coordination + Adaptive Weights · Bull–Bear Debate · Audit Trail |
| `dashboard_template.html` | long-form report layout |

```bash
# render the terminal dashboard from a run's JSON
python reports/render_dashboard.py reports/output/report.json reports/output/dashboard.html

# long-form layout
python reports/render_dashboard.py reports/output/report.json reports/output/report.html --report-layout

# print the full JSON schema the templates expect
python reports/render_dashboard.py --schema > docs/report.schema.json
```

Templates embed a default report between `/*REPORT_START*/ … /*REPORT_END*/`; the renderer deep-merges your JSON over it, so partial exports still render. All charts are plain Canvas — no external JS, works offline.

---

## Getting started

```bash
git clone https://github.com/<you>/MultiAgentFinancialResearch.git
cd MultiAgentFinancialResearch
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# local LLM for the bull/bear review
ollama pull qwen3:8b

cp .env.example .env        # API keys for market / news data providers
python main.py              # full run → reports/output/report.json + dashboard.html
```

---

## Configuration

| Setting | Default | Purpose |
|---|---|---|
| `LLM_MODEL` | `qwen3:8b` | Ollama model for the review roles |
| `CONFIDENCE_THRESHOLD` | `0.6` | below this an agent is flagged uncertain and may be re-queried |
| `MIN_SAMPLES` | `10` | forward samples before an agent's weight is fully adaptive |
| `MAX_ADJUSTMENT` | `0.20` | cap on adaptive weight change (±) |
| `MEMORY_LOOKBACK_DAYS` | `30` | market-memory window |
| `BASE_WEIGHTS` | Macro 1.2 · Stock 1.0 · Crypto 1.0 · OnChain 1.1 · Derivatives 1.1 · Technical 0.9 · News 0.7 · Geopolitical 0.8 · Risk 1.3 | starting agent influence |

---

## Project structure

```
MultiAgentFinancialResearch/
├── agents/               # nine specialist agents + risk agent
├── orchestrator/         # routing, consensus, regime, market memory
├── coordination/         # explain / challenge tasks, evidence review
├── review/               # local-LLM bull / bear / synthesis roles
├── evaluation/           # forward validation, leaderboard, adaptive weights
├── portfolio/            # portfolio intelligence & rebalancing
├── data/                 # market, on-chain, derivatives, news providers
├── reports/
│   ├── terminal.html             # 3×3 dashboard template
│   ├── dashboard_template.html   # long-form template
│   ├── render_dashboard.py       # JSON → HTML
│   └── output/                   # generated report.json / dashboard.html
├── main.py
└── requirements.txt
```

*(Adjust the tree to your actual module names.)*

---

## Roadmap

* Parallel agent dispatch (`asyncio.gather`) — cuts the ~20 s sequential chain to ~5 s
* Export OHLCV / on-chain / funding series into the report for chart panels
* Scheduled runs + Discord / Telegram delivery of the decision summary
* Larger forward-sample base → confirmed (non-provisional) leaderboard

---

## Disclaimer

This software is for research and educational purposes only. It is not financial advice and does not execute trades. Past performance of any agent or strategy is not indicative of future results.

---

**Author:** Tamas Nemeth · AI & Quantum Computing · Multi-Agent Systems ·
