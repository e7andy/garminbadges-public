# Android sync app (`garminbadges-android`)

Repository: [`e7andy/garminbadges-android`](https://github.com/e7andy/garminbadges-android)

> **Highlight**: this isn't a webview wrapper. It implements Garmin's SSO login flow end-to-end — including MFA — and the OAuth2 token exchange, natively in Java.

## Purpose

A native Android client that syncs earned badges, repeatable-badge history, active/virtual challenges, and available-badge progress from a user's Garmin Connect account to [garminbadges.com](https://garminbadges.com) — the Android equivalent of the [`garminbadges-updater`](updater-extension.md) Chrome extension, but without needing a browser. The repo's own docs describe it as a Java port of that extension's sync logic, explicitly maintained to stay in sync with it when Garmin's API behavior changes.

## Tech stack

- **Language**: Java 11 (not Kotlin), single `:app` Gradle module.
- **UI**: Traditional XML/View-based UI with a Material3 theme — no Jetpack Compose.
- **SDK**: `minSdk` 26 (Android 8.0), `targetSdk`/`compileSdk` 36.
- **Networking**: OkHttp 4.12.0 for all HTTP calls; `org.json` for JSON (no Retrofit, no Gson/Moshi).
- **Permissions**: `INTERNET` only.
- No dependency injection framework (no Hilt/Dagger) and no local database (no Room) — this is a thin, stateless sync client, not a full offline-first app.

## Architecture

A small, flat, activity-centric codebase — five production classes:

| Class | Role |
|---|---|
| `AuthActivity` | Native login UI, delegates to `GarminAuthClient`. |
| `GarminAuthClient` | Programmatic Garmin SSO login (including MFA) and OAuth2 token exchange. |
| `GarminApiClient` | OkHttp wrapper for `connectapi.garmin.com` calls, bearer-authenticated. |
| `GarminBadgesApiClient` | OkHttp wrapper for `api.garminbadges.com/api` — `/badges` and `/sync`. |
| `SyncManager` | Orchestrates a full sync run using an internal 8-thread pool, reporting progress via callback. |
| `MainActivity` | Persists sync state in `SharedPreferences`; posts UI updates on the main looper. |

## How syncing works

The app performs a genuine Garmin SSO login itself — including MFA — rather than riding on an existing browser session:

1. **SSO login**: `sso.garmin.com/sso/embed`, `/sso/signin`, and (if needed) `/sso/verifyMFA/loginEnterMfaCode`.
2. **OAuth token exchange**: `diauth.garmin.com/di-oauth2-service/oauth/token`, using Android-specific client IDs (e.g. `GARMIN_CONNECT_MOBILE_ANDROID_DI_2025Q2`).
3. **Data read**: `connectapi.garmin.com` — `/userprofile-service/socialProfile`, `/badge-service/badge/earned`, `/badgechallenge-service/...`, `/badge-service/badge/detail/v3/{id}`, and related endpoints.
4. **Upload**: `GET https://api.garminbadges.com/api/badges` (catalogue reconciliation) then `POST https://api.garminbadges.com/api/sync` with the assembled `user_badges` array and `garmin_username`, authenticated by the user's GarminBadges API key as a bearer token — the same contract used by the Chrome extension.

## Distribution

- `.github/workflows/build.yml` builds a debug APK on every push/PR, and builds + signs a release AAB on push to `main` when keystore secrets (`KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS`, `KEY_PASSWORD`) are configured.
- `PUBLISHING.md` is a full manual runbook for Play Store release: keystore setup, store listing copy, data-safety declarations, and track promotion (internal → closed → open → production). No Fastlane.

## Relationship to the rest of the ecosystem

- Explicitly paired with [`garminbadges-updater`](updater-extension.md) — described in-repo as a "Java port" of its sync logic, with an instruction to keep both in sync as Garmin's internal API evolves.
- Talks to the [core backend](core-backend.md) only via `api.garminbadges.com` (`/badges`, `/sync`).
- No reference to `garminbadges-watch` anywhere in the codebase — the two are unrelated at the code level.
