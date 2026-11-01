**Language:** [ภาษาไทย](bonus-recover-decision.md) | English

# Bonus (optional, take-home): Recover an unwritten decision (approximately 15 min)

> **Lecture link:** T5 (Dhar et al. generate decisions from code and diagrams to reduce *knowledge vaporization*), AICH4.
> **Question:** when AI explains *why* a system was built this way, how much of the explanation is evidence and how much is a confident guess?

## Run

Start a new chat or session in your agent and send:

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
```

Want the ADR in Thai? Add a line `language: Thai`.

The agent writes the reconstructed ADR, including its **Claim check** table, to [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md).

## Audit the claim check

Choose **five** claims from the agent's table: at least two it marked *Evidenced* and two it marked *Inferred*. Check each one yourself.

| # | Claim (short) | AI says | You say | Why (file path or reasoning) |
|---|---|---|---|---|
| 1 | `frontend` calls 7 back-end services over gRPC | Evidenced | Evidenced | `src/frontend/main.go` lines 141–147 have 7 `mustConnGRPC` calls (there's an 8th one at line 179, but that's the tracing collector) |
| 2 | Inter-service gRPC is plaintext | Evidenced | Evidenced | `insecure.NewCredentials()` in `checkoutservice/main.go` line 215 and `frontend/main.go` line 230 |
| 3 | A down dependency fails `PlaceOrder` | Evidenced | Partly wrong | True for payment and shipping, but email failure is only logged and the `EmptyCart` error is ignored (`_ = cs.emptyUserCart`). The AI even lists both of those itself in claims 31–32 |
| 4 | Other services are written in other languages | Inferred | Inferred (but true) | Can't be seen from the three files. Outside the evidence, `src/cartservice/` is C# and `src/recommendationservice/` is Python, so the guess is right |
| 5 | No message broker was used | Inferred | Inferred (but true) | Correctly not marked Evidenced, since the three files can't prove a negative. The manifests I checked in Task 1 also show no broker |

**Count:** of all the agent's claims, how many are *Evidenced* according to the agent? 20 of 38. According to you? 19 (claim 29 is too broad).

## Reflect

1. Not really a guess, but claim 29 is labelled *Evidenced* while being only half true: the code shows some failures are ignored. I checked by reading `PlaceOrder` in `checkoutservice/main.go`. Otherwise the agent was careful, and things like the decision drivers were all marked Inferred or Assumption.
2. They could think the team chose gRPC for performance or language-neutral contracts, because the drivers section reads like real reasons. None of that is in the code. The system is a demo, so the real reason may just be "to show off gRPC on Kubernetes".
3. AI can recover *what* the system does: one shared proto, unary calls, plaintext connections, the call order in checkout. It can't recover *why*, e.g. why the card is charged before shipping with no refund step. Only the original authors know if that was a conscious shortcut for a demo or something they forgot.
