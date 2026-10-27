**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย ทุกตัวเลขเขียนว่า "Not evidenced" และย้ายไป Open questions (ปริมาณ order, latency target, SLA/rate limit ของ MailFast) | ใช้ครบ: p95 ≤ 500 ms (เหลือ budget ~200 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5 % / 50 emails/s, loyalty ≤ 1 นาที exactly once, ทีมรู้ Go/gRPC/RabbitMQ, self-hosted บน Kubernetes และคำนวณ backlog อีเมล 150 ข้อความ/s ช่วงพีก |
| ทางเลือกที่พิจารณา (ระบุ) | 1) sync orchestration ใน `checkoutservice` 2) pub/sub ผ่าน broker + transactional outbox 3) in-process async fan-out 4) workflow/saga engine → เลือก 2 แต่**ไม่เลือกตัว broker** (ยกไป ADR ถัดไป) | 1) sync gRPC fan-out 2) goroutine async 3) RabbitMQ events 4) Kafka events → เลือก **RabbitMQ** โดยให้เหตุผลจากทักษะทีมและ constraint open-source self-hosted |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แบบทั่วไป: consumer idempotent ด้วย order id, retry + backoff + circuit breaker + DLQ แต่ไม่รู้ตัวเลข rate limit จึงไม่ได้ออกแบบ throttling จริง และไม่มีเวลา "ภายใน 1 นาที" | ชัดเจน: `loyaltyservice` dedup ด้วย `order_id` (at-least-once + idempotent), queue แยกจากอีเมลเพื่อไม่ติด backlog และ alert เมื่อ queue age ใกล้ 60 s; email consumer จำกัด ≤ 50/s รวมทุก replica + retry + DLQ |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่มีตัวเลขแต่ง ข้อสันนิษฐานเรื่องระบบเดิมติดป้าย [Inferred] แต่เสนอ transactional outbox โดยไม่รู้ว่า `checkoutservice` ไม่มี DB และยกตัวอย่าง Google Cloud Pub/Sub ซึ่งขัดกับ constraint self-hosted (ที่ไม่รู้) | ไม่มีตัวเลขแต่ง แต่มีข้อเสนอที่ไม่มีใน scenario: quorum queues (ติด [Inferred]), protobuf สำหรับ event schema, publisher confirms; และอ้างว่า Kafka "เกินความจำเป็น" ที่ 200 events/s (ติด [Inferred]) |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่ม order DB ให้ `checkoutservice` และใช้ transactional outbox แทนการ publish ตรงแล้ว log เมื่อล้มเหลว | ร่างของ AI ยอมให้ event หายได้เมื่อ publish ไม่สำเร็จ ลูกค้าถูกตัดเงินแต่ไม่ได้อีเมลและแต้ม ซึ่งขัดกับข้อกำหนดว่าต้องได้แต้มแน่นอน และ `checkoutservice` ไม่มี DB (`main.go` struct บรรทัด 66–86) outbox จึงต้องเพิ่ม DB ตั้งแต่รอบนี้ แลกกับ component เพิ่มและ DB อยู่ใน critical path (เพิ่มเป็น consequence และ validation) |
| 2 | อีเมลช่วง peak: ยอมรับความล่าช้าเป็น baseline **และ** ขอเพิ่ม rate limit กับ MailFast เป็น action item ก่อน 11.11 | 200 orders/s เทียบกับ 50 emails/s ทำให้ backlog โต 150 ข้อความ/s ร่าง AI แค่ตั้งคำถามไว้ การยอมรับความช้าทำให้ระบบยังถูกต้องแม้เจรจาไม่สำเร็จ ส่วนการขอเพิ่ม limit ลดความล่าช้าที่ลูกค้าเห็น |
| 3 | ตัด fraud scoring ออกจากขอบเขต ADR นี้ ทำ ADR แยก | fraud scoring ที่ต้อง block order เป็นขั้นตอน synchronous ใน critical path ไม่ใช่ consumer ของ `OrderPlaced` ร่าง AI ใส่ไว้ทั้งในรายการ action ที่ bind queue ได้และใน Open questions ซึ่งขัดกันเอง |
