# ADR-0002: Use RabbitMQ and transactional outbox for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** พรมงคล ตันนิกร / 6631337421

## Context and problem

R-12 เพิ่ม MailFast, loyaltyservice และ analytics พร้อมคาดว่าจะเพิ่ม SMS, fraud scoring และ recommendations ในปีหน้า (docs/scenario.md: R-12) เป้าหมายคือ PlaceOrder p95 ≤ 500 ms จากปัจจุบันประมาณ 300 ms ที่พีก 200 orders/s; MailFast p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5% และจำกัด 50 emails/s; แต้มต้องลงครั้งเดียวภายใน 1 นาที งานปลายทางล่มต้องไม่ทำให้ออเดอร์ล้มเหลว ทีม 4 คนถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ ใช้ Kubernetes และต้องการ open-source self-hosted (docs/scenario.md: Quality goals and constraints)

ปัจจุบัน PlaceOrder สร้าง UUID ใหม่ เรียก chargeCard -> shipOrder -> emptyUserCart แล้วรอ sendOrderConfirmation ก่อนตอบ อีเมลล้มเหลวเพียงบันทึก warning ไม่มี durable order/outbox ในฟังก์ชันนี้ (system/online-boutique/src/checkoutservice/main.go: PlaceOrder) proto มี ChargeRequest และ ShipOrderRequest แต่ไม่มี idempotency key; PlaceOrderRequest ไม่มี request key และยังไม่มี event contract (system/online-boutique/protos/demo.proto: message ดังกล่าว) ความสามารถ idempotency ภายในปลายทางคือ **Not evidenced in the provided files**

## Decision drivers

1. **Availability/correctness:** แยกความล้มเหลวงานหลังสั่งซื้อ และไม่ให้แต้มซ้ำ
2. **Performance:** checkout p95 ≤ 500 ms; แต้ม ≤ 60 วินาที; เคารพ MailFast 50/s
3. **Modifiability:** เพิ่ม consumer โดยลดการแก้ checkout เมื่อ contract รองรับ
4. **Operability/cost:** ทีมเล็กต้องกู้คืนระบบ stateful ที่เพิ่มขึ้นได้จริง

## Considered options

1. เพิ่ม synchronous gRPC สำหรับทุกงาน พร้อม timeout
2. durable order/outbox + RabbitMQ และ consumer แยกงาน
3. durable job table + worker กลาง polling และเรียกแต่ละบริการ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| synchronous gRPC | ใช้ทักษะเดิม; ส่วนประกอบน้อย | MailFast เกินงบ latency; timeout ไม่ทำให้งานสำเร็จภายหลัง; เพิ่มงานต้องแก้ checkout |
| outbox + RabbitMQ | แยก backlog/retry/scale ของแต่ละงาน; เพิ่ม subscriber ได้ | เพิ่มฐานข้อมูล, broker และ relay; ยังต้องกันซ้ำและกู้คืน external side effect |
| job table + worker กลาง | เก็บสถานะและ retry ส่วนกลาง; ไม่เพิ่ม broker | ต้องออกแบบ polling/locking และแยกงานช้า; worker รู้จักทุก action ใหม่ |

## Decision

เลือกข้อ 2 เป็น **ข้อเสนอให้สร้างใหม่ทั้งหมด** ใช้ RabbitMQ แบบ self-hosted บน Kubernetes ตามทักษะทีม และเสนอ PostgreSQL แบบ self-hosted สำหรับ durable checkout state, order record และ outbox เพราะต้องการ transaction และ unique constraint ข้ามข้อมูลเหล่านี้ นี่ไม่ใช่ฐานข้อมูลที่ยืนยันว่าระบบเดิมมีอยู่แล้ว และไม่ใช้ความทนทานของ cart เป็นหลักฐานสำหรับออเดอร์ทางการเงิน (docs/adr/0001-use-redis-for-cart-storage.md: Decision drivers, Consequences)

