Agent: Codex; Model: GPT-6

# ADR-0002: ใช้ RabbitMQ แยกงานหลังสั่งซื้อออกจาก checkout

- **Status:** Proposed
- **Date:** 2026-10-24
- **Deciders:** ยังไม่ระบุ

## Context and problem

- R-12 ต้องส่งอีเมลผ่าน MailFast แทน mock emailservice เพิ่มแต้มผ่าน loyaltyservice ใหม่ และบันทึก analytics event; ปีหน้าคาดว่าจะเพิ่ม SMS, fraud scoring และ recommendations update (`docs/scenario.md:7–14` และ requirement ของผู้ใช้)
- ปัจจุบัน PlaceOrder เรียกชำระเงิน จัดส่ง ล้างตะกร้า และส่งอีเมลก่อนตอบกลับ โดย email error เป็นเพียง warning และ order ยังสำเร็จ (`system/online-boutique/src/checkoutservice/main.go:252–279`, `PlaceOrder`) การส่งอีเมลใช้ gRPC (`system/online-boutique/src/checkoutservice/main.go:379–383`, `sendOrderConfirmation`)
- p95 checkout ต้อง ≤ 500 ms จากปัจจุบันประมาณ 300 ms; พีค 200 orders/s ส่วน MailFast มี p95 1.5 s, p99 8 s, failure ประมาณ 0.5% และ quota 50 emails/s (`docs/scenario.md:24–26`) **[Inferred]** การรอ MailFast เสี่ยงเกินเป้าหมาย latency; ต้องทดสอบจริง ไม่บวกค่า percentile เพื่อรับประกันผล
- แต้มต้องเข้าภายใน 1 นาทีและไม่ซ้ำ; order ต้องไม่ล้มเหลวเพราะ email, loyalty หรือ analytics ล่ม (`docs/scenario.md:27–28`) ทีมมี 4 คน ถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; ใช้ Kubernetes และชอบ open-source แบบ self-hosted (`docs/scenario.md:29–30`)
- durable order store, outbox, broker และฐานข้อมูลแต้ม: Not evidenced in the provided files. PlaceOrder สร้าง UUID และ OrderResult แต่ไม่พบ transaction บันทึก order/event (`system/online-boutique/src/checkoutservice/main.go:233–235,265–279`) Redis ใน ADR-0001 เป็น cart store ที่ยอมรับความทนทานจำกัด จึงไม่ใช่หลักฐานว่ามี durable order store (`docs/adr/0001-use-redis-for-cart-storage.md:20,38–45`)

## Decision drivers

1. **Availability:** ความขัดข้องของงานหลังสั่งซื้อต้องไม่ทำให้ order ล้มเหลว (`docs/scenario.md:28`)
2. **Performance:** p95 checkout ≤ 500 ms ที่พีค 200 orders/s (`docs/scenario.md:24–25`)
3. **Correctness และ timeliness:** แต้มหนึ่งครั้งต่อ order ภายใน 1 นาที (`docs/scenario.md:27`)
4. **Modifiability:** เพิ่มงานหลังสั่งซื้อได้ง่าย (`docs/scenario.md:14`)
5. **Operability และ cost:** เหมาะกับทีม 4 คน ทักษะที่มี และ self-hosted Kubernetes (`docs/scenario.md:29–30`)

## Considered options

ทุกตัวเลือกเป็นข้อเสนอใหม่ ไม่ใช่สิ่งที่พบว่าติดตั้งแล้ว

1. เรียกทุกปลายทางแบบ synchronous จาก checkout พร้อม timeout และยอมให้บางงานล้มเหลว
2. บันทึกรายการงานถาวร ให้ worker กลางเรียก API ของแต่ละปลายทางและติดตามสถานะ/retry
3. บันทึก order พร้อม outbox แล้วกระจาย OrderPlaced ผ่าน RabbitMQ ไปยัง consumer แยก

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. synchronous | โครงสร้างตรงไปตรงมา ใช้ทักษะ Go/gRPC เดิม (`docs/scenario.md:29`) | ต้องรอปลายทาง; timeout แล้วข้ามงานไม่รับประกันว่าทำครบ; เพิ่มงานต้องแก้ checkout (วิเคราะห์จาก `docs/scenario.md:14,24–28`) |
| 2. worker กลาง | แยก latency ปลายทางออกจาก checkout; ติดตาม workflow และ retry ที่จุดเดียว | ต้องพัฒนา scheduling และ recovery; เพิ่มงานต้องแก้ worker กลาง; ต้องแยก concurrency เพื่อไม่ให้อีเมลกีดขวางแต้ม (การประเมินข้อเสนอ) |
| 3. RabbitMQ | แยก backlog และกำลังประมวลผลแต่ละงาน; เพิ่ม subscriber ได้; ทีมบางคนมีประสบการณ์ (`docs/scenario.md:14,29`) | เพิ่ม broker, outbox และ storage; ต้องรับมือข้อความซ้ำและ eventual consistency; เพิ่มภาระทีมเล็ก (การประเมินข้อเสนอ; `docs/scenario.md:29–30`) |

