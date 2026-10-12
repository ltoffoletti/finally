# Unified Market Data Interface

How FinAlly gets stock prices. One Python API, two interchangeable sources:

- **Massive** (real data) when `MASSIVE_API_KEY` is set and non-empty
- **Simulator** (GBM, see `MARKET_SIMULATOR.md`) otherwise

Everything downstream (SSE stream, portfolio valuation, trade execution, chat) reads prices from one in-memory **`PriceCache`** and never talks to a source directly.

This document is the contract for the code in `backend/app/market/`. Most of it already exists. **§9 lists the changes still to make**, which come from PLAN.md §13 and from the API research in `MASSIVE_API.md`. One of them (§9 item 1) fixes a bug that currently stops Massive mode from producing any prices.

> Supersedes `planning/archive/MARKET_INTERFACE.md`.

---

## 1. Shape of the system

```
                      create_market_data_source(cache)        ← reads env
                                    │
              ┌─────────────────────┴─────────────────────┐
              ▼                                           ▼
   SimulatorDataSource                          MassiveDataSource
   (GBM step every 0.5 s)              (live: snapshot poll every N s
              │                          eod:  grouped-daily refresh hourly)
              │        writes                             │
              └──────────────►  PriceCache  ◄─────────────┘
                                    │  reads
          ┌─────────────┬───────────┼──────────────┬───────────────┐
          ▼             ▼           ▼              ▼               ▼
   SSE /api/stream  GET /api/    trade         GET /api/       chat context
      /prices       portfolio    execution     watchlist       (LLM prompt)
```

Design rules:

1. **Strategy pattern.** Both sources implement the `MarketDataSource` ABC. Only the factory knows which one is in use.
2. **The cache is the single source of truth.** A source *pushes* into the cache on its own schedule. Consumers *pull* from the cache. Nobody waits on the network to read a price.
3. **Sources own the set of tracked tickers.** The application decides *which* tickers to track (watchlist ∪ open positions, PLAN §6) and tells the source through `add_ticker` / `remove_ticker`.
4. **One process, one source, one cache.** They are created in the FastAPI `lifespan` and stored on `app.state` (PLAN §3: uvicorn runs exactly one worker).

---

## 2. Public API (what other agents import)

```python
from app.market import (
    PriceCache,                 # the shared cache
    PriceUpdate,                # immutable price record
    MarketDataSource,           # ABC (for type hints)
    create_market_data_source,  # factory
    create_stream_router,       # SSE router factory
    normalize_ticker,           # "aapl " -> "AAPL", or raises InvalidTickerError
    InvalidTickerError,         # malformed symbol          -> 400 INVALID_TICKER
    UnknownTickerError,         # well-formed, no data      -> 400 INVALID_TICKER
)
```

### 2.1 Reading prices: `PriceCache`

```python
cache.get("AAPL")          # PriceUpdate | None
cache.get_price("AAPL")    # float | None   ← trade execution, valuation
cache.get_all()            # dict[str, PriceUpdate] (shallow copy) ← SSE, watchlist, chat context
cache.version              # int, bumped on every write ← SSE change detection
"AAPL" in cache; len(cache)
```

`None` means no price has arrived yet. Trades then fail with `NO_PRICE`, the API returns `"price": null`, and a held position is valued at `avg_cost` (PLAN §8).

### 2.2 Changing what is tracked: `MarketDataSource`

```python
await source.start(tickers)      # once, at startup
await source.add_ticker("PYPL")  # validates and prices immediately; may raise UnknownTickerError
await source.remove_ticker("PYPL")  # stops tracking AND drops it from the cache
source.get_tickers()             # list[str]
await source.stop()              # at shutdown; idempotent
```

### 2.3 `PriceUpdate`

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float                       # rounded to cents
    previous_price: float              # previous cached price (== price on first update)
    timestamp: float                   # Unix seconds
    session_start_price: float | None  # first price seen this process (PLAN §6)

    change: float                  # price - previous_price, 4 dp
    change_percent: float          # tick-to-tick %, 4 dp
    direction: str                 # "up" | "down" | "flat"
    session_change_percent: float  # vs session_start_price, 4 dp → watchlist "daily change %"

    def to_dict(self) -> dict      # exactly the SSE per-ticker object in PLAN §6
