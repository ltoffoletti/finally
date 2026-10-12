# Market Simulator

How FinAlly makes up realistic-looking stock prices when no `MASSIVE_API_KEY` is set. The simulator is the default data source. It needs no network or API key, and it implements the same `MarketDataSource` interface as the Massive client (see `MARKET_INTERFACE.md`), so nothing downstream can tell the difference.

Code: `backend/app/market/simulator.py` (engine and source) and `backend/app/market/seed_prices.py` (data). The code exists and is tested. §8 lists the small changes that remain.

> Supersedes `planning/archive/MARKET_SIMULATOR.md`.

---

## 1. Goals

| Goal | How it is met |
|---|---|
| Prices *look* like stocks: random, never negative, percentage moves, not dollar moves | Geometric Brownian Motion (GBM) |
| Something visible every half second (flashes, sparklines) | 500 ms ticks; per-ticker volatility sized so moves are a few cents |
| Stocks move together the way sectors do | Correlated random draws (Cholesky factor of a sector correlation matrix) |
| Occasional drama for the demo | Random 2–5% "event" jumps |
| Realistic starting levels | Seed prices for the 10 default tickers; $50–$300 for anything else |
| Any symbol the user types works | Unknown symbols get default parameters (PLAN §6) |
| Cheap, in-process, testable | NumPy plus one asyncio task; the math engine is a plain class with no I/O |

Non-goals: order books, bid/ask spreads, volume, market hours, historical backfill, or reproducing real prices.

---

## 2. The model

### 2.1 GBM step

For each ticker *i*, every tick:

```
S_i(t+Δt) = S_i(t) · exp( (μ_i − σ_i²/2)·Δt  +  σ_i·√Δt · Z_i )
```

| Symbol | Meaning | Value |
|---|---|---|
| `S` | price | starts at the seed price |
| `μ` | annualized drift | 0.03–0.08 (default 0.05) |
| `σ` | annualized volatility | 0.17–0.50 (default 0.25) |
| `Δt` | tick length as a fraction of a **trading** year | `0.5 / (252 × 6.5 × 3600)` ≈ **8.48 × 10⁻⁸** |
| `Z_i` | standard normal, **correlated** across tickers (§2.2) | |

Why GBM: the exponential keeps prices positive, moves scale with price (a $800 stock moves more dollars than a $50 one), and the `−σ²/2` term (Itô correction) makes `μ` the true expected return. Treating each 500 ms of wall-clock time as 500 ms of trading time gives real-world volatility at real-world speed.

**What that produces** (one standard deviation, from the parameters below):

| Ticker | σ | per tick (500 ms) | per 2.5 min (one sparkline window, 300 ticks) | per hour (GBM only) |
|---|---|---|---|---|
| V | 0.17 | 0.0050% ≈ $0.014 | 0.09% | 0.42% |
| AAPL | 0.22 | 0.0064% ≈ $0.012 | 0.11% | 0.54% |
| NVDA | 0.40 | 0.0116% ≈ $0.093 | 0.20% | 0.99% |
| TSLA | 0.50 | 0.0146% ≈ $0.036 | 0.25% | 1.24% |

Moves are a few cents per tick, so prices are rounded to cents for display and in the cache. Some ticks therefore round to "no change" (`direction: "flat"`, no flash); for AAPL that is roughly one tick in three. This reads as natural. Sparklines auto-scale their y-axis, so a 0.1% window still fills the chart.

### 2.2 Correlated moves (Cholesky)

Independent normals `z ~ N(0, I)` become correlated normals `Z = L·z`, where `L` is the lower Cholesky factor of the correlation matrix `C` (`C = L·Lᵀ`). Then `Cov(Z) = L·I·Lᵀ = C`.

`C` comes from simple sector rules (`_pairwise_correlation`):

| Pair | ρ |
|---|---|
| tech–tech (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX) | 0.6 |
| finance–finance (JPM, V) | 0.5 |
| anything with TSLA ("does its own thing") | 0.3 |
| cross-sector, or any unknown ticker | 0.3 |
| diagonal | 1.0 |

