# TradingMoneyPrinting Android

Native Android client plan and implementation workspace for the existing TradingMoneyPrinting bot.

This repository is intentionally starting as a planning baseline. The Python/Flask bot remains the authoritative trading engine; the Android app will consume authenticated APIs and will never contain Bitget credentials or duplicate order execution logic.

## Source project

- `/home/opc/trading-futures-project/TradingMoneyPrinting`
- Existing dashboard: Flask on port `5000`
- Existing mobile web route: `/mobile`
- Android parity plan: [`PLAN.md`](PLAN.md)

## Safety boundary

The Android app must start read-only. Bot controls are a later phase and require token-based authentication, HTTPS, explicit confirmations, audit logging, and paper-trading validation.

## Planned stack

Kotlin, Jetpack Compose, Material 3, Retrofit/OkHttp, Coroutines/StateFlow, DataStore, Android Keystore, WorkManager, JUnit, MockWebServer, and Compose UI tests.

## Current status

- [x] Feature inventory and API mapping
- [x] Android architecture and phased delivery plan
- [ ] Android project bootstrap
- [ ] Authenticated API contract
- [ ] Read-only MVP
- [ ] Guarded bot controls

See [`PLAN.md`](PLAN.md) for scope, phases, security requirements, testing, and definition of done.
