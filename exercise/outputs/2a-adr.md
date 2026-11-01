> Agent: Claude Code (Anthropic), model: Claude Opus 5.5 (`claude-opus-5-5`). Run `a` (zero-shot, context: none; only the requirement and `docs/adr/template.md` were used).

# ADR-0002: Publish an "order placed" event to drive post-order actions

- **Status:** Proposed
- **Date:** 2026-11-01
- **Deciders:** Not evidenced in the provided files (to be confirmed; see Open questions)

## Context and problem

After an order is placed, Online Boutique must trigger three post-order actions (from the requirement):

| # | Action | Who handles it |
|---|---|---|
| 1 | Send the order confirmation email | **MailFast**, a new external email provider, replacing the mock `emailservice` logic |
| 2 | Award loyalty points | New `loyaltyservice` (to be built) |
| 3 | Record an "order placed" analytics event | For the marketing team (target system not stated) |

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update). The number of consumers of "an order was placed" will therefore grow.

The problem: **how should the step that places an order hand off to these post-order actions**, now and as more are added?

Facts not available in this zero-shot run (no system files read):

- Which service places the order today and how it calls `emailservice` (protocol, sync/async). **[Assumption]** some order-placing service calls `emailservice` directly after the order succeeds.
- Whether any message broker already exists in the system. Not evidenced in the provided files.
- Latency, throughput, availability targets, MailFast's API, rate limits and SLA. Not given (see Open questions).

## Decision drivers

Most important first (ordering is a proposal, to be confirmed with stakeholders):

1. **Modifiability / extensibility**: adding SMS, fraud scoring, recommendations next year should not require changing the order-placing code each time.
2. **Availability of order placement**: a failure or slowdown of MailFast, `loyaltyservice` or analytics must not make placing an order fail.
3. **Performance (checkout latency)**: post-order work should not add to the time the customer waits for order confirmation.
4. **Reliability of delivery**: each action should happen (eventually) for every placed order, without losing or silently dropping events; duplicates must be handled safely (e.g. points awarded twice).
5. **Integrability with an external provider**: MailFast is outside our control; its failures, rate limits and API changes should be isolated.
6. **Operational cost and complexity**: new infrastructure must be justified; team skills are unknown.

Note: fraud scoring (planned next year) may need to run *before* or *as part of* order acceptance rather than after it. That would be a different interaction pattern and is not decided here (see Open questions).

## Considered options

1. **Option A – Direct synchronous calls**: the order-placing service calls MailFast (or an email adapter), `loyaltyservice` and the analytics endpoint one after another, as part of placing the order.
2. **Option B – Publish an `OrderPlaced` event to a message broker (publish/subscribe)**: the order-placing service publishes one event; each post-order action is an independent subscriber (email adapter for MailFast, `loyaltyservice`, analytics). New actions subscribe without changing the publisher.
3. **Option C – Dedicated post-order orchestrator service**: the order-placing service makes one call to a new orchestrator, which then calls email, loyalty and analytics itself (with its own retries).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A – Direct sync calls | Simplest; no new infrastructure. Easy to trace in one call chain. Immediate error feedback. | Order-placing service is coupled to every consumer; each new action (SMS, fraud, recommendations) changes it (**modifiability**). Latency of all calls adds to checkout (**performance**). A MailFast or `loyaltyservice` outage can fail or slow the order unless every call gets custom timeout/retry/ignore logic (**availability**). Retries are ad hoc; actions can be lost if the service crashes mid-sequence (**reliability**). |
| B – `OrderPlaced` event via broker | Publisher knows nothing about consumers; new actions are new subscribers (**modifiability**). Checkout only waits for a publish, not for MailFast/loyalty/analytics (**performance**). Consumer outages do not affect ordering; the broker buffers events until consumers recover (**availability**). Per-consumer retry and dead-letter handling; MailFast is isolated behind its own subscriber (**integrability**). Analytics naturally consumes the same event stream. | New infrastructure to run, monitor and pay for (**cost**, **operability**). Eventual consistency: email/points arrive a little later and the customer is not told synchronously if they fail. At-least-once delivery means consumers must be idempotent (e.g. no double points). Publish must be reliable relative to the order being stored (risk of "order saved but event lost" unless an outbox or equivalent is used). Harder end-to-end tracing and debugging. Not suitable for actions that must block the order (e.g. if fraud scoring must approve it). |
| C – Post-order orchestrator | One place to see and change the post-order workflow; ordering and conditional steps are explicit. Order-placing service makes only one call (**modifiability** of that service). Can add retries centrally. | Orchestrator still changes for each new action, so the coupling has only moved (**modifiability**). If the call to it is synchronous, its latency and availability still sit on the checkout path; if asynchronous, it needs a queue anyway, approaching Option B (**availability**, **performance**). One more service to build and operate, and a potential single point of failure (**cost**, **availability**). |

