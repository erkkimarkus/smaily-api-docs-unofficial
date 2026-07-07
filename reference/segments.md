# Segments

Segments (called "lists" in the API) are saved, rule-based audiences. The
`list.php` endpoint lists and creates/updates them; subscriber listing lives on
`contact.php`.

| Operation | Method | Endpoint |
|---|---|---|
| [List segments](#list-segments) | `GET` | `/api/list.php` |
| [Create / update a segment](#create--update-a-segment) | `POST` | `/api/list.php` |
| [List subscribers of a segment](#list-subscribers-of-a-segment) | `GET` | `/api/contact.php?list=...` |

> **Note**
> "List" and "segment" are the same thing in this API. The numeric `id` you get
> here is what you pass as `list` when [launching a campaign](campaigns.md#launch-a-campaign)
> or [listing a segment's subscribers](subscribers.md#list-subscribers-of-a-segment).

---

## List segments

```
GET /api/list.php
```

Returns all segments, alphabetically sorted by `name`. No parameters required.

### Request

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/list.php"
```

### Response

```json
[
  { "id": 4, "name": "Women", "subscribers_count": 250 },
  { "id": 5, "name": "Men 40+", "subscribers_count": 48 }
]
```

| Field | Description |
|---|---|
| `id` | Segment identifier. |
| `name` | Segment name. |
| `subscribers_count` | Number of subscribers currently in the segment. |

---

## Create / update a segment

```
POST /api/list.php
```

Creates a segment, or updates one if you pass an existing `id`.

### Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | int | No | Segment ID. Include to **update** an existing segment; omit to **create**. |
| `name` | string | Yes | Segment name. |
| `filter_type` | `ALL` \| `ANY` | Yes | Combine rules with AND (`ALL`) or OR (`ANY`). |
| `filter_data` | array | Yes | Array of rules — see [rule structure](#rule-structure). |

### Rule structure

Each rule is `[field, [operator, value]]`:

```json
["email", ["EndsWith", "@domain.tld"]]
```

`filter_data` is an array of these rules. `field` can be a built-in field
(`email`, `gender`, …) or any [custom field](subscribers.md#custom-fields).

### Filter operators

| Operator | Meaning |
|---|---|
| `Equal` | Exact match. |
| `NotEqual` | Not equal. |
| `BeginsWith` | Starts with value. |
| `Contains` | Contains value. |
| `DoesNotContain` | Does not contain value. |
| `EndsWith` | Ends with value. |
| `LessThan` | `<` value. |
| `LessThanEqual` | `≤` value. |
| `GreaterThanEqual` | `≥` value. |
| `GreaterThan` | `>` value. |

> **Gotcha**
> Using a field that doesn't exist returns code `217` ("Unknown column in filter
> data"); using an unsupported operator returns `218`. Create the custom field
> (by writing it to at least one contact) before segmenting on it.

### Request (create)

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Women", "filter_type": "ALL", "filter_data": [["gender", ["Equal", "women"]]]}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/list.php"
```

### Response

```json
{ "code": 101, "message": "OK", "id": 1 }
```

| Field | Description |
|---|---|
| `code` | Status code (`101` = OK). |
| `message` | Human-readable status. |
| `id` | The created/updated segment's ID. |

### Update example

Pass the existing `id` to modify a segment in place:

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"id": 1, "name": "Women (EU)", "filter_type": "ALL",
       "filter_data": [["gender", ["Equal", "women"]], ["country", ["Equal", "EE"]]]}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/list.php"
```

---

## List subscribers of a segment

```
GET /api/contact.php?list={segment_id}
```

This lives on the subscribers endpoint. Full parameters (`offset`, `limit`,
`fields`), pagination notes, and response shape are documented under
[Subscribers → List subscribers of a segment](subscribers.md#list-subscribers-of-a-segment).

Quick version:

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php?list=3"
```

Returns a JSON array of subscriber objects, alphabetically by email, paginated in
25,000-record pages via `offset`.

---

*Source: based on <https://smaily.com/help/api/segments/list-segments/>, <https://smaily.com/help/api/segments/create-or-update-a-segment/>, <https://smaily.com/help/api/segments/list-subscribers-of-a-segment/>, and <https://smaily.com/help/api/segments/list-segment-rules/>.*
