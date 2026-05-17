---
name: ingest-ask
description: Cloud research+ingest path for the PWA "Ask" tab. Given a user question, run a web search, fetch the top results as Markdown, POST each to /ingest tagged source=ask_fill, draft a cited Markdown answer from the fetched content, then POST result + answer to /ask_callback. Stateless on the corpus side (dedupes on normalized URL).
tools: [Bash, WebFetch, WebSearch]
model: claude-sonnet-4-6
---

# ingest-ask

The cloud-routine half of the new "Ask" pipeline (ODR replacement). Fetches
fresh material on demand, drafts a cited Markdown answer from that material,
and posts the answer + per-URL outcomes to the webhook so the PWA can render
the result. The webhook does **no** server-side LLM synthesis — the answer is
authored here and persisted verbatim.

This routine is the **cloud quota** path. Mirrored locally as the
`.claude/commands/ingest-ask.md` slash command, which is invoked on the VPS via
`claude -p "/ingest-ask <json>"` when the cloud routine pool is exhausted. Both
paths post to the same webhook contract — the only thing that changes is who
is running the WebSearch + WebFetch + answer drafting.

## Inputs (from `/fire` payload `text` field)

The Routines `/fire` API delivers a single `text` string. The caller (the
Hetzner webhook's `/ask` endpoint) packs the parameters as JSON inside that
string. **First step of the routine: parse `text` as JSON** into the fields
below. If parsing fails, return a callback with `errors=[{"message":"text not JSON"}]` and stop.

- `run_id` (string, **required**) — opaque ID minted by the webhook (`ask_<hex>`). Echoed in the callback.
- `question` (string, **required**) — the user's question. Seed for the web-search query (see "Conversation history" if continuation).
- `thread_id` (string, optional) — ask-thread ID if the question is a continuation. Stored in metadata so the corpus can attribute later.
- `history` (array, optional) — prior turns in the same thread, oldest→newest, each `{"q": "...", "a": "..."}` (answers truncated by webhook). Use for query reformulation + answer context. Empty for first turn of a thread.
- `urls` (string[], optional) — skip search, fetch these directly.
- `max_fetches` (int, optional, default **10**) — cap on results fetched + ingested.
- `topic` (string, optional) — free-form tag stored in metadata.
- `depth` (string, optional, default `"standard"`) — tier selector. `standard` = full single-pass pipeline (search + fetch + ingest + draft answer + callback). `deep` = **search + fetch + ingest only**; the webhook orchestrates synthesis and the final report. Do not draft an answer when `depth=deep`.
- `budget` (object, optional) — per-tier caps `{max_searches, max_fetches, max_tokens, max_iterations}`. Honor `max_fetches` if present (overrides the top-level `max_fetches` field). The other budgets are advisory for the routine; the orchestrator enforces iteration/token caps.
- `parent_context` (object, optional) — present only when the user branched a thread from a prior turn. Shape: `{"question": "<parent turn question>", "answer_excerpt": "<truncated parent answer markdown>", "quote": "<verbatim selection — only on selection-level branches>", "carried_sources": [{"n": <int>, "title": "...", "url": "..."}, ...]}`. The first turn of a branched thread; `history` will be empty. See "Branched-thread prompt branch" below for handling.

## Environment
- `WEBHOOK_URL`, `WEBHOOK_API_KEY`.

Read from routine secrets. Fail fast if missing.

## Algorithm

1. Parse `text` as JSON. Extract fields above. Default `depth` to `"standard"` if missing.
2. Validate. If `question` empty AND `urls` empty → POST callback with `errors=[{"message":"question or urls required"}]` and stop.
3. Resolve the URL list:
   1. If `urls` non-empty → use verbatim (cap at `max_fetches`).
   2. Else if `parent_context` present → reformulate `question` using the parent quote / question / answer excerpt as context (resolve "this", "that", "the same" against the parent), then `WebSearch <reformulated>`. Include any `parent_context.carried_sources[*].url` at the top of the URL list before web-search results (cap total at `max_fetches`) so citations can re-resolve them.
   3. Else if `history` non-empty → reformulate `question` into a self-contained search query (resolve pronouns / "it" / "that" / "the same" against the prior turns), then `WebSearch <reformulated>`. Take the top `max_fetches` result URLs. Example: prior turn `q="What is FSRS?"`, current `question="how does it compare to SM-2?"` → search `"FSRS vs SM-2 spaced repetition algorithm"`.
   4. Else → `WebSearch question` verbatim. Take the top `max_fetches` result URLs.
4. For each URL:
   - Normalize: lowercase host, strip `#fragment`, remove `utm_*` / `fbclid` / `gclid` / `ref` query params.
   - `WebFetch` asking for Markdown. Extract document title from `<title>` or `<h1>`.
   - Skip with reason `"body < 300 chars"` if too short (paywall / failed fetch / low-content page).
5. POST each successful fetch to `/ingest`.
6. **Branch on `depth`:**
   - `depth="standard"` (default) → draft the cited Markdown answer from the fetched bodies (see "Answer drafting" below). Skip if `question` is empty or all fetches failed.
   - `depth="deep"` → **skip answer drafting**. The webhook orchestrator runs the multi-round synthesizer and assembles the final sectioned report. Your job is corpus-growth only.
7. After the loop (success OR partial failure), POST the aggregate summary to `/ask_callback`. Send it even on full failure. For `depth=deep`, send empty `answer_md` + `citations` (the orchestrator will replace those with the final report).

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

## Branched-thread prompt branch

When `parent_context` is present (first turn of a branched thread), prepend
the following preamble to the model context **before** the question, instead
of (not in addition to) the regular history block — `history` will be empty
for a branched first turn:

```
This question is a follow-up on prior context.
Parent question: "<parent_context.question>"
Parent answer excerpt: <parent_context.answer_excerpt>
<if parent_context.quote>Highlighted passage from the parent answer: "<parent_context.quote>"</if>
Carried sources from the parent turn (cite as needed; they will be re-fetched
and ingested above): <list parent_context.carried_sources as "n. title — url">
```

Treat `parent_context.quote` (when present) as the focal hint: the user
selected that span specifically. Steer the search + answer toward expanding
on that passage rather than re-summarizing the parent answer wholesale.

`parent_context.carried_sources` is a hint to ensure the URL list (step 3.2)
re-fetches the same sources so per-section citations can map back to them
in the new turn's `citations` array. Don't cite them by their parent index;
the new `citations` array starts at `[1]` for this turn.

## Answer drafting

After the fetch loop, before the callback, draft a Markdown answer to
`question` grounded in the fetched bodies. The answer is what the PWA renders
verbatim — the webhook does no further LLM synthesis.

- **Continuation context**: if `history` is non-empty, treat it as the
  conversation so far. Resolve anaphora ("it", "that", "the same") against
  prior turns. Don't repeat what was already said in `history[*].a` — pick up
  where it left off. Don't cite prior answers as sources; they're context, not
  evidence.
- **Branched first turn**: when `parent_context` is present, the parent answer
  is context (not evidence) and not part of `history`. Open by extending the
  parent's framing (especially the highlighted `quote` if present) rather than
  re-stating it. Cite only your own ingested sources — `carried_sources` are
  hints for retrieval, not pre-resolved citations.
- **Source set**: only the URLs you successfully fetched in this run (the
  `ingested` array). Don't cite skipped/errored URLs. Don't cite anything
  outside the fetched set (no model-prior facts, no remembered URLs, no prior
  turns).
- **Length**: 150–400 words. Tight, direct. No "based on the sources" preamble.
- **Citation style**: bracketed numeric footnotes inline, e.g. `Foo bar [1].
  Quux baz [2][3].` Number = 1-based index into the `citations` array below.
- **Coverage**: every non-trivial claim cites at least one source. If sources
  disagree, say so and cite both. If sources don't answer the question, say
  that explicitly — don't fabricate.
- **Format**: plain Markdown. Headers OK. No HTML. No code blocks unless
  quoting code from a source.
- **Citations array**: build a parallel array where index `i` corresponds to
  footnote `[i+1]` in `answer_md`. Each entry: `{"n": <1-based>, "title":
  "<page title>", "url": "<normalized url>"}`.

Skip drafting if `question` is empty (URL-only ingest) or `ingested` is empty
after the loop. In those cases send `answer_md: ""` and `citations: []`.

## /ask_callback payload (final step — REQUIRED)

This is what unblocks the PWA's polling loop. Send it even on full failure.
The webhook stores `answer_md` + `citations` verbatim — what you draft is what
the user sees.

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
  ],
  "answer_md": "Foo bar [1]. Quux baz [2][3].\n\n## Caveats\nSources disagree on X [1][2].",
  "citations": [
    { "n": 1, "title": "<page title>", "url": "<normalized url>" },
    { "n": 2, "title": "<page title>", "url": "<normalized url>" },
    { "n": 3, "title": "<page title>", "url": "<normalized url>" }
  ]
}
```

If the routine itself crashes early (bad input, search failure), still post —
empty `answer_md` + `citations` is fine in that case:

```json
{ "run_id": "<echoed>", "route": "cloud", "status": "failed", "ingested": [], "skipped": [], "errors": [{"message": "..."}], "answer_md": "", "citations": [] }
```

## Error handling
- Same retry policy on `/ingest` as other routines: 1s, 2s, 4s on 429/5xx, then bail that URL.
- Per-URL failures go into `errors`; batch continues.
- The callback POST itself: retry 1s, 2s, 4s on 429/5xx. If callback fails after retries, log and exit — the PWA will time out gracefully.

## Tips for the caller (Hetzner webhook `/ask`)
- Pack inputs as JSON into the `/fire` `text` field. Don't try natural-language phrasing.
- Mint `run_id` server-side (UUID, prefix `ask_`) before firing.
- After firing, server stores `{run_id: pending, route: cloud}`. The callback flips it to `complete` (or `failed`) and persists `answer_md` + `citations` verbatim. No further server-side LLM call.
- On routine pool 429 / quota exhaust → don't fail; spawn the local skill instead (`subprocess claude -p "/ingest-ask <json>"`) and set `route=local` in run state.

## Don't
- Don't use this for bulk seeding — that's `ingest-arxiv` / `ingest-news` / `ingest-url`. Keep `ask_fill` reserved for Ask-tab queries so the source tag stays meaningful.
- Don't widen the search beyond `max_fetches` — the user is waiting on a spinner.
- Don't cite sources outside the `ingested` set for this run. No model-prior facts, no remembered URLs.
- Don't skip the callback. Without it the PWA polls forever.
