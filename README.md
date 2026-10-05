# Team Lead Dashboard

Static dashboard (GitHub Pages) for weekly Freshservice reviews, backed by the
Google Apps Script project **Team Lead Automation** (file `Dashboard.gs`).

**Nothing secret or personal is stored in this repository.** API keys live only in
Apps Script Script Properties; reports and ticket data live only in the owner's
private Google spreadsheet ("Team Lead Dashboard — data store") and in the emailed report.

## How it works

| Piece | Where |
|---|---|
| UI (`index.html`) | GitHub Pages |
| API (`doPost`) | Apps Script web app, *Execute as: me*, *Access: Anyone* — every call is rejected unless the body carries the access token |
| Freshservice | Read-only `GET` calls from Apps Script |
| Claude | Called only from Apps Script (`api.anthropic.com/v1/messages`) |
| Report store | Private Google Sheet created by the script |

**Authenticated connection.** The browser sends `POST` with `Content-Type: text/plain` and a JSON
body `{token, action, params}` — a CORS "simple request", so no preflight (Apps Script cannot answer
`OPTIONS`). Apps Script answers with a 302 to `script.googleusercontent.com`, which `fetch` follows and
which returns `Access-Control-Allow-Origin: *`, so the JSON is readable (no `no-cors`). The token is never
put in a URL. The backend compares SHA-256 digests of the supplied and stored token.

## Script Properties

| Property | Required | Notes |
|---|---|---|
| `FRESHSERVICE_API_KEY` | yes | already present (shared with Code.gs) |
| `ANTHROPIC_API_KEY` | for AI notes | without it, reports still run with an "AI unavailable" notice |
| `DASHBOARD_ACCESS_TOKEN` | yes | ≥ 24 random characters you choose; type it into the dashboard |
| `TLD_CLAUDE_MODEL` | no | default `claude-sonnet-5-5`; run `tldVerifyClaudeModel()` to confirm it is available to your key |
| `TLD_REPORT_RECIPIENT` | no | default `tnasser@automated-health.com` (the browser cannot change it) |
| `TLD_REVIEW_BENCHMARK_HOURS` | no | default 24 — a review benchmark, **not** an SLA |
| `FS_WORKSPACE_ID` | no | shared with Code.gs |

## Report definitions

* **Week** – Monday 00:00 to next Monday 00:00 America/New_York; each boundary is resolved separately, so DST weeks are 167/169 hours.
* **Week hours** – time entries whose `executed_at` (fallback `created_at`) is in the week, credited to the entry's `agent_id` — *not* the ticket's current assignee. Billable and non-billable are separate.
* **Lifetime hours** – all-date totals on the same tickets, shown separately and labelled.
* **Resolved/closed** – `stats.resolved_at` / `stats.closed_at` only (never `updated_at`). Shown two ways: currently assigned to the agent, and agent logged time that week. Current assignment may not identify who did earlier work or closed the ticket.
* **Replies** – non-private outgoing conversations by the agent. Failed conversation requests are *unknown*, never "no reply".
* **"Billable time recorded; no public reply found—review required"** – only when the agent logged billable time that week *and* every conversation page was retrieved *and* the agent has no public reply on the ticket. Phone support, internal work and other documented circumstances can explain this; it is a review prompt, not evidence of misconduct.
* **Documentation** – share of time entries with a note, median note length. Length is a limited indicator; **no composite score is computed**.
* **SLA** – Freshservice `fr_due_by` / `due_by` compared with first response / resolution (current values on the ticket). The N-hour first-response item is a configurable **review benchmark**.

## Freshservice API behaviour relied on (verified against api.freshservice.com)

* `GET /api/v2/tickets` returns only tickets created in the last 30 days unless `updated_since` is supplied; the dashboard always supplies the week start, so older tickets updated since then are included. `include=stats` returns `resolved_at`, `closed_at`, `first_responded_at`.
* Time entries and conversations are per ticket (`/tickets/{id}/time_entries`, `/tickets/{id}/conversations`); default 30 per page, max 100; `Link` header signals the next page. All pages are fetched and records are de-duplicated by ID.
* Rate limits are per plan (100–500/min); `429` responses are retried after `Retry-After`.
* **Historical limit:** a ticket whose time was logged in the week but which has not been updated since the week start will not be listed by `updated_since`. Deleted/spam tickets are not returned.

## Execution limits

Each dashboard "step" call does ≤ ~40 s of work and saves progress; the browser keeps calling until done
(resumable; Cancel available). Hard cap of 6,000 Freshservice requests per report — anything not retrieved
is reported as incomplete. No selection (including a single employee) is guaranteed to finish in one call.

## Claude

Sent: aggregated metrics, ticket IDs, status/priority, hours, reply/note counts and flag codes, with agents
replaced by aliases. **Not sent:** ticket subjects, bodies, notes, requester/agent names or emails.
Truncating ticket text does not anonymise it; sending raw ticket text would require organisational
approval (PHI/HIPAA review and an appropriate agreement with Anthropic) and is not implemented.
Returned ticket references are validated against the evidence; unsupported ones are removed.
Output is labelled as an AI-generated draft for manager review.

## Email

`Email Report` sends the stored HTML whose SHA-256 matches the report on screen, only to the configured
recipient. A second send of the same report requires explicit confirmation; a script lock prevents concurrent sends.
No recurring email is installed by the dashboard.

## Editor helpers

`tldCheckSetup()`, `tldVerifyClaudeModel()`, `tldTestFreshservice()`, `tldSelfTestWeeks()`.
