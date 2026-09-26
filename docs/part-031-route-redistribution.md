# Part 31 — Route Redistribution & Route-maps
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 301–310 จาก 1000**

> ย้อนกลับ: [Part 30 — BGP Advanced](part-030-bgp-advanced.md)

> ถึงจุดนี้ Enterprise Lab ของหลักสูตรกำลังรัน **3 Routing Domain พร้อมกัน**: **OSPF** เป็น IGP
> หลักของ Campus (CORE-SW1/2, DIST-SW1–4 — ตั้งค่าเต็มรูปแบบใน [Part 11](part-011-ospfv2-fundamentals.md)
> และ [Part 12](part-012-ospfv2-advanced.md)), **EIGRP AS 100** ที่ไซต์เก่า Legacy Site "BKK-OLD"
> ซึ่งจำลองไว้บน Port สำรองของ DIST-SW3/DIST-SW4 เชื่อมกับ EIGRP-R1/EIGRP-R2
> ([Part 13](part-013-eigrp-fundamentals.md) และ Part 28), และ **BGP** ที่ WAN Edge
> (WAN-EDGE-1/WAN-EDGE-2 ทำ eBGP กับ ISP-RTR — Part 29-30) ปัญหาที่ตามมาคือ **Router แต่ละตัวใน
> แต่ละ Domain ไม่รู้จักเส้นทางของอีก Domain หนึ่งเลย** เพราะ OSPF, EIGRP, และ BGP ต่างก็มี
> Metric, Algorithm, และ Database เป็นของตัวเอง ไม่แลกเปลี่ยนข้อมูลกันโดยอัตโนมัติ — Part นี้
> จะสอน **Route Redistribution** เทคนิคการ "แปลภาษา" ระหว่าง Routing Protocol ที่ต่างชนิดกัน
> พร้อม **Route-map** เครื่องมือควบคุมที่ขาดไม่ได้เมื่อ Redistribute ในระบบจริงระดับ Enterprise/ISP
> ซึ่งเป็นหนึ่งในทักษะที่**ออกข้อสอบ CCNP ENCOR (350-401) หนักที่สุด** และเป็นสาเหตุอันดับ 1
> ของ Outage ระดับ Production ถ้าทำผิดวิธี

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 301 | ทำไมต้อง Redistribute — การรวม Routing Domain และอันตรายของมัน |
| 302 | Syntax พื้นฐานของ Redistribution และ Seed Metric ที่ต้องตั้งเสมอ |
| 303 | Redistribute EIGRP เข้า OSPF — Worked Config บน DIST-SW3 |
| 304 | Redistribute OSPF เข้า EIGRP — Worked Config พร้อม Seed Metric 5 ค่า |
| 305 | Route-map ฉบับเต็ม — Match/Set Clause ทั้งหมดที่ต้องรู้ |
| 306 | Selective Redistribution ด้วย Route-map + Prefix-list |
| 307 | ปัญหา Routing Loop จากการ Redistribute สองทิศทาง และการแก้ด้วย Route Tag |
| 308 | Redistribute BGP เข้า/ออก IGP — กับดักอันตรายที่สุดของหัวข้อนี้ |
| 309 | คำสั่ง Verify/Troubleshoot Redistribution ทั้งหมด |
| 310 | Lab เต็มรูปแบบ: Mutual Redistribution OSPF↔EIGRP พร้อม Tag และ Controlled Redistribution BGP→OSPF ที่ WAN Edge |

---

## Step 301 — ทำไมต้อง Redistribute: การรวม Routing Domain และอันตรายของมัน

### ทบทวน Topology ทั้ง 3 Domain ของ Lab ตอนนี้

```
                              ┌───────────────┐
                              │    ISP-RTR      │   AS 65000 (eBGP)
                              └───┬───────┬───┘
                       203.0.113.0/30   203.0.113.4/30
                          (.1)  │       │  (.5)
                                │(.2)  (.6)│
                       ┌────────┴┐     ┌┴────────┐
   ══ BGP Domain ══    │WAN-EDGE-1│═════│WAN-EDGE-2│   AS 65001 (Enterprise)
                       └────┬────┘10.10.254.8/30└────┬────┘
                  10.10.254.0/30(.1)          (.10) 10.10.254.4/30
                       (.2) │                          │ (.6)
                       ┌────┴────┐                ┌────┴────┐
                       │ CORE-SW1 │════Po1═════════│ CORE-SW2 │
                       └──┬───┬──┘  (Area 0)       └──┬───┬──┘
   ══ OSPF Domain ══  Area1│   │Area1            Area2│   │Area2
                       ┌───┴┐ ┌┴───┐              ┌───┴┐ ┌┴───┐
                       │DIST│ │DIST│              │DIST│ │DIST│
                       │-SW1│ │-SW2│              │-SW3│ │-SW4│ ★★ ASBR ★★
                       └────┘ └────┘              └──┬─┘ └──┬─┘
                                             Link1(Gi1/0/23)│  │(Gi1/0/22)Link3
   ══ EIGRP Domain ══                     10.199.34.0/30│  │10.199.34.8/30
                                            AS 100        │  │
                                              ┌──────────┴┐┌┴──────────┐
                                              │ EIGRP-R1   ││ EIGRP-R2   │
                                              │Lo0 9.9.9.1 ││Lo0 9.9.9.2 │
                                              └────────────┘│Lo1 10.199.200.0/24
                                                             └────────────┘
                                                          (Legacy Server LAN)
```

### ทำไมต้อง Redistribute

จาก Diagram ด้านบน สังเกตว่า **DIST-SW3/DIST-SW4** พูดทั้ง OSPF (Area 2 — เชื่อม Campus)
**และ** EIGRP AS 100 (เชื่อม Legacy Site) พร้อมกันบน Router ตัวเดียว ส่วน **WAN-EDGE-1/2**
พูดทั้ง OSPF (เชื่อม Campus) **และ** BGP (เชื่อม ISP) พร้อมกัน — Router ที่มี Interface อยู่ใน
มากกว่า 1 Routing Protocol แบบนี้เรียกว่า **ASBR (Autonomous System Boundary Router)**
(นิยามเดียวกับที่แนะนำไว้ใน Part 12 Step 112: ASBR = อยู่ระหว่าง OSPF กับ Routing Domain อื่น
ที่ไม่ใช่ OSPF)

ถ้าไม่ทำอะไรเลย ผลคือ:

| ปัญหา | รายละเอียด |
|---|---|
| PC1 (VLAN10, OSPF) คุยกับ Legacy Server (10.199.200.0/24, EIGRP) ไม่ได้ | CORE-SW1/DIST-SW1 ไม่มี Route ไปยัง `10.199.200.0/24` เลยในตาราง เพราะ OSPF ไม่รู้จัก EIGRP Topology |
| Legacy Site ออก Internet ไม่ได้ | EIGRP-R1/EIGRP-R2 ไม่มี Default Route หรือ Route ไปยัง ISP เพราะ EIGRP ไม่รู้จัก BGP Table |
| แต่ละ Domain ต้องพึ่ง Static Route จำนวนมาก | ย้อนกลับไปทำแบบ Part 9 (Static Routing) ทั้งหมด ซึ่งไม่ Scale และต้อง Maintain มือทุกจุดที่ Topology เปลี่ยน |

**Route Redistribution** คือกระบวนการที่ ASBR **"แปล" Route ที่เรียนรู้จาก Protocol หนึ่ง
แล้วฉีด (Inject) เข้าไปเป็น Route ของอีก Protocol หนึ่ง** ทำให้ Router ฝั่งตรงข้ามเห็น Route
นั้นราวกับเป็น Native Route ของ Protocol ตัวเอง (แต่ถูก Mark ไว้ว่าเป็น "External" เสมอ เช่น
`O E2` ใน OSPF หรือ `D EX` ใน EIGRP)

### อันตรายของ Redistribution ถ้าทำไม่ระวัง

Redistribution เป็นหัวข้อที่ **สร้าง Outage จริงในโลก Production มากที่สุดหัวข้อหนึ่ง** เพราะ:

1. **Routing Loop** — ถ้า Redistribute แบบสองทิศทาง (Mutual) ที่มากกว่า 1 จุดพร้อมกัน (เช่น
   DIST-SW3 **และ** DIST-SW4 ต่าง Redistribute OSPF↔EIGRP เองแยกกัน) Route ที่ถูก Redistribute
   ออกไปแล้วอาจถูก Redistribute **ย้อนกลับเข้ามาใหม่** ที่จุดที่สอง กลายเป็น Loop ของข้อมูล
   Routing เอง (ไม่ใช่ Loop ของ Packet ตรงๆ แต่ทำให้ Routing Table ผิดเพี้ยนจนเกิด Loop ของ
   Packet ตามมา) — รายละเอียดเต็มใน Step 307
2. **Suboptimal Path (เส้นทางไม่เหมาะสม)** — Metric ของแต่ละ Protocol **วัดกันคนละหน่วย**
   (OSPF ใช้ Cost จาก Bandwidth, EIGRP ใช้ Composite Metric จาก BW/Delay/Reliability/Load,
   BGP ใช้ AS-Path/Attribute) เมื่อแปลงข้ามกัน ตัวเลข Metric เดิมจะ**หายไปหมด** กลายเป็น
   "Seed Metric" ค่าเดียวคงที่ (ดู Step 302) ทำให้ Router ปลายทางไม่สามารถเปรียบเทียบ "เส้นทาง
   ไหนดีกว่าจริง" ได้อีกต่อไป อาจเลือกเส้นทางอ้อมโดยไม่รู้ตัว
