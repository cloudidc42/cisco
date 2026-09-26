# Part 42 — Multicast Advanced (RP Redundancy, Auto-RP/BSR, SSM)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 411–420 จาก 1000**

> ต่อจาก [Part 41 — Multicast Fundamentals (IGMP, PIM-DM/SM)](part-041-multicast-fundamentals.md)
> ที่เราสร้าง PIM-SM Domain สำเร็จ ส่ง All-Hands Video Stream จาก **STREAM-SRV** (`10.10.30.30`)
> ไปยัง Group `239.1.1.1` ให้พนักงานทั้ง VLAN 10 (SALES) และ VLAN 40 (WIFI) รับชมพร้อมกันได้จริง
> — แต่ปิด Part นั้นไปด้วยจุดอ่อนใหญ่ที่ทีมงานยังไม่ได้แก้: เราตั้งให้ **CORE-SW1 เป็น Static RP
> ตัวเดียวของทั้ง Domain** (`ip pim rp-address 1.1.1.1` Hardcode บนทุก Router) ซึ่งหมายความว่า
> ถ้า CORE-SW1 ล่มขึ้นมาระหว่างงาน All-Hands Meeting จริง **Multicast ทั้ง Domain จะพังทันที**
> โดยไม่มีกลไก Failover อัตโนมัติใดๆ เลย Part นี้จะแก้จุดอ่อนนี้ให้เสร็จสมบูรณ์ด้วย **Auto-RP, BSR,
> Anycast RP + MSDP** พร้อมเสริม **SSM (Source-Specific Multicast)** และ **Bidir-PIM** สำหรับ
> Use Case ขั้นสูงที่ CCNP ENCOR ต้องรู้

## Use Case ที่จะขับเคลื่อน Part นี้: "ถ้า CORE-SW1 ล่มระหว่างงาน All-Hands จะเกิดอะไรขึ้น?"

ทีม Network หลังจาก Deploy Lab ใน Part 41 สำเร็จ ได้นำ Config ไปทดสอบใน Maintenance Window
โดยจำลองสถานการณ์ **Power off CORE-SW1** กลางงาน Video Streaming — ผลลัพธ์ที่เจอคือ:

```
ก่อน CORE-SW1 ล่ม                              หลัง CORE-SW1 ล่ม (ไม่มี Redundant RP)

  STREAM-SRV ──► 239.1.1.1 ──► RP (CORE-SW1) ──► Receiver ทุกจุด    STREAM-SRV ──► 239.1.1.1 ──► ✗ ✗ ✗
  ทำงานปกติทุกอย่าง                                                  RP หายไปทั้ง Domain
                                                                    - PC ใหม่ที่พึ่งเปิดดู Video
                                                                      ส่ง IGMP Join แล้วไม่มี RP
                                                                      ให้ Register/Join เข้าไปถึง
                                                                    - (*,G) State ใหม่สร้างไม่ได้
                                                                    - Source ใหม่ที่พึ่งเริ่มส่ง
                                                                      Register ไปที่ RP ไม่ได้เลย
```

ทีมงานพบว่าปัญหานี้ **ร้ายแรงกว่าที่คิด** — แม้ Unicast Routing (OSPF จาก Part 11-12) จะ Converge
ใหม่รอบ CORE-SW1 ได้อย่างรวดเร็วผ่าน CORE-SW2 ก็ตาม แต่ **PIM-SM ทั้ง Domain ยังคงหา RP ไม่พบ**
เพราะ `ip pim rp-address 1.1.1.1` เป็นค่า Static ที่ทุก Router เชื่อว่า RP อยู่ที่ IP นี้เสมอ — ไม่มี
กลไกใดสอนให้ Router รู้ว่า "ถ้า RP เดิมหาย ให้ใช้ RP ตัวอื่นแทน" นี่คือช่องว่างที่ Part นี้จะปิดให้สมบูรณ์

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 411 | จุดอ่อนของ Static RP — Single Point of Failure และผลกระทบจริงเมื่อ CORE-SW1 ล่ม |
| 412 | Auto-RP (Cisco Proprietary) — Candidate-RP, Mapping Agent, Config เต็มรูปแบบ |
| 413 | BSR (Bootstrap Router, RFC 5059) — Candidate-BSR, Candidate-RP-Set, Config เต็มรูปแบบ |
| 414 | Auto-RP vs BSR — เปรียบเทียบและเลือกใช้ให้ถูกงาน พร้อมแนะนำ Anycast RP |
| 415 | Anycast RP (RFC 4610) — Redundancy + Load-Splitting ด้วย RP Address เดียวกันหลายตัว |
| 416 | MSDP (Multicast Source Discovery Protocol) — Source-Active, RPF Check, Inter-domain Multicast |
| 417 | SSM (Source-Specific Multicast) — `232.0.0.0/8`, IGMPv3 INCLUDE, ไม่ต้องมี RP เลย |
| 418 | Bidirectional PIM (Bidir-PIM) — Many-to-Many Multicast สำหรับ Financial Trading |
| 419 | Verification & Troubleshooting ขั้นสูง — RP Mapping, MSDP, Auto-RP Diagnostic |
| 420 | Lab เต็มรูปแบบ: Anycast RP + MSDP แทน Static RP พร้อมเปิด SSM สำหรับ Market-Data Feed |

---

## Step 411 — จุดอ่อนของ Static RP: Single Point of Failure

### ทวนกลไก Static RP จาก Part 41

```
ip pim rp-address 1.1.1.1     ← Config บรรทัดเดียว บอกทุก Router ว่า "RP อยู่ที่ 1.1.1.1 เสมอ"

ปัญหา: บรรทัดนี้เป็น "ความจริงที่ตายตัว" ไม่มีการตรวจสอบ Reachability แบบ Dynamic
       ต่างจาก Routing Protocol ที่ถ้า Neighbor ล่ม จะมีกลไก Reconverge อัตโนมัติ
       PIM ไม่มีแนวคิด "RP สำรอง" ในรูปแบบ Static เลย — มีแค่ RP เดียวที่ Hardcode ไว้
```

### วิเคราะห์ผลกระทบแยกตามประเภท State ที่มีอยู่ก่อน CORE-SW1 ล่ม

| State ที่มีอยู่แล้วก่อนล่ม | ผลกระทบเมื่อ CORE-SW1 (RP) ล่ม |
|---|---|
| `(S, G)` ที่ Switchover ไป SPT แล้ว (เช่น DIST-SW4 ใน Part 41 ที่ไม่ผ่าน RP อีกต่อไป) | **รอดชั่วคราว** — เพราะ RPF Interface ชี้ไปทาง Source ตรงๆ ไม่พึ่ง RP แล้ว Traffic ยังไหลต่อได้ตราบใดที่ Source ยังส่งอยู่ |
| `(*, G)` ที่ยังเป็น Shared Tree (ยังไม่ Switchover, หรือ Router ที่ตั้ง `spt-threshold infinity`) | **พังทันที** — RPF Interface ของ Entry นี้ชี้ไปทาง RP โดยตรง เมื่อ RP หาย Router หา Path ไป RP ไม่ได้ Data หยุดไหล |
| Receiver ใหม่ที่พึ่งเปิด Media Player ส่ง IGMP Join เข้ามาระหว่างที่ RP ล่ม | **Join ไม่สำเร็จเลย** — Last-Hop Router ไม่รู้จะส่ง PIM Join ไปหา RP ตัวไหน เพราะ Static Config ชี้ไปที่ IP เดียวที่ตายไปแล้ว |
| Source ใหม่ที่พึ่งเริ่มส่ง Traffic ระหว่างที่ RP ล่ม (First-Hop Router ต้อง Register) | **Register ไม่สำเร็จเลย** — First-Hop Router ส่ง PIM Register Message ไปยัง RP ที่ไม่มีชีวิตอยู่ Timeout ซ้ำไปเรื่อยๆ |

### ทำไม "เปลี่ยน Static RP เป็น IP อื่นตอนล่ม" ไม่ใช่คำตอบที่ยอมรับได้

ในทางทฤษฎี Network Engineer อาจแก้ปัญหาด้วยการ **Login เข้าทุก Router แล้วเปลี่ยน
`ip pim rp-address` เป็น CORE-SW2 (`1.1.1.2`) ทันทีที่ CORE-SW1 ล่ม** — แต่วิธีนี้มีปัญหาเชิง
Operation ร้ายแรง:

1. **ต้องมีมนุษย์ตรวจจับปัญหาและลงมือแก้ Manual** — ไม่ใช่ Automatic Failover ที่ Production
   Network ระดับ Enterprise ต้องมี (ตรงข้ามกับ HSRP/VRRP ที่เราเรียนไปแล้วใน
   [Part 18](part-018-fhrp.md) ซึ่ง Failover ในไม่กี่วินาทีโดยไม่ต้องมีคนแทรกแซง)
2. **ต้องเปลี่ยนทุก Router ในเวลาเดียวกัน** — ถ้าเปลี่ยนไม่ครบหรือไม่พร้อมกัน จะเกิด **RP Mapping
   Inconsistency** ที่ Router บางตัวยังชี้ไป CORE-SW1 (ตายแล้ว) แต่บางตัวชี้ไป CORE-SW2 ทำให้
   Domain แตกเป็นสองฝั่งที่ไม่คุยกันรู้เรื่อง (ปัญหาเดียวกับที่เตือนไว้ท้าย Step 407 ของ Part 41)
3. **Downtime ระหว่างรอคนแก้ไข** อาจกินเวลานานกว่าที่ Business ยอมรับได้ไปมาก โดยเฉพาะงาน
   Live Streaming ที่ต้องการความต่อเนื่องแบบ Real-time

### สรุปสิ่งที่ Part นี้จะแก้ไข

Cisco และมาตรฐานอุตสาหกรรมมีเครื่องมือ 3 กลุ่มที่แก้ปัญหานี้ได้ ซึ่งเราจะเรียนตามลำดับ:

```
412: Auto-RP  ──┐
413: BSR       ──┼──► ทำให้ Router "ค้นพบ RP โดยอัตโนมัติ" ผ่าน Protocol แทน Static Config
                │     (ยังมี RP ตัวเดียวใน Domain ในเวลาใดเวลาหนึ่ง แต่เปลี่ยนตัวได้เองถ้าตัวเดิมหาย)
                │
415: Anycast RP ─────► ทำให้ "มี RP มากกว่า 1 ตัวทำงานพร้อมกันจริง" ด้วย IP เดียวกัน
                       (ตัดปัญหา Single Point of Failure ได้เด็ดขาดที่สุด — เป้าหมายหลักของ Lab Step 420)
```

