# AI assistant (core backend feature)

> Part of the private `garminbadges` core repo — documented separately because it's a distinct subsystem, not because it's a separate deployable. See [`core-backend.md`](core-backend.md) for the app it lives inside.

## Purpose

A supporter-only feature set built on the Anthropic API, covering three distinct capabilities:

1. **Badge Q&A chat** — a conversational assistant a supporter can ask about specific badges or how the site works.
2. **Analytics recap** — a short, differently-toned blurb generated from a user's own earn stats (shown on their dashboard).
3. **Year in Review recap** — the same idea, scoped to a specific date range on the Year in Review page.

All three are implemented in one class, `AiAssistantService` (`backend/app/Services/AiAssistantService.php`), and exposed through `AiController` (`backend/app/Http/Controllers/Api/AiController.php`).

## How it's implemented

- Uses the official `anthropic-ai/sdk` PHP client (`Anthropic\Client`), constructed with `config('services.anthropic.api_key')` and calling `config('services.anthropic.model')` — currently pinned to a Haiku-class model (`claude-haiku-4-5`), not left to "latest".
- `AiAssistantService::isConfigured()` just checks the API key is set, so the feature degrades to a clean 503 rather than a crash if it's ever unconfigured.
- Every model call goes through one private `complete()` method that builds a `system` prompt, sends the conversation `messages` array, and extracts the first `text` content block from the response. API failures are caught narrowly (`RateLimitException`, `APIStatusException`, `APIConnectionException`) and turned into a plain user-facing message rather than leaking SDK exception details.

### Retrieval: keyword search, not a vector store

Badge Q&A is retrieval-augmented, but deliberately **not** via embeddings/vector search — a code comment explains the badge catalogue is "small and structured enough that a keyword search beats standing up a vector store." The retrieval logic:

1. Take the last two user turns of the conversation (not just the newest one, so a follow-up like "how many points is it worth?" still has context), split into words, and drop a curated stopword list (generic English words plus site-specific noise like "badge"/"badges"/"earn").
2. Run those keywords through a SQL `LIKE`-based relevance search: for each candidate row, score it by *how many* of the keywords match (via a `CASE WHEN column LIKE ? THEN 1 ELSE 0 END` sum), then order by that score descending. This applies to two different tables:
   - **Badges** (`matchBadges`): search `name` first; only fall back to searching `description` if nothing matched by name (names are a more precise signal — descriptions often contain generic activity words). If still nothing matches, fall back to 10 random active badges so the model always has *something* to reason about.
   - **Admin knowledge entries** (`matchKnowledge`, see below): same ranked search over `title` then `content`, but with **no random fallback** — if nothing matches, nothing is added; these notes are meant to be supplementary, not required.

### Context assembled into the system prompt

For each `askAboutBadges` call, the system prompt is built from several pieces, in order:

1. **Static site info** (`SITE_INFO` constant) — a fixed paragraph describing what the site is, how syncing works (extension/Android/Python, per-user API key), and what supporter perks exist. Always included, regardless of keyword matching, so meta-questions like "how do I sync my badges?" don't depend on hitting the right keywords.
2. **Matched badges** — up to 15, each rendered with category, points, difficulty, target value (unit-aware, converted to the user's metric/statute preference), description, and any admin `notes` field on the badge itself.
3. **Matched admin knowledge entries**, if any (see next section).
4. If a user is asking (not an anonymous/unauthenticated call): their **personal progress** on each matched badge, their **Challenges page** summary (active challenges, ongoing trackers, badges completed this month), and their **analytics summary** (monthly earns/points, best month, current/longest streak, top categories) — all pulled from the same controller methods that power those pages' own APIs (`AnalyticsController::summaryForUser`, `UserChallengesController::summaryForUser`), reused here rather than reimplemented.
5. Current date/time in the user's own timezone and clock format, so the model has a correct reference point for relative-time reasoning instead of guessing.

The recap features (`generateRecap`, `generateYearInReviewRecap`) don't use the keyword-search retrieval at all — they call the analytics/year-in-review summary methods directly, serialize a fixed set of numbers to JSON, and ask the model to phrase them. A tone and an "angle to lean into if the data supports it" are randomly picked from small fixed lists each call, specifically so the recap doesn't read as the same template every time a user regenerates it.

## Where the data comes from — summary

| Context piece | Source |
|---|---|
| Site description | Hardcoded constant in `AiAssistantService` |
| Badge details | `badges` table, keyword-matched, unit-converted at request time |
| Admin knowledge notes | `ai_knowledge_entries` table, keyword-matched |
| User's own progress/challenges | Live query against `user_badges` + the same summary methods `AnalyticsController`/`UserChallengesController` use for their own API responses |
| User's analytics | `AnalyticsController::summaryForUser` |
| Recap stats | `AnalyticsController::summaryForUser` / `YearInReviewController::summaryForUser` |

Nothing here is precomputed or cached specifically for the AI feature — every call re-derives its context from the same live tables and services the rest of the site reads from.

## API surface

All under `auth:sanctum` + a dedicated `throttle:20,1` (20 requests/minute) — tighter and separate from other authenticated routes, since each call is a paid LLM request rather than a DB read:

| Method & path | Controller action | Purpose |
|---|---|---|
| `POST /user/ai/ask` | `AiController::ask` | Badge Q&A chat turn |
| `GET /user/ai/recap` | `AiController::recap` | Dashboard analytics recap |
| `GET /user/year-in-review/recap` | `AiController::yearInReviewRecap` | Year in Review recap |

Every action independently checks `$user->is_supporter` (403 if not) and `$this->ai->isConfigured()` (503 if not) before calling the service. `ask` additionally validates that `messages` strictly alternates `user`/`assistant` starting with `user` — required by the Anthropic API — returning a 422 rather than letting a malformed request surface as a confusing 503 from a rejected API call.

## How admins add AI knowledge

Admin-curated notes exist for information that doesn't belong to any single badge record — site-wide rules, Garmin quirks, recurring FAQs — stored as simple `{title, content}` pairs in the `ai_knowledge_entries` table (`AiKnowledgeEntry` model, migration `2026_09_10_124605_create_ai_knowledge_entries_table`).

**Where**: the Angular admin panel (`/admin`) has an "AI Knowledge" tab (`frontend/src/app/features/admin/admin.component.ts`/`.html`), lazy-loaded the first time that tab is opened. It's a plain list UI: existing entries are listed by title, with inline add/edit/delete forms for title + content.

**API** (all under the `admin` route-group middleware — Sanctum token for the frontend, or the admin API key for the companion Python admin script — in `backend/app/Http/Controllers/Api/Admin/AdminAiKnowledgeController.php`):

| Method & path | Action |
|---|---|
| `GET /admin/ai-knowledge` | List all entries, ordered by title |
| `POST /admin/ai-knowledge` | Create an entry (`title` ≤255 chars, `content` ≤5000 chars) |
| `PATCH /admin/ai-knowledge/{id}` | Update an entry |
| `DELETE /admin/ai-knowledge/{id}` | Delete an entry |

**How it takes effect**: there's no publish/reindex step — an entry is live the moment it's saved. The next chat question whose keywords match its `title` or `content` will have that entry's text appended to the model's system prompt as an "Additional notes from the site admin" section. Because matching is a simple `LIKE`-based keyword search (see above), an admin effectively controls relevance by choice of wording in the title/content rather than any explicit tagging or category system.
