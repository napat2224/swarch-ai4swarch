Agent: Claude Code (Anthropic), model Claude Opus 5.5 (`claude-opus-5-5`). Run b: with context (`docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`).

# ADR-0002: Publish an OrderPlaced event through RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not given (see Open questions)

## Context and problem

Requirement R-12 (`docs/scenario.md`): after an order is placed,

1. send the confirmation email through **MailFast**, which replaces the mock `emailservice` logic;
2. award loyalty points through a new `loyaltyservice`;
3. record an "order placed" analytics event.

More post-order actions are expected next year: SMS, fraud scoring and a recommendations update.

**Today:** `checkoutservice.PlaceOrder` (Go) runs every step synchronously over gRPC in sequence. Its order of calls is: get the cart, prepare items and a shipping quote → `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` (`main.go`, `PlaceOrder`, l.230–280). An email failure is only logged with `log.Warnf`, and the order still succeeds (l.273–277). Each downstream service is a hard-wired `*_SERVICE_ADDR` with a gRPC connection created at startup (`main.go` l.111–123). Adding an action therefore means a new env var, a new connection and a new call inside `PlaceOrder`. The order ID is generated in memory (`uuid.NewUUID`, l.233), and `checkoutservice` has no database of its own.

**Facts that rule out the current pattern** (`docs/scenario.md`):

| Fact | Consequence |
|---|---|
| `PlaceOrder` p95 must be ≤ 500 ms. Today it is approximately 300 ms | About 200 ms of headroom |
| MailFast p95 1.5 s, p99 8 s, fails approximately 0.5 % of calls | A synchronous MailFast call alone breaks the 500 ms budget |
| Peak 200 orders/s (11.11) vs. MailFast rate limit 50 emails/s | Emails arrive 4× faster than MailFast accepts them, so about 150 emails/s must be buffered during peak |
| Loyalty points: within 1 minute, **exactly once** | Need reliable delivery and deduplication |
| An order must **never fail** because email, loyalty or analytics is down | Post-order actions must be decoupled from the order's success |
| 4 Go/gRPC developers; two have used RabbitMQ. Kubernetes; open-source and self-hosted preferred | Constrains the technology choice |

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms at 200 orders/s.
3. **Reliability / correctness:** loyalty points within 1 min, exactly once. Emails must survive MailFast's 0.5 % failure rate and its 50/s rate limit.
4. **Modifiability:** adding SMS, fraud scoring and recommendations next year without changing `checkoutservice` each time.
5. **Team fit and operations:** Go/gRPC skills, RabbitMQ experience, self-hosted open source on Kubernetes.
6. **Cost / simplicity:** a 4-person team must operate it.

## Considered options

1. **A: Extend the synchronous chain:** `checkoutservice` calls MailFast, `loyaltyservice` and analytics directly over gRPC/HTTP, like `sendOrderConfirmation` today.
2. **B: Fire-and-forget goroutines in `checkoutservice`:** return the order response first, then call each action from background goroutines with in-process retries.
3. **C: RabbitMQ publish/subscribe:** `checkoutservice` publishes one `OrderPlaced` event to a RabbitMQ fanout/topic exchange. Each action (email worker → MailFast, `loyaltyservice`, analytics consumer) has its own durable queue.
4. **D: Apache Kafka publish/subscribe:** same event model on a Kafka topic, with a consumer group per action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous chain | No new infrastructure. Same pattern as today. Easy to trace (OpenTelemetry is already wired in, `main.go` l.134–140) | MailFast p95 1.5 s alone exceeds the 500 ms p95 budget. 200 orders/s against a 50/s rate limit causes throttling. Every new action changes `PlaceOrder`. Latency grows with each action |
| B: Goroutines | Removes post-order latency from the response. No new infrastructure | Work is lost if a pod restarts. No durable buffer for the 150 emails/s excess at peak. Exactly-once loyalty can't be guaranteed. Still one code change per new action |
| C: RabbitMQ | Publish adds a few ms, so latency stays within budget. Durable queues absorb the 200 vs 50/s gap and survive outages, which meets availability. Per-queue retry and dead-letter handling for MailFast failures. New actions are just new queues bound to the exchange (modifiability). Open source, runs on Kubernetes, and two developers know it (team fit) | A new stateful component to operate (ops cost). At-least-once delivery, so consumers must be idempotent to get "exactly once". Eventual consistency for email and points. Harder end-to-end tracing |
| D: Kafka | Same decoupling as C. Replayable log, useful for analytics and future consumers. Very high throughput | 200 events/s is far below the throughput where Kafka pays off. Heavier to operate (brokers, partitions, consumer offsets). No team experience. Per-message retry and delay for MailFast throttling is harder than with RabbitMQ queues |

## Decision

Choose **Option C: RabbitMQ publish/subscribe**, self-hosted on Kubernetes.

