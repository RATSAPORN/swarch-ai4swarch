Agent: Codex; model: exact model identifier unavailable.

# ADR-BONUS: Use synchronous gRPC and a shared demo.proto for the evidenced business calls

- **Status:** Reconstructed
- **Date:** 2026-10-26 (reconstruction date; original date: Not evidenced in the provided files)
- **Deciders:** Not evidenced in the provided files

## Context and problem

Evidence abbreviations used throughout this ADR:

- **P:** system/online-boutique/protos/demo.proto
- **C:** system/online-boutique/src/checkoutservice/main.go
- **F:** system/online-boutique/src/frontend/main.go

| ID | Observed context | Evidence |
|---|---|---|
| C1 | One proto3 file defines nine business services in package hipstershop; every declared RPC is unary request/response. | P:15–19; CartService through AdService. |
| C2 | Contracts reuse Money across product, currency, payment and order structures, and CartItem across cart, shipping and order structures. | P:Product, CurrencyConversionRequest, ChargeRequest, OrderItem, Cart, GetQuoteRequest, ShipOrderRequest. |
| C3 | Checkout registers a gRPC server, configures six downstream gRPC connections, and waits on typed client calls. | C:main (110–146); quoteShipping, getUserCart, emptyUserCart, prepOrderItems, convertCurrency, chargeCard, sendOrderConfirmation, shipOrder. |
| C4 | PlaceOrder waits for preparation, payment, shipping, cart-emptying and confirmation attempts before returning. Preparation/payment/shipping errors fail the request; cart-emptying errors are ignored and email errors are logged. | C:PlaceOrder (238–279). |
| C5 | Frontend configures seven backend gRPC connections and exposes incoming HTTP routes. Its handler bodies are absent from this file, so their call execution is not established here. | F:main (132–171). |
| C6 | “All services communicate synchronously over gRPC using one shared demo.proto” exceeds the evidence: other implementations and generation inputs are unavailable. Frontend configures a shopping-assistant address without showing its transport; both Go files also configure OTLP trace export over gRPC, whose API is not defined in P. | F:139, initTracing; C:initTracing; P:service declarations. Universal coverage and derivation of every generated client from P: Not evidenced in the provided files. |
| C7 | Original rationale, historical alternatives, workload measurements and decision makers are unknown. | Not evidenced in the provided files. |

## Decision drivers

The ordering below is **[Inferred]**, not a documented historical priority.

1. **D1 — Modifiability/interoperability:** use a common typed contract vocabulary across RPC interfaces (P:Money, CartItem and service declarations; C:typed client calls).
2. **D2 — Functional suitability/understandability:** obtain payment and shipping results directly before returning an order response (C:PlaceOrder, chargeCard, shipOrder; P:OrderResult).
3. **D3 — Maintainability:** keep business RPC definitions discoverable in one place (P:all service/message declarations).

## Considered options

These are reconstructed alternatives, not an evidenced historical shortlist.

1. **A:** Synchronous gRPC with consolidated business contracts in demo.proto.
2. **B:** Synchronous REST/JSON with separately owned service schemas.
3. **C:** Asynchronous checkout commands/events via a broker, with an order-status API.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | **O1 [Inferred]:** shared typed messages and direct responses can simplify integration and checkout orchestration (P:shared messages; C:PlaceOrder and typed clients). | **O2 [Inferred]:** dependency waits expose checkout to downstream latency/failures; shared message changes can require consumer coordination (C:PlaceOrder; P:Money, CartItem). |
| B | **O3 [Assumption]:** HTTP/JSON is inspectable with common HTTP tools; separate schemas can support local ownership. | **O4 [Assumption]:** independently specified shared concepts can drift; synchronous HTTP still waits on dependencies. |
| C | **O5 [Assumption]:** durable brokered work can buffer temporary unavailability and decouple request completion from downstream processing. | **O6 [Assumption]:** adds broker operation, correlation, duplicate handling and recovery; returning pending status changes the immediate-result interaction. |

For B and C, implementation, evaluation and historical consideration are **Not evidenced in the provided files**. Their trade-offs are general design assumptions.

## Decision

**R1 — Reconstructed observation:** use option A for the evidenced checkout business calls (P:RPC declarations; C:PlaceOrder and typed client methods). Frontend's configured gRPC connections support this pattern but do not establish its handler execution details (F:141–147).

**R2 [Inferred] — Plausible rationale:** a common typed vocabulary and direct orchestration make it straightforward to obtain the values needed for OrderResult (P:OrderResult; C:PlaceOrder). This plausibly explains the combined synchronous/shared-contract choice. Historical intent, superiority to REST, measured performance benefits, a teaching motivation and rejection of asynchronous messaging are **Not evidenced in the provided files**.

## Consequences

- **Positive — K1 [Inferred], quality attribute: modifiability.** Reusing Money and CartItem can reduce independently specified representations (P:shared message uses). A single generation pipeline or shared runtime package is not established.
- **Negative — K2 [Inferred], quality attributes: performance and availability.** Sequential blocking checkout steps can accumulate latency and propagate dependency failures (C:PlaceOrder, prepareOrderItemsAndShippingQuoteFromCart). Actual latency and availability impact: Not evidenced in the provided files.
- **Negative — K3 [Inferred], quality attribute: modifiability.** Shared message semantic changes can affect several service contracts and require coordination (P:Money, CartItem and their uses). A common file does not establish lockstep deployment.
- **Risks and mitigations — K4 [Inferred], quality attribute: reliability.** Payment precedes shipping, and a shipping failure returns an error after the successful charge; PlaceOrder shows no compensation in this branch (C:252–261). Partial completion is a plausible risk. Proposed mitigation: define and verify recovery/compensation and idempotency behavior. System-wide mechanisms: Not evidenced in the provided files.
- **Risks and mitigations — K5 [Inferred], quality attributes: availability and performance.** convertCurrency passes context.TODO() instead of its supplied context, creating a plausible cancellation/deadline propagation risk (C:359–366). Proposed mitigation: propagate caller context and verify RPC deadlines. The three-second context in mustConnGRPC is discarded; it does not establish per-RPC deadlines (C:210–216).

