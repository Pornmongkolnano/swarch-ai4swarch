Agent: Codex; Model: GPT-6.1 Sol (ผู้ใช้ระบุ)

# ADR-0002: แยกงานหลังสั่งซื้อผ่าน RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ยังไม่ระบุผู้อนุมัติ

## Context and problem

R-12 ต้องส่งอีเมลผ่าน MailFast แทน mock emailservice เพิ่ม loyaltyservice และบันทึกเหตุการณ์สำหรับการตลาด โดยปีหน้าจะเพิ่ม SMS, fraud scoring และการปรับคำแนะนำ (docs/scenario.md:7–14)

ปัจจุบัน PlaceOrder เรียกชำระเงิน ส่งสินค้า ล้างตะกร้า แล้วรอส่งอีเมลก่อนตอบกลับ แม้อีเมลล้มเหลวก็เพียงบันทึก warning และยังคืนผลสำเร็จ (system/online-boutique/src/checkoutservice/main.go:230–279); การเรียกอีเมลใช้ gRPC ผ่าน SendOrderConfirmation (:379–383)

เป้าหมายคือ p95 checkout ≤ 500 ms จากปัจจุบันประมาณ 300 ms ที่ยอดสูงสุด 200 orders/s แต่ MailFast มี p95 1.5 s, p99 8 s, ล้มเหลวประมาณ 0.5% และจำกัด 50 emails/s จึงไม่ควรรอ MailFast ในเส้นทางตอบ checkout (docs/scenario.md:24–28) แต้มต้องลงภายใน 1 นาทีและไม่ซ้ำ ระบบอีเมล แต้ม หรือ analytics ล่มต้องไม่ทำให้ออเดอร์ล้มเหลว ทีมมี 4 คน ถนัด Go/gRPC และ 2 คนเคยใช้ RabbitMQ; ระบบอยู่บน Kubernetes และต้องการ open-source แบบ self-hosted (docs/scenario.md:27–30)

## Decision drivers

1. **Availability:** แยกความล้มเหลวของงานหลังสั่งซื้อออกจากผลออเดอร์ (docs/scenario.md:28)
2. **Performance:** รักษา p95 ≤ 500 ms ที่ 200 orders/s และเคารพเพดาน MailFast (docs/scenario.md:24–26)
3. **Correctness:** ลงแต้มครั้งเดียวภายใน 1 นาที (docs/scenario.md:27)
4. **Modifiability:** เพิ่มผู้รับเหตุการณ์ใหม่ได้โดยไม่เพิ่มการเรียกตรงใน checkout (docs/scenario.md:14)
5. **Operability/cost:** สอดคล้องกับทีมเล็ก ความคุ้นเคย RabbitMQ และ Kubernetes แบบ self-hosted (docs/scenario.md:29–30)

## Considered options

ทุกกลไกต่อไปนี้เป็น **ข้อเสนอใหม่** ไม่ใช่ความสามารถที่ยืนยันว่ามีอยู่แล้ว

1. เพิ่ม synchronous gRPC จาก checkout ไปยังตัวเชื่อม MailFast, loyaltyservice และ analytics โดย timeout และบันทึกความล้มเหลว
2. ใช้ durable outbox และ RabbitMQ กระจาย OrderPlaced ให้ consumer แยกตามงาน
3. ใช้ durable job table และ worker กลาง polling แล้วเรียกแต่ละบริการ

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| เรียก synchronous | ใช้ทักษะ Go/gRPC เดิม; ส่วนประกอบใหม่น้อย | การรอ MailFast เสี่ยงเกินงบ latency; timeout ไม่รับประกันส่งงานสำเร็จ; checkout ต้องรู้จักทุกงานใหม่ |
| outbox + RabbitMQ | แยก latency และความล้มเหลว; แต่ละงานมี backlog และ retry ของตน; เพิ่ม consumer ใหม่ได้ | เพิ่ม durable store และ broker; ต้องจัดการ duplicate, schema และช่องว่างระหว่างผลออเดอร์กับ event |
| job table + worker กลาง | เก็บสถานะและ retry ได้ในที่เดียว; ไม่ต้องดูแล broker | เพิ่มภาระ polling และฐานข้อมูล; worker กลางต้องแก้ตามงานใหม่และป้องกันงานช้าขวางแต้ม |

