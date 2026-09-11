# garminbadges-public

Public documentation of the architecture and ecosystem of **[GarminBadges](https://garminbadges.com)** ("Garmin Badge Database") — a community site that tracks Garmin Connect achievement badges — and the wider ecosystem of client apps that feed it data.

The core application source is private. This repo exists to document the system publicly: how the pieces fit together, what each component does, and how data flows between them.

## Highlights

A few things about this system worth a second look:

- **No official Garmin API, so there isn't one client — there are four.** Garmin doesn't expose badge/challenge data publicly. Rather than one fragile scraper, the ecosystem has four independent implementations (a Chrome extension, a native Android app, and two Python scripts) that each authenticate to Garmin their own way and push into one shared upload contract. Any one of them can break, get fixed, or get replaced without touching the others.
- **A real native Garmin login on Android, not a webview.** [`garminbadges-android`](docs/components/android-app.md) performs Garmin's SSO flow — including MFA — and the OAuth2 token exchange itself, then talks to Garmin's internal API directly. No embedded browser, no cookie-jar hand-off.
- **A watch app that respects its constraints.** The [Connect IQ app](docs/components/watch-app.md) (Monkey C) ships both a full app and a lightweight glance view, and explicitly drops support for older Instinct models that can't fit it in memory — a small but real embedded-device tradeoff, made deliberately rather than discovered by crash reports.
- **RAG without a vector database.** The [AI assistant](docs/components/ai-assistant.md) retrieves relevant badges and admin notes with plain keyword matching instead of standing up embeddings/vector search — a deliberate call that the badge catalogue is small and structured enough not to need it.
- **One backend that stays out of the scraping business.** The [core app](docs/components/core-backend.md) never talks to Garmin itself — it only owns storage and presentation behind a single sync contract, so the messy, fragile "reverse-engineer Garmin's API" problem lives entirely in swappable clients instead of the system of record.

## The ecosystem at a glance

GarminBadges is not a single app — it's one central backend plus several independent **ingestion clients** that each know how to talk to Garmin Connect and push data into it.

| Component | Repo | What it is | Role |
|---|---|---|---|
| **Core backend + web app** | [`garminbadges`](https://github.com/e7andy/garminbadges) *(private)* | Laravel 12 API + Angular 18 SPA | Central database, website, social/gamification layer, presentation |
| **Browser sync tool** | [`garminbadges-updater`](https://github.com/e7andy/garminbadges-updater) | Chrome extension (Manifest V3) | Syncs badge/challenge data from a logged-in Garmin Connect tab |
| **Android sync app** | [`garminbadges-android`](https://github.com/e7andy/garminbadges-android) | Native Java app | Syncs badge/challenge data via direct Garmin SSO login, no browser needed |
| **Watch companion app** | [`garminbadges-watch`](https://github.com/e7andy/garminbadges-watch) | Garmin Connect IQ app (Monkey C) | Read-only on-wrist view of upcoming badges & challenge progress |

## The core idea: one backend, many ingestion clients

Garmin does not offer a public, official API for badge/challenge data. GarminBadges works around this with a deliberate split:

- The **core backend never talks to Garmin directly**. It exposes a small REST surface — most importantly `POST /api/sync` — and stores whatever badge data is pushed to it, authenticated with a per-user API key that GarminBadges itself issues (not a Garmin credential).
- Three independent clients (**Chrome extension**, **Android app**, and companion **Python scripts** shipped inside the core repo) each implement their own logic for authenticating to Garmin Connect and reading a user's badge/challenge data from Garmin's internal endpoints. They normalize what they find into the same `user_badges` shape and `POST` it to `/api/sync`.
- The **watch app** is the odd one out: it doesn't sync anything. It's purely a consumer, calling `GET /api/watch` to render already-synced data on-device.

This decouples "getting data out of Garmin" (four independent, replaceable implementations) from "owning the data and presenting it" (one backend). See [`docs/architecture.md`](docs/architecture.md) for the full data-flow diagram.

## Component documentation

- [`docs/architecture.md`](docs/architecture.md) — system diagram, sync flow, API surface, data model
- [`docs/components/core-backend.md`](docs/components/core-backend.md) — the Laravel/Angular core app
- [`docs/components/ai-assistant.md`](docs/components/ai-assistant.md) — the supporter-only AI features (Q&A chat, recaps) and admin knowledge base
- [`docs/components/updater-extension.md`](docs/components/updater-extension.md) — the Chrome extension
- [`docs/components/android-app.md`](docs/components/android-app.md) — the Android sync app
- [`docs/components/watch-app.md`](docs/components/watch-app.md) — the Connect IQ watch app

## Status

This documentation reflects a point-in-time review of the repositories above. It is maintained separately from the source repos and may lag behind them.
