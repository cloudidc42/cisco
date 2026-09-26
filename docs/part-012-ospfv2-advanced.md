# Part 12 — OSPFv2 Advanced (Multi-Area, LSA Types, Authentication)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 111–120 จาก 1000**

> ต่อจาก [Part 11 — OSPFv2 Fundamentals](part-011-ospfv2-fundamentals.md) ที่นำ OSPFv2
> **Single-Area (Area 0 ล้วน)** มาแทนที่ Static Route ทั้งหมดบน CORE-SW1, CORE-SW2, DIST-SW1–4
> สำเร็จแล้ว Part นี้คือจุดเปลี่ยนสำคัญของระดับ **CCNP ENCOR** — เราจะรื้อ Design เดิมและ
> **แบ่ง OSPF ออกเป็น 3 Area** ตามหลัก Enterprise ที่ใช้งานจริง เจาะลึก **LSA แต่ละ Type**
> ที่ทำให้ OSPF scale ได้, เรียนรู้ **ABR/ASBR**, **Route Summarization**, **Stub/NSSA Area**,
> **Virtual Link**, และปิดท้ายด้วย Lab เต็มรูปแบบที่ทำให้ Topology หลักของหลักสูตรกลายเป็น
> Multi-Area OSPF ที่ถูกต้องตามมาตรฐาน Cisco จริง

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 111 | ทำไมต้องมี Multi-Area OSPF — ข้อจำกัดของ Single-Area |
| 112 | ABR และ ASBR — บทบาทของ CORE-SW1/CORE-SW2 ใน Lab นี้ |
| 113 | OSPF LSA Type ทั้งหมดโดยละเอียด (Type 1, 2, 3, 4, 5, 7) |
| 114 | Reconfigure Lab เป็น 3-Area Design |
| 115 | Inter-Area Route Summarization บน ABR (`area range`) |
| 116 | Area Type: Standard, Stub, Totally Stub, NSSA |
| 117 | Virtual Link — เชื่อม Area เข้า Backbone ทางอ้อม |
| 118 | OSPF Route Type และการคำนวณ Cost ข้าม Area (O, O IA, O E1, O E2) |
| 119 | Advanced Verification & Troubleshooting Multi-Area OSPF |
| 120 | Lab เต็มรูปแบบ: 3-Area OSPF พร้อม Route Summarization ทั้งสอง ABR |

---

## Step 111 — ทำไมต้องมี Multi-Area OSPF

[Part 11](part-011-ospfv2-fundamentals.md) นำ OSPF มาแทน Static Route ได้สำเร็จ แต่ Design ที่ใช้
คือ **Single-Area** — ทุกอุปกรณ์ (CORE-SW1, CORE-SW2, DIST-SW1–4) อยู่ใน **Area 0 เดียวกันทั้งหมด**
ใน Lab ขนาดเล็ก 6 อุปกรณ์นี้ยังไม่มีปัญหาให้เห็นชัด แต่เมื่อ Enterprise Network เติบโตขึ้นเป็นหลักร้อย
หรือหลักพัน Router/Switch การใช้ Area เดียวจะเริ่มมีปัญหา 3 เรื่องใหญ่ที่ทำให้ OSPF **ไม่ Scale**

### 111.1 ปัญหาที่ 1 — SPF Calculation Scalability (CPU Load)

OSPF ใช้ **Dijkstra's SPF (Shortest Path First) Algorithm** คำนวณ Best Path จาก Link-State
Database (LSDB) ทั้งหมด **ทุกครั้งที่ Topology เปลี่ยนแปลง** (Link Up/Down, Router เพิ่ม/ลด)
ความซับซ้อนของ SPF Algorithm อยู่ที่ประมาณ **O(n log n)** เมื่อ `n` คือจำนวน Router+Link ทั้งหมด
ใน Area นั้น — ถ้า Area เดียวมี Router 500 ตัว ทุกครั้งที่ Link ใดสักเส้นกระพริบ (Flap)
**ทุก Router ในทั้ง Area ต้องคำนวณ SPF ใหม่ทั้งหมด** ถึงจะเป็นแค่ Link เล็กๆ ที่ไม่เกี่ยวกับตัวเองเลย

```
Single-Area 500 Router:
  Link Flap ที่ Router #487  →  ทุก Router (1-500) รัน SPF ใหม่ทั้งหมด  →  CPU Spike ทั้ง Network

Multi-Area (แบ่งเป็น 10 Area x 50 Router):
  Link Flap ที่ Router #487 (อยู่ Area 9)  →  เฉพาะ Router ใน Area 9 (50 ตัว) รัน SPF ใหม่
                                              Area อื่นไม่กระทบเลย (เห็นแค่ Type 3 Summary เปลี่ยน)
```

### 111.2 ปัญหาที่ 2 — LSA Flooding Domain (ขนาด LSDB)

**Flooding Domain** คือขอบเขตที่ LSA ชนิด Type 1/2 (รายละเอียด Topology ภายใน) ถูก Flood ไปถึง
— ใน Single-Area ขอบเขตนี้คือ**ทั้ง Autonomous System** ทำให้ Router ทุกตัวต้องเก็บ LSDB ขนาดใหญ่
เท่ากันหมด แม้จะเป็น Router เล็กๆ ที่ Distribution Layer ก็ต้องรู้จัก Topology ภายในของทุก Site
ทั่วทั้งองค์กร ซึ่งไม่มีประโยชน์และกิน Memory โดยไม่จำเป็น — Multi-Area จำกัด Type 1/2 LSA ให้
Flood อยู่**ภายใน Area ของตัวเองเท่านั้น** ระหว่าง Area จะเห็นแค่ **Type 3 Summary LSA** (Step 113)
ซึ่งเป็นแค่ "สรุป Subnet + Cost" ไม่ใช่รายละเอียด Topology เต็มรูปแบบ

### 111.3 ปัญหาที่ 3 — Convergence Time

ยิ่ง LSDB ใหญ่ ยิ่งใช้เวลารัน SPF นานขึ้น (ไม่ใช่ Linear แต่เป็น n log n) และ Router ทุกตัวต้อง
ประมวลผล LSA จำนวนมากทุกครั้งที่มีการเปลี่ยนแปลง ทำให้ **เวลา Convergence โดยรวมของทั้ง AS ช้าลง**
เมื่อ Network โตขึ้น — การแบ่ง Area ทำให้ผลกระทบ (Blast Radius) ของการเปลี่ยนแปลงจำกัดอยู่แค่ Area
เดียว ทำให้ Convergence เร็วและคาดเดาได้มากขึ้น

### 111.4 กฎเหล็กของ Multi-Area OSPF — "Backbone Area 0 Rule"

> **กฎที่ต้องจำขึ้นใจ (ข้อสอบ CCNP ออกบ่อยที่สุดของหัวข้อนี้)**: OSPF Multi-Area ใช้สถาปัตยกรรม
> **Two-Layer Hierarchy** เสมอ — **Area 0 (Backbone Area)** ต้องเป็นศูนย์กลางที่ Area อื่นๆ
> **ทุก Area ต้องเชื่อมต่อกับ Area 0 โดยตรง** (ผ่าน ABR) หรือถ้าเชื่อมตรงไม่ได้ ต้องใช้
> **Virtual Link** (Step 117) เชื่อมทางอ้อมผ่าน Area อื่นที่เป็น Transit Area เท่านั้น
> **ห้าม Area สองใบที่ไม่ใช่ Area 0 คุยกันโดยตรงโดยไม่ผ่าน Area 0 เด็ดขาด** (No Area-to-Area
> Direct Adjacency) — นี่คือเหตุผลที่เรียกว่า "Backbone" เพราะทุก Inter-Area Traffic ต้องวิ่ง
> ผ่านมันเสมอ ไม่ว่า Physical Topology จะมี Shortcut ที่ตรงกว่าหรือไม่ก็ตาม

```
                     ❌ ผิดกฎ — Area 1 คุยกับ Area 2 ตรง ไม่ผ่าน Area 0
                     ┌─────────┐          ┌─────────┐
                     │ Area 1  │──────────│ Area 2  │
                     └─────────┘          └─────────┘

                     ✅ ถูกกฎ — ทุก Area ต้องผ่าน Area 0 เสมอ
                     ┌─────────┐          ┌─────────┐
                     │ Area 1  │          │ Area 2  │
                     └────┬────┘          └────┬────┘
                          │                     │
                     ┌────┴─────────────────────┴────┐
                     │           Area 0 (Backbone)     │
                     └──────────────────────────────────┘
```

### 111.5 Design ใหม่ของ Lab หลักสูตรนี้ — 3-Area ตามสถาปัตยกรรม 3-Tier

Part นี้จะจับ Topology เดิมของ Part 1–11 มาแบ่ง Area ตามแนว **Core / Distribution** ที่ Design
มาตั้งแต่ต้นอยู่แล้ว — นี่คือการ mapping ที่เป็น Best Practice ของ Cisco จริง (**Area Boundary
ควรตรงกับ Physical/Administrative Boundary เสมอ** ไม่ใช่ตัดตามใจชอบ):

| Area | สมาชิก | Subnet ที่ครอบคลุม |
|---|---|---|
| **Area 0 (Backbone)** | CORE-SW1 ↔ CORE-SW2 (เฉพาะ Link นี้เท่านั้น) | 10.255.0.0/30 |
| **Area 1** | CORE-SW1 (ครึ่งหนึ่ง), DIST-SW1, DIST-SW2 | VLAN 10, 20, 99 + 10.255.10.0/30, 10.255.20.0/30 |
| **Area 2** | CORE-SW2 (ครึ่งหนึ่ง), DIST-SW3, DIST-SW4 | VLAN 30, 40, 99 + 10.255.30.0/30, 10.255.40.0/30 |

```
                         Area 1                  Area 0                 Area 2
                    (SALES/VOICE/MGMT)     (Backbone — CORE↔CORE)   (SERVERS/WIFI/MGMT)
┌──────────┐  ┌──────────┐        ┌──────────────────────────┐        ┌──────────┐  ┌──────────┐
│ DIST-SW1 │──│          │        │                            │        │          │──│ DIST-SW3 │
└──────────┘  │ CORE-SW1 │════════│  10.255.0.0/30 (Po1)      │════════│ CORE-SW2 │  └──────────┘
┌──────────┐  │  (ABR)   │        │                            │        │  (ABR)   │  ┌──────────┐
│ DIST-SW2 │──│          │        │                            │        │          │──│ DIST-SW4 │
└──────────┘  └──────────┘        └──────────────────────────┘        └──────────┘  └──────────┘
     ▲              ▲                                                       ▲              ▲
     └── Area 1 ────┘                                                       └── Area 2 ────┘
```

สังเกตว่า **CORE-SW1 และ CORE-SW2 แต่ละตัวมี Interface อยู่ 2 Area พร้อมกัน** (Port-channel1
อยู่ Area 0, ส่วน Te1/0/1+Te1/0/2 อยู่ Area 1 หรือ Area 2) — นี่คือนิยามของ **ABR** ซึ่งเป็น
หัวข้อของ Step ถัดไปพอดี

