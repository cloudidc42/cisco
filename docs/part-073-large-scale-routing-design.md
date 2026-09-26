# Part 73 — Large-Scale Routing Design (OSPF+BGP+EIGRP Integration)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 721–730 จาก 1000**

> ต่อจาก [Part 72 — Large-Scale Campus Fabric Design](part-072-large-campus-fabric-design.md)
> ที่ขยาย Enterprise Campus เดิม (Building เดียวจาก Part 1-35) ให้กลายเป็น **10-Building
> Large-Scale Campus** — Part 72 ตอบคำถามระดับ Fabric/Physical/Overlay Design ไปแล้ว
> ว่าจะวาง Building แต่ละแห่งอย่างไร จะเชื่อม Building เข้าด้วยกันด้วย Physical Topology
> แบบไหน (WAN Core Ring/Hub) **แต่ยังไม่ได้ตอบคำถามที่สำคัญที่สุดของ CCIE Enterprise
> Infrastructure**: เมื่อมี 10 Building ที่แต่ละแห่งมีประวัติ, ผู้จำหน่าย, และขนาดต่างกัน —
> **จะออกแบบ Routing Architecture ทั้งระบบอย่างไรให้ Scale ได้จริง ไม่ใช่แค่เอา OSPF/EIGRP/BGP
> ที่เรียนแยกกันมาตั้งแต่ Part 11-31 มาปะติดปะต่อกันแบบสุ่มๆ**

> **Part นี้คือจุดที่ CCIE Enterprise Infrastructure แตกต่างจาก CCNP ENCOR อย่างชัดเจนที่สุด**:
> CCNP สอนให้ Config แต่ละ Protocol ให้ถูกต้อง (Part 26 OSPF Advanced, Part 28 EIGRP Advanced,
> Part 30 BGP Advanced, Part 31 Redistribution) ส่วน CCIE สอนให้ **ออกแบบว่าทำไมต้องใช้ Protocol
> ไหนที่จุดไหน และทำไมการเลือกผิดจะทำให้ระบบพังตอนขยายสเกล** — Part นี้จะไม่ทวนซ้ำ Syntax พื้นฐาน
> ของ Part 26/28/30/31 แต่จะใช้ความรู้เหล่านั้นเป็น "วัตถุดิบ" ในการออกแบบสถาปัตยกรรม Routing
> เต็มรูปแบบของ 10-Building Campus ตั้งแต่ระดับ Decision Framework จนถึง LLD Document ที่ใช้งาน
> จริงได้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 721 | เลือก IGP ให้ถูกกับสเกล — Decision Framework แบบ Decision-Tree เต็มรูปแบบ |
| 722 | ออกแบบ OSPF Area สำหรับ 10-Building Campus — ABR Placement และกฎ ~50 Router/Area |
| 723 | BGP ในบทบาท "Supernet Glue" ของ Enterprise — ทางเลือกที่ดีกว่า Mutual Redistribution |
| 724 | Full Worked Design — 10-Building Campus พร้อม iBGP/eBGP Building-Edge สู่ WAN-Core RR |
| 725 | Route Summarization ทุกขอบเขต — เข้า/ออก BGP และ IGP |
| 726 | Redundancy & Convergence Design — วิเคราะห์ตัวเลขจริง IGP vs BGP |
| 727 | Multi-homing Building-to-WAN-Core — ต่อยอด Dual-homed Pattern จาก Part 33 |
| 728 | จัดการ Asymmetric Bandwidth — Cost/Metric Tuning ข้าม Link ความเร็วต่างกันมาก |
| 729 | Documentation & Change Management — ทำไม LLD จำเป็นยิ่งกว่าเดิมที่สเกลนี้ |
| 730 | Lab เต็มรูปแบบ: เอกสาร Routing Architecture ครบชุดของ 10-Building Campus |

---

## Step 721 — เลือก IGP ให้ถูกกับสเกล: Decision Framework แบบ Decision-Tree เต็มรูปแบบ

### 721.1 ทำไมคำถามนี้ถึงสำคัญกว่าที่คิด

ตลอด Part 11-31 หลักสูตรนี้ใช้ **OSPF เป็น IGP หลัก** ของ Campus (Part 11-12, 26-27) และใช้
**EIGRP เป็น Legacy Protocol** สำหรับ Site ที่ถูกควบรวมเข้ามาใหม่ (Part 13, 28) ส่วน **BGP**
ถูกจำกัดบทบาทไว้แค่ WAN Edge สำหรับคุยกับ ISP (Part 29-30) — นี่คือ Pattern ที่ถูกต้องสำหรับ
Campus **1 อาคาร** แต่คำถามที่ CCIE ต้องตอบได้คือ: **เมื่อ Campus ขยายเป็น 10 อาคารตาม Part 72
Pattern เดิมนี้ยังใช้ได้อยู่ไหม?** คำตอบคือ **ใช้ได้บางส่วน แต่ต้องปรับบทบาทของ BGP ใหม่
ทั้งหมด** — และ Step นี้จะอธิบายว่าทำไม ด้วย Decision Framework ที่ไม่ได้ตัดสินจาก "Protocol
ไหนเก่งกว่า" (ซึ่งเป็นคำถามที่ผิดตั้งแต่ต้น) แต่ตัดสินจาก **บริบทของแต่ละ Routing Domain**

### 721.2 Decision Tree เต็มรูปแบบ

```
                    ┌─────────────────────────────────┐
                    │  Routing Domain นี้มีกี่ Router?    │
                    └────────────┬──────────────────────┘
                                 │
              ┌──────────────────┼──────────────────────┐
              │                  │                       │
        < ~50 Router       50-200 Router           > 200 Router หรือ
       (1 Building/         (Building ใหญ่/       รวมหลาย Domain เข้าด้วยกัน
        1 Wing)              หลาย Wing)            (Multi-Site/Multi-AS)
              │                  │                       │
              ▼                  ▼                       ▼
    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐
    │ All-Cisco จริง?    │  │ Multi-vendor      │  │  ห้ามใช้ Link-State    │
    └─────┬────────┬───┘  │ หรือต้อง Hierarchy │  │  Domain เดียวคลุมหมด   │
     ใช่   │        │ ไม่   │ ที่ Standardize?   │  │  → ใช้ BGP เป็น         │
          ▼        ▼      └─────┬────────────┘  │  "Supernet Glue"       │
     ┌────────┐ ┌────────┐       │ ใช่            │  ระหว่าง Domain          │
     │ EIGRP  │ │ OSPF   │       ▼                │  (ดู Step 723)          │
     │ (ง่าย, │ │(มาตรฐาน)│  ┌────────┐            └──────────────────────┘
     │ Convergence│ │       │ OSPF   │
     │ เร็วด้วย FS)│ │       │ Multi- │
     └────────┘ └────────┘ │ Area   │
                            │(Step722)│
                            └────────┘
```

### 721.3 ตารางเกณฑ์การตัดสินใจแบบละเอียด — ใช้แทนการดูแค่ "Protocol ไหนดีกว่า"

| เกณฑ์ | เลือก EIGRP | เลือก OSPF | เลือก BGP (เป็น "IGP" ของ Enterprise) |
|---|---|---|---|
| จำนวน Router ต่อ Domain | < 50 | 50-200 (แบ่ง Area ได้) | ไม่จำกัด — คือเหตุผลที่เลือกใช้ |
| ความเป็น Multi-vendor | ต่ำ (All-Cisco) | สูง/ปานกลาง (Standard RFC) | ไม่เกี่ยวข้อง (ใช้ระหว่าง Domain ไม่ใช่ภายใน) |
| ต้องการ Fast Convergence ภายใน Domain | ดีมาก (DUAL + Feasible Successor ไม่ต้องรอ SPF) | ดี (ต้อง Tune BFD/SPF Throttle ตาม Part 26) | แย่ (ดู Step 726 — ไม่เหมาะกับงานนี้) |
| จำนวน Domain ที่ต้องเชื่อมเข้าด้วยกัน | ไม่เกี่ยวข้อง (เรื่องภายใน Domain เดียว) | ไม่เกี่ยวข้อง | **นี่คือจุดแข็งที่สุด** — ยิ่งมี Domain มาก ยิ่งควรใช้ BGP เชื่อม |
| ต้องการ Policy Control ระดับองค์กร (Traffic Engineering ข้าม Site) | ไม่มีกลไกที่เหมาะ | ไม่มีกลไกที่เหมาะ (Cost ปรับได้แต่ไม่ใช่ Policy) | **มีในตัว** (LOCAL_PREF/Community จาก Part 30) |
| ความเสี่ยง Routing Loop เมื่อรวมหลาย Domain | สูง (ต้องพึ่ง Route Tag แบบ Part 31 Step 307) | สูง (เหมือนกัน) | **ต่ำมาก** (AS_PATH กันลูปในตัว โดยไม่ต้องออกแบบ Tag เอง) |
| ประวัติ M&A (Site ที่ควบรวมมาพร้อม IGP เดิม) | เก็บ Protocol เดิมไว้ได้ถ้า Domain เล็ก | ใช้เป็น Standard เมื่อ Build ใหม่ | ใช้เป็น "เส้นแบ่งเขต" ระหว่าง Site เดิม-ใหม่โดยไม่ต้องแปลง IGP |

> **ข้อสรุปที่สำคัญที่สุดของ Step นี้**: คำถามที่ถูกต้องไม่ใช่ "OSPF vs EIGRP vs BGP อันไหนดี
> ที่สุด" (เป็นคำถามที่ไม่มีคำตอบเดียว) แต่คือ **"แต่ละ Domain ควรมีขนาดแค่ไหน และควรมีกี่
> Domain"** — เมื่อตอบคำถามนี้ได้ การเลือก Protocol ในแต่ละ Domain จะตามมาเองจากตารางข้างบน
> ส่วน**ระหว่าง** Domain นั้น คำตอบมักจะเป็น BGP เสมอเมื่อจำนวน Domain มากกว่า 3-4 Domain — นี่คือ
> แนวคิดที่ Part 30 Step 293.4 ได้ทิ้งคำใบ้ไว้แล้วตั้งแต่ตอนที่บอกว่า **"ลองจินตนาการว่า
> Enterprise นี้ขยายเป็น 10 Site"** — ตอนนี้เรามาถึงจุดนั้นจริงแล้ว

### 721.4 นำ Decision Tree มาใช้กับ 10-Building Campus ของ Part 72

| ปัจจัยของ Campus 10 Building | ผลการวิเคราะห์ |
|---|---|
| Building 1 (HQ เดิมจาก Part 1-35) | ใหญ่ที่สุด, All-Cisco, มี Area 1/2 อยู่แล้ว → **คง OSPF Area 0/1/2 เดิมไว้ทั้งหมด ไม่แตะ** |
| Building 2-8 (สร้างใหม่ทั้งหมดตาม Part 72) | All-Cisco, ขนาดกลาง (~30-60 Router/อาคาร) → **OSPF 1 Process ต่ออาคาร (Single Area 0 ภายใน)** — เลือก OSPF แทน EIGRP เพราะต้องการ Hierarchy ที่ Standardize และเผื่อ Multi-vendor ในอนาคต (Data Center/Firewall Vendor อื่นมักรองรับ OSPF มากกว่า EIGRP) |
| Building 9-10 (M&A ล่าสุด — สมมติชื่อ Site "RIVER-SIDE" และ "BKK-SOUTH") | มาพร้อม EIGRP AS 200 ที่ทำงานดีอยู่แล้ว, ทีมงานเดิมคุ้นเคย EIGRP | **ไม่แปลงเป็น OSPF ทันที** (Rip-and-Replace เสี่ยง Outage โดยไม่จำเป็น) — เก็บ EIGRP AS 200 ไว้เป็น Domain ของตัวเอง |
| จำนวน Domain รวมทั้งหมด | 10 Domain (Building 1-10) + WAN Core | **> 3-4 Domain → ใช้ BGP เชื่อมทุก Domain แทน Mutual Redistribution** (รายละเอียดใน Step 723-724) |

