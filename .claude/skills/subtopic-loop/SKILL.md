---
name: subtopic-loop
description: Run the full research loop for a single subtopic — explore, source-approve, ingest (drafts only), structure-approve, write, lint, wrap-up. Standalone runnable; also invoked by /research-topic.
---

# /subtopic-loop — Per-subtopic research

Runs one subtopic end-to-end with two human approval gates:

- **GATE 1** after Explore — user approves which candidate sources to ingest now / save for later / discard
- **GATE 2** after draft assembly — user approves the proposed wiki changes before *any* writes happen

**Critical invariant:** never write to `wiki/` between GATE 1 and GATE 2 — drafts live in subagent return values and the conversation until GATE 2 passes. The only sanctioned write in that window is appending approved-for-later entries to `raw/sources.md`. This is enforced by `CLAUDE.md > Safety & Limits > Hard rules` (canonical statement); it is repeated here only as the operational reminder.

## Inputs

- Subtopic name (free text). If it matches an entry in `wiki/_outlines/<topic>-master.md`, that entry's checkbox is flipped on successful completion.
- Optional: key questions for the subtopic (passed in by `/research-topic`, or asked from the user if running standalone).

If running standalone with no key questions provided, ask the user briefly: *"Any specific questions this subtopic page should answer? (Optional — can also let the explorer infer from the name.)"*

## Step 1 — Explore

Spawn the **explorer** subagent. Pass it: the subtopic name, key questions, in/out-of-scope notes from CLAUDE.md, and a fetch budget (default 8, leaving headroom under the per-invocation total in CLAUDE.md > Budgets).

Subagent returns a candidate list with: URL, title, source type, one-line description, relevance to the subtopic / specific key question, dedup status against `raw/sources.md` and `wiki/log.md`.

## Step 2 — GATE 1: Source approval

Present the candidates to the user. For each, the user chooses:

- **ingest now** — use this source in Step 3
- **save for later** — append the candidate to `raw/sources.md` (this is the explicit user approval that CLAUDE.md's Explore operation requires)
- **discard** — drop it, don't touch `raw/sources.md`

If the user approves zero sources to ingest, ask whether to:
- **Re-explore** with revised framing (different keywords, broader/narrower scope) — re-spawn the explorer with the new framing, then re-prompt
- **Synthesis-only** — proceed to Step 4 using only existing wiki content (useful when the subtopic is already well-covered and just needs cross-linking or a summary page)
- **Abort** the loop

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
- **Section placement** — confirm each new page lands in a directory matching its primary section in the taxonomy. If no master outline exists yet (standalone run, no `wiki/_outlines/<topic>-master.md`), pick or create a numbered section directory and call out the choice in the preview so the user can rename or relocate before approving.

After the compact preview, prompt: *"Approve as-is, revise, abort, `show full <page>` to inspect a draft body, or `verify <page>` for an adversarial fact-check before deciding."* On `show full <page>`, print that page's full body inline and re-prompt.

**Optional verify (opt-in).** On `verify <page>` — or proactively for a page making high-stakes or surprising claims — spawn the **verifier** subagent with the drafted page and its sources (but *not* your framing of why it's correct). It returns a disprove-oriented findings list; fold any real issues into the draft before re-presenting. This is off the default fast path — only run it when asked or when a draft's confidence warrants it.

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

Run the **`/lint`** skill scoped to the pages just created or changed (pass it the list of touched paths). It checks contradictions, broken `[[wikilinks]]`, orphans, missing "See also", and frontmatter completeness — auto-fixing mechanical issues and surfacing judgment calls. Don't restate the checklist here; `/lint` owns it.

## Step 7 — Wrap-up

First check whether the project is under git: test for a `.git/` directory at the project root (via Bash `test -d .git`).

**If git is present** — run `git status` and `git diff --stat`. Print the staged-changes summary plus a proposed commit message in this form:

```
subtopic(<name>): <short summary>

Pages created: <list>
Pages updated: <list>
Sources ingested: <count>
```

**Do not run `git commit`.** The user runs the commit themselves.

**If git is not present** (e.g. the repo was downloaded as a ZIP) — skip the git commands and just print a plain summary using the same fields:

```
Subtopic: <name>
Pages created: <list>
Pages updated: <list>
Sources ingested: <count>
```

If invoked by `/research-topic`, also return a structured outcome (subtopic name + completed/skipped/aborted + page counts) so the master-file checkbox can be flipped. This return value is independent of whether git is in use.

## Budget

Follows the per-operation caps in `CLAUDE.md > Safety & Limits > Budgets`: ~15 combined fetches across the explorer + ingestor(s), and ingest at most ~5 sources per invocation (the "ingest now" pile at GATE 1; "save for later" entries don't count). If GATE 1 yields more "ingest now" picks than that, stop and ask the user to prioritize down or extend explicitly.
