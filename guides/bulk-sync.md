# Guide: Bulk contact sync

A recipe for reliably syncing large numbers of contacts (and their custom fields)
into Smaily — the kind of job a backfill or nightly sync does. It folds together
several quirks documented elsewhere: the [single-`101` batch response](../reference/subscribers.md#batch-create-update),
the [10/s rate limit](../conventions.md#rate-limits), [slow large
batches](gotchas.md#slow-large-batches), and the
[client-timeout caveat](gotchas.md#client-timeout-is-not-a-failure).

## The endpoint

All bulk contact writes go through one upsert:

```
POST /api/contact.php
```

with a **JSON array** of contact objects. It's idempotent on `email` — re-sending
the same contact updates it rather than duplicating. That idempotency is the
foundation of safe retries below.

```bash
curl -X POST -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d '[{"email":"a@x.tld","first_name":"A","rec_1_sku":"SKU-1"},
       {"email":"b@x.tld","first_name":"B","rec_1_sku":"SKU-9"}]' \
  "https://${SUBDOMAIN}.sendsmaily.net/api/contact.php"
```

Response — one aggregated status for the **whole** array:

```json
{ "code": 101, "message": "OK" }
```

---

## 1. Batch sizing

There's no hard published cap on contacts per `contact.php` call, but each batch
is processed **synchronously and slowly** server-side, so size for *latency*, not
for a row limit.

| Payload shape | Suggested batch size |
|---|---|
| Few fields per contact (email + a handful) | 500–1,000 contacts |
| Many custom fields per contact (e.g. 60+ recommendation fields) | **100 contacts** |

> **Why smaller for wide rows?** A batch of hundreds of contacts each carrying
> many custom fields takes **well over 25 seconds** to process — sometimes
> minutes. Smaller batches keep each request's latency bounded and make retries
> cheap. (Our own production sync uses batches of 100.)

For the separate **forget** operation, the docs allow up to **10,000 emails per
batch** — that endpoint is lighter. See
[Forget subscribers](../reference/subscribers.md#forget-subscribers).

---

## 2. Respect the rate limit (10/s)

The real limit is **10 requests/second per IP** (the public docs' 5/s is wrong —
see [Conventions](../conventions.md#rate-limits)). For a bulk job:

- Throttle **proactively** to stay under 10/s rather than relying on `429`
  recovery. With ~100-contact batches you're sending far fewer than 10 req/s
  anyway, because each request is slow.
- If you *do* see `429`, back off exponentially (0.5s → 1s → 2s → 4s, with jitter)
  and retry.
- If multiple integrations share one egress IP, the 10/s is **shared** — budget
  accordingly.

---

## 3. Use generous timeouts

> **Set client timeouts in the minutes range for bulk writes.**

Because large/wide batches routinely exceed 25 seconds server-side, a tight client
timeout (the common 30s default) will fire *before* Smaily responds — even though
the write is succeeding. Set the HTTP client timeout to **several minutes** for
bulk `contact.php` calls.

---

## 4. Handle the single-`101` response

The response is **one** `{"code":101,"message":"OK"}` for the entire array —
there is **no per-contact result**. Consequences:

- You **cannot** learn which individual rows were accepted or rejected from the
  response. A bad record inside the batch is not reported item-by-item.
- **Validate client-side first.** Drop or fix obviously invalid emails before
  sending, since the API won't tell you which one was bad.
- Treat `101` as "the batch was accepted as a whole," and rely on idempotent
  re-sync (below) to converge, not on parsing per-row status.

---

## 5. Client timeout ≠ definite failure

> **Critical caveat.** If your client times out before receiving the `101`, the
> contacts may **already have been saved.** A client-side timeout means *unknown*,
> not *failed*.

Do **not** react to a timeout by assuming nothing landed and, say, switching to a
different code path. Instead:

1. Log the batch as *uncertain*.
2. **Re-send the same batch.** Because `contact.php` is idempotent on `email`,
   re-sending is safe — already-saved contacts are simply updated to the same
   values, and any that didn't land the first time now do.

This "retry the idempotent batch" strategy turns the ambiguous-timeout problem
into a non-issue.

---

## 6. Idempotency by email

Lean on it throughout:

- The same `email` always maps to the same contact — re-syncing never duplicates.
- Safe to **replay** a batch after a timeout, a `429`, or a crash mid-job.
- Make your job **resumable**: track which batches are confirmed `101` and which
  are uncertain, and only replay the uncertain ones on the next run.

---

## Reference pseudo-loop

```text
for batch in chunk(contacts, size = 100):          # §1 batch sizing
    throttle_to_stay_under(10 req/s)               # §2 rate limit
    response = POST contact.php
        body    = json(batch)                       # JSON array
        timeout = minutes(3)                        # §3 generous timeout
    if response.code == 101:
        mark batch confirmed                        # §4 single-101 = whole batch OK
    elif response.status == 429:
        backoff_and_retry(batch)                    # §2
    elif client_timed_out:
        mark batch uncertain                        # §5 timeout != failure
        re-send batch (idempotent on email)         # §6 safe replay
    else:
        inspect code → see errors.md
```

---

## Related

- [Subscribers → Create/update (batch)](../reference/subscribers.md#batch-create-update)
- [Custom fields](../reference/subscribers.md#custom-fields)
- [Gotchas](gotchas.md)
- [Errors & response codes](../errors.md)

---

*Source: based on <https://smaily.com/help/api/subscribers-2/create-and-update-subscribers/>. Batch single-`101`, slow-batch latency, generous-timeout, and client-timeout-≠-failure behavior verified by the product owner.*