---

## Step 112 — ABR และ ASBR: บทบาทของ CORE-SW1/CORE-SW2 ใน Lab นี้

OSPF มีชื่อเรียก Router ตามบทบาทใน Topology 4 แบบหลัก — ต้องแยกให้ออกเพราะเป็นคำศัพท์ที่ใช้
ตลอดทั้ง CCNP/CCIE:

| บทบาท | นิยาม | ตัวอย่างใน Lab นี้ |
|---|---|---|
| **Internal Router** | ทุก Interface อยู่ใน Area เดียวกันหมด | DIST-SW1, DIST-SW2 (อยู่ Area 1 ล้วน), DIST-SW3, DIST-SW4 (อยู่ Area 2 ล้วน) |
| **ABR (Area Border Router)** | มี Interface อยู่มากกว่า 1 Area พร้อมกัน — เชื่อม Area นั้นเข้ากับ Area 0 | **CORE-SW1** (Area 0 + Area 1), **CORE-SW2** (Area 0 + Area 2) |
| **Backbone Router** | Interface อยู่ใน Area 0 อย่างน้อย 1 ใบ (ABR ทุกตัวเป็น Backbone Router ด้วยเสมอ) | CORE-SW1, CORE-SW2 |
| **ASBR (Autonomous System Boundary Router)** | เชื่อมต่อ OSPF เข้ากับ Routing Domain อื่น (Static Route, EIGRP, BGP) ด้วยการ **Redistribute** | ยังไม่มีใน Lab ปัจจุบัน — จะเป็น **WAN-EDGE-1/2** เมื่อ Redistribute Static/BGP เข้า OSPF ใน Part ถัดไปที่เกี่ยวกับ WAN/Internet Edge |

### 112.1 CORE-SW1 และ CORE-SW2 เป็น ABR ได้อย่างไร

```
                              CORE-SW1
                    ┌─────────────────────────┐
   Area 1  ────────►│ Te1/0/1  (Area 1)        │
   (DIST-SW1)       │ Te1/0/2  (Area 1)        │──────► Area 0
                    │ Port-channel1 (Area 0)   │        (CORE-SW2)
   Area 1  ────────►│                            │
   (DIST-SW2)       └─────────────────────────┘

   CORE-SW1 มี Interface ทั้งใน Area 1 (Te1/0/1, Te1/0/2) และ Area 0 (Po1) พร้อมกัน
   → ตรงตามนิยาม ABR ทุกประการ
```

**หน้าที่หลักของ ABR** ในสถาปัตยกรรมนี้มี 3 อย่าง:

1. **รักษา LSDB แยกกันสำหรับแต่ละ Area** — CORE-SW1 เก็บ LSDB ของ Area 1 ชุดหนึ่ง และ LSDB ของ
   Area 0 อีกชุดหนึ่ง แยกกันโดยสมบูรณ์ (ไม่ปนกัน)
2. **สร้าง Type 3 (Summary) LSA** แปลง Route ภายใน Area 1 ให้เป็นข้อมูลสรุป แล้ว Flood เข้า
   Area 0 — และในทางกลับกัน แปลง Route จาก Area 0/Area อื่นให้เป็น Type 3 แล้ว Flood กลับเข้า
   Area 1 (ทำงานสองทาง)
3. **บังคับใช้ Backbone Rule** — ถ้า ABR ไม่มี Adjacency กับ Area 0 ที่ Full State เลย
   (เช่น Port-channel1 Down) Area 1 ทั้งหมดจะขาดการเชื่อมต่อกับ Area 0 ทันที แม้ Internal
   Router ภายใน Area 1 จะยังคุยกันได้ปกติก็ตาม (ปัญหานี้จะเห็นตัวอย่างจริงใน Step 119)

### 112.2 เมื่อไหร่ Lab นี้จะมี ASBR

**ASBR** เกิดขึ้นเมื่อ Router ตัวหนึ่ง Run OSPF อยู่ แล้วมีคำสั่ง `redistribute` นำ Route จาก
Source อื่นเข้ามาใน OSPF Domain เช่น Static Route ไป Internet, หรือ BGP จาก ISP ในหลักสูตรนี้
**WAN-EDGE-1/WAN-EDGE-2** จะกลายเป็น ASBR เมื่อถึง Part ที่ Redistribute Static Default Route
(ไป ISP-RTR) เข้า OSPF — ตอนนั้น LSA **Type 5 (External)** จะเริ่มปรากฏใน Lab จริงเป็นครั้งแรก
(ดู Step 113.5) สำหรับ Part นี้ เราจะยังไม่ Config ASBR จริง แต่จะอธิบาย LSA Type 4/5/7 ให้ครบ
เพื่อเตรียมความเข้าใจไว้ก่อน

> **จำง่ายๆ**: **ABR** = อยู่ **ระหว่าง 2 Area ของ OSPF เดียวกัน** ส่วน **ASBR** = อยู่
> **ระหว่าง OSPF กับ Routing Domain อื่นที่ไม่ใช่ OSPF** — Router ตัวเดียวสามารถเป็นทั้ง ABR
> และ ASBR พร้อมกันได้ ถ้าทำทั้งสองหน้าที่

---

## Step 113 — OSPF LSA Type ทั้งหมดโดยละเอียด

**LSA (Link-State Advertisement)** คือหน่วยข้อมูลที่ OSPF ใช้อธิบาย Topology แล้ว Flood แจกจ่าย
ให้ Router อื่นสร้าง LSDB ที่เหมือนกันทุกตัวภายใน Area เดียวกัน — Multi-Area OSPF มี LSA
7 Type (Type 6 คือ MOSPF ที่เลิกใช้แล้ว ไม่รวมในที่นี้) แต่ Type ที่ต้องรู้จริงมี 6 Type ดังนี้

### ตารางสรุป LSA Type ทั้งหมด — **ต้องจำตารางนี้ให้ขึ้นใจสำหรับข้อสอบ CCNP ENCOR**

| Type | ชื่อ | ใครเป็นผู้สร้าง (Originator) | ขอบเขตการ Flood (Scope) | หน้าที่ |
|---|---|---|---|---|
| **1** | Router LSA | **ทุก Router** (สร้างของตัวเองเสมอ) | ภายใน Area เดียวกันเท่านั้น | อธิบาย Interface ทุกใบของ Router นั้น + Link/Cost ที่ต่อออกไป |
| **2** | Network LSA | **DR (Designated Router)** เท่านั้น | ภายใน Area เดียวกันเท่านั้น | อธิบาย Router ทุกตัวที่อยู่บน Multi-access Segment เดียวกัน (Broadcast/NBMA) |
| **3** | Summary (Inter-Area) LSA | **ABR** เท่านั้น | ข้าม Area ได้ (Area 0 ↔ Area อื่น) | สรุป Subnet ที่อยู่ใน Area หนึ่ง ให้ Area อื่นรู้จัก โดยไม่ต้องรู้รายละเอียด Topology ภายใน |
| **4** | ASBR Summary LSA | **ABR** เท่านั้น | ข้าม Area ได้ | บอกว่า "ASBR ตัวนี้อยู่ตรงไหน (Reachability) และ Cost เท่าไหร่จะไปถึง" ให้ Area อื่นที่ไม่มี ASBR รู้จัก |
| **5** | External (AS-External) LSA | **ASBR** เท่านั้น | ทั้ง AS (ทุก Area ยกเว้น Stub/NSSA) | ประกาศ Route จาก Routing Domain นอก OSPF (Redistributed Route) |
| **7** | NSSA External LSA | **ASBR ภายใน NSSA Area** เท่านั้น | ภายใน NSSA Area นั้นเท่านั้น (ABR แปลงเป็น Type 5 ก่อนส่งออก) | เหมือน Type 5 แต่ใช้เฉพาะภายใน NSSA (ดู Step 116) |

### 113.1 Type 1 — Router LSA (รายละเอียดที่สุด)

Router ทุกตัวสร้าง Router LSA ของตัวเองเสมอ (ไม่มีข้อยกเว้น) โดยระบุ:
- Router ID ของตัวเอง
- รายการ Interface ทั้งหมดที่ Run OSPF พร้อม Link Type (Point-to-Point, Transit, Stub Network)
- Cost ของแต่ละ Interface
- Neighbor ที่ Adjacency สมบูรณ์แล้ว (Full State) บน Interface นั้น

```
CORE-SW1# show ip ospf database router

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Router Link States (Area 1)

  LS age: 45
  Options: (No TOS-capability, DC)
  LS Type: Router Links
  Link State ID: 1.1.1.1
  Advertising Router: 1.1.1.1
  LS Seq Number: 8000002A
  Checksum: 0x4F2E
  Length: 60
  Number of Links: 2

    Link connected to: another Router (point-to-point)
     (Link ID) Neighboring Router ID: 1.1.1.11
     (Link Data) Router Interface address: 10.255.10.1
      Number of TOS metrics: 0
       TOS 0 Metrics: 1

    Link connected to: another Router (point-to-point)
     (Link ID) Neighboring Router ID: 1.1.1.12
     (Link Data) Router Interface address: 10.255.20.1
      Number of TOS metrics: 0
       TOS 0 Metrics: 1
```

> สังเกตว่า Router LSA ของ CORE-SW1 **แยกกันคนละชุดในแต่ละ Area** (ชุดหนึ่งอยู่ Area 1 อธิบาย
> Te1/0/1+Te1/0/2, อีกชุดอยู่ Area 0 อธิบาย Port-channel1) — นี่คือผลจากการเป็น ABR ตรงตาม
> Step 112

### 113.2 Type 2 — Network LSA (สร้างโดย DR เท่านั้น)

Network LSA เกิดเฉพาะบน **Multi-access Segment** (Broadcast เช่น Ethernet VLAN, หรือ NBMA)
ที่มี Router มากกว่า 1 ตัวอยู่ร่วมกันและต้องเลือก **DR/BDR** (ทวนจาก Part 11) — **DR เท่านั้น**
ที่จะสร้าง Network LSA เพื่ออธิบายว่า "Router ตัวใดบ้างอยู่บน Segment นี้" ในหลักสูตรนี้ Link
ระหว่าง CORE↔DIST และ CORE↔CORE เป็น **Point-to-Point ล้วน** (Routed Port/Port-channel
แบบไม่มี Broadcast Segment ที่มี Router 3+ ตัว) จึง**ไม่มี DR Election และไม่มี Type 2 LSA**
เกิดขึ้นเลยใน Backbone Link ของ Lab — Type 2 จะเกิดก็ต่อเมื่อมี Router 2+ ตัวแชร์ VLAN/Segment
เดียวกันแบบ Multi-access (เช่นถ้ามี Router สำรองหลายตัวอยู่ VLAN เดียวกันที่ Access Layer)