## Decision

เลือกข้อเสนอ **durable outbox + RabbitMQ แบบ self-hosted บน Kubernetes** เพราะแยกงานช้าจาก checkout และรองรับงานการตลาดเพิ่มเติม โดยสอดคล้องกับประสบการณ์ทีม (docs/scenario.md:14,24–30) รายละเอียดต่อไปนี้เป็นกลไกที่เสนอให้สร้าง:

- กำหนดจุดยืนยันออเดอร์ให้มี durable order record และ outbox row ใน transaction เดียวกัน ก่อนตอบสำเร็จ; event มี eventId, orderId, userId, occurredAt และ schemaVersion พร้อมข้อมูลขั้นต่ำสำหรับแต่ละงาน ห้ามใส่ข้อมูลบัตรเครดิต
- relay ส่ง OrderPlaced ไปยัง durable exchange และ durable queue แยกสำหรับอีเมล แต้ม และ analytics; ใช้ persistent message, publisher confirm และ retry การ publish จนยืนยันได้ แล้วจึงทำเครื่องหมาย outbox ว่าส่งแล้ว การ publish ซ้ำเป็นไปได้
- consumer อีเมลแทนการเรียก mock logic ด้วย MailFast adapter; ใช้ rate limiter รวมทุก replica ไม่เกิน 50 emails/s รวม retry และปรับ concurrency ตาม latency มี timeout, backoff และ jitter โดยไม่ขวาง queue แต้ม
- loyaltyservice บันทึก orderId ที่ประมวลผลแล้วด้วย unique constraint และเพิ่มแต้มใน transaction เดียวกัน จากนั้นจึง ack; ส่งซ้ำหรือ crash ก่อน ack ต้องไม่เพิ่มแต้มอีก เป็นการรับประกันผลข้างเคียงครั้งเดียว ไม่ใช่สมมติว่า broker ส่งครั้งเดียว
- consumer analytics บันทึกเหตุการณ์โดย deduplicate ด้วย eventId แล้ว ack; consumer ทุกตัว retry แบบมีขอบเขต ส่งรายการที่ยังล้มเหลวไป dead-letter queue พร้อมแจ้งเตือนและ replay ที่รักษา idempotency

**ขอบเขตความมั่นใจ:** durable order store/outbox, RabbitMQ และ transaction ดังกล่าวยังไม่ปรากฏใน main.go ที่อ่าน; สถานะการมีอยู่ในระบบส่วนอื่นคือ **Not evidenced in the provided files** ปัจจุบัน UUID ถูกสร้างใหม่ทุก PlaceOrder และ payment/shipping เกิดก่อนตอบกลับ (main.go:233,252–279) ดังนั้น outbox เพียงอย่างเดียวไม่แก้ crash หลัง external side effect แต่ก่อน commit ต้องออกแบบ durable checkout state, request idempotency และการ reconcile ผล payment/shipping ก่อนถือว่าตัดสินใจนี้พร้อมใช้งานจริง

เมื่อ broker หรือ consumer ล่ม checkout ยังบันทึกงานไว้ใน outbox โดยไม่รอระบบเหล่านั้น แต่หาก durable order/outbox store ล่ม ข้อเสนอนี้ยังไม่รับประกันทั้งการตอบสำเร็จและการไม่สูญเสียงาน ต้องตกลงขอบเขต failure และ recovery นี้ก่อนนำใช้

## Consequences

