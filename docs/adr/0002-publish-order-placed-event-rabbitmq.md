> ร่างแรกมาจาก `exercise/outputs/2b-adr.md` (Claude Code) แก้ไขและตัดสินใจขั้นสุดท้ายโดยสถาปนิก บุญชัย จินาพร (6632110421)

# ADR-0002: Publish an "order placed" event to RabbitMQ (via a transactional outbox) for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ทีมพัฒนา 4 คน (`docs/scenario.md`, Team) — ผู้อนุมัติขั้นสุดท้าย: ดู Open questions

## Context and problem

Requirement R-12 (`docs/scenario.md`): หลังจากสั่งซื้อสำเร็จ ระบบต้อง (1) ส่งอีเมลยืนยันผ่าน **MailFast** แทน logic mock ของ `emailservice`, (2) ให้ loyalty points ผ่าน `loyaltyservice` ใหม่, (3) บันทึก analytics event "order placed" และฝ่าย marketing คาดว่าจะมี post-order action เพิ่มอีกในปีหน้า (SMS, fraud scoring, recommendations update)

**ระบบปัจจุบัน (หลักฐานจากโค้ด):**

| ข้อเท็จจริง | หลักฐาน |
|---|---|
| `PlaceOrder` ทำงานตามลำดับแบบ synchronous: ดึง cart/quote → `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` | `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`, บรรทัด 230–280) |
| ถ้าส่งอีเมลล้มเหลว จะแค่ log warning และคำสั่งซื้อยังสำเร็จ | `main.go` บรรทัด 273–277 (`log.Warnf("failed to send order confirmation ...")`) |
| การเรียก email เป็น gRPC ตรงไปยัง `emailservice` โดยใช้ `ctx` เดียวกับ request (ไม่มี timeout แยก) | `main.go` (`sendOrderConfirmation`, บรรทัด 379–384) |
| `checkoutservice` ต้องรู้ address ของทุก dependency ผ่าน env var และเชื่อมต่อตอน start | `main.go` (`EMAIL_SERVICE_ADDR` บรรทัด 115, `mustConnGRPC` บรรทัด 118–123) |
| มี order ID ที่ไม่ซ้ำกันสร้างไว้ตั้งแต่ต้นของ `PlaceOrder` | `main.go` บรรทัด 233 (`uuid.NewUUID()`) |
| `checkoutservice` ไม่มีการเชื่อมต่อฐานข้อมูลของตัวเอง (จึงยังไม่มีที่เก็บ order และทำ outbox ไม่ได้ถ้าไม่เพิ่ม DB) | `main.go` (struct `checkoutService` บรรทัด 66–86 มีแต่ gRPC connections) |

**ตัวเลขที่สำคัญ (`docs/scenario.md`, Quality goals and constraints):**

- p95 ของ `PlaceOrder` ต้อง **≤ 500 ms** (ปัจจุบันประมาณ 300 ms) → เหลือ budget ราว **200 ms**
- Peak **200 orders/s** ช่วง 11.11
- MailFast: p95 **1.5 s**, p99 **8 s**, fail ประมาณ **0.5 %**, rate limit **50 emails/s**
- Loyalty points: ต้องให้ **ภายใน 1 นาที** และ **exactly once**
- Order ต้อง **ไม่ fail** เพราะ email, loyalty หรือ analytics ล่ม
- ทีมเก่ง Go และ gRPC; 2 คนเคยใช้ **RabbitMQ**; ทุกอย่างรันบน Kubernetes; ชอบ component แบบ **open-source, self-hosted**

**ปัญหา:** ถ้าเพิ่ม action ทั้งสามเข้าไปใน `PlaceOrder` แบบ synchronous เหมือนเดิม การเรียก MailFast อย่างเดียว (p95 1.5 s) ก็เกิน latency budget แล้ว และ peak 200 orders/s สูงกว่า rate limit ของ MailFast (50 emails/s) ถึง 4 เท่า เราต้องเลือกว่า `checkoutservice` จะสั่งงาน post-order action อย่างไร

