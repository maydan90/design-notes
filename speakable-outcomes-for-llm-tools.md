# Speakable outcomes for LLM tools

## The setting

A phone agent built on a cascade (speech-to-text, a model, text-to-speech) calls backend tools to check availability, book, change and cancel. The model is good at conversation and bad at the one thing a backend cares about: knowing what is true after a call. Left to itself it will retry a refusal, apologise for a success, and promise things no tool confirmed.

## The contract

Tools return outcomes, not errors. Each outcome has three fields.

- `code`: a stable identifier (`slot_unavailable`, `card_required`, `unconfirmed`). Evals and logs key on it.
- `effect`: what is true now. `booked`, `changed`, `cancelled`, `nothing_changed`, `unconfirmed`. This is the field that stops loops. A model that reads "nothing was booked" does not call the tool again with the same arguments; a model that reads "I could not confirm" stops writing and hands off.
- `speakable`: the sentence the agent says verbatim. Written for the ear: "That time is full. I do have 3 PM, 4 PM. Nothing was booked yet." Confirmation codes are read letter by letter from an alphabet without `0/o` and `1/i/l`, because text-to-speech invents words and speech-to-text mishears the readback.

The prompt then only has to say: relay what the tool says, do not reword the part about whether something was booked.

## Request ids are per change, not per attempt

The id sent to the backend is derived from the change itself: intent plus arguments within one call. If the model calls `book_appointment` twice with the same details, because it did not understand the first answer or because the first answer was lost, the backend sees the same id and returns the same booking. A different time or party size is a different change and gets a new id, so a legitimate second booking in the same call still works.

## Lost answers are recovered, not retried

A timeout from the backend is neither success nor failure. The tool looks the request id up first. If the write applied, that is the outcome and nothing is resent. If it did not, the tool resends once with the same id. If that is lost too, the outcome is `unconfirmed`: the caller hears that nothing was booked twice, that the agent will not keep trying, and that a person will call back. Two attempts, then stop. This is the same rule as in [third-party writes have three outcomes](third-party-writes-have-three-outcomes.md), applied inside a tool.

## Some decisions must never reach the model

Opening hours, lead time, booking horizon, party-size limits, card requirements, cancellation fees. These are enforced in code before the backend is called. The prompt tells the model the rules exist so it can set expectations, and the tool's refusal is what the model says when a caller pushes. A model can be talked into anything; a policy module cannot.

## Evals without a model

The deterministic half of a voice agent, "given these tool calls and these backend conditions, what does the caller hear and what does the backend hold", can be tested by replaying scripted tool-call sequences through the real tools with a backend that loses answers on purpose. That suite runs in a second and belongs in CI. Whether the model picks the right tool from speech is a different, non-deterministic eval that belongs with real transcripts and a rubric; mixing the two makes both worse.

A runnable version is in [voice-booking-agent](https://github.com/maydan90/voice-booking-agent).

## What I would do differently

Ship the typed outcomes before the first production call, not after the first retry loop. Every tool I have retrofitted this onto had at least one raw error reaching the model somewhere.
