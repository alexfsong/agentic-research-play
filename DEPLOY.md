# DEPLOY: Ask-pivot (ODR → routines/skill)

Migration runbook for replacing the LangGraph/ODR Ask flow with the routine-and-skill pipeline. Two repos involved:

- **`agentic-research-play`** (this repo) — routine + skill markdown, cloud-side flow docs.
- **`research-webhook`** (`github.com/alexfsong/research-webhook`) — FastAPI app, schema, PWA, systemd unit. Lives on the box at `/home/researcher/research-webhook/`.

Box conventions are in `research-webhook/INFRA.md`; follow them. Caddy at `lisearch.195-201-99-206.sslip.io` proxies to `127.0.0.1:8000`. No Tailscale needed — Caddy + bearer token is the perimeter.

## What changes (recap)

1. PWA "Ask" tab posts `{question, mode?, thread_id?}` to `POST /ask`.
2. Webhook mints `run_id`, fires `ingest-ask` cloud routine.
3. On routine pool exhaust → falls back to `subprocess claude -p "/ingest-ask <json>"` as `claude-runner` (subscription OAT).
4. Routine/skill posts each fetched doc to `/ingest`, then aggregate to `/ask_callback`.
5. Webhook handles callback → runs `/synthesize2` against enriched corpus → stores cited answer in run state.
6. PWA polls `GET /research/{run_id}` and renders answer + route badge.
7. Threads tab reframed as **Conversations** (`ask_threads` + `ask_turns` tables; continuation = `POST /threads/{id}/ask`).
8. ODR (`research-agent.service`, `~/open_deep_research/`, `/research`, old `/threads*`) decommissioned. Reclaim ~3 GB RAM.

## Prereqs

