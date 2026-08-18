---
name: rippit-support-shoutouts
description: >-
  Find the best "above and beyond" customer-support conversations — where an
  agent went the extra mile and the customer was visibly grateful — and turn
  them into a polished recognition message for Slack (or another chat channel).
  Use this whenever someone wants to celebrate, recognize, or shout out support
  agents; surface "great support moments," "wins," "kudos," or "appreciation"
  stories from support tickets/conversations; build a team recognition post; or
  schedule a recurring support-wins digest. Trigger it even if the person only
  says something like "find me some great support interactions to highlight" or
  "put together a kudos post for the support team" — this skill defines the full
  find → rank → draft → test-in-DM → confirm-channel → (optionally) schedule
  workflow, and it never posts to a shared channel without explicit confirmation.
---

# Support Shout-out Builder

Turn raw support conversations into a recognition post that celebrates agents
who went above and beyond — backed by real quotes — and deliver it safely to a
chat channel only after the user has reviewed it.

This skill is **support-specific** but **tool-agnostic**. It assumes you have
some way to (a) read/analyze support conversations and (b) post a message to a
chat tool. Use whatever data-analysis and messaging tools are connected in the
current environment. Discover them rather than assuming specific products.

## The core promise

The user does not want three *random* conversations that technically qualify.
They want the *best* ones: cases where the agent's extra effort is concrete and
specific, and the customer's gratitude is vivid and personal. Ranking quality is
the whole point — do not shortcut it.

And: **nothing gets posted to a shared channel without explicit confirmation.**
The default safe path is test-in-DM first.

---

## Workflow overview

1. Clarify scope (count, time window, channel, recurring?)
2. Locate the support-conversation data
3. Build the candidate worksheet — **without filtering on possibly-empty enriched columns**
4. Rank for genuine "above and beyond" + customer appreciation
5. Verify quotes against the real transcripts
6. Draft the recognition message
7. **Send to the user's own DM first to test** (confirm this is what they want)
8. Revise messaging as needed
9. Ask which channel to share in — **never post without confirmation**
10. Optionally schedule it on a recurring basis

Work through these in order. Steps 7–9 are guardrails, not suggestions.

---

## Step 1 — Clarify scope

Before doing anything, confirm a few things (ask only what isn't already clear
from the request):

- **How many wins** to feature (default: 3). It is fine to return fewer if not
  enough conversations clear the quality bar — say so rather than padding.
- **Time window** (e.g., last 30 days, last quarter, all-time). Recency matters
  for recurring digests.
- **Source filter**, if any (a particular team, queue, product line, channel).
- **Recurring?** Ask whether this is a one-off or something they want on a
  cadence (e.g., weekly/monthly). This changes the time window and how Step 10
  is handled.

Keep this light — one short round of questions, then proceed.

---

## Step 2 — Locate the support-conversation data

Discover what's available rather than assuming. Look for a data source that
holds support conversations/tickets. Useful signals that a table is the right
one: columns for conversation text or summaries, ticket status, customer
sentiment, CSAT, resolution, and any pre-computed/AI-enriched fields (sentiment,
"resolved?", quotes, themes).

If multiple sources exist, prefer the one that already has enrichment columns —
it saves work — but verify those columns are actually populated (see Step 3).

Inspect the schema before querying so you know the real column identifiers and
their distinct values. Don't guess column names.

---

## Step 3 — Build the candidate worksheet (critical guardrail)

Create a working slice (worksheet/filtered table) of plausible candidates.

**Guardrail — do not filter on enriched columns that may be unpopulated for
recent tickets.** Enrichment (sentiment, "issue resolved," extracted quotes,
etc.) is often computed on a lag. If you filter the initial population on an
enriched column, recently-closed conversations whose enrichment hasn't run yet
will be silently excluded — exactly the fresh wins a recurring digest wants.

So:

