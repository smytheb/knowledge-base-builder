---
name: start
description: Bootstrap a fresh knowledge-base-builder repo. Detects placeholder Topic in CLAUDE.md, interviews the user to fill it in, and offers to kick off /research-topic.
---

# /start — Bootstrap a fresh wiki

Use this once when the repo is freshly cloned and the Topic block in CLAUDE.md still contains placeholder text. After this skill runs, the wiki is configured and ready for `/research-topic`.

## Detect placeholder state

Read `CLAUDE.md` and inspect the Topic block. Treat any of these as placeholder state:

- Topic value is `Example Topic`
- Description is `A comprehensive knowledge base about Example Topic.`
- Scope is `Define what is in-scope and out-of-scope for this wiki.`

If none of those strings are present, the repo is already configured. Tell the user, and ask whether they want to skip to `/research-topic` or revise the Topic block manually. Do not run the interview in that case.

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