- New `WEBHOOK_API_KEY` and `ANTHROPIC_OAT` rotated and pasted into `/home/researcher/research-webhook/.env` AND the Anthropic Routines workspace secrets panel.
- Workspace secret `WEBHOOK_URL=https://lisearch.195-201-99-206.sslip.io` (public hostname; routines run on Anthropic infra and can't reach `127.0.0.1`).
- Cloud routine `ingest-ask` already created (trigger URL is in `.env` → `ROUTINE_INGEST_ASK_FIRE_URL`).
- Root access available via Hetzner Cloud Console → "Console" tab (or `ssh root@195.201.99.206` per INFRA.md).

---

## Part A — Changes in the `research-webhook` repo

Apply locally, push to `main`, then `git pull` on the box. Do not edit files in place on the VPS.

```bash
# locally
git clone https://github.com/alexfsong/research-webhook
cd research-webhook
git checkout -b ask-pivot
```

### A1. Schema delta in `courses_db.py`

Append a new migration block (keep idempotent — guard with `_MIGRATIONS` table check):

```python
_MIGRATIONS.append(("ask_threads_v1", """
CREATE TABLE IF NOT EXISTS ask_threads (
  id          TEXT PRIMARY KEY,
  title       TEXT NOT NULL,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);
CREATE TABLE IF NOT EXISTS ask_turns (
  id                  TEXT PRIMARY KEY,
  thread_id           TEXT NOT NULL REFERENCES ask_threads(id) ON DELETE CASCADE,
  idx                 INTEGER NOT NULL,
  question            TEXT NOT NULL,
  route               TEXT NOT NULL,                  -- 'cloud' | 'local'
  run_id              TEXT NOT NULL,
  ingested_doc_ids    TEXT NOT NULL DEFAULT '[]',
  answer_md           TEXT,
  citations_json      TEXT NOT NULL DEFAULT '[]',
  created_at          TEXT NOT NULL,
  UNIQUE(thread_id, idx)
);
CREATE INDEX IF NOT EXISTS idx_ask_turns_thread ON ask_turns(thread_id, idx);
CREATE INDEX IF NOT EXISTS idx_ask_turns_run ON ask_turns(run_id);
"""))
```

DAO functions to add:

- `create_ask_thread(title) -> id` (uses `secrets.token_hex(8)`, prefix `thr_`).
- `get_ask_thread(id) -> {id, title, turns: [...]}`.
- `list_ask_threads(limit=50) -> [{id, title, updated_at, turn_count}]`.
- `add_ask_turn(thread_id, question, route, run_id) -> turn_id` — auto-increments `idx`, bumps thread `updated_at`.
- `update_ask_turn(turn_id, *, ingested_doc_ids=None, answer_md=None, citations_json=None)`.
- `update_ask_turn_route(turn_id, route)`.
- `get_ask_turn_question(turn_id) -> str`.
- `delete_ask_thread(id)` — cascades.

### A2. Webhook handlers in `webhook.py`

New env vars (already in `.env`): `ROUTINE_INGEST_ASK_FIRE_URL`, `CLAUDE_FALLBACK_USER`, `CLAUDE_FALLBACK_TIMEOUT`, `ASK_DEFAULT_MAX_FETCHES`, `ASK_RUN_TTL`. Reuse existing `ANTHROPIC_OAT`, `ROUTINE_FIRE_BETA`, `WEBHOOK_API_KEY`.

#### POST /ask

```python
class AskRequest(BaseModel):
    question: str
    mode: Literal["auto", "cloud", "local"] = "auto"
    thread_id: Optional[str] = None
    max_fetches: int = int(os.getenv("ASK_DEFAULT_MAX_FETCHES", "10"))
    urls: Optional[List[str]] = None
    topic: Optional[str] = None

_runs: dict[str, dict] = {}
_runs_lock = asyncio.Lock()
_pool_full_at: float = 0.0   # epoch seconds; non-zero = recent 429

@app.post("/ask")
async def ask(req: AskRequest, _=Depends(check_auth)):
    if not req.question.strip() and not req.urls:
        raise HTTPException(400, "question or urls required")

    thread_id = req.thread_id or courses_db.create_ask_thread(
        title=_summarize_title(req.question)
    )
    run_id = "ask_" + secrets.token_hex(8)
    payload = {
        "run_id": run_id,
        "question": req.question,
        "thread_id": thread_id,
        "max_fetches": req.max_fetches,
        "topic": req.topic or "",
        "urls": req.urls or [],
    }

    route = await _pick_route(req.mode)
    turn_id = courses_db.add_ask_turn(thread_id, req.question, route, run_id)

    async with _runs_lock:
        _runs[run_id] = {
            "run_id": run_id, "route": route, "status": "pending",
            "thread_id": thread_id, "turn_id": turn_id,
            "ingested": [], "skipped": [], "errors": [], "synthesis": None,
            "created_at": _now_iso(),
        }

    if route == "cloud":
        asyncio.create_task(_fire_routine(payload, run_id))
    else:
        asyncio.create_task(_fire_local_skill(payload, run_id))

    _audit_log({"run_id": run_id, "route": route,
                "question_prefix": req.question[:80], "ts": _now_iso()})
    return {"run_id": run_id, "thread_id": thread_id, "turn_id": turn_id, "route": route}


async def _pick_route(mode: str) -> str:
    if mode == "cloud":
        return "cloud"
    if mode == "local":
        return "local"
    # auto: prefer cloud unless we recently saw the routine pool full
    if time.time() - _pool_full_at < 60:
        return "local"
    return "cloud"


async def _fire_routine(payload: dict, run_id: str):
    global _pool_full_at
    url = os.environ["ROUTINE_INGEST_ASK_FIRE_URL"]
    headers = {
        "Authorization": f"Bearer {os.environ['ANTHROPIC_OAT']}",
        "Content-Type": "application/json",
        "anthropic-beta": os.environ.get(
            "ROUTINE_FIRE_BETA", "experimental-cc-routine-2026-04-01"
        ),
    }
    async with httpx.AsyncClient(timeout=30) as client:
        r = await client.post(url, headers=headers,
                              json={"text": json.dumps(payload)})
    pool_full = (
        r.status_code in (429, 503)
        or "pool" in r.text.lower()
        or "capacity" in r.text.lower()
    )
    if pool_full:
        _pool_full_at = time.time()
        await _fire_local_skill(payload, run_id, fallback_from_cloud=True)
        return
    if r.status_code >= 400:
        await _fail_run(run_id, f"routine fire {r.status_code}: {r.text[:200]}")


async def _fire_local_skill(payload: dict, run_id: str,
                             fallback_from_cloud: bool = False):
    if fallback_from_cloud:
        async with _runs_lock:
            _runs[run_id]["route"] = "local"
        courses_db.update_ask_turn_route(_runs[run_id]["turn_id"], "local")

    user = os.environ["CLAUDE_FALLBACK_USER"]
    timeout_s = int(os.environ.get("CLAUDE_FALLBACK_TIMEOUT", "240"))
    arg = json.dumps(payload)
    cmd = [
        "sudo", "-n", "-u", user, os.environ["CLAUDE_BIN"],
        "-p", f"/ingest-ask {arg}",
        "--allowedTools", "WebSearch,WebFetch,Bash",
        "--permission-mode", "bypassPermissions",
    ]
    env = {
        "WEBHOOK_URL": "http://127.0.0.1:8000",
        "WEBHOOK_API_KEY": os.environ["WEBHOOK_API_KEY"],
        "PATH": "/usr/local/bin:/usr/bin:/bin",
        "HOME": f"/home/{user}",
    }
    proc = await asyncio.create_subprocess_exec(
        *cmd, env=env,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        await asyncio.wait_for(proc.wait(), timeout=timeout_s)
    except asyncio.TimeoutError:
        proc.kill()
        await _fail_run(run_id, f"local skill timed out after {timeout_s}s")
```

#### POST /ask_callback

```python
class AskCallback(BaseModel):
    run_id: str
    route: Literal["cloud", "local"]
    status: Literal["complete", "failed"]
    ingested: list = []
    skipped: list = []
    errors: list = []

@app.post("/ask_callback")
async def ask_callback(cb: AskCallback, _=Depends(check_auth)):
    async with _runs_lock:
        run = _runs.get(cb.run_id)
        if not run:
            raise HTTPException(404, "unknown run_id")
        run["ingested"] = cb.ingested
        run["skipped"] = cb.skipped
        run["errors"] = cb.errors
        run["status"] = "synthesizing"

    try:
        question = courses_db.get_ask_turn_question(run["turn_id"])
        syn = await _synthesize2_internal(question=question, k=8,
                                          rerank=True, subq=False)
        async with _runs_lock:
            run["synthesis"] = syn
            run["status"] = ("complete" if cb.status == "complete"
                             else "failed")
        courses_db.update_ask_turn(
            run["turn_id"],
            ingested_doc_ids=[i.get("doc_id") for i in cb.ingested],
            answer_md=syn["answer"],
            citations_json=json.dumps(syn["citations"]),
        )
    except Exception as e:
        async with _runs_lock:
            run["status"] = "failed"
            run["errors"].append({"message": f"synthesis failed: {e}"})

    return {"ok": True}
```

`_synthesize2_internal` is whatever helper your existing `POST /synthesize2` route already calls — extract the body into a reusable function so both routes share it.

#### GET /research/{run_id} (extend existing)

Already serves the run dict. Add `synthesis` and `route` to the returned shape — they're just dict keys, no contract change for existing callers.

#### Threads endpoints (rewrite)

Replace existing LangGraph-backed handlers; source of truth = `ask_threads` + `ask_turns`.

```python
@app.get("/threads")
async def list_threads(limit: int = 20, _=Depends(check_auth)):
    return {"threads": courses_db.list_ask_threads(limit=limit)}

@app.get("/threads/{thread_id}")
async def get_thread(thread_id: str, _=Depends(check_auth)):
    t = courses_db.get_ask_thread(thread_id)
    if not t:
        raise HTTPException(404, "no such thread")
    return t

@app.post("/threads/{thread_id}/ask")
async def thread_ask(thread_id: str, req: AskRequest, _=Depends(check_auth)):
    req.thread_id = thread_id
    return await ask(req)

@app.delete("/threads/{thread_id}")
async def delete_thread(thread_id: str, _=Depends(check_auth)):
    courses_db.delete_ask_thread(thread_id)
    return {"ok": True}
```

#### Code to remove

- `from langgraph_sdk import get_client` and any LangGraph imports.
- `POST /research` handler (the LangGraph kick-off).
- Old `/threads*` LangGraph handlers (replaced above).
- `POST /threads/{id}/continue` (LangGraph re-research wrapper).
- Any background poller of `langgraph runs.get` state.
- Env consumers of `LANGGRAPH_URL`, `LANGGRAPH_ASSISTANT`, `RUN_TIMEOUT_SECONDS`.

### A3. PWA — `static/index.html` + `static/app.js`

Nav (in `index.html`):

```html
<nav class="tabs">
  <a href="#/learn">Learn</a>
  <a href="#/ask">Ask</a>
  <a href="#/threads">Conversations</a>
  <a href="#/search">Search</a>
</nav>
<!-- Reports / Synthesize tabs removed.
     Stats + raw reports stay accessible from a footer link. -->
```

Ask tab (`app.js`):

```js
async function viewAsk() {
  const root = document.querySelector("#app");
  root.innerHTML = `
    <h2>Ask</h2>
    <textarea id="ask-q" rows="4" placeholder="What do you want to know?"></textarea>
    <div class="row">
      <label><input type="radio" name="mode" value="auto" checked> Auto</label>
      <label><input type="radio" name="mode" value="cloud"> Cloud quota</label>
      <label><input type="radio" name="mode" value="local"> My subscription</label>
    </div>
    <button id="ask-go">Ask</button>
    <div id="ask-status" class="meta"></div>
    <div id="ask-answer"></div>`;
  document.querySelector("#ask-go").onclick = submitAsk;
}

async function submitAsk(thread_id) {
  const q = document.querySelector("#ask-q").value.trim();
  const mode = document.querySelector('input[name="mode"]:checked').value;
  if (!q) return;
  const r = await api("POST",
    thread_id ? `/threads/${thread_id}/ask` : "/ask",
    { question: q, mode });
  document.querySelector("#ask-status").textContent =
    `via ${r.route} • ingesting…`;
  pollAsk(r.run_id);
}

async function pollAsk(runId, intervalMs = 2000, maxMs = 240000) {
  const t0 = Date.now();
  while (Date.now() - t0 < maxMs) {
    const r = await api("GET", `/research/${runId}`);
    if (r.status === "complete") {
      document.querySelector("#ask-status").textContent =
        `via ${r.route} • +${r.ingested.length} fetched`;
      renderAnswer(r.synthesis);
      return;
    }
    if (r.status === "failed") {
      document.querySelector("#ask-status").textContent =
        `failed: ${(r.errors[0] || {}).message || "unknown"}`;
      return;
    }
    if (r.status === "synthesizing") {
      document.querySelector("#ask-status").textContent =
        `via ${r.route} • synthesizing…`;
    }
    await new Promise(res => setTimeout(res, intervalMs));
  }
  document.querySelector("#ask-status").textContent = "timed out";
}

function renderAnswer(syn) {
  if (!syn) return;
  document.querySelector("#ask-answer").innerHTML = `
    <div class="report">${marked.parse(syn.answer)}</div>
    <details><summary>Citations (${syn.citations.length})</summary>
      <ol>${syn.citations.map(c =>
        `<li>${c.title || c.node_id}</li>`).join("")}</ol>
    </details>`;
}
```

Conversations tab — replace the LangGraph `viewThreads`/`viewThread`:

```js
async function viewThreads() {
  const r = await api("GET", "/threads?limit=50");
  // render r.threads (each: id, title, updated_at, turn_count)
  // each row → location.hash = `#/threads/${id}`
}