3. **Administrative Distance ชนกัน** — ถ้า Router เรียนรู้ Prefix เดียวกันจากทั้ง 2 Protocol
   (เช่นผ่าน DIST-SW3 เป็น `O E2` และผ่าน DIST-SW4 เป็น `D EX`) IOS จะเลือกตาม **AD** ก่อนเสมอ
   (`O`=110 ชนะ `D EX`=170) ซึ่งอาจไม่ใช่เส้นทางที่ Engineer ตั้งใจ ถ้าไม่ได้ออกแบบ AD ให้ตรงกับ
   Physical Topology จริง
4. **Full Table Leak** — อันตรายที่สุดคือ Redistribute BGP (ที่อาจมี Prefix นับแสนถ้าต่อ Internet
   จริง) เข้า IGP โดยไม่กรอง ทำให้ Router ทั้ง Campus ล่มจาก CPU/Memory หมด (ดู Step 308)

> **หลักการทองคำของ Redistribution**: "**Redistribute ให้น้อยที่สุดเท่าที่จำเป็น และควบคุมทิศทาง
> ด้วย Route-map/Tag เสมอ**" — Part นี้ทั้ง Part คือการสอนวิธีทำตามหลักการนี้ให้ถูกต้อง

---

## Step 302 — Syntax พื้นฐานของ Redistribution และ Seed Metric ที่ต้องตั้งเสมอ

### Syntax ทั่วไป (ใต้ Routing Protocol ปลายทางที่จะ "รับ" Route เข้ามา)

```
Router(config-router)# redistribute <protocol> [process-id/AS] [subnets] [metric <value>] [metric-type <1|2>] [route-map <name>] [tag <value>]
```

| พารามิเตอร์ | ใช้กับ | ความหมาย |
|---|---|---|
| `<protocol>` | ทุก Protocol | Protocol ต้นทางที่จะดึง Route เข้ามา เช่น `ospf`, `eigrp`, `bgp`, `static`, `connected` |
| `[process-id/AS]` | ospf ต้องระบุ process-id, eigrp ต้องระบุ AS | ระบุว่าดึงจาก Process/AS ไหน (ถ้ามีมากกว่า 1 Process บน Router เดียวกัน) |
| `subnets` | **OSPF เท่านั้น (สำคัญมาก)** | ถ้าไม่ใส่ OSPF จะ Redistribute เฉพาะ **Classful Major Network** เท่านั้น (ตัด Subnetทิ้งหมด) — ลืมใส่คำนี้คือกับดักข้อสอบ/Production อันดับ 1 |
| `metric <value>` | ทุก Protocol | **Seed Metric** — ค่า Metric เริ่มต้นที่ Route ที่ถูก Redistribute จะได้รับในระบบใหม่ (ดูรายละเอียดด้านล่าง) |
| `metric-type <1\|2>` | OSPF เท่านั้น | E1 (คิด Cost สะสมตาม Path ภายใน OSPF ด้วย) หรือ E2 (Fix Cost คงที่ตลอดทาง — **Default**) |
| `route-map <name>` | ทุก Protocol | กรองว่า Route ไหนจะถูก Redistribute จริง และ/หรือแก้ไขค่า Metric/Tag เฉพาะบาง Route (Step 305-306) |
| `tag <value>` | ทุก Protocol | ใส่ค่า Tag (32-bit) ติดไปกับ Route — ใช้ป้องกัน Loop ใน Step 307 |

### ทำไม "Seed Metric" ถึงเป็นเรื่อง **บังคับ** ไม่ใช่ทางเลือก

แต่ละ Protocol นิยาม "Metric" คนละความหมายกันโดยสิ้นเชิง Route ที่ข้ามจาก Protocol หนึ่งไปอีก
Protocol หนึ่งจึง**ไม่มีความหมายอะไรเลยถ้าไม่ตั้งค่า Metric ใหม่ให้มันโดยเฉพาะ**:

| Protocol ปลายทาง | Metric ที่ต้องการ | ถ้าไม่ระบุ Seed Metric จะเกิดอะไร |
|---|---|---|
| **OSPF** | ตัวเลขเดียว (Cost) | ใช้ **Default = 20** สำหรับทุก Protocol ยกเว้น BGP ที่ Default = 1 (ยังทำงานได้ แต่ควรตั้งเองเสมอเพื่อความชัดเจน) |
| **EIGRP** | ชุด 5 ค่า: Bandwidth, Delay, Reliability, Load, MTU | **ไม่มี Default ใดๆ ทั้งสิ้น** — ถ้าไม่ระบุ `metric` หรือ `default-metric` **Route จะไม่ถูกติดตั้งเข้า EIGRP เลย แบบเงียบๆ ไม่มี Error แจ้งเตือน** |
| **BGP** | Metric กลายเป็น MED (Multi-Exit Discriminator) | ใช้ค่า Metric ของ IGP ต้นทางเป็นค่าเริ่ม (ถ้าไม่ตั้ง Route-map ควบคุม) |

> **กับดักข้อสอบ/Production ที่พบบ่อยที่สุดของหัวข้อนี้**: วิศวกรพิมพ์
> `redistribute ospf 1` ใต้ `router eigrp` โดยไม่ใส่ `metric` แล้วงงว่า "ทำไม Route OSPF
> ไม่โผล่ใน EIGRP เลย ทั้งที่คำสั่งไม่ Error" — คำตอบคือ EIGRP **ปฏิเสธ Route แบบเงียบๆ**
> เพราะไม่มี Seed Metric ให้ใช้คำนวณ Composite Metric ได้ ต้องระบุ `metric` เสมอ หรือใช้
> `default-metric <bw> <delay> <rel> <load> <mtu>` ใต้ `router eigrp` แทนก็ได้ (มีผลเหมือนกัน
> กับทุก `redistribute` statement ที่ไม่ได้ระบุ `metric` เจาะจงของตัวเอง)

### `default-metric` — ทางลัดตั้ง Seed Metric ให้ทุก Redistribute Statement พร้อมกัน

```
! แทนที่จะพิมพ์ metric ซ้ำทุกบรรทัด redistribute ตั้งค่ากลางไว้ทีเดียว
Router(config-router)# default-metric <value>              ! สำหรับ OSPF/RIP (ค่าเดียว)
Router(config-router)# default-metric <bw> <delay> <rel> <load> <mtu>   ! สำหรับ EIGRP (5 ค่า)
```

`default-metric` มีผลกับทุก `redistribute` Statement ที่**ไม่ได้ระบุ `metric` ของตัวเองไว้
เจาะจง** — ถ้า Redistribute Statement ใดมี `metric` ต่อท้ายอยู่แล้ว ค่านั้นจะ Override
`default-metric` เสมอ (Local ชนะ Global)

---

## Step 303 — Redistribute EIGRP เข้า OSPF: Worked Config บน DIST-SW3

### เป้าหมาย

ให้ Campus ทั้งหมด (OSPF Area 0/1/2) เห็น Subnet ฝั่ง Legacy Site (EIGRP AS 100) โดย
DIST-SW3 ทำหน้าที่ ASBR ที่จุดนี้ (ยังไม่แตะ DIST-SW4 — จะเพิ่มเป็น Mutual 2 จุดใน Step 307)

### คอนฟิกบน DIST-SW3 (เพิ่มเข้าไปจาก OSPF Area 2 ที่มีอยู่แล้วจาก Part 12)

```
DIST-SW3(config)# router ospf 1
DIST-SW3(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 2 tag 90
```

| ส่วนของคำสั่ง | เหตุผล |
|---|---|
| `eigrp 100` | ดึง Route จาก EIGRP AS 100 (Legacy Demo Topology จาก Part 13) |
| `subnets` | **จำเป็นเสมอ** — ถ้าไม่ใส่ Subnet ทั้งหมดของ EIGRP (`10.199.34.0/30` ทุกเส้น, `10.199.200.0/24`) จะไม่ถูก Redistribute เลย เพราะไม่ใช่ Classful Major Network เดี่ยวๆ |
| `metric 20` | Seed Metric แบบ Cost คงที่ — ใช้ค่าเดียวกับ Default ของ OSPF เพื่อความสอดคล้องกับ Route External อื่น |
| `metric-type 2` | E2 (Default) — Cost ที่ Router ใน OSPF เห็นจะเป็น 20 คงที่ตลอดทาง ไม่บวกเพิ่มตาม Hop ภายใน OSPF |
| `tag 90` | ติด Tag ไว้ล่วงหน้าเพื่อใช้กับ Step 307 (90 = จำง่ายว่าตรงกับ AD ของ EIGRP Internal — บอกว่า "Route นี้มาจากฝั่ง EIGRP") |

### ผลลัพธ์ที่ CORE-SW1 (ผ่าน Area 0 จาก CORE-SW2 ที่เป็น ABR)