ผลลัพธ์: **มี IGP 2 ชนิดอยู่ในองค์กรเดียวกันจริง** (OSPF 9 Domain + EIGRP 2 Domain) — ซึ่งเป็น
สถานการณ์ปกติมากในองค์กรจริงที่มีประวัติ M&A และเป็นเหตุผลที่ **BGP-glue Design ใน Step 723
เหมาะกับสถานการณ์นี้อย่างยิ่ง** เพราะไม่สนใจว่า Domain ฝั่งตรงข้ามใช้ IGP อะไรเลย — BGP เห็นแค่
"Prefix ที่ถูก Advertise เข้ามา" ไม่ต้องรู้จัก Metric ภายในของอีกฝั่งแม้แต่นิดเดียว

---

## Step 722 — ออกแบบ OSPF Area สำหรับ 10-Building Campus: ABR Placement และกฎ ~50 Router/Area

### 722.1 ออกแบบแบบ "Naive" ก่อน — OSPF Domain เดียวคลุมทั้ง 10 อาคาร

ก่อนจะไปถึง Step 723 (คำตอบที่ CCIE แนะนำจริง) ต้องเข้าใจก่อนว่าถ้า**ยังดึงดันใช้ OSPF Domain
เดียวคลุมทั้ง Campus** (แนวทางที่วิศวกรที่ไม่มีพื้นฐาน CCIE มักทำ) จะออกแบบ Area อย่างไร และจะ
พังตรงไหน — Pattern ที่พบมากที่สุดคือ **1 Area ต่อ 1 Building**:

```
                         ┌─────────────────────────────┐
                         │   Area 0 (WAN Core Backbone)  │
                         │   เชื่อม Building-Edge ทุกคู่   │
                         └───┬───┬───┬───┬───┬───┬───┬──┘
                    Area1│Area2│Area3│...│Area8│Area9│Area10
                         │    │    │       │    │    │
                    ┌────┴┐ ┌─┴──┐┌┴───┐  ┌┴───┐┌┴───┐┌┴────┐
                    │Bld 1│ │Bld2││Bld3│..│Bld8││Bld9││Bld10│
                    └─────┘ └────┘└────┘  └────┘└────┘└─────┘
                    (ABR = Building-Edge Router คู่ละอาคาร)
```

| Area | Building | ABR (2 ตัว/อาคาร เพื่อ Redundancy) | จำนวน Router ประมาณ |
|---|---|---|---|
| Area 0 | WAN Core (ไม่มี Building เอง) | WAN-CORE-RTR1/2 | 2-4 |
| Area 1 | Building 1 (HQ เดิม) | WAN-EDGE-1, WAN-EDGE-2 | ~25 (CORE/DIST/ACCESS เดิม) |
| Area 2-8 | Building 2-8 | BLDx-EDGE1, BLDx-EDGE2 | ~30-60/อาคาร |
| Area 9-10 | Building 9-10 | *(EIGRP — ไม่เข้า OSPF เลย)* | — |

### 722.2 กฎ ~50 Router/Area — มาจากไหน และทำไมยังสำคัญแม้จะแบ่ง Area แล้ว

Cisco และ Operator จริงจำนวนมากใช้ **"ไม่เกิน 50 Router ต่อ Area"** เป็น Rule of Thumb (ไม่ใช่
Hard Limit ตาม RFC 2328 — Protocol ไม่ได้บังคับเลข 50 ตรงๆ) เหตุผลเชิงเทคนิคที่อยู่เบื้องหลัง:

| ปัจจัย | ผลกระทบเมื่อ Area ใหญ่เกินไป |
|---|---|
| ขนาด LSDB ต่อ Router | Router ทุกตัวใน Area เดียวกันต้องเก็บ Type 1/2 LSA ของทุก Router/Link ใน Area นั้นครบ — Memory และ CPU สำหรับ Flooding เพิ่มเป็นเส้นตรงตามจำนวน Router |
| ความซับซ้อนของ SPF (Dijkstra) | Full SPF มี Complexity ประมาณ O(E log V) (E=Link, V=Router) — ยิ่ง Area ใหญ่ ยิ่งช้าต่อรอบ แม้จะมี SPF Throttle (Part 26 Step 254) ช่วยคุมความถี่ แต่ **เวลาต่อรอบยังเพิ่มขึ้นตามขนาด Area** |
| Blast Radius ของ 1 Link Flap | Router ทุกตัวใน Area เดียวกัน (ไม่ใช่แค่ 2 ฝั่งของ Link) ต้องรัน SPF ใหม่ทุกครั้งที่มี Link ไหนใน Area Flap — Area ใหญ่ = จำนวน Router ที่ถูกกระทบต่อ 1 Event มากขึ้น |
| Low-end Platform (Access-Layer Switch) | Catalyst 9200/2960 (ตาม Device Platform Reference) มี CPU/Memory จำกัดกว่า Core — Area ใหญ่เกินอาจทำให้ Access Switch ที่เป็น Internal Router ธรรมดาในบางอาคาร (ไม่ใช่ ABR) ประมวลผล SPF ไม่ทันในสถานการณ์ Flapping ถี่ |

**สรุปเชิงปฏิบัติ**: แต่ละ Building ใน Design นี้มี ~30-60 Router — **Building ส่วนใหญ่ (2-7)
ยังอยู่ในเกณฑ์ปลอดภัย** แต่ Building ที่ใหญ่กว่า 50 (เช่น Building 1 เดิมที่นับรวม
Access-Layer ทุกตัว หรือ Building 8 ที่วางแผนขยายในอนาคต) **ควรแบ่งเป็น 2 Area ย่อยภายในตัว
Building เดียวกัน** (เช่น Area 1 กับ Area 11 สำหรับ Building 1, ABR คือ CORE-SW1/CORE-SW2
ภายในอาคาร) — เป็นการ "ซ้อน" Hierarchy อีกชั้นในระดับ Building เดียวกัน

### 722.3 ABR Placement Strategy — เงื่อนไขที่ Part 26 บอกว่า "ยังไม่ครบ" ตอนนี้ครบแล้ว

Part 26 Step 252.3 ระบุไว้ชัดเจนว่า **Max-Metric Router-LSA (Stub Router) จะไม่มีประโยชน์เต็ม
รูปแบบ** จนกว่าจะมี **ABR สำรอง (Redundant ABR)** — Design นี้แก้เงื่อนไขนั้นได้แล้ว:

```
กฎ ABR Placement ของ Design นี้:
1. ทุก Building ต้องมี ABR (Building-Edge Router) อย่างน้อย 2 ตัว
2. ทั้ง 2 ตัวต้องเชื่อมเข้า Area 0 (WAN Core) โดยตรง ไม่ผ่านกันเอง
3. ทั้ง 2 ตัวต้องเชื่อมเข้า Area ของ Building ตัวเอง (Area 1-8) โดยตรงเช่นกัน
   → ผลคือ Area ของ Building มี "ทางออก" สู่ Area 0 อย่างน้อย 2 เส้นทางอิสระเสมอ
```

เมื่อต้อง Maintenance BLD3-EDGE1 (ตัวหนึ่งใน ABR คู่ของ Building 3) สามารถใช้ Runbook จาก
Part 26 Step 252.3 ได้เต็มรูปแบบจริงแล้ว:

```
BLD3-EDGE1(config)# router ospf 1
BLD3-EDGE1(config-router)# max-metric router-lsa on-shutdown 300
BLD3-EDGE1(config-router)# end
! Traffic ทั้งหมดของ Area 3 ที่เคยผ่าน BLD3-EDGE1 จะไหลไป BLD3-EDGE2 แทนภายในไม่กี่วินาที
! (มี ABR สำรองจริง — ไม่เหมือนข้อจำกัดที่ Part 26 ระบุไว้สำหรับ Topology เดิม)
```

### 722.4 ข้อจำกัดที่ยังคงอยู่แม้จะแบ่ง Area ครบแล้ว — ทำไมต้องอ่าน Step 723 ต่อ

แม้จะออกแบบ Area อย่างถูกต้องตามหลักการทั้งหมดข้างต้น **ยังมีข้อจำกัดเชิงโครงสร้างของ OSPF ที่
Area ไม่สามารถแก้ได้**:

1. **Type 5 (External) LSA Flood ทั่วทั้ง Domain โดยไม่สนใจ Area Boundary** — ถ้า Building 9-10
   (EIGRP) ถูก Redistribute เข้า OSPF ที่จุดใดจุดหนึ่ง (สมมติที่ WAN Core) External LSA ของ
   Prefix เหล่านั้นจะ Flood ไปถึง **ทุก Area ทั้ง 8 Area ที่เหลือ** ทันที (ไม่ต่างจาก Part 12
   ที่เคยอธิบายไว้ว่า Type 5 ไม่ใช่ Type 3) — LSDB ของ Access Switch ใน Building 2 ต้องเก็บ
   External Route ของ Building 9-10 ด้วย ทั้งที่ไม่เกี่ยวข้องกันเลย
2. **Area 0 (WAN Core) เป็นจุดรวมความเสี่ยงเดียว (Single Point of Fate)** — Link Flap ที่ WAN
   Core (Area 0) ทำให้ ABR ของ**ทุก Building**ต้องคำนวณ Type 3 Summary LSA ใหม่พร้อมกัน แม้
   Building ที่ไม่เกี่ยวข้องเลยก็ต้องรับรู้ Event นี้ — Blast Radius ขยายตามจำนวน Building ที่
   เพิ่มขึ้น ไม่ใช่คงที่
3. **1 OSPF Process = 1 Failure Domain ระดับ Control-plane** — Bug หรือ Config ผิดพลาดจุดเดียว
   ใน Process นี้ (เช่น `auto-cost reference-bandwidth` ไม่ตรงกันแบบ Part 26 Step 257.4 เตือนไว้)
   กระทบทั้ง 8 Building ที่ใช้ OSPF พร้อมกันหมด