async function viewThread(id) {
  const t = await api("GET", `/threads/${id}`);
  // render t.turns as Q/A cards (question, route, answer_md, citations_json)
  // bottom of page: textarea + mode radio + "Continue" → submitAsk(id)
}
```

Drop `viewSynthesize`, `viewReports`, and the LangGraph "Continue thread" handler.

### A4. requirements.txt

Drop `langgraph-sdk` (and any langgraph-only deps). Keep everything else.

### A5. systemd unit — `deploy/research-webhook.service`

Edit the committed unit file:

```ini
[Unit]
Description=Research webhook API (FastAPI)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=researcher
Group=researcher
WorkingDirectory=/home/researcher/research-webhook
EnvironmentFile=/home/researcher/research-webhook/.env
ExecStart=/home/researcher/research-webhook/.venv/bin/uvicorn webhook:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Two changes:
- Drop `research-agent.service` from `After=`.
- `ExecStart` venv path moves from shared `~/open_deep_research/.venv` → per-app `~/research-webhook/.venv` (per INFRA.md convention).

### A6. INFRA.md updates

In the same PR, update INFRA.md:

- "Existing apps" table: keep `research-webhook`. Drop the open_deep_research / research-agent line if listed (currently no entry — ODR was off-doc; no change).
- Persistent state table: nothing changes.
- Hard rules: nothing changes.
- Add a brief "Ops" section about `claude-runner`:

