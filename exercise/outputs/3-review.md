> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish an "OrderPlaced" event to RabbitMQ

ADR ที่รีวิว: `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md` · ข้อเท็จจริงจาก `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | QA | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|---|
| S1 | Performance | ลูกค้าช่วง 11.11 | `PlaceOrder` 200 ครั้ง/s | peak, MailFast ปกติ (p95 1.5 s) | charge + ship + เขียน outbox แล้วตอบกลับทันที ไม่รอ post-order | p95 `PlaceOrder` ≤ 500 ms (ปัจจุบันประมาณ 300 ms → budget สำหรับ outbox insert ประมาณ 200 ms) |
| S2 | Availability | MailFast / `loyaltyservice` / RabbitMQ | ล่มทั้งหมด 10 นาที | peak 200 orders/s | order ยังสำเร็จ; event ค้างใน outbox/queue แล้วส่งต่อเมื่อกลับมา | 0 order fail เพราะ downstream; 0 event หาย |
| S3 | Reliability | RabbitMQ | redeliver ข้อความเดิมหลัง consumer crash ก่อน ack | peak 200 msg/s เข้า loyalty queue | `loyaltyservice` กันซ้ำด้วย `order_id` | points ได้ภายใน 60 s (p99), ซ้ำ = 0, ขาด = 0 เมื่อ reconcile |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Outbox "ใน transaction เดียวกับการสร้าง order" แต่ ADR เองบอกว่า `checkoutservice` ไม่มี data store — ไม่มี order record ให้ทำ transaction ร่วม | การรับประกัน "ไม่มี order ไหนไม่ได้ points" ไม่จริง: charge + ship เป็น gRPC ภายนอก ไม่อยู่ใน DB transaction ถ้า insert outbox fail หลัง charge แล้ว ADR ไม่บอกว่า order สำเร็จ (event หาย) หรือ fail (ลูกค้าถูกตัดเงิน) |
| Risk | PostgreSQL outbox กลายเป็น dependency ใหม่บน critical path | ขัดกับ S2 ในทางอ้อม: Postgres ล่ม = checkout ล่ม ไม่มีอยู่ใน fault-injection plan (Validation ทดสอบแค่ email/loyalty/analytics/RabbitMQ) |
| Risk | Email backlog: เข้า 200/s ออก 50/s → โต 150/s = 540,000 emails ต่อ 1 ชม. ของ peak, ใช้เวลา drain ประมาณ 3 ชม. | ไม่มี email latency target และไม่รู้ความยาว peak จึงยังพิสูจน์ไม่ได้ว่า "ยอมรับได้"; ต้อง size disk ของ RabbitMQ จากตัวเลขนี้ |
| Risk | Dead-letter สำหรับ loyalty ไม่มีขั้นตอน replay/reconcile | ข้อความที่ลง DLQ = points ไม่ถึงภายใน 1 นาที ผิดเงื่อนไข R-12 แบบเงียบ ๆ |
| Sensitivity point | อัตรา rate limiter 50/s "รวมทุก replica" | เกิน 50/s → MailFast ปฏิเสธ → retry storm; ADR ไม่ระบุกลไก (consumer เดียว? distributed limiter? prefetch?) |
| Sensitivity point | Relay polling interval + throughput ของ `loyaltyservice` consumer | lag รวม (outbox → relay → queue → consumer) ต้อง < 60 s ที่ 200 msg/s; alert queue age 30 s ไม่ครอบคลุม lag ใน outbox |
| Trade-off point | Async ผ่าน broker + outbox | ได้ performance/availability ของ checkout แต่เสีย operability/cost (Postgres + relay + RabbitMQ 3 nodes สำหรับทีม 4 คน) และ email กลายเป็น eventual |
| Non-risk | แยก queue ต่อ action (email vs loyalty) | loyalty ไม่ติด rate limit 50/s ของ MailFast → ทำ 1 นาทีได้แม้ email ค้างเป็นชั่วโมง |
| Non-risk | Idempotent consumer ด้วย unique constraint บน `order_id` ใน transaction เดียวกับการเพิ่ม points | at-least-once + dedup ใน DB เดียวกัน = exactly-once effect ที่ถูกต้องตามหลัก |

## 3. Missing or weak

- **Option ที่ไม่ได้พิจารณา:** (a) ในเมื่อเพิ่ม PostgreSQL อยู่แล้ว ใช้ outbox table เป็น queue ให้ consumer poll โดยตรง (ไม่มี RabbitMQ) — ลด component สำหรับทีม 4 คนที่ 200 msg/s; (b) publish ตรงไป RabbitMQ พร้อม publisher confirms โดยไม่มี outbox แล้ววัดว่า event loss ยอมรับได้แค่ไหน — ADR กระโดดไป outbox โดยไม่เทียบ cost
- **ไม่สอดคล้องภายใน:** Decision ข้อแรกบอก "publish ไป exchange แทน `sendOrderConfirmation`" และ diagram มี `Rel(checkout, mq, …)` ตรง แต่ข้อสุดท้ายบอกผ่าน outbox + relay; relay ไม่ปรากฏใน diagram
- **Validation วัดผิดจุด:** "วัดเวลา publish แยก" — กับ outbox สิ่งที่อยู่ใน critical path คือ DB insert ไม่ใช่ AMQP publish
- **Consequence ที่ไม่ได้ระบุ:** event มี email + items ของลูกค้า (PII) อยู่ใน 3 queues และ analytics; schema/versioning ของ `OrderPlaced` เมื่อเพิ่ม consumer ปีหน้า
- **Claim ที่อ่อน:** Kafka "rate limiting / retry ต้องเขียนเองมากกว่า" ไม่มีเหตุผลประกอบ; quorum queues 3 nodes ระบุ [Inferred] เองแล้ว ถูกต้องแต่ยังไม่มี owner

## 4. Verdict: **Accept with changes**

1. **แก้การออกแบบ outbox** ให้ตรงกับระบบจริง: กำหนดว่าเขียนอะไรลง DB, ลำดับเทียบกับ charge/ship, และ policy เมื่อ insert fail หลัง charge; เพิ่ม relay ใน diagram และใส่ Postgres ใน fault-injection + load test (S1, S2)
2. **ปิดประเด็น email backlog:** ได้ email latency target และความยาว peak จาก marketing, คำนวณขนาด queue/disk, ระบุกลไก global rate limiter 50/s และเจรจาเพิ่ม limit กับ MailFast
3. **ปิดช่อง loyalty 1 นาที / exactly-once:** วัด end-to-end lag ตั้งแต่ outbox, กำหนดขั้นตอน DLQ replay + reconcile orders เทียบ points (S3)
