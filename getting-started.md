# Getting started

This is the fastest path from "I have credentials" to "I created a subscriber."

## 1. Get API credentials

In the Smaily web app:

1. Click your account name (upper-right corner) → **Preferences**.
2. Open the **Integrations** tab.
3. Under **API Passwords**, click **Create a new user**.

You get three values:

- **Subdomain** — goes in the base URL.
- **API username**
- **API password** — shown **only once**. Store it in a secret manager now.

You can create multiple API users and give each a description (e.g. "order-sync
plugin"). See [Authentication](authentication.md) for details.

## 2. Make your first authenticated request

The simplest read is "list my segments" — it takes no parameters and confirms
auth works:

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/list.php"
```

A working response looks like:

```json
[
  { "id": 4, "name": "Women", "subscribers_count": 250 },
  { "id": 5, "name": "Men 40+", "subscribers_count": 48 }
]
```

If you get an HTTP 401, the username/password is wrong. If you get a `429`, you're
over the [rate limit](conventions.md#rate-limits).

## 3. Create a subscriber

`POST` to `contact.php` with at least an `email`. Any extra keys become
[custom fields](reference/subscribers.md#custom-fields), auto-created on first use:

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '{"email": "subscriber@domain.tld", "first_name": "Mari"}' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

Response:

```json
{ "code": 101, "message": "OK" }
```

`code: 101` means success. See the full [response-code table](errors.md).

> **Gotcha**
> `contact.php` returns a single `{"code":101,"message":"OK"}` even when you send
> a **batch** of many contacts — there is no per-contact result. Read the
> [batch section](reference/subscribers.md#batch-create-update) and the
> [bulk-sync guide](guides/bulk-sync.md) before syncing at scale.

## 4. Read it back

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php?email=subscriber%40domain.tld"
```

(Note the URL-encoded `@` → `%40`.)

## Next steps

- [Conventions](conventions.md) — formats, rate limits, dates.
- [Subscribers reference](reference/subscribers.md) — the full contact API.
- [Bulk sync guide](guides/bulk-sync.md) — for syncing many contacts reliably.

---

*Source: based on <https://smaily.com/help/api/general/common-principles/> and <https://smaily.com/help/api/general/create-api-user/>.*
