# Part 2 — Decision Notes (SLA Breach Tracking)

## What the spec left open, what I chose, and why

**Q1 — What counts as "responded"?**
Chose: a ticket is responded to once an **agent or admin** has commented on it. Requester comments and claiming do not count. Why: the feature is about support answering customers — claiming a ticket is not answering it, and a requester's follow-up is not a response. This is also what the seed data supports: resolved/closed tickets all carry comments.

**Q2 — Timer anchor**
Chose: `created_at`. Breached = elapsed time since creation exceeds the priority target AND the ticket still has no agent/admin response. `updated_at` is wrong because any comment (including the requester's) bumps it and would silently reset the clock.

**Q3 — Are resolved/closed tickets eligible?**
Chose: a ticket with an agent/admin response is never breached, whatever its status. Resolved/closed tickets have responses in the seed, so they can't be "unanswered"; I don't hard-exclude them by status, the response rule handles it.

**Q4 — Store or compute?**
Chose: compute on every request, no schema change. The brief says "compute breach status and expose it", and the product expects it to be quick. SLA targets already live in `config.slaTargets`.

**Q5 — What SLA state to expose**
Chose: `is_breached` (boolean) plus `sla_due_at` (deadline timestamp). The badge only needs the boolean; `sla_due_at` gives the detail page something meaningful to show ("due at …"). Nothing more.

**Q6 — How the breached filter works**
Chose: the `breached` filter is part of the SQL `WHERE`, repeating the breach predicate (or a subquery), never filtered in JS after paging. Filtering in JS would lose breached tickets across pages. Column/parameter interpolations stay whitelisted, applying the Part 1 SQL-injection fix.

**Q7 — Timezone consistency**
Chose: compute elapsed time in SQL with MySQL `NOW()`/`TIMESTAMPDIFF` so database time (compose sets `+05:30`) and application time agree. No mixing of JS `Date.now()` with DB timestamps.

**Q8 — Part 1 findings that changed Part 2**
`TicketList.jsx`'s `useEffect` depends only on `[page]`, so changing any filter never refetches — the existing search/status/priority controls are dead UI. The breached filter would hit the same bug, so the effect's dependency handling is fixed as part of this feature. Without that fix the feature would look broken after `db:reset`.

**Q9 — Finding 7 (pagination off-by-one) fixed too**
`listTickets` computed `offset = page * PAGE_SIZE`, so page 1 silently skipped the first 20 rows (Finding 7, documented as unfixed in Part 1). With a `breached` filter the symptom is visible: 35 breached tickets but page 1 shows only 15 (the newest 20 — exactly the tickets support wants first — are missing). Fixed to `(page - 1) * PAGE_SIZE` as part of this feature and noted here. Same rationale as Q8: a Part 1 finding that makes the finished feature look broken.

**Left ambiguous on purpose and not built:** no colour/markup beyond the requested red badge, no per-ticket "minutes left" countdown, no reporting/export. The product asked for a badge and a filter; anything more is scope creep inside a 48-hour window.