Every pair has at least 0.3 correlation, so the whole market drifts together a little, with tighter clusters inside sectors. A matrix built from these block rules is positive definite for any mix of tickers, so the Cholesky factorization always succeeds. (Checked: default 10 plus 3 unknown tickers gives a smallest diagonal of `L` of 0.68.) `L` is rebuilt, in O(n³) time but with n < 50, only when a ticker is added or removed, never per tick. With fewer than two tickers there is nothing to correlate and `L` is `None`.

### 2.3 Random events

After the GBM step, each ticker independently has probability `p = 0.001` per tick of a shock:

```
S ← S · (1 ± m),   m ~ Uniform(0.02, 0.05),   sign 50/50
```

With 10 tickers at 2 ticks/s that is about **one event every 50 seconds** somewhere on the board, and about 7 per ticker per hour.

> **Long-run effect: shocks dominate.** Each shock is about ±3.5% and shocks are permanent, so over an hour they add about **±9.5%** of random wander per ticker, against ±0.5–1.2% from GBM. They also add a small downward bias of about −0.47%/hour, because `E[log(1±m)] < 0`. For a demo session of tens of minutes this is what we want: visible action and P&L that actually changes. Over a container that runs for days, prices can drift a long way from their seeds. See §7 if that becomes a problem.

### 2.4 Parameters and seed data (`seed_prices.py`)

| Ticker | Seed $ | σ | μ | Group |
|---|---|---|---|---|
| AAPL | 190 | 0.22 | 0.05 | tech |
| GOOGL | 175 | 0.25 | 0.05 | tech |
| MSFT | 420 | 0.20 | 0.05 | tech |
| AMZN | 185 | 0.28 | 0.05 | tech |
| TSLA | 250 | 0.50 | 0.03 | (own) |
| NVDA | 800 | 0.40 | 0.08 | tech |
| META | 500 | 0.30 | 0.05 | tech |
| JPM | 195 | 0.18 | 0.04 | finance |
| V | 280 | 0.17 | 0.04 | finance |
| NFLX | 600 | 0.35 | 0.05 | tech |
| *any other* | Uniform(50, 300) | 0.25 | 0.05 | (none, ρ = 0.3) |

All of this is data. Adding a known ticker means adding a row to these dicts, with no code change.

---

## 3. Code structure

There are two classes, which keeps the math separate from the async plumbing:

```
seed_prices.py          constants only: SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS,
                        CORRELATION_GROUPS, INTRA_TECH_CORR, INTRA_FINANCE_CORR,
                        CROSS_GROUP_CORR, TSLA_CORR

simulator.py
├── GBMSimulator        pure, synchronous, no I/O, no asyncio
│   ├── step() -> dict[str, float]           advance every ticker one Δt
│   ├── add_ticker / remove_ticker           rebuild Cholesky
│   ├── get_price / get_tickers
│   └── _rebuild_cholesky / _pairwise_correlation
│
└── SimulatorDataSource(MarketDataSource)    asyncio adapter
    ├── start(tickers)   build GBMSimulator, seed cache, launch loop task
    ├── _run_loop()      step → cache.update(...) → sleep(0.5)
    ├── add_ticker       sim.add_ticker + seed cache immediately
    ├── remove_ticker    sim.remove_ticker + cache.remove
    └── stop()           cancel task (idempotent)
```

### 3.1 `GBMSimulator`: the engine

