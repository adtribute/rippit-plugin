---
name: rippit-double-click
description: Go progressively DEEPER on a specific customer question with the Rippit MCP by repeatedly narrowing into a SUBSET of the SAME tickets. Enrich a worksheet, filter to the interesting subset, build a child worksheet over just those rows, and enrich it more richly; repeat. The population only NARROWS each step (vs. rippit-iterative-learning, which moves to different tickets). Use to drill into one already-identified segment. Only use when explicitly invoked (e.g. via its slash command); do not auto-trigger from topic detection.
---

# Rippit Double-Click (drill into a subset)

A reusable pattern for drilling into a known segment. Start broad, identify the interesting subset, then create a child worksheet over just that subset and enrich it more richly. Each step is a strict subset of the prior step. Stop when the segment is understood at the granularity the user needs.

## When this fits vs. the sibling skills

- **Double-click (this skill):** sequential; each step is a SUBSET of the prior step's tickets, enriched deeper. The question targets a specific segment.
- **Iterative:** sequential; each step moves to a DIFFERENT population based on learning. Open-ended.
- **Distributed:** parallel branches over different populations.
- **Second opinion:** same population, competing analyses, debate.

Test: does the next step narrow INTO the current rows (double-click) or move to other rows (iterative)?

## The filter route (this skill's chosen implementation)

There are two valid ways to double-click; **this skill uses the child-worksheet route**:

> Enrich the parent worksheet → filter to the interesting subset → `create_worksheet` over that subset → enrich the child more deeply → repeat.

The alternative (running additional enrichments in-place on filtered rows of the SAME worksheet) is intentionally NOT the default here. Use the child-worksheet route so each depth level persists as its own worksheet the user can open, aggregate, and report on independently.

## Workflow overview

1. **Setup (main turn).** Discover source, confirm columns/labels, name the segment to drill into.
2. **Drill loop.** Each level: enrich → find the subset → child worksheet on that subset → deeper enrich. Run sequentially in the main turn (each level depends on the last).
3. **Deliver** the deepest finding with the trail of narrowing.

---

## Step 1 — Setup

1. `list_data_sources` → source table.
2. `describe_table` → date column, conversation id column (`external_id`), enum columns.
3. `aggregate_table` on relevant enums to read actual labels.
4. State the **starting population** and the **segment you intend to drill into** to the user.

---

## Step 2 — The drill loop

Keep a running note of the row count at each level so the funnel is visible (e.g. 4,000 → 600 → 90).

**Level 0 (parent):**
1. `create_worksheet` over the starting population.
2. `read_table` to confirm rows → `enrich_worksheet` with broad classifying columns (enough to identify which subset is worth drilling into). Batch into one call; catch-all on every enum.
3. `get_enrich_status` until done → `aggregate_table` to find the subset that warrants depth.

**Each subsequent level (N+1):**
1. **Define the filter** that isolates the interesting subset of level N's rows (e.g. `where primary_driver = "billing_dispute" AND severity >= 4`).
2. `create_worksheet` from level N's worksheet (or the source with the cumulative filter) restricted to that subset. This child is a strict subset.
3. Confirm rows, then `enrich_worksheet` with **deeper, more specific columns** than the parent — finer subcategories, exact failure mechanics, verbatim quotes, sequence-of-events. The whole point is that you can afford richer per-row judgment because the population is smaller and more homogeneous.
4. `get_enrich_status` until done → analyze.
5. Decide: drill again into a sub-subset, or stop.

**Exit criteria:**
- The segment is explained at the granularity the user asked for.
- The subset gets too small to generalize (e.g. <20–30 rows) — read those conversations directly instead of enriching further, and say the finding is illustrative not statistical.
- Further narrowing stops revealing new structure.

---

## Step 3 — Deliver

Present:
- The **narrowing funnel**: each level's filter and row count, top to bottom.
- The **deep finding** at the bottom level — the detailed mechanics of what's happening inside the segment.
- Evidence (`external_id`s + verbatim lines).
- The worksheetIds at each level (so the user can open any depth).
- A blunt bottom line on the segment.

## Operating principles

- **Each level is a strict subset.** If a step would pull in tickets outside the parent, that's iterative, not double-click.
- **Child-worksheet route, not in-place.** Persist each level as its own worksheet.
- **Enrich deeper as you narrow.** Parent = broad classifiers; children = fine mechanics and quotes. Don't just re-run the same columns on fewer rows.
- **Watch the floor.** Below ~20–30 rows, switch from enrichment to reading conversations directly and flag findings as illustrative.
- **Batch columns** within each level.
