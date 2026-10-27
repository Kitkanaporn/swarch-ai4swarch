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
| `loadgenerator` → `frontend` | `kubernetes-manifests/loadgenerator.yaml` (`FRONTEND_ADDR=frontend:80`) | ❓ | ลองเปิดโค้ดดูแล้ว loadgenerator เรียก frontend จริง ใน `locustfile.py` แต่พอไปดู `kustomization.yaml` บรรทัด 24 เจอว่าไฟล์นี้ถูกคอมเมนต์ปิดไว้ แปลว่าตอน deploy จริงจะไม่มีตัวนี้ |
| `checkoutservice` → `paymentservice` | `kubernetes-manifests/checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR=paymentservice:50051`) | ✅ | เปิด `src/checkoutservice/main.go` ดู เจอฟังก์ชัน `chargeCard` บรรทัด 369 ที่เรียก `Charge()` ของ paymentservice จริง คือตอนลูกค้ากดสั่งซื้อ checkout จะไปสั่งตัดเงินบัตรที่ payment ตรงกับที่ AI บอก |
| `frontend` → `checkoutservice` | `kubernetes-manifests/frontend.yaml` (`CHECKOUT_SERVICE_ADDR=checkoutservice:5050`) | ✅ | ตอนแรกหาใน `src/frontend/rpc.go` ไม่เจอ (service อื่นอยู่ในไฟล์นี้หมด) แต่ไปเจอใน `handlers.go` บรรทัด 354 แทน ที่เรียก `PlaceOrder()` ตอนลูกค้ากดปุ่มสั่งซื้อ สรุปว่ามีจริง แต่โค้ดอยู่คนละไฟล์ |
| `cartservice` → `redis-cart` | `kubernetes-manifests/cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ | เส้นนี้ไม่ได้ใช้ gRPC แต่ต่อ Redis ตรง ๆ ใน `src/cartservice/src/Startup.cs` บรรทัด 29 อ่านค่า `REDIS_ADDR` แล้วเอาไปใช้เก็บตะกร้าจริง (โค้ดเป็น C#) ที่น่าสังเกตคือ Redis ใช้ `emptyDir` (`cartservice.yaml` บรรทัด 141) ถ้า pod รีสตาร์ต ตะกร้าของทุกคนจะหาย และถ้าไม่ตั้ง `REDIS_ADDR` ระบบจะไปเก็บใน memory แทน ซึ่ง AI ไม่รู้ |
| `recommendationservice` → `productcatalogservice` | `kubernetes-manifests/recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`) | ✅ | ตัวนี้เขียนด้วย Python เปิด `recommendation_server.py` เจอว่าบรรทัด 136 สร้างตัวเชื่อมไปหา product catalog และบรรทัด 73 เรียก `ListProducts()` เพื่อดึงสินค้าทั้งหมดมาสุ่มแนะนำ ตรงกับที่ AI เขียนบนลูกศร |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **dependency ที่ไม่มีจริง:** ไม่เจอเลย ลูกศรทั้ง 17 เส้นที่ AI วาด พอเปิดโค้ดดูก็มีจริงหมด
  - **dependency ที่ขาดไป (1):** `frontend` → `shoppingassistantservice` มีอยู่ในโค้ด (`src/frontend/handlers.go` บรรทัด 464 ยิง HTTP ไปหา) และตัว shopping assistant เองยังไปใช้ AlloyDB กับ Gemini ของ Google ด้วย แต่ไม่อยู่ใน diagram เพราะไม่มีไฟล์ manifest
  - **dependency ที่ขาดไป (2):** หลาย service (เช่น `checkoutservice/main.go` บรรทัด 164) ส่งข้อมูล tracing ไปที่ OpenTelemetry collector ผ่าน `COLLECTOR_SERVICE_ADDR` เมื่อเปิด tracing ตัวนี้ AI ไม่เห็นเลย ไม่มีแม้แต่ใน Gaps เพราะใน manifest ไม่มี env var นี้ ต้องอ่านโค้ดถึงจะรู้
  - **container ที่น่าสงสัย:** `loadgenerator` ถูกวาดอยู่ในระบบ แต่จริง ๆ ถูกปิดไว้ใน `kustomization.yaml` และเป็นแค่เครื่องมือทดสอบ ไม่ควรอยู่ใน diagram ของระบบ
  - **ข้อมูลที่ขาด:** AI บอกภาษาของแต่ละ service ไม่ได้เลย แต่พอเปิด `src/` ดูก็รู้ทันที เช่น checkout เป็น Go, cart เป็น C#, recommendation กับ loadgenerator เป็น Python และ `emailservice` จริง ๆ เป็นตัว mock (`DummyEmailService`) ไม่ได้ส่งอีเมลจริง

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   **ตอบ:** จาก `.proto` อย่างเดียว AI รู้แค่ว่ามี service อะไรบ้างและแต่ละตัวมีฟังก์ชันอะไร แต่ไม่รู้ว่าใครเรียกใคร ไม่รู้ว่ามี frontend หรือ Redis และไม่รู้ภาษาที่ใช้เขียน เพราะ `.proto` เป็นแค่สัญญาว่า service "ให้บริการอะไร" เหมือนเมนูร้านอาหาร ไม่ได้บอกว่าใครมาสั่ง ส่วน manifest มี env var อย่าง `CART_SERVICE_ADDR` ที่บอกว่า service ไหนต้องรู้ที่อยู่ของใคร Run B เลยวาดลูกศรได้ 17 เส้น ขณะที่ Run A ได้ 0
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   **ตอบ:** ข้อที่น่ากลัวที่สุดคือ diagram ไม่มี OpenTelemetry collector กับ shopping assistant เพราะ diagram ดูครบและมีหลักฐานทุกเส้น คนใหม่จะเชื่อว่านี่คือระบบทั้งหมด ถ้าวันหนึ่งเปิด tracing หรือเปิด shopping assistant แล้วมีปัญหา เขาจะไม่รู้เลยว่ามีระบบพวกนี้อยู่ และยังไม่รู้ว่า shopping assistant ไปเรียก Gemini กับ AlloyDB ข้างนอกด้วย ซึ่งเกี่ยวกับค่าใช้จ่ายและความปลอดภัยของข้อมูล
