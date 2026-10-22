# QFI Project #2 — Derivative Pricing Engine & Greeks Calculation

A pricing library and an interactive risk dashboard. Eleven model families,
one uniform interface, 314 tests, and a validation harness that reruns every
closed-form and cross-model check against whatever contract you are looking at.

```bash
pip install -r requirements.txt
streamlit run dashboard/app.py          # the dashboard
pytest                                  # the tests
python -m pricing_engine.validation     # the harness, from the command line
```

---

## Full-stack app (Next.js + FastAPI)

The dashboard also ships as a web app: a Next.js front end in `web/` over a
FastAPI layer in `api/`. Run the two processes side by side:

```bash
pip install -r requirements.txt
uvicorn api.main:app --port 8000        # terminal 1: the pricing API
```

```bash
cd web
npm install                             # once
npm run dev                             # terminal 2: http://localhost:3000
```

The front end proxies `/api/*` to `http://127.0.0.1:8000` (override with the
`API_URL` environment variable). Like the Streamlit app, `api/main.py` holds no
pricing maths: `build_model` is the one constructor every endpoint goes through.

| Page | What it shows |
|---|---|
| **Pricer** | price and Greeks strip, value/P&L vs payoff, every engine side by side, risk metrics, delta and gamma curves, scenario grid, shock ranking, Greeks by engine, delta-gamma approximation |
| **Market & Smile** | live Yahoo chain with implied-vol smile and vega bars, or a synthetic round trip offline |
| **Greeks** | price/Greek sweeps against spot, strike, vol, time and rate; heatmaps; vanna, volga, charm; Greeks across products |
| **Scenarios** | the brief's shock grids, spot × vol matrix, full shock table |
| **Trees** | CRR vs Leisen-Reimer convergence and error, theta stability, American premium, exercise boundary |
| **Monte Carlo** | convergence with confidence bands, variance-reduction ratios, sample paths, payoff histogram, basket correlation |
| **Barrier PDE** | value vs spot, PDE vs Monte Carlo vs closed form, value surface, barrier sensitivity |
| **Validation** | the full harness against the contract in the sidebar |
| **Risk** | strategy board, editable position book, daily P&L attribution |

Black-76 has an **Underlying** selector. *Stock* derives the forward from spot
(spot Greeks, identical to Black-Scholes). *Index* and *Commodity* take a quoted
futures price and a contract multiplier (futures Greeks, rho = −T·V).

`tests/test_api.py` smoke-tests every endpoint for every product.

**Exercise, underlying and product are honoured on every page.** With American
exercise selected, Black-Scholes and Black-76 are switched off, the Trees page
benchmarks against a 2,001-step Leisen-Reimer tree, the Monte Carlo page runs
Longstaff-Schwartz, and the Greeks and Risk pages price on a tree. In futures
mode every page prices on the futures price. The Greeks page sweeps the selected
product with its own engine. `tests/test_api_consistency.py` pins all of this.

---

## Folder structure

