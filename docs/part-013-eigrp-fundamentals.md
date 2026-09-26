# Part 13 — EIGRP Fundamentals
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 121–130 จาก 1000**

> ต่อจาก [Part 12 — OSPFv2 Advanced](part-012-ospfv2-advanced.md) ที่ปิดท้ายด้วยการ tune OSPF
> เต็มรูปแบบบน Lab Topology หลักของหลักสูตร (CORE-SW1/2, DIST-SW1–4, WAN-EDGE-1/2)

## ทำไมต้องเรียน EIGRP ทั้งที่ Lab หลักใช้ OSPF ไปแล้ว

**ข้อสำคัญที่ต้องเข้าใจก่อนเริ่ม Part นี้**: หลักสูตรนี้เลือกใช้ **OSPF เป็น IGP หลัก** ของ
Enterprise Lab Topology มาตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md) และ
[Part 12](part-012-ospfv2-advanced.md) และ **จะใช้ OSPF ต่อไปเป็น IGP หลักตลอดหลักสูตรที่เหลือ**
(รวมถึง Part ที่เกี่ยวกับ BGP, MPLS, SD-WAN ในระดับ CCNP/CCIE) — Part นี้**ไม่ได้เปลี่ยนการออกแบบ
เครือข่ายหลักเป็น EIGRP** แต่อย่างใด

เหตุผลที่ยังต้องมี Part สอน EIGRP โดยละเอียดมี 2 ข้อ:

1. **ข้อสอบ CCNA (200-301) และ CCNP ENCOR (350-401) ยังมีเนื้อหา EIGRP อยู่จริง** — แม้สัดส่วนจะ
   น้อยกว่า OSPF แต่ Cisco ยังคาดหวังให้ผู้สอบเข้าใจ DUAL, Metric Calculation, Successor/Feasible
   Successor, และ Named Mode Configuration
2. **EIGRP ยังใช้งานจริงในหลาย Enterprise Network** โดยเฉพาะองค์กรที่เป็น **Cisco-only shop มานาน**
   (all-Cisco environment ที่ไม่มีความจำเป็นต้องใช้ Protocol แบบเปิด/Multi-vendor) เพราะ EIGRP
   ตั้งค่าง่ายกว่า OSPF มาก (ไม่ต้องคิด Area Design), Converge เร็ว, และรองรับ Unequal-Cost Load
   Balancing ที่ OSPF ทำไม่ได้เลย — วิศวกร Enterprise จำนวนมากจึงยังเจอ EIGRP ในงานจริง
   โดยเฉพาะที่ไซต์เก่า (Legacy Site) ที่ได้มาจากการควบรวมบริษัท (M&A) ซึ่งยังรัน EIGRP อยู่

> **เพื่อไม่ให้กระทบ Lab Topology หลักที่ใช้ OSPF ไปแล้ว** Part นี้จะสร้าง **Standalone Demo
> Topology แยกต่างหาก** สำหรับสาธิต EIGRP โดยเฉพาะ — ใช้ชื่ออุปกรณ์ **DIST-SW3** และ **DIST-SW4**
> ซ้ำ (Reuse hostname เดิมจาก Part 10) เพื่อความคุ้นเคย แต่เปิด Port สำรอง (Spare Port) ที่ยังไม่ได้
> ใช้งานเพื่อจำลองว่าไซต์นี้เพิ่งถูกควบรวมเข้ามาใหม่และมี Router เดิมที่พูด EIGRP-only อยู่ 2 ตัว คือ
> **EIGRP-R1** และ **EIGRP-R2** — Config EIGRP ทั้งหมดใน Part นี้เกิดขึ้น**บน Port สำรองเหล่านี้เท่านั้น**
> ไม่แตะ SVI, Trunk, หรือ Routed Port ที่ใช้ OSPF จริงของ DIST-SW3/DIST-SW4 แม้แต่บรรทัดเดียว
> การ **Redistribution ระหว่าง EIGRP ↔ OSPF** (เพื่อรวม 2 Domain นี้เข้าด้วยกันจริง) จะเรียนแยกใน
> **Part ว่าด้วย Route Redistribution** ในระดับ CCNP ต่อไป — Part นี้โฟกัสที่กลไกของ EIGRP เองล้วนๆ

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 121 | EIGRP คืออะไร — Advanced Distance-Vector/Hybrid Protocol, DUAL, AS Number, ประเภท Packet |
| 122 | สูตร Composite Metric ของ EIGRP (Bandwidth, Delay, Reliability, Load, K1-K5) |
| 123 | Successor, Feasible Successor, FD, RD, Feasibility Condition — ตัวอย่างคำนวณจริง |
| 124 | คอนฟิก EIGRP Classic Mode พื้นฐาน |
| 125 | เงื่อนไขการเป็น EIGRP Neighbor และการ Troubleshoot Neighbor ไม่ขึ้น |
| 126 | คอนฟิก EIGRP Named Mode (Syntax แนะนำปัจจุบัน) |
| 127 | EIGRP Route Summarization ด้วย `ip summary-address eigrp` |
| 128 | EIGRP Load Balancing — Equal-Cost และ Unequal-Cost (`variance`) |
| 129 | คำสั่ง Verify และ Troubleshoot EIGRP |
| 130 | Lab เต็มรูปแบบ: Named Mode + MD5 Authentication + Summarization บน Standalone Topology |

---

## Step 121 — EIGRP คืออะไร: Advanced Distance-Vector/Hybrid, DUAL, AS Number, Packet Types

**EIGRP (Enhanced Interior Gateway Routing Protocol)** เป็น Routing Protocol ที่ Cisco พัฒนาขึ้นเอง
(Cisco-proprietary จนถึงปี 2013 หลังจากนั้น Cisco เปิดส่วนหนึ่งเป็น Open Standard ผ่าน **RFC 7868**
แต่ Implementation ที่ใช้งานจริงในโลกแทบทั้งหมดยังอยู่บน Cisco IOS/IOS-XE/NX-OS)

### EIGRP จัดอยู่ในประเภทไหน?

| คุณสมบัติ | Distance Vector แท้ (RIP) | Link-State แท้ (OSPF) | **EIGRP** |
|---|---|---|---|
| รู้จัก Topology เต็มรูปแบบหรือไม่ | ไม่รู้ (รู้แค่ next-hop + metric) | รู้เต็มรูปแบบ (LSDB) | **รู้เฉพาะ Topology ของตัวเองและ Neighbor** (ไม่เต็มเหมือน Link-State) |
| วิธี Advertise | Broadcast/Multicast Routing Table ทั้งก้อนเป็นระยะ | Flood LSA เมื่อมีการเปลี่ยนแปลง | **Update แบบ Incremental** (ส่งเฉพาะที่เปลี่ยน ไม่ส่งทั้งตารางเป็นระยะ) |
| Convergence | ช้า (ต้องรอ Timer, มี Count-to-Infinity) | เร็ว (SPF คำนวณจาก LSDB ทันที) | **เร็วมาก** (DUAL คำนวณ Backup Path ไว้ล่วงหน้า) |
| เรียกชื่อประเภทว่า | Distance Vector | Link-State | **Advanced Distance-Vector** หรือ **Hybrid Protocol** |

EIGRP ถูกเรียกว่า **Advanced Distance-Vector** (หรือ **Hybrid**) เพราะมีจุดเด่นของทั้งสองฝั่ง:
จาก Distance-Vector มันสืบทอดแนวคิดพื้นฐาน (เชื่อ Neighbor บอกอะไรมาก็เก็บไว้เป็น metric,
ไม่ต้อง Flood LSA ทั้ง Network) แต่ก็ยืมแนวคิดจาก Link-State มาด้วย (เก็บ Backup Path ไว้ล่วงหน้า,
Hello/Keepalive แบบ Link-State, ไม่ส่ง Full Routing Table เป็นระยะเหมือน RIP)

### DUAL (Diffusing Update Algorithm) — แนวคิดโดยสรุป

**DUAL** คือ Algorithm หัวใจของ EIGRP ที่ทำให้ EIGRP Converge เร็วกว่า Distance-Vector ทั่วไปมาก
หลักการคร่าวๆ คือ:

1. ทุก Router คำนวณ **Metric ที่ดีที่สุด (Successor)** ไปยังทุก Destination พร้อมกับหา
   **Backup Path ที่ปลอดภัย (Feasible Successor)** เก็บไว้ล่วงหน้าเสมอ (รายละเอียดเต็มใน Step 123)
2. ถ้า Successor ล่ม แต่มี Feasible Successor สำรองอยู่แล้ว → **สลับไปใช้ Backup ทันทีโดยไม่ต้อง
   คำนวณใหม่หรือถาม Neighbor เลย** (Local Computation) — นี่คือเหตุผลที่ EIGRP Converge เร็วมาก
3. ถ้าไม่มี Feasible Successor สำรองไว้เลย → Route นั้นจะเข้าสถานะ **Active** (กำลังหา Path ใหม่)
   แล้วส่ง **Query Packet** ถาม Neighbor ทุกตัวว่ามี Path อื่นไปยัง Destination นั้นไหม
   (เรียกว่า Diffusing Computation — เป็นที่มาของชื่อ DUAL) จนกว่าจะได้คำตอบ (Reply) ครบทุกตัว
   จึงจะสรุปผลและกลับสู่สถานะ **Passive** (Stable) อีกครั้ง

