# Watch companion app (`garminbadges-watch`)

Repository: [`e7andy/garminbadges-watch`](https://github.com/e7andy/garminbadges-watch)

## Purpose

"Badge Tracker" — a Garmin Connect IQ watch app that surfaces a user's [garminbadges.com](https://garminbadges.com) stats directly on their watch: upcoming badges (next 7 days), challenges ending soon, and in-progress challenges ranked by how far behind schedule they are. Unlike every other component in the ecosystem, this app is **read-only** — it never syncs data, it only displays data that one of the other clients has already pushed to the backend. Data is fetched live and cached on-device for instant display on launch, with a "View Online" option linking out to garminbadges.com in the phone's browser.

## Tech stack

A genuine Garmin Connect IQ app written in **Monkey C** — 21 `.mc` source files, a `manifest.xml`, and a `jungle.xml` build config. `minApiLevel="3.3.0"`.

**Supported devices** (per `manifest.xml`): Fenix 6/7/8/9/e, Epix 2 (+ Pro), Enduro / Enduro 3, Venu 2/3/X1, Vivoactive 4/5/6, Forerunner 245–970, Instinct 3 AMOLED, Marq 2. Older Instinct models (2/2X/2S/3 Solar) are explicitly excluded due to glance-memory and screen limitations.

## Architecture

- Declared as `type="watch-app"` in `manifest.xml`, with entry point `GarminBadgesApp` (extends `AppBase`).
- Also implements a **glance view** (`GarminBadgesGlanceView` via `getGlanceView()`), giving it a preview in the watch's glance loop in addition to the full app.
- Follows a View/Delegate pattern per screen: main page, "Next Badges", "Ending Soon", and "All Challenges" list pages, plus detail views for individual challenges and upcoming badges.
- Shared helpers: `ScrollableView` / `ScrollDelegate` (scroll/drag/flick handling), `BadgeFormat` (formatting/drawing), `BadgeCache` (on-device response caching so the app shows data instantly on launch, before the network round-trip completes).
- Resources live in `resources/` — `properties.xml` defines the user-configurable settings schema (including the API base URL), `drawables/` holds icons and store screenshots.

## Backend communication

Uses the Connect IQ **Communications API** (`iq:uses-permission id="Communications"`) to call `GET {apiUrl}/watch`, authenticated with the user's GarminBadges API key as a bearer token. Default `ApiUrl` is `https://api.garminbadges.com/api`, user-configurable to point at a self-hosted or development backend. On the backend side, this hits a purpose-built endpoint (`WatchController`) that pre-shapes the response specifically for a small screen — see [`../architecture.md`](../architecture.md#api-surface-as-used-by-the-ecosystem).

## Distribution

No CI workflow in this repo. Building and publishing is manual: compile a signed `.prg` via `monkeyc`, then submit through the Connect IQ Developer Portal. Store assets (`app_store_icon.png`, screenshots) live under `resources/drawables/`.

## Relationship to the rest of the ecosystem

- The only ecosystem component that never authenticates to Garmin or writes data — it exclusively reads from the [core backend's](core-backend.md) `GET /api/watch` endpoint.
- No code-level relationship to `garminbadges-updater` or `garminbadges-android` — it depends only on data those clients have already synced.