---

## Step 412 — Auto-RP (Cisco Proprietary)

**Auto-RP** คือกลไก **เฉพาะของ Cisco** (ไม่ใช่มาตรฐาน RFC) ที่เก่าแก่ที่สุดในการทำให้ RP ถูก
ค้นพบแบบ Dynamic — ใช้ Multicast Message พิเศษ 2 กลุ่มสื่อสารกันระหว่าง Router ที่เกี่ยวข้อง

### บทบาท 2 อย่างที่ต้องเข้าใจ

| บทบาท | ทำหน้าที่อะไร | ส่ง/รับ Message ทาง Group ไหน |
|---|---|---|
| **Candidate-RP (C-RP)** | Router ที่ "เสนอตัว" เป็น RP ได้ — ประกาศตัวเองเป็นระยะๆ | ส่ง **RP-Announce** ไปที่ `224.0.1.39` (Cisco-RP-Announce) |
| **Mapping Agent (MA)** | Router ที่ฟัง RP-Announce จากทุก C-RP แล้ว **เลือกตัวที่ดีที่สุด** ต่อ Group-range จากนั้นประกาศผลให้ทุก Router ในโดเมนรู้ | ฟังที่ `224.0.1.39`, ส่ง **RP-Discovery** ไปที่ `224.0.1.40` (Cisco-RP-Discovery) |

```
                     ┌──────────────────┐
                     │   CORE-SW1        │──── RP-Announce (224.0.1.39) ────┐
                     │  C-RP Priority=0  │    "ฉันเสนอตัวเป็น RP นะ           │
                     │  (ค่ายิ่งต่ำ       │     Priority = 0 (ดีที่สุด)"       │
                     │   ยิ่ง Preferred)  │                                   ▼
                     └──────────────────┘                          ┌──────────────────┐
                                                                     │  Mapping Agent    │
                     ┌──────────────────┐                          │  (สมมติให้ตัวหนึ่ง │
                     │   CORE-SW2        │──── RP-Announce ─────────►  ในทั้งสอง Core ทำ  │
                     │  C-RP Priority=10 │    "ฉันก็เสนอตัวเป็น RP        หน้าที่นี้ด้วย)   │
                     │  (Backup)         │     Priority = 10 (ต่ำกว่า)"│                  │
                     └──────────────────┘                          └────────┬─────────┘
                                                                              │
                                                            RP-Discovery (224.0.1.40)
                                                            "ผลสรุป: RP ของ Group X = CORE-SW1
                                                             (เพราะ Priority ต่ำกว่า = ดีกว่า)"
                                                                              │
                                                                              ▼
                                                              ทุก Router ใน Domain ฟัง Group นี้
                                                              แล้วรู้จัก RP ที่ถูกต้องโดยอัตโนมัติ
```

> **จุดที่มักสอบผิด**: ใน Auto-RP **ค่า Priority ยิ่งต่ำยิ่ง Preferred** (ตรงข้ามกับความรู้สึกทั่วไป
> และตรงข้ามกับ BSR ใน Step 413 ที่ค่ายิ่งสูงยิ่ง Preferred) — ค่า Default ของ C-RP Priority บน
> Cisco IOS คือ `0`

### เงื่อนไขสำคัญ: Auto-RP ต้องใช้ Dense-Mode Flooding เว้นแต่มี `autorp listener`

Group `224.0.1.39` และ `224.0.1.40` โดยธรรมชาติต้องมี Router ทุกตัวรับ/ส่งได้แบบ Flood
(คล้าย PIM-DM) ซึ่งขัดกับ Design ที่เราต้องการให้ทั้ง Domain เป็น Sparse-Mode ล้วนๆ — Cisco IOS
จึงมีคำสั่ง **`ip pim autorp listener`** ที่ทำให้ Router "รับฟัง/ส่ง" เฉพาะ 2 Group นี้แบบ Dense
ได้โดยไม่ต้องเปิด Dense Mode ทั้ง Interface (นี่คือ Best Practice ที่หลักสูตรนี้จะใช้เสมอ)

### Config เต็มรูปแบบ: CORE-SW1 และ CORE-SW2 เป็น C-RP ทั้งคู่ (CORE-SW1 Priority ดีกว่า)

```
! ===== ACL กำหนดว่า C-RP นี้ดูแล Group ไหนบ้าง (ใช้ร่วมกับ Lab เดิมจาก Part 41) =====
CORE-SW1(config)# access-list 10 permit 239.0.0.0 0.255.255.255
CORE-SW2(config)# access-list 10 permit 239.0.0.0 0.255.255.255

! ===== CORE-SW1: Candidate-RP (Priority ดีที่สุด = 0, ค่า Default) + Mapping Agent =====
CORE-SW1(config)# ip pim autorp listener
CORE-SW1(config)# ip pim send-rp-announce Loopback0 scope 32 group-list 10 interval 60
CORE-SW1(config)# ip pim send-rp-discovery Loopback0 scope 32

! ===== CORE-SW2: Candidate-RP (Priority แย่กว่า = 10, ทำหน้าที่ Backup) =====
CORE-SW2(config)# ip pim autorp listener
CORE-SW2(config)# ip pim send-rp-announce Loopback0 scope 32 group-list 10 priority 10 interval 60

! หมายเหตุ: ไม่จำเป็นต้องมี Mapping Agent สองตัว — ตั้งไว้ตัวเดียวพอ (CORE-SW1 ในตัวอย่างนี้)
! แต่ถ้าต้องการ Redundancy ระดับ Mapping Agent ด้วย สามารถตั้ง send-rp-discovery บน CORE-SW2
! ได้อีกตัว โดย Router ทุกตัวจะเลือกฟัง Mapping Agent ที่มี IP สูงสุดเป็นค่าตัดสิน (Tie-break)

! ===== DIST-SW1/DIST-SW3/DIST-SW4: ไม่ต้อง Config ip pim rp-address อีกต่อไป! =====
! Router ที่ไม่ใช่ C-RP/Mapping Agent เพียงแค่ต้องรับ/ฟัง RP-Discovery ผ่าน autorp listener
DIST-SW1(config)# ip pim autorp listener
DIST-SW3(config)# ip pim autorp listener
DIST-SW4(config)# ip pim autorp listener
```

### ตรวจสอบผลลัพธ์

```
DIST-SW3# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is not a Mapping Agent

Group(s) 239.0.0.0/8
  RP 1.1.1.1 (?), v2v1
    Info source: 1.1.1.1 (?), via Auto-RP
         Uptime: 00:04:12, expires: 00:02:47
```

สังเกตว่า `Info source: via Auto-RP` ยืนยันว่า RP Mapping นี้มาจาก Auto-RP ไม่ใช่ Static Config
อีกต่อไป — และถ้า CORE-SW1 ล่ม, RP-Announce จากมันจะหายไป, Mapping Agent จะเลือก
CORE-SW2 (Priority ถัดไป) เป็น RP ใหม่โดยอัตโนมัติภายในไม่กี่ Announce Interval (Default 60s ต่อ
รอบ Announce, ค่า Holdtime ปกติ 3× Interval ก่อนถือว่า C-RP ตายไปจริง)

---

## Step 413 — BSR (Bootstrap Router, RFC 5059)

**BSR** คือกลไก **มาตรฐานเปิด (RFC 5059)** ที่ทำงานคล้าย Auto-RP ในเป้าหมาย (ทำให้ RP ถูก
ค้นพบแบบ Dynamic) แต่ต่างกันที่วิธีสื่อสาร — BSR **ไม่ใช้ Multicast Group แยกเลย** ทุกอย่างส่งผ่าน
**PIM Message ปกติ (Bootstrap Message)** ที่ Flood แบบ Hop-by-Hop ไปยัง PIM Neighbor ทุกตัว
โดยตรง (Unicast/Link-local ระหว่าง PIM Neighbor ผ่าน `224.0.0.13` เดิมที่ใช้ Hello อยู่แล้ว)

### บทบาท 2 อย่างของ BSR (คล้าย Auto-RP แต่ชื่อต่างและกลไกคนละแบบ)

| บทบาท | ทำหน้าที่อะไร | ต่างจาก Auto-RP อย่างไร |
|---|---|---|
| **Candidate-BSR (C-BSR)** | Router ที่เสนอตัวเป็น **BSR** (ตัวกลางเก็บรวบรวมและกระจาย RP-Set ทั้ง Domain) — มีได้ Router เดียวที่เป็น BSR จริงในแต่ละครั้ง (คัดเลือกจาก C-BSR ทั้งหมดด้วย Priority) | Auto-RP ไม่มีแนวคิด BSR เลย — มีแค่ Mapping Agent ที่ทำหน้าที่คล้ายกันแต่ไม่ต้อง Elect |
| **Candidate-RP (C-RP)** | Router ที่เสนอตัวเป็น RP — ส่งข้อมูลเสนอตัว **ไปที่ BSR โดยตรง (Unicast)** ไม่ใช่ Flood | Auto-RP C-RP Flood ไปทาง Multicast `224.0.1.39` ให้ Mapping Agent ทุกตัวได้ยินพร้อมกัน |

```
Step 1: C-BSR Election                       Step 2: C-RP → BSR (Unicast)
   CORE-SW1 (Priority 100) ┐                     CORE-SW1 ──C-RP msg──►┐
   CORE-SW2 (Priority 50)  ┼─► เทียบ Priority     CORE-SW2 ──C-RP msg──►│  BSR (CORE-SW1)
   (ค่ายิ่งสูงยิ่งชนะ,                              │                    │  รวบรวมเป็น
    เท่ากันเทียบ IP สูงสุด)                          ▼                    │  Candidate-RP-Set
   ผลลัพธ์: CORE-SW1 ชนะ = BSR                 CORE-SW1 ชนะเป็น BSR ─────┘

Step 3: BSR Flood Candidate-RP-Set ผ่าน PIM Bootstrap Message (Hop-by-Hop, ไม่ใช่ Multicast Group แยก)

   BSR (CORE-SW1) ──Bootstrap msg──► PIM Neighbor ──Bootstrap msg──► PIM Neighbor ──► ... ทั้ง Domain
   (แต่ละ Router Forward Bootstrap Message ต่อไปยัง PIM Neighbor ทุกตัว ยกเว้นทางที่รับมา)
```

