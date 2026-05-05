---
name: deep-research
description: Parallel supervisor/researcher web-research flow. Plans sub-questions, fans out Agent subagents with WebSearch+WebFetch, synthesizes a cited markdown report, journals progress for resume. Writes output to $REPORTS_DIR.
---

You are running `/deep-research`. Produce a structured, cited markdown research report.

## Arguments

`$ARGUMENTS` parsed in any order, remaining free text is the query:
- `<query>` (required) — the research question
- `--thread <id>` — resume/continue an existing thread (reuse plan journal)
- `--model <haiku|sonnet|opus>` — override subagent model for this run (default from `$SUBAGENT_MODEL`, else `haiku`)

## Environment

- `REPORTS_DIR` — reports output dir (required)
- `RUNS_DIR` — plan journal dir (default `$REPORTS_DIR/.runs`)
- `MAX_SUBQ` — max sub-questions (default 5)
- `MAX_WEB_PER_SUBQ` — per-researcher web-action budget (default 6)
- `SUBAGENT_MODEL` — default subagent model if `--model` not passed (default `haiku`)

## Flow

### 1. Resolve thread id

Parse `$ARGUMENTS`. If `--thread <id>` present → `THREAD_ID=<id>`, resume mode. Else mint new: `uuidgen | tr A-Z a-z`. Keep `QUERY` as the remaining text.

### 2. Load or create plan journal

Journal path: `$RUNS_DIR/<THREAD_ID>.json`. Shape:

```json
{
  "thread_id": "…",
  "query": "…",
  "plan": [{"id":"sq1","question":"…"}],
  "status": {"sq1":"pending"},
  "findings": {},
  "started_at": "<iso8601>",
  "updated_at": "<iso8601>"
}
```

If journal exists and has any `pending|failed` entries → resume mode: skip planning, fanout only those. If all `done` → synthesize only. If absent → plan.

### 3. Plan (new runs only)

Decompose `QUERY` into 3–`MAX_SUBQ` sub-questions. Each must be independently answerable with 1–3 web searches. Prefer orthogonal angles (definition / mechanism / evidence / counter / applications) over near-duplicates. Write the journal atomically (tmp file + rename).

### 4. Fanout

Spawn one `Agent` call per `pending|failed` sub-question **in a single assistant message** so they run in parallel. Use `subagent_type: general-purpose` and `model: <model>` where `<model>` comes from `--model` flag or `$SUBAGENT_MODEL` (default `haiku`). Haiku 4.5 is plenty for "search + fetch + summarize to JSON"; upgrade to `sonnet` or `opus` when source rigor matters more than quota. Reserve the supervisor (this skill itself) for planning and synthesis — never downgrade the supervisor.

Prompt template:

```
Research this sub-question and return ONLY a JSON object.

Sub-question: <question>
Parent query: <QUERY>
Budget: at most <MAX_WEB_PER_SUBQ> web actions (WebSearch + WebFetch combined).

SOURCES
- Prefer PRIMARY sources: peer-reviewed papers, official docs, first-party publications, named-author reports, recognized institutions.
- Use SECONDARY aggregators (blogs, Substacks, listicles, odds-comparison sites, content-farm articles) only when no primary source exists. Never cite tout sites, SEO aggregators, or AI-generated content farms.
- Prefer recent (last 3 years) where recency matters; older foundational papers are fine for theory.

QUOTES (hard rule)
- Every citation.quote MUST be a verbatim byte-for-byte excerpt from the page you fetched via WebFetch. No paraphrasing, no summarizing, no synthesis inside the quote field. <=200 chars.
- If you cannot find a <=200 char verbatim excerpt supporting your summary, drop that citation rather than fabricate a quote.

OUTPUT
Return ONLY the JSON below. No markdown code fences. No prose before or after. No extra keys.

{
  "summary_md": "150-300 words, factual, no citations inline",
  "citations": [{"url":"","title":"","quote":"verbatim <=200 chars"}],
  "entities": ["named things worth tracking"],
  "relations": [{"from":"","to":"","verb":""}],
  "confidence": "high|medium|low"
}
```

