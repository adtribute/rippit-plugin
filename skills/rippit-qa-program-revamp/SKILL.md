---
name: rippit-qa-program-revamp
description: >-
  Build and run a repeatable "ticket-insights" QA program that measures something
  meaningful on EVERY support ticket — contact reason, root cause, avoidability,
  resolution quality, product signal, and churn/at-risk — instead of just scoring
  agents. Use this whenever the user wants to redesign, rebuild, or stand up a QA
  or support-analytics program; shift QA from agent scorecards to ticket/insight
  focus; systematically tag, classify, or score every ticket; find contact
  drivers, avoidable volume, product feedback, or at-risk accounts from support
  conversations; or set up a recurring support-insights readout. Trigger even if
  the user doesn't say "QA" — e.g. "what should we measure on every ticket",
  "turn our support tickets into insights", "find what's driving contacts",
  "which tickets put revenue at risk". Works with ANY helpdesk or conversation
  data source and ANY set of connectors; do not assume a specific integration.
---

# Ticket-Insights Program Builder

Help the user design and run a QA/support-analytics program organized around the
**ticket**, not the agent. The old model asks "did the agent follow the rubric?"
This model asks, on every ticket: *why did this contact exist, was it preventable,
did the customer actually get helped, and what does it tell the rest of the
business?* Agent quality falls out as a lightweight byproduct.

You are acting as a **thought partner**, not an order-taker. Push back on vague or
over-broad goals, help the user sharpen them, and make sure every measure ladders
up to a decision someone can own. A measure nobody will act on is noise — cut it.

This skill is **integration-agnostic**. It assumes nothing about which helpdesk,
CRM, or connectors exist. You will discover what's available at runtime and adapt.

## The core idea: measure a causal chain, not a topic list

A strong per-ticket schema follows a chain, because one enrichment pass then feeds
many different questions:

**contact reason → root cause → avoidable? → resolution outcome → product signal → customer risk**

- Volume-reduction questions read *reason, root cause, avoidable*.
- Resolution-quality questions read *outcome, effort*.
- Product questions read *product signal, product area*.
- Revenue-risk questions read *sentiment, churn/escalation risk*.

Design around this chain and you avoid the classic failure mode of tagging surface
topics that look tidy but drive no action.

## Workflow

Follow these steps in order. Use a task list to track them. Don't skip the
elicitation step (Step 2) — getting the goals right is the whole game.

### Step 1 — Discover the data and connectors (before asking anything heavy)

Ground the conversation in what actually exists so your questions are concrete.

1. List the available source-of-truth data tables / conversation sources (e.g. a
   Rippit-style `list_data_sources`, a data-warehouse connector, an uploaded
   export). Inspect the schema of the most relevant ticket/conversation table to
   see what's already captured (status, timestamps, channel, subject, agent) vs.
   what must be derived from transcripts.
2. Take stock of **other connectors available in this Claude session** — search
   the connector registry and note anything that could supply business context:
   CRM/Salesforce/HubSpot (account value, ARR, tier), help center (docs to
   reconcile against), Slack/email (escalations), product analytics. You will use
   these later to enrich and to size at-risk tickets.
3. If no ticket data is reachable at all, say so plainly and offer paths: connect a
   helpdesk/warehouse connector, or upload an export. Don't fabricate data.

Report back briefly: what data you found, roughly how much (row count / date
range), and which connectors could add business context.

### Step 2 — Elicit goals and focus areas (REQUIRED, use a structured question)

Do not design the program from assumptions. Use the environment's structured
question tool (e.g. `AskUserQuestion`) to ask — and always include these two:

1. **What do they want to measure in the new program?** Offer the common goals as
   selectable options (they can pick several), and make clear these drive which
   columns get built:
   - Cut contact volume (reason, root cause, avoidability)
   - Improve resolution quality (outcome, effort, reopens)
   - Surface product/feature signal (bugs, requests, friction, doc gaps)
   - Protect at-risk revenue (frustration, churn/escalation signals)
   - Agent coaching (kept as a byproduct)
2. **Is there a particular area they want to focus on that you haven't asked
   about?** Leave explicit room for a custom answer — a specific product line,
   channel, customer segment, language, compliance concern, a known pain point,
   or a metric unique to their business. This catches the thing your menu missed.

Also worth clarifying when ambiguous: who consumes the insights (support/QA lead,
cross-functional, execs) and whether agent performance should be dropped entirely,
kept as a byproduct, or deferred.

Then reflect their answers back with a short POV: which goals are cheap to serve
together, where you'd push back, and what you recommend leading with. If they
picked everything, note that the same core measures serve all of it but the
program needs one clear lead metric and owner to avoid becoming a dashboard nobody
opens.

### Step 3 — Design the measurement schema

