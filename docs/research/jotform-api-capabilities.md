# Jotform API capabilities — research for the event-registration stats dashboard

Research ticket: [issue #2](https://github.com/theperiyar/periyarinternational/issues/2), "Assess Jotform's API
capabilities for pulling event/participant stats", a child of the wayfinder map
[issue #1](https://github.com/theperiyar/periyarinternational/issues/1) ("Stats dashboard for event
registrations — map").

This is preparatory research only. No implementation decisions are made here. All claims are sourced from
Jotform's own docs/support pages (linked inline); no third-party blog posts or Stack Overflow answers were
used as primary evidence.

---

## 1. Endpoints for retrieving form/submission data

Jotform's REST API is resource-based, rooted at `https://api.jotform.com` (regional variants exist:
`https://eu-api.jotform.com` for EU accounts and `https://hipaa-api.jotform.com` for HIPAA accounts).
Source: [api.jotform.com/docs/](https://api.jotform.com/docs/).

The relevant endpoints, confirmed against Jotform's own v1 API reference
([jotform.com/apidocs-v1/](https://www.jotform.com/apidocs-v1/)) and cross-checked against the docblocks in
Jotform's official PHP client ([jotform/jotform-api-php](https://github.com/jotform/jotform-api-php/blob/master/JotForm.php)):

| Purpose | Method + path | Notes |
|---|---|---|
| List all forms in the account | `GET /user/forms` | Supports `offset`, `limit`, `filter`, `orderby`. Returns title, creation date, new/total submission counts per form. |
| Get a single form's metadata | `GET /form/{id}` | |
| List a form's questions/fields | `GET /form/{id}/questions` | Returns question properties (type, name, text, order) for every field on the form. |
| Get a single question | `GET /form/{id}/question/{qid}` | |
| List a form's submissions | `GET /form/{id}/submissions` | Supports `offset`, `limit`, `filter`, `orderby` (see §4–5). |
| List all submissions across the account | `GET /user/submissions` | Same paging/filter parameters as above, account-wide instead of per-form. |
| Get a single submission | `GET /submission/{id}` | Returns full answer set for one submission. |
| Create/update/delete a submission | `POST /form/{id}/submissions`, `POST /submission/{id}`, `DELETE /submission/{id}` | Write operations; not needed for a read-only stats sync. |

Sources: [api.jotform.com/docs/](https://api.jotform.com/docs/), [jotform.com/apidocs-v1/](https://www.jotform.com/apidocs-v1/), [jotform-api-php JotForm.php](https://github.com/jotform/jotform-api-php/blob/master/JotForm.php).

For the planned Cloudflare Worker cron sync, the relevant pair is `GET /form/{id}/questions` (once, to map
field IDs to labels) and `GET /form/{id}/submissions` (repeatedly, paginated, to pull registration data).

## 2. Authentication

The API is authenticated with a per-account **API key**, sent either as:

- a query parameter: `https://api.jotform.com/user?apiKey={myApiKey}`, or
- an HTTP header: `APIKEY: {myApiKey}`

Source: [api.jotform.com/docs/#authentication](https://api.jotform.com/docs/).

Keys are created from the account's API settings page and can be scoped to one of two access levels at
creation time:

- **Read Access** — "view-only mode, so others can see the information without making any changes"
- **Full Access** — "full access to your data—meaning others can add, edit, or delete information as needed"

There is no per-form or per-scope restriction offered in the key-creation UI (users have requested this as a
feature; it isn't available). For a read-only stats sync, a **Read Access** key is sufficient and is the
correct minimum-privilege choice.

Source: [How to Create a Jotform API Key](https://www.jotform.com/help/253-how-to-create-a-jotform-api-key/).

Jotform does **not** offer general-purpose OAuth 2.0 for API authentication. The only OAuth support found is
unrelated — it's for configuring a custom outbound email sender (Gmail/Microsoft) for form notification
emails, not for authorizing API access to form/submission data.

Source: [Can I use OAuth 2 with your API?](https://www.jotform.com/answers/4620423-can-i-use-oauth-2-with-your-api)

## 3. Rate limits

Rate limiting is a **daily call-count cap**, tiered by subscription plan, and resets at midnight Eastern
Standard Time:

| Plan | Daily API calls |
|---|---|
| Starter (free) | 1,000 / day |
| Bronze | 10,000 / day |
| Silver | 50,000 / day |
| Gold | 100,000 / day |
| Enterprise | No limit |

When the cap is hit, forms keep working but integrations/API calls start returning an "API-Limit exceeded"
error until the next reset. Jotform says limits can be increased on request ("contact us").

Source: [Daily API Call Limits](https://www.jotform.com/help/406-daily-api-call-limits/) (also summarized on
[api.jotform.com/docs/](https://api.jotform.com/docs/)).

This daily call cap is a **separate axis** from the account's submission storage/monthly-submission caps
described in §7 below — a sync job can exhaust its daily call budget from polling frequency alone, independent
of how much data actually exists.

## 4. Pagination

`GET /form/{id}/submissions` and `GET /user/submissions` accept:

- `offset` (integer, optional) — start position of the result set, default `0`
- `limit` (integer, optional) — results per call, **default 20, hard maximum 1,000**

There is no documented way to raise the per-call maximum above 1,000 — full retrieval of a form with more than
1,000 submissions requires multiple calls, incrementing `offset` by the `limit` used each time, until a call
returns fewer than `limit` results.

Sources: [jotform.com/apidocs-v1/](https://www.jotform.com/apidocs-v1/), [Submission API limiting to 20](https://www.jotform.com/answers/14183481-submission-api-limiting-to-20).

The JSON response includes a `resultSet` object echoing the query (`offset`, `limit`, `orderby`, `filter`) and
a `count` — useful for detecting the end of a paginated walk during a cron sync.

## 5. Filtering submissions

`filter` is a JSON object, URL-encoded, passed as a query parameter. Supported patterns documented by Jotform
support:

- **By status**: `{"new":"1"}` — unread/new submissions only
- **By date**, using a `field:operator` key, e.g. `{"created_at:gt":"2020-08-29 00:00:00"}` — submissions
  created after a given timestamp (per Jotform support, `gt`/`lt` reliably work against date and numeric-ID
  fields)
- **By submission ID**: `{"id:gt":"4746216167904872631"}` — useful for incremental/delta polling, pulling only
  submissions newer than the last-seen ID
- Comparison operators mentioned: `gt` (greater than), `lt` (less than), `ne` (not equal to)

`orderby` sorts by a field name (e.g. a form's `created_at`/`submissionDate`) — direction is appended, e.g.
`submissionDate:DESC`.

Sources: [Using "filter" option on curl API call](https://www.jotform.com/answers/2545819-using-filter-option-on-curl-api-call), [JotForm API: Sort and filter submissions](https://www.jotform.com/answers/2078804-jotform-api-sort-and-filter-submissions), [jotform-api-php JotForm.php docblocks](https://github.com/jotform/jotform-api-php/blob/master/JotForm.php).

For the cron-based KV sync this is directly useful: filtering by `id:gt` (or `created_at:gt`) against the
last successfully-synced submission ID/timestamp lets the Worker do incremental syncs instead of re-pulling
the full submission set every run — important given the daily call cap in §3.

Filtering by a specific **answer/field value** (rather than metadata like status/date/id) is referenced as
possible via the same `{"field:operator":"value"}` syntax, but Jotform's own docs/support content does not
give a worked example against a custom form field — treat that as unconfirmed/needs-testing against a real
form rather than a documented guarantee.

## 6. Aggregation / analytics endpoints

No aggregation, reporting, or analytics endpoint was found in either the current API reference
([api.jotform.com/docs/](https://api.jotform.com/docs/)) or the v1 endpoint listing
([jotform.com/apidocs-v1/](https://www.jotform.com/apidocs-v1/)) — there is no `/form/{id}/analytics`,
`/form/{id}/reports` summary-stats endpoint, or submission-count-by-field endpoint documented as part of the
public API surface. `GET /user/forms` and `GET /form/{id}` return a coarse **new vs. total submission count**
per form (useful for a lightweight "how many registrations" tile without pulling all raw records), but nothing
more granular (e.g., breakdowns by answer value, time series, per-event dedup counts).

Jotform's own support desk, when asked directly "I'd like to access form analytics via the API," did not
point to any API endpoint — it redirected the user to the dashboard's visual Analytics feature (UI-only), which
is consistent with there being no such endpoint.

Source: [API for Form Analytics](https://www.jotform.com/answers/1387854-api-for-form-analytics).

**Conclusion: aggregation is a client-side responsibility.** Any stats dashboard (registration counts by
event, by date, by status, etc.) must be computed by the consumer — in this architecture, by the Cloudflare
Worker — over the raw submission records pulled via `GET /form/{id}/submissions`, not delegated to Jotform.
This matches the wayfinder map's premise (Worker syncs raw submissions into KV as a JSON snapshot) and confirms
that premise is necessary, not just a stylistic choice.

## 7. Full historical submissions: plan-tier dependency

Retrieving *all* submissions ever made to a form is **not gated by a special API permission or paid API
tier** — the `/submissions` endpoints are available on every plan including the free Starter tier. What
*does* vary by plan is how much history Jotform actually **retains**, via two separate account-level caps
(distinct from the daily API call cap in §3):

| Plan | Monthly new submissions | Total submission storage |
|---|---|---|
| Starter (free) | 100 / month | 500 total |
| Bronze | 1,000 / month | 10,000 total |
| Silver | 2,500 / month | 25,000 total |
| Gold | 10,000 / month | 100,000 total |

Source: [jotform.com/pricing/](https://www.jotform.com/pricing/).

The **total submission storage** cap is the one that matters for "full historical" backfill: it is
account-wide (all forms combined), not per-form, and once it's reached, **Jotform automatically deletes the
oldest stored submission to make room for each new one**. Forms keep accepting new entries; they don't get
disabled — the old data is just permanently gone (recoverable from trash for only 30 days after deletion).

Sources: [Understanding Your Jotform Account Usage and Limits](https://www.jotform.com/help/408-understanding-your-account-usage-and-limits/), [Managing Total Submission Storage](https://www.jotform.com/answers/36156471-managing-total-submission-storage), [Free Plan: Submission Limit per Form](https://www.jotform.com/answers/34730381-free-plan-submission-limit-per-form).

**Implication for this project:** on the free/Starter plan, "full historical" via the API can only ever mean
"whatever is left of the most recent ~500 account-wide submissions" — anything older that's already been
auto-purged by the storage cap is unrecoverable through the API (or any other means) regardless of API
tier. If Periyar International's Jotform account is on the free plan and has already exceeded 500
lifetime submissions across its forms, true full-historical backfill is **not possible** — only a partial
backfill of whatever Jotform still has stored. This is an account-plan/data-retention question, not an
API-capability question, and should be checked against the actual Jotform account before the Worker's sync
strategy is finalized (i.e., whether it needs to defensively assume gaps in history, and whether upgrading
the Jotform plan is worth it to increase retained history and monthly submission headroom).

---

## Summary of open questions for a later session

- Confirm which Jotform plan tier the Periyar International account is actually on, and how many lifetime
  submissions its event-registration forms have received, to know whether the 500-submission storage cap has
  already caused historical data loss.
- Filtering by a specific custom-field answer value (not just status/date/id) should be smoke-tested against
  a real form/API key before the Worker's sync logic depends on it.
- Decide the Worker's polling/backfill strategy given: 1,000–100,000 calls/day (plan-dependent), 1,000
  submissions/call hard cap, and `id`/`created_at` based incremental filtering as the practical way to stay
  within both the call budget and pagination limits on a cron schedule.