### 5. Gather

For each returned payload, extract the JSON object robustly:

1. Strip leading prose/explanation.
2. If output is wrapped in a ```` ```json ... ``` ```` or ```` ``` ... ``` ```` fence, strip the fence markers (common Haiku behavior despite the instruction).
3. Locate the first `{` and last `}` and JSON-parse the slice between them.
4. Validate required keys: `summary_md`, `citations`, `entities`, `relations`, `confidence`.
5. Validate each `citations[].quote` is ≥10 chars and appears to be a real extract (not obviously paraphrased meta-description). If a quote looks synthesized, drop that citation but keep the rest of the payload.

On success: update journal `status[id]=done`, `findings[id]=payload`. On parse failure, empty summary, or missing keys: retry once with the same prompt prefixed by "Your previous response was not parseable JSON. Return ONLY the JSON object, no fences, no prose." If retry fails → `status[id]=failed`, record `findings[id]={"error":"…"}`. Rewrite journal atomically after each update.

### 6. Synthesize

Aggregate citations across all sub-questions, dedupe by URL, assign global `[n]` numbers. Merge `entities` / `relations` lists, dedupe.

**Source-quality pass before writing the report:**

- Tag each source as `primary` or `secondary`:
  - **primary** — peer-reviewed journals (doi.org, pubmed, pmc.ncbi.nlm.nih.gov, arxiv, jstor, sciencedirect, springer, frontiersin, nature, wiley, etc.), official org publications (first-party .org/.gov/.edu documents, named-author reports from recognized institutions like GIIN, ANDE, NSCA, BIS, IMF), first-party company/product docs, authored books.
  - **secondary** — news aggregators, blogs, Substacks, generic listicles, odds-comparison sites, SEO content farms, Wikipedia (usable as orientation but downweight).
- If >50% of sources are `secondary`, add a `⚠ low source rigor — N/M sources are secondary aggregators; consider a follow-up run with --model sonnet` line to the report's Notes section AND a `source_rigor: low` field in the frontmatter.
- Drop citations whose quote field was already flagged as suspicious in Step 5; note dropped count in Notes.

Write `$REPORTS_DIR/<THREAD_ID>.md`:

```markdown
---
query: "<QUERY>"
thread_id: <THREAD_ID>
created_at: <iso8601>
entities: [<deduped>]
relations:
  - {from: "", to: "", verb: ""}
---

# <short title from QUERY>

## Summary

<1 paragraph synthesis across all sub-questions.>

## Findings

### <sub-question 1 text>

<summary_md rewritten with inline [n] citations where claims map to sources.>

### <sub-question 2 text>

…

## Sources

1. [<title>](<url>)
2. …

## Notes

<Any `failed` sub-questions called out explicitly — do not silently drop.>
```

### 7. Emit

Print exactly two lines to stdout:

```
thread_id=<THREAD_ID>
report=<REPORTS_DIR>/<THREAD_ID>.md
```

Then stop. No chatter.

## Rules

- Never call the Anthropic API directly. Only WebSearch, WebFetch, Bash, Read, Write, Edit, Glob, Grep, Agent.
- No `curl` for scraping web content — WebFetch only (respects robots.txt, consistent parsing).
- Cite every factual claim with `[n]` matching the Sources list.
- `citations[].quote` is a verbatim excerpt from the fetched page — never paraphrase or synthesize inside a quote field. A dropped citation is better than a fabricated one.
- Journal writes atomic: write `<path>.tmp` then `mv`.
- `failed` sub-questions surface in the report's Notes section.
- Low source rigor (>50% secondary) surfaces in Notes and frontmatter `source_rigor: low`.
- Do not overwrite an existing report if all sub-questions were already `done` and the report file exists — idempotent.
