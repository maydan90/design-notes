# Third-party writes have three outcomes

## The problem

Every integration I have worked on, payment processors, a reservations provider, a messaging API, has the same shape at the moment that matters: we send a write, and one of three things comes back.

1. A confirmation. The write applied.
2. A refusal with a reason. Nothing changed on their side.
3. Nothing. A timeout, a reset connection, a 502 from a proxy in between. The write may have applied.

Most code treats the third case as either the first or the second, and both are wrong in expensive ways. Treating it as failure and retrying means a duplicate when the first attempt did go through: a second reservation, a second charge, a second merchant record. Treating it as success means telling a customer they are booked when they are not.

## Two bugs that taught me

A reservations integration classified the provider's 403, meaning "only the host may touch this booking", as an unknown outcome. The operation stayed pending and was retried every minute for twenty-four hours. Callers were told their change was "still confirming". The retry volume was around six thousand a week; one booking was retried over a thousand times in a day. The fix was to separate a definitive refusal from a lost answer, and the subtlety was that a refusal met while *recovering* an earlier lost attempt cannot be read as "nothing changed", because the earlier attempt may have applied. Those two paths need different code.

The mirror image happened with an LLM phone agent. A provider refusal reached the model as raw text; the model, doing what models do, retried the same call three times in four seconds. Once the backend returned a typed outcome that said "that time needs a card, nothing was booked", the loop stopped. Same root cause, opposite symptom: the caller could not tell which of the three outcomes had happened.

## The design

Every write to a third party becomes a row before it becomes a request.

- **One request id for the life of the operation.** Minted when the row is created, sent on every attempt. A provider that dedupes on it cannot double-apply. A provider that does not still only sees one send per attempt, and the lookup rule below keeps that safe.
- **States that say what we know**: pending, in flight, succeeded, refused, unknown, stuck. "Failed" does not exist, because after a lost answer we do not know that it failed.
- **Look before you write again.** If any previous attempt may have reached the provider (an unknown outcome, or a worker that crashed mid-attempt and whose lease expired), the next attempt first asks the provider whether the request id applied. Found: record success, send nothing. Not found: resend with the same id.
- **Leases, with fencing.** Workers claim rows under `FOR UPDATE SKIP LOCKED` and hold a lease. Every state write is conditioned on still owning the lease, so a worker that stalled and wakes up late cannot overwrite what the new owner recorded.
- **Bounded, honest retries.** Exponential backoff with jitter. After the attempt budget the row is *stuck* and listed for a human or a reconciliation job. Nothing is guessed.

A runnable version with tests is in [durable-integrations](https://github.com/maydan90/durable-integrations).

## What I would do again

Make the unknown state visible in the product, not just the database. "Still confirming" was the right thing to say for a lost answer; the bug was saying it for a refusal. A truthful sentence per state is worth more than a cleverer retry policy.

## What I would do differently

Instrument the state transitions from day one. The six-thousand-retries number came from a query written while investigating a complaint. A per-operation-kind count of transitions into `unknown` would have shown the problem the week it started.
