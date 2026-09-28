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
| `since_seq_id` | one of these | — | Return only actions **after** this sequence number (the cursor). Requires `limit`. |
| `start_at` / `end_at` | one of these | — | Time window as **UNIX timestamps in UTC**. Mutually exclusive with `since_seq_id`. |
| `offset` | No | `0` | Page **number**, not a row offset: page `0` is actions 1–10 000, page `1` is 10 001–20 000, **whatever `limit` says**. **Incompatible with `since_seq_id`** (use it only with date windows). |
| `limit` | No (**yes** with `since_seq_id`) | — | Max results per request. **Capped at 10,000.** |
| `actions` | No | all | Filter to specific action types, **comma-separated** (`actions=click,view`; see below). |

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
| `seq_id` | Monotonic sequence number. **Only returned when querying with `since_seq_id`** — a `start_at`/`end_at` answer carries none (see [Date windows](#date-windows)). **This is your cursor.** |
| `email` | The subscriber the action belongs to. |
| `time` | `YYYY-MM-DD HH:MM:SS` in **Europe/Tallinn** local time. |
| `campaign_id` | The campaign that produced the action. Regular campaigns, workflow sends and A/B campaigns **share one id sequence** — see [Resolving `campaign_id`](#resolving-campaign_id). |
| `campaign_name` | Its name, already resolved in the row. For A/B campaigns one id carries **two** names — see [Detecting an A/B send](#detecting-an-ab-send). |
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

## Resolving `campaign_id`

Every row carries both `campaign_id` and `campaign_name`, so the *name* is
already in the log — you only need a lookup for the campaign's configuration or
statistics. That lookup is not always possible:

> **Verified (2026-08-06, live accounts)**
> `campaign_id` values come from **one numeric sequence shared by regular
> campaigns, workflow sends and A/B split-test campaigns** — but no single
> endpoint lists all three:
>
> | Row produced by | Resolve with |
> |---|---|
> | Regular campaign | [`GET campaign.php?id=N`](campaigns.md#campaign-statistics) |
> | Workflow / autoresponder send | [`GET workflows.php`](automations.md#list-workflows-undocumented), match on `id` |
> | A/B split-test campaign | **nothing** — no discovered endpoint exposes them |

`campaign.php?id=N` answers HTTP 200 with `{"code":216,...}` for anything it
doesn't own, so `216` means "not a regular campaign", not "unknown id". See
[Errors → code 216](../errors.md#code-216--not-a-regular-campaign).

### Detecting an A/B send

> **Verified (2026-08-06, live accounts)**
> For an A/B split-test campaign the **same** `campaign_id` appears with **two
> different `campaign_name` values**, split roughly 50/50 across its rows.

Since A/B campaigns can't be looked up anywhere, this is the usable detector:
group rows by `campaign_id` and count distinct `campaign_name` — more than one
name means an A/B send, and the names are the variants. A regular campaign or a
workflow send yields exactly one name per id.

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
  "https://${SUBDOMAIN}.sendsmaily.net/api/history.php?since_seq_id=0&limit=10000&actions=click,view"

# Next page — feed the highest seq_id you saw back in
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/history.php?since_seq_id=90412&limit=10000&actions=click,view"
```

> **Why `since_seq_id` and not dates?**
> The sequence cursor is gap-free and resumable; date windows can double-count or
> miss events around boundaries and don't survive clock skew. Use date windows
> only for one-off backfills within the 30-day retention.

## Date windows

`start_at` + `end_at` and `since_seq_id` are **two different queries, never one**:
a request states one or the other, and the answer differs between them.

> **Verified (2026-09-28, live account)**
> `history.php?start_at=…&end_at=…&actions=modify&limit=3` answered three rows
> and **none of them carried `seq_id`**. The comma-separated `actions` filter was
> honoured.

| | `since_seq_id` | `start_at` + `end_at` |
|---|---|---|
| Rows carry `seq_id` | yes | **no** |
| Ordered by | `seq_id` | `time` |
| `limit` | required | optional |
| `offset` | not allowed | page number (10 000 rows a page) |

What follows from that:

- **A window gives you no cursor.** Nothing in a window's answer can be fed to
  `since_seq_id`, so a poller that starts from a date has to go on reading
  windows, or start its cursor some other way (`since_seq_id=0` walks the whole
  30 days from the oldest).
- **Next window from the last one's end.** Rows are timed to the second in
  **Europe/Tallinn local time**, while `start_at`/`end_at` are **UTC** Unix
  seconds, and several rows can share one second.
- **`end_at` inclusivity is not documented.** Choose a boundary rule that cannot
  skip a row (for example, the next window starts at the previous `end_at + 1`
  with the previous `end_at` in the past), and say which rule you rely on.
- **Paging with `offset` is by 10 000-row pages.** A smaller `limit` with
  `offset=1` skips everything between `limit` and row 10 001, so page a window
  with `limit=10000` or not at all.

The per-message log behaves differently: there `seq_id` is returned on every row
(see [Messages → Message action log](messages.md#message-action-log)).

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