> ### Subscription fallback runner
>
> A locked-down `claude-runner` Linux user holds a Claude Code install logged in
> with a Pro/Max subscription OAT. The webhook invokes it via
> `sudo -n -u claude-runner /home/claude-runner/.npm-global/bin/claude -p ...`
> (NOPASSWD limited to that single binary) when the cloud routine pool is
> exhausted. Slash command lives at `/home/claude-runner/.claude/commands/ingest-ask.md`
> (NOT `skills/` — `commands/` is the dir Claude Code resolves `/name` against).
> Sudoers needs `Defaults>claude-runner env_keep += "WEBHOOK_URL WEBHOOK_API_KEY"`
> or the skill exits with "WEBHOOK_URL not set". Subprocess args:
> `--allowedTools "WebSearch,WebFetch,Bash" --permission-mode bypassPermissions`.
> Rotate the OAT every 90 days (`sudo -i -u claude-runner claude logout` → `claude login`).

### Push

```bash
git commit -am "Ask-pivot: routine + skill replace ODR; ask_threads schema; PWA Ask/Conversations"
git push -u origin ask-pivot
# open PR, merge to main
```

---

## Part B — Box-side ops

### B0. Confirm rotated secrets are live

```bash
ssh researcher@195.201.99.206
grep -E '^(WEBHOOK_API_KEY|ANTHROPIC_OAT)=' /home/researcher/research-webhook/.env | wc -l   # → 2
sudo systemctl restart research-webhook
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer <new_key>" \
  https://lisearch.195-201-99-206.sslip.io/health   # → 200
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer wrong" \
  https://lisearch.195-201-99-206.sslip.io/stats    # → 401
```

