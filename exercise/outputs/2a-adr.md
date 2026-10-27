Agent: Claude Code (Claude Opus 5.5, `claude-opus-5-5`) — run a (zero-shot, context: none)

# ADR-0002: Publish an "OrderPlaced" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not evidenced in the provided files (ดู Open questions)

> หมายเหตุ: ADR นี้เขียนจาก requirement เพียงอย่างเดียว (context: `none`) จึงไม่มีการอ้างอิงไฟล์ในระบบ ข้อความเกี่ยวกับโครงสร้างปัจจุบันของ Online Boutique ที่ไม่ได้มาจาก requirement จะติดป้าย **[Inferred]**

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ ระบบต้องทำ post-order actions ใหม่ 3 อย่าง (จาก requirement):

| # | Action | ผู้ดำเนินการ | ลักษณะ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | **MailFast** (external email provider ใหม่) แทน logic mock ใน `emailservice` | เรียกระบบภายนอกผ่านเครือข่าย อาจช้าหรือล่มได้ |
| 2 | ให้ loyalty points | `loyaltyservice` (ยังไม่ได้สร้าง) | service ภายในใหม่ |
| 3 | บันทึก analytics event "order placed" | ทีม marketing | ข้อมูลเพื่อการวิเคราะห์ |

- Marketing คาดว่าจะมี post-order actions เพิ่มขึ้นในปีหน้า (SMS, fraud scoring, การอัปเดต recommendations) ดังนั้นจำนวน consumer ของเหตุการณ์ "สั่งซื้อแล้ว" จะเพิ่มขึ้นเรื่อย ๆ
- **[Inferred]** ใน Online Boutique การสั่งซื้อถูกประสานโดย `checkoutservice` ซึ่งเรียก `emailservice` แบบ synchronous (gRPC) หลังชำระเงิน — ไม่ได้ตรวจสอบจากไฟล์ในรอบนี้
- ปัญหาที่ต้องตัดสินใจ: **จะเชื่อม `checkoutservice` (ผู้สร้างคำสั่งซื้อ) เข้ากับ post-order actions ที่มีอยู่และที่จะเพิ่มในอนาคตอย่างไร** โดยไม่ให้ความล้มเหลวหรือความช้าของ action ใด action หนึ่ง (โดยเฉพาะ MailFast ซึ่งอยู่นอกการควบคุม) ทำให้การสั่งซื้อล้มเหลวหรือช้า และไม่ต้องแก้ `checkoutservice` ทุกครั้งที่มี action ใหม่
- ตัวเลขสำคัญ (ปริมาณคำสั่งซื้อ, latency target, SLA ของ MailFast, ความล่าช้าที่ยอมรับได้ของอีเมล) — Not evidenced in the provided files (ดู Open questions)

## Decision drivers

1. **Modifiability / extensibility** — เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ `checkoutservice`
2. **Availability ของ checkout** — การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม
3. **Performance (checkout latency)** — ผู้ใช้ไม่ควรต้องรอ action ปลายทางทั้งหมด
4. **Reliability** — ทุก action ต้องเกิดขึ้นในที่สุด (ไม่หาย) และไม่ซ้ำจนเกิดผลเสีย เช่น ให้ points ซ้ำหรือส่งอีเมลซ้ำ
5. **Operability / cost** — จำนวน component ใหม่ที่ทีมต้องดูแล และความสามารถในการ trace ปัญหา
6. **Ordering constraint ที่อาจมีในอนาคต** — fraud scoring อาจต้องทำ *ก่อน* ยืนยันคำสั่งซื้อ (ยังไม่ทราบ ดู Open questions)

## Considered options

