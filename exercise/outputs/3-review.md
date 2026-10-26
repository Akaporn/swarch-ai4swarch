> Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 Publish `OrderPlaced` to RabbitMQ (ATAM-style)

อ่านเฉพาะ: `docs/adr/0002-publish-orderplaced-to-rabbitmq.md` (อ้างอิงเป็น `ADR:<บรรทัด>`) และ `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QA1 Performance | ลูกค้าช่วง 11.11 | `PlaceOrder` 200 ครั้ง/วินาที ต่อเนื่อง | peak, MailFast ทำงานที่ rate limit 50/s | order ถูกรับและบันทึก `OrderPlaced` ลง outbox | p95 `PlaceOrder` ≤ 500 ms; MailFast ถูกเรียก ≤ 50 emails/s, ไม่มีอีเมลถูกปฏิเสธเพราะ rate limit |
| QA2 Availability | `loyaltyservice` / MailFast | `loyaltyservice` ล่ม 30 วินาที; MailFast fail 0.5 % | load ปกติ | order สำเร็จ, message รอใน queue, retry อีเมล | order สำเร็จ 100 %; points ได้ภายใน 60 s หลังสั่งซื้อ และ exactly once |
| QA3 Reliability | infrastructure | pod `checkoutservice` crash หลัง `shipOrder` หรือ PostgreSQL ใช้ไม่ได้ | peak 200 orders/s | event ยังถูก publish ในที่สุด | 0 order ที่ charge แล้วแต่ไม่ได้ points; ไม่มี points ซ้ำ |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | outbox ไม่ได้ "transactional" จริง: `chargeCard`/`shipOrder` เป็น gRPC ภายนอก ไม่มี local transaction ร่วมกับการเขียน outbox (ADR:62-64, 121) | crash ระหว่าง `shipOrder` กับการเขียน outbox ทำให้ event หาย ลูกค้าโดน charge แต่ไม่ได้ points/อีเมล ขัดกับ QA3 |
| Risk | ถ้าเขียน outbox ไม่สำเร็จ "order ยังต้องสำเร็จ" + แค่ alert (ADR:63) | PostgreSQL ล่ม 1 นาทีที่ 200/s = 12,000 order ไม่มี event เลย → loyalty ได้ "zero times" ไม่ใช่ exactly once |
| Risk | mitigation "alert ตรวจ order ที่ไม่มี event" (ADR:121) ไม่มีแหล่งข้อมูลให้เทียบ: outbox คือที่เดียวที่บันทึก order ของ `checkoutservice` | mitigation นี้ทำไม่ได้ตามที่เขียน จึงไม่ลดความเสี่ยงจริง |
| Risk | ทีม 4 คนต้องดูแล stateful component ใหม่ **สองตัว** (PostgreSQL + RabbitMQ 3 node) และ relay ที่ไม่ระบุ HA (ADR:64, 69) | relay ตัวเดียว = SPOF ของ post-order; หลาย replica = publish ซ้ำ; on-call ยังไม่มีเจ้าของ (ADR:146) |
| Sensitivity point | concurrency/prefetch ของ email worker: ตาม Little's law ต้องมี in-flight ≈ 50 × 1.5 s = **75** ที่ p95 และถึง **400** ที่ p99 (8 s) เพื่อให้ได้ 50/s | ถ้าตั้ง prefetch ต่ำ throughput จริงจะต่ำกว่า 50/s มาก และ backlog ช่วง 11.11 โตเร็วกว่าที่ ADR คำนวณ (ADR:117) |
| Sensitivity point | การจำกัด 50/N ต่อ replica (ADR:124) ขึ้นกับ N ที่เปลี่ยนตาม autoscaling | N เปลี่ยน → เกิน rate limit หรือใช้ไม่เต็ม; retry ของ 0.5 % ก็กิน quota เดียวกัน |
| Sensitivity point | polling interval ของ outbox relay และ throughput ของ `loyaltyservice` (ต้อง ≥ 200/s) | ทั้งสองค่าตัดสินว่า loyalty ทันเป้า 60 s ที่ peak หรือไม่ ADR ไม่ระบุค่า |
| Trade-off point | outbox write timeout 100 ms แบบ fail-open (ADR:63) | ได้ availability/latency ของ checkout แต่เสีย reliability และ exactly once ของ loyalty |
| Trade-off point | buffer ด้วย queue แทน sync call | ได้ latency + availability แต่อีเมลช้าลงช่วง peak: backlog +150/s, drain 3 s ต่อ 1 s ของ peak (ADR:117) — peak 1 ชม. = 540,000 ฉบับ, drain ~3 ชม. |
| Non-risk | queue แยกต่อ action (ADR:65-68) | loyalty ไม่ติด backlog อีเมล ทำให้เป้า 60 s เป็นไปได้แม้อีเมลค้างหลายชั่วโมง |
| Non-risk | idempotent ด้วย `order_id` (UUID จาก ADR:19) แล้วค่อย ack (ADR:67) | at-least-once + dedupe ถูกต้อง (ถ้าบันทึก dedupe กับ points ใน transaction เดียวกัน) |
| Non-risk | เลือก RabbitMQ แทน Kafka (ADR:71) | ตรงทักษะทีม (2 คนเคยใช้), มี DLQ/prefetch ในตัว; replay ยังไม่ใช่ requirement |
| Non-risk | แยก fraud scoring ออก (ADR:126) | ถ้าต้องบล็อก order ต้องเป็น sync ก่อน `chargeCard` จับประเด็นได้ถูก |

## 3. Missing or weak

- **Option ที่ไม่ได้พิจารณา:** (a) เขียน outbox แถว `pending` **ก่อน** `chargeCard` แล้วเปลี่ยนเป็น `confirmed` หลัง `shipOrder` ทำให้ crash ระหว่างทางกู้คืนได้; (b) publish ตรงไป RabbitMQ ด้วย publisher confirms โดยไม่มี PostgreSQL — เมื่อ outbox ไม่ transactional อยู่แล้ว ข้อดีที่ได้เพิ่มจากการมี datastore ตัวที่สองต้องอธิบายให้ชัด
- **Claim ที่ไม่มีหลักฐาน/น่าจะผิด:** "ใช้ transactional outbox" (ADR:121); "ช่วงเวลานี้สั้นมาก จึงยอมรับได้" ไม่มีตัวเลข; "เพิ่ม action ใหม่โดยไม่ต้องแก้ `checkoutservice`" (ADR:115) จริงเฉพาะเมื่อ payload มีข้อมูลพอ — SMS ต้องใช้เบอร์โทรซึ่งไม่อยู่ใน payload (ADR:62) และไม่มีเรื่อง schema versioning
- **Consequence ที่ไม่ได้ระบุ:** PostgreSQL อยู่ใน hot path ของ `PlaceOrder` (latency + SPOF ใหม่); payload มี `email` (PII) อยู่ใน broker/DLQ; อีเมลถูกส่งเกินลำดับเวลาได้เมื่อ retry
- **Validation ไม่ครอบคลุม:** ไม่มี failure test สำหรับ PostgreSQL ล่ม, relay crash, RabbitMQ เสีย 1 node, หรือ kill pod ระหว่าง `shipOrder` กับการเขียน outbox; load test 200/s ไม่ได้กำหนดระยะเวลา peak (ADR:130, 139)
- **Language ของ `emailservice`:** ใช้ Python ต่อ (ADR:66) ทั้งที่ทีมเก่ง Go — ไม่ได้ชั่งน้ำหนักกับ team fit (driver 5)

## 4. Verdict: **Accept with changes**

1. **ปิดช่องว่าง event หาย:** บันทึก outbox `pending` ก่อน `chargeCard` และ `confirmed` หลัง `shipOrder`, มี reconciler สำหรับแถวค้าง, และกำหนดพฤติกรรมเมื่อ outbox เขียนไม่ได้ (อัตราสูญเสียที่ยอมรับได้ หรือ fallback) แทน fail-open แบบเงียบ — ไม่เช่นนั้นห้ามอ้าง exactly once
2. **ระบุ capacity ของฝั่งอีเมล:** global rate limiter 50/s ที่ไม่ขึ้นกับจำนวน replica (รวม retry), prefetch/concurrency ≥ 75–400, และ SLO ความช้าของอีเมลกับขนาด disk ของ RabbitMQ ตามระยะเวลา peak จริง
3. **ขยาย Validation และ ops:** failure test ตาม QA3 (PostgreSQL, relay, RabbitMQ node, pod kill), sustained-peak test ที่วัด loyalty p99 < 60 s, ระบุ HA ของ relay, schema version ของ `OrderPlaced`, และเจ้าของ on-call ของ PostgreSQL/RabbitMQ
