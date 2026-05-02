---
name: research-topic
description: Master orchestrator for building out a wiki on the configured Topic. Fetches an overview, derives a subtopic plan with checkboxes, and iterates each subtopic through /subtopic-loop with human approval at each step.
---

# /research-topic — Master orchestrator

Drives the end-to-end research flow for the Topic configured in `CLAUDE.md`. Runs subtopics **sequentially** in v1 — parallelism is reserved for the future SQLite-backed v2. Two human approval gates apply within each subtopic (delegated to `/subtopic-loop`).

## Preflight

Read `CLAUDE.md`. If the Topic block still contains the literal string `Example Topic`, the repo is unconfigured — offer to run `/start` first and wait for confirmation. Do not proceed with overview fetching against a placeholder topic.

## Phase A — Overview

1. Spawn the **explorer** subagent. Pass it the Topic, the in/out-of-scope notes from CLAUDE.md, and a fetch budget of 5. Ask for 1–3 high-quality *overview* sources (encyclopedic articles, canonical docs, well-regarded primers — not deep-dives on a single subtopic).
2. Present the candidates to the user. They pick 0–N to ingest as the overview source(s).
3. For each approved source, spawn the **ingestor** subagent. The subagent returns drafts only — it does not write to disk.
4. Show the draft(s) to the user. After approval:
   - Write the overview page to `wiki/00-overview/<topic-slug>.md` (create the section directory if needed).
   - Write any source-summary pages under `wiki/sources/`.
   - Update `wiki/index.md`.
   - Append an entry to `wiki/log.md` recording the overview ingest.

If the user declines all overview candidates, ask whether to retry with a different framing or skip Phase A and move to Phase B with the user supplying the subtopic structure manually.

## Phase B — Master plan

1. From the overview content (and any prior knowledge of the topic), derive a candidate subtopic list. Aim for 5–15 subtopics. Each entry needs:
   - **Name** — short, will become the page basename
   - **Type** — concept / entity / summary / overview (per CLAUDE.md page types)
   - **Key questions** — 1–3 specific questions the page should answer
2. Cross-check against the current `wiki/index.md`. Annotate each subtopic as **new**, **extends-existing**, or **already-covered**.
3. Write `wiki/_outlines/<topic-slug>-master.md` with this structure:

   ```markdown
   ---
   title: <Topic> — Master Plan
   type: outline
   created: YYYY-MM-DD
   updated: YYYY-MM-DD
   ---

   # <Topic> — Master Plan

   Generated from `wiki/00-overview/<topic-slug>.md`. Checkboxes track per-subtopic completion. `/subtopic-loop` toggles them on completion; user may edit this file freely.

   ## Subtopics

   - [ ] **subtopic-name** — *type:* concept — *status:* new
     - Key questions: Q1 / Q2 / Q3
   - [ ] ...
   ```

4. Present the file to the user for review. Wait for explicit approval (or revisions) before iterating. **Do not start Phase C without approval.**

## Phase C — Iterate

For each unchecked subtopic in the master file, top-down:

1. Ask: *"Run `/subtopic-loop` for **<name>**? (yes / skip / stop)"*
2. On `yes`, invoke the `subtopic-loop` skill with the subtopic name and its key questions.
3. On completion, edit the master file to flip `[ ]` → `[x]` for that entry, and update its `updated` field. Append a one-line entry to `wiki/log.md` referencing the subtopic and the pages created/updated.
4. On `skip`, leave the checkbox unchecked and continue to the next subtopic.
5. On `stop`, exit cleanly. The master file remains as-is so the run can be resumed later by re-invoking `/research-topic` (which will pick up at the first unchecked entry).

After the last subtopic, present a final summary: how many subtopics completed, how many skipped, and which (if any) of the originally-proposed subtopics are still unchecked.

## Budget

Per-phase and session-ceiling limits live in `CLAUDE.md > Safety & Limits > Budgets` (Research-topic entry). Track running totals across Phase A and every Phase C `/subtopic-loop` invocation. Before kicking off the next subtopic, check projected totals against the session ceiling — if the next loop's worst-case budget would exceed it, stop and ask the user whether to extend, narrow remaining scope, or end the run cleanly.
