# design-notes

Short write-ups of designs and incidents from building integrations, payments and an LLM phone agent at a restaurant-tech startup, and before that a warehouse platform at a payments hardware company. Each note is a few minutes' read: the problem, what we did, what I would do again, and what I would do differently. Details are generic on purpose; no employer code, prompts or data appear here.

| Note | One line |
|---|---|
| [Third-party writes have three outcomes](third-party-writes-have-three-outcomes.md) | A timeout is not a failure. How to make "unknown" a first-class state, and the two bugs that taught me. |
| [Speakable outcomes for LLM tools](speakable-outcomes-for-llm-tools.md) | The contract between a voice agent and its backend: typed outcomes, per-change request ids, bounded recovery, and decisions that never reach the model. |
| [Idempotency keys, scoped](idempotency-keys-scoped.md) | What an idempotency key protects, what it does not, and the scope bug a reviewer caught. |
| [Webhooks a partner can trust](webhooks-a-partner-can-trust.md) | Signatures with the timestamp inside, deterministic event ids, a delivery ledger, auto-disable, replay, and egress rules enforced at connect time. |
| [When logs silently stop](when-logs-silently-stop.md) | An observability outage with no error anywhere: a field limit, a batch-level rejection, and the monitor that would have caught it. |
| [Carrier filtering and sender resolution](carrier-filtering-and-sender-resolution.md) | Why 38% of payment reminders never arrived, and why the fix was a durable decision, not a rate limit. |

Two of these have runnable companions: [durable-integrations](https://github.com/maydan90/durable-integrations) (TypeScript + PostgreSQL) for the first, third and fourth notes, and [voice-booking-agent](https://github.com/maydan90/voice-booking-agent) (Python, LiveKit Agents) for the second.

## How I write these

Each note starts from a production number or a specific failure, because designs that start from principles tend to defend themselves against problems nobody has. The "what I would do differently" sections are the honest part; most of them come down to instrumenting earlier.

MIT licensed. [Yurii Maidaniuk](https://github.com/maydan90).
