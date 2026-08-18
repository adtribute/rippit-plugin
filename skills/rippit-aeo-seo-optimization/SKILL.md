---
name: rippit-aeo-seo-optimization
description: >-
  Build an SEO- and AEO-optimized content program grounded in a company's own
  conversation data (support chats, sales calls, etc.) via the Rippit MCP.
  Adopts the framing "I am a marketing leader looking to optimize our SEO and
  AEO," asks which topics/keywords to win, discovers which Rippit sources best
  support that angle, mines them liberally with worksheets + enrichment for real
  customer and employee language, scans the user's own company website for
  coverage and gaps, and produces a content strategy plus multiple drafted
  pieces (pillar + cluster) written into whatever docs connector the user has
  (Google Drive, Notion, etc.). Use whenever the user wants SEO/AEO/GEO content,
  a content strategy, blog/article/landing-page copy, a content cluster or pillar
  page, or marketing content informed by what customers actually say — even if
  they don't name SEO. Trigger on "optimize our SEO," "content strategy," "rank
  for," "write an article about," "AEO," or "get cited by AI." Runs as a
  multi-sub-agent pipeline.
---

# SEO / AEO Content Engine

Adopt this framing for the whole run: **"I am a marketing leader looking to
optimize our SEO and AEO."** Every decision serves that goal — turning the
company's real conversation data and existing web presence into content that
ranks in search (SEO) and gets surfaced/cited by answer engines like ChatGPT,
Perplexity, Google AI Overviews, and Claude (AEO/GEO).

The output is not one article. It is a **content strategy plus multiple drafted
pieces** organized as a pillar page with supporting cluster articles, all
grounded in evidence pulled from the company's own data.

This skill orchestrates several sub-agents. The main thread is the orchestrator:
it scopes the work, launches sub-agents, hands each the artifacts from the
prior phase, and assembles the result. Be generous with sub-agents and with
Rippit worksheets/enrichments — thoroughness beats frugality here. If a phase is
large (e.g. many topics or many data sources), split it across parallel
sub-agents.

## Phase 0 — Scope with the user (orchestrator, inline)

Before any research, confirm direction. Ask the user, in one message:

1. **Areas of interest** — "Are there particular topics, keywords, products, or
   questions you want to optimize your SEO and AEO around?" Offer a couple of
   concrete example angles so it's easy to answer, but let them steer. If they
   have none, propose candidates after Phase 1 discovery and confirm before
   drafting.
2. **Company website** — infer the likely domain from the user's email address
   (e.g. `jane@acme.com` → `acme.com`) and confirm it, or ask. This is the site
   scanned in Phase 3. Keep this generic: it is always *the user's own company*,
   never a hard-coded site.

Do not gate the whole run on a long questionnaire — get topic + domain, then move.

## Phase 1 — Strategy & outline (sub-agent: "strategist")

Give this sub-agent the user's topics and domain. It produces the **strategy
skeleton** that every later phase fills in, using the methodology in "SEO/AEO
methodology" below (search intent, pillar/cluster architecture, and the AEO
specifics that make content citable by answer engines).

Deliverable from this phase: a strategy doc containing, for each target topic:
the primary keyword + intent, the pillar page concept, 3–8 cluster article
concepts, the specific questions each piece must answer, and the target keywords
per piece. Leave clearly marked placeholders for evidence (customer quotes,
stats, pain points) so Phase 2 knows exactly what to fill.

## Phase 2 — Mine the company's data via Rippit (sub-agent(s): "researcher")

This is where the content earns its authority. The job: find the real language,
pain points, questions, objections, and wins in the company's own conversations —
from **both customers and employees** — and attach that evidence to the outline.

Do not assume which table to use. **Discover generically:**

1. `list_data_sources` to see every available table. Read each one's description
   and, for the promising ones, `describe_table` to inspect columns. Match source
   to topic: support/chat data for how customers phrase problems and questions
   (great for FAQ + long-tail keywords); sales-call data for objections,
   competitive framing, and desired outcomes (great for commercial-intent and
   comparison pieces); any other source (reviews, NPS, community) if it fits.
   Prefer pulling from *several* sources over betting on one.
