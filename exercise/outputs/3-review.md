> Reviewer: Claude Code (model: Claude Opus 5.5, `claude-opus-5-5`)

# Review: ADR-0002 Publish an "order placed" event to RabbitMQ (via a transactional outbox)

ไฟล์ที่ใช้: `docs/adr/0002-publish-order-placed-event-rabbitmq.md` และ `docs/scenario.md` เท่านั้น

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 (Performance) | ลูกค้า | สั่งซื้อ 200 orders/s | ช่วง peak 11.11, MailFast p95 1.5 s | `PlaceOrder` commit order + `outbox` แล้วตอบทันที ไม่รอ MailFast/broker | p95 ของ `PlaceOrder` ≤ 500 ms (การเขียน DB ต้องอยู่ใน budget ~200 ms ที่เหลือจาก ~300 ms) |
| QA2 (Availability) | MailFast และ `loyaltyservice` | ล่มทั้งคู่ระหว่างที่มี order เข้า | load ปกติจนถึง peak | order ยังสำเร็จ, event ค้างใน queue, ส่งต่อเมื่อ consumer กลับมา | order ที่ fail เพราะ email/loyalty/analytics = 0; event หายหลัง recovery = 0 |
| QA3 (Correctness) | RabbitMQ | redeliver `OrderPlaced` (consumer ตายหลังให้แต้มแต่ก่อน ack) | peak 200 orders/s | `loyaltyservice` dedup ด้วย `order_id` | points ต่อ `order_id` = 1 เสมอ; order → points ≤ 60 s |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Outbox ครอบเฉพาะ DB ↔ broker แต่ไม่ครอบ `chargeCard` ↔ DB: ถ้า charge สำเร็จแล้ว transaction เขียน order/`outbox` fail (DB ล่ม/timeout) ลูกค้าถูกตัดเงินแต่ไม่มี order และไม่มี event | ข้อความ "order สำเร็จ ⇔ event ถูกบันทึก" ถูก แต่ "ถูกตัดเงิน ⇔ ได้อีเมลและแต้ม" ยังไม่จริง ซึ่งเป็นเหตุผลหลักที่ ADR ใช้อ้างว่าต้องมี outbox |
| Risk | ADR ขัดกันเอง: Decision บอก `PlaceOrder` ไม่คุยกับ broker แต่ Risks เขียนว่า "`PlaceOrder` ไม่ block รอ broker เกิน timeout" และ Consequences บอก latency ที่เพิ่ม "มีแค่การ publish ไป broker" ทั้งที่จริงคือการเขียน DB | Board ไม่รู้ว่า design จริงคือแบบไหน และ latency ที่ต้องวัดใน QA1 ผิดตัว |
| Risk | Backlog อีเมลโต 150 ข้อความ/s แต่ไม่มีตัวเลข: เช่น peak 1 ชั่วโมง = 540,000 ข้อความ ต้องใช้ ≥ 3 ชั่วโมงที่ 50/s เพื่อระบายหลัง peak **[Inferred: ระยะ peak สมมติ]** | อีเมลยืนยันอาจช้าหลายชั่วโมง และกำหนดขนาด disk/memory ของ RabbitMQ; ยังไม่มี SLA อีเมลให้เทียบ |
| Risk | ไม่ระบุว่าจะทำ rate limit 50/s "รวมทุก replica" อย่างไร (single active consumer, prefetch, shared token bucket) | ถ้า scale `emailservice` แล้วเกิน 50/s จะโดน MailFast ปฏิเสธ → retry → ยิ่งเพิ่ม backlog |
| Sensitivity point | Concurrency ของ email consumer: ที่ 50/s และ p99 8 s ต้องมี in-flight ได้ถึง ~400 calls (p95 1.5 s ≈ 75) | ถ้า prefetch/worker น้อยไป throughput จริงต่ำกว่า 50/s มาก แม้ rate limit จะยอม |
| Sensitivity point | Polling interval ของ outbox relay และ latency การเขียน DB | ตัวแรกกำหนดว่า loyalty ทัน 60 s หรือไม่ ตัวหลังกำหนดว่า p95 ≤ 500 ms หรือไม่; ADR ไม่ให้ค่าเป้าหมายทั้งสองตัว |
| Trade-off point | Order DB + outbox: reliability (ไม่มี event หาย) ↑ แต่ availability (DB เข้า critical path), performance (เพิ่ม write) และ operability (stateful component ที่สอง สำหรับทีม 4 คน) ↓ | เป็นการตัดสินใจที่ใหญ่กว่าการเลือก broker แต่ไม่มี option เทียบ (ดูข้อ 3) |
| Trade-off point | Async fan-out: performance/availability ของ checkout ↑ แต่ความสดของอีเมล (user-perceived) และ observability ↓ | ADR ระบุไว้แล้ว ถือว่ายอมรับได้ถ้ามี SLA อีเมล |
| Non-risk | แยก queue ต่อ consumer ทำให้ loyalty ไม่ติด backlog ของ MailFast | รองรับ QA3 (≤ 60 s) ช่วง peak ได้ตรงจุด |
| Non-risk | เลือก RabbitMQ แทน Kafka ที่ 200 events/s | ตรงกับทักษะทีม (2 คนเคยใช้) และ constraint open-source self-hosted; ได้ DLQ/retry ต่อ message มาเลย |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:** (a) CDC (เช่น Debezium) แทน polling relay; (b) เขียน order เป็นสถานะ `PENDING` ก่อน `chargeCard` แล้วอัปเดต + `outbox` หลัง ship (ปิดช่องในข้อ Risk แรก); (c) publish ตรงไป RabbitMQ พร้อม publisher confirms + local retry โดยไม่มี DB แล้วยอมรับ risk ที่วัดได้ — ไม่มีการเทียบว่า outbox คุ้มกับ stateful component ที่สองหรือไม่
- **Consequence ที่ไม่ได้ระบุ:** อีเมลซ้ำเมื่อ retry (at-least-once + ไม่รู้ว่า MailFast มี idempotency key) — อยู่ใน Open questions แต่ไม่อยู่ใน Consequences; ผลต่อ `emptyUserCart` ว่าอยู่ก่อนหรือหลัง transaction
- **Claim ที่อ่อน:** "เพิ่ม action ใหม่โดยไม่แก้ `checkoutservice`" จริงเฉพาะเมื่อ `OrderPlaced` มีข้อมูลพอ — SMS ต้องมีเบอร์โทร ซึ่งไม่อยู่ใน payload (`order_id`, email, items, total) จึงต้องแก้ทั้ง schema และ `checkoutservice`
- **Exactly once ของ loyalty:** ต้องระบุว่า insert dedup key กับการบวกแต้มอยู่ใน transaction เดียวกัน และ ack หลัง commit เท่านั้น — ตอนนี้เขียนแค่ "dedup ด้วย `order_id`"
- **Quorum queues / HA ของ RabbitMQ** ติด **[Inferred]** และไม่มีค่าเป้าหมาย availability ของ order DB

## 4. Verdict: **Accept with changes**

1. **ปิดช่องว่าง `chargeCard` ↔ order DB:** เขียน order `PENDING` ก่อน charge (หรือกำหนด reconciliation/refund เมื่อ commit fail) และวัดใน fault-injection test ว่า "ถูกตัดเงินแต่ไม่มี event" = 0
2. **แก้ความขัดแย้งภายในและใส่ตัวเลข:** ตัด "timeout รอ broker" ออก, แก้ Consequence เรื่อง latency เป็นการเขียน DB, กำหนดเป้า DB write p95 และ relay interval (เช่น ≤ 1 s) ให้ผูกกับ QA1/QA3
3. **ออกแบบ email pipeline ให้ชัด:** กลไก rate limit 50/s รวมทุก replica, concurrency/prefetch สำหรับ p99 8 s, ขนาด backlog จากระยะ peak และ SLA อีเมลที่ยอมรับได้ พร้อมเพิ่มเบอร์โทร/field ที่ action ปีหน้าต้องใช้ใน schema `OrderPlaced`