## Validation

**V1 [Inferred] — Proposed checks; not executed:**

- Measure checkout and downstream latency/error rates under representative load; agree acceptance thresholds before testing. No thresholds/results are evidenced.
- Inject payment, shipping and email failures to verify PlaceOrder's branches, including recovery after payment succeeds but shipping fails (C:252–279).
- Test cancellation/deadline behavior of currency conversion (C:convertCurrency).
- Review shared-message compatibility against affected consumers (P:Money, CartItem). Full consumer coverage requires evidence beyond this task.

## Claim check

P, C and F expand to the exact paths above. Rows correspond to each marked claim in the ADR, including the associated proposed options, reasons and consequences.

| # | Claim | Type | Evidence |
|---|---|---|---|
| C1 | Nine unary business services share a proto3 file/package. | Evidenced | P:15–19; CartService, RecommendationService, ProductCatalogService, ShippingService, CurrencyService, PaymentService, EmailService, CheckoutService, AdService. |
| C2 | Contracts reuse Money and CartItem across services. | Evidenced | P:Product, CurrencyConversionRequest, ChargeRequest, OrderItem, Cart, GetQuoteRequest, ShipOrderRequest. |
| C3 | Checkout exposes gRPC and waits on typed calls to six dependencies. | Evidenced | C:main (110–146), quoteShipping through shipOrder (313–394). |
| C4 | Checkout waits for its steps and handles their errors differently. | Evidenced | C:PlaceOrder (238–279). |
| C5 | Frontend configures seven backend gRPC clients and exposes HTTP; handler bodies are unavailable. | Evidenced | F:main (132–171). |
| C6 | Universal synchronous/shared-schema coverage is unproven; assistant transport is unspecified and OTLP export uses an API absent from P. | Evidenced | F:139, initTracing; C:initTracing; P:service declarations. Scope limitation, not proof of every interaction. |
| C7 | Historical rationale, alternatives, measurements and deciders are unknown. | Evidenced | No decision account or workload results in P, C or F; Not evidenced in the provided files. |
| D1 | Common typed contracts plausibly motivated interoperability/modifiability. | Inferred | P:shared messages and services; C:typed clients; motivation not stated. |
| D2 | Immediate checkout results plausibly motivated synchronous orchestration. | Inferred | C:PlaceOrder, chargeCard, shipOrder; P:OrderResult; priority not stated. |
| D3 | Consolidation plausibly favors discoverability/maintenance. | Inferred | P:consolidated definitions; intended benefit not stated. |
| O1 | Option A can simplify integration and orchestration. | Inferred | P:shared messages; C:PlaceOrder; no comparative evaluation. |
| O2 | Option A incurs dependency waits and potential schema coordination. | Inferred | C:blocking calls/error branches; P:shared message uses. |
| O3 | REST/JSON tooling and separate-schema ownership offer potential benefits. | Assumption | General alternative-design reasoning; Not evidenced in the provided files. |
| O4 | Separate schemas can drift; synchronous HTTP retains waiting costs. | Assumption | General alternative-design reasoning; Not evidenced in the provided files. |
| O5 | Durable asynchronous messaging can buffer work and decouple completion. | Assumption | General alternative-design reasoning; Not evidenced in the provided files. |
| O6 | Brokered checkout requires operational/recovery mechanisms and a changed result interaction. | Assumption | General alternative-design reasoning; Not evidenced in the provided files. |
| R1 | Option A describes the observed checkout RPCs and consolidated definitions, with limited frontend confirmation. | Evidenced | P:RPC declarations; C:PlaceOrder and typed clients; F:141–147. Generation inputs not provided. |
| R2 | Shared vocabulary and direct orchestration plausibly explain the choice. | Inferred | P:OrderResult; C:PlaceOrder support the reconstruction, not historical intent. Other claimed motivations: Not evidenced in the provided files. |
| K1 | Reused definitions can reduce duplicated representations, aiding modifiability. | Inferred | P:Money, CartItem and their uses; generation pipeline unavailable. |
| K2 | Sequential dependency waits can worsen latency and availability. | Inferred | C:PlaceOrder, prepareOrderItemsAndShippingQuoteFromCart; impact unmeasured. |
| K3 | Shared semantic changes can require coordination, affecting modifiability. | Inferred | P:shared message uses; deployment policy unavailable. |
| K4 | Payment-before-shipping creates a plausible partial-completion risk; recovery/idempotency checks are proposed. | Inferred | C:PlaceOrder (252–261); no compensation in this branch; system-wide recovery unknown. |
| K5 | Conversion's discarded caller context creates a plausible cancellation risk; context/deadline verification is proposed. | Inferred | C:convertCurrency (359–366), mustConnGRPC (210–216); impact unmeasured. |
| V1 | Load, failure, cancellation and compatibility checks could validate these implications. | Inferred | Proposed from C:PlaceOrder, convertCurrency; P:shared messages. No checks executed or thresholds evidenced. |
