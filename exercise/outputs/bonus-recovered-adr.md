> Written by: Claude Code (agent), model Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-R001: Use synchronous gRPC with one shared `demo.proto` for all inter-service communication

- **Status:** Reconstructed
- **Date:** Original decision date not evidenced in the provided files (file headers show copyright 2018 for `src/*/main.go` and 2020 for `protos/demo.proto`). Reconstructed 2026-11-01.
- **Deciders:** Not evidenced in the provided files (copyright holder is Google LLC, see file headers).

Evidence base for this reconstruction (only these files were read):
`system/online-boutique/protos/demo.proto`, `system/online-boutique/src/checkoutservice/main.go`, `system/online-boutique/src/frontend/main.go`.

## Context and problem

Facts visible in the evidence:

- Online Boutique is split into separate services. `demo.proto` declares 9 gRPC services in one file and one package `hipstershop`: `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService`, `AdService`, plus shared message types such as `Money`, `Address`, `CartItem`, `Empty`.
- Message types are reused across service boundaries: `Money` is used by product catalog, shipping, currency, payment, email and checkout messages; `CartItem` by cart, shipping and email; `Address` by shipping, email and checkout (`demo.proto`).
- Every RPC is unary request/response (no `stream` keyword anywhere in `demo.proto`).
- `frontend` serves end users over HTTP (`frontend/main.go`, `http.ListenAndServe`, `mux.NewRouter`) and calls 7 back-end services over gRPC (`frontend/main.go`, `mustConnGRPC` for currency, product catalog, cart, recommendation, shipping, checkout, ad).
- `checkoutservice` is itself a gRPC server (`checkoutservice/main.go`, `pb.RegisterCheckoutServiceServer`, port `5050`) and a gRPC client of 6 services (`mustConnGRPC` for shipping, product catalog, cart, currency, email, payment).
- `PlaceOrder` orchestrates a long chain of blocking calls: `GetCart` → for each item `GetProduct` + `Convert` → `GetQuote` → `Convert` → `Charge` → `ShipOrder` → `EmptyCart` → `SendOrderConfirmation` (`checkoutservice/main.go`, `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `prepOrderItems`).
- Service addresses come from environment variables (`mustMapEnv`, e.g. `CART_SERVICE_ADDR`), and connections use plaintext transport (`insecure.NewCredentials()` in `mustConnGRPC`, both files).
- gRPC calls are instrumented with OpenTelemetry (`otelgrpc.NewClientHandler`, `otelgrpc.NewServerHandler`) and the checkout service exposes the standard gRPC health service (`healthpb.RegisterHealthServer`).
- Each service compiles its own copy of generated code from the proto (`checkoutservice/main.go` import `.../src/checkoutservice/genproto`), and Go services do not share packages (comment in `initProfiling`: "duplicated in other microservices using Go since they are not sharing packages").
- One exception: `frontend` reads `SHOPPING_ASSISTANT_SERVICE_ADDR` but does **not** open a gRPC connection for it (`frontend/main.go`, field `shoppingAssistantSvcAddr` has no `*grpc.ClientConn`). How it is called is not evidenced in the provided files.

Problem: how should the services talk to each other, and how should their interfaces be defined and kept consistent?

Not evidenced in the provided files: the implementation language of the other services, any load/latency targets, whether a message broker exists anywhere, and the authors' stated reasons.

## Decision drivers

Reconstructed, most important first. All of these are **[Inferred]** from the shape of the code, not stated anywhere:

- **Simplicity / understandability** of the call graph (request/response is easy to follow and trace).
- **Interoperability across services** with one strictly typed, language-neutral contract (generated stubs per service).
- **Performance** (binary encoding, persistent HTTP/2 connections) for a chatty call graph where one checkout fans out to many calls.
- **Observability**: end-to-end distributed tracing of a single user request (OpenTelemetry handlers are wired into every gRPC client/server, evidenced).
- **Consistency** of shared domain types (`Money`, `Address`, `CartItem`) across all services.

## Considered options

1. **A. Synchronous unary gRPC, all services defined in one shared `demo.proto`** (what the code shows).
2. **B. Synchronous REST/JSON over HTTP**, each service with its own OpenAPI spec.
3. **C. Synchronous gRPC with one `.proto` file per service** (shared types in a small common proto).
4. **D. Hybrid: gRPC for queries + asynchronous messaging through a message broker** (e.g. RabbitMQ/Kafka) for the post-payment steps (`ShipOrder`, `EmptyCart`, `SendOrderConfirmation`).

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A. Sync gRPC, one shared proto | One place shows the whole system's API; shared types (`Money`, `Address`) defined once; strong typing and generated stubs; compact binary encoding; built-in health checking and OTel interceptors; simple request/response mental model | Every service rebuilds when any part of the file changes; no ownership boundary per service; temporal coupling: a slow/failed callee directly fails or slows the caller; long synchronous chains add latency; browsers cannot call gRPC directly, so an HTTP frontend is needed |
| B. Sync REST/JSON | Human-readable, easy to debug with curl; browser-native; very widely known | Larger text payloads, slower serialization; contracts less strictly enforced unless OpenAPI is maintained; still has temporal coupling; no shared type definitions by default |
| C. Sync gRPC, per-service protos | Same runtime benefits as A; clear ownership and independent versioning per service; a service only compiles what it uses | More files and import paths to manage; shared types need a separate common proto (still a coupling point); still has temporal coupling |
| D. gRPC + async broker for side effects | Checkout no longer blocks on shipping/email/cart cleanup; failures in email or shipping can be retried from the queue; better availability and lower checkout latency | Extra infrastructure to run and operate (broker); eventual consistency; harder end-to-end tracing and debugging; more design work (idempotency, retries, dead letters) |

## Decision

Chosen option: **A. All services communicate synchronously over unary gRPC, with a single shared `demo.proto` as the system-wide contract.**

Reconstructed rationale (**[Inferred]**, the provided files do not state the reasons):

- One file gives a single, readable overview of every service interface and reuses the core domain types (`Money`, `Address`, `CartItem`) without a separate common-types package.
- Synchronous request/response keeps the call flow linear and easy to understand and trace, which suits a system whose purpose appears to be demonstrating microservices and their observability (OTel handlers and health service are wired in as a standard pattern in both files). The "demo" purpose itself is suggested only by the file name `demo.proto` and the Go module path `microservices-demo`.
- gRPC generates typed stubs per service (`genproto` in each service), which lets separate services (possibly in different languages, not evidenced) share one contract.
- A broker (option D) was not needed for the demo's goals and would add infrastructure; no broker appears in the provided files.

## Consequences

- **Positive:** One typed contract for all services; changes to a message are caught at compile time in every service that regenerates stubs (quality attribute: modifiability / interoperability).
- **Positive:** Every inter-service call is traced with the same OpenTelemetry gRPC handlers, giving end-to-end traces of a request (quality attribute: observability / testability).
- **Positive:** Uniform health checking via the standard `grpc.health.v1` service (quality attribute: availability, through orchestrator-driven restarts **[Inferred]**).
- **Positive:** Compact binary encoding and reused connections (`*grpc.ClientConn` created once at startup) (quality attribute: performance).
- **Negative:** Checkout latency is the sum of all sequential calls; product lookup and currency conversion run once per cart item in a loop (`prepOrderItems`) (quality attribute: performance / scalability).
- **Negative:** Temporal coupling: if cart, product catalog, currency, shipping or payment is down, `PlaceOrder` fails (quality attribute: availability).
- **Negative:** No compensation after partial failure: the card is charged (`chargeCard`) before `ShipOrder`; if shipping fails the call returns `codes.Unavailable` with no refund step; the `EmptyCart` error is ignored (`_ = cs.emptyUserCart(...)`) (quality attribute: reliability / data consistency).
- **Negative:** One shared proto file couples all services at build time; any edit touches every service's generated code (quality attribute: modifiability / independent deployability).
- **Negative:** Plaintext gRPC (`insecure.NewCredentials()`), including `CreditCardInfo` sent to payment service (quality attribute: security).
- **Risks and mitigations:**
  - Cascading slowness: `convertCurrency` uses `context.TODO()` instead of the request `ctx`, so it carries no caller deadline or cancellation (and, **[Inferred]**, the trace context of the request is not propagated on that call). Mitigation: pass `ctx` and set per-call deadlines; add retries with backoff **[Assumption: standard practice]**.
  - Email failure is already tolerated (logged as a warning only, `PlaceOrder`), which is a small move toward graceful degradation. Mitigation for the rest: move side effects after payment (ship, email, empty cart) to async messaging (option D) or add a saga/compensation for payment.
  - Contract coupling: split `demo.proto` per service (option C) if teams need independent release cycles.
  - Security: enable TLS/mTLS between services (e.g. via a service mesh **[Assumption]**).

## Validation

- Trace a `PlaceOrder` request end to end (OTel collector at `COLLECTOR_SERVICE_ADDR`) and measure p95/p99 checkout latency versus cart size.
- Fault-injection test: stop shipping, currency or payment service and observe checkout behaviour (expect failed orders; check for charged-but-not-shipped cases).
- Review the proto change history to see how often a change in one service's messages forced rebuilds of unrelated services (not evidenced in the provided files).
- Review date: Not evidenced in the provided files.

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | All service interfaces are defined in one file, package `hipstershop` | Evidenced | `protos/demo.proto` (`package hipstershop`, 9 `service` blocks) |
| 2 | Shared message types (`Money`, `Address`, `CartItem`) are reused across services | Evidenced | `protos/demo.proto` (`message Money`, used in `Product`, `GetQuoteResponse`, `ChargeRequest`, `OrderItem`) |
| 3 | All RPCs are unary request/response (no streaming) | Evidenced | `protos/demo.proto` (no `stream` keyword in any `rpc`) |
| 4 | `frontend` serves users over HTTP | Evidenced | `src/frontend/main.go` (`http.ListenAndServe`, `mux.NewRouter`) |
| 5 | `frontend` calls 7 back-end services over gRPC | Evidenced | `src/frontend/main.go` (`main`, `mustConnGRPC` calls) |
| 6 | `checkoutservice` is a gRPC server and calls 6 services over gRPC | Evidenced | `src/checkoutservice/main.go` (`pb.RegisterCheckoutServiceServer`, `mustConnGRPC` calls) |
| 7 | "All services" communicate over gRPC | Inferred | Only frontend and checkout are shown; other services' code was not in the evidence; the shopping assistant has no gRPC conn (`src/frontend/main.go`, `shoppingAssistantSvcAddr`) |
| 8 | `PlaceOrder` makes a blocking sequential chain of calls | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`) |
| 9 | Product lookup and currency conversion run once per cart item | Evidenced | `src/checkoutservice/main.go` (`prepOrderItems` loop) |
| 10 | Service addresses come from env vars | Evidenced | both `main.go` (`mustMapEnv`) |
| 11 | Inter-service gRPC is plaintext | Evidenced | both `main.go` (`mustConnGRPC`, `insecure.NewCredentials()`) |
| 12 | gRPC calls are traced with OpenTelemetry | Evidenced | both `main.go` (`otelgrpc.NewClientHandler`); checkout (`otelgrpc.NewServerHandler`) |
| 13 | Checkout exposes the gRPC health service | Evidenced | `src/checkoutservice/main.go` (`healthpb.RegisterHealthServer`) |
| 14 | Each service has its own generated stubs; Go services do not share packages | Evidenced | `src/checkoutservice/main.go` (import `.../checkoutservice/genproto`; comment in `initProfiling`) |
| 15 | Other services are written in other languages (polyglot) | Inferred | Not evidenced in the provided files |
| 16 | Original date and deciders | Inferred | Only copyright headers (Google LLC, 2018/2020) are visible; actual date/deciders not evidenced |
| 17 | Decision driver: simplicity of the call flow | Inferred | No stated rationale in the evidence |
| 18 | Decision driver: interoperability via typed, language-neutral contract | Inferred | Supported by gRPC/genproto use; motivation not stated |
| 19 | Decision driver: performance of binary encoding / HTTP/2 | Assumption | General knowledge about gRPC; no performance data in the evidence |
| 20 | Decision driver: observability of a single request | Inferred | OTel wiring is evidenced; that it drove the choice is not |
| 21 | System's purpose is a demo, which favours simplicity over resilience | Inferred | Suggested by names `demo.proto` and module `microservices-demo` only |
| 22 | No message broker was used | Inferred | No broker in the 3 files read; the rest of the system was not read |
| 23 | Option B (REST) pros/cons | Assumption | General industry knowledge |
| 24 | Option C (per-service protos) pros/cons | Assumption | General industry knowledge |
| 25 | Option D (async broker) pros/cons | Assumption | General industry knowledge |
| 26 | Option A cons: build-time coupling of one shared file | Inferred | Follows from one file + per-service `genproto`; not measured |
| 27 | Option A con: browsers cannot call gRPC directly | Assumption | General knowledge; frontend translating HTTP to gRPC is evidenced (`src/frontend/main.go`) |
| 28 | Consequence: checkout latency is the sum of sequential calls | Inferred | Sequential code is evidenced; no latency measurements |
| 29 | Consequence: a down dependency fails `PlaceOrder` | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder` returns `codes.Internal`/`codes.Unavailable` on errors) |
| 30 | Consequence: card charged before shipping, no compensation | Evidenced | `src/checkoutservice/main.go` (`chargeCard` then `shipOrder`; no refund call) |
| 31 | Consequence: `EmptyCart` error ignored | Evidenced | `src/checkoutservice/main.go` (`_ = cs.emptyUserCart(ctx, req.UserId)`) |
| 32 | Consequence: email failure only logged | Evidenced | `src/checkoutservice/main.go` (`PlaceOrder`, `log.Warnf("failed to send order confirmation ...")`) |
| 33 | Consequence: credit card data sent over plaintext gRPC | Evidenced | `protos/demo.proto` (`CreditCardInfo`); `src/checkoutservice/main.go` (`chargeCard`, `insecure.NewCredentials()`) |
| 34 | `convertCurrency` ignores caller deadline/cancellation | Evidenced | `src/checkoutservice/main.go` (`convertCurrency` uses `context.TODO()`) |
| 35 | Trace context is lost on the `Convert` call | Inferred | Follows from `context.TODO()`; not observed in a trace |
| 36 | Health service enables orchestrator restarts | Inferred | Orchestrator config not in the evidence |
| 37 | Connections are created once and reused | Evidenced | both `main.go` (`*grpc.ClientConn` fields set in `main`) |
| 38 | Mitigations (deadlines, retries, saga, TLS/mesh, split proto) | Assumption | General industry practice |
