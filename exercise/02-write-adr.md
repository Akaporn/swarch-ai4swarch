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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ | ใช้ครบ |
| ทางเลือกที่พิจารณา (ระบุ) | มี 3 ตัวเลือก (sync / goroutine / broker แบบไม่ระบุยี่ห้อ) | มี 4 ตัวเลือก (เพิ่มการแยก RabbitMQ กับ Kafka) |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | พูดถึง idempotency แบบทั่วไป | ใช้ order_id เป็น dedupe key และจำกัดอัตราที่ 50/s |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่แต่งตัวเลข ใช้ "TBD" และ "Not evidenced" แทน (RabbitMQ และ Kafka ยกมาเป็นแค่ตัวอย่าง) | ไม่แต่งตัวเลขของ scenario แต่ระบุว่า emailservice เป็น Go ซึ่งผิด (จริงๆ เป็น Python และไม่อยู่ใน context ที่ให้) และเลือกรายละเอียดเองโดยไม่ได้บอกว่าเป็นการเลือก เช่น RabbitMQ 3 node, quorum queue, PodDisruptionBudget |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เปลี่ยนจาก retry buffer ใน memory เป็น transactional outbox (เพิ่ม datastore ให้ `checkoutservice` และ relay ส่งไป RabbitMQ) และลบ Open question ข้อ 4 | buffer ใน memory หายเมื่อ pod crash ทำให้ลูกค้าไม่ได้ loyalty points ซึ่งขัดกับข้อกำหนด exactly once |
| 2 | ใช้ `emailservice` เดิมเป็น consumer ของ queue แทนการสร้าง "email worker" ใหม่ และแก้ภาษาให้ถูกต้อง และลบ Open question ข้อ 8 | R-12 ระบุให้แทน logic mock ใน `emailservice` ไม่ใช่สร้าง service ใหม่ และ AI ระบุภาษาว่า Go ซึ่งผิด (เป็น Python ตาม Task 1) |
| 3 | กำหนด timeout เป็น 100 ms แทน "timeout สั้น" | p95 ของ `PlaceOrder` ต้อง ≤ 500 ms และปัจจุบันประมาณ 300 ms จึงเหลือ budget ประมาณ 200 ms ถ้าไม่มีตัวเลขก็ทดสอบไม่ได้ |

