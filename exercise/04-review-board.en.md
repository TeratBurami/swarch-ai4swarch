**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | Terat Burami (the ADR's author) | defends the ADR |
| **AI analyst** | Terat Burami (role-play) | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Terat Burami (role-play) | role card P (business side) |
| **Site reliability engineer (SRE)** | Terat Burami (role-play) | role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md` by Terat Burami
**Board members and roles:** The class was online, so the board was a role-play instead of a live meeting.

**Decision:** accept with changes

**Required changes:**
1. **Show points on the confirmation page.** Marketing already promised customers they'd see their points "the moment" they order. So checkout makes a short gRPC call to `loyaltyservice` (tight timeout) to award and show the points. If that call fails or times out, the order still succeeds and the award goes through the queue instead, using the same order-ID de-duplication. Email and analytics stay fully async.
2. **Be ops-ready before 11.11.** Before go-live: dashboards, alerts on queue age and dead-letter count, backups and a runbook (about 1 week). Ask for the 3-node budget now. If it isn't approved in time, run one RabbitMQ node with a persistent volume for 11.11 and write that down as an accepted risk.
3. **Close the loyalty gaps.** Define what happens to points when an order is cancelled or refunded. Make replaying unpublished events a scripted, owned step in the runbook, not something done by hand from logs.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Confirmation page must show the new points immediately, but the ADR awards them "within 1 minute" | PO | No (though its "hybrid" option fits the fix) | No. The campaign promise is only on the PO's card, not in the scenario |
| 2 | Events can be lost if a pod crashes after payment or RabbitMQ is down, which breaks exactly-once | AI review, SRE agreed | Yes, its top risk | Yes. It follows from the ADR and the "exactly once" rule in the scenario |
| 3 | Nobody has run a broker in production; on-call is 2 people; a new stateful component needs dashboards, alerts, backups and a runbook (~1 week) | SRE | Partly: it said nobody owns the replay runbook | No. The scenario only says two devs "have used RabbitMQ", which sounds like real experience |
| 4 | 11.11 is in 3 weeks, and HA broker budget approval takes 2 weeks | PO and SRE | No | No. Deadline and budget process aren't in any file it read |
| 5 | Last year MailFast-style email outage went unnoticed for 20 min until customers complained | SRE | No, but the ADR already plans alerts on queue age | No, it's history only the SRE knows |
| 6 | What happens to points when an order is cancelled or refunded? | PO | No | Partly. Refunds aren't in the scenario, but a reviewer could have asked |
| 7 | Customer data (email, items) goes to every consumer, including analytics | AI review | Yes | Yes. It's visible from the ADR's message design |
| 8 | Fraud scoring must be able to cancel an order before it's confirmed | PO | Yes, marked as a non-risk because the ADR already moved it out | Partly. The ADR said so, but the business rule itself is on the PO's card |
| 9 | Tracing one order across the queue when a customer says "I never got my email" | SRE | No | Yes. The ADR mentions trace headers, so the AI could have checked that part |

**The AI's best finding** (one the humans had missed): customer data being sent to every consumer through the fan-out exchange. Nobody on our board had the security card, so without the AI this wouldn't have come up.
**The AI's weakest finding** (generic, wrong or irrelevant): pointing out that "300 ms + 1.5 s" adds two p95 values. It's technically true, but it admitted the conclusion doesn't change, so it didn't matter.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
