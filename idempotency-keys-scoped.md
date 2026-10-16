# Idempotency keys, scoped

## What a key protects

A client sends `Idempotency-Key: <key>` with a request. The first request with that key runs and its response is stored. A retry with the same key gets the stored response; the handler does not run again. That is the whole feature, and it protects exactly one thing: a client that did not see our answer and sends the same request again.

## What it does not protect

It does not make a write to a third party safe. If the handler calls a payment processor and the processor's answer is lost, the key will happily store whatever the handler returned, including an exception. Writes to third parties belong in a durable operation with their own request id (see [third-party writes have three outcomes](third-party-writes-have-three-outcomes.md)); the idempotency key sits in front of that, not instead of it.

## The rules

- **Claim with one statement.** `INSERT ... ON CONFLICT DO NOTHING`. Under concurrency exactly one caller wins the key. The others read the row and either replay it or get "in progress" (409) if the winner is still running. There is no check-then-act window.
- **Hash the request.** The same key with a different payload is a client bug, not a retry. Refuse it (422) rather than replay a response to a request the client did not make. Hash canonically, with sorted keys, so field order does not count as a difference.
- **Scope the key.** Per API client at least. One client's key must not replay or collide with another's.
- **A handler that throws frees the key.** The client should be able to retry. The stored thing is a response, not an exception.
- **Expire.** A day is a common TTL. Lazily reaping expired rows on the next claim is enough.

## The bug a reviewer caught

A partner endpoint for joining a waitlist took an app-scoped idempotency key, and the docs said so. The shared ledger behind it was keyed by location. The same key used at a second location added a second party instead of answering "key reused". The fix was to serialise keyed joins on the app's key, refuse a key still live elsewhere, add the composite index the cross-location lookup needed, and write the test that fails without it.

The lesson is less about idempotency than about documentation: the documented scope and the implemented scope have to be the same thing, and the test is what holds them together.

## When not to bother

Reads. Writes that are naturally idempotent (PUT of a full resource). Internal calls between services you control, where a durable operation table is the better tool. Keys are for the public edge, where you cannot see the client's retry logic.

A runnable version is in [durable-integrations](https://github.com/maydan90/durable-integrations) (`src/idempotency.ts`).