```
! ตัวอย่าง Type 2 LSA (สมมติมี Router 3 ตัวอยู่ VLAN เดียวกัน เพื่อสาธิต concept)
Router# show ip ospf database network

                Net Link States (Area 0)

  LS age: 120
  LS Type: Network Links
  Link State ID: 10.10.50.1 (Address of Designated Router)
  Advertising Router: 1.1.1.1        <- DR เป็นผู้สร้าง LSA นี้
  Network Mask: /24
     Attached Router: 1.1.1.1
     Attached Router: 1.1.1.2
     Attached Router: 1.1.1.3
```

### 113.3 Type 3 — Summary (Inter-Area) LSA — หัวใจของ Multi-Area

Type 3 คือ LSA ที่ทำให้ Multi-Area ทำงานได้จริง — **ABR แปลง Route ภายใน Area (Intra-area,
มาจาก Type 1/2 + SPF)** ให้กลายเป็น **1 บรรทัดสรุป (Prefix + Mask + Cost)** แล้ว Flood เข้า
Area อื่น โดย **ไม่ส่งรายละเอียด Topology ภายในไปด้วยเลย** — นี่คือกลไกที่ทำให้ Router ใน
Area 2 (เช่น DIST-SW3) **ไม่ต้องรู้ว่า Area 1 มี Router กี่ตัว ต่อกันอย่างไร** รู้แค่ "จะไป
10.10.10.0/24 ให้ผ่าน CORE-SW2 ด้วย Cost เท่านี้" ก็เพียงพอ

```
CORE-SW1# show ip ospf database summary

                Summary Net Link States (Area 0)

  LS age: 30
  LS Type: Summary Links
  Link State ID: 10.10.10.0 (summary Network Number)
  Advertising Router: 1.1.1.1          <- CORE-SW1 (ABR) เป็นผู้สร้าง
  Network Mask: /24
        TOS: 0   Metric: 2
```

### 113.4 Type 4 — ASBR Summary LSA

เมื่อมี ASBR อยู่ใน Area หนึ่ง แต่ Router ใน Area อื่นต้องรู้ "ASBR ตัวนี้อยู่ที่ไหน" เพื่อจะ
Route ไปหา External Route (Type 5) ให้ถูกทาง — **ABR ของ Area ที่ ASBR อยู่** จะสร้าง Type 4
บอก Reachability ไปยัง ASBR (ไม่ใช่บอก Subnet แบบ Type 3 แต่บอก **Router ID ของ ASBR + Cost
ไปถึง**) แล้ว Flood เข้า Area 0 และ Area อื่นๆ ต่อไป

```
Router# show ip ospf database asbr-summary

                Summary ASB Link States (Area 0)

  LS Type: Summary Links(AS Boundary Router)
  Link State ID: 1.1.1.21 (AS Boundary Router address)     <- Router ID ของ ASBR (WAN-EDGE-1)
  Advertising Router: 1.1.1.1
        TOS: 0   Metric: 5
```

> **ข้อสังเกต**: Type 3 มี Link State ID เป็น **Subnet Address**, ส่วน Type 4 มี Link State ID
> เป็น **Router ID ของ ASBR** — นี่คือวิธีแยกสองอย่างนี้ให้ชัดเจนตอนอ่าน `show ip ospf database`

### 113.5 Type 5 — External (AS-External) LSA

Type 5 คือ Route ที่มาจาก**นอก OSPF โดยสิ้นเชิง** — เกิดจาก ASBR ทำ `redistribute` (เช่น
Static Route ไป Internet, EIGRP, BGP) แล้วดัน Route เหล่านั้นเข้า OSPF Domain — **Flood ไป
ทั้ง AS ทุก Area** (ยกเว้น Area ที่ตั้งเป็น Stub/NSSA ซึ่งจะ Block Type 5 ไม่ให้เข้า ดู Step 116)
Type 5 มี Metric Type ให้เลือก 2 แบบคือ **E1** และ **E2** (รายละเอียดเต็มใน Step 118)

```
WAN-EDGE-1# show ip ospf database external

                Type-5 AS External Link States

  LS Type: AS External Link
  Link State ID: 0.0.0.0 (External Network Number)     <- ตัวอย่าง Default Route ไป Internet
  Advertising Router: 1.1.1.21
  Metric Type: 2 (Larger than any link state path)      <- นี่คือ E2
        Metric: 1
        Forward Address: 0.0.0.0
```

### 113.6 Type 7 — NSSA External LSA

Type 7 มีโครงสร้างเหมือน Type 5 ทุกประการ แต่ใช้เฉพาะภายใน **NSSA Area** (Step 116) เท่านั้น
เพราะกฎของ Stub/NSSA คือ**ห้าม Type 5 เข้ามาใน Area นั้นเด็ดขาด** — ถ้า Area นั้นมี ASBR ของ
ตัวเองที่ต้องประกาศ External Route จริงๆ (เช่น Site ปลายทางมี Internet Breakout ของตัวเอง)
ก็ต้องใช้ Type 7 แทน แล้วให้ **ABR ของ NSSA Area นั้นแปลง Type 7 → Type 5** ก่อนส่งออกไปยัง
Area อื่นที่ไม่ใช่ NSSA

```
                    NSSA Area                    Area 0                  Area อื่น
   ASBR ──[Type 7]──► ABR (แปลง 7→5) ──[Type 5]──► ทุก Area ที่ไม่ใช่ Stub/NSSA
```

---

## Step 114 — Reconfigure Lab เป็น 3-Area Design

ถึงเวลาลงมือจริง — เราจะเปลี่ยน Design จาก Single-Area (Area 0 ทั้งหมด จาก Part 11) เป็น
**3-Area** ตามที่วางไว้ใน Step 111.5 หลักการคือ: **Interface ไหนอยู่ฝั่งไหนของ Topology ก็ต้อง
เปลี่ยน Area ให้ตรงกับฝั่งนั้น** โดย **Port-channel1 (CORE↔CORE) เท่านั้นที่ยังอยู่ Area 0**

### 114.1 วิธี Config Area ต่อ Interface — แนะนำสำหรับ Multi-Area (แม่นยำกว่า `network` Wildcard)

Part 11 อาจใช้คำสั่ง `network <ip> <wildcard> area 0` ภายใต้ `router ospf 1` ซึ่งใช้ได้ดีตอน
เป็น Area เดียว แต่พอต้องแยก Area ตาม Interface การคำนวณ Wildcard Mask ให้ตรงเป๊ะทีละ Interface
เสี่ยง Error ง่าย **CCNP แนะนำให้ใช้คำสั่งต่อ Interface โดยตรงแทน**:

```
Router(config-if)# ip ospf <process-id> area <area-id>
```

คำสั่งนี้ผูก OSPF เข้ากับ Interface ตรงๆ ไม่ต้องพึ่ง Wildcard Mask เลย และ **override** ค่าที่มา
จาก `network` statement ถ้ามีการตั้งไว้ทั้งสองแบบพร้อมกัน (Interface-level มีสิทธิ์เหนือกว่า)
เราจะใช้วิธีนี้ตลอด Part นี้เพื่อความชัดเจนและลดความเสี่ยง Human Error

### 114.2 CORE-SW1 — เปลี่ยนจาก Area 0 ล้วน เป็น ABR (Area 0 + Area 1)

```
CORE-SW1# configure terminal

! ล้าง network statement เดิมของ Part 11 ที่ยัดทุก Interface ไว้ areea 0 (ถ้ามี)
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# no network 10.255.0.0 0.0.0.3 area 0
CORE-SW1(config-router)# no network 10.255.10.0 0.0.0.3 area 0
CORE-SW1(config-router)# no network 10.255.20.0 0.0.0.3 area 0
CORE-SW1(config-router)# exit

! Port-channel1 (CORE-SW1 ↔ CORE-SW2) ---> ยังคงเป็น Area 0 เหมือนเดิม
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# ip ospf 1 area 0
CORE-SW1(config-if)# exit

! Te1/0/1 (ลง DIST-SW1) ---> เปลี่ยนจาก Area 0 เป็น Area 1
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# ip ospf 1 area 1
CORE-SW1(config-if)# exit

! Te1/0/2 (ลง DIST-SW2) ---> เปลี่ยนจาก Area 0 เป็น Area 1
CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# ip ospf 1 area 1
CORE-SW1(config-if)# exit

CORE-SW1(config)# end
CORE-SW1# copy running-config startup-config
```

### 114.3 CORE-SW2 — เปลี่ยนจาก Area 0 ล้วน เป็น ABR (Area 0 + Area 2)

```
CORE-SW2# configure terminal
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# no network 10.255.0.0 0.0.0.3 area 0
CORE-SW2(config-router)# no network 10.255.30.0 0.0.0.3 area 0
CORE-SW2(config-router)# no network 10.255.40.0 0.0.0.3 area 0
CORE-SW2(config-router)# exit

CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# ip ospf 1 area 0
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/1
CORE-SW2(config-if)# ip ospf 1 area 2
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/2
CORE-SW2(config-if)# ip ospf 1 area 2
CORE-SW2(config-if)# exit

CORE-SW2(config)# end
CORE-SW2# copy running-config startup-config
```

### 114.4 DIST-SW1 และ DIST-SW2 — ย้ายทุก Interface (VLAN SVI + Uplink) เข้า Area 1

DIST-SW1/2 เป็น **Internal Router ของ Area 1** ล้วน — ทุก Interface ที่ Run OSPF (VLAN10,
VLAN20, VLAN99, และ Uplink ไป CORE-SW1) ต้องอยู่ Area 1 เหมือนกันหมด (ไม่มี Interface ใด
เหลืออยู่ Area 0 อีกแล้ว):

```
! ===== DIST-SW1 =====
DIST-SW1# configure terminal
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# no network 10.10.10.0 0.0.0.255 area 0
DIST-SW1(config-router)# no network 10.10.20.0 0.0.0.255 area 0
DIST-SW1(config-router)# no network 10.10.99.0 0.0.0.255 area 0
DIST-SW1(config-router)# no network 10.255.10.0 0.0.0.3 area 0
DIST-SW1(config-router)# exit

DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip ospf 1 area 1
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan20
DIST-SW1(config-if)# ip ospf 1 area 1
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan99
DIST-SW1(config-if)# ip ospf 1 area 1
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# ip ospf 1 area 1
DIST-SW1(config-if)# exit
DIST-SW1(config)# end
DIST-SW1# copy running-config startup-config

! ===== DIST-SW2 (โครงสร้างเดียวกัน) =====
DIST-SW2# configure terminal
DIST-SW2(config)# router ospf 1
DIST-SW2(config-router)# no network 10.10.10.0 0.0.0.255 area 0
DIST-SW2(config-router)# no network 10.10.20.0 0.0.0.255 area 0
DIST-SW2(config-router)# no network 10.10.99.0 0.0.0.255 area 0
DIST-SW2(config-router)# no network 10.255.20.0 0.0.0.3 area 0
DIST-SW2(config-router)# exit

DIST-SW2(config)# interface Vlan10
DIST-SW2(config-if)# ip ospf 1 area 1
DIST-SW2(config-if)# exit
DIST-SW2(config)# interface Vlan20
DIST-SW2(config-if)# ip ospf 1 area 1
DIST-SW2(config-if)# exit
DIST-SW2(config)# interface Vlan99
DIST-SW2(config-if)# ip ospf 1 area 1
DIST-SW2(config-if)# exit
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# ip ospf 1 area 1
DIST-SW2(config-if)# exit
DIST-SW2(config)# end
DIST-SW2# copy running-config startup-config
```

