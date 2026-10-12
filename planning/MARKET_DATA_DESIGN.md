# Market Data Backend — Detailed Design

**Status:** Implementation-ready design (2026-10-12). Supersedes `planning/archive/MARKET_DATA_DESIGN.md`.
**Scope:** Everything under `backend/app/market/`, plus the integration points other backend code uses (lifespan, watchlist, trades, SSE).
**Contract:** `planning/PLAN.md` §3, §5, §6 and §8. If this document and PLAN.md disagree, PLAN.md wins.

This document gives complete code for every market-data module: the unified API, the GBM simulator and the Massive (Polygon.io) client. It starts from the code already in `backend/app/market/` (73 passing tests) and specifies the changes that PLAN.md still requires, plus fixes for defects found while writing this design.

---

## Table of Contents

1. [What Changes and Why](#1-what-changes-and-why)
2. [Architecture](#2-architecture)
3. [Ticker Normalization — `tickers.py`](#3-ticker-normalization--tickerspy)
4. [Data Model — `models.py`](#4-data-model--modelspy)
5. [Price Cache — `cache.py`](#5-price-cache--cachepy)
6. [Unified Interface — `interface.py`](#6-unified-interface--interfacepy)
7. [Seed Data — `seed_prices.py`](#7-seed-data--seed_pricespy)
8. [GBM Simulator — `simulator.py`](#8-gbm-simulator--simulatorpy)
9. [Massive Client — `massive_client.py`](#9-massive-client--massive_clientpy)
10. [Factory — `factory.py`](#10-factory--factorypy)
11. [SSE Stream — `stream.py`](#11-sse-stream--streampy)
12. [Package Exports — `__init__.py`](#12-package-exports--__init__py)
13. [Integrating with the Rest of the Backend](#13-integrating-with-the-rest-of-the-backend)
14. [Worked Examples](#14-worked-examples)
15. [Testing](#15-testing)
16. [Edge Cases and Failure Modes](#16-edge-cases-and-failure-modes)
17. [Configuration Reference](#17-configuration-reference)
18. [Implementation Checklist](#18-implementation-checklist)

---

## 1. What Changes and Why

| # | Change | Source | Module |
|---|--------|--------|--------|
| 1 | Cache records a **session-start price** per ticker; `PriceUpdate` gains `session_start_price` and `session_change_percent`, both included in the SSE payload | PLAN §6, §13 | `models.py`, `cache.py` |
| 2 | Factory reads **`MASSIVE_POLL_INTERVAL`** (default 15s) | PLAN §5, §13 | `factory.py` |
| 3 | **Ticker normalization**: uppercase, must match `^[A-Z.]{1,10}$` | PLAN §6 | new `tickers.py` |
| 4 | **Ticker validation**: `add_ticker()` raises `InvalidTickerError` when the source can't price a ticker. The simulator accepts any well-formed symbol; Massive rejects symbols that return no snapshot | PLAN §6 | `interface.py`, `massive_client.py` |
| 5 | `add_ticker()` **seeds the cache** before it returns, in both sources, so a trade or watchlist add right after it has a price | PLAN §8 (`NO_PRICE` is rare) | both sources |
| 6 | **Bug fix:** Massive parsing read `snap.last_trade.timestamp`, but the `massive` package's `LastTrade` model has no such field. The field is `sip_timestamp`, in **nanoseconds**. In real Massive mode every snapshot raised `AttributeError` and was skipped, so no prices ever reached the cache. The tests missed it because they use `MagicMock`. | found in review | `massive_client.py` |
| 7 | **Bug fix:** a poll in flight while a ticker is removed wrote the removed ticker back into the cache | found in review | `massive_client.py` |
| 8 | `PriceCache.remove()` bumps `version`, so SSE clients see the removal on the next tick. `version` is read under the lock | found in review | `cache.py` |
| 9 | `create_stream_router()` builds a new `APIRouter` on every call (it used to reuse a module-level router). Cancellation is no longer swallowed. A heartbeat comment keeps idle connections alive | archive review §3.6 | `stream.py` |
| 10 | The simulator uses an injectable `numpy.random.Generator` (`seed=`), so tests can be deterministic | testability | `simulator.py` |
| 11 | The Massive client can be injected (`client=`), so tests use a fake client instead of patching module globals | testability | `massive_client.py` |

The public API stays backward compatible. `PriceCache.update/get/get_price/get_all/remove/version`, `MarketDataSource.start/stop/add_ticker/remove_ticker/get_tickers`, `create_market_data_source(cache)` and `create_stream_router(cache)` keep their signatures. New parameters are optional keyword arguments.

---

## 2. Architecture

```
                         ┌──────────────────────────────┐
  MASSIVE_API_KEY ──────▶│ create_market_data_source()  │  factory.py
  MASSIVE_POLL_INTERVAL  └──────────────┬───────────────┘
                                        │ returns one of
                 ┌──────────────────────┴───────────────────────┐
                 ▼                                              ▼
   ┌───────────────────────────┐                ┌───────────────────────────┐
   │ SimulatorDataSource       │                │ MassiveDataSource         │
   │  └─ GBMSimulator (numpy)  │                │  └─ massive.RESTClient    │
   │  tick every 0.5s          │                │  poll every 15s (thread)  │
   └─────────────┬─────────────┘                └─────────────┬─────────────┘
                 │ cache.update(ticker, price, ts)            │
                 └───────────────────┬────────────────────────┘
                                     ▼
                     ┌───────────────────────────────┐
                     │ PriceCache (threading.Lock)   │  cache.py
                     │  latest PriceUpdate / ticker  │
                     │  session-start price / ticker │
                     │  version counter              │
                     └──┬───────────┬────────────┬───┘
                        │           │            │
          GET /api/stream/prices    │            │
          (stream.py, every 0.5s)   │            │
                        ▼           ▼            ▼
                   Browser     Portfolio     Trade execution,
                 EventSource   valuation     watchlist GET, LLM context
```

**Rules**

- **Producers push, consumers pull.** A data source never returns prices to callers. It writes to `PriceCache`, and everything downstream reads the cache. The only path from caller to source is the tracked-ticker set (`start`, `add_ticker`, `remove_ticker`).
- **One producer at a time.** The factory picks exactly one source per process.
- **One process.** The cache lives in memory, so uvicorn runs with one worker (PLAN §3).
- **The source does not know about the database.** Deciding *which* tickers to track (watchlist ∪ open positions) is the caller's job (§13.3). The source only tracks what it's told.

### File layout

```
backend/app/market/
├── __init__.py         # public exports
├── tickers.py          # NEW: normalize_ticker(), InvalidTickerError
├── models.py           # PriceUpdate (+ session fields)
├── cache.py            # PriceCache (+ session-start price)
├── interface.py        # MarketDataSource ABC
├── seed_prices.py      # unchanged
├── simulator.py        # GBMSimulator + SimulatorDataSource
├── massive_client.py   # parse_snapshot() + MassiveDataSource
├── factory.py          # create_market_data_source()
└── stream.py           # create_stream_router(), SSE generator
```

Every full-file listing below starts with a `# backend/app/market/<file>.py` comment and replaces that file entirely.

---

## 3. Ticker Normalization — `tickers.py`

Everything that takes a ticker from outside the process (REST body, URL path, LLM output) passes it through `normalize_ticker()` first. The sources call it too, so a malformed ticker can never reach the cache.

```python
# backend/app/market/tickers.py
"""Ticker symbol normalization and validation errors."""

from __future__ import annotations

import re

TICKER_PATTERN = re.compile(r"^[A-Z.]{1,10}$")


class InvalidTickerError(ValueError):
    """The ticker is malformed, or the market data source cannot price it.

    API routes map this to HTTP 400 {"code": "INVALID_TICKER"}.
    """

    def __init__(self, ticker: str, reason: str = "Invalid ticker") -> None:
        super().__init__(f"{reason}: {ticker!r}")
        self.ticker = ticker
        self.reason = reason


def normalize_ticker(raw: str) -> str:
    """Trim and uppercase a symbol, then check it against ^[A-Z.]{1,10}$.

    >>> normalize_ticker(" brk.b ")
    'BRK.B'
    >>> normalize_ticker("AAPL1")
    Traceback (most recent call last):
    ...
    app.market.tickers.InvalidTickerError: Invalid ticker: 'AAPL1'
    """
    if not isinstance(raw, str):
        raise InvalidTickerError(str(raw))
    ticker = raw.strip().upper()
    if not TICKER_PATTERN.fullmatch(ticker):
        raise InvalidTickerError(ticker)
    return ticker
```

| Input | Result |
|-------|--------|
| `"aapl"` | `"AAPL"` |
| `"  brk.b "` | `"BRK.B"` |
| `""`, `"AAPL1"`, `"BRK-B"`, `"ABCDEFGHIJK"` (11 chars), `"$TSLA"` | `InvalidTickerError` |

---

## 4. Data Model — `models.py`

`PriceUpdate` is the only type that leaves the market data layer. The new `session_start_price` field has a default, so existing constructor calls (`PriceUpdate(ticker, price, previous_price, timestamp)`) still work.

```python
# backend/app/market/models.py
"""Data models for market data."""

from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Literal

Direction = Literal["up", "down", "flat"]


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time.

    Tick-to-tick fields (change, change_percent, direction) compare against
    previous_price. Session fields compare against session_start_price: the
    first price the cache recorded for this ticker since the process started
    (simulator: the seed price; Massive: the first polled price).
    """

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds
    session_start_price: float | None = None  # None only for hand-built instances

    @property
    def change(self) -> float:
        """Absolute price change from the previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percent change from the previous update (0.0578 means 0.0578%)."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> Direction:
        if self.price > self.previous_price:
            return "up"
        if self.price < self.previous_price:
            return "down"
        return "flat"

    @property
    def session_change_percent(self) -> float:
        """Percent change since session start; shown as "daily change %" in the UI."""
        start = self.session_start_price
        if not start:
            return 0.0
        return round((self.price - start) / start * 100, 4)

    def to_dict(self) -> dict:
        """Serialize for the SSE payload (PLAN §6). Keys and units are the contract."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
            "session_change_percent": self.session_change_percent,
        }
```

**Design notes**

- Derived fields are properties, so they can never disagree with `price` and `previous_price`.
- `session_start_price` is stored but not serialized. The PLAN payload exposes only the percentage, and the frontend has no use for the raw value.
- Percentages are in "percent units", rounded to 4 decimals: `0.2737` means +0.2737 %. The REST `session_change_percent` in `GET /api/watchlist` uses the same value.

Example:

```python
>>> u = PriceUpdate("AAPL", price=190.52, previous_price=190.41,
...                 timestamp=1791369600.123, session_start_price=190.00)
>>> u.to_dict()
{'ticker': 'AAPL', 'price': 190.52, 'previous_price': 190.41,
 'timestamp': 1791369600.123, 'change': 0.11, 'change_percent': 0.0578,
 'direction': 'up', 'session_change_percent': 0.2737}
```

---

## 5. Price Cache — `cache.py`

The cache is the hub between producers and consumers. It is thread-safe because the Massive client's blocking HTTP call runs in a worker thread (`asyncio.to_thread`). The cache writes happen on the event loop today, but that is an implementation detail callers should not rely on.

```python
# backend/app/market/cache.py
"""Thread-safe in-memory price cache."""

from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Latest price per ticker, plus a session-start baseline and a version counter.

    Writers: exactly one MarketDataSource.
    Readers: SSE stream, portfolio valuation, trade execution, watchlist API, LLM context.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._session_start: dict[str, float] = {}
        self._lock = Lock()
        self._version = 0  # bumped on every mutation (update or remove)

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price and return the resulting PriceUpdate.

        - Prices are rounded to cents.
        - First update for a ticker: previous_price == price (direction "flat"),
          and this price becomes the session-start price.
        """
        with self._lock:
            ts = time.time() if timestamp is None else float(timestamp)
            price = round(float(price), 2)
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price
            session_start = self._session_start.setdefault(ticker, price)

            update = PriceUpdate(
                ticker=ticker,
                price=price,
                previous_price=previous_price,
                timestamp=ts,
                session_start_price=session_start,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Shallow copy of all current prices (PriceUpdate is immutable)."""
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        """Forget a ticker entirely, including its session-start baseline."""
        with self._lock:
            removed = self._prices.pop(ticker, None)
            self._session_start.pop(ticker, None)
            if removed is not None:
                self._version += 1  # SSE clients must see the ticker disappear

    @property
    def version(self) -> int:
        with self._lock:
            return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: object) -> bool:
        with self._lock:
            return ticker in self._prices
```

### Session-start semantics

| Event | Session-start price |
|-------|---------------------|
| Simulator `start()` seeds the cache | the seed price (e.g. AAPL 190.00) |
| Massive first successful poll | the first polled price |
| `add_ticker()` for a new ticker | the first price written by the source |
| `remove()` then a later re-add | reset; the baseline is the first price after the re-add |

The last row clarifies PLAN §6 for a case it doesn't cover. A ticker is only removed when it is neither watched nor held. The simulator also restarts a re-added ticker from its seed (or a new random) price, so keeping the old baseline would make a meaningless percentage.

### Why a version counter

SSE polls the cache every 500 ms. In Massive mode prices change only every 15 s, so without the counter the stream would resend identical payloads 30 times. The stream sends only when `version` has moved (§11).

---

## 6. Unified Interface — `interface.py`

The five methods are unchanged. What changes is the documented contract, mainly for `add_ticker`, which now validates and seeds.

```python
# backend/app/market/interface.py
"""Abstract interface for market data sources."""

from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    A source pushes prices into a shared PriceCache on its own schedule.
    Callers never get prices from the source directly; they read the cache.
    Callers decide WHICH tickers are tracked (watchlist ∪ open positions);
    the source only tracks what it is told.

    All methods must be called from the event loop thread.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL"])
        await source.add_ticker("PYPL")      # may raise InvalidTickerError
        await source.remove_ticker("GOOGL")
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing prices for `tickers` and start the background task.

        Tickers are normalized; malformed ones are logged and skipped (startup
        must not fail because of one bad DB row). Call once.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Cancel the background task. Idempotent. No cache writes afterwards."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Start tracking `ticker`. No-op if it is already tracked.

        Raises:
            InvalidTickerError: the symbol is malformed, or the source cannot
                price it (Massive: no snapshot returned).

        On success the cache holds a price for the ticker when this returns,
        if the source has one. The simulator always does; Massive does whenever
        the snapshot carried a price.
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Stop tracking `ticker` and drop it from the cache. No-op if absent."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Currently tracked tickers, in insertion order."""
```

---

## 7. Seed Data — `seed_prices.py`

Unchanged. For reference:

| Ticker | Seed $ | σ (vol) | μ (drift) | Correlation group |
|--------|-------:|--------:|----------:|-------------------|
| AAPL | 190 | 0.22 | 0.05 | tech |
| GOOGL | 175 | 0.25 | 0.05 | tech |
| MSFT | 420 | 0.20 | 0.05 | tech |
| AMZN | 185 | 0.28 | 0.05 | tech |
| TSLA | 250 | 0.50 | 0.03 | none (0.3 with everything) |
| NVDA | 800 | 0.40 | 0.08 | tech |
| META | 500 | 0.30 | 0.05 | tech |
| JPM | 195 | 0.18 | 0.04 | finance |
| V | 280 | 0.17 | 0.04 | finance |
| NFLX | 600 | 0.35 | 0.05 | tech |
| *other* | U(50, 300) | 0.25 | 0.05 | none |

Correlations: tech–tech 0.6, finance–finance 0.5, everything else 0.3.

---

## 8. GBM Simulator — `simulator.py`

Two classes:

- **`GBMSimulator`**: pure, synchronous math engine. Holds current prices and advances them one step at a time. It has no asyncio and no cache, so it's trivial to unit-test.
- **`SimulatorDataSource`**: the `MarketDataSource` adapter. It runs `step()` every 0.5 s in an asyncio task and writes the results to the cache.

### 8.1 The math

Per tick, for each ticker *i*:

```
S_i(t+dt) = S_i(t) · exp( (μ_i − σ_i²/2)·dt + σ_i·√dt·Z_i )
Z = L · ε,   ε ~ N(0, I),   L = cholesky(C)
```

- `dt = 0.5 / (252 · 6.5 · 3600) ≈ 8.48e-8`: one 500 ms tick as a fraction of a trading year. So one wall-clock second of simulation equals one second of a trading session.
- `C` is the sector correlation matrix from §7. `L·ε` gives standard normals with correlation `C`.
- **Shock events:** with probability `p = 0.001` per ticker per tick, the price is multiplied by `1 ± U(0.02, 0.05)`. With 10 tickers at 2 ticks/s, that's about one event every 50 s somewhere on the board.

How big are the moves? For AAPL (σ = 0.22) the per-tick standard deviation is `0.22 · √8.48e-8 ≈ 6.4e-5`, about 1.2¢ on $190. So most ticks change the displayed price, and an hour of running gives a realistic intraday range (`0.22·√(3600/5896800) ≈ 0.54 %`, about 1σ).

### 8.2 Full listing

```python
# backend/app/market/simulator.py
"""GBM-based market simulator."""

from __future__ import annotations

import asyncio
import logging
import math

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    SEED_PRICES,
    TICKER_PARAMS,
    TSLA_CORR,
)
from .tickers import InvalidTickerError, normalize_ticker

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Correlated Geometric Brownian Motion over a dynamic set of tickers.

        S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)

    Z is drawn from N(0, C) via the Cholesky factor of the sector correlation
    matrix C. Occasional random shocks of 2-5% add visual drama.
    """

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8 (one 500ms tick)

    def __init__(
        self,
        tickers: list[str] | None = None,
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
        seed: int | None = None,
    ) -> None:
        self._dt = dt
        self._sqrt_dt = math.sqrt(dt)
        self._event_prob = event_probability
        self._rng = np.random.default_rng(seed)

        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers or []:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    # --- Public API ---

    def step(self) -> dict[str, float]:
        """Advance every ticker one tick. Returns {ticker: price rounded to cents}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z = self._rng.standard_normal(n)
        if self._cholesky is not None:
            z = self._cholesky @ z

        shocks = self._rng.random(n) < self._event_prob

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * self._sqrt_dt * z[i]
            price = self._prices[ticker] * math.exp(drift + diffusion)

            if shocks[i]:
                magnitude = self._rng.uniform(0.02, 0.05)
                sign = 1.0 if self._rng.random() < 0.5 else -1.0
                price *= 1.0 + sign * magnitude
                logger.debug("Shock on %s: %+.1f%%", ticker, sign * magnitude * 100)

            self._prices[ticker] = price
            result[ticker] = round(price, 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker (no-op if present) and rebuild the correlation factor."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker (no-op if absent) and rebuild the correlation factor."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        price = self._prices.get(ticker)
        return None if price is None else round(price, 2)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        seed = SEED_PRICES.get(ticker)
        self._prices[ticker] = seed if seed is not None else float(self._rng.uniform(50.0, 300.0))
        self._params[ticker] = dict(TICKER_PARAMS.get(ticker, DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """O(n^3) for n tickers, run only on add/remove; n stays small (< 100)."""
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho

        try:
            self._cholesky = np.linalg.cholesky(corr)
        except np.linalg.LinAlgError:
            # Cannot happen with the current constants, but never kill the feed over it.
            logger.warning("Correlation matrix not positive definite; using independent moves")
            self._cholesky = None

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR


class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by GBMSimulator, ticking every `update_interval` seconds."""

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
        seed: int | None = None,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        # Created eagerly so add_ticker() also works before start().
        self._sim = GBMSimulator(event_probability=event_probability, seed=seed)
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        for raw in tickers:
            try:
                self._track(normalize_ticker(raw))
            except InvalidTickerError:
                logger.warning("Simulator: skipping malformed ticker %r", raw)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(self._sim.get_tickers()))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)  # simulator accepts any well-formed symbol
        if ticker in self._sim.get_tickers():
            return
        self._track(ticker)
        logger.info("Simulator: added %s at %.2f", ticker, self._cache.get_price(ticker))

    async def remove_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers()

    # --- Internals ---

    def _track(self, ticker: str) -> None:
        """Add to the simulation and seed the cache so the price exists immediately."""
        self._sim.add_ticker(ticker)
        price = self._sim.get_price(ticker)
        if price is not None and ticker not in self._cache:
            self._cache.update(ticker, price)  # also sets the session-start price

    async def _run_loop(self) -> None:
        while True:
            try:
                for ticker, price in self._sim.step().items():
                    self._cache.update(ticker, price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

### 8.3 Behavior notes

- **Immediate prices.** `start()` and `add_ticker()` write the seed price to the cache synchronously. The first SSE event and any trade right after a watchlist add already have a price. In simulator mode `NO_PRICE` cannot happen.
- **No locking needed for `_sim`.** `step()` runs on the event loop between `await`s, and `add_ticker`/`remove_ticker` run on the same loop, so they can never interleave.
- **Drift over a session is negligible.** `μ·dt` is about 4e-9 per tick, so prices wander mostly through σ and shocks. That is what we want for a demo.
- **Prices can't go negative.** Both `exp(·)` and `1 − 0.05` are positive.

---

## 9. Massive Client — `massive_client.py`

### 9.1 What the API actually returns

One call fetches every tracked ticker:

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT
```

Raw JSON for one ticker (abridged). The `massive` package maps these keys onto dataclasses:

```json
{
  "ticker": "AAPL",
  "todaysChange": 1.63,
  "todaysChangePerc": 0.86,
  "updated": 1791369599123456789,
  "day":     {"o": 189.10, "h": 191.20, "l": 188.70, "c": 190.90, "v": 51234567, "vw": 190.1},
  "min":     {"c": 190.95, "t": 1791369540000, "v": 81234},
  "prevDay": {"c": 189.34, "o": 188.0, "h": 190.1, "l": 187.6, "v": 48000000},
  "lastTrade": {"p": 190.97, "s": 100, "t": 1791369599123456789, "x": 4},
  "lastQuote": {"p": 190.96, "P": 190.98, "s": 2, "S": 3, "t": 1791369599200000000}
}
```

| JSON | Python attribute (`massive` 2.x) | Unit / note |
|------|----------------------------------|-------------|
| `lastTrade.p` | `snap.last_trade.price` | dollars; **primary price** |
| `lastTrade.t` | `snap.last_trade.sip_timestamp` | **nanoseconds** (there is no `.timestamp`) |
| `min.c` | `snap.min.close` | latest minute bar close |
| `day.c` | `snap.day.close` | zero/absent between 00:00 ET and the first trade |
| `prevDay.c` | `snap.prev_day.close` | previous session close |
| `updated` | `snap.updated` | nanoseconds |
| `min.t`, `day.t` | `.timestamp` | milliseconds |

Every nested object may be `None` (the package only builds it when the key is present), and every numeric field may be `None` or `0`. Unknown tickers are **silently omitted** from the list; a request for one unknown ticker returns `[]`. Non-200 responses raise `massive.exceptions.BadResponse`; an empty key raises `AuthError`.

### 9.2 Parsing strategy

`parse_snapshot()` is a pure function, tested against the real `massive` models (§15.3). It reduces one snapshot to `(ticker, price, timestamp_seconds)`:

1. **Price:** the first positive value of `last_trade.price` → `min.close` → `day.close` → `prev_day.close`. The fallbacks matter pre-market and on weekends, when the trade may be missing but a close exists.
2. **Timestamp:** the first positive value of `last_trade.sip_timestamp` → `updated` → `time.time()`, normalized to seconds by magnitude. Massive mixes ns and ms, so the unit is never assumed.
3. **Reject** (return `None`) if there's no ticker or no positive price.

### 9.3 Full listing

```python
# backend/app/market/massive_client.py
"""Massive (formerly Polygon.io) REST poller for real market data."""

from __future__ import annotations

import asyncio
import logging
import time
from dataclasses import dataclass
from typing import Any

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource
from .tickers import InvalidTickerError, normalize_ticker

logger = logging.getLogger(__name__)

MAX_BACKOFF_SECONDS = 120.0


@dataclass(frozen=True, slots=True)
class SnapshotQuote:
    """The three fields FinAlly needs from a Massive TickerSnapshot."""

    ticker: str
    price: float
    timestamp: float  # Unix seconds


def to_unix_seconds(ts: float) -> float:
    """Normalize a Massive timestamp (s, ms, us or ns) to Unix seconds by magnitude."""
    ts = float(ts)
    if ts >= 1e17:  # nanoseconds   (~1.7e18 today)
        return ts / 1e9
    if ts >= 1e14:  # microseconds  (~1.7e15)
        return ts / 1e6
    if ts >= 1e11:  # milliseconds  (~1.7e12)
        return ts / 1e3
    return ts  # already seconds (~1.7e9)


def _nested(obj: Any, *path: str) -> Any:
    for name in path:
        if obj is None:
            return None
        obj = getattr(obj, name, None)
    return obj


def _first_positive(*values: Any) -> float | None:
    for value in values:
        if isinstance(value, (int, float)) and value > 0:
            return float(value)
    return None


def parse_snapshot(snap: Any) -> SnapshotQuote | None:
    """Reduce a massive TickerSnapshot to (ticker, price, timestamp), or None if unusable."""
    raw_ticker = getattr(snap, "ticker", None)
    if not raw_ticker:
        return None

    price = _first_positive(
        _nested(snap, "last_trade", "price"),
        _nested(snap, "min", "close"),
        _nested(snap, "day", "close"),
        _nested(snap, "prev_day", "close"),
    )
    if price is None:
        return None

    raw_ts = _first_positive(
        _nested(snap, "last_trade", "sip_timestamp"),
        getattr(snap, "updated", None),
    )
    timestamp = to_unix_seconds(raw_ts) if raw_ts is not None else time.time()
    return SnapshotQuote(ticker=str(raw_ticker).upper(), price=price, timestamp=timestamp)


class MassiveDataSource(MarketDataSource):
    """Polls the Massive snapshot endpoint for all tracked tickers in one request.

    Rate budget: free tier = 5 requests/min. Polling every 15s uses 4/min, which
    leaves 1/min for add_ticker() validation calls.
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
        client: Any | None = None,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        # retries=0: the poll loop is the retry mechanism. urllib3's built-in
        # retries would burn free-tier quota on 429s.
        self._client = client if client is not None else RESTClient(api_key=api_key, retries=0)

    # --- MarketDataSource ---

    async def start(self, tickers: list[str]) -> None:
        for raw in tickers:
            try:
                ticker = normalize_ticker(raw)
            except InvalidTickerError:
                logger.warning("Massive: skipping malformed ticker %r", raw)
                continue
            if ticker not in self._tickers:
                self._tickers.append(ticker)

        # Startup tickers come from our own DB, so they are not validated one by
        # one (that would cost one request per ticker). Unknown ones simply never
        # get a price. First poll is immediate so the UI has data at once.
        await self._poll_once()
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info("Massive poller started: %d tickers every %.1fs", len(self._tickers), self._interval)

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        if ticker in self._tickers:
            return

        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots, [ticker])
        except Exception as exc:  # BadResponse (429, 5xx), network errors, ...
            logger.warning("Massive: could not validate %s: %s", ticker, exc)
            raise InvalidTickerError(ticker, "Could not verify ticker with Massive, try again") from exc

        quote = next(
            (q for q in map(parse_snapshot, snapshots) if q is not None and q.ticker == ticker),
            None,
        )
        if quote is None:
            raise InvalidTickerError(ticker, "Unknown ticker")

        if ticker not in self._tickers:  # re-check: another add may have raced the await
            self._tickers.append(ticker)
        if ticker not in self._cache:
            self._cache.update(ticker, quote.price, quote.timestamp)
        logger.info("Massive: added %s at %.2f", ticker, quote.price)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = normalize_ticker(ticker)
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Polling ---

    async def _poll_loop(self) -> None:
        """Sleep, poll, repeat. Back off exponentially while polls keep failing."""
        failures = 0
        while True:
            delay = min(self._interval * (2**failures), MAX_BACKOFF_SECONDS)
            await asyncio.sleep(delay)
            ok = await self._poll_once()
            failures = 0 if ok else failures + 1

    async def _poll_once(self) -> bool:
        """One poll cycle. Returns False if the request failed (the cache keeps old prices)."""
        tickers = list(self._tickers)  # snapshot: the list may change while we await
        if not tickers:
            return True

        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots, tickers)
        except Exception as exc:
            # 401 bad key, 429 rate limit, 5xx, timeouts. Keep serving the last prices.
            logger.error("Massive poll failed: %s", exc)
            return False

        tracked = set(self._tickers)  # re-read AFTER the await (fix: in-flight removal race)
        updated = 0
        for snap in snapshots:
            quote = parse_snapshot(snap)
            if quote is None:
                logger.warning("Massive: unusable snapshot for %s", getattr(snap, "ticker", "?"))
                continue
            if quote.ticker not in tracked:
                continue  # removed while the request was in flight
            self._cache.update(quote.ticker, quote.price, quote.timestamp)
            updated += 1

        logger.debug("Massive poll: %d/%d tickers updated", updated, len(tickers))
        return True

    def _fetch_snapshots(self, tickers: list[str]) -> list:
        """Blocking HTTP call; always run via asyncio.to_thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=tickers,
        )
```

### 9.4 Design notes

- **Thread boundary.** Only `_fetch_snapshots` runs in a worker thread, and it gets an immutable copy of the ticker list. All mutation of `self._tickers` and all cache writes happen on the event loop.
- **Validation cost.** Each *new* ticker costs one request. Tickers that are already tracked, such as a held position being re-added to the watchlist, cost nothing. On the free tier, adding several tickers within a minute may get a 429. The user then sees "Could not verify ticker with Massive, try again", which is honest and recoverable.
- **Why startup doesn't validate.** Startup tickers come from our own database, and they were validated when they were added. Validating 10+ tickers one by one would blow the free-tier budget in the first minute.
- **Stale prices are accepted** (PLAN §6). Off-hours, polls return the same last trade, so `direction` is `"flat"` and the version still bumps. The README must say updates are every 15 s and stale outside market hours.
- **Backoff.** Repeated failures stretch the interval 15 → 30 → 60 → 120 s, then hold. One success resets it. This stops a bad key or an exhausted quota from producing a 429 storm.

---

## 10. Factory — `factory.py`

```python
# backend/app/market/factory.py
"""Select and construct the market data source from environment variables."""

from __future__ import annotations

import logging
import os
from collections.abc import Mapping

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)

DEFAULT_POLL_INTERVAL = 15.0
MIN_POLL_INTERVAL = 2.0


def _poll_interval(env: Mapping[str, str]) -> float:
    raw = env.get("MASSIVE_POLL_INTERVAL", "").strip()
    if not raw:
        return DEFAULT_POLL_INTERVAL
    try:
        value = float(raw)
    except ValueError:
        logger.warning("Invalid MASSIVE_POLL_INTERVAL=%r; using %.0fs", raw, DEFAULT_POLL_INTERVAL)
        return DEFAULT_POLL_INTERVAL
    if value < MIN_POLL_INTERVAL:
        logger.warning("MASSIVE_POLL_INTERVAL=%s too low; clamping to %.0fs", raw, MIN_POLL_INTERVAL)
        return MIN_POLL_INTERVAL
    return value


def create_market_data_source(
    price_cache: PriceCache,
    env: Mapping[str, str] | None = None,
) -> MarketDataSource:
    """Return an UNSTARTED source; the caller must `await source.start(tickers)`.

    - MASSIVE_API_KEY set and non-blank -> MassiveDataSource polling every
      MASSIVE_POLL_INTERVAL seconds (default 15, minimum 2)
    - otherwise -> SimulatorDataSource
    """
    env = os.environ if env is None else env
    api_key = env.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        interval = _poll_interval(env)
        logger.info("Market data source: Massive API (poll every %.1fs)", interval)
        return MassiveDataSource(api_key=api_key, price_cache=price_cache, poll_interval=interval)

    logger.info("Market data source: GBM simulator")
    return SimulatorDataSource(price_cache=price_cache)
```

The `env` parameter lets tests pass a plain dict instead of using `monkeypatch.setenv`. In local development, `python-dotenv` must load `../.env` **before** the factory runs (PLAN §5). In Docker, `--env-file` provides the variables.

| `MASSIVE_API_KEY` | `MASSIVE_POLL_INTERVAL` | Result |
|-------------------|-------------------------|--------|
| unset / `""` / `"  "` | any | `SimulatorDataSource` |
| `"abc"` | unset / `""` | Massive, 15 s |
| `"abc"` | `"5"` | Massive, 5 s |
| `"abc"` | `"0.5"` | Massive, 2 s (clamped, warning) |
| `"abc"` | `"fast"` | Massive, 15 s (warning) |

---

## 11. SSE Stream — `stream.py`

```python
# backend/app/market/stream.py
"""SSE endpoint: GET /api/stream/prices."""

from __future__ import annotations

import asyncio
import json
import logging
import time
from collections.abc import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache
from .models import PriceUpdate

logger = logging.getLogger(__name__)

SSE_HEADERS = {
    "Cache-Control": "no-cache",
    "Connection": "keep-alive",
    "X-Accel-Buffering": "no",  # disable proxy buffering (nginx, some PaaS routers)
}
HEARTBEAT_SECONDS = 15.0


def format_price_event(prices: dict[str, PriceUpdate]) -> str:
    """One SSE `data:` frame holding ALL tracked tickers, keyed by symbol (PLAN §6)."""
    payload = {ticker: update.to_dict() for ticker, update in sorted(prices.items())}
    return f"data: {json.dumps(payload, separators=(',', ':'))}\n\n"


async def generate_price_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
    heartbeat: float = HEARTBEAT_SECONDS,
) -> AsyncGenerator[str, None]:
    """Yield `retry:` first, then a full snapshot whenever the cache version moves."""
    yield "retry: 1000\n\n"

    client = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client)
    last_version = -1
    last_sent = time.monotonic()
    try:
        while not await request.is_disconnected():
            version = price_cache.version
            if version != last_version:
                last_version = version
                # Sent even when empty: "{}" tells clients the last ticker went away.
                yield format_price_event(price_cache.get_all())
                last_sent = time.monotonic()
            elif time.monotonic() - last_sent >= heartbeat:
                yield ": ping\n\n"  # SSE comment; EventSource ignores it
                last_sent = time.monotonic()
            await asyncio.sleep(interval)
    finally:
        logger.info("SSE client disconnected: %s", client)


def create_stream_router(price_cache: PriceCache, interval: float = 0.5) -> APIRouter:
    """Build a fresh router bound to `price_cache` (safe to call more than once)."""
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            generate_price_events(price_cache, request, interval),
            media_type="text/event-stream",
            headers=SSE_HEADERS,
        )

    return router
```

**Notes**

- **Wire format.** The first frame is `retry: 1000`. After that there is one `data:` frame per version change, holding all tickers, plus `: ping` comments when idle. No `event:` names are used, so the client listens with `onmessage`.
- **Version/read race.** `version` and `get_all()` are separate lock acquisitions. If an update lands between them, we send slightly newer data and resend once on the next tick. That's harmless and keeps the cache API simple.
- **Disconnect.** `request.is_disconnected()` is checked every tick. Starlette also cancels the generator when the client goes away, and the `finally` block logs either way. Cancellation propagates instead of being swallowed.
- **`sorted()`** gives a stable key order, which makes test assertions and debugging easier.

---

## 12. Package Exports — `__init__.py`

```python
# backend/app/market/__init__.py
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate, PriceCache, MarketDataSource
    create_market_data_source, create_stream_router
    normalize_ticker, InvalidTickerError
"""

from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router
from .tickers import InvalidTickerError, normalize_ticker

__all__ = [
    "InvalidTickerError",
    "MarketDataSource",
    "PriceCache",
    "PriceUpdate",
    "create_market_data_source",
    "create_stream_router",
    "normalize_ticker",
]
```

Code outside `app/market/` imports only from `app.market`.

---

## 13. Integrating with the Rest of the Backend

This section is the contract for the backend agent who builds `main.py`, the DB layer and the routes. Function names in the `db` module are placeholders; that layer is not part of the market data component.

### 13.1 App construction and lifespan (PLAN §3 startup order)

```python
# backend/app/main.py — sketch; DB/portfolio helpers are owned by the backend agent
import asyncio
import contextlib
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.staticfiles import StaticFiles

from app import db
from app.market import (
    InvalidTickerError,
    PriceCache,
    create_market_data_source,
    create_stream_router,
)


def create_app() -> FastAPI:
    price_cache = PriceCache()
    source = create_market_data_source(price_cache)  # env must already be loaded

    @asynccontextmanager
    async def lifespan(app: FastAPI):
        db.init_db()                                           # 1. schema + seed
        await source.start(db.get_tracked_tickers())           # 2. watchlist ∪ positions
        db.record_snapshot(price_cache)                        # 3. initial snapshot
        snapshot_task = asyncio.create_task(                   # 4. every 30s
            db.snapshot_loop(price_cache, interval=30.0), name="snapshot-loop"
        )
        try:
            yield
        finally:
            snapshot_task.cancel()
            with contextlib.suppress(asyncio.CancelledError):
                await snapshot_task
            await source.stop()

    app = FastAPI(title="FinAlly", lifespan=lifespan)
    app.state.price_cache = price_cache
    app.state.market_source = source

    @app.exception_handler(InvalidTickerError)
    async def _invalid_ticker(request: Request, exc: InvalidTickerError) -> JSONResponse:
        return JSONResponse(status_code=400, content={"error": exc.reason, "code": "INVALID_TICKER"})

    app.include_router(create_stream_router(price_cache))
    # app.include_router(portfolio_router); watchlist_router; chat_router; health_router
    app.mount("/", StaticFiles(directory="static", html=True), name="static")  # LAST (PLAN §11)
    return app


app = create_app()
```

The stream router is included at construction time, not inside `lifespan`. Routers added after startup would land *behind* the `/` static mount and be unreachable.

`db.get_tracked_tickers()`:

```sql
SELECT ticker FROM watchlist WHERE user_id = 'default'
UNION
SELECT ticker FROM positions WHERE user_id = 'default'
```

### 13.2 Dependencies for routes

```python
# backend/app/deps.py
from fastapi import Request

from app.market import MarketDataSource, PriceCache


def get_price_cache(request: Request) -> PriceCache:
    return request.app.state.price_cache


def get_market_source(request: Request) -> MarketDataSource:
    return request.app.state.market_source
```

### 13.3 Tracked-ticker coordination

PLAN §6 says to track **watchlist ∪ open positions**. Two helpers keep the source in sync. Every route, and the chat action executor, goes through them, never through `source.add_ticker`/`remove_ticker` directly.

```python
# backend/app/services/tracking.py
from app import db
from app.market import MarketDataSource


async def ensure_tracked(source: MarketDataSource, ticker: str) -> None:
    """Start tracking (no-op if tracked). Raises InvalidTickerError if unpriceable."""
    await source.add_ticker(ticker)


async def release_if_unused(source: MarketDataSource, ticker: str) -> None:
    """Stop tracking a ticker that is neither watched nor held."""
    if not db.in_watchlist(ticker) and not db.has_position(ticker):
        await source.remove_ticker(ticker)
```

| Flow | Steps |
|------|-------|
| **POST /api/watchlist** `{ticker}` | `t = normalize_ticker(...)` (400 `INVALID_TICKER`) → `await ensure_tracked(source, t)` (400 if unknown in Massive mode) → `INSERT OR IGNORE` into watchlist → return `{ticker, price, session_change_percent, added_at}` from `cache.get(t)` (price `null` if absent). Idempotent: an existing entry returns 200 with the same shape. |
| **DELETE /api/watchlist/{ticker}** | normalize → 404 `NOT_IN_WATCHLIST` if absent → delete row → `await release_if_unused(source, t)`. A held ticker keeps streaming. |
| **POST /api/portfolio/trade** | normalize → `await ensure_tracked(source, t)` (a trade on an untracked ticker starts tracking it, PLAN §6) → `price = cache.get_price(t)`; if `None`, 400 `NO_PRICE` → run the trade transaction (PLAN §8) → **always** (success or failure) `await release_if_unused(source, t)`. This drops a ticker whose buy failed and a position sold to zero that is not on the watchlist. |
| **Chat actions** | the same functions as the two rows above, called per action. Each failure is recorded (`status: "failed"`, `error: exc.reason`) and execution continues. |

Why the order **validate in the source first, then write the DB**: if the source rejects the ticker, nothing was written, so there's nothing to roll back.

### 13.4 Reading prices elsewhere

```python
# Portfolio valuation: price unavailable → value at avg_cost (PLAN §8)
price = cache.get_price(pos.ticker)
current = price if price is not None else pos.avg_cost

# GET /api/watchlist item
u = cache.get(row.ticker)
item = {
    "ticker": row.ticker,
    "price": u.price if u else None,
    "session_change_percent": u.session_change_percent if u else None,
    "added_at": row.added_at,
}

# LLM context: watchlist with live prices
lines = [f"{t}: {u.price:.2f} ({u.session_change_percent:+.2f}% today)"
         for t, u in cache.get_all().items()]
```

### 13.5 Error mapping

| Raised by market layer | HTTP | Body |
|------------------------|------|------|
| `InvalidTickerError` (malformed) | 400 | `{"error": "Invalid ticker", "code": "INVALID_TICKER"}` |
| `InvalidTickerError` (Massive: no snapshot) | 400 | `{"error": "Unknown ticker", "code": "INVALID_TICKER"}` |
| `InvalidTickerError` (Massive: API error during validation) | 400 | `{"error": "Could not verify ticker with Massive, try again", "code": "INVALID_TICKER"}` |
| `cache.get_price(t) is None` at trade time (caller-detected) | 400 | `{"error": "No price yet for AAPL", "code": "NO_PRICE"}` |

---

## 14. Worked Examples

### 14.1 Standalone use (scripts, REPL, demo)

```python
import asyncio
from app.market import PriceCache, create_market_data_source

async def main():
    cache = PriceCache()
    source = create_market_data_source(cache)          # simulator unless MASSIVE_API_KEY
    await source.start(["AAPL", "TSLA", "NVDA"])

    print(cache.get("AAPL").to_dict())
    # {'ticker': 'AAPL', 'price': 190.0, 'previous_price': 190.0, 'timestamp': ...,
    #  'change': 0.0, 'change_percent': 0.0, 'direction': 'flat',
    #  'session_change_percent': 0.0}

    await source.add_ticker("pypl")                    # normalized to PYPL, priced at once
    await asyncio.sleep(2)
    for t, u in cache.get_all().items():
        print(f"{t:5} {u.price:9.2f} {u.direction:>4} {u.session_change_percent:+.4f}%")

    await source.remove_ticker("TSLA")
    await source.stop()

asyncio.run(main())
```

### 14.2 Watching the stream

```bash
$ curl -N http://localhost:8000/api/stream/prices
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.0,"previous_price":190.0,"timestamp":1791369600.0,"change":0.0,"change_percent":0.0,"direction":"flat","session_change_percent":0.0},"AMZN":{...}}

data: {"AAPL":{"ticker":"AAPL","price":190.02,"previous_price":190.0,"timestamp":1791369600.5,"change":0.02,"change_percent":0.0105,"direction":"up","session_change_percent":0.0105},"AMZN":{...}}
```

### 14.3 Frontend consumption (reference for the frontend agent)

```ts
type PriceTick = {
  ticker: string; price: number; previous_price: number; timestamp: number;
  change: number; change_percent: number; direction: "up" | "down" | "flat";
  session_change_percent: number;
};

const es = new EventSource("/api/stream/prices");
es.onopen = () => setStatus("connected");                       // green
es.onerror = () => setStatus(es.readyState === EventSource.CLOSED
  ? "disconnected" : "reconnecting");                           // red / yellow
es.onmessage = (e) => {
  const prices: Record<string, PriceTick> = JSON.parse(e.data);
  for (const p of Object.values(prices)) {
    appendHistory(p.ticker, p.timestamp, p.price, /* cap */ 300);
    if (p.direction !== "flat") flash(p.ticker, p.direction);
  }
};
```

### 14.4 Massive mode, end to end

```bash
MASSIVE_API_KEY=xxxxx MASSIVE_POLL_INTERVAL=15 uv run uvicorn app.main:app --port 8000
```

```
INFO  Market data source: Massive API (poll every 15.0s)
INFO  Massive poller started: 10 tickers every 15.0s
...
POST /api/watchlist {"ticker":"pypl"}  → 200 {"ticker":"PYPL","price":71.42,...}
POST /api/watchlist {"ticker":"ZZZZZ"} → 400 {"error":"Unknown ticker","code":"INVALID_TICKER"}
```

---

## 15. Testing

Tests live in `backend/tests/market/`. Run them with `uv run --extra dev pytest -v`. `asyncio_mode = "auto"` is already configured. The existing 73 tests stay; the changes below update or add tests.

### 15.1 Tickers

```python
# tests/market/test_tickers.py
import pytest

from app.market.tickers import InvalidTickerError, normalize_ticker


@pytest.mark.parametrize("raw,expected", [("aapl", "AAPL"), (" brk.b ", "BRK.B"), ("V", "V")])
def test_normalize_valid(raw, expected):
    assert normalize_ticker(raw) == expected


@pytest.mark.parametrize("raw", ["", "  ", "AAPL1", "BRK-B", "ABCDEFGHIJK", "$TSLA", None])
def test_normalize_invalid(raw):
    with pytest.raises(InvalidTickerError):
        normalize_ticker(raw)
```

### 15.2 Model and cache session fields

```python
# tests/market/test_cache.py  (additions)
from app.market.cache import PriceCache


def test_session_start_is_first_price():
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    cache.update("AAPL", 191.00)
    u = cache.update("AAPL", 190.95)
    assert u.session_start_price == 190.00
    assert u.session_change_percent == 0.5
    assert u.to_dict()["session_change_percent"] == 0.5


def test_remove_resets_session_and_bumps_version():
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    v = cache.version
    cache.remove("AAPL")
    assert cache.version == v + 1
    assert cache.update("AAPL", 200.00).session_change_percent == 0.0


def test_remove_unknown_does_not_bump_version():
    cache = PriceCache()
    v = cache.version
    cache.remove("NOPE")
    assert cache.version == v


def test_payload_matches_plan_example():
    from app.market.models import PriceUpdate
    u = PriceUpdate("AAPL", 190.52, 190.41, 1791369600.123, session_start_price=190.00)
    assert u.to_dict() == {
        "ticker": "AAPL", "price": 190.52, "previous_price": 190.41,
        "timestamp": 1791369600.123, "change": 0.11, "change_percent": 0.0578,
        "direction": "up", "session_change_percent": 0.2737,
    }
```

### 15.3 Massive parsing against the **real** `massive` models

This is the test that would have caught the `last_trade.timestamp` bug. Build snapshots from raw JSON with `TickerSnapshot.from_dict`, not `MagicMock`.

```python
# tests/market/test_massive_parse.py
from massive.rest.models import TickerSnapshot

from app.market.massive_client import parse_snapshot, to_unix_seconds

RAW = {
    "ticker": "AAPL",
    "updated": 1791369599123456789,
    "day": {"c": 190.90},
    "min": {"c": 190.95, "t": 1791369540000},
    "prevDay": {"c": 189.34},
    "lastTrade": {"p": 190.97, "s": 100, "t": 1791369599123456789, "x": 4},
}


def test_parse_real_model():
    q = parse_snapshot(TickerSnapshot.from_dict(RAW))
    assert q.ticker == "AAPL"
    assert q.price == 190.97
    assert abs(q.timestamp - 1791369599.123) < 0.001


def test_fallback_to_prev_close_when_no_trade():
    raw = {"ticker": "AAPL", "day": {"c": 0}, "prevDay": {"c": 189.34}}
    q = parse_snapshot(TickerSnapshot.from_dict(raw))
    assert q.price == 189.34  # timestamp falls back to now


def test_unusable_snapshot():
    assert parse_snapshot(TickerSnapshot.from_dict({"ticker": "X"})) is None
    assert parse_snapshot(TickerSnapshot.from_dict({"lastTrade": {"p": 1.0}})) is None


def test_to_unix_seconds_units():
    s = 1791369600.0
    for raw in (s, s * 1e3, s * 1e6, s * 1e9):
        assert abs(to_unix_seconds(raw) - s) < 1e-3
```

### 15.4 Massive source with a fake client (no patching)

```python
# tests/market/test_massive.py  (rewrite using client injection)
import asyncio

import pytest
from massive.rest.models import TickerSnapshot

from app.market.cache import PriceCache
from app.market.massive_client import MassiveDataSource
from app.market.tickers import InvalidTickerError


def snap(ticker, price, ts_ns=1791369600_000_000_000):
    return TickerSnapshot.from_dict({"ticker": ticker, "lastTrade": {"p": price, "t": ts_ns}})


class FakeClient:
    def __init__(self, prices=None, error=None):
        self.prices = dict(prices or {})
        self.error = error
        self.calls: list[list[str]] = []

    def get_snapshot_all(self, market_type, tickers):
        self.calls.append(list(tickers))
        if self.error:
            raise self.error
        return [snap(t, self.prices[t]) for t in tickers if t in self.prices]


def make(prices=None, error=None):
    cache = PriceCache()
    client = FakeClient(prices, error)
    return cache, client, MassiveDataSource("k", cache, poll_interval=60, client=client)


async def test_start_polls_immediately():
    cache, client, src = make({"AAPL": 190.5, "MSFT": 420.0})
    await src.start(["aapl", "MSFT"])
    assert cache.get_price("AAPL") == 190.5
    assert client.calls == [["AAPL", "MSFT"]]
    await src.stop()


async def test_add_known_ticker_seeds_cache():
    cache, client, src = make({"PYPL": 71.42})
    await src.add_ticker("pypl")
    assert src.get_tickers() == ["PYPL"]
    assert cache.get_price("PYPL") == 71.42


async def test_add_unknown_ticker_rejected():
    cache, client, src = make({})
    with pytest.raises(InvalidTickerError, match="Unknown ticker"):
        await src.add_ticker("ZZZZ")
    assert src.get_tickers() == []
    assert "ZZZZ" not in cache


async def test_add_api_error_rejected():
    cache, client, src = make(error=RuntimeError("429"))
    with pytest.raises(InvalidTickerError, match="try again"):
        await src.add_ticker("AAPL")


async def test_add_already_tracked_costs_no_request():
    cache, client, src = make({"AAPL": 190.0})
    await src.start(["AAPL"])
    client.calls.clear()
    await src.add_ticker("AAPL")
    assert client.calls == []
    await src.stop()


async def test_poll_failure_keeps_last_price():
    cache, client, src = make({"AAPL": 190.0})
    await src.start(["AAPL"])
    client.error = RuntimeError("boom")
    assert await src._poll_once() is False
    assert cache.get_price("AAPL") == 190.0
    await src.stop()


async def test_removed_during_poll_not_rewritten():
    cache, client, src = make({"AAPL": 190.0, "MSFT": 420.0})
    src._tickers = ["AAPL", "MSFT"]
    real = client.get_snapshot_all

    def slow(market_type, tickers):
        result = real(market_type, tickers)
        asyncio.run_coroutine_threadsafe(src.remove_ticker("MSFT"), loop).result()
        return result

    loop = asyncio.get_running_loop()
    client.get_snapshot_all = slow
    await src._poll_once()
    assert "MSFT" not in cache
    assert cache.get_price("AAPL") == 190.0
```

### 15.5 Simulator

The existing tests keep passing. Add:

```python
# tests/market/test_simulator.py  (additions)
from app.market.simulator import GBMSimulator


def test_seeded_runs_are_reproducible():
    a = GBMSimulator(["AAPL", "MSFT"], seed=42)
    b = GBMSimulator(["AAPL", "MSFT"], seed=42)
    assert [a.step() for _ in range(50)] == [b.step() for _ in range(50)]


def test_full_default_set_builds_cholesky():
    from app.market.seed_prices import SEED_PRICES
    sim = GBMSimulator(list(SEED_PRICES) + ["PYPL", "BRK.B"])
    assert sim._cholesky is not None and sim._cholesky.shape == (12, 12)


def test_correlated_tickers_move_together():
    import numpy as np
    sim = GBMSimulator(["AAPL", "MSFT"], event_probability=0.0, seed=1)
    prev = {t: sim._prices[t] for t in ("AAPL", "MSFT")}
    ra, rm = [], []
    for _ in range(5000):
        sim.step()
        ra.append(np.log(sim._prices["AAPL"] / prev["AAPL"]))
        rm.append(np.log(sim._prices["MSFT"] / prev["MSFT"]))
        prev = {t: sim._prices[t] for t in ("AAPL", "MSFT")}
    assert 0.5 < np.corrcoef(ra, rm)[0, 1] < 0.7   # target 0.6
```

```python
# tests/market/test_simulator_source.py  (additions)
import pytest

from app.market.cache import PriceCache
from app.market.simulator import SimulatorDataSource
from app.market.tickers import InvalidTickerError


async def test_add_ticker_normalizes_and_prices_immediately():
    cache = PriceCache()
    src = SimulatorDataSource(cache, update_interval=10)
    await src.start(["AAPL"])
    await src.add_ticker(" pypl ")
    assert "PYPL" in src.get_tickers()
    assert 50 <= cache.get_price("PYPL") <= 300
    assert cache.get("PYPL").session_change_percent == 0.0
    await src.stop()


async def test_add_malformed_rejected():
    src = SimulatorDataSource(PriceCache())
    with pytest.raises(InvalidTickerError):
        await src.add_ticker("NOT-VALID")


async def test_start_skips_malformed_db_rows():
    cache = PriceCache()
    src = SimulatorDataSource(cache, update_interval=10)
    await src.start(["AAPL", "bad-one"])
    assert src.get_tickers() == ["AAPL"]
    await src.stop()
```

### 15.6 Factory

```python
# tests/market/test_factory.py  (additions)
from app.market.cache import PriceCache
from app.market.factory import create_market_data_source
from app.market.massive_client import MassiveDataSource
from app.market.simulator import SimulatorDataSource


def test_blank_key_uses_simulator():
    assert isinstance(create_market_data_source(PriceCache(), {"MASSIVE_API_KEY": "  "}),
                      SimulatorDataSource)


def test_poll_interval_read_and_clamped():
    def interval(v):
        env = {"MASSIVE_API_KEY": "k", "MASSIVE_POLL_INTERVAL": v}
        src = create_market_data_source(PriceCache(), env)
        assert isinstance(src, MassiveDataSource)
        return src._interval

    assert interval("") == 15.0
    assert interval("5") == 5.0
    assert interval("0.5") == 2.0
    assert interval("fast") == 15.0
```

### 15.7 SSE generator

Test the generator directly with a fake request. `httpx.ASGITransport` buffers whole responses, so it can't read an infinite stream.

```python
# tests/market/test_stream.py
import json

from app.market.cache import PriceCache
from app.market.stream import create_stream_router, format_price_event, generate_price_events


class FakeRequest:
    client = None

    def __init__(self, ticks: int):
        self._ticks = ticks

    async def is_disconnected(self) -> bool:
        self._ticks -= 1
        return self._ticks < 0


async def collect(cache, ticks, **kw):
    return [f async for f in generate_price_events(cache, FakeRequest(ticks), interval=0, **kw)]


async def test_first_frame_is_retry_then_snapshot():
    cache = PriceCache()
    cache.update("AAPL", 190.0)
    cache.update("MSFT", 420.0)
    frames = await collect(cache, ticks=3)
    assert frames[0] == "retry: 1000\n\n"
    payload = json.loads(frames[1].removeprefix("data: "))
    assert set(payload) == {"AAPL", "MSFT"}
    assert payload["AAPL"]["session_change_percent"] == 0.0
    assert len(frames) == 2  # nothing changed afterwards -> no resend


async def test_heartbeat_when_idle():
    frames = await collect(PriceCache(), ticks=3, heartbeat=0)
    assert ": ping\n\n" in frames


def test_format_is_single_line_json():
    cache = PriceCache()
    cache.update("AAPL", 190.0)
    frame = format_price_event(cache.get_all())
    assert frame.startswith("data: {") and frame.endswith("}\n\n") and "\n" not in frame[:-2]


def test_router_factory_is_reentrant():
    r1, r2 = create_stream_router(PriceCache()), create_stream_router(PriceCache())
    assert r1 is not r2 and len(r1.routes) == len(r2.routes) == 1
```

### 15.8 Tests that need updating

- `test_massive.py`: the existing tests build `MagicMock` snapshots with `last_trade.timestamp` and patch `RESTClient`. Replace them with the fake-client tests in §15.4.
- `test_simulator.py`: tests that assert `random.uniform` behavior or patch `random`/`np.random` should use `seed=` instead.
- `test_cache.py`: no breaking changes; `remove()` now bumps `version` only when something was removed.

---

## 16. Edge Cases and Failure Modes

| Situation | Behavior |
|-----------|----------|
| Empty tracked set at startup | Simulator `step()` returns `{}`. Massive skips the request. SSE sends one `{}` frame, then heartbeats. |
| User removes the last watched ticker that is not held | Cache `remove` bumps version, so SSE sends `{}` or the reduced set; the frontend drops that row. |
| Remove a watched ticker that is still held | `release_if_unused` keeps it tracked. It keeps streaming and stays valued. |
| Trade on a ticker nobody tracks | `ensure_tracked` starts it (simulator: seed/random price; Massive: validation snapshot price), then the trade fills at that price. |
| Buy fails (insufficient cash) on a newly tracked ticker | `release_if_unused` after the attempt stops tracking it again. |
| Massive: invalid API key | Startup succeeds; every poll logs `BadResponse` 401 and backs off to 120 s; no prices; trades get `NO_PRICE`; watchlist adds get "Could not verify…". The fix is to correct `.env` and restart. |
| Massive: 429 rate limit | That poll fails and the cache keeps the last prices. Backoff doubles the delay until a success. |
| Massive: ticker delisted mid-session | It is omitted from the response and its cache entry goes stale (last price stays). Acceptable. |
| Massive: weekend / overnight | The same last trade repeats; `direction` is `"flat"` and session change is ~0. Documented as accepted. |
| Massive: midnight–4am ET (`day` cleared) | `last_trade` → `prev_day.close` fallback still yields a price. |
| Simulator unknown ticker | Random U(50, 300) start, σ = 0.25, μ = 0.05, 0.3 correlation with all. |
| Re-adding a removed ticker | New session baseline (§5). Simulator restarts it from seed/random. |
| Two `add_ticker("X")` calls racing (Massive) | Both may validate (2 requests), but the re-check after `await` prevents a duplicate entry. |
| Client disconnects from SSE | The generator exits on the next tick or on cancellation, and the `finally` block logs it. |
| Simulator step raises (should never happen) | Logged with traceback; the loop continues on the next tick. |

---

## 17. Configuration Reference

| Setting | Where | Default | Notes |
|---------|-------|---------|-------|
| `MASSIVE_API_KEY` | env | unset | non-blank → Massive, else simulator |
| `MASSIVE_POLL_INTERVAL` | env | `15` | seconds; min 2; free tier must stay ≥ 15 |
| `update_interval` | `SimulatorDataSource(...)` | `0.5` s | tick cadence |
| `event_probability` | `SimulatorDataSource/GBMSimulator(...)` | `0.001` | shock chance per ticker per tick |
| `seed` | `SimulatorDataSource/GBMSimulator(...)` | `None` | set in tests for determinism |
| `dt` | `GBMSimulator(...)` | `8.48e-8` | one 500 ms tick in trading-years |
| `MAX_BACKOFF_SECONDS` | `massive_client.py` | `120` | cap for failed-poll backoff |
| SSE interval | `create_stream_router(cache, interval=)` | `0.5` s | push cadence |
| `HEARTBEAT_SECONDS` | `stream.py` | `15` | idle `: ping` cadence |
| SSE `retry:` | `stream.py` | `1000` ms | browser reconnect delay |
| Ticker pattern | `tickers.py` | `^[A-Z.]{1,10}$` | after trim + uppercase |

---

## 18. Implementation Checklist

Work in this order. Each step should leave `uv run --extra dev pytest` and `uv run --extra dev ruff check app tests` green.

1. [ ] Add `tickers.py` and `test_tickers.py` (§3, §15.1).
2. [ ] Update `models.py` and `cache.py` with the session-start price, the version bump on remove, and the locked `version` (§4, §5); add the §15.2 tests.
3. [ ] Update the `interface.py` docstrings (§6).
4. [ ] Update `simulator.py`: RNG/seed, normalization, `_track()` seeding, malformed-startup skip (§8); add the §15.5 tests and fix any tests that patched `random`.
5. [ ] Rewrite `massive_client.py`: `parse_snapshot`, `to_unix_seconds`, client injection, `retries=0`, validation in `add_ticker`, in-flight race fix, backoff (§9). Replace the `MagicMock`-based tests with §15.3 and §15.4.
6. [ ] Update `factory.py` for `MASSIVE_POLL_INTERVAL` and the `env` parameter (§10); add the §15.6 tests.
7. [ ] Update `stream.py`: per-call router, `format_price_event`, heartbeat, no swallowed cancellation (§11); add `test_stream.py` (§15.7).
8. [ ] Update the `__init__.py` exports (§12) and `backend/CLAUDE.md` (mention `session_change_percent`, `normalize_ticker`, `InvalidTickerError`, `MASSIVE_POLL_INTERVAL`).
9. [ ] Update `market_data_demo.py` to show `session_change_percent` as "Day %".
10. [ ] Hand off to the backend agent: §13 (lifespan order, `get_tracked_tickers`, `ensure_tracked`/`release_if_unused`, error mapping).
11. [ ] Update `planning/MARKET_DATA_SUMMARY.md` and remove the "Code changes implied by these decisions" items for the market data component from PLAN.md §13.
