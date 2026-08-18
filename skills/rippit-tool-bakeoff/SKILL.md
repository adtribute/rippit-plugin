---
name: rippit-tool-bakeoff
description: >-
  Run a controlled head-to-head ("bake-off") between two or more analysis tools
  (e.g. the Rippit MCP vs. a Snowflake/SQL MCP, or any data/connector MCP) on the
  SAME deep-dive question, using isolated Cowork sub-agents, then present each tool's
  output and the process behind it side by side in the chat so the user can judge for
  themselves. Use when the user
  wants to compare tools/MCPs, benchmark connectors, run a tool bake-off, see how
  different tools answer the same question, or A/B two analysis engines on real data.
  Triggers: "compare X MCP and Y MCP", "bake-off", "which tool is better at", "run
  the same analysis with both tools", "head-to-head", "benchmark these connectors".
---

# Tool Bake-off

Run the same analytical question through two or more tools in parallel, each handled
by an **independent, isolated Cowork sub-agent** that cannot see the others' work,
then present each tool's results — output and process — side by side **directly in the
chat**, so the user can see exactly how the tools differ and decide for themselves
which they trust. This produces a defensible, apples-to-apples view of "how does each
tool answer this question?" — with the *difference* between the outputs unmistakable.

## Inputs (collect these first)

The user should provide two things when they invoke the skill. If either is missing,
ASK for it (use the AskUserQuestion tool, one concise question per missing item)
before doing anything else:

1. **Tools to compare** — the **Rippit MCP is ALWAYS one side of the comparison**;
   never ask whether to include it, just include it. Ask only for the OTHER tool(s)
   to compare it against (e.g. Snowflake/SQL, another connector, or a variant such
   as "Snowflake raw + Cortex" vs "Snowflake raw, no Cortex"). Confirm the exact MCP
   server identifiers so each sub-agent can be locked to its tool.
2. **The question** — the deep-dive the user wants answered (e.g. "why are customers
   reaching out over the past 3 months?").

Results are presented in the chat — do NOT write to or ask for any external doc
(Notion, Google Docs, Slack, a file). The whole comparison surfaces inline.

Do not start work until both are known.

## Step 1 — Confirm scope

Restate the tools and the question in one line so the user can correct you. Identify
each tool's MCP server prefix (e.g. `mcp__<server-id>__*`) so sub-agents can be made
**tool-pure**.

## Step 2 — Beef up the question (then proceed)

A thin question produces thin, hard-to-differentiate answers. Deepen the user's
question into a rigorous brief by layering in the dimensions below **that are
relevant to their question** — apply them tool-agnostically (the goal is a better
analysis, not a rigged one):

- **Coverage + reproducibility** — aim to analyze all of the relevant records; but if
  the question spans so many records that full coverage is impractical, the tool may
  sample, and should choose a sensible sample (e.g. representative, not just whatever's
  newest or easiest). Either way it must be upfront about exactly what it covered (all
  records, or a sample of what size and how chosen) so the user can weigh it. Results
  should be reproducible run-to-run.
- **Structured taxonomy / segmentation** — if the question is "why/what kind",
  require a multi-level, mutually-exclusive taxonomy with dynamic depth and the
  volume/% per bucket; where useful, break results down by segment (e.g. tier, ARR
  band, tenure, region, product area).
- **Severity dimensions** — urgency, business impact, and sentiment/CSAT impact per
  finding, not just counts.
- **Evidence with receipts** — for each theme, representative verbatim quotes with
  links/IDs to source records, plus the single most severe example.
- **Quotes for color** — weave representative customer quotes into the analysis to
  add color and human texture to the quantitative story, not just as an appendix.
- **Root cause vs. symptom** — classify the underlying driver (product gap, doc gap,
  policy, training, expected behavior) and what would have prevented it.
- **Trend** — what's rising, falling, or newly emerging vs. the prior period.
- **Closed-loop action** — a tactical, implementable set of top recommendations
  (product / policy / agent training), and where supported, drafted artifacts
  (tickets, help articles, coaching guidance) and tagged records.

Briefly tell the user you're enriching their question — something like *"Adding a few
extra angles to your question so it gets answered as fully as possible"* — then show
the beefed-up brief and **proceed straight to dispatching the sub-agents**. Do not ask
for approval or pause; presenting the brief is a heads-up, not a checkpoint. (If the
user objects or edits after seeing it, adjust and re-run.)

## Step 3 — Dispatch one isolated sub-agent per tool (in parallel)

Launch the sub-agents in a single message so they run concurrently. Each sub-agent
gets the **identical brief** and these hard rules in its prompt:

- **Tool-pure:** it may use ONLY its assigned tool's MCP for data (give the exact
  `mcp__<server-id>__*` prefix). It must NOT use any other data tool, and must find
  the data within its own tool.
