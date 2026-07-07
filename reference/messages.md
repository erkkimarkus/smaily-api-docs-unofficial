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
| `context` | object | No | Key-value variables for template personalization. |
| `attachments` | array | No | Attachment objects: `{ "content": base64, "filename": "..." }`. |

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
| `message_ids` | One ID per recipient — use to query the [message action log](#message-action-log). |

> **Gotcha**
> Attachment `content` must be **base64-encoded** (the example `dGVzdA==` decodes
> to `test`). A `203` ("Invalid data submitted") usually means a malformed
> recipient email or bad payload.

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
| `offset` | No | `0` | Page index. **Cannot be combined with `since_seq_id`.** |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/message/action/log.php?since_seq_id=1"
```

### Response

A JSON array of action objects:

| Field | Description |
|---|---|
| `seq_id` | Sequence number — use as the cursor for incremental sync. |
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
