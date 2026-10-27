> Agent: Claude Code — model: Claude Opus 5.5 (`claude-opus-5-5`)

# ADR-R001: ใช้ gRPC แบบ synchronous ผ่านสัญญา (contract) เดียวคือ `demo.proto` สำหรับการสื่อสารระหว่าง services ทั้งหมด

- **Status:** Reconstructed
- **Date:** Not evidenced in the provided files (header ลิขสิทธิ์ระบุปี 2018 ใน `main.go` ทั้งสองไฟล์ และ 2020 ใน `demo.proto`)
- **Deciders:** Not evidenced in the provided files (header ระบุเพียง "Google LLC")

> ไฟล์ที่ใช้เป็นหลักฐาน: `system/online-boutique/protos/demo.proto`, `system/online-boutique/src/checkoutservice/main.go`, `system/online-boutique/src/frontend/main.go` เท่านั้น ข้อสรุปเกี่ยวกับ services อื่น (ภาษา, การ deploy) จึงจำกัดอยู่ที่สิ่งที่สามไฟล์นี้แสดง

## Context and problem

- Online Boutique แบ่งเป็น backend services หลายตัว ซึ่ง `demo.proto` นิยามไว้ 9 services ในไฟล์เดียว ภายใต้ `package hipstershop` คือ `CartService`, `RecommendationService`, `ProductCatalogService`, `ShippingService`, `CurrencyService`, `PaymentService`, `EmailService`, `CheckoutService` และ `AdService` (`demo.proto` บรรทัด 17, 23–244)
- services ใช้ message types ร่วมกันหลายตัว ได้แก่ `Money`, `Address`, `CartItem` และ `Empty` เช่น `GetQuoteRequest` ของ `ShippingService` ใช้ `CartItem`, ส่วน `OrderResult` ใช้ทั้งใน `EmailService` และ `CheckoutService` (`demo.proto` บรรทัด 29, 52, 114, 130, 146, 208–218, 237–239)
- `frontend` รับ HTTP จากผู้ใช้ (`gorilla/mux`, `http.ListenAndServe`) แล้วเรียก backend ผ่าน gRPC client 7 ตัว (`frontend/main.go` บรรทัด 141–147, 149–171)
- `checkoutservice` ทำหน้าที่ orchestrator: `PlaceOrder` เรียก services อื่นแบบ synchronous ทีละขั้นตามลำดับ: `GetCart` → (`GetProduct` + `Convert` ต่อสินค้าแต่ละชิ้น) → `GetQuote` → `Convert` → `Charge` → `ShipOrder` → `EmptyCart` → `SendOrderConfirmation` (`checkoutservice/main.go` `PlaceOrder`, `prepareOrderItemsAndShippingQuoteFromCart`, `prepOrderItems` บรรทัด 230–357)
- services ไม่ได้แชร์ code package กัน ดูจาก comment "this method is duplicated in other microservices using Go since they are not sharing packages" (`checkoutservice/main.go` บรรทัด 181–182; `frontend/main.go` บรรทัด 195–196) และแต่ละ service import generated code จาก `genproto` ของตัวเอง (`checkoutservice/main.go` บรรทัด 34) สิ่งที่ services ใช้ร่วมกันจริงจึงมีแค่สัญญาใน `.proto`
- ทั้งสามไฟล์ไม่มีการใช้ message broker หรือการสื่อสารแบบ asynchronous ระหว่าง services
- **โจทย์ที่ต้องตัดสินใจ:** services จะสื่อสารกันด้วย protocol และรูปแบบใด (synchronous หรือ asynchronous) และจะจัดการ interface contract อย่างไร (ไฟล์เดียวรวม หรือแยกตาม service)

## Decision drivers