- **Positive:** checkout ไม่รอ MailFast; งานที่ล่ม retry แยกกัน และเพิ่มงานใหม่ผ่าน consumer ได้ (quality attribute: performance, availability, modifiability)
- **Positive:** unique orderId และ transaction ใน loyaltyservice ป้องกันแต้มซ้ำแม้รับข้อความซ้ำ (quality attribute: correctness)
- **Negative:** ต้องดูแล broker, durable store, relay, queue และ schema เพิ่มสำหรับทีม 4 คน (quality attribute: operability, cost)
- **Negative:** ผลอีเมล/แต้ม/analytics เป็น eventual consistency; ที่ 200 orders/s อีเมลใหม่เกินเพดาน 50/s อย่างน้อย 150/s ก่อนนับ retry จึงสะสม backlog และไม่มีวิธีเพิ่ม worker ให้พ้นเพดานนี้ (คำนวณจาก docs/scenario.md:25–26; quality attribute: latency, scalability)
- **Risks and mitigations:** ตรวจ queue age ของแต้มและเร่งกู้ก่อนครบ 1 นาที; outage นานเกินกำหนดยังทำให้ผิด SLA ได้ ส่วนอีเมลต้องตกลงเวลาส่ง ยอด burst และโควตาเพิ่มเติม; ตรวจ outbox backlog, dead-letter และพื้นที่เก็บ พร้อมทดสอบกู้คืน broker/store (quality attribute: reliability, recoverability)
- **Risks and mitigations:** การไม่รู้ผล MailFast หลัง timeout อาจทำให้อีเมลซ้ำ ต้องตรวจ provider idempotency/reconciliation; จำกัดข้อมูลและสิทธิ์เข้าถึงแต่ละ queue (quality attribute: correctness, security)

## Validation

- ทดสอบ 200 orders/s โดยจำลอง MailFast ตาม latency, error และ rate limit ใน scenario; ยืนยัน PlaceOrder p95 ≤ 500 ms, อัตรารวม MailFast ≤ 50/s และไม่มีออเดอร์ล้มเพราะ consumer/broker หยุดทำงานเมื่อ outbox store ยังพร้อมใช้งาน
- วัดเวลาตั้งแต่ยืนยันออเดอร์ถึง commit แต้ม ต้องไม่เกิน 60 วินาทีในขอบเขต failure ที่ตกลง; inject duplicate และ crash ก่อน/หลัง commit/ack แล้วตรวจว่าแต้มเพิ่มครั้งเดียวและ analytics ไม่ซ้ำ
- จำลอง crash ก่อน/หลัง order-outbox commit และ publish confirm; เทียบออเดอร์ที่ยืนยันกับงานทั้งสามให้ครบ รวม replay และ reconcile payment/shipping; การทดสอบต้องไม่ถือว่าการคืน gRPC สำเร็จพิสูจน์ durability
- ทดสอบ burst ตามระยะเวลาที่ตกลงและวัดเวลาระบายอีเมลจริง; ทดสอบเพิ่ม consumer ใหม่โดยไม่แก้ checkout แล้วให้ทีมทบทวนผลก่อนเปลี่ยนสถานะ ADR

## Open questions

- ใครเป็นผู้อนุมัติ และนิยาม “order placed” อยู่ที่จุดใดเมื่อ payment สำเร็จแต่ shipping หรือ durable commit ล้มเหลว?
- จะใช้ durable order/outbox store ใด และจะทำ request idempotency/reconciliation กับ payment/shipping อย่างไร? ความสามารถรองรับจากบริการเหล่านี้ **Not evidenced in the provided files**
- burst 200 orders/s นานเท่าใด อีเมลต้องถึงเมื่อใด โควตา MailFast เพิ่มได้หรือไม่ และ API รองรับ idempotency หรือไม่?
- ขอบเขต outage สำหรับ SLA แต้ม 1 นาทีคืออะไร สูตรแต้ม/การยกเลิกออเดอร์เป็นอย่างไร และยอมรับกรณี durable store ล่มได้อย่างไร?
- analytics จะเก็บที่ใด ต้องการ retention เท่าใด และใครรับผิดชอบ dead-letter/replay, broker HA, backup และข้อมูลส่วนบุคคล? ตัวเลขและความสามารถเหล่านี้ **Not evidenced in the provided files**