- For the **initial candidate population**, filter only on **reliably-populated,
  source-of-truth fields**: source/channel, date range, ticket status (e.g.
  closed), team/queue. These are written at ingest and are safe.
- **Before** you rely on any enriched column as a filter, check that it's
  actually populated for the time window in question. A quick way: aggregate the
  count of non-empty values for that column within the date range, or sample a
  few recent rows. If a meaningful share of recent rows is empty/blank/unknown,
  **do not use that column as a filter** — instead, pull the broader population
  and let your own ranking pass (Step 4) do the work on every row.
- If enriched columns *are* well-populated for the window, you may use them to
  narrow (e.g., positive sentiment + resolved) — but treat this as an
  optimization, not a requirement.

The safe default when in doubt: filter only on source + date + status, then
enrich/score every row yourself in Step 4.

---

## Step 4 — Rank for genuine "above and beyond" + appreciation

This is where quality is won. Judge against the **full conversation transcript**,
not a one-line summary — the extra-mile behavior and the gratitude live in the
back-and-forth.

Score each candidate on two independent dimensions. If the environment supports
adding AI/enrichment columns to the worksheet, add them; otherwise read the
transcripts and score directly. Cost scales with rows, so add all the columns
you need in a single pass.

**Dimension A — "Above and Beyond" (0–10).** How far past ordinary competent
service did the agent go? Be a harsh grader.
- Score **8–10** only for unmistakable extra-mile behavior: proactively
  investigating beyond what was asked, building or finding a custom
  workaround/fix, doing manual work on the customer's behalf, personally
  escalating and following through, jumping on an unscheduled call, anticipating
  and preventing a *future* problem, working outside hours, unusual ownership.
- Score **0–3** for routine "here's the answer / here's the doc / fixed it" —
  even when friendly and resolved. Most resolved-positive tickets are routine.

**Dimension B — Customer Appreciation Intensity (0–10).** How strongly and
explicitly did the *customer* express gratitude? Be a harsh grader.
- Score **8–10** for effusive, specific praise — naming the agent, "you went
  above and beyond," "best support I've had," "you saved us," exclamation-heavy
  thanks, praising the *person* not just the outcome.
- Score **4–6** for a sincere "thank you so much, really appreciate it."
- Score **0–3** for a perfunctory "thanks" or none at all.

Also extract, per row, **one verbatim agent quote** showing the extra mile and
**one verbatim customer quote** showing the praise (return a clear "NONE"
sentinel when absent, so you can exclude those rows).

**Selecting the winners.** Filter to rows scoring high on **both** dimensions
(e.g., both ≥ 8) with non-empty quotes. Then read the actual quotes and pick the
final set by judgment, favoring:
- a **concrete, specific** agent action (built/recreated/cleaned up/prevented),
  over vague "I flagged it to the team";
- the **most vivid, personal** customer praise;
- **variety of agents** for a team shout-out (spread recognition) unless the
  user asked strictly for top-N-by-score.

If fewer than the requested number clear the bar, **return fewer** and say why.
Do not lower the bar to hit a number.

---

## Step 5 — Verify quotes against the real transcripts

Before any quote goes into a message, confirm it against the source
conversation. Pull the full records for your finalists and check that:
- both quotes appear **verbatim** and are correctly attributed (agent vs.
  customer);
- the customer's name, account/company, and the agent's name are right;
- the described "above and beyond" action actually happened.

Catching a misattribution here is cheap; catching it after posting is not. Note
and fix any discrepancy (e.g., wrong company pulled from an email domain).

Also capture, for each finalist, a **link to the conversation** if the data
exposes one — useful in the post.

---

## Step 6 — Draft the recognition message

Write a warm, scannable post. A reliable structure:

```
[eye-catching header — celebrating support wins]

[one-line intro: these are cases where the team went the extra mile
 and the customer noticed]

[For each win, numbered:]
*N. [Agent] → [Customer/Account]*
[1–2 sentences on what the agent did that was exceptional]
> [Agent]: "[verbatim agent quote]"
> [Customer]: "[verbatim customer quote]"
[link to the conversation, if available]

[short closing line of appreciation to the agents]
```

