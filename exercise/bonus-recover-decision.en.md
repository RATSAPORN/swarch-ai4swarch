**Language:** [ภาษาไทย](bonus-recover-decision.md) | English

# Bonus (optional, take-home): Recover an unwritten decision (approximately 15 min)

> **Lecture link:** T5 (Dhar et al. generate decisions from code and diagrams to reduce _knowledge vaporization_), AICH4.
> **Question:** when AI explains _why_ a system was built this way, how much of the explanation is evidence and how much is a confident guess?

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

Choose **five** claims from the agent's table: at least two it marked _Evidenced_ and two it marked _Inferred_. Check each one yourself.

| #   | Claim (short)                                                         | AI says   | You say          | Why (file path or reasoning)                                                                                                                                                                              |
| --- | --------------------------------------------------------------------- | --------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | C1: Nine unary business services share one proto file/package.        | Evidenced | Evidenced — LGTM | `system/online-boutique/protos/demo.proto`: nine `service` declarations, package `hipstershop`; no RPC declares `stream`.                                                                                 |
| 2   | C4: Checkout waits for its steps and handles errors differently.      | Evidenced | Evidenced — LGTM | `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder` (238–279): sequential calls; early errors stop execution, cart-emptying errors are ignored, email errors are logged.                   |
| 3   | C5: Frontend configures seven backend gRPC clients and exposes HTTP.  | Evidenced | Evidenced — LGTM | `system/online-boutique/src/frontend/main.go`: seven `mustConnGRPC` calls (141–147), HTTP routes (150–163), and `http.ListenAndServe` (171).                                                              |
| 4   | K3: Shared semantic changes can require coordination among consumers. | Inferred  | Inferred — LGTM  | `system/online-boutique/protos/demo.proto` reuses `Money` and `CartItem` across contracts. Coordination is plausible, but no coordination policy is shown.                                                |
| 5   | K4: Payment-before-shipping creates a partial-completion risk.        | Inferred  | Inferred — LGTM  | `system/online-boutique/src/checkoutservice/main.go` (252–261): payment succeeds before shipping; shipping failure returns an error without compensation in this branch. System-wide recovery is unknown. |

**Count:** of all the agent's claims, how many are _Evidenced_ according to the agent? **8 of 24 According to you? **7 of 24

## Reflect

1. Did the agent ever label a guess as _Evidenced_? Give an example, or explain how you checked that it didn't.
   ANS. One “Evidenced” label was too strong. C7 classified the absence of historical rationale as Evidenced. The files support the narrower statement “Not evidenced in the provided files”; they cannot establish that the rationale was never documented elsewhere. For the checked implementation claims, I counted service declarations and traced calls and error branches directly.

2. A new developer reads only this reconstructed ADR. What wrong belief could they form?
   ANS. A developer could mistake the reconstructed rationale for the original decision. They might believe the architects deliberately chose gRPC and one shared schema to simplify integration after comparing REST and asynchronous messaging. Those reasons and alternatives were reconstructed, not documented. They could also overgeneralize checkout’s blocking calls to every service interaction.
3. What can AI recover from code, and what can only the original architects know? Use one example from your ADR.

   ANS. AI can recover implementation structure and behavior; code alone usually cannot establish historical intent. For example, checkoutservice/main.go shows payment occurring before shipping, with an error returned if shipping fails. AI can identify a plausible partial-completion risk from that sequence. Why the architects accepted that risk—and whether deadlines, teaching goals, or another constraint influenced them—requires original decision records or testimony from the people involved.
