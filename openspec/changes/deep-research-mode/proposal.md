## Why

Current Ask flow is a single fast pass: one WebSearch round, fetch top hits, one cited synthesis. Good for quick lookups, weak for hard questions that need iterative gap-filling and a structured long-form report. Users have no way to opt into more compute when the question warrants it.

## What Changes

- Add `depth` parameter (`standard` | `deep`) to the Ask request, exposed in PWA as a segmented control on the Ask tab. (`fast` tier deferred — ship after standard+deep prove out.)
- `standard`: current behavior — 1 search round, ~5 fetches, single cited Markdown answer.
- `deep`: iterative loop (5-10 rounds) of `synthesize partial + identify gaps (single LLM call) → re-search → fetch → ingest` until gap list empty or budget exhausted. Output is an ODR-style sectioned long-form report with table of contents and per-section citations, not a single answer block.
- Thread `depth` end-to-end: PWA → `POST /ask` → routine fire URL query param → `ingest-ask` routine + local skill prompts → `/ask_callback` payload.
- Persist `depth` on the `ask_turns` row (new column) so threads can be re-rendered with the depth that produced them.
- Per-depth budgets (env-tunable): `ASK_DEPTH_<TIER>_MAX_SEARCHES`, `_MAX_FETCHES`, `_MAX_TOKENS`, `_MAX_ITERATIONS`.
- Concurrency: `deep` runs on its own per-bearer slot (cap 1 deep-in-flight per bearer) so a user's deep turn doesn't block their own standard turns or other users' work.

## Capabilities

### New Capabilities
- `ask-depth-control`: User-selectable depth tier on Ask requests, with end-to-end propagation and per-tier compute budgets.
- `deep-research-loop`: Iterative gap-driven research loop that produces a sectioned long-form report instead of a single answer.

### Modified Capabilities
<!-- No prior specs in openspec/specs/ — first feature on the new spec workflow. Leave empty. -->

## Impact

- **Schema**: `ask_turns` gets `depth TEXT NOT NULL DEFAULT 'standard'`. Migration in `courses_db.py`.
- **Webhook (Hetzner `~/research-webhook/`)**: `webhook.py` `/ask` accepts `depth`; routes the deep tier through new orchestrator (loop + section assembler) instead of single-pass synthesizer; `/ask_callback` accepts a `report` payload shape (sections + TOC) in addition to today's flat `answer`.
- **Routines**: `routines/ingest-ask.md` extended with depth-aware prompt and tool budget; new prompt branch for the iterative loop.
- **Local skill**: `.claude/skills/ingest-ask/` mirror updated for fallback parity.
- **PWA**: Ask tab gets depth control; Threads/turn renderer handles both `answer` (flat) and `report` (sectioned) shapes.
- **Cost**: `deep` materially raises per-turn Anthropic + web-fetch spend. Gate behind explicit user choice; surface estimated tier cost in UI.
- **Concurrency**: `deep` may run longer than the current `concurrency=1` rate-limit assumes — separate slot or queue for deep turns to avoid blocking fast/standard.
- **Touches**: `[[anthropic_auth_split]]` (deep loop still uses ANTHROPIC_API_KEY for direct SDK calls), `[[ask_pivot_architecture]]` (extends, doesn't replace), `[[course_layer_architecture]]` (no overlap).