Formatting tips:
- Match the target chat tool's markup (e.g., Slack uses `*bold*`, `_italic_`,
  `>` blockquotes).
- To **tag/mention agents** in the message, resolve each agent's *name* (from
  the transcript) to that chat tool's *user ID* via its user-lookup capability,
  then use the tool's mention syntax (e.g., Slack `<@USERID>`). Do not paste raw
  names and assume they'll notify anyone. Confirm each match is unambiguous
  (e.g., name + work email) before using it.
- If a quote contains a placeholder or broken link from the original
  conversation, trim it rather than reproducing it.

---

## Step 7 — Test in the user's own DM first

**Always offer to send the draft to the user's own direct message first**, so
they can see exactly how it renders before anyone else does. Confirm this is
what they want:

> "Want me to send this to your own DM first so you can see how it looks?"

If yes, look up the current user's own ID and send the draft there. Note that
mentions typically render as plain linked names in a DM (they don't notify the
tagged people from a self-DM), which makes it a safe test.

Share the resulting message link so they can open it.

---

## Step 8 — Revise as needed

Iterate on wording, ordering, which wins are included, tone, length — whatever
the user asks. Re-send the revised version to their DM if they want to see it
again. Stay in this loop until they're happy.

If the user changes their mind about content they previously cut (links,
quotes, an extra win), accommodate it without friction.

---

## Step 9 — Confirm the destination channel, then post

**Never post to a shared/team channel without explicit confirmation.** This is
the most important rule in the skill.

Ask which channel they want to share it in:

> "Which channel should I post this to?"

Then:
1. Resolve the channel name to its ID via the chat tool's channel-search.
2. **Confirm before sending**, and surface anything irreversible or
   easy-to-regret in that confirmation:
   - it will notify everyone tagged (agents get pinged in a real channel,
     unlike the DM test);
   - it names real customers, if it does — fine internally, worth a beat before
     it's company-wide;
   - the audience size (e.g., a company-wide #general vs. a small team channel).
3. Only after a clear "yes," post it.
4. Return the posted message link.

If the user names the channel and says "send it" in the same breath, you may
treat that as the confirmation — but still state the audience/mention
consequences in one line before sending.

---

## Step 10 — Optional recurring schedule

If the user wants this on a cadence:

- Many chat tools support **scheduled messages**; if available, you can schedule
  the *next* run's post. But the *content* must be regenerated each period from
  fresh data — a scheduled message is static text, so don't schedule a stale
  post far in advance as if it were dynamic.
- The honest framing for true recurrence: this skill is the repeatable workflow.
  Each period, re-run Steps 1–9 against the new time window (Step 3's guardrail
  matters most here — recent wins are exactly the ones whose enrichment may lag).
- If the environment has a scheduler/automation/cron capability, offer to wire
  the workflow to it. If it doesn't, tell the user plainly that you can't create
  a standing automation from here, and the practical path is to re-run this
  skill on their chosen cadence (e.g., "ping me every Monday and I'll build that
  week's post"). Don't imply a recurring job exists when it doesn't.
- For a recurring **digest**, default the time window to the period since the
  last run and lead with the freshest wins.

---

## Guardrails recap

- Rank for *genuine* quality; harsh grading on both dimensions; return fewer than
  requested rather than padding with weak entries.
- Build the initial candidate set from **reliably-populated source fields only**;
  verify any enriched column is populated for recent tickets **before** filtering
  on it.
- Verify every quote verbatim against the transcript and check attribution.
- **Test in the user's DM first**; revise; then **confirm the channel**; **never
  post to a shared channel without explicit confirmation**, and flag mention/
  customer-name/audience consequences before sending.
- Be honest about what "recurring" can and can't do in the current environment.
