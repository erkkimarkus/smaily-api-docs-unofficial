# Guide: Gotchas

The quirks worth knowing **before** you build against the Smaily API. Each one has
bitten someone. Cross-references point to the full reference.

---

## Pull-only — no webhooks

> **Gotcha**
> Smaily does **not** offer webhooks. There is no way to have engagement events
> pushed to you.

You ingest events by **polling** the [action log](../reference/action-log.md)
(`history.php`) and advancing a `since_seq_id` cursor. Architect a scheduled
poller, not an event listener.

**Implication:** combined with the ~30-day retention (below), your poller is the
*only* path for events to reach you — if it's down too long, the data is gone.

---

## Action-log retention is ~30 days

> **Gotcha**
> The action log keeps only the **last 30 days** of events.

Persist events to your own store as you poll. If your poller is offline for more
than 30 days, you permanently lose that window — there's no replay. See
[Action log → retention](../reference/action-log.md).

---

## Single `101` for batch writes {#single-101-batch-response}

> **Gotcha**
> A batch `POST /api/contact.php` returns **one** aggregated
> `{"code":101,"message":"OK"}` for the entire array — **not** a per-contact
> result.

You can't tell which rows succeeded or whether one was rejected. Validate emails
client-side, and rely on idempotent re-sync rather than parsing per-item status.
See [Subscribers → batch](../reference/subscribers.md#batch-create-update) and the
[bulk-sync guide](bulk-sync.md).

---

## Slow large batches {#slow-large-batches}

> **Gotcha**
> Large/wide batches (hundreds of contacts × many custom fields) take **well over
> 25 seconds** — sometimes minutes — to process server-side.

This is normal, not a hang, and is unrelated to the rate limit. Use **generous
client timeouts** (minutes) and smaller batches (~100) for wide rows. See
[bulk-sync → batch sizing](bulk-sync.md#1-batch-sizing).

---

## Client timeout is not a failure {#client-timeout-is-not-a-failure}

> **Gotcha**
> If your client times out before the `101` arrives, the contacts may have
> **already been saved.** A client-side timeout means *unknown*, not *failed*.

Because `contact.php` is idempotent on `email`, safely **re-send the same batch**
rather than assuming nothing landed. See
[bulk-sync → §5](bulk-sync.md#5-client-timeout--definite-failure).

---

## The public docs understate the rate limit

> **Gotcha**
> The [official docs](https://smaily.com/help/api/general/common-principles/) say
> **5 requests/second per IP**. The real, enforced limit is **10 requests/second
> per IP** (verified by the product owner).

Build to 10/s. Keep headroom if integrations share an egress IP. Over-limit
returns HTTP `429`. See [Conventions → rate limits](../conventions.md#rate-limits).

---

## `offset` is a page index, not a row offset

> **Gotcha**
> On list endpoints, `offset`/`page` is a **0-based page index**, not a number of
> records to skip. `offset=1` skips one *whole page* (10,000 or 25,000 records),
> not one record.

Page sizes differ per endpoint (25,000 for segment subscribers, 10,000 for
campaign stats/list, 1,000 for templates). See
[Conventions → pagination](../conventions.md#pagination).

---

## JSON body with a `text/html` content type

> **Gotcha**
> Responses are always JSON, but the HTTP `Content-Type` header is `text/html`.

Parse the body as JSON regardless of the declared type. Don't branch on
`Content-Type`. See [Conventions → response format](../conventions.md#response-format).

---

## Errors come back with HTTP 200

> **Gotcha**
> Most validation/business errors return HTTP **200** with an error **code in the
> JSON body** (e.g. `{"code":204,"message":"..."}`), not a `4xx` HTTP status.

Check the body `code`, not just the HTTP status. Only `401` (auth) and `429`
(rate limit) are real HTTP-level errors. See [Errors](../errors.md).

---

## Reserved-TLD emails are rejected with `203`, not `204` {#reserved-tld-emails}

> **Gotcha**
> `contact.php` rejects a *syntactically valid* address whose TLD is an
> RFC-6761/2606 **reserved** TLD — `…@example.test`, `…@host.invalid`,
> `…@x.localhost`, `…@y.example` — with `{"code":203,"message":...}` ("invalid
> data"), **not** the email-syntax code `204`. Smaily validates the TLD against
> deliverable domains, so the address parses fine yet is still refused.

Probe-confirmed 2026-07-01 on the sandbox account: the **same** contact body returns
`203` with `…@example.test` and `101` with `…@example.com` / `…@mailinator.com`,
**across every body encoding** (JSON object, JSON array, and form-encoded) — i.e. the
`203` is purely the domain, independent of how you serialise the request.

**For testing:** use a real-but-non-delivering domain such as **`@example.com`** (the
RFC-2606 documentation domain — registered, resolves, accepts no mail) for synthetic
live contacts. A `.test`/`.example`/`.invalid` address makes a correct write *look
like* a malformed-payload (`203`) bug when it's only the TLD. (The single aggregated
`101` for batch writes — see [the batch gotcha](#single-101-batch-response) — means a
rejected row inside a batch is **not** surfaced per-item either, so validate the
domain client-side.)

---

## Two different "code" systems

> **Gotcha**
> `code` in a response body (`101` = OK) is **not** the same as
> `last_response_code` on a subscriber (`200` = delivered). One is request status;
> the other is last-email-delivery status.

A subscriber's `last_response_code: 200` means *delivery succeeded* — confusingly,
the *request* success code is `101`. See
[Errors → delivery codes](../errors.md#delivery--smtp-response-codes-last_response_code).

---

## Mixed timestamp formats and timezone

> **Gotcha**
> `history.php` returns `YYYY-MM-DD HH:MM:SS` in **Europe/Tallinn**, while
> `templates.php` and the message action log return **RFC 3339** with an offset.
> Action-log *query* bounds (`start_at`/`end_at`) are **UNIX UTC**.

Don't assume one format or UTC across the API. See
[Conventions → dates](../conventions.md#dates-timezone--encoding).

---

## "List" means "segment"

> **Gotcha**
> The `list` parameter and `list.php` endpoint refer to **segments**, not mailing
> lists in the classic sense.

The `id` from [List segments](../reference/segments.md#list-segments) is what you
pass as `list` to `contact.php?list=` and to campaign launch.

---

## `contact.php` doesn't trigger automations

> **Gotcha**
> Creating/updating a contact via `contact.php` does **not** fire automation
> workflows.

To enroll a contact into a workflow (welcome series, double opt-in), use
[`autoresponder.php`](../reference/automations.md#trigger-a-workflow) /
[opt-in](../reference/subscribers.md#opt-in-subscribers).

---

## Workflow enroll: `221` can mean "wrong trigger type", `101` can mean "nothing sent" {#enroll-pitfalls}

> **Gotcha**
> `POST autoresponder.php` only works on workflows with a **"form submitted"**
> trigger — but the failure modes are misleading (verified 2026-07):
> - Enrolling into an **ACTIVE workflow with a different trigger** (e.g. an
>   opt-in-triggered welcome series) fails with `221 invalid autoresponder ID`,
>   even though the ID clearly exists in the list response.
> - Enrolling into an **INACTIVE workflow** returns `101` OK, upserts the
>   contact — and silently sends **nothing**.

The list response exposes neither the trigger type nor enough to predict the
first case, so the only reliable validation is a **test enroll against a test
address** before going live. Also validate `status == "ACTIVE"` yourself — a
`101` alone does not mean an email went out. See
[Automations → Trigger a workflow](../reference/automations.md#trigger-a-workflow).

---

## Enroll is not idempotent

> **Gotcha**
> Calling `POST autoresponder.php` twice for the same contact enrolls them
> **twice** — two send events, two identical emails, even within the same
> second.

Deduplication is entirely **your** responsibility: persist your own
"already enrolled" record *before* making the call, and only retry when you
are certain the first request never reached Smaily. (Contrast with
`contact.php`, whose upserts *are* idempotent on `email`.)

---

## Send message: a multi-section workflow fires ALL its messages {#send-message-multi-section}

> **Gotcha**
> `POST message/send.php` takes a **workflow** ID (a section/template ID fails
> with `221`) — and if that workflow has several sections, **every section's
> message is sent immediately**: one call against a 7-section workflow produced
> 7 messages to a single recipient (verified 2026-07).

For transactional sending, build a dedicated **single-section** workflow.
Related notes: there is **no per-send subject override** — a `subject` param
is accepted but silently ignored (subject always comes from the template);
batching works via the `to` array (context is shared across the batch, not
per-recipient); attachments are supported (base64 or URL). See
[Messages → Send message](../reference/messages.md#send-message).

---

## Campaign launch has a 5-minute grace period

> **Gotcha**
> Launching a campaign without a `due` date does **not** send immediately —
> there's a built-in **5-minute grace period**.

Set `due` explicitly for precise timing. Also note: `html` must be a **publicly
reachable URL** or launch fails with `209`. See
[Campaigns → launch](../reference/campaigns.md#launch-a-campaign).

---

## Merge tags don't re-resolve inside custom-field values {#nested-merge-tags}

> **Gotcha**
> Smaily templates use `{{...}}` merge tags — both custom fields
> (`{{rec_1_name}}`) and **system variables**. The system variable for the sending
> campaign's ID is **`{{sys.campaign.id}}`**, which resolves to the real campaign
> ID at send time (verified by the product owner). But a merge tag placed **inside
> a custom-field value** is rendered in a **single pass**: Smaily substitutes the
> field, then does **not** re-scan the inserted value for further tags.

So if you bake a tag into a custom field — e.g. put `{{sys.campaign.id}}` inside a
recommendation link stored in `rec_N_link_path` — it lands **literally** in the
output (URL-encoded as `%7B%7B...%7D%7D`), polluting the recipient's URL and their
Google Analytics. Put system variables **directly in the template body** (top
level), never inside a custom-field value. Custom fields should carry only
already-resolved data. See
[Custom fields](../reference/subscribers.md#custom-fields).

---

## Things that are *not* gotchas — platform strengths

- **Unlimited custom fields per contact**, auto-created on first use — great for
  rich personalization. See [Custom fields](../reference/subscribers.md#custom-fields).
- **Effectively unlimited sending volume** — no per-message quota to engineer
  around; the constraint is request rate (10/s), not send count.
- **Idempotent upserts on `email`** — makes retries and re-syncs safe by design.

---

*Source: synthesized from the official Smaily API docs at <https://smaily.com/help/api/>, with rate-limit, batch-response, timeout, and platform-characteristic corrections verified by the product owner.*