```

---

## 3. The abstract interface

`backend/app/market/interface.py`. The method set is unchanged from the existing code. The docstrings below tighten the contract, and `add_ticker` gains a validation duty.

```python
class MarketDataSource(ABC):
    """Produces prices for a set of tickers and writes them to a PriceCache.

    All methods are called from the asyncio event loop. Implementations must not
    block it; run blocking I/O with asyncio.to_thread.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin tracking `tickers` and start the background task.

        Must seed the cache before returning, so the first SSE event and the
        startup portfolio snapshot (PLAN §3 step 3) see prices. Called once.
        Must not raise on network failure: log it and let the loop retry.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Cancel the background task. Idempotent. No cache writes afterwards."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Start tracking an already-normalized ticker. No-op if already tracked.

        On return the ticker is either tracked AND has a price in the cache, or
        UnknownTickerError was raised and nothing changed. (Simulator: any
        well-formed symbol is accepted. Massive: symbols with no data are rejected.)
        """

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Stop tracking and remove the ticker from the cache. No-op if absent."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Currently tracked tickers (a copy)."""
```

Why `add_ticker` validates and prices synchronously:

- `POST /api/watchlist` has to reject unknown symbols in Massive mode (PLAN §6), and the source is the only component that can tell.
- A trade on an untracked ticker (PLAN §6) calls `add_ticker` first, then reads the price. If the price only appeared on the next 15-second poll, the trade would fail with `NO_PRICE`.

---

## 4. Ticker normalization and errors

New module `backend/app/market/tickers.py`. This module is the only place the ticker format rule lives.

```python
"""Ticker normalization and market-data errors."""

from __future__ import annotations

import re

TICKER_RE = re.compile(r"^[A-Z.]{1,10}$")


class InvalidTickerError(ValueError):
    """The symbol is malformed (fails TICKER_RE)."""


class UnknownTickerError(InvalidTickerError):
    """The symbol is well-formed but the data source has no data for it."""


def normalize_ticker(raw: str) -> str:
    """Upper-case and strip a user-supplied ticker; raise if it is malformed."""
    ticker = raw.strip().upper()
    if not TICKER_RE.fullmatch(ticker):
        raise InvalidTickerError(f"Invalid ticker: {raw!r}")
    return ticker
```

`UnknownTickerError` subclasses `InvalidTickerError`, so API code can map both to `400 {"code": "INVALID_TICKER"}` with a single `except InvalidTickerError`.

Sources assume their input is already normalized. Routes and the chat executor call `normalize_ticker` on every user- or LLM-supplied symbol before using it.

---

## 5. `PriceCache` (with session-start price)

`backend/app/market/cache.py`. The changes from today are the session-start map and the `timestamp is None` check.

```python
class PriceCache:
    """Thread-safe latest-price store. One writer (the source), many readers."""

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._session_start: dict[str, float] = {}
        self._lock = Lock()
        self._version = 0

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        with self._lock:
            price = round(price, 2)
            prev = self._prices.get(ticker)
            start = self._session_start.setdefault(ticker, price)
            update = PriceUpdate(
                ticker=ticker,
                price=price,
                previous_price=prev.price if prev else price,
                timestamp=time.time() if timestamp is None else timestamp,
                session_start_price=start,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)
            self._session_start.pop(ticker, None)   # re-adding starts a fresh session
            self._version += 1                      # so SSE pushes the shrunken set

    # get / get_all / get_price / version / __len__ / __contains__ unchanged
```

**Session-start semantics** (PLAN §6): the first price recorded for a ticker since the process started. In simulator mode that is the seed price. In Massive mode it is the first polled price. If a ticker is removed and added back, it starts a new session.

`PriceUpdate` additions in `models.py`:

```python
    session_start_price: float | None = None   # after `timestamp`; keeps existing constructors valid

    @property
    def session_change_percent(self) -> float:
        start = self.session_start_price
        if not start:
            return 0.0
        return round((self.price - start) / start * 100, 4)

    def to_dict(self) -> dict:
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

---

## 6. `MassiveDataSource`: two modes, chosen at startup

Research finding (`MASSIVE_API.md` §2): **the free Basic plan has no snapshot access**. It gets end-of-day bars only, at 5 calls/min. Paid plans get snapshots, 15-minute delayed on Starter and Developer and real-time on Advanced, and only Developer and above include `lastTrade`. So the source decides at startup which mode it is in:

| Mode | When | Poll | Endpoint | Effect in the UI |
|---|---|---|---|---|
| `live` | Snapshot probe succeeds (Starter+) | every `MASSIVE_POLL_INTERVAL` s (default 15) | `get_snapshot_all(tickers=…)`: one call for all tickers | Prices move every poll during market hours |
| `eod` | Probe returns 403 `NOT_AUTHORIZED` (Basic) | every hour | `get_grouped_daily_aggs(date)`: one call, whole market | Last close, static all day; a warning is logged once |

