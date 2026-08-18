---
name: rippit-escalation-finder
description: >-
  Detect brewing escalations in a Rippit customer's support tickets — conversations
  at risk of turning into legal action, a public/social-media blow-up, or an exec/board
  complaint — and deliver the top examples as a ready-to-post Slack update. Use this
  whenever the user asks to find, surface, monitor, or get ahead of escalations,
  "brewing" or "at-risk" or "about-to-blow-up" tickets, churn threats, legal/PR/exec
  risk in support, or wants a recurring escalation digest posted to Slack. Trigger even
  if the user only says something like "what support issues are about to blow up on us?"
  or "set up an escalation watch for a specific account". This runs against the Rippit MCP
  toolchain (list_data_sources, describe_table, create_worksheet, enrich_worksheet,
  get_enrich_status, then aggregate/read), so prefer it over ad-hoc querying for any
  escalation-detection request.
---

# Escalation Radar

Surface support conversations that are quietly turning into escalations — legal exposure,
a social-media flare-up, or a complaint headed for an executive's inbox — before they do.
The output is a tight, ranked Slack post of the most pressing examples, plus an optional
recurring automation.

The core idea: **the data already knows which tickets are unresolved and noisy; what it
doesn't know is which ones carry real escalation risk for *this specific company right
now*.** A web search grounds the run in what's currently combustible for the customer
(recent outages, regulatory heat, public complaints), and a fresh enrichment pass scores
each conversation against those vectors. Don't trust pre-existing "churn risk" style
columns to do this — they're inconsistent across customers and weren't built for this
question. Build the judgment fresh.

## When NOT to over-engineer

