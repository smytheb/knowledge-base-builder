# Knowledge Base Builder

A local, AI-maintained knowledge base for any topic — powered by Claude Code and viewed in Obsidian.

Built on top of [Karpathy's LLM Wiki concept](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) and extended into a research workflow: the agent helps you find sources, plan structure, ingest material, verify claims across sources, and answer questions over the resulting wiki.

The wiki itself is the backbone — plain markdown files you fully own. The agent gains a set of guided workflows on top (`/start`, `/research-topic`, `/subtopic-loop`) plus granular operations (Explore, Outline, Ingest, Triangulate, Query, Lint) for finer control.

---

## Quick Start

*No prior terminal experience required.*

### 1. Install prerequisites

- **Claude Code** — Anthropic's CLI. Install instructions: <https://docs.claude.com/en/docs/claude-code/quickstart>
- **Obsidian** *(optional but recommended)* — for browsing the wiki with backlinks and a graph view: <https://obsidian.md>

### 2. Download the repo

On this repo's GitHub page, look just above the file list for the green **Code** button. Click it, then choose **Download ZIP**.

Unzip the file wherever you'd like to keep your knowledge base. (If you're not sure how to unzip, a quick web search for "how to unzip a file on \[macOS / Windows\]" will get you there.)

> If you already use git, feel free to `git clone` the repo instead.

### 3. Open a terminal

- **macOS** — press `Cmd+Space`, type `Terminal`, press Enter
- **Windows** — open the Start menu, type `PowerShell`, press Enter
- **Linux** — open your usual terminal

Then change into the unzipped folder:

```bash
cd path/to/knowledge-base-builder
```

### 4. Start Claude Code

```bash
claude
```

Claude Code opens inside your terminal. The first time you run it, it'll walk you through signing in.

### 5. (Optional) Pre-approve common tool permissions

To cut down on harness approval prompts during research, copy the example permissions file:

```bash
cp .claude/settings.example.json .claude/settings.local.json
```

Edit `.claude/settings.local.json` to add the `WebFetch(domain:...)` entries that match your topic (e.g., `kernel.org`, `man7.org`). `settings.local.json` is gitignored — it's your personal config.

The `WebSearch` entry is safe to leave on for everyone (read-only, no domain). Anything not pre-approved still prompts, with an "always allow this domain" option that grows the file automatically.

> CLAUDE.md's *Safety & Limits* (hard rules, denylist, budgets) sits on top regardless of what you allow here.

### 6. Run `/start`

Type `/start` and press Enter. The agent will:

1. Ask a handful of questions about your topic (in/out of scope, audience, depth, success criteria) in a single batched message.
2. Save your answers into `CLAUDE.md`.
3. Offer to run `/research-topic` next, which builds out a topic overview and a master subtopic plan you approve before any pages get written.

From there, the agent guides you through each step. You approve sources before they're ingested, and approve the page structure before anything is written to the wiki.

### 7. (Optional) Open the folder in Obsidian

Obsidian → **Open vault** → pick the `knowledge-base-builder` folder.

The vault is pre-configured to use basename `[[wikilinks]]`, route attachments into `wiki/attachments/`, and hide `raw/` from search. You can browse pages and click links while the agent works.

---

## Workflows

Three guided slash commands wrap the lower-level operations:

- **`/start`** — bootstrap a fresh repo. Interviews you for topic, scope, audience, depth, and success criteria, then writes the answers into `CLAUDE.md`. Run this once after downloading.

- **`/research-topic`** — master orchestrator for end-to-end build-out of the configured topic. Phase A: fetch and approve an overview source. Phase B: derive a subtopic plan saved as a checklist in `wiki/_outlines/`. Phase C: iterate each subtopic through `/subtopic-loop` with approval at each step. Sequential in v1; SQLite-backed parallelism is reserved for v2.

- **`/subtopic-loop <name>`** — runs the full per-subtopic research loop. Standalone-runnable, also invoked by `/research-topic`. Two human approval gates:
  - **Gate 1** after source exploration — you choose which candidates to ingest now, save for later (appended to `raw/sources.md`), or discard.
  - **Gate 2** after draft assembly — you approve the proposed new pages, updates, and links *before* any writes happen.

> The agent never writes to `wiki/` between Gate 1 and Gate 2 — drafts live in the conversation until you approve them.

---

## Granular Operations

The agent also recognizes these natural-language operations (defined in `CLAUDE.md`):