### 114.5 DIST-SW3 และ DIST-SW4 — ย้ายทุก Interface เข้า Area 2

```
! ===== DIST-SW3 =====
DIST-SW3# configure terminal
DIST-SW3(config)# router ospf 1
DIST-SW3(config-router)# no network 10.10.30.0 0.0.0.255 area 0
DIST-SW3(config-router)# no network 10.10.40.0 0.0.0.255 area 0
DIST-SW3(config-router)# no network 10.10.99.0 0.0.0.255 area 0
DIST-SW3(config-router)# no network 10.255.30.0 0.0.0.3 area 0
DIST-SW3(config-router)# exit

DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ip ospf 1 area 2
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface Vlan40
DIST-SW3(config-if)# ip ospf 1 area 2
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface Vlan99
DIST-SW3(config-if)# ip ospf 1 area 2
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface TenGigabitEthernet1/1/1
DIST-SW3(config-if)# ip ospf 1 area 2
DIST-SW3(config-if)# exit
DIST-SW3(config)# end
DIST-SW3# copy running-config startup-config

! ===== DIST-SW4 (โครงสร้างเดียวกัน) =====
DIST-SW4# configure terminal
DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# no network 10.10.30.0 0.0.0.255 area 0
DIST-SW4(config-router)# no network 10.10.40.0 0.0.0.255 area 0
DIST-SW4(config-router)# no network 10.10.99.0 0.0.0.255 area 0
DIST-SW4(config-router)# no network 10.255.40.0 0.0.0.3 area 0
DIST-SW4(config-router)# exit

DIST-SW4(config)# interface Vlan30
DIST-SW4(config-if)# ip ospf 1 area 2
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface Vlan40
DIST-SW4(config-if)# ip ospf 1 area 2
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface Vlan99
DIST-SW4(config-if)# ip ospf 1 area 2
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface TenGigabitEthernet1/1/1
DIST-SW4(config-if)# ip ospf 1 area 2
DIST-SW4(config-if)# exit
DIST-SW4(config)# end
DIST-SW4# copy running-config startup-config
```

### 114.6 Verify ทันทีหลัง Reconfigure

```
CORE-SW1# show ip ospf interface brief
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Po1          1     0               10.255.0.1/30      1     P2P   1/1
Te1/0/1      1     1               10.255.10.1/30     1     P2P   1/1
Te1/0/2      1     1               10.255.20.1/30     1     P2P   1/1

CORE-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:38    10.255.0.2      Port-channel1
1.1.1.11          0   FULL/  -        00:00:32    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:35    10.255.20.2     TenGigabitEthernet1/0/2
```

> **สังเกต State `FULL/  -`**: บน Point-to-Point Link ไม่มี DR/BDR Election เลย (ทวนจาก
> Part 11) ดังนั้นช่อง Pri/State จะไม่แสดง DR/BDR/DROTHER เหมือน Broadcast Segment

---

## Step 115 — Inter-Area Route Summarization บน ABR

### 115.1 Syntax และแนวคิด

```
Router(config-router)# area <area-id> range <network> <mask> [advertise | not-advertise] [cost <value>]
```

คำสั่งนี้ Config ใต้ `router ospf` บน **ABR เท่านั้น** (ไม่มีผลถ้าใส่บน Internal Router)
โดยระบุ **Area ต้นทาง** ที่มี Subnet ย่อยๆ อยู่ แล้วบอกว่า "รวมทุก Subnet ที่อยู่ใน Range นี้
เป็น 1 Type 3 LSA เดียว" — ผลลัพธ์คือ **จำนวน Type 3 LSA ที่ Flood เข้า Area อื่นลดลง**
ทำให้ LSDB เล็กลง, SPF เร็วขึ้น, และเวลา Subnet ย่อยเปลี่ยน (Flap) **จะไม่กระทบ Area อื่นเลย**
ตราบใดที่ยัง Summarize อยู่ในขอบเขตเดิม (เพราะ Area อื่นเห็นแค่ Summary Route เดียว ไม่รู้ว่า
ข้างในมี Subnet ไหน Flap)

> **สิ่งที่ `area range` ทำไม่ได้**: สรุปเฉพาะ Route ที่เป็น **Intra-area ของ Area ที่ระบุ**
> เท่านั้น (ไม่ใช่สรุป External/Type 5 — นั่นเป็นหน้าที่ของ `summary-address` ซึ่งใช้บน ASBR
> คนละคำสั่งกัน อย่าสับสน)

### 115.2 Worked Example — CORE-SW1 (ABR ของ Area 1) สรุป VLAN 10/20/99

ก่อน Summarize, CORE-SW1 (ในฐานะ ABR) จะสร้าง Type 3 LSA แยกทีละ Subnet เข้า Area 0:

```
CORE-SW1# show ip ospf database summary

                Summary Net Link States (Area 0)

Link State ID   ADV Router      Age    Seq#       Checksum
10.10.10.0      1.1.1.1         120    0x8000001  0x00A1
10.10.20.0      1.1.1.1         120    0x8000001  0x00B2
10.10.99.0      1.1.1.1         120    0x8000001  0x00C3
```

VLAN10 (10.10.10.0/24), VLAN20 (10.10.20.0/24) และ VLAN99 (10.10.99.0/24) **ไม่ได้อยู่ในขอบเขต
CIDR ที่ต่อเนื่องแบบแคบๆ** (Third Octet คือ 10, 20, 99 ซึ่งไม่มี Boundary เล็กกว่า /16 ที่
ครอบทั้งสามค่านี้พร้อมกันได้) — นี่คือผลจากการที่ IP Plan ของ Lab หลักสูตรนี้ (Part 1, `00-ip-
address-plan.md`) **ไม่ได้ถูกออกแบบมาเพื่อ Summarization ตั้งแต่แรก** (ทวนจากบทเรียนใน Part 7)
ในโลกจริง นี่คือเหตุผลสำคัญที่สุดข้อหนึ่งที่ต้อง**วางแผน Address Block ต่อ Site/Area ให้เป็น
CIDR ก้อนใหญ่ต่อเนื่องกันตั้งแต่ Day 1** เพื่อให้ Summarize ได้สะดวกในอนาคต — สำหรับ Lab นี้
เราจะสาธิตด้วย Range ที่กว้างพอจะครอบทั้งสาม Subnet คือ **10.10.0.0/16**:

```
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# area 1 range 10.10.0.0 255.255.0.0
CORE-SW1(config-router)# end
CORE-SW1# copy running-config startup-config
```

หลัง Summarize — CORE-SW1 จะสร้าง Type 3 LSA **เพียง 1 บรรทัด** แทน 3 Subnet เดิม และ
**Suppress (ระงับ)** ไม่ Flood Type 3 ของ 10.10.10.0/24, 10.10.20.0/24, 10.10.99.0/24 แยก
ทีละใบเข้า Area 0 อีกต่อไป:

```
CORE-SW1# show ip ospf database summary

                Summary Net Link States (Area 0)

Link State ID   ADV Router      Age    Seq#       Checksum
10.10.0.0       1.1.1.1         12     0x8000002  0x00D4      <- 1 บรรทัดแทนที่ 3 บรรทัดเดิม

CORE-SW1# show ip ospf summary-address
OSPF Process 1, Summary-address
  10.10.0.0/16    Metric -1, Type: Summary, Mask: 255.255.0.0
                  Advertised, Ranges Configured 1, Discard Route Not configured
```

> **ข้อควรระวังสำคัญที่ต้องพูดถึงตรงๆ**: VLAN 99 (MGMT, 10.10.99.0/24) ใน Lab นี้ **ถูกใช้
> Subnet เดียวกันซ้ำทั้งฝั่ง Area 1 (DIST-SW1/2) และฝั่ง Area 2 (DIST-SW3/4)** ตามที่ Part 10
> ได้ระบุไว้ชัดเจนว่าเป็นข้อจำกัดของ Design (ปกติแก้ด้วย Static Host Route แบบ /31 ที่เจาะจงมาก
> ซึ่งใช้ได้กับ Static Routing เท่านั้น) — เมื่อเปลี่ยนมาเป็น OSPF ทั้ง CORE-SW1 (ตัวแทน Area 1)
> และ CORE-SW2 (ตัวแทน Area 2) จะ Originate Type 3 LSA สำหรับ 10.10.99.0/24 เข้า Area 0
> **พร้อมกันจากสองฝั่ง** ซึ่งทาง OSPF มองว่าเป็น Route ที่ถูกต้องทั้งคู่ (คนละ Advertising
> Router) แต่ในทางกายภาพเป็นคนละ Subnet ที่ทับ Address กัน — ผลคือ Router ฝั่งตรงข้ามอาจเลือก
> เส้นทางไปยัง Host MGMT ผิดฝั่งได้ในบางกรณี **นี่คือบทเรียนสำคัญของ CCNP**: การ Summarize
> แบบกว้าง (Broad Range) ต้องมั่นใจว่า Address Space ภายใน Range นั้น **ไม่มีการซ้ำกันข้าม
> Administrative Boundary** — วิธีแก้ที่ถูกต้องในระบบจริงคือแยก MGMT Network ออกเป็น
> Out-of-Band Network ต่างหาก (ตามที่ Part 17 จะสอน) ไม่ใช่ใช้ VLAN เดียวกันข้าม Site

### 115.3 การใช้ `not-advertise` — สรุปแบบ "ซ่อนไปเลย"

ถ้า Subnet ภายใน Area ไม่มีความจำเป็นต้องให้ Area อื่นรู้จักเลย (เช่น Point-to-Point Link
ภายใน Area ที่ไม่มี Host จริงใช้งาน) สามารถใช้ `not-advertise` เพื่อ**สรุปแล้วไม่ส่งออกเลย**
(ต่างจาก Default ที่สรุปแล้วยังส่ง Summary LSA 1 บรรทัดออกไป):

```
CORE-SW1(config-router)# area 1 range 10.255.10.0 255.255.255.252 not-advertise
CORE-SW1(config-router)# area 1 range 10.255.20.0 255.255.255.252 not-advertise
```

ผลคือ Subnet /30 ของ Link CORE-SW1↔DIST-SW1 และ CORE-SW1↔DIST-SW2 จะ**ไม่ปรากฏใน Routing
Table ของ Area อื่นเลย** — ลด Route ที่ไม่จำเป็นออกจาก Backbone ได้อีกชั้นหนึ่ง (แนวคิดคล้าย
`passive-interface` แต่ทำงานที่ระดับ LSA Advertisement ไม่ใช่ระดับ Adjacency)

---

