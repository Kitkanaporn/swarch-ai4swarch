**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** Kitkanaporn Umpaimungkorn / 6630014521
**agent ที่ใช้:** Claude Code
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** Claude Code (บัญชี Claude ส่วนตัว) และ Google Antigravity CLI (บัญชี Google ส่วนตัว, ใช้เทียบเพิ่มในแถว 6–7) · หมายเหตุ: Gemini Code Assist / Gemini CLI สำหรับบัญชีส่วนตัวหยุดให้บริการแล้ว จึงใช้ Antigravity แทน

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `Follow prompts/1-recover-architecture.md` run A, evidence: `demo.proto` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | 3 | ได้ 9 service แต่ไม่มีลูกศรเลย เพราะ proto ไม่บอกว่าใครเรียกใคร ไม่มี frontend กับ data store สิ่งที่เดาไว้ AI ย้ายไปใส่ใน Gaps ไม่ได้วาดลงไป |
| 2 | 1-B | Claude Code + Opus 5.5 | `Follow prompts/1-recover-architecture.md` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `protos/demo.proto` และทุกไฟล์ใน `kubernetes-manifests/` (13 ไฟล์) | ใช่ | 5 | ได้ 12 container กับ 17 ลูกศรจาก env var `*_ADDR` เปิดโค้ดเช็กแล้วถูกทุกเส้น แต่ขาด `shoppingassistantservice` กับ tracing collector |
| 3 | 2-a | Claude Code + Opus 5.5 | `Follow prompts/2-write-adr.md` run a, context: none, requirement R-12, language: Thai | `prompts/2-write-adr.md`, `docs/adr/template.md` | ใช่ (ใน ADR ไม่มีตัวเลขจาก scenario เลย แปลว่าไม่ได้เปิด `scenario.md`) | 2 | เลือก pub/sub แต่ไม่บอกว่าใช้ broker ตัวไหน ไม่มีตัวเลขเลย ย้ายไปเป็นคำถาม 12 ข้อใน Open questions |
| 4 | 2-b | Claude Code + Opus 5.5 | `Follow prompts/2-write-adr.md` run b, context: scenario + ADR 0001 + `checkoutservice/main.go`, requirement R-12, language: Thai | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go` | ใช่ | 4 | เลือก RabbitMQ ใช้ตัวเลขจาก scenario ครบ อ้างบรรทัดใน `main.go` ถูก แต่แต่งบางอย่างเอง (frontend เป็น Go, ใช้ Locust) และไม่ตัดสินใจเรื่อง event หาย |
| 5 | 3 | Claude Code + Opus 5.5 | `Follow prompts/3-review-adr.md` adr: `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md`, language: Thai | `prompts/3-review-adr.md`, `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md`, `docs/scenario.md` | ใช่ (ไม่ได้แก้ ADR) | 5 | Accept with changes จับได้ว่า outbox ที่เขียนว่า "transaction เดียวกับ order" ทำไม่ได้ เพราะ checkout ไม่มี DB เก็บ order |
| 6 | 1-A (เทียบ agent) | Antigravity CLI + Gemini 3.8 Flash (High) | ข้อความเดียวกับแถว 1 | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | 3 | ผลใกล้เคียง Claude (9 service ไม่มีลูกศร) แต่ใน Gaps อ้างไฟล์ `redis.yaml` ที่ไม่มีจริง เก็บผลไว้ที่ `outputs/1A-architecture-antigravity.md` |
| 7 | 1-B (เทียบ agent) | Antigravity CLI + Gemini 3.8 Flash (High) | ข้อความเดียวกับแถว 2 | `protos/demo.proto` และทุกไฟล์ใน `kubernetes-manifests/` | ใช่ | 4 | 17 ลูกศรตรงกับ Claude แต่ใส่ชื่อ RPC โดยไม่บอกว่าเดา เช่น `SearchProducts` ที่ frontend ไม่ได้เรียกจริง เก็บผลไว้ที่ `outputs/1B-architecture-antigravity.md` |
| 8 | ช่วยตรวจ + ร่างใบงาน | Claude (แชตในแอป Claude) + Opus 5.5 | ขอให้อธิบายโจทย์ ตรวจผลของ agent เทียบกับซอร์สโค้ด และช่วยร่างคำตอบในใบงาน 01–05 กับแก้ ADR ข้อ 3 | ไฟล์ใน repo รวม `system/online-boutique/src/` (ไม่ได้เปิดการ์ดใน `docs/review-board/`) | ไม่ใช่ run ของโจทย์ (ไม่ได้เขียนไฟล์ใน `outputs/`) | 4 | ใช้เป็นผู้ช่วยตรวจงาน ไม่ได้ใช้แทน agent ใน 5 run; คำตอบในใบงานแก้เป็นภาษาตัวเองก่อนส่ง |
