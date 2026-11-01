**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| `frontend` → `cartservice` | `frontend.yaml` (`CART_SERVICE_ADDR`) | ✅ | `CART_SERVICE_ADDR=cartservice:7070` is in `frontend.yaml` line 73 |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | Env var is at line 63, and `main.go` really calls `SendOrderConfirmation` (line 380) |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | Env var at line 61, and `chargeCard` calls `Charge` in `main.go` (line 370) |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | Env var at line 60. `recommendation_server.py` also creates a `ProductCatalogServiceStub` (line 136) |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR`) | ✅ | `REDIS_ADDR=redis-cart:6379` at line 50, and `Startup.cs` reads `REDIS_ADDR` (line 29) |
| `loadgenerator` → `frontend` | `loadgenerator.yaml` (`FRONTEND_ADDR`) | ❓ | The dependency is real, but `loadgenerator.yaml` is commented out in `kustomization.yaml`, so it isn't deployed by default. It's a test tool, not really part of the system |

- **Wrong or missing** in Run B:
  - Missing: `frontend` → `shoppingassistantservice`. The AI saw `SHOPPING_ASSISTANT_SERVICE_ADDR` in `frontend.yaml` but left it out because no manifest deploys it. The service does exist in `src/shoppingassistantservice/`, and the frontend calls it over **HTTP**, not gRPC (`src/frontend/handlers.go` line 464). So it's an optional part, but not a fake one.
  - Questionable: `loadgenerator` is drawn inside the system boundary even though it's optional and only generates test traffic.
  - Not wrong, but the env vars only show that a service *can* reach another. They don't show which calls are made or in what order. For example, checkout calls payment, then shipping, then email in sequence (`main.go` `PlaceOrder`), which the diagram can't show.
  - Languages are all "not evidenced". The source folders show a mix (e.g. Go for checkout, Python for recommendation, C# for cart), which the manifests alone couldn't tell.

## Reflect (2–3 sentences each)

1. From the `.proto` file alone the agent could only list the 9 services and what each one offers. It couldn't tell who calls whom, that there is a Redis store for the cart, or that the frontend is the entry point, because a proto file describes interfaces, not how they're wired together. The manifests filled that gap through the `*_ADDR` env vars.
2. Leaving out `shoppingassistantservice` would mislead a new team member most. They would think the frontend only talks to gRPC services, when it also makes an HTTP call to a service that only runs when an optional component is turned on. Someone debugging the frontend could easily miss it.
