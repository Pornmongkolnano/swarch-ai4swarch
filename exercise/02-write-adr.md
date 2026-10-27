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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่มีตัวเลข เพราะไม่มี context; เปิดคำถามเรื่องโหลด/latency/SLA ไว้ | ระบุ checkout p95 ≤ 500 ms จากประมาณ 300 ms, พีก 200 orders/s, MailFast p95 1.5 s/p99 8 s, error 0.5%, 50/s และแต้มภายใน 1 นาที; คำนวณ backlog อย่างน้อย 150/s |
| ทางเลือกที่พิจารณา (ระบุ) | synchronous โดยตรง; durable async coordinator; event ผ่าน broker และเลือกข้อหลังโดยยังไม่เลือกผลิตภัณฑ์ | synchronous gRPC; durable outbox + RabbitMQ; durable job table + worker กลาง และเลือก RabbitMQ |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | เสนอ at-least-once + กันซ้ำด้วย eventId/orderId และบันทึกผลร่วมกับ dedup; ยังถาม rate limit/provider idempotency และไม่มีกลไกจำกัดอัตราที่ระบุแน่นอน | ระบุ unique orderId และเพิ่มแต้ม transaction เดียวก่อน ack; แยก exactly-once effect จาก delivery; limiter รวม replica/retry ≤ 50/s, แยก queue และเตือน backlog/ข้อจำกัด SLA |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่พบตัวเลขแต่ง; OrderPlaced, broker, outbox และ idempotency เป็นข้อเสนอที่ระบุชัด ไม่ใช่ implementation เดิม | RabbitMQ/outbox/order store, event fields, limiter และ DLQ เป็นข้อเสนอใหม่ที่ติดป้ายไว้ ไม่ใช่ข้อเท็จจริงเดิม; 150/s เป็นผลคำนวณ ส่วน storage และ payment/shipping recovery ยังเปิดคำถาม ไม่ควรตีความว่าแก้แล้ว |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เลือก PostgreSQL แบบ self-hosted สำหรับ order/state/outbox และระบุ completion กับ outbox ต้อง commit ร่วมกัน | Run b มี atomic outbox อยู่แล้วแต่ยังไม่เลือก store; ปิดตัวเลือกส่วนนี้ให้มี transaction/unique constraint ชัด และแยกว่าเป็นส่วนเพิ่มใหม่ |
| 2 | ขยาย request idempotency เป็น key + payload fingerprint + stable orderId พร้อมพฤติกรรม retry/ปฏิเสธ payload ต่าง และเพิ่มหลักฐานจาก demo.proto | Run b เตือน payment/shipping crash gap แล้ว; ฉบับสุดท้ายทำ recovery gate ให้ชัดว่า local outbox ไม่ครอบคลุม RPC และ API เดิมยังไม่มี key |
| 3 | เพิ่มการตรวจ unroutable publish และ binding ทั้งสามก่อน mark outbox sent | Run b มี confirm/durable queue แล้ว; confirm เพียงอย่างเดียวไม่อธิบายการส่งไปครบ queue จึงเพิ่มเงื่อนไข routing เพื่อไม่ถือว่าส่งสำเร็จทั้งที่ไม่มีผู้รับ |
| 4 | ระบุ limiter ต้องใช้ permit ร่วมแบบ atomic และ fail closed เมื่อใช้ไม่ได้ | Run b มี global limiter รวม replica/retry อยู่แล้ว; ขยายพฤติกรรม failure ให้ป้องกัน fallback ไปจำกัดเฉพาะ replica โดย quota window จริงยังต้องยืนยัน |
| 5 | กำหนดบทบาท checkout/consumer owners และเสนอ RabbitMQ primary/backup ภายในทีม 4 คน พร้อม runbook/restore | Run b ยังถามผู้รับผิดชอบไว้; เพิ่มความรับผิดชอบที่นำไปตกลงได้ โดยไม่สมมติว่ามีทีม operations เพิ่มหรือมีการมอบหมายจริงแล้ว |

ADR ฉบับสุดท้ายที่เป็นร่างเสนอ: [0002-use-rabbitmq-and-transactional-outbox.md](../docs/adr/0002-use-rabbitmq-and-transactional-outbox.md) — สถานะ Proposed; Validation เป็นแผนที่ยังไม่ได้รัน และยังต้องปิด Open questions ก่อนใช้งานจริง
