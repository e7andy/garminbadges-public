# garminbadges-public

Public documentation of the architecture and ecosystem of **[GarminBadges](https://garminbadges.com)** ("Garmin Badge Database") — a community site that tracks Garmin Connect achievement badges — and the wider ecosystem of client apps that feed it data.

The core application source is private. This repo exists to document the system publicly: how the pieces fit together, what each component does, and how data flows between them.

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
