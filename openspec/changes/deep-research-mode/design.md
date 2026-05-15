## Context

Ask today is a single-pass routine: PWA → `POST /ask` → cloud routine `ingest-ask` (or local skill on pool exhaust) → 1× WebSearch → ~5 fetches → POST each to `/ingest` → draft cited Markdown → `/ask_callback` → render. Works well for shallow lookups; struggles when answering requires iteratively refining queries (e.g. "what's the current state of FSRS-rs vs anki-rs and how do they handle concurrent reviews?"). Users have no escape hatch.

Deep research mode adds a third tier on the same plumbing rather than a parallel system. Reuses corpus, ingest path, citation format, threading model, OAT-vs-API-key auth split (see `[[anthropic_auth_split]]`), and PWA Threads view.

Constraints:
- Webhook code lives on Hetzner (`~/research-webhook/`), separate repo (see `[[box_infra_conventions]]`). Changes ship there, not in this repo.
- Routine concurrency=1 today. Deep turns can take minutes — must not block fast/standard.
- Cost: every deep turn is ≫ a standard turn in tokens + web fetches. Need explicit user opt-in and visible cost signal.
- No new infra: must stay on existing FastAPI + LlamaIndex + ChromaDB stack.

## Goals / Non-Goals

**Goals:**
- One control in PWA picks standard or deep on every Ask (slider/segmented control sized to leave room for `fast` later).
- Deep produces a sectioned long-form report with TOC + per-section citations.
- Deep loop is gap-driven with a hard iteration/token ceiling.
- Both tiers share the same `ask_turns` row shape (depth column added) so thread history is uniform.
- Cloud routine and local skill fallback stay parity.

**Non-Goals:**
- No new ranking model or graph-RAG additions (those gated by `LI_GRAPH_ENABLED` separately).
- No new persisted "report" entity — report payload lives inside the existing `ask_turns.answer` column as JSON when depth=deep, with a `payload_shape` discriminator.
- No background streaming UI in v1 — deep turn returns when done; PWA polls thread until then.
- No automatic depth selection (no router LLM picks tier for the user).

## Decisions

### Decision: Reuse `ask_turns.answer` with shape discriminator over a new `ask_reports` table
- **Choice:** Add `payload_shape TEXT NOT NULL DEFAULT 'flat'` to `ask_turns`. Flat = today's Markdown string. `report` = JSON `{toc, sections[]}` serialized into the same `answer` column.
- **Rationale:** Threads render uniformly, no join, fewer migrations. Sectioned report is just a richer answer.
- **Alternative considered:** Separate `ask_reports` table keyed by turn_id. Rejected — doubles read paths in PWA + webhook for marginal gain.

### Decision: Depth threaded as explicit parameter, not derived from `max_iterations` magnitude
- **Choice:** `depth: 'standard'|'deep'` in request, expanded server-side to a budget tuple from env. (`fast` reserved for a future tier.)
- **Rationale:** UX-stable contract, env tuning doesn't break PWA. Future tiers (`fast`, `exhaustive`?) just add an enum value.
- **Alternative:** Pass raw budgets from client. Rejected — leaks billing knobs to UI, lets clients DOS the box.

### Decision: Ship v1 with two tiers, defer `fast`
- **Choice:** v1 = `standard` (current behavior) + `deep` (new). `fast` deferred until standard+deep prove out.
- **Rationale:** Less PWA + test surface. `fast` is mostly a smaller-budget knob on `standard` and can be added with one enum entry + new env vars without API changes.
- **Alternative:** Ship all three at once. Rejected — adds UI surface and a third set of budgets to tune for marginal v1 value.

### Decision: Loop orchestration in `webhook.py`, not in the routine prompt
- **Choice:** Cloud routine and local skill produce one round of work; webhook orchestrates the loop, calls back into the routine (or runs the synthesize step in-process via Anthropic SDK) for next-round queries.
- **Rationale:** Loop control + budget enforcement + dedup live in code, not in a prompt that drifts. Routine prompt stays declarative.
- **Alternative:** Loop inside the routine. Rejected — bigger blast radius from prompt changes, no central budget enforcement, dedup state hard to track across routine invocations.

