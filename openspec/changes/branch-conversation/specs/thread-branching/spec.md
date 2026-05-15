## ADDED Requirements

### Requirement: Branch from a prior turn
The system SHALL allow a user to create a new `ask_thread` from any prior `ask_turn`, recording the linkage in `ask_thread_parents`.

#### Scenario: Turn-level branch creates linked thread
- **WHEN** the PWA POSTs `/ask` with `parent_thread_id=T` and `parent_turn_id=U`
- **THEN** a new `ask_thread` is created and one row is inserted into `ask_thread_parents` with `parent_thread_id=T`, `parent_turn_id=U`, `parent_quote=NULL`, and the response returns the new thread_id

#### Scenario: Branch first turn is seeded with parent context
- **WHEN** the cloud routine (or local skill fallback) runs the first turn of a branched thread
- **THEN** the prompt context includes the parent turn's question and a reference to its answer

### Requirement: Branch from a highlighted span
The system SHALL allow a user to highlight a text span inside an answer body and create a new `ask_thread` seeded with that span. The `parent_quote` is stored verbatim with no length cap enforced at the API layer.

#### Scenario: Selection-level branch persists quote
- **WHEN** the PWA POSTs `/ask` with `parent_thread_id=T`, `parent_turn_id=U`, and `parent_quote="..."`
- **THEN** a new `ask_thread_parents` row stores all three values and the first turn's prompt context includes the quoted span verbatim

#### Scenario: Citations from quoted span carried forward
- **WHEN** the highlighted span overlaps one or more `[n]` citation markers in the source answer
- **THEN** the new turn's prompt context includes those resolved source URLs/titles so the new turn can cite them too

### Requirement: Parent linkage exposed via API
The system SHALL expose parent linkage on `GET /threads/{id}` as a `parents` array (DAG-ready, length ≤ 1 in v1) and provide `GET /threads/{id}/children` to list branched descendants.

#### Scenario: GET /threads/{id} includes parents array
- **WHEN** a client GETs a branched thread
- **THEN** the response body includes `parents: [{thread_id, turn_id, quote?}]` with one entry (with `quote` omitted when null)

#### Scenario: GET /threads/{id} for non-branched thread returns empty parents
- **WHEN** a client GETs a thread with no row in `ask_thread_parents`
- **THEN** the response includes `parents: []` (always present, never omitted)

#### Scenario: GET /threads/{id}/children lists direct children
- **WHEN** a client GETs `/threads/{id}/children`
- **THEN** the response is an array of immediate-child thread summaries (id, first-turn question, created_at, has_quote flag)

### Requirement: Tree shape enforced in v1, schema is DAG-ready
The system SHALL allow at most one `ask_thread_parents` row per `thread_id` in v1 (enforced by a unique index), so the API surface remains a tree even though the underlying schema can hold multiple parents.

#### Scenario: Single-parent invariant enforced
- **WHEN** any code path attempts to insert a second `ask_thread_parents` row for the same `thread_id`
- **THEN** the SQLite UNIQUE constraint rejects the insert and the webhook surfaces HTTP 400

#### Scenario: Re-parenting forbidden
- **WHEN** a client attempts to update parent linkage on an existing `ask_thread`
- **THEN** the operation is rejected (no API surface exists to perform it)

### Requirement: PWA renders parent breadcrumb and branch affordances
The system SHALL render a parent breadcrumb on branched-thread detail pages and a "Branch" affordance on every turn in the Threads view.

#### Scenario: Parent breadcrumb shown
- **WHEN** the user opens a branched thread's detail page
- **THEN** the header shows a breadcrumb chip linking back to the parent thread, with the `parent_quote` rendered as a quote block when present

#### Scenario: Turn-level branch button visible per turn
- **WHEN** the user views any turn in any thread
- **THEN** a "Branch" action is visible and creates a new branched thread when activated

#### Scenario: Selection branch action appears on highlight
- **WHEN** the user highlights any non-empty text span inside an answer body
- **THEN** an "Ask about this" floating action appears anchored near the selection

### Requirement: Threads index renders branched threads as collapsible tree
The system SHALL render the Threads index as a tree where branched threads appear nested under their parent thread, with per-node collapse/expand controls.

#### Scenario: Branched thread nested under parent
- **WHEN** the user opens the Threads index
- **THEN** any thread with a row in `ask_thread_parents` appears nested under its parent thread (one indentation level per nesting depth)

#### Scenario: Collapse hides descendants
- **WHEN** the user collapses a parent thread node in the index
- **THEN** all descendant branched threads (transitive) are hidden until expanded again

#### Scenario: Collapse state persists per session
- **WHEN** the user navigates away from the Threads index and returns within the same PWA session
- **THEN** previously collapsed nodes remain collapsed