> **จุดที่ต้องจำสำหรับ CCNP ENCOR**: BSR ใน Priority ทั้ง C-BSR และ C-RP **ค่ายิ่งสูงยิ่ง Preferred**
> (ตรงข้ามกับ Auto-RP ที่ค่ายิ่งต่ำยิ่งดี) — ถ้า Priority เท่ากัน ตัดสินด้วย **IP Address สูงสุด**

### Config เต็มรูปแบบ: BSR แทน Auto-RP (ใช้แทนกัน ไม่ใช้พร้อมกันในทางปฏิบัติ)

```
! ===== CORE-SW1: Candidate-BSR (Priority สูงกว่า = ชนะเป็น BSR) + Candidate-RP =====
CORE-SW1(config)# ip pim bsr-candidate Loopback0 32 100
CORE-SW1(config)# ip pim rp-candidate Loopback0 group-list 10 priority 0 interval 60

! ===== CORE-SW2: Candidate-BSR (Priority ต่ำกว่า = สำรอง) + Candidate-RP (Priority แย่กว่า) =====
CORE-SW2(config)# ip pim bsr-candidate Loopback0 32 50
CORE-SW2(config)# ip pim rp-candidate Loopback0 group-list 10 priority 10 interval 60

! ===== Router อื่นๆ ในโดเมน (DIST-SW1/3/4) ไม่ต้อง Config อะไรเพิ่มเลย! =====
! เพราะ Bootstrap Message กระจายผ่าน PIM Hello ที่เปิดด้วย "ip pim sparse-mode" อยู่แล้ว
! (นี่คือข้อได้เปรียบสำคัญของ BSR เทียบกับ Auto-RP ที่ยังต้องมี "ip pim autorp listener"
!  บนทุก Router แม้จะไม่ได้เป็น C-RP/Mapping Agent เลยก็ตาม)
```

> **หมายเหตุ**: คำสั่ง `ip pim rp-candidate <interface>` ในบาง IOS Version ใช้ Syntax
> `ip pim rp-candidate Loopback0 group-list 10 priority 0` (ไม่มี Keyword `interval` ในบาง Platform
> — Interval ของ C-RP Advertisement ไป BSR อาจถูกกำหนดโดย Timer ภายในของ BSR Protocol เอง
> ตรวจสอบ `?` ใน Config Mode ของแต่ละ Platform เสมอก่อนใช้งานจริง)

### ตรวจสอบผลลัพธ์

```
DIST-SW3# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is not a Candidate RP

Group(s) 239.0.0.0/8
  RP 1.1.1.1 (?), v2
    Info source: 1.1.1.1 (?), via bootstrap, priority 0, holdtime 150
         Uptime: 00:02:33, expires: 00:02:07

DIST-SW3# show ip pim bsr-router
PIMv2 Bootstrap Router is 1.1.1.1
  Priority 100, up for 00:15:44, expires: 00:01:52
```

`show ip pim bsr-router` เป็นคำสั่งเฉพาะของ BSR ที่ไม่มีในระบบ Auto-RP — ใช้ตรวจสอบว่า Router
ตัวไหนชนะ Election เป็น BSR ปัจจุบัน และ Timer จะ Expire เมื่อไหร่ (ถ้า BSR ตัวปัจจุบันหาย จะมี
C-BSR ตัวถัดไปเข้ามาแทนอัตโนมัติหลัง Timeout — Failover ในตัวเช่นเดียวกับ Auto-RP)

---

## Step 414 — Auto-RP vs BSR: เปรียบเทียบและเลือกใช้ให้ถูกงาน

| คุณสมบัติ | Auto-RP | BSR |
|---|---|---|
| มาตรฐาน | Cisco Proprietary | เปิด (RFC 5059) — Vendor ใดก็ Implement ได้ |
| วิธีกระจายข้อมูล RP-Set | Multicast Group แยก (`224.0.1.39`, `224.0.1.40`) | PIM Bootstrap Message ผ่าน PIM Neighbor ปกติ (Hop-by-Hop) |
| ต้อง Config เพิ่มบน Router ที่ไม่ใช่ C-RP/Agent | ต้องมี `ip pim autorp listener` ทุกตัว | **ไม่ต้อง** — ใช้ PIM Neighbor ที่มีอยู่แล้ว |
| ทิศทาง Priority ที่ Preferred | **ค่าต่ำ** ชนะ (Default 0) | **ค่าสูง** ชนะ |
| รองรับ Mixed-Vendor (มี Router ยี่ห้ออื่นในโดเมน) | ไม่รองรับ (ยี่ห้ออื่นไม่รู้จัก Group พิเศษนี้) | รองรับ — เป็น Standard ที่ทุกยี่ห้อ (Juniper, Arista, ฯลฯ) Implement ตาม RFC |
| ประวัติ/ความนิยมปัจจุบัน | เก่ากว่า ยังพบใน Legacy Cisco-only Network | ใหม่กว่า เป็นตัวเลือกแนะนำสำหรับ Design ปัจจุบัน |
| Failover เมื่อ RP ตัวหลักตาย | อัตโนมัติ (รอ Holdtime ของ RP-Announce) | อัตโนมัติ (รอ Holdtime ของ Candidate-RP ผ่าน BSR) |

### กฎการเลือกใช้ (สรุปสำหรับข้อสอบ CCNP ENCOR)

```
เลือก Auto-RP เมื่อ:
  - Network เป็น Cisco ล้วนทั้งหมด (ไม่มี Vendor อื่นปน)
  - เป็น Legacy Network ที่มี Auto-RP ใช้งานอยู่แล้วมานาน ไม่มีเหตุผลต้อง Migrate

เลือก BSR เมื่อ:
  - Network เป็น Mixed-Vendor (มี Router ที่ไม่ใช่ Cisco ต้องเข้าร่วม Domain เดียวกัน)
  - เป็น Design ใหม่ที่ไม่มีข้อจำกัดจาก Legacy — แนะนำเป็น Default choice สำหรับ Dynamic RP
    Discovery ในโครงการใหม่ เพราะเป็น Standard และ Config ง่ายกว่า (ไม่ต้องมี Listener Command)

ทั้งสองแบบยังมีข้อจำกัดร่วมกัน:
  - ยังมี "RP ตัวเดียวที่ Active" ในเวลาใดเวลาหนึ่งสำหรับแต่ละ Group-range
    (แค่เปลี่ยนตัว Active ได้อัตโนมัติเมื่อตัวเดิมตาย — ไม่ใช่ Load-Split ทำงานพร้อมกันจริง)
  - ระหว่างช่วง Failover (แม้จะสั้น) ยังมี Traffic Loss เกิดขึ้นเสมอ เพราะ RP ใหม่ต้องเรียนรู้
    Source ทั้งหมดใหม่จาก PIM Register อีกครั้ง (State เดิมที่ RP ตัวตายมีไว้ไม่ได้ถูกส่งต่อ)
```

### ตัวเลือกที่ 3 ที่แก้ข้อจำกัดสุดท้ายนี้ได้: Anycast RP

ถ้าต้องการ **RP มากกว่า 1 ตัวทำงานพร้อมกันจริง** (ไม่ใช่แค่ 1 Active + Standby ที่สลับกัน) พร้อม
**Load-Splitting** งาน RP ระหว่างหลาย Router ไปในตัว คำตอบคือ **Anycast RP** ซึ่งเป็นหัวข้อของ
Step 415-416 ต่อไป — และเป็นวิธีที่ Lab Step 420 ของ Part นี้จะใช้จริงเพื่อแก้ปัญหา CORE-SW1
Single Point of Failure จาก Step 411 ให้เด็ดขาดที่สุด

---

## Step 415 — Anycast RP (RFC 4610)

### แนวคิดหลัก: RP มากกว่า 1 ตัว ใช้ IP Address เดียวกัน

**Anycast RP** ไม่ได้พึ่งพา "การเลือก RP ที่ดีที่สุด" แบบ Auto-RP/BSR — แต่ใช้เทคนิคที่ตรงไปตรงมา
กว่ามาก: **ให้ Router มากกว่า 1 ตัว (ในตัวอย่างนี้คือ CORE-SW1 และ CORE-SW2) ประกาศ Loopback
Address เดียวกัน** (`1.1.1.100`) เข้าไปใน Unicast Routing Table (OSPF) — ทำให้ Router ตัวอื่นๆ
ใน Domain เห็น RP Address นี้ **มีสองเส้นทางไปถึง** และ Unicast Routing (OSPF SPF) จะเลือก
เส้นทางที่ **ใกล้ที่สุด (Cost ต่ำสุด)** โดยอัตโนมัติ — Router ที่อยู่ใกล้ CORE-SW1 มากกว่าจะ Register/
Join ไปทาง CORE-SW1 และ Router ที่อยู่ใกล้ CORE-SW2 มากกว่าจะ Register/Join ไปทาง CORE-SW2

```
                    ┌──────────────────┐              ┌──────────────────┐
                    │  CORE-SW1          │              │  CORE-SW2          │
                    │  Lo0: 1.1.1.1       │              │  Lo0: 1.1.1.2       │
                    │  Lo1: 1.1.1.100 ◄───┼──────────────┼──► Lo1: 1.1.1.100    │
                    │  (Anycast RP)       │   MSDP Peer  │  (Anycast RP)       │
                    │                    │◄────sync SA──►│                    │
                    └─────────┬──────────┘   (1.1.1.1↔1.1.1.2)└──────┬──────────┘
                              │                                       │
                    OSPF เห็น 1.1.1.100 ผ่าน Lo1                OSPF เห็น 1.1.1.100 ผ่าน Lo1
                    (2 เส้นทางเข้าถึง Address เดียวกัน — เลือกตามเส้นทางที่ Cost ต่ำสุดจากแต่ละจุด)

   DIST-SW1 (ใกล้ CORE-SW1)          DIST-SW3/4 (ใกล้ CORE-SW2)
   Register/Join ─────► CORE-SW1     Register/Join ─────► CORE-SW2
   (ใช้งาน RP "ตัวที่ใกล้ที่สุด" แบบอัตโนมัติ = Load-Splitting โดยไม่ต้อง Config เพิ่มอะไรฝั่ง DIST)
```

### ทำไมต้องมี MSDP คู่กับ Anycast RP เสมอ

