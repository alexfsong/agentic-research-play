---
name: ingest-news
description: Pull recent articles from a topic's RSS feeds (or a caller-supplied feed list) and POST each to the research webhook /ingest endpoint. Stateless — re-fires are safe (dedupe on normalized URL).
tools: [Bash, WebFetch, Read]
model: claude-sonnet-4-6
---

# ingest-news

## Inputs (from `/fire` payload)
- `topic` (string, optional) — looked up in `seeds.json` `news.topic_feeds`.
- `max` (int, optional, default **30**) — total articles cap across all feeds.
- `feeds` (string[], optional) — explicit feed URLs that override any topic mapping.

## Environment
- `WEBHOOK_URL`, `WEBHOOK_API_KEY` (bearer).

## Algorithm

1. Load `seeds.json` (Read tool). Resolve the feed list:
   1. If `feeds` is non-empty → use it verbatim.
   2. Else if `topic` is in `news.topic_feeds` → use the mapped list.
   3. Else → `news.general_feeds`.
2. For each feed URL (use `WebFetch` asking for the raw feed):
   - Parse items (RSS 2.0 `<item>` or Atom `<entry>`).
   - For each item, extract: `title`, `link` (article URL), `pubDate`/`published`, `author`/`creator` if present, `description`/`summary`.
   - Normalize the article URL:
     - Lowercase host.
     - Strip `#fragment`.
     - Remove query params whose name starts with `utm_` or is in `{"fbclid","gclid","ref"}`.
   - Fetch the article body with `WebFetch` asking for Markdown, strip obvious nav/footer boilerplate. If body is < 500 chars, fall back to `description`. Skip if still < 500 chars.
3. Iterate until `max` articles are POSTed or all feeds exhausted.
4. Return the aggregate JSON.

## URL normalization example
- In:  `https://Foo.com/Post?utm_source=rss&id=42#top`
- Out: `https://foo.com/Post?id=42`

## /ingest payload

```json
{
  "source": "news",
  "title": "<article title>",
  "content": "<markdown body>",
  "metadata": {
    "type": "news",
    "topic": "<resolved topic or 'general'>",
    "url": "<normalized url>",
    "source_domain": "example.com",
    "feed": "https://example.com/rss",
    "author": "Alice Liu",
    "published_at": "2026-04-21T12:00:00Z",
    "fetched_at": "<now UTC ISO>"
  },
  "dedupe_key": "news:<normalized url>"
}
```

No `chunk_size` override — use the server default.

## Error handling
- On `429` / `5xx` from `/ingest`: retry 1s, 2s, 4s, then bail that article. Don't abort the batch.
- On feed parse failure: log the feed in `errors`, skip to next.
- On article fetch failure: log in `errors`.

## Output JSON

```json
{
  "ingested": [
    { "doc_id": "ingest_...", "title": "<title>", "url": "<normalized>", "existed": false, "nodes": 4 }
  ],
  "skipped": [
    { "reason": "body < 500 chars", "url": "<normalized>" }
  ],
  "errors": [
    { "url": "<normalized>", "status": 502, "message": "..." }
  ]
}
```

## Don't
- Don't fetch paywalled articles repeatedly — if a fetch returns a paywall/login page (look for keywords `subscribe`, `sign in`, body < 500 chars), skip and log.
- Don't include feeds from `seeds.json` that are outside the resolved feed list even if they look relevant — trust the mapping.
- Don't re-normalize an already-normalized URL on retry (idempotent anyway, but it's noisy in logs).