**ขอบเขต:** ADR นี้ครอบคลุมเฉพาะ post-order action ที่ทำ*หลัง*ยืนยัน order แล้ว (email, loyalty, analytics และ SMS / recommendations update ในอนาคต) **fraud scoring อยู่นอกขอบเขต** เพราะถ้าต้อง block order ก่อนยืนยัน จะเป็นขั้นตอน synchronous ใน critical path ของ `PlaceOrder` ซึ่งต้องมี ADR แยก

## Decision drivers

1. **Availability:** order ต้องไม่ fail เพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s
3. **Correctness (reliability):** loyalty points ต้องได้ exactly once ภายใน 1 นาที; อีเมลต้องไม่หายแม้ MailFast fail 0.5 % หรือติด rate limit 50/s
4. **Modifiability:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ `checkoutservice` ทุกครั้ง
5. **Team fit และ operations:** ใช้ทักษะที่ทีมมี (Go, gRPC, RabbitMQ) และเป็น open-source self-hosted บน Kubernetes

## Considered options

1. **Synchronous gRPC fan-out ใน `PlaceOrder`** — เรียก MailFast (ผ่าน `emailservice`), `loyaltyservice` และ analytics ตามลำดับเหมือนโค้ดปัจจุบัน
2. **Asynchronous in-process (goroutine) ใน `checkoutservice`** — ตอบ client ก่อน แล้วเรียก action ทั้งหมดใน background goroutine พร้อม retry ใน memory
3. **Event-driven ผ่าน RabbitMQ (self-hosted)** — `checkoutservice` publish event `OrderPlaced` ไปยัง exchange เดียว แต่ละ consumer (email worker → MailFast, `loyaltyservice`, analytics) มี queue ของตัวเอง
4. **Event-driven ผ่าน Apache Kafka (self-hosted)** — แนวคิดเดียวกับข้อ 3 แต่ใช้ Kafka topic และ consumer groups

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC fan-out | ง่ายที่สุด ใช้แพทเทิร์นเดิมใน `main.go`; ไม่มี component ใหม่; ทีมถนัด gRPC | MailFast p95 1.5 s ทำให้ p95 เกิน 500 ms แน่นอน; 200 orders/s ชน rate limit 50/s; ทุก action ใหม่ต้องแก้ `checkoutservice` (coupling สูง); retry ไม่มีที่เก็บ |
| 2. Goroutine async | latency ของ `PlaceOrder` ไม่เพิ่ม; ไม่ต้องมี infrastructure ใหม่ | งานใน memory หายเมื่อ pod restart/scale down (อีเมล/points หาย); ทำ rate limit 50/s ข้ามหลาย replica ได้ยาก; exactly once ทำไม่ได้จริง; ยังต้องแก้ `checkoutservice` ทุกครั้งที่เพิ่ม action |
| 3. RabbitMQ events | decouple: เพิ่ม action ใหม่ = เพิ่ม queue/consumer ใหม่โดยไม่แก้ `checkoutservice`; queue เป็น buffer ระหว่าง 200/s กับ 50/s ของ MailFast; durable queue + ack + retry/DLQ; open-source self-hosted บน Kubernetes; 2 คนในทีมเคยใช้ | component ใหม่ที่ต้อง operate (cluster, monitoring); ให้แค่ at-least-once จึงต้องทำ idempotency เอง; publish หลัง charge card มีปัญหา dual-write จึงต้องเพิ่ม order DB + transactional outbox (ดู Decision); อีเมลอาจล่าช้าช่วง peak |
| 4. Kafka events | throughput สูง; เก็บ event ไว้ replay ได้ (เหมาะกับ analytics/recommendations); open-source | ไม่มีใครในทีมเคยใช้; operate ยากกว่า (หลาย broker, partition); per-message retry/DLQ และ rate limit ต่อ consumer ต้องสร้างเอง; ที่ 200 events/s ความสามารถเกินความจำเป็น **[Inferred]** |