- **Isolation:** it works alone and must not assume any other analysis exists.
- **Return, don't write:** it must NOT write its report to any external doc (no Notion,
  Google Docs, Slack, file). It composes its full report and **returns it as its final
  message** to the orchestrator, which surfaces it in the chat.
- **Format for the depth it found:** it chooses whatever formatting best fits the depth
  and shape of its own answer — tables, nested taxonomy, callouts, quote blocks,
  whatever conveys its findings most clearly. The format itself is a signal: a tool that
  found rich, multi-axis structure should look richer than one that found a flat list.
  Do not force a shared template; let each report's structure reflect what the tool
  actually produced.
- **Deferred tools:** remind it to load tool schemas via ToolSearch
  (`select:<toolname>`) before calling them.
- **No fabrication:** every number must come from its tool's results; it must report
  its methodology, exact coverage (full vs. sample), and candid limitations.
- **Time itself:** it must record a wall-clock timestamp the moment it starts work and
  again when its report is fully written, compute the elapsed time, and include a
  **"Time to insight: Xm Ys"** line near the top of its report. This must be a real
  measured duration, not an estimate — it's the speed signal for the final comparison.
- **Log its own process — in plain language:** it must include a **"How this was done"**
  section that a non-technical reader can follow. The substance still matters — how much
  of the data it actually looked at, whether it read full conversations or just the
  opening message or a summary field, whether it judged each conversation individually
  or grouped them by keywords/labels, and anything that got in the way (it couldn't
  reach some data, had to work from a sample, etc.). But it must say this in everyday
  words and explain *why it matters for trusting the answer*, not list the machinery.
  Drop the jargon: no tool/function names, error codes, parameters, "pagination,"
  "tokens," "enrichment," "DSL," and the like. Translate the mechanics into plain
  outcomes. For example, instead of *"search_conversations paged 12× at 150/page; pages
  exceeded the token cap so I persisted them to temp files and grepped Body: lines;
  ERR_NETWORK_CHANGED on retries; 16% body sample, recency-weighted"*, write something
  like *"I was able to pull exact totals for the whole period, but I could only read the
  full text of about 1 in 6 conversations — the newest ones — because the connection
  kept dropping on larger pulls. So the overall counts are solid, but the breakdown of
  themes is a directional estimate from a recent sample, not an exact read of every
  conversation."* Keep it honest and specific about coverage and limits; just make it
  readable. This section is what lets the user judge how much to trust each report, so
  it can't be a sanitized everything-went-great summary.
- **Permanent artifact:** it must note whether the analysis leaves behind a durable,
  re-openable artifact and link/identify it. For the **Rippit** sub-agent this is the
  **worksheet (or set of worksheets)** it created — capture their IDs/URLs. For other
  tools, note whatever persists (a saved query, a table, a notebook) or state plainly
  that the analysis leaves no artifact beyond the report itself.

Each sub-agent returns its **complete, self-formatted report** as its final message,
which must include: its **time to insight (elapsed duration)**, its plain-language
**"How this was done"** section (coverage, how conversations were analyzed, and any
limits — no jargon), the record count + date range, the full findings (taxonomy,
severity, root cause, trend, recommendations as the brief required), **whether it
produced a permanent artifact (and its ID/link)**, and any limitations it hit.

