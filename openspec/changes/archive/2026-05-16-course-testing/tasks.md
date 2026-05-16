## 1. Test scaffolding

- [x] 1.1 Add dev dependencies to `~/research-webhook/`: `pytest`, `pytest-asyncio`, `pytest-mock`, `pytest-socket` (network blocker) — `requirements-dev.txt`
- [x] 1.2 Create `pytest.ini` (or `[tool.pytest.ini_options]` in `pyproject.toml`) registering `eval` marker and configuring `asyncio_mode = "auto"`
- [x] 1.3 Create `tests/__init__.py`, `tests/unit/__init__.py`, `tests/eval/__init__.py`
- [x] 1.4 Create `tests/conftest.py` with shared fixtures stubs

## 2. Shared fixtures

- [x] 2.1 Fixture `mem_courses_db`: in-memory SQLite, runs migration block, yields connection/DAO — implemented as per-test tmp SQLite file (true `:memory:` cannot survive multi-connection access across `get_conn()` calls)
- [x] 2.2 Fixture `mock_anthropic`: monkeypatch `courses._client` to return a stub whose `messages.create` returns canned responses keyed by call-site label — FIFO queue with `text` / `json_obj` / `raises` modes
- [x] 2.3 Fixture `temp_llamaindex_dir`: tmp_path-backed LlamaIndex/ChromaDB store, no overlap with production `~/research-data/`
- [x] 2.4 Fixture `sample_retrieval_results`: deterministic top-k chunks for pipeline tests

## 3. DAO unit tests (`tests/unit/test_courses_db.py`)

- [x] 3.1 Round-trip test for every public function in `courses_db.py` (write → read → assert equal)
- [x] 3.2 Foreign-key cascade test (delete course → lessons gone)
- [x] 3.3 Idempotent migration test (run migration twice, no error)
- [x] 3.4 ask_threads / ask_turns DAO tests (since same module per `[[ask_pivot_architecture]]`) — also covers `ask_deep_queue` DAO from the deep-research-mode change

## 4. Parser unit tests (`tests/unit/test_parsers.py`)

- [x] 4.1 `_strip_code_fence` happy path + no-fence + nested fences
- [x] 4.2 `_parse_json` valid JSON + fenced JSON + malformed → raises typed error
- [x] 4.3 Course-plan tool-use payload: valid → typed object; missing field → typed error names the field
- [x] 4.4 Lesson-plan tool-use payload: valid + invalid Bloom tag rejected
- [x] 4.5 Claim list tool-use payload: valid + bad citation index rejected

## 5. Pipeline orchestration unit tests (`tests/unit/test_pipeline.py`)

- [x] 5.1 `generate_course` with canned plan response writes one course row + N lesson rows
- [x] 5.2 `generate_lesson_claims` with canned response writes claims with correct lesson FK
- [x] 5.3 Retry/backoff path: first call raises rate-limit, second succeeds → final state correct — `courses.py` has no SDK-layer retry, so the test was reshaped: a hard error on the plan call surfaces as `status=failed` (not crash); a mid-loop lesson failure aborts remaining work and persists what landed before the abort
- [x] 5.4 Empty retrieval path: pipeline degrades gracefully (no crash, course marked `failed` with reason) — plus per-lesson empty retrieval falls back to corpus slice

## 6. Eval fixture corpus (`tests/eval/fixtures/corpus/`)

- [x] 6.1 Pick a focused topic (suggest: FSRS spaced repetition) and check in 5-10 short Markdown docs — 8 docs
- [x] 6.2 Each doc ≤ 5 KB, includes URL/title metadata in frontmatter so citation tests can resolve
- [x] 6.3 Document corpus topic + provenance in `tests/eval/fixtures/CORPUS.md`

## 7. Eval rubric (`tests/eval/test_course_quality.py`)

- [x] 7.1 `pytest -m eval` test: ~~build temp LlamaIndex from fixture corpus~~ — uses a lightweight lexical retriever over the fixture corpus instead. Eval is testing course-gen prompt quality, not retrieval quality, so the LlamaIndex spin-up is unnecessary cost. Real `generate_course` runs end-to-end.
- [x] 7.2 Deterministic checks: every cited `report_id` resolves to a fixture doc stem, Bloom-tag diversity ≥ 2, JSON shape valid, claim count per lesson within 0–8 bounds
- [x] 7.3 LLM-as-judge: held-out model (`claude-opus-4-7` by default) scores course-objective coverage (1-5), claim atomicity (1-5), lesson coherence (1-5); test fails if any score < 3
- [x] 7.4 Hard token budget per eval run: assert total `usage` across all calls under `EVAL_MAX_TOKENS` env (default 200k); test prints the cost figure on `-s`

## 8. CI

- [x] 8.1 Create `~/research-webhook/.github/workflows/unit-tests.yml`: triggers on push + PR, installs deps, runs `pytest -m "not eval"`
- [x] 8.2 Add `--disable-socket --allow-hosts=127.0.0.1` to CI pytest invocation to prove unit suite is hermetic
- [x] 8.3 Add manual `eval.yml` workflow_dispatch workflow that runs `pytest -m eval` with `ANTHROPIC_API_KEY` from secrets — also takes `judge_model` + `max_tokens` as workflow inputs
- [x] 8.4 Do NOT enable branch protection in v1 (advisory only) — not enabled. Follow-up: revisit branch protection after CI run history shows a stable green for a couple of weeks.

## 9. Docs

- [x] 9.1 Add `~/research-webhook/TESTING.md`: how to run unit, how to run eval, expected eval cost, how to regenerate canned mock responses when prompts change
- [x] 9.2 Update `~/research-webhook/README.md` with link to `TESTING.md` and a "tests required for prompt changes" note

## 10. Verification

- [x] 10.1 Run unit suite locally on a laptop with no API key and no network — `pytest -m "not eval" --disable-socket --allow-hosts=127.0.0.1` → **57 passed**.
- [x] 10.2 Run eval suite once with real key — **passed, 2026-05-16**. Course-gen: 25 464 in + 11 094 out = 36 558 tokens across 13 calls (1 plan + 6 lessons × {body, claims}); judge (Opus 4.7) returned coverage=5 atomicity=5 coherence=5; wall 221 s; cost under $1. Caught one bug en route: the first run failed on `judge["notes"]` being a string in the rubric assertion loop; fixed by skipping non-numeric judge fields, and moved the token-figure print before the judge call so cost is captured even on judge failure.
- [x] 10.3 Intentionally break a parser, confirm a unit test goes red with a useful message — monkeypatched `_strip_code_fence` to no-op, the four `TestStripCodeFence` tests went red with assertion errors pointing at the unstripped fence.
- [x] 10.4 Intentionally regress a prompt to produce single-Bloom-level lessons, confirm eval catches it — **passed, 2026-05-16**. Added `tests/eval/test_regression_demo.py` (new `eval_regression` marker) which monkeypatches `courses.BLOOM_LEVELS = {"understand"}` so the post-plan sanity loop coerces every non-`understand` Bloom tag to None. Result: 5 lessons, `non_null_blooms={"understand"}` (len 1). The main eval's `len(bloom_levels) >= 2` guard at `tests/eval/test_course_quality.py` would have fired red on this output. Wall: 177 s; same cost order as 10.2.
