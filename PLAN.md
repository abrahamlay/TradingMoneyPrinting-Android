# TradingMoneyPrinting Android App — Implementation Plan

> Status: Planning
> Source project: `/home/opc/trading-futures-project/TradingMoneyPrinting`
> Target: Native Android client for monitoring and controlling the existing TradingMoneyPrinting bot

## 1. Executive decision

Build a **native Android client**, not a second trading engine.

- Android owns presentation, local preferences, cached read-only data, authentication/session handling, and mobile notifications.
- The existing Python/Flask application remains the source of truth for trading execution, risk management, exchange credentials, journals, strategies, and bot state.
- The Android app communicates with the Flask dashboard API over HTTPS.
- No Bitget API keys, exchange signing secrets, or trading strategy code are placed in the APK.

This is the safest way to copy the user-facing capabilities of TradingMoneyPrinting without duplicating financial logic or creating two competing trading engines.

## 2. Feature parity inventory

### 2.1 Existing bot capabilities to expose

The Android app should expose the existing backend capabilities, grouped by mobile priority:

| Area | Existing source | Android capability | MVP |
|---|---|---|---|
| Bot status | `dashboard/app.py` `/api/status` | Running/active state, paper/live indicator, mode state, balance and cycle summary | Yes |
| Open positions | `/api/positions` | Position list, mode filter, entry price, PnL, leverage, margin, liquidation/break-even details when available | Yes |
| Trade history | `/api/trades` | Recent trades with symbol, mode, entry/exit time, PnL and strategy | Yes |
| Performance | `/api/performance` | PnL, win rate, trade counts and mode breakdown | Yes |
| Watchlist | `/api/watchlist`, `/api/watchlist/add`, `/api/watchlist/remove`, `/api/watchlist/reorder` | Add/remove/reorder symbols, local display ordering | Yes |
| Journal signals | `/api/signals/journal` | Last recorded signal per watched symbol; explicitly historical, not live analysis | Yes |
| Coin search | `/api/coins/search` | Search scanned/top-gainer symbols and add to watchlist | Yes |
| Start/stop | `/api/bot/start`, `/api/bot/stop` | Start bot and emergency stop with confirmation | Yes |
| Mode control | `/api/bot/toggle_mode` | Toggle Spot, Futures, Fast, Micro with confirmation and current-state refresh | Yes |
| Fast Futures | `/api/fast_futures/summary`, `/api/fast_futures/logs` | Fast mode summary and logs | Phase 2 |
| Micro Scalper | `/api/micro_scalper/summary`, `/api/micro_scalper/logs` | Micro summary, sub-mode, logs and reset/stop actions | Phase 2 |
| Learning | `/api/learning_summary`, `/api/learning`, `/api/strategies` | Learning progress, strategy performance, evolution status | Phase 2 |
| Market analysis | `/api/top_gainers`, `/api/predictions`, `/api/coin_analysis`, scanner endpoints | Market overview and selected analysis | Phase 3 |
| Cycles/logs | `/api/cycles`, `/api/cycle_stats`, `/api/logs` | Diagnostics and cycle monitoring | Phase 3 |

### 2.2 Explicitly not copied into Android

- `main.py` trading loop
- `core/` exchange and Binance/Bitget credentials
- `strategies/` execution logic
- `risk/` authority and sizing logic
- `execution/` order placement
- SQLite database as an authoritative client-side trading database
- Any secret loaded from `.env`

The Android app can cache API responses for display, but all mutations must go through authenticated backend endpoints.

## 3. Proposed Android stack

- Kotlin
- Jetpack Compose + Material 3
- Single-activity architecture
- Navigation Compose
- ViewModel + Kotlin Coroutines + StateFlow
- Retrofit + OkHttp for HTTP
- Kotlin serialization or Moshi for JSON
- DataStore for server URL, non-secret preferences, and UI settings
- Encrypted storage (Android Keystore-backed) for session/token material if the backend authentication contract supports tokens
- WorkManager only for bounded background refresh/notifications; do not run a trading loop in Android
- JUnit, Turbine, MockWebServer, and Compose UI tests

## 4. Backend contract and prerequisites

The current Flask dashboard uses session login and HTTP Basic Auth fallback. Before production Android release, add a mobile-safe authentication contract:

1. Prefer short-lived access token + refresh token endpoints.
2. Scope tokens to the mobile client and revoke them server-side.
3. Require HTTPS outside localhost/LAN development.
4. Return consistent JSON error envelopes for `401`, `403`, `409`, `422`, `429`, and `5xx`.
5. Add an API version prefix or compatibility policy, for example `/api/v1`.
6. Add an idempotency key to destructive control requests where practical.
7. Include server time, bot state version, and `updated_at` in status responses.
8. Add pagination/limits for trades, logs, and analysis payloads.
9. Keep existing web dashboard endpoints backward-compatible during migration.

