Agent: Codex; Model: GPT-6

## Quality-attribute scenarios

| คุณลักษณะ | Source · Stimulus · Environment · Response · Response measure |
|---|---|
| Performance | ลูกค้า · ส่งคำขอ PlaceOrder · ช่วง 11.11 ที่ 200 orders/s และ MailFast p95 1.5 s, p99 8 s · checkout บันทึก order/outbox แล้วตอบโดยแยกงานอีเมล · p95 PlaceOrder ≤ 500 ms รวม commit; ระยะพีคยังต้องกำหนด (docs/scenario.md:24–26; docs/adr/0002-myadr.md:45,67) |
| Availability | ปลายทางงานหลังสั่งซื้อ · email, loyalty หรือ analytics หยุดทำงาน · ระหว่างรับ order · checkout ยังสำเร็จและเก็บงานรอ · จำนวน order ที่ล้มเหลวเพราะสามปลายทางนี้ต้องเป็น 0; ระยะล่มที่รองรับยังไม่ระบุ (docs/scenario.md:28; docs/adr/0002-myadr.md:51,68) |
| Correctness / timeliness | ระบบส่งงาน · ส่ง event ซ้ำหรือ consumer ล่มหลังเพิ่มแต้มก่อน ack · ระหว่างประมวลผลและ replay · loyaltyservice กู้คืนโดยไม่เพิ่มแต้มซ้ำ · ทุก order ได้แต้มหนึ่งครั้งภายใน 60 วินาที; ข้อกำหนดเดิมไม่ยกเว้นช่วงล่ม (docs/scenario.md:27; docs/adr/0002-myadr.md:48,59,69) |

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Risk | บันทึก order/outbox หลัง payment และ shipping; ADR ยอมรับว่ายังไม่มี recovery ข้ามขั้นตอน (docs/adr/0002-myadr.md:45,51,76) | **[Inferred]** เมื่อ commit ล้มเหลว อาจชำระเงินแล้วแต่ไม่มีงานหลังสั่งซื้อ; retry อาจทำผลข้างเคียงซ้ำ กระทบ correctness และ recoverability |
| Non-risk | เสนอ ledger ที่เพิ่มแต้มและบันทึก deduplication ใน transaction เดียวก่อน ack (docs/adr/0002-myadr.md:48) | เป็นแนวทางที่สอดคล้องกับข้อห้ามแต้มซ้ำ แม้ส่งซ้ำหลัง crash; ยังต้องยืนยันด้วยการทดสอบที่ ADR ระบุ (docs/scenario.md:27; docs/adr/0002-myadr.md:69) |
| Sensitivity point | หากหนึ่ง order ต้องส่งหนึ่งอีเมล ที่พีค 200/s และ quota 50/s งานรอเพิ่มอย่างน้อย 150 รายการ/s ก่อนคิด retry (docs/scenario.md:10,25–26; docs/adr/0002-myadr.md:58) | ระยะพีคและอัตราหลังพีคกำหนดพื้นที่เก็บและเวลารอ; เพิ่ม consumer อย่างเดียวไม่เพิ่ม quota กระทบ capacity และ timeliness |
| Risk | แผนตรวจรับแต้มจำกัดเป็น “เงื่อนไขความพร้อมที่ตกลง” แต่ scenario กำหนดภายใน 1 นาทีโดยไม่มีข้อยกเว้น (docs/adr/0002-myadr.md:59,69; docs/scenario.md:27–28) | ยังอ้างว่าสนอง requirement ไม่ได้; retry หลังล่มเกิน 60 วินาทีไม่ทำให้ผ่าน timeliness ย้อนหลัง |
| Trade-off point | ตอบสำเร็จหลัง durable commit โดยไม่รอ broker แต่เพิ่ม storage ในเส้นทาง checkout (docs/adr/0002-myadr.md:45,51,61) | แยกความขัดข้องของปลายทางเพื่อ availability แต่ commit เพิ่ม latency และ storage ล่ม/เต็มอาจขัดขวางการรับ order; ต้องวัด performance และกำหนด failure policy |
| Trade-off point | แยก consumer ผ่าน RabbitMQ พร้อม quorum queues และ outbox (docs/adr/0002-myadr.md:39,43–49,57) | เพิ่ม modifiability และแยก backlog แลกกับ cost และ operability สำหรับทีม 4 คน; ประสบการณ์ RabbitMQ ของ 2 คนยังไม่ยืนยันกำลังดูแลระบบ (docs/scenario.md:14,29–30) |

## Missing or weak

- **ตัวเลือก:** มีสามแนวทางต่างกันแล้ว แต่ยังไม่เทียบต้นทุนและกำลังกู้คืนของ worker กลางกับ RabbitMQ ภายใต้โหลดเดียวกัน จึงยังตัดสินความเหมาะสมต่อทีมเล็กได้ไม่ครบ (docs/adr/0002-myadr.md:29–39; docs/scenario.md:25,29)
- **ผลกระทบ:** ต้องเลือก store, กำหนด recovery หลัง payment/shipping, ขนาด outbox, ระยะเก็บ และผู้ดูแล; ADR ระบุเป็นคำถาม แต่ยังไม่มีคำตอบที่ใช้ตรวจรับได้ (docs/adr/0002-myadr.md:75–82)
- **หลักฐาน:** ข้ออ้างเรื่องโค้ด, UUID, การล้างตะกร้า และ Redis อ้างไฟล์นอกขอบเขตการทบทวน จึงตรวจยืนยันไม่ได้จากสองไฟล์ที่อนุญาต; scenario ยืนยันได้เพียงลำดับ payment → shipping → email ผ่าน gRPC และ email ล้มเหลวแล้ว order ยังสำเร็จ (docs/adr/0002-myadr.md:12,15; docs/scenario.md:18)
- **ข้ออ้างที่ต้องพิสูจน์:** latency, recovery และผลแต้ม exactly-once ยังเป็นข้อเสนอ ไม่มีผลทดสอบ; MailFast protocol/idempotency และขนาดทรัพยากร: Not evidenced in the provided files (docs/adr/0002-myadr.md:47–48,65–70,80,82)

## Verdict

**accept with changes — ยอมรับโดยต้องแก้ไข** แนวทางแยกงานสอดคล้องกับ R-12 แต่ต้องปิดสามประเด็นก่อนอนุมัติ:

1. ระบุ store, จุดยอมรับ order และ recovery/idempotency เมื่อ payment, shipping หรือ commit ล้มเหลว พร้อมผลทดสอบจุดขัดข้อง (docs/adr/0002-myadr.md:45,51,70)
2. ตกลงความหมายของแต้มภายใน 60 วินาทีระหว่างเหตุล่มกับเจ้าของ requirement; ห้ามลดเงื่อนไขตรวจรับเอง และทดสอบทุก order รวมกรณีซ้ำ/replay (docs/scenario.md:27–28; docs/adr/0002-myadr.md:59,69)
3. กำหนดระยะพีค เวลารออีเมลที่ยอมรับ ความจุและนโยบายเมื่อเต็ม พร้อมผู้รับผิดชอบ; แสดงผลทดสอบ 200 orders/s ว่า p95 checkout ≤ 500 ms และอัตราส่งรวม retry ≤ 50 emails/s (docs/scenario.md:24–26; docs/adr/0002-myadr.md:58,61,67,75)
