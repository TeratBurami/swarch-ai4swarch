Agent: Claude Code. Model: Claude Opus 5.5 (`claude-opus-5-5`). Run: 2b (with context).

# ADR-0002: Publish an "OrderPlaced" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-11-01
- **Deciders:** Not given in the provided files (see Open questions)

## Context and problem

Requirement R-12 (`docs/scenario.md`): after an order is placed, Online Boutique must (1) send the confirmation email through **MailFast**, an external provider that replaces the mock `emailservice` logic, (2) award **loyalty points** through a new `loyaltyservice`, and (3) record an **"order placed" analytics event**. Marketing expects **more post-order actions next year** (SMS, fraud scoring, recommendations update).

**How it works today** (evidence: `system/online-boutique/src/checkoutservice/main.go`):

| Fact | Evidence |
|---|---|
| `checkoutservice` is written in Go and talks to every dependency over gRPC | `main.go` imports `google.golang.org/grpc`; `mustConnGRPC` (lines 210-220) |
| `PlaceOrder` calls payment, shipping, cart and email **in sequence** on the request path | `PlaceOrder` (lines 230-280): `chargeCard` (l. 252), `shipOrder` (l. 258), `emptyUserCart` (l. 263), `sendOrderConfirmation` (l. 273) |
| Email failure only logs a warning; the order still succeeds | `log.Warnf("failed to send order confirmation ...")` (l. 274) |
| No timeout is set on the email call; it uses the request `ctx` | `sendOrderConfirmation(ctx, ...)` (l. 273, 379-384) |
| The order ID is a UUID generated in memory | `uuid.NewUUID()` (l. 233) |
| `checkoutservice` has no data store of its own (only gRPC client connections) | `checkoutService` struct (lines 66-86) |
| The email dependency is configured by address | `EMAIL_SERVICE_ADDR` (l. 115) |

**Facts that constrain the decision** (`docs/scenario.md`, *Quality goals and constraints*):

| Topic | Fact | Consequence for this decision |
|---|---|---|
| Checkout latency | p95 `PlaceOrder` ≤ **500 ms**, today approximately **300 ms** | Only approximately 200 ms of headroom for all new work on the request path |
| MailFast | p95 **1.5 s**, p99 **8 s**, approximately **0.5 %** failures, rate limit **50 emails/s** | A synchronous call would push p95 to approximately 1.8 s (300 ms + 1.5 s), breaking the 500 ms goal |
| Peak load | **200 orders/s** (11.11 sale) | 200 orders/s > 50 emails/s: emails must be **buffered**; the backlog grows by 150 emails/s, i.e. **9,000 emails per minute of peak** |
| Loyalty | within **1 minute**, **exactly once** | Needs durable delivery plus de-duplication |
| Availability | an order must **never fail** because email, loyalty or analytics is down | Post-order actions must be decoupled from the order's success |
| Team | 4 devs; strong in Go and gRPC; **two have used RabbitMQ** | Favours a technology the team already knows |
| Operations | Kubernetes; prefers **open-source, self-hosted** | Rules out cloud-only managed queues |

The problem: **how should `checkoutservice` trigger post-order actions** so that checkout stays fast and available, loyalty is exactly-once within 1 minute, MailFast's rate limit is respected, and new actions can be added next year without changing checkout each time?

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics (or MailFast) is down.
2. **Performance:** p95 `PlaceOrder` stays ≤ 500 ms at 200 orders/s.
3. **Reliability / correctness:** loyalty points awarded exactly once, within 1 minute; no lost post-order work.
4. **Modifiability:** add SMS, fraud scoring and recommendations next year without editing `checkoutservice` each time.
5. **Interoperability with MailFast:** respect the 50 emails/s rate limit and tolerate its 8 s p99 and 0.5 % failure rate.
6. **Cost and operability:** open-source, self-hosted on Kubernetes; fits a 4-person team's skills.

## Considered options

