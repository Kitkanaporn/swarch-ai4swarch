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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย เพราะไม่ได้อ่าน scenario ตัวเลขทุกอย่าง (ปริมาณ order, latency, SLA ของ MailFast) ถูกย้ายไปไว้ใน Open questions แทน ไม่ได้เดาค่าเอง | ใช้ครบ มีตารางตัวเลข: p95 ≤ 500 ms (ตอนนี้ 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / ล้ม 0.5% / 50 emails/s, แต้มภายใน 1 นาที แล้วเอาไปคำนวณต่อ เช่น ถ้าเรียก MailFast ตรง ๆ p95 จะเป็น ~1.8 s และช่วงพีก email จะค้าง ~150 ฉบับ/วินาที |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทาง: (A) เรียกทุกอย่างแบบ synchronous ใน checkout (B) publish/subscribe ผ่าน message broker (C) ทำ post-order orchestrator แยก → เลือก B แต่**ไม่เลือกยี่ห้อ broker** | 4 ทาง: (1) sync gRPC fan-out (2) ยิง goroutine แบบ async ใน checkout (3) RabbitMQ (4) Kafka → เลือก RabbitMQ เพราะทีมเคยใช้และ self-hosted ได้ ตัวเลือกเจาะจงกับสถานการณ์มากกว่า |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: บอกกว้าง ๆ ว่า consumer ต้อง idempotent ใช้ order ID กันซ้ำ / rate limit: พูดแค่ว่า "ถ้า MailFast ล่มหรือ rate-limit ให้ retry + dead-letter queue" ไม่รู้ว่าจำกัดที่ 50/s จึงไม่ได้ออกแบบรองรับ | แต้ม: `loyaltyservice` เก็บ `order_id` ที่ทำแล้วใน transaction เดียวกับการเพิ่มแต้ม (unique constraint), แยก queue ไม่ให้ติดคิว email, alert ถ้าค้างเกิน 30 s / rate limit: ใช้ queue เป็น buffer แล้วส่งออกด้วย rate limiter 50/s รวมทุก replica + retry + DLQ |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | แทบไม่มี สิ่งที่เดาติดป้าย [Inferred] ไว้ (เช่น "checkout น่าจะเรียก emailservice") ยกชื่อ Kafka / Pub/Sub แค่เป็นตัวอย่าง | มีบ้าง: บอกว่า frontend เขียนด้วย Go และจะใช้ Locust `loadgenerator` ทำ load test ซึ่งไม่มีในไฟล์ที่ให้อ่าน (น่าจะมาจากความรู้เดิม), ตัดสินใจเองว่าจะสร้าง `email-notifier` เป็น Go service ใหม่แทน `emailservice` ทั้งตัว และเดาว่า email อาจช้า "หลายนาทีถึงหลายชั่วโมง" ส่วน RabbitMQ 3 nodes ติดป้าย [Inferred] ไว้ |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่ม transactional outbox (PostgreSQL) ให้ `checkoutservice` ใน Decision, Risks, Consequences และ diagram แล้วลบข้อ Outbox ออกจาก Open questions | ร่างของ AI บอกว่าถ้าส่ง event ไป RabbitMQ ไม่สำเร็จ order ก็ยังสำเร็จต่อ แปลว่า event หายได้ ลูกค้าจ่ายเงินแล้วแต่ไม่ได้แต้ม ซึ่งผิด requirement ที่ต้องได้แต้ม AI รู้ปัญหานี้แต่ไม่ได้ตัดสินใจ ปล่อยไว้ใน Open questions |
| 2 | ลบข้อมูลที่ไม่มีหลักฐาน: "frontend เป็น Go" ใน diagram และ "ใช้ Locust `loadgenerator`" ใน Validation | ไฟล์ที่ให้ AI อ่าน (scenario, ADR 0001, `checkoutservice/main.go`) ไม่ได้บอกเรื่องนี้ AI เดาเอาเอง ADR ควรมีแต่ข้อมูลที่มีหลักฐาน |
| 3 | ใช้ `emailservice` เดิม (เปลี่ยนจาก mock เป็น consumer ที่อ่านจาก queue) แทนการสร้าง `email-notifier` ใหม่ | requirement บอกให้แทนแค่ logic ที่เป็น mock ใน `emailservice` ไม่ได้บอกให้สร้าง service ใหม่ และทีมมีแค่ 4 คน การแก้ตัวเดิมใช้แรงน้อยกว่าและไม่ต้อง deploy ของใหม่เพิ่ม |
