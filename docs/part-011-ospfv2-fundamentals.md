# Part 11 — OSPFv2 Fundamentals
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 101–110 จาก 1000**

> ต่อจาก [Part 10 — Inter-VLAN Routing](part-010-inter-vlan-routing.md) ที่ปิดท้ายด้วยข้อสรุปว่า
> **Static Route ไม่ scale** — ทุกครั้งที่ Topology เปลี่ยน ต้องไปแก้ config ด้วยมือทุกจุด
> Part นี้จะแทนที่ Static Route ทั้งหมดที่ใช้เชื่อม **DIST-SW1–4 ↔ CORE-SW1/CORE-SW2** และ
> **CORE-SW1/CORE-SW2 ↔ WAN-EDGE-1/2** ด้วย **OSPFv2 (Open Shortest Path First version 2)**
> ซึ่งเป็น Dynamic Routing Protocol แบบ **Link-State** ที่ใช้กันมากที่สุดในโลก Enterprise/Campus
> Network — เราจะเรียนตั้งแต่ทฤษฎี Link-State, Neighbor State Machine, Router ID, การคอนฟิกทั้ง
> สอง Syntax (`network` statement และ `ip ospf area` แบบ per-interface), Network Type/DR-BDR,
> Timer, Passive-Interface และ Authentication ปิดท้ายด้วย Lab เต็มรูปแบบที่แปลง Topology หลักของ
> หลักสูตรทั้งหมดให้วิ่งด้วย **OSPF Area 0 เดี่ยว (Single Area)** — ส่วน Multi-Area OSPF (Area
> Border Router, Area 0 แบ่งเป็นหลาย Area) จะเรียนใน [Part 12](part-012-ospfv2-advanced.md)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 101 | OSPF คืออะไร — Link-State vs Distance-Vector, Dijkstra SPF, OSPF Cost Metric, แนวคิด Area |
| 102 | OSPF Neighbor State Machine — Down → Init → 2-Way → ExStart → Exchange → Loading → Full |
| 103 | Router ID (RID) — กฎการเลือกและการกำหนดแบบ Explicit |
| 104 | คอนฟิก OSPF พื้นฐาน — `network` Statement vs `ip ospf area` แบบ Per-Interface |
| 105 | OSPF Network Type — Point-to-Point vs Broadcast และการเลือกตั้ง DR/BDR |
| 106 | Hello/Dead Timer และปัญหา Timer Mismatch |
| 107 | Passive-Interface — ทำไม SVI ของ VLAN ผู้ใช้งานไม่ควรส่ง Hello |
| 108 | OSPF Authentication — Plaintext และ MD5 |
| 109 | คำสั่ง Verify OSPF ที่ต้องใช้ทุกวัน |
| 110 | Lab เต็มรูปแบบ: แทนที่ Static Route ทั้ง Topology ด้วย OSPF Area 0 |

---

## Step 101 — OSPF คืออะไร: Link-State, Dijkstra SPF, Cost Metric, แนวคิด Area

### 101.1 Distance-Vector vs Link-State

Static Route ที่เรียนใน [Part 9](part-009-static-routing.md) ต้องเขียนด้วยมือทุกเส้นทาง —
**Dynamic Routing Protocol** คือกลุ่ม Protocol ที่ Router/Switch แลกเปลี่ยนข้อมูล Topology กันเอง
แล้วคำนวณ Routing Table โดยอัตโนมัติ แบ่งเป็น 2 ตระกูลใหญ่:

| | Distance-Vector (เช่น RIP, EIGRP) | Link-State (เช่น **OSPF**, IS-IS) |
|---|---|---|
| มองเห็น Topology อย่างไร | เห็นแค่ "ทิศทางกับระยะทาง" ที่ Neighbor บอก (Routing by Rumor) | เห็น **แผนที่ทั้ง Network เหมือนกันทุกตัว** (Topology Database เต็มรูปแบบ) |
| วิธีคำนวณ Best Path | บวก Metric สะสมจาก Neighbor ไปเรื่อยๆ | รัน Algorithm (**Dijkstra SPF**) บน Topology Graph ของตัวเอง |
| ความเร็ว Convergence | ช้ากว่า (ต้องรอ Neighbor ส่งต่อ Update เป็นทอดๆ) | เร็วกว่า (รู้ Topology เปลี่ยนทันทีที่ Flood ถึง แล้วคำนวณ SPF ใหม่ทันที) |
| Traffic ที่ใช้ Update | ส่ง Routing Table ทั้งก้อนเป็นระยะ (Periodic) | ส่ง **Link-State Advertisement (LSA)** เฉพาะตอนมีการเปลี่ยนแปลง (Event-driven) |
| Scalability | จำกัดกว่า (Loop-prone ต้องพึ่ง Split-horizon ฯลฯ) | Scale ได้ดีกว่ามากด้วยแนวคิด **Area** (ดู 101.4) |

> **สรุปสำหรับข้อสอบ**: OSPF เป็น **Link-State**, **Classless** (รองรับ VLSM/CIDR เต็มรูปแบบ),
> ใช้ **Administrative Distance = 110**, ทำงานตรงบน IP Protocol Number **89** (ไม่ใช่ TCP/UDP)
> และเป็น **Open Standard** (RFC 2328 สำหรับ OSPFv2/IPv4) ต่างจาก EIGRP ที่เป็น Cisco Proprietary

### 101.2 Link-State ทำงานอย่างไร — 3 ขั้นตอนหลัก

```
ขั้นที่ 1: Neighbor Discovery         ขั้นที่ 2: Database Synchronization    ขั้นที่ 3: SPF Calculation
(หา Router ข้างเคียงด้วย Hello)        (แลก LSA จนทุกตัวมี LSDB เหมือนกัน)      (รัน Dijkstra บน LSDB → Routing Table)

  R1 ---Hello--> R2                    R1 <===LSA (Link-State Update)===> R2      R1 สร้าง "แผนที่" ทั้ง Network
  R1 <--Hello--- R2                    ทุก Router เก็บ LSA ทั้งหมดไว้ใน            แล้วรันอัลกอริทึม Dijkstra
                                        Link-State Database (LSDB)                หา Shortest Path ไปทุกปลายทาง
                                        LSDB ต้อง**เหมือนกันทุกตัว**ใน Area        แล้วใส่ Best Path ลง Routing Table
                                        เดียวกันเสมอ (Database Synchronization)
```

**ประเด็นสำคัญที่ต่างจาก Distance-Vector อย่างสิ้นเชิง**: Router ที่รัน OSPF **ไม่เคย**ส่ง "Routing
Table" ของตัวเองให้ Neighbor เหมือน RIP — มันส่งแค่ **Link-State Advertisement (LSA)** ที่บอกว่า
"ฉันมี Link นี้ๆ อยู่ ด้วย Cost เท่านี้" แล้วให้ทุกตัวไปคำนวณ Best Path **ด้วยตัวเอง** จาก LSDB
ที่เหมือนกันทุกตัว — นี่คือเหตุผลที่ OSPF ไม่มีปัญหา Routing Loop แบบที่ Distance-Vector รุ่นเก่าเจอ

### 101.3 Dijkstra Shortest Path First (SPF) Algorithm — แนวคิด

**Edsger Dijkstra** คิดค้น Algorithm นี้ในปี 1956 เพื่อหา **เส้นทางที่ถูกที่สุด (Lowest Cost)** จาก
Node หนึ่งไปยังทุก Node บน Weighted Graph — OSPF ใช้ Algorithm นี้ตรงๆ โดยมอง Topology เป็น Graph
ที่ Router = Node และ Link = Edge ที่มี "น้ำหนัก" คือ **OSPF Cost**

```
ตัวอย่าง Graph แบบง่าย (ตัวเลขบนเส้นคือ Cost):

        R1
       /   \
     1      10
     /        \
   R2 ---1---- R3
     \        /
     5      2
       \   /
        R4

R1 ต้องการไปหา R4 — มีเส้นทางเป็นไปได้ 3 เส้น:
  Path A: R1→R2→R4        Cost = 1+5  = 6
  Path B: R1→R3→R4        Cost = 10+2 = 12
  Path C: R1→R2→R3→R4     Cost = 1+1+2 = 4   <== ต่ำที่สุด! Dijkstra จะเลือกเส้นนี้

แม้ Path C มี "จำนวน Hop" มากกว่า Path A (3 hop vs 2 hop) แต่ OSPF สนใจ **Cost สะสมรวม** เท่านั้น
ไม่สนใจจำนวน Hop เหมือน RIP — นี่คือความแตกต่างสำคัญจาก Hop-Count-based Protocol
```

Dijkstra ทำงานแบบ **Greedy**: เริ่มจากตัวเอง (Root ของ SPF Tree) แล้วเลือก Node ที่ Cost สะสม
ต่ำที่สุดที่ยังไม่ได้ประมวลผลมาต่อยอดไปเรื่อยๆ จนครอบคลุมทุก Node ในกราฟ ผลลัพธ์สุดท้ายคือ
**SPF Tree** ที่ Router ตัวนั้นเป็น Root — Router ทุกตัวใน Area เดียวกันสร้าง SPF Tree ของตัวเอง
จาก LSDB ชุดเดียวกัน (แต่ Root ต่างกันเพราะแต่ละตัวมองจากตัวเอง)

### 101.4 OSPF Cost Metric — สูตรคำนวณ

OSPF ใช้ **Cost** เป็น Metric เดียว (ไม่ผสมหลายตัวแปรเหมือน EIGRP) คำนวณจาก:

```
                    Reference Bandwidth (ค่า Default = 100,000 Kbps = 100 Mbps)
   OSPF Cost   =   ─────────────────────────────────────────────────────────
                    Interface Bandwidth (Kbps) ที่ตั้งไว้บน Interface นั้น
```

| ประเภท Interface | Bandwidth Default | คำนวณ Cost (Reference=100,000) | Cost จริงที่ได้ (ปัดเศษ, ขั้นต่ำ=1) |
|---|---|---|---|
| FastEthernet (100 Mbps) | 100,000 Kbps | 100,000 / 100,000 = 1.0 | **1** |
| GigabitEthernet (1 Gbps) | 1,000,000 Kbps | 100,000 / 1,000,000 = 0.1 | **1** (ปัดขั้นต่ำ) |
| TenGigabitEthernet (10 Gbps) | 10,000,000 Kbps | 100,000 / 10,000,000 = 0.01 | **1** (ปัดขั้นต่ำ) |
| Serial (1.544 Mbps, T1) | 1,544 Kbps | 100,000 / 1,544 = 64.7 | **64** |

