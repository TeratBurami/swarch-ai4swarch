Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review of ADR-0002: Publish an "OrderPlaced" event to RabbitMQ

Inputs read: `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md` and `docs/scenario.md`. I was not allowed to read `main.go`, so the ADR's line references (l. 233, 252-277, 66-86, 134-140) are **taken on trust and not verified**.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | Customers during the 11.11 sale | 200 `PlaceOrder` calls/s | Peak load; MailFast at p99 8 s | Order confirmed after one confirmed publish | p95 `PlaceOrder` ≤ **500 ms** (today ~300 ms, so publish plus retries must fit in **~200 ms**) |
| QA2 Availability | MailFast, `loyaltyservice`, analytics consumer, or one RabbitMQ node | Becomes unavailable for 10 min | Peak load | Orders keep succeeding; work is queued and completes after recovery | **0** orders failed because of post-order actions; **0** `OrderPlaced` events lost |
| QA3 Reliability (loyalty) | `checkoutservice` | `OrderPlaced` delivered; loyalty consumer killed mid-message and the message is redelivered | Peak load | Points awarded once | **100 %** of orders awarded within **60 s**; **0** duplicates; **0** missing |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | An event is lost if a checkout pod crashes after `chargeCard` but before the publish is confirmed. The ADR "accepts that small risk" (Risks, first bullet). | The scenario says loyalty is **exactly once**. A lost event means zero awards, which fails that requirement just as double points would. The requirement is not "small risk acceptable". |
| Risk | When RabbitMQ is down, the fallback is a log line that is replayed **by hand** after an alert. The Validation section leaves this case out ("except while RabbitMQ is down"). | Any broker outage longer than 60 s breaks the loyalty SLO. Replay from logs depends on log retention and on people, and nobody owns it. QA2 is not tested where it is weakest. |
| Risk | The fan-out exchange sends the full payload (email address, items, total) to **every** consumer, including analytics and next year's consumers. | Data protection. The ADR lists this only as Open question 10, but the decision already fixes the payload. |
| Risk | It does not say how analytics handles at-least-once delivery. | Duplicate "order placed" events inflate marketing numbers. Only loyalty is de-duplicated. |
| Sensitivity point | Publish timeout, retry count and queue type (replicated queue plus publisher confirms). None of them has a value. | p95 latency (QA1) and event loss (QA3) both depend on these. With only ~200 ms of headroom, "a couple of retries within a short timeout" is not a design. |
| Sensitivity point | Throughput of the email consumer (concurrency × MailFast latency) and the length of the peak | At 1.5 s p95, 50 emails/s needs about 75 calls in flight across all replicas. Too few and the backlog never drains. Too many and MailFast rate-limits. Email lateness grows with peak length (10 min of peak → ~30 min late, per the ADR). |
| Trade-off point | Synchronous, confirmed publish on the request path, compared with fire-and-forget | Confirms improve durability and loyalty correctness but add latency and a failure point. The log-and-succeed fallback protects availability but gives up loyalty correctness. |
| Trade-off point | RabbitMQ chosen over Kafka | Better operability and use of team skills (two developers know RabbitMQ). Worse replay ability for analytics and for back-filling future consumers such as recommendations. |
| Non-risk | One durable queue per consumer | The MailFast backlog (9,000 emails per minute of peak) cannot delay loyalty points. Failures stay isolated. |
| Non-risk | Loyalty de-duplicates on order ID, with a unique constraint and ack after commit | This is the right way to get exactly-once *processing* on at-least-once delivery. QA3 duplicates are covered. |
| Non-risk | Fraud scoring is moved out of scope because it must act before `chargeCard` | Correct. A post-order event cannot block a fraudulent charge. |

## 3. Missing or weak

- **Biased option comparison:** the ADR lists "per-message rate limiting … must be built on top" as a Kafka con. RabbitMQ has no global consumer rate limit either. The "≤ 50/s across all replicas" limit must be built in both cases, and the ADR never says how (single active consumer? a shared token bucket?).
- **Options not considered:** (a) **transactional outbox from day one**, with a small store for checkout. This is the standard way to close the publish-loss gap, and it is only postponed. (b) A **hybrid**: queue for email and analytics, but a gRPC call to `loyaltyservice` with a tight timeout and the queue as fallback.
- **Consequences not stated:** email lateness also grows with post-peak traffic. Normal orders keep arriving while the backlog drains, so "~30 min late" is optimistic. Retries of MailFast's 0.5 % failures use the same 50/s budget. Nobody owns the replay runbook.
- **Weak or unverified claims:** "a few milliseconds" for a confirmed publish to a replicated queue is **[Inferred]**, and the ADR labels it so. It is the claim QA1 depends on. "300 ms + 1.5 s ≈ 1.8 s" adds two p95 values, which is only a rough estimate, though the conclusion (option 1 fails) still holds. The code facts cannot be checked from the scenario.

## 4. Verdict: **Accept with changes**

The direction (asynchronous events, one queue per action, RabbitMQ for team fit) is sound and well argued. Three changes before approval:

1. **Close the event-loss gap for loyalty.** Either adopt an outbox (and decide its store) or replace the log replay with an automatic, durable local spool and re-publisher. Remove the RabbitMQ-down exception from Validation, or get the product owner to sign off on the loyalty-SLO breach explicitly.
2. **Put numbers on the publish path.** Give the publish timeout, retry count and queue type. Load-test p95 of a confirmed publish at 200/s and show it fits in the ~200 ms headroom.
3. **Specify the MailFast limiter and the payload.** Name the global 50/s mechanism and the consumer concurrency, fix the Kafka/RabbitMQ comparison, agree an email-delay target with the product owner, and limit what personal data goes into the fan-out message.