## Step 4 — Present a side-by-side comparison in the chat

Read every sub-agent's returned report **in full**, then build a single, structured
**side-by-side** comparison **directly in the Cowork chat** — no external doc, and
**no verdict**. The point is for the user to scan the two tools next to each other,
dimension by dimension, and judge for themselves. This is the opposite of dumping two
long reports back to back: pull the findings into shared, parallel structures so the
differences jump out.

Use the section structure below (it mirrors a clean at-a-glance comparison). Label the
two tools consistently throughout (e.g. "A — [Other tool]" and "B — Rippit"). Use real
chat tables for the columnar sections.

1. **Title + question callout.** A title naming the two tools, then a short callout with
   the exact question, the date range analyzed, and one line noting both tools ran the
   *identical* brief in isolation, with no verdict — "compare for yourself."
2. **At a glance** — a table with one row per process/meta dimension and one column per
   tool. Include: **Time to insight**, **Records analyzed**, **Coverage** (full vs.
   sample, stated plainly), **How reasons were classified** (read every conversation vs.
   leaned on an existing label / opening message), **Data hygiene** (what was filtered
   in/out to get to the analyzed set), **Permanent artifact** (Rippit's re-openable
   worksheet vs. none — use a ✅/❌), **Self-reported reliability** (how much each tool
   says to trust its own numbers), and **Friction hit** (what got in the way). Keep every
   cell plain-language, not jargon.
3. **Top [findings] side by side** — a table of the headline findings (e.g. contact
   reasons / themes) with one column per tool showing each tool's number or bucket for
   that row. Because tools often bucket differently, add a short callout right after
   noting the taxonomies aren't 1:1 and *how* each organized its categories — that
   mismatch is itself one of the most revealing differences.
4. **Where they agreed** — a few bullets on the substantive conclusions both tools
   reached independently (genuine agreement across isolated agents is meaningful signal).
5. **Most severe example each surfaced** — a two-column table (or paired blocks) with the
   single worst/most-escalated case each tool pulled, including the verbatim quote and a
   source ID/link. The two will usually differ, which is informative.
6. **Process, in each tool's own words** — a short verbatim excerpt from each tool's
   plain-language "How this was done" section, attributed to that tool. Let each tool
   describe its own method in its own voice.
7. **Artifacts & links** — Rippit's re-openable worksheet link(s); for the other tool,
   state plainly there's no durable artifact (or link whatever persists).
8. **Footer** — one muted line: run date · identical brief, isolated tool-pure agents ·
   presented without a winner.

Do **not** declare a winner, score the tools against each other, or nudge the user
toward one. The side-by-side itself is the deliverable — surface the differences (and
the agreements) faithfully and let the user draw the conclusion. Where one tool covered
a dimension the other couldn't, show that as a plain factual contrast, not as a point
scored. If the user then asks which is better, answer that as a follow-up.

## Principles

- **Fairness is the product.** Identical brief, isolated agents, tool-pure access.
  The comparison is only credible if neither side was advantaged in setup.
- **Present, don't judge.** The skill produces a faithful side-by-side of the two
  tools — each one's output and the process behind it — and stops there. Don't pick a
  winner, score the tools against each other, or steer the user toward one; let them
  decide and ask follow-ups. (If they explicitly ask which is better, that's a fair
  follow-up to answer then.)
- **Normalize the layout, not the substance.** Pull both tools into a shared,
  parallel side-by-side so differences are easy to scan — but never let the tidy layout
  disguise that one tool did less. If a tool sampled, couldn't classify individually, or
  left a cell thin, show that plainly in its column (and in the empty/qualified cells)
  rather than papering over it to make the table look balanced. The shape is shared; the
  honesty about depth and coverage is not sacrificed to it.
- **Show receipts.** Prefer findings backed by verifiable numbers and source links.
- **Surface friction.** Document the operational realities each tool hit (timeouts,
  sampling, reproducibility, schema/permission constraints) — they're often as
  decision-relevant as the analysis quality.
