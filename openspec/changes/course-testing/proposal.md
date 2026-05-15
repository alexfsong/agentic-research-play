## Why

Course generation is the most expensive and most prompt-sensitive surface in the stack: backward-design pipeline runs ~6+ Claude calls per course (plan → per-lesson scaffold → per-lesson claims), persists structured JSON across multiple tables, and silently produces lower-quality courses when prompts drift, models change, or retrieval shifts. Today we have zero automated tests — every regression is caught by a human noticing a bad lesson. Need both (a) cheap unit gates that catch parsing/DAO/contract breakage in CI on every webhook commit, and (b) an opt-in eval harness that scores end-to-end course quality on a fixed corpus when prompts/models change.

## What Changes

- Add a `tests/` tree to `~/research-webhook/` with `pytest` + `pytest-asyncio` configured.
- **Unit layer** (`tests/unit/`): mock the Anthropic client (`anthropic.AsyncAnthropic`) and exercise `courses.py` pipeline functions, JSON tool-use parsers, `_strip_code_fence`, `_parse_json`, Bloom-tag validation, lesson-objective shape, claim extraction shape, and `courses_db.py` DAO writes against an in-memory SQLite. No network, no LLM spend, runs in seconds.
- **Eval layer** (`tests/eval/`): a small fixed corpus (5-10 docs checked into `tests/eval/fixtures/corpus/`) plus a `pytest -m eval` suite that runs the real pipeline end-to-end against a temporary LlamaIndex/ChromaDB store, then scores outputs on a rubric (course objective coverage, lesson-objective Bloom diversity, citation grounding, claim atomicity) using LLM-as-judge with a held-out model.
- CI: GitHub Actions on `~/research-webhook` repo runs unit suite on every push. Eval suite is manual (`make eval` or workflow_dispatch) since it costs API tokens.
- Add `tests/conftest.py` fixtures: in-memory `courses_db`, mocked `AsyncAnthropic`, ephemeral LlamaIndex dir, sample retrieval results.
- Document running tests in `~/research-webhook/README.md` (or new `TESTING.md`).

## Capabilities

### New Capabilities
- `course-test-suite`: Unit + eval test infrastructure for the course generation pipeline, with deterministic mocks for unit tests and rubric scoring for eval runs.

### Modified Capabilities
<!-- No prior specs in openspec/specs/. Leave empty. -->

## Impact

- **Repo**: New `tests/` tree in `~/research-webhook/` (Hetzner repo, not this repo). New `pytest.ini` / `pyproject.toml` test config. New dev deps: `pytest`, `pytest-asyncio`, `pytest-mock`, `respx` (or anthropic mock).
- **CI**: New GitHub Actions workflow on `research-webhook` repo for unit suite. No CI on this repo (per `[[box_infra_conventions]]`, two-repo split).
- **Cost**: Unit suite = $0. Eval suite = ~1-2 full course generations per run (~$0.50-$2 depending on models per `[[anthropic_auth_split]]`). Run rarely (pre-deploy on prompt changes).
- **Schema**: No production schema changes. Eval suite uses ephemeral SQLite fixture.
- **Touches**: `[[course_layer_architecture]]` (testing the in-webhook course code path), `[[anthropic_auth_split]]` (eval runs need `ANTHROPIC_API_KEY`).
- **Out of scope**: PWA component tests, ingest path tests, deep-research-loop tests (separate change).
