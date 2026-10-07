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
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
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
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

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
│   └── db/                   # Schema definitions, seed data, migration logic
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
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Optional convenience wrapper
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/db/`** contains schema SQL definitions and seed logic. The backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all watched tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, and timestamp for each ticker
- SSE streams read from this cache and push updates to connected clients
- This architecture supports future multi-user scenarios without changes to the data layer

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- Server pushes price updates for all tickers known to the system at a regular cadence (~500ms) — in the single-user model this is equivalent to the user's watchlist
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column defaulting to `"default"`. This is hardcoded for now (single-user) but enables future multi-user support without schema migration.

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

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded every 30 seconds by a background task, and immediately after each trade execution.
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

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

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
| POST | `/api/chat` | Send a message, receive complete JSON response (message + executed actions) |

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads recent conversation history from the `chat_messages` table
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter, requesting structured output, using the cerebras-inference skill
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
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

If a trade fails validation (e.g., insufficient cash), the error is included in the chat response so the LLM can inform the user.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart (accumulated from SSE since page load)
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill.
- **AI chat panel** — docked/collapsible sidebar. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Trade executions and watchlist changes shown inline as confirmations.
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- Canvas-based charting library preferred (Lightweight Charts or Recharts) for performance
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Docker Volume

The SQLite database persists via a named Docker volume:

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Runs the container with the volume mount, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline
- SSE resilience: disconnect and verify reconnection

---

## 13. Review: Questions, Clarifications & Simplifications

*Review of this plan against the completed market data code in `backend/app/market/` (2026-10-07). Each item is tagged **[Q]** question to resolve, **[C]** clarification to add to the spec, or **[S]** opportunity to simplify. Suggested answers are given where there is an obvious default, so an agent can proceed unless the owner overrides.*

### 13.1 Inconsistencies with existing code / environment

1. **[C] "Daily change %" has no data source.** §10 asks the watchlist to show *daily* change %, but `PriceUpdate.change_percent` is tick-to-tick (vs. the previous ~500ms price), which will hover near 0%. The simulator has no "open" or "previous close". *Suggested:* define daily change as change versus the **session-start price** (the seed price for the simulator, or the first price polled from Massive), kept in `PriceCache` and added to the SSE payload (e.g. `session_change_percent`).
2. **[C] The SSE payload shape differs from §6.** §6 says "each SSE event contains ticker, price, …". The implementation sends **one event per tick holding a dict of all tickers**, keyed by symbol (`{"AAPL": {...}, "GOOGL": {...}}`), with fields `ticker, price, previous_price, timestamp (unix seconds), change, change_percent, direction`. It is sent only when the cache version changes. Document this exact shape here so the frontend agent codes against it.
3. **[C] Wrong skill name in §9.** The text says "use cerebras-inference skill", but the skill available to agents is named `cerebras`. Fix the reference so agents load the right skill.
4. **[C] `db/` directory and gitignore.** §4 says `db/.gitkeep` exists and `finally.db` is gitignored. Neither is true yet: there is no `db/` directory, and `.gitignore` only ignores `db.sqlite3`. Add `db/.gitkeep` and ignore `db/*.db` (plus `*.db-wal`, `*.db-shm` if WAL mode is used).
5. **[C] Node 20 is end-of-life.** Node 20 reached EOL in April 2026, and current Next.js releases target newer Node versions. *Suggested:* use `node:22-slim` (or 24) for build stage 1 in §11.
6. **[C] "Canvas-based charting library preferred (Lightweight Charts or Recharts)".** Recharts renders SVG, not canvas. Lightweight Charts has no treemap. See S3 for a recommendation.
7. **[C] `users_profile` and `user_id`.** §7 says "all tables include a `user_id` column", but `users_profile` uses `id` as the user key. That's fine; just state it explicitly as the one exception.

### 13.2 Behavior that is currently undefined

8. **[Q] Prices for held tickers that are not on the watchlist.** If the user (or the AI) removes a ticker from the watchlist while still holding a position in it, the market source stops tracking it. The portfolio can then no longer be valued, and the position can't be sold (no current price). The same applies to buying a ticker that isn't on the watchlist. *Suggested:* the market source tracks the **union of watchlist tickers and open-position tickers**. A trade on an untracked ticker starts tracking it (without adding it to the watchlist).
9. **[Q] Which tickers are valid?** The simulator can generate a price for any string (e.g. `ASDFG`), while Massive returns nothing for unknown symbols. *Suggested:* normalize to uppercase and validate against `^[A-Z.]{1,10}$`. In simulator mode, accept any well-formed symbol, and document which seed price and params unknown tickers get. In Massive mode, reject tickers that return no snapshot.
10. **[Q] Startup ordering versus "lazy initialization".** §4/§7 say the DB is initialized lazily on first request. But the market source needs the watchlist tickers *at startup* (`source.start(tickers)`), so the DB must be read before any request arrives. *Suggested:* initialize and seed eagerly in the FastAPI `lifespan` handler, then start the market source, then start the snapshot task (see S1).
11. **[C] Trade validation rules.** Spell out the following:
    - quantity must be > 0
    - minimum and precision for fractional shares (e.g. up to 4 decimal places)
    - reject a trade when no price is cached yet for that ticker
    - on a sell, `avg_cost` stays the same; is realized P&L tracked at all?
    - a sell that reaches 0 deletes the position row (treat `abs(qty) < 1e-9` as zero to avoid float residue)
    - cash rounded to cents
12. **[C] Error response format and status codes.** Define one error shape (e.g. `{"error": "Insufficient cash", "code": "INSUFFICIENT_CASH"}`) with HTTP 400 for business-rule failures and 404 for an unknown ticker on watchlist DELETE. Also say whether adding a duplicate watchlist ticker is a 409 or an idempotent 200 (*suggested:* idempotent 200).
13. **[C] Response shapes for REST endpoints.** §8 lists the endpoints but not their JSON responses. At minimum, define:
    - `GET /api/portfolio`: `cash`, `total_value`, and `positions[]` with ticker, quantity, avg_cost, current_price, market_value, unrealized_pnl, unrealized_pnl_percent
    - `POST /api/portfolio/trade`: the executed trade plus the updated portfolio
    - `GET /api/portfolio/history`: `[{recorded_at, total_value}]`
    - `GET /api/watchlist`: price is nullable until the first tick
    - `POST /api/chat`: `message`, plus `trades[]` and `watchlist_changes[]`, each with `status: "executed" | "failed"` and an `error`

    Frontend and backend agents work in parallel, so this contract matters more than anything else in this list.
14. **[Q] Portfolio snapshots.**
    - With 30s snapshots, a fresh start shows an empty P&L chart for 30s. Record one snapshot at startup.
    - Snapshots grow without limit (~2,900/day), though that's fine for SQLite at this scale. Decide whether `/history` returns everything or downsamples.
15. **[C] How the header total value updates "live".** Is it recomputed on the client from `/api/portfolio` positions + SSE prices, or is `/api/portfolio` polled? *Suggested:* compute it on the client, and refetch `/api/portfolio` after each trade or chat response.
16. **[C] LLM details.**
    - How many history messages are sent? (*Suggested:* the last 20.)
    - What happens when the LLM call fails or returns invalid JSON? (*Suggested:* return a friendly assistant message, execute nothing, return HTTP 200.)
    - Are actions executed in order, with each failure reported individually and not aborting the others?
    - Do chat trades also trigger a portfolio snapshot? (They should, to match manual trades.)
    - With strict structured outputs, optional fields are usually awkward, so define `trades` and `watchlist_changes` as required arrays that may be empty.
17. **[C] Deterministic mock LLM behavior.** The E2E test "trade execution appears inline" needs the mock to actually trade. Specify the mock rules, e.g.:
    - a message containing `buy <N> <TICKER>` / `sell <N> <TICKER>` produces that trade
    - `add <TICKER>` / `remove <TICKER>` produces that watchlist change
    - anything else produces a fixed reply with no actions
18. **[C] Single worker.** The price cache, market source and snapshot task all live in memory, so uvicorn **must run with one worker**. State this explicitly so nobody adds `--workers N`.
19. **[Q] Dev workflow for the frontend.** Running `next dev` on :3000 against FastAPI on :8000 is cross-origin, and Next `rewrites` are not applied in a static export. *Suggested:* in dev only, use Next `rewrites` to proxy `/api/*` to :8000 (they work under `next dev`). If SSE buffers through the dev proxy, point `EventSource` straight at :8000 and enable CORS for `localhost:3000` in dev only. Production stays same-origin.
20. **[C] Static file serving.** Mount `StaticFiles(directory="static", html=True)` at `/` *after* the API routers. Confirm the app has no client-side routes, so no SPA catch-all fallback is needed.
21. **[Q] Massive mode cadence.** With 15s polling, prices flash only every 15s and sparklines are sparse. Massive also returns stale prices outside market hours. Is that acceptable? A one-line note in the README would set expectations.
22. **[Q] Reset.** Is there a way to reset the portfolio to $10k (useful for demos and E2E test isolation)? *Suggested:* no reset endpoint in the core build; the E2E compose file uses a fresh volume per run.
23. **[C] The Massive poll interval isn't actually configurable.** §6 says Massive polls "on a configurable interval" (2–15s by tier), but `create_market_data_source()` never passes `poll_interval`, so it is always the 15s default. *Suggested:* add an optional `MASSIVE_POLL_INTERVAL` env var (seconds, default 15) to §5 and read it in the factory.
24. **[Q] Chat history on page load.** `chat_messages` is persisted, but §8 has no endpoint to read it, so a page refresh shows an empty chat even though the LLM still "remembers" the conversation. *Suggested:* add `GET /api/chat` returning recent messages with their `actions`. Otherwise, state explicitly that the chat panel starts empty on every load.
25. **[C] "% change" in the positions table.** It's ambiguous next to the watchlist's daily change. *Suggested:* define it as unrealized P&L % relative to cost: `(current_price - avg_cost) / avg_cost × 100`.
26. **[C] Timestamp format.** SSE `timestamp` is Unix seconds (float), while every DB and REST timestamp is ISO 8601. Pick one convention for REST responses (*suggested:* ISO 8601 UTC with a `Z` suffix) and note that SSE uses Unix seconds, so the frontend converts consistently.
27. **[C] Trade atomicity.** A trade updates `users_profile.cash_balance`, upserts `positions`, appends to `trades` and writes a snapshot. Do the validation and all of these writes in **one SQLite transaction**, so a failure partway (or a chat trade racing a manual one) can't leave cash and positions out of sync.

### 13.3 Opportunities to simplify

- **S1. Eager init in `lifespan`, not lazy init on first request.** Run `CREATE TABLE IF NOT EXISTS` and seed when `users_profile` is empty, once, at startup. This removes per-request checks and the first-request race, and fixes item 10. The user-facing behavior is the same: no migration step and fresh volumes self-seed.
- **S2. Pick start scripts *or* docker-compose, not both.** §4 lists `docker-compose.yml` as an "optional convenience wrapper" alongside four start/stop scripts. That's two ways to run the same container that can drift apart. *Suggested:* keep the scripts (the documented student path) and drop the root `docker-compose.yml`. `test/docker-compose.test.yml` stays.
- **S3. Settle on one charting approach.** *Suggested:*
  - Lightweight Charts for the main price chart and the P&L line (canvas, built for streaming financial data)
  - inline SVG polylines for sparklines (a few lines of code; a chart instance per row is overkill)
  - a small squarified-treemap layout rendered as positioned divs, or Recharts `Treemap` if a library is preferred

  Avoid pulling in two full chart libraries.
- **S4. Fall back to mock automatically when there's no OpenRouter key.** If `OPENROUTER_API_KEY` is missing, behave as if `LLM_MOCK=true` and log a warning. Students without a key still get a working app, and the key becomes optional rather than "Required".
- **S5. Read env vars only through `--env-file`.** §5 allows either mounting `.env` into the container or passing `--env-file`. Choose `--env-file` only, so the container never needs the file. For local `uv run` development, `python-dotenv` can load `../.env`.
- **S6. Trim the frontend unit-test scope.** Focus frontend unit tests on pure logic (P&L/weight calculations, number formatting, the price-flash hook, SSE message parsing) and leave rendering and CRUD flows to Playwright. That avoids testing the same thing twice.
- **S7. Make E2E assertions robust.** "Heatmap renders with correct colors" is brittle; assert instead that each held ticker has a tile with `data-pnl="positive|negative"`. For SSE resilience, abort `/api/stream/prices` with `page.route` and check that the status dot goes to "reconnecting" and back. A container restart isn't needed.
- **S8. Cap client-side price history.** Keep sparkline and main-chart history on the client only, as specified, but cap the buffer per ticker (e.g. the last 300 points ≈ 2.5 min at 500ms) so memory stays bounded in a long-open tab. No backend price-history table is needed.