> **ข้อสังเกตสำคัญระดับ CCNP**: ด้วย Reference Bandwidth ค่า Default (100 Mbps) ทั้ง
> GigabitEthernet และ TenGigabitEthernet จะได้ **Cost = 1 เท่ากันหมด** เพราะค่าถูกปัดเศษขั้นต่ำที่ 1
> ทำให้ OSPF **มองไม่เห็นความแตกต่างของ Bandwidth** ระหว่าง Link 1 Gbps กับ 10 Gbps เลย — นี่คือ
> ปัญหาจริงที่ Enterprise สมัยใหม่ (ที่มี Gig/10G/40G ปนกัน) ต้องแก้ด้วยคำสั่ง
> `auto-cost reference-bandwidth <Mbps>` เพื่อยกค่า Reference ให้สูงขึ้น (เช่น 100000 = 100 Gbps)
> ให้ Cost แยกความต่างของ Bandwidth ได้จริง — รายละเอียดเชิงลึกเรื่องนี้จะกลับมาพูดถึงอีกครั้งใน
> [Part 12](part-012-ospfv2-advanced.md) ตอนนี้ให้จำไว้ว่า **Lab ของหลักสูตรนี้ทุก Backbone Link
> เป็น Gig/TenGig ล้วน จึงจะเห็น Cost = 1 เกือบทุกเส้นใน `show ip ospf interface`**

Cost เป็นแบบ **สะสมทุก Outgoing Interface ตลอดเส้นทาง (Additive)** — ถ้า Packet ต้องผ่าน 3 Link
ที่มี Cost 1, 1, 1 ตามลำดับ Total Cost ที่ปลายทางคือ 3 (คิดจาก Cost ขาออกของแต่ละ Hop เท่านั้น
ไม่รวม Cost ของ Interface ขาเข้าที่ปลายทาง)

### 101.5 แนวคิด Area — เกริ่นนำ (รายละเอียดเต็มใน Part 12)

OSPF แก้ปัญหา Scale ของ Link-State (ที่ต้องรัน SPF ทุกครั้งที่มีการเปลี่ยนแปลง ซึ่งกินทรัพยากร CPU
มากถ้า Topology ใหญ่) ด้วยการแบ่ง Network ออกเป็น **Area** — Router ใน Area เดียวกันเท่านั้นที่ต้อง
มี LSDB เหมือนกันทั้งหมด (LSA Type 1/2 ไม่ข้าม Area) ทำให้ SPF Calculation ถูกจำกัดขอบเขตไม่ต้อง
ประมวลผลทั้ง Enterprise ทีเดียว

```
Multi-Area OSPF (แนวคิด — จะ config จริงใน Part 12)
                    ┌─────────────┐
                    │   Area 0    │  <- Backbone Area (ต้องมีเสมอ ทุก Area อื่นต้องต่อผ่าน Area 0)
                    └──┬───────┬──┘
                  ┌─────┴──┐ ┌──┴─────┐
                  │ Area 1 │ │ Area 2 │  <- Area ปกติ ต่อกับ Area 0 ผ่าน ABR (Area Border Router)
                  └────────┘ └────────┘
```

> **Part 11 นี้ใช้ Single Area (Area 0) ทั้งหมด** — ทุก Router/Switch ทุกตัวในหลักสูตรนี้จะอยู่ใน
> Area 0 เดียวกัน ยังไม่มีการแบ่ง Area ย่อย เพราะ Topology ของหลักสูตรยังมีขนาดเล็กพอที่ Area
> เดียวจะไม่มีปัญหา Scale — Multi-Area (แบ่ง Area 1, Area 2 ให้ DIST-SW1/2 และ DIST-SW3/4) จะ
> เรียนใน Part 12 พร้อมเหตุผลว่าทำไมองค์กรขนาดใหญ่ถึงต้องแบ่ง

---

## Step 102 — OSPF Neighbor State Machine

ก่อน Router สองตัวจะกลายเป็น **Full Neighbor** (แลก LSA กันได้สมบูรณ์) ต้องผ่าน State หลายขั้น
เรียงตามลำดับ — คำสั่ง `debug ip ospf adj` หรือดู Syslog จะเห็น State เปลี่ยนแบบนี้ทุกครั้งที่ Neighbor
ก่อตัวขึ้นใหม่

### 102.1 Packet Type ที่ OSPF ใช้ (5 ชนิด)

| Packet Type | ชื่อ | หน้าที่ |
|---|---|---|
| Type 1 | **Hello** | ค้นหา/รักษาสถานะ Neighbor — ส่งทุก Hello-Interval ตลอดเวลา (ดู Step 106) |
| Type 2 | **DBD** (Database Description) | สรุป "หัวข้อ" LSA ที่ตัวเองมี (แค่ Header ไม่ใช่เนื้อ LSA เต็ม) เพื่อเทียบว่าใครขาด LSA อะไร |
| Type 3 | **LSR** (Link-State Request) | ขอ LSA ฉบับเต็มที่ตัวเองยังไม่มี (จากผลเทียบ DBD) |
| Type 4 | **LSU** (Link-State Update) | ส่ง LSA ฉบับเต็มจริงๆ (คำตอบของ LSR, และใช้ Flood ตอน Topology เปลี่ยนด้วย) |
| Type 5 | **LSAck** | ยืนยันว่าได้รับ LSU แล้ว (OSPF Reliable — ทุก LSA ต้องมีการ Acknowledge) |

### 102.2 ลำดับ State ทั้ง 7 ขั้น

```
DOWN ──Hello──> INIT ──เจอ RID ตัวเองใน Hello ปลายทาง──> 2-WAY ──เลือก Master/Slave──> EXSTART
                                                                                            │
                                                                                     แลก DBD (Type2)
                                                                                            ▼
FULL <──ได้ LSA ครบ, LSDB Sync แล้ว── LOADING <──ขอ LSA ที่ขาดด้วย LSR/LSU── EXCHANGE
```

| State | เกิดอะไรขึ้น | คำอธิบาย |
|---|---|---|
| **Down** | ยังไม่เคยได้รับ Hello จากปลายทางเลย | State เริ่มต้นของทุก Neighbor ก่อนเริ่มกระบวนการ |
| **Init** | ได้รับ Hello Packet จากปลายทางแล้ว 1 ครั้ง | แต่ Hello นั้น**ยังไม่มี** RID ของตัวเราอยู่ใน Neighbor List ของเขา (เขายังไม่รู้ว่าเราเห็นเขา) |
| **2-Way** | เห็น RID ของตัวเองอยู่ใน Neighbor List ของ Hello ที่ได้รับ | ยืนยันว่าเป็น **Bidirectional Communication** แล้ว — บน Broadcast/NBMA จะเลือก **DR/BDR** ที่ State นี้ (ดู Step 105) |
| **ExStart** | เริ่มกระบวนการ Database Synchronization | ตกลงว่าใครเป็น **Master** (RID สูงกว่า) ใครเป็น **Slave** — Master เป็นผู้กำหนด Sequence Number ของ DBD |
| **Exchange** | แลก **DBD (Database Description)** กัน | แต่ละฝั่งส่งแค่ **Summary/Header** ของ LSA ที่มี ไม่ใช่เนื้อ LSA เต็ม — เทียบกันว่าใครขาด LSA ตัวไหน |
| **Loading** | ส่ง **LSR** ขอ LSA ฉบับเต็มที่ขาด แล้วได้ **LSU** ตอบกลับ | ทำซ้ำจนกว่า LSDB ทั้งสองฝั่งจะเหมือนกันสมบูรณ์ (ทุก LSU ต้องมี **LSAck** ตอบกลับด้วย) |
| **Full** | LSDB Synchronized สมบูรณ์แล้ว | Neighbor พร้อม Advertise ผ่าน LSA และเป็น Adjacency ที่**สมบูรณ์**เท่านั้นที่จะปรากฏใน SPF Tree |

> **ข้อสังเกตสำคัญ**: บน Multi-access Segment (Broadcast) Router ที่**ไม่ใช่** DR/BDR จะคุยกัน
> ด้วย State สูงสุดที่ **2-Way** เท่านั้น (ไม่ไปต่อ Full ระหว่างกันเอง) — จะ Full ก็ต่อกับ DR/BDR
> เท่านั้น รายละเอียดเต็มอยู่ใน Step 105 ส่วนบน **Point-to-Point Link** (ซึ่งเป็น Network Type
> หลักของ Backbone Link ในหลักสูตรนี้หลัง Step 105) ทุก Neighbor จะไป **Full** เสมอเพราะไม่มี
> DR/BDR ให้ต้องเลือก

### 102.3 ตัวอย่างจริงจาก `debug ip ospf adj` (ย่อ)

```
CORE-SW1# debug ip ospf adj
OSPF-1 ADJ  Te1/0/1: 2 Way Communication to 1.1.1.11, state 2WAY
OSPF-1 ADJ  Te1/0/1: Nbr 1.1.1.11: Prepare dbase exchange
OSPF-1 ADJ  Te1/0/1: Nbr 1.1.1.11 has larger interface MTU
OSPF-1 ADJ  Te1/0/1: Send DBD to 1.1.1.11 seq 0x1A02 opt 0x52 flag 0x7 len 32
OSPF-1 ADJ  Te1/0/1: Rcv DBD from 1.1.1.11 seq 0x1A03 opt 0x52 flag 0x2 len 132  state EXCHANGE
OSPF-1 ADJ  Te1/0/1: Nbr 1.1.1.11: dbase exchange - as SLAVE, seq 0x1A03
OSPF-1 ADJ  Te1/0/1: Synchronized with 1.1.1.11, state FULL
%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.11 on TenGigabitEthernet1/0/1 from LOADING to FULL, Loading Done
```

Syslog `%OSPF-5-ADJCHG` คือสิ่งที่ต้องคุ้นเคยที่สุดในการ Troubleshoot OSPF — บอก Neighbor RID,
Interface, State ก่อนและหลัง, และเหตุผล (`Loading Done` = ปกติสมบูรณ์)

---

## Step 103 — Router ID (RID): กฎการเลือกและการกำหนดแบบ Explicit

### 103.1 RID คืออะไรและทำไมสำคัญ

**Router ID (RID)** คือเลข 32-bit (แสดงเป็นรูปแบบ Dotted-Decimal เหมือน IP) ที่ใช้**ระบุตัวตน**
ของ Router หนึ่งตัวใน OSPF Domain — ต้อง **ไม่ซ้ำกัน** ระหว่าง Router ทุกตัวในหลักสูตรนี้ เพราะ RID
ถูกใช้เป็น Key หลักในการอ้างอิง LSA ทุกฉบับ (LSA แต่ละ Type จะระบุว่า "Originate จาก RID ไหน")
ถ้า RID ซ้ำกันโดยไม่ตั้งใจ → LSDB จะสับสน มองว่า LSA มาจาก Router เดียวกันทั้งที่จริงเป็นคนละตัว

### 103.2 กฎการเลือก RID (ตามลำดับความสำคัญ)