```
CORE-SW1# show ip route ospf
      10.0.0.0/8 is variably subnetted, 12 subnets, 3 masks
O E2    10.199.34.0/30 [110/20] via 10.255.30.1, 00:02:14, TenGigabitEthernet1/0/1
O E2    10.199.34.4/30 [110/20] via 10.255.30.1, 00:02:14, TenGigabitEthernet1/0/1
O E2    10.199.34.8/30 [110/20] via 10.255.30.1, 00:02:14, TenGigabitEthernet1/0/1
O E2    10.199.34.12/30 [110/20] via 10.255.30.1, 00:02:14, TenGigabitEthernet1/0/1
O E2    10.199.34.16/30 [110/20] via 10.255.30.1, 00:02:14, TenGigabitEthernet1/0/1
O E2    10.199.200.0/24 [110/20] via 10.255.30.1, 00:02:14, TenGigabitEthernet1/0/1
```

สังเกต Code `O E2` (OSPF External Type 2) และ `[110/20]` — **AD ยังคงเป็น 110 เท่ากับ OSPF
ปกติทุกประการ** (Cisco ไม่มี AD แยกสำหรับ External Route) ส่วน Metric `20` คือ Seed Metric
คงที่ที่ตั้งไว้ใน `redistribute` Statement — **ไม่ใช่ Composite Metric ของ EIGRP เดิมอีกต่อไป**
(ค่า Metric เดิม เช่น 130,816 ของ EIGRP หายไปหมดตามหลักการ Step 301 ข้อ 2)

> **ทำไมต้องผ่าน `TenGigabitEthernet1/0/1` (ไป DIST-SW3) แทนที่จะเห็น Path ตรงไปยัง Legacy
> Site เลย**: เพราะ CORE-SW1 อยู่ Area 1/0 เท่านั้น ไม่มี Interface ต่อ EIGRP โดยตรง — Route
> ต้องวิ่งผ่าน OSPF Backbone ปกติไปหา DIST-SW3 (ASBR) ก่อน แล้ว DIST-SW3 จะ Forward ต่อเข้า
> EIGRP Domain ให้เองอีกที (Recursive ผ่าน CEF ตามปกติ ไม่ต่างจาก Static Route ใน Part 9)

---

## Step 304 — Redistribute OSPF เข้า EIGRP: Worked Config พร้อม Seed Metric 5 ค่า

### เป้าหมาย

ให้ EIGRP-R1/EIGRP-R2 (Legacy Site) เห็น Subnet ทั้งหมดของ Campus (VLAN 10/20/30/40/99
และ Loopback ทุกตัว) ผ่าน OSPF ที่ Redistribute เข้ามาที่ DIST-SW3

### ทำไมต้องตั้ง Seed Metric เป็น "ชุด 5 ค่า" สำหรับ EIGRP

EIGRP คำนวณ Composite Metric จาก Bandwidth/Delay/Reliability/Load (ทบทวนสูตรเต็มได้ที่
Part 13 Step 122) เมื่อ Route ถูก Redistribute เข้ามา EIGRP **ต้องการค่าตั้งต้นครบทั้ง 5**
เพื่อเอาไปคำนวณ Metric ต่อตามปกติเหมือน Route EIGRP แท้ๆ:

```
Router(config-router-af)# redistribute ospf 1 metric <bandwidth> <delay> <reliability> <load> <mtu>
```

| ตำแหน่ง | ความหมาย | ค่าที่ใช้ในตัวอย่างนี้ | เหตุผลที่เลือกค่านี้ |
|---|---|---|---|
| 1. Bandwidth | หน่วย Kbit/s | `1000000` | เทียบเท่า GigE — ใกล้เคียง Backbone Link จริงของ Campus (Po1, TenGig) |
| 2. Delay | หน่วย **10 ไมโครวินาที** | `100` | เทียบเท่า Delay 1000 usec = ใกล้เคียง GigE ทั่วไป (Part 13 ใช้ Delay 10 หน่วยเดียวกันนี้กับ GigE) |
| 3. Reliability | 0–255 (255=100%) | `255` | สมมติ Link เสถียร 100% (ค่ามาตรฐานที่ใช้กันทั่วไปเมื่อไม่มีข้อมูลจริง) |
| 4. Load | 0–255 (255=Load เต็ม) | `1` | สมมติ Load ต่ำมาก (ค่ามาตรฐาน) |
| 5. MTU | หน่วย Byte | `1500` | Ethernet มาตรฐาน |

### คอนฟิกบน DIST-SW3 (Named Mode ตามที่ใช้ใน Part 13 Step 126)

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# redistribute ospf 1 metric 1000000 100 255 1 1500 tag 110
DIST-SW3(config-router-af-topology)# exit-af-topology
```

(ถ้าใช้ Classic Mode แทน คำสั่งจะย้ายมาอยู่ตรงๆ ใต้ `router eigrp 100` โดยไม่มี
`address-family`/`topology base` แต่ Syntax ของ `redistribute` เหมือนกันทุกประการ)

`tag 110` ใส่ไว้ล่วงหน้าเช่นกัน (110 = จำง่ายว่าตรงกับ AD ของ OSPF — บอกว่า "Route นี้มาจาก
ฝั่ง OSPF") เพื่อใช้ป้องกัน Loop ใน Step 307

### ผลลัพธ์ที่ EIGRP-R1

```
EIGRP-R1# show ip route eigrp
      10.0.0.0/8 is variably subnetted, 16 subnets, 3 masks
D EX    10.10.10.0/24 [170/28160256] via 10.199.34.1, 00:01:05, GigabitEthernet0/0
D EX    10.10.20.0/24 [170/28160256] via 10.199.34.1, 00:01:05, GigabitEthernet0/0
D EX    10.10.30.0/24 [170/28160256] via 10.199.34.1, 00:01:05, GigabitEthernet0/0
D EX    10.10.40.0/24 [170/28160256] via 10.199.34.1, 00:01:05, GigabitEthernet0/0
D EX    10.10.99.0/24 [170/28160256] via 10.199.34.1, 00:01:05, GigabitEthernet0/0
D EX    1.1.1.1/32 [170/28160256] via 10.199.34.1, 00:01:05, GigabitEthernet0/0
```

สังเกต Code `D EX` (EIGRP External) และ **AD = 170** (ตรงกับตาราง AD ใน Part 9 Step 85
พอดี — EIGRP External แพงกว่า EIGRP Internal ที่ AD=90 อย่างมีนัยสำคัญ เพื่อให้ Route ภายใน
EIGRP แท้ๆ ชนะ Route ที่ถูก Redistribute เข้ามาเสมอถ้ามีทั้งคู่) ค่า Metric `28160256` คำนวณ
จาก Seed Metric 5 ค่าที่ตั้งไว้ผ่านสูตร Composite Metric ปกติของ EIGRP

---

## Step 305 — Route-map ฉบับเต็ม: Match/Set Clause ทั้งหมดที่ต้องรู้

### Syntax โครงสร้างพื้นฐาน

```
Router(config)# route-map <name> {permit | deny} <sequence-number>
Router(config-route-map)#  match <condition>
Router(config-route-map)#  set <action>
```

**กฎการทำงานของ Route-map** (สำคัญพอๆ กับตัว Syntax เอง):

1. ประเมินทีละ `sequence-number` **จากน้อยไปมาก** เจอ Sequence แรกที่ `match` ผ่านทุกเงื่อนไข
   (AND ระหว่าง `match` ต่างชนิด, OR ระหว่าง `match` ชนิดเดียวกันหลายค่า) จะหยุดทันที — ไม่ไป
   ประเมิน Sequence ถัดไปอีก
2. ถ้า Sequence นั้นเป็น `permit` และ Match ผ่าน → Route ผ่านและถูก Redistribute (พร้อม `set`
   แก้ไขค่าตามที่กำหนด)
3. ถ้า Sequence นั้นเป็น `deny` และ Match ผ่าน → Route ถูกบล็อกทันที ไม่ถูก Redistribute
4. ถ้าไม่มี `match` เลยใน Sequence นั้น → ถือว่า Match ทุก Route เสมอ (Match "any")
5. **ถ้าไล่จนหมดทุก Sequence แล้วไม่มีอันไหน Match เลย → Implicit Deny ท้ายสุดเสมอ**
   (เหมือน ACL) — ต้องมี Sequence สุดท้ายที่เป็น "Permit All" เองถ้าต้องการให้ Route ที่เหลือ
   ผ่านไปด้วย

### ตาราง `match` Clause ที่ใช้บ่อยที่สุด

| Match Clause | ใช้จับอะไร | ตัวอย่าง |
|---|---|---|
| `match ip address <acl\|prefix-list>` | Prefix ปลายทางตรงกับ ACL หรือ Prefix-list ที่ระบุ | `match ip address prefix-list LEGACY-SERVER` |
| `match tag <value> [value2 ...]` | Route ที่มี Tag ตรงกับค่าที่ระบุ (ใช้กับ Step 307) | `match tag 110` |
| `match interface <name>` | Route ที่เรียนรู้ผ่าน Interface นี้ | `match interface GigabitEthernet1/0/23` |
| `match metric <value>` | Route ที่มี Metric ตรงตามค่าที่ระบุ (ก่อนแปลง) | `match metric 156160` |
| `match route-type {internal\|external}` | ประเภท Route ของ OSPF/EIGRP (Internal vs External) | `match route-type internal` |
| `match ip next-hop <acl\|prefix-list>` | Route ที่มี Next-hop ตรงตามที่ระบุ | `match ip next-hop prefix-list NH-LIST` |

### ตาราง `set` Clause ที่ใช้บ่อยที่สุด

| Set Clause | ใช้ทำอะไร | ตัวอย่าง |
|---|---|---|
| `set metric <value>` | เปลี่ยน Seed Metric เฉพาะ Route ที่ Match (Override `redistribute metric` ตัวหลัก) | `set metric 30` |
| `set metric-type {type-1\|type-2}` | เปลี่ยนประเภท OSPF External เฉพาะ Route นี้ | `set metric-type type-1` |
| `set tag <value>` | ติด/เปลี่ยน Tag ให้ Route ที่ Match | `set tag 90` |
| `set local-preference <value>` | เปลี่ยน Local Preference (BGP เท่านั้น — ดู Part 30) | `set local-preference 150` |
| `set community <value>` | ติด BGP Community (BGP เท่านั้น) | `set community 65001:100` |
| `set ip next-hop <ip>` | เปลี่ยน Next-hop ของ Route (ใช้ทำ PBR ใน Part 32) | `set ip next-hop 10.10.254.2` |
| `set weight <value>` | เปลี่ยน BGP Weight (Local ต่อ Router เดียว) | `set weight 200` |

### ตัวอย่าง Multi-clause เต็มรูปแบบ (สาธิต Match+Set หลายเงื่อนไขในไฟล์เดียว)

```
! Sequence 10: Subnet Legacy Server LAN เท่านั้น -> ตั้ง Metric ต่ำเป็นพิเศษ + Tag เฉพาะ
route-map DEMO-RMAP permit 10
 match ip address prefix-list LEGACY-SERVER
 set metric 15
 set metric-type type-1
 set tag 999
