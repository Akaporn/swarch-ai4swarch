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
| `checkoutservice` → `cartservice` | `checkoutservice.yaml` (`CART_SERVICE_ADDR=cartservice:7070`) | ✅ | env var มีจริงใน `checkoutservice.yaml` และในโค้ดเรียกจริงที่ `src/checkoutservice/main.go:325` (`GetCart`) และ `:333` (`EmptyCart`) |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR=paymentservice:50051`) | ✅ | เรียก `Charge` ที่ `main.go:370` แต่ `paymentservice` เป็น mock: `src/paymentservice/charge.js:85` แค่สุ่ม uuid ไม่ได้ต่อ payment gateway จริง |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | เรียก `SendOrderConfirmation` ที่ `main.go:380` และ port 5000 → 8080 ตรงกับ `emailservice.yaml:82-83` |
| `frontend` → `shippingservice` | `frontend.yaml` (`SHIPPING_SERVICE_ADDR=shippingservice:50051`) | ✅ | AI ระบุ label ว่าเป็น [Inferred] แต่ในโค้ดเรียก `GetQuote` จริงที่ `src/frontend/rpc.go:88` |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | อ่าน env var และสร้าง `ProductCatalogServiceStub` ที่ `src/recommendationservice/recommendation_server.py:131-136` |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ | `redis-cart` ถูกนิยามอยู่ใน `cartservice.yaml` เอง (ไม่มีไฟล์แยก) และใช้ `emptyDir` (`:141`) จึงทำให้ข้อมูล cart หายเมื่อ pod restart |


- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
   - ไม่พบ dependency ที่ไม่มีจริง: ทั้ง 17 รายการมีหลักฐาน และ 15 รายการที่เป็น service เรียก service มีทั้ง env var และการเรียกจริงในโค้ด
  - **container ที่น่าสงสัย:** `loadgenerator` ถูกวาดไว้ใน System_Boundary แต่เป็นเครื่องมือสร้าง traffic ทดสอบ และไม่ถูก deploy โดย default (ถูก comment ไว้ใน `kustomization.yaml:24`)
  - **คำอธิบายที่ทำให้เข้าใจผิด:** `emailservice` ถูกอธิบายว่า "Sends order confirmations" แต่โค้ดใช้ `DummyEmailService` (`email_server.py:108`) และโหมดจริงยังไม่ได้ implement (`:123`) จึงไม่ได้ส่งอีเมลจริง
  - **dependency ที่ขาดไป:** `frontend` → `shoppingassistantservice` (HTTP, `src/frontend/handlers.go:464`) ใช้เฉพาะเมื่อเปิด `ENABLE_ASSISTANT` และไม่มี manifest ซึ่ง AI ใส่ไว้ใน Gaps ถูกต้องแล้ว นอกจากนี้ยังมี OpenTelemetry collector (`COLLECTOR_SERVICE_ADDR` ใน `checkoutservice/main.go:164`, `frontend/main.go:178`) ที่ไม่อยู่ใน manifest เลย AI จึงมองไม่เห็น
  - **ภาษาของแต่ละ service:** AI ตอบว่า "Not evidenced" ซึ่งซื่อตรงต่อ evidence แต่จริงๆ ระบบใช้หลายภาษา: Go, Node.js, Python, Java, C#


## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   - ไฟล์ `.proto` บอกแค่ว่าแต่ละ service *ให้บริการ* อะไร แต่ไม่บอกว่าใคร *เรียก* ใคร Run A จึงได้ 9 container แต่ไม่มี dependency เลย นอกจากนี้ยังไม่เห็น `frontend` (เพราะให้บริการผ่าน HTTP ไม่มีใน proto), ไม่เห็น `redis-cart` และไม่เห็นข้อมูลการ deploy ส่วน Run B ได้ 12 container และ 17 dependency เพราะ env var `*_ADDR` ใน manifest เผยให้เห็นว่าแต่ละ service ต่อกับใคร

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
   - คำอธิบาย `emailservice` ว่า "Sends order confirmations" ทั้งที่จริงเป็น dummy ที่ไม่ได้ส่งอีเมลเลย สมาชิกใหม่อาจเข้าใจว่าระบบส่งอีเมลยืนยันให้ลูกค้าได้แล้ว และประเมินงานเชื่อมต่อ email provider จริง (เช่น MailFast ใน Task 2) ต่ำเกินไป ข้อนี้ตรวจเจอได้จากการอ่านโค้ดเท่านั้น manifest อย่างเดียวไม่พอ

