# Campaigns

Bulk email sends to one or more segments. The `campaign.php` script lists
campaigns and their statistics (`GET`) and launches new ones (`POST`);
unsubscribes have their own endpoint.

| Operation | Method | Endpoint |
|---|---|---|
| [List campaigns](#list-campaigns) | `GET` | `/api/campaign.php` |
| [Launch a campaign](#launch-a-campaign) | `POST` | `/api/campaign.php` |
| [Campaign statistics](#campaign-statistics) | `GET` | `/api/campaign.php?id=...` |
| [Unsubscribe a recipient](#unsubscribe-a-recipient) | `POST` | `/api/unsubscribe.php` |

---

## List campaigns

```
GET /api/campaign.php
```

Returns campaigns (without `id`, this is the *list*; with `?id=N`, it's
[statistics](#campaign-statistics)).

### Parameters

| Parameter | Default | Description |
|---|---|---|
| `status` | — | Filter by `DRAFT`, `PENDING`, `COMPLETED`, `CANCELLED`. Multiple via `status[]=`. |
| `tags` | — | Filter by tags. Multiple via `tags[]=`. |
| `sort_by` | `created_at` | Only `created_at` supported. |
| `sort_order` | `ASC` | `ASC` or `DESC`. |
| `limit` | — | Records per request. `0` = all. |
| `page` | `0` | Page index (10,000 records per page). |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/campaign.php?status[]=COMPLETED&sort_order=DESC"
```

### Response

A JSON array of campaign objects:

| Field | Description |
|---|---|
| `id` | Campaign identifier. |
| `name` | Campaign name / subject. |
| `template` | `{id, name, preview_url}`. Returns `"DELETED"` if the template was removed. |
| `tags` | List of tags. |
| `created_at` | `YYYY-MM-DD HH:MM:SS`, Europe/Tallinn. |
| `completed_at` | `null` unless `status` is `COMPLETED`. |
| `status` | `DRAFT`, `PENDING`, `COMPLETED`, or `CANCELLED`. |

---

## Launch a campaign

```
POST /api/campaign.php
```

Creates and (unless saved as draft) launches a campaign to one or more segments.

### Required parameters

| Parameter | Description |
|---|---|
| `subject` | Campaign name **and** message subject. |
| `from` | From email address (its domain must be verified — else code `211`). |
| `list` | Segment ID, or array of segment IDs, to send to. |

**Content source** — provide exactly one:

| Parameter | Description |
|---|---|
| `template_id` | ID of an existing template. |
| `html` | URL to HTML content (**must be publicly accessible** — else code `209`). |
| `html_raw` | Raw HTML content inline. |

### Optional parameters

| Parameter | Description |
|---|---|
| `from_name` | From name. |
| `reply_to` | Reply-To email address. |
| `due` | Schedule time, `YYYY-MM-DD HH:MM:SS` (Europe/Tallinn). If omitted, sends immediately with a **5-minute grace period**. |
| `tags` | List of tags. |
| `save_as_draft` | `1` creates the campaign without launching. Default `0`. |

### Request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{
        "subject": "Offers of the week",
        "from": "offers@domain.tld",
        "html": "https://domain.tld/path/to/content.html",
        "list": [10, 11],
        "tags": ["offers"]
      }' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/campaign.php"
```

### Response

```json
{ "code": 101, "message": "OK", "id": 1 }
```

The `id` is the new campaign — keep it to pull [statistics](#campaign-statistics).

> **Gotcha**
> Omitting `due` does **not** send instantly — there's a built-in **5-minute grace
> period** before delivery starts. Set `due` explicitly if you need precise timing.

> **Gotcha**
> `html` must be a **publicly reachable URL** at launch time (Smaily fetches it).
> A URL behind auth, on localhost, or returning non-200 fails with code `209`. Use
> `html_raw` or `template_id` if you can't host public HTML.

### Common codes

`101` OK · `209` couldn't load `html` URL · `210` invalid `due` date · `211`
unverified From domain · `212` launch failed · `208`/`219` segment not found. See
[Errors](../errors.md).

---

## Campaign statistics

```
GET /api/campaign.php?id={campaign_id}
```

Returns aggregate (and optionally per-recipient) stats for one campaign.

### Parameters

| Parameter | Required | Default | Description |
|---|---|---|---|
| `id` | Yes | — | Campaign ID. |
| `detailed` | No | `0` | `1` includes per-recipient open/click data in `addresses`. |
| `offset` | No | `0` | Page index (for `detailed`). |
| `limit` | No | `10000` | Records per request (capped at 10,000). |

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/campaign.php?id=1&detailed=1"
```

### Response (selected fields)

| Group | Fields |
|---|---|
| Metadata | `id`, `name`, `status`, `created_at`, `completed_at` |
| Sender | `from`, `reply_to`, template details |
| Delivery | `total_count`, `delivered_count`, `bounce_count` |
| Engagement | `opened_count`, `opened_percent`, `click_count`, `unique_click_count`, `click_percent` |
| Image opens | `view_count`, `unique_view_count`, `view_percent` |
| Actions | `unsubscribe_count`, `complaint_count`, `forward_count` |
| Detailed (`detailed=1`) | `addresses[]` — per-recipient opens, clicks, link-level tracking |

> **Note**
> "Opens" here are **image/pixel opens** (`view_*`), distinct from `opened_*`.
> Apple Mail Privacy Protection inflates these — weight clicks higher for real
> engagement.

### Common codes

`215` invalid campaign ID · `216` campaign not found · `220` no statistics entry
for the given recipient.

---

## Unsubscribe a recipient

```
POST /api/unsubscribe.php
```

Unsubscribes a single recipient, attributed to a specific campaign.

### Parameters

| Parameter | Required | Description |
|---|---|---|
| `email` | Yes | Recipient's email address. |
| `campaign_id` | Yes | The campaign ID (returned by [launch](#launch-a-campaign)). |

### Request

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"email": "recipient@domain.tld", "campaign_id": 5}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/unsubscribe.php"
```

### Response

```json
{ "code": 101, "message": "OK" }
```

> **Note**
> To stop mailing someone **without** campaign attribution, set
> `is_unsubscribed: 1` via [create/update](subscribers.md#create--update) instead.

---

*Source: based on <https://smaily.com/help/api/campaigns-3/list-campaigns/>, <https://smaily.com/help/api/campaigns-3/launch-a-campaign/>, <https://smaily.com/help/api/campaigns-3/api-campaign-statistics/>, and <https://smaily.com/help/api/campaigns-3/unsubscribe-a-recipient/>.*
