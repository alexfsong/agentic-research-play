## 1. Test scaffolding

- [ ] 1.1 Add dev dependencies to `~/research-webhook/`: `pytest`, `pytest-asyncio`, `pytest-mock`, `pytest-socket` (network blocker)
- [ ] 1.2 Create `pytest.ini` (or `[tool.pytest.ini_options]` in `pyproject.toml`) registering `eval` marker and configuring `asyncio_mode = "auto"`
- [ ] 1.3 Create `tests/__init__.py`, `tests/unit/__init__.py`, `tests/eval/__init__.py`
- [ ] 1.4 Create `tests/conftest.py` with shared fixtures stubs

## 2. Shared fixtures

- [ ] 2.1 Fixture `mem_courses_db`: in-memory SQLite, runs migration block, yields connection/DAO
- [ ] 2.2 Fixture `mock_anthropic`: monkeypatch `courses._client` to return a stub whose `messages.create` returns canned responses keyed by call-site label
- [ ] 2.3 Fixture `temp_llamaindex_dir`: tmp_path-backed LlamaIndex/ChromaDB store, no overlap with production `~/research-data/`
- [ ] 2.4 Fixture `sample_retrieval_results`: deterministic top-k chunks for pipeline tests

## 3. DAO unit tests (`tests/unit/test_courses_db.py`)

- [ ] 3.1 Round-trip test for every public function in `courses_db.py` (write → read → assert equal)
- [ ] 3.2 Foreign-key cascade test (delete course → lessons gone)
- [ ] 3.3 Idempotent migration test (run migration twice, no error)
- [ ] 3.4 ask_threads / ask_turns DAO tests (since same module per `[[ask_pivot_architecture]]`)

## 4. Parser unit tests (`tests/unit/test_parsers.py`)

- [ ] 4.1 `_strip_code_fence` happy path + no-fence + nested fences
- [ ] 4.2 `_parse_json` valid JSON + fenced JSON + malformed → raises typed error
- [ ] 4.3 Course-plan tool-use payload: valid → typed object; missing field → typed error names the field
- [ ] 4.4 Lesson-plan tool-use payload: valid + invalid Bloom tag rejected
- [ ] 4.5 Claim list tool-use payload: valid + bad citation index rejected

## 5. Pipeline orchestration unit tests (`tests/unit/test_pipeline.py`)

- [ ] 5.1 `generate_course` with canned plan response writes one course row + N lesson rows
- [ ] 5.2 `generate_lesson_claims` with canned response writes claims with correct lesson FK
- [ ] 5.3 Retry/backoff path: first call raises rate-limit, second succeeds → final state correct
- [ ] 5.4 Empty retrieval path: pipeline degrades gracefully (no crash, course marked `failed` with reason)

## 6. Eval fixture corpus (`tests/eval/fixtures/corpus/`)

- [ ] 6.1 Pick a focused topic (suggest: FSRS spaced repetition) and check in 5-10 short Markdown docs
- [ ] 6.2 Each doc ≤ 5 KB, includes URL/title metadata in frontmatter so citation tests can resolve
- [ ] 6.3 Document corpus topic + provenance in `tests/eval/fixtures/CORPUS.md`

## 7. Eval rubric (`tests/eval/test_course_quality.py`)

- [ ] 7.1 `pytest -m eval` test: build temp LlamaIndex from fixture corpus, run real `generate_course`, capture output
- [ ] 7.2 Deterministic checks: every cited URL resolves to a fixture doc, Bloom-tag diversity ≥ 2, JSON shape valid, claim count per lesson within bounds
- [ ] 7.3 LLM-as-judge: held-out model scores course-objective coverage (1-5), claim atomicity (1-5), lesson coherence (1-5); test fails if any score < 3
- [ ] 7.4 Hard token budget per eval run: assert total `usage` across all calls under `EVAL_MAX_TOKENS` env (default 200k)

## 8. CI

- [ ] 8.1 Create `~/research-webhook/.github/workflows/unit-tests.yml`: triggers on push + PR, installs deps, runs `pytest -m "not eval"`
- [ ] 8.2 Add `--disable-socket --allow-hosts=127.0.0.1` to CI pytest invocation to prove unit suite is hermetic
- [ ] 8.3 Add manual `eval.yml` workflow_dispatch workflow that runs `pytest -m eval` with `ANTHROPIC_API_KEY` from secrets
- [ ] 8.4 Do NOT enable branch protection in v1 (advisory only); add a follow-up note to revisit branch protection after suite stabilizes

## 9. Docs

- [ ] 9.1 Add `~/research-webhook/TESTING.md`: how to run unit, how to run eval, expected eval cost, how to regenerate canned mock responses when prompts change
- [ ] 9.2 Update `~/research-webhook/README.md` with link to `TESTING.md` and a "tests required for prompt changes" note

## 10. Verification

- [ ] 10.1 Run unit suite locally on a laptop with no API key and no network — passes
- [ ] 10.2 Run eval suite once on box with real key — passes, captures cost figure for docs
- [ ] 10.3 Intentionally break a parser, confirm a unit test goes red with a useful message
- [ ] 10.4 Intentionally regress a prompt to produce single-Bloom-level lessons, confirm eval catches it
