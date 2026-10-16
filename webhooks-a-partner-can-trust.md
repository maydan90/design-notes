# Webhooks a partner can trust

## What a partner needs

A partner receiving webhooks needs five things: to know the event came from us, to never see the same event fan out twice, to get it eventually if their receiver was down, to recover after they fix a bug on their side, and to be left alone when their endpoint is dead. Each one is a design decision, and most webhook implementations I have seen get two or three.

## Signatures

Header: `webhook-signature: t=<unix seconds>,v1=<hex hmac-sha256(secret, t + "." + body)>`.

The timestamp is inside the signed message. If it were only a header, a captured body could be replayed later with a fresh timestamp. Receivers verify with a constant-time comparison and reject timestamps outside a tolerance (five minutes is usual). Secrets are shown once at creation and stored encrypted at rest (AES-GCM); a one-shot signed test delivery from the console lets the partner check their verifier before anything real flows.

## Deterministic event ids

A publish that might be retried passes an id derived from the domain change (the order id and the transition, say). The ledger insert is `ON CONFLICT DO NOTHING`, so the second publish is a no-op: no second event, no second deliveries. One delivery row per (endpoint, event), unique, so fan-out is idempotent too. A pre-launch audit flagged exactly this: without deterministic ids, a retried transition could double-notify.

## A delivery ledger, not a fire-and-forget

Deliveries are rows with attempts, a next-attempt time and a lease. Workers claim due rows under `SKIP LOCKED`, POST, and on a non-2xx or a transport error schedule the next attempt with exponential backoff. After the attempt budget the delivery is failed and visible in the partner's request history. An endpoint that fails N times in a row is disabled, so a dead URL stops consuming retries; a success resets the counter; the partner re-enables it from the console.

## Replay

`replay(endpoint, since, until)` returns delivered and failed rows in the window to pending with a fresh budget. The partner sees the same event ids again, so an idempotent receiver is unaffected. This is the feature partners ask for the week after they fix a bug.

## Egress rules at connect time

Webhook URLs are partner-supplied, which makes the sender an SSRF vector. Validating the URL's hostname at registration is not enough: DNS can resolve to a private address later. The egress policy has to be enforced at TCP connect time, on the resolved address, every time. The same audit that found the event-id issue found this one.

## What I would do differently

Build the partner-facing request history first. Every support question about webhooks is "did you send it, and what did we return", and the ledger answers both; the console view of it is what turns the ledger into fewer tickets.

A runnable version is in [durable-integrations](https://github.com/maydan90/durable-integrations) (`src/webhooks.ts`).