1. **Modifiability / ความสอดคล้องของ contract:** services ต้องใช้ types ร่วมกัน (`Money`, `Address`, `CartItem`) ให้ตรงกัน และต้องสร้าง client/server stub ได้จากต้นฉบับเดียว
2. **Simplicity / understandability:** ผู้อ่าน (นักพัฒนาหรือผู้เรียน) ควรเห็น interface ของทั้งระบบได้ในที่เดียว **[Inferred]**
3. **Performance (latency):** เส้นทางของผู้ใช้ (เปิดหน้าเว็บ, checkout) ต้องได้คำตอบทันทีภายใน request เดียว
4. **Observability:** ต้อง trace ข้าม services ได้ (มี `otelgrpc` และ `otelhttp` อยู่ในโค้ด)
5. **Interoperability ข้ามภาษา:** **[Inferred]** ไฟล์ที่ได้รับไม่ได้แสดงว่ามี services ที่เขียนด้วยภาษาอื่น

## Considered options

1. **A: gRPC แบบ synchronous + `demo.proto` ไฟล์เดียวที่ทุก service ใช้ร่วมกัน** (สถานะปัจจุบัน)
2. **B: gRPC แบบ synchronous + แยกไฟล์ `.proto` ตาม service** (แต่ละ service เป็นเจ้าของ contract ของตัวเอง และแยก package ที่มีเวอร์ชัน เช่น `cart.v1`)
3. **C: REST/JSON over HTTP** (มีสเปก OpenAPI ต่อ service)
4. **D: Hybrid: gRPC แบบ synchronous สำหรับ query + asynchronous messaging ผ่าน message broker (เช่น RabbitMQ หรือ Kafka)** สำหรับขั้นตอนที่ไม่ต้องรอผล เช่น `SendOrderConfirmation` และ `ShipOrder`

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A | เห็น contract ทั้งระบบได้ในไฟล์เดียว; types ร่วม (`Money`, `Address`) นิยามไว้ครั้งเดียว; generate stub ด้วย toolchain เดียวกันได้; binary protocol และ HTTP/2 มี overhead ต่ำ **[Assumption]**; มี `grpc.health` และ `otelgrpc` ให้ใช้ทันที | ทุก service ผูกกับไฟล์เดียว (shared-contract coupling): แก้ service หนึ่งแล้ว generated code ของทุก service เปลี่ยน; package `hipstershop` ไม่มีเวอร์ชัน; การเรียกแบบ sync ทำให้ latency สะสมและความล้มเหลวลามต่อกันได้ |
| B | ownership ชัดเจน, version และ deploy แต่ละ service ได้อิสระกว่า; breaking change กระทบเฉพาะ consumer ของ service นั้น | types ร่วม (`Money`, `Address`) ต้องแยกเป็น common proto หรือนิยามซ้ำ; มีไฟล์และขั้นตอน build ให้ดูแลมากขึ้น; ภาพรวมของระบบอ่านยากขึ้น |
| C | debug ง่าย (curl, browser); tooling และคนที่รู้จักมีมาก; ไม่ต้อง codegen | payload ใหญ่กว่าและ parse ช้ากว่า **[Assumption]**; ถ้าไม่บังคับใช้ OpenAPI สัญญาจะไม่ type-safe; ต้องเขียน client เองหรือใช้ generator แยก |
| D | ลด latency ของ `PlaceOrder` เพราะไม่ต้องรอ email และ shipping; แยกความล้มเหลว (emailservice ล่มแต่ order ยังผ่าน); รองรับ retry ได้ทนทานกว่า | ต้องเพิ่ม infrastructure (broker) และงาน operate; ได้แค่ eventual consistency; debug และ trace flow แบบ async ยากกว่า; demo ซับซ้อนขึ้น |

## Decision

เลือก **Option A**: services ทั้งหมดใน `demo.proto` สื่อสารกันด้วย gRPC แบบ synchronous (unary RPC ทุกตัว ไม่มี streaming; `demo.proto` บรรทัด 23–244) และใช้ `demo.proto` ไฟล์เดียวเป็นแหล่งความจริงของ contract