## Step 116 — Area Type: Standard, Stub, Totally Stub, NSSA

Area ปกติที่ทำมาทั้งหมดใน Part นี้เรียกว่า **Standard Area** — รับ Type 1,2,3,4,5 ได้ครบทุก Type
Cisco มี Area Type พิเศษอีก 3 แบบ ที่ **จำกัดชนิด LSA ที่ยอมให้เข้า Area** เพื่อลด LSDB
ให้เล็กลงไปอีกในกรณีที่ Area นั้นเป็น "ปลายทาง" ไม่ใช่ Transit

### ตารางเปรียบเทียบ Area Type ทั้งหมด

| Area Type | รับ Type 3 (Inter-area) | รับ Type 5 (External) | Default Route อัตโนมัติ | คำสั่ง Config |
|---|---|---|---|---|
| **Standard** | ✅ รับ | ✅ รับ | ❌ ไม่มี | (ค่าปกติ ไม่ต้อง config) |
| **Stub** | ✅ รับ | ❌ Block ทั้งหมด | ✅ ABR ฉีด Default Route (`0.0.0.0`) แทน | `area <id> stub` |
| **Totally Stub** (Cisco Proprietary) | ❌ Block ด้วย (เหลือแค่ Intra-area) | ❌ Block ทั้งหมด | ✅ ABR ฉีด Default Route แทน | `area <id> stub` (ที่ ABR เติม `no-summary`) |
| **NSSA** | ✅ รับ | ❌ Block Type 5 แต่**อนุญาต ASBR ภายใน NSSA เอง**สร้าง Type 7 ได้ | ✅ (ตัวเลือก, ต้องสั่ง `default-information-originate` เพิ่ม) | `area <id> nssa` |
| **Totally NSSA** | ❌ Block | ❌ Block Type 5 (Type 7 ภายในยังได้) | ✅ | `area <id> nssa no-summary` |

### 116.1 ทำไมต้องมี Stub Area

Stub Area เหมาะกับ Area ที่เป็น **"ปลายทางจริงๆ"** ไม่มี Path อื่นออกไปนอกเหนือจาก ABR ตัวเดียว
(ไม่ใช่ Transit Area ที่มี Traffic วิ่งผ่าน) — ในกรณีนี้ Router ภายใน Area ไม่มีความจำเป็นต้องรู้
รายละเอียด External Route (Type 5) ทีละเส้นเลย เพราะไม่ว่าจะไปปลายทางไหนที่ไม่ใช่ในนี้ ก็ต้อง
ออกทาง ABR ตัวเดียวอยู่ดี — **ใช้ Default Route (0.0.0.0/0) แทนได้เลย** ลด LSDB ลงไปมาก

```
Router(config)# router ospf 1
Router(config-router)# area 2 stub                    ! Stub Area ธรรมดา — ยัง block เฉพาะ Type 5
```

### 116.2 Totally Stub — เข้มกว่า Stub อีกขั้น (Cisco Proprietary เท่านั้น)

Totally Stub บล็อกทั้ง **Type 3 (Inter-area)** และ **Type 5 (External)** เหลือไว้แค่ Intra-area
(Type 1/2) ของตัวเอง + Default Route เดียวจาก ABR — ใช้คำสั่งเดียวกับ Stub แต่เพิ่ม `no-summary`
**ที่ฝั่ง ABR เท่านั้น** (Internal Router ไม่ต้องรู้ว่าเป็น Totally Stub เลย มันจะทำงานเหมือน
Stub ปกติจากมุมมองของมันเอง):

```
! สั่งที่ ABR (เช่น CORE-SW2 ถ้าจะทำ Area 2 เป็น Totally Stub)
CORE-SW2(config-router)# area 2 stub no-summary
```

### 116.3 NSSA — ทางเลือกเมื่อ Area ต้องมี ASBR ของตัวเอง

ปัญหาของ Stub/Totally Stub คือ **ห้าม Router ภายใน Area นั้นเป็น ASBR เด็ดขาด** (เพราะ Stub
Block Type 5 ทั้ง**ทางเข้าและทางออก**) — ถ้า Site ปลายทางมี Internet Breakout ของตัวเอง หรือ
ต้อง Redistribute Static Route ของ Local Site เข้า OSPF แต่ก็อยากได้ประโยชน์ของ Stub
(LSDB เล็ก ไม่ต้องรับ External จาก Area อื่นทั้งหมด) ให้ใช้ **NSSA (Not-So-Stubby Area)** แทน:

```
Router(config-router)# area 2 nssa
```

NSSA ยัง Block Type 5 จาก Area อื่น**ไม่ให้เข้ามา** เหมือน Stub แต่ **อนุญาตให้ ASBR ที่อยู่
ภายใน NSSA เอง** สร้าง **Type 7** ได้ (ตามที่อธิบายไว้ Step 113.6) แล้ว ABR ของ NSSA Area นั้น
จะแปลง Type 7 → Type 5 ก่อนส่งออกไปยัง Area อื่น

### 116.4 เมื่อไหร่ Lab นี้ควรใช้ Area Type พิเศษ

Area 1 และ Area 2 ของ Lab หลักสูตรนี้**ยังไม่เหมาะเป็น Stub/NSSA** ในขั้นตอนปัจจุบัน เพราะ
ทั้งสอง Area ยังอาจต้องมี ASBR ของตัวเองในอนาคต (เช่นถ้า DIST-SW ตัวใดต้อง Redistribute
Static Route เฉพาะกิจ) และยังเป็น Standard Area ที่ให้ Visibility เต็มรูปแบบเพื่อการเรียนรู้
— **Stub/Totally Stub เหมาะกับ Branch Office ปลายทางที่ไม่มี ASBR แน่นอน 100%** และ NSSA
เหมาะกับ Branch ที่มี Local Internet Breakout ของตัวเอง — เนื้อหานี้จึงถูกจัดเป็น "ตัวเลือก
Design" ที่ต้องเลือกให้ตรงกับสถานการณ์จริง ไม่ใช่ใช้ทุกที่โดยอัตโนมัติ

> **สรุปสำหรับข้อสอบ**: คำถามแนว "Area ปลายทางที่ไม่มี ASBR ต้องการลด LSDB ให้เล็กที่สุด"
> → **Totally Stub**. คำถามแนว "Area ปลายทางมี Internet Breakout ของตัวเองแต่ยังอยากลด LSDB"
> → **NSSA**. คำถามแนว "ต้องการ Block เฉพาะ External แต่ยังอยากเห็น Inter-area Route เต็ม"
> → **Stub ธรรมดา** (ไม่ใส่ `no-summary`)

---

## Step 117 — Virtual Link: เชื่อม Area เข้า Backbone ทางอ้อม

### 117.1 ปัญหาที่ Virtual Link แก้

ตาม Backbone Rule ใน Step 111.4 **ทุก Area ต้องแตะ Area 0 โดยตรง** — แต่ในความเป็นจริง
บางครั้ง Network ขยายตัวแบบไม่ได้วางแผนไว้ล่วงหน้า ทำให้เกิด **Area ใหม่ที่ไม่มี Physical Link
ตรงไปยัง Area 0 เลย** (มีแต่ Link ไปยัง Area อื่นที่ไม่ใช่ Area 0) — ถ้าไม่มีทางแก้ Area นั้นจะ
ขาดการเชื่อมต่อกับ Backbone และไม่สามารถแลกเปลี่ยน Inter-area Route กับ Area อื่นได้เลย

**Virtual Link** คือการสร้าง "Backbone เสมือน" ทะลุผ่าน Area ตรงกลาง (เรียกว่า **Transit Area**)
เพื่อให้ Area ปลายทางมองเห็นตัวเองว่า "เชื่อมกับ Area 0" ได้ทางตรรกะ แม้ Physical Link จริงจะ
ไม่ได้ต่อตรงก็ตาม

### 117.2 เงื่อนไขก่อนใช้ Virtual Link ได้

| เงื่อนไข | รายละเอียด |
|---|---|
| Transit Area ต้องไม่เป็น Stub | Stub Area block การทำ Transit เพราะออกแบบมาให้เป็น "ปลายทาง" เท่านั้น |
| ต้องมี Full Area Border Router 2 ตัวที่ประกาศตัวเป็นปลาย Virtual Link | ตัวหนึ่งอยู่ Area 0 จริง (ติด Backbone), อีกตัวอยู่ Area ที่ไม่มี Backbone Link |
| Router ทั้งสองต้อง Config `area <transit-area-id> virtual-link <router-id ของอีกฝั่ง>` | ต้องรู้ Router ID ของกันและกันล่วงหน้า |

### 117.3 ตัวอย่างเชิง Concept (สมมติ — ไม่ใช่ Config จริงของ Lab ปัจจุบัน)

สมมติในอนาคตมี **Area 3** ใหม่เกิดขึ้น (เช่น Site สาขาใหม่ที่ต่อพ่วงหลัง DIST-SW1 เท่านั้น
ไม่มี Physical Link ไปหา CORE-SW1/CORE-SW2 ตรง) — Area 3 จะเชื่อมกับ Area 0 ทางอ้อมผ่าน
**Area 1 เป็น Transit Area** โดยใช้ DIST-SW1 (ABR ระหว่าง Area 1 กับ Area 3) และ CORE-SW1
(ABR ระหว่าง Area 1 กับ Area 0) เป็นปลายทั้งสองข้างของ Virtual Link:

```
        Area 3                    Area 1 (Transit)              Area 0
   ┌───────────┐         ┌───────────┐         ┌───────────┐
   │  (สมมติ)   │─────────│  DIST-SW1  │═════════│ CORE-SW1  │
   └───────────┘         └───────────┘         └───────────┘
                                    ╲___________________╱
                                  Virtual Link (ทะลุผ่าน Area 1)
                                  ทำให้ Area 3 "มองเห็น" ตัวเองว่าติด Area 0
```

```
! ฝั่ง DIST-SW1 (ABR ระหว่าง Area 1 และ Area 3 สมมติ)
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# area 1 virtual-link 1.1.1.1     ! 1.1.1.1 = Router ID ของ CORE-SW1

! ฝั่ง CORE-SW1 (ABR ระหว่าง Area 1 และ Area 0 จริง)
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# area 1 virtual-link 1.1.1.11    ! 1.1.1.11 = Router ID ของ DIST-SW1
```

Area-id ที่ใส่ในคำสั่ง (`area 1`) คือ **Transit Area ID** (Area 1 ที่ทั้งสองฝั่งใช้ร่วมกัน)
ไม่ใช่ Area 3 หรือ Area 0 — Router ID ที่ใส่คือ **Router ID ของอีกฝั่ง** เสมอ (Mutual Config)

> **ในสถาปัตยกรรมปัจจุบันของ Lab หลักสูตรนี้ ไม่จำเป็นต้องใช้ Virtual Link เลย** เพราะทั้ง
> Area 1 (ผ่าน CORE-SW1) และ Area 2 (ผ่าน CORE-SW2) มี ABR ที่ต่อ Area 0 โดยตรงอยู่แล้วครบ
> — เนื้อหา Step นี้เตรียมไว้สำหรับสถานการณ์ Network ขยายตัวในอนาคต ซึ่งเป็นสถานการณ์จริงที่
> พบได้บ่อยเวลา Merge หลาย Network เข้าด้วยกัน (M&A) หรือต่อ Site ใหม่แบบเร่งด่วน