In `eod` mode the full grouped-daily result (~10k tickers, a few hundred KB) is kept in memory. Validating a new ticker is then a dictionary lookup with no API call, which matters with a 5 calls/min budget.

```python
"""Massive (Polygon.io) market data source."""

from __future__ import annotations

import asyncio
import logging
import time
from datetime import date, timedelta

from massive import RESTClient
from massive.exceptions import BadResponse
from massive.rest.models import SnapshotMarketType, TickerSnapshot

from .cache import PriceCache
from .interface import MarketDataSource
from .tickers import UnknownTickerError

logger = logging.getLogger(__name__)

PROBE_TICKER = "AAPL"
EOD_LOOKBACK_DAYS = 7


def snapshot_price(s: TickerSnapshot) -> float | None:
    """Best price available in a snapshot, whatever the plan or time of day."""
    if s.last_trade and s.last_trade.price:   # Developer+
        return s.last_trade.price
    if s.min and s.min.close:                 # Starter+: latest minute bar
        return s.min.close
    if s.day and s.day.close:                 # today's bar so far
        return s.day.close
    if s.prev_day and s.prev_day.close:       # 3:30-4:00am ET reset window
        return s.prev_day.close
    return None


def snapshot_timestamp(s: TickerSnapshot) -> float:
    return s.updated / 1e9 if s.updated else time.time()   # `updated` is NANOseconds


class MassiveDataSource(MarketDataSource):
    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
        eod_refresh_interval: float = 3600.0,
        client: RESTClient | None = None,          # injectable for tests
    ) -> None:
        self._cache = price_cache
        self._poll_interval = poll_interval
        self._eod_refresh_interval = eod_refresh_interval
        self._client = client or RESTClient(api_key=api_key, retries=1)
        self._tickers: list[str] = []
        self._mode: str = "live"
        self._eod: dict[str, tuple[float, float]] = {}   # ticker -> (close, ts) in eod mode
        self._task: asyncio.Task | None = None

    # --- lifecycle ---

    async def start(self, tickers: list[str]) -> None:
        self._tickers = list(dict.fromkeys(tickers))
        self._mode = await asyncio.to_thread(self._detect_mode)
        await self._refresh()                          # seed the cache before returning
        self._task = asyncio.create_task(self._loop(), name="massive-poller")
        logger.info("Massive source started in %s mode, %d tickers", self._mode, len(self._tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker: str) -> None:
        if ticker in self._tickers:
            return
        quote = await asyncio.to_thread(self._fetch_one, ticker)
        if quote is None:
            raise UnknownTickerError(f"No market data for {ticker}")
        self._tickers.append(ticker)
        self._cache.update(ticker, *quote)

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- background loop ---

    async def _loop(self) -> None:
        interval = self._poll_interval if self._mode == "live" else self._eod_refresh_interval
        while True:
            await asyncio.sleep(interval)
            await self._refresh()

    async def _refresh(self) -> None:
        tickers = list(self._tickers)                  # copy: add/remove may run meanwhile
        if not tickers:
            return
        try:
            if self._mode == "live":
                snaps = await asyncio.to_thread(
                    self._client.get_snapshot_all, SnapshotMarketType.STOCKS, tickers
                )
                for s in snaps:
                    price = snapshot_price(s)
                    if price is not None and s.ticker in self._tickers:
                        self._cache.update(s.ticker, price, snapshot_timestamp(s))
            else:
                self._eod = await asyncio.to_thread(self._fetch_grouped_daily)
                for t in tickers:
                    if t in self._eod and t in self._tickers:
                        self._cache.update(t, *self._eod[t])
        except Exception:
            logger.exception("Massive refresh failed; will retry")   # 401/403/429/network

    # --- blocking helpers (run in a thread) ---

    def _detect_mode(self) -> str:
        try:
            self._client.get_snapshot_ticker(SnapshotMarketType.STOCKS, PROBE_TICKER)
            return "live"
        except BadResponse as e:
            if "NOT_AUTHORIZED" in str(e):
                logger.warning(
                    "Massive plan has no snapshot access (free tier): using end-of-day "
                    "prices, which will not move during the day"
                )
                return "eod"
            logger.error("Massive probe failed (%s); assuming live mode", e)
            return "live"
        except Exception as e:                         # network down at boot
            logger.error("Massive probe failed (%s); assuming live mode", e)
            return "live"

    def _fetch_one(self, ticker: str) -> tuple[float, float] | None:
        if self._mode == "eod":
            if not self._eod:
                self._eod = self._fetch_grouped_daily()
            return self._eod.get(ticker)
        try:
            s = self._client.get_snapshot_ticker(SnapshotMarketType.STOCKS, ticker)
        except BadResponse:
            return None
        if not isinstance(s, TickerSnapshot):          # client returns [] when body is empty
            return None
        price = snapshot_price(s)
        return None if price is None else (price, snapshot_timestamp(s))

    def _fetch_grouped_daily(self) -> dict[str, tuple[float, float]]:
        day = date.today()
        for _ in range(EOD_LOOKBACK_DAYS):             # skip weekends / holidays / today
            day -= timedelta(days=1)
            bars = self._client.get_grouped_daily_aggs(day.isoformat(), adjusted=True)
            if bars:
                return {
                    b.ticker: (b.close, b.timestamp / 1e3)   # bar timestamps are ms
                    for b in bars
                    if b.ticker and b.close
                }
        return {}
```

