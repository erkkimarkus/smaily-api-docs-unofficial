# Automations

Automation workflows (autoresponders) are multi-step, trigger-based sequences
built in the Smaily UI. The API lets you **list** them and **enroll contacts** to
fire a workflow.

| Operation | Method | Endpoint |
|---|---|---|
| [List automation workflows](#list-automation-workflows) | `GET` | `/api/autoresponder.php` |
| [Trigger a workflow](#trigger-a-workflow) | `POST` | `/api/autoresponder.php` |

> **Note**
> The same `autoresponder.php` script handles both — `GET` lists workflows,
> `POST` enrolls contacts. The `POST` form is also how you
> [opt subscribers in](subscribers.md#opt-in-subscribers).

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
>   plainly exists in the [list response](#list-workflows).
> - **You cannot pre-validate the trigger type**: the list response does not
>   include the workflow's trigger, so the only reliable check is a test
>   enroll against a test address.
> - **Enrolling into an INACTIVE workflow returns `101` OK but sends
>   nothing**: the contact data is upserted, no email goes out, and no error
>   is returned. Validate `status == "ACTIVE"` from the list response
>   yourself before trusting a `101`.

---

*Source: based on <https://smaily.com/help/api/automations-2/list-automation-workflows/> and <https://smaily.com/help/api/automations-2/autoresponder/>.*
