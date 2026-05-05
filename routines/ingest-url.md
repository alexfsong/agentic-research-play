---
name: ingest-url
description: Ingest an ad-hoc list of URLs the user has dropped in. Fetch each as Markdown and POST to /ingest with a user-supplied topic tag.
tools: [Bash, WebFetch]
model: claude-sonnet-4-6
---

# ingest-url

## Inputs (from `/fire` payload)
- `urls` (string[], **required**) — 1..N URLs.
- `topic` (string, optional) — free-form tag stored in metadata (not used for filtering yet, but the course layer will read it later).
- `note` (string, optional) — free-form context (e.g. "follow-up reading for RAG eval lesson").

## Environment
- `WEBHOOK_URL`, `WEBHOOK_API_KEY`.

## Algorithm

1. If `urls` empty → return `{ "errors": [{"message": "urls required"}] }`.
2. For each URL:
   - Normalize (same rules as ingest-news: lowercase host, strip `#fragment`, remove `utm_*` / `fbclid` / `gclid` / `ref` query params).
   - `WebFetch` asking for Markdown. Extract the document title from `<title>` or `<h1>`.
   - Skip with reason `"body < 300 chars"` if too short (paywall / failed fetch).
3. POST each to `/ingest`.

## /ingest payload

```json
{
  "source": "adhoc",
  "title": "<extracted title>",
  "content": "<markdown body>",
  "metadata": {
    "type": "adhoc",
    "topic": "<provided topic or empty string>",
    "note": "<provided note or empty string>",
    "url": "<normalized>",
    "fetched_at": "<now UTC ISO>"
  },
  "dedupe_key": "adhoc:<normalized url>"
}
```

No `chunk_size` override.

## Error handling
- Same retry policy as the other routines (1s, 2s, 4s on 429/5xx, then bail per-URL).

## Output JSON

```json
{
  "ingested": [
    { "doc_id": "ingest_...", "title": "<title>", "url": "<normalized>", "existed": false, "nodes": 2 }
  ],
  "skipped": [
    { "reason": "body < 300 chars", "url": "<normalized>" }
  ],
  "errors": [
    { "url": "<normalized>", "status": 500, "message": "..." }
  ]
}
```

## Tips for the caller
- Dropping the same URL twice is a no-op on the corpus (dedupe on normalized URL). The second call's `existed: true` on each entry confirms it.
- Use `note` to thread context — e.g. `"for lesson lesson_abc123 on RAG eval"`. The course layer's gap-fill flow uses its own `source=research_fill` tag, so adhoc stays clean.
