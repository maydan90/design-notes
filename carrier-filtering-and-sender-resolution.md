# Carrier filtering and sender resolution

## The number

After adding delivery status callbacks to the SMS integration, which we should have had from the start, we could finally see how customer texts ended. In one window, 525 payment reminders had been blocked by carriers with the "carrier filtering" error code. All of them had been sent from the shared sending number; merchants with their own registered numbers had none. About 38% of reminders from the shared number were not arriving.

## The cause

The reminder schedule sent the same payment link three times in ten minutes from one number. That pattern, identical content, high frequency, one sender, many recipients, is what carrier spam filters are built to catch. The shared number was doing exactly what a spammer does, for a good reason.

## Why not a rate limit

A rate limit would have spread the sends out and still sent the same content three times. The actual decision that needed to be right was *which number sends this customer's messages*, and that decision was being made per message, at send time, so a later reminder could fall back to the shared number even when an earlier one had gone from the merchant's own line.

## The fix

Sender resolution became a durable decision, made once per order as a step in the pay-by-link workflow and stored. A merchant's own registered number keeps the three-reminder schedule. The shared number sends one reminder, with different wording, and a later reminder can never fall back to it. Reminder log lines carry the provider's message id so delivery reports join exactly to what we sent.

Then the same query was run again after release. Blocked reminders went to zero (none of the 43 blocked messages in the next window were reminders). Reminder sends fell by about 60%. The paid rate on shared-number orders held, 70% against 68% on a one-day sample, re-checked a week later. The follow-up was to move the busiest merchants onto their own numbers, which is the real fix for the shared-number problem.

## What I would do again

Start from the delivery data, change the decision rather than the schedule, and re-run the same query after release. The before and after numbers are the review.

## What I would do differently

Add status callbacks on the first day the integration sends a message. Every one of these problems was invisible until it was instrumented, and the instrumentation took an afternoon.
