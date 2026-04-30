# Knowledge Base Builder

A local, AI-maintained knowledge base for any topic — powered by Claude Code and viewed in Obsidian. Built on top of [Karpathy's LLM Wiki concept](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) and extended into a research workflow: the agent helps you find sources, plan structure, ingest material, verify claims across sources, and answer questions over the resulting wiki.

The wiki itself is the backbone — plain markdown files you fully own. The agent gains a small set of operations on top: Explore, Outline, Ingest, Question-driven ingest, Triangulate, Query, and Lint.

## Setup

1. **Clone this repo** to a directory of your choice.
2. **Open the folder as an Obsidian vault** (File → Open vault → pick the folder). Optional but recommended for browsing `[[wikilinks]]`.
3. **Set your topic** in `CLAUDE.md` — edit the `**Topic**`, `**Description**`, and `**Scope**` lines at the top. This is the only required edit.
4. **(Optional) Tighten safety rules** in `CLAUDE.md` under *Safety & Limits* — fill in domain allow/denylists, file-type extensions, or budget overrides if the defaults are too loose for your topic.
5. **Open Claude Code** in this directory: `claude` (or use the IDE extension).
6. **Seed sources** — either add URLs to `raw/sources.md`, drop files into `raw/`, or ask the agent to *Explore* the topic and propose sources.

## Folder Structure

```
CLAUDE.md         # Agent configuration — Topic + operations + Safety & Limits
README.md         # This file
raw/              # User-owned source materials
  sources.md      # URL/source index; agent toggles [ ]/[x] and appends with approval
  (your files)    # Drop markdown, text, PDFs, images here — never modified by the agent
wiki/             # Agent-maintained knowledge base
  index.md        # Catalog of all wiki pages
  log.md          # Append-only chronological record of every operation
  _outlines/      # Working outlines produced by the Outline operation (not indexed)
  (wiki pages)    # Created and updated by the agent
attachments/      # Shared image library — both user and agent may add files here
.obsidian/        # Obsidian vault configuration
```

**Ownership rules:**
- `raw/` is user-owned. The agent reads but never modifies content. The only sanctioned writes are toggling `[ ]`/`[x]` checkboxes and *appending* approved entries to `raw/sources.md`.
- `wiki/` is agent-maintained. You can read and edit by hand, but the agent treats it as its workspace.
- `attachments/` is shared. Either side may add images; the agent will not delete user-added files without approval.

## Operations

Open Claude Code in this directory and use natural language. The agent recognizes these operations (defined in `CLAUDE.md`):

- **Explore** — `"Explore [topic]"` — research-only; agent surfaces candidate sources and asks before adding any to `raw/sources.md`. Offers an Outline as a follow-up.
- **Outline** — `"Outline [topic]"` — planning-only; agent proposes a wiki page structure (pages, types, key questions, cross-links) before any ingestion. Saved to `wiki/_outlines/`.
- **Ingest** — `"Ingest [file or URL]"` or `"Process my sources"` — read source material, create summary + entity/concept pages, update the index and log.
- **Question-driven ingest** — `"Research [question]"` — agent runs a brief interview to scope the question, decomposes into sub-questions, ingests only sources that answer them, and ends with a coverage report.
- **Triangulate** — `"Triangulate [page or claim]"` — verify claims against ≥2 independent sources; classify as confirmed, contested, single-source, or unsupported; annotate pages accordingly.
- **Query** — `"[any question about your topic]"` — search the wiki and answer with `[[wikilinks]]` citations. Flags gaps.
- **Lint** — `"Lint the wiki"` — health-check for contradictions, stale claims, orphan pages, and cross-reference gaps.

## Safety & Limits

`CLAUDE.md` includes a *Safety & Limits* section with three tiers:

- **Hard rules** — what the agent will refuse regardless of instruction (no script execution, no auth flows, no writes outside sanctioned paths, no exfiltration).
- **Soft rules** — what requires explicit per-operation approval (file-type allowlist, domain allow/denylist, file-size cap, mirroring images, page deletion, large batch creates).
- **Budgets** — agent self-limits on fetches and pages-per-operation; user can override per run.

Edit the lists in that section to tighten behavior for your topic. Approvals and budget overrides are recorded in `wiki/log.md` for auditability.

## Design Principles

- **`raw/` is user-owned**: source materials are never silently modified.
- **`wiki/` is agent-maintained**: every page is regeneratable from sources.
- **Everything is markdown**: works with Obsidian, git, grep, and any editor.
- **Sources are tracked**: every claim traces to a source via inline citations, frontmatter, and the log.
- **Local-first**: the wiki lives entirely in this directory. No external service required to read or own it.

## Credits

Inspired by [Andrej Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) idea — a personal wiki where an LLM does the writing and a human steers. This project takes that core and adds source-gathering, planning, and verification operations so it scales to building a serious knowledge base on any topic.
