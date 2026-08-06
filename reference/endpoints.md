# Endpoint index

Every Smaily endpoint is a distinct `.php` script under
`https://{subdomain}.sendsmaily.net/api/` — there is no `/v1/` prefix and no
resource nesting (see [Conventions](../conventions.md)). This page is the map:
what exists, and — just as usefully — what does **not**, so you don't spend an
afternoon guessing script names.

---

## Endpoints that exist

| Script | Method | What it does |
|---|---|---|
| `contact.php` | `GET` / `POST` | [Get](subscribers.md#get-a-subscriber) / [create-update](subscribers.md#create--update) subscribers (single + batch), [list a segment's subscribers](subscribers.md#list-subscribers-of-a-segment). |
| `contact/forget.php` | `POST` | [Forget subscriber(s)](subscribers.md#forget-subscribers) (GDPR erase). |
| `list.php` | `GET` / `POST` | [List](segments.md#list-segments) / [create-update](segments.md#create--update-a-segment) segments. "List" means *segment*. |
| `campaign.php` | `GET` / `POST` | [List campaigns](campaigns.md#list-campaigns), [statistics](campaigns.md#campaign-statistics) (`?id=N`), [launch](campaigns.md#launch-a-campaign). **Regular campaigns only** — see [what the list does not contain](campaigns.md#what-the-list-does-not-contain). |
| `unsubscribe.php` | `POST` | [Unsubscribe a recipient](campaigns.md#unsubscribe-a-recipient) with campaign attribution. |
| `autoresponder.php` | `GET` / `POST` | [List automation workflows](automations.md#list-automation-workflows) (with their sections/templates) / [enroll contacts](automations.md#trigger-a-workflow), [opt-in](subscribers.md#opt-in-subscribers). |
| `workflows.php` | `GET` | [List workflows](automations.md#list-workflows-undocumented) — id, title, trigger type, enabled flag. **Undocumented officially**; verified 2026-08-06. |
| `message/send.php` | `POST` | [Send message](messages.md#send-message) — transactional send of a workflow's message. |
| `message/action/log.php` | `GET` | [Message action log](messages.md#message-action-log), keyed by `message_id`. |
| `history.php` | `GET` | [Action log](action-log.md) — the pull-based engagement event stream. |
| `templates.php` | `GET` | Templates (per the official docs; not covered in detail here). |

---

## Endpoints that do **not** exist

> **Verified (2026-08-06, live accounts)** — every name below answers
> **HTTP 404**. They are plausible guesses, not endpoints:
>
> `automation.php` · `automations.php` · `workflow.php` (singular) ·
> `trigger.php` · `flow.php` · `journey.php` · `stats.php` ·
> `campaign_stats.php` · `autoresponder_stats.php` · `newsletter.php` ·
> `sms.php` · `campaign_send.php`

Note the near-misses: automations live under **`autoresponder.php`** and
**`workflows.php`** (plural), statistics are a **parameter** on
`campaign.php?id=N` rather than a separate script, and there is no dedicated
send endpoint beyond `message/send.php` and a campaign launch.

> **Tip**
> A `404` here is bare — no JSON body. That distinguishes "wrong script name"
> from a real API error, which arrives as a JSON `code` on
> [HTTP 200](../errors.md), and from a plan-blocked account, which answers
> [`403` + code `227`](../errors.md#http-403--plan-block-code-227) on *every*
> script.

---

*Compiled from the official Smaily API documentation at <https://smaily.com/help/api/>, plus endpoint probing verified against live accounts, 2026-08-06.*
