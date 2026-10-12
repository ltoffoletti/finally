# Massive API Reference (formerly Polygon.io)

Reference for the parts of the Massive REST API that FinAlly uses to get **real-time** and **end-of-day** US stock prices for several tickers at once.

Researched 2026-10-11 from the official docs (`https://massive.com/docs/rest/stocks/...`, index at `https://massive.com/docs/llms.txt`), the pricing page (`https://massive.com/pricing`), and the source of the `massive` Python package **v2.2.0** (the version pinned in `backend/uv.lock`). Where the docs and the package differ, the package source wins, because that is what our code runs.

> Supersedes `planning/archive/MASSIVE_API.md`. That earlier version got three things wrong that matter to our code: timestamp units, Python attribute names, and which plans include snapshots. See §9.

---

## 1. Basics

| Item | Value |
|---|---|
| Base URL | `https://api.massive.com` (legacy `https://api.polygon.io` still answers) |
| Python package | `massive` (`uv add massive`), import `from massive import RESTClient` |
| Auth | `Authorization: Bearer <KEY>` header (the client sets it), or `?apiKey=<KEY>` query param for curl |
| Env var read by the client | `MASSIVE_API_KEY`. `RESTClient()` with no argument raises `massive.exceptions.AuthError` if it is unset |
| Ticker symbols | **Case-sensitive**, upper case (`AAPL`, `BRK.B`) |
| Sync/async | The `massive` REST client is **synchronous** (urllib3). Call it from asyncio with `asyncio.to_thread(...)` |

```python
from massive import RESTClient

client = RESTClient()                         # reads MASSIVE_API_KEY
client = RESTClient(api_key="...",            # or explicit
                    retries=3,                # urllib3 retries on 413/429/499/500/502/503/504
                    connect_timeout=10.0,
                    read_timeout=10.0,
                    trace=False)              # True logs URLs (with the key redacted)
```

---

## 2. Plans: what each tier can call

From `massive.com/pricing` and the per-endpoint "Plan Access" tables (individual, non-professional plans):

| Plan | Price/mo | Rate limit | Data recency | Snapshots | Last trade | Last quote | Aggregates, prev close, grouped daily |
|---|---|---|---|---|---|---|---|
| **Basic (free)** | $0 | **5 calls/min** | **End of day** | **No** | No | No | Yes (EOD) |
| Starter | $29 | Unlimited | 15-min delayed | Yes | **No** | No | Yes (15-min delayed) |
| Developer | $79 | Unlimited | 15-min delayed | Yes | Yes | No | Yes (15-min delayed) |
| Advanced | $199 | Unlimited | Real-time | Yes | Yes | Yes | Yes (real-time) |

"Unlimited" plans should still stay well under ~100 requests/second.

**What this means for FinAlly:**

1. **A free key cannot poll live prices at all.** Snapshot endpoints return `403 NOT_AUTHORIZED` on Basic. On Basic the only useful data is the previous trading day's bars (§5). PLAN.md §5/§6 assume "free tier: poll snapshots every 15 s", which does not work.
2. **On Starter, snapshots have no `lastTrade`.** The response omits fields the plan does not cover, so the price must come from the minute bar (`min.c`) or day bar (`day.c`), not from `lastTrade.p`.
3. "Real-time" requires Advanced. Starter and Developer are 15 minutes behind.

---

## 3. Timestamp units (easy to get wrong)

| Where | Unit | Example |
|---|---|---|
| Snapshot `updated` | **nanoseconds** | `1605192894630916600` |
| Snapshot `lastTrade.t`, `lastQuote.t` | **nanoseconds** | `1605192894630916600` |
| Last-trade / last-quote endpoints `t`, `y`, `f` | **nanoseconds** | |
| Unified snapshot `last_trade.sip_timestamp`, `last_updated`, `fmv_last_updated` | **nanoseconds** | |
| Bar `t` (snapshot `min.t`, aggregates, prev close, grouped daily) | **milliseconds** | `1684428600000` |
| Market status `serverTime` | RFC 3339 string | `2020-11-10T17:37:37-05:00` |

