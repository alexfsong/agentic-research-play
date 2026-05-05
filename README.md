# agentic-research-play

Companion repo for the [research-webhook](https://github.com/alexfsong/research-webhook) FastAPI app. Holds the **cloud routines** (uploaded to the Anthropic Routines workspace) and the **local-fallback skill** (installed under `claude-runner` on the box) that fetch web content and ingest it into the corpus. Architecture and migration notes live here; live ops conventions live in `research-webhook/INFRA.md` on the VPS.

The corpus grows by asking questions: each `Ask` fires a routine (or, on routine pool exhaust, the local skill) that WebSearches + WebFetches + POSTs each doc to the webhook's `/ingest`. The webhook then runs `/synthesize2` over the enriched LlamaIndex corpus and returns a cited answer. Multi-lesson courses are generated on top of the same corpus with a backward-design pipeline. PWA is mobile-first.

Retrieval stack: LlamaIndex hybrid (BM25 + vector) over a `research_li` ChromaDB collection, cross-encoder rerank via `bge-reranker-base`, `CitationQueryEngine` for answers, `SubQuestionQueryEngine` for decomposed queries. A `PropertyGraphIndex` scaffold is wired but gated off until the corpus is worth the extra LLM pass per ingest.

> **Migration in progress.** Target state below assumes the Ask-pivot is fully deployed (ODR retired, `ingest-ask` routine + skill live, ask_threads/ask_turns schema applied). The runbook lives in `DEPLOY.md`.

## Repos

| Repo | What's in it |
|------|--------------|
| `agentic-research-play` (this) | `routines/*.md` (cloud), `.claude/skills/*.md` (local fallback), `DEPLOY.md`, this README. Off-box. |
| `research-webhook` ([github](https://github.com/alexfsong/research-webhook)) | FastAPI app, SQLite schema, PWA static assets, systemd unit, Caddy snippet, `INFRA.md` (canonical box ops). On-box at `/home/researcher/research-webhook/`. |

The box hosts multiple unrelated services; `INFRA.md` is the source of truth for filesystem layout, port registry, Caddy + sslip hostnames, ufw policy, and per-app conventions. Read it before deploying anything new.

## Architecture

```
phone PWA / curl ──HTTPS──▶ Caddy (lisearch.195-201-99-206.sslip.io)
                                │
                                └──▶ 127.0.0.1:8000 FastAPI webhook (research-webhook.service)
                                       │   serves static PWA (/), API, /synthesize2
                                       │
                                       ├─▶ ChromaDB collection "research_li"
                                       │       └ /search2 (hybrid+rerank)
                                       │       └ /synthesize2 (CitationQueryEngine, internal helper for /ask)
                                       │
                                       ├─▶ SQLite courses.db
                                       │       (courses, lessons, follow_ups, ask_threads, ask_turns)
                                       │
                                       ├─▶ Anthropic /fire API ──▶ ingest-ask cloud routine
                                       │                              └ WebSearch + WebFetch + POST /ingest
                                       │
                                       └─▶ sudo -u claude-runner /usr/bin/claude -p "/ingest-ask <json>"
                                              (Pro/Max subscription OAT, fallback when routine pool full)
                                              └ same WebSearch + WebFetch + POST /ingest
```

Two layers:

1. **Research ingest** — cloud routine `ingest-ask` fires on every `/ask`. On routine pool exhaust (HTTP 429 / "pool full"), webhook falls back to a local Claude Code subprocess running as `claude-runner`, billed against the Pro/Max subscription. Both paths post to the same `/ingest` endpoint.
2. **Retrieval + course layer + webhook** — FastAPI on port 8000 (loopback only; Caddy fronts HTTPS), ingests each fetched doc into LlamaIndex + ChromaDB, serves the PWA, exposes hybrid retrieval, runs `/synthesize2` (cited answer) after every ingest, generates and edits courses, runs lesson follow-up Q&A.

## Server layout

All paths on the VPS under user `researcher` unless noted. The repo root is `/home/researcher/research-webhook/` (cloned from `github.com/alexfsong/research-webhook`).

| Path | Purpose |
|------|---------|
| `~/research-webhook/persist.py` | Shared ChromaDB `PersistentClient` (singleton; avoids Chroma's same-path-different-settings error). |
| `~/research-webhook/llamaindex_store.py` | LlamaIndex path: ingest into `research_li`, hybrid retrieval, citation/sub-question synthesis, optional `PropertyGraphIndex`. |
| `~/research-webhook/courses_db.py` | SQLite DAO for courses + ask_threads + ask_turns. |
| `~/research-webhook/courses.py` | Backward-design course generation pipeline. |
| `~/research-webhook/webhook.py` | FastAPI app: `/ask`, `/ask_callback`, `/ingest`, `/synthesize2`, `/search2`, `/courses*`, `/threads*`, PWA backend. |
| `~/research-webhook/static/` | PWA assets (`index.html`, `app.js`, `style.css`, manifest, icons). |
| `~/research-webhook/.env` | Per-app secrets (chmod 600, gitignored). |
| `~/research-webhook/.venv/` | Per-app venv (post-pivot; previously shared with `~/open_deep_research/.venv`). |
| `~/research-webhook/deploy/` | Committed `research-webhook.service` and Caddy snippet. |
| `~/research-data/chroma/` | ChromaDB vector store (collection `research_li`). |
| `~/research-data/courses.db` | SQLite (courses, lessons, follow_ups, ask_threads, ask_turns). |
| `~/research-data/reports/` | Pre-pivot ODR reports — already in the corpus, not written to anymore. |
| `~/research-data/hf-cache/` | HuggingFace cache for the embedder + reranker models. |
| `~/research-data/ask-audit.log` | One JSON line per `/ask` (run_id, route, question prefix, ts). |
| `/etc/systemd/system/research-webhook.service` | Systemd unit. |
| `/etc/caddy/Caddyfile` | Caddy global config; per-app block in `~/research-webhook/deploy/Caddyfile.snippet`. |
| `/home/claude-runner/.claude/` | OAT-authed Claude Code install (subscription fallback runner). |
| `/home/claude-runner/.claude/skills/ingest-ask.md` | Mirror of the cloud routine, run via `claude -p`. |

## Services

```bash
sudo systemctl status research-webhook        # FastAPI on 127.0.0.1:8000
sudo systemctl status caddy                   # reverse proxy on :80/:443
sudo journalctl -u research-webhook -f        # live webhook logs
```

`researcher` has NOPASSWD sudo on `systemctl` + `journalctl` only. Anything else (`apt`, `adduser`, `visudo`, editing `/etc/sudoers.d/*`) needs root via Hetzner Cloud Console or `ssh root@…`.

## Hostname

Public HTTPS: `https://lisearch.195-201-99-206.sslip.io/` (Let's Encrypt cert auto-issued by Caddy on first request). The box uses [sslip.io](https://sslip.io) wildcards because Anthropic Routines' sandbox proxy blocks DuckDNS-style providers but accepts sslip. Pattern: `<app>.<dotted-ip>.sslip.io`.

## Usage

### 1. PWA (phone or laptop)

Open `https://lisearch.195-201-99-206.sslip.io/` in Safari or Chrome, paste the webhook bearer token once (stored in `localStorage`), and use the tab bar:

- **Learn** — list of courses; tap to view editable detail with per-lesson follow-up Q&A. New courses generated from a corpus-wide query via the backward-design pipeline.
- **Ask** — submit a question. Pick **Auto / Cloud quota / My subscription** (default Auto). The chosen route fetches and ingests sources, then `/synthesize2` returns a cited answer. Auto prefers cloud, falls back to local on routine pool exhaust.
- **Conversations** — list of ask-threads; tap one to read prior Q/A turns, then continue with another question. Each continuation appends a new turn and ingests fresh material on demand.
- **Search** — hybrid (BM25 + vector) search with cross-encoder rerank over all ingested chunks. Power-user surface for inspecting the corpus.

A footer link exposes raw corpus stats and any pre-pivot ODR reports still on disk.

**Install to iPhone home screen:** open the URL in Safari, tap Share → Add to Home Screen. The PWA launches standalone (no browser chrome) with the app icon.

Sign out clears the token from `localStorage`.

### 2. Trigger from terminal

```bash
curl -X POST https://lisearch.195-201-99-206.sslip.io/ask \
  -H "Authorization: Bearer <WEBHOOK_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"question":"your question","mode":"auto"}'
```

Response is immediate. Poll `GET /research/{run_id}` until `status=complete`; the response includes `synthesis.answer` and `synthesis.citations`. Typical run: 30–90 s. Subprocess (local) skill runs are capped at 240 s by default (`CLAUDE_FALLBACK_TIMEOUT`).

### 3. Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `GET`  | `/`                          | —      | PWA shell (`static/index.html`). |
| `GET`  | `/static/*`                  | —      | PWA assets. |
| `GET`  | `/manifest.webmanifest`      | —      | PWA manifest. |
| `GET`  | `/health`                    | —      | Liveness check. |
| `GET`  | `/stats`                     | Bearer | Corpus stats (LlamaIndex node count, course count, ask_thread count). |
| `POST` | `/ingest`                    | Bearer | Generic doc ingest. Body `{source, title, content, metadata?, dedupe_key?, chunk_size?}`. Used by routines, skill, and the legacy report ingest path. |
| `DELETE` | `/corpus/{doc_id}`         | Bearer | Drop a doc's nodes from the LlamaIndex vector store + docstore. Matches `^(report\|ingest)_`. |
| `GET`  | `/search2?q=…&k=10&hybrid=true&rerank=true` | Bearer | Hybrid retrieval over `research_li`, optionally reranked. |
| `POST` | `/synthesize2`               | Bearer | `CitationQueryEngine` (or `SubQuestionQueryEngine` when `subq=true`) over `research_li`. Internal helper used by `/ask_callback`. Rate-limited. |
| `POST` | `/ask`                       | Bearer | Body `{question, mode?, thread_id?, max_fetches?, urls?, topic?}`. Mints `run_id`, fires routine (or local skill), returns `{run_id, thread_id, turn_id, route}`. |
| `POST` | `/ask_callback`              | Bearer | Routine/skill callback. Triggers `/synthesize2` against the enriched corpus and stores the answer in run state. |
| `GET`  | `/research/{run_id}`         | Bearer | Run state poll: `{status, route, ingested[], skipped[], errors[], synthesis}`. |
| `GET`  | `/threads?limit=20`          | Bearer | List ask-threads with title, turn count, updated_at. |
| `GET`  | `/threads/{thread_id}`       | Bearer | Thread detail: turns array (Q/A/citations). |
| `POST` | `/threads/{thread_id}/ask`   | Bearer | Continuation turn — same shape as `/ask`. |
| `DELETE` | `/threads/{thread_id}`     | Bearer | Drop a thread + cascading turns. (Ingested docs stay in the corpus.) |
| `POST` | `/courses`                   | Bearer | Generate a course (backward-design pipeline). |
| `GET`  | `/courses/{id}/status`       | Bearer | Poll `pending → generating → draft → failed`. |
| `PATCH`/`POST`/`DELETE` | `/courses/*` | Bearer | Edit, regenerate, add, delete. (See `courses.py`.) |
| `POST` | `/courses/{cid}/lessons/{lid}/research_fire` | Bearer | Lesson gap-fill via the older `ingest-research` routine (unchanged). |

Auth is a static bearer token set via `WEBHOOK_API_KEY`. If empty, all endpoints are open.

### 4. iOS Shortcut

Same as before, but POST `/ask` and read `synthesis.answer` from the polled response.

## Configuration

Webhook env vars live in `/home/researcher/research-webhook/.env` (loaded by systemd via `EnvironmentFile=`; chmod 600). Bare `KEY=VALUE` per line — no quotes (systemd treats them as literal), no `export`.

| Env var | Purpose |
|---------|---------|
| `WEBHOOK_API_KEY` | Static bearer token. Empty = no auth. |
| `ANTHROPIC_API_KEY` | Used by `/synthesize2` and the course generator (Sonnet/Haiku via Anthropic SDK). |
| `ANTHROPIC_OAT` | OAuth token for the routine `/fire` API (`sk-ant-oat01-…`). |
| `ROUTINE_INGEST_ASK_FIRE_URL` | The cloud routine's `trig_*/fire` URL. |
| `ROUTINE_RESEARCH_FIRE_URL` | The lesson gap-fill routine's `trig_*/fire` URL. |
| `ROUTINE_FIRE_BETA` | Anthropic-beta header, default `experimental-cc-routine-2026-04-01`. |
| `CLAUDE_FALLBACK_USER` | Linux user for the subscription fallback subprocess (default `claude-runner`). |
| `CLAUDE_FALLBACK_TIMEOUT` | Per-run subprocess cap, seconds (default `240`). |
| `ASK_DEFAULT_MAX_FETCHES` | Default `max_fetches` for `/ask` if not specified (default `10`). |
| `ASK_RUN_TTL` | In-memory run state TTL, seconds (default `3600`). |
| `RESEARCH_RUN_TTL` | Same, for lesson gap-fill runs (default `3600`). |
| `SYNTHESIS_MODEL` | Claude model used by `/synthesize2`, default `claude-sonnet-4-6`. |
| `SYNTHESIS_RATE_PER_MIN` | Per-IP rate cap on `/synthesize2`, default `10`. |
| `LLAMAINDEX_ENABLED` | Default `true`. Set `false` to disable the LlamaIndex path entirely. |
| `LI_COLLECTION` | ChromaDB collection name, default `research_li`. |
| `LI_EMBED_MODEL` | HuggingFace embedder, default `BAAI/bge-small-en-v1.5`. |
| `LI_RERANK_MODEL` | Cross-encoder reranker, default `BAAI/bge-reranker-base`. |
| `LI_RERANK_TOP_N` | Rerank output size, default `5`. |
| `LI_RETRIEVE_TOP_K` | Base retriever fan-out before rerank, default `20`. |
| `LI_CHUNK_SIZE` / `LI_CHUNK_OVERLAP` | `SentenceSplitter` chunking, defaults `512` / `64`. |
| `LI_GRAPH_ENABLED` | Default `false`. Enable to extract triplets per ingest. |
| `HF_HOME` | HuggingFace cache dir. Set to `~/research-data/hf-cache` to persist model weights. |
| `COURSE_PLAN_MODEL` / `COURSE_LESSON_MODEL` / `COURSE_CLAIMS_MODEL` | Course generation models. |
| `COURSE_CORPUS_K` / `COURSE_LESSON_K` | Retrieval fan-out for course generation. |

## Cost / quota notes

- **Cloud routine path**: pool capped at 5 concurrent runs in our tier (shared across all routines in the workspace, not per-routine). WebSearch + WebFetch under the routine quota; no per-search Anthropic fee.
- **Local skill path**: subprocess `claude -p` under `claude-runner`'s subscription OAT. Counts against the user's 5h Pro/Max window. WebSearch is free under subscription. Heavy use may compete with interactive Claude Code sessions on the same account.
- **Synthesis (`/synthesize2`)**: Anthropic API tokens, billed per use. Defaults to `claude-sonnet-4-6`. The multi-agent rate-limit pain that motivated the ODR retirement is no longer a factor — single Sonnet pass per Ask.

## Security

- Port 8000 binds `127.0.0.1` only. Caddy fronts HTTPS on `:443`; ufw allows 22/80/443 only.
- Bearer-token auth on every API path.
- `claude-runner` Linux user has no sudo, no shell login, no docker. Webhook reaches it only via `sudo -u claude-runner /usr/bin/claude -p ...` (NOPASSWD limited to that single binary).
- Subprocess invoked with `--allowedTools "WebSearch,WebFetch,Bash"` — no Edit/Write/Read of arbitrary files.
- `.env` files chmod 600. Subscription OAT in `~claude-runner/.claude/.credentials.json` chmod 600. Rotate every 90 days.
- One-line audit log at `~/research-data/ask-audit.log` (run_id, route, question prefix, timestamp).
- Never paste `.env` contents into chats / logs / PRs (per `INFRA.md`'s hard rules). Rotate immediately if it happens.

## Routines + skill workflow

This repo is the source of truth for the routine + skill markdown.

**Cloud routines** (`routines/*.md`): upload to the Anthropic Routines workspace and capture the `trig_*/fire` URL into the box's `.env` as `ROUTINE_*_FIRE_URL`. Workspace secrets needed: `WEBHOOK_URL=https://lisearch.195-201-99-206.sslip.io`, `WEBHOOK_API_KEY` (matching the box's `.env`).

- `ingest-ask.md` — primary Ask path. Reads `{run_id, question, thread_id, max_fetches, urls, topic}`, fires `/ingest` per fetched doc, callbacks to `/ask_callback`.
- `ingest-research.md` — lesson gap-fill (existing). Reads `{run_id, question, lesson_id, course_id, urls, max_fetches, topic}`, fires `/ingest` with `source=research_fill`, callbacks to `/research_callback`.
- `ingest-arxiv.md`, `ingest-news.md`, `ingest-url.md` — bulk seed routines. Schedule via the workspace UI's cron.

**Local skill** (`.claude/skills/ingest-ask.md`): mirrors `ingest-ask.md`. Drop into `/home/claude-runner/.claude/skills/`. Webhook invokes via `subprocess sudo -u claude-runner /usr/bin/claude -p "/ingest-ask <json>"`. Same payload contract; only the `route` field in the callback differs (`local` vs `cloud`).

When extending the Ask flow, change both the routine and the skill in lockstep — they share the `/ask_callback` shape.

## Provisioning quick reference

The box was bootstrapped from a fresh Ubuntu 24.04 Hetzner CPX21 (3 vCPU / 4 GB / 80 GB). Per-app provisioning follows the checklist in `INFRA.md` ("Add a new app — checklist"). For this repo's app:

1. Caddy already running. Caddyfile snippet in `research-webhook/deploy/Caddyfile.snippet` proxies `lisearch.195-201-99-206.sslip.io` → `127.0.0.1:8000`.
2. `research-webhook` cloned at `/home/researcher/research-webhook/`, per-app venv, `.env` chmod 600.
3. `research-webhook.service` installed from `deploy/`, `daemon-reload`, `enable --now`.
4. `claude-runner` user provisioned per `DEPLOY.md` Part B.1.
5. Routines uploaded to the Anthropic Routines workspace; trigger URLs in `.env`.
6. Smoke test: `curl https://lisearch.195-201-99-206.sslip.io/health` → 200.

## Graph-RAG scaffold (LlamaIndex `PropertyGraphIndex`)

Graph-RAG scaffolding lives inside `llamaindex_store.py` — gated by `LI_GRAPH_ENABLED` (default `false`). When enabled, `ingest_document` additionally runs `SimpleLLMPathExtractor` over each doc and persists triplets to `SimplePropertyGraphStore` at `~/research-data/llamaindex-graph/`. Flipping the flag on costs one extra LLM pass per ingest — turn it on when the corpus is worth the spend (useful once cross-corpus course generation starts demanding structured connections).

No separate framework to install; this is the same LlamaIndex stack used for vector retrieval. Swapping in Neo4j as the graph store is a future step if the graph outgrows the in-memory JSON file.
