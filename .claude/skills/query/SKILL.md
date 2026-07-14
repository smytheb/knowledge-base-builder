---
name: query
description: Answer a question from the existing wiki. Searches wiki pages, answers with [[wikilink]] citations and honest confidence, flags gaps, and offers to save a substantial reusable answer as a new page. Wiki-only by default — no web fetching unless the user asks.
---

# /query — Answer from the wiki

Answers a question using the knowledge already in `wiki/`. This is a read-first operation: it does **not** fetch the web or write pages unless the user explicitly opts in.

## Steps

1. **Search the wiki.** Grep `wiki/` for the question's key entities and synonyms; open the most relevant pages. Follow `[[wikilinks]]` one hop out when a page points at something on-topic.
2. **Synthesize an answer** from what you found. Follow these output rules (adapted from knowledge-synthesis):
   - **Lead with the answer**, then support it. Group by topic, not by source page.
   - **Cite inline** with `[[wikilink]]` to the page each claim comes from.
   - **State confidence honestly.** If the wiki's support is thin, one-sided, or possibly stale (`review-date` past), say so.
   - **Surface conflicts** — if two pages disagree, present both positions with citations rather than silently picking one.
3. **Flag gaps.** Explicitly name what the wiki does *not* cover that the question needed. Suggest the next step: `/subtopic-loop`, `/ingest`, or an Explore for that gap.
4. **Offer to save** — if the answer is substantial, reusable, and not already a page, offer to turn it into a wiki page (routing through the normal `/ingest` structure preview + write gate). Only write on approval.

## Scope

Wiki-only by default. If the wiki can't answer and the user asks you to go wider, hand off to Explore / `/ingest` / `/subtopic-loop` — don't quietly start fetching sources inside a query. No writes without approval.

## Log

A pure read-only query needs no log entry. If the user approves saving the answer as a page, that write is logged via the `/ingest` path.