## Decision

เลือก **Option 3: publish event `OrderPlaced` ไปยัง RabbitMQ (self-hosted บน Kubernetes)**

- **เพิ่ม order DB ให้ `checkoutservice` และใช้ transactional outbox:** หลัง `shipOrder` สำเร็จ `PlaceOrder` จะเขียน order record และข้อความ `OrderPlaced` (มี `order_id` จาก `uuid.NewUUID()`, email, items, total) ลงตาราง `outbox` ใน **transaction เดียวกัน** แทนการเรียก `sendOrderConfirmation` ตรง (แทน `main.go` บรรทัด 273–277) แล้วคืนผลสำเร็จทันที `PlaceOrder` ไม่ได้คุยกับ broker โดยตรง
- **Outbox relay:** process แยก (worker ใน `checkoutservice` หรือ deployment แยก) อ่านแถวที่ยังไม่ส่งจาก `outbox` แล้ว publish ไปยัง exchange แบบ fanout/topic ของ RabbitMQ ด้วย publisher confirms และ mark ว่าส่งแล้วเมื่อได้ confirm ถ้า broker ล่ม ข้อความยังอยู่ใน `outbox` และจะถูกส่งเมื่อ broker กลับมา (relay ส่งซ้ำได้ จึงเป็น at-least-once)
- order DB เลือกจากผลิตภัณฑ์ open-source self-hosted บน Kubernetes ตาม constraint ของบริษัท (ตัวผลิตภัณฑ์: ดู Open questions)
- **Email:** `emailservice` เปลี่ยนเป็น consumer ที่เรียก MailFast โดยจำกัด rate ≤ 50 emails/s รวมทุก replica, retry แบบ exponential backoff และส่งไป dead-letter queue เมื่อ retry หมด
- **Email ช่วง peak:** ยอมรับว่าอีเมลยืนยันจะล่าช้าช่วง 11.11 เป็น baseline (queue เป็น buffer) **และ** เป็น action item ก่อน 11.11: เจรจาขอเพิ่ม rate limit กับ MailFast ช่วง peak ถ้าได้ limit ใหม่ให้ปรับค่า rate limit ของ consumer ตาม
- **Loyalty:** `loyaltyservice` consume จาก queue ของตัวเอง และทำ **idempotent consumer** โดยใช้ `order_id` เป็น unique key (at-least-once delivery + dedup = ผลแบบ exactly once)
- **Analytics:** consumer แยกที่ส่ง event ไปยังปลายทางของ marketing
- Action ใหม่ในปีหน้า (SMS, recommendations update) = bind queue ใหม่เข้ากับ exchange เดิม ส่วน fraud scoring ไม่ใช่ consumer ของ `OrderPlaced` (นอกขอบเขต ทำ ADR แยก)

**เหตุผล:** เป็นตัวเลือกเดียวที่ผ่าน driver 1–4 พร้อมกัน — เอา MailFast ออกจาก critical path (performance), order ไม่ขึ้นกับ consumer (availability), queue แบบ durable รับ backlog ระหว่าง 200/s กับ 50/s และรองรับ retry (reliability), และเพิ่ม action โดยไม่แตะ `checkoutservice` (modifiability) RabbitMQ ดีกว่า Kafka ในบริบทนี้เพราะทีมมีประสบการณ์และตรงกับ constraint open-source self-hosted (driver 5)