Translate their goals into a concrete set of per-ticket measures along the causal
chain. Start from the default set below and **adapt**: drop measures for goals they
didn't pick, and add measures for the focus area they named in Step 2. For each
measure, note the goal it serves and the owner who'd act on it.

Default measure set (adapt freely):

| Measure | Type | Serves |
|---|---|---|
| Contact reason (L1) | fixed categories | volume, product |
| Contact reason (L2) | free / open-ended, clustered | volume, product |
| Root cause type | bug / limitation / knowledge gap / UX / working-as-intended / external | volume |
| Avoidable | self-service / product fix / proactive comms / not avoidable | volume |
| Resolution outcome | resolved / workaround / escalated / awaiting / unresolved | resolution |
| Customer effort | low / med / high | resolution |
| Repeat / recurring signal | first-time / recurring / systemic | volume, resolution |
| Product signal type | bug / feature request / usability / docs gap / none | product |
| Product / feature area | open-ended, ~10–15 clusters | product routing |
| Customer sentiment | positive / neutral / frustrated / angry-at-risk | risk |
| Churn / escalation risk | none / low / elevated / high | risk |
| Agent handling quality | strong / adequate / needs-improvement (byproduct) | coaching |
| One-line issue summary | free text | readout fuel |
| Customer account/company | free text, extracted from transcript | risk sizing |

For every enum, include a catch-all ("unknown"/"unclear") so rows the model can't
judge don't get forced into a real bucket. Write instructions assuming the model
reads the FULL transcript — root cause, whether it was truly resolved, and how the
customer reacted live in the conversation, not the subject line.

Show the user the proposed schema and get a quick thumbs-up before running.

### Step 4 — Validate on a small sample first

Never commit to a full run cold. Build a worksheet over a **representative random
sample** (~50–75 tickets) and enrich it with the full schema in ONE pass (all
columns fill together per row, so cost scales with rows, not columns — batch them).
Read back the distributions and sanity-check that the labels are clean and useful.
If a measure is producing mush, fix its instructions or enum and re-run just that
column before scaling.

Present the pilot results as proof the framework works, and confirm scope for the
full run (which source, what time window).

### Step 5 — Run at scale

Run the same enrichment across the agreed population (e.g. "all tickets, last 90
days"). Poll the job to completion — don't ask the user to check back later. Then
compute the real distributions with aggregation queries (never hand-transform
numbers that will appear in a deliverable).

### Step 6 — Connect business context (esp. for at-risk sizing)

If protecting revenue is a goal, size the at-risk tickets by account value using
whatever is reachable — this is where the connector discovery from Step 1 pays off:

- If a CRM/warehouse carries account value/ARR/tier keyed by account or domain,
  extract each ticket's account (from metadata if present, else from the
  transcript) and join to it.
- Be honest about coverage. Report matched exposure as a **floor**, name what
  couldn't be matched, and recommend the connection that would close the gap
  (e.g. connect the CRM directly, or map a company ID).

If no business-context source exists, still deliver the at-risk *queue* (accounts
+ issue + risk level) and note that value-sizing needs a CRM connection.

### Step 7 — Deliver the report

Write an executive-quality, scannable report. If the environment provides a report
format guide (e.g. a `get_report_guide` tool), read and follow it. Lead with the
answer; use a rendered visual for any multi-cut distribution rather than walls of
prose; ground themes in real quotes/summaries; end with 2–4 owner-tagged actions
and concrete follow-ups. Link the underlying worksheet/dataset so the reader can
dig in. Keep a genuine niche insight only if one truly emerged — a forced insight
erodes trust in the rest.

### Step 8 — Offer to make it recurring and self-building

This is not a one-off. After delivering, **offer to schedule it** as a recurring
task and let the user choose the cadence (daily, weekly, monthly — their call).
Use the environment's scheduling tool. Frame the value as compounding over time:

- Each run appends to / builds on the prior data rather than starting fresh, so
  trends emerge (is avoidable volume shrinking after a doc fix? is a new contact
  driver rising? which product area is heating up?).
- The recurring readout can highlight **deltas vs. the previous period** and flag
  new at-risk accounts the day they appear, not at quarter-end.
- Over time the L2/product-area clusters stabilize into a taxonomy the whole org
  shares.

Make the offer concrete: propose a cadence based on their volume (high volume →
weekly; lower → monthly), state what each run will produce, and set it up if they
say yes.

## Principles

- **Adapt, don't impose.** The default schema is a starting point; the user's goals
  and the named focus area from Step 2 should reshape it every time.
- **Every measure needs an owner.** If you can't name who acts on it, cut it.
- **Sample before you scale.** Cheap validation prevents an expensive wrong run.
- **Be honest about coverage and limits.** A floor with a named gap beats a
  confident number built on a shaky join.
- **Build toward a habit, not a one-time report.** The recurring, self-building
  cadence is where this becomes a program instead of an analysis.
