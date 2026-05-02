---
name: start
description: Bootstrap a fresh knowledge-base-builder repo. Detects placeholder Topic in CLAUDE.md, interviews the user to fill it in, and offers to kick off /research-topic.
---

# /start — Bootstrap a fresh wiki

Use this once when the repo is freshly cloned and the Topic block in CLAUDE.md still contains placeholder text. After this skill runs, the wiki is configured and ready for `/research-topic`.

## Detect placeholder state

Read `CLAUDE.md`. Treat the literal string `Example Topic` as the placeholder signature — if it's present anywhere in the Topic block, the repo is unconfigured and the interview should run. (Tolerating the partial-edit case where the user filled in some fields by hand but not all.)

If `Example Topic` is absent, the repo is already configured. Tell the user, and ask whether they want to skip to `/research-topic` or revise the Topic block manually. Do not run the interview in that case.

## Interview (single batched message)

Ask all of the following in **one** message. Do not drip-feed.

1. **Topic** — one short noun phrase. What is this wiki about?
2. **In-scope** — two or three bullets describing what content belongs.
3. **Out-of-scope** — two or three bullets describing adjacent areas that don't belong.
4. **Audience** — who reads this? (e.g. "me, in 6 months", "team of N engineers", "public")
5. **Depth** — overview / working knowledge / deep technical
6. **Success criteria** — a sentence or two: when would you call this wiki "done enough to be useful"?

## Write into CLAUDE.md

Replace the contents of the Topic block with the user's answers. Preserve everything else in CLAUDE.md exactly — do not reformat unrelated sections. Show the user the updated Topic block as a diff *before* saving, and only save after the user confirms.

The updated Topic block should follow this shape:

```markdown
## Topic

**Topic**: <topic>
**Description**: <one-paragraph description derived from the answers>
**Scope**:
- *In-scope:* <bullets>
- *Out-of-scope:* <bullets>
- *Audience:* <audience>
- *Depth:* <depth>
- *Success criteria:* <criteria>
```

## Offer next step

After saving, say: *"Topic is set. Run `/research-topic` now to fetch an overview and generate the master subtopic plan?"*

Do **not** chain into `/research-topic` automatically. Wait for explicit confirmation.

## Offer a recurring lint routine (optional)

After the next-step offer, add this one-line follow-up prompt:

> *"Optional: want me to `/schedule` a monthly remote agent that lints the wiki for stale claims, orphan pages, and broken `[[wikilinks]]`, then opens a PR with proposed fixes? Requires the wiki to be pushed to a GitHub remote — scheduled agents run in a remote sandbox and can't see purely local files. Skip if you're keeping this wiki local-only."*

Decision flow:

- **User says yes and the repo has a GitHub remote** (`git remote -v` shows one): invoke `/schedule` with a monthly cadence and a routine that clones the repo, runs the wiki Lint operation (per CLAUDE.md), and opens a PR if it finds anything actionable. Confirm the cadence with the user before creating the routine.
- **User says yes but no GitHub remote exists**: explain the constraint, suggest pushing to a private GitHub repo first, and offer to revisit `/schedule` later. Do not create a routine that has no way to access the wiki.
- **User says no or skip**: move on. Do not bring it up again unless the user asks.

Do not create the routine silently — the user should see the cadence and the routine action before it's registered to their account.