## Decision

Choose **Option B: publish an `OrderPlaced` event to a message broker**, with post-order actions as independent subscribers:

- The order-placing service publishes one `OrderPlaced` event after the order is successfully placed, and no longer calls `emailservice` directly. **[Assumption]** about the current call; to be verified against the code.
- An email subscriber (the replacement for the mock `emailservice` logic) consumes the event and calls **MailFast**.
- The new `loyaltyservice` consumes the event and awards points.
- An analytics subscriber consumes the event and records the "order placed" event for marketing.
- Next year's SMS, recommendations update and any *post-order* fraud scoring become additional subscribers.

Why: it best satisfies the top drivers. It removes per-action changes to the order-placing code (modifiability), keeps MailFast and `loyaltyservice` failures off the checkout path (availability, performance), and gives each consumer its own retries (reliability, integrability). The extra infrastructure cost is accepted because the number of consumers is expected to grow from three to six or more.

Specific broker technology, event schema format and delivery guarantees are **not decided here**; they depend on facts not available in this run (see Open questions) and should be a follow-up ADR.

## Consequences

- **Positive:** New post-order actions are added by deploying a new subscriber, without changing the order-placing service. (quality attribute: modifiability)
- **Positive:** Checkout no longer waits on email, loyalty or analytics calls. (quality attribute: performance)
- **Positive:** An outage of MailFast, `loyaltyservice` or analytics does not fail order placement; events wait in the broker. (quality attribute: availability)
- **Positive:** MailFast integration is isolated in one subscriber; switching providers later affects only that component. (quality attribute: modifiability, integrability)
- **Negative:** A message broker must be deployed, monitored, secured and paid for. (quality attribute: cost, operability)
- **Negative:** Post-order effects are eventually consistent; emails and points may arrive with a delay, and failures are not visible to the customer at checkout. (quality attribute: usability, consistency)
- **Negative:** Flows span several asynchronous components and are harder to trace and test end to end. (quality attribute: testability, observability)
- **Risks and mitigations:**
  - *Event lost between order placement and publish* → use a transactional outbox or equivalent reliable-publish mechanism if the order-placing service has a data store; otherwise define retry-on-publish behaviour. (reliability) Whether an order data store exists is not evidenced in the provided files.
  - *Duplicate delivery (at-least-once)* → make consumers idempotent using the order ID (e.g. award points once per order ID, send one email per order ID). (reliability, correctness)
  - *MailFast outage or rate limiting* → retries with backoff and a dead-letter queue in the email subscriber; alert on dead-letter growth. (availability, integrability)
  - *Event schema changes break consumers* → version the `OrderPlaced` schema and only add optional fields. (modifiability)
  - *Personal data (e.g. customer email) in events* → include only the fields each consumer needs and restrict who may subscribe. (security, privacy)
  - *Fraud scoring may need to block orders* → treat it as out of scope for this pattern until its timing is decided. (correctness)
  - *Team unfamiliar with broker operations* → unknown; see Open questions. (cost, operability)

## Validation

- **Checkout latency:** measure order-placement latency before and after the change under the same load test; it should not increase (target value not given; see Open questions).
- **Failure isolation:** fault-injection test where MailFast, `loyaltyservice` and analytics are each unavailable; order placement must still succeed, and the actions must complete after recovery.
- **Delivery completeness:** reconcile per day the number of placed orders against confirmation emails sent, loyalty awards and analytics events recorded; the gap should be zero after retries.
- **Idempotency:** test redelivering the same `OrderPlaced` event; points and emails must not be duplicated.
- **Extensibility:** when the first of next year's actions (e.g. SMS) is added, confirm it required no change to the order-placing service.
- **Review date:** to be agreed (not given); suggested at the start of next year's post-order work.

## Open questions

Information needed but not given in the requirement (not invented here):

1. Which service places orders today, and how does it currently call `emailservice` (protocol, sync/async)?
2. Is a message broker or event-streaming platform already in use or allowed? Which technologies are permitted (cloud vendor, Kubernetes, budget)?
3. Checkout latency and availability targets (numbers), and expected order volume / peak throughput.
4. Required delivery time for the confirmation email and for loyalty points (e.g. within seconds, minutes)?
5. MailFast's API (protocol, authentication), rate limits, SLA, cost per message and failure behaviour.
6. Where should marketing analytics events go (existing analytics platform, data warehouse, third party)? Required retention and data format?
7. Does the order-placing service have its own data store (needed for a transactional outbox)?
8. Will fraud scoring run after the order (subscriber) or must it approve the order before acceptance?
9. Privacy and compliance constraints on sending customer data (email, address, purchase details) to MailFast and analytics.
10. Team skills and capacity for building `loyaltyservice` and operating a broker; who owns each subscriber?
11. Who are the deciders for this ADR, and what is the review date?
