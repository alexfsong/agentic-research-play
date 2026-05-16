## 1. Schema + DAO

- [x] 1.1 Add `depth TEXT NOT NULL DEFAULT 'standard'` and `payload_shape TEXT NOT NULL DEFAULT 'flat'` columns to `ask_turns` in `courses_db.py` migration block
- [x] 1.2 Update `add_ask_turn` / `update_ask_turn` / `get_ask_turn_*` DAO helpers to accept and persist `depth` and `payload_shape`
- [x] 1.3 Update `list_ask_threads` / `get_ask_thread` to include `depth` and `payload_shape` in returned turn rows

## 2. Webhook `/ask` request side

- [x] 2.1 Extend `/ask` request model with `depth: Literal['standard','deep'] = 'standard'`; reject unknown values with 422
- [x] 2.2 Read per-tier env budgets `ASK_DEPTH_<TIER>_MAX_{SEARCHES,FETCHES,TOKENS,ITERATIONS}` at startup; expose as a tier→budget dict
- [x] 2.3 Add `ASK_DEPTH_DEEP_MAX_PER_DAY` per-bearer counter (SQLite-backed via ask_deep_queue, counted at enqueue time); 429 on cap
- [x] 2.4 ~~Add per-bearer in-flight counter for deep; second deep submission from same bearer returns 429 with `error="deep_in_flight"`~~ **Superseded by §9 — second deep enqueues instead of 429.**
- [x] 2.5 Route `depth='deep'` to new orchestrator; standard stays on existing single-pass path
- [x] 2.6 Send `depth` in routine fire payload; pick fire URL by tier (existing `ROUTINE_ASK_FIRE_URL` for standard, new `ROUTINE_ASK_DEEP_FIRE_URL` for deep)

## 3. Deep-research loop orchestrator

- [x] 3.1 Create new module `deep_research.py` in `~/research-webhook/` with async `run_deep(turn_id, question, budgets) -> Report`
- [x] 3.2 Define single tool-use schema for the round call returning `{sections[], gap_queries[]}` together
- [x] 3.3 Implement round loop: invoke synthesizer (one LLM call per round) → dedupe `gap_queries[]` → fire next round of search+fetch → ingest → repeat
- [x] 3.4 Track per-turn dedupe set of executed queries; skip duplicates without budget charge
- [x] 3.5 Enforce stop conditions: empty gap list, iteration cap, token cap; record `termination` reason on final report
- [x] 3.6 Snapshot citation map per section at draft time so corpus growth mid-loop doesn't renumber refs
- [x] 3.7 Assemble final `{toc, sections[]}` report payload; serialize to JSON in `ask_turns.answer` with `payload_shape='report'`
- [x] 3.8 Decrement per-bearer in-flight deep counter on completion (success, error, or timeout)

## 4. Webhook `/ask_callback`

- [x] 4.1 Accept payload with either `answer` (flat) or `report` (sectioned + toc + termination)
- [x] 4.2 Persist via DAO with correct `payload_shape`
- [x] 4.3 Validate report shape (toc length matches sections, every citation index resolves) before write

## 5. Cloud routine + local skill

- [x] 5.1 Extend `routines/ingest-ask.md` JSON input contract with `depth` field
- [x] 5.2 Add prompt branch: standard=existing single-pass behavior, deep=single round of work for the orchestrator (search+fetch+ingest only, no full draft — orchestrator handles synthesis)
- [x] 5.3 Mirror identical changes in `.claude/skills/ingest-ask/` local fallback (mirrored in `.claude/commands/ingest-ask.md`, the actual local-skill location)
- [x] 5.4 Verify local skill receives `depth` from webhook fallback path and applies matching budget

## 6. PWA

- [x] 6.1 Add depth segmented control to Ask tab (standard / deep) with per-tier description label including cost note (sized to allow a third tier later without redesign)
- [x] 6.2 Send `depth` in `/ask` POST body; default `standard`
- [x] 6.3 Threads view: detect `payload_shape` per turn; render `flat` as today, render `report` with TOC + collapsible sections + per-section citations
- [x] 6.4 Render `termination` reason in report footer (`empty_gaps` | `iteration_cap` | `token_cap`)
- [x] 6.5 Handle 429 `deep_in_flight` from `/ask`: show "You already have a deep run in flight on this bearer — wait for it to finish" and keep composer state

## 7. Env + ops

- [x] 7.1 Add new env vars to `~/research-webhook/.env.example`: `ASK_DEPTH_STANDARD_*`, `ASK_DEPTH_DEEP_*`, `ASK_DEPTH_DEEP_MAX_PER_DAY`, `ASK_DEPTH_ENABLED`, `ROUTINE_ASK_DEEP_FIRE_URL`
- [x] 7.2 Add rollback env flag `ASK_DEPTH_ENABLED=false` short-circuit at top of `/ask`
- [x] 7.3 Confirm `ANTHROPIC_API_KEY` set on box (per `[[anthropic_auth_split]]`) — deep loop is direct SDK (note in `.env.example`)
- [x] 7.4 Document deep-tier cost expectation in `README.md` (env table + Ask section)

