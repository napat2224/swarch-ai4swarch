Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`).

# Review of ADR "RabbitMQ for Post Order Action" (`docs/adr/0002-use-rabbitmq-for-post-order-actions.md`)

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QAS-1 Performance | Shoppers | 200 `PlaceOrder` requests/s | 11.11 peak, MailFast at p95 1.5 s | Checkout publishes `OrderPlaced` and returns without waiting for MailFast | `PlaceOrder` p95 ≤ 500 ms. Zero orders rejected |
| QAS-2 Availability | MailFast / `loyaltyservice` / RabbitMQ | One of them is down for 10 min | Normal load | The order still succeeds. Post-order work is kept and completed after recovery | 0 failed orders. 0 lost emails or points |
| QAS-3 Reliability | RabbitMQ | Redelivers an `OrderPlaced` message (consumer crash before ack) | 11.11 peak | `loyaltyservice` awards points once | Points within 1 min of the order. 0 double awards |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Taking MailFast off the synchronous `PlaceOrder` path | MailFast p95 1.5 s alone breaks the 500 ms budget. Asynchronous delivery is the only way to meet QAS-1 |
| Non-risk | RabbitMQ over other brokers | Open source, self-hosted on Kubernetes, and two of the four developers know it. It fits the stated constraints |
| Risk | "Outbox in `checkoutservice`" is a one-line mitigation, but the ADR never says what database the outbox writes to. Checkout has no stated datastore. The outbox is also not in the Decision section | If the publish fails after payment and shipping succeeded, the email and points are silently lost (fails QAS-2). Today the email is at least attempted |
| Risk | The ADR doesn't say whether the email, loyalty and analytics queues are throttled separately | If the loyalty consumer is slowed by the email backlog, the 1-minute loyalty deadline fails at peak (QAS-3) |
| Risk | Exactly-once rests on "de-duplicate by order ID", but the dedupe store, its transaction boundary and its retention are unspecified | Points are money-like. A dedupe check that isn't atomic with the points write still double-awards points |
| Sensitivity point | Email consumer rate cap vs. arrival rate (50/s vs. 200/s) | About 150 emails/s pile up. Every minute of peak adds about 9,000 queued emails and about 3 min of drain time. Email delay depends directly on how long the peak lasts (not given) |
| Sensitivity point | Consumer prefetch, retry backoff and timeout vs. MailFast p99 8 s | Too short a timeout means duplicate emails. Too long means consumers stall and the backlog grows |
| Trade-off point | Asynchronous delivery: better performance, availability and modifiability, but worse consistency (late emails) and observability | The customer may get no confirmation for minutes during 11.11. The business must accept this explicitly |
| Trade-off point | A clustered RabbitMQ with quorum queues: better availability, but higher cost and operational load for a 4-person team | A new stateful system to run around the busiest sale of the year |

## 3. Missing or weak

- **Options not considered:** Kafka or NATS as alternative brokers, and a hybrid in which loyalty is a direct idempotent gRPC call while email and analytics go through the queue. With only three options, the comparison of broker choices is thin.
- **Fraud scoring** is listed as a future post-order consumer. Fraud checks usually have to block the order before payment, so it may not fit this event model at all. The ADR is silent on this.
- **Unquantified consequence:** "emails may be delayed at peak" gives no backlog size or drain time, and no maximum acceptable delay.
- **Missing details:** where in `PlaceOrder` the event is published (after shipping?) and what the event contains. Analytics receiving customer email and address is a privacy issue that isn't mentioned.
- **Unsupported claim:** "orders never blocked". This holds only if the publish is non-blocking and has a short timeout. If the broker hangs, so does checkout, unless the outbox exists.
- **Validation gaps:** no redelivery test for duplicates, no backlog drain measurement, no MailFast p99 or 0.5 % failure injection, no review date.
- **Housekeeping:** the title still reads `ADR-NNNN` (should be ADR-0002), and the date 2027-10-25 is in the future.

## 4. Verdict: **Accept with changes**

1. **Specify the outbox:** its store, the publish point in `PlaceOrder`, and the relay. Alternatively, explicitly accept the risk of losing an event after payment. Move the chosen approach into the Decision section.
2. **Quantify the 11.11 email backlog** (150/s excess) and agree a maximum email delay with the business. Add separate queues with independent throughput so that loyalty's 1-minute target is not affected by MailFast throttling.
3. **Define exactly-once:** the dedupe table, atomic with the points write, plus a forced-redelivery test. Also decide whether fraud scoring belongs on this asynchronous path.
