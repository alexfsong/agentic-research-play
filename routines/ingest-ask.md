---
name: ingest-ask
description: Cloud research+ingest path for the PWA "Ask" tab. Given a user question, run a web search, fetch the top results as Markdown, POST each to /ingest tagged source=ask_fill, then POST a final summary to /ask_callback so the webhook can run /synthesize2 and resolve the PWA's polling loop. Stateless on the corpus side (dedupes on normalized URL).
tools: [Bash, WebFetch, WebSearch]
model: claude-sonnet-4-6
---

# ingest-ask

The cloud-routine half of the new "Ask" pipeline (ODR replacement). Fetches
fresh material on demand so the subsequent `/synthesize2` call lands against an
enriched corpus, then notifies the webhook so the PWA can render a cited
answer.

This routine is the **cloud quota** path. Mirrored locally as the
`.claude/skills/ingest-ask.md` skill, which is invoked on the VPS via
`claude -p "/ingest-ask <json>"` when the cloud routine pool is exhausted. Both
paths post to the same webhook contract — the only thing that changes is who
is running the WebSearch + WebFetch.

## Inputs (from `/fire` payload `text` field)

The Routines `/fire` API delivers a single `text` string. The caller (the
Hetzner webhook's `/ask` endpoint) packs the parameters as JSON inside that
string. **First step of the routine: parse `text` as JSON** into the fields
below. If parsing fails, return a callback with `errors=[{"message":"text not JSON"}]` and stop.

- `run_id` (string, **required**) — opaque ID minted by the webhook (`ask_<hex>`). Echoed in the callback.
- `question` (string, **required**) — the user's question. Used as the web-search query.
- `thread_id` (string, optional) — ask-thread ID if the question is a continuation. Stored in metadata so the corpus can attribute later.
- `urls` (string[], optional) — skip search, fetch these directly.
- `max_fetches` (int, optional, default **10**) — cap on results fetched + ingested.
- `topic` (string, optional) — free-form tag stored in metadata.

## Environment
- `WEBHOOK_URL`, `WEBHOOK_API_KEY`.

Read from routine secrets. Fail fast if missing.

## Algorithm

1. Parse `text` as JSON. Extract fields above.
2. Validate. If `question` empty AND `urls` empty → POST callback with `errors=[{"message":"question or urls required"}]` and stop.
3. Resolve the URL list:
   1. If `urls` non-empty → use verbatim (cap at `max_fetches`).
   2. Else → `WebSearch question`. Take the top `max_fetches` result URLs.
4. For each URL:
   - Normalize: lowercase host, strip `#fragment`, remove `utm_*` / `fbclid` / `gclid` / `ref` query params.
   - `WebFetch` asking for Markdown. Extract document title from `<title>` or `<h1>`.
   - Skip with reason `"body < 300 chars"` if too short (paywall / failed fetch / low-content page).
5. POST each successful fetch to `/ingest`.
6. After the loop (success OR partial failure), POST the aggregate summary to `/ask_callback` (see below). Send it even on full failure.

## /ingest payload

```json
{
  "source": "ask_fill",
  "title": "<extracted title>",
  "content": "<markdown body>",
  "metadata": {
    "type": "ask_fill",
    "topic": "<provided topic or empty string>",
    "question": "<original question>",
    "thread_id": "<thread_id or empty string>",
    "url": "<normalized>",
    "fetched_at": "<now UTC ISO>"
  },
  "dedupe_key": "ask_fill:<normalized url>"
}
```

No `chunk_size` override.

## /ask_callback payload (final step — REQUIRED)

This is what unblocks the PWA's polling loop. Send it even on full failure. The
webhook reads the callback, then runs `/synthesize2` against the (now enriched)
corpus, stores the cited answer in the run state, and PWA's poll picks it up.

```bash
curl -sS -X POST "$WEBHOOK_URL/ask_callback" \
  -H "Authorization: Bearer $WEBHOOK_API_KEY" \
  -H "Content-Type: application/json" \
  -d @callback.json
```

```json
{
  "run_id": "<echoed from input>",
  "route": "cloud",
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
{ "run_id": "<echoed>", "route": "cloud", "status": "failed", "ingested": [], "skipped": [], "errors": [{"message": "..."}] }
```

## Error handling
- Same retry policy on `/ingest` as other routines: 1s, 2s, 4s on 429/5xx, then bail that URL.
- Per-URL failures go into `errors`; batch continues.
- The callback POST itself: retry 1s, 2s, 4s on 429/5xx. If callback fails after retries, log and exit — the PWA will time out gracefully.

## Tips for the caller (Hetzner webhook `/ask`)
- Pack inputs as JSON into the `/fire` `text` field. Don't try natural-language phrasing.
- Mint `run_id` server-side (UUID, prefix `ask_`) before firing.
- After firing, server stores `{run_id: pending, route: cloud}`. The callback flips it to `complete` (or `failed`) with the result body, then triggers `/synthesize2` server-side.
- On routine pool 429 / quota exhaust → don't fail; spawn the local skill instead (`subprocess claude -p "/ingest-ask <json>"`) and set `route=local` in run state.

## Don't
- Don't use this for bulk seeding — that's `ingest-arxiv` / `ingest-news` / `ingest-url`. Keep `ask_fill` reserved for Ask-tab queries so the source tag stays meaningful.
- Don't widen the search beyond `max_fetches` — the user is waiting on a spinner.
- Don't run synthesis here. Synthesis happens server-side via `/synthesize2` after callback.
- Don't skip the callback. Without it the PWA polls forever.
