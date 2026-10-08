# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change (unrealized P&L % vs. average cost)
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background tasks: market data, snapshots       │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Single Process

The price cache, market data source and snapshot task all live in memory, so uvicorn **must run with exactly one worker**. Never add `--workers N`.

### Startup Sequence

All initialization happens eagerly in the FastAPI `lifespan` handler, in this order:

1. Initialize the database: `CREATE TABLE IF NOT EXISTS` for every table, then seed default data if `users_profile` is empty
2. Create the market data source and start it with the tracked tickers (watchlist ∪ open positions)
3. Record an initial portfolio snapshot
4. Start the 30-second snapshot background task

On shutdown, stop the snapshot task and the market data source.

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   └── db/                   # Schema definitions, seed data, init logic
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; *.db files are gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

There is no root `docker-compose.yml`: the start/stop scripts are the single supported way to run the app. (`test/docker-compose.test.yml` is separate and only for E2E tests.)

`.gitignore` must include `db/*.db`, `db/*.db-wal` and `db/*.db-shm`.

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend initializes the database at startup (see §3 Startup Sequence) — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# OpenRouter API key for LLM chat functionality.
# If not set, the backend falls back to mock LLM mode (see below).
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Massive polling interval in seconds (default 15 = free tier; paid tiers can use 2-15)
MASSIVE_POLL_INTERVAL=15

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data, polling every `MASSIVE_POLL_INTERVAL` seconds
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- If `OPENROUTER_API_KEY` is absent or empty → backend behaves as if `LLM_MOCK=true` and logs a warning at startup, so the app works without a key
- In Docker, environment variables are passed **only** via `docker run --env-file .env`; the container never reads a `.env` file itself
- For local development outside Docker (`uv run`), the backend may load `../.env` with `python-dotenv`

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Tracked Tickers

The market data source tracks the **union of watchlist tickers and tickers with open positions**. Removing a ticker from the watchlist stops tracking it only if no position is held. A trade on an untracked ticker starts tracking it (without adding it to the watchlist), so held positions can always be valued and sold.

### Ticker Validation

- Tickers are normalized to uppercase and must match `^[A-Z.]{1,10}$`
- **Simulator mode**: any well-formed symbol is accepted. Tickers without seed data start at a random price between $50 and $300 and use the default GBM params (σ = 0.25, μ = 0.05)
- **Massive mode**: tickers that return no snapshot are rejected

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all tracked tickers every `MASSIVE_POLL_INTERVAL` seconds (default 15)
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- With 15s polling, prices flash and sparklines grow only every 15s, and prices are stale outside market hours. This is accepted; the README should say so.

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds, for each ticker, the latest price, previous price, timestamp, and the **session-start price** (the first price recorded for that ticker since the process started: the seed price in simulator mode, the first polled price in Massive mode)
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The first message is `retry: 1000` (reconnect after 1s); after that, client reconnection is automatic (EventSource built-in retry)
- Every ~500ms, if any price has changed (cache version bumped), the server sends **one event containing all tracked tickers**, as a JSON object keyed by symbol:

```json
{
  "AAPL": {
    "ticker": "AAPL",
    "price": 190.52,
    "previous_price": 190.41,
    "timestamp": 1791369600.123,
    "change": 0.11,
    "change_percent": 0.0578,
    "direction": "up",
    "session_change_percent": 0.2737
  },
  "GOOGL": { "...": "..." }
}
```

- `timestamp` is Unix seconds (float)
- `change` / `change_percent` are tick-to-tick (vs. `previous_price`)
- `direction` is `"up"`, `"down"` or `"flat"`; the frontend drives the price flash directly from it
- `session_change_percent` is the change vs. the session-start price; this is what the watchlist shows as "daily change %"

---

## 7. Database

### SQLite, Initialized at Startup

The backend initializes the database once, at startup (see §3 Startup Sequence). If the file doesn't exist or tables are missing, it creates the schema with `CREATE TABLE IF NOT EXISTS` and seeds default data when `users_profile` is empty. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables except `users_profile` include a `user_id` column defaulting to `"default"`. `users_profile` uses its `id` column as the user key. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded once at startup, every 30 seconds by a background task, and immediately after each trade execution (manual or via chat).
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — trades executed, watchlist changes made; null for user messages)
- `created_at` TEXT (ISO timestamp)

All timestamps stored in the database are ISO 8601 UTC strings with a `Z` suffix (e.g. `2026-10-07T14:30:00.123Z`).

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates (see §6 for the payload) |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{ticker}` |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

### Chat
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/chat` | Recent chat history (last 50 messages, oldest first), so the panel survives a page refresh |
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment): `{"status": "ok"}` |

### Conventions

- All REST timestamps are ISO 8601 UTC with a `Z` suffix. (The SSE stream is the one exception: it uses Unix seconds.)
- Money values are rounded to cents; quantities to 4 decimal places.
- Prices are `null` when no price has been received yet for a ticker.

