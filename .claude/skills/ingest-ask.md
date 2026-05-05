---
name: ingest-ask
description: Local-subscription fallback for the PWA "Ask" tab. Mirror of the cloud `ingest-ask` routine. Given a JSON blob with question + run_id, run a web search, fetch the top results as Markdown, POST each to /ingest tagged source=ask_fill, then POST a final summary to /ask_callback. Invoked headlessly via `claude -p "/ingest-ask <json>"` on the Hetzner VPS when the cloud routine pool is exhausted. Stateless on the corpus side (dedupes on normalized URL).
---

You are running `/ingest-ask`. Cloud-routine fallback under the user's Claude
Pro/Max subscription. Identical contract to `routines/ingest-ask.md` — same
`/ingest` payload, same `/ask_callback` callback. The only difference is who
runs the WebSearch + WebFetch (you, locally) and the `route` field in the
callback (`local` instead of `cloud`).

## Arguments

`$ARGUMENTS` is a single JSON blob (passed verbatim from the webhook's
`subprocess claude -p "/ingest-ask <json>"` invocation). Parse it with `jq`:

```bash
PAYLOAD="$ARGUMENTS"
RUN_ID=$(echo "$PAYLOAD" | jq -r '.run_id')
QUESTION=$(echo "$PAYLOAD" | jq -r '.question')
THREAD_ID=$(echo "$PAYLOAD" | jq -r '.thread_id // ""')
MAX=$(echo "$PAYLOAD" | jq -r '.max_fetches // 10')
TOPIC=$(echo "$PAYLOAD" | jq -r '.topic // ""')
URLS=$(echo "$PAYLOAD" | jq -r '.urls // [] | .[]')
```

If JSON parse fails or `run_id` empty → POST callback with `errors=[{"message":"bad payload"}]` and stop.

## Environment

- `WEBHOOK_URL` — base URL of the research webhook (e.g. `http://127.0.0.1:8000` when invoked on the VPS).
- `WEBHOOK_API_KEY` — bearer token. Read from env, fail fast if missing.

Both injected by the webhook's subprocess call. Do not embed.

## Flow

### 1. Validate

If `QUESTION` empty AND `URLS` empty → POST callback with `errors=[{"message":"question or urls required"}]` and stop.

### 2. Resolve URLs

- If `URLS` non-empty → use verbatim (cap at `MAX`).
- Else → `WebSearch QUESTION`. Take the top `MAX` result URLs.

### 3. Fetch loop

For each URL:

- Normalize: lowercase host, strip `#fragment`, remove `utm_*` / `fbclid` / `gclid` / `ref` query params.
- `WebFetch` asking for Markdown. Extract document title from `<title>` or `<h1>`.
- Skip with reason `"body < 300 chars"` if too short.

### 4. POST each successful fetch to `/ingest`

```bash
curl -sS -X POST "$WEBHOOK_URL/ingest" \
  -H "Authorization: Bearer $WEBHOOK_API_KEY" \
  -H "Content-Type: application/json" \
  -d @payload.json
```

Payload:

```json
{
  "source": "ask_fill",
  "title": "<extracted title>",
  "content": "<markdown body>",
  "metadata": {
    "type": "ask_fill",
    "topic": "<TOPIC or empty>",
    "question": "<QUESTION>",
    "thread_id": "<THREAD_ID or empty>",
    "url": "<normalized>",
    "fetched_at": "<now UTC ISO>"
  },
  "dedupe_key": "ask_fill:<normalized url>"
}
```

Retry 1s/2s/4s on 429/5xx, then bail that URL. Track per-URL outcomes in
`ingested`, `skipped`, `errors` arrays.

### 5. Callback

After the loop (success OR partial failure), POST the aggregate to
`/ask_callback`. **Always send this.** Without it the PWA polls forever.

```json
{
  "run_id": "<RUN_ID>",
  "route": "local",
  "status": "complete",
  "ingested": [...],
  "skipped": [...],
  "errors": [...]
}
```

On routine-level crash (bad input, search totally failed):

```json
{ "run_id": "<RUN_ID>", "route": "local", "status": "failed", "ingested": [], "skipped": [], "errors": [{"message": "..."}] }
```

Callback retry policy: 1s/2s/4s on 429/5xx. Log and exit if all retries fail.

### 6. Emit

Print exactly two lines to stdout, then stop:

```
run_id=<RUN_ID>
route=local
```

No chatter.

## Rules

- Tools allowed: `WebSearch`, `WebFetch`, `Bash`. Nothing else (no Edit, no Write, no Read of arbitrary paths). The webhook invokes you with `--allowedTools "WebSearch,WebFetch,Bash"`; do not request elevated tools.
- Use `WebFetch` for content, not `curl`. Reserve `curl` for the `/ingest` and `/ask_callback` POSTs.
- Do not run synthesis here. Server runs `/synthesize2` after callback.
- Do not write files outside `/tmp`. Payload temp files go in `mktemp -d`.
- Do not log the bearer token.
- If WebSearch returns zero results → callback with `status="complete"`, empty `ingested`, `errors=[{"message":"no search results"}]`. The webhook's synthesis pass will still run against the existing corpus.
