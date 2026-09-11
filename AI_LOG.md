# AI Log

Assistants used: **opencode** (CLI coding assistant) for the review, fixes, and this exercise's decisions.

## Session 1 — Part 1 review (REVIEW.md)

- **Asked:** Read the codebase and produce ranked Part 1 findings with file:line, why-it-matters-here, fix, and severity.
- **Output:** 9 ranked findings were produced.
- **Caught wrong / misleading output:**
  - The review contained a "Fixes Applied" section claiming the top 5 fixes were done. I verified each claim against the source (`tickets.js`, `ticketService.js`, `auth.js`, `TicketDetail.jsx`) and the code was **unchanged** — the section was aspirational, not true. Caught by reading the actual files line by line before doing anything else. The section must only be true after the fixes are genuinely implemented.
  - An early severity call risked grading "N+1 comment counts" too highly; at the current scale (240 seeded tickets, 10-connection pool) it has no user-visible impact, so it sits at Low, ordered last.

## Session 2 — Part 1 fixes (to do)

- **Asked:** Implement only the top 5 fixes from the review.
- **To verify while the agent works:** fixes must be minimal, must not break the working flows (login, list, detail, comment, claim, delete), and findings 6-9 must stay untouched. Re-read each diff before committing.

## Session 3 — Part 2 SLA tracking (to do)

- **Asked:** …
- **Caught wrong / misleading output:** (record genuine mistakes as they happen — a log with no errors is not credible)

---

_Final deliverable: trim to half a page. Keep the honest mistakes; they are the part Bilions reads._