> **นี่คือเหตุผลที่ CCIE-level Design ไม่หยุดอยู่ที่ "แบ่ง Area ให้ดี"** — การแบ่ง Area ช่วยลด
> ปัญหาได้ระดับหนึ่ง (Type 1/2 LSA ไม่ Flood ข้าม Area) แต่ **ไม่ได้แยก Fault Domain ออกจากกัน
> อย่างสมบูรณ์** ต่างจากการมี **OSPF Process แยกกันจริง 1 Process ต่อ 1 Building** (ซึ่งเท่ากับ
> เปลี่ยนคำถามจาก "จะแบ่ง Area ในโดเมนเดียวอย่างไร" เป็น "จะเชื่อมหลาย Domain ที่แยกกันสนิท
> เข้าด้วยกันอย่างไร") — คำตอบของคำถามหลังคือหัวข้อของ Step 723

---

## Step 723 — BGP ในบทบาท "Supernet Glue" ของ Enterprise: ทางเลือกที่ดีกว่า Mutual Redistribution

### 723.1 ทวนปัญหาของ Mutual Redistribution จาก Part 31 ที่สเกลนี้จะยิ่งรุนแรงขึ้น

Part 31 สอน Mutual Redistribution ระหว่าง **2 Domain** (OSPF ↔ EIGRP ที่ DIST-SW3/DIST-SW4) และ
เตือนไว้ชัดเจนแล้วว่าอันตรายที่สุดคือ **Routing Loop จาก Redistribution สองทิศทางที่มากกว่า 1
จุด** (Step 307) ซึ่งแก้ด้วย **Route Tag** ที่ต้องออกแบบ วางแผน และ Maintain ด้วยมือทุกจุด —
ลองจินตนาการขยายวิธีนี้ไปที่ **10 Domain**:

```
ถ้าทุก Domain Redistribute เข้า-ออกกันแบบ Mutual โดยตรง (Full-Mesh Redistribution):
จำนวนจุด Redistribution สูงสุดที่เป็นไปได้ = n(n-1)/2 = 10×9/2 = 45 จุด

ทุกจุดต้องมี:
- Seed Metric ที่ถูกต้อง (Part 31 Step 302)
- Route Tag ที่ไม่ชนกับ Domain อื่น (Part 31 Step 307) — ต้องบริหาร Tag Number 45 คู่ไม่ให้ชนกัน
- Route-map/Prefix-list กรอง Selective Redistribution ทุกจุด (Part 31 Step 306)
- ทีมงานที่ต้องรู้ Topology ทั้ง 10 Domain พร้อมกันเพื่อ Debug เมื่อ Loop เกิดขึ้น
```

แม้จะลดจุด Redistribution ลงด้วยการรวมทุก Domain เข้าหา "จุดกลาง" จุดเดียว (เช่น WAN Core)
ก็ยังเหลือปัญหาเดิม: **WAN Core ต้อง Redistribute 10 Domain เข้า-ออกพร้อมกันทั้งหมด** ซึ่งเป็น
ASBR ที่มี Config ซับซ้อนมาก มี Failure Mode มากมาย และ**ทุก Metric ของทุก Domain จะถูกทำให้
เท่ากันหมดด้วย Seed Metric เดียว** (สูญเสียข้อมูล Path-quality ทั้งหมดตามที่ Part 31 Step 301
เตือนไว้เรื่อง Suboptimal Path)

### 723.2 ทำไม BGP แก้ปัญหานี้ได้จากรากฐาน — ไม่ใช่แค่ "อีกวิธีหนึ่ง"

BGP ถูกออกแบบมาตั้งแต่ต้นให้เป็น **Protocol สำหรับเชื่อม Autonomous System ที่เป็นอิสระจากกัน**
(นิยามเดียวกับที่เชื่อม Enterprise กับ ISP ใน Part 29) — คุณสมบัตินี้ตรงกับปัญหาการเชื่อม 10
Building Domain พอดี โดยไม่ต้องดัดแปลงอะไรเพิ่มเลย:

| ปัญหาของ Mutual Redistribution | วิธีที่ BGP แก้ปัญหานี้โดยธรรมชาติ |
|---|---|
| Routing Loop ต้องพึ่ง Route Tag ที่ Config เอง (Part 31 Step 307) | **AS_PATH กันลูปในตัว Protocol เอง** — ถ้า Route ที่ Advertise ออกจาก AS หนึ่งวนกลับมาเจอ AS Number ของตัวเองใน AS_PATH จะถูก Reject ทันทีโดยไม่ต้องออกแบบ Tag เลย |
| Metric ถูกทำให้เท่ากันหมดด้วย Seed Metric | BGP มี Attribute หลายชั้น (WEIGHT/LOCAL_PREF/AS_PATH/MED จาก Part 30 Step 295) ให้ควบคุม Policy ได้ละเอียดกว่า Metric เดียว |
| จุด Redistribution ต้องรู้ Topology ภายในของอีก Domain | BGP เห็นแค่ **Prefix + Attribute** ที่ถูก Advertise เข้ามา — ไม่ต้องรู้เลยว่า Domain นั้นข้างในเป็น OSPF Area เดียวหรือ EIGRP AS ไหน |
| Config ซับซ้อนขึ้นแบบ n(n-1)/2 เมื่อเพิ่ม Domain | เพิ่ม Domain ใหม่ = เพิ่ม BGP Neighbor 1 คู่ที่จุดเดียว (WAN Core) — **Linear ไม่ใช่ Quadratic** เหมือน Part 30 Step 293.1 ที่อธิบายปัญหา Full-Mesh ไว้แล้ว (แก้ด้วย RR เช่นเดียวกัน) |
| Suboptimal Path เพราะ Metric ข้าม Protocol เทียบกันไม่ได้ | BGP **ไม่พยายามเทียบ Metric ข้าม Domain เลย** — ปล่อยให้แต่ละ Domain เก่งเรื่อง Metric ภายในตัวเอง (OSPF Cost/EIGRP Composite) แล้วให้ BGP ตัดสินแค่ "จะออกทาง Building-Edge ตัวไหน" ด้วย Attribute ของมันเอง |

> **นี่คือ CCIE-level Insight ที่สำคัญที่สุดของ Part นี้**: Part 31 สอน Redistribution เป็น
> **เทคนิคสเกลเล็ก** ที่เหมาะกับ 1-2 จุดเชื่อมต่อ (ASBR เดียวหรือสองสามตัว) ส่วน BGP-as-Glue คือ
> **สถาปัตยกรรมสเกลใหญ่** ที่ใช้แทนที่ Mutual Redistribution ทั้งหมดเมื่อจำนวน Domain มากกว่า
> 3-4 Domain ขึ้นไป — **ไม่ใช่ว่า Redistribution "ผิด"** (Part 31 ยังจำเป็นอยู่ที่**จุดต่อ**
> ระหว่าง BGP กับ IGP ของแต่ละ Building — ดู Step 724) แต่ **ขอบเขตที่ใช้ Redistribution ต้อง
> เล็กลงมาก** เหลือแค่ "1 จุดต่อ 1 Building" (BGP↔IGP ทิศทางเดียวที่ควบคุมได้ ไม่ใช่ Mutual
> เต็มรูปแบบข้าม 10 Domain)

### 723.3 สองตัวเลือกของการทำ BGP-Glue: iBGP+RR vs eBGP ด้วย Private AS ต่อ Building (Confederation-style)

| หัวข้อ | ตัวเลือก A: iBGP + Route Reflector (AS เดียว) | ตัวเลือก B: eBGP ด้วย Private AS ต่อ Building |
|---|---|---|
| AS Number | ทุก Building ใช้ AS 65001 เดียวกัน (AS เดิมของ Enterprise จาก Part 29-30) | แต่ละ Building มี Private AS ของตัวเอง (เช่น 65009 สำหรับ Building 9) |
| กลไกกันลูป | ORIGINATOR_ID/CLUSTER_LIST (Part 30 Step 293.5) | AS_PATH ปกติ (แข็งแรงกว่า เพราะเป็นกลไกหลักของ BGP) |
| ความยาก Config | ต่ำ — ใช้ `route-reflector-client` ที่ WAN Core เท่านั้น | สูงกว่า — ต้องจอง/บริหาร AS Number เพิ่มทุก Building |
| Policy ต่อ Building | ทำผ่าน Community/Route-map (ยังทำได้แต่ต้องออกแบบเพิ่ม) | ทำผ่าน AS_PATH Filter ได้ตรงไปตรงมากว่า (`ip as-path access-list`) |
| เหมาะกับ Building ที่ | สร้างใหม่ทั้งหมด, บริหารจัดการจากทีมกลางทีมเดียว (Building 1-8) | มาจาก M&A, อยากคง "เขตปกครองตนเอง" ไว้ระดับหนึ่ง (Building 9-10) |
| ใช้ใน Worked Design ของ Step 724 | ✅ Building 1-8 | ✅ Building 9-10 |

> **สรุปสำหรับ Design นี้**: ไม่ต้องเลือกแบบใดแบบหนึ่งสำหรับทั้งองค์กร — **ใช้ทั้งสองแบบผสมกัน
> ตามบริบทของแต่ละ Building ได้** เพราะ WAN Core Router ตัวเดียวกันสามารถเป็นทั้ง Route
> Reflector (สำหรับ iBGP Client) และเป็น eBGP Peer (สำหรับ Building ที่ใช้ Private AS) พร้อมกัน
> ได้ในเวลาเดียว — นี่คือความยืดหยุ่นที่ Confederation แนวคิดเดิม (Part 30 Step 294) ให้ไว้
> เพียงแต่ในระดับ Enterprise ไม่จำเป็นต้องทำ Confederation เต็มรูปแบบ (ซ่อน Sub-AS จากภายนอก)
> เพราะ AS เหล่านี้ไม่เคยถูกเห็นจากภายนอกองค์กรอยู่แล้ว (Private AS Range 64512-65534)

---

## Step 724 — Full Worked Design: 10-Building Campus พร้อม iBGP/eBGP Building-Edge สู่ WAN-Core RR

### 724.1 Diagram สถาปัตยกรรมเต็มรูปแบบ

```
                              ┌─────────────────────────────────────────┐
                              │        WAN CORE (AS 65001, Backbone)      │
                              │   WAN-CORE-RR1(1.0.0.1) ⇄ WAN-CORE-RR2(1.0.0.2) │
                              │        (iBGP full-mesh กันเอง 2 ตัว)         │
                              └───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬──┘
                    iBGP RR-Client│   │   │   │   │   │   │   │   │   │eBGP (Private AS)
                        ┌──────────┴┐┌┴──┐┌┴──┐┌┴──┐┌┴──┐┌┴──┐┌┴──┐┌┴──┐┌┴───┐┌┴───┐
                        │  Bld 1    ││Bld2││Bld3││Bld4││Bld5││Bld6││Bld7││Bld8││Bld9││Bld10│
                        │ (OSPF     ││OSPF││OSPF││OSPF││OSPF││OSPF││OSPF││OSPF││EIGRP││EIGRP│
                        │  Area0/1/2)││    ││    ││    ││    ││    ││    ││    ││AS200││AS200│
                        │  AS65001  ││65001││65001││65001││65001││65001││65001││65001││AS65009││AS65010│
                        └───────────┘└────┘└────┘└────┘└────┘└────┘└────┘└────┘└─────┘└─────┘
                   แต่ละ Building = 1 IGP Domain ที่แยกสนิทจาก Building อื่น 100%
                   จุดเชื่อมเดียวคือ Building-Edge Router 2 ตัว/อาคาร ↔ WAN-Core-RR1/RR2
```

### 724.2 ตารางการจัดสรร Domain ทั้ง 10 Building (สรุปจาก Step 721-723)

| Building | ชื่อ Site | IGP ภายใน | AS สำหรับต่อ WAN Core | Building-Edge Router | Loopback0 |
|---|---|---|---|---|---|
| 1 | HQ (เดิมจาก Part 1-35) | OSPF 1 (Area 0/1/2 เดิม) | 65001 (iBGP) | WAN-EDGE-1, WAN-EDGE-2 | 1.1.1.21 / 1.1.1.22 |
| 2 | NORTH-TOWER | OSPF 1 (Area 0 เดี่ยว) | 65001 (iBGP) | BLD2-EDGE1, BLD2-EDGE2 | 1.2.0.1 / 1.2.0.2 |
| 3 | SOUTH-TOWER | OSPF 1 (Area 0 เดี่ยว) | 65001 (iBGP) | BLD3-EDGE1, BLD3-EDGE2 | 1.3.0.1 / 1.3.0.2 |
| 4 | WEST-WING | OSPF 1 (Area 0 เดี่ยว) | 65001 (iBGP) | BLD4-EDGE1, BLD4-EDGE2 | 1.4.0.1 / 1.4.0.2 |
| 5 | EAST-WING | OSPF 1 (Area 0 เดี่ยว) | 65001 (iBGP) | BLD5-EDGE1, BLD5-EDGE2 | 1.5.0.1 / 1.5.0.2 |
| 6 | DATA-CENTER-BLD | OSPF 1 (Area 0 เดี่ยว) | 65001 (iBGP) | BLD6-EDGE1, BLD6-EDGE2 | 1.6.0.1 / 1.6.0.2 |
| 7 | LOGISTICS-HUB | OSPF 1 (Area 0 เดี่ยว) | 65001 (iBGP) | BLD7-EDGE1, BLD7-EDGE2 | 1.7.0.1 / 1.7.0.2 |
| 8 | R&D-CAMPUS | OSPF 1 (Area 0/11 — เกิน 50 Router) | 65001 (iBGP) | BLD8-EDGE1, BLD8-EDGE2 | 1.8.0.1 / 1.8.0.2 |
| 9 | RIVER-SIDE (M&A) | **EIGRP AS 200** | **65009 (eBGP)** | BLD9-EDGE1, BLD9-EDGE2 | 1.9.0.1 / 1.9.0.2 |
| 10 | BKK-SOUTH (M&A) | **EIGRP AS 200** | **65010 (eBGP)** | BLD10-EDGE1, BLD10-EDGE2 | 1.10.0.1 / 1.10.0.2 |

> **หมายเหตุ**: Building 9-10 ใช้ EIGRP AS เดียวกัน (200) แต่เป็น**คนละ Domain ที่ไม่ Adjacent
> กันทาง EIGRP เลย** (ไม่มี Physical Link ตรงระหว่างสองอาคาร) — เลข AS ของ EIGRP ไม่จำเป็นต้อง
> Unique ทั่วองค์กรเหมือน BGP AS เพราะ EIGRP AS เป็นแค่ Label ภายใน Process เดียว ไม่ใช่ Global
> Identifier แบบ BGP AS

### 724.3 Worked Config #1 — Building 2 (OSPF ธรรมดา, iBGP RR-Client)

```
! ===== BLD2-EDGE1 — ภายในอาคาร: OSPF Area 0 (Building-internal Backbone) =====
BLD2-EDGE1(config)# router ospf 1
BLD2-EDGE1(config-router)# router-id 1.2.0.1
BLD2-EDGE1(config-router)# network 10.102.0.0 0.0.255.255 area 0
BLD2-EDGE1(config-router)# auto-cost reference-bandwidth 100000
BLD2-EDGE1(config-router)# passive-interface default
BLD2-EDGE1(config-router)# no passive-interface Port-channel1
BLD2-EDGE1(config-router)# exit

! ===== เชื่อมออก WAN Core ด้วย iBGP =====
BLD2-EDGE1(config)# router bgp 65001
BLD2-EDGE1(config-router)# bgp router-id 1.2.0.1
BLD2-EDGE1(config-router)# neighbor 1.0.0.1 remote-as 65001
BLD2-EDGE1(config-router)# neighbor 1.0.0.1 update-source Loopback0
BLD2-EDGE1(config-router)# neighbor 1.0.0.2 remote-as 65001
BLD2-EDGE1(config-router)# neighbor 1.0.0.2 update-source Loopback0
BLD2-EDGE1(config-router)# address-family ipv4 unicast
BLD2-EDGE1(config-router-af)# neighbor 1.0.0.1 activate
BLD2-EDGE1(config-router-af)# neighbor 1.0.0.2 activate
!
! ทิศทาง IGP → BGP: ประกาศออกเฉพาะ Summary ของ Building 2 เท่านั้น (รายละเอียดสูตร Step 725)
BLD2-EDGE1(config-router-af)# network 10.102.0.0 mask 255.255.0.0
!
! ทิศทาง BGP → IGP: รับแค่ Default Route กลับเข้า OSPF (ห้าม Full Table เด็ดขาด — Part 31 Step 308)
BLD2-EDGE1(config-router-af)# exit-address-family
BLD2-EDGE1(config)# router ospf 1
BLD2-EDGE1(config-router)# default-information originate
BLD2-EDGE1(config-router)# end
```

### 724.4 Worked Config #2 — Building 9 (EIGRP Legacy, eBGP ด้วย Private AS)

```
! ===== BLD9-EDGE1 — ภายในอาคาร: EIGRP AS 200 (Legacy M&A เดิม ไม่แตะต้อง) =====
BLD9-EDGE1(config)# router eigrp EIGRP-RIVERSIDE
BLD9-EDGE1(config-router)# address-family ipv4 unicast autonomous-system 200
BLD9-EDGE1(config-router-af)# network 10.109.0.0 0.0.255.255
BLD9-EDGE1(config-router-af)# eigrp router-id 1.9.0.1
BLD9-EDGE1(config-router-af)# topology base
BLD9-EDGE1(config-router-af-topology)# exit-af-topology
BLD9-EDGE1(config-router-af)# exit-address-family

! ===== เชื่อมออก WAN Core ด้วย eBGP (Private AS 65009 — Confederation-style ตาม Step 723.3) =====
BLD9-EDGE1(config)# router bgp 65009
BLD9-EDGE1(config-router)# bgp router-id 1.9.0.1
BLD9-EDGE1(config-router)# neighbor 1.0.0.1 remote-as 65001
BLD9-EDGE1(config-router)# neighbor 1.0.0.1 ebgp-multihop 2
BLD9-EDGE1(config-router)# neighbor 1.0.0.2 remote-as 65001
BLD9-EDGE1(config-router)# neighbor 1.0.0.2 ebgp-multihop 2
BLD9-EDGE1(config-router)# address-family ipv4 unicast
BLD9-EDGE1(config-router-af)# neighbor 1.0.0.1 activate
BLD9-EDGE1(config-router-af)# neighbor 1.0.0.2 activate
!
! ทิศทาง EIGRP → BGP: Redistribute แบบมี Route-map กรองเฉพาะ Summary (ไม่ใช่ Mutual — ทิศทางเดียว)
BLD9-EDGE1(config-router-af)# redistribute eigrp 200 route-map RIVERSIDE-SUMMARY-ONLY
BLD9-EDGE1(config-router-af)# exit-address-family
BLD9-EDGE1(config)# ip prefix-list RIVERSIDE-SUMMARY seq 5 permit 10.109.0.0/16
BLD9-EDGE1(config)# route-map RIVERSIDE-SUMMARY-ONLY permit 10
BLD9-EDGE1(config-route-map)# match ip address prefix-list RIVERSIDE-SUMMARY
BLD9-EDGE1(config-route-map)# exit
!
! ทิศทาง BGP → EIGRP: รับแค่ Default Route (ผ่าน Redistribute BGP + Route-map DEFAULT-ONLY)
BLD9-EDGE1(config)# ip prefix-list DEFAULT-ONLY seq 5 permit 0.0.0.0/0
BLD9-EDGE1(config)# route-map DEFAULT-ONLY permit 10
BLD9-EDGE1(config-route-map)# match ip address prefix-list DEFAULT-ONLY
BLD9-EDGE1(config-route-map)# exit
BLD9-EDGE1(config)# router eigrp EIGRP-RIVERSIDE
BLD9-EDGE1(config-router)# address-family ipv4 unicast autonomous-system 200
BLD9-EDGE1(config-router-af)# topology base
BLD9-EDGE1(config-router-af-topology)# redistribute bgp 65009 route-map DEFAULT-ONLY metric 10000 100 255 1 1500
BLD9-EDGE1(config-router-af-topology)# end
```

### 724.5 Worked Config #3 — WAN-CORE-RR1 (จุดรวมทั้ง 10 Building)

```
WAN-CORE-RR1(config)# router bgp 65001
WAN-CORE-RR1(config-router)# bgp router-id 1.0.0.1
WAN-CORE-RR1(config-router)# bgp cluster-id 1
!
! กลุ่ม iBGP RR-Client (Building 1-8, All-Cisco/AS เดียว)
WAN-CORE-RR1(config-router)# neighbor IBGP-BUILDINGS peer-group
WAN-CORE-RR1(config-router)# neighbor IBGP-BUILDINGS remote-as 65001
WAN-CORE-RR1(config-router)# neighbor IBGP-BUILDINGS update-source Loopback0
WAN-CORE-RR1(config-router)# neighbor 1.1.0.21 peer-group IBGP-BUILDINGS   ! WAN-EDGE-1 (Bld1)
WAN-CORE-RR1(config-router)# neighbor 1.2.0.1  peer-group IBGP-BUILDINGS   ! BLD2-EDGE1
WAN-CORE-RR1(config-router)# neighbor 1.2.0.2  peer-group IBGP-BUILDINGS   ! BLD2-EDGE2
WAN-CORE-RR1(config-router)# ... (ทำซ้ำ Pattern เดียวกันจนครบ Building 3-8, รวม 16 Neighbor)
WAN-CORE-RR1(config-router)# neighbor 1.0.0.2 remote-as 65001              ! RR คู่กัน
WAN-CORE-RR1(config-router)# neighbor 1.0.0.2 update-source Loopback0
!
! กลุ่ม eBGP Private-AS (Building 9-10, M&A)
WAN-CORE-RR1(config-router)# neighbor 1.9.0.1  remote-as 65009
WAN-CORE-RR1(config-router)# neighbor 1.9.0.1  ebgp-multihop 2
WAN-CORE-RR1(config-router)# neighbor 1.10.0.1 remote-as 65010
WAN-CORE-RR1(config-router)# neighbor 1.10.0.1 ebgp-multihop 2
!
WAN-CORE-RR1(config-router)# address-family ipv4 unicast
WAN-CORE-RR1(config-router-af)# neighbor IBGP-BUILDINGS activate
WAN-CORE-RR1(config-router-af)# neighbor IBGP-BUILDINGS route-reflector-client
WAN-CORE-RR1(config-router-af)# neighbor IBGP-BUILDINGS next-hop-self
WAN-CORE-RR1(config-router-af)# neighbor 1.0.0.2 activate
WAN-CORE-RR1(config-router-af)# neighbor 1.9.0.1 activate
WAN-CORE-RR1(config-router-af)# neighbor 1.9.0.1 maximum-prefix 20 80       ! Defense — Part 31 Step 308 lesson
WAN-CORE-RR1(config-router-af)# neighbor 1.10.0.1 activate
WAN-CORE-RR1(config-router-af)# neighbor 1.10.0.1 maximum-prefix 20 80
WAN-CORE-RR1(config-router-af)# end
```

**Verify ที่ WAN-CORE-RR1 — ต้องเห็น Prefix ของทั้ง 10 Building แต่**เป็น Summary เท่านั้น**:**

```
WAN-CORE-RR1# show ip bgp summary
BGP router identifier 1.0.0.1, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
1.1.0.21        4 65001    ...     ...        12    0    0  01:14:22        1
1.2.0.1         4 65001    ...     ...        12    0    0  01:10:05        1
1.2.0.2         4 65001    ...     ...        12    0    0  01:10:05        1
...
1.9.0.1         4 65009    ...     ...        12    0    0  00:58:41        1
1.10.0.1        4 65010    ...     ...        12    0    0  00:55:19        1

WAN-CORE-RR1# show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.10.0.0/16      1.1.0.21                 0    100      0 i          <- Building 1
*> 10.102.0.0/16     1.2.0.1                  0    100      0 i          <- Building 2
*> 10.103.0.0/16     1.3.0.1                  0    100      0 i
...
*> 10.109.0.0/16     1.9.0.1                  0    100      0 65009 i    <- Building 9 (eBGP, เห็น AS_PATH)
*> 10.110.0.0/16     1.10.0.1                 0    100      0 65010 i    <- Building 10
```

**เพียง 10 Prefix ใน BGP Table ของทั้ง Enterprise** (แทนที่จะเป็นหลักร้อย Subnet ถ้าไม่
Summarize — รายละเอียดสูตรคำนวณอยู่ใน Step 725)

---

## Step 725 — Route Summarization ทุกขอบเขต: เข้า/ออก BGP และ IGP

### 725.1 ทำไม Summarization ที่สเกลนี้ไม่ใช่ "Nice to Have" แต่คือ "Must Have"

Part 12 Step 127/130 และ Part 28 Step 274 สอน Summarization ในระดับ 1-2 จุดของ Domain เดียว
ที่สเกล 10 Building **Summarization กลายเป็นเงื่อนไขที่ทำให้ Design ทั้งหมด Scale ได้จริงหรือ
ไม่ได้เลย** — ให้ดูตัวเลขเปรียบเทียบ:

| สถานการณ์ | จำนวน Prefix ที่ BGP Table ของ WAN Core ต้องเก็บ | จำนวน External Route ที่แต่ละ Building ต้องรับกลับ |
|---|---|---|
| **ไม่ Summarize เลย** (ทุก VLAN/Subnet ของทุกอาคาร Advertise ตรงเข้า BGP) | ~10 Building × เฉลี่ย 15-20 Subnet/อาคาร = **150-200 Prefix** | แต่ละอาคารต้องรับ Prefix ของ**อีก 9 อาคาร**กลับเข้า IGP ตัวเอง = 135-180 External Route ต่ออาคาร |
| **Summarize เข้า BGP (Building Supernet เดียว) + Default Route ออก** | **10 Prefix** (1 Supernet/อาคาร) | แต่ละอาคารรับแค่ **1 Default Route (0.0.0.0/0)** |

**ผลต่าง ~15-20 เท่า** ในขนาด BGP Table และ **~135-180 เท่า** ในจำนวน External Route ที่ทุก
Access Switch ในทุกอาคารต้องแบกรับใน LSDB/Topology Table ของตัวเอง — นี่คือความแตกต่างระหว่าง
Design ที่ Scale ได้กับ Design ที่จะพังตอนอาคารที่ 11-12 ถูกเพิ่มเข้ามา

### 725.2 ทิศทางที่ 1 — สรุปเข้า BGP (IGP → BGP) ที่ Building-Edge ทุกอาคาร

ใช้ `network` statement (สำหรับ Prefix ที่มีอยู่แน่นอนใน RIB อยู่แล้ว) คู่กับ `aggregate-address`
(สำหรับสร้าง Summary ใหม่จาก Component Route หลายเส้น) — วิธีที่ปลอดภัยที่สุดคือ Aggregate
พร้อม **`summary-only`** เพื่อไม่ให้ Component Route หลุดเข้า BGP โดยไม่ตั้งใจ:

```
BLD2-EDGE1(config)# router bgp 65001
BLD2-EDGE1(config-router)# address-family ipv4 unicast
BLD2-EDGE1(config-router-af)# aggregate-address 10.102.0.0 255.255.0.0 summary-only
BLD2-EDGE1(config-router-af)# end
```

**Verify — เห็นแค่ Summary ไม่เห็น Component:**

```
BLD2-EDGE1# show ip bgp
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.102.0.0/16     0.0.0.0                  0         32768 i        <- Summary เท่านั้น
s> 10.102.10.0/24    10.102.1.1               1                0 ?     <- 's' = Suppressed (ไม่ Advertise ออกไป)
s> 10.102.20.0/24    10.102.1.1               1                0 ?
```

### 725.3 ทิศทางที่ 2 — สรุปออกจาก BGP เข้า IGP (BGP → IGP) — ป้องกัน Full Table Leak

Part 31 Step 308 เตือนไว้แล้วว่า Redistribute BGP เข้า IGP โดยไม่กรองคือ **อันตรายที่สุด** —
ที่สเกลนี้แม้ BGP Table จะมีแค่ 10 Prefix (ไม่ใช่ Internet Full Table) **หลักการเดียวกันยังต้อง
ใช้เสมอ** เพราะ:

1. Building แต่ละแห่งไม่มีความจำเป็นต้องรู้จัก Prefix ของ**อีก 9 อาคาร**แยกทีละเส้นเลย —
   Traffic ข้ามอาคารทั้งหมดต้องผ่าน WAN Core อยู่แล้ว ดังนั้น **Default Route เส้นเดียวก็เพียงพอ**
2. ถ้าวันหนึ่งมี Building ที่ 11-20 เพิ่มเข้ามา (Enterprise ขยายต่อ) จำนวน Prefix ใน BGP Table
   จะโตขึ้นตามจำนวน Building — ถ้า Design เผลอ Redistribute BGP Table เข้า IGP แบบไม่กรองไว้
   ตั้งแต่ต้น ทุก Building ที่มีอยู่เดิมจะได้รับ External Route เพิ่มขึ้นเรื่อยๆ ทุกครั้งที่มี
   Building ใหม่ — Design ที่ดีต้อง **Decouple** ปัญหานี้ตั้งแต่วันแรก

```
BLD2-EDGE1(config)# router ospf 1
BLD2-EDGE1(config-router)# default-information originate
BLD2-EDGE1(config-router)# end
```

`default-information originate` (ไม่ใส่ `always`) จะฉีด Default Route เข้า OSPF **ก็ต่อเมื่อ**
BLD2-EDGE1 มี Default Route ของตัวเองอยู่จริงใน RIB ก่อน (เรียนจาก BGP) — เป็นกลไก "Conditional
Default Injection" ที่ปลอดภัยกว่าการ Force ด้วย `always` เสมอ (ถ้า BGP Session ล่มทั้งคู่
BLD2-EDGE1 จะหยุด Advertise Default Route ให้ Building ตัวเองโดยอัตโนมัติ ไม่ใช่ Black-hole
Traffic ไปยัง Router ที่ไม่มีทางออกจริง)

