---
name: triangulate
description: Verify specific claims against ≥2 independent sources, classify each as confirmed / contested / single-source / unsupported, and annotate the affected wiki page(s) accordingly. Optionally runs an adversarial verifier pass to try to disprove a claim before confirming it.
---

# /triangulate — Cross-source claim verification

Takes one or more claims (usually from a wiki page) and checks how well the evidence actually supports them.

## Steps

1. **Extract the claims.** Identify the specific, checkable claims to verify. If a page is passed rather than explicit claims, pull out its load-bearing factual assertions and confirm the list with the user if it's ambiguous.
2. **Find independent support.** For each claim, find **≥2 independent sources**. Independence is strict: two pages on the same site, a page and its own mirror, or one source citing the other do **not** count as independent. Respect the per-operation fetch cap in `CLAUDE.md > Budgets` (~15 combined; spend them across the claims, not all on one).
3. **(Optional) Adversarial pass.** For a high-stakes or surprising claim, spawn the **verifier** subagent — give it the claim and the candidate sources but **not** your reason for believing it, and let it try to disprove. Fold its findings in before classifying.
4. **Classify** each claim:
   - **confirmed** — ≥2 independent sources agree
   - **contested** — independent sources disagree
   - **single-source** — only one source found
   - **unsupported** — no source found, or sources contradict the claim
5. **Annotate the affected page(s):**
   - *confirmed* → add inline citations to the supporting sources
   - *contested* → add a "Contested" note stating both positions with a citation for each
   - *single-source* / *unsupported* → add a `> [!warning]` callout naming the gap; consider setting the page's `status: needs-update`
6. Bump `updated` on any page you edit.

## Log

Append one line to `wiki/log.md`: date, `triangulate`, claims checked, and the classification tally (e.g. `3 confirmed, 1 contested, 1 single-source`).
