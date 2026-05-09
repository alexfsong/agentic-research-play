---
name: ingest-ask
description: Local-subscription fallback for the PWA "Ask" tab. Mirror of the cloud `ingest-ask` routine. Given a JSON blob with question + run_id, run a web search, fetch the top results as Markdown, POST each to /ingest tagged source=ask_fill, draft a cited Markdown answer from those bodies, then POST result + answer to /ask_callback. Invoked headlessly via `claude -p "/ingest-ask <json>"` on the Hetzner VPS when the cloud routine pool is exhausted. Stateless on the corpus side (dedupes on normalized URL).
---

You are running `/ingest-ask`. Cloud-routine fallback under the user's Claude
Pro/Max subscription. Identical contract to `routines/ingest-ask.md` — same
`/ingest` payload, same `/ask_callback` callback (including `answer_md` +
`citations`). The only difference is who runs the WebSearch + WebFetch +
answer drafting (you, locally) and the `route` field in the callback (`local`
instead of `cloud`).

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
HISTORY=$(echo "$PAYLOAD" | jq -c '.history // []')   # JSON array of {q,a}, oldest→newest, may be empty
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
- Else if `HISTORY` non-empty (length > 0) → reformulate `QUESTION` into a self-contained search query (resolve pronouns / "it" / "that" / "the same" against prior turns), then `WebSearch <reformulated>`. Take the top `MAX` result URLs. Example: prior `q="What is FSRS?"`, current `QUESTION="how does it compare to SM-2?"` → search `"FSRS vs SM-2 spaced repetition algorithm"`.
- Else → `WebSearch QUESTION` verbatim. Take the top `MAX` result URLs.

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

### 5. Draft cited answer

From the markdown bodies you fetched in step 3, draft a Markdown answer to
`QUESTION`. The webhook stores this verbatim — what you write here is what
the user sees in the PWA. No server-side LLM rewrite.

- **Continuation context**: if `HISTORY` is non-empty, treat it as the
  conversation so far. Resolve anaphora ("it", "that", "the same") against
  prior turns. Don't repeat what was already said in `HISTORY[*].a` — pick
  up where it left off. Prior answers are context, not evidence — don't
  cite them.
- **Source set**: only successfully-fetched URLs from this run (the
  `ingested` array). No skipped/errored URLs. No model-prior facts. No
  remembered URLs. No prior turns.
- **Length**: 150–400 words. Tight. No "based on the sources" preamble.
- **Citations**: bracketed numeric footnotes inline, e.g. `Foo [1]. Quux
  [2][3].` Number = 1-based index into the `citations` array below. Every
  non-trivial claim cites at least one source. If sources disagree, say so
  and cite both. If sources don't answer the question, say that — don't
  fabricate.
- **Format**: plain Markdown. Headers OK. No HTML.
- **Citations array**: parallel array, entry `i` corresponds to footnote
  `[i+1]`. Each entry: `{"n": <1-based>, "title": "<page title>", "url":
  "<normalized url>"}`.

Skip this step if `QUESTION` is empty (URL-only ingest) or `ingested` is
empty. In those cases use `answer_md=""` and `citations=[]` in the callback.

### 6. Callback

After the loop (success OR partial failure), POST the aggregate to
`/ask_callback`. **Always send this.** Without it the PWA polls forever.

```json
{
  "run_id": "<RUN_ID>",
  "route": "local",
  "status": "complete",
  "ingested": [...],
  "skipped": [...],
  "errors": [...],
  "answer_md": "Foo [1]. Quux [2][3].\n\n## Caveats\nSources disagree on X [1][2].",
  "citations": [
    { "n": 1, "title": "<page title>", "url": "<normalized url>" },
    { "n": 2, "title": "<page title>", "url": "<normalized url>" },
    { "n": 3, "title": "<page title>", "url": "<normalized url>" }
  ]
}
```

On routine-level crash (bad input, search totally failed) — empty
`answer_md` + `citations` is fine:

```json
{ "run_id": "<RUN_ID>", "route": "local", "status": "failed", "ingested": [], "skipped": [], "errors": [{"message": "..."}], "answer_md": "", "citations": [] }
```

Callback retry policy: 1s/2s/4s on 429/5xx. Log and exit if all retries fail.

### 7. Emit

Print exactly two lines to stdout, then stop:

```
run_id=<RUN_ID>
route=local
```

No chatter.

## Rules

- Tools allowed: `WebSearch`, `WebFetch`, `Bash`. Nothing else (no Edit, no Write, no Read of arbitrary paths). The webhook invokes you with `--allowedTools "WebSearch,WebFetch,Bash"`; do not request elevated tools.
- Use `WebFetch` for content, not `curl`. Reserve `curl` for the `/ingest` and `/ask_callback` POSTs.
- Don't cite sources outside the `ingested` set for this run. No model-prior facts, no remembered URLs.
- Do not write files outside `/tmp`. Payload temp files go in `mktemp -d`.
- Do not log the bearer token.
- If WebSearch returns zero results → callback with `status="complete"`, empty `ingested`, `answer_md=""`, `citations=[]`, `errors=[{"message":"no search results"}]`.
