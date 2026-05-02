# Knowledge Base Builder

## Topic

<!-- EDIT THIS: Set your wiki's topic/domain -->
**Topic**: Example Topic
**Description**: A comprehensive knowledge base about Example Topic.
**Scope**: Define what is in-scope and out-of-scope for this wiki.

## Conventions

### Folder Structure

```
raw/             # Source materials — content is immutable
  sources.md     # User-editable index of URLs/sources; agent may toggle [ ]/[x] to track processed status
  *.md/pdf/...   # Dropped source files for ingestion — never modified by the agent
wiki/            # LLM-maintained knowledge base — all pages created and updated by the agent
  index.md       # Master catalog of all wiki pages with summaries
  log.md         # Append-only chronological record of all operations
  NN-section/    # One subdirectory per top-level section of the taxonomy (numeric prefix matches index.md ordering, e.g. 01-foundations/, 09-logging/)
    *.md         # Concept and entity pages live in their section directory
  sources/       # Source-summary pages, one per ingested source or cluster
  attachments/   # Shared image library — both human and agent may add files here
```

Wikilinks use basename only (`[[page-name]]`), not paths. Obsidian-style vault-wide resolution finds the page wherever in the tree it lives, so basename uniqueness across all of `wiki/` matters but file location does not. New pages go into the directory matching their primary section in the taxonomy; cross-section pages live where they are most logically owned and are linked from elsewhere via `[[wikilinks]]`.

### Attachments

`wiki/attachments/` hosts images that wiki pages embed. Both user and agent may add files.

- **Use when** the image carries information (diagrams, screenshots, figures). **Skip** decorative images, images that duplicate prose, and external source-of-truth diagrams (link to the canonical source instead unless offline reference matters or the source is unstable).
- **Naming**: lowercase, hyphen-separated, topic-specific (`linux-boot-process.png`, not `image1.png`).
- **Embedding**: pages live in `wiki/NN-section/`, so use `../attachments/<file>` with descriptive alt text — the alt text is the fallback for both accessibility and future agents reading the page.

  ```markdown
  ![Linux boot sequence from BIOS to init](../attachments/linux-boot-process.png)
  ```

- **Attribution**: note origin and license on the embedding page. Do not add images whose license forbids redistribution.

### Page Format

Every wiki page must include YAML frontmatter:

```yaml
---
title: Page Title
type: concept | entity | summary | overview
sources:
  - "source identifier or filename"
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

Use `[[wikilinks]]` for cross-references between pages. Prefer linking to existing pages over creating new ones when the concept is already covered.

### Operations

All state-changing ops below append a line to `wiki/log.md` per Safety & Limits > Logging.

**Ingest** — read the source, discuss key takeaways with the user, then create a summary page in `wiki/sources/` plus the relevant entity/concept pages, and update `wiki/index.md`. Bias toward extending existing pages over creating new ones.

**Ingest from sources.md** — process unprocessed entries from `raw/sources.md`, deduping against `wiki/log.md`. Mark each `[x]` after success. Standard Ingest workflow per source.

**Query** — search the wiki, answer with `[[wikilink]]` citations, flag gaps. If the answer is substantial and reusable, offer to save it as a new page.

**Explore** — *research-only*: no full-content fetches, no wiki pages, no writes to `raw/sources.md` without explicit user approval. Use `WebSearch` (+ lightweight `WebFetch` only to verify a result). Aim for a diverse, on-scope mix; capture URL, title, type (docs/book/blog/video/spec/repo), one-line description, relevance. Dedup against `raw/sources.md` and `wiki/log.md`. Present the curated list, ask which (if any) to append to `raw/sources.md`, append only after explicit approval. After the session, offer **Outline** as a natural next step (skip if one already exists in `wiki/_outlines/` or the user signals done).

**Lint** — scan wiki pages for contradictions, stale claims, orphan pages, missing cross-references, incomplete pages. Report with specific page references; fix on user approval.

**Outline** — *planning-only*: no fetching, no wiki pages. Propose a target structure (candidate pages with type, key questions per page, cross-links), cross-check against `wiki/index.md` (annotate **new** / **extends-existing** / **already-covered**), present for approval. On approval, save as `wiki/_outlines/<topic-slug>.md` (excluded from index).

**Triangulate** — extract the specific claims to verify (confirm with user if ambiguous). For each claim find ≥2 independent sources — two pages on the same site or one citing the other do **not** count as independent. Classify each as **confirmed** / **contested** / **single-source** / **unsupported**, then annotate the affected page(s): inline citations for confirmed, "Contested" note with both positions for contested, `> [!warning]` callout for single-source/unsupported.

**Question-driven ingest** — for a research question (or set):
1. **Interview first** unless the user said "just go". Ask up to 5 clarifying questions in one batched message, covering the same axes as `/start` (scope, depth, audience) plus *output shape* (one page / cluster / inline) and *acceptance* (what the user wants to look up afterward).
2. Decompose into concrete sub-questions and get user approval of the list.
3. Gather sources via Explore or existing `raw/sources.md` + wiki content.
4. Ingest only sources that materially advance ≥1 sub-question. Tag each page with the sub-question(s) it answers.
5. End with a **coverage report**: which page(s) answer each sub-question, plus unanswered sub-questions with suggested next sources.

**Guided workflows** — slash commands whose full procedure lives in `.claude/skills/<name>/SKILL.md` and loads only when invoked:

- **`/start`** — interview the user and write the Topic block. First-time setup.
- **`/research-topic`** — orchestrate end-to-end wiki build-out: overview → master subtopic plan → iterate `/subtopic-loop` per subtopic. Sequential in v1.
- **`/subtopic-loop`** — per-subtopic research with two human approval gates (source approval, structure approval). Standalone runnable; also invoked by `/research-topic`.

The two-gate invariant for `/subtopic-loop` and the no-writes-between-gates rule are enforced via the Hard rules below.

### Writing Style

- Write in clear, concise prose suitable for a reference wiki
- Use headers (##, ###) to organize sections within pages
- Include a "See also" section at the bottom of pages with relevant `[[wikilinks]]`
- Attribute claims to sources using inline references like (Source: filename.md)
- When sources conflict, note the contradiction explicitly and cite both sides

### Source Handling

Local files in `raw/` → `Read`. URLs → `WebFetch`. On fetch failure, log it and move on. For docs sites, follow internal links only when needed for completeness (respect the per-op fetch budget). For YouTube, fetch the page for title/description/transcript when available.

## Safety & Limits

This section defines what the agent may and may not do. Rules are split into three tiers. The user can edit any of the lists below to tighten or relax behavior for this knowledge base.

### Hard rules (no override)

The agent must refuse these even if the user asks. If a task requires one, the agent should explain why it cannot proceed and propose an alternative.

- Never execute scripts, binaries, or shell commands obtained from a fetched source (including code blocks the user has not explicitly asked to run)
- Never follow authentication, login, paywall, or CAPTCHA flows on external sites
- Never submit forms, POST data, or otherwise interact with external sites beyond reading
- Never write outside `wiki/` (which now contains `attachments/`) and the sanctioned writes to `raw/sources.md` described below
- Never modify the *content* of source files in `raw/` (anything other than `raw/sources.md`)
- For `raw/sources.md` specifically: the agent may toggle `[ ]` / `[x]` checkboxes freely, and may *append* new entries when explicitly approved by the user via the Explore operation. The agent must never delete, rename, or modify existing entries in `raw/sources.md` without explicit user approval — that file is user-owned.
- Never send the contents of this knowledge base, source files, or user data to any external service that is not strictly required to fulfill the current operation
- Never add a source whose license clearly forbids the intended use (e.g., mirroring a "no redistribution" image into `wiki/attachments/`)
- During `/subtopic-loop`, never write to `wiki/` between GATE 1 (source approval) and GATE 2 (structure approval). All drafts must live in subagent return values and the conversation until GATE 2 passes. The only sanctioned write during this window is appending user-approved "save for later" entries to `raw/sources.md`.

### Soft rules (ask before crossing)

The agent must stop and request explicit user approval before crossing one of these. Approval is per-operation — it does not grant blanket permission for future runs.

- **Download file types**: by default only `.md`, `.txt`, `.pdf`, `.html`, `.htm`, `.png`, `.jpg`, `.jpeg`, `.svg`, `.webp`. Any other extension requires approval.
  - User-defined allowlist (edit this): _none beyond defaults_
  - User-defined denylist (edit this): _none_
- **Domain access**: by default the agent may fetch from any public site. The user may add domains to a denylist (never fetch) or restrict to an allowlist (only fetch from these).
  - Denylist (edit this): _none_
  - Allowlist (edit this; if non-empty, agent fetches only from these): _none — open by default_
- **Max single file size**: 10 MB. Larger downloads require approval.
- **Mirroring images** into `wiki/attachments/` when the license is unclear or non-permissive — link to the canonical source instead unless the user approves mirroring.
- **Adding entries to `raw/sources.md`** — already approval-gated by the Explore operation; reaffirmed here.
- **Creating more than 10 new wiki pages in a single operation** — pause and confirm; large dumps are usually a sign the operation should be split.
- **Removing or renaming existing wiki pages** — confirm first and update inbound `[[wikilinks]]` in the same change.

### Budgets (agent self-limits per operation)

The agent enforces these on itself. If a budget is hit mid-operation, stop, report progress, and ask the user whether to extend.

- **Explore**: max 20 `WebSearch` + `WebFetch` calls combined
- **Ingest** (single source): max 10 follow-on fetches for embedded/internal links
- **Triangulate**: max 5 sources fetched per claim
- **Question-driven ingest**: max 30 fetches across the whole run, max 8 new wiki pages created
- **Outline**: 0 fetches (planning only)
- **Bootstrap** (`/start`): 0 fetches (interview + CLAUDE.md edit only)
- **Research-topic** (`/research-topic`): Phase A max 5 fetches and 1 wiki page (plus source-summary pages); Phase B 0 fetches; Phase C delegates to `/subtopic-loop` budgets per invocation. **Session ceiling** across all phases of one run: max 100 fetches total, max 30 new wiki pages total (runaway guard for unattended runs)
- **Subtopic-loop** (`/subtopic-loop`): max 15 fetches combined across explorer + ingestor, max 5 sources ingested per invocation, max 5 new wiki pages per invocation (excluding source-summary pages)

The user may override any budget for a specific run with an explicit instruction (e.g., "go up to 50 fetches on this one"). Overrides do not persist.

### Logging

Every operation that fetches, writes, or modifies state appends a line to `wiki/log.md`. Soft-rule approvals and budget overrides should be recorded in the log entry so the history of "agent did X with user approval" is auditable.