## Decision

เลือก **ตัวเลือก 3: RabbitMQ แบบ self-hosted บน Kubernetes พร้อม durable outbox และ consumer แยก** เพื่อรองรับงานที่เพิ่มขึ้นและสอดคล้องกับทักษะ/ข้อจำกัด (`docs/scenario.md:14,29–30`) รายละเอียดต่อไปนี้เป็นการออกแบบที่เสนอ:

- กำหนดจุดยอมรับ order หลังชำระเงินและจัดส่งสำเร็จตามลำดับปัจจุบัน (`system/online-boutique/src/checkoutservice/main.go:252–271`) เพิ่ม durable transactional store แล้วบันทึกสถานะ order และ outbox event ใน transaction เดียว; ตอบสำเร็จหลัง commit โดยไม่รอ broker/consumer ห้ามใช้ goroutine หรือ publish แบบ best-effort เป็นหลักประกันการส่งครบ
- relay ส่ง OrderPlaced เข้า RabbitMQ exchange พร้อม durable queue แยกสำหรับอีเมล, loyalty และ analytics; เสนอ persistent messages, publisher confirms และ replicated quorum queues ต้องได้รับ confirm และตรวจว่า routing มี queue รับก่อนทำเครื่องหมาย outbox ว่าส่งแล้ว การ confirm ไม่ได้แปลว่างานธุรกิจเสร็จ
- consumer อีเมลแทน mock logic ด้วย MailFast adapter และยกเลิก synchronous call เดิมที่ `system/online-boutique/src/checkoutservice/main.go:273–277,379–383` จำกัดการส่งรวมทุก replica รวม retry ≤ 50 emails/s (`docs/scenario.md:26`) protocol และ idempotency ของ MailFast: Not evidenced in the provided files.
- loyaltyservice มี ledger ถาวร ใช้ unique key ตาม `order_id + action_type` โดยบันทึกการประมวลผลและเพิ่มแต้มใน transaction เดียวแล้วจึง ack ใช้ at-least-once delivery พร้อม deduplication เพื่อให้ผลการเพิ่มแต้มเป็น exactly-once ไม่อ้างว่า broker ส่ง exactly-once
- consumer analytics บันทึก event ถาวรแล้วจึง ack และ deduplicate ตาม event ID; ยังไม่เลือกผลิตภัณฑ์จัดเก็บ แต่ละ consumer มี retry/backoff และ dead-letter queue แยก พร้อมเจ้าของตรวจสอบและ replay โดยคง ID เดิม
- event มี ID คงที่, order_id, เวลาเกิดเหตุการณ์, schema version และข้อมูลขั้นต่ำที่จำเป็น; ไม่ใส่ข้อมูลบัตรเครดิต จำกัดสิทธิ์และ retention ข้อมูลลูกค้า เพิ่มงานปีหน้าด้วย subscriber ใหม่และ schema ที่รองรับผู้ใช้เดิม fraud scoring ในที่นี้เป็นงานหลัง order ตาม requirement; หากต้องบล็อกการชำระเงินต้องพิจารณาใหม่
- เมื่อ broker/consumer ล่ม ให้ outbox เก็บงานรอ แต่ durable store เป็น dependency ใหม่ หากบันทึก order/event ไม่ได้ จะรับประกันทั้งตอบสำเร็จและส่งงานครบไม่ได้ ต้องตกลง failure policy ก่อนยอมรับ ADR ห้าม log แล้วทิ้ง event ทั้งนี้ outbox ไม่ทำให้ payment/shipping เป็น transaction เดียวกับฐานข้อมูล ต้องกำหนด recovery และ idempotency สำหรับขั้นตอนเหล่านั้นเพิ่มเติม

## Consequences

