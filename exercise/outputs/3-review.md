Agent: Codex | Model: GPT-6.1 Sol (ตามที่ผู้ใช้ระบุ)

## Quality-attribute scenarios

เกณฑ์ต่อไปนี้ใช้ประเมินข้อเสนอ ยังไม่ใช่ผลทดสอบระบบ

| สถานการณ์ | Source · Stimulus · Environment · Response · Response measure |
|---|---|
| latency และ quota | ผู้ซื้อ · ส่ง 200 orders/s · ช่วง 11.11 โดย MailFast p95 1.5 s, p99 8 s และล้มเหลว 0.5% · checkout ยืนยันผ่าน durable commit แล้วส่งอีเมลภายหลัง · PlaceOrder p95 ≤ 500 ms และส่ง ≤ 50 emails/s; ADR เสนอจำกัด attempts รวม retry จึงต้องยืนยันวิธีนับ quota (docs/scenario.md:24–26; ADR:38,42,58) |
| แต้มครั้งเดียว | relay/consumer · ส่ง OrderPlaced ซ้ำหลัง crash ก่อน ack · ทั้งการทำงานปกติและช่วงกู้คืน · loyalty บันทึก ledger และเพิ่มแต้มแบบ atomic แล้ว ack · หนึ่ง award ต่อ orderId และเสร็จ ≤ 60 s นับจากออเดอร์; ระยะ outage ที่ยังทำได้ต้องตกลง (docs/scenario.md:27; ADR:41,51,62) |
| แยกความล้มเหลว | บริการปลายทาง · email, loyalty หรือ analytics หยุดทำงาน · store ยังพร้อมและมีพื้นที่ · ยืนยันออเดอร์และเก็บ event intent โดยไม่รอ consumer · ออเดอร์ล้มเหลวเพราะปลายทางเหล่านี้ 0 รายการ และกู้คืนงานครบ; ต้องกำหนดระยะ outage เพื่อทดสอบ capacity (docs/scenario.md:28; ADR:38,47,59) |

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Risk | payment/shipping สำเร็จแต่ crash ก่อนบันทึกยังไม่มีวิธี reconcile ที่ปิดประเด็นแล้ว (ADR:39,66) | retry อาจสร้างผลซ้ำ หรือค้างจนไม่ยืนยันออเดอร์; outbox ไม่ทำให้ remote RPC อยู่ใน transaction เดียวกัน ข้อนี้เป็น gate ที่ ADR รับรู้ ไม่ใช่ข้อผิดพลาดที่พิสูจน์จาก implementation |
| Non-risk | เลือก commit completed order กับ outbox พร้อมกัน และทำ loyalty ledger/award กับ unique orderId ใน transaction ก่อน ack (ADR:38,41) | โครงสร้างข้อเสนอรองรับ event intent ไม่หลุดจาก local completion และกันแต้มซ้ำเมื่อ replay; ความถูกต้องของ implementation ยังไม่ได้ยืนยัน |
| Sensitivity point | ระยะพีกและ retry ทำให้ email backlog ไวต่อ quota 50/s: ที่ 200 orders/s โตอย่างน้อย 150/s ก่อน retry (ADR:42,50; docs/scenario.md:25–26) | พีกนาน T วินาทีเพิ่มอย่างน้อย 150T งาน; ยังเลือก storage/retention หรือรับประกันเวลาส่งไม่ได้ และเพิ่ม worker อย่างเดียวไม่แก้ quota |
| Trade-off point | ย้ายงานไป RabbitMQ/outbox ลดการรอปลายทาง แต่เพิ่ม PostgreSQL, broker, relay และภาระกู้คืน (ADR:36,43,47–49) | performance และการแยก failure ดีขึ้น แลกกับ cost/operability ของทีม 4 คน และ store เป็น dependency ของการยืนยันออเดอร์ |

## Missing or weak

- มีสามทางเลือกที่ต่างกันจริง แต่ยังขาดการเทียบต้นทุน/การกู้คืนของ job table กับ RabbitMQ ด้วยโหลดเดียวกัน และยังไม่ประเมินแนวผสม job table แยก worker ต่อ action ซึ่งอาจลดข้อเสียของ worker กลาง (ADR:20–32)
- topology, RPO/RTO, capacity, alert threshold และ replay horizon ยังเปิดอยู่ จึงยังพิสูจน์ทั้ง 60 s และการทน outage ไม่ได้; fault envelope ต้องไม่ใช้ลดข้อกำหนดธุรกิจโดยไม่ตกลง (ADR:43,51,62,68–69)
- ข้ออ้างเกี่ยวกับโค้ด/proto และ ADR-0001 (ADR:11,36) ตรวจยืนยันไม่ได้จากขอบเขตนี้: **Not evidenced in the provided files**. scenario ยืนยันได้เพียงลำดับ synchronous payment/shipping/email (docs/scenario.md:18); รายการ Validation เป็นแผนที่ยังไม่ได้รัน (ADR:56) จึงยังสรุปว่าผ่าน p95 หรือ exactly-once effect จริงไม่ได้

## Verdict

**accept with changes** สำหรับทิศทางการออกแบบ; ยังไม่พร้อม rollout จนปิด gate และมีหลักฐานทดสอบ โดยแก้สามเรื่องก่อน:

1. ระบุ protocol และเจ้าของ recovery ของ payment/shipping/cart cleanup พร้อมผลทดสอบ crash/timeout และ key ซ้ำ (ADR:39,61,66)
2. ตกลง burst duration, email delivery target, quota window และ outage/recovery envelope ของแต้ม แล้วคำนวณ capacity/retry budget และทดสอบเกณฑ์เชิงตัวเลข (ADR:50,58,62,67–68)
3. ระบุ primary/backup จริง, topology, RPO/RTO, retention และ runbook พร้อมทดสอบ restore/replay ว่างานครบและแต้มไม่ซ้ำ (ADR:43,51,59–60,69)