Convert to the Unix seconds the price cache uses:

```python
ts_seconds = snap.updated / 1e9          # nanoseconds
ts_seconds = bar.timestamp / 1e3         # milliseconds
```

---

## 4. Real-time / delayed prices for many tickers

### 4.1 Full Market Snapshot, filtered (primary endpoint)

One call returns the latest state for every ticker you list. It needs Starter or above.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,NVDA
```

| Query param | Notes |
|---|---|
| `tickers` | Comma-separated, case-sensitive. **Omit it and you get the whole market** (~10k rows), so always pass it |
| `include_otc` | Default `false` |

The docs do not say what happens to unknown tickers in the filter. Assume they are simply missing from `tickers[]`, and compare the symbols you asked for with the symbols you got back.

**Raw response** (from the docs, one element):

```json
{
  "status": "OK",
  "count": 1,
  "tickers": [
    {
      "ticker": "BCAT",
      "todaysChange": -0.124,
      "todaysChangePerc": -0.601,
      "updated": 1605192894630916600,
      "day":     {"o": 20.64, "h": 20.64, "l": 20.506, "c": 20.506, "v": 37216, "vw": 20.616},
      "min":     {"av": 37216, "o": 20.506, "h": 20.506, "l": 20.506, "c": 20.506,
                  "v": 5000, "vw": 20.5105, "n": 1, "t": 1684428600000},
      "prevDay": {"o": 20.79, "h": 21, "l": 20.5, "c": 20.63, "v": 292738, "vw": 20.6939},
      "lastTrade": {"p": 20.506, "s": 2416, "x": 4, "c": [14, 41],
                    "i": "71675577320245", "t": 1605192894630916600},
      "lastQuote": {"p": 20.5, "s": 13, "P": 20.6, "S": 22, "t": 1605192959994246100}
    }
  ]
}
```

`lastTrade` is present only on Developer and above, and `lastQuote` only on Advanced. In `lastQuote`, lower-case `p`/`s` are the **bid** and upper-case `P`/`S` are the **ask**.

**Python client:**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()
snaps = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,    # or "stocks"
    tickers=["AAPL", "MSFT", "NVDA"],         # list is joined with commas
)                                             # -> list[TickerSnapshot]

for s in snaps:
    print(s.ticker, s.updated, s.todays_change_percent)
    print("  day   ", s.day.open, s.day.close if s.day else None)
    print("  min   ", s.min.close if s.min else None)
    print("  prev  ", s.prev_day.close if s.prev_day else None)
    if s.last_trade:                          # None below Developer plan
        print("  trade ", s.last_trade.price, s.last_trade.sip_timestamp)
```

**JSON key → Python attribute** (from `massive/rest/models`):

| JSON | `TickerSnapshot` attribute | Type |
|---|---|---|
| `ticker` | `ticker` | `str` |
| `updated` | `updated` | `int`, ns |
| `todaysChange` / `todaysChangePerc` | `todays_change` / `todays_change_percent` | `float` |
| `day`, `prevDay` | `day`, `prev_day` | `Agg` (`open, high, low, close, volume, vwap, timestamp, transactions`) |
| `min` | `min` | `MinuteSnapshot` (`open, high, low, close, volume, vwap, accumulated_volume, timestamp`) |
| `lastTrade` | `last_trade` | `LastTrade` (`price, size, exchange, conditions, id, sip_timestamp, participant_timestamp, trf_timestamp`) |
| `lastQuote` | `last_quote` | `LastQuote` (`bid_price, bid_size, ask_price, ask_size, sip_timestamp`) |

> **`LastTrade` has no `timestamp` attribute.** `lastTrade.t` becomes `last_trade.sip_timestamp`, in nanoseconds. Any field missing from the JSON becomes `None` (e.g. `last_trade is None` on Starter).

**Choosing the price robustly across plans and times of day:**

