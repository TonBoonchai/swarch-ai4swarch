**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 | (#1) `demo.proto` นิยาม 9 services ในไฟล์เดียว ภายใต้ `package hipstershop` | Evidenced | Evidenced | `protos/demo.proto` บรรทัด 17 (`package hipstershop;`) และ `service` 9 ตัวที่บรรทัด 23, 56, 71, 107, 140, 177, 199, 224, 243 ตรงตามที่อ้าง |
| 2 | (#26) timeout ใน `mustConnGRPC` ใช้ไม่ได้จริง | Evidenced | Evidenced | `src/checkoutservice/main.go` บรรทัด 212 `_, cancel := context.WithTimeout(ctx, time.Second*3)` ทิ้ง context ที่มี timeout ไป และ `grpc.NewClient` (บรรทัด 214) ไม่ได้รับ ctx เลย timeout 3 วินาทีจึงไม่มีผล |
| 3 | (#23) downstream ตัวใดล้ม checkout ก็ล้ม (availability ผูกกัน) | Evidenced | Evidenced บางส่วน | `main.go` บรรทัด 252–261: payment และ shipping ล้มแล้ว `PlaceOrder` คืน error จริง แต่ข้อความ "ตัวใดก็ได้" กว้างเกินไป เพราะ email ล้มแค่ log warning (บรรทัด 273–277) และผลของ `emptyUserCart` ถูกทิ้ง (`_ =` บรรทัด 263) |
| 4 | (#14) driver: interoperability ข้ามภาษา (polyglot) | Inferred | Inferred (แต่เป็นจริง) | ป้ายถูกต้อง เพราะไฟล์หลักฐาน 3 ไฟล์เป็น Go ทั้งหมด แต่เมื่อดูนอกหลักฐานพบว่าเป็น polyglot จริง: `src/currencyservice/server.js` (Node.js), `src/emailservice/email_server.py` (Python), `src/adservice/build.gradle` (Java), `src/cartservice/cartservice.sln` (C#) และ `src/currencyservice/proto/demo.proto` เป็นสำเนาของ proto กลาง |
| 5 | (#9) `genproto` ของแต่ละ service สร้างจาก `demo.proto` | Inferred | Inferred | ยืนยันไม่ได้แม้ดูทั้ง repo เพราะ `system/README.md` บรรทัด 13 ระบุว่าลบ generated code (`genproto/`) ออกแล้ว เห็นเพียง import `.../checkoutservice/genproto` (`main.go` บรรทัด 34) และชื่อ type `pb.*` ที่ตรงกับ proto |

**นับ:** จากข้อกล่าวอ้างทั้งหมด (31 ข้อ) agent ระบุว่า *Evidenced* กี่ข้อ **18** ตามที่นิสิตตรวจ **17** (อีก 1 ข้อคือ #23 ถูกเพียงบางส่วนเพราะกล่าวกว้างเกินหลักฐาน)

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี
   - ไม่พบการเดาที่ติดป้าย *Evidenced* แบบผิดชัดเจน ตรวจโดยเปิดไฟล์และบรรทัดที่ agent อ้างทีละข้อ (เช่น `grep stream` ใน `demo.proto` ได้ 0 ครั้งสำหรับ #3, import ใน `main.go` บรรทัด 17–43 ไม่มีไลบรารี broker สำหรับ #10) แต่มี 1 ข้อที่ *ข้อความ* กว้างกว่าหลักฐาน คือ #23 "downstream ตัวใดล้ม checkout ก็ล้ม" ทั้งที่ email และ `EmptyCart` ล้มแล้ว checkout ยังสำเร็จ
2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด
   - อาจเข้าใจว่า *ทุก* service คุยกันด้วย gRPC และใช้ `demo.proto` ไฟล์เดียวร่วมกันจริง ๆ ทั้งที่ `frontend` เรียก `shoppingassistantservice` ด้วย HTTP (`src/frontend/handlers.go` บรรทัด 464) และ `currencyservice` มีสำเนา proto ของตัวเอง (`src/currencyservice/proto/demo.proto`) นอกจากนี้เหตุผลการตัดสินใจ (เช่น type-safety หรือ polyglot) เป็นเพียง *Inferred* แต่อ่านแล้วเหมือนเป็นเหตุผลที่ทีมเดิมบันทึกไว้
3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต
   - AI กู้คืน *สิ่งที่เป็น* ได้ดี: services, message types, ลำดับการเรียกใน `PlaceOrder`, การไม่มี TLS และ bug อย่าง `context.TODO()` แต่ *ทำไม* ต้องเลือกแบบนี้ (ข้อจำกัด, ทางเลือกที่เคยลองแล้วทิ้ง, ทักษะทีม) มีแต่สถาปนิกเดิมที่รู้ ตัวอย่างจาก ADR 0002 ของผม: การตัดสินใจใช้ RabbitMQ อาศัยข้อมูลว่า "สองคนในทีมเคยใช้ RabbitMQ" ซึ่งไม่มีในโค้ด และในการประชุม SRE เปิดเผยว่าเคยใช้แค่บนเครื่องตัวเอง ข้อเท็จจริงนี้กู้คืนจากโค้ดไม่ได้เลย
