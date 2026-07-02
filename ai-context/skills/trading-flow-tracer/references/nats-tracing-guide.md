# NATS / Message-Queue Tracing Guide

How to reconstruct the asynchronous half of an order flow. Applies to NATS (core + JetStream); the same approach works for Kafka.

## Step 1: Find all publishers

Grep the codebase for publish calls:

```
nc.publish, nc.Publish, js.publish, js.Publish
.publish(, producer.send, producer.Produce
```

For each hit record: subject/topic (literal or built from a constant), payload type, the service, and the triggering handler.

## Step 2: Find all consumers/subscribers

Grep for subscription calls:

```
nc.subscribe, nc.Subscribe, js.subscribe, queueSubscribe, QueueSubscribe
.subscribe(, consumer.consume, AddConsumer, PullSubscribe
```

For each hit record: subject filter, queue group (if any), durable name (JetStream), the service, and the handler callback.

## Step 3: Build the producer → subject → consumer chain

Match publishers to consumers by subject. Watch for:
- **Wildcards** — `order.*` / `order.>` subscriptions catch multiple publish subjects.
- **Subject built from variables** — resolve the constant/template to the literal subject.
- **Queue groups** — only one member of a queue group receives each message (load balancing); a wrong queue group = missed delivery.

## Step 4: Inspect delivery semantics

| Concern | What to check |
|---|---|
| Ack | Is the consumer manual-ack? Is `ack()` actually called on all paths (including error paths)? |
| Redelivery | JetStream redelivers un-acked messages — handler must be idempotent. Non-idempotent handler + redelivery = duplicate processing. |
| Ordering | Core NATS gives no ordering guarantee across subjects; JetStream ordered consumers are opt-in. Do not assume hold-then-release arrives in order. |
| Durability | Core NATS subscription that starts after publish misses the message. JetStream durable consumers replay. |
| Max deliver / DLQ | After max redelivery the message is dropped or dead-lettered — a silently lost event. |

## Step 5: Correlate with logs

Search runtime logs for the order's ClOrdID/order ID across services. Build a timestamp-ordered list of publish and consume events. A gap (published, never consumed) or a duplicate (consumed twice) is the divergence point.

## Common MQ-related flow bugs

- **Missing subscription** — publisher exists, no consumer for that subject (typo, wrong constant).
- **Consumer crash before ack** — message redelivered, handler re-runs non-idempotently.
- **Race: release before hold** — two events, no ordering guarantee, consumer assumes order.
- **Wrong queue group** — message goes to the wrong instance / a sibling service.
- **Slow consumer / backpressure** — JetStream pending grows, processing lags, order looks "stuck".
- **Subject env mismatch** — subject prefixed per environment (`uat.order.new` vs `order.new`).

## Quick diagnostic commands

If a NATS CLI is available:

```
nats stream ls
nats consumer report <stream>
nats stream view <stream>
nats sub "order.>"        # observe live traffic
```