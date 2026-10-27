Reviewer: Claude Code (model: Claude Opus 5.5, `claude-opus-5-5`)

# Review: ADR-0002 (Event-driven `OrderPlaced` via RabbitMQ for post-order actions)

Reviewed file: `docs/adr/0002-use-event-driven-rabbitmq-for-post-order.md`. Facts are from `docs/scenario.md`.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | Shoppers | 200 `PlaceOrder` calls/s | 11.11 peak, all consumers healthy | Checkout publishes `OrderPlaced` and returns | p95 `PlaceOrder` ≤ 500 ms (today ~300 ms, so ≤ ~200 ms for publish + confirm + retries) |
| QA2 Availability | MailFast | Outage or p99 8 s / 0.5 % errors | Peak, RabbitMQ up | Orders succeed. Emails queue, retry and dead-letter | 0 orders failed because of email. Send rate never > 50/s. Every email eventually sent (backlog drain time stated, see M2) |
| QA3 Correctness | RabbitMQ | Redelivers `OrderPlaced` after a `loyaltyservice` crash before ack | Peak load | Points awarded once | 0 double awards. 100 % of orders get points ≤ 60 s after the order |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | **Lost event between `shipOrder` and publish confirm.** Checkout has no database (ADR l.17, l.82), so a failed publish is "logged for replay" (l.86) but there is no durable record to replay from. | The card is charged and the order shipped, but the customer gets no email and no points, silently. This conflicts with the Validation claim that "every email and point is delivered" (l.92). |
| Risk | **"Retry with a short timeout" on publish is unbounded in the ADR** (l.61). During a RabbitMQ slowdown, every order waits for the retries. | A broker problem then becomes a checkout latency problem. That breaks QA1 even though orders still "succeed". |
| Risk | **Email backlog numbers were removed.** The commented-out line (l.78) states ~90,000 queued emails and ~30 min to drain after a 10-minute peak. The live line (l.79) gives only the rate. | The board can't judge whether a 30+ min confirmation delay is acceptable. Retries of the 0.5 % failures also spend the 50/s budget, so the drain is slightly slower. |
| Sensitivity point | **Publish timeout / retry budget** in `PlaceOrder`. | p95 latency (QA1) and the lost-event rate both depend directly on this one number. |
| Sensitivity point | **Email consumer rate (≤ 50/s) vs. the peak duration.** | Confirmation delay grows linearly with the peak: (200 − 50) × peak seconds ÷ 50 to drain. |
| Trade-off point | **Return to the customer before the publish is durable vs. block until confirmed.** | Faster, always-successful checkout (performance, availability) vs. a guaranteed post-order effect (reliability). The ADR chooses the first one but doesn't name this as a trade-off. |
| Trade-off point | **RabbitMQ 3-node cluster with quorum queues** (l.86). | Better broker availability and durability, but more operational cost for a 4-person team, and higher confirm latency than a single node. |
| Non-risk | **Separate durable queue per consumer** (l.62, l.84). | The throttled MailFast backlog can't delay loyalty, so QA3 (≤ 60 s) is independent of QA2. |
| Non-risk | **Idempotent `loyaltyservice` keyed on `order_id`, with a unique constraint, acknowledged after commit** (l.64). | This is the correct way to get "exactly once" on top of at-least-once delivery. |
| Non-risk | **RabbitMQ over Kafka for 200 msg/s**, given the team's experience and the need for per-queue throttling (l.68). | Fits the scenario's team and self-hosted constraints. |

## 3. Missing or weak

- **Option list mixes levels.** E (event-driven style) is not an alternative to C and D; C and D implement it (l.45). The real comparison is A vs. B vs. {C, D}. Missing alternatives: a **transactional outbox** (give checkout a small store, or have the relay publish from it), and a **lighter broker** (e.g. NATS JetStream or Redis Streams, Redis is already in the system per ADR-0001's name). Also missing: **async only for email**, with loyalty kept as a synchronous idempotent gRPC call.
- **Unsupported claims:** "a local publish of a few ms" (l.72). A publish with confirms to a 3-node quorum cluster is a network round trip plus a replicated write, and it isn't measured. "Without being rejected" (l.74) depends on queue length limits and disk, which the ADR doesn't size (~90k messages, see the commented line).
- **Unstated consequences:** the `loyaltyservice` dedup store is a second new data store that needs ownership, backup and retention. The ADR doesn't specify the event schema or versioning (Protobuf vs. JSON) for future consumers. The ADR doesn't say whether the publish happens before or after `emptyUserCart`.
- **Not evidenced in the provided files:** "marketing's analytics store" (l.65) and its interface. Who replays the dead-letter queues, and how.
- **Governance:** one decider (l.5) for a cross-team change (marketing, loyalty).

## 4. Verdict: **Accept with changes**

1. **Close the lost-event gap.** Add a durable outbox (or persist failed publishes to a store with a replay job), or explicitly accept the risk with a measured target (e.g. ≤ 1 in 10⁵ orders) and an alert.
2. **Bound the publish path:** a hard timeout and retry count that keep p95 ≤ 500 ms. Prove it in the QA1 load test with the 3-node quorum cluster.
3. **Restore the backlog numbers** (~90k emails, ~30 min drain) and get business sign-off on the maximum confirmation delay. Size queue length limits and disk to match.
