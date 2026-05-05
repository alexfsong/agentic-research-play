---
name: ingest-research
description: Gap-fill for course lessons. Given a learner's question, run a web search, fetch the top results as Markdown, POST each to /ingest tagged source=research_fill with lesson/course context, then POST a final summary to /research_callback so the PWA can stop polling. Stateless on the corpus side (dedupes on normalized URL).
tools: [Bash, WebFetch, WebSearch]
model: claude-sonnet-4-6
---

# ingest-research

Corpus gap-fill triggered from the PWA's "Research this" button on a lesson
follow-up. Fetches fresh material on demand so the subsequent `/ask` call lands
against an enriched corpus, then notifies the webhook so the PWA can resolve
its polling loop.

## Inputs (from `/fire` payload `text` field)

The Routines `/fire` API delivers a single `text` string. The caller (the
Hetzner webhook's `/research_fire` endpoint) packs the parameters as JSON
inside that string. **First step of the routine: parse `text` as JSON** into
the fields below. If parsing fails, return a callback with `errors=[{"message":"text not JSON"}]` and stop.

- `run_id` (string, **required**) — opaque ID minted by the webhook. Echoed in the callback so the server can correlate.
- `question` (string, **required**) — the learner's follow-up question. Used as the web-search query.
- `lesson_id` (string, **required**) — stored in metadata so the course layer can attribute.
- `course_id` (string, **required**) — stored in metadata.
- `urls` (string[], optional) — skip search, fetch these directly. Same shape as `ingest-url`.
- `max_fetches` (int, optional, default **5**) — cap on results fetched + ingested.
- `topic` (string, optional) — free-form tag (derived from the lesson objective by the caller). Stored in metadata.

## Environment
- `WEBHOOK_URL`, `WEBHOOK_API_KEY`.

## Algorithm

1. Parse `text` as JSON. Extract fields above.
2. Validate. If `question` empty AND `urls` empty → POST callback with `errors=[{"message":"question or urls required"}]` and stop.
3. Resolve the URL list:
   1. If `urls` non-empty → use verbatim (cap at `max_fetches`).
   2. Else → `WebSearch question`. Take the top `max_fetches` result URLs.
4. For each URL:
   - Normalize (same rules as ingest-news/ingest-url: lowercase host, strip `#fragment`, remove `utm_*` / `fbclid` / `gclid` / `ref` query params).
   - `WebFetch` asking for Markdown. Extract document title from `<title>` or `<h1>`.
   - Skip with reason `"body < 300 chars"` if too short (paywall / failed fetch / low-content page).
5. POST each successful fetch to `/ingest`.
6. After the loop (success OR partial failure), POST the aggregate summary to `/research_callback` (see below).

## /ingest payload

```json
{
  "source": "research_fill",
  "title": "<extracted title>",
  "content": "<markdown body>",
  "metadata": {
    "type": "research_fill",
    "topic": "<provided topic or empty string>",
    "question": "<original question>",
    "course_id": "<course_id>",
    "lesson_id": "<lesson_id>",
    "url": "<normalized>",
    "fetched_at": "<now UTC ISO>"
  },
  "dedupe_key": "research_fill:<normalized url>"
}
```

No `chunk_size` override.

## /research_callback payload (final step — REQUIRED)

This is what unblocks the PWA's polling loop. Send it even on full failure.

```bash
curl -sS -X POST "$WEBHOOK_URL/research_callback" \
  -H "Authorization: Bearer $WEBHOOK_API_KEY" \
  -H "Content-Type: application/json" \
  -d @callback.json
```

```json
{
  "run_id": "<echoed from input>",
  "status": "complete",
  "ingested": [
    { "doc_id": "ingest_...", "title": "<title>", "url": "<normalized>", "existed": false, "nodes": 3 }
  ],
  "skipped": [
    { "reason": "body < 300 chars", "url": "<normalized>" }
  ],
  "errors": [
    { "url": "<normalized>", "status": 500, "message": "..." }
  ]
}
```

If the routine itself crashes early (bad input, search failure), still post:

```json
{ "run_id": "<echoed>", "status": "failed", "ingested": [], "skipped": [], "errors": [{"message": "..."}] }
```

## Error handling
- Same retry policy on `/ingest` as other routines: 1s, 2s, 4s on 429/5xx, then bail that URL.
- Per-URL failures go into `errors`; batch continues.
- The callback POST itself: retry 1s, 2s, 4s on 429/5xx. If callback fails after retries, log and exit — the PWA will time out gracefully.

## Tips for the caller (Hetzner webhook `/research_fire`)
- Pack inputs as JSON into the `/fire` `text` field. Don't try natural-language phrasing.
- Mint `run_id` server-side (UUID) before firing.
- After firing, server stores `{run_id: pending}`. The callback flips it to `complete` (or `failed`) with the result body.

## Don't
- Don't use this for bulk seeding — that's what `ingest-arxiv` / `ingest-news` / `ingest-url` are for. Keep `research_fill` reserved for gap-fill so the tag stays meaningful.
- Don't widen the search beyond `max_fetches` — the learner is waiting on a spinner.
- Don't re-ask the lesson question from the routine. The PWA runs `/ask` separately once the callback fires.
- Don't skip the callback. Without it the PWA polls forever.