## 9. Deep queue + subscription auth

- [x] 9.1 Add `ask_deep_queue` table (`run_id, bearer, payload_json, status, depth, enqueued_at, started_at, finished_at, error`) to `courses_db.py` with an index on `(bearer, status, enqueued_at)`
- [x] 9.2 DAO helpers: `enqueue_deep`, `pop_next_for_bearer`, `mark_deep_done`, `count_deep_today(bearer)`, `queue_position(run_id)`, `bearers_with_queued()`, `revive_stale_running()`
- [x] 9.3 Replace in-memory inflight counter with the queue: `/ask` enqueues; `_drain_deep(bearer)` async loop pops + runs `deep_research.run_deep` serially; daily cap checked at enqueue
- [x] 9.4 Spawn drainers at FastAPI startup for every bearer with queued work; spawn on enqueue if no drainer exists for that bearer; global `asyncio.Semaphore(ASK_DEPTH_DEEP_MAX_DRAINERS)` caps concurrent subprocesses
- [x] 9.5 `/ask_runs/{run_id}` returns `status='queued'` + `queue_position` + `queue_total` for queued deep runs; `started_at` once running
- [x] 9.6 PWA shows "queued · position N of M" until the drainer picks it up; refreshed pollAskRun handles `queued`/`running` (continue polling, status callback)
- [x] 9.7 Created `.claude/commands/deep-synth.md` local skill: input `{question, excerpts[], prior_headings[], executed_queries[], iteration, round_max_tokens}`; output `{sections:[{heading, body, citation_ns}], gap_queries:[]}` JSON to stdout
- [x] 9.8 `deep_research._synth_round` invokes `claude -p "/deep-synth <json>"` subprocess (mirrors `_fire_ask_local_skill` plumbing) when `ASK_DEPTH_DEEP_BACKEND` is unset or `subscription`; uses Anthropic SDK only when `ASK_DEPTH_DEEP_BACKEND=api`
- [x] 9.9 Env: `ASK_DEPTH_DEEP_BACKEND`, `ASK_DEPTH_DEEP_MAX_DRAINERS`, `ASK_DEPTH_DEEP_SUBPROCESS_TIMEOUT`, `ASK_DEPTH_DEEP_PER_SNIPPET` documented in `.env.example`

## 8. Verification

Manual E2E checks below run on the Hetzner box (where the LlamaIndex corpus + Anthropic SDK + caddy + PWA are wired). Local static checks done in this branch: schema migration (`courses_db.init_db()` adds `depth`+`payload_shape` columns), syntax/parse on `webhook.py`, `deep_research.py`, `courses_db.py`, and `static/app.js`. The deploy steps below execute the runtime scenarios — leave unchecked until run on the box.

- [x] 8.1 Manual E2E: each tier returns the right payload shape, persists `depth` correctly, renders correctly — verified via run `ask_9049ac1c0b4f6a7f` (FSRS/SM-2/anki-rs deep query): `ask_turns.depth='deep'`, `payload_shape='report'`, PWA renders TOC + sections + per-section `citations[]`.
- [x] 8.2 Manual E2E: deep run hits at least 2 iterations on a question with known multi-hop structure — same run produced 14 sections terminating on `empty_gaps`, implying multi-round synth/gap-fill loop ran. (Iteration count not surfaced at top-level of report payload — see follow-up note below.)
- [ ] 8.3 Manual E2E: cap a `deep` run at 2 iterations via env (`ASK_DEPTH_DEEP_MAX_ITERATIONS=2`), verify `termination='iteration_cap'` appears in report footer
- [ ] 8.4 Manual E2E: fire standard while deep in-flight on same bearer, confirm standard returns without waiting
- [ ] 8.5 ~~Manual E2E: fire second deep on same bearer while first in-flight, confirm 429 `deep_in_flight`~~ **Stale** — superseded by §9 queue (second deep now enqueues, no 429).
- [x] 8.6 Verify thread mixing flat + report turns renders in PWA without errors — confirmed in PWA.

### Follow-ups (not blocking archive)

- Deep report payload does not include top-level `iterations` or `executed_queries[]`; per-section citation shape is `section.citations[{n, report_id, thread_id, query, ...}]`, which works but diverges from the `citation_ns[]` shape sketched in §3.2 / §9.7. Reconcile schema (or update spec) in a later pass.
- 8.3 / 8.4 still worth running before relying on iteration-cap + standard-vs-deep isolation in production.
