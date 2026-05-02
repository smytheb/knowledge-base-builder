---
name: subtopic-loop
description: Run the full research loop for a single subtopic — explore, source-approve, ingest (drafts only), structure-approve, write, lint, commit-prep. Standalone runnable; also invoked by /research-topic.
---

# /subtopic-loop — Per-subtopic research

Runs one subtopic end-to-end with two human approval gates:

- **GATE 1** after Explore — user approves which candidate sources to ingest now / save for later / discard
- **GATE 2** after draft assembly — user approves the proposed wiki changes before *any* writes happen

**Critical invariant:** the agent must never write to `wiki/` between GATE 1 and GATE 2. All drafts live in subagent return values and in the conversation until GATE 2 passes. The only sanctioned writes during this window are appending approved-for-later entries to `raw/sources.md` (per the GATE 1 user approval).

## Inputs

- Subtopic name (free text). If it matches an entry in `wiki/_outlines/<topic>-master.md`, that entry's checkbox is flipped on successful completion.
- Optional: key questions for the subtopic (passed in by `/research-topic`, or asked from the user if running standalone).

If running standalone with no key questions provided, ask the user briefly: *"Any specific questions this subtopic page should answer? (Optional — can also let the explorer infer from the name.)"*

## Step 1 — Explore

Spawn the **explorer** subagent. Pass it: the subtopic name, key questions, in/out-of-scope notes from CLAUDE.md, fetch budget of 8.

Subagent returns a candidate list with: URL, title, source type, one-line description, relevance to the subtopic / specific key question, dedup status against `raw/sources.md` and `wiki/log.md`.

## Step 2 — GATE 1: Source approval

Present the candidates to the user. For each, the user chooses:

- **ingest now** — use this source in Step 3
- **save for later** — append the candidate to `raw/sources.md` (this is the explicit user approval that CLAUDE.md's Explore operation requires)
- **discard** — drop it, don't touch `raw/sources.md`

If the user approves zero sources to ingest, ask whether to:
- Abort the loop
- Proceed to Step 4 using only existing wiki content (synthesis-only mode — useful when the subtopic is already well-covered and just needs cross-linking or a summary page)

## Step 3 — Ingest (drafts only)

For each "ingest now" source, spawn the **ingestor** subagent with the source identifier, subtopic context, and a copy of the current `wiki/index.md`.

Subagent returns drafts as text:
- Proposed new pages (full content including frontmatter)
- Proposed updates to existing pages (as diffs)
- Source summary page for `wiki/sources/`
- `wiki/index.md` delta
- `wiki/log.md` entry to append
- Notes / follow-up flags

**Do not write any of this to disk yet.**

## Step 4 — Structure preview

Assemble a single consolidated preview from all ingestor returns plus any synthesis the main agent wants to add (e.g., cross-links between newly-drafted pages).

**Default (compact) preview** — the goal is a screen-sized scannable summary, not a full read-through. Drafts are kept in conversation memory and shown in full only on request.

- **New pages** — for each: full path (e.g. `wiki/03-foo/bar.md`), `type`, frontmatter `title` + `sources`, the H2/H3 outline, and word count. Do **not** print the body.
- **Updates** — target path + diff hunks only (no surrounding-context dump).
- **Index delta** — lines to add to `wiki/index.md` (full, they are short).
- **Log entry** — line to append to `wiki/log.md` (full).
- **Cross-link audit** — every `[[wikilink]]` in new/updated pages; flag any pointing to pages that don't exist and aren't being created in this batch.
- **Section placement** — confirm each new page lands in a directory matching its primary section in the taxonomy.

After the compact preview, prompt: *"Approve as-is, revise, abort, or `show full <page>` to inspect any draft body before deciding."* On `show full <page>`, print that page's full body inline and re-prompt.

## Step 5 — GATE 2: Structure approval

User options:
- **approve** — proceed to writes
- **revise** — describe changes; revise drafts in-context (do not re-spawn the ingestor unless the source needs to be re-read) and re-present
- **abort** — exit without writing anything

After approval, write all files in one batch:
1. New pages
2. Updates to existing pages
3. `wiki/index.md` update
4. Append to `wiki/log.md`

## Step 6 — Lint

Scan the new and changed pages for:

- **Contradictions** with existing wiki content (search for related pages and compare claims)
- **Broken `[[wikilinks]]`** — link target file doesn't exist anywhere in `wiki/`
- **Orphan pages** — newly-created pages with no inbound links from other wiki pages (flag, don't auto-fix; sometimes a fresh page legitimately has no inbound links yet)
- **Missing "See also"** section
- **Frontmatter completeness** — title, type, sources, created, updated all present

Auto-fix mechanical issues (incomplete frontmatter, obvious wikilink target typos where the intended target is unambiguous). Surface judgment calls (contradictions, orphans, missing cross-references) for the user.

## Step 7 — Commit prep

Run `git status` and `git diff --stat` (via Bash). Propose a commit message in this form:

```
subtopic(<name>): <short summary>

Pages created: <list>
Pages updated: <list>
Sources ingested: <count>
```

Print the proposed message and the staged-changes summary. **Do not run `git commit`.** The user runs the commit themselves.

If invoked by `/research-topic`, also return a structured outcome (subtopic name + completed/skipped/aborted + page counts) so the master-file checkbox can be flipped.

## Budget

- Max 15 fetches per invocation, combined across explorer + ingestor calls
- Max 5 sources ingested per invocation (counts the "ingest now" pile at GATE 1; "save for later" entries do not count)
- Max 5 new wiki pages per invocation (excluding source-summary pages)

If GATE 1 yields more than 5 "ingest now" picks, stop and ask the user to prioritize down to 5 (or extend explicitly). If any budget is hit mid-loop, stop, report progress, ask the user whether to extend.
