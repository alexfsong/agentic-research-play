---
name: ingest-arxiv
description: Pull recent arXiv abstracts on a topic (or from the default seed list) and POST each to the research webhook /ingest endpoint. Stateless — safe to re-fire (server dedupes via dedupe_key).
tools: [Bash, WebFetch, Read]
model: claude-sonnet-4-6
---

# ingest-arxiv

## Inputs (from `/fire` payload)
- `topic` (string, optional) — e.g. `"retrieval augmented generation"`. If omitted, iterate over the `arxiv.default_topics` array in `seeds.json`.
- `max` (int, optional, default **20**) — per-topic cap.
- `since` (ISO date, optional, default **yesterday UTC**) — earliest submittedDate.

## Environment
- `WEBHOOK_URL` — base URL of the research webhook (e.g. `https://research.example.com`).
- `WEBHOOK_API_KEY` — bearer token for `/ingest`.

Both are read from routine secrets. Fail fast if missing.

## Algorithm

1. Load `seeds.json` with the Read tool. Resolve the topic list (input `topic` OR `arxiv.default_topics`).
2. For each topic:
   - Query arXiv: `http://export.arxiv.org/api/query?search_query=all:<url-encoded topic>+AND+submittedDate:[<since>000000+TO+<since>235959]&max_results=<max>&sortBy=submittedDate&sortOrder=descending`
   - Parse the Atom response; for each `<entry>`:
     - `arxiv_id` = last path segment of `<id>` (e.g. `2405.12345v2`). Strip any `vN` only if present, keep the version in a separate field.
     - `title`, `summary` (the abstract), `published`, `<author><name>` list, primary `<category term="">`.
     - Skip if abstract length < 100 chars.
   - POST each to `/ingest` (see payload below). Honour the Error handling section.
3. Return the aggregate result JSON.

## /ingest payload

```json
{
  "source": "arxiv",
  "title": "<arxiv title>",
  "content": "<abstract>",
  "metadata": {
    "type": "arxiv",
    "topic": "<resolved topic>",
    "arxiv_id": "2405.12345v2",
    "version": "v2",
    "authors": "Alice Liu, Bob Chen",
    "category": "cs.IR",
    "url": "https://arxiv.org/abs/2405.12345",
    "published_at": "2026-04-21T00:00:00Z",
    "fetched_at": "<now UTC ISO>"
  },
  "dedupe_key": "arxiv:2405.12345v2",
  "chunk_size": 768
}
```

Notes:
- `chunk_size: 768` (papers have long sections; the server default 512 fragments them).
- `dedupe_key` includes the version so `v1` → `v2` re-reads land as an upsert, not a duplicate row.

## POST invocation (Bash tool, keep it plain)

```bash
curl -sS -X POST "$WEBHOOK_URL/ingest" \
  -H "Authorization: Bearer $WEBHOOK_API_KEY" \
  -H "Content-Type: application/json" \
  -d @payload.json
```

## Error handling
- On HTTP `429` or `5xx` from `/ingest`: retry with exponential backoff **1s, 2s, 4s**, then bail.
- Per-paper failures go into the `errors` array; the batch continues.
- On arXiv API failure: log the topic in `errors`, move to next topic.

## Output JSON (returned to `/fire` caller)

```json
{
  "ingested": [
    { "doc_id": "ingest_abc123...", "title": "<title>", "existed": false, "nodes": 3, "arxiv_id": "2405.12345v2" }
  ],
  "skipped": [
    { "reason": "abstract < 100 chars", "arxiv_id": "2405.00001" }
  ],
  "errors": [
    { "arxiv_id": "2405.00002", "status": 429, "message": "rate limited" }
  ]
}
```

`doc_id`, `existed`, and `nodes` come straight from the `/ingest` response — do not invent them.

## Don't
- Don't fetch PDFs yet. Abstract only is enough for corpus seeding; PDF extraction is parked.
- Don't silently skip version mismatches — include them so a re-fire picks up `v2` of a previously `v1` paper.
- Don't lowercase/rewrite arxiv titles.