### Response Shapes

This is the contract between the frontend and backend agents, who work in parallel.

**`GET /api/portfolio`**
```json
{
  "cash": 8094.50,
  "total_value": 10012.30,
  "positions": [
    {
      "ticker": "AAPL",
      "quantity": 10.0,
      "avg_cost": 190.55,
      "current_price": 191.78,
      "market_value": 1917.80,
      "unrealized_pnl": 12.30,
      "unrealized_pnl_percent": 0.6455
    }
  ]
}
```
`unrealized_pnl_percent` = `(current_price - avg_cost) / avg_cost × 100`. This is the positions table's "% change". If a price is momentarily unavailable, the position is valued at `avg_cost`.

**`POST /api/portfolio/trade`** — request `{"ticker": "AAPL", "quantity": 10, "side": "buy"}`; response:
```json
{
  "trade": {"ticker": "AAPL", "side": "buy", "quantity": 10.0, "price": 190.55, "executed_at": "2026-10-07T14:30:00.123Z"},
  "portfolio": { "...": "same shape as GET /api/portfolio" }
}
```

**`GET /api/portfolio/history`** — all snapshots, oldest first:
```json
[{"recorded_at": "2026-10-07T14:30:00.000Z", "total_value": 10000.00}]
```

**`GET /api/watchlist`**
```json
[{"ticker": "AAPL", "price": 190.52, "session_change_percent": 0.2737, "added_at": "2026-10-07T14:00:00.000Z"}]
```

**`POST /api/watchlist`** — request `{"ticker": "PYPL"}`; returns the added entry (same shape as one `GET /api/watchlist` item). Adding a ticker that is already on the watchlist is idempotent and returns 200.

**`DELETE /api/watchlist/{ticker}`** — returns `{"ticker": "PYPL", "removed": true}`; 404 if the ticker is not on the watchlist.

**`GET /api/chat`**
```json
[{"id": "...", "role": "assistant", "content": "...", "actions": {"trades": [], "watchlist_changes": []}, "created_at": "..."}]
```

**`POST /api/chat`** — request `{"message": "Buy 5 NVDA"}`; response:
```json
{
  "message": "Done — bought 5 NVDA at $875.20.",
  "trades": [
    {"ticker": "NVDA", "side": "buy", "quantity": 5.0, "price": 875.20, "status": "executed", "error": null}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add", "status": "failed", "error": "Invalid ticker"}
  ]
}
```

### Errors

All errors use one shape:

```json
{"error": "Insufficient cash", "code": "INSUFFICIENT_CASH"}
```

| Status | When | Example codes |
|--------|------|---------------|
| 400 | Business-rule or input failure | `INSUFFICIENT_CASH`, `INSUFFICIENT_SHARES`, `INVALID_QUANTITY`, `INVALID_TICKER`, `NO_PRICE` |
| 404 | Unknown resource | `NOT_IN_WATCHLIST` |
| 422 | Malformed request body (FastAPI default validation) | — |

### Trade Execution Rules

These rules apply to both manual trades and trades executed by the AI:

- `quantity` must be > 0 and is rounded to 4 decimal places (minimum 0.0001)
- The trade is rejected (`NO_PRICE`) if no price is cached for the ticker yet
- Fill price is the current cached price; cash changes are rounded to cents
- **Buy**: requires `cash >= quantity × price`. New `avg_cost` = weighted average of the old position and the new shares
- **Sell**: requires `quantity <= held quantity`. `avg_cost` is unchanged. Realized P&L is not tracked (it is reflected only in the cash balance)
- A position whose quantity drops below `1e-9` is treated as zero and its row is deleted
- Validation, the cash update, the position upsert, the `trades` insert and the post-trade snapshot all run in **one SQLite transaction**, so a failure partway through (or a chat trade racing a manual one) cannot leave cash and positions out of sync

---

## 9. LLM Integration

When writing code to make calls to LLMs, use the `cerebras` skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the last 20 messages of conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the `cerebras` skill
5. Parses the complete structured JSON response
6. Auto-executes any trades or watchlist changes specified in the response
7. Stores the message and executed actions in `chat_messages`
8. Returns the complete JSON response to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (required, may be empty): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (see §8 Trade Execution Rules)
- `watchlist_changes` (required, may be empty): Array of watchlist modifications; `action` is `"add"` or `"remove"`

All three fields are required in the schema because strict structured outputs handle optional fields poorly; "no action" is an empty array.

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

Execution details:
- Actions run in order: trades first, then watchlist changes
- Each action is executed independently; a failure is recorded with `status: "failed"` and an `error`, and does not stop the remaining actions
- Each successful trade records a portfolio snapshot, as manual trades do
- Results are returned in the response (§8) and stored in `chat_messages.actions`, so the LLM sees failures in later turns and the user sees them inline