### Decision: Synthesizer + gap identification in one tool-use call per round
- **Choice:** Each round invokes one Anthropic call whose tool-use schema returns both the round's section drafts and the next-round `gap_queries[]`. No separate gap-only call.
- **Rationale:** Halves per-round LLM cost. The model already has the partial state in context, asking it to also enumerate gaps is cheap; cleanly separating the steps would re-pay context tokens.
- **Alternative:** Separate gap call. Rejected — independently tunable but ~2× cost for marginal quality gain. If gap quality regresses, can split later without API changes.

### Decision: Deep concurrency = 1 in-flight per bearer, dedicated routine slot
- **Choice:** Per-bearer counter in webhook tracks in-flight deep runs; second deep submission from same bearer returns 429. Deep runs go through `ROUTINE_ASK_DEEP_FIRE_URL` (separate from `ROUTINE_ASK_FIRE_URL` used by standard) so deep work doesn't sit in the standard concurrency=1 queue.
- **Rationale:** Multi-user fairness without serializing across users. Same-bearer cap prevents one user firing many parallel deep runs and blowing budget.
- **Alternative A:** 1 globally. Rejected — second user always waits. **Alternative B:** unlimited per bearer. Rejected — invites per-user cost runaway.

### Decision: Cost signal in PWA = static per-tier badge, not live estimate
- **Choice:** Show "Standard — default", "Deep — minutes, ~$X-Y" labels. No dynamic estimate in v1.
- **Rationale:** Live estimation requires a router or pre-flight call; not worth complexity. Label sets expectation.

## Risks / Trade-offs

- **[Cost runaway]** A user repeatedly firing deep turns burns tokens fast → Mitigation: per-day deep-turn count cap per bearer (env: `ASK_DEPTH_DEEP_MAX_PER_DAY`), 429 over cap.
- **[Long-running deep blocks Caddy/uvicorn worker]** → Mitigation: deep work runs in a background task, `/ask` returns the run_id immediately and PWA polls (same pattern as `/courses` per `[[course_layer_architecture]]`).
- **[Citation drift across iterations]** Corpus grows mid-loop, citation indices may renumber → Mitigation: snapshot the citation map at draft time per section; assemble final TOC from the per-section maps.
- **[Routine prompt + webhook drift]** Loop logic in webhook means routine prompt stays small, but two paths (cloud + local) still diverge → Mitigation: same orchestrator path runs whichever produced the round, single drafter step.
- **[Cap hits feel like silent failure]** → Mitigation: report includes a `termination` field (`"empty_gaps"` | `"iteration_cap"` | `"token_cap"`) rendered in PWA footer of the report.

## Migration Plan

1. Schema: `ALTER TABLE ask_turns ADD COLUMN depth TEXT NOT NULL DEFAULT 'standard'; ADD COLUMN payload_shape TEXT NOT NULL DEFAULT 'flat';` in `courses_db.py` migration block. Backfill existing rows = no-op (defaults match historical behavior).
2. Webhook: add depth handling to `/ask`, deep orchestrator module, `/ask_callback` `report` shape, deep-tier rate limit.
3. Routine: extend `routines/ingest-ask.md` with depth-aware prompt branches; ship cloud routine; mirror local skill in `.claude/skills/ingest-ask/`.
4. PWA: depth control on Ask tab, sectioned-report renderer in Threads view, dual-shape thread API.
5. Env on box: add `ASK_DEPTH_*` vars, `ROUTINE_ASK_DEEP_FIRE_URL`, deep-per-day cap.
6. Rollback: env-flag `ASK_DEPTH_ENABLED=false` short-circuits `/ask` to ignore `depth` and run today's behavior.

## Open Questions

- Per-section SSE streaming is a v2 candidate (v1 is polling, matching `/courses`). Worth the new endpoint + PWA SSE handling once deep usage volume justifies it.
- Should the report payload also persist `gap_queries[]` from each round (debug/audit trail), or only the final assembled sections?
- For the bearer in-flight counter: process-local map (lost on restart) or persist in SQLite? Process-local is simpler; restart "leaks" stuck slots until the timeout reaper runs.