- **Producer:** after `shipOrder` succeeds, `PlaceOrder` publishes one persistent `OrderPlaced` message (`event_id` = `orderID`, email, items, totals, timestamp) to an `orders` exchange with publisher confirms. It then returns. This replaces the `sendOrderConfirmation` gRPC call and the `EMAIL_SERVICE_ADDR` dependency. A publish failure is logged and retried with a short timeout, but it never fails the order (driver 1).
- **Consumers, each with its own durable queue and dead-letter queue:**
  - **Email worker** (the rewritten `emailservice`, Go): calls MailFast. It is throttled with prefetch and a token bucket at ≤ 50 emails/s, and retries with exponential backoff on MailFast failures.
  - **`loyaltyservice`** (Go, new): awards points idempotently. It records the processed `order_id` with a unique constraint in its own store and acknowledges only after commit. At-least-once delivery plus deduplication gives the required "exactly once" effect.
  - **Analytics consumer:** forwards an "order placed" event to marketing's analytics store.
- **Future** SMS, fraud-scoring and recommendations consumers bind new queues to the same exchange, with no change to `checkoutservice`.

C is the only option that satisfies drivers 1–3 together and also makes next year's actions additive (driver 4). It is preferred over D because it matches the team's RabbitMQ experience and the modest 200 msg/s load, and because per-queue throttling and retries suit the MailFast rate limit (drivers 5–6).

## Consequences

- **Positive:** `PlaceOrder` latency drops. It loses one synchronous call and gains one local publish of a few ms, staying under the 500 ms p95 target (performance).
- **Positive:** orders succeed while MailFast, `loyaltyservice` or analytics is down, and messages wait in durable queues (availability).
- **Positive:** the 200 orders/s peak is buffered and drained at MailFast's 50/s limit without being rejected (scalability / throttling).
- **Positive:** new post-order actions are new consumers, with no checkout change or redeploy (modifiability).
- **Positive:** consumers deploy and scale separately (deployability).
- **Negative:** RabbitMQ is a new stateful component to run, monitor, back up and upgrade on Kubernetes, for a 4-person team (operational cost).
- **Negative:** emails are delayed during peaks. At 11.11 the backlog grows by about 150 emails/s, so a 10-minute peak queues approximately 90,000 emails, which takes about 30 min to drain at 50/s (usability: customers get confirmations late).
- **Negative:** consistency is eventual, and tracing must now cross the broker. Trace context must be propagated in message headers (observability).
- **Risks and mitigations:**
  - **Lost event:** the card is charged and shipped, but the publish fails or the pod crashes before the confirm. `checkoutservice` has no database, so a true transactional outbox is not possible today. Mitigation: publisher confirms plus a retry and an alert. Accept the small residual risk for now, or add an outbox store later (open question).
  - **Double points:** caused by redelivery. Mitigation: idempotent `loyaltyservice` keyed on `order_id`. Test with forced redelivery.
  - **Loyalty within 1 minute at peak:** the loyalty queue is separate from the throttled email queue, so the MailFast backlog does not delay it. Monitor loyalty queue lag (target < 60 s).
  - **MailFast p99 8 s or outage:** consumer timeouts, backoff and a dead-letter queue with replay. Alert on dead-letter queue depth.
  - **RabbitMQ unavailable:** run a 3-node cluster with quorum queues. Checkout still returns success if the publish fails (driver 1), with failed publishes logged for replay.
  - **PII in events** (email, address) reaching analytics: the analytics consumer strips fields marketing doesn't need.

## Validation

- **Load test** at 200 orders/s (e.g. with the existing Locust `loadgenerator`): `PlaceOrder` p95 ≤ 500 ms, and zero orders failed because of post-order actions.
- **Fault injection:** stop the email worker, `loyaltyservice` and analytics, or point MailFast at a stub with p99 8 s and 0.5 % errors. Orders still succeed, and after recovery every email and point is delivered.
- **Exactly-once test:** force message redelivery and consumer crashes. No order gets points twice, and 100 % of orders get points within 60 s (loyalty queue lag metric).
- **Rate-limit test:** MailFast send rate never exceeds 50/s. Measure the email backlog drain time after a simulated 11.11 peak.
- **Extensibility check:** add a stub SMS consumer with zero changes to `checkoutservice`.
- **Review** after the first 11.11 sale, or when the first new post-order action is added.

## Open questions

- Who are the deciders?
- How long does the 11.11 peak last, and what is the maximum acceptable delay for a confirmation email? This sets the queue size and decides whether a higher MailFast tier is needed.
- Is the small risk of losing an event between charging and publishing acceptable, or should `checkoutservice` get an outbox store? There is no persistence in `main.go` today.
- Does MailFast support idempotency keys? Without them, a retry after a timeout may send a duplicate email.
- Which analytics system does marketing use, and which event fields are allowed under privacy rules?
- Must fraud scoring run **before** payment (`chargeCard`)? If so, it cannot be an asynchronous `OrderPlaced` consumer and needs its own ADR.
- Where does `loyaltyservice` store points and processed order IDs (which database), and what happens to points on a refund?
- Is there in-house capacity and a budget to run a highly available RabbitMQ cluster (node sizing, backups)?
