## 1. Schema + DAO

- [ ] 1.1 Add `depth TEXT NOT NULL DEFAULT 'standard'` and `payload_shape TEXT NOT NULL DEFAULT 'flat'` columns to `ask_turns` in `courses_db.py` migration block
- [ ] 1.2 Update `add_ask_turn` / `update_ask_turn` / `get_ask_turn_*` DAO helpers to accept and persist `depth` and `payload_shape`
- [ ] 1.3 Update `list_ask_threads` / `get_ask_thread` to include `depth` and `payload_shape` in returned turn rows

## 2. Webhook `/ask` request side

- [ ] 2.1 Extend `/ask` request model with `depth: Literal['standard','deep'] = 'standard'`; reject unknown values with 422
- [ ] 2.2 Read per-tier env budgets `ASK_DEPTH_<TIER>_MAX_{SEARCHES,FETCHES,TOKENS,ITERATIONS}` at startup; expose as a tier→budget dict
- [ ] 2.3 Add `ASK_DEPTH_DEEP_MAX_PER_DAY` per-bearer counter (in-memory or SQLite); 429 on cap
- [ ] 2.4 Add per-bearer in-flight counter for deep; second deep submission from same bearer returns 429 with `error="deep_in_flight"`
- [ ] 2.5 Route `depth='deep'` to new orchestrator; standard stays on existing single-pass path
- [ ] 2.6 Send `depth` in routine fire payload; pick fire URL by tier (existing `ROUTINE_ASK_FIRE_URL` for standard, new `ROUTINE_ASK_DEEP_FIRE_URL` for deep)

## 3. Deep-research loop orchestrator

- [ ] 3.1 Create new module `deep_research.py` in `~/research-webhook/` with async `run_deep(turn_id, question, budgets) -> Report`
- [ ] 3.2 Define single tool-use schema for the round call returning `{sections[], gap_queries[]}` together
- [ ] 3.3 Implement round loop: invoke synthesizer (one LLM call per round) → dedupe `gap_queries[]` → fire next round of search+fetch → ingest → repeat
- [ ] 3.4 Track per-turn dedupe set of executed queries; skip duplicates without budget charge
- [ ] 3.5 Enforce stop conditions: empty gap list, iteration cap, token cap; record `termination` reason on final report
- [ ] 3.6 Snapshot citation map per section at draft time so corpus growth mid-loop doesn't renumber refs
- [ ] 3.7 Assemble final `{toc, sections[]}` report payload; serialize to JSON in `ask_turns.answer` with `payload_shape='report'`
- [ ] 3.8 Decrement per-bearer in-flight deep counter on completion (success, error, or timeout)

## 4. Webhook `/ask_callback`

- [ ] 4.1 Accept payload with either `answer` (flat) or `report` (sectioned + toc + termination)
- [ ] 4.2 Persist via DAO with correct `payload_shape`
- [ ] 4.3 Validate report shape (toc length matches sections, every citation index resolves) before write

## 5. Cloud routine + local skill

- [ ] 5.1 Extend `routines/ingest-ask.md` JSON input contract with `depth` field
- [ ] 5.2 Add prompt branch: standard=existing single-pass behavior, deep=single round of work for the orchestrator (search+fetch+ingest only, no full draft — orchestrator handles synthesis)
- [ ] 5.3 Mirror identical changes in `.claude/skills/ingest-ask/` local fallback
- [ ] 5.4 Verify local skill receives `depth` from webhook fallback path and applies matching budget

## 6. PWA

- [ ] 6.1 Add depth segmented control to Ask tab (standard / deep) with per-tier description label including cost note (sized to allow a third tier later without redesign)
- [ ] 6.2 Send `depth` in `/ask` POST body; default `standard`
- [ ] 6.3 Threads view: detect `payload_shape` per turn; render `flat` as today, render `report` with TOC + collapsible sections + per-section citations
- [ ] 6.4 Render `termination` reason in report footer (`empty_gaps` | `iteration_cap` | `token_cap`)
- [ ] 6.5 Handle 429 `deep_in_flight` from `/ask`: show "You already have a deep run in flight on this bearer — wait for it to finish" and keep composer state

## 7. Env + ops

- [ ] 7.1 Add new env vars to `~/research-webhook/.env.example`: `ASK_DEPTH_STANDARD_*`, `ASK_DEPTH_DEEP_*`, `ASK_DEPTH_DEEP_MAX_PER_DAY`, `ASK_DEPTH_ENABLED`, `ROUTINE_ASK_DEEP_FIRE_URL`
- [ ] 7.2 Add rollback env flag `ASK_DEPTH_ENABLED=false` short-circuit at top of `/ask`
- [ ] 7.3 Confirm `ANTHROPIC_API_KEY` set on box (per `[[anthropic_auth_split]]`) — deep loop is direct SDK
- [ ] 7.4 Document deep-tier cost expectation in `README.md` (env table + Ask section)

## 8. Verification

- [ ] 8.1 Manual E2E: each tier returns the right payload shape, persists `depth` correctly, renders correctly
- [ ] 8.2 Manual E2E: deep run hits at least 2 iterations on a question with known multi-hop structure
- [ ] 8.3 Manual E2E: cap a `deep` run at 2 iterations via env, verify `termination='iteration_cap'` appears in report footer
- [ ] 8.4 Manual E2E: fire standard while deep in-flight on same bearer, confirm standard returns without waiting
- [ ] 8.5 Manual E2E: fire second deep on same bearer while first in-flight, confirm 429 `deep_in_flight`
- [ ] 8.6 Verify thread mixing flat + report turns renders in PWA without errors
