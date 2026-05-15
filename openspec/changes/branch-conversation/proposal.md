## Why

Ask threads today are linear: every question hangs off the same thread, mixing tangents with the main line of inquiry. When an answer surfaces something interesting (a referenced concept, a contradicting source, a sub-question), the user has to either pollute the current thread or copy text into a new Ask manually and lose the citation context. Branching is the natural primitive — fork a new thread seeded with the trigger context so deeper analysis can happen without derailing the parent.

## What Changes

- **Turn-level branch action**: every `ask_turn` row in the PWA Threads view gets a "Branch from here" affordance. Tapping creates a new `ask_thread` whose first turn's prompt is seeded with the parent turn's question + answer reference + a "follow up:" prefix, with `parent_thread_id` and `parent_turn_id` set on the new thread.
- **Selection-level branch action**: the user can highlight any text span inside an answer body and a floating "Ask about this" button appears. Tapping creates a new thread seeded with the highlighted text + the answer's citation indices that overlap the selection + a "Going deeper on:" prefix, with `parent_thread_id`, `parent_turn_id`, and `parent_quote` (the selected span) set on the new thread.
- New join table `ask_thread_parents` (thread_id, parent_thread_id, parent_turn_id, parent_quote, created_at) with indexes on both directions. Schema is DAG-ready — multi-parent supported physically — but v1 enforces single parent via a unique index on `thread_id` so the API surface stays tree-shaped. Dropping that unique index later unlocks DAG without a migration.
- New API: `GET /threads/{id}` returns `parents` array (length ≤ 1 in v1; `[{thread_id, turn_id, quote?}, ...]` in DAG-ready shape) when present; `GET /threads/{id}/children` lists immediate child threads.
- PWA Threads index renders branched threads as a tree with collapse/expand under the parent thread (no flat parent-chip fallback).
- Branch creation reuses existing `/ask` request shape — no new endpoint — by adding optional `parent_thread_id`, `parent_turn_id`, `parent_quote` fields.

## Capabilities

### New Capabilities
- `thread-branching`: Fork an Ask thread from a prior turn (turn-level) or from a highlighted span inside an answer (selection-level), with parent linkage persisted and surfaced in PWA navigation.

### Modified Capabilities
<!-- No prior specs in openspec/specs/. Leave empty. -->

## Impact

- **Schema**: New `ask_thread_parents` join table + 3 indexes (forward, reverse, v1 single-parent unique). Migration in `courses_db.py`.
- **Webhook (`~/research-webhook/`)**: `/ask` accepts new optional parent fields; `/threads/{id}` enriched with `parents[]` array; new `/threads/{id}/children` endpoint; DAO updates in `courses_db.py`.
- **PWA**: Threads view gains turn-level "Branch" button + text-selection "Ask about this" floating action; Threads index renders tree with collapse/expand of branched descendants under their parent; new thread detail header shows parent link + quote when present.
- **Composability**: Composes cleanly with deep-research-mode (per `[[deep-research-mode]]` proposal) — branched threads can themselves be deep. Composes with course follow-ups (per `[[course_layer_architecture]]`) but stays separate (lessons keep their own follow-up Q&A; branching is Ask-side only).
- **Touches**: `[[ask_pivot_architecture]]` (extends ask_threads/ask_turns model).
- **Out of scope**: cross-thread search, branched-thread merge. Multi-parent (DAG) branching is schema-ready but API-disabled in v1 (drop the unique index + add DAG endpoints later).