---

## Step 118 — OSPF Route Type และการคำนวณ Cost ข้าม Area

### 118.1 ประเภท Route ใน `show ip route` ที่มาจาก OSPF

| Code | ชื่อเต็ม | มาจาก LSA Type | ความหมาย |
|---|---|---|---|
| **O** | Intra-area | Type 1 + 2 (ผ่าน SPF) | ปลายทางอยู่ **ใน Area เดียวกัน** กับ Router ที่ดู |
| **O IA** | Inter-area | Type 3 | ปลายทางอยู่ **คนละ Area** — เรียนรู้ผ่าน ABR ที่สร้าง Summary LSA |
| **O E1** | External Type 1 | Type 5 (Metric Type 1) | Route จากนอก OSPF, **Cost = External Cost + Internal Cost สะสมจนถึง ASBR** |
| **O E2** | External Type 2 | Type 5 (Metric Type 2, **ค่า Default**) | Route จากนอก OSPF, **Cost = External Cost เท่านั้น** (ไม่บวก Internal Cost เลย) |
| **O N1 / O N2** | NSSA External Type 1/2 | Type 7 | เหมือน E1/E2 แต่มาจาก NSSA Area (Step 116) |

### 118.2 กฎการเลือก Best Path เมื่อมีหลาย Route Type แข่งกัน — **ลำดับ Priority ตายตัว**

```
Priority สูง → ต่ำ:
   1. O   (Intra-area)      ชนะเสมอ ไม่สนใจ Cost เลยถ้ามี Intra-area ให้เลือก
   2. O IA (Inter-area)
   3. O E1 / O N1            (เทียบ Cost กับ Intra-area ปกติได้ เพราะรวม internal cost แล้ว)
   4. O E2 / O N2            (เทียบ Cost กันเองในกลุ่ม E2 เท่านั้น ไม่เทียบกับ Internal Cost)
```

> **กฎเหล็ก**: แม้ O IA จะมี Cost ต่ำกว่า O แบบผิดปกติ (ซึ่งไม่ควรเกิดถ้า Design ถูกต้อง)
> **OSPF จะเลือก O (Intra-area) เสมอ** เพราะ Path Type มี Priority เหนือ Cost ทุกกรณี — Cost
> ใช้เทียบกันเฉพาะ**ภายใน Path Type เดียวกัน** เท่านั้น (เช่น เทียบ O กับ O ด้วยกัน)

### 118.3 Worked Example — คำนวณ Cost จริงในสถาปัตยกรรม 3-Area ของ Lab

Cost ของ OSPF คำนวณจาก **Reference Bandwidth (Default 100 Mbps) หารด้วย Interface Bandwidth**
`Cost = Reference-Bandwidth(Mbps) / Interface-Bandwidth(Mbps)` (ค่าต่ำสุดคือ 1 เสมอ) — สังเกตว่า
Interface ในหลักสูตรนี้เป็น **GigabitEthernet (SVI)** และ **TenGigabitEthernet (Uplink)** ซึ่งเร็ว
กว่า Reference Bandwidth Default มาก ผลคือ **ทั้งสองความเร็วถูกปัดเป็น Cost = 1 เท่ากันหมด**
(100/1000 = 0.1 → ปัด 1, 100/10000 = 0.01 → ปัด 1) — นี่คือข้อจำกัดจริงที่พบได้บ่อยเมื่อ Link
ความเร็วสูงกว่า 100 Mbps ทั้งหมด **Cost จะไม่สะท้อนความต่างของ Bandwidth เลย**

**วิธีแก้**: ปรับ `auto-cost reference-bandwidth` ให้สูงขึ้นบน **ทุก Router ในหลักสูตรนี้เท่าๆ
กัน** (ค่านี้ต้อง Match กันทุกตัวไม่งั้น Cost จะคำนวณไม่ตรงกันและอาจเกิด Routing Loop เชิง Cost):

```
Router(config-router)# auto-cost reference-bandwidth 100000     ! หน่วย Mbps (= 100 Gbps)
% OSPF: Reference bandwidth is changed.
        Please ensure reference bandwidth is consistent across all routers.
```

หลังปรับ: `Cost = 100000 / Interface-Bandwidth(Mbps)` → GigabitEthernet (SVI) = 100000/1000
= **100**, TenGigabitEthernet (Uplink) = 100000/10000 = **10** — ตอนนี้ Cost สะท้อนความต่าง
Bandwidth ได้จริงแล้ว

**ตัวอย่างคำนวณ Cost จาก PC1 (VLAN10, DIST-SW1) ไปถึง Server1 (VLAN30, DIST-SW3):**

```
PC1 ──[SVI Vlan10, cost 100]── DIST-SW1 ──[Te1/1/1, cost 10]── CORE-SW1
                                                                    │
                                                          [Po1, cost 10]
                                                                    │
                                                               CORE-SW2
                                                                    │
                                                          [Te1/0/1, cost 10]
                                                                    │
                                                              DIST-SW3 ──[SVI Vlan30, cost 100]── Server1

Total Cost (มองจาก DIST-SW1) = 10 (uplink) + 10 (Po1) + 10 (downlink CORE-SW2→DIST-SW3) = 30
show ip route ที่ DIST-SW1 จะเห็น: O IA  10.10.30.0/24 [110/30] via 10.255.10.1, TenGigabitEthernet1/1/1
```

> ตัวเลข **[110/30]** ใน `show ip route` — `110` คือ **Administrative Distance ของ OSPF**
> (ค่า Default, ทวนจาก Part 9) ส่วน `30` คือ **OSPF Cost สะสม** ที่คำนวณได้ข้างต้น — สอง
> ตัวเลขนี้ทำหน้าที่ต่างกันเสมอ AD ใช้เทียบระหว่าง Routing Protocol ต่างชนิด ส่วน Cost/Metric
> ใช้เทียบ Path ภายใน Protocol เดียวกัน

### 118.4 Worked Example — O E1 vs O E2 (สมมติเพื่ออธิบาย Concept)

สมมติในอนาคต **WAN-EDGE-1** (ASBR) Redistribute Static Default Route (ไป ISP) เข้า OSPF ด้วย
External Cost = 20 และ Internal Cost จาก CORE-SW1 ไปถึง WAN-EDGE-1 (ผ่าน Backbone) = 11:

```
                              External Cost = 20 (กำหนดตอน redistribute)
   Internet ──[ASBR: WAN-EDGE-1]──────────────────────────────────► OSPF Domain
                     │
                     │ Internal Cost = 11 (คำนวณจาก SPF ปกติ ไปถึง WAN-EDGE-1)
                     ▼
                CORE-SW1 (มองเห็น Route นี้เป็น O E1 หรือ O E2 แล้วแต่ตั้งค่า)
```

| Metric Type | สูตรคำนวณ Cost ที่ CORE-SW1 เห็น | ผลลัพธ์ |
|---|---|---|
| **E1** (`metric-type 1`) | External Cost + Internal Cost = 20 + 11 | **31** |
| **E2** (`metric-type 2`, Default) | External Cost เท่านั้น = 20 | **20** |

> **นัยสำคัญ**: ถ้ามี ASBR สองตัว Redistribute Route ปลายทางเดียวกันเข้า OSPF จากสองจุดที่ห่างกัน
> — **E2 จะไม่สนใจว่า ASBR ตัวไหนอยู่ใกล้กว่า** (เทียบ External Cost อย่างเดียว) ในขณะที่
> **E1 จะเลือก Path ที่รวม Internal Cost แล้วใกล้กว่าจริง** — ถ้าต้องการให้ OSPF เลือก Path
> ที่ "ใกล้ที่สุดจริงในเชิง Topology" เมื่อมี ASBR ซ้ำซ้อนหลายตัว **ต้องใช้ E1** ไม่ใช้ E2
> (ค่า Default ของ Cisco คือ E2 เสมอ ต้องสั่ง `redistribute ... metric-type 1` เพื่อเปลี่ยน)

---

## Step 119 — Advanced Verification & Troubleshooting Multi-Area OSPF

### 119.1 `show ip ospf database` — อ่านสรุป LSA แยกตาม Area

```
CORE-SW1# show ip ospf database

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Link count
1.1.1.1         1.1.1.1         200         0x8000003  1
1.1.1.2         1.1.1.2         198         0x8000003  1

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Link count
1.1.1.1         1.1.1.1         200         0x8000005  2
1.1.1.11        1.1.1.11        190         0x8000004  4
1.1.1.12        1.1.1.12        188         0x8000004  4

                Summary Net Link States (Area 0)

Link ID         ADV Router      Age         Seq#
10.10.0.0       1.1.1.1         12          0x8000002        <- Summary จาก Area 1 (Step 115)
10.10.30.0      1.1.1.2         45          0x8000001        <- Summary จาก Area 2 (ยังไม่ summarize)
10.10.40.0      1.1.1.2         45          0x8000001

                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#
10.10.30.0      1.1.1.1         50          0x8000001        <- CORE-SW1 re-advertise เข้า Area 1
10.10.40.0      1.1.1.1         50          0x8000001
```

**สิ่งที่ต้องอ่านให้เป็น**: จำนวน **Router Link States (Type 1)** ต่อ Area บอกว่า Area นั้นมี
Router กี่ตัว (Area 0 มี 2 = CORE-SW1+CORE-SW2 เท่านั้น, Area 1 มี 3 = CORE-SW1+DIST-SW1+
DIST-SW2) — ส่วน **Summary Net Link States** ที่ปรากฏใน Area 0 มาจาก ABR ทั้งสองฝั่ง (CORE-SW1
สร้างจาก Area 1, CORE-SW2 สร้างจาก Area 2) แล้ว **แต่ละ ABR ก็ re-originate Summary ที่เห็นจาก
Area 0 กลับเข้า Area ของตัวเองด้วย** (เห็น Summary ของ Area 2 ปรากฏใน Area 1 ตามภาพ) — นี่คือ
พฤติกรรมมาตรฐานของ ABR ที่ทำหน้าที่ "แปลข้อมูล" ระหว่าง Area สองทาง

### 119.2 `show ip ospf border-routers` — ดูว่า ABR/ASBR ที่มองเห็นคือใคร

```
CORE-SW1# show ip ospf border-routers

OSPF Router with ID (1.1.1.1) (Process ID 1)

                Base Topology (MTID 0)

Internal Router Routing Table
Codes: i - Intra-area route, I - Inter-area route

i    1.1.1.2 [10] via 10.255.0.2, Port-channel1, ABR, Area 0, SPF 12
i    1.1.1.11 [10] via 10.255.10.2, TenGigabitEthernet1/0/1, ABR, Area 1, SPF 8
i    1.1.1.12 [10] via 10.255.20.2, TenGigabitEthernet1/0/2, ABR, Area 1, SPF 8
```

