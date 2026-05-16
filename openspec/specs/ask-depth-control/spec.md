## ADDED Requirements

### Requirement: Ask depth tier selection
The system SHALL accept a `depth` parameter on every Ask request with values `standard` or `deep`, defaulting to `standard` when omitted. (`fast` reserved for a future tier.)

#### Scenario: Default depth applied
- **WHEN** the PWA submits an Ask request with no `depth` field
- **THEN** the webhook treats the request as `standard` and persists `depth='standard'` on the resulting `ask_turns` row

#### Scenario: Invalid depth rejected
- **WHEN** an Ask request arrives with `depth` outside the allowed set
- **THEN** the webhook returns HTTP 422 and does not create an `ask_turns` row

### Requirement: Depth propagated end-to-end
The system SHALL forward the chosen `depth` from PWA through `/ask`, the routine fire URL, the routine prompt, and the local-skill fallback so each layer receives the same tier.

#### Scenario: Routine receives depth
- **WHEN** `/ask` fires the cloud routine for an Ask request with `depth='deep'`
- **THEN** the routine receives `depth='deep'` in its JSON input and the prompt branches into the deep-research loop

#### Scenario: Local skill receives depth
- **WHEN** the routine pool is exhausted and the local skill fallback runs
- **THEN** the skill receives the same `depth` value and uses the matching tier behavior

### Requirement: Per-tier compute budgets
The system SHALL apply per-tier limits configurable via env vars `ASK_DEPTH_<TIER>_MAX_SEARCHES`, `ASK_DEPTH_<TIER>_MAX_FETCHES`, `ASK_DEPTH_<TIER>_MAX_TOKENS`, and `ASK_DEPTH_<TIER>_MAX_ITERATIONS`.

#### Scenario: Standard tier matches today's behavior
- **WHEN** an Ask runs at `depth='standard'`
- **THEN** total web searches MUST NOT exceed `ASK_DEPTH_STANDARD_MAX_SEARCHES` (default 1) and total fetches MUST NOT exceed `ASK_DEPTH_STANDARD_MAX_FETCHES` (default 5)

#### Scenario: Deep tier raises ceiling
- **WHEN** an Ask runs at `depth='deep'`
- **THEN** the loop MUST stop on whichever fires first: gap list empty, `ASK_DEPTH_DEEP_MAX_ITERATIONS` reached (default 8), or `ASK_DEPTH_DEEP_MAX_TOKENS` exhausted

### Requirement: Depth persisted on ask_turns
The system SHALL store `depth` on every `ask_turns` row so threads render with the depth that produced each turn.

#### Scenario: Turn replay shows depth
- **WHEN** the PWA fetches a thread containing turns produced at different depths
- **THEN** each turn carries its own `depth` field in the response

### Requirement: Deep queue per bearer
The system SHALL maintain a persisted FIFO queue per bearer for `deep` Asks, run at most one deep turn in flight per bearer, and surface queue position so the PWA can display "queued, position N" until the drainer picks the run up.

#### Scenario: Deep does not block standard
- **WHEN** a `deep` Ask is in flight (or queued) for bearer B and any `standard` Ask arrives
- **THEN** the `standard` Ask runs without waiting on the `deep` request to finish

#### Scenario: Second deep from same bearer enqueues
- **WHEN** bearer B already has a `deep` Ask in flight and submits another with `depth='deep'`
- **THEN** the webhook accepts the request (HTTP 200), persists the new run with `status='queued'`, and returns the run_id; the run starts after the in-flight run completes

#### Scenario: Queue position visible to the PWA
- **WHEN** the PWA polls `/ask_runs/{run_id}` for a queued deep run
- **THEN** the response includes `status='queued'`, `queue_position` (number of same-bearer deep runs queued ahead of it), and `queue_total` (total queued for this bearer)

#### Scenario: Daily cap enforced at enqueue time
- **WHEN** a bearer has already enqueued `ASK_DEPTH_DEEP_MAX_PER_DAY` deep runs in the current UTC day (any status) and submits another with `depth='deep'`
- **THEN** the webhook returns HTTP 429 with a "deep_daily_cap" error code

#### Scenario: Different bearers can run deep in parallel
- **WHEN** bearer B has a `deep` Ask in flight and bearer C submits a `deep` Ask
- **THEN** both runs proceed concurrently, capped by the global drainer ceiling (`ASK_DEPTH_DEEP_MAX_DRAINERS`)

#### Scenario: Queue survives webhook restart
- **WHEN** the webhook process restarts while one or more deep runs are queued
- **THEN** on startup the queue is reloaded from SQLite and drainers resume in FIFO order per bearer

### Requirement: Deep synthesis charges subscription quota
The system SHALL run each deep-loop synthesizer round through a `claude -p` subprocess (the `/deep-synth` local skill) by default, so per-round LLM cost is billed against the operator's Claude Pro/Max subscription rather than `ANTHROPIC_API_KEY`.

#### Scenario: Subprocess synthesis is default
- **WHEN** `ASK_DEPTH_DEEP_BACKEND` is unset or set to `subscription`
- **THEN** each round of `deep_research.run_deep` invokes `claude -p "/deep-synth <json>"` and parses the printed JSON; the orchestrator does NOT call `anthropic.AsyncAnthropic` directly for the synthesizer step

#### Scenario: API-key override for testing
- **WHEN** `ASK_DEPTH_DEEP_BACKEND=api` is set
- **THEN** the orchestrator uses the Anthropic SDK with `ANTHROPIC_API_KEY` instead, with no implicit fallback when subprocess invocation fails
