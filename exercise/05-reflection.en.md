**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:*Ratsaporn Prathumma 6630290321*
**My role on the review board: _Architect_

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

After Parts A and B, I would trust AI to check an architecture’s consistency, compare alternatives, calculate capacity, and identify failure scenarios. I would keep business priorities, acceptable risk, ownership, and final approval with humans.

In our board meeting about the RabbitMQ ADR, AI highlighted a failure window where payment could succeed before the order was durably recorded. This helped us examine whether the outbox actually protected the whole checkout process. However, humans still needed to decide the recovery policy and who would be accountable for charged customers whose orders were missing.

AI’s weaker contribution was repeating the email backlog problem already documented in the ADR. This showed me that a technically correct observation is not always a useful new finding. I see AI as a critical reviewer whose suggestions need evaluation, while architects remain responsible for decisions and consequences.