```python
class GBMSimulator:
    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600        # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR        # ≈ 8.48e-8

    def __init__(self, tickers, dt=DEFAULT_DT, event_probability=0.001, rng=None):
        self._dt = dt
        self._event_prob = event_probability
        self._rng = rng or np.random.default_rng()     # §8 item 2: injectable for tests
        self._tickers: list[str] = []                  # order == row order of the Cholesky factor
        self._prices: dict[str, float] = {}            # full precision; rounded only on output
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None
        for t in tickers:
            self._add_ticker_internal(t)               # batch add, then one rebuild
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        n = len(self._tickers)
        if n == 0:
            return {}
        z = self._rng.standard_normal(n)
        if self._cholesky is not None:
            z = self._cholesky @ z                     # correlate
        out = {}
        for i, t in enumerate(self._tickers):
            mu, sigma = self._params[t]["mu"], self._params[t]["sigma"]
            self._prices[t] *= math.exp((mu - 0.5 * sigma**2) * self._dt
                                        + sigma * math.sqrt(self._dt) * z[i])
            if self._rng.random() < self._event_prob:  # drama
                m = self._rng.uniform(0.02, 0.05)
                self._prices[t] *= 1 + m if self._rng.random() < 0.5 else 1 - m
            out[t] = round(self._prices[t], 2)
        return out
```

Design points:

- **Full-precision internal state.** Only the returned values are rounded. Rounding the state itself would make sub-cent moves vanish and bias low-priced stocks.
- **The list order is the matrix order.** `_tickers` (a list, not a set) fixes which row of `L` belongs to which ticker. Add and remove keep the list and the matrix in sync by rebuilding.
- **No I/O and no clock.** `step()` advances exactly one Δt whenever it is called. That makes tests deterministic and fast (call it 10,000 times and check the statistics) and keeps timing in the adapter.

### 3.2 `SimulatorDataSource`: the adapter

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache, update_interval=0.5, event_probability=0.001):
        ...

    async def start(self, tickers):
        self._sim = GBMSimulator(tickers, event_probability=self._event_prob)
        for t in tickers:                                     # seed so SSE / startup snapshot
            self._cache.update(t, self._sim.get_price(t))     # have prices right away
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def add_ticker(self, ticker):
        self._sim.add_ticker(ticker)                          # no-op if present
        self._cache.update(ticker, self._sim.get_price(ticker))   # priced before returning

    async def remove_ticker(self, ticker):
        self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    async def _run_loop(self):
        while True:
            try:
                for t, p in self._sim.step().items():
                    self._cache.update(t, p)
            except Exception:
                logger.exception("Simulator step failed")     # never let the loop die
            await asyncio.sleep(self._interval)
