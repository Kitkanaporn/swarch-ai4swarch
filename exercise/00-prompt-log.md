**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
**agent ที่ใช้:** Claude Code
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `Follow prompts/1-recover-architecture.md` run A, evidence: `demo.proto` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ (อ่านเฉพาะหลักฐานที่อนุญาต) | | ได้ 9 service แต่ไม่มี dependency เลย (proto ไม่บอกว่าใครเรียกใคร) ไม่มี frontend/data store; AI ติดป้าย [Inferred] ให้สิ่งที่เดาและย้ายไปไว้ใน Gaps ไม่ใส่ใน diagram |
| 2 | 1-B | Claude Code + Opus 5.5 | `Follow prompts/1-recover-architecture.md` run B, evidence: `demo.proto` + `kubernetes-manifests/` | `protos/demo.proto` และทุกไฟล์ใน `kubernetes-manifests/` (13 ไฟล์ รวม `kustomization.yaml`, `README.md`) | ใช่ (อ่านเฉพาะหลักฐานที่อนุญาต) | | ได้ 12 container (เพิ่ม `frontend`, `redis-cart`, `loadgenerator`) และ 17 dependency จาก env var `*_ADDR`; ชี้ `shoppingassistantservice` เป็น gap เพราะไม่มี manifest; บอกเองว่าชื่อ RPC บนลูกศรเป็น [Inferred] |
| 3 | 2-a | Claude Code + Opus 5.5 | `Follow prompts/2-write-adr.md` run a, context: none, requirement R-12, language: Thai | (ควรมีแค่ `prompts/2-write-adr.md`, `docs/adr/template.md` — ตรวจจาก log ของ agent) | | | เลือก pub/sub แต่ไม่เลือกยี่ห้อ broker; ไม่มีตัวเลขเลย ย้ายไปไว้ใน Open questions 12 ข้อ |
| 4 | 2-b | Claude Code + Opus 5.5 | `Follow prompts/2-write-adr.md` run b, context: scenario + ADR 0001 + `checkoutservice/main.go`, requirement R-12, language: Thai | (ควรมีแค่ไฟล์โจทย์, template และ 3 ไฟล์ใน context — ตรวจจาก log ของ agent) | | | เลือก RabbitMQ ใช้ตัวเลขจาก scenario ครบ อ้างบรรทัดใน `main.go` ถูก แต่มีของที่ไม่อยู่ใน context เช่น frontend เป็น Go, ใช้ Locust ทำ load test |
| 5 | 3 | Claude Code + Opus 5.5 | `Follow prompts/3-review-adr.md` adr: `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md`, language: Thai | (ควรมีแค่ไฟล์โจทย์, ADR 0002 และ `docs/scenario.md` — ตรวจจาก log ของ agent) | | | ไม่ได้แก้ ADR (เวลาแก้ไขไฟล์ ADR ไม่เปลี่ยน); verdict: Accept with changes; จับได้ว่า outbox "transaction เดียวกับ order" ใช้ไม่ได้เพราะ checkout ไม่มี order DB |
| 6 | 1-A (เทียบ agent) | Antigravity CLI + Gemini 3.8 Flash (High) | ข้อความเดียวกับแถว 1 (run A, evidence: `demo.proto`) | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ | | ผลเกือบเหมือน Claude: 9 service, ไม่มี dependency, เดาไว้ใน Gaps; ต่างกันที่ Gaps ละเอียดกว่าเล็กน้อย แต่อ้างไฟล์ `kubernetes-manifests/redis.yaml` ที่ไม่มีอยู่จริง เก็บผลที่ `outputs/1A-architecture-antigravity.md` |
| 7 | 1-B (เทียบ agent) | Antigravity CLI + Gemini 3.8 Flash (High) | ข้อความเดียวกับแถว 2 (run B, evidence: `demo.proto` + `kubernetes-manifests/`) | `protos/demo.proto` และทุกไฟล์ใน `kubernetes-manifests/` | ใช่ | | 12 container และ 17 dependency ตรงกับ Claude ทุกเส้น; ใส่เลขบรรทัดเป็นหลักฐานละเอียดกว่า แต่ใส่ชื่อ RPC บนลูกศรเหมือนเป็นข้อเท็จจริง เช่น `frontend` → `SearchProducts` ซึ่งไม่มีในโค้ด frontend (Claude ติดป้าย [Inferred] ไว้) เก็บผลที่ `outputs/1B-architecture-antigravity.md` |