- **จุดยืนยันและ API:** เสนอเพิ่ม request idempotency key ใน PlaceOrderRequest ผูกกับ fingerprint และ orderId ถาวรก่อน external side effect; key เดิมกับ payload เดิมคืนผลเดิม และ payload ต่างต้องถูกปฏิเสธ เก็บ checkout progress อย่าง durable เมื่อยืนยัน payment/shipping สำเร็จ จึง commit สถานะ order ว่า completed พร้อม OrderPlaced outbox row ใน transaction เดียว แล้วตอบลูกค้า ไม่รอ broker หรือ consumer; นำการเรียกอีเมล synchronous เดิมออก Event ต้องมี eventId, orderId, userId, occurredAt, schemaVersion และข้อมูลขั้นต่ำที่ผู้รับต้องใช้ ห้ามข้อมูลบัตรเครดิต
- **ขอบเขต atomicity:** transaction ข้างต้นครอบคลุมเฉพาะ local durable completion กับ event intent ไม่ครอบคลุม Charge/ShipOrder RPC ถ้า remote สำเร็จแต่ crash ก่อนบันทึก ต้องตรวจผลและ reconcile ก่อน retry; ห้ามอ้างว่า outbox กันการเก็บเงินหรือส่งสินค้าซ้ำได้ API ปัจจุบันยังไม่มี key เหล่านี้ การกำหนด provider idempotency/status lookup หรือ compensation และการกู้คืน cart cleanup เป็น **gate ก่อนนำใช้จริง** ที่ยังไม่ปิด ห้ามตอบ order completed ในสถานะที่ผล payment/shipping ยังไม่แน่นอน
- **การส่งเหตุการณ์:** relay อ่าน outbox แล้ว publish persistent message ไป durable exchange ซึ่ง bind durable queue แยก email, loyalty และ analytics ใช้ publisher confirm พร้อมตรวจ unroutable message; ทำเครื่องหมาย sent เมื่อยืนยัน routing/publish แล้วเท่านั้น ต้องสร้าง binding ทั้งสามก่อนเปิดใช้งาน การ crash หลัง confirm ก่อน mark sent ทำให้ส่งซ้ำได้ จึงใช้ **at-least-once** transport ไม่อ้าง broker exactly-once
- **แต้มและ analytics:** loyaltyservice มี unique constraint ที่ business key orderId สำหรับการ award และบันทึก ledger/เพิ่มแต้มใน transaction เดียวกัน แล้ว ack; redelivery/replay ต้องไม่เพิ่มแต้มซ้ำ จึงได้ exactly-once loyalty effect สำหรับออเดอร์ที่ยืนยัน ไม่ใช่ exactly-once delivery analytics บันทึก eventId unique ร่วมกับข้อมูลเหตุการณ์ใน transaction เดียวก่อน ack การแก้แต้มจาก refund ต้องเป็น action ต่างหาก ไม่ลบ dedup key เพื่อ replay
- **MailFast:** email worker ใช้ตัวควบคุมอัตราร่วมที่เก็บ permit แบบ atomic ใน shared durable store; ทุก replica และ retry ต้องขอ permit ก่อนเรียก ปรับ concurrency โดยยังเคารพเพดานรวม ≤ 50 attempts/s และหยุดส่งเมื่อ limiter ใช้งานไม่ได้ ไม่ใช้ limiter ใน memory แยก replica ใช้ timeout, backoff/jitter และเคารพ provider throttling; รูปแบบ window/ขอบเขต quota ต้องยืนยันก่อนตั้งค่าจริง การ timeout หลัง provider รับงานอาจทำให้อีเมลซ้ำ จึงต้องตรวจ idempotency/status API ไม่รับรอง exactly-once email
- **การกู้คืนและเจ้าของ:** แยก retry และ dead-letter ตามงาน; ack หลัง durable success เท่านั้น ตั้ง retry budget จาก SLA และไม่ทิ้งงานเมื่อหมด budget ผู้พัฒนา checkout รับผิดชอบ store/outbox/reconciliation; ผู้พัฒนาแต่ละ consumer รับผิดชอบผลธุรกิจและ replay; เสนอให้สองคนที่เคยใช้ RabbitMQ เป็น primary/backup ของ broker และ relay โดยแบ่งบทบาทภายในทีม 4 คน ไม่สมมติว่ามีทีม operations เพิ่ม ต้องยืนยันชื่อผู้รับผิดชอบ เวรแจ้งเตือน และ runbook ก่อน rollout รวม broker replication, persistence, backup/restore และ disk-full recovery

## Consequences