> คำสั่งนี้มีประโยชน์มากตอน Troubleshoot ว่า "OSPF มองเห็น ABR/ASBR ตัวไหนอยู่บ้าง และ Cost
> ไปถึงเท่าไหร่" — ถ้า ABR ที่ควรจะเห็นหายไปจาก List นี้ แปลว่า Backbone Connectivity มีปัญหา
> จริง (ไม่ใช่แค่ Neighbor Down แต่ Path ทั้งเส้นขาด)

### 119.3 `show ip ospf summary-address` — ตรวจสอบ Summarization ที่ Config ไว้

```
CORE-SW1# show ip ospf summary-address

OSPF Process 1, Summary-address
  10.10.0.0/16    Metric -1, Type: Summary, Mask: 255.255.0.0
                  Advertised, Ranges Configured 1, Discard Route Not configured
```

`Metric -1` หมายถึงใช้ Cost อัตโนมัติ (Cost สูงสุดของ Component Routes ที่ถูกรวม ไม่ใช่ค่าคงที่)
— ถ้า Config ด้วย `cost <value>` ชัดเจน ค่านี้จะเปลี่ยนไปตามที่ตั้งไว้แทน

### 119.4 Common Misconfig #1 — Area Mismatch ระหว่าง Neighbor (สาเหตุ Down อันดับ 1)

ถ้า Interface สองฝั่งของ Link เดียวกัน Config Area คนละเลข OSPF **จะไม่ยอมขึ้น Neighbor เลย**
(ไม่ใช่แค่ Adjacency ค้าง แต่จะไม่ Establish ตั้งแต่ Hello Packet แรก):

```
! สมมติผิดพลาด: CORE-SW1 ลืมเปลี่ยน Te1/0/1 เป็น Area 1 (ยังค้าง Area 0 จาก Part 11)
CORE-SW1# show ip ospf interface TenGigabitEthernet1/0/1 | include Area
  Process ID 1, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 10
  Area 0

! DIST-SW1 (ถูกเปลี่ยนเป็น Area 1 แล้วตามขั้นตอน Step 114.4)
DIST-SW1# show ip ospf interface TenGigabitEthernet1/1/1 | include Area
  Area 1

! ผลลัพธ์: ไม่มี Neighbor ขึ้นเลย
DIST-SW1# show ip ospf neighbor
% (ว่างเปล่า — ไม่มี Neighbor)

! Log ที่จะเห็นจริง (ต้องเปิด debug ip ospf adj เพื่อยืนยัน)
DIST-SW1# debug ip ospf adj
*Sep 26 10:15:22.101: OSPF-1 ADJ  Te1/1/1: Rcv pkt from 10.255.10.1 : Mismatch Area ID
DIST-SW1# undebug all
```

**วิธีแก้**: ตรวจ Area ID ให้ตรงกันทั้งสองฝั่งของ Link เสมอด้วย `show ip ospf interface brief`
เทียบกันทั้งคู่ — นี่คือ Checklist ข้อแรกที่ต้องเช็คทุกครั้งที่ Neighbor ไม่ขึ้นหลัง Multi-Area
Migration

### 119.5 Common Misconfig #2 — Area ที่ไม่มี Path เชื่อม Area 0 (ขาด Backbone Connectivity)

ถ้า ABR ตัวเดียวของ Area หนึ่งเกิด Link ไป Area 0 Down (เช่น Port-channel1 บน CORE-SW1 หลุด)
Area 1 ทั้งหมดจะขาดการเชื่อมต่อกับ Area 0 **ทันที** แม้ Internal Router ภายใน Area 1 จะยังคุย
กันได้ปกติทุกตัว (เพราะ Adjacency ภายใน Area ไม่ได้พึ่ง Area 0 เลย):

```
CORE-SW1# show ip ospf interface Port-channel1
% OSPF: interface Port-channel1 is down                 <- Backbone Link ขาด

CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11          0   FULL/  -        00:00:35    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:31    10.255.20.2     TenGigabitEthernet1/0/2
                                                                   <- ไม่มี Neighbor กับ CORE-SW2 เลย

! ผลกระทบ: DIST-SW1 ยัง ping DIST-SW2 ได้ (อยู่ Area 1 เดียวกัน)
! แต่ DIST-SW1 ping ไป Server1 (VLAN30, Area 2) ไม่ได้อีกต่อไป
DIST-SW1# show ip route | include O IA
                                                                   <- ไม่มี O IA เข้า Area 2 เหลืออยู่เลย
```

**วิธีแก้/ป้องกัน**: นี่คือเหตุผลที่ Production Network มักทำ **CORE-SW1 และ CORE-SW2 เชื่อมกัน
มากกว่า 1 เส้น** (EtherChannel แบบ Multi-link ตามที่ Part 6 สอน) เพื่อไม่ให้ ABR เดี่ยวกลายเป็น
Single Point of Failure ของ Backbone Connectivity ทั้ง Area — ในระบบใหญ่ระดับ Enterprise จริง
มักมี ABR มากกว่า 1 ตัวต่อ Area ด้วย (Redundant ABR) เพื่อไม่ให้ Area ทั้งใบขาดจาก Backbone
ถ้า ABR ตัวเดียวล่ม

### 119.6 Checklist Troubleshooting Multi-Area OSPF แบบเป็นลำดับ

```
1. show ip ospf interface brief          → ตรวจ Area ID ของทุก Interface ตรงตาม Design หรือไม่
2. show ip ospf neighbor                 → Neighbor ครบ และ State = FULL ทุกคู่หรือไม่
3. show ip ospf database                 → นับจำนวน Router LSA ต่อ Area ตรงกับจำนวน Router จริงไหม
4. show ip ospf border-routers           → ABR/ASBR ที่มองเห็นครบตาม Design หรือไม่
5. show ip route ospf                    → Route Type (O/O IA/O E1/O E2) ตรงตามที่คาดหวังหรือไม่
6. show ip ospf summary-address          → Summarization (ถ้ามี) ทำงานตามที่ Config หรือไม่
```

---

## Step 120 — Lab เต็มรูปแบบ: 3-Area OSPF พร้อม Route Summarization ทั้งสอง ABR

Lab สรุปของ Part นี้ — ทำ Reconfigure ครบทุกอุปกรณ์ตาม Step 114, เปิด Route Summarization
บน **ทั้งสอง ABR** ตาม Step 115, แล้ว Verify ให้เห็นผลลัพธ์ทั่ว Topology

### 120.1 Topology สุดท้ายพร้อม Area และ Summary Range

```
                    Area 1                        Area 0                       Area 2
              (10.10.10/20/99.0/24)          (10.255.0.0/30)             (10.10.30/40/99.0/24)
┌──────────┐                          ┌────────────────────────┐                          ┌──────────┐
│ DIST-SW1 │──Te1/1/1 (Area1)──┐      │                          │      ┌──Te1/1/1 (Area2)──│ DIST-SW3 │
└──────────┘                    │      │                          │      │                    └──────────┘
                          ┌─────┴─────┐│  Po1 .1 ════ Po1 .2      │┌─────┴─────┐
                          │ CORE-SW1  ││  10.255.0.0/30 (Area 0)  ││ CORE-SW2  │
                          │  (ABR)    │└────────────────────────┘│  (ABR)    │
                          └─────┬─────┘                            └─────┬─────┘
┌──────────┐                    │      area 1 range 10.10.0.0/16          │      ┌──────────┐
│ DIST-SW2 │──Te1/1/1 (Area1)──┘      (สรุป VLAN10/20/99 → Area 0)        └──Te1/1/1 (Area2)──│ DIST-SW4 │
└──────────┘                                                            area 2 range 10.10.0.0/16 └──────────┘
                                                                          (สรุป VLAN30/40/99 → Area 0)
```

### 120.2 Running-config เต็ม — CORE-SW1

```
hostname CORE-SW1
!
ip routing
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW2 - Area 0 **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Routed downlink to DIST-SW1 - Area 1 **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip ospf 1 area 1
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Routed downlink to DIST-SW2 - Area 1 **
 no switchport
 ip address 10.255.20.1 255.255.255.252
 ip ospf 1 area 1
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.1 255.255.255.255
!
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
 area 1 range 10.10.0.0 255.255.0.0
 passive-interface Loopback0
!
end
```

### 120.3 Running-config เต็ม — CORE-SW2

```
hostname CORE-SW2
!
ip routing
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW1 - Area 0 **
 no switchport
 ip address 10.255.0.2 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Routed downlink to DIST-SW3 - Area 2 **
 no switchport
 ip address 10.255.30.1 255.255.255.252
 ip ospf 1 area 2
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Routed downlink to DIST-SW4 - Area 2 **
 no switchport
 ip address 10.255.40.1 255.255.255.252
 ip ospf 1 area 2
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.2 255.255.255.255
!
router ospf 1
 router-id 1.1.1.2
 auto-cost reference-bandwidth 100000
 area 2 range 10.10.0.0 255.255.0.0
 passive-interface Loopback0
!
end
```

### 120.4 Running-config เต็ม — DIST-SW1 (Internal Router, Area 1 ล้วน)

```
hostname DIST-SW1
!
ip routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
 ip ospf 1 area 1
 no shutdown
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.2 255.255.255.0
 ip ospf 1 area 1
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.2 255.255.255.0
 ip ospf 1 area 1
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Trunk to ACCESS-SW1 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Trunk to ACCESS-SW2 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW1 - Area 1 **
 no switchport
 ip address 10.255.10.2 255.255.255.252
 ip ospf 1 area 1
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.11 255.255.255.255
!
router ospf 1
 router-id 1.1.1.11
 auto-cost reference-bandwidth 100000
 passive-interface Loopback0
!
end
```

> **DIST-SW2, DIST-SW3, DIST-SW4** ใช้โครงสร้างเดียวกันทุกประการ เปลี่ยนแค่ Hostname, IP Host
> (`.3`/`.2`/`.3` ตามตาราง Part 10), Area (1 สำหรับ DIST-SW2, 2 สำหรับ DIST-SW3/DIST-SW4),
> VLAN ที่รับผิดชอบ (30/40/99 สำหรับ DIST-SW3/4), และ Router ID (1.1.1.12/1.1.1.13/1.1.1.14
> ตามลำดับ) — ไม่ต่างจาก Pattern ที่วางไว้ตั้งแต่ Part 10 Step 96-97

### 120.5 Verify #1 — `show ip ospf database` บน CORE-SW1 (เห็น Type 3 Summary LSA)

