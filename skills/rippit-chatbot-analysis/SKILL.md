---
name: rippit-chatbot-analysis
description: >-
  Analyze support conversations to find where an AI chatbot underperforms and human
  agents excel, turn those gaps into a prioritized knowledge-base build list, and (when a
  help center is connected) reconcile the gaps against existing documentation. Works with
  any helpdesk, any chatbot, and any help center. Use when the user asks to: analyze
  chatbot / bot gaps, compare bot vs agent (human) performance, run a deflection analysis,
  understand why the bot escalates or hands off, decide what the bot should be able to
  handle, find knowledge-base or documentation gaps, or improve / train the chatbot.
---

# Bot deflection gap analysis

Find where the chatbot fails but humans succeed, and turn that into concrete actions:
new/updated help articles, and (if a help center is connected) a coverage reconciliation
that separates "content doesn't exist" from "content exists but the bot can't find it."

This skill is intentionally **generic**. Customers run different helpdesks (Zendesk,
Intercom, Salesforce, Gladly, Kustomer, Front, …), different chatbots (Fin, Zendesk bot,
Ada, a custom LLM agent, …), and different help centers (Intercom Articles, Zendesk Guide,
a public site, or none). **Never assume** how any of these are wired. Confirm the three
things below before doing analysis.

## Analysis engine

Use the Rippit MCP tools as the analysis engine: `list_data_sources`, `describe_table`,
`describe_column`, `read_table`, `aggregate_table`, `pivot_table`, `create_worksheet`,
`enrich_worksheet` + `get_enrich_status`, `find_quotes`, and `get_report_guide`. The
worksheet-enrich path is the core: it reads each full conversation transcript and adds the
per-row judgments this analysis needs (topic, resolution, satisfaction, escalation driver).

Cost of enrichment scales with **rows, not columns** — every column in one `enrich_worksheet`
call is filled in a single LLM pass per row. So batch all needed columns into ONE call.

---

## MANDATORY: confirm these three things first

Do not skip. Use the `AskUserQuestion` tool (one question per item is fine). These are the
only parts that vary by customer, and getting them wrong invalidates the whole analysis.

### 1. The data source
- Call `list_data_sources` and look for the table of support conversations/tickets. There
  are often several (per-helpdesk tables, prior worksheets, raw tables). **Do not guess.**
- Present the candidate table(s) and ask the user to confirm which one holds the chatbot
  conversations to analyze. If multiple channels exist, confirm whether to filter to one
  source (e.g. only `intercom`) or analyze all.

### 2. How to distinguish bot-handled vs bot-escalated-to-human
This is customer- and tool-specific. `describe_table` / `describe_column` first, then ask
the user to confirm the exact rule. Common signals:
- **Tags** (e.g. `fin handled`, `fin escalated`, `bot resolved`, `handoff`, `agent bot`).
- **A team/assignee field** (bot team vs a human team), or presence of any human agent in
  an `agents`/participants column.
- **An escalation column** or status.
- **Handled by bot only** = the bot engaged and the conversation never reached a human.
- **Escalated** = the bot handed off to a human.

**Verify the rule empirically before trusting it.** A frequent trap: the "bot handled" tag
is a *superset* that also covers escalated conversations (i.e. escalated ⊂ handled). Check
overlap with an `aggregate_table` count using `CONTAINS_ALL` on the two tags. If they
overlap, define **bot-only = handled AND NOT escalated**, and **handoff = escalated**.
Report the two population sizes back to the user and confirm they look right.

### 3. Whether a help center is connected (for doc reconciliation)
- Ask the user whether their help center / knowledge base is connected as a tool (e.g. an
  Intercom, Zendesk Guide, or Help Scout connector with article-listing tools).
- If **yes**: you will run the reconciliation step (compare gaps to existing articles).
- If **no**: skip reconciliation and deliver the gap list + KB build kit only. Offer that
  they can connect the help center later to get the coverage comparison.

Also confirm a **time window** and whether to sample (use judgment; last 30–90 days is
typical). Prefer the full population if it's under ~3–5k conversations; otherwise sample
representatively with `sampleRows`.

---

## Process

### Step A — Scope and size
Confirm the three items above. Then use `aggregate_table` to count bot-only vs handoff
conversations in the chosen window so you can size the enrichment.

### Step B — Build the worksheet
`create_worksheet` over the conversations in scope, filtered to the bot-touched population
(both bot-only and handoff), for the confirmed window. Keep all source columns so nothing
blocks later enrichment. The distinguishing tag/field stays on the worksheet so you can
split the two groups in aggregation later.

### Step C — Enrich (one call, all columns)
Add these columns with `enrich_worksheet` (adapt names/labels to the customer's product,
but keep the intent). Each judgment is made against the **full transcript**.

- **contact_reason** — `unspecified_enum` (~15–20): the customer's primary reason for
  contacting support, as a concise topic.
