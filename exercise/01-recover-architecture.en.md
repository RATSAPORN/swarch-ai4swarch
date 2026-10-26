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

| Caller → Callee                  | AI's evidence                                                                                                                                                                                                                                          | Verified? ✅ / ❌ / ❓ | What you found |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------- | -------------- |
| customer → frontend              | system/online-boutique/kubernetes-manifests/frontend.yaml (Service frontend-external: type LoadBalancer, ports[name=http] 80 → 8080)                                                                                                                   | ✅                     | -              |
| frontend → productcatalogservice | system/online-boutique/kubernetes-manifests/frontend.yaml (PRODUCT_CATALOG_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/productcatalogservice.yaml (Service port grpc); system/online-boutique/protos/demo.proto (ProductCatalogService) | ✅                     | -              |
| frontend → currencyservice       | system/online-boutique/kubernetes-manifests/frontend.yaml (CURRENCY_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/currencyservice.yaml (Service port grpc); system/online-boutique/protos/demo.proto (CurrencyService)                    | ✅                     | -              |
| frontend → cartservice           | system/online-boutique/kubernetes-manifests/frontend.yaml (CART_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/cartservice.yaml (Service port grpc); system/online-boutique/protos/demo.proto (CartService)                                | ✅                     | -              |
| frontend → recommendationservice | system/online-boutique/kubernetes-manifests/frontend.yaml (RECOMMENDATION_SERVICE_ADDR); system/online-boutique/kubernetes-manifests/recommendationservice.yaml (Service port grpc); system/online-boutique/protos/demo.proto (RecommendationService)  | ✅                     | -              |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question): …

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
   ANS. The .proto file describes service interfaces and message types, but alone it cannot show which services actually call each other, how they are deployed, or which data stores they use. Declaring an RPC does not prove a runtime dependency; that requires implementation or configuration evidence. I would need to see A and B to identify their specific differences.
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
   ANS. The most misleading mistake would be publishing an assumed service-to-service dependency as a verified connection. A new team member could rely on that arrow when assessing change impact or troubleshooting, so it should be checked against evidence or labelled