```
1. ถ้ามีคำสั่ง `router-id x.x.x.x` ระบุไว้ตรงๆ ใต้ router ospf   → ใช้ค่านี้เสมอ (สูงสุด)
2. ถ้าไม่มี router-id คำสั่ง → เลือก IP ของ Loopback Interface ที่ IP สูงที่สุด (ถ้ามีหลายใบ)
3. ถ้าไม่มี Loopback เลย    → เลือก IP ของ Physical Interface ที่ Active (up/up) ที่ IP สูงที่สุด
```

> **เหตุผลที่ Loopback ชนะ Physical Interface เสมอ**: Loopback **ไม่มีวัน Down** ตราบใดที่ตัว
> Router/Switch ยังเปิดอยู่ (ไม่ผูกกับสถานะสาย/Physical Link) — ทำให้ RID มีเสถียรภาพสูงสุด ถ้าใช้
> Physical Interface IP เป็น RID แล้ว Interface นั้น Down ไป (สายหลุด/พอร์ตเสีย) RID อาจเปลี่ยน
> โดยไม่ตั้งใจ ทำให้ Adjacency ทั้งหมดต้องสร้างใหม่ — **นี่คือเหตุผลที่ Best Practice บอกให้ทุก
> Router/Switch ที่รัน OSPF ต้องมี Loopback0 เสมอ** (เป็นเหตุผลเดียวกับที่หลักสูตรนี้วางแผน
> Loopback0 ไว้ให้ทุกอุปกรณ์ตั้งแต่ [00-ip-address-plan.md](00-ip-address-plan.md))

### 103.3 ข้อสำคัญ: RID ไม่เปลี่ยนเองแม้ Loopback ใหม่ที่ IP สูงกว่าจะถูกเพิ่มเข้ามาทีหลัง

RID ถูกเลือก **ครั้งเดียวตอน OSPF Process เริ่มทำงาน** (ตอน Neighbor แรกก่อตัว) — ถ้าเพิ่ม
Loopback ใหม่ที่ IP สูงกว่าเข้ามาทีหลัง OSPF **จะไม่เปลี่ยน RID ให้เองทันที** จนกว่าจะ Restart
OSPF Process ด้วย `clear ip ospf process` หรือ Reload อุปกรณ์ — นี่คือเหตุผลสำคัญที่ควรตั้ง
**`router-id` แบบ Explicit ตรงๆ** เสมอในงาน Production เพื่อความชัดเจนและป้องกันพฤติกรรมที่
คาดเดาไม่ได้นี้

### 103.4 คอนฟิก Explicit Router ID — ใช้ค่าตาม Loopback0 Plan

Lab ของหลักสูตรนี้จะตั้ง `router-id` แบบ Explicit ให้ตรงกับ Loopback0 ของแต่ละอุปกรณ์เป๊ะๆ ตาม
[00-ip-address-plan.md](00-ip-address-plan.md) เพื่อให้ RID อ่านออกและจดจำได้ง่ายทันทีที่เห็น
(RID = Loopback0 เสมอ ไม่ต้องเดา):

```
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# router-id 1.1.1.1
Reload or use "clear ip ospf process" command, for this to take effect
```

> **หมายเหตุ**: ถ้า OSPF Process มี Neighbor อยู่แล้วตอนสั่ง `router-id` ใหม่ IOS จะเตือนให้
> `clear ip ospf process` เพื่อ Restart Adjacency ทั้งหมดด้วย RID ใหม่ — คำสั่งนี้จะทำให้ Neighbor
> ทุกตัวหลุดแล้วสร้างใหม่ชั่วคราว (Disruptive) จึงควรตั้ง `router-id` **ตั้งแต่ครั้งแรกที่สร้าง
> OSPF Process** ก่อนมี Neighbor ใดๆ เกิดขึ้นเลย เพื่อไม่ต้อง Clear ทีหลัง

### 103.5 ตาราง Router ID เต็มของหลักสูตรนี้ (อ้างอิง Loopback0 Plan)

| Device | Loopback0 (= Router ID ที่ตั้ง Explicit) |
|---|---|
| CORE-SW1 | 1.1.1.1 |
| CORE-SW2 | 1.1.1.2 |
| DIST-SW1 | 1.1.1.11 |
| DIST-SW2 | 1.1.1.12 |
| DIST-SW3 | 1.1.1.13 |
| DIST-SW4 | 1.1.1.14 |
| WAN-EDGE-1 | 1.1.1.21 |
| WAN-EDGE-2 | 1.1.1.22 |

---

## Step 104 — คอนฟิก OSPF พื้นฐาน: `network` Statement vs `ip ospf area` Per-Interface

OSPF มี**สอง Syntax หลัก**ในการบอกให้ Interface หนึ่งเข้าร่วม Area ใดๆ ทั้งสองวิธีให้ผลลัพธ์
เหมือนกันทุกประการในทางปฏิบัติ แต่ต่างกันที่ตำแหน่งที่พิมพ์คำสั่ง

### 104.1 วิธีที่ 1 (ดั้งเดิม): `network` Statement ใต้ `router ospf`

```
Router(config)# router ospf <process-id>
Router(config-router)# network <ip-address> <wildcard-mask> area <area-id>
```

- `<process-id>`: เลข Local-Significant เท่านั้น (ไม่ต้องตรงกันระหว่าง Router — ต่างจาก RID/Area
  ที่ต้องตรงกัน) ใช้แค่แยก Process ถ้ามีหลาย Instance บนเครื่องเดียว
- `<wildcard-mask>`: Inverse Mask (ตรงข้าม Subnet Mask) — `0.0.0.255` = /24, `0.0.0.3` = /30,
  `0.0.0.0` = Host เดียว (มักใช้กับ Loopback /32)
- คำสั่งนี้**ไม่ได้กำหนดว่าจะ Advertise Subnet ไหน** — มันแค่บอกว่า **Interface ไหนที่มี IP ตรงกับ
  Range นี้ ให้เปิด OSPF บน Interface นั้น** (Subnet ที่ Advertise คือ Subnet ที่ Interface
  Configured อยู่จริง ไม่ใช่ Range ที่พิมพ์ใน `network`)

```
! ตัวอย่างบน DIST-SW1 — สองแบบนี้ให้ผลเหมือนกัน (แบบ /24 เต็ม Subnet, แบบ Host เจาะจง)
DIST-SW1(config-router)# network 10.10.10.0 0.0.0.255 area 0      ! แบบ match ทั้ง subnet
DIST-SW1(config-router)# network 10.10.10.2 0.0.0.0 area 0        ! แบบ match เฉพาะ IP นี้ (เทียบเท่ากัน เพราะมี Interface เดียวใน Subnet)
```

### 104.2 วิธีที่ 2 (แนะนำ — สมัยใหม่): `ip ospf <process-id> area <area-id>` ใต้ Interface

```
Router(config)# interface <interface-id>
Router(config-if)# ip ospf <process-id> area <area-id>
```

วิธีนี้พิมพ์**ตรงใต้ Interface เลย** ไม่ต้องมานั่งคำนวณ Wildcard Mask ให้ตรงกับ Subnet — บอกชัดๆ
ว่า "Interface นี้ เข้า Process นี้ เข้า Area นี้" อ่านง่ายกว่า, ลด Human Error เรื่อง Wildcard ผิด,
และเห็นในทันทีจาก `show running-config interface` ว่า Interface ไหนเปิด OSPF บ้าง (ไม่ต้องไปนั่ง
ไล่ Wildcard ใน Router OSPF Block เทียบเอง) — **Cisco แนะนำวิธีนี้เป็นวิธีหลักตั้งแต่ IOS 15.x
เป็นต้นไป** และเป็นวิธีที่หลักสูตรนี้จะใช้เป็นหลักตั้งแต่ Step 110 เป็นต้นไป

```
! ตัวอย่างเดียวกันบน DIST-SW1 แบบ Per-Interface
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip ospf 1 area 0
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# ip ospf 1 area 0
```

### 104.3 เปรียบเทียบสองวิธี

| | `network` Statement | `ip ospf area` Per-Interface |
|---|---|---|
| พิมพ์ที่ไหน | ใต้ `router ospf` (รวมศูนย์) | ใต้แต่ละ Interface (กระจาย) |
| ต้องคำนวณ Wildcard Mask | ต้อง | ไม่ต้อง |
| เห็นภาพรวมง่ายไหม | ง่าย (ดูที่เดียวเห็นหมด) | ต้องไล่ดูทุก Interface |
| เสี่ยง Human Error | สูงกว่า (Wildcard ผิดพลาดได้ เผลอไป match Interface อื่นที่ไม่ตั้งใจ) | ต่ำกว่า (ระบุตรงๆ) |
| Cisco แนะนำ (IOS สมัยใหม่) | เป็นวิธีเก่า ยังใช้ได้ | **แนะนำเป็นวิธีหลัก** |
| ใช้ในหลักสูตรนี้ | สาธิตให้เข้าใจ Concept (Step นี้) | **ใช้จริงในทุก Lab ตั้งแต่ Step 110** |

> ทั้งสองวิธีสามารถ**ผสมกันได้ในอุปกรณ์เดียวกัน** (บาง Interface ใช้ `network`, บาง Interface ใช้
> `ip ospf area`) — OSPF ไม่สนใจว่า Interface ถูกเปิดใช้งานด้วยวิธีไหน ผลลัพธ์สุดท้ายเหมือนกัน

### 104.4 ทำไมต้องมี `router ospf 1` ก่อนใช้วิธี Per-Interface ด้วย?

แม้จะใช้วิธี Per-Interface เป็นหลัก ก็ยังต้องมีคำสั่ง `router ospf 1` (แม้จะไม่มี `network`
statement อยู่ข้างในเลยก็ตาม) เพื่อ**สร้าง Process** ขึ้นมาก่อน — Process ID `1` ในคำสั่ง
`ip ospf 1 area 0` ต้องตรงกับ Process ID ที่สร้างไว้ใต้ `router ospf` เสมอ (เลข 1 ทั้งสองที่ต้อง
match กัน ไม่ใช่ Global Auto-detect):

```
DIST-SW1(config)# router ospf 1              ! สร้าง Process ID 1 ก่อน (จำเป็นเสมอ)
DIST-SW1(config-router)# router-id 1.1.1.11  ! ตั้ง RID แบบ Explicit (Step 103.4)
DIST-SW1(config-router)# exit
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip ospf 1 area 0        ! เลข "1" ตรงนี้ ต้องตรงกับ Process ID ข้างบน
```

---

## Step 105 — OSPF Network Type: Point-to-Point vs Broadcast และ DR/BDR Election

### 105.1 ทำไมต้องมี Network Type

OSPF ต้องรู้ว่า Segment ที่ Interface เชื่อมอยู่เป็น "ประเภท" ไหน เพราะพฤติกรรมการหา Neighbor และ
การ Flood LSA **ต่างกันโดยพื้นฐาน** ระหว่าง Link จุดต่อจุด (มี Router 2 ตัวเท่านั้น) กับ Link แบบ
Multi-access (Router หลายตัวอยู่ Segment เดียวกัน เช่น Ethernet Switch ที่มี Router 3-4 ตัวต่ออยู่)

