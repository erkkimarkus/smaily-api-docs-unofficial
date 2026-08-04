# Errors & response codes

Smaily uses two distinct code systems. Don't confuse them:

1. **API response codes** — returned in the JSON body of every write. A code of
   `101` means success; everything else is an error. (HTTP status is usually 200
   even for these body-level errors.)
2. **Delivery / SMTP response codes** — the `last_response_code` value on a
   subscriber, describing the outcome of the *last email* sent to them. These are
   data, not request errors.

Plus the transport-level **HTTP `429`** for rate limiting and **`401`** for bad auth.

---

## API response codes (JSON body)

A successful write returns:

```json
{ "code": 101, "message": "OK" }
```

| Code | Meaning | How to handle |
|---|---|---|
| **101** | OK — request was successful. | Success. Note: for batch writes this is a single aggregated `101`, not per-item — see [the batch gotcha](guides/gotchas.md#single-101-batch-response). |
| **201** | Data must be posted with `POST` method. | You used `GET` for a write. Switch to `POST`. |
| **203** | Invalid data submitted (parameter validation error). | Inspect your payload; e.g. malformed email in a `send` recipient list. Do not retry unchanged. |
| **204** | Invalid email address provided (syntax error). | Fix/validate the email before retrying. |
| **206** | Could not find requested email address. | The subscriber doesn't exist. Expected on lookups for unknown contacts. |
| **207** | Following fields are required (missing required fields). | Add the missing required parameters. The message lists which. |
| **208** | Could not find list with ID (segment doesn't exist). | Verify the segment/list `id`. |
| **209** | Could not load content from remote URL. | The `html` URL for a campaign wasn't publicly reachable. Check it returns 200. |
| **210** | Invalid due date provided (empty or wrong format). | Use `YYYY-MM-DD HH:MM:SS` (Europe/Tallinn). |
| **211** | Domain of From address is unverified. | Verify the sending domain in Smaily before launching. |
| **212** | Launch failed with error (campaign / A/B test error). | A general launch failure; check campaign config. |
| **213** | Invalid win date provided (A/B test). | Fix the A/B-test win date. |
| **214** | Invalid winning condition provided. | Use a supported A/B winning condition. |
| **215** | Invalid campaign ID provided. | Check the campaign `id`. |
| **216** | Could not find campaign matching provided ID. | The campaign doesn't exist. |
| **217** | Unknown column in filter data. | A segment `filter_data` field name is wrong/unknown. |
| **218** | Unknown operator for field (unsupported segmentation). | Use a [supported operator](reference/segments.md#filter-operators). |
| **219** | Could not find list with ID (segment doesn't exist). | Same class as 208 — verify the `id`. |
| **220** | Could not find statistics entry by email. | No stats for that recipient/campaign pair. |
| **221** | Invalid autoresponder ID provided. | Check the automation workflow `id`. |
| **223** | Missing start or end date (subscriber action log). | Provide `start_at`/`end_at` (or use `since_seq_id`). |
| **224** | End date cannot be before start. | Swap/fix the date range. |
| **225** | Database insert failed (internal server error). | Server-side. Safe to retry with backoff. |
| **226** | Failed to download file from URL. | A referenced file/asset URL was unreachable. |
| **227** | A paid package is required. | The account's plan blocks API access entirely — see [HTTP 403 — plan block](#http-403--plan-block-code-227). Needs a plan change, not a request change. |

> **Note**
> Codes `204` and `206` are subscriber-scoped; `203`/`207` are generic
> validation. When in doubt, log the full `message` — it's human-readable and
> often names the exact missing/invalid field.

### Handling pattern

- `101` → success.
- `201`, `203`, `204`, `207`, `210`, `213`, `214`, `217`, `218` → **client
  errors**. Fix the request; retrying unchanged won't help.
- `206`, `208`, `215`, `216`, `219`, `220`, `221` → **not found**. Often expected
  (e.g. looking up a contact that doesn't exist yet).
- `209`, `225`, `226` → **transient / server-side**. Retry with backoff.
- `211`, `227` → **account/config**. Needs a human (verify domain / upgrade plan).

---

## HTTP 429 — Too Many Requests

Returned when you exceed the rate limit.

> **Verified**
> The real limit is **10 requests/second per IP** — not the 5/s stated in the
> public docs. See [Conventions → Rate limits](conventions.md#rate-limits).

**Handle it:** back off and retry. A simple exponential backoff (e.g. 0.5s, 1s,
2s, 4s with jitter) is sufficient. For bulk jobs, throttle proactively to stay
under 10/s rather than relying on `429` recovery — see the
[bulk-sync guide](guides/bulk-sync.md).

---

## HTTP 401 — Unauthorized

Bad or missing Basic-auth credentials, or the API user was deleted. Re-check the
username/password and that the API user still exists. See
[Authentication](authentication.md).

> **Caveat**
> You will never see a `401` from a plan-blocked (freemium) account — the
> package check runs *before* authentication. See the next section.

---

## HTTP 403 — plan block (code 227)

> **Verified** (2026-08-04, live against a freemium account)

When an account's plan does not include API access (freemium), **every** API
endpoint answers:

```
HTTP/1.1 403 Forbidden
Content-Type: text/html   ← despite the JSON body

{"code":227,"message":"A paid package is required."}
```

Observed behavior, all confirmed on the same account:

- **The package check runs BEFORE authentication.** The response is identical
  with correct credentials, a wrong password, a wrong username, and no
  `Authorization` header at all. Consequence: **you cannot verify credentials
  against a plan-blocked account** — a connection test can only report "the
  plan blocks access; credentials could not be checked".
- **Code 227 is a positive signal.** Nothing else produces it, so
  `403` + body code `227` can safely be classified as *plan block* (an
  account/billing problem), distinct from bad credentials (`401`) and from
  outages (`5xx`). Retrying will not help; a human has to change the plan.
- Endpoints confirmed: `autoresponder.php` (list), `contact.php` (read and
  `list=1`), `history.php` — i.e. reads are blocked too, not only sends.
- **No `WWW-Authenticate` and no `Retry-After` header** on these responses.
- Note the HTTP status here is a real `403` — unlike validation errors, which
  arrive as body codes on HTTP 200.

Related edge: a **subdomain that does not exist** answers `404` with an empty
body (no JSON at all) — distinguish it from the JSON-bodied errors above.

---

## Delivery / SMTP response codes (`last_response_code`)

These describe the **last delivery attempt** to a subscriber (visible via
[Get a subscriber](reference/subscribers.md#get-a-subscriber) and in bounce
action-log entries). They are not request errors.

| Code | Meaning |
|---|---|
| `0` | No emails have been sent to the subscriber yet. |
| `200` | Delivery successful. |
| `4xx` / `5xx` | Standard SMTP status codes (transient `4xx`, permanent `5xx`). |
| `1001` | Subscriber unsubscribed in the latest campaign. |
| `1002` | Subscriber reported the latest campaign as spam — automatically unsubscribed. |

### Delivery classification strings

Bounce/delivery events also carry a classification string:

| Classification | Meaning |
|---|---|
| `ok` | Successfully delivered. |
| `bad-content` | Receiving server flagged the message as SPAM. |
| `bad-relay` | Recipient server temporary error, misconfiguration, or maintenance. |
| `blocked` | Delivery denied by Smaily (or another service). |
| `client-error` | Smaily misconfiguration/error denying delivery. |
| `expired` | Delivery exceeded the max time (2 days). |
| `mailbox-disabled` | Recipient mailbox disabled or inactive. |
| `mailbox-full` | Recipient mailbox full. |
| `mailbox-unknown` | Recipient mailbox does not exist. |
| `rate-limit` | Hourly/daily sending limits exceeded. |
| `signature-error` | Misconfigured SPF or invalid DKIM signature. |
| `temp-error` | Temporary delivery error. |
| `unknown` | Status unknown. |

> **Note**
> `mailbox-unknown`, `mailbox-disabled`, and `5xx` are effectively permanent —
> stop mailing those addresses. `bad-relay`, `temp-error`, `rate-limit`, and
> `4xx` are transient and may succeed on a later send.

---

*Source: based on <https://smaily.com/help/api/general/response-codes/> and <https://smaily.com/help/api/general/delivery-response-codes/>, with the rate-limit correction verified by the product owner.*
