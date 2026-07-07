# Action log

The action log is how you get **engagement events** out of Smaily: opens, clicks,
bounces, opt-outs, complaints, and more. It is the single most important endpoint
for keeping an external system in sync with email activity.

```
GET /api/history.php
```

> **Event model: PULL, not push.**
> Smaily has **no webhooks.** You ingest events by **polling** this endpoint on a
> schedule and advancing a cursor. Design your integration as a poller, not a
> listener. See [Gotchas → Pull model](../guides/gotchas.md#pull-only-no-webhooks).

> **Retention: ~30 days.**
> "Subscriber Action Log contains only actions from the last 30 days." Anything
> older is gone — persist events to your own store as you poll. If your poller is
> down longer than 30 days, you permanently lose the gap.

---

## Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `since_seq_id` | one of these | — | Return only actions **after** this sequence number (the cursor). |
| `start_at` / `end_at` | one of these | — | Time window as **UNIX timestamps in UTC**. Mutually exclusive with `since_seq_id`. |
| `offset` | No | `0` | Page index. **Incompatible with `since_seq_id`** (use it only with date windows). |
| `limit` | No | `10000` | Max results per request. **Capped at 10,000.** |
| `actions` | No | all | Filter to specific action types (see below). |

### Action types

`bounce`, `click`, `complaint`, `create`, `delete`, `modify`, `optin`, `optout`,
`send`, `view`.

> **Tip**
> For engagement sync you usually want `send`, `view`, `click`, `bounce`,
> `optout`, `complaint` and can ignore lifecycle noise (`create`, `delete`,
> `modify`, `optin`). Filter with `actions` to cut volume.

---

## Response

A JSON array of action objects:

| Field | Description |
|---|---|
| `seq_id` | Monotonic sequence number (present when querying with `since_seq_id`). **This is your cursor.** |
| `email` | The subscriber the action belongs to. |
| `time` | `YYYY-MM-DD HH:MM:SS` in **Europe/Tallinn** local time. |
| `campaign_id` / `campaign_name` | The campaign that produced the action. |
| `action` | One of the action types above. |
| `value` | Context-dependent: SMTP code for `bounce`, clicked **URL** for `click`, device/OS/IP for `view`, etc. |

Example shape:

```json
[
  {
    "seq_id": 90412,
    "email": "recipient@domain.tld",
    "time": "2026-06-21 09:14:02",
    "campaign_id": 5,
    "campaign_name": "Offers of the week",
    "action": "click",
    "value": "https://shop.example/products/cat-food?utm_content=abc123"
  }
]
```

> **Note**
> For `click` actions, `value` is the destination URL — including any UTM
> parameters you embedded. This is how you attribute clicks back to specific
> content (e.g. parse a `utm_content` recommendation ID out of the URL).

---

## Incremental polling with `since_seq_id`

The reliable, resumable pattern: track the highest `seq_id` you've processed and
ask only for newer events.

1. Start with `since_seq_id=0` (or your last saved cursor).
2. Fetch up to `limit=10000` actions.
3. Process them; record the **max** `seq_id` seen.
4. If you got a full page (10,000), loop immediately with the new cursor — there's
   likely more.
5. Persist the cursor durably between runs.

```bash
# First page
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/history.php?since_seq_id=0&limit=10000&actions[]=click&actions[]=view"

# Next page — feed the highest seq_id you saw back in
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/history.php?since_seq_id=90412&limit=10000&actions[]=click&actions[]=view"
```

> **Why `since_seq_id` and not dates?**
> The sequence cursor is gap-free and resumable; date windows can double-count or
> miss events around boundaries and don't survive clock skew. Use date windows
> only for one-off backfills within the 30-day retention.

### Recommended polling cadence

- A **frequent incremental poll** (e.g. every few hours) to keep current.
- A **daily catch-up poll** with a wider window as a safety net.
- Stay under the [10 req/s rate limit](../conventions.md#rate-limits); the action
  log returns 10k/page, so even busy accounts need only a handful of requests per
  cycle.

---

## Per-message variant

For **transactional** messages sent via [Send message](messages.md), there's a
parallel log at `/api/message/action/log.php` keyed by `message_id` — same pull
model and 30-day window. See [Messages → Message action log](messages.md#message-action-log).

---

## Common codes

`223` missing start/end date (when not using `since_seq_id`) · `224` end date
before start. See [Errors](../errors.md).

---

*Source: based on <https://smaily.com/help/api/subscribers-2/subscriber-action-log/>. Pull-only event model (no webhooks) verified by the product owner.*