| Network Type | Default บน Interface ประเภทไหน | ต้องเลือก DR/BDR ไหม | Hello/Dead Default |
|---|---|---|---|
| **Point-to-Point** | Serial ที่ใช้ HDLC/PPP Encapsulation | **ไม่ต้อง** | 10s / 40s |
| **Broadcast** | Ethernet ทุกชนิด (รวม Routed Port ที่ `no switchport`!) | **ต้อง** | 10s / 40s |
| **Non-Broadcast (NBMA)** | Frame Relay แบบเดิม (Legacy) | ต้อง (แต่ต้องตั้ง Neighbor เอง) | 30s / 120s |
| **Point-to-Multipoint** | Frame Relay Hub-and-Spoke สมัยใหม่ | ไม่ต้อง | 30s / 120s |

> **จุดที่คนเข้าใจผิดบ่อยที่สุด**: Routed Port แบบ `no switchport` บน Catalyst Switch (เช่น
> `TenGigabitEthernet1/1/1` ที่ใช้เชื่อม DIST-SW1↔CORE-SW1 ในหลักสูตรนี้) แม้จะเป็น **Point-to-Point
> ทางกายภาพจริง** (มี Router แค่ 2 ตัวคุยกัน ไม่มีตัวที่ 3) แต่ **Default Network Type ของ OSPF
> ยังเป็น "Broadcast" เสมอ** เพราะ OSPF ตัดสิน Network Type จาก**ประเภท Interface** (Ethernet =
> Broadcast เสมอโดย Default) ไม่ได้ตัดสินจากจำนวน Router ที่ต่ออยู่จริง

### 105.2 ผลกระทบ: DR/BDR ถูกเลือกทั้งที่ Link มี Router แค่ 2 ตัว

เพราะ Backbone Link ทุกเส้นในหลักสูตรนี้เป็น Ethernet (Routed Port) → Default Network Type =
Broadcast → OSPF จะ**เลือก DR/BDR แม้จะมี Router แค่ 2 ตัวบน Link นั้น** ซึ่งไม่ได้ผิดอะไร
(ยังทำงานได้ปกติ) แต่เป็น **Overhead ที่ไม่จำเป็น**:
- ต้องรอ Wait Timer (เท่ากับ Dead Interval) ก่อนเลือก DR/BDR รอบแรก ทำให้ Adjacency ก่อตัวช้าลง
- สร้าง **LSA Type 2 (Network LSA)** เพิ่มขึ้นมาโดยไม่จำเป็น (เพราะ Link มี Router แค่ 2 ตัว ไม่มี
  ประโยชน์อะไรจากการมี Network LSA เทียบกับ Point-to-Point ที่ไม่ต้องมี)

**วิธีแก้ที่เป็น Best Practice**: บังคับ Network Type ให้เป็น Point-to-Point ด้วยมือ บน Link ที่รู้
แน่ชัดว่าเป็น Point-to-Point จริง (Backbone /30 ทุกเส้นในหลักสูตรนี้เข้าเกณฑ์นี้):

```
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# ip ospf network point-to-point
```

หลังตั้งค่านี้ Neighbor จะข้ามขั้น DR/BDR Election ไปเลย ทำ Adjacency เร็วขึ้น และไม่มี LSA Type 2
เกิดขึ้นบน Link นั้นอีก — **Lab เต็มรูปแบบใน Step 110 จะตั้งค่านี้บนทุก Backbone Link**

### 105.3 DR/BDR Election ทำงานอย่างไรจริงๆ (ตัวอย่างสมมติ — Multi-access Segment)

เพื่อให้เข้าใจ Concept DR/BDR อย่างสมบูรณ์ (แม้ Lab จริงของหลักสูตรนี้จะบังคับ Point-to-Point จน
ไม่ต้องใช้) ให้ดูตัวอย่างสมมติ: มี Router 3 ตัว (R1, R2, R3) ต่อ Ethernet Segment เดียวกันผ่าน
Switch (ไม่ใช่ Routed Port แบบในหลักสูตรนี้ แต่เป็น VLAN เดียวที่มี 3 Router จริงๆ):

```
        Ethernet Segment เดียวกัน (VLAN 100, ผ่าน Switch)
   ┌───────────┬───────────┬───────────┐
   │           │           │           │
   R1         R2          R3         (Switch เชื่อมทั้งหมด)
 Pri=1      Pri=100      Pri=0
 RID .1     RID .2       RID .3
```

**กฎการเลือก DR/BDR**:
1. **OSPF Priority สูงสุด** (0–255, Default = 1) ชนะเสมอ — ตั้งด้วย `ip ospf priority <0-255>`
2. ถ้า Priority เท่ากัน → **Router ID สูงสุด** ชนะ (Tie-break)
3. Router ที่ตั้ง Priority = **0** จะไม่มีสิทธิ์เป็น DR **หรือ** BDR เลย (ประกาศตัวว่า "ห้ามเลือกฉัน")

จากตัวอย่าง: R2 (Priority 100) ชนะเป็น **DR**, R1 (Priority 1, RID สูงกว่า R3 ที่ Priority=0) เป็น
**BDR**, R3 (Priority 0) เป็น **DROTHER** (ไม่มีสิทธิ์เป็น DR/BDR เลยเพราะ Priority=0)

```
DR   = R2   (Designated Router — ทุกตัวบน Segment สร้าง Full Adjacency กับ DR เสมอ)
BDR  = R1   (Backup Designated Router — สำรองไว้เผื่อ DR ล่ม, ก็ Full กับทุกตัวเหมือนกัน)
R3   = DROTHER (คุยกับ DR/BDR แบบ Full แต่คุยกับ Router อื่นที่เป็น DROTHER ด้วยกันแค่ 2-Way เท่านั้น)
```

> **เหตุผลที่ต้องมี DR**: บน Multi-access Segment ถ้าทุก Router Flood LSA ถึงกันหมดแบบ Full-Mesh
> (n × (n-1)/2 ความสัมพันธ์) จะสิ้นเปลือง Bandwidth/CPU มาก — DR ทำหน้าที่เป็น "ศูนย์กลาง" ที่ทุกตัว
> ส่ง Update ให้ทีเดียว แล้ว DR Flood กระจายต่อให้ทุกตัวอีกที ลดความสัมพันธ์ทั้งหมดลงเหลือแค่
> ผ่าน DR/BDR เท่านั้น — BDR มีไว้ Take over ทันทีถ้า DR ล่ม (ไม่ต้อง Re-election รอนาน)

> **ข้อสำคัญ**: DR/BDR Election **ไม่ Preempt** — ถ้า Router ใหม่ที่มี Priority สูงกว่าเข้ามาทีหลัง
> (หลังเลือก DR ไปแล้ว) จะ**ไม่แย่งตำแหน่ง DR คืนจากตัวที่เป็นอยู่** จนกว่า DR ตัวเดิมจะ Down ไปก่อน
> — นี่คือเหตุผลที่ควรตั้ง Priority ให้ถูกต้องตั้งแต่แรก ไม่ใช่ไปตั้งทีหลังแล้วคาดหวังให้ Election ใหม่ทันที

---

## Step 106 — Hello/Dead Timer และปัญหา Timer Mismatch

### 106.1 ค่า Default ตาม Network Type

| Network Type | Hello Interval | Dead Interval | อัตราส่วน |
|---|---|---|---|
| Broadcast | 10 วินาที | 40 วินาที | Dead = 4× Hello |
| Point-to-Point | 10 วินาที | 40 วินาที | Dead = 4× Hello |
| NBMA | 30 วินาที | 120 วินาที | Dead = 4× Hello |
| Point-to-Multipoint | 30 วินาที | 120 วินาที | Dead = 4× Hello |

**Hello Interval**: ความถี่ที่ส่ง Hello Packet ออกไปเพื่อรักษาสถานะ Neighbor
**Dead Interval**: ถ้าไม่ได้รับ Hello จาก Neighbor ตัวใดเลยภายในเวลานี้ → ประกาศว่า Neighbor
นั้น**ตาย** ทันที (ลบออกจาก Neighbor Table, ต้อง SPF Recalculation ใหม่)

