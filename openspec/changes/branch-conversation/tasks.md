## 1. Schema + DAO

- [ ] 1.1 Create `ask_thread_parents` join table in `courses_db.py` migration: `(thread_id INTEGER NOT NULL REFERENCES ask_threads(id), parent_thread_id INTEGER NOT NULL REFERENCES ask_threads(id), parent_turn_id INTEGER NOT NULL REFERENCES ask_turns(id), parent_quote TEXT, created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')))`
- [ ] 1.2 Add indexes: `idx_ask_thread_parents_thread ON ask_thread_parents(thread_id)`, `idx_ask_thread_parents_parent ON ask_thread_parents(parent_thread_id)`, and `UNIQUE INDEX idx_ask_thread_parents_single_v1 ON ask_thread_parents(thread_id)` (drop this unique index when enabling DAG)
- [ ] 1.3 Extend `create_ask_thread` to accept optional `parent_thread_id`, `parent_turn_id`, `parent_quote` and insert one row into `ask_thread_parents` when present
- [ ] 1.4 Add `get_ask_thread_parents(thread_id) -> list[ParentLink]` DAO function (always returns ≤ 1 in v1)
- [ ] 1.5 Add `get_ask_thread_children(thread_id) -> list[ThreadSummary]` DAO function
- [ ] 1.6 Add `get_ask_thread_descendants(thread_id) -> list[ThreadSummary]` DAO function (transitive, used by tree-collapse renderer)

## 2. Webhook `/ask` request side

- [ ] 2.1 Extend `/ask` request model with optional `parent_thread_id`, `parent_turn_id`, `parent_quote`
- [ ] 2.2 When parent fields present: validate parent_thread_id and parent_turn_id exist + parent_turn_id belongs to parent_thread_id; reject with 400 otherwise
- [ ] 2.3 Force a new thread when parent fields present (bypass existing thread_id reuse path)
- [ ] 2.4 Resolve overlapping citations: parse parent answer, find `[n]` markers within `parent_quote` span, look up source URLs/titles, cap at top 5
- [ ] 2.5 Add `parent_context` block to routine fire payload: `{question, answer_excerpt, quote?, carried_sources[]}`
- [ ] 2.6 Add env flag `THREAD_BRANCHING_ENABLED` (default true); when false, reject parent-bearing requests with 400

## 3. Webhook thread API

- [ ] 3.1 `GET /threads/{id}` enrich response: always include `parents: []` array (length ≤ 1 in v1, populated when `ask_thread_parents` row exists)
- [ ] 3.2 New `GET /threads/{id}/children` endpoint: returns array of immediate child thread summaries (id, first-turn question, created_at, has_quote)
- [ ] 3.3 New `GET /threads/{id}/descendants` endpoint (transitive): for tree-collapse renderer
- [ ] 3.4 Update OpenAPI / docstrings for endpoint changes

## 4. Cloud routine + local skill

- [ ] 4.1 Extend `routines/ingest-ask.md` JSON input contract with optional `parent_context` block
- [ ] 4.2 Add prompt branch: when `parent_context` present, prepend "This question is a follow-up on prior context: <quote / answer excerpt>. Carried sources: <list>" before the question
- [ ] 4.3 Mirror identical changes in `.claude/skills/ingest-ask/` local fallback
- [ ] 4.4 Verify carried sources are included in the routine's WebFetch + ingest planning so citations can re-resolve them

## 5. PWA: turn-level branch

- [ ] 5.1 Add "Branch" button to every rendered turn in Threads detail view
- [ ] 5.2 Tapping Branch: opens Ask composer prefilled with "Follow up on: <parent question>" and `parent_thread_id` + `parent_turn_id` queued
- [ ] 5.3 On submit, POST `/ask` with parent fields; on response, navigate to new thread

## 6. PWA: selection-level branch

- [ ] 6.1 Implement text-selection listener on answer body containers
- [ ] 6.2 On non-empty selection, render floating "Ask about this" button anchored to selection rect
- [ ] 6.3 Tapping button: opens Ask composer prefilled with "Going deeper on: <quote>" and `parent_thread_id` + `parent_turn_id` + `parent_quote` queued
- [ ] 6.4 iOS Safari fallback: if native context menu conflicts, expose a persistent "Ask about selection" toolbar button instead
- [ ] 6.5 On submit, POST `/ask` with parent fields; on response, navigate to new thread

## 7. PWA: parent breadcrumb + tree-collapse index

- [ ] 7.1 Thread detail header: render parent breadcrumb chip when thread has parent (links to parent thread, scrolls to parent_turn_id)
- [ ] 7.2 If `parent_quote` present, render it as a quote block under the breadcrumb
- [ ] 7.3 Thread detail bottom: render "Branched threads (N)" list from `GET /threads/{id}/children`, each linking to the child thread
- [ ] 7.4 Threads index: render as tree — fetch threads with parent linkage, group children under parents, indent one level per nesting depth
- [ ] 7.5 Per-node collapse/expand control on each parent in the index
- [ ] 7.6 Persist collapse state in sessionStorage keyed by thread_id; restore on index re-render within session

## 8. Verification

- [ ] 8.1 Manual E2E: turn-level branch creates a new thread with correct parent linkage, parent breadcrumb renders
- [ ] 8.2 Manual E2E: selection-level branch persists `parent_quote`, breadcrumb shows quote block, prompt context includes the quote
- [ ] 8.3 Manual E2E: select text overlapping `[2]` and `[5]` citations → branched first turn re-cites at least one of those sources
- [ ] 8.4 Manual E2E: branch a deep-tier turn (composes with `[[deep-research-mode]]`) — branched first turn can itself be deep
- [ ] 8.5 Manual: verify re-parenting attempt has no API surface (try editing parent fields via existing endpoints, confirm 405/no-op)
- [ ] 8.6 Manual: set `THREAD_BRANCHING_ENABLED=false`, verify PWA hides affordances and webhook rejects parent-bearing requests
- [ ] 8.7 Manual iOS Safari: confirm selection action works without breaking native text-select / copy