```
qfi-pricing-engine/
├── pricing_engine/                 THE LIBRARY — no Streamlit, no network
│   ├── core.py                     MarketInputs, OptionContract, PricingModel, registry
│   ├── models/
│   │   ├── black_scholes.py        Black-Scholes-Merton, closed form
│   │   ├── black76.py              Black-76, spot mode and futures mode
│   │   ├── trees.py                CRR and Leisen-Reimer lattices
│   │   ├── monte_carlo.py          European, Asian, lookback, basket
│   │   └── barrier.py              Crank-Nicolson PDE + Brownian-bridge MC
│   ├── analytics/
│   │   └── closed_forms.py         ground truth: Asian, lookback, barrier, basket
│   ├── calibration/
│   │   └── implied_vol.py          implied vol, surface, implied dividend yield
│   ├── market_data/
│   │   └── yahoo.py                collector + chain cleaner  (optional, needs network)
│   ├── scenario/
│   │   └── shocks.py               shock grids + PositionBook
│   ├── strategies/
│   │   └── multi_leg.py            eight strategies + P&L attribution backtest
│   └── validation/
│       ├── checks.py               35 checks across 11 model families
│       └── __main__.py             `python -m pricing_engine.validation`
│
├── dashboard/                      THE FRONT END
│   ├── app.py                      layout and the eight tabs
│   ├── state.py                    sidebar -> Settings -> build_model
│   └── theme.py                    colours, chart style, version shims
│
├── tests/                          314 tests
│   ├── conftest.py                 the shared six-month ATM call
│   ├── test_core.py                validation, units, the registry
│   ├── test_black_scholes.py       against an independent reference
│   ├── test_trees.py               convergence, theta, exercise boundary
│   ├── test_black76.py             both Greek conventions
│   ├── test_monte_carlo.py         estimator, error bars, variance reduction
│   ├── test_barrier.py             PDE, parity, grid sizing
│   ├── test_basket.py              correlation, Cholesky, bad input
│   ├── test_calibration.py         implied vol round trips, chain cleaning
│   ├── test_strategies.py          leg additivity, P&L attribution
│   ├── test_scenario.py            shock grids, portfolio aggregation
│   ├── test_validation.py          the harness itself
│   └── test_cross_model.py         every model against every other model
│
├── .streamlit/config.toml          theme
├── pyproject.toml                  package metadata, pytest and coverage config
├── requirements.txt                runtime
├── requirements-dev.txt            runtime + pytest
├── Makefile                        make test / make run / make validate
└── Dockerfile                      docker run -p 8501:8501 pricing-desk
```

The split is deliberate: **`pricing_engine/` imports nothing from `dashboard/`**.
The library runs in a notebook, a script, a test, or a CI job with no Streamlit
installed and no network available. The dashboard is one consumer of it.

---

## Quick start

### Run the dashboard

```bash
pip install -r requirements.txt
streamlit run dashboard/app.py
```

Opens on <http://localhost:8501>. Market data is manual by default so it works
offline; tick **Fetch live from Yahoo** in the sidebar to pull a real chain.

### Use the library

```python
from datetime import date, timedelta
from pricing_engine import MarketInputs, OptionContract, BlackScholesMerton

market = MarketInputs(underlying_price=100.0, risk_free_rate=0.05,
                      dividend_yield=0.02, valuation_date=date(2024, 1, 1))
contract = OptionContract("AAPL", "call", strike=100.0,
                          expiry_date=date(2024, 7, 1), implied_volatility=0.20)

BlackScholesMerton(market, contract).all_greeks()
# {'price': 6.298209, 'delta': 0.566896, 'gamma': 0.027725,
#  'vega': 0.276272, 'theta': -0.121632, 'rho': 0.236789}
```

Or through the registry, which is what the dashboard uses:

```python
from pricing_engine import build_engine

engine = build_engine()
engine.available_models()
# ['asian_mc', 'barrier_mc', 'barrier_pde', 'black_76',
#  'black_scholes_merton', 'crr_binomial', 'leisen_reimer', 'lookback_mc']

engine.create_model("leisen_reimer", market, contract, N=501, american=True).price()
engine.create_model("barrier_pde", market, contract,
                    barrier=90.0, barrier_type="down-and-out").price()
```

### Docker

```bash
docker build -t pricing-desk .
docker run -p 8501:8501 pricing-desk
```

---

## Greek conventions

Every model reports the same six numbers in the same units. This matters more
than it sounds: the whole point of a comparison table is that the rows answer
the same question.

| Greek | Units | Sign for a long option |
|---|---|---|
| `delta` | dV/dS, dimensionless | + call, − put |
| `gamma` | d²V/dS² | + |
| `vega`  | per **one volatility point** (not per 1.00 of sigma) | + |
| `theta` | per **one calendar day**, already dV/dt | − |
| `rho`   | per **one percentage point** of rate | + call, − put |

