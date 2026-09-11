# Browser sync tool (`garminbadges-updater`)

Repository: [`e7andy/garminbadges-updater`](https://github.com/e7andy/garminbadges-updater)

> **Highlight**: no login screen of its own — it rides on the user's already-authenticated Garmin Connect tab, so there's no credential handling in the extension at all.

## Purpose

A Chrome extension that syncs a user's badge and challenge data from Garmin Connect to [garminbadges.com](https://garminbadges.com). This is one of the ecosystem's ingestion clients — the mechanism by which a user's real Garmin activity/achievement data enters the GarminBadges platform.

## Tech stack

Plain vanilla JavaScript — no build system, no bundler, no framework. Standard Chrome Extension **Manifest V3** structure: `manifest.json`, `background.js` (service worker), `content.js`, `sync.js`, `popup.html`/`popup.js`, `options.html`/`options.js`. Entirely client-side; no server-side component of its own.

## Architecture

| File | Role |
|---|---|
| `background.js` | Service worker. Handles install/update lifecycle, relays sync state between tabs, and acts as an allowlisted fetch proxy (only `garminbadges.com` and `localhost` are permitted destinations). |
| `content.js` | Injected on `*://connect.garmin.com/*`. Adds a floating "Sync Badges" button to the Garmin Connect page. |
| `sync.js` | The actual sync logic. Injected into the active tab on demand (via `chrome.scripting.executeScript`) when the user clicks "Sync now" — not statically loaded. |
| `popup.html` / `popup.js` | Toolbar popup UI showing sync progress/results. |
| `options.html` / `options.js` | Settings page for the API key and API base URL (so it can point at a local dev backend). |

Execution is entirely **user-initiated** — there's no scheduled or automatic sync.

## How syncing works

`sync.js` runs inside the `connect.garmin.com` tab, reusing the user's existing logged-in session/CSRF token to call Garmin's internal endpoints directly:

- `/gc-api/badge-service/badge/earned`
- `/badgechallenge-service/...`
- `/badge-service/badge/available`
- `/badge-service/badge/detail/v3/{id}`
- repeatable-badge earn-history endpoints

It normalizes the results into `user_badges` records and uploads them with `POST {apiBase}/sync`, authenticated via a per-user **GarminBadges API key** (obtained from the site's dashboard, not a Garmin credential) as a bearer token. It also calls `GET /api/sync/whoami` and `GET /api/badges` for reconciliation. Default API base is `https://api.garminbadges.com/api`.

## Distribution

No Dockerfile. `.github/workflows/release.yml` triggers on push to `main` (or manual dispatch): auto-increments the semver patch in `manifest.json`, commits the bump back to `main`, zips the extension, and publishes it as a GitHub Release asset. The [core `garminbadges` repo's](core-backend.md) own CI then fetches that release zip and serves it from the main site's `/tools/` download page — so a new extension release flows through to the website automatically.

## Relationship to the rest of the ecosystem

- Talks only to Garmin's internal endpoints and the [core backend's](core-backend.md) `POST /api/sync` / `GET /api/sync/whoami` / `GET /api/badges` API.
- No direct code relationship to `garminbadges-android` or `garminbadges-watch`, but `garminbadges-android` is explicitly maintained as a "Java port" of this extension's `sync.js` logic — the two are meant to be kept behaviorally in sync when Garmin's internal API changes.