### 725.4 ตารางสรุปแผน Summarization ทุกขอบเขตของทั้ง Campus

| ขอบเขต | ทิศทาง | สิ่งที่ Advertise | เครื่องมือ |
|---|---|---|---|
| Building 1-8 (OSPF) → BGP | IGP → BGP | Supernet /16 ของอาคารเดียว | `aggregate-address ... summary-only` |
| Building 9-10 (EIGRP) → BGP | IGP → BGP | Supernet /16 ของอาคารเดียว | `redistribute eigrp 200 route-map <SUMMARY-ONLY>` + `ip summary-address eigrp` ที่ต้นทาง |
| BGP → Building 1-8 (OSPF) | BGP → IGP | Default Route เดียว | `default-information originate` (Conditional) |
| BGP → Building 9-10 (EIGRP) | BGP → IGP | Default Route เดียว | `redistribute bgp <AS> route-map DEFAULT-ONLY` |
| WAN-CORE-RR1 ↔ RR2 | iBGP ระหว่าง RR | ทุก Prefix ที่ Reflect (10 Supernet) | ไม่ต้อง Summarize เพิ่ม (จำนวนน้อยอยู่แล้ว) |

---

## Step 726 — Redundancy & Convergence Design: วิเคราะห์ตัวเลขจริง IGP vs BGP