If the user just wants a count or a simple distribution ("how many unresolved tickets this
week"), that's a plain `aggregate_table` call, not this skill. This skill earns its weight
when the user wants *which specific conversations are dangerous and why* — a ranked,
human-readable shortlist with evidence.

---

## Workflow

Run these in order. Confirm with the user at the two marked checkpoints — don't barrel
through them, because a wrong status filter or a wrong company guess silently corrupts
everything downstream.

### Step 1 — Identify the customer (the company whose tickets these are)

Call `list_data_sources`. In the common case there is one obvious support-cases table; use
it. If several tables exist, pick the one that holds support conversations (look for
columns like contact reason, status, sentiment, conversation summary) and, if it's
genuinely ambiguous, ask the user which table.

Determine **which company this data belongs to** — i.e. the Rippit customer. This is NOT a
per-row "requester's employer"; every row is the customer's own support traffic. Infer the
company from the table name and from `describe_table` content — product-area values, case
origins, and competitor mentions usually name the company outright (e.g. product areas like
"[Fintech Company] Card", "[Fintech Company] Dashboard" → the company is that Fintech
Company). If you cannot infer it confidently,
ask the user: "Which company's support data is this?"

### Step 2 — Ground the run in the company's current risk surface (web search)

Web-search the company for **recent, specific** escalation fuel — not a generic industry
lecture. Aim 4–8 searches at things that would make a support ticket more likely to
escalate *this month*:

- recent outages, incidents, or degraded-service reports
- regulatory actions, investigations, lawsuits, or compliance scrutiny
- public complaints, viral threads, BBB/Trustpilot/Reddit/X flare-ups, press
- anything sector-specific that raises the stakes of a mishandled ticket (for a fintech:
  frozen funds, account closures, fraud liability, KYC lockouts; for a marketplace:
  refunds, safety; etc.)

Synthesize this into a short list of **escalation vectors** that are live for this company
right now. You'll feed these into the enrichment in Step 5. Keep the synthesis honest: if
the search turns up little, say so and fall back to generic high-severity vectors (legal
threat, public/social threat, executive escalation, regulatory) rather than inventing
company-specific drama.

### Step 3 — Read the schema and choose two raw columns

Call `describe_table`. From the raw table you only need **two** columns (everything else
about escalation gets enriched fresh):

1. **A signal-strength / interest column**, if one exists — something that flags whether a
   conversation is substantive vs. noise (e.g. an "Interest Level" column with values like
   low / moderate / high / very high / ignore). Use it to drop noise before enriching, so
   you don't pay to score auto-replies and empty tickets. If no such column exists, skip
   this filter.
2. **A status column** indicating whether the issue is open/closed/solved/pending/etc.
   Identify its distinct values.

Also note the **recency column** (a last-activity / last-seen / updated datetime) for the
7-day window, and the **id columns** (`ticket_sid` / `external_id`) for later quoting via
`read_conversations`.

### Step 4 — CHECKPOINT 1: confirm the unresolved filter, then build the worksheet

"Unresolved" varies by customer, so confirm before filtering. Default definition:

- **status NOT in the closed set** — closed/solved/merged and their variants (e.g. "Closed",
  "Solved", "Merged", "Closed - No Action Required", "Solved - No Response From Customer").
  List the exact statuses you're treating as resolved and ask the user to confirm or adjust.
- **recency within the last 7 days** on the last-activity column.
- **(optional)** signal-strength not in the ignore/low bucket, if that column exists.

State the filter in plain language, get a yes, then `create_worksheet` on that slice. Title
it like `Escalation Radar — <Company> — <date>`. Omit `columns` so all source columns carry
over (keeps enrichment unblocked). After it returns, `read_table` (a couple rows) or a quick
count to confirm rows actually landed before enriching — worksheet hydration is async.

If the slice is huge (thousands), that's fine for the worksheet, but tell the user the
enrichment cost scales with rows and offer to tighten (e.g. signal-strength = high/very high
only) before running.

### Step 5 — Enrich: escalation vectors + a composite score (ONE call)

Cost scales with rows, not columns, so batch **all** of these into a single
`enrich_worksheet` call. Each row's judgment is made against the full conversation
transcript. Create:

1. **Escalation vector** — `enum`, `multi: true`. Values tuned to what Step 2 surfaced, but
   always include the four base vectors plus a catch-all:
   `["legal_threat", "social_media_threat", "executive_escalation", "regulatory_complaint",
   "<company-specific vector(s) from web search>", "none"]`.
   Instruction: classify which escalation vector(s) this conversation shows real evidence
   of — an explicit or strongly implied threat to (a) take legal action / involve a lawyer,
   (b) go public / post on social / contact press, (c) escalate to an executive, named
   leader, or "your CEO", (d) file a regulatory/agency complaint (e.g. CFPB, FTC, state AG
   for a fintech). Return `none` when there's no real escalation signal — do not inflate
   ordinary frustration into a threat.
   **Critical — judge the customer's state, not the company's outbound messaging.** Many
   tickets contain the *company's own* automated notices (compliance/dunning/closure emails,
   "your account is at risk", "we require documentation by X"). A company telling a customer
   their account is at risk is NOT itself an escalation — the escalation is the *customer's*
   reaction to it. A vector requires evidence in the customer's own words/behavior. If the
   transcript is only the company's outbound notice with no customer distress yet, that's a
   lower-risk watch item, not a live escalation — lean toward `none` or a low score.

2. **Escalation score** — `integer` (1–10). Instruction: rate how likely this conversation
   is to become a real escalation if left alone, weighing: explicitness and credibility of
   any threat (an actual "I'm contacting my lawyer" >> generic anger); finality of tone
   (giving up / "done with you" / deadline given); unresolved duration and repeated contact;
   and stakes named in the ticket (contractual risk, money frozen, security/data exposure,
   business-blocking). 1 = routine, 10 = imminent and severe. Score on *escalation
   probability × blast radius*, NOT emotional volume — a calm, specific legal threat
   outranks an angry all-caps rant about a minor issue.

3. **Escalation rationale** — `string`. One sentence: why this score, naming the specific
   trigger. This is what makes the Slack post credible and skimmable.

