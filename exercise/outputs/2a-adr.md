Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`). Run a: zero-shot, no context files.

# ADR-0002: Publish an "OrderPlaced" event to trigger post-order actions asynchronously

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not given (see Open questions)

## Context and problem

After an order is placed, Online Boutique must do three new things:

1. Send the order confirmation email through **MailFast**, a new external email provider. MailFast replaces the mock `emailservice` logic.
2. Award loyalty points through a new `loyaltyservice`, which has not been built yet.
3. Record an "order placed" analytics event for the marketing team.

Marketing expects more post-order actions next year: SMS, fraud scoring and a recommendations update.

The problem is how the order-placing step should trigger a growing set of post-order actions. Some of those actions depend on systems outside our control (MailFast). The current system architecture is not part of this run's input, so this ADR assumes only that some service places the order. Later text calls that service the "order placer" **[Assumption]**.

## Decision drivers

1. **Modifiability / extensibility:** at least three more consumers are expected next year. Adding one should not require changing the order placer.
2. **Availability of order placement:** if MailFast, `loyaltyservice` or analytics fails, placing an order must still succeed **[Assumption: the business prefers accepting the order to blocking it]**.
3. **Performance:** the user-facing order latency should not grow with each new post-order action.
4. **Reliability of side effects:** each action must eventually happen (at least once) and must not double-award points.
5. **Cost and operational complexity:** keep new infrastructure proportionate to the team's skills (unknown).

## Considered options

1. **A: Synchronous calls from the order placer.** After the order is saved, the order placer calls MailFast (or an email adapter), `loyaltyservice` and analytics directly, one after another.
2. **B: Event-driven publish/subscribe.** The order placer publishes one `OrderPlaced` event to a message broker. Independent consumers react to it: an email adapter that calls MailFast, `loyaltyservice` and an analytics forwarder.
3. **C: A dedicated post-order orchestrator.** The order placer makes one call to a new orchestrator service. The orchestrator runs the post-order steps, with retries, possibly as a workflow engine.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous calls | Simplest: no new infrastructure. Easy to trace and debug. The user gets an immediate result for each step | Order latency is the sum of all calls. A MailFast outage can fail or slow the order. Every new action (SMS, fraud…) means changing and redeploying the order placer. Retries are hand-written |
| B: Pub/sub events | The order placer doesn't know its consumers, so new actions are added without touching it. Slow or failing consumers don't affect the order. The broker buffers bursts and supports retries and dead-letter queues | A new broker to run (cost, ops skills). Eventual consistency: the email may arrive after a delay. At-least-once delivery needs idempotent consumers. End-to-end tracing is harder. The event schema becomes a contract to version |
| C: Orchestrator | One place to see and control the post-order flow and its retries. Can enforce step ordering (e.g. fraud scoring before loyalty). The order placer changes only once | A new service that is coupled to every step and must change for each new action. If the call to it is synchronous, its availability still affects ordering. A workflow engine may be heavy for three steps |

## Decision

Choose **Option B: event-driven publish/subscribe**.

- On a successful order, the order placer publishes one `OrderPlaced` event (order ID, user, items, totals, email address, timestamp, event ID).
- Consumers:
  - **email adapter:** replaces the mock `emailservice` logic and calls the MailFast API.
  - **`loyaltyservice`:** awards points, deduplicating by order ID / event ID.
  - **analytics forwarder:** sends the event to marketing's analytics system.
- Future SMS, fraud scoring and recommendations consumers subscribe to the same event without changes to the order placer.
- To avoid losing events when the order is saved but the publish fails, use a transactional outbox, or publish only after the order is committed and retry **[to confirm against how orders are stored]**.

B best fits the top two drivers: extensibility (new consumers are additive) and order availability (post-order failures are isolated). A fails both. C centralises the flow but keeps a single place that must change for every new action.

Fraud scoring may need to run **before** the order is confirmed. If so, it does not belong on this asynchronous path and needs its own decision (see Open questions).

## Consequences

- **Positive:** new post-order actions are new subscribers, with no change to the order placer (quality attribute: modifiability).
- **Positive:** a MailFast, loyalty or analytics outage doesn't block orders, and messages wait in the broker until consumers recover (quality attribute: availability).
- **Positive:** order latency adds only one publish call, however many consumers there are (quality attribute: performance).
- **Positive:** each consumer scales and deploys on its own (quality attribute: scalability, deployability).
- **Negative:** a message broker must be provisioned, monitored and secured (quality attributes: cost, operability).
- **Negative:** the confirmation email and points are eventually consistent, so the user may see them with a delay (quality attribute: usability / consistency).
- **Negative:** asynchronous flows are harder to trace and debug (quality attribute: observability / testability).
- **Risks and mitigations:**
  - Duplicate delivery could double-award points or send two emails → idempotent consumers keyed by event ID.
  - A lost event between saving the order and publishing → transactional outbox, or a publish retry.
  - MailFast rate limits or outages → retry with backoff, a dead-letter queue and an alert.
  - Event schema changes break consumers → versioned schema with additive-only changes.
  - Customer PII (email, address) in events goes to analytics → send analytics only the fields it needs.
  - The team has no experience with brokers → pick a managed broker if available (open question).

## Validation

- **Load test:** order latency (p95) with the event publish is no worse than today's baseline plus an agreed margin (number not given).
- **Fault injection:** with MailFast or `loyaltyservice` stopped, orders still succeed. After recovery, all emails and points are delivered, with no duplicates.
- **Metrics:** consumer lag, dead-letter queue size and the share of orders whose confirmation email was sent within an agreed time. Alerts on each.
- **Extensibility check:** add a stub SMS consumer with zero changes to the order placer.
- **Review date:** after the first new post-order action next year (date not given).

## Open questions

- Who are the deciders?
- What is the current architecture? Which service places orders, how do services communicate, and how are orders stored? This decides whether an outbox is possible.
- Which message broker? Is a managed cloud service allowed? What are the cost limits?
- What skills does the team have with message brokers and asynchronous systems?
- Latency and throughput targets: order p95, peak orders per second, and the acceptable delay for the confirmation email and loyalty points.
- MailFast details: API type, rate limits, SLA, cost per email, whether it supports idempotency keys, and data-processing terms for customer PII.
- Does the business really prefer accepting an order when the email cannot be sent yet? Is any action allowed to block the order?
- Must fraud scoring run before the order is confirmed (synchronously) or after it?
- Which analytics system does marketing use, and which event fields are allowed for privacy (GDPR / consent)?
- `loyaltyservice` rules: points per order, and what happens to points when an order is cancelled or refunded.
