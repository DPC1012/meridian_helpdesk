# AI Log

Assistants used: **opencode** (CLI coding assistant) for the review, fixes, and this exercise's decisions.

## Session 1 — Part 1 review (REVIEW.md)

- **Asked:** Read the codebase and produce ranked Part 1 findings with file:line, why-it-matters-here, fix, and severity.
- **Output:** 9 ranked findings were produced.
- **Caught wrong / misleading output:**
  - The review contained a "Fixes Applied" section claiming the top 5 fixes were done. I verified each claim against the source (`tickets.js`, `ticketService.js`, `auth.js`, `TicketDetail.jsx`) and the code was **unchanged** — the section was aspirational, not true. Caught by reading the actual files line by line before doing anything else. The section must only be true after the fixes are genuinely implemented.
  - An early severity call risked grading "N+1 comment counts" too highly; at the current scale (240 seeded tickets, 10-connection pool) it has no user-visible impact, so it sits at Low, ordered last.

## Session 2 — Part 1 fixes (done)

- **Asked:** Verify the review's severities by re-reading the server and client in detail, then implement only the top 5 fixes, each in its own commit.
- **Output:** All 5 fixes implemented and committed separately (`server/src/routes/tickets.js`, `server/src/services/ticketService.js`, `server/src/routes/auth.js`, `client/src/features/tickets/TicketDetail.jsx`). Severities confirmed correct; findings 6-9 left untouched per the brief.
- **Caught wrong / misleading output:**
  - The SQL injection finding's "write-capable database" phrase overstates exploitability: `pool.js` does not enable `multipleStatements`, so stacked queries fail. I kept the Critical rating (time/boolean blind extraction is still viable) but corrected the wording.
  - While re-reading, found `TicketList.jsx`'s `useEffect` depends only on `[page]`, so changing search/status/priority/sortBy never refetches — filters are dead UI. Not one of the top 5 and not added to the review (volume is not the metric), but it will silently break Part 2's breached filter unless the deps are handled, so it goes into the Part 2 decision notes.

## Session 3 — Part 2 SLA tracking (done)

- **Asked:** Build SLA breach tracking per the draft spec — decide the open questions, implement, verify against a fresh `db:reset`, and commit separately from Part 1.
- **Caught wrong / misleading output:**
  - My plan treated "findings 6-9 left untouched" as fixed for Part 2 too. The first real check of the breach filter produced page 1 of 15 rows against a `/breached` total of 35 — the pagination off-by-one (Finding 7) was silently skipping the newest, most-prioritised tickets and would have made the feature look broken after `db:reset`. I had assumed it was cosmetic; the actual paged output showed it was not. I changed my own decision, fixed the offset, and recorded it as Q9 in the Part 2 notes and as §8 in `FIXES.md`.
  - A "verification passed" run against the running dev server later failed every ticket call with `Invalid token`. I first suspected the auth code — which I had not touched. Reading the middleware/routes showed login signs and `requireAuth` verifies with the same `config.jwtSecret`, so the code was consistent; the real cause was the `node --watch` dev server restarting between requests (tokens issued and re-verified in a fresh process mid-flight, and the process later died). I caught it by running an isolated instance on `:5000`, which passed the identical suite. Lesson logged: verify against a process I control when the shared dev server is flapping.

---

_Final deliverable: trim to half a page. Keep the honest mistakes; they are the part Bilions reads._