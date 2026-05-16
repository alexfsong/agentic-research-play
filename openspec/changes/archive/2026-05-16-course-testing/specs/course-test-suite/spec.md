## ADDED Requirements

### Requirement: Unit suite runs without network or LLM spend
The system SHALL provide a `tests/unit/` pytest suite that completes with no outbound network calls and no real LLM invocations.

#### Scenario: Network blocked during unit suite
- **WHEN** the unit suite is run with outbound network disabled (e.g. `pytest --disable-socket` or in a sandboxed runner)
- **THEN** all unit tests MUST pass

#### Scenario: Anthropic client mocked
- **WHEN** any test in `tests/unit/` instantiates a course pipeline function
- **THEN** `anthropic.AsyncAnthropic` is replaced by a fixture that returns deterministic canned responses keyed on call site

### Requirement: DAO tests run against in-memory SQLite
The system SHALL provide DAO unit tests that exercise `courses_db.py` against an in-memory SQLite database created per test.

#### Scenario: Each DAO test gets a fresh DB
- **WHEN** a DAO test starts
- **THEN** it receives a freshly migrated in-memory SQLite handle with no rows from prior tests

#### Scenario: DAO round-trips are covered
- **WHEN** the suite runs
- **THEN** every public function in `courses_db.py` has at least one round-trip test (write then read back)

### Requirement: Pipeline-shape tests cover JSON tool-use parsing
The system SHALL include unit tests for the JSON tool-use payload shapes consumed by the course pipeline (course plan, lesson plan, claim list).

#### Scenario: Valid payload parses
- **WHEN** the parser receives a well-formed Anthropic tool-use response for each shape
- **THEN** parsing returns the expected typed structure

#### Scenario: Malformed payload raises a typed error
- **WHEN** the parser receives a malformed payload (missing field, wrong Bloom tag, bad citation index)
- **THEN** parsing raises a domain-specific error and the test asserts the error message includes the offending field

### Requirement: Eval suite runs end-to-end on a fixed corpus
The system SHALL provide a `tests/eval/` suite that runs the real course pipeline end-to-end against a checked-in fixture corpus, gated behind `pytest -m eval`.

#### Scenario: Eval is opt-in
- **WHEN** the default `pytest` command is run with no marker
- **THEN** the eval suite is skipped

#### Scenario: Eval runs on a temp LlamaIndex store
- **WHEN** the eval suite runs
- **THEN** it builds an ephemeral LlamaIndex/ChromaDB store from `tests/eval/fixtures/corpus/`, runs at least one full course generation, and tears the store down afterwards

### Requirement: Eval rubric scores course quality
The system SHALL score every eval-generated course on at least: course-objective coverage, per-lesson Bloom-tag diversity, citation grounding (every cited index resolves to a fixture doc), and claim atomicity.

#### Scenario: Failing course flagged
- **WHEN** the rubric scorer detects a citation index that doesn't resolve to any fixture doc
- **THEN** the eval test fails with a message naming the offending lesson and citation index

#### Scenario: Bloom diversity threshold
- **WHEN** a generated course has fewer than 2 distinct Bloom levels across its lessons
- **THEN** the eval test fails with a diversity warning

### Requirement: CI runs unit suite on every push (advisory in v1)
The system SHALL configure GitHub Actions on the `research-webhook` repo to run the unit suite on every push and pull request. Branch protection is NOT required in v1 — the check is advisory until the suite stabilizes; merge-blocking gets enabled later.

#### Scenario: CI red on unit failure
- **WHEN** any unit test fails
- **THEN** the GitHub Actions check is red and visible on the PR (does not block merge in v1)

#### Scenario: CI does not run eval
- **WHEN** the standard CI workflow runs
- **THEN** the eval suite is not executed (eval is workflow_dispatch only)