4. **Money-quote** — `string`. The single most telling verbatim line (≤25 words) spoken by
   the **customer** (never the agent, and never the company's automated/system text) that
   captures the risk — ideally a threat, ultimatum, or stark statement of harm. Verbatim, no
   paraphrasing. Return an empty string if the customer said nothing notable (e.g. the ticket
   is only an outbound company notice). Used directly in the Slack post, so a wrong
   attribution here is highly visible.

Then `get_enrich_status` in a loop until `done: true` (it blocks; keep calling — don't hand
back to the user). Capture the returned column **ids** (not labels) for the next step.

### Step 6 — Rank and pull the top examples

`aggregate_table` / `read_table` over the worksheet, ordered by escalation score descending,
filtered to vector ≠ `none`. Take the **top 10** (fewer if fewer qualify — never pad). For
those rows, pull the id, vector(s), score, rationale, money-quote, contact reason, status,
and recency. Then `read_conversations` on the finalist ids (≤10 per call) to (a) verify each
money-quote is the customer's verbatim words and correctly attributed, and (b) capture each
ticket's **conversation link** (`ticketDoc.link`) for the post — every featured item should
deep-link to the actual conversation.

Sanity-check the ranking by eye: if the top item is obviously mis-scored — e.g. it's just the
company's own compliance email with no customer reaction — drop it or note it rather than
laundering it into the post.

### Step 7 — Format the Slack post, show it in chat, then test in DM (CHECKPOINT 2)

Compose a Slack-ready post and **show it in the chat first** — do not post anything yet.
Borrow the structure of the `support-shoutout-builder` skill (numbered items, one blockquote
quote per item, a per-item link, a warm one-line header and closer), inverted for risk rather
than praise. Format (Slack mrkdwn, tuned for skimming on mobile):

```
:rotating_light: *Escalation Radar — <Company>* — <N> conversations brewing toward escalation
_Unresolved tickets, last 7 days · <M> substantive conversations scanned · ranked by escalation risk_

<one line on what's currently combustible from the web search — the dominant vector, if notable>

*1. <contact reason>* — <vector emoji + label> · <X>/10 · _<status>_
<one-sentence rationale naming the specific trigger>
> "<verbatim customer money-quote>"
<https://<conversation link>|Open case <case number> →>

*2. …*

_<short closing line>. Generated with Rippit._
```

Notes on format:
- Highest score first; lead with the single most urgent item implicitly via ordering.
- Each item deep-links to its real conversation (the `ticketDoc.link` captured in Step 6) —
  never invent a link.
- A worksheet link is optional; offer it but don't force it (users often want a clean
  conversation-only post). If included, put a single scoped-worksheet link at the very end.
  To make one, `create_worksheet` from the enriched worksheet filtered to just the featured
  ticket ids, and link its `url`.
- Use a small, consistent set of vector emoji (e.g. :balance_scale: legal, :ice_cube: frozen
  funds, :arrow_up: exec, :rotating_light: fraud/severe) so the post is scannable.

Then ask the user, in chat: any refinements (threshold, count, drop a vector, reword)? And —
following the shoutout skill's guardrail — **offer to send the draft to the user's own Slack
DM first** so they can see exactly how it renders before anyone else does. Their own user ID
is usually available from the Slack tools' context; send there with `slack_send_message`. Note
that DMs can't be edited via API, so each revision posts a new message — tell the user that
rather than appearing to overwrite.

Only after the user is happy AND explicitly names a destination channel do you post to it.
Posting to a shared channel is a send action requiring explicit per-action approval; surface
the consequences first (who gets @-mentioned, that it names real customers, the audience
size). Default to a **draft** (`slack_send_message_draft`) unless they say send live. For an
escalation digest, @-mention a triage owner or the channel — never the agents on the tickets.

### Step 8 — Offer the recurring automation

Once the user is happy with a run, offer to set it up as a **recurring Cowork task** so the
digest posts itself on a schedule (e.g. every weekday 8am). Key point: a recurring run can't
do the interactive checkpoints, so **lock in the parameters from this run** as the task
config — the resolved-status set, the channel, the score threshold, the example count, and
the company. Write the recurring task instruction to reference those fixed values explicitly
so it runs unattended without re-asking. Confirm the schedule and channel with the user
before creating it.

---

## Output discipline

- This is a findings deliverable — when you summarize the run in chat (beyond the Slack
  post itself), follow the Rippit report format from `get_report_guide`: lead with the
  headline, ground claims in verbatim quotes, no walls of prose.
- Never fabricate a quote, a score, or a web-search finding. If a step turns up little,
  say so plainly — a forced "everything's on fire" digest destroys trust the first time a
  reader clicks through and finds a routine ticket.
- Respect the action boundaries: building worksheets and enriching is free to do; **posting
  to Slack and creating a recurring task both require explicit user approval** at the moment
  of doing them.
