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

> ขอบเขตการยืนยัน: ตรวจค่า env ของผู้เรียกเทียบกับชื่อและพอร์ต Service ปลายทางจากไฟล์จริง เป็นการยืนยันระดับ configuration เท่านั้น ไม่ใช่การสังเกต runtime หรือหลักฐานว่า RPC ถูกเรียกสำเร็จ

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| frontend -> productcatalogservice | `system/online-boutique/kubernetes-manifests/frontend.yaml` (PRODUCT_CATALOG_SERVICE_ADDR); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (Service) | ✅ | ผู้เรียกกำหนด `productcatalogservice:3550` ที่ `frontend.yaml:69–70`; Service ชื่อ `productcatalogservice` ที่ `productcatalogservice.yaml:71` เปิด `grpc` port 3550 -> targetPort 3550 ที่บรรทัด 78–81 ตรงกัน |
| frontend -> cartservice | `system/online-boutique/kubernetes-manifests/frontend.yaml` (CART_SERVICE_ADDR); `system/online-boutique/kubernetes-manifests/cartservice.yaml` (Service) | ✅ | `frontend.yaml:73–74` กำหนด `cartservice:7070`; Service ชื่อ `cartservice` ที่ `cartservice.yaml:72` เปิด `grpc` port 7070 -> targetPort 7070 ที่บรรทัด 79–82 |
| frontend -> checkoutservice | `system/online-boutique/kubernetes-manifests/frontend.yaml` (CHECKOUT_SERVICE_ADDR); `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (Service) | ✅ | `frontend.yaml:79–80` กำหนด `checkoutservice:5050`; Service ชื่อ `checkoutservice` ที่ `checkoutservice.yaml:80` เปิด `grpc` port 5050 -> targetPort 5050 ที่บรรทัด 87–90 |
| checkoutservice -> paymentservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (PAYMENT_SERVICE_ADDR); `system/online-boutique/kubernetes-manifests/paymentservice.yaml` (Service) | ✅ | `checkoutservice.yaml:61–62` กำหนด `paymentservice:50051`; Service ชื่อ `paymentservice` ที่ `paymentservice.yaml:71` เปิด `grpc` port 50051 -> targetPort 50051 ที่บรรทัด 78–81; `system/online-boutique/protos/demo.proto:177–179` ประกาศ `PaymentService.Charge` แต่ยังไม่พิสูจน์ call site |
| checkoutservice -> emailservice | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (EMAIL_SERVICE_ADDR); `system/online-boutique/kubernetes-manifests/emailservice.yaml` (Service) | ✅ | `checkoutservice.yaml:63–64` กำหนด `emailservice:5000`; Service ชื่อ `emailservice` ที่ `emailservice.yaml:73` เปิด `grpc` port 5000 -> targetPort 8080 ที่บรรทัด 80–83 จึงไม่ใช่พอร์ตขัดแย้ง; `system/online-boutique/protos/demo.proto:199–201` ประกาศ `EmailService.SendOrderConfirmation` |
| cartservice -> redis-cart | `system/online-boutique/kubernetes-manifests/cartservice.yaml` (REDIS_ADDR, Redis Deployment และ Service) | ✅ | `cartservice.yaml:50–51` กำหนด `redis-cart:6379`; Service ชื่อ `redis-cart` ที่บรรทัด 146 เปิด `tcp-redis` port 6379 -> targetPort 6379 ที่บรรทัด 153–156 และ image `redis:alpine` อยู่บรรทัด 118; ไม่ยืนยันคำสั่ง Redis หรือผลการเก็บข้อมูล |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): ไม่พบชื่อหรือพอร์ตผิดใน 6 dependency ที่ตรวจ แต่ยังขาดหลักฐาน call site, ลำดับการทำงาน และผล runtime ตามที่ Run B ระบุไว้แล้วใน Gaps ไม่ใช่ข้อผิดพลาดที่ AI ปกปิด
- จุดที่ควรปรับในแผนภาพคือกล่อง `loadgenerator` ยังดูเหมือน deploy ร่วมกับกล่องอื่นโดยไม่มีเงื่อนไขกำกับในตัวภาพ ทั้งที่ `system/online-boutique/kubernetes-manifests/kustomization.yaml:24` comment resource นี้ออก; Run B อธิบายข้อจำกัดไว้ในข้อความแล้ว จึงควรใส่เงื่อนไขนี้ในภาพด้วยเพื่อป้องกันการอ่านภาพแยกจากคำอธิบาย
- `SHOPPING_ASSISTANT_SERVICE_ADDR=shoppingassistantservice:80` มีจริงที่ `system/online-boutique/kubernetes-manifests/frontend.yaml:83–84` แต่ `ENABLE_ASSISTANT` ถูก comment ที่บรรทัด 93–94 และ component ถูก comment ที่ `system/online-boutique/kubernetes-manifests/kustomization.yaml:35`; ยังสรุปไม่ได้ว่าเป็น dependency ที่ทำงานอยู่ จึงไม่ถือว่าการเว้นออกจากภาพเป็น dependency ที่ขาดโดยแน่นอน
- Redis ใช้ `emptyDir` ที่ `system/online-boutique/kubernetes-manifests/cartservice.yaml:129–141`; การมีสัญลักษณ์ฐานข้อมูลไม่ยืนยัน durable persistence ซึ่ง Run B caveat ไว้แล้ว และยังต้องตรวจ implementation ก่อนสรุป semantics ของ cart

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ


**คำตอบข้อ 1:** Run A ระบุได้ว่า interface มี RPC และ message อะไร เช่น `PaymentService.Charge` ใน `system/online-boutique/protos/demo.proto:177–195` แต่ไฟล์นี้ไม่มีตัวผู้เรียก, endpoint, frontend, Redis หรือ deployment จึงยังระบุเส้น dependency และ container ที่ deploy จริงไม่ได้ และ `go_package` ที่บรรทัด 19 ก็ไม่ยืนยันภาษาของทุก service. Run B เพิ่มภาพระดับ configuration ได้จาก env ของผู้เรียกและ Service ปลายทาง เช่น `checkoutservice.yaml:63–64` กับ `emailservice.yaml:80–83` ภายใต้ `system/online-boutique/kubernetes-manifests/` แต่ยังไม่ทราบว่ามีการเรียกจริง ลำดับใด หรือสำเร็จหรือไม่

**คำตอบข้อ 2:** จุดที่เสี่ยงทำให้สมาชิกใหม่เข้าใจผิดที่สุดคืออ่านกล่อง `loadgenerator` ในภาพเป็นส่วนที่ deploy อยู่แน่นอน ทั้งที่ `system/online-boutique/kubernetes-manifests/kustomization.yaml:24` comment resource ออก และ Run B เองก็แจ้งข้อจำกัดนี้ไว้ในข้อความแล้ว. ควรกำกับเงื่อนไขในกล่องหรือคำอธิบายที่ติดกับภาพ เพื่อไม่ให้วางแผนตรวจระบบหรือคาดว่ามี traffic ทดสอบจากภาพเพียงอย่างเดียว ทั้งนี้ยังไม่มีหลักฐานว่า loadgenerator ไม่ได้ถูกรันผ่านช่องทางอื่น