ปัญหาที่เกิดจากการมี RP สองตัวคือ: **Source อาจ Register เข้า RP ตัวหนึ่ง แต่ Receiver อาจ Join
เข้า RP อีกตัว** (เพราะแต่ละฝั่งเลือก RP ที่ใกล้ที่สุดของตัวเอง ซึ่งอาจเป็น RP คนละตัวกัน) — ถ้า
CORE-SW1 รู้จัก Source ตัวหนึ่งที่ Register เข้ามา แต่ CORE-SW2 ไม่รู้จัก Source ตัวนั้นเลย
Receiver ที่ Join เข้า CORE-SW2 จะไม่ได้รับ Traffic นี้ — **MSDP (Step 416)** คือกลไกที่ทำให้
CORE-SW1 และ CORE-SW2 **บอกกันเองว่าใครมี Source อะไรอยู่บ้าง** ทำให้ RP ทั้งสองตัวมีข้อมูล
Source ครบเหมือนกันเสมอ แม้จะ Register เข้ามาคนละตัว

### Config เต็มรูปแบบ: Anycast RP บน CORE-SW1 + CORE-SW2

```
! ===== CORE-SW1 =====
CORE-SW1(config)# interface Loopback1
CORE-SW1(config-if)# ip address 1.1.1.100 255.255.255.255
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# network 1.1.1.100 0.0.0.0 area 0
CORE-SW1(config-router)# exit
CORE-SW1(config)# ip pim rp-address 1.1.1.100 10

! ===== CORE-SW2 =====
CORE-SW2(config)# interface Loopback1
CORE-SW2(config-if)# ip address 1.1.1.100 255.255.255.255
CORE-SW2(config-if)# ip pim sparse-mode
CORE-SW2(config-if)# exit
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# network 1.1.1.100 0.0.0.0 area 0
CORE-SW2(config-router)# exit
CORE-SW2(config)# ip pim rp-address 1.1.1.100 10

! ===== MSDP Peering ระหว่าง CORE-SW1 ↔ CORE-SW2 (ใช้ Loopback0 เดิม ไม่ใช่ Anycast Address) =====
CORE-SW1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW1(config)# ip msdp description 1.1.1.2 CORE-SW2-AnycastRP-Peer

CORE-SW2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
CORE-SW2(config)# ip msdp description 1.1.1.1 CORE-SW1-AnycastRP-Peer

! ===== DIST-SW1/DIST-SW3/DIST-SW4: เปลี่ยน Static RP ให้ชี้ Anycast Address ตัวเดียวกัน =====
DIST-SW1(config)# no ip pim rp-address 1.1.1.1
DIST-SW1(config)# ip pim rp-address 1.1.1.100 10
DIST-SW3(config)# no ip pim rp-address 1.1.1.1
DIST-SW3(config)# ip pim rp-address 1.1.1.100 10
DIST-SW4(config)# no ip pim rp-address 1.1.1.1
DIST-SW4(config)# ip pim rp-address 1.1.1.100 10
```

> **จุดสำคัญที่สุดของ Anycast RP**: MSDP Peer ใช้ **Loopback0 (Unique Address ของแต่ละ
> Router: `1.1.1.1` และ `1.1.1.2`)** ไม่ใช่ Anycast Address (`1.1.1.100`) — เพราะถ้าใช้ Anycast
> Address เป็น MSDP Source Address ทั้งคู่จะพยายาม Peer กับตัวเอง (Address เดียวกัน) ซึ่งใช้งาน
> ไม่ได้เลย MSDP ต้องใช้ Address ที่ **เจาะจงแยกแต่ละ Router จริง** เพื่อสร้าง TCP Session
> ระหว่างกัน (MSDP รันบน TCP Port 639) ในขณะที่ `ip pim rp-address` ทุก Router ในโดเมนต้องชี้
> ไปที่ Anycast Address (`1.1.1.100`) ตัวเดียวกันเหมือนเดิมทุกตัวอักษร

---

## Step 416 — MSDP (Multicast Source Discovery Protocol) เจาะลึก

### Source-Active (SA) Message — หัวใจของ MSDP

MSDP ทำงานผ่าน TCP Peering (คล้าย BGP ในเชิง Transport) ระหว่าง RP ของแต่ละ PIM-SM Domain
(หรือ RP คู่ Anycast เดียวกันแบบใน Step 415) โดยส่ง **Source-Active (SA) Message** ที่บอกว่า
**"ฉันมี Source ตัวนี้ (`S`) กำลังส่ง Group นี้ (`G`) อยู่นะ"** ทุกครั้งที่ RP เรียนรู้ Source ใหม่จาก
PIM Register:

```
STREAM-SRV Register เข้า CORE-SW1 (RP ที่ใกล้กว่า)
        │
        ▼
CORE-SW1 สร้าง (S,G) State ในตัวเอง
        │
        ▼
CORE-SW1 ส่ง SA Message ผ่าน MSDP Peer ──► CORE-SW2
        "Source-Active: (10.10.30.30, 239.1.1.1)"
        │
        ▼
CORE-SW2 บันทึกลง SA Cache ของตัวเอง (ไม่ต้องมี Receiver ก่อนก็บันทึกได้)
        │
        ▼
ถ้ามี Receiver Join เข้า CORE-SW2 ในอนาคต (แม้ Source Register ที่ CORE-SW1 เท่านั้น)
CORE-SW2 รู้จัก Source นี้อยู่แล้วจาก SA Cache → สร้าง (S,G) Join ตรงไปยัง Source ได้ทันที
```

### RPF Check สำหรับ SA Message — ป้องกัน SA Message วนลูป/ปลอมแปลง

เช่นเดียวกับ Multicast Data (Step 408 ของ Part 41), **SA Message เองก็ต้องผ่าน RPF Check**
ก่อนถูกยอมรับและ Forward ต่อไปยัง MSDP Peer อื่น — มิฉะนั้น SA Message อาจวนลูปไปมาไม่จบ
ในโดเมนที่มี MSDP Peer หลายเส้นทาง Cisco ใช้กฎ **Peer-RPF Check** ตามลำดับนี้:

| ลำดับกฎ | เงื่อนไข |
|---|---|
| 1 | ถ้ามี MSDP Peer เดียว (Point-to-point อย่างในตัวอย่าง Anycast RP ของเรา) → **ผ่านเสมอ** ไม่ต้องตรวจ RPF ซับซ้อน |
| 2 | ถ้ามีหลาย Peer และรัน MBGP/BGP ร่วมกัน → เลือก Peer ที่เป็น BGP Next-hop ไปยัง RP ต้นทางของ SA |
| 3 | ถ้าไม่มี BGP เลย → ใช้ Unicast Routing Table เทียบ Peer Address กับ Originator RP Address ของ SA |
| 4 (ทางลัดที่ Cisco แนะนำสำหรับ Design ง่ายๆ) | Config `ip msdp default-peer <peer-address>` เพื่อบอกว่า "เชื่อ SA ทั้งหมดจาก Peer นี้โดยไม่ต้องตรวจ RPF" — เหมาะกับกรณีมี MSDP Peer จำนวนน้อยแบบ Anycast RP 2 ตัวในหลักสูตรนี้ |

เพราะ Lab ของหลักสูตรนี้มี MSDP Peer แค่คู่เดียว (CORE-SW1 ↔ CORE-SW2) กฎข้อ 1 จึงครอบคลุม
อยู่แล้วโดยไม่ต้อง Config เพิ่ม — แต่การรู้จัก `ip msdp default-peer` ไว้สำคัญมากสำหรับ Design ระดับ
Enterprise ที่มี MSDP Peer มากกว่า 2 ตัว หรือไม่ได้รัน BGP ร่วมด้วย

### Use Case ที่สำคัญกว่า Anycast RP: Inter-Domain Multicast

MSDP ไม่ได้ถูกออกแบบมาเพื่อ Anycast RP เท่านั้น — งานดั้งเดิมของมันคือ **เชื่อม PIM-SM Domain
สองโดเมนที่แยกกันโดยสิ้นเชิง (มี RP ของตัวเองคนละตัว) ให้แลกเปลี่ยน Multicast Source กันได้**
ตัวอย่างที่พบบ่อยในโลกจริง:

```
บริษัท A ควบรวมกับบริษัท B (Merger & Acquisition)

  Site A (RP = 1.1.1.1, PIM-SM Domain ของตัวเอง)     Site B (RP = 2.2.2.1, PIM-SM Domain ของตัวเอง)
  มี Video Conference Server ส่ง 239.5.5.5              มี Employee ต้องการดู Video จาก Site A
       │                                                        │
       └──────────────── MSDP Peer ระหว่าง RP ทั้งสอง Site ──────┘
              (Site A RP ↔ Site B RP ผ่าน WAN Link ที่เชื่อมสองบริษัทหลัง M&A)

  ผลลัพธ์: Employee ที่ Site B Join Group 239.5.5.5 ได้ทันที แม้ Source อยู่คนละ PIM-SM Domain
  โดยไม่ต้องรวม Domain เป็นอันเดียวกัน (แต่ละ Site ยังคุม RP ของตัวเองแยกกันเหมือนเดิม)
```

นี่คือเหตุผลที่ MSDP ถูกจัดเป็น Protocol แยกจาก PIM-SM โดยสิ้นเชิงในหลักสูตร CCNP/CCIE — มันคือ
"BGP ของโลก Multicast" ที่ทำให้ Multicast ข้าม Administrative Domain ได้ ไม่ใช่แค่ Feature เสริม
ของ Anycast RP เท่านั้น (แม้ในทางปฏิบัติ Anycast RP เป็น Use Case ที่พบบ่อยที่สุดของ MSDP ในองค์กร
เดี่ยวๆ ก็ตาม)

---

## Step 417 — SSM (Source-Specific Multicast)

### แนวคิดหลัก: ไม่มี RP เข้ามาเกี่ยวข้องเลย