### Existing route mapping

```text
GET  /api/status
GET  /api/positions?mode=all|spot|futures|fast|micro
GET  /api/trades?limit=N&mode=...
GET  /api/performance
GET  /api/watchlist
POST /api/watchlist/add             {"symbol":"BTC/USDT"}
POST /api/watchlist/remove          {"symbol":"BTC/USDT"}
PUT  /api/watchlist/reorder         {"symbols":[...]}
GET  /api/signals/journal?watchlist_only=true&limit=N
GET  /api/coins/search?q=BTC
POST /api/bot/start
POST /api/bot/stop
POST /api/bot/toggle_mode           {"mode":"spot|futures|fast|micro","active":true}
```

## 5. UX and navigation

### Bottom navigation

1. **Overview** — bot status, paper/live warning, balance, PnL, active position count, last refresh.
2. **Positions** — active positions with Spot/Futures/Fast/Micro filters.
3. **Watchlist** — search, add/remove/reorder, latest journal signal.
4. **History** — recent trades, filters, trade detail.
5. **Controls** — start/stop and mode toggles behind explicit confirmations.

### Global UX rules

- Display a persistent connection state: Connected, Refreshing, Offline, Authentication required, or Server error.
- Show the timestamp and age of every remote snapshot.
- Never display stale values as current.
- Use red/amber warnings for live mode and destructive actions.
- Require a second confirmation for Kill Switch and live-mode changes.
- Use Indonesian copy to match the existing dashboard where appropriate, with English technical labels only when needed.
- Support dark theme first; preserve accessible contrast and touch targets of at least 48dp.
- Do not auto-execute trades from a push notification or background task.

## 6. Data and state model

### Remote models

Create typed models for:

- `BotStatus`
- `PortfolioSummary`
- `Position`
- `Trade`
- `PerformanceSummary`
- `WatchlistItem`
- `JournalSignal`
- `CoinSearchResult`
- `ModeState`
- `ApiError`

Models must tolerate additive backend fields and missing optional fields. PnL, prices, leverage, and timestamps need explicit parsing rules and tests.

### Local state

- `DataStore`: server URL, theme, selected filters, refresh interval, onboarding completion.
- Secure credential store: only access/refresh tokens or equivalent session material.
- In-memory ViewModel state: current screen state and request status.
- Optional bounded cache: last successful read-only snapshot; never treat it as permission to mutate bot state.

## 7. Project structure

```text
app/
  src/main/java/.../
    data/
      remote/          Retrofit service, auth interceptor, DTOs
      repository/      API repositories and error mapping
      local/           DataStore and secure session store
    domain/
      model/           app models
      usecase/         refresh/watchlist/control use cases
    ui/
      navigation/
      overview/
      positions/
      watchlist/
      history/
      controls/
      components/
      theme/
    worker/            optional refresh/notification workers
  src/test/             unit/repository/ViewModel tests
  src/androidTest/      Compose and navigation tests
```

## 8. Delivery phases

### Phase 0 — Repository and safety baseline

- Create Android project with Kotlin, Compose, Material 3, lint, formatting, and CI.
- Add app ID, development/staging/prod build variants.
- Add no-secret policy and `.gitignore`.
- Add `PLAN.md`, README, architecture decision record, and threat model.
- Add a mock API fixture layer so UI work never requires live trading credentials.

**Exit criteria:** app launches to a mock Overview screen; CI builds debug APK and runs unit tests.

### Phase 1 — Read-only MVP

- Configure server URL and authentication.
- Implement API client and typed models.
- Implement Overview, Positions, Watchlist read/add/remove, and History.
- Implement 10-second foreground polling with lifecycle-aware cancellation.
- Implement loading, empty, offline, unauthorized, and server-error states.
- Do not add control mutations yet.

**Exit criteria:** a tester can connect to a paper-trading backend and safely inspect state; no trading action is available.

### Phase 2 — Safe bot controls

- Add Start, Stop/Kill Switch, and mode toggles.
- Add confirmation dialogs, live-mode warnings, request progress, duplicate-request protection, and post-action refresh.
- Add audit display showing action, user, timestamp, and backend result.

**Exit criteria:** controls are tested against a mock server and a paper backend; destructive controls cannot be triggered accidentally by navigation or retries.

### Phase 3 — Notifications and richer monitoring

- Add FCM notifications for backend-generated events only: position closed, stop-loss/take-profit, bot stopped, authentication failure, and server health failure.
- Add notification preferences and deep links.
- Add Fast/Micro summaries, learning status, and logs.
- Add bounded background work; no continuous background polling guarantee.

**Exit criteria:** notifications are authenticated, opt-in, deduplicated, and do not contain secrets.