1. **Extend synchronous orchestration:** `PlaceOrder` calls MailFast (via `emailservice`), `loyaltyservice` and an analytics service over gRPC on the request path, as it does today for email.
2. **In-process asynchronous calls:** `PlaceOrder` returns, then fires goroutines inside `checkoutservice` that call the three actions (no broker).
3. **Publish an `OrderPlaced` event to RabbitMQ (self-hosted):** `checkoutservice` publishes one event to a fan-out exchange; each action (email worker, `loyaltyservice`, analytics) consumes from its own durable queue.
4. **Publish an `OrderPlaced` event to Apache Kafka (self-hosted):** same publish/subscribe shape, using a replayable event log with consumer groups.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous orchestration | Simplest; same gRPC style as today (`main.go` l. 252-277); immediate result per action | MailFast p95 1.5 s / p99 8 s on the request path breaks the 500 ms p95 goal; 200 orders/s exceeds the 50 emails/s limit, so calls would be rejected or queue up inside checkout; every new action means editing `PlaceOrder` (poor modifiability); a hang in any action slows every order (no timeout today, l. 273) |
| 2. In-process async (goroutines) | No new infrastructure; checkout latency unaffected | Work is lost if a `checkoutservice` pod restarts or crashes (no store, struct l. 66-86); the 50 emails/s limit is global, hard to enforce across many checkout replicas; retries and exactly-once loyalty must be hand-built; still edits checkout for every new action |
| 3. RabbitMQ event | Checkout only does one fast, local publish; durable queues survive consumer and MailFast outages; one queue per action isolates failures; email consumer drains at ≤ 50/s, absorbing the 11.11 backlog; new actions = new queue bound to the exchange, no checkout change; open source and self-hosted; **two team members already know it** | New stateful component to run (a clustered setup for durability); delivery is at-least-once, so `loyaltyservice` must de-duplicate; the publish itself becomes a new failure point on the request path; eventual consistency (email and points arrive later) |
| 4. Kafka event | Same decoupling as option 3; durable, replayable log (useful for analytics and for back-filling future consumers such as recommendations); very high throughput | **No team experience** (scenario lists only RabbitMQ); heavier to operate on Kubernetes for a 4-person team; throughput far beyond the 200 events/s needed; per-message rate limiting and retry/dead-letter handling must be built on top; still at-least-once, so de-duplication is still needed |

## Decision

Choose **option 3: publish an `OrderPlaced` event to a self-hosted RabbitMQ**.

Design outline:

- After payment and shipping succeed, `PlaceOrder` publishes **one** `OrderPlaced` message (order ID from `uuid.NewUUID()` l. 233, user, email address, items, total) to a fan-out exchange, with persistent delivery and publisher confirms, under a short timeout. It replaces the synchronous `sendOrderConfirmation` call (l. 273-277). If the publish fails, checkout **logs and still returns success**, as email failures are handled today (l. 274), so availability is kept.
- **Email:** the `emailservice` role becomes a queue consumer that calls MailFast. It limits its calls to **≤ 50 emails/s in total** across all replicas, retries failures with backoff, and moves messages that keep failing to a dead-letter queue.
- **Loyalty:** `loyaltyservice` consumes its own queue and **de-duplicates on order ID**: it records each processed order ID atomically with the points update, then acks. Redelivered messages are ignored, which gives the effect of exactly-once on top of at-least-once delivery.
- **Analytics:** an analytics consumer records the "order placed" event from its own queue.
- **Next year:** SMS, fraud scoring and recommendations each bind a new queue to the same exchange. `checkoutservice` does not change.