- **Explore** — `"Explore [topic]"` — research-only; surfaces candidate sources, asks before adding any to `raw/sources.md`. Offers an Outline as a follow-up.

- **Outline** — `"Outline [topic]"` — planning-only; proposes a wiki page structure before any ingestion. Saved to `wiki/_outlines/`.

- **Ingest** — `"Ingest [file or URL]"` or `"Process my sources"` — read source material, create summary + entity/concept pages, update the index and log.

- **Question-driven ingest** — `"Research [question]"` — brief interview to scope the question, decomposes into sub-questions, ingests only sources that answer them, ends with a coverage report.

- **Triangulate** — `"Triangulate [page or claim]"` — verify claims against ≥2 independent sources; classify as confirmed, contested, single-source, or unsupported; annotate pages accordingly.

- **Query** — `"[any question about your topic]"` — search the wiki and answer with `[[wikilinks]]` citations. Flags gaps.

- **Lint** — `"Lint the wiki"` — health-check for contradictions, stale claims, orphan pages, and cross-reference gaps.

---

## Folder Structure

```
CLAUDE.md         # Agent configuration — Topic + operations + Safety & Limits
README.md         # This file
.claude/
  skills/         # Guided workflow definitions (start, research-topic, subtopic-loop)
  agents/         # Subagent definitions (explorer, ingestor)
.obsidian/        # Pre-configured Obsidian vault settings
raw/              # User-owned source materials
  sources.md      # URL/source index; agent toggles [ ]/[x] and appends with approval
  (your files)    # Drop markdown, text, PDFs, images here — never modified by the agent
wiki/             # Agent-maintained knowledge base
  index.md        # Catalog of all wiki pages
  log.md          # Append-only chronological record of every operation
  _outlines/      # Working outlines + master subtopic plans (not indexed)
  attachments/    # Shared image library — both user and agent may add files here
  (wiki pages)    # Created and updated by the agent
```

**Ownership rules**

- `raw/` is user-owned. The agent reads but never modifies content. The only sanctioned writes are toggling `[ ]`/`[x]` checkboxes and *appending* approved entries to `raw/sources.md`.
- `wiki/` is agent-maintained. You can read and edit by hand, but the agent treats it as its workspace.
- `wiki/attachments/` is shared. Either side may add images; the agent will not delete user-added files without approval.
- `.claude/` and `.obsidian/` are configuration — edit if you want to tune behavior, otherwise leave alone.

---

## Safety & Limits

`CLAUDE.md` includes a *Safety & Limits* section with three tiers:

- **Hard rules** — what the agent will refuse regardless of instruction (no script execution, no auth flows, no writes outside sanctioned paths, no exfiltration, no writes between Gate 1 and Gate 2 in `/subtopic-loop`).

- **Soft rules** — what requires explicit per-operation approval (file-type allowlist, domain allow/denylist, file-size cap, mirroring images, page deletion, large batch creates).

- **Budgets** — agent self-limits on fetches and pages-per-operation; user can override per run.

Edit the lists in that section to tighten behavior for your topic. Approvals and budget overrides are recorded in `wiki/log.md` for auditability.

---

## Manual Setup

*An alternative to `/start` if you'd rather configure the topic by hand.*

1. Open `CLAUDE.md` and edit the `**Topic**`, `**Description**`, and `**Scope**` lines under the Topic heading.
2. *(Optional)* Tighten safety rules in the Safety & Limits section — fill in domain allow/denylists, file-type extensions, or budget overrides.
3. Seed sources — add URLs to `raw/sources.md` or drop files into `raw/`.
4. Run `claude` and use any operation directly (e.g. `"Explore [topic]"`).

---

## Design Principles

- **`raw/` is user-owned** — source materials are never silently modified.
- **`wiki/` is agent-maintained** — every page is regeneratable from sources.
- **Everything is markdown** — works with Obsidian, git, grep, and any editor.
- **Sources are tracked** — every claim traces to a source via inline citations, frontmatter, and the log.
- **Local-first** — the wiki lives entirely in this directory. No external service required to read or own it.
- **Human in the loop by default** — `/subtopic-loop` requires explicit approval at the source-selection and structure-write gates. Removing the human is a v2 concern, paired with SQLite-backed coordination.

---

## Credits

Inspired by [Andrej Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) idea — a personal wiki where an LLM does the writing and a human steers.

This project takes that core and adds source-gathering, planning, verification, and orchestration so it scales to building a serious knowledge base on any topic.