### Phase 4 — Market intelligence and analytics

- Add top gainers, predictions, coin analysis, market sessions, scanner output, cycle statistics, and strategy metrics.
- Add charts only after payload size and mobile performance are measured.
- Add pagination and server-side aggregation before shipping large logs or history.

**Exit criteria:** analytics remain read-only, have freshness labels, and render acceptably on low-end Android devices.

### Phase 5 — Release hardening

- Threat model review and penetration test of mobile API/auth flows.
- ProGuard/R8, certificate/network security configuration, crash reporting, privacy policy, and release signing.
- Offline behavior and token expiry tests.
- Play internal testing, staged rollout, rollback runbook, and operator documentation.

**Exit criteria:** signed release candidate passes functional, security, performance, and operational checklists.

## 9. Testing strategy

### Unit tests

- JSON parsing with missing/additive fields.
- PnL and timestamp formatting.
- Repository error mapping.
- Polling cancellation and retry/backoff.
- Watchlist reorder and duplicate handling.
- Control confirmation and state transitions.

### API integration tests

Use MockWebServer to verify:

- Correct route, method, query, and JSON body.
- Auth header/cookie behavior.
- `401` token expiry and refresh.
- `409` duplicate watchlist item.
- `429` retry policy.
- Timeout and malformed response handling.

### Compose/UI tests

- Navigation between all MVP tabs.
- Loading/empty/offline/error states.
- Live-mode warning and kill-switch confirmation.
- Position filters and watchlist add/remove.
- Accessibility labels and minimum touch targets.

### Backend contract tests

Add a checked-in API contract fixture generated from Flask responses. Run compatibility tests against a local paper-trading Flask instance before Android releases.

### Manual device matrix

- Android emulator: current API and minimum supported API.
- One low-end physical device.
- One recent Pixel/Samsung-class device.
- Wi-Fi, cellular, offline, slow network, expired session.
- Portrait and dark theme at minimum.

## 10. Security and operational requirements

- Never commit Bitget keys, `.env`, access tokens, refresh tokens, signing keys, or production URLs containing credentials.
- Use HTTPS with certificate validation; do not ship a trust-all HTTP client.
- Keep bot controls disabled until authentication and authorization are confirmed.
- Separate paper and live server profiles visibly.
- Do not log Authorization headers, cookies, tokens, or sensitive request bodies.
- Rate-limit control endpoints server-side.
- Record audit events for every control mutation.
- Use least-privilege mobile API credentials.
- Treat Kill Switch as high-impact functionality; test it against a known paper state before release.

## 11. Repository workflow

Suggested new repository name:

```text
TradingMoneyPrinting-Android
```

Initial commits:

1. `docs: add Android parity and safety plan`
2. `build: bootstrap Android Compose project`
3. `feat: add authenticated API client and models`
4. `feat: add read-only mobile dashboard`
5. `feat: add guarded bot controls`
6. `test: add API contract and UI coverage`
7. `release: add Android deployment documentation`

Recommended branches:

- `main` — releasable code
- `develop` — integration branch if needed
- `feature/*` — focused changes

## 12. Definition of done for Android MVP

- The app launches and connects to a configured Flask server.
- A user can authenticate without embedding exchange credentials.
- Overview, Positions, Watchlist, and History display current data with freshness state.
- Watchlist add/remove/reorder works against the backend.
- Foreground refresh runs every 10 seconds and stops when the app is not visible.
- Offline, expired-session, malformed-response, and server-error states are understandable.
- No Android code can place exchange orders directly.
- Debug build, unit tests, Compose tests, and API contract tests pass in CI.
- README documents local setup, paper mode, live-mode warning, and release limitations.

## 13. Risks and decisions to revisit

| Risk | Mitigation | Revisit when |
|---|---|---|
| Current Flask auth is browser-oriented | Add token auth before controls/release | Before Phase 1 production use |
| Polling can become expensive | 10s foreground only; add server aggregation | When user/device count grows |
| Backend schema changes can break app | Version/contract tests and tolerant DTOs | Before first public release |
| Push delivery is not guaranteed | Notifications are advisory; app refreshes on open | Before relying on alerts operationally |
| Mobile network exposes controls | HTTPS, token scope, confirmation, audit | Before Phase 2 |
| Copying engine into Android creates divergence | Keep Python engine authoritative | Always |

## 14. Immediate next tasks

1. Confirm repository visibility and final Android package name.
2. Add token-based mobile auth to the Flask backend.
3. Create the Android Compose skeleton and CI build.
4. Freeze API response fixtures from the paper-trading dashboard.
5. Implement read-only Overview and Positions first.
6. Add watchlist repository and UI.
7. Validate on a real Android device before enabling controls.
