## Context

Today's Ask model is one flat `ask_threads` row per thread, with N `ask_turns` rows in chronological order. Each turn is question + answer (per `[[ask_pivot_architecture]]`). Threads are linear — there's no parent/child concept. Users with research workflows often want to fork: a long answer surfaces a sub-question, but asking it inline pollutes the parent thread, and copying the question into a new Ask loses the answer context (citations, original framing).

Two natural fork triggers in the UI:
1. **Per-turn fork** — "this turn opened up a new line of inquiry, follow it separately"
2. **Per-selection fork** — "this specific phrase / paragraph in the answer, dig deeper here"

These are two UX paths to the same underlying primitive: a thread with `parent_thread_id`, `parent_turn_id`, optional `parent_quote`.

Constraints:
- All work lands in `~/research-webhook/` (Hetzner) per `[[box_infra_conventions]]`.
- Reuse `/ask` endpoint — no new POST surface.
- Tree, not DAG. Multi-parent is a different feature with much harder UX.
- Composes cleanly with `[[deep-research-mode]]` — branched threads can be any depth.

## Goals / Non-Goals

**Goals:**
- One primitive (`parent_*` columns) backs both UX paths.
- Branched thread is a normal `ask_thread` everywhere — same list view, same detail view, same callbacks — just with a parent breadcrumb.
- Selection-level branching carries citations across the fork.
- API stays minimal: extend existing endpoints, add only `/threads/{id}/children`.

**Non-Goals:**
- No DAG / multi-parent surface in v1 API (schema is DAG-ready via join table; v1 enforces single parent via UNIQUE INDEX, drop later to enable).
- No merging branched threads.
- No automatic branching (no LLM-driven "this turn looks like a new topic, want to fork?" prompts in v1).
- No cross-thread search redesign — search stays per-thread + global as today.
- No re-parenting or moving turns between threads.

## Decisions

### Decision: One `ask_thread` row per branch, parent linkage in join table
- **Choice:** A branch is a new `ask_thread` row. Parent linkage lives in a separate `ask_thread_parents` join table (`thread_id, parent_thread_id, parent_turn_id, parent_quote, created_at`). Not a "branch turn type" inside the parent thread.
- **Rationale:** Threads are the natural unit users navigate, share, and re-open. Keeping branches as first-class threads means zero new render paths. The join table is DAG-ready from day one — adding multi-parent later is a `DROP INDEX` instead of a migration.
- **Alternative:** Branch as a special turn type. Rejected — every UI surface that lists turns would need branch handling, and a branched conversation is logically a separate dialog.
- **Alternative:** 3 columns on `ask_threads` (single parent only). Rejected — locks future DAG behind a real migration.

### Decision: Reuse `/ask`, don't add `/threads/{id}/branch`
- **Choice:** `/ask` accepts optional `parent_thread_id`, `parent_turn_id`, `parent_quote`. When present, the webhook creates a new thread with parent linkage instead of appending to an existing thread.
- **Rationale:** Branching is "ask a new question with parent context attached". The endpoint already creates threads. Adding a sibling endpoint splits the surface.
- **Alternative:** New `POST /threads/{id}/branch` endpoint. Rejected — duplicates `/ask` validation, routing, depth handling.

### Decision: Selection citations carried forward inline in prompt context, not as DB rows
- **Choice:** When the PWA sends `parent_quote`, the webhook resolves which `[n]` markers in the parent answer overlap that span and inserts the resolved source URLs/titles into the routine prompt context for the first turn. No new table.
- **Rationale:** Citations on the new turn will be regenerated against the corpus at draft time anyway. The parent's resolved sources are just hints to make sure the corpus retrieval pulls them.
- **Alternative:** New `branch_carried_citations` table. Rejected — over-engineered for a hint-passing flow.

### Decision: `parent_quote` stored verbatim, no API-layer length cap
- **Choice:** Plain TEXT in the join table, no length validation at the webhook. Trust the client; SQLite handles what the client sends.
- **Rationale:** Quote needs to render in PWA breadcrumb; verbatim is what user sees. Length is bounded by user attention span. Capping invites edge-case rejections for paragraph-quote workflows that aren't actually broken.
- **Alternative:** 1 KB or 4 KB API-layer cap. Rejected — premature limit on a usage pattern we haven't observed yet.

### Decision: Tree shape enforced via UNIQUE INDEX on join table, not DB trigger
- **Choice:** `UNIQUE INDEX idx_ask_thread_parents_single_v1 ON ask_thread_parents(thread_id)` enforces single-parent in v1. To enable DAG later: `DROP INDEX idx_ask_thread_parents_single_v1` and add multi-parent endpoints.
- **Rationale:** SQLite UNIQUE INDEX is the cheapest enforcement; failing inserts surface as constraint violations the webhook converts to HTTP 400. No SQL trigger needed for cycle detection — cycles are physically impossible on creation since a new thread can't already be a parent.
- **Alternative:** Enforce in webhook only. Rejected — DB-layer constraint catches DAO bypass and direct sqlite3 fixes.