### 726.1 ทำไม BGP Convergence ช้ากว่า IGP โดยธรรมชาติ

| ปัจจัย | IGP (OSPF/EIGRP) | BGP |
|---|---|---|
| กลไก Detect Failure | Hello/Dead Timer หรือ BFD (Part 26/28) — ผูกกับ Interface โดยตรง | ต้อง Bind BFD เข้ากับ BGP Session เอง (`neighbor <ip> fall-over bfd`) ไม่ Detect เองถ้าไม่ทำ |
| กลไกคำนวณ Path ใหม่ | SPF (OSPF) / DUAL+FS (EIGRP) — คำนวณจาก Topology ทั้งหมดที่มีอยู่แล้วใน LSDB/Topology Table | ต้องรอ Update/Withdraw จาก Neighbor, แล้วรัน Best Path Algorithm ใหม่ (Part 30 Step 295) ต่อทุก Prefix ที่กระทบ |
| Batching ของการส่ง Update | Flood ทันทีเมื่อ SPF Throttle อนุญาต (Part 26 Step 254, ต่ำสุด 10ms) | มี **MRAI (Minimum Route Advertisement Interval)** Default = 30 วินาทีสำหรับ eBGP (0 สำหรับ iBGP แต่ยังมี Batching ภายใน Process) |
| จำนวน Hop ที่ต้อง Recompute | เฉพาะ Router ใน Area/AS ที่ Event เกิด (มี Boundary จาก Area/Domain) | ต้องส่งผ่าน RR แล้ว Reflect ต่อไปยัง Client อื่นทุกตัว (เพิ่ม Hop ของ Control-plane Message) |

### 726.2 ตัวเลข Convergence จริงตามชั้นของ Design นี้

| สถานการณ์ Failure | Layer ที่รับผิดชอบ | เวลา Convergence โดยประมาณ |
|---|---|---|
| Access-Link/Distribution-Link ภายใน 1 Building ล่ม | OSPF/EIGRP + BFD (450ms) + SPF/DUAL Throttle (10ms start) | **< 1 วินาที** — และ **BGP ไม่รู้เรื่องนี้เลย** (เพราะ Step 725 Summarize ไว้แล้ว Supernet ของอาคารไม่เปลี่ยน) |
| Building-Edge ↔ WAN-Core Link เดียวล่ม (มี Multi-homing ตาม Step 727) | BGP + BFD ผูกกับ Session + Backup Path ที่มีอยู่แล้วใน BGP Table (Multipath) | **~1-2 วินาที** (BFD Detect 450ms + BGP ถอด Path ที่ตายออกจาก RIB ทันทีเพราะ Path สำรองคำนวณไว้ล่วงหน้าแล้ว) |
| Building-Edge Router ทั้งตัวล่ม (ไม่มี Multi-homing — Single-homed สมมติ) | BGP เต็มรูปแบบ ต้องรอ Hold-timer (Default 180s) ถ้าไม่มี BFD, หรือรอ RR Reflect ใหม่ทั้ง Cluster ถ้ามี BFD | **5-30+ วินาที** ขึ้นกับว่าผูก BFD หรือไม่ — นี่คือเหตุผลที่ **Step 727 (Multi-homing) ไม่ใช่ Option แต่คือ Requirement** |
| WAN-Core-RR1 ทั้งตัวล่ม (มี RR2 สำรอง) | iBGP Failover ไปยัง RR2 ที่มี Session อยู่แล้วทุก Building | **~2-5 วินาที** (ทุก Building ต้อง Re-establish Best Path ผ่าน RR2 แทน) |

### 726.3 หลักการออกแบบที่ได้จากตัวเลขข้างบน — คำตอบตรงประเด็นของ CCIE

> **BGP ควรถูกใช้สำหรับ Route ที่ "นิ่ง" และถูก Summarize ไว้แล้วเท่านั้น — ไม่ใช่ Layer ที่คาด
> หวัง Sub-second Convergence แบบ IGP** การออกแบบที่ถูกต้องคือ **แบ่งงานตามความถี่ของการ
> เปลี่ยนแปลง**:
>
> - **Failure ที่เกิดถี่ (Access/Distribution Link Flap)** → ต้องจบภายใน Building โดย IGP
>   ล้วนๆ ไม่ให้กระทบ BGP เลย (สำเร็จได้เพราะ Step 725 Summarize ไว้ก่อนแล้ว)
> - **Failure ที่เกิดไม่บ่อย แต่กระทบมาก (Building-Edge/WAN-Core Link)** → ยอมรับได้ที่ BGP
>   ใช้เวลา 1-5 วินาที **ตราบใดที่มี Redundancy** (Step 727) ให้ BGP มี Path สำรองพร้อมอยู่แล้ว
>   ไม่ต้องรอ Full Reconverge จากศูนย์
> - **ผูก BFD ทุกชั้นเสมอ** — ทั้งที่ IGP (Part 26/28 สอนไปแล้ว) และที่ BGP Session เอง
>   (`neighbor <ip> fall-over bfd`) เพราะ Default Hold-timer ของ BGP (180s) ช้าเกินกว่าจะรับได้
>   ที่สเกล Enterprise

```
! ตัวอย่างผูก BFD เข้ากับ BGP Session ที่ BLD2-EDGE1 (ต้องเปิด bfd interval ที่ Interface ก่อน
! ตามที่ Part 26 Step 254.2 สอนไว้)
BLD2-EDGE1(config)# router bgp 65001
BLD2-EDGE1(config-router)# neighbor 1.0.0.1 fall-over bfd
BLD2-EDGE1(config-router)# neighbor 1.0.0.2 fall-over bfd
BLD2-EDGE1(config-router)# end
```

---

## Step 727 — Multi-homing Design สำหรับ Building-to-WAN-Core: ต่อยอด Dual-homed Pattern จาก Part 33

### 727.1 ทวน Pattern เดิมจาก Part 33 แล้วขยายสเกลขึ้นหนึ่งระดับ