- **Positive:** แยกเวลารอและความขัดข้องของปลายทางออกจาก checkout (คุณลักษณะ: performance, availability); เพิ่มงานผ่าน subscriber (คุณลักษณะ: modifiability) เป็นผลที่คาดหวังจากข้อเสนอ ต้องยืนยันด้วยการทดสอบ
- **Positive:** ledger และ deduplication ป้องกันแต้มซ้ำเมื่อ replay หรือ crash หลัง commit ก่อน ack (คุณลักษณะ: correctness)
- **Negative:** เพิ่ม storage, broker, relay และงานดูแลแก่ทีม 4 คน; ผลงานหลังสั่งซื้อจะปรากฏภายหลัง response (คุณลักษณะ: cost, operability, consistency; ขนาดทีมจาก `docs/scenario.md:29`)
- **Risks and mitigations:** **[Inferred]** ที่ 200 orders/s แต่ส่งอีเมลได้สูงสุด 50/s backlog โตอย่างน้อย 150 รายการ/s ก่อนคิด retry คิวไม่เพิ่ม quota ต้องกำหนดระยะพีค พื้นที่เก็บ เวลาส่งที่ยอมรับ หรือขอเพิ่ม quota (คุณลักษณะ: capacity, timeliness; คำนวณจาก `docs/scenario.md:25–26`)
- **Risks and mitigations:** แต้มภายใน 1 นาทีรับประกันไม่ได้เมื่อ loyalty หรือโครงสร้างส่งงานล่มเกินเวลานั้น แยกคิว/กำลังประมวลผลและแจ้งเตือนอายุงาน แต่ต้องตกลง SLA ขณะล่ม การ retry ไม่แก้ข้อขัดแย้งนี้ (คุณลักษณะ: availability, timeliness; `docs/scenario.md:27–28`)
- **Risks and mitigations:** ข้อความซ้ำ ข้อความเสีย และ MailFast timeout หลังรับคำขอ อาจทำให้ส่งซ้ำ ใช้ deduplication, dead-letter และ replay พร้อมตรวจสอบผลคลุมเครือ การป้องกันอีเมลซ้ำขึ้นกับ API ที่ยังไม่มีหลักฐาน (คุณลักษณะ: reliability, correctness)
- **Risks and mitigations:** outbox เต็มหรือ storage ล่มอาจกระทบ checkout; วัดพื้นที่คงเหลือ ทดสอบ recovery และกำหนดนโยบายเมื่อพื้นที่หมด (คุณลักษณะ: durability, availability)

## Validation

แผนตรวจรับที่เสนอ ยังไม่มีผลทดสอบ:

- ทดสอบ 200 orders/s ตามระยะพีคที่ตกลง วัด p95 PlaceOrder ≤ 500 ms รวม commit outbox จำลอง MailFast p95 1.5 s, p99 8 s, failure 0.5% และ quota 50 emails/s ตรวจอัตรารวมทุก replica รวม retry และเวลาระบาย backlog (`docs/scenario.md:24–26`)
- ปิด email, loyalty, analytics และ broker ทีละส่วน ตรวจ order ยังสำเร็จเมื่อ durable store ใช้งานได้ เปิดกลับแล้วตรวจว่างานไม่สูญหาย วัด outbox age, queue age, retry, dead-letter และพื้นที่คงเหลือ (`docs/scenario.md:28`)
- ส่ง event ซ้ำ ทำ loyalty consumer ล่มหลัง commit ก่อน ack และ replay; ตรวจแต้มหนึ่งครั้งต่อ order และ analytics ไม่ซ้ำ ตรวจทุก order ได้แต้มภายใน 60 วินาทีในเงื่อนไขความพร้อมที่ตกลง ไม่ใช้เพียง p95 แทนข้อกำหนดทุก order (`docs/scenario.md:27`)
- ทำ relay ล่มก่อน/หลัง confirm และจำลองไม่มี queue รับ ตรวจ recovery; ทำ checkout ล่มรอบ payment, shipping และ commit เพื่อตรวจว่าไม่เก็บเงิน/จัดส่งซ้ำ และไม่สูญเสียงานที่ยอมรับไว้ ประเมิน failure window ก่อนอนุมัติ
- ทดลองเพิ่ม consumer งานใหม่โดยไม่แก้รายการเรียกใน checkout และตรวจ schema compatibility (`docs/scenario.md:14`)

## Open questions

- ใครตัดสินใจ ดูแล broker/storage และทบทวน ADR เมื่อใด?
- นิยาม order สำเร็จและวิธีกู้คืนเมื่อชำระเงินแล้วแต่ shipping หรือ commit ล้มเหลวคืออะไร? API รองรับ idempotency/reconciliation หรือไม่?
- เลือก transactional store ใดสำหรับ order/outbox และ ledger? นโยบายเมื่อ storage ล่ม/เต็ม, RPO/RTO และ retention คืออะไร?
- SLA แต้ม 1 นาทีครอบคลุมเหตุล่มใดบ้าง? สูตรแต้มและ identity สมาชิกคืออะไร?
- พีค 200 orders/s นานเท่าใด อีเมลรอได้สูงสุดเท่าใด และเพิ่ม MailFast quota ได้หรือไม่?
- MailFast ใช้ protocol/authentication อะไร รองรับ idempotency key และตรวจสถานะหลัง timeout หรือไม่?
- analytics ต้องเก็บที่ใด มี schema, retention, ความสด และข้อกำหนดข้อมูลส่วนบุคคลอย่างไร? subscriber ใหม่ต้อง replay ประวัติหรือไม่?
- งบประมาณ ทรัพยากร Kubernetes และความพร้อมดูแล quorum queues/backup มีเท่าใด?
