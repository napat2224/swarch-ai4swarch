# ADR-NNNN: RabbitMQ for Post Order Action

- **Status:** Proposed
- **Date:** 2027-10-25
- **Deciders:** Napat Srisamut

## Context and problem

> After an order is placed, Online Boutique must:
> 1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
> 2. award **loyalty points** through a new `loyaltyservice` (to be built);
> 3. record an **"order placed" analytics event** for the marketing team.
>
> Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).


How it works now: `checkoutservice.PlaceOrder` calls `paymentservice`, `shippingservice` and then `emailservice` **synchronously over gRPC**, in sequence. If the email call fails, checkout only logs a warning, and the order still succeeds.


| Topic | Fact |
|---|---|
| Checkout latency | p95 of `PlaceOrder` must stay **≤ 500 ms** (today approximately 300 ms) |
| Peak load | **200 orders/s** during the 11.11 sale |
| MailFast | p95 **1.5 s**, p99 **8 s**; fails approximately **0.5 %** of calls; rate limit **50 emails/s** |
| Loyalty points | must be awarded **within 1 minute** of the order, **exactly once** (no double points) |
| Availability | an order must **never fail** because email, loyalty or analytics is down |
| Team | 4 developers; strong in Go and gRPC; two have used **RabbitMQ** before |
| Operations | everything already runs on Kubernetes; the company prefers **open-source, self-hosted** components over cloud-only managed services |


## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** p95 of `PlaceOrder` must stay ≤ 500 ms (today about 300 ms), even though MailFast's p95 is 1.5 s.
3. **Scalability:** must handle the peak of 200 orders/s during the 11.11 sale while respecting MailFast's rate limit of 50 emails/s.
4. **Reliability:** loyalty points must be awarded within 1 minute and exactly once; MailFast's 0.5 % failures must be retried, not lost.
5. **Modifiability:** adding post-order actions next year (SMS, fraud scoring, recommendations) should not require changing `checkoutservice`.
6. **Constraints:** open-source and self-hosted on Kubernetes; fits a 4-person team that is strong in Go and gRPC, two of whom know RabbitMQ.

## Considered options

1. **Synchronous gRPC chain:** extend `checkoutservice.PlaceOrder` so it calls MailFast (through `emailservice`), `loyaltyservice` and analytics one after another, like the email call works today.
2. **In-process async calls:** `PlaceOrder` returns as soon as the order is placed, and goroutines inside `checkoutservice` call MailFast, `loyaltyservice` and analytics in the background, with retries.
3. **Event-driven with RabbitMQ:** `checkoutservice` publishes one `OrderPlaced` event to a self-hosted RabbitMQ broker. Email, loyalty and analytics are separate consumers, each with its own durable queue.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC chain | Simplest change; no new infrastructure (cost) | p95 > 500 ms (performance); exceeds 50 emails/s limit (scalability); MailFast failures block checkout (availability); each new action changes `checkoutservice` (modifiability) |
| 2. In-process async calls | Fast checkout (performance); no new infrastructure (cost) | Work lost on pod restart (reliability); no exactly-once loyalty (reliability); manual rate limiting (scalability); each new action changes `checkoutservice` (modifiability) |
| 3. Event-driven with RabbitMQ | Fast checkout (performance); queue absorbs peak (scalability); retries survive failures (reliability); orders never blocked (availability); new actions = new consumers (modifiability); team knows RabbitMQ (constraints) | Extra component to operate (cost); needs de-duplication by order ID (reliability); delayed emails at peak (consistency); broker downtime needs an outbox (availability) |

## Decision

Use **option 3: event-driven with RabbitMQ.** `checkoutservice` publishes an `OrderPlaced` event; email, loyalty and analytics consume it from their own durable queues. It is the only option that meets availability, performance and scalability.

## Consequences

- **Positive:** fast checkout (performance); orders never blocked by post-order actions (availability); peak load buffered (scalability); new actions = new consumers (modifiability).
- **Negative:** one more component to run (cost, operability); emails may be delayed at peak (consistency); harder end-to-end tracing (observability).
- **Risks and mitigations:**
  - Duplicate events → `loyaltyservice` de-duplicates by order ID (exactly once).
  - RabbitMQ down → outbox in `checkoutservice`, publish later.
  - MailFast failures or rate limit → retries with backoff; email consumer capped at 50 emails/s.
  - Broker failure → run RabbitMQ clustered with quorum queues.

## Validation

- Load test at 200 orders/s: p95 `PlaceOrder` ≤ 500 ms.
- Loyalty points awarded within 1 minute, with no duplicates.
- Stopping MailFast or RabbitMQ causes no failed orders.