> **P (Passive) และ A (Active)** ที่เห็นใน `show ip eigrp topology` คือสถานะของ Route แต่ละเส้น
> P = เสถียร มี Successor ที่ใช้งานได้แน่นอน, A = กำลังอยู่ระหว่างกระบวนการ DUAL คำนวณหา Path ใหม่
> — Route ที่ค้างอยู่สถานะ Active นานเกินไปเรียกว่า **SIA (Stuck-In-Active)** ซึ่งเป็นปัญหา
> Troubleshooting ระดับ CCNP/CCIE ที่พบบ่อยใน Network ขนาดใหญ่

### Autonomous System (AS) Number — จุดที่มักสับสนกับ OSPF Process ID

ทั้ง EIGRP และ OSPF มีเลขที่ต้องตั้งตอน enable Routing Protocol แต่ **ความหมายต่างกันโดยสิ้นเชิง**:

| | OSPF Process ID (เช่น `router ospf 1`) | **EIGRP AS Number (เช่น `router eigrp 100`)** |
|---|---|---|
| Significant ระดับไหน | **Local เท่านั้น** — แต่ละ Router ตั้งเลขต่างกันได้ ไม่กระทบการเป็น Neighbor | **ต้องตรงกันทุก Router ในโดเมนเดียวกัน** — ถ้าไม่ตรง Neighbor จะไม่ขึ้นเด็ดขาด |
| เปรียบเทียบได้กับ | ป้ายชื่อ config ภายในเครื่องตัวเอง | รหัสยืนยันตัวตนของ "โดเมน EIGRP" เดียวกัน |
| ผลถ้าตั้งไม่ตรงกัน | ไม่มีผลอะไรเลย (OSPF ใช้ Area/Router-ID ตัดสิน) | **Router จะไม่เห็นกันเป็น Neighbor เลยแม้ Hello ไปถึง** |

> Part 11-12 สอน OSPF Process ID ไปแล้วว่าเป็น Local Significant — จำไว้ให้ดีว่า **EIGRP AS Number
> ตรงข้ามกันเลย** ต้องตรงกันทุกตัวในระบบเดียวกัน ข้อสอบ CCNA ชอบเอาความต่างนี้มาถามเปรียบเทียบ

### EIGRP Packet Types — 5 ประเภทที่ต้องจำ

EIGRP ส่ง Packet ผ่าน **RTP (Reliable Transport Protocol)** ซึ่งเป็นกลไกที่ EIGRP สร้างขึ้นเอง
(ไม่ใช่ TCP) เพื่อควบคุมว่า Packet ประเภทไหนต้อง Acknowledge (Reliable) และประเภทไหนไม่ต้อง
(Unreliable) — ใช้ **IP Protocol Number 88** และ Multicast Address **224.0.0.10**

| Packet Type | Reliable? | หน้าที่ | ส่งแบบ |
|---|---|---|---|
| **Hello** | ไม่ (Unreliable) | ค้นหาและรักษาสถานะ Neighbor, ส่งเป็นระยะตาม Hello Timer | Multicast 224.0.0.10 |
| **Update** | ใช่ (Reliable) | แจ้ง Route ใหม่/เปลี่ยนแปลง Metric | Multicast (ตอน Convergence) หรือ Unicast (ตอน Neighbor ใหม่เข้ามา) |
| **Query** | ใช่ (Reliable) | ถาม Neighbor หา Path สำรองตอน Route เข้าสถานะ Active (DUAL) | Multicast หรือ Unicast |
| **Reply** | ใช่ (Reliable) | ตอบกลับ Query ด้วยข้อมูล Path ที่มี (หรือไม่มี) | Unicast เท่านั้น |
| **Ack** | ไม่ (เป็นตัว Acknowledge เอง) | ยืนยันว่าได้รับ Reliable Packet (Update/Query/Reply) แล้ว | Unicast เท่านั้น |

> **ทำไม Hello ไม่ต้อง Reliable?** เพราะ Hello ถูกส่งซ้ำเป็นระยะอยู่แล้วตาม Timer (ปกติทุก 5 วินาที
> บน LAN/Point-to-Point ความเร็วสูง, ทุก 60 วินาทีบน WAN ความเร็วต่ำ) ถ้า Hello ฉบับหนึ่งหายไป
> ฉบับถัดไปก็มาแทนได้ ไม่จำเป็นต้อง Retransmit — **Hold Time** (ปกติ = 3× Hello Timer) คือเวลาที่
> รอ Hello โดยไม่ได้รับก่อนจะประกาศว่า Neighbor ตายไปแล้ว (คล้าย Dead Timer ของ OSPF ที่เรียนใน
> Part 11)

---

## Step 122 — สูตร Composite Metric ของ EIGRP (Bandwidth, Delay, Reliability, Load, K1-K5)

EIGRP ไม่ใช้ Hop Count เป็น Metric แบบ RIP และไม่ใช้ Cost ตัวเดียวแบบ OSPF แต่ใช้
**Composite Metric** ที่รวมค่าได้ถึง 4 ปัจจัยจาก Interface จริง โดยมี **K-value (K1-K5)** เป็น
ตัวคูณ/น้ำหนักที่กำหนดว่าปัจจัยไหนถูกนำมาคิดบ้าง

### สูตรเต็ม (Classic Metric)

```
Metric = 256 × [ (K1×BW) + ( (K2×BW) / (256−Load) ) + (K3×Delay) ] × [ K5 / (Reliability+K4) ]
```

| ตัวแปร | ที่มา | หน่วย |
|---|---|---|
| **BW (Bandwidth term)** | `10,000,000 / ค่า Bandwidth ที่ต่ำที่สุดตลอด Path (Kbps)` | ยิ่ง Bandwidth ต่ำ ยิ่งได้ค่า BW term สูง (แย่กว่า) |
| **Delay (Delay term)** | `ผลรวม Delay ของทุก Interface ตลอด Path (หน่วย tens of microsecond) หารด้วย 10` | ยิ่ง Delay สะสมมาก ยิ่งแย่ |
| **Reliability** | ค่าเฉลี่ยความน่าเชื่อถือของ Link (0-255, ยิ่งสูงยิ่งดี) | จาก `show interfaces` |
| **Load** | ค่า Load ปัจจุบันของ Interface ที่แย่ที่สุดตลอด Path (0-255, ยิ่งต่ำยิ่งดี) | จาก `show interfaces` |

### ค่า Default ของ K-value

| K1 (Bandwidth) | K2 (Load) | K3 (Delay) | K4 (Reliability) | K5 (Reliability exponent) |
|---|---|---|---|---|
| **1** | **0** | **1** | **0** | **0** |

