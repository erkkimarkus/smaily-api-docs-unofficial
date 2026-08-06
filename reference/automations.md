# Automations

Automation workflows (autoresponders) are multi-step, trigger-based sequences
built in the Smaily UI. The API lets you **list** them and **enroll contacts** to
fire a workflow.

| Operation | Method | Endpoint |
|---|---|---|
| [List automation workflows](#list-automation-workflows) | `GET` | `/api/autoresponder.php` |
| [Trigger a workflow](#trigger-a-workflow) | `POST` | `/api/autoresponder.php` |
| [List workflows (undocumented)](#list-workflows-undocumented) | `GET` | `/api/workflows.php` |

> **Note**
> The same `autoresponder.php` script handles both — `GET` lists workflows,
> `POST` enrolls contacts. The `POST` form is also how you
> [opt subscribers in](subscribers.md#opt-in-subscribers).

> **Two views of the same thing**
> `autoresponder.php` (documented) returns the workflow's *content* — sections,
> subjects, templates. [`workflows.php`](#list-workflows-undocumented)
> (undocumented, but it exists) returns the workflow's *identity* — id, title,
> trigger type, enabled flag. Neither is a superset of the other.

---

## List automation workflows

```
GET /api/autoresponder.php
```

Returns the organization's automation workflows.

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `limit` | — | Records per request. `0` = no limit (returns all). |
| `page` | `0` | Page number; applies only when `limit` is set. |
| `status` | — | Filter by `ACTIVE` or `INACTIVE`. Supports multiple values. |
| `sort_by` | `created_at` | Sort field (only `created_at` supported). |
| `sort_order` | `ASC` | `ASC` or `DESC`. |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/autoresponder.php"
```

### Response

A JSON array of workflow objects:

| Field | Description |
|---|---|
| `id` | Numeric workflow identifier. |
| `name` | Workflow name. |
| `status` | `ACTIVE` or `INACTIVE`. |
| `created_at` | `YYYY-MM-DD HH:MM:SS`, Europe/Tallinn. |
| `activated_at` | Activation timestamp (when `ACTIVE`). |
| `tags` | List of workflow tags. |
| `sections` | Array of "Send Message" actions in the workflow. |

Each `sections` entry contains `id`, `name` (the message subject), and `template`
(`{id, name, preview_url}`).

```json
[
  {
    "id": 1,
    "name": "Welcome series",
    "status": "ACTIVE",
    "created_at": "2026-01-10 09:00:00",
    "activated_at": "2026-01-10 09:05:00",
    "tags": ["onboarding"],
    "sections": [
      {
        "id": 11,
        "name": "Welcome aboard!",
        "template": { "id": 2, "name": "Welcome", "preview_url": "https://${SUBDOMAIN}.sendsmaily.net/template/preview/id/2/" }
      }
    ]
  }
]
```

> **Tip**
> Use this to discover the `id` you need for [Trigger a workflow](#trigger-a-workflow)
> and the `autoresponder_id` for [Send message](messages.md).

> **Verified (2026-08-06, live accounts)** — `?id=N` is **silently ignored**.
> `GET /api/autoresponder.php?id=123` returns the **full list**, exactly as if
> no `id` had been passed — not a single workflow, and not an error. Unlike
> [`campaign.php?id=N`](campaigns.md#campaign-statistics), there is no by-id
> lookup here. Filter client-side, and never assume a one-element response:
> code that does `response[0]` gets an arbitrary workflow, not the one you
> asked for.

---

## Trigger a workflow

```
POST /api/autoresponder.php
```

Enrolls one or more contacts into a workflow, **upserting their data first** so
the workflow's templates can use it. The targeted workflow must use a
**"form submitted"** trigger.

### Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `autoresponder` | int | Yes | Workflow ID (must have a "form submitted" trigger). |
| `addresses` | array | Yes | Array of address objects (see below). |
| `force_opt_in` | bool | No | Defaults to `true`. When `true`, enrolling **opts the contact in** (sets `is_unsubscribed=0`) — so a previously unsubscribed contact is re-subscribed and receives the email. Set `false` to fire the workflow **without changing** the contact's subscription/consent status: an unsubscribed contact is not re-subscribed and is not mailed. Use `false` for consent-based (opt-in-only) sending. |
| `from` | string | No | *Deprecated* — override default From address. |
| `from_name` | string | No | *Deprecated* — override default From name. |

**Address object:**

| Field | Required | Description |
|---|---|---|
| `email` | Yes | Subscriber's email. |
| `is_unsubscribed` | No | `0` / `1`. |
| `is_deleted` | No | `0` / `1`. |
| *(any other key)* | No | Custom field, auto-created. Available in the workflow's templates. |

### Request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"addresses": [{"email": "subscriber+1@domain.tld", "location": "Tallinn"}], "autoresponder": 1}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/autoresponder.php"
```

### Response

```json
{ "code": 101, "message": "OK", "addresses": ["subscriber+1@domain.tld"] }
```

The `addresses` array echoes the enrolled emails.

> **Trigger vs. Send message**
> - **Trigger a workflow** (here) runs the *whole* workflow, including its filters
>   and delays — use it to enroll someone into a sequence.
> - [**Send message**](messages.md) does a *direct* transactional send of one
>   workflow's message, skipping filters and delays — use it for one-off,
>   per-recipient transactional email.

### Common codes

`101` OK · `204` invalid email · `221` invalid autoresponder ID · `207` missing
required field. See [Errors](../errors.md).

> **Verified (2026-07)** — three behaviors worth knowing before you build:
> - **`221` also means "wrong trigger type"**: enrolling into an ACTIVE
>   workflow whose trigger is *not* "form submitted" (e.g. an opt-in-triggered
>   welcome series) returns `221 invalid autoresponder ID` even though the ID
>   plainly exists in the [list response](#list-automation-workflows).
> - **You cannot pre-validate the trigger type**: the list response does not
>   include the workflow's trigger, so the only reliable check is a test
>   enroll against a test address.
> - **Enrolling into an INACTIVE workflow returns `101` OK but sends
>   nothing**: the contact data is upserted, no email goes out, and no error
>   is returned. Validate `status == "ACTIVE"` from the list response
>   yourself before trusting a `101`.

---

## List workflows (undocumented)

```
GET /api/workflows.php
```

> **Verified (2026-08-06, live accounts)** — this endpoint is **absent from the
> official docs**, but it exists and answers with the account's workflows. It is
> the only discovered way to resolve a workflow send back to a name.

Returns a JSON array of workflow objects:

| Field | Description |
|---|---|
| `id` | Numeric workflow identifier. |
| `trigger_type` | The workflow's trigger — **not** exposed by [`autoresponder.php`](#list-automation-workflows). |
| `title` | Workflow name. |
| `is_enabled` | Whether the workflow is turned on. |

Shape: `[{id, trigger_type, title, is_enabled}]`.

### Why it matters: resolving a `campaign_id`

> **Verified (2026-08-06, live accounts)**
> Workflow ids come from the **same numeric sequence as campaign ids**, and they
> are the values that workflow sends carry as `campaign_id` in
> [action-log](action-log.md) rows.

So an unknown `campaign_id` from `history.php` is resolved in two steps:

1. `GET /api/campaign.php?id=N` — regular campaigns only. Unknown ids answer
   HTTP 200 with `{"code":216,...}`.
2. `GET /api/workflows.php` — match `id == N` for workflow sends.

Some ids resolve in **neither** — A/B split-test campaigns are exposed by no
discovered endpoint. See
[Campaigns → what the list does not contain](campaigns.md#what-the-list-does-not-contain).

> **Note**
> `trigger_type` is the workflow's trigger as the API reports it. Whether its
> values map cleanly onto the "form submitted" requirement of
> [Trigger a workflow](#trigger-a-workflow) has **not** been verified — a test
> enroll is still the only reliable check.

---

*Source: based on <https://smaily.com/help/api/automations-2/list-automation-workflows/> and <https://smaily.com/help/api/automations-2/autoresponder/>. `workflows.php` and the `?id=` behavior of `autoresponder.php` are not in the official docs — verified against live accounts, 2026-08-06.*