### B1. Set up `claude-runner` (root)

Log in as root via Hetzner Cloud Console or `ssh root@195.201.99.206`:

```bash
adduser --disabled-password --gecos "" claude-runner

# Node available on the box (skip apt install if `node -v` already works)
apt update && apt install -y nodejs npm

# Per-user npm-prefix install (cleaner blast radius than `sudo npm -g`)
sudo -i -u claude-runner bash <<'INNER'
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=$HOME/.npm-global/bin:$PATH' >> ~/.bashrc
export PATH=$HOME/.npm-global/bin:$PATH
npm install -g @anthropic-ai/claude-code
which claude    # → /home/claude-runner/.npm-global/bin/claude
INNER

# Login as claude-runner once — interactive
sudo -i -u claude-runner
claude login    # paste the OAT or use the device-code flow
exit            # back to root

# Allow researcher to run `claude` only as claude-runner.
# env_keep is REQUIRED — without it sudo strips WEBHOOK_URL/WEBHOOK_API_KEY.
cat > /etc/sudoers.d/claude-runner <<'EOF'
Defaults>claude-runner env_keep += "WEBHOOK_URL WEBHOOK_API_KEY"
researcher ALL=(claude-runner) NOPASSWD: /home/claude-runner/.npm-global/bin/claude
EOF
chmod 440 /etc/sudoers.d/claude-runner
chown root:root /etc/sudoers.d/claude-runner
visudo -c                                  # parses ALL of /etc/sudoers.d/

# Drop the slash command into claude-runner's commands dir.
# NOTE: `commands/`, NOT `skills/` — Claude Code only fires `/name` syntax
# against ~/.claude/commands/<name>.md. Skills (~/.claude/skills/<name>/SKILL.md)
# are a different mechanism, auto-loaded by the model rather than invoked
# explicitly.
sudo -u claude-runner mkdir -p /home/claude-runner/.claude/commands
# Paste the contents of agentic-research-play/.claude/commands/ingest-ask.md
sudo -u claude-runner tee /home/claude-runner/.claude/commands/ingest-ask.md > /dev/null <<'EOF'
<paste full slash-command markdown including frontmatter>
EOF
sudo -u claude-runner chmod 600 /home/claude-runner/.claude/commands/ingest-ask.md

# Add CLAUDE_BIN to /home/researcher/research-webhook/.env (must match sudoers Cmnd byte-for-byte):
#   CLAUDE_BIN=/home/claude-runner/.npm-global/bin/claude
```