Theta is the one people get wrong. It is already a derivative with respect to
time and it is already negative for a long option, so a P&L attribution adds
`+θ × days`, never `−θ`.

---

## Models

| Model | Products | Method | Notes |
|---|---|---|---|
| `BlackScholesMerton` | European vanilla | closed form | the reference every other model is checked against |
| `Black76Model` | European on a future | closed form | two Greek conventions — see below |
| `CRRBinomialTree` | European + American | lattice | O(1/N), oscillating |
| `LeisenReimerTree` | European + American | lattice | O(1/N²), inside a cent by N ≈ 21 |
| `LongstaffSchwartzMC` | American (or European) vanilla | least-squares simulation | rule fitted on one path set, priced on another, so low-biased; European control variate |
| `AsianOptionMonteCarlo` | arithmetic average | simulation | geometric control variate, ~34× SE reduction |
| `LookbackOptionMonteCarlo` | floating strike | simulation | Broadie-Glasserman-Kou continuity correction |
| `BasketOptionMonteCarlo` | weighted basket | simulation | Cholesky, PSD-checked correlation |
| `BarrierOptionPDE` | four barrier types | Crank-Nicolson | auto-sized grid, exact in–out parity |
| `BarrierOptionMonteCarlo` | four barrier types | simulation | Brownian-bridge survival probability |

### Black-76 has two Greek conventions, and you pick one

```python
Black76Model(market, contract)                          # spot Greeks
Black76Model(market, contract, futures_price=103.0)     # futures Greeks
```

With no `futures_price`, the forward is derived from spot, which makes the model
*algebraically identical* to Black-Scholes — so every Greek is a spot Greek and
the comparison row is meaningful. Supply one and it becomes a real option on a
futures contract: delta is dV/dF, and rho is −T·V, because a future has no carry
to offset the discounting.

---

## The eleven corrections

Every one of these was found by execution, not inspection, and every one has a
test that fails without the fix. They are folded into the code as real
implementations rather than a patch appendix.

| # | Where | What was wrong | Effect |
|---|---|---|---|
| **B-01** | `core.py` | `float(iv) if iv else 0.20` — **zero is falsy**, so a genuine 0% vol silently became 20% | wrong price, no error |
| **M-01** | `black76.py` | theta returned `t1 − rV` where the futures convention wants `t1 + rV` | landed between the two right answers |
| **M-02** | `black76.py` | delta and gamma in futures space, vega and rho in spot space | one table row, two different questions |
| **M-03** | `trees.py` | exercise boundary root-found `american(S) − intrinsic(S)`, which is ≥ 0 everywhere and never crosses zero | Brent always raised; every maturity returned the bracket endpoint 0.1K — a flat line |
| **M-04** | `multi_leg.py` | P&L attribution used `−θ` when theta is already dV/dt | decay double-counted with the wrong sign, every day |
| **M-04b** | `multi_leg.py` | `daily_pnl` never reset | a second `run()` returned twice the rows |
| **M-05** | `monte_carlo.py` | standard error computed over all 2n antithetic paths as if independent | reported 0.0669 where the truth is 0.0498 — overstated precision, from the technique meant to improve it |
| **M-07** | `barrier.py` | fixed 280-node grid regardless of barrier distance | at H = 10 S only a handful of nodes between spot and barrier |
| **M-09** | `yahoo.py` | vendor dividend yield taken at face value | a unit error in the feed became a pricing error |
| **M-10** | `trees.py` | theta read off node (2,1), valid only when `u·d = 1` | Leisen-Reimer bias **does not shrink with N**: −0.14351/day vs a true −0.12163 at N = 101, 201, 501 and 1001 alike |
| **M-11** | `monte_carlo.py` | lookback simulated a continuous extremum on a discrete grid | one-sided ~2.7% bias that does not average out and does not show in the standard error |

M-11 was not in the original review. It surfaced while writing the
explanatory document, which is its own argument for writing things down.