```

- **Concurrency.** Everything runs on the event loop: `step()`, `add_ticker` and `remove_ticker` never interleave, so the simulator needs no lock. (`PriceCache` has its own lock for any thread readers.)
- **Cost.** One step for 10 tickers is a 10×10 mat-vec plus a 10-iteration Python loop, a few microseconds. It does not need a thread.
- **Cadence.** `sleep(0.5)` after the step gives a period of 0.5 s plus step time. The drift is negligible, and nothing depends on exact timing because each `PriceUpdate` carries its own `time.time()` timestamp.
- **Validation.** The simulator accepts any symbol that `normalize_ticker` accepted (PLAN §6), so its `add_ticker` never raises `UnknownTickerError`.

---

## 4. How it fits the rest of the system

| Concern | Behaviour |
|---|---|
| Session-start price (PLAN §6) | The cache records the first price it sees for each ticker. In simulator mode that is the seed price, so `session_change_percent` = move since seed. |
| SSE | The loop writes every tracked ticker every tick, so `cache.version` bumps every 500 ms and the stream sends one event per tick. |
| Trades | Fill at `cache.get_price(t)`, which is the latest rounded simulator price. |
| New ticker via watchlist or a trade | `add_ticker` puts it into the next `step()` and prices it immediately, so a trade on a never-seen symbol fills at once. |
| Removed ticker | Dropped from the simulator and the cache. If it is added back, it **restarts at its seed price** (a fresh random one if it is unknown) with a fresh session start. |
| Restart | All simulator state is in memory, so every ticker restarts at its seed. Held positions keep their `avg_cost`, so unrealized P&L jumps to "seed vs. avg_cost" after a restart. Accepted for a simulator. |

---

## 5. Configuration and tunables

| Knob | Where | Default | Effect |
|---|---|---|---|
| `update_interval` | `SimulatorDataSource(...)` | 0.5 s | Tick rate. Change `dt` with it to keep volatility realistic |
| `dt` | `GBMSimulator(...)` | 8.48e-8 | Bigger values speed up time (e.g. `×60` makes one tick a simulated half-minute) |
| `event_probability` | both constructors | 0.001 | Shocks per ticker per tick. 0 turns drama off (useful in tests) |
| shock size | `step()` | U(2%, 5%) | |
| σ, μ per ticker | `TICKER_PARAMS` / `DEFAULT_PARAMS` | table §2.4 | |
| correlations | `seed_prices.py` constants | 0.6 / 0.5 / 0.3 | |

These are constructor arguments, not environment variables. PLAN §5 defines no simulator env vars, and the demo needs none.

---

## 6. Testing (`backend/tests/market/test_simulator*.py`)

Already covered: prices stay positive; `step()` returns every ticker; add/remove rebuild the matrix; unknown tickers get defaults; correlations follow the sector rules; the Cholesky factor reproduces `C`; the source seeds the cache, the loop updates it, and stop is idempotent.

Statistical checks that are worth keeping, run with `event_probability=0` and a seeded `rng` (§8 item 2):

- **Log-return moments.** Over N steps, `mean(log(S_{t+1}/S_t)) ≈ (μ − σ²/2)·Δt` and `std ≈ σ·√Δt`, within a tolerance of a few standard errors.
- **Correlation.** The empirical correlation of AAPL/MSFT log returns over ~20k steps is ≈ 0.6; AAPL/JPM ≈ 0.3.
- **Event rate.** With `event_probability=0.1` and σ ≈ 0, the share of steps with |return| ≥ 2% is ≈ 0.1.
- **Determinism.** Two simulators built with `np.random.default_rng(42)` produce identical paths.

---

## 7. Known limitations and optional refinements

None of these is required for the MVP. They are listed so the trade-offs are explicit.

1. **Shock wander over long sessions (§2.3).** To keep the drama but stop the drift, make shocks *transient*: keep a per-ticker `jump` factor that decays back towards 1 (e.g. multiply by `exp(-Δ/τ)` each tick, with τ ≈ 60 s) and publish `S_gbm · jump`. The alternative is lowering `event_probability` to 0.0003.
2. **Restart jumps (§4).** `start()` could accept `initial_prices: dict[str, float]` (for example the last trade price per held ticker, from the `trades` table) so held positions resume near where they were.
3. **Unknown-ticker seeds differ on every restart.** `random.uniform(50, 300)` gives a held `PYPL` a new price each boot. A per-symbol deterministic seed, `random.Random(hashlib.sha256(t.encode()).digest()).uniform(50, 300)`, makes it stable (§8 item 3).
4. **No market hours or overnight gaps.** Prices tick around the clock. That suits a demo.
5. **Fixed correlation structure.** No regime changes, no volatility clustering. A GARCH-style σ would add realism at the cost of complexity.

---

## 8. Remaining changes to the existing code

| # | Change | Why |
|---|---|---|
| 1 | `SimulatorDataSource.add_ticker` must work (or raise clearly) before `start()`; today it silently does nothing when `_sim is None` | Avoids a silent no-op if the call order changes |
| 2 | Inject `rng: np.random.Generator` into `GBMSimulator` and use it for **all** randomness (normal draws, event draws, unknown-ticker seeds) instead of the global `np.random` / `random` modules | Deterministic tests; no hidden global state |
| 3 | Deterministic seed price for unknown tickers (§7 item 3) | Stable valuations across restarts; one line |
| 4 | Nothing else in the simulator changes for PLAN §13: the session-start price lives in `PriceCache` (see `MARKET_INTERFACE.md` §5), so it works the same for both sources | |