เหตุผลที่สร้างขึ้นใหม่ (ส่วนใหญ่เป็น **[Inferred]** เพราะไฟล์ไม่ได้บันทึกเหตุผลไว้):
- types ร่วมมีมาก การรวมไว้ไฟล์เดียวจึงทำให้ contract สอดคล้องกันง่ายที่สุด **[Inferred]**
- ระบบเป็น demo ที่ต้องการให้ผู้อ่านเห็น interface ทั้งหมดในที่เดียว **[Inferred]**
- flow ของผู้ใช้เป็นแบบ request/response ที่ต้องได้ผลทันที (แสดงหน้าเว็บ, ส่งผล `PlaceOrderResponse` กลับ) ซึ่งเข้ากับ sync RPC **[Inferred]**
- gRPC มาพร้อม health check (`healthpb.RegisterHealthServer`) และ tracing (`otelgrpc`) ที่ใช้อยู่จริงในโค้ด (Evidenced)

## Consequences

- **Positive:** contract เดียวและ type-safe; `Money`/`Address`/`CartItem` ตรงกันทุก service (quality attribute: modifiability, interoperability)
- **Positive:** การเรียกแบบ sync ให้ผลลัพธ์ทันที ทำให้ `PlaceOrder` ตอบผู้ใช้ด้วย `OrderResult` ที่สมบูรณ์ได้ใน request เดียว (quality attribute: usability / consistency)
- **Positive:** trace context ส่งต่อผ่าน `otelgrpc` stats handler ทั้งฝั่ง client และ server (quality attribute: observability)
- **Negative:** latency ของ `PlaceOrder` เท่ากับผลรวมของการเรียกทีละตัว และ `prepOrderItems` เรียก `GetProduct` + `Convert` ต่อสินค้าแต่ละชิ้น (N+1) (quality attribute: performance)
- **Negative:** ถ้า downstream ตัวใดตัวหนึ่ง (cart, catalog, currency, shipping, payment) ล้ม checkout จะล้มทั้งหมด เพราะ availability ของ checkout ขึ้นกับทุก service ที่เรียก (quality attribute: availability)
- **Negative:** การแก้ `demo.proto` กระทบ generated code (`genproto`) ของทุก service และไม่มีเวอร์ชันใน package (quality attribute: modifiability, deployability)
- **Negative:** ใช้ `insecure.NewCredentials()` ไม่มี TLS ระหว่าง services (quality attribute: security)
- **Risks and mitigations:**
  - *ตัดเงินแล้วส่งของไม่สำเร็จ:* `chargeCard` ทำก่อน `shipOrder` ถ้า shipping ล้มจะคืน `codes.Unavailable` โดยไม่มี refund หรือ compensation → mitigation: ใช้ saga/compensation หรือทำ shipping แบบ async ผ่าน queue ที่ retry ได้ (Option D) **[Inferred]**
  - *ไม่มี deadline ที่ใช้ได้จริง:* ใน `mustConnGRPC` timeout context ถูกทิ้ง (`_, cancel := context.WithTimeout(...)`) และ `PlaceOrder` ไม่ได้ตั้ง deadline ต่อการเรียก → mitigation: ตั้ง deadline ต่อ RPC, เพิ่ม retry/backoff และ circuit breaker **[Assumption]**
  - *trace ขาดช่วง:* `convertCurrency` ใช้ `context.TODO()` แทน `ctx` จึงไม่ได้ส่ง trace context และ cancellation ต่อไป → mitigation: ส่ง `ctx` ต่อให้ถูกต้อง
  - *email ล้มเหลวเงียบ:* `SendOrderConfirmation` ที่ล้มจะถูกแค่ log warning และ `EmptyCart` ไม่ตรวจ error → mitigation: ใช้ outbox/queue สำหรับ email **[Inferred]**
  - *proto breaking change:* mitigation: เพิ่มเวอร์ชันใน package และตรวจ breaking change ใน CI (เช่น `buf breaking`) **[Assumption]**
  - *ข้อยกเว้นของคำว่า "ทุก service":* `frontend` อ่าน `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่ได้เปิด gRPC connection ไปที่ service นี้ และ `demo.proto` ก็ไม่มี service นี้ ไฟล์ที่ได้รับไม่ได้แสดงว่า frontend เรียก service นี้ด้วย protocol อะไร (Not evidenced in the provided files)

## Validation

- **Load test** `POST /cart/checkout`: วัด p95/p99 latency เทียบกับจำนวนสินค้าในตะกร้า เพื่อยืนยันผลของ N+1
- **Fault injection:** ปิด `emailservice` แล้วต้องยังสั่งซื้อได้; ปิด `shippingservice` แล้วตรวจว่ามีการตัดเงินที่ไม่มีการส่งของหรือไม่ และใช้ผลนี้ตัดสินว่าจะย้ายไป Option D บางส่วนหรือไม่
- **Distributed trace review:** ตรวจใน trace ว่า span ของ `CurrencyService.Convert` ต่อกับ trace ของ `PlaceOrder` ถูกต้อง (ปัจจุบันน่าจะขาดเพราะ `context.TODO()`)
- **Contract check:** เพิ่ม breaking-change check ของ `demo.proto` ใน CI และทบทวน ADR นี้เมื่อจำนวน services หรือทีมเพิ่มขึ้น **[Assumption]**

## Claim check

| # | Claim | Type | Evidence |
|---|---|---|---|
| 1 | `demo.proto` นิยาม 9 services ในไฟล์เดียวภายใต้ `package hipstershop` | Evidenced | `system/online-boutique/protos/demo.proto` บรรทัด 17, `service CartService` … `service AdService` (23–244) |
| 2 | services ใช้ message types ร่วมกัน (`Money`, `Address`, `CartItem`, `Empty`, `OrderResult`) | Evidenced | `demo.proto` `GetQuoteRequest.items` (114), `ChargeRequest.amount` (189), `SendOrderConfirmationRequest.order` (218), `PlaceOrderResponse.order` (238) |
| 3 | RPC ทั้งหมดเป็น unary (request/response) ไม่มี streaming | Evidenced | `demo.proto` ทุก `rpc` ไม่มีคีย์เวิร์ด `stream` |
| 4 | package ของ proto ไม่มีเวอร์ชัน | Evidenced | `demo.proto` `package hipstershop;` (17) |
| 5 | `frontend` รับ HTTP จากผู้ใช้และเรียก backend 7 ตัวผ่าน gRPC | Evidenced | `system/online-boutique/src/frontend/main.go` `mux.NewRouter` (149), `http.ListenAndServe` (171), `mustConnGRPC` (141–147) |
| 6 | `checkoutservice` เป็น orchestrator ที่เรียก 6 services แบบ synchronous ตามลำดับ | Evidenced | `system/online-boutique/src/checkoutservice/main.go` `PlaceOrder` (230–280), `mustConnGRPC` (118–123) |
| 7 | `prepOrderItems` เรียก `GetProduct` + `Convert` ต่อสินค้าแต่ละชิ้น (N+1) | Evidenced | `checkoutservice/main.go` `prepOrderItems` loop (343–355) |
| 8 | services ไม่แชร์ code package กัน; แต่ละ service มี `genproto` ของตัวเอง | Evidenced | comment ใน `initProfiling` (`checkoutservice/main.go` 181–182, `frontend/main.go` 195–196); import `.../src/checkoutservice/genproto` (34) |
| 9 | `genproto` ของแต่ละ service สร้างจาก `demo.proto` | Inferred | import path และชื่อ `pb.*` ตรงกับ `demo.proto` แต่ไม่เห็น script ที่ใช้ generate |
| 10 | ไม่มี message broker หรือการสื่อสาร async ในไฟล์ที่ได้รับ | Evidenced | import ของ `checkoutservice/main.go` (17–43) และ `frontend/main.go` (17–36) มีแค่ gRPC/HTTP/OTel |
| 11 | services ทั้งระบบไม่มี async messaging เลย | Inferred | เห็นแค่ 2 services จาก 3 ไฟล์ |
| 12 | driver: contract ที่สอดคล้องกันและ type-safe | Inferred | อนุมานจากข้อ 2; ไม่มีเอกสารบันทึกเหตุผล |
| 13 | driver: ความง่ายและการเห็น interface ทั้งระบบในที่เดียว (ลักษณะของ demo) | Inferred | Not evidenced in the provided files |
| 14 | driver: interoperability ข้ามภาษา (polyglot) | Inferred | `demo.proto` มีแค่ `go_package` (19); ไฟล์ที่ได้รับไม่ได้แสดงว่ามี services ภาษาอื่น |
| 15 | gRPC/HTTP2/protobuf มี overhead ต่ำกว่า REST/JSON | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 16 | Option B: แยก proto ต่อ service ทำให้ version และ deploy ได้อิสระกว่า | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 17 | Option C: REST debug ง่ายกว่าแต่ไม่ type-safe ถ้าไม่มี OpenAPI | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 18 | Option D: broker ลด latency และแยกความล้มเหลว แต่ได้ eventual consistency และเพิ่มงาน operate | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
| 19 | เหตุผลที่เลือก A คือ types ร่วม, เป็น demo และ flow เป็นแบบ request/response | Inferred | ไม่มีเอกสารเหตุผลในไฟล์ |
| 20 | มี gRPC health check และ tracing (`otelgrpc`, propagator `TraceContext`) | Evidenced | `checkoutservice/main.go` `healthpb.RegisterHealthServer` (144), `otelgrpc.NewServerHandler` (139), `SetTextMapPropagator` (135–137); `mustConnGRPC` `WithStatsHandler` (216); `frontend/main.go` (231, 168) |
| 21 | การสื่อสารระหว่าง services ไม่มี TLS | Evidenced | `insecure.NewCredentials()` ใน `mustConnGRPC` (`checkoutservice/main.go` 215, `frontend/main.go` 230) |
| 22 | latency ของ `PlaceOrder` เป็นผลรวมของการเรียกทีละตัว | Evidenced | การเรียกเรียงกันใน `PlaceOrder` / `prepareOrderItemsAndShippingQuoteFromCart` (238–277, 290–305) ไม่มี goroutine ทำงานขนานกัน |
| 23 | downstream ตัวใดล้ม checkout ก็ล้ม (availability ผูกกัน) | Evidenced | error ของ cart/catalog/currency/shipping/payment ถูกคืนเป็น `status.Errorf` (240, 254, 260) |
| 24 | ตัดเงินก่อนส่งของ และไม่มี compensation เมื่อ shipping ล้ม | Evidenced | `chargeCard` (252) ตามด้วย `shipOrder` → `codes.Unavailable` (258–261) โดยไม่มีการ refund |
| 25 | email ที่ล้มเหลวถูกแค่ log และไม่ตรวจ error ของ `EmptyCart` | Evidenced | `checkoutservice/main.go` (263, 273–277) |
| 26 | timeout ใน `mustConnGRPC` ใช้ไม่ได้จริง และไม่มี deadline ต่อ RPC | Evidenced | `_, cancel := context.WithTimeout(...)` (`checkoutservice/main.go` 212, `frontend/main.go` 227); `PlaceOrder` ไม่ได้ตั้ง deadline |
| 27a | `convertCurrency` ใช้ `context.TODO()` แทน `ctx` | Evidenced | `checkoutservice/main.go` `convertCurrency` (360) |
| 27b | ผลของข้อ 27a คือ span ของ `Convert` ขาดจาก trace ของ `PlaceOrder` และไม่ได้รับ cancellation | Inferred | ต้องยืนยันจาก trace จริง (ดู Validation) |
| 28 | การแก้ `demo.proto` กระทบ generated code ของทุก service | Inferred | อนุมานจากข้อ 1 และ 9 |
| 29 | `shoppingassistantservice` ไม่อยู่ใน `demo.proto` และ frontend ไม่ได้เปิด gRPC connection ไปที่ service นี้ | Evidenced | `frontend/main.go` `shoppingAssistantSvcAddr` (88, 139) ไม่มี `mustConnGRPC`; `demo.proto` ไม่มี service นี้ |
| 30 | mitigations (saga, deadline/retry/circuit breaker, outbox, `buf breaking`) | Assumption | ความรู้ทั่วไปในอุตสาหกรรม |
