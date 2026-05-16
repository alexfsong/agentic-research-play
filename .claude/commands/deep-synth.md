---
name: deep-synth
description: One round of the deep-research synthesizer. Headless local skill invoked by the webhook's deep_research orchestrator via `claude -p "/deep-synth <json>"`. Reads a JSON blob describing the question, prior section headings, executed queries, and a numbered list of corpus excerpts; emits one JSON object (no prose) containing 1-3 new section drafts (each with `heading`, `body`, `citation_ns`) and a list of `gap_queries` for the next round. Runs under the operator's Claude Pro/Max subscription so deep-research cost charges against the subscription quota rather than the Anthropic API key.
---

You are running `/deep-synth`. The webhook's `deep_research` orchestrator is calling you in a subprocess to produce ONE round of a multi-round sectioned-report build. Your job is narrow: read structured input, return structured output, exit.

## Arguments

`$ARGUMENTS` is a single JSON blob (passed verbatim from the orchestrator). Parse it with `jq`. Fields:

```bash
PAYLOAD="$ARGUMENTS"
RUN_ID=$(echo "$PAYLOAD" | jq -r '.run_id')
ITER=$(echo "$PAYLOAD" | jq -r '.iteration')
QUESTION=$(echo "$PAYLOAD" | jq -r '.question')
PRIOR_HEADINGS=$(echo "$PAYLOAD" | jq -c '.prior_headings // []')
EXECUTED=$(echo "$PAYLOAD" | jq -c '.executed_queries // []')
EXCERPTS=$(echo "$PAYLOAD" | jq -c '.excerpts')   # array of {n, query, score, text}
```

If parsing fails or `excerpts` is empty/missing → print `{"sections":[],"gap_queries":[],"error":"bad payload"}` and stop.

## Output contract

Print **exactly one JSON object** to stdout, then stop. No commentary, no Markdown fences, no log lines, no second JSON object. The orchestrator parses the last JSON object on stdout.

```json
{
  "sections": [
    {
      "heading": "Short section title",
      "body": "Markdown body. Cite excerpts inline as [n] using the excerpt numbers given.",
      "citation_ns": [1, 3, 7]
    }
  ],
  "gap_queries": [
    "concrete search query that fills a remaining gap",
    "another follow-up query"
  ]
}
```

Rules for the contents:

- **Sections**: 1-3 NEW sections per round. Do NOT repeat any heading in `prior_headings`. Each section must cite at least one excerpt inline as `[n]`, where `n` is the number from the input `excerpts` list.
- **citation_ns**: the set of excerpt numbers referenced in this section's `body`. Orchestrator uses this to snapshot per-section citations.
- **Body**: Markdown only. No HTML. No code blocks unless quoting code from an excerpt. Keep each section tight — 2-5 short paragraphs.
- **Grounding**: every non-trivial claim must trace back to an excerpt. If the excerpts don't support a claim, omit it. No model-prior facts.
- **Gap queries**: concrete search strings that would surface evidence for what's missing. Don't repeat any string in `executed`. Return `[]` (empty list) when you believe the report is complete — that ends the loop.
- **Iteration awareness**: later iterations should refine / fill specific gaps, not re-cover the basics already drafted earlier.

## Algorithm

1. Read inputs.
2. Skim excerpts; decide what 1-3 sections best advance the report next.
3. Draft each section with inline `[n]` citations.
4. List the citation_ns set per section.
5. Identify remaining gaps as concrete search queries (or empty list if done).
6. Print the single JSON object. Exit.

## Rules

- Tools allowed: nothing (no WebSearch, no WebFetch, no Bash, no Read, no Write). The orchestrator runs you with `--allowedTools ""`. Don't request elevated tools.
- Don't print anything except the final JSON object. The orchestrator parses stdout.
- Don't cite excerpts that aren't in the input list.
- Don't fabricate URLs.
- If you can't honestly produce a new section (truly nothing left to add), emit `{"sections":[],"gap_queries":[]}` so the loop terminates with `empty_gaps`.