### Decision: Two children endpoints — immediate and transitive
- **Choice:** `/threads/{id}/children` returns immediate children only (sorted by `created_at DESC`). `/threads/{id}/descendants` returns the full transitive subtree, used by the tree-collapse renderer in the Threads index.
- **Rationale:** Thread detail bottom shows immediate children only (one level). Threads index needs the whole subtree to render the tree with per-node collapse. Two endpoints keeps response sizes bounded for the common case while supporting tree render.

### Decision: Threads index renders tree with per-node collapse/expand
- **Choice:** Index fetches all roots + their descendants; renders nested rows with one indentation level per nesting depth and a collapse/expand control on each parent. Collapse state persists in `sessionStorage` keyed by thread_id.
- **Rationale:** Branching meaningfully clusters related work; flat-with-chip understates the relationship. Tree render makes parent/child obvious without an extra navigation step.
- **Alternative:** Flat with breadcrumb chip. Rejected — buries the structure that branching is meant to surface.
- **Trade-off:** More PWA render code; sessionStorage state is per-tab not persisted across browser restarts (acceptable for v1).

## Risks / Trade-offs

- **[Branched-thread proliferation clutters Threads index]** Heavy users may end up with a deep forest → Mitigation: tree-collapse on every parent + sessionStorage of collapse state means deep trees collapse cheaply; v2 could add depth limits or pagination if real complaint.
- **[Selection action conflicts with native text selection on iOS]** PWA floating button on long-press might fight Safari's native context menu → Mitigation: prototype on iOS first, fall back to a persistent "Ask about selection" button in toolbar if native conflict is unresolvable.
- **[Quote drift if parent answer is later regenerated]** Edit/regenerate flows on courses exist; if Ask answers ever become editable, `parent_quote` could become orphaned text → Mitigation: today Ask answers are immutable; if that changes, store `parent_turn_answer_snapshot_at` timestamp alongside.
- **[Carried citations leak corpus state]** If the parent turn's source list is large, prompt context bloats → Mitigation: cap carried citations at the N (e.g. 5) most relevant overlapping the selection.
- **[Composes with deep-research-mode but spec-tested separately]** Branched + deep is the obvious power-user combo; needs explicit verification but no design conflict — the deep loop just runs as the first turn of the branched thread.

## Migration Plan

1. Schema: `CREATE TABLE ask_thread_parents (thread_id INTEGER NOT NULL REFERENCES ask_threads(id), parent_thread_id INTEGER NOT NULL REFERENCES ask_threads(id), parent_turn_id INTEGER NOT NULL REFERENCES ask_turns(id), parent_quote TEXT, created_at INTEGER NOT NULL DEFAULT (strftime('%s','now')))` plus `CREATE INDEX idx_ask_thread_parents_thread`, `CREATE INDEX idx_ask_thread_parents_parent`, and `CREATE UNIQUE INDEX idx_ask_thread_parents_single_v1` in `courses_db.py` migration block. No backfill (table is new).
2. DAO: extend `create_ask_thread` to accept and persist optional parent fields; add `get_ask_thread_parents`, `get_ask_thread_children`, `get_ask_thread_descendants`.
3. Webhook: extend `/ask` request model; add parent handling in thread-creation path; enrich `/threads/{id}` response with `parents[]` array; add `GET /threads/{id}/children` and `GET /threads/{id}/descendants`.
4. Routine: extend `routines/ingest-ask.md` JSON input contract with optional `parent_context` block; mirror in local skill.
5. PWA: turn-level Branch button, selection-level "Ask about this" action, parent breadcrumb chip + quote block, immediate children list at thread bottom, tree-collapse Threads index with sessionStorage state.
6. Rollback: env flag `THREAD_BRANCHING_ENABLED=false` hides PWA affordances and rejects `/ask` requests with parent fields (HTTP 400).
7. Future DAG enable: `DROP INDEX idx_ask_thread_parents_single_v1`, add `POST /threads/{id}/parents` endpoint, expand PWA to allow merging branches.

## Open Questions

- Do branched threads inherit the parent thread's depth tier by default in PWA UI, or always reset to standard?
- For selection-level branching with no overlapping citations, do we still seed prompt context with the parent answer's full citation list, or none?
- `/threads/{id}/descendants` response size is unbounded for deep trees — do we add a `?max_depth=N` parameter from day one?
- Should the tree-collapse state be persisted in localStorage (across browser restarts) instead of sessionStorage?
