---
name: ingest
description: Ingest one approved source (URL or raw/ file) into the wiki, or batch-process unprocessed entries in raw/sources.md. Drafts pages via the ingestor subagent, previews the structure, and writes only after approval. Biases toward extending existing pages.
---

# /ingest — Add source material to the wiki

Turns one or more approved sources into wiki pages. Two entry modes; both share the same draft → preview → write path (the same machinery `/subtopic-loop` Steps 3–5 use — don't reinvent it).

## Modes

- **Single source** — a URL or a path under `raw/` the user names directly.
- **From `raw/sources.md`** — process the unprocessed (`[ ]`) entries. Dedup against `wiki/log.md` first (skip anything already ingested). Mark each entry `[x]` only after its pages are successfully written.

If a batch would ingest more than ~5 sources or create more than ~10 pages in one run, pause and confirm scope (per `CLAUDE.md > Budgets` and soft rules) before proceeding.

## Steps

1. **Confirm the source list** with the user (which URLs / files, or which `sources.md` entries).
2. **Draft (no writes).** For each source, spawn the **ingestor** subagent with the source identifier, any topical context, and a copy of the current `wiki/index.md`. It returns drafts as text — new pages, updates-as-diffs, a source-summary page, index delta, and a log line. **Write nothing yet.**
3. **Structure preview.** Assemble a single compact preview using the format in `/subtopic-loop` Step 4 (paths, types, frontmatter title+sources, H2/H3 outline, word counts; diffs for updates; full index/log deltas; a cross-link audit flagging any `[[wikilink]]` to a page that won't exist). Confirm each new page lands in a section directory matching its taxonomy.
4. **GATE — structure approval.** Prompt: approve / revise / abort / `show full <page>`. Do not write until approved. (Unlike `/subtopic-loop`, there's no separate source-approval gate here — the user already chose the sources in step 1.)
5. **Write** in one batch: new pages → updates → `wiki/index.md` → append `wiki/log.md`. In `sources.md` mode, flip each processed entry to `[x]`.
6. **Lint.** Run `/lint` scoped to the pages just written.

## Budget

Per-operation caps in `CLAUDE.md > Safety & Limits > Budgets` apply: ~15 combined fetches (including the ingestor's follow-on link fetches) and the ~10-new-pages confirm threshold.
