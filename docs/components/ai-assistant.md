# AI assistant (core backend feature)

> Part of the private `garminbadges` core repo — documented separately because it's a distinct subsystem, not because it's a separate deployable. See [`core-backend.md`](core-backend.md) for the app it lives inside.

## Purpose

A supporter-only feature set built on the Anthropic API, covering three capabilities:

1. **Badge Q&A chat** — a conversational assistant a supporter can ask about specific badges or how the site works.
2. **Analytics recap** — a short blurb summarizing a user's own earn stats (shown on their dashboard).
3. **Year in Review recap** — the same idea, scoped to a specific date range.

## How it works

- Built on the Anthropic API, using a small, fast Claude model chosen for cost/latency given how lightweight each request is.
- The badge Q&A feature is retrieval-augmented: relevant badges (and any relevant admin-curated notes — see below) are looked up based on the user's question and included as context for the model, rather than the model relying on built-in knowledge. Given the size of the badge catalogue, this uses ordinary keyword matching rather than a vector database.
- For a logged-in request, the model is also given a summary of that user's own progress, active challenges, and earn history — reusing the same data the site's own dashboard and analytics pages already compute, not a separate copy.
- The recap features summarize a user's precomputed stats into a short blurb, with some randomized variation in tone/angle so regenerating it doesn't just repeat the same phrasing.
- If the AI integration isn't configured, or the underlying API call fails, requests fail with a clear error rather than an unhandled crash.

## Access & API

The assistant is a supporter-only feature, gated separately from the site's general authentication, and rate-limited more tightly than typical endpoints since each request is a paid model call rather than a database read. It's exposed through a small set of authenticated endpoints: one for the Q&A chat, and one each for the two recap variants.

## Admin knowledge base

Admins can add free-text notes to help the assistant answer things that don't belong to any single badge — site-wide rules, Garmin quirks, recurring FAQs.

- **Where**: a dedicated tab in the Angular admin panel, with a simple list UI to add, edit, and delete entries (a title and a body of text each).
- **API**: a small admin-only set of endpoints to list, create, update, and delete entries, alongside the rest of the site's admin API.
- **How it takes effect**: entries go live immediately — there's no publish or reindex step. A given note is only pulled into the model's context when it's relevant to what the user asked, so in practice an admin influences relevance mainly through how they word the title and content rather than through any explicit tagging system.