- **Positive:** checkout ไม่รอ MailFast; consumer ล่มยังค้างงานและกู้คืนแยกกันได้เมื่อ store พร้อมและมีพื้นที่ (quality attribute: performance, availability)
- **Positive:** เพิ่ม subscriber ได้ และ transaction + business uniqueness กันแต้มซ้ำเมื่อ replay (quality attribute: modifiability, correctness)
- **Negative:** เพิ่ม PostgreSQL, RabbitMQ, relay และการติดตามสำหรับทีมเล็ก; store ล่มยังทำให้ยืนยันออเดอร์ไม่ได้ จึงไม่ได้รับประกัน availability ทุก failure (quality attribute: cost, operability, availability)
- **Negative:** ที่ 200 orders/s และหนึ่งอีเมลต่อออเดอร์ backlog โตอย่างน้อย 150 emails/s ก่อน retry; เพิ่ม worker ไม่พ้น quota และยังยืนยันเวลาส่งอีเมลไม่ได้ถ้าไม่ทราบ burst duration/โหลดหลังพีก (คำนวณจาก docs/scenario.md: Peak load, MailFast; quality attribute: scalability, latency)
- **Risks and mitigations:** outage แต้มเกิน 60 วินาทีทำให้ผิดเป้าหมายแม้ไม่สูญงาน ต้องวัด order-to-award age, oldest outbox/queue age, DLQ, duplicate conflicts, MailFast attempt rate และพื้นที่เก็บ พร้อม alert ให้มีเวลาลงมือก่อนผิด SLA; retention ต้องเก็บ dedup key ครอบคลุม replay horizon (quality attribute: reliability, recoverability)
- **Risks and mitigations:** จำกัดสิทธิ์และข้อมูลต่อ queue โดยเฉพาะ email/PII; restore/replay ต้องคง eventId/orderId และห้ามเพิ่มแต้มซ้ำ (quality attribute: security, correctness)

## Validation

รายการต่อไปนี้เป็น **แผนตรวจสอบ ยังไม่ได้รันทดสอบ**:

- โหลด 200 orders/s ตาม burst duration ที่ตกลง จำลอง MailFast p95 1.5 s/p99 8 s และ error 0.5%; วัด PlaceOrder p95 ≤ 500 ms รวม durable commit และตรวจเพดาน MailFast รวมทุก replica/retry ตาม quota window จริง
- หยุด broker/consumer ขณะ store พร้อม ตรวจว่าออเดอร์ยังยืนยันได้และ outbox กู้คืนครบ; ทดสอบ disk full, store failover และ restore พร้อมระบุช่วงที่ไม่สามารถตอบสำเร็จได้
- inject duplicate/crash ก่อนและหลัง order-outbox commit, publish confirm, consumer transaction และ ack; reconcile completed orders กับทั้งสามงาน ตรวจแต้มเพิ่มครั้งเดียวและ analytics ไม่ซ้ำ
- ทดสอบ key ซ้ำ/ต่าง payload และ crash รอบ Charge/ShipOrder รวมกรณี remote สำเร็จแต่ตอบ timeout; ต้องผ่าน recovery/idempotency gate โดยไม่เรียกผลไม่แน่นอนว่าสำเร็จ
- วัด order-to-award ≤ 60 วินาทีใน failure envelope ที่ตกลง; วัดเวลาระบายอีเมล, DLQ/replay และให้ primary/backup ฝึก restore ตาม runbook ก่อนพิจารณาเปลี่ยนสถานะ ADR

## Open questions

- Payment/shipping รองรับ idempotency และ lookup อย่างไร; จะ reconcile/compensate ผลไม่แน่นอนและกู้คืน cart cleanup อย่างไร? **Not evidenced in the provided files** ต้องปิดก่อน implementation พร้อมใช้งาน
- burst นานเท่าใด อีเมลต้องถึงเมื่อใด quota นับตามบัญชี/ช่วงเวลาใด เพิ่มได้หรือไม่ และ MailFast มี idempotency/status API หรือไม่?
- สูตรแต้ม การคืนเงิน failure envelope สำหรับ 60 วินาที ปลายทาง analytics, retention/replay horizon และข้อกำหนด PII คืออะไร?
- ใครรับบทบาท primary/backup จริง และจะจัดสรร HA/backup/storage capacity กับงบทีมอย่างไร? ตัวเลข RPO/RTO, alert thresholds และ topology ยังต้องตกลงและทดสอบ
- fraud scoring ในอนาคตเป็นงานหลังสั่งซื้อจริงหรือเป็น gate ก่อนยืนยัน? ถ้าเป็น gate ต้องตัดสินใจใหม่ ไม่ต่อ consumer แล้วอ้างว่าป้องกันออเดอร์ได้
