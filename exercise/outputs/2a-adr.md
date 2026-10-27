> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "order placed" event to a message broker for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** ไม่ได้ระบุใน requirement (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ post-order actions เพิ่มเติม 3 อย่าง (จาก requirement):

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** ซึ่งเป็น external email provider รายใหม่ แทน logic แบบ mock ใน `emailservice`
2. ให้คะแนนสะสม (loyalty points) ผ่าน `loyaltyservice` ที่ต้องสร้างขึ้นใหม่
3. บันทึก analytics event "order placed" ให้ทีม marketing

นอกจากนี้ marketing คาดว่าจะมี post-order actions เพิ่มขึ้นในปีหน้า (SMS, fraud scoring, recommendations update)

ปัญหาที่ต้องตัดสินใจ: **ระบบควรกระจายเหตุการณ์ "สั่งซื้อสำเร็จ" ไปยัง post-order actions เหล่านี้อย่างไร** ให้การสั่งซื้อไม่ช้าลงหรือล้มเหลวเพราะ dependency ภายนอก (MailFast) และเพิ่ม action ใหม่ได้โดยไม่ต้องแก้ service ที่ทำ checkout ทุกครั้ง

ข้อเท็จจริงที่ทราบจาก requirement เท่านั้น:

- MailFast เป็นระบบภายนอก ทีมเราควบคุม availability และ latency ของมันไม่ได้
- `loyaltyservice` ยังไม่มีอยู่ จึงออกแบบ interface ได้อิสระ
- จำนวน consumer ของเหตุการณ์นี้จะเพิ่มจาก 3 เป็นอย่างน้อย 6

ตัวเลขที่เกี่ยวข้อง (ปริมาณคำสั่งซื้อ, latency budget ของ checkout, SLA ของ MailFast) **ไม่ได้ระบุใน requirement**: ดู Open questions

**[Inferred]** การสั่งซื้อน่าจะทำโดย service ที่ประสานงาน checkout (เช่น `checkoutservice`) ซึ่งเรียก `emailservice` อยู่แล้ว ข้อนี้ไม่ได้อ่านจากไฟล์ใด (context: none) จึงต้องตรวจสอบกับโค้ดจริง

## Decision drivers

เรียงจากสำคัญที่สุด (ลำดับนี้เป็นการตีความของผู้เขียนจาก requirement; ควรยืนยันกับทีม):

- **Modifiability / extensibility:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ checkout flow
- **Availability ของ checkout:** การสั่งซื้อต้องสำเร็จได้แม้ MailFast, `loyaltyservice` หรือ analytics ล่ม
- **Performance (latency ของ checkout):** ไม่เพิ่มเวลาที่ลูกค้ารอ ด้วยการเรียก external call แบบ synchronous
- **Reliability ของแต่ละ action:** อีเมลและ loyalty points ต้องไม่หายเงียบ ๆ (ต้อง retry ได้ และป้องกันการให้คะแนนซ้ำ)
- **Cost / operational complexity:** จำนวนชิ้นส่วนใหม่ที่ทีมต้อง deploy และดูแล

## Considered options

1. **A: Synchronous orchestration ใน checkout:** service ที่ทำ checkout เรียก email adapter (MailFast), `loyaltyservice` และ analytics ตามลำดับด้วย gRPC/HTTP ก่อนตอบลูกค้า
2. **B: Event-driven publish/subscribe:** หลังสั่งซื้อสำเร็จ publish event `OrderPlaced` ไปยัง message broker แต่ละ consumer (`emailservice` → MailFast, `loyaltyservice`, analytics) subscribe แยกกันและทำงานแบบ asynchronous
3. **C: Dedicated post-order orchestrator:** checkout ส่งคำสั่งซื้อไปยัง service ใหม่หนึ่งตัว (เช่น `postorderservice`) ผ่าน queue แล้ว service นี้เรียก action ทุกตัวตามลำดับ/workflow ที่กำหนด

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous orchestration | เรียบง่าย ไม่มี infrastructure ใหม่; ไล่ flow และ debug ได้ง่าย; รู้ผลทันทีว่าแต่ละ action สำเร็จหรือไม่ | checkout latency = ผลรวมของทุก call รวมถึง external MailFast; ถ้า action ใดล่ม checkout ช้าหรือล้มด้วย (availability ลด); เพิ่ม action ใหม่ต้องแก้และ redeploy checkout ทุกครั้ง (coupling สูง); retry ทำในเส้นทางที่ลูกค้ารออยู่ |
| B: Event-driven pub/sub | checkout publish ครั้งเดียวแล้วตอบลูกค้าได้ (latency ต่ำ); consumer ล่มไม่กระทบการสั่งซื้อ (broker เก็บ message ไว้); เพิ่ม SMS/fraud/recommendations = เพิ่ม subscriber โดยไม่แตะ checkout; scale และ retry แต่ละ consumer แยกกันได้ | ต้องมี message broker ใหม่ให้ deploy/monitor (cost, ops); eventual consistency: อีเมล/คะแนนมาช้ากว่าการสั่งซื้อ; ต้องจัดการ duplicate delivery (idempotency) และ dead-letter; ไล่ flow ข้าม service ยากขึ้น (ต้องมี tracing); ต้องดูแล schema ของ event ไม่ให้ breaking change; fraud scoring ที่อาจต้อง *บล็อก* คำสั่งซื้อไม่เข้ากับ fire-and-forget |
| C: Post-order orchestrator | แยก post-order ออกจาก checkout path (checkout เร็ว); มีที่เดียวที่เห็นและควบคุมลำดับ, retry, compensation ทั้งหมด; รองรับ action ที่ต้องเรียงลำดับหรือขึ้นต่อกัน | orchestrator กลายเป็นจุดที่ต้องแก้ทุกครั้งที่เพิ่ม action (coupling ย้ายไปอยู่ที่นี่); เป็น single point of failure ของ post-order ทั้งหมด; ยังต้องมี queue/transport ระหว่าง checkout กับ orchestrator; อาจกลายเป็น "god service" เมื่อ action โตขึ้น |

## Decision

เลือก **Option B: Event-driven publish/subscribe** ด้วย event `OrderPlaced`

เหตุผลเทียบกับ decision drivers:

- **Modifiability:** requirement ระบุชัดว่าจะมี action เพิ่มอีกอย่างน้อย 3 อย่างในปีหน้า B ให้เพิ่มได้ด้วยการเพิ่ม subscriber โดย checkout ไม่ต้องรู้จัก consumer ใด ขณะที่ A และ C ต้องแก้ศูนย์กลางทุกครั้ง
- **Availability และ performance ของ checkout:** MailFast เป็น external dependency ที่เราควบคุมไม่ได้ B ตัด MailFast ออกจากเส้นทางที่ลูกค้ารอ (เช่นเดียวกับ C แต่ไม่มี orchestrator เป็น single point of failure)
- **Reliability:** broker เก็บ event ไว้จนกว่า consumer จะประมวลผลสำเร็จ แต่ละ consumer retry ของตัวเองได้

รายละเอียดการออกแบบ:

- `emailservice` เปลี่ยนจาก mock logic เป็น adapter ที่ subscribe `OrderPlaced` แล้วเรียก MailFast
- `loyaltyservice` (ใหม่) subscribe `OrderPlaced` และให้คะแนนแบบ idempotent โดยใช้ order ID เป็น key
- analytics consumer subscribe `OrderPlaced` และส่งต่อไปยังปลายทางของทีม marketing (ปลายทางยังไม่ระบุ)
- ใช้ **transactional outbox** หรือกลไกเทียบเท่า เพื่อไม่ให้ event หายถ้า checkout สำเร็จแต่ publish ล้มเหลว **[Inferred]** จำเป็นหรือไม่ขึ้นกับว่า checkout เก็บ order ไว้ที่ใด ซึ่งไม่ได้ระบุใน requirement
- เทคโนโลยี broker (เช่น RabbitMQ, Kafka, Google Pub/Sub) **ยังไม่เลือก** ในเอกสารนี้ เพราะขึ้นกับปริมาณงาน, ความสามารถของทีม และงบ (ดู Open questions)
- **ข้อยกเว้น:** fraud scoring ในอนาคต ถ้าต้อง *อนุมัติหรือปฏิเสธ* คำสั่งซื้อก่อนยืนยัน ต้องเป็น synchronous step ใน checkout หรือเป็น ADR แยก ไม่ใช่ subscriber ของ `OrderPlaced`

## Consequences

- **Positive:** checkout ไม่ต้องรอ MailFast, `loyaltyservice` หรือ analytics จึงไม่เพิ่ม latency (quality attribute: performance)
- **Positive:** การสั่งซื้อสำเร็จได้แม้ consumer หรือ MailFast ล่ม event รออยู่ใน broker (quality attribute: availability)
- **Positive:** เพิ่ม SMS, fraud scoring (แบบไม่บล็อก) และ recommendations update ได้ด้วยการเพิ่ม subscriber โดยไม่แก้ checkout (quality attribute: modifiability / extensibility)
- **Positive:** เปลี่ยน email provider ในอนาคตกระทบแค่ `emailservice` adapter (quality attribute: modifiability)
- **Positive:** scale แต่ละ consumer แยกกันตามภาระงานของตัวเองได้ (quality attribute: scalability)
- **Negative:** ต้องมี message broker ใหม่ให้ deploy, monitor และดูแล (quality attribute: cost / operability)
- **Negative:** อีเมลยืนยันและคะแนนสะสมมาถึงช้ากว่าการสั่งซื้อ (eventual consistency) ลูกค้าอาจยังไม่เห็นคะแนนทันที (quality attribute: consistency / usability)
- **Negative:** การ debug flow ข้ามหลาย service ยากขึ้น ต้องมี distributed tracing และ correlation ID (quality attribute: observability / testability)
- **Risks and mitigations:**
  - *Duplicate delivery ทำให้ส่งอีเมลซ้ำหรือให้คะแนนซ้ำ* → consumer ทุกตัวต้อง idempotent (dedupe ด้วย order ID หรือ event ID)
  - *Event หายระหว่าง checkout กับ broker* → transactional outbox หรือ publish พร้อม retry และ alert
  - *MailFast ล่มหรือ rate-limit* → retry ด้วย exponential backoff, dead-letter queue และ alert เมื่อ DLQ ไม่ว่าง
  - *Event schema เปลี่ยนแล้วทำให้ consumer พัง* → version ของ schema (เช่น protobuf ที่เพิ่ม field แบบ backward-compatible เท่านั้น)
  - *ข้อมูลส่วนบุคคล (อีเมล, ที่อยู่) อยู่ใน event ที่ทุก consumer อ่านได้* → ใส่เฉพาะข้อมูลที่จำเป็นใน event หรือให้ consumer ดึงรายละเอียดเพิ่มเองตามสิทธิ์

## Validation

- **Load test:** วัด p95/p99 latency ของการสั่งซื้อก่อนและหลังเปลี่ยน ต้องไม่เพิ่มขึ้นเกินค่าที่ตกลงกัน (ยังไม่มีตัวเลข ดู Open questions)
- **Fault injection:** ปิด MailFast (หรือ stub), `loyaltyservice` และ analytics consumer ระหว่างทดสอบ การสั่งซื้อต้องยังสำเร็จ และเมื่อเปิดกลับมา event ที่ค้างต้องถูกประมวลผลครบ
- **Idempotency test:** ส่ง event `OrderPlaced` เดิมซ้ำ ต้องไม่มีอีเมลหรือคะแนนซ้ำ
- **Metrics ใน production:** consumer lag, ขนาด dead-letter queue, อัตราส่ง MailFast สำเร็จ, เวลาจากสั่งซื้อจนถึงส่งอีเมล
- **Extensibility check:** เมื่อเพิ่ม consumer ตัวแรกของปีหน้า (เช่น SMS) ต้องไม่ต้องแก้ service ที่ทำ checkout
- **Review date:** ไม่ได้กำหนดใน requirement (ดู Open questions)

## Open questions

ข้อมูลเหล่านี้ไม่ได้ระบุใน requirement และไม่ได้เดาค่าไว้:

1. **Deciders:** ใครเป็นผู้ตัดสินใจและอนุมัติ ADR นี้
2. **ปริมาณงาน:** จำนวนคำสั่งซื้อต่อวินาทีหรือต่อวัน ทั้งปกติและช่วง peak
3. **Latency budget ของ checkout:** p95/p99 ปัจจุบันและเป้าหมาย
4. **ความคาดหวังเรื่องเวลา:** อีเมลยืนยันต้องถึงลูกค้าภายในกี่วินาทีหรือนาที; คะแนนต้องแสดงทันทีหรือไม่
5. **MailFast:** API protocol, SLA/availability, rate limit, ราคา, รองรับ idempotency key หรือไม่, ข้อกำหนดด้าน data residency และ PII
6. **Delivery guarantee:** อีเมลและคะแนนยอมรับ at-least-once (+ idempotency) ได้หรือไม่; analytics ยอมให้ event หายได้บ้างหรือไม่
7. **ปลายทาง analytics:** ทีม marketing ใช้ระบบหรือ data warehouse ใด
8. **Broker technology และ infrastructure:** มี broker ใช้อยู่แล้วหรือไม่; ทีมมีประสบการณ์กับ RabbitMQ, Kafka หรือ managed service (เช่น Google Pub/Sub) หรือไม่; งบประมาณด้าน infrastructure
9. **Order persistence:** service ที่ทำ checkout เก็บ order ไว้ที่ใด (มีผลต่อการทำ transactional outbox)
10. **Fraud scoring:** ในอนาคตต้องบล็อกคำสั่งซื้อก่อนยืนยัน หรือแค่ให้คะแนนหลังสั่งซื้อ
11. **Loyalty rules:** กฎการคำนวณคะแนน และต้องย้อนคะแนนเมื่อมีการยกเลิกหรือคืนสินค้าหรือไม่
12. **Review date** ของ ADR นี้