> สังเกตว่าเมื่อทำ [Step 105.2](#step-105--ospf-network-type-point-to-point-vs-broadcast-และ-drbdr-election)
> เปลี่ยน Network Type จาก Broadcast → Point-to-Point แล้ว **Hello/Dead Timer ไม่เปลี่ยนเลย**
> (ทั้งสองแบบ Default คือ 10s/40s เท่ากัน) — สิ่งที่เปลี่ยนคือแค่พฤติกรรม DR/BDR Election เท่านั้น

### 106.2 ปัญหา Timer Mismatch — Neighbor ค้างไม่ขึ้น Full

Hello Packet ที่แต่ละ Router ส่งออกไป **บรรจุค่า Hello Interval และ Dead Interval ของตัวเองไปด้วย
เสมอ** — เมื่อฝั่งรับได้รับ Hello จะ**เทียบค่าที่ระบุมากับค่าที่ตั้งไว้บน Interface ของตัวเอง** ถ้า
**ค่าใดค่าหนึ่งไม่ตรงกัน** (Hello หรือ Dead ก็ตาม) จะ**ปฏิเสธ Hello นั้นทันที** — Neighbor จะไม่มี
วันขึ้นเกิน State **Init** เลย (หรือไม่ปรากฏใน Neighbor Table เลยด้วยซ้ำในบางกรณี)

```
! ตัวอย่างจงใจตั้ง Hello Interval ไม่ตรงกัน (DIST-SW1 ใช้ 10s ปกติ, DIST-SW2 ตั้งเป็น 5s)
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# ip ospf hello-interval 5

! ผลลัพธ์ที่ CORE-SW1 (ฝั่งตรงข้าม ยังใช้ Default 10s) — เห็น Log แจ้งเตือนทันที
%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.12 on TenGigabitEthernet1/0/2 from 2WAY to DOWN, Neighbor Down: Dead timer expired
%OSPF-4-ERRRCV: Received invalid packet: mismatch hello parameters from 10.255.20.2, TenGigabitEthernet1/0/2

CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.12          0   INIT/  -        00:00:32    10.255.20.2     TenGigabitEthernet1/0/2   <-- ค้างที่ INIT ตลอด
```

### 106.3 วิธีแก้: ปรับ Timer ให้ตรงกันทั้งสองฝั่ง

```
! ต้องแก้ทั้งสองฝั่งให้ตรงกัน — เลือกค่าใดก็ได้ แต่ต้อง "เท่ากันทุกด้าน" ของ Link เดียวกันเสมอ
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# ip ospf hello-interval 5
DIST-SW2(config-if)# ip ospf dead-interval 20      ! ปรับ Dead ให้เป็น 4x Hello ตามธรรมเนียม (ไม่บังคับ แต่แนะนำ)

CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# ip ospf hello-interval 5
CORE-SW1(config-if)# ip ospf dead-interval 20

! หลังแก้ทั้งสองฝั่งให้ตรงกัน — Neighbor ขึ้น FULL ปกติทันที
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.12          0   FULL/  -        00:00:17    10.255.20.2     TenGigabitEthernet1/0/2
```

> **กฎจำง่าย**: Hello/Dead Timer **ไม่ต้องเป็นค่า Default** แต่**ต้องตรงกันทุกด้านของ Link
> เดียวกันเสมอ** — Lab หลักของหลักสูตรนี้ (Step 110) จะใช้ **ค่า Default (10s/40s)** ทุก Link
> ไม่ปรับแต่งเพิ่ม เพื่อความเรียบง่าย แต่ต้องเข้าใจกรณี Mismatch นี้ให้แม่นสำหรับการ Troubleshoot
> จริงและข้อสอบ CCNA/CCNP ที่มักถามเรื่องนี้บ่อยมาก

---

## Step 107 — Passive-Interface: ทำไม SVI ของ VLAN ผู้ใช้งานไม่ควรส่ง Hello

### 107.1 ปัญหาที่เกิดขึ้นถ้าไม่ตั้ง Passive-Interface

DIST-SW1–4 มี SVI ของ VLAN ผู้ใช้งาน (เช่น Vlan10, Vlan20, Vlan30, Vlan40, Vlan99) ที่ต้อง
**Advertise Subnet เหล่านี้เข้า OSPF** เพื่อให้ Router อื่นรู้จัก Subnet นั้น — แต่ **ไม่ควรส่ง Hello
Packet ออกทาง SVI เหล่านี้เลย** เพราะ:

| เหตุผล | รายละเอียด |
|---|---|
| **ไม่มี Router อื่นรออยู่ฝั่งนั้น** | ปลาย VLAN คือ PC/Server ของผู้ใช้งานทั่วไป ไม่มี Router ใดที่จะตอบ Hello กลับมาเป็น Neighbor อยู่แล้ว — ส่งไปก็เสียเปล่า |
| **ความเสี่ยงด้าน Security** | ถ้ามีใครในองค์กร (ไม่ว่าจงใจหรือไม่) เอา Router/Switch ที่รัน OSPF ไปต่อเข้า VLAN ผู้ใช้งาน มันอาจ**สร้าง OSPF Adjacency ปลอมกับ DIST-SW ได้ทันที** และแทรก LSA ปลอมเข้า Topology จริงได้ — เป็นช่องโหว่ร้ายแรง |
| **สิ้นเปลือง Bandwidth/CPU โดยไม่จำเป็น** | Hello ทุก 10 วินาทีตลอดเวลาบน Interface ที่ไม่มีทางมี Neighbor เกิดขึ้นจริง |

### 107.2 คำสั่ง `passive-interface`

`passive-interface` ทำให้ Interface นั้น**หยุดส่ง (และรับ) Hello Packet** แต่**ยังคง
Advertise Subnet ของ Interface นั้นเข้า OSPF ตามปกติทุกประการ** (ไม่กระทบการ Advertise เลย
กระทบแค่การสร้าง Neighbor เท่านั้น) — นี่คือจุดที่คนสับสนบ่อย: Passive ≠ ปิด OSPF บน Interface,
Passive = "Advertise ได้ แต่ไม่คุยกับใคร"

```
! วิธีที่ 1: ระบุทีละ Interface (เหมาะกับ Case ที่มี Passive Interface น้อย)
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# passive-interface Vlan10
DIST-SW1(config-router)# passive-interface Vlan20
DIST-SW1(config-router)# passive-interface Vlan99
```

```
! วิธีที่ 2 (แนะนำ — ปลอดภัยกว่ามาก): ตั้ง Passive เป็น Default ทุก Interface ก่อน
! แล้วค่อยเปิดเฉพาะ Interface ที่รู้แน่ชัดว่าต้องมี Neighbor จริง (Transit Link) ด้วย "no passive-interface"
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# passive-interface default
DIST-SW1(config-router)# no passive-interface TenGigabitEthernet1/1/1
```

> **ทำไมวิธีที่ 2 ปลอดภัยกว่า**: วิธีที่ 1 ต้องคอยจำไปเปิด Passive ทุกครั้งที่เพิ่ม SVI ใหม่
> (ถ้าลืม = เปิดช่องโหว่ Security ทันที) วิธีที่ 2 เป็นแนวคิด **"Deny by Default, Allow by
> Exception"** — Interface ใหม่ใดๆ ที่เพิ่มเข้ามาทีหลัง (SVI ใหม่, VLAN ใหม่) จะ**เป็น Passive
> โดยอัตโนมัติทันที** โดยไม่ต้องจำไปตั้งเพิ่มเลย ต้อง**ตั้งใจเปิด (`no passive-interface`) เฉพาะ
> Link ที่รู้แน่ชัดว่าเป็น Transit Link ระหว่าง Router/Switas เท่านั้น** — **หลักสูตรนี้ใช้วิธีที่ 2
> เป็นมาตรฐานตั้งแต่ Step 110 เป็นต้นไป**

### 107.3 ตัวอย่างเต็ม — DIST-SW1 ก่อน/หลังตั้ง Passive-Interface

```
! ก่อนตั้ง Passive — SVI ส่ง Hello ออกทาง Vlan10/20/99 ด้วย (เห็นจาก show ip ospf interface)
DIST-SW1# show ip ospf interface Vlan10 | include Timer
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

! ตั้ง Passive-interface default + เปิดเฉพาะ Transit Link
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# passive-interface default
DIST-SW1(config-router)# no passive-interface TenGigabitEthernet1/1/1

! หลังตั้ง — Vlan10 ยัง Advertise Subnet ปกติ (ยังเห็นใน show ip ospf database)
! แต่ "No Hellos" แสดงชัดเจนใน show ip ospf interface
DIST-SW1# show ip ospf interface Vlan10 | include Hello|Passive
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    No Hellos (Passive interface)

! Uplink ไป Core ยังคง Active — เห็น Neighbor สร้างได้ปกติทาง Interface นี้เท่านั้น
DIST-SW1# show ip ospf interface TenGigabitEthernet1/1/1 | include Hello|Passive
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
```

`show ip route ospf` บน Router อื่น (เช่น DIST-SW3) จะยังเห็น Route ไปยัง `10.10.10.0/24` ปกติ
ทุกประการ ยืนยันว่า Passive-Interface **ไม่กระทบการ Advertise เลย** กระทบแค่การสร้าง Neighbor
Adjacency บน Interface นั้นเท่านั้น

---

## Step 108 — OSPF Authentication: Plaintext และ MD5

### 108.1 ทำไมต้องมี Authentication

OSPF โดย Default **ไม่มี Authentication เลย** — Router/Switch ใดๆ ที่ต่อเข้า Segment เดียวกันและรู้
Area ID ที่ถูกต้องจะสามารถส่ง Hello แล้วสร้าง Neighbor Adjacency ได้ทันที (ปัญหาเดียวกับที่กล่าวใน
Step 107.1) — Authentication ทำให้ Router ที่ไม่มี **Key ที่ถูกต้อง** ไม่สามารถสร้าง Adjacency ได้
เลย แม้จะเดา Area ID ถูกก็ตาม เป็นการป้องกันอีกชั้นที่สำคัญโดยเฉพาะบน **Backbone Link** ที่ถ้าถูก
แทรกแซง (Inject LSA ปลอม) จะกระทบทั้ง Topology

### 108.2 Plaintext Authentication (Type 1) — ทำความเข้าใจ ไม่แนะนำใช้จริง

```
DIST-SW1(config-if)# ip ospf authentication              ! เปิด Plaintext Authentication
DIST-SW1(config-if)# ip ospf authentication-key C1sco123  ! Key แบบ Cleartext
```

Plaintext Authentication ส่ง Key **แบบไม่เข้ารหัสเลยในทุก Hello Packet** ใครใช้ Packet Capture
ดักดูก็เห็น Key ตรงๆ — ใช้ป้องกันได้แค่ **Misconfiguration โดยไม่ตั้งใจ** เท่านั้น (เช่น กันไม่ให้
Interface ที่ Area/Network ผิดพลาดมาสร้าง Adjacency สลับกันโดยไม่ตั้งใจ) **ไม่ควรใช้เป็นมาตรการ
Security จริงในงาน Production เด็ดขาด**

### 108.3 MD5 Authentication (Type 2) — มาตรฐานที่ใช้จริง

MD5 Authentication **ไม่ส่ง Key ตรงๆ** แต่ใช้ Key ไปคำนวณ **MD5 Hash** ของทั้ง Packet แล้วส่ง Hash
นั้นแนบไปแทน — ฝั่งรับคำนวณ Hash ด้วย Key ของตัวเองแล้วเทียบ ถ้าตรงกันจึงยอมรับ Packet — Key จริง
**ไม่เคยถูกส่งผ่าน Network เลย**

```
DIST-SW1(config-if)# ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
DIST-SW1(config-if)# ip ospf authentication message-digest
```

- `1` คือ **Key ID** — ต้องตรงกันทั้งสองฝั่งของ Link เดียวกัน (ใช้เปลี่ยน Key แบบ Rolling ได้โดยไม่
  Drop Adjacency ทันที ด้วยการเพิ่ม Key ID ใหม่คู่กับตัวเก่าก่อนค่อยลบตัวเก่าทีหลัง)
- `md5 OSPF-BACKBONE-K3y!` คือ Key String จริง (Case-sensitive, ตรงกันทั้งสองฝั่งเสมอ)
- `ip ospf authentication message-digest` คือคำสั่งเปิดใช้ MD5 Mode บน Interface นั้น (แยกจาก
  คำสั่งตั้ง Key — ต้องมีทั้งสองคำสั่งจึงทำงาน)

### 108.4 อีกวิธี: เปิด Authentication ทั้ง Area พร้อมกัน (Area-wide)

นอกจากตั้งราย Interface แล้ว ยังตั้งแบบ **Area-wide** ได้ (สั่งครั้งเดียว บังคับทุก Interface ใน
Area นั้นต้องมี MD5 Key ของตัวเอง — แต่ยังต้องตั้ง `message-digest-key` รายอินเตอร์เฟซเหมือนเดิม):

```
DIST-SW1(config)# router ospf 1
DIST-SW1(config-router)# area 0 authentication message-digest
```

> **หลักสูตรนี้เลือกใช้วิธี Per-Interface (`ip ospf authentication message-digest`)** เพื่อความ
> ชัดเจนเห็นตรงๆ ว่า Interface ไหนมี Authentication เปิดอยู่จริงบ้าง — ทั้งสองวิธีให้ผลลัพธ์เดียวกัน

### 108.5 คอนฟิกจริง — MD5 บน CORE-SW1↔CORE-SW2 และ CORE↔DIST

ตามที่กำหนดใน IP Plan ให้เปิด MD5 Authentication บนทุก **Backbone Transit Link** (ไม่รวม SVI ที่
เป็น Passive อยู่แล้ว และไม่รวม WAN Link ไปยัง WAN-EDGE ในขอบเขตของ Step นี้) ใช้ Key เดียวกัน
ทั้ง Backbone เพื่อความง่ายในการดูแล (Key ID 1, Key String `OSPF-BACKBONE-K3y!`):

```
! ===== CORE-SW1 =====
CORE-SW1(config)# interface Port-channel1                       ! ↔ CORE-SW2
CORE-SW1(config-if)# ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
CORE-SW1(config-if)# ip ospf authentication message-digest
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/1              ! ↔ DIST-SW1
CORE-SW1(config-if)# ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
CORE-SW1(config-if)# ip ospf authentication message-digest
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/2              ! ↔ DIST-SW2
CORE-SW1(config-if)# ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
CORE-SW1(config-if)# ip ospf authentication message-digest
CORE-SW1(config-if)# exit

! ===== DIST-SW1 (ฝั่งตรงข้าม Te1/0/1) =====
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
DIST-SW1(config-if)# ip ospf authentication message-digest
```

> **ข้อระวังที่สำคัญที่สุด**: Key ID และ Key String **ต้องตรงกันทั้งสองฝั่งของ Link เดียวกันเป๊ะ**
> (Case-sensitive) — ถ้าไม่ตรง Neighbor จะไม่ก่อตัวเลย พร้อม Syslog แจ้งเตือนชัดเจน:
> ```
> %OSPF-4-ERRRCV: Received invalid packet: mismatch authentication Key from 10.255.10.2, TenGigabitEthernet1/0/1
> ```
> คำสั่ง Verify คือ `show ip ospf interface <if>` (ดูบรรทัด Authentication) และ `debug ip ospf adj`

---

## Step 109 — คำสั่ง Verify OSPF ที่ต้องใช้ทุกวัน

### 109.1 `show ip ospf neighbor` — เช็ค Adjacency ก่อนอื่นใดเสมอ

```
CORE-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:38    10.255.0.2      Port-channel1
1.1.1.11          0   FULL/  -        00:00:33    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:39    10.255.20.2     TenGigabitEthernet1/0/2
1.1.1.21          1   FULL/BDR        00:00:31    10.10.254.1     GigabitEthernet1/0/1
```

- **Pri = 0** บน Link ที่ตั้ง `ip ospf network point-to-point` ไว้แล้ว (ไม่มี DR/BDR ให้ Priority
  มีความหมาย) — ส่วน Link WAN (Gi1/0/1 ไป WAN-EDGE-1) ยังเป็น Broadcast ปกติจึงเห็น Priority
  และ Role (BDR) จริง
- **State = FULL/ -** (มีขีดกลาง แทน Role) หมายถึง Point-to-Point ไม่มี Role DR/BDR ให้แสดง
- ถ้าเห็น State ค้างที่ **2WAY/DROTHER** ตลอดบน Broadcast Segment ที่มี Router มากกว่า 2 ตัว
  **ไม่ใช่ปัญหา** — เป็นพฤติกรรมปกติ (DROTHER คุยกับ DROTHER ตัวอื่นแค่ 2-Way เท่านั้น ตาม Step
  105.3) แต่ถ้าค้างที่ **INIT** หรือ **EXSTART/EXCHANGE** นานเกินไป = มีปัญหาแน่นอน (Timer
  Mismatch หรือ Authentication Mismatch ตามที่กล่าวใน Step 106/108)

### 109.2 `show ip ospf interface` — เช็ค Timer, Cost, Network Type, Authentication ราย Interface

```
CORE-SW1# show ip ospf interface TenGigabitEthernet1/0/1
TenGigabitEthernet1/0/1 is up, line protocol is up
  Internet Address 10.255.10.1/30, Area 0, Attached via Network Statement
  Process ID 1, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 1
  Topology-MTID    Cost    Disabled    Shutdown      Topology Name
        0           1         no          no            Base
  Transmit Delay is 1 sec, State POINT_TO_POINT
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    oob-resync timeout 40
    Hello due in 00:00:07
  Supports Link-local Signaling (LLS)
  Cisco NSF helper support enabled
  IETF NSF helper support enabled
  Index 1/2, flag 0x0
  Silent interface (Passive interface) : No
  Cryptographic authentication enabled
    Youngest key id is 1
  Multi-area interface Count is 0
```

Field สำคัญที่ต้องเช็คทุกครั้ง: **Network Type** (ต้องตรงกับที่ตั้งใจ), **Cost**, **Timer
intervals**, **Silent interface (Passive)**, และบรรทัดสุดท้าย **Cryptographic authentication
enabled** (ยืนยันว่า MD5 ทำงานอยู่จริง)

### 109.3 `show ip ospf database` — ดู LSDB ทั้งหมดที่ Router เห็น

```
CORE-SW1# show ip ospf database

            OSPF Router with ID (1.1.1.1) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Link count
1.1.1.1         1.1.1.1         245         0x8000002A  4
1.1.1.2         1.1.1.2         312         0x80000027  4
1.1.1.11        1.1.1.11        198         0x80000015  4
1.1.1.12        1.1.1.12        201         0x80000013  4
1.1.1.13        1.1.1.13        350         0x80000012  4
1.1.1.14        1.1.1.14        355         0x80000011  4
1.1.1.21        1.1.1.21        410         0x8000000A  2
```

- ทุกแถวคือ **LSA Type 1 (Router LSA)** — Router ทุกตัวใน Area 0 Originate LSA ประเภทนี้เสมอ
  1 ฉบับต่อตัว บอก "ฉันมี Link อะไรบ้าง"
- **Age**: อายุ LSA (วินาที) — ถ้าครบ 3600s (MaxAge) โดยไม่มีการ Refresh จะถูกลบออกจาก LSDB
- **Seq#**: Sequence Number — เพิ่มขึ้นทุกครั้งที่ Router Originate LSA ฉบับใหม่ (กัน LSA เก่าที่
  วนกลับมาไม่ให้ Overwrite ฉบับใหม่กว่า)
