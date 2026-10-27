**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
**agent ที่ใช้:** Claude Code
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `Follow prompts/1-recover-architecture.md` run A, evidence: `demo.proto` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ (อ่านเฉพาะหลักฐานที่อนุญาต) | 3
| 2 | 1-B | Claude Code + Opus 5.5 | `Follow prompts/1-recover-architecture.md` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `protos/demo.proto` และทุกไฟล์ใน `kubernetes-manifests/` (13 ไฟล์ รวม `kustomization.yaml`, `README.md`) | ใช่ (อ่านเฉพาะหลักฐานที่อนุญาต) | 5
| 3 | 2-a | Claude Code + Opus 5.5 | `Follow prompts/2-write-adr.md` run a, context: none, requirement R-12, language: Thai | (ควรมีแค่ `prompts/2-write-adr.md`, `docs/adr/template.md` — ตรวจจาก log ของ agent) | | | 2
| 4 | 2-b | Claude Code + Opus 5.5 | `Follow prompts/2-write-adr.md` run b, context: scenario + ADR 0001 + `checkoutservice/main.go`, requirement R-12, language: Thai | (ควรมีแค่ไฟล์โจทย์, template และ 3 ไฟล์ใน context — ตรวจจาก log ของ agent) | | | 4
| 5 | 3 | Claude Code + Opus 5.5 | `Follow prompts/3-review-adr.md` adr: `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md`, language: Thai | (ควรมีแค่ไฟล์โจทย์, ADR 0002 และ `docs/scenario.md` — ตรวจจาก log ของ agent) | | | 5
| 6 | 1-A (เทียบ agent) | Antigravity CLI + Gemini 3.8 Flash (High) | ข้อความเดียวกับแถว 1 (run A, evidence: `demo.proto`) | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | 3
| 7 | 1-B (เทียบ agent) | Antigravity CLI + Gemini 3.8 Flash (High) | ข้อความเดียวกับแถว 2 (run B, evidence: `demo.proto` + `kubernetes-manifests/`) | `protos/demo.proto` และทุกไฟล์ใน `kubernetes-manifests/` | ใช่ | 4
| 8 | ช่วยตรวจ + ร่างใบงาน | Claude (แชตในแอป Claude, `claude-opus-5-5`) | ขอให้อธิบายโจทย์ ตรวจผลของ agent เทียบกับซอร์สโค้ด และช่วยร่างคำตอบในใบงาน 01–05 กับแก้ ADR ข้อ 3 | ไฟล์ใน repo ทั้งหมด รวม `system/online-boutique/src/` (ไม่ได้เปิด `docs/review-board/` การ์ด) | ไม่ใช่ run ของโจทย์ (ไม่ได้สร้างไฟล์ใน `outputs/`) | 4 | ใช้เป็นผู้ช่วยตรวจงาน ไม่ได้ใช้แทน agent ทั้ง 5 run; คำตอบในใบงานแก้เป็นภาษาของตัวเองก่อนส่ง |
