# Messages

Transactional messaging: send an individual message to one or more recipients
using an existing automation workflow's template, and read the per-message action
log.

| Operation | Method | Endpoint |
|---|---|---|
| [Send message](#send-message) | `POST` | `/api/message/send.php` |
| [Message action log](#message-action-log) | `GET` | `/api/message/action/log.php` |

> **Note**
> Sending volume on Smaily is effectively **unlimited** — there's no per-message
> quota to design around. The constraint to respect is the request
> [rate limit](../conventions.md#rate-limits) (10/s), not a send cap.

---

## Send message

```
POST /api/message/send.php
```

Sends a **direct transactional message** to one or more recipients, rendering an
automation workflow's template with per-call context. Unlike
[triggering a workflow](automations.md#trigger-a-workflow), this **skips the
workflow's filters and delays** — it sends immediately.

### Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `autoresponder_id` | int | Yes | Workflow ID whose template/message is sent. |
| `to` | array | Yes | List of recipient email addresses. |
| `from` | object | No | Sender: `{ "email": "...", "name": "..." }`. |
| `reply_to` | object | No | Reply-To address object. |
| `context` | object | No | Key-value variables for template personalization. **Shared across the whole `to` batch** — there is no per-recipient context; for individual personalization, make one call per recipient. |
| `attachments` | array | No | Attachment objects: either `{ "content": <base64>, "filename": "..." }` **or** `{ "url": "https://...", "filename": "..." }` — one of `content`/`url` per attachment, not both. Max request body 64 MB. |

### Request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{
        "autoresponder_id": 1,
        "from": {"email": "offers@domain.tld", "name": "Offers"},
        "to": ["recipient+1@domain.tld"],
        "context": {"name": "John Doe"},
        "attachments": [{"content": "dGVzdA==", "filename": "File 1.txt"}]
      }' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/message/send.php"
```

### Response

```json
{ "code": 101, "message": "OK", "message_ids": [140737488355328] }
```

| Field | Description |
|---|---|
| `code` | Status code (`101` = OK). |
| `message` | Human-readable status. |
| `message_ids` | One ID **per message sent** — with a multi-section workflow this is sections × recipients, not one per recipient (see gotcha below). Use to query the [message action log](#message-action-log). |

> **Gotcha**
> Attachment `content` must be **base64-encoded** (the example `dGVzdA==` decodes
> to `test`). A `203` ("Invalid data submitted") usually means a malformed
> recipient email or bad payload.

> **Gotcha — no subject override, and unknown params fail silently**
> (verified 2026-07): there is **no way to override the subject line** per
> send — the subject always comes from the workflow template. A `subject`
> parameter in the payload is **accepted (`101` OK) but silently ignored** —
> as, presumably, is any unknown parameter. Do not take `101` as evidence
> that a parameter worked; verify the received email.
>
> **The working pattern for dynamic subjects** (verified end-to-end 2026-07):
> put a merge tag in the workflow template's **subject line** (e.g.
> `{{subject}}`) and pass the value via `context` — merge tags resolve in the
> subject just like in the body (UTF-8 incl. emoji arrives intact). A
> dedicated single-section "transactional carrier" workflow with
> `{{subject}}` as its subject gives you a fully dynamic subject per send.

> **Gotcha — a `message_id` is not proof of sending** (verified 2026-07):
> a send against a template with a **malformed merge tag** in the subject
> line (`{{subject]]` — broken closing braces) returned `101` OK **with a
> `message_id`**, but the email was never sent and the
> [message action log](#message-action-log) for that ID stays empty forever.
> After sending, verify a `send` action exists in the log — that, not the
> `101` or the `message_id`, is the delivery evidence.

> **Gotcha — a multi-section workflow sends ALL its messages at once**
> (verified 2026-07): `autoresponder_id` must be the **workflow** ID — passing a
> section/template ID fails with `221`. But one call against a 7-section
> workflow returned **7 `message_ids` for a single recipient**: every section's
> message was sent immediately. For transactional sending, point this endpoint
> at a **single-section workflow** built for the purpose.

> **Gotcha — context values are NOT HTML-escaped**
> (live-verified 2026-07-23, smailydemo sandbox; probe script
> `bin/walk-pro1537-escape-probe.cjs` in the `smaily-wordpress-plugin` repo):
> merge-tag substitution inserts `context` values verbatim into the rendered
> HTML. Callers MUST escape user-controlled values themselves (e.g.
> `htmlspecialchars`) before sending, or risk content injection into the
> email — Smaily adds no escaping of its own. Do not double-escape: a
> pre-escaped value passed as `&lt;i&gt;...&lt;/i&gt;` displayed correctly as
> literal text, confirming a single decode-on-render, not two.

> **Gotcha — the subject line substitutes merge tags too**
> (same live verification, 2026-07-23): the `{{subject}}`-in-template pattern
> above (dynamic subject via `context.subject`) substitutes exactly like the
> body — values pass through raw, with no stripping or escaping. Keep
> user-controlled content out of a merge-tagged subject unless it has been
> escaped/sanitized first.

### Common codes

`101` OK · `203` validation error (e.g. invalid recipient email) · `221` invalid
autoresponder ID. See [Errors](../errors.md).

---

## Message action log

```
GET /api/message/action/log.php
```

Returns engagement actions for **transactional messages** sent via
[Send message](#send-message) — sends, views, clicks, bounces, opt-outs,
complaints. Retention is the **last 30 days**.

> **Note**
> This is the transactional counterpart to the account-wide
> [subscriber action log](action-log.md) (`history.php`). Same pull model, same
> 30-day window, but scoped to messages and keyed by `message_id`.

### Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| One of `since_seq_id`, `message_id`, or `start_at`/`end_at` | Yes | — | Selects the window. |
| `limit` | No | `10000` | Max records (capped at 10,000). |
| `offset` | No | `0` | Page **number** (page `0` is records 1–10 000, page `1` is 10 001–20 000). **Cannot be combined with `since_seq_id`.** |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/message/action/log.php?since_seq_id=1"
```

### Response

A JSON array of action objects:

| Field | Description |
|---|---|
| `seq_id` | Sequence number — use as the cursor for incremental sync. Returned on **every** row, a date-window answer included — unlike the [subscriber action log](action-log.md#date-windows). Rows are ordered by `seq_id`, or by date under `start_at`/`end_at`. |
| `message_id` | The message this action belongs to. |
| `email` | Recipient. |
| `campaign_id` | Associated campaign/workflow, if any. |
| `section_id` | Workflow section (message step). |
| `action` | `send`, `view`, `click`, `bounce`, `optout`, `complaint`, … |
| `value` | Contextual data (e.g. clicked URL, bounce code). |
| `created_at` | RFC 3339 timestamp, e.g. `2026-01-10T14:47:42+03:00`. |

> **Incremental sync**
> Pass `since_seq_id` = the highest `seq_id` you've already processed to fetch only
> new actions. See the cursor pattern in [Action log](action-log.md#incremental-polling-with-since_seq_id).

---

*Source: based on <https://smaily.com/help/api/messages/send-message/> and <https://smaily.com/help/api/messages/message-action-log/>. Unlimited-sending characteristic verified by the product owner.*