- **Link count**: จำนวน Link ที่ Router นั้นมี (Point-to-Point + Stub Network รวมกัน)

### 109.4 `show ip route ospf` — ดู Route ที่เรียนรู้จาก OSPF เท่านั้น

```
CORE-SW1# show ip route ospf

      10.0.0.0/8 is variably subnetted, 14 subnets, 2 masks
O        10.10.10.0/24 [110/2] via 10.255.10.2, 00:12:04, TenGigabitEthernet1/0/1
O        10.10.20.0/24 [110/2] via 10.255.20.2, 00:12:04, TenGigabitEthernet1/0/2
O        10.10.30.0/24 [110/2] via 10.255.0.2, 00:12:04, Port-channel1
O        10.10.40.0/24 [110/2] via 10.255.0.2, 00:12:04, Port-channel1
O        10.10.99.0/24 [110/2] via 10.255.10.2, 00:12:04, TenGigabitEthernet1/0/1
O        10.10.99.0/24 [110/2] via 10.255.20.2, 00:12:04, TenGigabitEthernet1/0/2
      1.0.0.0/32 is subnetted, 6 subnets
O        1.1.1.11 [110/1] via 10.255.10.2, 00:12:04, TenGigabitEthernet1/0/1
O        1.1.1.12 [110/1] via 10.255.20.2, 00:12:04, TenGigabitEthernet1/0/2
```

- **`O`** = เรียนรู้จาก OSPF (แยกจาก `C` connected, `S` static)
- **`[110/2]`** = `[Administrative Distance/Cost]` — 110 คือ AD มาตรฐานของ OSPF เสมอ, 2 คือ Cost
  สะสม (SVI ปลายทาง Cost 1 + Link ที่ผ่านมา Cost 1 = 2)
- `10.10.99.0/24` มี**สอง Path เท่ากัน (ECMP)** เพราะทั้ง DIST-SW1 และ DIST-SW2 มี SVI Vlan99
  Subnet เดียวกัน (`10.10.99.0/24`) และ Cost เท่ากันทั้งสองทาง — OSPF จะ Load-balance ระหว่างสอง
  Path นี้โดยอัตโนมัติ (ต่างจาก Part 10 ที่ต้องแยก `10.10.99.4/31` ด้วยมือ ปัญหานี้หายไปเพราะ
  OSPF มองแค่ Cost ไม่สนใจว่ามาจาก Switch ตัวไหน)

### 109.5 `show ip protocols` — สรุปภาพรวม Process ทั้งหมด

```
CORE-SW1# show ip protocols
*** IP Routing is NSF aware ***

Routing Protocol is "ospf 1"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Router ID 1.1.1.1
  It is an area border router
  Number of areas in this router is 1. 1 normal 0 stub 0 nssa
  Maximum path: 4
  Routing for Networks:
  Passive Interface(s):
    Loopback0
  Routing Information Sources:
    Gateway         Distance      Last Update
    1.1.1.2              110      00:15:23
    1.1.1.11              110      00:15:19
    1.1.1.12              110      00:15:20
    1.1.1.21              110      00:04:02
  Distance: (default is 110)
```

Field ที่มักถูกถามในข้อสอบ: **Router ID**, **Maximum path** (ค่า Default OSPF ทำ ECMP ได้สูงสุด
4 Path พร้อมกัน — ปรับได้ด้วย `maximum-paths <n>`), **Distance: (default is 110)** และรายชื่อ
**Passive Interface(s)** ที่ตั้งไว้

---

## Step 110 — Lab เต็มรูปแบบ: แทนที่ Static Route ทั้ง Topology ด้วย OSPF Area 0

นี่คือ Lab สรุปของ Part 11 — นำทุกแนวคิดจาก Step 101–109 มาประกอบเป็นคอนฟิกจริงบน
**CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2, DIST-SW3, DIST-SW4** ทุกตัว โดย**ลบ Static Route
ทั้งหมดที่ [Part 10](part-010-inter-vlan-routing.md) ตั้งไว้ระหว่าง DIST↔CORE** แล้วแทนที่ด้วย
OSPF Area 0 เดี่ยว ครบทั้ง Loopback0, Passive-Interface บนทุก SVI, Network Type
Point-to-Point และ MD5 Authentication บน Backbone Link ทุกเส้นตามที่ออกแบบไว้

### 110.1 Topology เต็มของ Lab (พร้อม Router ID และ OSPF Cost)