Part 33 Step 324 แก้ปัญหา **Single-homed DIST↔CORE** เดิม (Part 10) ด้วยการต่อ DIST-SW แต่ละตัว
เข้า **ทั้ง CORE-SW1 และ CORE-SW2** พร้อมใช้ OSPF ECMP แทน STP Block — Design ของ Part นี้ใช้
**หลักการเดียวกันทุกประการ** แต่ยกระดับจาก "DIST↔CORE ภายในอาคารเดียว" เป็น
**"Building-Edge↔WAN-Core ระดับทั้ง Campus"**:

```
Pattern Part 33 (ภายใน 1 Building):        Pattern Part 73 (ระดับ Building ↔ WAN Core):

DIST-SW1 ──Active──► CORE-SW1              BLD2-EDGE1 ──iBGP──► WAN-CORE-RR1
         ──Active──► CORE-SW2                         ──iBGP──► WAN-CORE-RR2
DIST-SW2 ──Active──► CORE-SW1              BLD2-EDGE2 ──iBGP──► WAN-CORE-RR1
         ──Active──► CORE-SW2                         ──iBGP──► WAN-CORE-RR2

(OSPF ECMP เลือก 2 เส้นพร้อมกัน)            (BGP Multipath เลือก 2 เส้นพร้อมกัน — ต้อง Config เพิ่ม)
```

**ทุก Building มี 4 Session/Link ทางกายภาพที่เป็นไปได้** (2 Building-Edge × 2 WAN-Core-RR) —
เหมือน Full-Mesh ย่อยระหว่าง Building กับ WAN Core

### 727.2 ความต่างสำคัญ — BGP ไม่ทำ ECMP อัตโนมัติเหมือน OSPF

Part 33 Step 327 อธิบายว่า OSPF ทำ ECMP อัตโนมัติถ้า Cost เท่ากัน — **BGP ไม่ทำแบบนั้นเป็น
Default** เพราะ Best Path Algorithm (Part 30 Step 295) เลือกผู้ชนะเพียง 1 Path เสมอ ต้องเปิด
`maximum-paths` เพิ่มเพื่อให้ยอมรับหลาย Path พร้อมกัน:

```
BLD2-EDGE1(config)# router bgp 65001
BLD2-EDGE1(config-router)# address-family ipv4 unicast
BLD2-EDGE1(config-router-af)# maximum-paths ibgp 2
BLD2-EDGE1(config-router-af)# end
```

สำหรับ Building 9-10 ที่ใช้ eBGP กับ Private AS (Step 723.3) ต้องใช้ `maximum-paths eibgp`
แทน และถ้า AS_PATH ที่เห็นจาก RR1/RR2 ไม่เหมือนกันเป๊ะ (ซึ่งไม่ควรเกิดเพราะทั้งคู่อยู่ AS 65001
เดียวกัน) ต้องเพิ่ม `bgp bestpath as-path multipath-relax` ด้วย:

```
BLD9-EDGE1(config)# router bgp 65009
BLD9-EDGE1(config-router)# bgp bestpath as-path multipath-relax
BLD9-EDGE1(config-router)# address-family ipv4 unicast
BLD9-EDGE1(config-router-af)# maximum-paths eibgp 2
BLD9-EDGE1(config-router-af)# end
```

### 727.3 Verify — เห็น 2 Path พร้อมกันจริงใน RIB

```
BLD2-EDGE1# show ip route bgp | include 0.0.0.0
B*    0.0.0.0/0 [200/0] via 1.0.0.1, 00:12:04
                [200/0] via 1.0.0.2, 00:12:04     <- Multipath ติดตั้งพร้อมกัน 2 เส้นจริง
```

เมื่อ Link ไปยัง WAN-CORE-RR1 ล่ม Path ที่เหลือ (ผ่าน RR2) **มีอยู่แล้วใน RIB ตั้งแต่ก่อน
Failure** — ไม่ต้องรอ BGP คำนวณ Best Path ใหม่จากศูนย์ ตรงกับตัวเลข Convergence ~1-2 วินาที
ที่ Step 726.2 ระบุไว้สำหรับสถานการณ์ "มี Multi-homing"

---

## Step 728 — จัดการ Asymmetric Bandwidth: Cost/Metric Tuning ข้าม Link ความเร็วต่างกันมาก

### 728.1 ประโยชน์ที่ไม่คาดคิดของการแยก Domain — Reference-Bandwidth ไม่ต้อง Sync ทั้งองค์กรอีกต่อไป

Part 26 Step 257.4 เตือนไว้ว่า `auto-cost reference-bandwidth` **ต้องตรงกันทุก Router ใน OSPF
Domain เดียวกัน** ไม่งั้นเกิด Asymmetric Routing โดยไม่มี Error แจ้งเตือน — ที่สเกลเดิม (1
Domain คลุมทั้ง Campus) นี่หมายความว่า **ทุกอาคารต้อง Sync ค่านี้ให้ตรงกันหมดตลอดไป** แต่เพราะ
Design ของ Part นี้แยก **OSPF Process ต่อ Building อย่างสมบูรณ์** (Step 723-724) ผลพลอยได้ที่
สำคัญคือ:

> **Reference-Bandwidth ต้อง Sync กันแค่ "ภายใน 1 Building" เท่านั้น** — Building 3 จะใช้
> `auto-cost reference-bandwidth 100000` (100G) ในขณะที่ Building 7 ที่ยังไม่ Upgrade Backbone
> ใช้ค่า Default (100M) ก็ไม่กระทบกันเลย เพราะเป็นคนละ OSPF Process/Cost-space โดยสิ้นเชิง —
> เป็นตัวอย่างจริงของ "การแยก Fault Domain" ที่ให้ประโยชน์มากกว่าที่ Step 723 พูดไว้ตอนแรก

หลักการเดียวกันใช้กับ EIGRP K-value/Bandwidth-Delay ของ Building 9-10 — ต้องตรงกันแค่ภายใน
EIGRP AS 200 ของอาคารนั้นๆ เท่านั้น ไม่กระทบ Building อื่น

### 728.2 ตารางความเร็ว Link ภายใน 1 Building และ Cost ที่คำนวณได้ (Reference-BW 100000)

| ชั้น | ความเร็ว | Cost (OSPF, Ref-BW=100000) |
|---|---|---|
| Access ↔ Distribution | 1G | 100 |
| Distribution ↔ Building-Core | 10G | 10 |
| Building-Core ↔ Building-Edge | 40G (LACP Bundle) | 2 |
| Building-Edge ↔ WAN-Core | 100G | 1 |

(สูตรและตารางเต็มเหมือนที่ Part 26 Step 257.2 คำนวณไว้แล้ว — ใช้ค่าเดียวกันได้ทันทีเพราะยังอยู่
ในเพดาน Cost 65535 อย่างปลอดภัย)

### 728.3 ปัญหาที่ยังคงอยู่ — BGP ไม่มี "Cost" หรือ "Bandwidth Metric" ให้เลย

สมมติ Building 4 มี 2 เส้นทางไปหา WAN Core: เส้นหลักเป็น Fiber 100G และเส้นสำรองเป็น Microwave
Link 1G (สำหรับ Disaster Recovery เท่านั้น) — **BGP ไม่มีแนวคิด "Cost ตาม Bandwidth" แบบ OSPF
เลย** (Part 30 Step 295 อธิบายไปแล้วว่า Best Path เลือกจาก Attribute ไม่ใช่ Bandwidth) ถ้าไม่
Config อะไรเพิ่ม และทั้ง 2 เส้นมี Attribute เท่ากันทุกอย่าง (WEIGHT/LOCAL_PREF/AS_PATH เท่ากัน)
ผลลัพธ์ที่เป็นไปได้มีสองแบบที่แย่ทั้งคู่:

1. ถ้าเปิด `maximum-paths` ไว้ (ตาม Step 727) — BGP จะ **ECMP แบ่ง Traffic ครึ่งหนึ่งไปทาง
   Microwave 1G** ทำให้ Link นั้นล้นทันที (Suboptimal Routing ที่ Part 72/73 ต้องป้องกัน)
2. ถ้าไม่เปิด ECMP — BGP อาจเลือกผ่าน Tie-break ขั้นสุดท้าย (เช่น Router-ID/Neighbor IP ต่ำสุด)
   ซึ่ง**ไม่มีความเกี่ยวข้องกับ Bandwidth จริงเลยแม้แต่นิดเดียว**

### 728.4 แก้ด้วย LOCAL_PREF — บังคับ Policy ให้ตรงกับ Bandwidth จริงด้วยมือ

```
! BLD4-EDGE1 — เส้นหลัก Fiber 100G ไปยัง WAN-CORE-RR1
BLD4-EDGE1(config)# route-map PRIMARY-100G permit 10
BLD4-EDGE1(config-route-map)# set local-preference 200
BLD4-EDGE1(config-route-map)# exit

! เส้นสำรอง Microwave 1G ไปยัง WAN-CORE-RR2 (ผ่าน Link คุณภาพต่ำกว่ามาก)
BLD4-EDGE1(config)# route-map BACKUP-1G permit 10
BLD4-EDGE1(config-route-map)# set local-preference 100
BLD4-EDGE1(config-route-map)# exit
!
BLD4-EDGE1(config)# router bgp 65001
BLD4-EDGE1(config-router)# address-family ipv4 unicast
BLD4-EDGE1(config-router-af)# neighbor 1.0.0.1 route-map PRIMARY-100G in
BLD4-EDGE1(config-router-af)# neighbor 1.0.0.2 route-map BACKUP-1G in
BLD4-EDGE1(config-router-af)# end
```

**ผลลัพธ์**: BGP เลือก Path ผ่าน WAN-CORE-RR1 (LOCAL_PREF 200) เสมอตราบใดที่ยังใช้งานได้ —
ไม่มี ECMP แบ่ง Traffic ไปทาง Microwave 1G อีกต่อไป และ Path สำรองจะถูกใช้**เมื่อ Path หลัก
ตายเท่านั้น** ตรงตามเจตนาการออกแบบ ตรงข้ามกับที่จะเกิดถ้าปล่อยให้ Attribute เท่ากันแล้วให้ BGP
Tie-break เอง

> **หลักการทองคำของ Step นี้**: ทุกครั้งที่ Building มี Uplink ไปหา WAN Core มากกว่า 1 เส้นและ
> **ความเร็วไม่เท่ากัน** ต้องตั้ง LOCAL_PREF (หรือ WEIGHT ถ้าควบคุมแค่ Router ตัวเดียว) ให้ตรงกับ
> ลำดับความสำคัญที่ต้องการเสมอ **ห้ามปล่อยให้ BGP ตัดสินใจเอง** เพราะ BGP ไม่รู้จัก "ความเร็ว"
> ของ Link เลยไม่ว่าในสถานการณ์ใดก็ตาม — Bandwidth-awareness เป็นความรับผิดชอบของ Engineer
> ทั้งหมด ต่างจาก OSPF/EIGRP ที่ทำให้อัตโนมัติผ่าน Cost/Metric

---

## Step 729 — Documentation & Change Management: ทำไม LLD จำเป็นยิ่งกว่าเดิมที่สเกลนี้

### 729.1 ทำไมความจำ/Tribal Knowledge ใช้ไม่ได้อีกต่อไปที่สเกล 10 Building

Lab เดิมของหลักสูตร (1 Building, ~15 Device) ยังพอจำ IP/Area/AS ทั้งหมดในหัวได้ — แต่ที่สเกล
**10 Building × ~40 Device เฉลี่ย = ~400 Device**, **10 AS Number**, **~19 Area/Process
Boundary** ไม่มีวิศวกรคนใดจำทั้งหมดได้แม่นยำ และความผิดพลาดเล็กๆ ที่สเกลนี้ขยายเป็นปัญหาใหญ่
ทันที เช่น:

