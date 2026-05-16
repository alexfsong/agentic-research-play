## Context

`courses.py` is ~600 lines of orchestration, prompt construction, JSON tool-use parsing, retry/backoff, and DAO calls. Every course gen costs real money and minutes of wall time. There is no test scaffolding today. Failure modes seen so far: silent prompt drift after model bumps, a JSON parser regression that crashed on legitimate fenced code, malformed Bloom tags slipping past validation. None caught by automation.

Two distinct test boundaries are needed because they answer different questions:
- **Unit**: "Does the code still work?" — fast, deterministic, runs on every commit.
- **Eval**: "Does the model still produce decent courses?" — slow, expensive, runs when prompts or models change.

Constraints:
- All work lands in `~/research-webhook/` repo on the Hetzner box (per `[[box_infra_conventions]]`), not in this repo.
- Eval needs `ANTHROPIC_API_KEY` per `[[anthropic_auth_split]]`.
- Unit suite must be runnable on a developer laptop with no API key and no network.
- LlamaIndex / ChromaDB fixtures must not collide with the production `~/research-data/` store.

## Goals / Non-Goals

**Goals:**
- Block obvious regressions in CI without paying for LLM calls.
- Provide a reproducible "does the model still produce a decent course?" check that runs on demand.
- Make adding a new pipeline-shape parser test trivial (single fixture file, single assertion).
- Anthropic mocking strategy that survives SDK version bumps.

**Non-Goals:**
- Not testing the PWA, the ingest pipeline, the routine layer, or `/synthesize2`.
- Not building a CI eval pipeline that runs on every push (cost prohibitive, signal noisy).
- Not replacing human review of generated courses — eval is a guardrail, not an oracle.
- Not testing FSRS scheduling (parked per `[[course_layer_architecture]]`).

## Decisions

### Decision: Mock at the `AsyncAnthropic` boundary, not at HTTP level
- **Choice:** Fixture replaces `courses.AsyncAnthropic` (or the `_client()` factory) with a stub that returns canned `Message` objects keyed by call-site label.
- **Rationale:** Survives Anthropic SDK transport changes (httpx version, base URL changes) without breaking tests. Keeps test fixtures readable (Python objects, not raw JSON HTTP bodies).
- **Alternative:** `respx` HTTP mocking. Rejected — couples tests to wire format, brittle across SDK upgrades.

### Decision: Single `_client()` factory in courses.py, replaced by tests via monkeypatch
- **Choice:** Today `courses.py:41` is `def _client(): return anthropic.AsyncAnthropic()`. Keep that factory; tests monkeypatch it. Clean seam.
- **Rationale:** No production code change needed, no DI framework. Single function to replace.

### Decision: In-memory SQLite for DAO tests, not pytest-postgres
- **Choice:** `sqlite3.connect(":memory:")` per test, run migration block, hand to test.
- **Rationale:** Production uses SQLite (`~/research-data/courses.db`). Same engine = same SQL semantics. Per-test DB = clean isolation, no fixtures to clean up.

### Decision: Eval rubric is hybrid (deterministic + LLM-as-judge)
- **Choice:**
  - **Deterministic checks**: citation indices resolve to fixture docs, Bloom-tag diversity, JSON shape validity, claim count per lesson within bounds.
  - **LLM-as-judge** (held-out model, e.g. Opus when production runs Sonnet): course-objective coverage, claim atomicity, lesson coherence.
- **Rationale:** Deterministic catches the cheap wins. LLM-as-judge handles quality dimensions code can't easily check. Held-out model reduces shared-bias risk.
- **Alternative:** Pure deterministic. Rejected — can't catch "lesson is technically well-formed but says nothing useful". Pure LLM-as-judge. Rejected — flaky, expensive, hard to debug failures.

### Decision: Eval gated by `pytest -m eval`, not separate test runner
- **Choice:** Single pytest invocation, marker filters which suite runs. CI runs `pytest -m "not eval"`. Manual runs use `pytest -m eval`.
- **Rationale:** One install, one config, one test discovery path. Reduces drift between what unit tests assume and what eval tests integrate.

### Decision: CI is advisory in v1 (no merge-block)
- **Choice:** Workflow runs on every push + PR, surfaces red checks, does NOT enable branch protection until the unit suite has stabilized (no flakes for ~2 weeks of normal commits).
- **Rationale:** New test suites always have early-life flakes. Hard-gating from day one trains the team to ignore CI ("just retry"); soft-gating builds trust first. Flip on branch protection once flake rate is provably low.
- **Alternative:** Block from day one. Rejected — premature gate degrades trust in CI.

### Decision: Fixture corpus is small + version-controlled
- **Choice:** 5-10 short Markdown docs (~2-5 KB each) covering one focused topic (e.g. "FSRS spaced repetition"), checked into `tests/eval/fixtures/corpus/`.
- **Rationale:** Reproducible. Small enough that LlamaIndex build is fast (<5s). Topical enough that quality scoring is meaningful.
- **Alternative:** Snapshot of production corpus. Rejected — non-reproducible, leaks user data, drifts.

## Risks / Trade-offs

- **[Mock fixtures rot as prompts evolve]** Prompts change → canned tool-use responses no longer reflect what production returns → unit tests pass but eval still red → Mitigation: keep canned responses as small as possible (just the shape under test), regenerate periodically from real eval runs, document the regen workflow.
- **[Eval scoring is itself an LLM call → flaky]** → Mitigation: run scorer at temperature=0, use held-out model, allow 1 retry on transport error, treat single-run failures as warnings (not red CI), require 2 consecutive failures to be considered a regression in the docs.
- **[Eval cost creep]** Each eval run = 1-2 course generations + N scorer calls → Mitigation: gate behind `pytest -m eval`, document cost per run in `TESTING.md`, set a per-run hard token budget.
- **[Test infra splits attention from product]** Engineers ignore eval, only run unit → Mitigation: pre-deploy checklist mandates eval run when prompts or models change in `courses.py`.

## Migration Plan

1. Add dev deps + `pytest.ini` (or `pyproject.toml [tool.pytest.ini_options]`) with `markers = ["eval: full pipeline runs requiring ANTHROPIC_API_KEY"]`.
2. Create `tests/conftest.py` with shared fixtures: `mock_anthropic`, `mem_courses_db`, `temp_llamaindex_dir`, `sample_retrieval_results`.
3. Land DAO unit tests first (highest signal-to-cost ratio).
4. Land parser unit tests for course plan / lesson plan / claim list shapes.
5. Land pipeline orchestration tests with mocked LLM (e.g. "given canned plan response, generate_course writes correct rows").
6. Land eval fixture corpus + rubric scorer.
7. Add GitHub Actions workflow `unit-tests.yml` running `pytest -m "not eval"`.
8. Document `make eval` command + cost in `TESTING.md`.

## Open Questions

- Do we want a `make test` target in `~/research-webhook/`, or just standard `pytest`?
- Should eval results be logged to a file for trend tracking (regression detection over time), or only printed?
- Which model is "held-out" for LLM-as-judge if production already uses Opus for some calls?
- Do we add a smoke test on the deployed box (post-deploy), or trust CI + manual eval?
