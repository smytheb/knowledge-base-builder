---
name: lint
description: Health-check wiki pages for contradictions, stale/overdue pages, orphans, broken [[wikilinks]], and incomplete frontmatter. Reports with specific page references and fixes on user approval. Runs standalone or scoped to a set of just-changed pages (e.g. from /subtopic-loop Step 6).
---

# /lint — Wiki health check

Scans wiki pages for problems and fixes them on approval. This is the single owner of the lint checklist — other skills (e.g. `/subtopic-loop`) invoke it rather than duplicating the checks.

## Scope

- **Full lint** (default) — scan all of `wiki/` except `_outlines/`.
- **Scoped lint** — if the caller passes a list of touched paths (e.g. from `/subtopic-loop`), check only those pages plus their immediate link neighbourhood (pages they link to / that link to them). Say which mode you're in up front.

`wiki/index.md` and `wiki/log.md` are machinery, not content pages: check their `[[wikilinks]]` resolve, but skip the frontmatter, orphan, "See also", and staleness checks on them.

## Checks

Run all of these; fetch nothing (this is a local, read-only scan until fixes are approved):

1. **Broken `[[wikilinks]]`** — a link whose basename resolves to no file anywhere under `wiki/`. Report the source page + missing target.
2. **Orphan pages** — pages with no inbound `[[wikilinks]]` from any other wiki page. Flag only; don't auto-fix — a freshly created page legitimately may not have inbound links yet.
3. **Frontmatter completeness** — every page has `title`, `type`, `status`, `sources`, `created`, `updated`, `review-date` (per `CLAUDE.md > Page Format`).
4. **Stale / overdue pages** — `review-date` in the past, or `status: needs-update`. List them so the user can schedule a refresh.
5. **Missing "See also"** — a page with no "See also" section (or an empty one).
6. **Contradictions** — claims on a page that conflict with claims on a related page. Search for related pages by shared `[[wikilinks]]` and topic keywords, compare, and report both sides with page references.
7. **Missing cross-references** — two pages clearly about related concepts that don't link to each other.

## Report

Group findings by severity, most actionable first. For each: the page path, the specific issue, and (where applicable) the suggested fix. Keep it scannable — one line per finding where possible.

## Fix

Split findings into two piles:

- **Auto-fixable mechanical issues** — incomplete frontmatter (fill sensible defaults: `status: published`, `review-date` ~12 months out from `updated`), unambiguous `[[wikilink]]` target typos, obviously-missing "See also" stubs. Apply these after a single batched confirmation.
- **Judgment calls** — contradictions, orphans, missing cross-references, stale content. Surface each with a proposed action and let the user decide per item.

Never delete or rename a page as a "fix" without explicit approval (soft rule in `CLAUDE.md`). When editing pages, bump each touched page's `updated` field.

## Log

Append one line to `wiki/log.md`: date, `lint` (scoped or full), pages checked, issues found, fixes applied.
