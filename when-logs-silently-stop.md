# When logs silently stop

## The incident

HTTP request logs from several services, backend, public API, auth, webhooks, stopped arriving at the log platform from every environment. Nothing errored. No alert fired. The gap was noticed days later while investigating an unrelated question, because the query that should have answered it returned nothing.

## The cause

The log platform's dataset had a limit on the number of distinct field names. We were at it. The platform's behaviour at the limit is to reject the entire OTLP batch that would introduce a new field, and to do so silently from the sender's point of view. A new annotation on the request-context middleware, the forwarded-for chain that every request behind the load balancer carries, added one key to every HTTP log line. Every batch now contained a new field, so every batch was rejected.

Two properties combined: a hard limit with a silent failure mode, and a change that touched every record at once.

## The fix

- Scope-wide keys (the forwarded chain, the API client identity, device identity) moved from log attributes to span attributes, where they land in a map rather than as top-level fields. Trace ids stayed on log lines for correlation.
- A rule in the repository's contributor docs: do not add a log attribute key without checking the dataset's field count. The later partner-diagnostics and SMS-delivery-tracking work was designed with keys on spans or in the message body from the start.

## What would have caught it

A monitor on log volume per service, alerting on a drop to zero over a window. It is the cheapest observability-of-observability check there is, and the one most teams do not have because logs feel like infrastructure rather than a dependency. The platform's rejection metric, where exposed, is the other side of the same alert.

## The general lesson

Anything that ingests data has a limit, and the interesting question about every limit is what happens at it. Reject the record, reject the batch, truncate, or sample? Loud or silent? I now ask that question about every sink before I rely on it, and I would rather have a sink that drops one record loudly than one that drops a batch quietly.
