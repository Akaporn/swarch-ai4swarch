**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต: อรรฆพร อึ้งสิทธิพูนพร**
**agent ที่ใช้: Claude Code** 
**โมเดล: Opus 5.5** …
**แพ็กเกจที่ใช้: Pro** 

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code Opus 5.5 | `prompts/1-…` run A | exercise/01-recover-architecture.md, CLAUDE.md, demo.proto, AGENTS.md | yes แต่ผิดกฎห้ามใช้ terminal | 3 | agent รันคำสั่ง terminal (find/ls) ซึ่ง AGENTS.md ห้ามไว้ คำสั่งนั้นแค่แสดงรายชื่อไฟล์และโฟลเดอร์ agent ไม่ได้อ่านไฟล์อื่นเพิ่ม |
| 2 | 1-B | Claude Code Opus 5.5 | `prompts/1-…` run B | demo.proto, kubernetes-manifests/: frontend.yaml, checkoutservice.yaml, cartservice.yaml, recommendationservice.yaml, loadgenerator.yaml, kustomization.yaml, adservice.yaml, currencyservice.yaml, emailservice.yaml, paymentservice.yaml, productcatalogservice.yaml, shippingservice.yaml, prompts/1-recover-architecture.md, CLAUDE.md, AGENTS.md | No | 4 | Run B เปิด worksheet exercise/01-recover-architecture.md ผ่าน shell command ซึ่งอยู่นอก evidence |
| 3 | 2-a | Claude Code Opus 5.5 | `prompts/2-…` run a, context: none, language: Thai | prompts/2-write-adr.md, AGENTS.md, CLAUDE.md, docs/adr/template.md | yes | 2 | ไม่ใช้ตัวเลขจาก scenario เลย ใช้ "TBD" และ "Not evidenced" แทน มี Open questions 11 ข้อ ซึ่งส่วนใหญ่ scenario ตอบไว้แล้ว ใช้คำว่า "exactly once" แม้ไม่ได้เปิด scenario.md |
| 4 | 2-b | Claude Code Opus 5.5 | `prompts/2-…` run b, context: scenario.md + ADR 0001 + checkoutservice/main.go, language: Thai | prompts/2-write-adr.md, AGENTS.md, CLAUDE.md, docs/adr/template.md, docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go | yes | 4 | ใช้ตัวเลขจาก scenario ครบ อ้างบรรทัดใน main.go ถูกทุกจุด แต่ระบุว่า emailservice เป็น Go ซึ่งผิด (จริงๆ เป็น Python) และใช้ retry buffer ใน memory ซึ่งขัดกับ exactly once |
| 5 | 3 | Claude Code Opus 5.5 | `prompts/3-…` adr: docs/adr/0002-publish-orderplaced-to-rabbitmq.md, language: Thai | prompts/3-review-adr.md, AGENTS.md, CLAUDE.md, docs/adr/0002-publish-orderplaced-to-rabbitmq.md, docs/scenario.md | yes | 4 | ไม่ได้แก้ ADR อ้างเลขบรรทัด ADR ถูกทุกจุด จับได้ว่า outbox ไม่ได้ transactional จริง แต่รีวิวยาวเกินหนึ่งหน้า (13 findings) |
| 6 | | | | | | | |
| 7 | | | | | | | |