!
! Sequence 20: Route ที่มี Tag 90 อยู่แล้ว (เคย Redistribute มาจาก EIGRP รอบก่อน) -> ปฏิเสธทิ้ง
route-map DEMO-RMAP deny 20
 match tag 90
!
! Sequence 30: ที่เหลือทั้งหมด -> ผ่านตามปกติด้วย Metric Default
route-map DEMO-RMAP permit 30
 set tag 110
```

จาก Route-map นี้: Route ที่ Match `LEGACY-SERVER` Prefix-list จะได้ Metric/Tag พิเศษ (หยุดที่
Sequence 10 ทันที ไม่ตกไป Sequence 20/30 อีก), Route ที่มี Tag 90 อยู่แล้วจะถูกบล็อกที่
Sequence 20, ส่วน Route อื่นที่เหลือทั้งหมดผ่านที่ Sequence 30 พร้อมติด Tag 110 ใหม่

---

## Step 306 — Selective Redistribution ด้วย Route-map + Prefix-list

### ปัญหา: Redistribute ทุก Subnet เข้า OSPF อาจไม่จำเป็น/ไม่ปลอดภัย

จาก Step 303 เรา Redistribute **ทุก Subnet** ของ EIGRP เข้า OSPF รวมถึง Transit Link ภายใน
(`10.199.34.0/30`, `10.199.34.4/30` ฯลฯ) ซึ่งเป็นแค่ Link ระหว่าง Router ภายใน EIGRP Domain
เอง **ไม่มีประโยชน์อะไรที่ Campus ต้องรู้จัก** (เสีย Memory/CPU โดยไม่จำเป็น และเพิ่มพื้นผิว
สำหรับปัญหาที่ Step 307 จะอธิบาย) — สิ่งที่ Campus ต้องการจริงๆ มีแค่ `10.199.200.0/24`
(Legacy Server LAN) เท่านั้น

### คอนฟิก Prefix-list + Route-map บน DIST-SW3

```
! กำหนด Prefix ที่ต้องการอนุญาตเท่านั้น (Legacy Server LAN)
DIST-SW3(config)# ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24
!
! Route-map: อนุญาตเฉพาะ Prefix ที่ match, ที่เหลือปฏิเสธ (Implicit Deny ท้ายสุดอยู่แล้ว)
DIST-SW3(config)# route-map EIGRP-TO-OSPF-SELECTIVE permit 10
DIST-SW3(config-route-map)# match ip address prefix-list LEGACY-SERVER
DIST-SW3(config-route-map)# set tag 90
DIST-SW3(config-route-map)# exit
!
! ผูก Route-map เข้ากับคำสั่ง redistribute แทนแบบเดิมใน Step 303
DIST-SW3(config)# router ospf 1
DIST-SW3(config-router)# no redistribute eigrp 100 subnets metric 20 metric-type 2 tag 90
DIST-SW3(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF-SELECTIVE
```

### Verify — CORE-SW1 เห็นแค่ Subnet ที่อนุญาตเท่านั้น

```
CORE-SW1# show ip route ospf
      10.0.0.0/8 is variably subnetted, 7 subnets, 3 masks
O E2    10.199.200.0/24 [110/20] via 10.255.30.1, 00:00:12, TenGigabitEthernet1/0/1
```

Transit Link ทั้ง 5 เส้น (`10.199.34.0/30` ถึง `10.199.34.16/30`) **หายไปจากตาราง OSPF ของ
Campus ทั้งหมด** เพราะไม่ Match Prefix-list `LEGACY-SERVER` และไม่มี Sequence ไหนใน
Route-map อนุญาตไว้ (ตก Implicit Deny) — Campus ยังคงคุยกับ Legacy Server ได้ปกติทุกประการ
เพราะ `10.199.200.0/24` ยังถูก Redistribute อยู่ แค่ลด "Noise" ที่ไม่จำเป็นออกไปเท่านั้น

> **หลักปฏิบัติ Production**: ทุกครั้งที่ Redistribute ควรถามตัวเองเสมอว่า **"ฝั่งตรงข้ามจำเป็น
> ต้องรู้จัก Subnet นี้จริงหรือไม่"** — Transit Link ภายใน Domain หนึ่งแทบไม่มีเหตุผลต้องให้
> Domain อื่นรู้จักเลย ยกเว้นกรณีทำ Traceroute/Troubleshooting ข้าม Domain เป็นพิเศษ

---

## Step 307 — ปัญหา Routing Loop จากการ Redistribute สองทิศทาง และการแก้ด้วย Route Tag

### สร้างปัญหาให้เห็นจริง: Mutual Redistribution ที่ 2 จุดพร้อมกัน

จาก Diagram ใน Step 301 สังเกตว่า **ทั้ง DIST-SW3 และ DIST-SW4 ต่างมี Interface อยู่ทั้งใน
OSPF Area 2 และ EIGRP AS 100 พร้อมกัน** (DIST-SW3 ต่อ EIGRP-R1 ผ่าน Link1, DIST-SW4 ต่อ
EIGRP-R2 ผ่าน Link3 — ดู Topology เต็มที่ Part 13 Step 123) นั่นหมายความว่า **ทั้งคู่เป็น
ASBR ที่ทำ Redistribution แยกกัน 2 จุด** — ถ้าทั้งสองตัวต่าง Redistribute Mutual (ทั้ง
OSPF→EIGRP และ EIGRP→OSPF) โดยไม่มีการป้องกัน จะเกิด Loop ของข้อมูล Routing ดังนี้:

```
ขั้นที่ 1: OSPF Advertise 10.10.10.0/24 (VLAN10 ของ Campus) เข้ามาถึง DIST-SW3 (O, Intra-Area)

ขั้นที่ 2: DIST-SW3 Redistribute เข้า EIGRP -> กลายเป็น D EX บน EIGRP-R1/EIGRP-R2

ขั้นที่ 3: EIGRP-R2 Advertise D EX นี้ต่อไปให้ DIST-SW4 (ผ่าน Link3 ปกติของ EIGRP)

ขั้นที่ 4: DIST-SW4 (ซึ่งก็ Redistribute EIGRP->OSPF อยู่ด้วย) เห็น D EX ของ 10.10.10.0/24
          และ Redistribute มันกลับเข้า OSPF Area 2 อีกครั้ง กลายเป็น O E2 ใบใหม่!

ขั้นที่ 5: O E2 (จาก DIST-SW4) วิ่งกลับเข้า Area 2 แข่งกับ O (Intra-Area) ตัวจริงจาก CORE-SW2
          -- ถ้า AD/Metric คำนวณผิดจังหวะ Router บางตัวอาจเลือกเส้นทางอ้อมผ่าน EIGRP Domain
          ไปกลับ (DIST-SW3 -> EIGRP -> DIST-SW4 -> OSPF) แทนที่จะไปตรงผ่าน Backbone Area 0
          ทำให้เกิด Suboptimal Routing หรือในกรณีเลวร้ายที่สุดคือ Routing Loop เต็มรูปแบบเมื่อ
          Topology เปลี่ยนแปลงระหว่าง Convergence
```

```
                    ┌─────────────┐
        ┌──────────►│  CORE-SW2   │◄──────────┐
        │           │  (Area 0/2) │            │
        │           └──────┬──────┘            │
   Route กลับมา            │ Area 2      Route ต้นฉบับ
   ทาง "ประตูผิด"          │             วิ่งมาทางนี้
        │            ┌─────┴─────┐              │
        │      ┌─────┤ DIST-SW3  │              │
        │      │     │(OSPF+EIGRP)              │
        │      │     └─────┬─────┘              │
        │      │           │ Redistribute       │
        │      │           │ OSPF -> EIGRP      │
        │      ▼           ▼                    │
        │  ┌────────┐ ┌────────┐                │
        │  │EIGRP-R1│═│EIGRP-R2│                │
        │  └────────┘ └───┬────┘                │
        │                 │                      │
        │           ┌─────┴─────┐                │
        └───────────┤ DIST-SW4  ├────────────────┘
     Redistribute   │(OSPF+EIGRP)  กลับเข้า Area 2
     EIGRP -> OSPF  └───────────┘  (จุดเกิด Loop!)
```

### วิธีแก้: Route Tag + Route-map ปฏิเสธ Route ที่เคยผ่านมาแล้ว

หลักการคือ **"ติดป้ายบอกที่มา (Tag) ให้ทุก Route ที่ถูก Redistribute แล้วปฏิเสธไม่ให้
Redistribute Route ที่มี Tag บอกว่ามันมาจากฝั่งเราอยู่แล้ว กลับเข้าไปอีกรอบ"** — ใช้ Tag ที่
ตั้งไว้ล่วงหน้าแล้วตั้งแต่ Step 303/304 (Tag 90 = มาจาก EIGRP, Tag 110 = มาจาก OSPF — จำง่าย
เพราะตรงกับค่า AD ของแต่ละฝั่งพอดี):

```
! ===== DIST-SW3 =====
! ทิศทาง OSPF -> EIGRP: ถ้า Route มี Tag 90 อยู่แล้ว (แปลว่ามันมาจาก EIGRP ตั้งแต่ต้น
! ผ่านการ Redistribute ของ DIST-SW4 มาก่อนแล้ว) ห้าม Redistribute กลับเข้า EIGRP อีก
DIST-SW3(config)# route-map OSPF-TO-EIGRP deny 10
DIST-SW3(config-route-map)# match tag 90
DIST-SW3(config-route-map)# exit
DIST-SW3(config)# route-map OSPF-TO-EIGRP permit 20
DIST-SW3(config-route-map)# set tag 110
DIST-SW3(config-route-map)# exit
!
! ทิศทาง EIGRP -> OSPF: ถ้า Route มี Tag 110 อยู่แล้ว (มาจาก OSPF ตั้งแต่ต้น) ห้าม Redistribute
! กลับเข้า OSPF อีก
DIST-SW3(config)# route-map EIGRP-TO-OSPF deny 10
DIST-SW3(config-route-map)# match tag 110
DIST-SW3(config-route-map)# exit
DIST-SW3(config)# route-map EIGRP-TO-OSPF permit 20
DIST-SW3(config-route-map)# set tag 90
DIST-SW3(config-route-map)# exit
!
DIST-SW3(config)# router ospf 1
DIST-SW3(config-router)# no redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF-SELECTIVE
DIST-SW3(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
DIST-SW3(config-router)# exit
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
```

**DIST-SW4 ใช้ Route-map ชื่อเดียวกันทุกประการ** (Config เหมือนกันเป๊ะ เพราะ Logic การป้องกัน
Loop เป็นกฎเดียวกันทั้งระบบ ไม่ขึ้นกับว่า ASBR ตัวไหน):

```
! ===== DIST-SW4 (Route-map/Logic เหมือน DIST-SW3 ทุกประการ) =====
DIST-SW4(config)# route-map OSPF-TO-EIGRP deny 10
DIST-SW4(config-route-map)# match tag 90
DIST-SW4(config-route-map)# exit
DIST-SW4(config)# route-map OSPF-TO-EIGRP permit 20
DIST-SW4(config-route-map)# set tag 110
DIST-SW4(config-route-map)# exit
DIST-SW4(config)# route-map EIGRP-TO-OSPF deny 10
DIST-SW4(config-route-map)# match tag 110
DIST-SW4(config-route-map)# exit
DIST-SW4(config)# route-map EIGRP-TO-OSPF permit 20
DIST-SW4(config-route-map)# set tag 90
DIST-SW4(config-route-map)# exit
!
DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
DIST-SW4(config-router)# exit
DIST-SW4(config)# router eigrp EIGRP-DEMO
DIST-SW4(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW4(config-router-af)# topology base
DIST-SW4(config-router-af-topology)# redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
```

### ตรวจสอบว่า Loop ถูกป้องกันแล้วจริง

```
CORE-SW1# show ip route 10.10.10.0
Routing entry for 10.10.10.0/24
  Known via "ospf 1", distance 110, metric 1, type intra area
  Last update from 10.255.10.2 on TenGigabitEthernet1/0/1, 00:12:03 ago
```

`10.10.10.0/24` ยังคงเป็น **`type intra area` (O ปกติ) ผ่านเส้นทางเดิมใน Area 1 เท่านั้น**
ไม่มี `O E2` ของ Prefix เดียวกันโผล่กลับมาจากทาง DIST-SW4/Area 2 อีกเลย — ยืนยันว่า Tag
ทำงานถูกต้อง ป้องกันไม่ให้ Route วิ่งวนกลับเข้า Domain ต้นฉบับของตัวเอง

> **หมายเหตุ**: ในยุค IOS เก่าก่อนที่ `route-map` จะรองรับ `match tag`/`set tag` ได้สะดวก
> วิศวกรบางกลุ่มใช้ `distribute-list <acl> out <protocol> <process/AS>` ร่วมกับ Tag แบบ
> Static Route (`ip route ... tag <value>`) แทน แต่วิธีนี้ยุ่งยากกว่าและใช้ได้จำกัดกว่ามาก —
> **แนวทางที่แนะนำในปัจจุบันคือผูก Route-map ที่มี `match tag`/`set tag` เข้ากับคำสั่ง
> `redistribute` โดยตรง** ตามตัวอย่างข้างต้นเสมอ

---

## Step 308 — Redistribute BGP เข้า/ออก IGP: กับดักอันตรายที่สุดของหัวข้อนี้

### อันตรายที่ต้องเข้าใจก่อน: ห้าม Redistribute Full BGP Table เข้า IGP โดยเด็ดขาด

ใน Lab นี้ WAN-EDGE-1/WAN-EDGE-2 ทำ eBGP กับ ISP-RTR (AS 65000) ในนาม Enterprise AS 65001
(ตั้งค่าไว้ตั้งแต่ Part 29-30) ถ้า Internet Router จริงแบบนี้ต่อกับ ISP จริง BGP Table ที่ได้
รับมาอาจมีมากกว่า **900,000 Prefix** — ถ้าพิมพ์คำสั่งไร้เดียงสาแบบนี้บน WAN-EDGE-1:

```
! ❌❌❌ ห้ามทำแบบนี้เด็ดขาดในระบบจริง ❌❌❌
WAN-EDGE-1(config-router)# redistribute bgp 65001 subnets
```

ผลที่ตามมา:

| ผลกระทบ | รายละเอียด |
|---|---|
| OSPF Database ระเบิด | CORE-SW1/CORE-SW2/DIST-SW1-4 ทุกตัวต้องเก็บ LSA Type 5 นับแสนรายการ |
| CPU/Memory Router ทุกตัว Spike | Router รุ่นเล็ก (เช่น Catalyst 9300 บน DIST-SW) อาจ Crash หรือ SPF คำนวณไม่ทัน |
| SPF Recalculation ค้าง | ทุกครั้งที่ Internet Route เปลี่ยน (เกิดขึ้นหลายพันครั้ง/วันบน Internet จริง) OSPF ทั้ง Domain ต้องคำนวณ SPF ใหม่ทุกครั้ง |
| Routing Table ทุก Endpoint บวม | PC/Server ทุกเครื่องที่พึ่ง Router พวกนี้ได้รับผลกระทบทางอ้อมจาก Router หลักช้าลง |

> **นี่คือกับดักข้อสอบ CCNP ENCOR ข้อหนึ่งที่ออกบ่อยที่สุด**: คำถามมักถามว่า "อะไรคือ Best
> Practice เมื่อต้อง Redistribute BGP เข้า IGP" — คำตอบที่ถูกต้องเสมอคือ **"อย่าทำถ้าไม่จำเป็น
> จริงๆ ให้ใช้ Default Route หรือ Route ที่ Summarize แล้วเท่านั้น"**

### Safe Pattern ที่ 1: ใช้ Default Route แทนการ Redistribute ทั้ง Table (แนะนำที่สุด)

```
! บน WAN-EDGE-1: หลังจากมี Default Route จาก eBGP อยู่แล้ว (0.0.0.0/0 ผ่าน BGP)
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# default-information originate always metric 1 metric-type 1
```

`default-information originate` ฉีด **เฉพาะ** `0.0.0.0/0` เข้า OSPF เป็น `O*E1`/`O*E2`
โดยไม่ต้อง Redistribute Route อื่นใดจาก BGP เลยแม้แต่เส้นเดียว — Campus ทั้งหมดจะมี Gateway
of Last Resort ชี้มาทาง WAN-EDGE-1 อัตโนมัติ (`always` บังคับให้ Advertise แม้ WAN-EDGE-1
เองจะไม่มี Default Route ของตัวเองในตารางก็ตาม — ใช้ระวังเฉพาะกรณีที่มั่นใจว่า Router ตัวนี้
จะมี Path ออก Internet เสมอ)

### Safe Pattern ที่ 2: Redistribute เฉพาะ Prefix ที่กรองแล้วผ่าน Route-map (ถ้าจำเป็นต้องส่ง
มากกว่า Default Route เดียว เช่น ต้องการ Summary เฉพาะบาง Region ของ ISP)

```
WAN-EDGE-1(config)# ip prefix-list BGP-SAFE-LIST seq 5 permit 0.0.0.0/0
WAN-EDGE-1(config)# ip prefix-list BGP-SAFE-LIST seq 10 permit 198.51.100.0/24    ! ตัวอย่าง Partner Block เจาะจง
!
WAN-EDGE-1(config)# route-map BGP-TO-OSPF-SAFE permit 10
WAN-EDGE-1(config-route-map)# match ip address prefix-list BGP-SAFE-LIST
WAN-EDGE-1(config-route-map)# set metric 20
WAN-EDGE-1(config-route-map)# exit
!
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# redistribute bgp 65001 subnets route-map BGP-TO-OSPF-SAFE
```

เพราะ Route-map มี Sequence เดียวที่ `permit` และ Implicit Deny ท้ายสุด **มีแค่ 2 Prefix
เท่านั้น** ที่จะถูก Redistribute เข้า OSPF ไม่ว่า BGP Table จริงจะมีกี่แสน Prefix ก็ตาม

### ทิศทางกลับ: Redistribute OSPF/EIGRP เข้า BGP — ก็ต้อง Summarize เช่นกัน

ในทางกลับกัน การส่ง Route ภายใน Campus ออกไปให้ ISP รู้จัก (ผ่าน eBGP) ก็ **ห้าม
Redistribute ทุก Subnet ภายในออกไปตรงๆ** เช่นกัน เพราะ:
- ISP ไม่จำเป็นต้องรู้จัก Topology ภายในละเอียดขนาด VLAN แต่ละวง (Security + Best Practice)
- Route ภายในที่ Flap บ่อย (เช่น Subnet ทดสอบ, EIGRP External ที่ผ่านมาจาก Step ก่อนหน้า)
  จะทำให้ BGP Update Message ส่งออกบ่อยเกินจำเป็น กระทบเสถียรภาพของ eBGP Session เอง

**วิธีที่ถูกต้อง**: สรุป (Summarize) Subnet ทั้งหมดของ Campus ให้เหลือเป็น Static
Route/Aggregate เดียว (`10.10.0.0/16` ตาม IP Plan) แล้ว Redistribute หรือ `network`
เฉพาะ Route สรุปนั้นเข้า BGP:

```
WAN-EDGE-1(config)# ip route 10.10.0.0 255.255.0.0 Null0 250     ! Static เพื่อทำให้ BGP "network" ใช้ได้ (AD สูงมาก ไม่รบกวน Forwarding จริง)
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# network 10.10.0.0 mask 255.255.0.0
```

วิธีนี้ทำให้ ISP เห็น Enterprise เป็น **1 Prefix เดียว** (`10.10.0.0/16`) แทนที่จะเห็นทุก VLAN
แยกกัน — ตรงกับหลักการ Route Summarization ที่เรียนมาแล้วทั้งใน Static Route (Part 9),
OSPF `area range` (Part 12), และ EIGRP `ip summary-address` (Part 13)

---

## Step 309 — คำสั่ง Verify/Troubleshoot Redistribution ทั้งหมด

### `show ip protocols` — เห็นภาพรวมการ Redistribute ทันที

```
DIST-SW3# show ip protocols
*** IP Routing is NSF aware ***

Routing Protocol is "ospf 1"
  ...
  Redistributing External Routes from,
    eigrp 100, includes subnets in redistribution
Routing Protocol is "eigrp EIGRP-DEMO"
  ...
  Redistributing: eigrp EIGRP-DEMO
    Redistributing External Routes from,
      ospf 1
```

บรรทัด `Redistributing External Routes from` คือหลักฐานยืนยันว่า Process ไหนกำลังดึง Route
จาก Process ใดบ้าง — ใช้ตรวจสอบเร็วที่สุดว่า Redistribution ถูก Config ไว้จริงหรือไม่ (บางครั้ง
วิศวกร Config แล้วลืมว่าเคย `no redistribute` ไปแล้วหรือยัง)

### `show route-map` — เห็นว่า Route-map แต่ละ Sequence "โดนใช้งานกี่ครั้ง"

```
DIST-SW3# show route-map EIGRP-TO-OSPF
route-map EIGRP-TO-OSPF, permit, sequence 20
  Match clauses:
  Set clauses:
    tag 90
  Policy routing matches: 0 packets, 0 bytes
route-map EIGRP-TO-OSPF, deny, sequence 10
  Match clauses:
    tag 110
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
```

> **ข้อควรระวัง**: `show route-map` แสดง Sequence **ไม่เรียงตามเลขเสมอไป** (ขึ้นกับลำดับที่
> IOS จัดเก็บ Memory) — อย่าเข้าใจผิดว่า Sequence ที่แสดงก่อนคือ Sequence ที่ทำงานก่อน ให้ดู
> ที่ตัวเลข `sequence` ในแต่ละ Block เป็นหลักเสมอ (ตัวเลขน้อยกว่า = ประเมินก่อนเสมอ ไม่ว่า
> ลำดับการแสดงผลจะเป็นอย่างไร)

### สืบต้นตอ Route Redistributed ด้วย OSPF LSA Type 5 (External LSA)

```
CORE-SW1# show ip ospf 1 database external

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Type-5 AS External Link States

  Link ID         ADV Router      Age         Seq#       Checksum Tag
  10.199.200.0    1.1.1.13        45          0x80000001 0x00A1B2 90
```

คอลัมน์ `ADV Router` (**1.1.1.13** = DIST-SW3 ตาม IP Plan Loopback0) บอกว่า **ASBR ตัวไหน**
เป็นคนฉีด Route นี้เข้ามาใน OSPF Domain — เป็นวิธีสืบต้นตอ Route External ที่แม่นยำที่สุด
เมื่อสงสัยว่า Route หนึ่งๆ "มาจากไหนกันแน่" และคอลัมน์ `Tag` (**90**) ยืนยันว่า Route-map
ใน Step 307 ติด Tag ให้จริงตามที่ Config ไว้

### `show ip route <prefix> tag` และ `show ip eigrp topology` สำหรับฝั่ง EIGRP

```
EIGRP-R1# show ip eigrp topology 10.10.10.0/24
EIGRP-IPv4 (AS 100): Topology entry for 10.10.10.0/24
  State is Passive, Query origin flag is 1, 1 Successor(s), FD is 28160256
  Routing Descriptor Blocks:
  10.199.34.1 (GigabitEthernet0/0), from 10.199.34.1, Send flag is 0x0
      Composite metric is (28160256/0), route is External
      ...
      External data:
        Originating router is 1.1.1.13
        AS number of route is 0
        External protocol is OSPF, external metric is 20
        Administrator tag is 110 (0x0000006E)
```

`show ip eigrp topology <prefix>` แสดง **`External data`** เต็มรูปแบบ ทั้ง Originating
Router (ASBR), External Protocol ต้นทาง (OSPF), External Metric เดิม, และ Tag — ครบถ้วน
สำหรับ Debug ปัญหา Redistribution ฝั่ง EIGRP

### สรุปข้อผิดพลาดที่พบบ่อยที่สุด 5 แบบของหัวข้อนี้

| # | ข้อผิดพลาด | อาการ | วิธีแก้ |
|---|---|---|---|
| 1 | ลืมใส่ `subnets` ตอน Redistribute เข้า OSPF | Route ที่เป็น Subnet (ไม่ใช่ Classful Major Net) หายไปทั้งหมด | เติม `subnets` ต่อท้าย `redistribute` เสมอ |
| 2 | ลืมใส่ `metric` (Seed Metric) ตอน Redistribute เข้า EIGRP | Route ไม่ถูกติดตั้งเข้า EIGRP เลย แบบไม่มี Error | ใส่ `metric <bw> <delay> <rel> <load> <mtu>` หรือใช้ `default-metric` |
| 3 | Redistribute Mutual 2 จุดโดยไม่ใส่ Tag | Route วิ่งวนกลับเข้า Domain ต้นฉบับ, Suboptimal Path หรือ Loop | ใช้ Route-map + `match tag`/`set tag` ตาม Step 307 |
| 4 | Redistribute BGP เข้า IGP แบบไม่กรอง | CPU/Memory Spike, SPF/DUAL ค้าง, Router อาจ Crash | ใช้ `default-information originate` หรือ Route-map+Prefix-list กรองเฉพาะที่จำเป็น (Step 308) |
| 5 | AD ของ External Route ชนกับ Internal Route โดยไม่ตั้งใจ | เลือกเส้นทางผิด (เช่น เดินทางอ้อมผ่าน Domain อื่นแทนที่จะไปตรง) | ตรวจสอบด้วย `show ip route <prefix>` ดูว่า Route ที่ถูกเลือกจริงคือ Internal (O/D) หรือ External (O E2/D EX) — ถ้าผิดคาดให้ทบทวน Route-map/Tag |

---

## Step 310 — Lab เต็มรูปแบบ: Mutual Redistribution OSPF↔EIGRP พร้อม Tag และ Controlled Redistribution BGP→OSPF ที่ WAN Edge

### Topology สรุปทั้งหมดของ Lab นี้

```
                              AS 65000                       AS 65001 (Enterprise)
                          ┌───────────────┐
                          │    ISP-RTR      │ Lo1: 8.8.8.8/32
                          └───┬───────┬───┘  (Simulated Internet)
                   203.0.113.0/30   203.0.113.4/30
                      (.1)  │       │  (.5)
                            │(.2)  (.6)│
                   ┌────────┴┐     ┌┴────────┐
                   │WAN-EDGE-1│═════│WAN-EDGE-2│  eBGP to ISP (AS 65000)
                   └────┬────┘10.10.254.8/30└────┬────┘  redistribute bgp -> ospf (Default Route เท่านั้น)
              10.10.254.0/30(.1)          (.10) 10.10.254.4/30  redistribute ospf -> bgp (10.10.0.0/16 summary เท่านั้น)
                   (.2) │                          │ (.6)
                   ┌────┴────┐                ┌────┴────┐
                   │ CORE-SW1 │════Po1═════════│ CORE-SW2 │        OSPF Process 1
                   └──┬───┬──┘   Area 0        └──┬───┬──┘
                  Area1│   │Area1            Area2│   │Area2
                   ┌───┴┐ ┌┴───┐              ┌───┴┐ ┌┴───┐
                   │DIST│ │DIST│              │DIST│ │DIST│
                   │-SW1│ │-SW2│              │-SW3│ │-SW4│  ★ASBR (OSPF<->EIGRP, มี Tag ป้องกัน Loop)★
                   └────┘ └────┘              └──┬─┘ └──┬─┘
                                       Link1 (Gi1/0/23) │ │ (Gi1/0/22) Link3
                                    10.199.34.0/30 AS100│ │10.199.34.8/30
                                                   ┌─────┴┐┌┴─────┐
                                                   │EIGRP-R1││EIGRP-R2│
                                                   │Lo0 9.9.9.1││Lo0 9.9.9.2
                                                   └────────┘│Lo1 10.199.200.0/24
                                                              └────────┘ Legacy Server LAN
```

### สรุปการออกแบบ Redistribution ทั้ง Lab

| จุด ASBR | ทิศทาง | ควบคุมด้วย | เหตุผล |
|---|---|---|---|
| DIST-SW3 | EIGRP AS100 ↔ OSPF Process1 Area2 | Route-map + Tag (90/110) | ป้องกัน Loop จากการมี 2 จุด Redistribute พร้อมกัน (Step 307) |
| DIST-SW4 | EIGRP AS100 ↔ OSPF Process1 Area2 | Route-map + Tag (90/110) เหมือน DIST-SW3 | เช่นเดียวกัน |
| WAN-EDGE-1 | BGP AS65001 → OSPF Process1 | `default-information originate` (Default Route เท่านั้น) | ป้องกัน Full BGP Table Leak (Step 308) |
| WAN-EDGE-1 | OSPF Process1 → BGP AS65001 | Static Summary + `network` Statement (`10.10.0.0/16`) | ส่งแค่ Summary เดียวออก Internet ไม่ Leak รายละเอียด VLAN ภายใน |
| WAN-EDGE-2 | เหมือน WAN-EDGE-1 ทุกประการ (Redundant Path) | เหมือนกัน | Backup เส้นทางออก Internet |

### Running-config เต็ม: DIST-SW3 (ส่วน Redistribution — ต่อยอดจาก OSPF/EIGRP ที่มีอยู่แล้ว)

```
hostname DIST-SW3
!
ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24
!
route-map OSPF-TO-EIGRP deny 10
 match tag 90
!
route-map OSPF-TO-EIGRP permit 20
 set tag 110
!
route-map EIGRP-TO-OSPF deny 10
 match tag 110
!
route-map EIGRP-TO-OSPF permit 20
 set tag 90
!
router ospf 1
 router-id 1.1.1.13
 redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
!
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.13
  topology base
   redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
  exit-af-topology
 exit-address-family
!
end
```

### Running-config เต็ม: DIST-SW4 (เหมือน DIST-SW3 ทุกประการ ต่างแค่ Router-ID)

```
hostname DIST-SW4
!
route-map OSPF-TO-EIGRP deny 10
 match tag 90
!
route-map OSPF-TO-EIGRP permit 20
 set tag 110
!
route-map EIGRP-TO-OSPF deny 10
 match tag 110
!
route-map EIGRP-TO-OSPF permit 20
 set tag 90
!
router ospf 1
 router-id 1.1.1.14
 redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
!
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.14
  topology base
   redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
  exit-af-topology
 exit-address-family
!
end
```

### Running-config เต็ม: WAN-EDGE-1 (ส่วน Redistribution — ต่อยอดจาก eBGP ที่ Config ไว้แล้วใน Part 29-30)

```
hostname WAN-EDGE-1
!
ip route 10.10.0.0 255.255.0.0 Null0 250
!
interface Loopback0
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Link to CORE-SW1 (now running OSPF Area 0, replaces static from Part 9) **
 ip address 10.10.254.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.21
 default-information originate always metric 1 metric-type 1
!
router bgp 65001
 bgp router-id 1.1.1.21
 neighbor 203.0.113.1 remote-as 65000
 network 10.10.0.0 mask 255.255.0.0
!
end
```

### Running-config เต็ม: WAN-EDGE-2 (Redundant Path — เหมือน WAN-EDGE-1)

```
hostname WAN-EDGE-2
!
ip route 10.10.0.0 255.255.0.0 Null0 250
!
interface Loopback0
 ip address 1.1.1.22 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Link to CORE-SW2 (OSPF Area 0) **
 ip address 10.10.254.5 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.22
 default-information originate always metric 5 metric-type 1
!
router bgp 65001
 bgp router-id 1.1.1.22
 neighbor 203.0.113.5 remote-as 65000
 network 10.10.0.0 mask 255.255.0.0
!
end
```

> **หมายเหตุ Design**: WAN-EDGE-2 ตั้ง `metric 5` (แพงกว่า WAN-EDGE-1 ที่ `metric 1`) สำหรับ
> Default Route เพื่อให้ Campus **เลือกออก Internet ผ่าน WAN-EDGE-1 เป็นหลักเสมอ** และสลับไป
> WAN-EDGE-2 อัตโนมัติเมื่อ WAN-EDGE-1 มีปัญหาเท่านั้น — หลักการเดียวกับ Floating Static
> Route ใน Part 9 Step 86 แต่ทำผ่าน OSPF Cost แทน

### Verification 1: CORE-SW1 เห็น Default Route ผ่าน OSPF (จาก BGP ที่ Redistribute แบบควบคุม)

```
CORE-SW1# show ip route ospf | include ^O
O*E1 0.0.0.0/0 [110/1] via 10.10.254.1, 00:05:44, GigabitEthernet1/0/1
O E2 10.199.200.0/24 [110/20] via 10.255.30.1, 00:12:03, TenGigabitEthernet1/0/1
```

`O*E1` ยืนยันว่า Default Route จาก WAN-EDGE-1 ถูกเลือกเป็น Gateway of Last Resort (`*`)
ด้วย Cost `1` (ต่ำกว่า WAN-EDGE-2 ที่ Cost `5` แน่นอน) — **ไม่มี Prefix อื่นจาก BGP Table
รั่วเข้ามาเลยแม้แต่เส้นเดียว** ตรงตามหลักการ Safe Pattern ใน Step 308

### Verification 2: EIGRP-R1 เห็นทั้ง Campus (ผ่าน OSPF Redistribute) และ Internet (ผ่าน Default Route ที่ลอยมาถึง)

```
EIGRP-R1# show ip route
Gateway of last resort is 10.199.34.1 to network 0.0.0.0

D EX 0.0.0.0/0 [170/28160256] via 10.199.34.1, 00:04:02, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 17 subnets, 3 masks
D EX    10.10.10.0/24 [170/28160256] via 10.199.34.1, 00:20:15, GigabitEthernet0/0
D EX    10.10.20.0/24 [170/28160256] via 10.199.34.1, 00:20:15, GigabitEthernet0/0
D EX    10.10.30.0/24 [170/28160256] via 10.199.34.1, 00:20:15, GigabitEthernet0/0
D EX    10.10.40.0/24 [170/28160256] via 10.199.34.1, 00:20:15, GigabitEthernet0/0
D EX    10.10.99.0/24 [170/28160256] via 10.199.34.1, 00:20:15, GigabitEthernet0/0
      9.0.0.0/32 is subnetted, 2 subnets
D        9.9.9.2 [90/156160] via 10.199.34.5, 00:25:40, GigabitEthernet0/1
```

Legacy Site เห็นทั้ง Default Route ออก Internet (`D EX 0.0.0.0/0`, Redistribute จาก
Domain OSPF ต่อผ่านมาจาก BGP อีกทีหนึ่ง — วิ่งผ่านการ Redistribute 2 ต่อ: BGP→OSPF ที่
WAN-EDGE-1 แล้ว OSPF→EIGRP ที่ DIST-SW3) และ Campus VLAN ทุกวง (`D EX 10.10.x.0/24`) ครบถ้วน

### Verification 3: ยืนยันไม่มี Routing Loop ด้วย Traceroute ข้าม Domain

```
PC-Legacy(บนวง 10.199.200.0/24)> traceroute 10.10.10.55

Tracing route to 10.10.10.55 over a maximum of 30 hops:
  1   1 ms   1 ms   1 ms   10.199.200.254      <- EIGRP-R2 (Gateway ของ Legacy Server LAN)
  2   2 ms   2 ms   2 ms   10.199.34.9         <- DIST-SW4 (ASBR ฝั่ง EIGRP)
  3   3 ms   3 ms   3 ms   10.255.40.1         <- CORE-SW2 (ผ่าน OSPF Area 2)
  4   4 ms   4 ms   4 ms   10.10.10.1          <- CORE-SW1/DIST-SW1 (SVI Gateway VLAN10)
  5   4 ms   4 ms   4 ms   10.10.10.55         <- ปลายทาง PC1

Trace complete.
```

Traceroute แสดงเส้นทาง **ตรงไปตรงมาไม่มี Hop ซ้ำหรือวนกลับเลย** (เทียบกับอาการ Recursive
Loop ที่ Part 9 Step 89 สอนให้สังเกต Pattern สลับ IP ไปมา) — ยืนยันว่าการออกแบบ Tag ป้องกัน
Loop ใน Step 307 ทำงานถูกต้องสมบูรณ์ทั้ง Lab แม้จะมีจุด Redistribute พร้อมกันถึง 4 จุด
(DIST-SW3, DIST-SW4, WAN-EDGE-1, WAN-EDGE-2)

### Verification 4: ISP-RTR เห็น Enterprise เป็น Summary เดียวเท่านั้น (ไม่ Leak รายละเอียดภายใน)

```
ISP-RTR# show ip route bgp
      10.0.0.0/8 is variably subnetted, 1 subnets, 1 masks
B       10.10.0.0/16 [20/0] via 203.0.113.2, 00:08:21
```

ISP มองเห็น Enterprise ทั้งหมด (Campus + Legacy Site) เป็น **Prefix เดียว** `10.10.0.0/16`
เท่านั้น (AD=20 ตรงตามตาราง eBGP ใน Part 9 Step 85) ไม่มี VLAN หรือ Subnet ภายในรั่วออกไป
ให้โลกภายนอกเห็นเลยแม้แต่เส้นเดียว — ครบถ้วนตามหลักการออกแบบ Redistribution ที่ปลอดภัยทั้ง
3 Domain ของ Enterprise Lab

---

## แบบฝึกหัดทวนความเข้าใจ Part 31

1. เพราะเหตุใดการ Redistribute EIGRP เข้า OSPF จึงต้องใส่คำว่า `subnets` ต่อท้ายเสมอ ถ้าไม่ใส่
   จะเกิดอะไรขึ้น?
2. เพราะเหตุใด Seed Metric จึงเป็นสิ่ง**บังคับ**เมื่อ Redistribute เข้า EIGRP (ต่างจาก OSPF ที่
   ยังมี Default Metric ให้ใช้)? ถ้าลืมตั้งจะเกิดอาการอะไร?
3. อธิบายว่า Routing Loop จากการ Redistribute สองทิศทางที่ 2 จุดพร้อมกัน (เช่น DIST-SW3 และ
   DIST-SW4) เกิดขึ้นได้อย่างไร และ Route Tag ช่วยป้องกันปัญหานี้ด้วยหลักการอะไร?
4. เพราะเหตุใดการ Redistribute Full BGP Table เข้า IGP โดยตรงจึงเป็นสิ่งที่ห้ามทำในระบบจริง
   และมี Safe Pattern อะไรบ้างที่ใช้แทนได้?
5. คำสั่งใดใช้ตรวจสอบว่า OSPF ASBR ตัวไหนเป็นคนฉีด External Route เข้ามาใน Domain และคำสั่ง
   ใดใช้ตรวจสอบสถานะการทำงานของแต่ละ Sequence ใน Route-map?

**เฉลย:**

1. เพราะโดย Default OSPF จะ Redistribute เฉพาะ **Classful Major Network** เท่านั้น (เช่น
   `10.0.0.0/8` เส้นเดียว) ถ้าไม่ใส่ `subnets` ทุก Subnet ที่ VLSM แบ่งย่อยไว้ (เช่น
   `10.199.200.0/24`) จะ**ไม่ถูก Redistribute เข้าไปเลย** เพราะไม่ใช่ Major Network เดี่ยวๆ
2. เพราะ EIGRP ต้องใช้ค่า Composite Metric (Bandwidth/Delay/Reliability/Load) ในการคำนวณ
   Metric เสมอ และ**ไม่มี Default Value ใดๆ ให้ใช้แทน**ต่างจาก OSPF (ที่มี Default=20) —
   ถ้าลืมตั้ง `metric` หรือ `default-metric` Route จาก Protocol อื่นจะ**ไม่ถูกติดตั้งเข้า
   EIGRP เลยแบบเงียบๆ ไม่มี Error แจ้งเตือน**
3. เกิดขึ้นเมื่อ Route ที่ถูก Redistribute จาก Domain A ไป Domain B ที่จุดหนึ่ง ถูก
   Redistribute ย้อนกลับจาก Domain B ไป Domain A ที่**อีกจุดหนึ่ง**ซ้ำอีกครั้ง ทำให้ Route
   วิ่งวนข้าม Domain ไปมา อาจทำให้เลือก Path ผิดพลาด (Suboptimal) หรือเกิด Loop เต็มรูปแบบ —
   แก้ด้วยการติด **Tag** ให้ทุก Route ที่ถูก Redistribute ระบุว่า "มาจาก Domain ไหน" แล้วใช้
   Route-map ที่มี `match tag`/`deny` ปฏิเสธไม่ให้ Redistribute Route ที่มี Tag ของ Domain
   ตัวเองอยู่แล้วกลับเข้าไปซ้ำอีก
4. เพราะ BGP Table จริง (โดยเฉพาะ Internet Full Table) อาจมีหลายแสน Prefix ซึ่งจะทำให้ OSPF/
   EIGRP Database ระเบิด, CPU/Memory Router Spike, และ SPF/DUAL คำนวณไม่ทัน จนอาจทำให้
   Router Crash ทั้ง Domain — Safe Pattern ที่ใช้แทนคือ `default-information originate`
   (ฉีดแค่ Default Route เดียว) หรือ Redistribute ผ่าน Route-map+Prefix-list ที่กรองเฉพาะ
   Prefix ที่จำเป็นจริงๆ เท่านั้น
5. ใช้ `show ip ospf <process-id> database external` ดูคอลัมน์ `ADV Router` เพื่อหาว่า ASBR
   ตัวไหนเป็นคนฉีด External LSA (Type 5) เข้ามา ส่วนการตรวจสอบสถานะ Route-map แต่ละ
   Sequence ใช้คำสั่ง `show route-map <name>` (แสดงจำนวนครั้งที่แต่ละ Sequence ถูกใช้งาน
   ผ่าน Policy Routing Matches Counter)

---

## สรุป Part 31

Part นี้ปิดช่องว่างสำคัญที่สุดของการมี **3 Routing Domain แยกกัน** ในหลักสูตร: **Route
Redistribution** ตั้งแต่เหตุผลที่ต้องทำและอันตรายที่ต้องระวัง, Syntax พื้นฐานและความสำคัญของ
**Seed Metric** ที่แตกต่างกันในแต่ละ Protocol, Worked Config ทั้งสองทิศทางระหว่าง **EIGRP ↔
OSPF** บน DIST-SW3/DIST-SW4, **Route-map** ฉบับเต็มทั้ง Match/Set Clause, **Selective
Redistribution** ด้วย Prefix-list, ปัญหา **Routing Loop จากการ Redistribute หลายจุด** และ
วิธีแก้ด้วย **Route Tag**, อันตรายของการ **Redistribute BGP เข้า IGP แบบไม่ระวัง** พร้อม
Safe Pattern ที่ถูกต้อง, ชุดคำสั่ง Verify/Troubleshoot ครบถ้วน และปิดท้ายด้วย **Lab เต็ม
รูปแบบ** ที่เชื่อมทั้ง 3 Domain (OSPF Campus, EIGRP Legacy Site, BGP WAN Edge) เข้าด้วยกัน
อย่างปลอดภัย ไม่มี Routing Loop และไม่ Leak ข้อมูลเกินความจำเป็นในทุกทิศทาง ✅

**พร้อมสำหรับ Part 32**: เมื่อ Routing Table ของทุก Router ถูกต้องสมบูรณ์แล้ว ขั้นต่อไปคือ
การควบคุม **เส้นทางที่ Packet เดินทางจริง** ให้ละเอียดยิ่งกว่า Routing Table ปกติ ด้วย
**Policy-Based Routing (PBR)** — เทคนิคที่ใช้ Route-map ตัวเดียวกับที่เรียนใน Part นี้
มา "บังคับ" เส้นทางของ Traffic เฉพาะกลุ่มให้ต่างไปจากที่ Routing Table บอกไว้ ใช้บ่อยที่สุด
ในงาน Load Balancing แบบ Manual และ Traffic Engineering ระดับ Enterprise

**ไปต่อ:** [Part 32 — Policy-Based Routing (PBR) →](part-032-policy-based-routing.md)
