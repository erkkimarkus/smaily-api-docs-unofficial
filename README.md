# Smaily API — unofficial developer docs

> **UNOFFICIAL.** This is not Smaily product documentation. It is a
> developer-written re-write of the
> [official Smaily help pages](https://smaily.com/help/api/), reorganized for
> building integrations and corrected against **observed production behavior**
> (last verified **2026-08**; see [Gotchas](guides/gotchas.md)). Where this set
> says **"In practice"** or **"Verified"**, it reflects behavior that differs
> from — or is missing from — the official docs. Behavior can change without
> notice; when something here disagrees with what you observe, trust your
> observation and open an issue.

The Smaily API is a REST-style HTTP API for managing email marketing and
automation: subscribers, segments, campaigns, automation workflows, transactional
messages, and engagement history. It powers everything from a single
transactional send to bulk-syncing hundreds of thousands of contacts.

---

## At a glance

| Property | Value |
|---|---|
| Base URL | `https://{subdomain}.sendsmaily.net/api/{endpoint}` |
| Auth | HTTP Basic (API username + password) |
| Request body | `application/json` (recommended) or `application/x-www-form-urlencoded` |
| Response body | Always JSON (even though `Content-Type: text/html` is returned) |
| Rate limit | **10 requests/second per IP** (the public docs say 5 — see [Conventions](conventions.md#rate-limits)) |
| Event model | **Pull only** — poll the action log; there are no webhooks |
| Sending volume | Effectively **unlimited** |
| Custom fields | Effectively **unlimited** per contact, auto-created on first use |

> **Note**
> Every endpoint is a distinct `.php` script (`contact.php`, `campaign.php`,
> `autoresponder.php`, …). There is no `/v1/` version prefix and no resource-path
> nesting in the REST sense — the "resource" is the script name, and the HTTP
> method (`GET` vs `POST`) selects read vs write.

---

## Base URL

```
https://{subdomain}.sendsmaily.net/api/{endpoint}
```

- `{subdomain}` — your Smaily account subdomain (shown when you create an API user).
- `{endpoint}` — e.g. `contact.php`, `campaign.php`, `autoresponder.php`.

All requests must use HTTPS. Plain HTTP is redirected and the request fails.

---

## Table of contents

### Start here
- [Getting started](getting-started.md) — your first authenticated request in under a minute.
- [Authentication](authentication.md) — HTTP Basic auth and credential management.
- [Conventions](conventions.md) — request/response formats, rate limits, pagination, dates.
- [Errors & response codes](errors.md) — full code table and how to handle each.

### Reference
- [Endpoint index](reference/endpoints.md) — every `.php` script that exists, and the plausible names that don't.
- [Subscribers](reference/subscribers.md) — create/update (single + batch), get, list by segment, opt-in, forget, custom fields, action log.
- [Segments](reference/segments.md) — list, create/update, list subscribers of a segment, segment rules.
- [Automations](reference/automations.md) — list workflows (`autoresponder.php` and the undocumented `workflows.php`), trigger a workflow.
- [Messages](reference/messages.md) — transactional `send`, message action log.
- [Campaigns](reference/campaigns.md) — list, launch, statistics, unsubscribe a recipient.
- [Action log](reference/action-log.md) — the pull-based engagement event stream.

### Guides
- [Bulk contact sync](guides/bulk-sync.md) — reliably syncing many contacts (batching, timeouts, idempotency, the single-`101` response).
- [Gotchas](guides/gotchas.md) — the quirks worth knowing before you build.

---

*Source: based on the official Smaily API documentation at <https://smaily.com/help/api/>.*
