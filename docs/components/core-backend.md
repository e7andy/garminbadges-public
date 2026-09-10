# Core backend + web app (`garminbadges`)

> Source repository is **private**. This page documents its architecture without reproducing source code.

## Purpose

The core of [garminbadges.com](https://garminbadges.com) ("Garmin Badge Database"). A community site where users track earned and in-progress Garmin Connect badges, follow challenge progress, browse a badge catalogue, see leaderboards and year-in-review analytics, follow other users, read a blog, and — for paying supporters — use an AI badge assistant.

Critically, this app is a **passive data store and presentation layer**. It does not talk to Garmin's servers itself in normal operation; it only accepts data pushed to it by the ecosystem's ingestion clients (see [`../architecture.md`](../architecture.md)).

## Tech stack

- **Backend**: PHP 8.4, Laravel 12, MySQL/MariaDB. Queue, cache, and session drivers all backed by the database (no Redis).
- **Auth**: Laravel Sanctum (bearer tokens for the SPA), Laravel Socialite (Google OAuth).
- **Frontend**: Angular 18 SPA (standalone components), Angular Material, `dompurify`, `marked`/`turndown` for markdown/blog content. Built as a static bundle via `ng build`.
- **Other libraries**: `intervention/image` (avatar processing), `stripe/stripe-php`, `anthropic-ai/sdk`.
- **Companion scripts** (shipped inside this repo, not separate apps): `garminbadges-admin.py` (admin bulk import) and `garminbadges-updater.py` (end-user sync CLI, distributed from the site's `/tools/` page), both built on the third-party `garth-ng` Garmin client library.

## Structure

A monorepo with two independently-built halves:

- `backend/` — Laravel API. Key directories: `app/Http/Controllers/Api` (~30 REST controllers, including an `Admin/` subset), `app/Models`, `app/Services` (e.g. `AiAssistantService`, `TranslationDiffService`, `UpcomingNotificationsService`), `app/Console/Commands`, `database/migrations`, `routes/api.php`.
- `frontend/` — Angular SPA, built to a static bundle. `frontend/public/tools/` is where the downloadable Python scripts and the built Chrome extension `.zip` (fetched from `garminbadges-updater`'s releases at build time) are served from.

## Data model

See the [Core data model summary](../architecture.md#core-data-model-summary) in the architecture doc — `User`, `Badge` + lookup tables, `UserBadge`, `Country`, `SocialAccount`, social/blog tables, `AiKnowledgeEntry`.

## API surface

The full REST API backs the Angular SPA. The subset consumed by external ecosystem clients:

- `POST /api/sync` — accept a batch of `user_badges` from any sync client, authenticated by per-user `api_key` bearer token.
- `GET /api/sync/whoami` — confirm which user an API key belongs to.
- `GET /api/badges` — the badge catalogue, used by clients to reconcile badge IDs before uploading.
- `GET /api/watch` — a purpose-built, pre-shaped read endpoint for the Connect IQ watch app (upcoming badges, challenges ending soon, in-progress challenges ranked by how far behind schedule they are).

## Background jobs & scheduling

A cron-driven Laravel scheduler (`php artisan schedule:run`, run every minute) drives periodic jobs: `translations:check` every 6 hours, `supporters:expire` daily. A database-backed queue processes jobs such as `PublishBlogPostNotifications`.

## External integrations

- **Stripe** — supporter donations/subscriptions (checkout sessions, billing portal, signature-verified webhook at `POST /api/support/webhook`).
- **Anthropic API** — powers a supporter-only AI assistant (pinned to a Haiku-class model).
- **Google OAuth** (Socialite) — social login, callback at `/api/auth/google/callback`.
- **Mailjet** — transactional email (SMTP relay).
- **Google Analytics 4** — frontend analytics.

## Deployment

No containers. CI (`.github/workflows/build.yml`) builds the Angular frontend and the Laravel backend (production dependencies only) on every push to `main` or a version tag, and publishes versioned `frontend-X.Y.Z.zip` / `backend-X.Y.Z.zip` artifacts as a GitHub Release. A deploy script (`backend/scripts/deploy.sh`) rsyncs those artifacts to a traditional Apache + PHP-FPM server and runs migrations.

The same CI workflow also downloads the latest release of `garminbadges-updater` (the Chrome extension) via the GitHub CLI and bundles it into the frontend's `/tools/` download page — the one build-time coupling point between this repo and a sibling repo.

## Relationship to the rest of the ecosystem

This repo is the hub every other component talks to:

- **`garminbadges-updater`** (Chrome extension) and **`garminbadges-android`** both authenticate independently against Garmin, then push data in via `POST /api/sync` using a per-user API key issued by this backend.
- **`garminbadges-watch`** only reads, via the dedicated `GET /api/watch` endpoint.
- The Chrome extension's build artifact is pulled into this repo's own frontend at CI time for distribution.