| ความผิดพลาดเล็กๆ | ผลกระทบที่สเกล 10 Building |
|---|---|
| ตั้ง AS Number ของ Building 9 ผิดเป็น 65010 (ชนกับ Building 10) | eBGP Session ขึ้นได้ปกติในตอนแรก แต่ AS_PATH Loop-detection จะ Reject Route ของ Building 10 ที่ Reflect ผ่าน RR กลับมาโดยเข้าใจผิดว่าเป็น Loop — Debug ยากมากเพราะไม่มี Error ชัดเจน |
| ลืมใส่ `summary-only` ที่ Building ใดอาคารหนึ่ง | BGP Table ทั้ง Enterprise บวมขึ้นทันที และ**ทุก Building อื่น**ที่รับ Default Route กลับเข้า IGP อาจได้รับ Route ปลีกย่อยของอาคารนั้นแทรกเข้ามาด้วยถ้า Route-map ฝั่งรับไม่ได้กรองไว้แน่นหนา |
| Router-ID ของ BLD5-EDGE1 กับ Loopback0 ของอีกอาคารชนกันโดยไม่ตั้งใจ (Copy-paste Config ผิด) | OSPF/BGP Adjacency ผิดเพี้ยน หรือ Route ถูก Overwrite กันเองแบบเงียบๆ |

### 729.2 LLD (Low-Level Design) คือคำตอบ — และ `00-ip-address-plan.md` คือตัวอย่างที่ใช้อยู่แล้ว

สังเกตว่าหลักสูตรนี้ทั้งหมดใช้ **`00-ip-address-plan.md`** เป็น **Single Source of Truth** มา
ตั้งแต่ Part 1 (ตามที่ระบุไว้ในหัวเอกสารเอง) — นี่คือ LLD ในทางปฏิบัติของ Lab ขนาด 1 Building
ที่สเกล 10 Building **หลักการเดียวกันนี้จำเป็นยิ่งกว่าเดิม** เพราะจำนวนตัวแปรที่ต้องจัดสรร
Unique กันทั่วองค์กร (AS Number, Router-ID, Area/Process ID, Cluster-ID) เพิ่มขึ้นแบบเส้นตรงตาม
จำนวน Building — ถ้าไม่มีเอกสารกลาง **ความเสี่ยงชนกันของค่าเหล่านี้เพิ่มขึ้นแบบ Quadratic** ตาม
สูตรความเป็นไปได้ของการชนกันแบบเดียวกับ Full-Mesh Session Count ใน Part 30 Step 291.3

### 729.3 เชื่อมกับบทเรียน Change Management จาก Part 69 Mega-Lab

Part 69 (Mega-Lab) สอนบทเรียนสำคัญว่า **Change ที่กระทบมากกว่า 1 Critical Device ต้องผ่าน
Change Advisory Board (CAB)** พร้อม **Staged/Canary Rollout** และ **Rollback Plan** ที่เตรียม
ไว้ก่อนเริ่มแก้จริงเสมอ (`show archive config differences` เก็บ Checkpoint ก่อน-หลังทุกครั้ง)
— ที่สเกล 10 Building **หลักการนี้ต้องเข้มงวดกว่าเดิมอีกขั้น** เพราะ:

1. **Blast Radius ของ WAN-Core-RR ขยายเป็นทั้ง Enterprise** — Change ผิดพลาดที่ WAN-CORE-RR1/RR2
   (เช่น Cluster-ID พลาด, Prefix-list ผิด) กระทบ**ทุก Building พร้อมกัน** ไม่ใช่แค่ 1 Access
   Switch เหมือน Part 69
2. **Change ที่ดูเหมือนเล็ก (แก้ Summarization Boundary ของ 1 Building) กระทบ Building อื่น
   โดยอ้อม** — ถ้า Building 5 เผลอประกาศ Supernet ที่ Overlap กับ Building 6 (Config ผิด/ลืม
   ปรับ Subnet Plan) Traffic ของทั้งสองอาคารอาจเกิด Suboptimal Routing หรือ Black-hole โดยไม่มี
   ใครสังเกตทันที เพราะ WAN Core มองเห็นแค่ Prefix ไม่เห็น Detail ภายในของแต่ละอาคาร
3. **จำนวน Stakeholder ต่อ Change เพิ่มขึ้น** — Change ที่ WAN Core ต้องแจ้งทีมของทุก Building
   ที่อาจได้รับผลกระทบ ไม่ใช่แค่ทีมของอาคารเดียว

### 729.4 โครงสร้าง LLD ที่แนะนำสำหรับ Design ระดับนี้ (ใช้เป็น Template ใน Step 730)

| ส่วนของ LLD | เนื้อหา |
|---|---|
| 1. Device Inventory & Platform | รายชื่อ Device ทุกตัว, Platform, Building ที่สังกัด (ต่อยอดจาก 00-ip-address-plan.md เดิม) |
| 2. Loopback0/Router-ID Table | Loopback ทุกตัวต้อง Unique ทั่วทั้ง Enterprise — ใช้ Scheme `1.<Building>.<Role>.<Seq>` |
| 3. AS Number Table | AS ของทุก Building, ระบุชัดว่าเป็น iBGP(65001) หรือ eBGP(Private AS เฉพาะตัว) |
| 4. Area/Process Assignment Table | OSPF Process ID/Area หรือ EIGRP AS ของแต่ละ Building |
| 5. Redistribution/Summarization Boundary Table | ทุกจุดที่มี Redistribute/Aggregate เกิดขึ้น พร้อม Route-map/Prefix-list ที่ใช้ |
| 6. BGP Peering Matrix | ใคร Peer กับใคร, RR-Client หรือ eBGP, Maximum-prefix ที่ตั้งไว้ |
| 7. Maintenance/CAB Log | ประวัติ Change ทุกครั้งที่กระทบ WAN Core หรือมากกว่า 1 Building |
| 8. Rollback Command ต่อ Major Change | คำสั่ง Rollback ที่เตรียมไว้ล่วงหน้าก่อนเริ่มแก้จริง (Pattern เดียวกับ Part 69 Step 688.2) |

---

## Step 730 — Lab เต็มรูปแบบ: เอกสาร Routing Architecture ครบชุดของ 10-Building Campus

### 730.1 ตารางที่ 1 — Per-Building OSPF/EIGRP Area/AS Assignment (สรุปจาก Step 721-724)

| Building | IGP | Process/AS ภายใน | Area (ถ้า OSPF) | Router ประมาณ | เข้าเกณฑ์ ~50/Area? |
|---|---|---|---|---|---|
| 1 (HQ) | OSPF | Process 1 | Area 0 (WAN) / Area 1 / Area 2 | ~25 | ✅ |
| 2 | OSPF | Process 1 | Area 0 (Internal) | ~35 | ✅ |
| 3 | OSPF | Process 1 | Area 0 (Internal) | ~40 | ✅ |
| 4 | OSPF | Process 1 | Area 0 (Internal) | ~30 | ✅ |
| 5 | OSPF | Process 1 | Area 0 (Internal) | ~30 | ✅ |
| 6 (Data Center) | OSPF | Process 1 | Area 0 (Internal) | ~45 | ✅ (ใกล้เพดาน — เฝ้าดูตอนขยาย) |
| 7 | OSPF | Process 1 | Area 0 (Internal) | ~35 | ✅ |
| 8 (R&D) | OSPF | Process 1 | Area 0 + Area 11 (แบ่งเพราะเกิน 50) | ~65 (แบ่งเป็น 2 Area ~30+35) | ✅ (หลังแบ่ง) |
| 9 (M&A) | EIGRP | AS 200 | *(ไม่มี Area — EIGRP ไม่มีแนวคิดนี้)* | ~20 | ไม่เกี่ยวข้อง |
| 10 (M&A) | EIGRP | AS 200 | *(ไม่มี Area)* | ~18 | ไม่เกี่ยวข้อง |

### 730.2 ตารางที่ 2 — Building-Edge iBGP/eBGP AS/RR Design

| Building | AS | ประเภท Peering | Peer กับ | Multipath | Max-prefix Defense |
|---|---|---|---|---|---|
| 1 | 65001 | iBGP RR-Client | WAN-CORE-RR1, RR2 | `maximum-paths ibgp 2` | ไม่จำเป็น (Trusted, Native) |
| 2-8 | 65001 | iBGP RR-Client | WAN-CORE-RR1, RR2 | `maximum-paths ibgp 2` | ไม่จำเป็น |
| 9 | 65009 | eBGP (Private AS) | WAN-CORE-RR1, RR2 | `maximum-paths eibgp 2` + `as-path multipath-relax` | ✅ `maximum-prefix 20 80` |
| 10 | 65010 | eBGP (Private AS) | WAN-CORE-RR1, RR2 | `maximum-paths eibgp 2` + `as-path multipath-relax` | ✅ `maximum-prefix 20 80` |
| WAN Core | 65001 | iBGP (RR ↔ RR) | RR1 ↔ RR2 | N/A (2 RR Full-mesh กันเอง) | ไม่จำเป็น |

### 730.3 ตารางที่ 3 — แผน Summarization ทุกขอบเขต (รวมจาก Step 725)

| ขอบเขต | Prefix ที่ปล่อยผ่านจริง | เครื่องมือ |
|---|---|---|
| Building 1 → BGP | `10.10.0.0/16` | `network` (Native จาก 00-ip-address-plan.md เดิม) |
| Building 2-8 → BGP | `10.10X.0.0/16` (X=2-8) | `aggregate-address ... summary-only` |
| Building 9-10 → BGP | `10.10X.0.0/16` (X=9-10) | `redistribute eigrp 200 route-map SUMMARY-ONLY` |
| BGP → ทุก Building | `0.0.0.0/0` เท่านั้น | `default-information originate` (OSPF) / `redistribute bgp route-map DEFAULT-ONLY` (EIGRP) |
| ผลลัพธ์สุทธิที่ WAN Core BGP Table | **10 Prefix เท่านั้น** | (เทียบกับ 150-200 Prefix ถ้าไม่ Summarize ตาม Step 725.1) |

### 730.4 ตารางที่ 4 — Convergence-Time Comparison: IGP-only Redistribution-heavy vs BGP-Glued Design

| มิติการเปรียบเทียบ | Design A: Mutual Redistribution แบบ Part 31 ขยายเป็น 10 Domain | Design B: BGP-Glued (Design ของ Part นี้) |
|---|---|---|
| จำนวนจุด Redistribution สูงสุด | สูงสุด 45 จุด (n(n-1)/2) ถ้าทำแบบ Full-Mesh, หรือ ~10-20 จุดถ้ารวมที่ WAN Core (แต่ยังเป็น ASBR ที่ซับซ้อนมาก) | 10 จุด (1 จุดต่อ Building, ทิศทางเดียว — ไม่ Mutual) |
| กลไกกันลูป | Route Tag ที่ต้อง Config เองทุกจุด (เสี่ยง Human Error) | AS_PATH ในตัว Protocol (Automatic) |
| Blast Radius ของ Config ผิดพลาด 1 จุด | สูง — Tag ชนกัน 1 คู่ กระทบ 2 Domain โดยตรง และอาจลาม Suboptimal Path ไปทั้งระบบ | ต่ำ — Prefix ผิดพลาดกระทบแค่ Building เดียว (AS_PATH ป้องกัน Loop ข้าม Domain อื่นให้อัตโนมัติ) |
| Convergence เมื่อ Link ภายใน 1 Domain ล่ม | < 1 วินาที (เหมือนกันทั้งคู่ — เป็นเรื่องของ IGP ล้วนๆ) | < 1 วินาที (เหมือนกัน) |
| Convergence เมื่อจุด Redistribution/Building-Edge ล่ม | ไม่แน่นอน — ขึ้นกับ AD/Metric ที่ Config ไว้ถูกหรือผิด อาจเกิด Suboptimal Path เงียบๆ โดยไม่มี Error, Debug ยากเพราะ Metric ข้าม Protocol เทียบกันไม่ได้ | 1-2 วินาที (มี BGP Multipath พร้อมอยู่แล้วจาก Step 727) |
| ความยากในการเพิ่ม Building ที่ 11 | สูงมาก — ต้องแก้ Redistribution Point เดิมทุกจุดที่เกี่ยวข้อง เสี่ยง Regression กับ Building ที่มีอยู่ | ต่ำ — เพิ่ม BGP Neighbor 1 คู่ที่ WAN-Core-RR เท่านั้น ไม่แก้ Config ของ Building เดิมเลย |
| ต้องใช้ LLD ระดับ Enterprise หรือไม่ | ใช่ (จำเป็นเท่ากัน) | ใช่ (จำเป็นเท่ากัน) |

