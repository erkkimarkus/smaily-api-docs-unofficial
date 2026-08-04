# Authentication

Every Smaily API request uses **HTTP Basic authentication** with an API username
and password. There are no API tokens, OAuth flows, or signed requests.

## Credentials

You authenticate as an **API user**, which is separate from your login account.
Each API user has:

- a **username**
- a **password** (shown only once at creation)

The account **subdomain** is not a credential, but you need it to build the
[base URL](README.md#base-url): `https://{subdomain}.sendsmaily.net/api/...`.

## Creating an API user

In the Smaily web app:

1. Click your account name (upper-right) → **Preferences**.
2. Open the **Integrations** tab.
3. Under **API Passwords**, click **Create a new user**.

> **Note**
> The password is displayed **only at creation time** — "Save the password to a
> secure location, because this will be the only time the password is shown." If
> you lose it, create a new API user.

You can:

- Create **multiple API users** per account — useful for giving each integration
  (plugin, sync job, internal tool) its own credentials.
- Add an optional **description** per user to track where it's used and who owns it.
- **Delete** a user via the trash icon. Deletion immediately breaks any active
  API connection using that user — rotate before you revoke.

## Sending the credentials

Pass them with HTTP Basic auth. With `curl`, use `-u`:

```bash
curl -X GET -u "${USERNAME}:${PASSWORD}" \
  "https://${SUBDOMAIN}.sendsmaily.net/api/list.php"
```

Equivalently, set the header yourself:

```
Authorization: Basic base64("username:password")
```

## Requirements & failure modes

- **HTTPS is mandatory.** Plain HTTP requests are redirected and fail.
- **Wrong credentials** → HTTP `401 Unauthorized` (transport-level; not one of
  the JSON `1xx`/`2xx` body codes in [errors.md](errors.md)).
- **Plan-blocked (freemium) account** → HTTP `403` with body
  `{"code":227,"message":"A paid package is required."}` on every endpoint,
  and the check runs **before** authentication — identical response whether the
  credentials are right, wrong, or absent, so credentials cannot be verified
  against such an account. Verified live 2026-08-04; details in
  [Errors → HTTP 403 — plan block](errors.md#http-403--plan-block-code-227).
- **Nonexistent subdomain** → HTTP `404` with an empty body (no JSON).
- A correctly authenticated request that fails validation still returns HTTP 200
  with a JSON error code in the body — see [Errors](errors.md).

## Operational advice

- **One API user per integration.** It limits blast radius and makes revocation
  surgical — kill one plugin's user without breaking the others.
- **Never embed credentials in client-side code.** All calls are server-to-server.
- **Rotate by creating-then-revoking**, since the old user breaks the instant
  it's deleted.

---

*Source: based on <https://smaily.com/help/api/general/create-api-user/> and <https://smaily.com/help/api/general/common-principles/>.*