**ทำไมต้องมี outbox ตั้งแต่รอบนี้:** ถ้า publish ตรงหลัง `chargeCard` แล้ว broker ล่มหรือ pod ตาย ลูกค้าถูกตัดเงินแต่ event หาย จึงไม่ได้อีเมลและแต้ม ซึ่งขัดกับ driver 3 (loyalty ต้องได้แต้มแน่นอน) การ log อย่างเดียวรับไม่ได้ และการให้ order fail เมื่อ broker ล่มก็ขัดกับ driver 1 outbox ทำให้ "order สำเร็จ ⇔ event ถูกบันทึก" เสมอ แลกกับการมี DB เพิ่ม

## Consequences

- **Positive:** MailFast latency (p95 1.5 s, p99 8 s) ไม่อยู่ใน `PlaceOrder` อีกต่อไป; latency ที่เพิ่มมีแค่การ publish ไป broker ใน cluster (quality attribute: performance)
- **Positive:** email, loyalty หรือ analytics ล่ม ไม่ทำให้ order fail; ข้อความรอใน queue จนกว่า consumer จะกลับมา (quality attribute: availability)
- **Positive:** เพิ่ม post-order action ใหม่ได้โดยไม่แก้หรือ redeploy `checkoutservice` และไม่ต้องเพิ่ม `*_SERVICE_ADDR` ใหม่ (quality attribute: modifiability)
- **Positive:** queue ดูดซับ burst; MailFast ถูกเรียกไม่เกิน rate limit (quality attribute: scalability / reliability)
- **Negative:** อีเมลยืนยันจะล่าช้าช่วง peak: เข้ามา 200/s แต่ส่งได้ 50/s → backlog โต **150 ข้อความ/s** ตลอดช่วง peak (ระยะเวลา peak ไม่ได้ระบุ — ดู Open questions) (quality attribute: performance จากมุมมองผู้ใช้)
- **Negative:** มี stateful component ใหม่ **สองตัว** (RabbitMQ และ order DB) ที่ต้อง deploy, monitor, backup, upgrade บน Kubernetes ทีม 4 คนต้องรับภาระเพิ่ม (quality attribute: operability / cost)
- **Negative:** order DB อยู่ใน critical path ของ `PlaceOrder` แล้ว: ถ้า DB ล่ม order จะ fail และการเขียน DB เพิ่ม latency (ต้องอยู่ใน budget ~200 ms) (quality attribute: availability / performance)
- **Negative:** มีความล่าช้าเพิ่มจาก outbox relay (polling interval) ก่อน event ถึง broker ต้องเล็กพอให้ loyalty ได้แต้มภายใน 1 นาที (quality attribute: performance)
- **Negative:** debug ยากขึ้นเพราะ flow ไม่เป็นเส้นตรง; ต้องส่ง trace context ผ่าน message header (ปัจจุบันใช้ OpenTelemetry กับ gRPC, `main.go` บรรทัด 134–140) (quality attribute: observability)
- **Risks and mitigations:**
  - **Dual-write / event หาย:** card ถูก charge แล้ว (`chargeCard`, บรรทัด 252) แต่ publish ล้มเหลว → แก้ด้วย order DB + transactional outbox (ดู Decision): event ถูกบันทึกพร้อม order ใน transaction เดียว และ relay ส่งซ้ำจนได้ confirm; alert เมื่อมีแถวค้างใน `outbox` นานเกินกำหนด
  - **Order DB ล่ม:** order fail ได้ → ใช้ DB แบบ replicated บน Kubernetes และวัด availability ใน load test (ระดับ HA ที่ต้องการ: ดู Open questions)
  - **Relay ส่งซ้ำ:** relay อาจ publish ซ้ำถ้าล่มหลังได้ confirm แต่ก่อน mark → consumer ทุกตัวต้อง idempotent ด้วย `order_id` อยู่แล้ว
  - **Double points:** RabbitMQ redeliver ได้ → `loyaltyservice` dedup ด้วย `order_id` ใน storage ของตัวเอง
  - **Points ช้ากว่า 1 นาที:** queue ของ loyalty แยกจาก email จึงไม่ติด backlog ของ MailFast; ตั้ง alert เมื่อ queue age ใกล้ 60 s
  - **RabbitMQ ล่ม:** ใช้ cluster หลาย node กับ quorum queues **[Inferred]**; `PlaceOrder` ไม่ block รอ broker เกิน timeout
  - **MailFast fail 0.5 %:** retry with backoff + DLQ และ dashboard ของ DLQ
  - **Schema ของ event เปลี่ยน:** กำหนด `OrderPlaced` เป็น protobuf (ทีมถนัด) และเพิ่ม field แบบ backward-compatible

