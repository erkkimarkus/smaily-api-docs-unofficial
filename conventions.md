# Conventions

How requests and responses are shaped across every endpoint.

## Request formats

The API accepts two content types:

| Content type | When to use |
|---|---|
| `application/json` | **Recommended.** Required for batch arrays and nested structures. Set `Content-Type: application/json`. |
| `application/x-www-form-urlencoded` | Simple single-record calls. The default for plain form `POST`s. |

**JSON (recommended):**

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"email": "subscriber@domain.tld"}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

**Form-encoded:**

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -d "email=subscriber%40domain.tld" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

> **Gotcha**
> Anything that takes an **array** — a batch of contacts, a list of segment IDs,
> segment `filter_data` rules — should be sent as **JSON**. Form-encoding nested
> arrays is error-prone; use `Content-Type: application/json`.

### Method selects read vs write

Most endpoints share a single `.php` script for both reading and writing, and use
the HTTP method to disambiguate:

- `GET /api/contact.php?email=...` → read a subscriber.
- `POST /api/contact.php` → create/update subscriber(s).
- `GET /api/list.php` → list segments.
- `POST /api/list.php` → create/update a segment.
- `GET /api/campaign.php` → list campaigns; `GET /api/campaign.php?id=N` → statistics.
- `POST /api/campaign.php` → launch a campaign.

## Response format

> **Note**
> "Response will be always given in JSON format" — even though the HTTP
> `Content-Type` header is `text/html`. Parse the body as JSON regardless of the
> declared content type.

Two response shapes exist:

**1. Action result** (writes) — a status object:

```json
{ "code": 101, "message": "OK" }
```

Some writes add a created `id` (`campaign.php`, `list.php`) or an array
(`autoresponder.php` echoes `addresses`, `message/send.php` returns `message_ids`).

**2. Data payload** (reads) — a JSON object or array of records, e.g. the segment
list or a subscriber's fields. These do **not** wrap results in a `code`/`message`
envelope.

See [Errors & response codes](errors.md) for the full code table.

## Rate limits

> **Verified: the real limit is 10 requests/second per IP-address.**
>
> The [official docs](https://smaily.com/help/api/general/common-principles/)
> state *"API requests are limited to up to 5 API requests per second per
> IP-address."* **That number is wrong.** The product owner has confirmed the
> actual enforced limit is **10 req/s per IP**. Build to 10/s, but keep headroom
> if you share an egress IP across integrations.

When you exceed the limit, the API returns:

```
HTTP 429 Too Many Requests
```

Handle `429` by backing off and retrying — see [Errors](errors.md#http-429-too-many-requests)
and the [bulk-sync guide](guides/bulk-sync.md).

> **Note on timeouts vs rate limits**
> The 10/s limit is about *request frequency*, not per-request duration. Large
> writes (a batch of hundreds of contacts) can take **well over 25 seconds** to
> process server-side. That is normal and unrelated to rate limiting — set
> generous client timeouts (minutes, not seconds) for bulk writes. See
> [Gotchas](guides/gotchas.md#slow-large-batches).

## Pagination

Pagination is **per-endpoint and inconsistent** — there's no global scheme. Two
patterns exist:

### Offset/page pagination (most list endpoints)

`offset` (or `page`) is a **0-based page index**, not a record offset. The page
size differs by endpoint:

| Endpoint | Param | Page size | Notes |
|---|---|---|---|
| List subscribers of a segment | `offset` | 25,000 | `limit` capped at 25,000. `offset=0` → records 0–25,000; `offset=1` → 25,001–50,000. |
| Campaign statistics (`detailed=1`) | `offset` | 10,000 | `limit` capped at 10,000. |
| List campaigns | `page` | 10,000 | |
| List automation workflows | `page` | depends on `limit` | `limit=0` returns all. |
| List templates | `page` | 1,000 | `limit` capped at 1,000. |

> **Gotcha**
> `offset=1` does **not** mean "skip 1 record" — it means "page index 1," i.e.
> skip one full page (10,000 or 25,000 records). This trips up almost everyone.

### Cursor pagination (action logs)

The action logs (`history.php`, `message/action/log.php`) support a sequence
cursor via `since_seq_id` — pass the highest `seq_id` you've seen to fetch only
newer events. This is the right tool for incremental sync. `since_seq_id` is
**mutually exclusive** with `offset`/date filters. See [Action log](reference/action-log.md).

## Dates, timezone & encoding

- **Date input/output** in most endpoints: `YYYY-MM-DD HH:MM:SS`.
- **Timezone**: action-log timestamps and many record timestamps are in
  **Europe/Tallinn** local time (not UTC). Convert deliberately. Action-log
  *query* bounds (`start_at`/`end_at`) are **UNIX timestamps in UTC**.
- Some newer endpoints (templates, message action log) return **RFC 3339** with
  an explicit offset, e.g. `2022-04-21T14:47:42+03:00`.
- **URL-encode** query values — most importantly `@` → `%40` in emails.

> **Gotcha**
> Timestamp formats are mixed: `history.php` returns `YYYY-MM-DD HH:MM:SS` in
> Europe/Tallinn, while `templates.php` and `message/action/log.php` return
> RFC 3339 with offset. Don't assume a single format across the API.

---

*Source: based on <https://smaily.com/help/api/general/common-principles/>, with rate-limit and timeout corrections verified by the product owner.*