Notes:

- **Blocking I/O.** The `massive` client is synchronous, so every call goes through `asyncio.to_thread`. The ticker list is copied before the thread starts, and results for tickers removed in the meantime are dropped (`s.ticker in self._tickers`).
- **Retries.** `retries=1` stops urllib3 from spending the free tier's 5 calls/min retrying 429s. The loop already retries on the next tick.
- **Errors never escape the loop.** A bad key, rate limiting or a network outage is logged, and the cache keeps its last prices. The SSE stream keeps working, because the version counter simply stops changing.
- **Off-hours.** In `live` mode the snapshot price stops moving when the market closes. PLAN §6 accepts this, and the README must say so. Starter and Developer prices are 15 minutes delayed even during market hours.
- **Grouped daily on a trading day.** Before about 4 pm ET today's bar does not exist, so the loop's first step back (yesterday) is the right answer. After the close, "yesterday" is one session stale until the next hourly refresh rolls the date. This is acceptable for an EOD fallback.

---

## 7. Factory

`backend/app/market/factory.py`. The change is reading `MASSIVE_POLL_INTERVAL` (PLAN §5) and taking `env` as a parameter, which makes testing easier.

```python
MIN_POLL_INTERVAL = 2.0
DEFAULT_POLL_INTERVAL = 15.0


def create_market_data_source(
    price_cache: PriceCache, env: Mapping[str, str] = os.environ
) -> MarketDataSource:
    """MASSIVE_API_KEY non-empty → MassiveDataSource, else SimulatorDataSource. Unstarted."""
    api_key = env.get("MASSIVE_API_KEY", "").strip()
    if not api_key:
        logger.info("Market data source: simulator")
        return SimulatorDataSource(price_cache=price_cache)

    interval = _poll_interval(env.get("MASSIVE_POLL_INTERVAL", ""))
    logger.info("Market data source: Massive (poll every %.0fs)", interval)
    return MassiveDataSource(api_key=api_key, price_cache=price_cache, poll_interval=interval)


def _poll_interval(raw: str) -> float:
    try:
        value = float(raw) if raw.strip() else DEFAULT_POLL_INTERVAL
    except ValueError:
        logger.warning("Bad MASSIVE_POLL_INTERVAL=%r; using %.0fs", raw, DEFAULT_POLL_INTERVAL)
        return DEFAULT_POLL_INTERVAL
    return max(value, MIN_POLL_INTERVAL)
```

---

## 8. How the application uses it

### 8.1 Lifespan (PLAN §3)

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()                                                  # 1. schema + seed
    cache = PriceCache()
    source = create_market_data_source(cache)
    tracked = db.watchlist_tickers() | db.position_tickers()   # watchlist ∪ positions
    await source.start(sorted(tracked))                        # 2. seeds cache before returning
    app.state.price_cache, app.state.market = cache, source
    record_snapshot(cache)                                     # 3. sees real prices
    snap_task = asyncio.create_task(snapshot_loop(cache))      # 4. every 30 s
    yield
    snap_task.cancel()
    await source.stop()

