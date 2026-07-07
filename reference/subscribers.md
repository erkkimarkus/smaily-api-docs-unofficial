# Subscribers

Subscribers (a.k.a. contacts) are the core of the API. One endpoint —
`contact.php` — handles create, update, single-read, and segment-listing,
disambiguated by HTTP method and parameters. Removal and the engagement log live
on their own endpoints.

| Operation | Method | Endpoint |
|---|---|---|
| [Create / update (single + batch)](#create--update) | `POST` | `/api/contact.php` |
| [Get a subscriber](#get-a-subscriber) | `GET` | `/api/contact.php?email=...` |
| [List subscribers of a segment](#list-subscribers-of-a-segment) | `GET` | `/api/contact.php?list=...` |
| [Opt-in subscriber(s)](#opt-in-subscribers) | `POST` | `/api/autoresponder.php` |
| [Forget subscriber(s)](#forget-subscribers) | `POST` | `/api/contact/forget.php` |
| [Subscriber action log](#subscriber-action-log) | `GET` | `/api/history.php` |

---

## Create / update

```
POST /api/contact.php
```

Creates a subscriber if the email is new, or updates the existing one (upsert).
Idempotent on `email`.

> **Note**
> This endpoint does **not** trigger automation workflows. To enroll a contact
> into a "form submitted" / "subscriber opted-in" workflow, use
> [Opt-in](#opt-in-subscribers) or [Trigger automation](automations.md).

### Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `email` | string | Yes | Subscriber's email address. |
| `is_unsubscribed` | `0` \| `1` | No | `0` = subscribed (emails delivered); `1` = unsubscribed. |
| `is_deleted` | `0` \| `1` | No | `0` = keep; `1` = delete from database (also purges custom data). |
| *(any other key)* | any | No | [Custom field](#custom-fields) — auto-created on first use. |

### Single — request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"email": "subscriber@domain.tld", "first_name": "Mari", "city": "Tartu"}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

### Response

```json
{ "code": 101, "message": "OK" }
```

### Batch (create / update)

Send a **JSON array** of subscriber objects to the same endpoint:

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '[{"email": "subscriber+1@domain.tld", "first_name": "Mari"},
       {"email": "subscriber+2@domain.tld", "first_name": "Jaan"}]' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

### Batch response

```json
{ "code": 101, "message": "OK" }
```

> **Gotcha — the single `101`**
> A batch returns **one aggregated** `{"code":101,"message":"OK"}` for the whole
> array. There is **no per-contact status**. You cannot tell from the response
> which rows succeeded or whether any individual record was rejected — a malformed
> contact inside the batch is not surfaced item-by-item. Validate emails
> client-side before sending. See the [bulk-sync guide](../guides/bulk-sync.md).

> **Gotcha — client timeout ≠ failure**
> Large batches (hundreds of contacts × many custom fields) process **slowly
> server-side** — well over 25 seconds, sometimes minutes. If your client times
> out before receiving the `101`, the contacts may have **already been saved**.
> Treat a client-side timeout as *unknown*, not *failed*: use a generous timeout
> (minutes for bulk), and because the operation is idempotent on `email`, simply
> re-send the batch rather than assuming nothing landed. Details:
> [bulk-sync guide](../guides/bulk-sync.md).

### Common codes

`101` OK · `204` invalid email · `207` missing required field (`email`). Full
table in [Errors](../errors.md).

---

## Get a subscriber

```
GET /api/contact.php?email={email}
```

Returns one subscriber's full record, including engagement stats and all custom
fields.

### Parameters

| Parameter | Required | Description |
|---|---|---|
| `email` | Yes | The subscriber's email (URL-encode the `@` → `%40`). |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php?email=recipient%40domain.tld"
```

### Response

A subscriber object. Core fields:

| Field | Description |
|---|---|
| `email` | Email address. |
| `is_unsubscribed` | `0` subscribed / `1` unsubscribed. |
| `last_response_code` | Delivery code of the last email — see [delivery codes](../errors.md#delivery--smtp-response-codes-last_response_code). |
| `last_response_at` | Timestamp of the last delivery response. |
| `created_at` | When the subscriber was added. |
| `subscribed_at` | When subscription became active. |
| `modified_at` | Last data update. |
| `last_open_at` / `total_opens` | Latest open timestamp / lifetime open count. |
| `last_click_at` / `total_clicks` | Latest click timestamp / lifetime click count. |
| `unsubscribed_at` | Unsubscription timestamp (if any). |
| *(custom fields)* | Any custom data on the contact. |

> **Note**
> Timestamps here are `YYYY-MM-DD HH:MM:SS` in **Europe/Tallinn** — see
> [Conventions → Dates](../conventions.md#dates-timezone--encoding).

If the email isn't found, you get code `206`.

---

## List subscribers of a segment

```
GET /api/contact.php?list={segment_id}
```

Returns all subscribers belonging to a segment, paginated, sorted alphabetically
by email.

### Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `list` | Yes | — | Segment ID (from [List segments](segments.md#list-segments)). |
| `offset` | No | `0` | **Page index** (not record offset). `0` → records 0–25,000; `1` → 25,001–50,000; etc. |
| `limit` | No | `25000` | Subscribers per request. **Capped at 25,000.** |
| `fields` | No | all | Restrict returned fields. `email` is always included. |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php?list=3"
```

### Response

A JSON array of subscriber objects (same field set as [Get a subscriber](#get-a-subscriber)).

> **Gotcha**
> `offset` is a **page index of 25,000-record pages**, not a row offset. To walk a
> large segment, increment `offset` by 1 each call until you get fewer than
> `limit` records back. See [pagination](../conventions.md#pagination).

---

## Opt-in subscriber(s)

```
POST /api/autoresponder.php
```

Use this — rather than plain [create/update](#create--update) — when you want to
**opt a contact in and fire a "form submitted" automation workflow** (e.g. a
double-opt-in confirmation or welcome series). It upserts the contact's data
first, then enrolls them in the targeted workflow.

This is the same endpoint as [Trigger automation](automations.md) — see that page
for full parameter detail.

### Key parameters

| Parameter | Required | Description |
|---|---|---|
| `autoresponder` | Yes | Workflow ID. Must be a workflow with a **"form submitted"** trigger. |
| `addresses` | Yes | Array of address objects: `{email, is_unsubscribed?, is_deleted?, ...customFields}`. |
| `from` / `from_name` | No | *Deprecated* From overrides. |

### Request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"addresses": [{"email": "subscriber+1@domain.tld"}], "autoresponder": 1}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/autoresponder.php"
```

### Response

```json
{ "code": 101, "message": "OK", "addresses": ["subscriber+1@domain.tld"] }
```

---

## Forget subscriber(s)

```
POST /api/contact/forget.php
```

**Permanently** removes subscribers and their statistics (GDPR-style erasure).
"Forgetting permanently affects subscribers and their statistics." This is
irreversible.

### Body

A JSON **array of email strings**.

### Request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '["recipient+1@domain.tld", "recipient+2@domain.tld"]' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact/forget.php"
```

### Response

```json
{ "code": 101, "message": "OK" }
```

> **Note**
> For large erasures, call in **batches** to avoid request timeouts — the docs
> recommend **up to 10,000 subscribers per batch**.

> **Forget vs. delete vs. unsubscribe**
> - `is_unsubscribed: 1` (on `contact.php`) — keeps the contact, stops mailing them.
> - `is_deleted: 1` (on `contact.php`) — deletes the contact and its custom data.
> - `forget.php` — permanently erases the contact **and its statistics** (GDPR).

---

## Subscriber action log

```
GET /api/history.php
```

The pull-based engagement event stream (opens, clicks, bounces, opt-outs, …) for
your account. This is how you ingest activity — **there are no webhooks.** Full
documentation, parameters, and incremental-sync pattern: see the dedicated
[Action log](action-log.md) page.

---

## Custom fields

> **Strength: custom fields are effectively unlimited.**
> You can attach an effectively unbounded number of custom fields to each contact
> — there is no practical per-contact field cap. This makes Smaily a good fit for
> rich, per-recipient personalization (e.g. dozens of recommendation slots, scores,
> and metadata on one contact).

**There is no "create field" call.** Any key you send that isn't a reserved field
(`email`, `is_unsubscribed`, `is_deleted`) is treated as a custom field and
**auto-created on first use**:

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"email": "subscriber@domain.tld",
       "rec_1_name": "Premium Cat Food 2kg",
       "rec_1_price": "24.90",
       "rec_segment": "loyal"}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

After this call, `rec_1_name`, `rec_1_price`, and `rec_segment` exist as custom
fields for every contact (empty for those not set), usable in template
personalization and [segment rules](segments.md).

### Tips

- **Clear a field** by writing `null` (or empty) to it on the next upsert — it
  overwrites the previous value.
- **Naming** is up to you; pick a stable convention (e.g. `rec_N_*`) since fields
  persist account-wide once created.
- Fields auto-created via the API show up in the Smaily UI like any other field.

---

*Source: based on <https://smaily.com/help/api/subscribers-2/create-and-update-subscribers/>, <https://smaily.com/help/api/subscribers-2/get-a-subscriber/>, <https://smaily.com/help/api/subscribers-2/opt-in-subscribers/>, <https://smaily.com/help/api/subscribers-2/forget-subscribers/>, and <https://smaily.com/help/api/segments/list-subscribers-of-a-segment/>. Batch single-`101`, client-timeout, slow-batch, and unlimited-custom-field behavior verified by the product owner.*