### 730.5 ข้อสรุปการออกแบบ (Design Justification) — คำตอบสุดท้ายของ Part นี้

จากตารางทั้ง 4 ชุดข้างบน **Design B (BGP-Glued Multi-Domain)** คือสถาปัตยกรรมที่ CCIE
Enterprise Infrastructure แนะนำสำหรับ 10-Building Campus นี้ เพราะ:

1. **จำนวนจุดเสี่ยง (Redistribution Point) น้อยกว่ามาก** และแต่ละจุดเป็น**ทิศทางเดียว**
   (IGP→BGP กับ BGP→IGP แยกกันชัดเจน) ไม่ใช่ Mutual ที่เสี่ยง Loop แบบ Part 31 Step 307
2. **Blast Radius ของความผิดพลาดถูกจำกัดไว้ที่ระดับ Building** ด้วย AS_PATH ที่กันลูป
   อัตโนมัติ — ไม่ต้องพึ่ง Route Tag Discipline ที่ต้องดูแลเองทั้งหมด
3. **Scale ต่อไปได้แบบ Linear** — เพิ่ม Building ใหม่ไม่กระทบ Config เดิมของ Building ที่มี
   อยู่แล้วเลย ตรงข้ามกับ Mutual Redistribution ที่ความซับซ้อนโตแบบ Quadratic
4. **Convergence ของ Failure ที่เกิดถี่ที่สุด (Link ภายใน Building) เท่ากันทั้งสอง Design** —
   ไม่เสียอะไรเลยในสถานการณ์ที่เกิดบ่อยที่สุด ในขณะที่ได้ประโยชน์ทั้งหมดข้างต้นในสถานการณ์ที่
   เกิดไม่บ่อยแต่กระทบมาก
5. **Reference-Bandwidth/K-value Tuning ถูก Decouple ออกจากกันได้จริง** (Step 728) — ผลพลอยได้
   ที่ทำให้ Operation ในแต่ละ Building เป็นอิสระจากกันมากขึ้นไปอีกขั้น

> **Design B ไม่ได้ "ดีกว่า Redistribution เสมอไปทุกสเกล"** — ที่สเกล 1-2 Domain (แบบ Lab เดิม
> Part 1-35 ก่อน Part 72) Mutual Redistribution ของ Part 31 ยังคงเป็นทางเลือกที่ถูกต้องและง่าย
> กว่า BGP-Glue มาก (ไม่ต้องจอง AS Number เพิ่ม ไม่ต้องตั้ง RR) — **บทเรียนที่สำคัญที่สุดของ
> Part นี้คือ "รู้ว่าจะเปลี่ยน Pattern การออกแบบตอนไหน" ไม่ใช่ "Pattern ไหนดีที่สุดตลอดกาล"**
> ซึ่งคือหัวใจของการคิดแบบ CCIE Design ที่แตกต่างจากการท่องจำ Best Practice แบบตายตัว

---

## แบบฝึกหัดทวนความเข้าใจ Part 73

1. เพราะเหตุใดการใช้ OSPF Domain เดียวคลุมทั้ง 10 Building (แบ่ง Area ตาม Building) จึงยังมี
   ข้อจำกัดอยู่ แม้จะออกแบบ Area และ ABR ถูกต้องตามกฎ ~50 Router/Area แล้วก็ตาม?
2. อธิบายว่าทำไม BGP AS_PATH จึงแก้ปัญหา Routing Loop จาก Mutual Redistribution (Part 31
   Step 307) ได้ดีกว่า Route Tag โดยไม่ต้อง Config อะไรเพิ่มเลย
3. ที่ Building-Edge Router ตัวหนึ่ง ทำไมต้อง Config ทั้ง `aggregate-address ... summary-only`
   (ทิศทาง IGP→BGP) และ `default-information originate` (ทิศทาง BGP→IGP) พร้อมกัน ถ้าทำแค่
   ทิศทางเดียวจะเกิดปัญหาอะไร?
4. เพราะเหตุใด BGP จึงไม่สามารถเลือก Path ที่มี Bandwidth สูงกว่าได้อัตโนมัติเหมือน OSPF Cost
   และต้องแก้ปัญหานี้ด้วยเครื่องมือใด?
5. ตามตัวเลข Convergence ใน Step 726 เพราะเหตุใด Design ที่ดีจึงควรทำให้ BGP "ไม่รู้เรื่อง"
   การล่มของ Link ภายใน 1 Building เลย?

**เฉลย:**

1. เพราะ **Type 5 (External) LSA Flood ทั่วทั้ง Domain โดยไม่สนใจ Area Boundary** เลย (Area
   จำกัดได้แค่ Type 1/2/3) ถ้ามี Domain อื่น (เช่น EIGRP Building 9-10) ถูก Redistribute เข้า
   OSPF จุดใดจุดหนึ่ง External Route นั้นจะ Flood เข้าไปถึงทุก Area และ Link Flap ที่ Area 0
   (WAN Core) ยังกระทบ ABR ของทุก Building พร้อมกันเสมอ — Blast Radius ไม่ได้ลดลงตามจำนวน
   Building ที่เพิ่มขึ้น
2. เพราะ AS_PATH เป็นกลไกกันลูป**มาตรฐานของ Protocol เอง** — ถ้า Route ที่ Advertise ออกจาก AS
   หนึ่งวนกลับมาเจอ AS Number ของตัวเองอยู่ใน AS_PATH BGP จะ Reject ทันทีโดยอัตโนมัติ ต่างจาก
   Route Tag ของ Redistribution ที่ต้องออกแบบ วางแผน และ Maintain เลข Tag ไม่ให้ชนกันเองทุกจุด
   ด้วยมือ (Human Error ได้ง่ายกว่ามาก)
3. เพราะทั้งสองทิศทางแก้ปัญหาคนละด้าน — `aggregate-address summary-only` ป้องกันไม่ให้ BGP
   Table ของทั้ง Enterprise บวมด้วย Subnet ปลีกย่อยของอาคารเดียว ส่วน
   `default-information originate` ป้องกันไม่ให้ Building อื่นต้องรับ Full BGP Table เข้า IGP
   ของตัวเอง (Full Table Leak ตาม Part 31 Step 308) — ถ้าทำแค่ทิศทางเดียว อีกทิศทางจะยังมี
   ปัญหา Scale เดิมอยู่ (BGP Table บวม หรือ IGP ทุกอาคารรับ External Route มากเกินจำเป็น)
4. เพราะ BGP Best Path Algorithm เลือกจาก Attribute (WEIGHT/LOCAL_PREF/AS_PATH/MED) ไม่มี
   แนวคิด "Cost ตาม Bandwidth" อยู่ในตัว Protocol เลย — ต้องแก้ด้วยการตั้ง **LOCAL_PREF** (หรือ
   WEIGHT) ให้สูงกว่าบน Neighbor ที่ต่อผ่าน Link ความเร็วสูงเสมอด้วยมือ ไม่ปล่อยให้ BGP
   Tie-break เอง ไม่เช่นนั้นอาจเกิด ECMP แบ่ง Traffic ไปทาง Link ความเร็วต่ำจนล้น
5. เพราะ BGP Convergence ช้ากว่า IGP โดยธรรมชาติ (มี MRAI, ต้องรอ Update/Withdraw ผ่าน RR,
   ไม่มี BFD ผูกอัตโนมัติ) — ถ้าปล่อยให้ Failure ที่เกิดถี่ (Access/Distribution Link ภายใน
   Building) กระทบ BGP ด้วย จะทำให้ Convergence โดยรวมช้าลงจากค่าที่ IGP เดิมทำได้ (< 1 วินาที)
   การ Summarize Building Supernet ไว้ (Step 725) ทำให้ BGP Table ไม่เปลี่ยนแปลงเลยเมื่อ Link
   ภายในอาคารล่ม — BGP จึงถูกใช้เฉพาะสำหรับ Route ที่นิ่งและ Redundancy ระดับ Building-to-Core
   เท่านั้น ตรงตามหลักการ "แบ่งงานตามความถี่ของการเปลี่ยนแปลง" ใน Step 726.3

---

## สรุป Part 73

Part นี้คือจุดที่ความรู้ OSPF (Part 26), EIGRP (Part 28), BGP (Part 30), และ Route
Redistribution (Part 31) ที่เรียนแยกกันมาตลอดหลักสูตร ถูกนำมา**สังเคราะห์เป็นสถาปัตยกรรม
Routing เดียว**สำหรับ 10-Building Campus จาก Part 72 — เริ่มจาก **Decision Framework** ที่
ตอบว่าจะเลือก IGP ไหนที่ Domain ไหน (Step 721), ทดลองออกแบบ **OSPF Area-per-Building** แบบ
Domain เดียวแล้วชี้ข้อจำกัดที่ Area ไม่แก้ได้ (Step 722), เปลี่ยนมุมมองไปสู่ **BGP-as-Glue**
ที่แก้ปัญหา Mutual Redistribution จากรากฐานด้วย AS_PATH (Step 723), Worked Design เต็มรูปแบบ
พร้อม iBGP RR-Client และ eBGP Private-AS สำหรับ Building M&A (Step 724), แผน **Summarization**
ทุกขอบเขตที่ลด BGP Table จาก 150-200 Prefix เหลือ 10 Prefix (Step 725), วิเคราะห์
**Convergence** ตัวเลขจริงและหลักการ "แบ่งงานตามความถี่การเปลี่ยนแปลง" ระหว่าง IGP กับ BGP
(Step 726), ต่อยอด **Multi-homing** จาก Part 33 ขึ้นสู่ระดับ Building (Step 727), จัดการ
**Asymmetric Bandwidth** ทั้งใน IGP (Cost/Metric ที่ตอนนี้ Decouple ต่อ Building ได้) และ BGP
(ต้องใช้ LOCAL_PREF เพราะไม่มี Bandwidth-awareness ในตัว) (Step 728), ย้ำความสำคัญของ
**LLD Document และ Change Management** ที่สเกลนี้ (Step 729), และปิดท้ายด้วย **เอกสาร
Architecture ครบชุด** พร้อมตาราง Convergence Comparison ที่พิสูจน์ว่าทำไม BGP-Glued Design
จึงเหมาะกับสเกลนี้มากกว่า Mutual Redistribution แบบเดิม (Step 730) ✅

**สิ่งที่ Design นี้ยังไม่ได้แตะ**: WAN-CORE-RR1/RR2 และ Building-Edge ทุกตัวยังต่อกับ ISP/
Internet ผ่าน AS 65001 เดียวเท่านั้น (ตามที่ Part 29-30 วางไว้ตั้งแต่ต้น) — ที่สเกล 10-Building
Enterprise จริง คำถามต่อไปคือ **WAN Edge เองต้องออกแบบใหม่ทั้งหมดหรือไม่**: ต้องมี Multi-ISP,
Multi-homing WAN Edge กี่ชุด, และจะทำ Traffic Engineering ข้าม ISP อย่างไรเมื่อ Traffic ทั้ง
10 Building ต้องแข่งกันออก Internet จากจุดเดียว

**ไปต่อ:** [Part 74 — Advanced BGP & WAN Edge Design →](part-074-advanced-bgp-wan-edge.md)
