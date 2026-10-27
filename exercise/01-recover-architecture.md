**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | manifest บรรทัด 61–62 ตั้งค่า `paymentservice:50051`; `src/checkoutservice/main.go` บรรทัด 116, 123 map env และเปิด gRPC conn; `chargeCard` บรรทัด 369–370 เรียก `PaymentService.Charge` จริง |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | manifest บรรทัด 63–64 (`emailservice:5000`); `main.go` บรรทัด 115, 122 และบรรทัด 380 เรียก `SendOrderConfirmation` ผ่าน gRPC |
| `checkoutservice` → `cartservice` | `checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | manifest บรรทัด 67–68 (`cartservice:7070`); `main.go` บรรทัด 325 เรียก `GetCart` และบรรทัด 333 เรียก `EmptyCart` |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | manifest บรรทัด 60–61; `src/recommendationservice/recommendation_server.py` บรรทัด 131–137 อ่าน env แล้วสร้าง `ProductCatalogServiceStub` ผ่าน `grpc.insecure_channel` |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR`) | ✅ | manifest บรรทัด 50–51 (`redis-cart:6379`); `src/cartservice/src/Startup.cs` บรรทัด 29–40 อ่าน `REDIS_ADDR` แล้วใช้ `AddStackExchangeRedisCache` + `RedisCartStore` |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **dependency ที่ขาดไป: `frontend` → `shoppingassistantservice` (HTTP ไม่ใช่ gRPC)** Run B เห็น `SHOPPING_ASSISTANT_SERVICE_ADDR` ใน `frontend.yaml` แต่ใส่ไว้ใน Gaps เพราะไม่มี manifest โค้ดยืนยันว่ามี service นี้จริง (`src/shoppingassistantservice/shoppingassistantservice.py`) และ frontend เรียกด้วย HTTP (`src/frontend/main.go` บรรทัด 139, `src/frontend/handlers.go` บรรทัด 464 `url := "http://" + fe.shoppingAssistantSvcAddr`) จึงเป็นข้อยกเว้นของข้อสรุป "ทุก service คุยกันด้วย gRPC" แต่ไม่อยู่ใน `kubernetes-manifests/` เพราะ deploy ผ่าน kustomize component ที่ปิดไว้โดย default
  - **dependency แบบมีเงื่อนไขที่ไม่ปรากฏ:** `checkoutservice` ต่อ OpenTelemetry collector ผ่าน `COLLECTOR_SERVICE_ADDR` เฉพาะเมื่อ `ENABLE_TRACING=1` (`main.go` บรรทัด 90–92, 164–165) manifest ไม่ได้ตั้งค่านี้ จึงถูกต้องที่ไม่อยู่ในแผนภาพ แต่ควรระบุว่าเป็น optional
  - **ข้อมูลที่ขาด (ไม่ใช่ความผิด):** Run B ระบุภาษาไม่ได้ แต่ซอร์สโค้ดแสดงว่า `checkoutservice` เป็น Go, `recommendationservice` เป็น Python, `cartservice` เป็น C# (.NET) แสดงว่าระบบเป็น polyglot ซึ่งอธิบายว่าทำไมต้องใช้ `demo.proto` กลาง
  - ไม่พบ dependency ที่ไม่มีจริงใน 5 รายการที่ตรวจ

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   - AI สามารถวิเคราะห์โครงสร้างข้อมูลและสัญญาการสื่อสาร (API contract) ได้ค่อนข้างดี แต่ไม่สามารถยืนยันได้ว่าระบบทำงานจริงอย่างไร
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   - การวาดลำดับการทำงาน (flow) ที่ไม่มีหลักฐานรองรับ และการเชื่อมความสัมพันธ์ระหว่าง service ผิด
