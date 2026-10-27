**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** บุญชัย จินาพร / 6632110421
**agent ที่ใช้:** Claude Code (Claude desktop app, Code tab)
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code (subagent, session ใหม่ต่อ run) + Claude Opus 5.5 | `prompts/1-recover-architecture.md` run A, evidence: `protos/demo.proto` | `AGENTS.md`, `prompts/1-recover-architecture.md`, `protos/demo.proto` | ใช่ | 5
| 2 | 1-B | Claude Code (subagent, session ใหม่ต่อ run) + Claude Opus 5.5 | `prompts/1-recover-architecture.md` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `AGENTS.md`, prompt, `demo.proto`, ทุกไฟล์ใน `kubernetes-manifests/` (13 ไฟล์ รวม `README.md`, `kustomization.yaml`, `loadgenerator.yaml`) | ใช่ | 5
| 3 | 2-a | Claude Code (subagent, session ใหม่ต่อ run) + Claude Opus 5.5 | `prompts/2-write-adr.md` run a, context: none, R-12, language: Thai | `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md` | ใช่ (ไม่ได้เปิด `docs/scenario.md`) | 5
| 4 | 2-b | Claude Code (subagent, session ใหม่ต่อ run) + Claude Opus 5.5 | `prompts/2-write-adr.md` run b, context: scenario + ADR 0001 + `checkoutservice/main.go`, language: Thai | `AGENTS.md`, prompt, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go` | ใช่ | 5
| 5 | 3 | Claude Code (subagent, session ใหม่) + Claude Opus 5.5 | `prompts/3-review-adr.md`, adr: `docs/adr/0002-publish-order-placed-event-rabbitmq.md`, language: Thai | `AGENTS.md`, `prompts/3-review-adr.md`, ADR 0002, `docs/scenario.md` | ใช่ (ไม่ได้แก้ ADR) | 5 | |
| 6 | bonus | Claude Code (subagent, session ใหม่ต่อ run) + Claude Opus 5.5 | `prompts/bonus-recover-decision.md`, decision: sync gRPC + shared `demo.proto`, language: Thai | `AGENTS.md`, prompt, `docs/adr/template.md`, `demo.proto`, `src/checkoutservice/main.go`, `src/frontend/main.go` | ใช่ | 5
| 7 | ตรวจ 1-B | Claude Code + Claude Opus 5.5 (session หลัก) | ขอให้ช่วยตรวจ dependency 5 รายการของ Run B และหาสิ่งที่ผิด/ขาด (ใบงาน 01) | `kubernetes-manifests/{checkoutservice,recommendationservice,cartservice}.yaml`, `src/checkoutservice/main.go`, `src/recommendationservice/recommendation_server.py`, `src/cartservice/src/Startup.cs`, `src/frontend/main.go`, `src/frontend/handlers.go` | ใช่ (ใบงานอนุญาตให้ตรวจซอร์สโค้ด) | | ผลการตรวจในใบงาน 01 มาจาก AI ตามคำขอของนิสิต |
| 8 | เทียบ 2-a/2-b | Claude Code + Claude Opus 5.5 (session หลัก) | ขอให้ช่วยกรอกตารางเทียบ Run a กับ Run b (ใบงาน 02) | `exercise/outputs/2a-adr.md`, `exercise/outputs/2b-adr.md` | ใช่ | | นิสิตตรวจและแก้ต่อเอง |