```
                                   Loopback0 = Router ID (ทุกตัว)
                                   1.1.1.1 (CORE-SW1)   1.1.1.2 (CORE-SW2)
                                        │                      │
                          Po1 (10.255.0.0/30) MD5, P2P, Cost=1
                          CORE-SW1 .1 ═══════════════════ .2 CORE-SW2
                            │                                   │
              Te1/0/1 (10.255.10.0/30)             Te1/0/1 (10.255.30.0/30)
              MD5, P2P, Cost=1                     MD5, P2P, Cost=1
              CORE-SW1 .1 ──────── DIST-SW1 .2      CORE-SW2 .1 ──────── DIST-SW3 .2
              (1.1.1.11)  Te1/1/1                   (1.1.1.13)  Te1/1/1
                            │                                   │
              Te1/0/2 (10.255.20.0/30)             Te1/0/2 (10.255.40.0/30)
              MD5, P2P, Cost=1                     MD5, P2P, Cost=1
              CORE-SW1 .1 ──────── DIST-SW2 .2      CORE-SW2 .1 ──────── DIST-SW4 .2
              (1.1.1.12)  Te1/1/1                   (1.1.1.14)  Te1/1/1

  DIST-SW1 SVI (Passive):  Vlan10=10.10.10.2/24  Vlan20=10.10.20.2/24  Vlan99=10.10.99.2/24
  DIST-SW2 SVI (Passive):  Vlan10=10.10.10.3/24  Vlan20=10.10.20.3/24  Vlan99=10.10.99.3/24
  DIST-SW3 SVI (Passive):  Vlan30=10.10.30.2/24  Vlan40=10.10.40.2/24  Vlan99=10.10.99.4/24
  DIST-SW4 SVI (Passive):  Vlan30=10.10.30.3/24  Vlan40=10.10.40.3/24  Vlan99=10.10.99.5/24

  CORE-SW1 ── Gi1/0/1 (10.10.254.0/30, Broadcast ปกติ, ไม่ MD5 ในขอบเขต Step นี้) ── WAN-EDGE-1 (1.1.1.21)
  CORE-SW2 ── Gi1/0/1 (10.10.254.4/30, Broadcast ปกติ, ไม่ MD5 ในขอบเขต Step นี้) ── WAN-EDGE-2 (1.1.1.22)
```

### 110.2 Running-Config เต็ม: CORE-SW1

```
hostname CORE-SW1
!
ip routing
!
interface Loopback0
 description ** OSPF Router ID Anchor **
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW2 **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Routed downlink to DIST-SW1 **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Routed downlink to DIST-SW2 **
 no switchport
 ip address 10.255.20.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Link to WAN-EDGE-1 **
 ip address 10.10.254.2 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface TenGigabitEthernet1/0/1
 no passive-interface TenGigabitEthernet1/0/2
 no passive-interface GigabitEthernet1/0/1
!
! ===== ลบ Static Route เดิมของ Part 10 ทั้งหมดที่ชี้ไปยัง Subnet ฝั่ง DIST-SW3/4 =====
no ip route 10.10.30.0 255.255.255.0 10.255.0.2
no ip route 10.10.40.0 255.255.255.0 10.255.0.2
no ip route 10.10.99.4 255.255.255.254 10.255.0.2
```

### 110.3 Running-Config เต็ม: CORE-SW2

```
hostname CORE-SW2
!
ip routing
!
interface Loopback0
 description ** OSPF Router ID Anchor **
 ip address 1.1.1.2 255.255.255.255
 ip ospf 1 area 0
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW1 **
 no switchport
 ip address 10.255.0.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Routed downlink to DIST-SW3 **
 no switchport
 ip address 10.255.30.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Routed downlink to DIST-SW4 **
 no switchport
 ip address 10.255.40.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Link to WAN-EDGE-2 **
 ip address 10.10.254.6 255.255.255.252
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.2
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface TenGigabitEthernet1/0/1
 no passive-interface TenGigabitEthernet1/0/2
 no passive-interface GigabitEthernet1/0/1
!
! ===== ลบ Static Route เดิมของ Part 10 ทั้งหมดที่ชี้ไปยัง Subnet ฝั่ง DIST-SW1/2 =====
no ip route 10.10.10.0 255.255.255.0 10.255.0.1
no ip route 10.10.20.0 255.255.255.0 10.255.0.1
no ip route 10.10.99.2 255.255.255.254 10.255.0.1
```

### 110.4 Running-Config เต็ม: DIST-SW1

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
interface Loopback0
 description ** OSPF Router ID Anchor **
 ip address 1.1.1.11 255.255.255.255
 ip ospf 1 area 0
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.2 255.255.255.0
 ip ospf 1 area 0
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
 description ** Routed uplink to CORE-SW1 **
 no switchport
 ip address 10.255.10.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.11
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
!
! ===== ลบ Default Static Route เดิมของ Part 10 (แทนที่ด้วย OSPF Intra-area Route ทั้งหมด) =====
no ip route 0.0.0.0 0.0.0.0 10.255.10.1
```

### 110.5 Running-Config เต็ม: DIST-SW2

```
hostname DIST-SW2
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
interface Loopback0
 description ** OSPF Router ID Anchor **
 ip address 1.1.1.12 255.255.255.255
 ip ospf 1 area 0
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.3 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.3 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.3 255.255.255.0
 ip ospf 1 area 0
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
 description ** Routed uplink to CORE-SW1 **
 no switchport
 ip address 10.255.20.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.12
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
!
no ip route 0.0.0.0 0.0.0.0 10.255.20.1
```

### 110.6 Running-Config เต็ม: DIST-SW3

```
hostname DIST-SW3
!
ip routing
!
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
interface Loopback0
 description ** OSPF Router ID Anchor **
 ip address 1.1.1.13 255.255.255.255
 ip ospf 1 area 0
!
interface Vlan30
 description ** Gateway VLAN30-SERVERS **
 ip address 10.10.30.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan40
 description ** Gateway VLAN40-WIFI **
 ip address 10.10.40.2 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT (block 2) **
 ip address 10.10.99.4 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Trunk to ACCESS-SW3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40,99
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Trunk to ACCESS-SW4 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40,99
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW2 **
 no switchport
 ip address 10.255.30.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.13
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
!
no ip route 0.0.0.0 0.0.0.0 10.255.30.1
```

### 110.7 Running-Config เต็ม: DIST-SW4

```
hostname DIST-SW4
!
ip routing
!
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
interface Loopback0
 description ** OSPF Router ID Anchor **
 ip address 1.1.1.14 255.255.255.255
 ip ospf 1 area 0
!
interface Vlan30
 description ** Gateway VLAN30-SERVERS **
 ip address 10.10.30.3 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan40
 description ** Gateway VLAN40-WIFI **
 ip address 10.10.40.3 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT (block 2) **
 ip address 10.10.99.5 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Trunk to ACCESS-SW3 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40,99
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Trunk to ACCESS-SW4 **
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,40,99
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW2 **
 no switchport
 ip address 10.255.40.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 no shutdown
!
router ospf 1
 router-id 1.1.1.14
 passive-interface default
 no passive-interface TenGigabitEthernet1/1/1
!
no ip route 0.0.0.0 0.0.0.0 10.255.40.1
```

### 110.8 ส่วนเสริม: WAN-EDGE-1/WAN-EDGE-2 เข้าร่วม Area 0 ด้วย (แทนที่ Static เดิมของ Part 9)

Static Route ที่เชื่อม CORE↔WAN-EDGE จาก [Part 9](part-009-static-routing.md) ก็ถูกแทนที่ด้วย
OSPF เช่นกัน — WAN-EDGE-1/2 เข้าร่วม Area 0 ผ่าน Transit Link เดิม (Loopback0 + Gi0/0/1 ตั้ง
`ip ospf 1 area 0`, `router-id` ตรงกับ Loopback0) แล้วใช้ `default-information originate` ใต้
`router ospf 1` เพื่อฉีด Default Route (ทางออก Internet ผ่าน ISP-RTR ที่ยังเป็น Static เดิม เพราะ
ISP-RTR เป็นแค่อุปกรณ์จำลอง ไม่ได้อยู่ใน OSPF Domain) เข้า Area 0 แทนการเขียน Static Default
Route ไว้ที่ CORE-SW1/CORE-SW2 ด้วยมือ — คำสั่งนี้สร้าง **Default Route LSA (Type 5, External)**
โดย WAN-EDGE-1 **ต้องมี** `0.0.0.0/0` อยู่ใน Routing Table ของตัวเองอยู่แล้วก่อน (จาก Static
Route ไป ISP-RTR) จึงจะยอม Originate ต่อ (ถ้าต้องการฉีดแม้ไม่มี Default Route ของตัวเองใช้
Keyword เพิ่ม `always`) ตั้งเดียวกันทำที่ WAN-EDGE-2 (Router ID `1.1.1.22`) เพื่อให้มีสองทางออก
Internet พร้อม Redundancy อัตโนมัติผ่าน OSPF Cost แทน Floating Static AD เดิม

### 110.9 Verify — `show ip ospf neighbor` ที่ทุกจุดต้องขึ้น FULL

```
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:35    10.255.0.2      Port-channel1
1.1.1.11          0   FULL/  -        00:00:31    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:38    10.255.20.2     TenGigabitEthernet1/0/2
1.1.1.21          1   FULL/BDR        00:00:33    10.10.254.1     GigabitEthernet1/0/1

CORE-SW2# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:00:32    10.255.0.1      Port-channel1
1.1.1.13          0   FULL/  -        00:00:36    10.255.30.2     TenGigabitEthernet1/0/1
1.1.1.14          0   FULL/  -        00:00:39    10.255.40.2     TenGigabitEthernet1/0/2
1.1.1.22          1   FULL/BDR        00:00:29    10.10.254.5     GigabitEthernet1/0/1

DIST-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:00:37    10.255.10.1     TenGigabitEthernet1/1/1
```

ทุก Neighbor ขึ้น **FULL** ครบ — ยืนยันว่า MD5 Authentication (Step 108) และ Network Type
Point-to-Point (Step 105) ทำงานถูกต้องตรงกันทั้งสองฝั่งของทุก Link

### 110.10 Verify — `show ip route` เห็น VLAN Subnet ครบทั่ว Topology ด้วย Code `O`

```
DIST-SW3# show ip route
Codes: L - local, C - connected, S - static, R - RIP, O - OSPF, ...

Gateway of last resort is 1.1.1.21 to network 0.0.0.0

O*E2    0.0.0.0/0 [110/1] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
      10.0.0.0/8 is variably subnetted, 12 subnets, 3 masks
O        10.10.10.0/24 [110/3] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        10.10.20.0/24 [110/3] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
C        10.10.30.0/24 is directly connected, Vlan30
L        10.10.30.2/32 is directly connected, Vlan30
C        10.10.40.0/24 is directly connected, Vlan40
L        10.10.40.2/32 is directly connected, Vlan40
C        10.10.99.0/24 is directly connected, Vlan99
L        10.10.99.4/32 is directly connected, Vlan99
O        10.255.0.0/30 [110/2] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
C        10.255.30.0/30 is directly connected, TenGigabitEthernet1/1/1
L        10.255.30.2/32 is directly connected, TenGigabitEthernet1/1/1
O        10.255.10.0/30 [110/3] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        10.255.20.0/30 [110/3] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        10.255.40.0/30 [110/2] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
      1.0.0.0/32 is subnetted, 6 subnets