Why it fits the drivers best: it is the only option that meets **availability** and **performance** (MailFast's 1.5 s / 8 s latency is off the request path) **and** survives restarts (durable queues). Unlike options 1 and 2, it enforces the **MailFast rate limit** in one place and gives **modifiability** for next year's actions. Option 4 offers the same decoupling, but RabbitMQ matches the **team's existing skills** and the modest load (200 events/s), while meeting the **open-source, self-hosted** constraint.

## Consequences

- **Positive:** `PlaceOrder` latency no longer depends on MailFast, `loyaltyservice` or analytics; only one in-cluster publish is added (quality attribute: performance). Its cost is **[Inferred]** to be a few milliseconds; not evidenced in the provided files, so it must be measured.
- **Positive:** an outage of MailFast, `loyaltyservice` or analytics only grows a queue; orders keep succeeding and the work completes when the consumer recovers (quality attribute: availability, fault isolation).
- **Positive:** the email queue absorbs the 11.11 burst (200 orders/s in, 50 emails/s out) without breaching MailFast's rate limit (quality attribute: interoperability, reliability).
- **Positive:** new post-order actions are added as new consumers without changing `checkoutservice` (quality attribute: modifiability, extensibility).
- **Positive:** open source, self-hosted on the existing Kubernetes platform, known to two developers (quality attribute: cost, operability).
- **Negative:** one more stateful component to deploy, monitor and upgrade; for durability it should run as a replicated cluster (quality attribute: operability, cost).
- **Negative:** eventual consistency: confirmation emails will be **late during peaks**. At 200 orders/s the email backlog grows by 9,000 per minute of peak and drains at ≤ 50/s afterwards (quality attribute: performance as seen by the customer, usability).
- **Negative:** end-to-end flow is harder to trace and debug across asynchronous hops (quality attribute: observability). Mitigation: carry the trace context in message headers. `checkoutservice` already propagates OpenTelemetry context (`main.go` l. 134-140).
- **Negative:** `loyaltyservice` needs a data store for de-duplication and balances. The store is not decided here (quality attribute: modifiability, cost).
- **Risks and mitigations:**
  - *Event lost between payment and publish* (pod crash, or RabbitMQ unreachable). The order succeeds but no email or points follow. Mitigation: publisher confirms plus a bounded local retry. If this loss is unacceptable for loyalty, add a **transactional outbox**, which needs a store that `checkoutservice` does not have today (struct l. 66-86). See Open questions.
  - *Duplicate delivery → double points.* Mitigation: idempotent `loyaltyservice` keyed on order ID (unique constraint), ack only after commit.
  - *Duplicate emails* on MailFast retry after a timeout. Mitigation: pass the order ID as an idempotency key if MailFast supports one (not evidenced).
  - *Loyalty lag > 1 minute* if its consumer falls behind. Mitigation: separate queue from email (so the MailFast backlog never delays points), autoscale consumers, alert on queue age.
  - *Publish adds latency or hangs checkout.* Mitigation: short publish timeout. On timeout, log and return success.
  - *Poison messages* block a queue. Mitigation: retry limit plus dead-letter queue per consumer, with an alert.
  - *RabbitMQ is a single point of failure* for post-order actions (not for orders). Mitigation: replicated queues in a multi-node cluster, and Kubernetes health checks.

## Validation

- **Load test** at **200 orders/s** (e.g. the Locust `loadgenerator` used in ADR-0001) with a MailFast stub that mimics p95 1.5 s / p99 8 s / 0.5 % errors / 50 per s. Pass if p95 `PlaceOrder` ≤ **500 ms**.
- **Loyalty SLO:** the time from order to points awarded is measured per order. Pass if **100 % within 60 s** at peak, and **0 duplicate awards** when consumers are killed mid-message and messages are redelivered.
- **Rate limit:** MailFast calls never exceed **50/s**. Pass if the stub returns zero rate-limit rejections and the email queue drains after the peak ends.
- **Fault injection:** stop MailFast, `loyaltyservice`, the analytics consumer and RabbitMQ in turn. Pass if every order still succeeds and the queued work completes after recovery (except while RabbitMQ is down, which is logged; see Open questions).
- **Monitoring in production:** queue depth and oldest-message age per queue, dead-letter counts, publish failures in `checkoutservice` logs.
- **Review** after the next 11.11 sale, using the metrics above.

## Open questions

Not given in the provided files. Not invented here:

1. **Deciders:** who approves this ADR?
2. **Email delay tolerance:** how late may a confirmation email arrive during 11.11? The backlog size depends on the **duration of the 200 orders/s peak**, which is not given.
3. **Broker-down policy:** if RabbitMQ is unreachable at publish time, is it acceptable to lose that order's email, points and analytics event? Or must we add an outbox, which needs a new store for `checkoutservice`?
4. **`loyaltyservice` data store and points rules:** which database, and how points are calculated. Not evidenced.
5. **Analytics destination:** where marketing's "order placed" events must end up (tool, store, schema, retention).
6. **MailFast API details:** does it support idempotency keys, is the 50/s limit per account or per API key, and is bursting allowed?
7. **Current `emailservice` implementation** (language, interface): not in the provided files, so the effort to turn it into a MailFast consumer is unknown.
8. **Publish latency budget:** the actual added latency of a confirmed RabbitMQ publish in this cluster is not evidenced and must be measured.
9. **Message format and versioning:** protobuf (matching the team's gRPC skills) or JSON, and how schema changes are handled for next year's consumers.
10. **Data protection:** whether customer email addresses and order details may be placed in messages consumed by analytics and future services.
11. **RabbitMQ operations:** cluster size, persistent storage and who operates it. Only the general preference for self-hosted on Kubernetes is given.