```python
def snapshot_price(s) -> float | None:
    """Best available price from a TickerSnapshot, whatever the plan."""
    if s.last_trade and s.last_trade.price:       # Developer+
        return s.last_trade.price
    if s.min and s.min.close:                     # latest minute bar (Starter+)
        return s.min.close
    if s.day and s.day.close:                     # today's bar so far
        return s.day.close
    if s.prev_day and s.prev_day.close:           # pre-market / just after the 3:30am reset
        return s.prev_day.close
    return None

def snapshot_timestamp(s) -> float:
    import time
    return s.updated / 1e9 if s.updated else time.time()
```

**Daily reset:** snapshot data is cleared at 3:30 AM ET and refills as exchanges report, starting around 4:00 AM ET. Between those times `day` and `min` can be empty or zero, so fall back to `prev_day.close`.

**curl:**

```bash
curl -s "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT&apiKey=$MASSIVE_API_KEY" | jq '.tickers[] | {ticker, updated, day: .day.c, min: .min.c}'
```

### 4.2 Unified Snapshot (v3): alternative multi-ticker call

```
GET /v3/snapshot?ticker.any_of=AAPL,MSFT,NVDA&limit=250
```

- Takes up to **250** tickers per call. `limit` defaults to **10** (max 250), so always set it. Pagination uses `next_url`.
- Unknown tickers come back as **error rows** (`{"ticker": "XXXX", "error": "NOT_FOUND", "message": "Ticker not found."}`). That makes it convenient for validation.
- Each row has a `session` object (`price, change, change_percent, open, high, low, close, previous_close, volume`, plus early/regular/late-trading changes) and a `market_status` (`open | closed | early_trading | late_trading`).
- Same plan rules as §4.1: Starter and above, and `last_trade`/`last_quote` only where the plan includes them.

```python
for s in client.list_universal_snapshots(
    type="stocks",
    ticker_any_of=["AAPL", "MSFT", "NOPE"],
    limit=250,
):                                            # Iterator[UniversalSnapshot]; follows next_url
    if s.error:
        print(s.ticker, "->", s.error, s.message)
        continue
    print(s.ticker, s.market_status, s.session.price, s.session.previous_close)
```

We use §4.1 for polling because it is the more established endpoint and we already depend on it. §4.2 is a reasonable choice if we ever need explicit not-found errors.

### 4.3 Single-ticker snapshot

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}
```

```python
s = client.get_snapshot_ticker(SnapshotMarketType.STOCKS, "AAPL")   # -> TickerSnapshot
```

Same object shape as one element of §4.1. Useful for validating and pricing a single newly added ticker straight away, without waiting for the next poll. The docs do not specify the not-found behaviour, so treat both outcomes as "unknown ticker": a `BadResponse` (non-200) and an empty result (the client returns `[]` when the `ticker` key is missing).

### 4.4 Last trade / last quote (per ticker)

```
GET /v2/last/trade/{ticker}     # Developer+  -> results.p, results.s, results.t (ns)
GET /v2/last/nbbo/{ticker}      # Advanced    -> results.p (bid), results.P (ask), results.t (ns)
```

```python
t = client.get_last_trade("AAPL")    # LastTrade: .price, .size, .sip_timestamp (ns)
q = client.get_last_quote("AAPL")    # LastQuote: .bid_price, .ask_price, .sip_timestamp (ns)
```

These take one ticker per call, so they are not suited to polling a watchlist. The snapshot already contains the same data.

---

## 5. End-of-day prices

All of these work on **every plan, including the free Basic tier**. On Basic they are the only way to get prices.

### 5.1 Grouped Daily ("Daily Market Summary"): all tickers, one call

```
GET /v2/aggs/grouped/locale/us/market/stocks/{date}?adjusted=true
```

This returns one daily bar for **every** US stock on `date` (YYYY-MM-DD) in a single call. It is the most efficient way to get EOD prices for a watchlist: one call, then filter.

```json
{
  "status": "OK", "adjusted": true, "queryCount": 3, "resultsCount": 3,
  "results": [
    {"T": "AAPL", "o": 324.66, "h": 326.2, "l": 322.3, "c": 325.12,
     "v": 26122646, "vw": 324.9, "n": 412345, "t": 1673298000000}
  ]
}
```

```python
bars = client.get_grouped_daily_aggs("2026-10-09", adjusted=True)   # list[GroupedDailyAgg]
wanted = {"AAPL", "MSFT", "NVDA"}
eod = {b.ticker: b.close for b in bars if b.ticker in wanted}
# GroupedDailyAgg: ticker, open, high, low, close, volume, vwap, timestamp (ms), transactions
```

Weekends, holidays and today before the close have no bars. The response then has no `results` key, and the client returns `[]` rather than raising. To find the latest completed session, step back day by day:

```python
from datetime import date, timedelta