app.include_router(create_stream_router(cache))   # see §9 item 7 for the router-object note
```

### 8.2 Who calls `add_ticker` / `remove_ticker`

The rule is that the source tracks **watchlist ∪ open positions**.

| Event | Call, in this order |
|---|---|
| `POST /api/watchlist {ticker}` | `t = normalize_ticker(raw)` → `await market.add_ticker(t)` (may raise `UnknownTickerError` → 400 `INVALID_TICKER`) → insert row (idempotent) |
| `DELETE /api/watchlist/{t}` | delete row (404 `NOT_IN_WATCHLIST` if absent) → `if t not held: await market.remove_ticker(t)` |
| Trade (manual or chat) on ticker `t` | `t = normalize_ticker(raw)` → `if t not in market.get_tickers(): await market.add_ticker(t)` → `price = cache.get_price(t)` (`None` → `NO_PRICE`) → execute in one DB transaction |
| Sell that closes a position | after commit: `if t not in watchlist: await market.remove_ticker(t)` |
| Chat watchlist change | same as the two watchlist rows above, per action |

Calling `add_ticker` before the DB write means an invalid symbol never reaches the database. If the DB write then fails, the ticker stays tracked until the next restart, which is harmless.

### 8.3 Consumers

| Consumer | Reads |
|---|---|
| SSE `/api/stream/prices` | `cache.version`, `cache.get_all()` → `{t: u.to_dict()}` every 500 ms if the version changed |
| `GET /api/watchlist` | `cache.get(t)` per row → `price`, `session_change_percent` (or `null`) |
| `GET /api/portfolio`, snapshots | `cache.get_price(t)`, falling back to `avg_cost` when `None` |
| Trade execution | `cache.get_price(t)` as the fill price |
| Chat context | `cache.get_all()` for watchlist prices and position valuation |

---

## 9. Changes required to the existing code

| # | File | Change | Why |
|---|---|---|---|
| 1 | `massive_client.py` | Replace `snap.last_trade.price` / `snap.last_trade.timestamp / 1000` with `snapshot_price()` / `snapshot_timestamp()` (§6) | **Bug.** `LastTrade` has no `timestamp` attribute, so every snapshot raises `AttributeError` and is skipped: Massive mode never prices anything. Timestamps are also ns, not ms. `last_trade` is `None` on Starter |
| 2 | `massive_client.py` | Add `live`/`eod` mode detection and the grouped-daily path (§6) | Free keys get 403 on snapshots |
| 3 | `massive_client.py`, `simulator.py` | `add_ticker` validates and seeds the cache before returning, and raises `UnknownTickerError` when there is no data (§3) | PLAN §6 validation; avoids `NO_PRICE` on the first trade of a new ticker |
| 4 | new `tickers.py`, `__init__.py` | `normalize_ticker`, `InvalidTickerError`, `UnknownTickerError`, exported (§4) | One home for the `^[A-Z.]{1,10}$` rule |
| 5 | `models.py`, `cache.py` | `session_start_price`, `session_change_percent` in `to_dict`, session map cleared on `remove`, `remove` bumps the version, `timestamp is None` check (§5) | PLAN §6 / §13 |
| 6 | `factory.py` | Read `MASSIVE_POLL_INTERVAL`, minimum 2 s (§7) | PLAN §5 / §13 |
| 7 | `stream.py` | Create the `APIRouter` **inside** `create_stream_router` instead of at module level | Today the module-level router collects a duplicate `/prices` route every time the factory is called (e.g. once per test app) |
| 8 | `tests/market/test_massive.py` | Build snapshots with real `TickerSnapshot.from_dict({...})` payloads (from `MASSIVE_API.md` §4.1) instead of `MagicMock`; cover Starter (no `lastTrade`), the 403 → `eod` switch, unknown ticker, ns timestamps | A `MagicMock` accepts `.timestamp`, so it hid bug 1 |

### Corresponding PLAN.md updates (for the owner to approve)

- §5 / §6: "Free tier (5 calls/min): poll every 15 seconds" is not possible, because Basic has no snapshot endpoint. Replace it with: *free keys get end-of-day prices (static during the day); live polling needs Starter ($29, 15-min delayed) or Advanced ($199, real-time)*.
- §6 Ticker Validation, Massive mode: "tickers that return no snapshot" → "tickers with no snapshot (paid) or no previous-day bar (free)".

---

## 10. Testing the interface

- **Contract test, run against both sources.** Parametrize over `[SimulatorDataSource, MassiveDataSource(client=FakeClient())]`: `start` seeds the cache; `add_ticker` puts a price in the cache before returning; `remove_ticker` removes it from both `get_tickers()` and the cache; `stop` is idempotent.
- **`FakeClient`** returns `TickerSnapshot.from_dict(...)` / `GroupedDailyAgg.from_dict(...)` built from the documented JSON. It can raise `BadResponse('{"status":"NOT_AUTHORIZED"}')` to drive `eod` mode.
- **Cache:** session-start is fixed at the first update, survives later updates, and resets after `remove`. `session_change_percent` math. `version` bumps on update and on remove.
- **Factory:** empty or whitespace key → simulator; `MASSIVE_POLL_INTERVAL` parsing (missing, bad, below the minimum).