1. **Synchronous orchestration ใน `checkoutservice`** — `checkoutservice` เรียก MailFast adapter (`emailservice`), `loyaltyservice` และ analytics โดยตรงทีละตัว (gRPC/HTTPS) ภายใน request ของการสั่งซื้อ
2. **Event-driven publish/subscribe ผ่าน message broker** — `checkoutservice` publish event `OrderPlaced` หนึ่งครั้งไปยัง message broker; `emailservice` (เรียก MailFast), `loyaltyservice` และ analytics consumer subscribe แยกกันแบบ asynchronous พร้อม retry และ dead-letter queue; ใช้ transactional outbox (หรือกลไกเทียบเท่า) เพื่อไม่ให้ event หาย
3. **Asynchronous fan-out ภายใน `checkoutservice` (in-process)** — `checkoutservice` ตอบผู้ใช้ทันที แล้วเรียก action ต่าง ๆ ใน background goroutine/worker ของตัวเอง โดยไม่มี broker
4. **Workflow / orchestration engine (saga orchestrator)** — `checkoutservice` เริ่ม workflow "post-order" ใน orchestration engine ที่กำหนดขั้นตอน, retry และลำดับของแต่ละ action ไว้ตรงกลาง

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous orchestration | เรียบง่าย ไม่มี infrastructure ใหม่; ทราบผลทันที; debug ง่ายด้วย call chain เดียว | checkout latency = ผลรวม latency ของทุก action (performance); MailFast ล่ม = checkout ล้มหรือต้องเขียน error handling เอง (availability); ต้องแก้ `checkoutservice` ทุกครั้งที่เพิ่ม action (modifiability); coupling สูงขึ้นตามจำนวน action |
| 2. Event-driven pub/sub + broker | `checkoutservice` ไม่รู้จัก consumer — เพิ่ม SMS/fraud/recommendations โดยเพิ่ม subscriber ใหม่ (modifiability); action ล้มเหลวไม่กระทบ checkout และ retry ได้อิสระ (availability); checkout ตอบเร็วขึ้น (performance); แต่ละ consumer scale แยกกันได้ | ต้องเพิ่มและดูแล message broker (cost, operability); eventual consistency — อีเมล/points มาช้าได้; delivery แบบ at-least-once ต้องทำ consumer ให้ idempotent; tracing ข้ามระบบยากขึ้น; ไม่เหมาะกับ action ที่ต้อง block การสั่งซื้อ (เช่น fraud scoring แบบก่อนยืนยัน) |
| 3. In-process async fan-out | ไม่มี infrastructure ใหม่; checkout ตอบเร็ว | งานหายถ้า pod `checkoutservice` restart ระหว่างทำ (reliability); ยังต้องแก้ `checkoutservice` เมื่อเพิ่ม action (modifiability); retry/backoff ต้องเขียนเอง; โหลดของ action ไปกิน resource ของ checkout |
| 4. Workflow engine | retry, timeout, ลำดับขั้นตอน และสถานะแต่ละ order มองเห็นได้จากจุดเดียว (operability); รองรับขั้นตอนที่ต้องเรียงลำดับหรือ compensation ได้ดี | เพิ่ม component ที่ซับซ้อนและต้องเรียนรู้ (cost, ทักษะทีมไม่ทราบ); การเพิ่ม action ยังต้องแก้ workflow definition ตรงกลาง (modifiability น้อยกว่า Option 2); engine กลายเป็นจุดพึ่งพาสำคัญ (availability) |

## Decision

เลือก **Option 2: Event-driven publish/subscribe ผ่าน message broker** โดย:

- `checkoutservice` publish event `OrderPlaced` (เช่น order id, user id, email, รายการสินค้า, ยอดรวม) **หลัง** คำสั่งซื้อสำเร็จ แทนการเรียก `emailservice` โดยตรง
- ใช้ **transactional outbox** (หรือกลไก publish ที่รับประกันว่า event ไม่หายเมื่อ order สำเร็จ) — รายละเอียด implementation ขึ้นกับว่า `checkoutservice` มี data store หรือไม่ (Not evidenced in the provided files)
- Subscribers แยกกัน แต่ละตัวมี queue/consumer group ของตัวเอง, retry with backoff และ dead-letter queue:
  - `emailservice` → เรียก **MailFast** (แทน mock logic) โดยส่ง idempotency key = order id ถ้า MailFast รองรับ
  - `loyaltyservice` → ให้ points แบบ idempotent (ตรวจ order id ซ้ำ)
  - analytics consumer → ส่ง/บันทึก "order placed" event ให้ marketing
- กำหนด schema ของ `OrderPlaced` แบบมี version และเปลี่ยนแบบ backward-compatible เท่านั้น
- เลือกผลิตภัณฑ์ message broker (เช่น RabbitMQ, Kafka, Google Cloud Pub/Sub) แยกเป็น ADR ถัดไป เพราะยังไม่ทราบข้อจำกัดด้าน platform, cost และทักษะทีม