def latest_eod(client, tickers: set[str], max_lookback: int = 7) -> dict[str, float]:
    day = date.today()
    for _ in range(max_lookback):
        day -= timedelta(days=1)
        bars = client.get_grouped_daily_aggs(day.isoformat())
        if bars:
            return {b.ticker: b.close for b in bars if b.ticker in tickers}
    return {}
```

On the free tier each step back costs one of the 5 calls per minute. Calling `get_market_status()` first (§6), or caching the result, avoids wasting them.

### 5.2 Previous Day Bar (per ticker)

```
GET /v2/aggs/ticker/{ticker}/prev
```

```json
{"ticker": "AAPL", "status": "OK", "resultsCount": 1, "adjusted": true,
 "results": [{"T": "AAPL", "o": 115.55, "h": 117.59, "l": 114.13, "c": 115.97,
              "v": 131704427, "vw": 116.3058, "t": 1605042000000}]}
```

```python
bars = client.get_previous_close_agg("AAPL")      # list[PreviousCloseAgg]
prev_close = bars[0].close
# PreviousCloseAgg: ticker, open, high, low, close, volume, vwap, timestamp (ms)
```

It always returns the last completed session, so no date logic is needed. The cost is one call per ticker: on the free tier 10 tickers take 2 minutes. Prefer §5.1 for a watchlist.

### 5.3 Daily Ticker Summary ("open/close") for a given date

```
GET /v1/open-close/{ticker}/{date}
```

```json
{"status": "OK", "symbol": "AAPL", "from": "2023-01-09",
 "open": 324.66, "high": 326.2, "low": 322.3, "close": 325.12,
 "preMarket": 324.5, "afterHours": 322.1, "volume": 26122646}
```

```python
oc = client.get_daily_open_close_agg("AAPL", "2023-01-09")
print(oc.open, oc.close, oc.pre_market, oc.after_hours)
```

Use this when you need pre-market and after-hours prices for one ticker on a specific day.

### 5.4 Custom Bars (aggregates over a range)

```
GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}
    ?adjusted=true&sort=asc&limit=5000
```

`timespan` is one of `second | minute | hour | day | week | month | quarter | year`. `from` and `to` are YYYY-MM-DD dates or ms timestamps. `limit` defaults to 5000, max 50000. History goes back 2 years on Basic, 5 on Starter, 10 on Developer, and all of it on Advanced.

```python
for bar in client.list_aggs("AAPL", 1, "day", "2026-09-01", "2026-10-09", limit=50000):
    print(bar.timestamp, bar.open, bar.high, bar.low, bar.close, bar.volume)   # Agg; ms timestamp

bars = client.get_aggs("AAPL", 5, "minute", "2026-10-09", "2026-10-09")    # list[Agg], no pagination
```

FinAlly does not need history (PLAN §10: charts accumulate from SSE). This is listed only as a possible way to backfill charts later.

---

## 6. Market status

```
GET /v1/marketstatus/now        # all plans, real-time
```

```json
{"market": "extended-hours", "earlyHours": false, "afterHours": true,
 "serverTime": "2020-11-10T17:37:37-05:00",
 "exchanges": {"nasdaq": "extended-hours", "nyse": "extended-hours", "otc": "closed"},
 "currencies": {"fx": "open", "crypto": "open"}}
