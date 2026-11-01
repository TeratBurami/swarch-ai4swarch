**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It didn't have them, so it listed them as open questions instead | Yes, all of them: 500 ms p95, 200 orders/s, MailFast 1.5 s / 8 s / 0.5 % / 50 per s, loyalty within 1 min. It also worked out that the email backlog grows by 9,000 per minute at peak |
| Options considered (list them) | A: direct sync calls, B: event via a message broker, C: an orchestrator service | 1: sync gRPC like today, 2: goroutines inside checkout, 3: RabbitMQ event, 4: Kafka event |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Only in general terms: "make consumers idempotent using the order ID" and "retry with backoff". No rate limit, because it didn't know MailFast has one | Yes. `loyaltyservice` de-duplicates on order ID and acks after commit; the email consumer caps MailFast calls at 50/s across replicas, with a dead-letter queue |
| Anything invented (a number, tool or fact not in the scenario)? | Nothing invented. It labelled the one guess about `emailservice` as [Assumption]. But it didn't pick a broker, so the decision is too vague to act on | Nothing invented that I found. The `main.go` line numbers I checked are right. The "few milliseconds" publish cost is marked [Inferred]. It does reuse the Locust `loadgenerator` idea from ADR-0001 |

Run a was reasonable, but generic: it could fit almost any shop. Run b was much more specific, because the numbers and team skills made RabbitMQ the clear choice over Kafka.

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Took fraud scoring out of the "just add a new queue next year" list and said it needs its own ADR | A fraud check is only useful before the card is charged. Running it after the order is placed can't stop anything, so the AI's claim that every future action is just a new subscriber was wrong for this one |
| 2 | Changed what happens when the publish to RabbitMQ fails: retry, then log the full event and alert so it can be replayed. I accepted the small remaining risk for v1 | The draft just logged and moved on, which means points can be lost and breaks "exactly once". A full outbox needs a new database for checkout, which felt like too much for now |
| 3 | Added an example of how late emails get at peak (10-minute peak → about 90,000 emails, last ones roughly 30 min late) and said the PO must accept it or we ask MailFast for a higher limit | The draft said "emails will be late" but not how late. A number makes it a real trade-off someone has to sign off on |
| 4 | Filled in Deciders, removed the agent line at the top | It's my ADR now |