## Validation

- **Load test** ที่ 200 orders/s (เช่นด้วย Locust `loadgenerator` ตามแนว ADR-0001): p95 ของ `PlaceOrder` ≤ 500 ms
- **Fault injection:** ปิด `emailservice`, `loyaltyservice`, analytics consumer และ MailFast (mock) ระหว่าง load test → อัตรา order failure ที่เกิดจาก action เหล่านี้ = 0
- **Broker outage:** ปิด RabbitMQ ระหว่าง load test → order ยังสำเร็จ, แถวสะสมใน `outbox` และเมื่อ broker กลับมา ทุก `order_id` ได้อีเมลและแต้มครบ (ไม่มี event หาย)
- **Outbox latency:** วัดเวลาจากเขียน `outbox` ถึง publish สำเร็จที่ 200 orders/s และ p95 ของ `PlaceOrder` หลังเพิ่มการเขียน DB ยัง ≤ 500 ms
- **Loyalty correctness:** ส่งข้อความซ้ำ (redeliver) โดยเจตนา → points ต่อ `order_id` = 1 เสมอ; วัด order → points awarded ≤ 60 s ที่ peak
- **Email throughput:** อัตราเรียก MailFast ไม่เกิน 50/s; วัดเวลาที่ backlog หมดหลัง peak; DLQ rate ≈ อัตรา fail หลัง retry
- **Review date:** ทบทวน ADR หลังช่วง 11.11 แรกที่ใช้งานจริง หรือก่อนเพิ่ม action ใหม่ตัวแรกในปีหน้า
- **Action item ก่อน 11.11:** ผลการเจรจาเพิ่ม rate limit กับ MailFast

## Open questions

- **ผู้ตัดสินใจ (Deciders):** ใครเป็นผู้อนุมัติ ADR นี้ — Not evidenced in the provided files
- **ระยะเวลา peak 11.11:** นานเท่าไร? ใช้คำนวณขนาด backlog อีเมล (150 ข้อความ/s × ระยะเวลา) และ disk/memory ของ RabbitMQ
- **SLA ของอีเมลยืนยัน:** ลูกค้ายอมรับความล่าช้าได้แค่ไหนช่วง peak? และ MailFast จะให้เพิ่ม rate limit ช่วง 11.11 ได้เท่าไร?
- **MailFast API:** รองรับ idempotency key หรือไม่ (ป้องกันส่งอีเมลซ้ำเมื่อ retry)? ใช้ protocol อะไร? — Not evidenced in the provided files
- **Latency ของการ publish ไป RabbitMQ** ใน cluster ปัจจุบัน — ยังไม่มีตัวเลข ต้องวัด
- **Storage ของ `loyaltyservice`** (สำหรับ points และตาราง dedup ด้วย `order_id`) — ยังไม่ได้กำหนด
- **ปลายทาง analytics** ของทีม marketing คืออะไร (tool, format, ความล่าช้าที่ยอมรับได้) — Not evidenced in the provided files
- **Order DB:** ใช้ผลิตภัณฑ์ใด (open-source, self-hosted), ระดับ HA และ backup ที่ต้องการ และ relay ควร poll ถี่แค่ไหน
- **ความพร้อมของทีมในการ operate RabbitMQ บน Kubernetes** (operator, backup, monitoring) — มีแค่ข้อมูลว่า 2 คนเคยใช้