> **กฎพิเศษของ K5**: ถ้า K5 = 0 (ค่า Default) ให้ตัดพจน์ `[K5/(Reliability+K4)]` ออกทั้งก้อน
> (ถือว่าเท่ากับ 1) — เพราะฉะนั้นด้วยค่า Default ทั้งหมด สูตรจะยุบลงเหลือแค่:
> ```
> Metric = 256 × (BW + Delay)
> ```
> นี่คือสูตรที่ใช้จริงในทุก Lab ของ Part นี้ (Reliability/Load ไม่ถูกนำมาคิดเลยตราบใดที่ K2=K4=K5=0
> ซึ่งเป็นค่า Default ที่แทบไม่มีใครไปแก้ในงานจริง — การแก้ K-value ทำได้แต่ **ไม่แนะนำ** เพราะ Router
> ทุกตัวใน AS เดียวกันต้องมี K-value ตรงกันทุกตัวเป๊ะ ไม่งั้น Neighbor จะไม่ขึ้น (ดู Step 125)

### ค่า Default ของ Bandwidth/Delay ต่อประเภท Interface (ค่าจาก `show interfaces` จริง)

| ประเภท Interface | BW (Kbit/sec) | DLY (usec) |
|---|---|---|
| Loopback | 8,000,000 | 5,000 |
| GigabitEthernet | 1,000,000 | 10 |
| FastEthernet | 100,000 | 100 |
| Ethernet (10 Mbps) | 10,000 | 1,000 |
| Serial (Default, เช่น 1.544 Mbps) | 1,544 | 20,000 |

### ตัวอย่างคำนวณ Metric แบบทั่วไป (Generic Worked Example)

สมมติ Path หนึ่งมี 2 hop: hop แรกเป็น **GigabitEthernet** (BW 1,000,000, DLY 10 usec), hop ที่สอง
เป็น **FastEthernet** (BW 100,000, DLY 100 usec) — คำนวณตามสูตรทีละขั้น:

```
ขั้น 1 — หา Bandwidth term (ใช้ค่า Bandwidth ที่ "ต่ำที่สุด" ตลอด Path)
   min(BW) = min(1,000,000 , 100,000) = 100,000 Kbps
   BW term = 10,000,000 / 100,000 = 100

ขั้น 2 — หา Delay term (บวก DLY ทุก hop รวมกัน แล้วหารด้วย 10)
   Sum(DLY) = 10 + 100 = 110 usec
   Delay term = 110 / 10 = 11

ขั้น 3 — แทนสูตร (ใช้ K-value default: Metric = 256 × (BW + Delay))
   Metric = 256 × (100 + 11) = 256 × 111 = 28,416
```

> **จุดสำคัญที่ต้องจำ**: Bandwidth term ใช้ **ค่าต่ำที่สุด (Bottleneck)** ตลอดเส้นทาง ไม่ใช่ค่าเฉลี่ย
> และไม่ใช่ผลรวม — ต่างจาก Delay term ที่เป็น**ผลรวม (Sum)** ของทุก Interface ตลอด Path นี่คือ
> เหตุผลที่ Link ความเร็วต่ำที่สุดเพียงเส้นเดียวใน Path ยาวๆ สามารถทำให้ Metric ทั้ง Path แย่ลงอย่าง
> มหาศาลได้ (เหมือนโซ่ที่ขาดง่ายที่สุดคือจุดอ่อนที่สุดของทั้งโซ่)

### คำสั่งปรับ Bandwidth/Delay ด้วยมือ (มีผลกับ Metric แต่ไม่เปลี่ยน Physical Speed จริง)

```
Router(config-if)# bandwidth 100000     ! ตั้งค่า Bandwidth ที่ EIGRP/OSPF ใช้คิด metric (Kbps)
Router(config-if)# delay 10             ! ตั้งค่า Delay หน่วย tens-of-microsecond (10 = 100 usec)
```

> **คำเตือนสำคัญ**: คำสั่ง `bandwidth` และ `delay` เป็นแค่ค่าที่บอก Routing Protocol ใช้คิด Metric
> เท่านั้น **ไม่ได้เปลี่ยน Physical Link Speed จริงแม้แต่นิดเดียว** ถ้าใส่ `bandwidth` ผิดจากความเร็วจริง
> ของสาย อาจทำให้ Routing Protocol เลือก Path ที่ช้าที่สุดเป็น Best Path ได้ — ใน Lab สาธิตของ
> Part นี้ (Step 130) เราจะใช้ประโยชน์จากกลไกนี้ **จำลอง Link ความเร็วต่ำบน Physical Interface
> GigabitEthernet จริง** เพื่อสาธิตสถานการณ์ Legacy WAN Link โดยไม่ต้องมี Hardware Serial จริง

---

## Step 123 — Successor, Feasible Successor, FD, RD, Feasibility Condition

นี่คือหัวใจของ DUAL ที่ทำให้ EIGRP มี Backup Path พร้อมสลับได้ทันทีโดยไม่ต้องคำนวณใหม่

### คำศัพท์ที่ต้องจำให้แม่น

| ศัพท์ | ความหมาย |
|---|---|
| **FD (Feasible Distance)** | Metric รวมที่**ดีที่สุด (ต่ำสุด)**ที่ Router ตัวเองคำนวณได้ไปยัง Destination หนึ่ง — คือ Metric ของ Successor |
| **RD (Reported Distance)** | Metric ที่ **Neighbor แต่ละตัว "รายงาน" มาให้เรา** ว่าตัวเขาเองไปถึง Destination นั้นด้วย Metric เท่าไหร่ (ไม่รวม Link ระหว่างเรากับเขา) |
| **Successor** | Neighbor ที่ให้ Path ที่มี Metric รวมต่ำที่สุด (ต่ำกว่าทุกตัวเลือกอื่น) — ถูกใส่ใน Routing Table จริง |
| **Feasible Successor (FS)** | Neighbor ตัวอื่นที่ **ผ่าน Feasibility Condition** — ถูกเก็บไว้ใน Topology Table เป็น Backup พร้อมสลับใช้ได้ทันที |
| **Feasibility Condition (FC)** | เงื่อนไขที่ใช้เช็คว่า Path หนึ่งจะเป็น FS ได้หรือไม่: **RD ของ Path นั้น ต้อง < FD (ของ Successor)** |

> **ทำไมต้องมี Feasibility Condition?** เพื่อ**ป้องกัน Routing Loop** — ถ้า RD ของ Neighbor
> ตัวหนึ่ง**มากกว่าหรือเท่ากับ** FD ของเรา แสดงว่า Neighbor ตัวนั้น**อาจจะ**เรียนรู้ Path ไปยัง
> Destination ผ่านตัวเราเองอยู่ (วนกลับมา) การเลือกมันเป็น Backup อาจทำให้เกิด Loop ได้
> — DUAL จึงยอมรับเป็น FS **เฉพาะ Neighbor ที่พิสูจน์ได้ทางคณิตศาสตร์ว่าไม่มีทางวนกลับมาทางเรา**
> เท่านั้น (RD < FD คือเงื่อนไขนั้น)

### นำสูตรจาก Step 122 มาคำนวณจริงกับ Standalone Topology ของ Part นี้

```
                          EIGRP-R1  (Legacy Site "BKK-OLD" #1)
                          Lo0: 9.9.9.1/32
                         ┌─────────┴─────────┐
                Link1    │                   │   Link2
        10.199.34.0/30   │                   │  10.199.34.4/30
     BW=100,000 DLY=100  │                   │  BW=1,000,000 DLY=10   (GigE เต็มสปีด)
      (จำลอง FastE)      │                   │
                   ┌──────┴──────┐     ┌──────┴──────┐
                   │  DIST-SW3   │     │  EIGRP-R2    │  Lo0: 9.9.9.2/32
                   │ (จาก Part10)│     │(Legacy Site  │  Lo1: 10.199.200.1/24  <- "Legacy Server LAN"
                   └──┬───────┬──┘     │  "BKK-OLD"#2)│      (ปลายทางที่จะคำนวณ metric ไปถึง)
             Link4     │       │Link5  └──┬────────┬──┘
        10.199.34.12/30│       └──────────┘        │
       BW=1,000,000    │      10.199.34.16/30      │ Link3
       DLY=10 (GigE)   │      BW=100,000 DLY=100    │ 10.199.34.8/30
       เต็มสปีด         │      (จำลอง FastE)         │ BW=1,544 DLY=20,000
                   ┌────┴────────┐                  │ (จำลอง Serial WAN)
                   │  DIST-SW4    │──────────────────┘
                   │ (จาก Part10) │
                   └──────────────┘
```

**คำนวณหา Metric ที่ DIST-SW3 ใช้ไปถึง `10.199.200.0/24` (Lo1 ของ EIGRP-R2) ผ่านทั้ง 3 เส้นทาง:**

**ขั้นที่ 1 — Metric ของ EIGRP-R2 เอง ไปยัง 10.199.200.0/24 (Directly Connected)**
```
BW term    = 10,000,000 / 8,000,000 (Loopback)  = 1   (ปัดเศษลง)
Delay term = 5,000 usec (Loopback) / 10          = 500
Metric_R2  = 256 × (1 + 500) = 256 × 501 = 128,256
```
ค่านี้คือ Metric ที่ EIGRP-R2 **ประกาศ (Advertise)** ไปให้ Neighbor ทุกตัว (DIST-SW3 ทาง Link5,
EIGRP-R1 ทาง Link2, DIST-SW4 ทาง Link3) — เมื่อ Neighbor ได้รับมา ค่านี้จะกลายเป็น **RD** ของพวกเขา

**ขั้นที่ 2 — Metric ของ EIGRP-R1 ไปยัง 10.199.200.0/24 (ผ่าน Link2 ไปหา EIGRP-R2)**
```
min(BW)    = min(1,000,000 [Link2], 8,000,000 [Lo1]) = 1,000,000
BW term    = 10,000,000 / 1,000,000 = 10
Sum(DLY)   = 10 [Link2] + 5,000 [Lo1] = 5,010
Delay term = 5,010 / 10 = 501
Metric_R1  = 256 × (10 + 501) = 256 × 511 = 130,816   <- นี่คือ RD ที่ EIGRP-R1 ประกาศให้ DIST-SW3
```

**ขั้นที่ 3 — Metric ของ DIST-SW4 ไปยัง 10.199.200.0/24 (ผ่าน Link3 ไปหา EIGRP-R2)**
```
min(BW)    = min(1,544 [Link3], 8,000,000 [Lo1]) = 1,544
BW term    = 10,000,000 / 1,544 = 6,476   (ปัดเศษลง)
Sum(DLY)   = 20,000 [Link3] + 5,000 [Lo1] = 25,000
Delay term = 25,000 / 10 = 2,500
Metric_SW4 = 256 × (6,476 + 2,500) = 256 × 8,976 = 2,297,856  <- RD ที่ DIST-SW4 ประกาศให้ DIST-SW3
```

**ขั้นที่ 4 — Metric รวม (Total Metric) ที่ DIST-SW3 คำนวณได้ในแต่ละเส้นทาง**

| Path | ผ่าน Neighbor | คำนวณ Bottleneck BW / Sum Delay | Total Metric | RD (จากขั้น 1-3) |
|---|---|---|---|---|
| **A** | EIGRP-R2 (ตรง, Link5) | min(100,000,8,000,000)=100,000 → BW=100 / Sum DLY=100+5,000=5,100 → Delay=510 | **256×(100+510)=156,160** | 128,256 |
| **B** | EIGRP-R1 (Link1) | min(100,000,1,000,000,8,000,000)=100,000 → BW=100 / Sum DLY=100+10+5,000=5,110 → Delay=511 | **256×(100+511)=156,416** | 130,816 |
| **C** | DIST-SW4 (Link4) | min(1,000,000,1,544,8,000,000)=1,544 → BW=6,476 / Sum DLY=10+20,000+5,000=25,010 → Delay=2,501 | **256×(6,476+2,501)=2,298,112** | 2,297,856 |

### สรุปผล — ใครคือ Successor, ใครคือ FS

```
FD (Feasible Distance) = Metric ต่ำที่สุด = 156,160 (Path A)  → Path A คือ Successor

ตรวจ Feasibility Condition ของ Path อื่น (RD < FD=156,160 หรือไม่?):
  Path B: RD = 130,816  < 156,160  →  ✅ ผ่าน  → Path B เป็น Feasible Successor (FS)
  Path C: RD = 2,297,856 < 156,160  →  ❌ ไม่ผ่าน (RD สูงกว่า FD มาก) → Path C ไม่ใช่ FS
```

| Path | Role | เก็บไว้ที่ไหน |
|---|---|---|
| A (ผ่าน EIGRP-R2 ตรง, metric 156,160) | **Successor** | Routing Table + Topology Table |
| B (ผ่าน EIGRP-R1, metric 156,416) | **Feasible Successor** | Topology Table เท่านั้น (Backup พร้อมสลับทันที) |
| C (ผ่าน DIST-SW4, metric 2,298,112) | Possible Successor เฉยๆ (ไม่ใช่ FS) | Topology Table เฉพาะกับ `show ... topology all` — ถ้า Path A ตายและไม่มี FS สำรอง ต้องเข้า DUAL Query ก่อนจะได้ใช้ Path นี้ |

> **ทำไม Path C ไม่ผ่าน FC ทั้งที่ยังเป็น Path ที่ใช้งานได้จริง?** เพราะ RD ของ Path C (2,297,856)
> สูงกว่า FD ปัจจุบันมาก — DUAL "ไม่ไว้ใจ" ตัวเลขนี้ในทางคณิตศาสตร์ว่าจะไม่วนกลับมาทางเรา (แม้ในทาง
> Physical จริงจะไม่มี Loop ก็ตาม) กรณีนี้แสดงให้เห็นข้อจำกัดของ FC: มันปลอดภัย 100% แต่บางครั้ง
> ปัดตก Path ที่ปลอดภัยจริงๆทิ้งไปด้วย (False Negative) — ถ้า Path A และ B ล่มพร้อมกันทั้งคู่
> Route นี้จะเข้าสถานะ **Active** และต้องส่ง Query ไปถาม Neighbor ก่อนจะยอมรับ Path C

---

## Step 124 — คอนฟิก EIGRP Classic Mode พื้นฐาน

**Classic Mode** คือ Syntax ดั้งเดิมของ EIGRP ที่ใช้กันมาตั้งแต่ IOS รุ่นแรกๆ ยังพบได้ในข้อสอบ CCNA
และอุปกรณ์เก่าจำนวนมาก แม้ Cisco จะแนะนำ **Named Mode** (Step 126) สำหรับ Deployment ใหม่แล้ว

### Syntax หลัก

```
Router(config)# router eigrp <AS-number>            ! AS ต้องตรงกันทุก Router (ดู Step 121)
Router(config-router)# network <network> [wildcard-mask]
Router(config-router)# no auto-summary               ! ปิด Auto-Summarization (อธิบายด้านล่าง)
Router(config-router)# eigrp router-id <a.b.c.d>     ! กำหนด Router-ID เอง (แนะนำเสมอ)
```

### ทำไมต้องมี `no auto-summary` เสมอ

Classic EIGRP มีพฤติกรรม Default ที่**อันตรายมาก**สำหรับ Network สมัยใหม่: มันจะ**สรุป Route
อัตโนมัติที่ Classful Boundary** (เช่น สรุปทุก Subnet ภายใน `10.0.0.0/8` ให้เหลือ `10.0.0.0/8`
เส้นเดียวเมื่อส่งออกไปนอกขอบเขต Major Network) ซึ่งในองค์กรที่ใช้ VLSM/Discontiguous Network
แบบ Lab หลักสูตรนี้ (ที่ใช้ `10.0.0.0/8` ทั้งองค์กรแต่แบ่งเป็น Subnet ย่อยจำนวนมาก) การเปิด
Auto-Summarization ทิ้งไว้จะทำให้ Subnet รายละเอียดหายไปและเกิดปัญหา Routing ทันที
**ต้องปิดด้วย `no auto-summary` เสมอในทุก Deployment สมัยใหม่**

### Wildcard Mask — ตัวแปลงกลับของ Subnet Mask ที่ `network` command ต้องใช้

| Prefix | Subnet Mask | Wildcard Mask (= 255.255.255.255 − Subnet Mask) |
|---|---|---|
| /8 | 255.0.0.0 | 0.255.255.255 |
| /16 | 255.255.0.0 | 0.0.255.255 |
| /24 | 255.255.255.0 | 0.0.0.255 |
| /30 | 255.255.255.252 | 0.0.0.3 |
| /32 | 255.255.255.255 | 0.0.0.0 |

### ตัวอย่างคอนฟิกเต็ม (Illustrative — DIST-SW3 กับ EIGRP-R1 ผ่าน Link1 เท่านั้น)

> ตัวอย่างนี้เพื่อสาธิต Syntax Classic Mode เท่านั้น ในการ Deploy จริงของ Part นี้ (Step 130) จะใช้
> **Named Mode** แทนตามที่ Cisco แนะนำปัจจุบัน

```
! ===== DIST-SW3 (Classic Mode, สาธิตเฉพาะ Link1) =====
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# description ** DEMO: EIGRP link to EIGRP-R1 (legacy site) **
DIST-SW3(config-if)# no switchport
DIST-SW3(config-if)# ip address 10.199.34.1 255.255.255.252
DIST-SW3(config-if)# bandwidth 100000
DIST-SW3(config-if)# delay 10
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit
!
DIST-SW3(config)# router eigrp 100
DIST-SW3(config-router)# eigrp router-id 1.1.1.13
DIST-SW3(config-router)# no auto-summary
DIST-SW3(config-router)# network 10.199.34.0 0.0.0.3       ! เจาะจงเฉพาะ subnet ของ Link1
```

```
! ===== EIGRP-R1 (Classic Mode) =====
EIGRP-R1(config)# interface GigabitEthernet0/0
EIGRP-R1(config-if)# description ** Link to DIST-SW3 **
EIGRP-R1(config-if)# ip address 10.199.34.2 255.255.255.252
EIGRP-R1(config-if)# bandwidth 100000
EIGRP-R1(config-if)# delay 10
EIGRP-R1(config-if)# no shutdown
EIGRP-R1(config-if)# exit
!
EIGRP-R1(config)# router eigrp 100
EIGRP-R1(config-router)# eigrp router-id 9.9.9.1
EIGRP-R1(config-router)# no auto-summary
EIGRP-R1(config-router)# network 10.199.34.0 0.0.0.3
```

### Verify

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
0   10.199.34.2     Gi1/0/23        13  00:00:24    1    100  0  3
```

---

## Step 125 — เงื่อนไขการเป็น EIGRP Neighbor และ Troubleshooting

EIGRP Neighbor จะขึ้น (Established) ได้ **ต้องผ่านเงื่อนไขทุกข้อพร้อมกัน**:

| เงื่อนไข | รายละเอียด | ผลถ้าไม่ตรงกัน |
|---|---|---|
| **1. AS Number ตรงกัน** | `router eigrp <AS>` ต้องเลขเดียวกันทุกตัว | Hello มาถึงแต่ไม่มี Log แจ้งเตือนใดๆ — Neighbor ไม่ขึ้นแบบเงียบๆ |
| **2. K-value ตรงกัน** | K1-K5 ต้องเหมือนกันทุกตัว (Default เหมือนกันหมดถ้าไม่ไปแก้) | ขึ้น Neighbor ได้ชั่วครู่แล้ว **ตัดทันที** พร้อม Log ชัดเจน (ดูด้านล่าง) |
| **3. Primary Subnet ตรงกัน** | IP Address หลัก (Primary) ของทั้งสองฝั่งต้องอยู่ Subnet เดียวกัน | Hello ถูก Drop เพราะไม่อยู่ Subnet เดียวกันจาก Neighbor's มุมมอง — **Secondary Address ไม่ถูกใช้ตั้ง Adjacency** |
| **4. Authentication ตรงกัน** (ถ้าเปิดใช้) | Key/Password/รูปแบบ Authentication ต้องตรงกัน (ดู Step 130) | Neighbor ไม่ขึ้น พร้อม Log แจ้ง Authentication failure |
| **5. Interface ต้องไม่เป็น Passive** | Interface ต้องไม่ถูกสั่ง `passive-interface` (จะไม่ส่ง Hello เลย) | ไม่มี Hello ส่งออกจาก Interface นั้นเลย — Neighbor ฝั่งนั้นไม่เห็นเราเลย |

> **หมายเหตุ CCNA/CCNP สำคัญ**: EIGRP **ไม่ตรวจ Hello/Hold Timer ต้องตรงกัน** (ต่างจาก OSPF ที่
> Hello/Dead Interval ต้องตรงกันทุกตัวจึงจะขึ้น Neighbor — เรียนไปแล้วใน Part 11) EIGRP ยอมให้
> สอง Neighbor ตั้ง Hello Timer ต่างกันได้ (ฝั่งหนึ่งส่งทุก 5 วิ อีกฝั่งส่งทุก 10 วิ ก็ยังขึ้น Neighbor ได้)
> เพราะ Hold Time แต่ละฝั่งคำนวณเป็นอิสระจากค่าที่ตัวเองส่งไปเอง

### สาเหตุ Log ที่เจอบ่อยที่สุด — K-value Mismatch

```
EIGRP-R1(config)# router eigrp 100
EIGRP-R1(config-router)# metric weights 0 1 0 2 0 0      ! เปลี่ยน K3 จาก 1 เป็น 2 (ผิดปกติ)

! บน DIST-SW3 จะเห็น Log ทันที:
DIST-SW3#
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.2 (GigabitEthernet1/0/23) is down: K-value mismatch
```

**วิธีตรวจ K-value ของทั้งสองฝั่ง:**
```
Router# show ip protocols | include K1|K2|K3|K4|K5
  K1=1, K2=0, K3=1, K4=0, K5=0
```

### สาเหตุ Log ที่เจอบ่อยรองลงมา — Primary Subnet ไม่ตรง

```
! ตัวอย่าง DIST-SW3 ตั้ง 10.199.34.1/30 แต่ EIGRP-R1 ตั้งผิดเป็น 10.199.35.2/30 (คนละ Subnet)
DIST-SW3# show ip eigrp neighbors
! ไม่มีบรรทัดใดปรากฏเลย — ไม่มี Neighbor ขึ้นแม้แต่ตัวเดียว

! ตรวจด้วย debug ชั่วคราว (ใช้ระมัดระวังตาม Step 129)
DIST-SW3# debug eigrp packets hello
EIGRP: Received HELLO on GigabitEthernet1/0/23 nbr 10.199.35.2
EIGRP: Discarding: not in Common subnet     <-- สาเหตุตรงนี้
```

### ลำดับ Troubleshooting มาตรฐานเมื่อ EIGRP Neighbor ไม่ขึ้น

```
1. show ip interface brief         → Interface up/up ก่อนหรือยัง (Layer 1-2 ต้องผ่านก่อน)
2. show running-config interface   → ตรวจ IP/Subnet Mask ของทั้งสองฝั่งตรงกันหรือไม่
3. show ip protocols                → ตรวจ AS Number, K-value, network statement ครอบคลุมหรือไม่
4. show ip eigrp interfaces         → Interface นั้นถูกเปิด EIGRP อยู่จริงหรือไม่ (ไม่ใช่ Passive)
5. ping ข้าม Link ก่อนเสมอ           → ถ้า ping ไม่ผ่าน ปัญหาไม่ใช่ EIGRP แต่เป็น Layer 1-3 พื้นฐาน
```

---

## Step 126 — คอนฟิก EIGRP Named Mode (Syntax แนะนำปัจจุบัน)

**Named Mode** คือ Syntax ที่ Cisco แนะนำให้ใช้ตั้งแต่ IOS 15.x เป็นต้นไป ข้อดีหลักคือ:
รวม IPv4/IPv6 ไว้ใน Process เดียวกันได้ (Address-Family), จัดกลุ่ม Feature ต่างๆ (Authentication,
Summarization, Timers) ไว้ใต้ `af-interface` ให้เป็นระเบียบกว่า, และรองรับ Feature ใหม่ๆที่ Classic
Mode ไม่มี (เช่น Wide Metric สำหรับ Interface ความเร็วสูงกว่า 10 Gbps)

### โครงสร้าง Syntax

```
Router(config)# router eigrp <process-name>                          ! ชื่อ (ไม่ใช่เลข AS)
Router(config-router-af)# address-family ipv4 unicast autonomous-system <AS>
Router(config-router-af)# network <network> [wildcard-mask]
Router(config-router-af)# af-interface <interface>
Router(config-router-af-interface)# ...   ! ตั้งค่าเฉพาะ Interface (bandwidth-percent, authentication)
Router(config-router-af)# exit-af-interface
Router(config-router-af)# topology base
Router(config-router-af-topology)# ...    ! ตั้งค่า variance, summarization ระดับ topology
```

### ตัวอย่างคอนฟิกเต็ม — เทียบเท่า Step 124 แต่ใช้ Named Mode (DIST-SW3 ↔ EIGRP-R1, Link1 เดิม)

```
! ===== DIST-SW3 (Named Mode) — ผลลัพธ์เหมือน Step 124 ทุกประการ =====
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# eigrp router-id 1.1.1.13
DIST-SW3(config-router-af)# network 10.199.34.0 0.0.0.3
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/23
DIST-SW3(config-router-af-interface)# exit-af-interface
DIST-SW3(config-router-af)# exit-address-family
```

```
! ===== EIGRP-R1 (Named Mode) =====
EIGRP-R1(config)# router eigrp EIGRP-DEMO
EIGRP-R1(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R1(config-router-af)# eigrp router-id 9.9.9.1
EIGRP-R1(config-router-af)# network 10.199.34.0 0.0.0.3
EIGRP-R1(config-router-af)# af-interface GigabitEthernet0/0
EIGRP-R1(config-router-af-interface)# exit-af-interface
EIGRP-R1(config-router-af)# exit-address-family
```

> **ข้อสังเกตสำคัญ**: `no auto-summary` **ไม่จำเป็นต้องพิมพ์เองใน Named Mode** เพราะ
> **Named Mode ปิด Auto-Summarization เป็น Default อยู่แล้ว** (ต่างจาก Classic Mode ที่เปิดไว้
> เป็น Default) — นี่คือความปลอดภัยที่มากขึ้นอีกข้อของ Named Mode ที่ Cisco ตั้งใจแก้ไขจาก Classic
> Mode ที่มักลืมปิดจนเกิดปัญหา

### เปรียบเทียบ Classic Mode vs Named Mode

| | Classic Mode | Named Mode |
|---|---|---|
| คำสั่งเริ่ม Process | `router eigrp <AS-number>` | `router eigrp <name>` (ชื่ออิสระ ไม่ใช่ AS) |
| ที่เก็บ AS Number จริง | ในคำสั่งเริ่ม Process เลย | ใน `address-family ipv4 unicast autonomous-system <AS>` |
| Auto-Summary Default | **เปิด** (ต้อง `no auto-summary` เอง) | **ปิด** อยู่แล้ว |
| รองรับ IPv4+IPv6 Process เดียว | ไม่ได้ (ต้องแยก `router eigrp` กับ `ipv6 router eigrp`) | ได้ (สร้าง address-family เพิ่มใน Process เดียวกัน) |
| Cisco แนะนำสำหรับ Deployment ใหม่ | ไม่แนะนำแล้ว (ยังใช้งานได้ ยังออกสอบ) | **แนะนำ** |
| ใช้ใน Lab เต็มของ Part นี้ (Step 130) | ไม่ใช้ | **ใช้** |

### Verify ว่าเป็น Process แบบ Named Mode จริง

```
DIST-SW3# show ip protocols
*** IP Routing is NSF aware ***

EIGRP-IPv4 VR(EIGRP-DEMO) Address-Family Protocol for AS(100)
  Metric weight K1=1, K2=0, K3=1, K4=0, K5=0, K6=0
  NSF-aware route hold timer is 240
  Router-ID: 1.1.1.13
  Topology : 0 (base)
    Active Timer: 3 min
    Distance: internal 90 external 170
    Maximum path: 4
    Maximum hopcount 100
    Maximum metric variance 1
  Automatic Summarization: disabled
  Maximum path: 4
  Routing for Networks:
    10.199.34.0/30
  Routing Information Sources:
    Gateway         Distance      Last Update
    10.199.34.2          90       00:03:41
```

---

## Step 127 — EIGRP Route Summarization ด้วย `ip summary-address eigrp`

EIGRP **ไม่ทำ Summarization อัตโนมัติที่ขอบเขต Area แบบ OSPF** (OSPF ทำที่ ABR ผ่าน
`area range` ตามที่เรียนใน Part 12) — EIGRP ต้อง **สั่ง Summarize เองทีละ Interface** ด้วยคำสั่ง
`ip summary-address eigrp` ซึ่งมีผลเฉพาะ **Traffic ที่ออกจาก Interface นั้น** เท่านั้น

### Syntax

```
Router(config-if)# ip summary-address eigrp <AS> <network> <mask>
```

### ตัวอย่าง — DIST-SW3 สรุป Subnet ทั้ง 5 เส้น (10.199.34.0/24) เป็น 1 เส้นก่อนส่งไปหา EIGRP-R1

จาก Diagram ใน Step 123 ทั้ง 5 Link ในระบบ (Link1, 2, 3, 4, 5) ล้วนอยู่ในขอบเขต `10.199.34.0/24`
เดียวกัน — DIST-SW3 สามารถส่งแค่ **1 Summary Route** แทนการส่ง 5 Subnet /30 แยกกัน:

```
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# ip summary-address eigrp 100 10.199.34.0 255.255.255.0
```

### ผลลัพธ์ก่อน/หลัง Summarization บน EIGRP-R1

```
! ก่อน Summarize — EIGRP-R1 เห็นทุก Subnet ย่อยแยกกัน
EIGRP-R1# show ip route eigrp
D    10.199.34.8/30 [90/2298112] via 10.199.34.6, 00:04:12, GigabitEthernet0/1
D    10.199.34.12/30 [90/158720] via 10.199.34.6, 00:04:12, GigabitEthernet0/1
D    10.199.34.16/30 [90/156416] via 10.199.34.1, 00:04:12, GigabitEthernet0/0
D    10.199.200.0/24 [90/130816] via 10.199.34.6, 00:04:12, GigabitEthernet0/1

! หลัง Summarize (สั่งที่ DIST-SW3 อธิบายด้านบน) — subnet ที่มาจากฝั่ง DIST-SW3 รวมเป็นเส้นเดียว
EIGRP-R1# show ip route eigrp
D    10.199.34.0/24 [90/156416] via 10.199.34.1, 00:00:08, GigabitEthernet0/0   <- รวมเป็น 1 เส้นแล้ว
D    10.199.200.0/24 [90/130816] via 10.199.34.6, 00:04:12, GigabitEthernet0/1
```

> **ผลข้างเคียงที่ต้องรู้**: ทันทีที่สั่ง `ip summary-address eigrp` DIST-SW3 จะสร้าง
> **Discard Route (Null0)** ขึ้นมาเองในตารางของตัวเอง (`D    10.199.34.0/24 is a summary, 00:00:05,
> Null0`) เพื่อป้องกัน Routing Loop ในกรณีที่มี Packet ปลายทางอยู่ใน Summary Range แต่ไม่ตรงกับ
> Subnet ย่อยใดๆที่มีอยู่จริง (ป้องกันไม่ให้ Packet วนกลับออก Interface เดิมที่ Summary ถูกส่งออก)
> — Administrative Distance ของ Discard Route นี้คือ **5** เสมอ (ค่า Fixed ที่ Cisco กำหนดไว้)

```
DIST-SW3# show ip route | include Null0
D        10.199.34.0/24 is a summary, 00:00:05, Null0
```

---

## Step 128 — EIGRP Load Balancing: Equal-Cost และ Unequal-Cost (`variance`)

### Equal-Cost Load Balancing — `maximum-paths`

เมื่อมี Path มากกว่า 1 เส้นที่มี **Metric เท่ากันเป๊ะ** EIGRP จะใส่ทุก Path ลง Routing Table
โดย Default พร้อมกัน (ECMP) จำนวนสูงสุดถูกกำหนดด้วย `maximum-paths` (Default = 4, ปรับได้
1-32 บน IOS-XE)

```
Router(config-router-af)# maximum-paths 6     ! เพิ่มจำนวน Equal-Cost Path สูงสุดเป็น 6 เส้น
```

> ในตัวอย่างของ Step 123 **ไม่มีคู่ Path ใดที่ Metric เท่ากันเป๊ะเลย** (156,160 / 156,416 /
> 2,298,112 ต่างกันทั้งหมด) ดังนั้น `maximum-paths` เพียงอย่างเดียว**ไม่ทำให้เกิด Load Balancing**
> ในสถานการณ์นี้ — ต้องใช้ **Unequal-Cost Load Balancing** ด้วย `variance` เท่านั้น

### Unequal-Cost Load Balancing — `variance`

`variance <multiplier>` อนุญาตให้ Path ที่มี Metric **มากกว่า FD ได้ไม่เกินกี่เท่า** ถูกใส่ลง
Routing Table ร่วมกับ Successor ได้ — **แต่ Path นั้นต้องเป็น Feasible Successor อยู่แล้วเท่านั้น**
(ต้องผ่าน Feasibility Condition ก่อน `variance` จะพิจารณาเลือกมันเลย)

```
เงื่อนไข: Metric ของ Path ≤ variance × FD   (และต้องเป็น FS อยู่แล้วด้วย)
```

### นำตัวเลขจาก Step 123 มาคำนวณจริง

```
FD (Successor, Path A) = 156,160
Path B (FS) metric      = 156,416

หา variance ต่ำสุดที่ทำให้ Path B ถูกเลือกร่วมด้วย:
   variance × 156,160  ≥  156,416
   variance  ≥  156,416 / 156,160  =  1.00164...

   variance ต้องเป็นจำนวนเต็ม (1-128) → ค่าต่ำสุดที่ใช้ได้จริงคือ variance = 2
```

```
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# variance 2
```

**ตรวจผลลัพธ์:**

```
DIST-SW3# show ip route 10.199.200.0
Routing entry for 10.199.200.0/24
  Known via "eigrp 100", distance 90, metric 156160, type internal
  Redistributing via eigrp 100
  Last update from 10.199.34.6 on GigabitEthernet1/0/23, 00:00:11 ago
  Routing Descriptor Blocks:
  * 10.199.34.18, from 10.199.34.18, 00:00:11 ago, via GigabitEthernet1/0/25
      Route metric is 156160, traffic share count is 3
  10.199.34.2, from 10.199.34.2, 00:00:11 ago, via GigabitEthernet1/0/23
      Route metric is 156416, traffic share count is 3
```

> `variance 2` ทำให้ Path A (Successor, metric 156,160) และ Path B (FS, metric 156,416)
> ถูกใส่ลง Routing Table **พร้อมกันทั้งคู่** — ส่วน Path C (metric 2,298,112 และไม่ผ่าน FC ด้วย)
> ยังคง**ไม่ถูกเลือกเด็ดขาด** ไม่ว่าจะตั้ง variance สูงเท่าไหร่ก็ตาม เพราะ variance
> **ทำงานร่วมกับ FC เท่านั้น ไม่ใช่ตัวแทนที่ FC**

### `traffic-share` — วิธีแบ่งสัดส่วน Traffic ระหว่าง Path

| คำสั่ง | พฤติกรรม |
|---|---|
| `traffic-share balanced` (Default) | แบ่ง Traffic **ตามสัดส่วนผกผันกับ Metric** (Path metric ต่ำกว่าได้รับ Traffic มากกว่า ไม่ใช่แบ่งเท่ากัน) |
| `traffic-share min across-interfaces` | ใช้เฉพาะ Path ที่มี Metric ต่ำสุด แบ่งเท่ากันข้าม Interface (ปิด Unequal-cost ที่ยังตั้งไว้ในทางปฏิบัติ) |

---

## Step 129 — คำสั่ง Verify และ Troubleshoot EIGRP

### `show ip eigrp neighbors` — ดูรายชื่อ Neighbor ที่ขึ้นอยู่จริง

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        12  00:15:33    2    100  0  8
0   10.199.34.2     Gi1/0/23        14  00:20:07    1    100  0  12
```

| Column | หมายถึง |
|---|---|
| H | Handle (เลขลำดับภายใน ไม่มีความหมายพิเศษ) |
| Hold | เวลาที่เหลือก่อน Hold Timer หมด (นับถอยหลังตลอด รีเซ็ตทุกครั้งที่ได้ Hello) |
| Uptime | Neighbor ขึ้นมานานแค่ไหนแล้ว |
| SRTT/RTO | Smooth Round-Trip Time / Retransmit Timeout ของ RTP (Step 121) |
| Q | Queue Count — ค้างมากกว่า 0 นานๆ = สัญญาณปัญหา Reliable Transport |

### `show ip eigrp topology` — ดู FD, Successor, Feasible Successor ตรงกับ Step 123 เป๊ะ

```
DIST-SW3# show ip eigrp topology
EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.13)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status

P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
```

- บรรทัดแรกหลัง prefix = **Successor** (metric ต่ำสุด, ตัวเลข `(156160/128256)` อ่านว่า
  `(Total Metric/RD)` — ตรงกับ Path A ใน Step 123 ทุกตัวเลข)
- บรรทัดถัดมา = **Feasible Successor** (metric สูงกว่าแต่ผ่าน FC) — ตรงกับ Path B
- `P` แปลว่า Route นี้อยู่สถานะ **Passive (เสถียร)** — ถ้าเห็น `A` แสดงว่า Route กำลังถูก DUAL
  คำนวณอยู่ (Active) เพราะ Successor เดิมล่มและไม่มี FS สำรอง

**ดู Path ที่ไม่ผ่าน FC ด้วย (Path C) ด้วย `all` keyword:**
```
DIST-SW3# show ip eigrp topology all
P 10.199.200.0/24, 1 successors, FD is 156160, serno 12
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
        via 10.199.34.14 (2298112/2297856), GigabitEthernet1/0/24     <- Path C ปรากฏเมื่อใช้ all
```

### `show ip route eigrp` — ดูเฉพาะ Route ที่มาจาก EIGRP ในตาราง Routing จริง

```
DIST-SW3# show ip route eigrp
      10.0.0.0/8 is variably subnetted, ...
D        10.199.200.0/24 [90/156160] via 10.199.34.18, 00:20:11, GigabitEthernet1/0/25
```

| Code | ความหมาย | Administrative Distance |
|---|---|---|
| `D` | EIGRP Internal Route (เรียนรู้จาก Router ใน AS เดียวกัน) | **90** |
| `D EX` | EIGRP External Route (เรียนรู้จากการ Redistribute เข้ามา) | **170** |

### `show ip protocols` — สรุปการตั้งค่า Process ทั้งหมด

ใช้เช็ค AS Number, K-value, Network Statement, Distance และจำนวน Neighbor ทั้งหมดในหน้าเดียว
(ตัวอย่างเต็มแสดงไว้แล้วใน Step 126)

### `debug eigrp packets` — ใช้อย่างระมัดระวังเท่านั้น

```
DIST-SW3# debug eigrp packets hello
EIGRP Packet debugging is on
    (HELLO)

DIST-SW3#
EIGRP: Sending HELLO on GigabitEthernet1/0/23
EIGRP: Received HELLO on GigabitEthernet1/0/25 nbr 10.199.34.18

DIST-SW3# undebug all      ! ปิด debug ทุกตัวทันที (ใช้คำสั่งนี้เสมอหลัง debug เสร็จ)
```

> **คำเตือนสำคัญ**: `debug eigrp packets` (โดยเฉพาะ `hello`) สร้าง Log จำนวนมากอย่างต่อเนื่อง
> เพราะ Hello ถูกส่งทุก 5 วินาที บน Production Router ที่มี Neighbor จำนวนมาก คำสั่งนี้อาจทำให้
> **CPU พุ่งสูงจนกระทบ Traffic จริง** — ควร Debug เฉพาะ Packet Type ที่สงสัยจริงๆ (เช่น
> `debug eigrp packets query reply` ตอนสงสัยปัญหา SIA) และ **ปิดด้วย `undebug all` ทันทีที่ได้
> ข้อมูลพอแล้ว** ไม่ควรเปิดทิ้งไว้นานบน Production

---

## Step 130 — Lab เต็มรูปแบบ: Named Mode + MD5 Authentication + Summarization

Lab สุดท้ายของ Part นี้รวมทุก Feature ที่เรียนมา (Step 121-129) เข้าด้วยกันบน **Standalone Demo
Topology เต็มรูปแบบทั้ง 4 อุปกรณ์** (DIST-SW3, DIST-SW4, EIGRP-R1, EIGRP-R2) — ใช้ **Named Mode**,
**MD5 Authentication**, และ **Summarization** ตามที่เกริ่นไว้ตอนต้น Part

### ทวน Topology และ Address Plan เต็ม (สรุปจาก Step 123 + เพิ่ม Loopback ครบ)

| Link | Subnet | ปลาย A | ปลาย B | Bandwidth/Delay ที่ตั้ง |
|---|---|---|---|---|
| Link1 | 10.199.34.0/30 | DIST-SW3 Gi1/0/23 (.1) | EIGRP-R1 Gi0/0 (.2) | `bandwidth 100000` `delay 10` (จำลอง FastE) |
| Link2 | 10.199.34.4/30 | EIGRP-R1 Gi0/1 (.5) | EIGRP-R2 Gi0/0 (.6) | ค่า Default ของ GigabitEthernet |
| Link3 | 10.199.34.8/30 | EIGRP-R2 Gi0/1 (.9) | DIST-SW4 Gi1/0/24 (.10) | `bandwidth 1544` `delay 2000` (จำลอง Serial) |
| Link4 | 10.199.34.12/30 | DIST-SW3 Gi1/0/24 (.13) | DIST-SW4 Gi1/0/23 (.14) | ค่า Default ของ GigabitEthernet |
| Link5 | 10.199.34.16/30 | DIST-SW3 Gi1/0/25 (.17) | EIGRP-R2 Gi0/2 (.18) | `bandwidth 100000` `delay 10` (จำลอง FastE) |
| Loopback0 | EIGRP-R1: 9.9.9.1/32, EIGRP-R2: 9.9.9.2/32 | — | — | Router-ID |
| Loopback1 | EIGRP-R2: 10.199.200.1/24 | — | — | "Legacy Server LAN" (ปลายทางที่ Step 123 คำนวณถึง) |

### 1) Key-chain สำหรับ MD5 Authentication (ต้องตั้งเหมือนกันทุกตัวที่จะเป็น Neighbor กัน)

```
DIST-SW3(config)# key chain EIGRP-KEY
DIST-SW3(config-keychain)# key 1
DIST-SW3(config-keychain-key)# key-string C1sco123!EIGRP
DIST-SW3(config-keychain-key)# exit
DIST-SW3(config-keychain)# exit
```
*(ทำ Key-chain เดียวกันนี้ซ้ำบน EIGRP-R1, EIGRP-R2, DIST-SW4 ทุกตัว — Key-string ต้องตรงกันเป๊ะ)*

### 2) คอนฟิกเต็มบน DIST-SW3

```
DIST-SW3(config)# interface GigabitEthernet1/0/23
DIST-SW3(config-if)# description ** DEMO-EIGRP: Link1 to EIGRP-R1 (legacy site) **
DIST-SW3(config-if)# no switchport
DIST-SW3(config-if)# ip address 10.199.34.1 255.255.255.252
DIST-SW3(config-if)# bandwidth 100000
DIST-SW3(config-if)# delay 10
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet1/0/24
DIST-SW3(config-if)# description ** DEMO-EIGRP: Link4 to DIST-SW4 **
DIST-SW3(config-if)# no switchport
DIST-SW3(config-if)# ip address 10.199.34.13 255.255.255.252
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet1/0/25
DIST-SW3(config-if)# description ** DEMO-EIGRP: Link5 direct to EIGRP-R2 **
DIST-SW3(config-if)# no switchport
DIST-SW3(config-if)# ip address 10.199.34.17 255.255.255.252
DIST-SW3(config-if)# bandwidth 100000
DIST-SW3(config-if)# delay 10
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

! ===== EIGRP Named Mode =====
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# eigrp router-id 1.1.1.13
DIST-SW3(config-router-af)# network 10.199.34.0 0.0.0.255
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# variance 2
DIST-SW3(config-router-af-topology)# exit-af-topology
!
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/23
DIST-SW3(config-router-af-interface)# authentication mode md5
DIST-SW3(config-router-af-interface)# authentication key-chain EIGRP-KEY
DIST-SW3(config-router-af-interface)# ip summary-address eigrp 100 10.199.34.0 255.255.255.0
DIST-SW3(config-router-af-interface)# exit-af-interface
!
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/24
DIST-SW3(config-router-af-interface)# authentication mode md5
DIST-SW3(config-router-af-interface)# authentication key-chain EIGRP-KEY
DIST-SW3(config-router-af-interface)# exit-af-interface
!
DIST-SW3(config-router-af)# af-interface GigabitEthernet1/0/25
DIST-SW3(config-router-af-interface)# authentication mode md5
DIST-SW3(config-router-af-interface)# authentication key-chain EIGRP-KEY
DIST-SW3(config-router-af-interface)# exit-af-interface
DIST-SW3(config-router-af)# exit-address-family
```

### 3) คอนฟิกเต็มบน DIST-SW4

```
DIST-SW4(config)# interface GigabitEthernet1/0/23
DIST-SW4(config-if)# description ** DEMO-EIGRP: Link4 to DIST-SW3 **
DIST-SW4(config-if)# no switchport
DIST-SW4(config-if)# ip address 10.199.34.14 255.255.255.252
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface GigabitEthernet1/0/24
DIST-SW4(config-if)# description ** DEMO-EIGRP: Link3 to EIGRP-R2 (legacy WAN sim) **
DIST-SW4(config-if)# no switchport
DIST-SW4(config-if)# ip address 10.199.34.10 255.255.255.252
DIST-SW4(config-if)# bandwidth 1544
DIST-SW4(config-if)# delay 2000
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# key chain EIGRP-KEY
DIST-SW4(config-keychain)# key 1
DIST-SW4(config-keychain-key)# key-string C1sco123!EIGRP
DIST-SW4(config-keychain-key)# exit
DIST-SW4(config-keychain)# exit

DIST-SW4(config)# router eigrp EIGRP-DEMO
DIST-SW4(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW4(config-router-af)# eigrp router-id 1.1.1.14
DIST-SW4(config-router-af)# network 10.199.34.0 0.0.0.255
DIST-SW4(config-router-af)# af-interface GigabitEthernet1/0/23
DIST-SW4(config-router-af-interface)# authentication mode md5
DIST-SW4(config-router-af-interface)# authentication key-chain EIGRP-KEY
DIST-SW4(config-router-af-interface)# exit-af-interface
DIST-SW4(config-router-af)# af-interface GigabitEthernet1/0/24
DIST-SW4(config-router-af-interface)# authentication mode md5
DIST-SW4(config-router-af-interface)# authentication key-chain EIGRP-KEY
DIST-SW4(config-router-af-interface)# exit-af-interface
DIST-SW4(config-router-af)# exit-address-family
```

### 4) คอนฟิกเต็มบน EIGRP-R1 และ EIGRP-R2 (Legacy Site Router ทั้งคู่)

```
! ===== EIGRP-R1 =====
EIGRP-R1(config)# interface Loopback0
EIGRP-R1(config-if)# ip address 9.9.9.1 255.255.255.255
EIGRP-R1(config-if)# exit
EIGRP-R1(config)# interface GigabitEthernet0/0
EIGRP-R1(config-if)# description ** Link1 to DIST-SW3 **
EIGRP-R1(config-if)# ip address 10.199.34.2 255.255.255.252
EIGRP-R1(config-if)# bandwidth 100000
EIGRP-R1(config-if)# delay 10
EIGRP-R1(config-if)# no shutdown
EIGRP-R1(config-if)# exit
EIGRP-R1(config)# interface GigabitEthernet0/1
EIGRP-R1(config-if)# description ** Link2 to EIGRP-R2 **
EIGRP-R1(config-if)# ip address 10.199.34.5 255.255.255.252
EIGRP-R1(config-if)# no shutdown
EIGRP-R1(config-if)# exit

EIGRP-R1(config)# key chain EIGRP-KEY
EIGRP-R1(config-keychain)# key 1
EIGRP-R1(config-keychain-key)# key-string C1sco123!EIGRP
EIGRP-R1(config-keychain-key)# exit
EIGRP-R1(config-keychain)# exit

EIGRP-R1(config)# router eigrp EIGRP-DEMO
EIGRP-R1(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R1(config-router-af)# eigrp router-id 9.9.9.1
EIGRP-R1(config-router-af)# network 9.9.9.1 0.0.0.0
EIGRP-R1(config-router-af)# network 10.199.34.0 0.0.0.255
EIGRP-R1(config-router-af)# af-interface GigabitEthernet0/0
EIGRP-R1(config-router-af-interface)# authentication mode md5
EIGRP-R1(config-router-af-interface)# authentication key-chain EIGRP-KEY
EIGRP-R1(config-router-af-interface)# exit-af-interface
EIGRP-R1(config-router-af)# af-interface GigabitEthernet0/1
EIGRP-R1(config-router-af-interface)# authentication mode md5
EIGRP-R1(config-router-af-interface)# authentication key-chain EIGRP-KEY
EIGRP-R1(config-router-af-interface)# exit-af-interface
EIGRP-R1(config-router-af)# exit-address-family
```

```
! ===== EIGRP-R2 =====
EIGRP-R2(config)# interface Loopback0
EIGRP-R2(config-if)# ip address 9.9.9.2 255.255.255.255
EIGRP-R2(config-if)# exit
EIGRP-R2(config)# interface Loopback1
EIGRP-R2(config-if)# description ** Legacy Server LAN (simulated) **
EIGRP-R2(config-if)# ip address 10.199.200.1 255.255.255.0
EIGRP-R2(config-if)# exit
EIGRP-R2(config)# interface GigabitEthernet0/0
EIGRP-R2(config-if)# description ** Link2 to EIGRP-R1 **
EIGRP-R2(config-if)# ip address 10.199.34.6 255.255.255.252
EIGRP-R2(config-if)# no shutdown
EIGRP-R2(config-if)# exit
EIGRP-R2(config)# interface GigabitEthernet0/1
EIGRP-R2(config-if)# description ** Link3 to DIST-SW4 (legacy WAN sim) **
EIGRP-R2(config-if)# ip address 10.199.34.9 255.255.255.252
EIGRP-R2(config-if)# bandwidth 1544
EIGRP-R2(config-if)# delay 2000
EIGRP-R2(config-if)# no shutdown
EIGRP-R2(config-if)# exit
EIGRP-R2(config)# interface GigabitEthernet0/2
EIGRP-R2(config-if)# description ** Link5 direct to DIST-SW3 **
EIGRP-R2(config-if)# ip address 10.199.34.18 255.255.255.252
EIGRP-R2(config-if)# bandwidth 100000
EIGRP-R2(config-if)# delay 10
EIGRP-R2(config-if)# no shutdown
EIGRP-R2(config-if)# exit

EIGRP-R2(config)# key chain EIGRP-KEY
EIGRP-R2(config-keychain)# key 1
EIGRP-R2(config-keychain-key)# key-string C1sco123!EIGRP
EIGRP-R2(config-keychain-key)# exit
EIGRP-R2(config-keychain)# exit

EIGRP-R2(config)# router eigrp EIGRP-DEMO
EIGRP-R2(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R2(config-router-af)# eigrp router-id 9.9.9.2
EIGRP-R2(config-router-af)# network 9.9.9.2 0.0.0.0
EIGRP-R2(config-router-af)# network 10.199.34.0 0.0.0.255
EIGRP-R2(config-router-af)# network 10.199.200.0 0.0.0.255
EIGRP-R2(config-router-af)# af-interface GigabitEthernet0/0
EIGRP-R2(config-router-af-interface)# authentication mode md5
EIGRP-R2(config-router-af-interface)# authentication key-chain EIGRP-KEY
EIGRP-R2(config-router-af-interface)# exit-af-interface
EIGRP-R2(config-router-af)# af-interface GigabitEthernet0/1
EIGRP-R2(config-router-af-interface)# authentication mode md5
EIGRP-R2(config-router-af-interface)# authentication key-chain EIGRP-KEY
EIGRP-R2(config-router-af-interface)# exit-af-interface
EIGRP-R2(config-router-af)# af-interface GigabitEthernet0/2
EIGRP-R2(config-router-af-interface)# authentication mode md5
EIGRP-R2(config-router-af-interface)# authentication key-chain EIGRP-KEY
EIGRP-R2(config-router-af-interface)# exit-af-interface
EIGRP-R2(config-router-af)# exit-address-family
```

### 5) Verification เต็มรูปแบบหลัง Deploy

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        13  00:04:02    2    100  0  6
2   10.199.34.14    Gi1/0/24        14  00:04:02    1    100  0  4
0   10.199.34.2     Gi1/0/23        12  00:04:02    1    100  0  5
```

```
DIST-SW3# show ip eigrp topology
EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.13)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status

P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
P 10.199.34.0/24, 1 successors, FD is 128256
        via Summary (128256/0), Null0
```

```
DIST-SW3# show ip route eigrp
D        10.199.200.0/24 [90/156160] via 10.199.34.18, 00:04:22, GigabitEthernet1/0/25
                          [90/156416] via 10.199.34.2, 00:04:22, GigabitEthernet1/0/23
```

```
EIGRP-R1# show ip route eigrp
D        10.199.34.0/24 [90/156416] via 10.199.34.1, 00:04:30, GigabitEthernet0/0     <- Summary จาก DIST-SW3
D        10.199.200.0/24 [90/130816] via 10.199.34.6, 00:04:30, GigabitEthernet0/1
```

```
! ทดสอบ Authentication mismatch (จงใจตั้ง key-string ผิดที่ EIGRP-R1 ชั่วคราวเพื่อสาธิต)
DIST-SW3#
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.2 (GigabitEthernet1/0/23) is down:
Auth failure
```

> **Lab นี้พิสูจน์ทุกแนวคิดของ Part 13 ครบวงจร**: DUAL เลือก Successor/FS ตรงกับที่คำนวณด้วยมือ
> ใน Step 123 เป๊ะ (156,160 / 156,416), `variance 2` ทำให้เกิด Unequal-Cost Load Balancing จริง,
> Summarization ย่อ 3 Subnet ย่อยเหลือ 1 เส้นสำเร็จ, และ MD5 Authentication ป้องกัน Router
> ที่ไม่รู้ Key ไม่ให้เข้าร่วม EIGRP Domain ได้จริง — ทั้งหมดนี้เกิดขึ้น**บน Port สำรองของ DIST-SW3/
> DIST-SW4 เท่านั้น ไม่กระทบ OSPF Design ของ Lab หลักแม้แต่บรรทัดเดียว**

---

## แบบฝึกหัดทวนความเข้าใจ Part 13

1. เพราะเหตุใด EIGRP AS Number จึงต้อง "ตรงกันทุก Router" ในขณะที่ OSPF Process ID เป็นแค่
   Local Significant?
2. จากสูตร `Metric = 256 × (BW + Delay)` — Bandwidth term ใช้ค่า "ต่ำที่สุด" หรือ "ผลรวม" ของ
   ทุก Interface ตลอด Path? ส่วน Delay term ใช้ค่าแบบไหน?
3. Feasibility Condition คืออะไร และเพราะเหตุใด Path ที่ RD สูงกว่า FD จึงไม่ถูกยอมรับเป็น
   Feasible Successor แม้จะเป็น Path ที่ใช้งานได้จริงในทางกายภาพก็ตาม?
4. ถ้าต้องการทำ Unequal-Cost Load Balancing ระหว่าง Path ที่มี FD=100,000 กับอีก Path ที่มี
   Metric=180,000 ต้องตั้ง `variance` อย่างน้อยเท่าไหร่ (เป็นจำนวนเต็ม) และ Path นั้นต้องผ่าน
   เงื่อนไขอะไรก่อนด้วย?
5. คำสั่งใดใช้ตรวจดู K-value ปัจจุบันของ Router เพื่อ Troubleshoot ปัญหา Neighbor ไม่ขึ้น
   จาก K-value Mismatch?

**เฉลย:**
1. เพราะ EIGRP ใช้ AS Number เป็นส่วนหนึ่งของ Hello Packet เพื่อยืนยันว่าทั้งสอง Router อยู่
   "โดเมน EIGRP" เดียวกันจริง ถ้าไม่ตรงกัน Router จะไม่ยอมรับ Hello นั้นเป็น Neighbor เลย
   ต่างจาก OSPF ที่ใช้ Area Number/Router-ID/Network Type เป็นตัวตัดสิน Adjacency แทน
2. Bandwidth term ใช้ค่า **ต่ำที่สุด (Bottleneck)** ตลอด Path (ไม่ใช่ผลรวม) ส่วน Delay term
   ใช้ **ผลรวม (Sum)** ของ Delay ทุก Interface ตลอด Path แล้วหารด้วย 10
3. Feasibility Condition คือ RD ของ Path นั้นต้อง < FD ปัจจุบัน — ถ้า RD ≥ FD แปลว่าในทาง
   คณิตศาสตร์ DUAL ไม่สามารถพิสูจน์ได้ว่า Path นั้นจะไม่วนกลับมาผ่านตัวเราเอง (Routing Loop)
   จึงถูกปัดตกเป็นแค่ Possible Successor แม้จะยังใช้งานได้จริงทางกายภาพก็ตาม (ป้องกัน Loop
   เป็นสำคัญกว่า)
4. variance ≥ 180,000/100,000 = 1.8 → ค่าจำนวนเต็มต่ำสุดที่ใช้ได้คือ **variance 2**
   และ Path นั้น**ต้องเป็น Feasible Successor อยู่แล้วก่อน** (ผ่าน Feasibility Condition
   RD < FD) — variance ไม่ทำให้ Path ที่ไม่ผ่าน FC ถูกใช้งานได้เลย
5. `show ip protocols` (ดูบรรทัด K1-K5) หรือ `show ip eigrp topology` ประกอบกับ Log
   `%DUAL-5-NBRCHANGE: ... K-value mismatch` ที่ปรากฏทันทีที่ Neighbor หลุด

---

## สรุป Part 13

Part นี้ปูพื้นฐาน EIGRP ครบทั้ง DUAL Algorithm, Composite Metric (K1-K5), Successor/Feasible
Successor/FD/RD พร้อมตัวอย่างคำนวณจริงที่สอดคล้องกันตลอดทั้ง Part, Classic Mode และ Named Mode
Configuration, Neighbor Requirement/Troubleshooting, Summarization, Unequal-Cost Load
Balancing ด้วย `variance`, และปิดท้ายด้วย Lab เต็มรูปแบบพร้อม MD5 Authentication — ทั้งหมดนี้
เกิดขึ้นบน **Standalone Demo Topology** ที่แยกจาก Lab หลักโดยสมบูรณ์ **Lab Topology หลักของ
หลักสูตรยังคงใช้ OSPF เป็น IGP ต่อไปตามที่ออกแบบไว้ตั้งแต่ Part 11-12**

✅ **พร้อมสำหรับ Part 14**: กลับเข้าสู่ Lab Topology หลัก เพื่อเรียนเรื่อง **Access Control List
(ACL)** — เครื่องมือกรอง Traffic พื้นฐานที่สุดของ Cisco IOS ทั้ง Standard ACL, Extended ACL,
Named ACL, และการนำไปประยุกต์ใช้ควบคุมการเข้าถึงระหว่าง VLAN ต่างๆใน Topology จริงของเรา

**ไปต่อ:** [Part 14 — Access Control Lists →](part-014-access-control-lists.md)