- **issue_resolved** — enum: `resolved` / `partially_resolved` / `not_resolved` / `unknown`.
  Was the underlying issue actually resolved by the end of the thread?
- **customer_satisfied** — enum: `satisfied` / `neutral` / `dissatisfied` / `unknown`,
  judged from the customer's own words and tone at the end.
- **handled_well** — enum: `excellent` / `adequate` / `poor` / `unknown`: accuracy,
  completeness, empathy, efficiency, independent of who handled it.
- **escalation_driver** — enum: `not_escalated` / `account_specific_action` /
  `knowledge_gap` / `technical_bug` / `complex_multistep` / `customer_demanded_human` /
  `billing_or_contract` / `feature_request` / `other`. For handoffs, the primary reason the
  bot couldn't resolve it. (`knowledge_gap` = the answer existed and a human simply knew it;
  this is the prime documentation opportunity.)

Poll `get_enrich_status` until `done: true`.

### Step D — Compare bot vs human
Call `get_report_guide` once before writing the final report. Then quantify the gap:
- Resolution, satisfaction, and handling quality for **bot-only** vs **handoff**
  (filter each group by the confirmed distinguishing rule). Use `aggregate_table` /
  `pivot_table`.
- Same cuts **by contact_reason** to find topics where the bot fails but humans resolve/
  satisfy well.
- **Escalation drivers** overall and by topic, with human resolution + satisfaction per
  driver — this reveals *where humans do exceptionally well*, i.e. the closeable gaps.

Watch for the common story: the bot's resolution rate may trail humans only slightly, while
its **satisfaction** trails badly because many bot chats end with **no closure signal**
(customer disengages). Call this out if the data shows it — it's a conversation-design fix
(add "did that solve it?" confirmations), not only a knowledge fix.

### Step E — Drill the knowledge gaps into doc topics
Create a **child worksheet** filtered to `escalation_driver = knowledge_gap` and enrich it
(one call) with:
- **kb_topic** — `unspecified_enum` (~20): the specific, article-sized documentation topic.
- **question_asked** — string: the customer's question, phrased as an FAQ question.
- **resolution_answer** — string: the substance of the answer that resolved it (steps,
  settings, rules, limits) — raw material for an article; `NO CLEAR ANSWER` if none.
- **answerable_by_docs** — enum: `yes` / `partial` / `no` (could a good article have
  resolved it without a staff-only backend action?).

Aggregate `kb_topic` (with `answerable_by_docs`) to rank the article opportunities. Read
back the Q/A per top topic (via a subagent if the read is large) and assemble a
**KB Build Kit**: topics ranked into tiers by volume, each with its real Q→A pairs. Also
surface the **cross-cutting question shapes** that recur across topics (e.g. "how long until
X updates?" → one canonical latency reference; "why can't I see X?" → one permissions guide;
"is there an API/export for X?" → one capabilities matrix). These punch above their weight.

### Step F — Reconcile against the help center (only if connected)
Pull the **entire** article library (paginate — help centers often exceed one page; check
`total_pages`/`total_count`). List responses can be very large; when a tool result spills to
a file, parse it with a subagent so bulk text stays out of context. For each gap topic mark
**Covered / Partial / Missing** (published only), naming the matching article(s). Then flag
the failure modes that make a bot miss content that exists:
- **Published-but-empty articles** (title, no body) — a bot "finds" them and returns nothing.
- **Draft-only content** — the answer is written but not live/retrievable.
- **Naming/synonym mismatches** — the customer's term differs from the article's term.
- **Sub-question gaps** inside otherwise-covered topics — augment with Build Kit Q/A.

The key reframe this step enables: distinguish "content doesn't exist" (write it) from
"content exists but the bot can't retrieve it" (fix stubs, publish drafts, add synonyms,
investigate the bot's KB indexing). Don't recommend writing articles that already exist.

### Step G — Deliver
Write the report inline per `get_report_guide` (findings-led, visual cards for multi-cut
findings, a prioritized action table). Produce the **KB Build Kit** as a document
(e.g. .docx) containing the ranked topics and their Q→A pairs. If reconciliation ran,
include the coverage table and an action list ordered cheapest-first (fill empty stubs →
add synonyms → publish drafts → augment sub-questions → write net-new → audit bot retrieval).
Link the worksheet(s) so the user can open the underlying data.

---

## Pitfalls & verification
- **Confirm, don't assume** the data source, the bot/handoff rule, and the help-center
  connection — every customer differs. This is the whole point of the skill.
- **Verify the bot/handoff split empirically** (tag overlap) before analyzing.
- **Batch enrichment** into one call per worksheet; poll to completion; never ask the user
  to check back later.
- **Never hand-compute** report numbers — take each figure from an aggregate/pivot call.
- **Always include a catch-all** (`unknown`/`not enough info`) in enum enrichments.
- **Paginate the full article library** and handle spilled-to-file tool results via a
  subagent.
- Sanity-check that enriched group sizes match the tag/field counts before trusting cuts.