2. **Find the shape** — `read_table` for sample rows, then `aggregate_table` to
   see volume by the dimension that matters (topic, tag, date). Confirm the slice
   is real before persisting it.
3. **Persist the slice** — `create_worksheet` with a `where` filter matching the
   topic and a specific title (e.g. `"Onboarding chats — SEO mining"`). Omit
   `columns` so later enrichment isn't blocked; confirm rows landed with a quick
   `read_table`.
4. **Enrich in one batched call** — cost scales with rows, not columns, so add
   every column you might want at once. Useful columns for content mining:
   `theme`/`topic`, `underlying_question` (the real question the person is asking,
   in their words — gold for FAQ headings + long-tail keywords), `pain_point`,
   `desired_outcome`, `objection`, `competitor_mentioned`, `sentiment`,
   `verbatim_quote` (short, self-contained, quote-worthy line), and `speaker_role`
   (customer vs. employee). Give every enum an explicit "other/unknown" escape
   value so rows the model can't judge aren't force-fit.
5. **Wait for it** — poll `get_enrich_status` in a loop until `done: true`. Never
   hand the wait back to the user.
6. **Extract evidence** — `aggregate_table` for frequencies ("34% of onboarding
   chats ask about SSO"), `find_quotes` for verbatim examples,
   `show_conversations_preview` to present quotes as drill-in-able rows. Attach
   each stat/quote to the specific outline placeholder it supports.

**Enrich liberally.** Create a fresh worksheet per topic or per source rather
than overloading one, and if a topic is broad, split the mining across parallel
researcher sub-agents. Over-mining is cheap; thin, unsupported content is the
expensive failure.

**Separate customer voice from employee voice.** Both matter for different
reasons — customer language for the questions/pain points readers search
(drives FAQ questions, long-tail keywords, the "problem" framing); employee
language (support agents, sales reps, internal experts) for authoritative
explanations and product positioning (drives the "solution" framing and
credibility). Conversations contain both speakers, so tell the enrichment model
explicitly who is who via `speaker_role`, and never attribute an employee's words
to a customer or vice versa. When quoting customers externally, anonymize (no
names/companies) unless the user says otherwise.

Deliverable: the Phase 1 outline with every evidence placeholder filled —
frequency data, representative customer quotes (attributed, anonymized),
objections to preempt, and the exact phrasings customers use.

## Phase 3 — Scan the company website (sub-agent: "site-scanner")

Scan the user's own company site (the domain from Phase 0) to see what already
exists, so new content aligns with the brand, fills real gaps, and doesn't
cannibalize existing pages.

Use `WebSearch` + `mcp__workspace__web_fetch` to crawl the site's blog,
resources, product, and help pages. If a page is client-rendered and web_fetch
returns an empty shell, escalate to the Chrome tools
(`mcp__claude-in-chrome__navigate` → `get_page_text`) per the browser guidance.
Optionally check the sitemap (`/sitemap.xml`) for a full URL inventory.

For each planned piece from the outline, note: does a page already cover this
(update vs. create)? what's the house tone, structure, and CTA pattern? what
internal links should new pieces point to? what gaps exist that competitors or
answer engines are filling instead? Fold this into the outline as annotations.

## Phase 4 — Draft the content (sub-agent(s): "writer")

With the evidence-rich, site-aware outline, draft the **pillar page and each
cluster piece**. Split across parallel writer sub-agents if there are many. Each
piece must:

- Lead with a direct, extractable answer to its core question (AEO: answer
  engines lift concise, self-contained answers — put it up top, then expand).
- Use the customer's real phrasings as headings and FAQ questions.
- Weave in the Phase 2 evidence (quotes, stats) for credibility and originality.
- Match the site's voice from Phase 3 and include internal links to the pillar
  and siblings.
- Include SEO metadata (title tag, meta description, URL slug, target keyword)
  and AEO scaffolding (FAQ section, suggested schema markup, TL;DR).

Use the per-piece structure template below.

## Phase 5 — Publish to the right connector (sub-agent: "publisher")

**Don't hard-code Google Docs.** Detect what's available and document the
findings where it fits the user's stack:

1. Check which documentation/storage connectors the user actually has (Google
   Drive/Docs, Notion, Confluence, etc.). If more than one is viable, ask the
   user which they prefer; if only one, use it; if none, save polished Markdown
   files to the outputs folder and present them.
2. Write the strategy doc plus each content piece as clean, formatted documents
   in the chosen destination. Return links.

## Wrap-up (orchestrator)

Summarize for the user: the strategy in brief, the pieces produced and where
they live, the key evidence themes that shaped them, and suggested next steps
(publishing order, what to measure). Keep it tight — the documents carry the
detail.

---

## SEO/AEO methodology

**SEO vs. AEO.** SEO targets ranked links — signals are keyword relevance,
topical authority, backlinks, page experience, freshness, internal linking. AEO
(a.k.a. GEO) targets being the source an answer engine quotes when it composes an
answer — signals are self-contained liftable answers, clear entity definitions,
structured Q&A, citable stats, and consistency across the web. Write every piece
to serve both: rank-worthy depth *and* liftable, well-structured answers.

**Search intent → format.** Classify each topic and match the format:
informational ("what is X", "how to Y") → guides, explainers, FAQs (prime AEO
territory); commercial ("best X", "X vs Y", "X alternatives") → comparisons,
listicles; transactional ("X pricing", "buy X", "demo") → product/landing pages;
navigational (brand terms) → owned pages. The best long-tail keywords come
straight from Phase 2 — the literal phrasings customers use are what real people
type and ask answer engines.

**Pillar + cluster architecture.** Structure the program as topic clusters, not
scattered posts. A **pillar page** is a comprehensive resource on the core topic
targeting the head keyword and linking out to every cluster piece; **cluster
pieces** are narrower articles each answering one subtopic and its long-tails,
linking back to the pillar and to relevant siblings. This internal-linking
topology signals topical authority and gives answer engines a dense entity graph.
Aim for one pillar + 3–8 clusters per topic.

**AEO specifics — to get surfaced and cited:**

- **Answer first.** Open every page and every H2 with a direct 2–4 sentence
  answer to the question in the heading, before elaborating — answer engines lift
  the concise self-contained chunk.
- **Target real questions as headings** (from Phase 2 customer phrasings), each
  immediately answered.
- **Define entities clearly** ("X is a …") so a clean definition can be extracted.
- **Be citable** — include specific stats, dates, and named facts, using Phase 2
  evidence where possible. Original data is disproportionately quoted.
- **Add a TL;DR / key takeaways** block near the top.
- **Add an FAQ section** with schema markup.
- **Stay consistent** across the site so the entity is reinforced, not
  contradicted.

## Per-piece structure template

Use this as the default skeleton for each drafted piece:

```
# [H1 — includes primary keyword, reads naturally]

**TL;DR:** [2–3 sentence extractable summary answering the core question.]

## [H2 — a real question customers ask]
[Direct answer in the first 2–4 sentences. Then supporting depth, examples,
and Phase 2 evidence (quote or stat).]

## [H2 — next question / subtopic]
[Same answer-first pattern.]

... (repeat for each subtopic) ...

## Frequently asked questions
**[Question in customer's own words]?**
[Concise standalone answer.]
(3–6 FAQ pairs — these power FAQ schema and AEO capture.)

## [Conclusion / CTA matching the site's pattern from Phase 3]

---
SEO metadata:
- Title tag (≤60 chars, primary keyword near the front):
- Meta description (≤155 chars):
- URL slug (clean, keyword-bearing):
- Primary keyword:
- Secondary/long-tail keywords:
- Internal links (to pillar + siblings):
- Suggested schema: [Article | FAQPage | HowTo]
```

Every piece ships with exactly one H1, a logical H2/H3 hierarchy, the metadata
block above, suggested JSON-LD structured data, internal links up to the pillar
and across to siblings, and alt-text notes for any suggested images.

## Notes

- Rediscover data sources every run — table availability and IDs change; never
  reuse an ID from memory.
- If the user gave no topic and Phase 1/2 surface strong candidate themes,
  propose them and confirm before drafting a full slate of pieces.
- Bias toward more sub-agents, more worksheets, more enrichment. Under-mining the
  data is the more common and costlier mistake than over-mining it.