O        1.1.1.1 [110/2] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        1.1.1.2 [110/1] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        1.1.1.11 [110/3] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        1.1.1.12 [110/3] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
O        1.1.1.14 [110/2] via 10.255.30.1, 00:22:10, TenGigabitEthernet1/1/1
```

- `O*E2 0.0.0.0/0` = Default Route ที่เรียนรู้จาก OSPF แบบ **External Type 2 (E2)** — มาจาก
  `default-information originate` ที่ WAN-EDGE-1 ฉีดเข้ามา (Step 110.8) เครื่องหมาย `*` หมายถึง
  เป็น **Gateway of Last Resort** ปัจจุบัน — DIST-SW3 ตอนนี้**ไม่มี Static Default Route เหลืออยู่
  เลย** ทุกอย่างเรียนรู้จาก OSPF ทั้งหมด
- เห็น Subnet ของทุก VLAN ทั่ว Topology (`10.10.10.0/24`, `10.10.20.0/24` ที่อยู่ไกลสุดฝั่ง
  DIST-SW1/2) ครบถ้วนด้วย Code `O` แม้ DIST-SW3 เองไม่มี VLAN เหล่านี้อยู่บนตัวเลยก็ตาม — พิสูจน์
  ว่า OSPF เรียนรู้ Topology ทั้งหมดโดยอัตโนมัติ ตรงข้ามกับ Static Route ของ Part 10 ที่ต้องเขียน
  ทีละบรรทัดด้วยมือทุกจุด

### 110.11 Verify — ping ข้าม Topology ยืนยันผลลัพธ์สุดท้าย

```
DIST-SW1# ping 10.10.30.2 source vlan10

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.30.2, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.2
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms

DIST-SW1# traceroute 10.10.30.2

Type escape sequence to abort.
Tracing the route to 10.10.30.2

  1 10.255.10.1 1 msec 1 msec 1 msec        <- CORE-SW1 (Te1/0/1)
  2 10.255.0.2 1 msec 1 msec 2 msec         <- CORE-SW2 (Po1) ข้าม Backbone
  3 10.10.30.2 2 msec 2 msec 1 msec         <- ถึง DIST-SW3 (ปลายทาง)
```

Path ที่ได้เหมือนกับ Static Route เดิมใน Part 10 เป๊ะ (DIST-SW1 → CORE-SW1 → CORE-SW2 →
DIST-SW3) แต่ครั้งนี้**ไม่มี Static Route แม้แต่บรรทัดเดียวเหลืออยู่ในทั้ง Topology** — ทุกเส้นทาง
คำนวณโดย OSPF SPF Algorithm อัตโนมัติทั้งหมด และถ้า Backbone Link เส้นใดล่มไป OSPF จะคำนวณ
เส้นทางใหม่ให้เองภายในไม่กี่วินาที โดยไม่ต้องมีใครเข้าไปแก้ Config เลย — นี่คือข้อได้เปรียบหลักของ
Dynamic Routing เทียบกับ Static Route ตามที่ Part 10 ทิ้งปัญหาไว้

---

## แบบฝึกหัดทวนความเข้าใจ Part 11

1. เพราะเหตุใด OSPF (Link-State) จึงไม่มีปัญหา Routing Loop เหมือนที่ Distance-Vector Protocol
   รุ่นเก่าเจอ และ Router ที่รัน OSPF แต่ละตัวคำนวณ Routing Table ของตัวเองด้วยวิธีใด?
2. ถ้า `show ip ospf neighbor` แสดง Neighbor ค้างอยู่ที่ State **INIT** ตลอดเวลาไม่ขยับไป 2-Way
   สาเหตุที่เป็นไปได้มากที่สุดคืออะไร และแตกต่างจากกรณีค้างที่ **EXSTART/EXCHANGE** อย่างไร?
3. เพราะเหตุใด Routed Port (Ethernet, `no switchport`) ที่เชื่อม DIST-SW1↔CORE-SW1 แบบ
   Point-to-Point จริงทางกายภาพ จึงยังต้องตั้งคำสั่ง `ip ospf network point-to-point` เพิ่มเติม
   ทั้งที่มี Router แค่ 2 ตัวคุยกัน?
4. คำสั่ง `passive-interface default` ร่วมกับ `no passive-interface <interface>` มีข้อดีอย่างไร
   เทียบกับการไล่ใส่ `passive-interface` ทีละ Interface ทีละใบ และ Passive-Interface กระทบการ
   Advertise Subnet ของ OSPF หรือไม่?
5. `show ip route ospf` แสดง Route หนึ่งเป็น `O 10.10.99.0/24 [110/2]` — ตัวเลข `110` และ `2` แต่ละ
   ตัวหมายถึงอะไร และถ้า Path เดียวกันแสดงสอง Entry ที่ Cost เท่ากันพอดี จะเกิดอะไรขึ้น?

**เฉลย:**
1. เพราะ Router ทุกตัวใน Area เดียวกันมี **Link-State Database (LSDB) ที่เหมือนกันทุกตัว** และแต่
   ละตัวคำนวณ **SPF Tree ของตัวเอง** ด้วย Dijkstra Algorithm จาก LSDB ชุดเดียวกัน (ไม่ใช่การเชื่อ
   คำบอกของ Neighbor ทอดต่อทอดแบบ Distance-Vector) ทำให้ไม่มี Router ตัวใดมี "มุมมอง Topology"
   ที่ขัดแย้งกัน — Loop จึงไม่เกิดขึ้นในทางทฤษฎี
2. State ค้างที่ INIT ตลอด มักเกิดจาก **Hello/Dead Timer Mismatch** หรือ **Authentication
   Mismatch** — ฝั่งรับปฏิเสธ Hello Packet ตั้งแต่ขั้นต้นเพราะ Parameter ไม่ตรงกัน ต่างจากค้างที่
   EXSTART/EXCHANGE ซึ่งมักเกิดจากปัญหาระหว่างขั้น Database Synchronization เช่น MTU Mismatch
   (DBD ถูกปฏิเสธเพราะขนาด Packet ไม่ตรงกัน) — ทั้งสองกรณีต้องเช็คคนละจุดกัน
3. เพราะ OSPF ตัดสิน Network Type จาก**ประเภท Interface** (Ethernet ทุกชนิด Default เป็น
   "Broadcast" เสมอ ไม่ว่าจะมี Router ต่ออยู่กี่ตัวจริง) ไม่ได้ตัดสินจากจำนวน Router ที่ต่ออยู่จริง
   ถ้าไม่ตั้ง `ip ospf network point-to-point` OSPF จะยังคง DR/BDR Election โดยไม่จำเป็น
   (แม้ Link จะมีแค่ 2 ตัว) ทำให้ Adjacency ก่อตัวช้าลงและมี LSA Type 2 เกิดขึ้นโดยไม่มีประโยชน์
4. ข้อดี: ไม่ต้องจำไปเปิด Passive ทุกครั้งที่เพิ่ม SVI/VLAN ใหม่ (ปลอดภัยกว่าตามหลัก "Deny by
   Default, Allow by Exception") ลดความเสี่ยงลืมเปิด Passive บน Interface ที่หันเข้าหา VLAN
   ผู้ใช้งาน — และ **ไม่กระทบการ Advertise Subnet เลย** Passive-Interface หยุดแค่การส่ง/รับ Hello
   Packet (จึงไม่มี Neighbor เกิดขึ้นบน Interface นั้น) แต่ Subnet ของ Interface ยังถูก Advertise
   เข้า OSPF ตามปกติทุกประการ
5. `110` คือ **Administrative Distance** มาตรฐานของ OSPF (ค่า Default เสมอ ไม่เปลี่ยนตาม Cost)
   ส่วน `2` คือ **OSPF Cost สะสม** ของเส้นทางนั้น (บวกรวม Cost ของทุก Outgoing Interface ตลอดเส้น
   ทาง) — ถ้ามีสอง Path ที่ Cost เท่ากันพอดี OSPF จะทำ **ECMP (Equal-Cost Multi-Path)** ใส่ทั้งสอง
   Path ลง Routing Table พร้อมกัน (ค่า Default รองรับสูงสุด 4 Path พร้อมกัน ปรับได้ด้วย
   `maximum-paths`) และ Load-balance Traffic ระหว่างสอง Path นั้นโดยอัตโนมัติ

---

## สรุป Part 11

Part นี้แทนที่ Static Route ทุกบรรทัดที่เชื่อม **DIST-SW1–4 ↔ CORE-SW1/CORE-SW2** และ
**CORE-SW1/CORE-SW2 ↔ WAN-EDGE-1/2** จาก [Part 9](part-009-static-routing.md) และ
[Part 10](part-010-inter-vlan-routing.md) ด้วย **OSPFv2 Area 0 เดี่ยว** ทั้งหมด — เราเรียนรู้ทฤษฎี
Link-State และ Dijkstra SPF, OSPF Cost Metric, Neighbor State Machine ครบ 7 State, กฎการเลือก
Router ID, สอง Syntax การคอนฟิก (`network` Statement กับ `ip ospf area` Per-Interface), Network
Type Point-to-Point vs Broadcast พร้อม DR/BDR Election, Timer และปัญหา Mismatch,
Passive-Interface สำหรับความปลอดภัยของ SVI, และ MD5 Authentication บน Backbone Link ทุกเส้น
ปิดท้ายด้วย Lab เต็มรูปแบบที่พิสูจน์ว่า **ทุก VLAN Subnet ทั่ว Topology เรียนรู้กันเองผ่าน OSPF
ได้ครบถ้วน โดยไม่มี Static Route เหลืออยู่แม้แต่บรรทัดเดียว** ✅

อย่างไรก็ตาม Part นี้ยังใช้ **Single Area (Area 0)** เท่านั้น ซึ่งเหมาะกับ Topology ขนาดเล็กแบบ
Lab หลักสูตรนี้ — เมื่อ Enterprise Network ขยายตัวใหญ่ขึ้น (Router หลักร้อยตัว, LSDB ใหญ่จนกิน CPU
มากเกินไปสำหรับ SPF Calculation ทุกครั้งที่มีการเปลี่ยนแปลง) จะต้องแบ่ง OSPF ออกเป็น **Multi-Area**
พร้อมแนวคิดใหม่ๆ อย่าง Area Border Router (ABR), LSA Type 3/4/5/7, Route Summarization ระหว่าง
Area, และ Stub/Totally-Stub/NSSA Area ซึ่งเป็นหัวข้อทั้งหมดของ **Part 12 — OSPFv2 Advanced**

**ไปต่อ:** [Part 12 — OSPFv2 Advanced →](part-012-ospfv2-advanced.md)