### Failure Handling

If the LLM call fails, times out, or returns JSON that doesn't match the schema, the backend executes nothing and returns HTTP 200 with a friendly assistant message (e.g. "Sorry, I couldn't process that — please try again.") and empty action arrays.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true` (or when `OPENROUTER_API_KEY` is not set), the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

Mock rules (case-insensitive):
- A message containing `buy <N> <TICKER>` or `sell <N> <TICKER>` produces that trade
- A message containing `add <TICKER>` or `remove <TICKER>` produces that watchlist change
- Multiple matches in one message produce multiple actions
- Anything else produces a fixed reply with no actions
- The mock `message` describes what it did (e.g. "Mock: buying 5 NVDA.")

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change % (`session_change_percent` from SSE), and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss). Each tile carries a `data-pnl="positive|negative"` attribute for testing.
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change (unrealized P&L % vs. avg cost)
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Loads history from `GET /api/chat` on page load. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes (including failures) shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Data Flow

- **Live total value**: computed on the client as `cash + Σ(quantity × latest SSE price)`, using positions from `GET /api/portfolio`. No polling.
- **Refetch** `/api/portfolio`, `/api/portfolio/history` and `/api/watchlist` after each trade or chat response.
- **Price history** for sparklines and the main chart is kept on the client only, capped at the last 300 points per ticker (≈ 2.5 minutes at 500ms). No backend price-history table.
- **Connection status**: map `EventSource` state to the dot: open = green, connecting = yellow, closed = red.

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Charting:
  - **Lightweight Charts** (canvas) for the main price chart and the P&L line chart
  - Inline SVG polylines for sparklines (no chart library instance per row)
  - A small squarified-treemap layout rendered as positioned divs for the heatmap (or Recharts `Treemap` if a library is preferred)
  - Don't pull in two full chart libraries
- Price flash effect: on receiving a new price, briefly apply a CSS class (based on the SSE `direction`) with a background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed in production
- The app has no client-side routes, so no SPA catch-all fallback is needed
- Tailwind CSS for styling with a custom dark theme

### Development Workflow

In development, `next dev` runs on :3000 and FastAPI on :8000. Use Next `rewrites` (dev only; they are not applied to a static export) to proxy `/api/*` to `http://localhost:8000`. If SSE buffers through the dev proxy, point `EventSource` directly at `http://localhost:8000` in dev and enable CORS for `http://localhost:3000` in the backend **in dev only**. Production stays same-origin with no CORS.

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 22 slim
  - Copy frontend/
  - npm ci && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app (single worker)
```

FastAPI serves the static frontend files and all API routes on port 8000. Mount `StaticFiles(directory="static", html=True)` at `/` **after** registering the API routers, so `/api/*` takes precedence.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `--env-file .env`
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

There is no reset endpoint. To start over, stop the container and remove the `finally-data` volume.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss, selling an entire position, fractional quantities, no price yet)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow, mock-mode rules
- API routes: correct status codes, response shapes, error handling

**Frontend (Vitest/Jest)** — pure logic only; rendering and CRUD flows are covered by E2E tests:
- P&L, weight and total-value calculations
- Number and percentage formatting
- Price-flash hook (class applied on change, removed afterwards)
- SSE message parsing and history-buffer capping

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image. Each run uses a fresh volume, so tests start from the seeded state.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: each held ticker has a heatmap tile with the right `data-pnl` attribute; the P&L chart has data points
- AI chat (mocked): send `buy 1 AAPL`, receive a response, trade execution appears inline and the position appears
- SSE resilience: abort `/api/stream/prices` with `page.route`, verify the status dot goes yellow/red, then restore and verify it returns to green

---

## 13. Review Decisions Log

The 2026-10-07 review of this plan (against the completed market data code) raised 27 questions and 8 simplifications. Their suggested answers have been applied to the sections above. Where the review asked a question without proposing an answer, a default was chosen. Those defaults are listed here so the owner can override them:

| Topic | Decision | Where |
|---|---|---|
| Realized P&L | Not tracked; only reflected in cash | §8 Trade Execution Rules |
| Snapshot history | `/api/portfolio/history` returns all snapshots (~2,900/day), no downsampling | §8 |
| Massive cadence | 15s updates and stale off-hours prices accepted; documented in README | §6 |
| Chat history endpoint size | Last 50 messages for display; last 20 sent to the LLM | §8, §9 |
| Price unavailable for a held position | Valued at `avg_cost` until a price arrives | §8 |

Code changes implied by these decisions (not yet made):
- `PriceCache` / `PriceUpdate`: add session-start price and `session_change_percent` to the SSE payload (§6)
- Market data factory: read `MASSIVE_POLL_INTERVAL` (§5)
- Repo: add `db/.gitkeep` and the `db/*.db*` ignore rules (§4)
