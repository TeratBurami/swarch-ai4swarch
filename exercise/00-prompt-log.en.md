**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Terat Burami / 6970143621
**Agent(s) used:** Claude Code
**Model(s):** Claude Opus 5.5
**Plan / tier:** e.g. Copilot Student, Copilot Free, Gemini free tier: Claude Pro

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Opus 5.5 | `prompts/1-…` run A, evidence `protos/demo.proto` | `AGENTS.md`, `prompts/1-recover-architecture.md`, `protos/demo.proto` | Yes | 4 | |
| 2 | 1-B | Claude Code, Opus 5.5 | `prompts/1-…` run B, evidence `demo.proto` + `kubernetes-manifests/` | `AGENTS.md`, `prompts/1-recover-architecture.md`, `protos/demo.proto`, all 13 files in `kubernetes-manifests/` | Yes | 5 | Also tried to open `kubernetes-manifests/redis.yaml`, which doesn't exist |
| 3 | 2-a | Claude Code, Opus 5.5 | `prompts/2-…` run a, context none, R-12 pasted | `prompts/2-write-adr.md`, `docs/adr/template.md` | Yes | 5 | Did not open `docs/scenario.md` |
| 4 | 2-b | Claude Code, Opus 5.5 | `prompts/2-…` run b, R-12 pasted | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go` | Yes | 5 | |
| 5 | 3 | Claude Code, Opus 5.5 | `prompts/3-…`, adr `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md` | `AGENTS.md`, `prompts/3-review-adr.md`, `docs/adr/0002-…`, `docs/scenario.md` | Yes | 5 | Did not edit the ADR |
| 6 | bonus | Claude Code, Opus 5.5 | `prompts/bonus-…`, gRPC + shared proto decision | `prompts/bonus-recover-decision.md`, `docs/adr/template.md`, `protos/demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go` | Yes | 5 | |