ทุกอย่างที่เรียนมาใน Part 41 และ Step 412-416 ของ Part นี้ล้วนพยายามแก้ปัญหาให้ RP มี
Redundancy — แต่ **SSM เลือกวิธีที่ต่างไปเลย: ตัด RP ออกจากสมการทั้งหมด** เพราะถ้า **Client รู้
IP Address ของ Source ล่วงหน้าอยู่แล้ว** (เช่น Application บอกไว้ชัดเจนว่า "รับ Feed จาก Server
ตัวนี้เท่านั้น") ก็ไม่มีความจำเป็นต้องมี RP เป็นจุดนัดพบเลย — Client บอก Router ไปตรงๆ ว่า
"ฉันต้องการ Group `G` จาก Source `S` นี้เท่านั้น" แล้ว Router สร้าง `(S,G)` Join ตรงไปยัง Source
ทันทีโดยไม่ผ่าน Shared Tree/RP เลยแม้แต่ขั้นตอนแรก

```
PIM-SM ทั่วไป (Step 406 ของ Part 41)              SSM (Step 417 นี้)

  Receiver ไม่รู้ Source ล่วงหน้า                    Receiver รู้ Source ล่วงหน้าแล้ว (App บอกไว้)
       │                                                   │
       ▼ ต้อง Join ผ่าน RP ก่อน (*,G)                       ▼ Join ตรงไปหา Source ทันที (S,G)
  Source ──Register──► RP ◄──Join── Receiver          Source ◄─────(S,G) Join ตรง──── Receiver
       │                                                   (ไม่มี RP, ไม่มี Register,
       ▼ Switchover ไป SPT ทีหลัง (Step 406)               ไม่มี Shared Tree เลยทั้งกระบวนการ)
  Source ──data ตรง (หลัง Threshold)──► Receiver
```

### `232.0.0.0/8` — ช่วง Address เฉพาะสำหรับ SSM

จากตาราง Multicast Address Scope ใน [Part 41 Step 401](part-041-multicast-fundamentals.md#step-401--multicast-คืออะไร-unicast-vs-broadcast-vs-multicast)
เราได้กล่าวถึงช่วงนี้ไว้แล้วสั้นๆ — Part นี้จะลงรายละเอียดเต็มรูปแบบ:

| คุณสมบัติ | รายละเอียด |
|---|---|
| ช่วง Address | `232.0.0.0` – `232.255.255.255` (`232.0.0.0/8`) |
| กำหนดโดย | IANA จองไว้เฉพาะสำหรับ SSM โดยเฉพาะ (แยกจาก `239.0.0.0/8` ที่ใช้กับ PIM-SM ปกติ) |
| IGMP Version ที่ต้องใช้ | **IGMPv3 เท่านั้น** (ต้องมี Source Filtering — IGMPv1/v2 บอก Source ไม่ได้เลย) |
| กลไก Filtering | **INCLUDE Mode** — Client ระบุ Source ที่ต้องการรับชัดเจน (ตรงข้ามกับ EXCLUDE Mode ที่บอกปฏิเสธ Source บางตัว) |

### Config: เปิด SSM บน Router (`ip pim ssm`)

```
! วิธีที่ 1: ใช้ Default Range (232.0.0.0/8) ตรงตาม IANA — แนะนำสำหรับ Design ทั่วไป
CORE-SW1(config)# ip pim ssm default

! วิธีที่ 2: ขยาย/กำหนด Range เอง ผ่าน Access-list (ถ้าต้องการใช้ Address นอกช่วง 232/8 เป็น SSM ด้วย)
CORE-SW1(config)# access-list 20 permit 232.0.0.0 0.255.255.255
CORE-SW1(config)# access-list 20 permit 239.100.0.0 0.0.255.255
CORE-SW1(config)# ip pim ssm range 20

! เปิด IGMPv3 บน Interface ฝั่ง Receiver (จำเป็นเสมอสำหรับ SSM — ต้องเปิดทุก Interface ที่มี Client)
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip igmp version 3
```

> **หมายเหตุสำคัญ**: ถ้า Client ส่ง IGMPv3 Report พร้อม Source Address สำหรับ Group ในช่วง
> `232.0.0.0/8` Router **จะสร้าง `(S,G)` State ตรงทันที โดยไม่มี `(*,G)` เกิดขึ้นเลยแม้แต่ชั่วขณะ**
> ต่างจาก PIM-SM ปกติที่ต้องผ่าน `(*,G)` ก่อนเสมอในตอนเริ่ม Join — นี่คือเหตุผลที่ `show ip mroute`
> ของ SSM Group จะไม่มี Field `RP` แสดงเลย (จะเห็นตัวอย่างจริงใน Step 420)

### เมื่อไหร่ควรใช้ SSM แทน PIM-SM ทั่วไป

| สถานการณ์ | เหตุผลที่ SSM เหมาะกว่า |
|---|---|
| **IPTV / Video Streaming จาก Head-end เดียวที่รู้จักแน่นอน** | Client (Set-top Box) ถูก Provision ให้รู้ IP ของ Head-end Server อยู่แล้วเสมอ ไม่มีความจำเป็นต้อง Discover Source ผ่าน RP |
| **Financial Market Data Feed** (เช่น NYSE/NASDAQ Feed, ตลาดหลักทรัพย์) | ผู้ให้บริการ Feed ประกาศ IP Source ที่ตายตัวอย่างเป็นทางการให้ลูกค้าทุกรายใช้ — Client ทุกตัว Config Source ตรงตาม Documentation ได้เลย ไม่ต้องพึ่ง RP Infrastructure ของผู้ให้บริการเลย |
| **ต้องการลด Attack Surface** | ไม่มี RP หมายความว่าไม่มี PIM Register/Shared Tree ให้โจมตี (เช่น การปลอม Source ส่ง Register ไปยัง RP เพื่อโจมตี Denial-of-Service ต่อ RP) — SSM ตัดช่องโหว่นี้ทั้งหมด |
| **ต้องการ Latency ต่ำที่สุดตั้งแต่ Packet แรก** | ไม่ต้องผ่าน Shared Tree ก่อนแล้ว Switchover ทีหลัง (เหมือน PIM-SM) — SSM ใช้ Shortest-Path ตั้งแต่ Join แรกเสมอ |

---

## Step 418 — Bidirectional PIM (Bidir-PIM)

### ปัญหาที่ Bidir-PIM แก้: Many-to-Many Multicast

ทุกอย่างที่เรียนมาจนถึงจุดนี้ (PIM-SM ปกติ, SSM) ถูกออกแบบสำหรับรูปแบบ **One-to-Many** (Source
หนึ่งตัว ส่งไปหลาย Receiver — เหมือน STREAM-SRV ของเรา) — แต่มี Use Case จำนวนหนึ่งที่ต้องการ
รูปแบบ **Many-to-Many** ที่ **ทุก Host ในกลุ่มเป็นทั้ง Source และ Receiver พร้อมกัน** ตัวอย่างคลาสสิก
คือ **Financial Trading Floor** ที่ Trader ทุกคนต้องส่ง/รับ Order Update ของกันและกันแบบ Real-time

```
PIM-SM ปกติสำหรับ Many-to-Many (ไม่มีประสิทธิภาพ)      Bidir-PIM สำหรับ Many-to-Many (ออกแบบมาโดยเฉพาะ)

Trader A,B,C,D ทุกคนเป็นทั้ง Source/Receiver           Trader A,B,C,D ทุกคนเป็นทั้ง Source/Receiver
ของ Group เดียวกัน (239.9.9.9)                          ของ Group เดียวกัน (239.9.9.9)
     │                                                       │
     ▼ ต้องสร้าง (S,G) แยกทุก Source                          ▼ ไม่มี (S,G) เลย — มีแค่ (*,G) เท่านั้น
  (A,G), (B,G), (C,G), (D,G) ... = N (S,G) Entries         Traffic ไหลผ่าน "Shared Tree ทั้งสองทาง"
  ถ้ามี Trader 1000 คน = 1000 (S,G) State บนทุก Router       (ขึ้นและลง RP Tree พร้อมกันได้) — RP
  ใช้ Memory มหาศาลและ RP ต้องจัดการ Register จาก             ไม่ต้องเห็น Traffic จริงเลยด้วยซ้ำ (แค่เป็น
  ทุกคนพร้อมกัน (Scale ไม่ได้เลยในทางปฏิบัติ)                 "จุดอ้างอิงของ Tree" เท่านั้น) → Scale ดีมาก
```

### กลไกหลักที่ต่างจาก PIM-SM: ไม่มี (S,G), ไม่มี Register, มี DF แทน

| แนวคิด | PIM-SM ปกติ | Bidir-PIM |
|---|---|---|
| Source Tree (`S,G`) | มี — ใช้หลัง SPT Switchover | **ไม่มีเลย** — ใช้ Shared Tree (`*,G`) เท่านั้นตลอดชีวิตของ Flow |
| PIM Register Message | มี — First-Hop Router ต้อง Register ไปยัง RP | **ไม่มี** — Traffic ไหลขึ้น-ลง Shared Tree ได้ทั้งสองทางโดยตรง |
| ตัวควบคุม Loop-prevention บนแต่ละ Link | RPF Check ปกติ | **DF (Designated Forwarder)** — Router ที่ถูกเลือกให้เป็นทางเดียวที่ Forward Traffic ขึ้น-ลง Link นั้นไปทาง RP (เลือกจาก Router ที่มี Metric ไปยัง RP ดีที่สุดบน Link) |
| RP ต้องเป็น Real Device ที่ Process Traffic จริงหรือไม่ | ต้อง (RP อยู่ใน Path จริงช่วงก่อน Switchover) | **ไม่จำเป็น** — ในหลาย Design RP Address เป็นแค่ Phantom Address (ไม่มี Physical Device ใช้ Address นี้จริง) เพราะ RP ทำหน้าที่แค่เป็น "จุดอ้างอิงของ Tree" ไม่ต้องประมวลผล Data Plane เลย |

### Config โดยสรุป (แนะนำให้รู้จักไว้ ไม่ลง Lab เต็มใน Part นี้)

```
CORE-SW1(config)# ip pim rp-address 1.1.1.100 20 bidir
! Keyword "bidir" ต่อท้าย ip pim rp-address ทำให้ RP นี้ทำงานแบบ Bidirectional
! สำหรับ Group ที่ Match Access-list 20 เท่านั้น (Group อื่นยัง PIM-SM ปกติได้ตามเดิม)
```

> **สรุปสำหรับ CCNP ENCOR**: Bidir-PIM เป็นหัวข้อที่ข้อสอบมักถามแค่ **"เมื่อไหร่ควรใช้"** (คำตอบ:
> Many-to-Many Application ที่มี Source จำนวนมากพร้อมกัน เช่น Financial Trading, Multi-party
> Video Conference แบบ Mesh) และ **"ต่างจาก PIM-SM อย่างไร"** (ไม่มี (S,G), ใช้ DF แทน RPF Check
> แบบเดิม) มากกว่าจะถาม Syntax Config เชิงลึก — หลักสูตรนี้จึงให้แค่ภาพรวมที่เพียงพอสำหรับ
> ระดับ ENCOR โดยไม่ลง Lab เต็มรูปแบบ (Bidir-PIM Lab เชิงลึกเหมาะกับระดับ CCIE Service Provider
> มากกว่า Enterprise Track ที่หลักสูตรนี้มุ่งเน้น)

---

## Step 419 — Verification & Troubleshooting ขั้นสูง

### `show ip pim rp` — สรุป RP ทั้งหมดที่ Router รู้จัก พร้อมวิธีที่เรียนรู้มา

```
CORE-SW2# show ip pim rp
Group: 239.1.1.1, RP: 1.1.1.100, uptime 00:45:12, expires never
Group: 224.0.1.39, RP: 0.0.0.0, uptime 00:00:00, expires never
```

### `show ip pim rp mapping` — เจาะดูว่า Group-range ไหนแม็พกับ RP ตัวไหน และมาจากวิธีไหน

```
DIST-SW3# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via Auto-RP
         Uptime: 00:20:14, expires: 00:02:41
```

ให้สังเกต Field **`Info source:`** เสมอเมื่อ Troubleshoot — จะบอกตรงๆ ว่า Router ได้ RP Mapping
นี้มาจาก **`Static`, `Auto-RP`,** หรือ **`bootstrap` (BSR)** — ถ้า Router สองตัวใน Domain เดียวกัน
แสดงผลต่างกัน (เช่นตัวหนึ่งบอก `Static` อีกตัวบอก `via Auto-RP`) นี่คือสัญญาณของ **RP Mapping
Inconsistency** ที่ต้องแก้ทันที

### Diagnosing RP-Mapping Inconsistency — Checklist มาตรฐาน

| อาการที่เจอ | สาเหตุที่เป็นไปได้ | วิธีตรวจสอบ |
|---|---|---|
| Router บางตัวเห็น RP ถูกต้อง บางตัวไม่เห็น RP เลย | ลืม Config `ip pim autorp listener` (Auto-RP) บาง Router | `show running-config \| include autorp` เทียบทุก Router |
| Router บางตัวเห็น RP เป็นตัวเก่า (Static) บางตัวเห็น RP ใหม่ (Dynamic) | Migrate จาก Static RP ไป Auto-RP/BSR ไม่ครบทุก Router (ลืมลบ `ip pim rp-address` เดิม) | `show running-config \| include ip pim rp-address` — Static Config จะ Override Dynamic เสมอถ้ายังเหลืออยู่ |
| Auto-RP ทำงานได้บางส่วน (Local Segment) แต่ไม่กระจายไปทั้ง Domain | Group `224.0.1.39/40` ถูก Deny โดย Multicast Boundary/ACL ที่ตั้งไว้ (เช่น `ip multicast boundary`) โดยไม่ตั้งใจ | ตรวจ ACL ที่ผูกกับ `ip multicast boundary` ทุก Interface กลาง Path |
| BSR ไม่เห็น Bootstrap Message ข้าม Link บางเส้น | PIM Neighbor ยังไม่ขึ้น (ไม่ได้เปิด `ip pim sparse-mode` บน Interface นั้น) | `show ip pim neighbor` ตรวจให้ครบทุกทิศทางก่อนสงสัย BSR |
| Anycast RP ทำงานได้แค่ครึ่งเดียว (Source ฝั่งหนึ่งไปไม่ถึง Receiver อีกฝั่ง) | MSDP Peer ไม่ขึ้น หรือ SA ไม่ถูก Sync | `show ip msdp summary` และ `show ip msdp sa-cache` (ดูรายละเอียดถัดไป) |

### `show ip msdp summary` — ตรวจสถานะ MSDP Peer

```
CORE-SW1# show ip msdp summary
MSDP Peer Status Summary
Peer Address    AS    State   Uptime/   Reset  SA     Peer Name
                              Downtime  Count   Count
1.1.1.2          -     Up      00:32:10  0       3      CORE-SW2-AnycastRP-Peer
```

`State: Up` คือค่าที่ต้องเห็นเสมอ — ถ้าเป็น `Connecting` หรือ `Listen` ค้างนาน แปลว่า TCP Port 639
ระหว่างสอง Peer ถูก Block (ตรวจ ACL/Firewall ระหว่างทาง) หรือ `connect-source` Config ผิด
Interface ที่ Peer ปลายทางเข้าไม่ถึง

### `show ip msdp sa-cache` — ดู Source ทั้งหมดที่เรียนรู้ผ่าน MSDP

```
CORE-SW2# show ip msdp sa-cache
MSDP Source-Active Cache - 1 entries
(10.10.30.30, 239.1.1.1), RP 1.1.1.1, MBGP/AS -, 00:04:12/00:05:48, Peer 1.1.1.1
```

Entry นี้ที่เห็นบน **CORE-SW2** (ไม่ใช่ RP ที่ STREAM-SRV Register เข้าไปตรงๆ) พิสูจน์ว่า MSDP
ทำงานถูกต้อง — CORE-SW2 รู้จัก Source `10.10.30.30` ผ่าน SA Message ที่ CORE-SW1 ส่งมา
(`Peer 1.1.1.1`) แม้ Source จะ Register เข้า CORE-SW1 เท่านั้นก็ตาม

### `show ip pim autorp` — สถานะเฉพาะของ Auto-RP (ถ้ายังใช้อยู่)

```
CORE-SW1# show ip pim autorp
AutoRP Information:
 AutoRP Discovery/Announce timer expires in: 00:00:34
 PIM AutoRP Statistics: Sent/Received
    RP-Announce: 45/12, RP-Discovery: 45/0
 This system is an RP-Mapping Agent for these groups:
     Group-acl: 10, RP: 1.1.1.1, expires: 00:02:47
 This system is a Candidate RP for these groups:
     Group-acl: 10, priority: 0
```

คำสั่งนี้ให้ภาพรวมทั้งบทบาท C-RP และ Mapping Agent ในหน้าจอเดียว — สะดวกมากสำหรับตรวจสอบ
ว่า Router ตัวนี้กำลังทำหน้าที่ไหนอยู่บ้าง (Router หนึ่งตัวสามารถเป็นทั้ง C-RP และ Mapping Agent
พร้อมกันได้ เหมือน CORE-SW1 ในตัวอย่าง Step 412)

---

## Step 420 — Lab เต็มรูปแบบ: Anycast RP + MSDP แทน Static RP พร้อมเปิด SSM

### เป้าหมายของ Lab นี้

1. **แทนที่ Static RP (`1.1.1.1`) ของ Part 41 ด้วย Anycast RP (`1.1.1.100`)** บน CORE-SW1 และ
   CORE-SW2 พร้อม MSDP Peering — แก้ปัญหา Single Point of Failure จาก Step 411 ให้เด็ดขาด
2. **เพิ่ม Source ใหม่แบบ Low-Latency Market-Data-Style** ที่ใช้ **SSM** (ไม่ต้องพึ่ง RP เลย)
   เพื่อสาธิตว่าทั้งสองเทคนิค (Anycast RP สำหรับ PIM-SM ทั่วไป และ SSM สำหรับ Traffic ที่รู้ Source
   ล่วงหน้า) ใช้งานร่วมกันในโดเมนเดียวกันได้พร้อมกัน
3. **ทดสอบ RP Failover จริง** ด้วยการ Shutdown Loopback ที่ทำหน้าที่ Anycast RP บน CORE-SW1
   แล้วพิสูจน์ว่า CORE-SW2 ยังให้บริการ Multicast ต่อได้โดยไม่มี Downtime ของทั้ง Domain

### Topology ที่อัปเดต (ต่อจาก Part 41 Step 410)

```
                                   ┌──────────────────────┐
                                   │     CORE-SW1            │
                                   │   Lo0: 1.1.1.1           │
                                   │   Lo1: 1.1.1.100 (Anycast RP #1) │
                                   └───────────┬───────────┘
                                          Po1  │◄──MSDP Peer (1.1.1.1 ↔ 1.1.1.2)──►
                                   ┌───────────┴───────────┐
                                   │     CORE-SW2            │
                                   │   Lo0: 1.1.1.2           │
                                   │   Lo1: 1.1.1.100 (Anycast RP #2) │
                                   └────┬─────────────┬────┘
                        Te1/0/1 (10.255.30.0/30)  │   │ Te1/0/2 (10.255.40.0/30)
                          ┌──────────────┘           └──────────────┐
                    ┌─────┴─────┐                              ┌─────┴─────┐
                    │  DIST-SW3   │                              │  DIST-SW4   │
                    └─────┬───────┘                              └─────┬───────┘
                          │ Trunk                                      │ Trunk
                    ┌─────┴───────┐                                ┌─────┴───────┐
                    │  ACCESS-SW3   │                                │  ACCESS-SW4   │
                    └───────┬───────┘                                └───────┬───────┘
                            │ Gi0/5 (Vlan30)         │ Gi0/6 (Vlan30, ใหม่)      │ Gi0/10 (Vlan40)
                      ┌─────┴─────┐             ┌─────┴──────┐              ┌─────┴─────┐
                      │ STREAM-SRV  │             │MARKET-SRV   │              │ PC-WIFI1    │
                      │10.10.30.30  │──239.1.1.1─►│10.10.30.40  │──232.50.1.1─►│10.10.40.51  │
                      └────────────┘             └────────────┘              └────────────┘
                        (PIM-SM ปกติ ผ่าน Anycast RP)  (SSM — ไม่มี RP เกี่ยวข้องเลย)
                                                                        ┌──────────────┐
                                                        (S,G) Join ตรง ►│  PC-SALES1     │
                                                        ไปยัง MARKET-SRV │  10.10.10.11   │
                                                                        └──────────────┘
```

### ส่วนที่ 1: Config Anycast RP + MSDP (แทนที่ Static RP เดิมทั้งหมด)

```
! ===== CORE-SW1: ลบ Static RP เดิม, เพิ่ม Loopback1 Anycast, MSDP Peer =====
CORE-SW1(config)# no ip pim rp-address 1.1.1.1
CORE-SW1(config)# access-list 10 permit 239.0.0.0 0.255.255.255
CORE-SW1(config)# interface Loopback1
CORE-SW1(config-if)# ip address 1.1.1.100 255.255.255.255
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# network 1.1.1.100 0.0.0.0 area 0
CORE-SW1(config-router)# exit
CORE-SW1(config)# ip pim rp-address 1.1.1.100 10
CORE-SW1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW1(config)# ip msdp description 1.1.1.2 CORE-SW2-AnycastRP-Peer

! ===== CORE-SW2: เหมือนกันทุกประการ (ยกเว้น Peer Address ชี้กลับมาที่ CORE-SW1) =====
CORE-SW2(config)# no ip pim rp-address 1.1.1.1
CORE-SW2(config)# access-list 10 permit 239.0.0.0 0.255.255.255
CORE-SW2(config)# interface Loopback1
CORE-SW2(config-if)# ip address 1.1.1.100 255.255.255.255
CORE-SW2(config-if)# ip pim sparse-mode
CORE-SW2(config-if)# exit
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# network 1.1.1.100 0.0.0.0 area 0
CORE-SW2(config-router)# exit
CORE-SW2(config)# ip pim rp-address 1.1.1.100 10
CORE-SW2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
CORE-SW2(config)# ip msdp description 1.1.1.1 CORE-SW1-AnycastRP-Peer

! ===== DIST-SW1, DIST-SW3, DIST-SW4: ชี้ไป Anycast Address แทน Static เดิม =====
DIST-SW1(config)# no ip pim rp-address 1.1.1.1
DIST-SW1(config)# ip pim rp-address 1.1.1.100 10
DIST-SW3(config)# no ip pim rp-address 1.1.1.1
DIST-SW3(config)# ip pim rp-address 1.1.1.100 10
DIST-SW4(config)# no ip pim rp-address 1.1.1.1
DIST-SW4(config)# ip pim rp-address 1.1.1.100 10
```

### ส่วนที่ 2: Config SSM สำหรับ MARKET-SRV (`10.10.30.40` → `232.50.1.1`)

```
! ===== เปิด SSM Range บน Router ทุกตัวที่เกี่ยวข้อง (Global Config เดียวกันทุก Router) =====
CORE-SW1(config)# ip pim ssm default
CORE-SW2(config)# ip pim ssm default
DIST-SW1(config)# ip pim ssm default
DIST-SW3(config)# ip pim ssm default

! ===== DIST-SW1: เปิด IGMPv3 บน Vlan10 SVI (ฝั่ง Receiver PC-SALES1) =====
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip igmp version 3
DIST-SW1(config-if)# exit

! ===== ACCESS-SW3: เพิ่ม Port ใหม่สำหรับ MARKET-SRV (VLAN 30 เดิม) =====
ACCESS-SW3(config)# interface GigabitEthernet0/6
ACCESS-SW3(config-if)# switchport mode access
ACCESS-SW3(config-if)# switchport access vlan 30
ACCESS-SW3(config-if)# exit

! ===== ACCESS-SW1: จำลอง PC-SALES1 ทำ IGMPv3 (S,G) Join แบบเจาะจง Source =====
ACCESS-SW1(config)# interface GigabitEthernet0/8
ACCESS-SW1(config-if)# ip igmp version 3
ACCESS-SW1(config-if)# ip igmp static-group 232.50.1.1 source 10.10.30.40
ACCESS-SW1(config-if)# exit
```

> **สังเกต**: SSM Config ไม่มี `ip pim rp-address` เกี่ยวข้องเลยแม้แต่บรรทัดเดียว — Group
> `232.50.1.1` ไม่ต้องมี RP มาเกี่ยวข้องทั้งกระบวนการ ตรงตามหลักการ Step 417 ทุกประการ

### Verification 1: RP Mapping หลัง Migrate ไป Anycast RP

```
DIST-SW3# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is not a Mapping Agent

Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via static
         Uptime: 00:10:05, expires never

DIST-SW3# show ip route 1.1.1.100
Routing entry for 1.1.1.100/32
  Known via "ospf 1", distance 110, metric 2
  Routing Descriptor Blocks:
  * 10.255.30.1, from 1.1.1.2, 00:10:05 ago, via TenGigabitEthernet1/1/1
```

สังเกตว่า **DIST-SW3 เลือกเส้นทางไป Anycast RP ผ่าน CORE-SW2 (`from 1.1.1.2`)** เพราะอยู่ใกล้กว่า
(DIST-SW3 ต่อ CORE-SW2 โดยตรงตาม Topology จาก
[`00-ip-address-plan.md`](00-ip-address-plan.md)) — นี่คือ Load-Splitting ที่เกิดขึ้นอัตโนมัติจาก
คุณสมบัติของ Anycast RP โดยไม่ต้อง Config อะไรเพิ่มที่ DIST-SW3 เลย

### Verification 2: MSDP ทำงานถูกต้อง (SA Sync ระหว่าง CORE-SW1 ↔ CORE-SW2)

```
CORE-SW1# show ip msdp summary
MSDP Peer Status Summary
Peer Address    AS    State   Uptime/   Reset  SA     Peer Name
                              Downtime  Count   Count
1.1.1.2          -     Up      00:15:40  0       1      CORE-SW2-AnycastRP-Peer

CORE-SW2# show ip msdp sa-cache
MSDP Source-Active Cache - 1 entries
(10.10.30.30, 239.1.1.1), RP 1.1.1.1, MBGP/AS -, 00:02:18/00:05:42, Peer 1.1.1.1
```

STREAM-SRV Register เข้า CORE-SW1 (เพราะ ACCESS-SW3/DIST-SW3 อยู่ใกล้ CORE-SW2 แต่ Traffic
Register ครั้งแรกอาจใช้เส้นทางที่ OSPF เลือกในจังหวะนั้น — ในตัวอย่างนี้คือ CORE-SW1) แต่ CORE-SW2
ก็รู้จัก Source นี้ผ่าน MSDP SA Cache ทันที พร้อม Forward ให้ Receiver ที่ Join เข้าฝั่งตัวเองได้เสมอ

### Verification 3: `show ip mroute` ของ SSM Group — ไม่มี RP Field เลย

```
DIST-SW1# show ip mroute 232.50.1.1
IP Multicast Routing Table
Flags: D - Dense, S - Sparse, s - SSM Group, C - Connected, L - Local, ...

(10.10.30.40, 232.50.1.1), 00:03:44/00:03:15, flags: sTI
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr 10.255.10.1
  Outgoing interface list:
    Vlan10, Forward/Sparse, 00:03:44/00:03:15
```

สังเกต Flag **`s`** (SSM Group) และ **ไม่มี Field `RP`** เลยเมื่อเทียบกับ Entry ของ `239.1.1.1`
ใน Part 41 ที่มี `RP 1.1.1.1, flags: SJC` — และไม่มี `(*, G)` Entry สำหรับ Group นี้เกิดขึ้นเลยแม้แต่
ช่วงเริ่ม Join ตรงตามหลักการ SSM ที่ Join ตรงไปยัง Source ทันทีตั้งแต่แรก

### Verification 4: RP Failover Test — Shutdown Anycast RP บน CORE-SW1

```
! จำลองสถานการณ์ "CORE-SW1 ไม่สามารถทำหน้าที่ RP ได้อีกต่อไป" (เช่น Process Crash เฉพาะ
! ส่วน Multicast แต่ตัว Router ยังไม่ตายทั้งเครื่อง หรือจำลอง Maintenance เจาะจง Loopback1)
CORE-SW1(config)# interface Loopback1
CORE-SW1(config-if)# shutdown
```

**ผลลัพธ์ทันทีหลัง Shutdown**:

```
! OSPF ของ CORE-SW1 ถอด Route 1.1.1.100/32 ออกจาก LSA ทันที (Loopback1 Down)
! ทุก Router ใน Domain Reconverge ผ่าน OSPF ปกติ — เหลือแค่เส้นทางไปทาง CORE-SW2 เท่านั้น

DIST-SW1# show ip route 1.1.1.100
Routing entry for 1.1.1.100/32
  Known via "ospf 1", distance 110, metric 3
  Routing Descriptor Blocks:
  * 10.255.20.1, from 1.1.1.2, 00:00:03 ago, via TenGigabitEthernet1/1/1
  ! เปลี่ยนจาก "from 1.1.1.1" (ก่อน Shutdown) เป็น "from 1.1.1.2" ทันที — เส้นทางเดียวที่เหลือ

DIST-SW1# show ip pim rp mapping
Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via static
         Uptime: 00:25:11, expires never
! RP Address ยังเป็น "1.1.1.100" เดิมเป๊ะ — DIST-SW1 ไม่ต้องเปลี่ยน Config อะไรเลยแม้แต่บรรทัดเดียว
! เพราะ RP Address (1.1.1.100) ไม่เปลี่ยน มีแค่ "ใครตอบเป็นเจ้าของ Address นี้" ที่เปลี่ยนไป
! ผ่าน OSPF Reconvergence เท่านั้น — Root Cause ของความเร็ว Failover ทั้งหมดอยู่ที่นี่

CORE-SW2# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:00:08/00:02:59, RP 1.1.1.100, flags: SJC
  Incoming interface: Loopback1, RPF nbr 0.0.0.0
  Outgoing interface list:
    TenGigabitEthernet1/0/1, Forward/Sparse, 00:00:08/00:02:59
    TenGigabitEthernet1/0/2, Forward/Sparse, 00:00:08/00:02:59
! CORE-SW2 กลายเป็น RP ตัวเดียวที่ Active สำหรับ (*,G) State ใหม่ทั้งหมด (Incoming interface
! เปลี่ยนเป็น Loopback1 ของตัวเอง) — Receiver ใหม่ที่ Join เข้ามาระหว่างนี้ยังทำงานได้ปกติทุกประการ
! เทียบกับ Step 411 ที่ Static RP ตายแล้ว Domain พังทั้งหมดโดยไม่มีทางแก้อัตโนมัติ
```

**สรุปผลการทดสอบ**: หลัง Shutdown Loopback1 บน CORE-SW1 (จำลอง RP Function ล่ม) — Traffic
ที่กำลังไหลอยู่ (`(S,G)` ที่ Switchover ไป SPT แล้ว) **ไม่กระทบเลย** เพราะไม่พึ่ง RP อยู่แล้ว และ
ที่สำคัญกว่าคือ **Receiver ใหม่ที่ Join เข้ามาระหว่างที่ CORE-SW1 ตาย ยังใช้งานได้ปกติทุกประการ**
ผ่าน CORE-SW2 โดยอัตโนมัติ ภายในเวลาเท่ากับ OSPF Convergence เท่านั้น (ระดับ Sub-second ถึง
ไม่กี่วินาที) — นี่คือความแตกต่างเชิง Operation ที่ชัดเจนที่สุดเมื่อเทียบกับ Static RP เดี่ยวใน Part 41
ที่ต้องรอ Human เข้าไปแก้ Manual ตามที่วิเคราะห์ไว้ใน Step 411

```
CORE-SW1(config)# interface Loopback1
CORE-SW1(config-if)# no shutdown
! คืนสภาพ Lab กลับปกติ — Anycast RP #1 กลับมาทำหน้าที่ Load-Split ร่วมกับ CORE-SW2 ตามเดิม
```

---

## แบบฝึกหัดทวนความเข้าใจ Part 42

1. อธิบายว่าทำไม Static RP ตัวเดียว (`ip pim rp-address 1.1.1.1` แบบ Part 41) จึงถือเป็น Single
   Point of Failure และยกตัวอย่าง State 2 ประเภทใน `show ip mroute` ที่ได้รับผลกระทบต่างกัน
   เมื่อ RP ตัวนั้นล่ม
2. เปรียบเทียบทิศทางของค่า Priority ที่ Preferred ระหว่าง Auto-RP กับ BSR พร้อมอธิบายว่าทำไม
   Auto-RP ต้องมีคำสั่ง `ip pim autorp listener` บนทุก Router แต่ BSR ไม่ต้องมี Config เพิ่มเลย
   บน Router ที่ไม่ใช่ Candidate
3. Anycast RP ต้องใช้ MSDP คู่กันเสมอเพราะอะไร และทำไม MSDP Peer ต้องใช้ Loopback0 (Unique
   Address) ไม่ใช่ Anycast Address (`1.1.1.100`) เป็น `connect-source`?
4. อธิบายว่าทำไม SSM (`232.0.0.0/8`) ไม่ต้องมี RP เลย และต้องใช้ IGMP Version ใดเป็นอย่างน้อย
   พร้อมยกตัวอย่าง Use Case ที่ SSM เหมาะกว่า PIM-SM ทั่วไป
5. ใน Lab Step 420 หลัง Shutdown Loopback1 ของ CORE-SW1 (จำลอง RP ล่ม) เหตุใด Traffic ของ
   `239.1.1.1` ที่กำลังไหลผ่าน SPT อยู่แล้ว (State แบบ `(S,G)`) จึงไม่กระทบเลย ในขณะที่ Receiver
   ใหม่ที่ Join เข้ามาระหว่างนั้นยังคงใช้งานได้ปกติผ่าน CORE-SW2 — อะไรคือกลไกที่ทำให้ Failover
   เกิดขึ้นได้เร็วเพียงนี้โดยไม่ต้องมีคน Manual แก้ไข?

**เฉลย:**

1. Static RP เป็น IP เดียวที่ Hardcode ไว้ในทุก Router โดยไม่มีกลไกตรวจสอบ Reachability หรือ
   Failover อัตโนมัติเลย — ถ้า RP ล่ม State แบบ `(S,G)` ที่ Switchover ไป SPT แล้ว **ไม่กระทบ**
   เพราะ RPF Interface ชี้ไปทาง Source ตรงๆ ไม่พึ่ง RP อีกต่อไป แต่ State แบบ `(*,G)` (Shared
   Tree ที่ยังไม่ Switchover หรือ Receiver ใหม่ที่พึ่ง Join) **จะพังทันที** เพราะ RPF Interface ของ
   Entry นี้ต้องชี้ไปทาง RP เสมอ เมื่อ RP หายไปก็ไม่มีทางสร้าง/คง State นี้ต่อได้เลย
2. Auto-RP: **ค่า Priority ต่ำกว่าชนะ** (Default 0) ส่วน BSR: **ค่า Priority สูงกว่าชนะ** — Auto-RP
   ต้องมี `ip pim autorp listener` บนทุก Router เพราะ RP-Announce/RP-Discovery ถูกส่งผ่าน
   Multicast Group พิเศษ (`224.0.1.39`/`224.0.1.40`) ที่ต้อง Flood แบบ Dense-mode คำสั่งนี้ทำให้
   Router รับ/ส่งสอง Group นี้ได้โดยไม่ต้องเปิด Dense Mode เต็ม Interface ในขณะที่ BSR ส่ง
   Bootstrap Message ผ่าน PIM Neighbor ปกติ (Hop-by-Hop ผ่าน PIM Hello Mechanism ที่เปิดด้วย
   `ip pim sparse-mode` อยู่แล้ว) จึงไม่ต้องมี Config เพิ่มเติมใดๆ บน Router ที่ไม่ใช่ Candidate
3. Source อาจ Register เข้า RP ตัวหนึ่ง แต่ Receiver อาจ Join เข้า RP อีกตัว (เพราะแต่ละฝั่งเลือก
   RP ที่ Unicast Routing มองว่าใกล้ที่สุดของตัวเอง ซึ่งอาจเป็น RP คนละตัว) — MSDP ทำให้ RP ทั้ง
   สองตัวแลกเปลี่ยน SA Message บอกกันว่าใครมี Source อะไรอยู่บ้าง ทำให้ข้อมูล Source Sync กัน
   ครบเสมอ ส่วนการใช้ Loopback0 (Unique) เป็น `connect-source` เพราะถ้าใช้ Anycast Address
   ทั้งสอง Router จะมี Address เดียวกันเป๊ะ ทำให้พยายาม TCP Peer กับตัวเอง ใช้งานไม่ได้เลย
   MSDP ต้องอาศัย Address ที่แยกแต่ละ Router จริงเพื่อสร้าง TCP Session ที่ถูกต้อง
4. SSM ไม่ต้องมี RP เพราะ Client (ผ่าน IGMPv3 INCLUDE Mode) บอก Source Address ที่ต้องการ
   ชัดเจนอยู่แล้วในขั้นตอน Join เอง Router จึงสร้าง `(S,G)` ตรงไปยัง Source ได้ทันทีโดยไม่ต้องมี
   RP เป็นจุดนัดพบเลย ต้องใช้ **IGMPv3 เป็นอย่างน้อย** (v1/v2 ไม่มี Source Filtering) — เหมาะกับ
   Use Case ที่ Client รู้ Source ล่วงหน้าแน่นอน เช่น IPTV จาก Head-end ที่ตายตัว หรือ Financial
   Market Data Feed ที่ผู้ให้บริการประกาศ Source Address เป็นทางการให้ลูกค้าทุกรายใช้เหมือนกัน
5. Traffic ที่ไหลผ่าน SPT (`(S,G)`) ไม่กระทบเพราะ RPF Interface ของ Entry นี้ชี้ไปทาง Source
   โดยตรงอยู่แล้ว ไม่เคยพึ่ง RP ตั้งแต่ Switchover สำเร็จ ส่วน Receiver ใหม่ยังใช้งานได้เพราะ
   **Anycast RP Address (`1.1.1.100`) ยังคงเป็น Address เดียวกันเสมอ** มีแค่ "ใครตอบเป็นเจ้าของ
   Address นี้อยู่จริง" ที่เปลี่ยนไปตาม OSPF Reconvergence เท่านั้น — เมื่อ CORE-SW1 ถอด Route
   `1.1.1.100/32` ออกจาก LSA ทุก Router จะเห็นเส้นทางเดียวที่เหลือคือผ่าน CORE-SW2 โดยอัตโนมัติ
   ภายในเวลา OSPF Convergence ปกติ (Sub-second ถึงไม่กี่วินาที) โดยไม่ต้องมีคนเข้าไปเปลี่ยน
   Config `ip pim rp-address` บน Router ตัวใดเลยแม้แต่บรรทัดเดียว — นี่คือข้อได้เปรียบหลักของ
   Anycast RP เทียบกับ Static RP เดี่ยว

---

## สรุป Part 42

Part นี้ปิดจุดอ่อนสำคัญที่สุดที่ Part 41 ทิ้งไว้ — **Single Point of Failure ของ Static RP** — ด้วย
เครื่องมือ 3 ระดับ: **Auto-RP** (Cisco Proprietary, ใช้ Multicast Group พิเศษ, Priority ต่ำชนะ),
**BSR** (มาตรฐาน RFC 5059, ใช้ PIM Message ปกติ Hop-by-Hop, Priority สูงชนะ, เหมาะกับ
Mixed-Vendor Network) และที่สำคัญที่สุดคือ **Anycast RP (RFC 4610)** ที่ให้ RP มากกว่า 1 ตัว
ทำงานพร้อมกันจริงด้วย IP Address เดียวกัน พร้อม **MSDP** ทำหน้าที่ Sync Source-Active
Information ระหว่าง RP ทั้งสองตัว — เราได้เรียนรู้ **SSM (Source-Specific Multicast)** ที่ตัด RP
ออกจากสมการทั้งหมดสำหรับ Traffic ที่ Client รู้ Source ล่วงหน้า (เหมาะกับ IPTV/Market Data Feed)
และปิดท้ายด้วยภาพรวมของ **Bidirectional PIM** สำหรับ Use Case Many-to-Many แบบ Financial
Trading Lab เต็มรูปแบบใน Step 420 พิสูจน์ด้วยการทดสอบจริงว่าเมื่อ Shutdown Anycast RP Function
บน CORE-SW1 ไป Domain ทั้งหมดยังให้บริการต่อได้ผ่าน CORE-SW2 โดยอัตโนมัติ ไม่ต้องมี Human
เข้าไปแก้ Config Manual เหมือนที่วิเคราะห์ไว้ใน Step 411 อีกต่อไป — Multicast Design ของเราตอนนี้
พร้อมใช้งานระดับ Production เต็มรูปแบบแล้ว

Multicast ทั้งสอง Part (41-42) ปิดหัวข้อ Multicast ของหลักสูตรนี้ในระดับ CCNP ENCOR — Part
ถัดไปจะเปลี่ยนโฟกัสไปที่หัวข้อใหญ่อีกด้านของ Enterprise Network สมัยใหม่ที่ทุกองค์กรต้องมี:
**Security Architecture** เริ่มตั้งแต่ Firepower NGFW และ Cisco ISE (Identity Services Engine)
สำหรับ Network Access Control — รากฐานของการป้องกัน Campus Network ที่เราสร้างมาตลอด
41 Part ให้ปลอดภัยจาก Threat จริงในโลกปัจจุบัน

**ไปต่อ:** [Part 43 — Security Architecture: Firepower & ISE →](part-043-security-architecture-firepower-ise.md)