Verify (back as `researcher`):

```bash
# 1. NOPASSWD rule resolves
sudo -n -u claude-runner /home/claude-runner/.npm-global/bin/claude -p "echo hi" 2>&1 | tail -5
# 2. Slash command is registered
sudo -n -u claude-runner /home/claude-runner/.npm-global/bin/claude -p "/help" 2>&1 | grep -i ingest-ask
# 3. End-to-end (in another shell: `sudo journalctl -u research-webhook -f`)
KEY=$(sudo grep ^WEBHOOK_API_KEY /home/researcher/research-webhook/.env | cut -d= -f2-)
sudo -n -u claude-runner \
  env WEBHOOK_URL=http://127.0.0.1:8000 WEBHOOK_API_KEY="$KEY" \
  /home/claude-runner/.npm-global/bin/claude \
  -p '/ingest-ask {"run_id":"smoke1","question":"what is FSRS?","thread_id":"t1","max_fetches":1}' \
  --allowedTools "WebSearch,WebFetch,Bash" \
  --permission-mode bypassPermissions
# Webhook journal should show POST /ingest (200) then POST /ask_callback (200).
```

If step 3's POST fails with "WEBHOOK_URL not set" → `env_keep` line missing or
sudoers not reloaded. If Bash tool calls fail with "exit code 1" → forgot
`--permission-mode bypassPermissions` (default mode auto-denies tool prompts in
non-interactive `-p` mode).

### B2. Migrate the venv (researcher)

```bash
cd /home/researcher/research-webhook

python3.11 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -r requirements.txt

# Sanity import
.venv/bin/python -c "import fastapi, anthropic, llama_index.core, chromadb; print('ok')"
```

### B3. Pull the ask-pivot changes (researcher)

```bash
cd /home/researcher/research-webhook
git fetch && git pull --ff-only
ls deploy/research-webhook.service
```

### B4. Swap the systemd unit (root)

```bash
cp /home/researcher/research-webhook/deploy/research-webhook.service \
   /etc/systemd/system/research-webhook.service
systemctl daemon-reload
systemctl restart research-webhook
systemctl status research-webhook
```

Watch logs:

```bash
journalctl -u research-webhook -n 100 --no-pager
```

Confirm the new endpoints are live:

```bash
curl -s https://lisearch.195-201-99-206.sslip.io/openapi.json \
  | jq '.paths | keys' | grep -E '/ask|/threads'
# Expect: "/ask", "/ask_callback", "/threads", "/threads/{thread_id}", "/threads/{thread_id}/ask"
```

### B5. End-to-end verification