```

```python
st = client.get_market_status()
print(st.market, st.early_hours, st.after_hours, st.server_time)
```

`market` is `open`, `closed` or `extended-hours`. We can use it to log that prices are frozen outside trading hours, or to slow polling down.

---

## 7. Errors

| Situation | What happens in the `massive` client |
|---|---|
| No key at construction | `massive.exceptions.AuthError` |
| HTTP 401 (bad key) | `massive.exceptions.BadResponse` with the JSON body as the message |
| HTTP 403 (`NOT_AUTHORIZED`, endpoint not in plan) | `BadResponse`. **This is what snapshots return on the free tier** |
| Any other non-200 (e.g. 404) | `BadResponse` |
| 200 whose body lacks the result key (e.g. grouped daily on a holiday) | Returns `[]`, no exception |
| HTTP 429 (rate limit) and 413/499/5xx | Retried automatically by urllib3 (`retries=3`, backoff 0.1 → 0.2 → 0.4 s), then `BadResponse`/urllib3 error |
| Network timeout | urllib3 exception (`MaxRetryError`, etc.) |

On the free tier, automatic retries on 429 spend more of the 5 calls/min budget. With a 5/min budget, construct the client with `retries=0` or `1`.

Error bodies look like:

```json
{"status": "NOT_AUTHORIZED", "request_id": "…", "message": "You are not entitled to this data. Please upgrade your plan at https://massive.com/pricing"}
```

To tell plan errors apart from transient ones, check the status text in the message: `"NOT_AUTHORIZED" in str(e)`.

---

## 8. Recipes

### 8.1 Poll many tickers (paid plans) from asyncio

```python
import asyncio, logging, time
from massive import RESTClient
from massive.exceptions import BadResponse
from massive.rest.models import SnapshotMarketType

log = logging.getLogger(__name__)

async def poll_forever(client: RESTClient, get_tickers, on_price, interval: float = 15.0):
    while True:
        tickers = get_tickers()
        if tickers:
            try:
                snaps = await asyncio.to_thread(
                    client.get_snapshot_all, SnapshotMarketType.STOCKS, tickers
                )
                for s in snaps:
                    price = snapshot_price(s)                 # §4.1 helper
                    if price is not None:
                        on_price(s.ticker, price, snapshot_timestamp(s))
            except BadResponse as e:
                log.error("Massive poll failed: %s", e)       # keep looping
        await asyncio.sleep(interval)
```

### 8.2 One EOD price per ticker (free plan)

```python
eod = await asyncio.to_thread(latest_eod, client, {"AAPL", "MSFT", "NVDA"})   # §5.1
for ticker, close in eod.items():
    on_price(ticker, close, time.time())
```

### 8.3 Detect the plan at startup

```python
def snapshots_available(client: RESTClient) -> bool:
    try:
        client.get_snapshot_ticker(SnapshotMarketType.STOCKS, "AAPL")
        return True
    except BadResponse as e:
        if "NOT_AUTHORIZED" in str(e):
            return False
        raise
```

---

## 9. Corrections to the archived doc and current code

These are facts from this research that the existing code in `backend/app/market/massive_client.py` gets wrong. `MARKET_INTERFACE.md` turns them into required changes.

| # | Archived doc / current code | Reality |
|---|---|---|
| 1 | `snap.last_trade.timestamp` | No such attribute: it is `snap.last_trade.sip_timestamp`. The current poller raises `AttributeError` for every ticker, catches it, and skips the ticker, **so Massive mode never produces a price**. The unit tests pass because they build snapshots with `MagicMock`, which accepts any attribute |
| 2 | "Timestamps are Unix milliseconds" (`/ 1000.0`) | Trade, quote and `updated` timestamps are **nanoseconds** (`/ 1e9`). Only bar timestamps are ms |
| 3 | `snap.last_trade.price` always present | `last_trade` is `None` on Starter and Basic. Use the fallback chain in §4.1 |
| 4 | "Free tier: poll snapshot every 15 s" | Basic has **no snapshot access** (403). Free keys only get end-of-day data (§5) |
| 5 | `snap.day.previous_close`, `day.change_percent` | Not fields of `Agg`. Previous close is `snap.prev_day.close` and day change % is `snap.todays_change_percent` |
| 6 | `last_quote.bid_price` in JSON as `bid_price` | The JSON keys are `p` (bid) and `P` (ask). The Python attributes are `bid_price` and `ask_price` |
