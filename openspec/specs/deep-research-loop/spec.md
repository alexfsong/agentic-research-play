## ADDED Requirements

### Requirement: Iterative gap-driven loop
The system SHALL implement a multi-round research loop for `depth='deep'` that alternates synthesize → identify gaps → re-search → fetch → ingest until a stop condition fires.

#### Scenario: Loop terminates on empty gap list
- **WHEN** the synthesize step returns an empty gap list
- **THEN** the loop exits and the assembler produces the final report

#### Scenario: Loop terminates on iteration cap
- **WHEN** the loop reaches `ASK_DEPTH_DEEP_MAX_ITERATIONS` without an empty gap list
- **THEN** the loop exits with a partial report and notes the cap as the termination reason

### Requirement: Gap identification piggybacks on synthesizer call
The system SHALL produce, in a single Anthropic tool-use call per round, both the partial section drafts and the structured list of follow-up search queries representing remaining knowledge gaps.

#### Scenario: One LLM call yields both draft and gaps
- **WHEN** a round invokes the synthesizer
- **THEN** the tool-use response carries both `sections[]` for that round and `gap_queries[]` for the next round

#### Scenario: Gaps drive next round
- **WHEN** the synthesizer returns N follow-up queries (N > 0)
- **THEN** the next round issues exactly those queries (deduplicated against prior rounds in the same turn)

#### Scenario: Duplicate queries skipped
- **WHEN** a gap query was already executed in an earlier round of the same turn
- **THEN** it is skipped and counted against neither the search nor token budget

### Requirement: Sectioned long-form report output
The system SHALL produce, for `depth='deep'`, a sectioned report with table of contents, per-section body text, and per-section citations rather than a single answer block.

#### Scenario: Report shape on /ask_callback
- **WHEN** a deep Ask completes
- **THEN** the `/ask_callback` payload includes a `report` field containing `{ "toc": [...], "sections": [{ "heading": "...", "body": "...", "citations": [...] }] }` and may omit the flat `answer` field

#### Scenario: PWA renders both shapes
- **WHEN** the PWA fetches a thread containing both flat `answer` turns and sectioned `report` turns
- **THEN** the renderer displays each turn in its native shape without errors

### Requirement: Citations remain corpus-grounded
The system SHALL ensure every claim in every section cites at least one document already in the LlamaIndex corpus, using the same citation format as `/synthesize2`.

#### Scenario: No fabricated citations
- **WHEN** the loop drafts a section
- **THEN** every `[n]` marker resolves to a doc URL/title present in the corpus snapshot at draft time