### M-10 in one paragraph

Both lattices read theta as `(V[2,1] − V[0,0]) / (2Δt)`. That is only theta if
node (2,1) sits at the starting spot — i.e. if `u·d = 1`. CRR satisfies that by
construction. Leisen-Reimer does not, and the error is **O(1)**, not O(1/N): the
node offset and the `2Δt` denominator both scale as 1/N, so they cancel and the
bias survives any number of steps. `_theta_from_lattice` corrects the node value
back to spot with the model's own delta and gamma first; the correction vanishes
identically when `u·d = 1`, so CRR is untouched.

---

## Validation harness

Phase 13 of the brief asks for model validation. The draft's harness covered
three models, was never called anywhere, and its own comment admitted the gap.
That was not academic — **every defect above lived in a model the harness did
not cover.**

```bash
python -m pricing_engine.validation
python -m pricing_engine.validation --spot 250 --strike 260 --vol 0.35 --days 90
python -m pricing_engine.validation --fast --csv report.csv
```

Exits non-zero on any failure, so it drops into CI unchanged. 35 checks, 11
model families, three kinds of argument:

- **closed form** — compare against an independently derived formula
- **cross model** — compare two methods that share no code past `core`
- **invariant** — a relationship that must hold under no-arbitrage alone

A few checks carry a domain rather than an assertion, and say so in the `note`
column instead of going red — variance reduction is not assertable on a contract
so far out of the money that one path in a hundred thousand pays, and a barrier
eight standard deviations away at 90% vol is 1,339× spot, beyond what a uniform
4,000-node grid can represent. Reporting the contract as a model failure would
be worse than useless.

---

## Tests

```bash
make test       # all 314, ~3 min
make fast       # skip the slow simulation modules, ~15s
make cov        # with a coverage report
```

`test_cross_model.py` is the file the draft did not have, and it is where the
value is: M-01, M-02, M-05, M-10 and M-11 were each a disagreement between two
models that nobody ever put side by side.

Reference values every test asserts against, at S = K = 100, r = 5%, q = 2%,
σ = 20%, T = 182/365:

| Quantity | Value |
|---|---|
| European call (Black-Scholes) | 6.298209 |
| European put | 4.828184 |
| Geometric Asian call, 50 fixings | 3.529347 |
| Floating-strike lookback call | 11.348428 |
| Down-and-out call, H = 90 | 5.851066 |
| Perpetual American put boundary | 64.92 |

---

## Dashboard

Eight tabs, covering all eleven features in the brief plus three the brief
implies:

| Tab | What it shows |
|---|---|
| **Payoff & value** | payoff today vs at expiry, and the delta–gamma approximation against full revaluation — the gap *is* the unexplained risk |
| **Scenarios** | the brief's grids (spot ±1/5/10%, vol ±5/10 pts, rates ±50/100bp, time +1/7/30d), every row a full reprice |
| **Strategies** | eight multi-leg strategies with additive Greeks and overlaid payoffs |
| **Position book** | an editable book, aggregated Greeks, shocked P&L — a pricer becomes a risk system |
| **Volatility smile** | live implied vols with a vega bar chart underneath, or a synthetic round trip offline |
| **Convergence** | MC error bars, CRR vs LR, and the theta-stability chart that shows M-10 |
| **P&L attribution** | daily decomposition into delta, gamma, theta and vega, with the residual |
| **Validation** | the full 35-check harness, rerun against the contract in the sidebar |

The sidebar is the only place that reads widgets; `state.build_model` is the only
place that constructs a pricer. That is what keeps the headline metric, the
payoff curve and the scenario grid the *same* model rather than three that
happen to agree.

---

## Requirements

The engine needs numpy, scipy and pandas. The dashboard adds streamlit and
plotly. `yfinance` is optional at runtime — every model prices with no network
at all, and the live-data toggle is off by default.

Python 3.10 or newer.