**เหตุผล:** Option 2 ตอบ decision driver อันดับ 1–3 ได้ดีที่สุด — action ใหม่ในปีหน้าเพิ่มได้เพียงแค่ subscribe event เดิม, ความล้มเหลวของ MailFast หรือ `loyaltyservice` ไม่ทำให้ checkout ล้ม, และ checkout ไม่ต้องรอ action ปลายทาง Option 1 และ 3 ต้องแก้ `checkoutservice` ทุกครั้งที่มี action ใหม่ ส่วน Option 4 ให้ visibility ดีกว่าแต่ซับซ้อนเกินความจำเป็นสำหรับ action ที่เป็นอิสระต่อกันทั้งหมดในปัจจุบัน ถ้าในอนาคต fraud scoring ต้อง block การสั่งซื้อ ให้ทำเป็น synchronous call ก่อนยืนยัน order (ไม่ใช่ subscriber ของ `OrderPlaced`) และพิจารณา Option 4 ใหม่

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) โดยไม่ต้องแก้หรือ redeploy `checkoutservice` (quality attribute: modifiability)
- **Positive:** MailFast, `loyaltyservice` หรือ analytics ล่มชั่วคราวไม่ทำให้การสั่งซื้อล้มเหลว — event รอใน queue แล้ว retry (quality attribute: availability)
- **Positive:** checkout ไม่ต้องรอการเรียก MailFast และ service อื่น จึงตอบผู้ใช้เร็วขึ้น [Inferred: ขึ้นกับ latency ปัจจุบันซึ่งไม่ทราบ] (quality attribute: performance)
- **Positive:** consumer แต่ละตัว scale และ deploy แยกกันได้ (quality attribute: scalability, deployability)
- **Negative:** ต้องติดตั้ง ดูแล และ monitor message broker เพิ่ม (quality attribute: cost, operability)
- **Negative:** eventual consistency — อีเมลยืนยันและ points อาจมาถึงช้ากว่าการสั่งซื้อ (quality attribute: usability / data consistency)
- **Negative:** debug และ trace flow ข้าม asynchronous boundary ยากขึ้น ต้องส่ง trace context ไปกับ event (quality attribute: observability)
- **Negative:** event schema กลายเป็น contract ร่วมระหว่างหลายทีม การเปลี่ยนต้องระวัง (quality attribute: modifiability)
- **Risks and mitigations:**
  - *Event หาย* (order สำเร็จแต่ publish ไม่สำเร็จ) → transactional outbox หรือ publish แบบ retry จนสำเร็จ
  - *ส่งซ้ำ* (at-least-once delivery) → consumer ทุกตัว idempotent โดยใช้ order id; ใช้ idempotency key กับ MailFast ถ้ารองรับ
  - *MailFast ล่มนาน / rate limit* → retry with exponential backoff, circuit breaker, dead-letter queue พร้อม alert และ replay
  - *ส่งข้อมูลส่วนบุคคล (อีเมลลูกค้า) ไปยัง external provider และ analytics* → ส่งเฉพาะ field ที่จำเป็นให้แต่ละ consumer; ตรวจข้อกำหนด privacy (ดู Open questions)
  - *Broker เป็น single point of failure ของ post-order actions* → ใช้ broker แบบ replicated/managed; outbox เก็บ event ไว้จน broker กลับมา
  - *Vendor lock-in กับ MailFast* → ซ่อน MailFast ไว้หลัง `emailservice` (adapter) เพื่อเปลี่ยน provider ได้โดยไม่กระทบ consumer อื่น

## Validation

- **Fault-injection test:** ปิด MailFast (หรือ stub ให้ error/timeout) และ `loyaltyservice` แล้ววัดว่าอัตราการสั่งซื้อสำเร็จไม่ลดลง และเมื่อกลับมา อีเมลและ points ถูกส่งครบทุก order
- **Load test:** เปรียบเทียบ checkout latency (p95/p99) ก่อนและหลังการเปลี่ยน — target: Not evidenced in the provided files
- **Metrics:** consumer lag ต่อ subscriber, จำนวนข้อความใน dead-letter queue, เวลาจาก order ถึงอีเมลถูกส่ง (end-to-end delay), อัตราการซ้ำของ points
- **Modifiability check:** เพิ่ม subscriber ตัวอย่าง (เช่น SMS stub) โดยไม่แก้ `checkoutservice` — ถ้าทำได้ ถือว่า driver อันดับ 1 ผ่าน
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มทำ fraud scoring (เพราะอาจต้องเป็น synchronous) — วันที่ที่แน่นอน: Not evidenced in the provided files

## Open questions

- ปริมาณคำสั่งซื้อ (ปัจจุบัน/peak) และ checkout latency target เท่าไร?
- ความล่าช้าสูงสุดที่ยอมรับได้ของอีเมลยืนยันและการให้ points คือเท่าไร?
- MailFast มี SLA, rate limit, API (REST/SMTP), webhook สถานะการส่ง และรองรับ idempotency key หรือไม่? ค่าใช้จ่ายต่ออีเมลเท่าไร?
- `checkoutservice` มี data store สำหรับทำ transactional outbox หรือไม่? (ไม่ได้ตรวจสอบโค้ดในรอบนี้)
- Platform ที่ deploy และข้อจำกัดด้าน cost: มี managed message broker ให้ใช้หรือไม่? ทีมมีทักษะกับ broker ตัวใด?
- ทีม marketing ต้องการ analytics event ไปที่ไหน (data warehouse, analytics tool ภายนอก) และรูปแบบใด?
- Fraud scoring ในอนาคตต้อง block การสั่งซื้อ (ก่อนยืนยัน) หรือทำหลังสั่งซื้อได้?
- `loyaltyservice` ใช้กฎการให้ points อย่างไร และต้องหัก points คืนเมื่อยกเลิก/คืนสินค้าหรือไม่?
- ข้อกำหนดด้าน privacy/PDPA สำหรับการส่งอีเมลและข้อมูลลูกค้าไปยัง MailFast และ analytics มีอะไรบ้าง?
- ใครคือ Deciders ของ ADR นี้?