```bash
KEY=<new_webhook_api_key>
HOST=https://lisearch.195-201-99-206.sslip.io

# 1. Cloud route
RUN=$(curl -s -X POST "$HOST/ask" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"question":"what is FSRS","mode":"cloud"}' | jq -r .run_id)
# Poll
for i in $(seq 1 60); do
  S=$(curl -s -H "Authorization: Bearer $KEY" "$HOST/research/$RUN")
  echo "$S" | jq -r '.status'
  [ "$(echo "$S" | jq -r '.status')" = "complete" ] && break
  sleep 3
done
echo "$S" | jq '.synthesis.answer | .[0:200]'

# 2. Local route
curl -s -X POST "$HOST/ask" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"question":"what is FSRS","mode":"local"}' | jq .

# 3. Continuation
TID=$(echo "$S" | jq -r .thread_id)
curl -s -X POST "$HOST/threads/$TID/ask" \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"question":"how does FSRS differ from SM-2?","mode":"auto"}' | jq .

# 4. Schema check
sqlite3 /home/researcher/research-data/courses.db ".schema ask_threads"
sqlite3 /home/researcher/research-data/courses.db ".schema ask_turns"
sqlite3 /home/researcher/research-data/courses.db \
  "SELECT id, title, (SELECT COUNT(*) FROM ask_turns WHERE thread_id=ask_threads.id) AS turns FROM ask_threads;"
```

If any of (1)–(3) fails, **stop here** and debug. Do not proceed to ODR decommission.

### B6. PWA smoke test

Open `https://lisearch.195-201-99-206.sslip.io/` in Safari/Chrome:

- Sign in with the new bearer token.
- **Learn** tab loads existing courses. No regressions.
- **Ask** tab: ask a question with each mode, watch the badge flip and the answer render.
- **Conversations** tab: previous Ask shows up; tap → see Q/A; continue with another turn.
- **Search** tab: still works.

### B7. Decommission ODR (only after B5 + B6 pass)

```bash
# As root
systemctl stop research-agent
systemctl disable research-agent
rm /etc/systemd/system/research-agent.service
systemctl daemon-reload

# As researcher
rm -rf /home/researcher/open_deep_research

# Reclaim check
free -h
df -h /home
```

ODR's old reports under `~/research-data/reports/*.md` stay on disk — they're already in the LlamaIndex corpus. Nothing references them at runtime.

### B8. Rotate the bak

```bash
# Once everything is verified stable for a few days
rm -rf /home/researcher/research-agent-tools.bak.20260501
```

---

## Rollback

Cheap before B7:

```bash
# As root: revert systemd unit to old venv + research-agent After=
git -C /home/researcher/research-webhook checkout main~1 -- deploy/research-webhook.service
cp /home/researcher/research-webhook/deploy/research-webhook.service \
   /etc/systemd/system/research-webhook.service
systemctl daemon-reload && systemctl restart research-webhook

# As researcher: git revert ask-pivot merge
cd /home/researcher/research-webhook
git revert <merge-sha>
git push
```

Schema migration is additive (`CREATE TABLE IF NOT EXISTS …`) — no rollback needed; tables can stay empty.

After B7: rollback means re-cloning ODR (`git clone https://github.com/langchain-ai/open_deep_research`) + redoing the original provisioning. Painful — be confident before B7.

## Failure-mode quick checks

| Symptom | First thing to look at |
|---------|------------------------|
| `/ask` returns `500` immediately | `journalctl -u research-webhook -n 50` — usually missing env or SQLite lock. |
| `/ask_callback` 404s | `_runs` was evicted past `ASK_RUN_TTL`. Bump TTL or speed up the routine. |
| Cloud route `route` flips to `local` always | Check if Anthropic returned 429 / "pool" / "capacity" in `_fire_routine` — the sniff covers status code OR body. |
| Local route subprocess hangs | `pgrep -af claude` to see if claude is stuck. Kill it; check `sudo -u claude-runner claude -p "echo hi"` still works. |
| PWA "Conversations" empty after Ask | `sqlite3 ... "SELECT * FROM ask_threads;"` — was the thread row written? If yes, frontend bug; if no, DAO bug. |
| Caddy serves `502` | Webhook crashed: `systemctl status research-webhook`, then logs. |