```
CORE-SW1# show ip ospf database

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Router Link States (Area 0)
Link ID         ADV Router      Age    Seq#       Link count
1.1.1.1         1.1.1.1         210    0x8000006  1
1.1.1.2         1.1.1.2         208    0x8000006  1

                Router Link States (Area 1)
Link ID         ADV Router      Age    Seq#       Link count
1.1.1.1         1.1.1.1         210    0x8000007  2
1.1.1.11        1.1.1.11        195    0x8000005  4
1.1.1.12        1.1.1.12        192    0x8000005  4

                Summary Net Link States (Area 0)
Link ID         ADV Router      Age    Seq#
10.10.0.0       1.1.1.1         30     0x8000003          <- Type 3 จาก Area 1 (Summarized, Step 115)
10.10.0.0       1.1.1.2         28     0x8000003          <- Type 3 จาก Area 2 (Summarized เหมือนกัน)

                Summary Net Link States (Area 1)
Link ID         ADV Router      Age    Seq#
10.10.0.0       1.1.1.1         30     0x8000004          <- Re-advertise Summary จาก Area 2 กลับเข้า Area 1
```

**สังเกต**: หลัง Summarize ทั้งสองฝั่ง Area 0 เหลือ Type 3 LSA เพียง **2 บรรทัด** (จาก 6 Subnet
เดิม — VLAN10/20/99 ของ Area 1 และ VLAN30/40/99 ของ Area 2) แสดงผลของ Route Summarization
ชัดเจนตามที่ตั้งใจใน Step 115

### 120.6 Verify #2 — `show ip route` บน DIST-SW1 (เห็น O IA ของ VLAN30/40 ผ่าน Summary)

**ก่อน Summarize** (ถ้า CORE-SW2 ยังไม่ทำ `area 2 range`) DIST-SW1 จะเห็น Route แยกทีละ Subnet:

```
DIST-SW1# show ip route ospf
O IA    10.10.30.0/24 [110/30] via 10.255.10.1, 00:12:10, TenGigabitEthernet1/1/1
O IA    10.10.40.0/24 [110/30] via 10.255.10.1, 00:12:10, TenGigabitEthernet1/1/1
O IA    10.10.99.0/24 [110/30] via 10.255.10.1, 00:12:10, TenGigabitEthernet1/1/1   ! ระวัง overlap (Step 115.2)
```

**หลัง Summarize บน CORE-SW2** (`area 2 range 10.10.0.0 255.255.0.0`) — Type 3 ของ VLAN30/40/99
แยกกันจะถูก Suppress และแทนที่ด้วย **1 Summary Route เดียว** ที่ครอบคลุมทั้ง VLAN 30 และ
VLAN 40 (ตามที่ Step 120.5 ยืนยันจาก LSDB):

```
DIST-SW1# show ip route ospf

     10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
O IA    10.10.0.0/16 [110/20] via 10.255.10.1, 00:00:45, TenGigabitEthernet1/1/1
                                     ▲
                    Route เดียวนี้ครอบคลุม Reachability ไปยังทั้ง
                    VLAN 30 (10.10.30.0/24) และ VLAN 40 (10.10.40.0/24) ของ Area 2
                    (แทนที่ Route แยก 3 บรรทัดแบบ "ก่อน Summarize" ข้างบน)

DIST-SW1# ping 10.10.30.10 source Vlan10
Sending 5, 100-byte ICMP Echos to 10.10.30.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms

DIST-SW1# traceroute 10.10.30.10
  1  10.255.10.1  1 msec
  2  10.255.0.2   2 msec
  3  10.255.30.2  2 msec
  4  10.10.30.10  3 msec
```

Traffic จาก DIST-SW1 (Area 1) ไปถึง VLAN30 (Area 2) สำเร็จ **ผ่าน Route เดียวที่ Summarize
มาแล้ว** พิสูจน์ว่า Summarization ไม่กระทบ Connectivity จริง (แค่ลดจำนวน Entry ใน Routing
Table และ LSDB) และ Path ที่ Trace ได้ก็ตรงตามที่ออกแบบ (DIST-SW1 → CORE-SW1 → CORE-SW2 →
DIST-SW3 → Server1) — สอดคล้องกับ Backbone Rule ที่บังคับให้ทุก Inter-Area Traffic ต้องผ่าน
Area 0 (Port-channel1) เสมอ ไม่มี Shortcut อื่น

### 120.7 Verify #3 — สรุปภาพรวมทั้ง Topology

```
CORE-SW1# show ip ospf interface brief
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Po1          1     0               10.255.0.1/30      10    P2P   1/1
Te1/0/1      1     1               10.255.10.1/30      10    P2P   1/1
Te1/0/2      1     1               10.255.20.1/30      10    P2P   1/1

CORE-SW2# show ip ospf interface brief
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Po1          1     0               10.255.0.2/30      10    P2P   1/1
Te1/0/1      1     2               10.255.30.1/30      10    P2P   1/1
Te1/0/2      1     2               10.255.40.1/30      10    P2P   1/1

DIST-SW3# show ip route ospf
     10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
O IA    10.10.0.0/16 [110/20] via 10.255.30.1, 00:00:45, TenGigabitEthernet1/1/1
```

**ผลลัพธ์สุดท้ายของ Part นี้**: Lab หลักสูตรทั้งหมดเปลี่ยนจาก OSPF Single-Area เป็น **3-Area
OSPF ที่ถูกต้องตามมาตรฐาน Cisco** — CORE-SW1/CORE-SW2 ทำหน้าที่ ABR อย่างสมบูรณ์, LSDB ของ
แต่ละ Area เล็กลงและแยกขอบเขตกันชัดเจน, Route Summarization ลด Entry ใน Routing Table ได้จริง
โดยไม่กระทบ End-to-End Connectivity ✅

---

## แบบฝึกหัดทวนความเข้าใจ Part 12

1. ตามกฎ "Backbone Area 0 Rule" Area สองใบที่ไม่ใช่ Area 0 สามารถแลกเปลี่ยน Route กันโดยตรง
   (ไม่ผ่าน Area 0) ได้หรือไม่ และถ้า Physical Topology ไม่มี Link ตรงไปยัง Area 0 เลย ต้องแก้
   ด้วยวิธีใด?
2. LSA Type ใดที่สร้างโดย ABR และมีหน้าที่สรุป Subnet จาก Area หนึ่งไปให้ Area อื่นรู้จัก
   โดยไม่ต้องรู้รายละเอียด Topology ภายใน?
3. `area 2 stub` และ `area 2 stub no-summary` (Totally Stub) ต่างกันอย่างไร และ Totally Stub
   บล็อก LSA Type อะไรเพิ่มจาก Stub ธรรมดา?
4. ระหว่าง Route ประเภท **O E1** และ **O E2** ประเภทใดที่นำ Internal Cost (จาก Router ที่ดู
   ไปจนถึง ASBR) มารวมกับ External Cost ด้วย และประเภทใดที่ใช้ External Cost อย่างเดียว?
5. ในสถานการณ์ Neighbor OSPF ไม่ขึ้นเลยหลังทำ Multi-Area Migration (ทั้งที่ Physical Link
   Up/Up ปกติ) สาเหตุอันดับหนึ่งที่ต้องตรวจสอบก่อนคืออะไร และใช้คำสั่งใดตรวจสอบ?

**เฉลย:**
1. ไม่ได้ — ทุก Area ที่ไม่ใช่ Area 0 ต้องเชื่อมกับ Area 0 เท่านั้น (ห้าม Area-to-Area ตรง)
   ถ้าไม่มี Physical Link ตรงไปยัง Area 0 ต้องใช้ **Virtual Link** ทะลุผ่าน Transit Area
   (Step 117)
2. **Type 3 (Summary/Inter-Area LSA)** สร้างโดย ABR เท่านั้น (Type 4 สรุป Reachability ของ
   ASBR ไม่ใช่ Subnet, อย่าสับสนกัน)
3. Stub ธรรมดา Block เฉพาะ **Type 5 (External)** แต่ยังรับ Type 3 (Inter-area) เต็มรูปแบบ
   ส่วน Totally Stub บล็อกเพิ่มทั้ง **Type 3 ด้วย** เหลือแค่ Intra-area ของตัวเอง + Default
   Route เดียวจาก ABR — ทำให้ LSDB เล็กที่สุด
4. **O E1** นำ Internal Cost มารวมกับ External Cost (Cost = External + Internal) ส่วน
   **O E2** (ค่า Default ของ Cisco) ใช้ External Cost อย่างเดียว ไม่สนใจ Internal Cost เลย
5. **Area ID Mismatch** ระหว่าง Interface สองฝั่งของ Link เดียวกัน — ตรวจด้วย
   `show ip ospf interface brief` หรือ `show ip ospf interface <if>` เทียบ Area ID ทั้งสองฝั่ง
   ให้ตรงกัน (ปัญหานี้ทำให้ Hello Packet ถูก Reject ตั้งแต่ต้น ไม่มี Adjacency เกิดขึ้นเลย)

---

## สรุป Part 12

Part นี้ยกระดับ OSPF ของ Lab หลักสูตรจาก **Single-Area (Part 11)** ขึ้นเป็น **3-Area Design
เต็มรูปแบบ** ตามมาตรฐาน Enterprise จริง — เราเข้าใจว่าทำไมต้องมี Multi-Area (SPF Scalability,
Flooding Domain, Backbone Rule), บทบาท **ABR/ASBR** โดย CORE-SW1/CORE-SW2 กลายเป็น ABR ของ
Lab นี้อย่างเป็นทางการ, เจาะลึก **LSA ทั้ง 6 Type** ที่ขับเคลื่อน OSPF ทั้งระบบ, ลงมือ
**Reconfigure Interface ทุกใบ** ให้ตรงกับ Area Design ใหม่, ทำ **Route Summarization** บน
ทั้งสอง ABR ให้เห็นผลจริงใน LSDB และ Routing Table, เรียนรู้ **Area Type พิเศษ** (Stub/Totally
Stub/NSSA) และ **Virtual Link** สำหรับสถานการณ์ Design ที่ซับซ้อนขึ้น, และปิดท้ายด้วยการ
คำนวณ **Cost ข้าม Area** พร้อม Route Type ทั้งหมด (O, O IA, O E1, O E2) ✅

**สิ่งที่ Lab หลักสูตรนี้ยังไม่มี**: OSPF Authentication (ป้องกัน Router ปลอมเข้าร่วม Area),
Passive Interface แบบ Best Practice เต็มรูปแบบ (ใส่แค่ Loopback ใน Part นี้), และ Redistribution
เข้า-ออก OSPF จริง (รอ ASBR ตัวจริงที่ WAN Edge) — สิ่งเหล่านี้จะกลับมาเสริมในบท Advanced Routing
ถัดๆ ไป **Part 13 จะเปลี่ยนไปสำรวจ Routing Protocol อีกตัวที่สำคัญของ CCNP คือ EIGRP** ซึ่งใช้
แนวคิด Distance Vector ที่ต่างจาก OSPF (Link-State) โดยสิ้นเชิง — เหมาะสำหรับเปรียบเทียบและ
ทำความเข้าใจว่าทำไม Enterprise บางแห่งยังเลือกใช้ EIGRP แทน OSPF ในบางสถานการณ์

**ไปต่อ:** [Part 13 — EIGRP Fundamentals →](part-013-eigrp-fundamentals.md)
