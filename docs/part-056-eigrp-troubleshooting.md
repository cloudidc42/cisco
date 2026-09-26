# Part 56 — Advanced EIGRP Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 551–560 จาก 1000**

> ต่อจาก [Part 55 — ENCOR Capstone Lab & Exam Prep](part-055-encor-capstone-lab.md) ที่ปิดระดับ
> **CCNP ENCOR (350-401)** อย่างสมบูรณ์ **Part นี้คือจุดเริ่มต้นของระดับ CCNP ENARSI (300-410)**
> อย่างเป็นทางการ — ENARSI มีสูตรข้อสอบที่ต่างจาก ENCOR อย่างชัดเจน: ENCOR เน้น "ออกแบบและตั้งค่า
> Feature ให้ถูกตั้งแต่แรก" ส่วน **ENARSI เน้น "ให้ Config ที่เสียมาแล้ว หาสาเหตุให้เจอ แก้ให้เร็วที่สุด"**
> (Troubleshooting-first Exam Format) ข้อสอบจริงจะให้ Topology พร้อม Running-config ที่มี Bug
> ซ่อนอยู่ 1-3 จุด แล้วถามว่า "เพราะเหตุใด Router A กับ Router B จึงไม่เห็นกันเป็น Neighbor" หรือ
> "เพราะเหตุใด Traffic จาก Site A ไปยัง Site B จึงไม่วิ่งตาม Path ที่ควรจะเป็น"
>
> Part นี้และอีก 14 Part ถัดไป (Part 56-70) จะไม่สอน Feature ใหม่เป็นหลักเหมือน Part 1-55 ที่ผ่านมา
> แต่จะ**นำ Feature ที่เรียนไปแล้วทั้งหมดกลับมาทำให้เสียโดยตั้งใจ** บน Standalone EIGRP Demo
> Topology เดิมที่วางฐานไว้ตั้งแต่ [Part 13](part-013-eigrp-fundamentals.md) (DUAL, Metric,
> Classic/Named Mode, Neighbor Requirement, Summarization, Variance) และขยายเพิ่มใน
> [Part 28](part-028-eigrp-advanced.md) (Stub Routing, `distribute-list`, Summarization ระดับ
> Enterprise, SIA, `bandwidth-percent`, Key Rotation, BFD) — ใช้อุปกรณ์ชุดเดิมทุกตัว: **DIST-SW3**,
> **DIST-SW4** (Port สำรองของ Lab หลัก) และ **EIGRP-R1**, **EIGRP-R2** (Legacy Site Router) รัน
> **EIGRP AS 100** (Named Mode process `EIGRP-DEMO`) ไม่แตะ OSPF ของ Lab หลักแม้แต่บรรทัดเดียว
> เช่นเดิม

## ทวน Topology และ Baseline Config ที่ "ควรจะเป็น" (จาก Part 13 Step 130 + Part 28 Step 280)

ก่อนไล่หา Bug ทุก Step ในนี้ **ต้องจำ Baseline ที่ถูกต้องให้แม่นก่อน** — ทุก Scenario ใน Part นี้คือ
การเบี่ยงเบนจาก Baseline นี้ 1 จุด (หรือมากกว่าใน Step 560):

```
                          EIGRP-R1  (Legacy Site "BKK-OLD" #1)  Lo0: 9.9.9.1/32
                          eigrp stub connected summary
                         ┌─────────┴─────────┐
                Link1    │                   │   Link2
        10.199.34.0/30   │                   │  10.199.34.4/30 (GigE เต็มสปีด)
     BW=100,000 DLY=100  │                   │
    MD5 (key-chain EIGRP-KEY)                │
                   ┌──────┴──────┐     ┌──────┴──────┐
                   │  DIST-SW3   │     │  EIGRP-R2    │  Lo0: 9.9.9.2/32
                   │ AS 100      │     │  AS 100      │  Lo1: 10.199.200.1/24 "Legacy Server LAN"
                   │ summary→R1  │     │(Legacy Site  │  eigrp stub connected summary
                   └──┬───────┬──┘     │  "BKK-OLD"#2)│
             Link4     │       │Link5  └──┬────────┬──┘
        10.199.34.12/30│       └──────────┘        │
       BW=1,000,000    │      10.199.34.16/30      │ Link3
       DLY=10 (GigE)   │      BW=100,000 DLY=100    │ 10.199.34.8/30
       MD5             │      MD5                  │ BW=1,544 DLY=20,000 (จำลอง Serial WAN)
                   ┌────┴────────┐                  │ MD5
                   │  DIST-SW4    │──────────────────┘
                   │ AS 100       │
                   └──────────────┘
```

| รายการ Baseline | ค่าที่ถูกต้อง |
|---|---|
| AS Number / Process Name | AS 100, `router eigrp EIGRP-DEMO` (Named Mode) |
| K-value | Default ทุกตัว: K1=1, K2=0, K3=1, K4=0, K5=0 |
| MD5 Authentication | Key-chain `EIGRP-KEY`, Key-string `C1sco123!EIGRP` (Key 2 หลัง Key Rotation Part 28 Step 277) |
| Stub Router | EIGRP-R1, EIGRP-R2 = `eigrp stub connected summary` |
| Summarization | DIST-SW3 Gi1/0/23 (Link1) = `ip summary-address eigrp 100 10.199.34.0 255.255.255.0` |
| Passive Interface | ไม่มี Interface ใดใน EIGRP AS 100 ควรเป็น Passive (ทุก Link เป็น Transit Link ทั้งหมด) |
| Distribute-list | ไม่มี Distribute-list ที่ควรบล็อก `10.199.200.0/24` หรือ Summary Route |
| Bandwidth Link3 | จำลอง Serial ตั้งใจ = `bandwidth 1544` — **แต่ Step 556 จะสาธิตกรณี Physical Link ถูกอัปเกรดจริงแล้ว Config ไม่ได้อัปเดตตาม** |

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 551 | Systematic EIGRP Troubleshooting Methodology — Decision Tree ที่ใช้ตลอด Part |
| 552 | Scenario 1 — Neighbor ไม่ขึ้นจาก K-value Mismatch (Custom K-value จาก Config เก่า) |
| 553 | Scenario 2 — Neighbor ไม่ขึ้นจาก Primary Subnet vs Secondary IP Address ไม่ตรงกัน |
| 554 | Scenario 3 — Route หายจาก `passive-interface` ที่ใส่ผิดบน Transit Link |
| 555 | Scenario 4 — SIA จาก Link Flapping ร่วมกับ Summarization ที่ไม่ครบ (ต่อจาก Part 28 Step 275) |
| 556 | Scenario 5 — Asymmetric/Suboptimal Routing จาก `bandwidth` ที่ไม่ตรงกับ Physical Link จริง |
| 557 | Scenario 6 — Route อยู่ใน Topology Table แต่ไม่อยู่ใน Routing Table (`distribute-list` ที่ลืมถอด) |
| 558 | Scenario 7 — Authentication Mismatch หลัง Key Rotation (Key-chain Lifetime พลาดกำหนดเวลา) |
| 559 | Scenario 8 — Stub Site กลายเป็น Transit Path เพราะลืม `eigrp stub` |
| 560 | Lab เต็มรูปแบบ — 3 Bug ซ้อนกันในโจทย์เดียว พร้อม Before/After ทุกจุด |

---

## Step 551 — Systematic EIGRP Troubleshooting Methodology: Decision Tree

ข้อสอบ ENARSI ไม่ถามว่า "EIGRP ทำงานอย่างไร" (นั่นคือข้อสอบ CCNA/ENCOR) แต่ถามว่า **"ทำไมมันไม่ทำงาน
ตามที่ควร"** — สิ่งที่ทำให้คนตกข้อสอบ ENARSI บ่อยที่สุดไม่ใช่ไม่รู้ Theory แต่คือ**ไล่เช็คแบบสุ่ม
ไม่มีระบบ** จนเสียเวลาหมดในข้อที่ตอบได้ถ้าไล่ถูกลำดับ Part นี้เสนอ **Decision Tree 5 ชั้น** ที่ใช้ซ้ำ
ทุก Scenario ตั้งแต่ Step 552-560 — ก่อนแก้ปัญหาใดๆ ต้องไล่ตามลำดับนี้เสมอ **ห้ามข้ามชั้น**

### Decision Tree เต็มรูปแบบ

```
┌─────────────────────────────────────────────────────────────────────┐
│ ชั้น 1: Neighbor ขึ้นหรือไม่?  (show ip eigrp neighbors)              │
└───────────────────────┬───────────────────────────────────────────┬──┘
                    ไม่ขึ้น│                                    ขึ้น│
                          ▼                                       ▼
        ┌─────────────────────────────────┐      ┌───────────────────────────────────┐
        │ ไล่เช็คตามลำดับ (Step 125 เดิม): │      │ ชั้น 2: Topology Table ถูกต้องหรือไม่? │
        │ 1. show ip int brief (L1/L2 up?) │      │ (show ip eigrp topology)             │
        │ 2. show run int (Primary Subnet) │      │  - มี Successor ที่สมเหตุสมผลไหม        │
        │ 3. show ip protocols (AS, K-val) │      │  - ค้างสถานะ Active (SIA) ไหม           │
        │ 4. show ip eigrp interfaces      │      │  - Prefix ที่ควรมีหายไปจาก Table เลยไหม  │
        │    (Passive หรือไม่)              │      └──────────────────┬────────────────────┘
        │ 5. debug eigrp packets           │                    ถูกต้อง│      │ผิดปกติ/SIA
        │    (Authentication/K-value log)  │                         ▼      ▼
        │ → ตรงกับ Step 552/553/554/558    │        ┌───────────────────┐  ┌─────────────────┐
        └─────────────────────────────────┘        │ ชั้น 3: Route ถูก  │  │ ตรงกับ Step 555   │
                                                    │ Install ลง RIB    │  │ (SIA) / Step 559  │
                                                    │ หรือไม่?           │  │ (Stub หาย → กลาย  │
                                                    │ show ip route     │  │ เป็น Transit)      │
                                                    │ vs show ip eigrp  │  └─────────────────┘
                                                    │ topology          │
                                                    └────────┬──────────┘
                                                    มีใน RIB │   ไม่มีใน RIB
                                                             ▼        ▼
                                          ┌─────────────────────┐  ┌───────────────────┐
                                          │ ชั้น 4: Metric/Path  │  │ ตรงกับ Step 557     │
                                          │ ที่เลือกสมเหตุสมผล   │  │ (distribute-list    │
                                          │ หรือไม่? (bandwidth, │  │ บล็อกอยู่)           │
                                          │ delay, variance)     │  └───────────────────┘
                                          │ → ตรงกับ Step 556    │
                                          └──────────┬───────────┘
                                          สมเหตุสมผล │  ผิดปกติ
                                                     ▼        ▼
                                          ┌────────────────┐ (กลับไป Step 556)
                                          │ ชั้น 5: Traffic  │
                                          │ วิ่งจริงตาม Path │
                                          │ ที่ RIB บอกไหม?  │
                                          │ ping/traceroute/ │
                                          │ show ip cef       │
                                          └────────────────┘
```

### ทำไมต้องไล่เป็นชั้นๆ ห้ามข้าม

| ถ้าข้ามชั้น... | จะเกิดอะไร |
|---|---|
| ข้ามชั้น 1 ไปเช็คชั้น 2 (Topology Table) ทั้งที่ Neighbor ไม่ขึ้นด้วยซ้ำ | เสียเวลาไล่ Metric/Summarization ทั้งที่ต้นเหตุจริงคือ Layer ต่ำกว่า (K-value/Subnet/Passive) — ข้อสอบชอบซ่อน Bug ไว้ที่ Neighbor แต่ตั้งคำถามให้ดูเหมือนปัญหา Route |
| ข้ามชั้น 3 ไปดูชั้น 4 (Metric) ทั้งที่ Route ไม่อยู่ใน RIB เลย | เสียเวลาคำนวณ Metric ของ Route ที่ไม่มีทางถูกใช้อยู่ดี เพราะมันถูก Filter ออกไปตั้งแต่ชั้น 3 แล้ว |
| ข้ามชั้น 5 (Traffic จริง) | บาง Case Config EIGRP ถูกทุกอย่าง (RIB มี Route ที่ถูกต้อง) แต่ Traffic ยังไปไม่ถึงเพราะ ACL/NAT/Firewall คั่นอยู่ — Bug ไม่ใช่ EIGRP เลยแต่เป็น Layer อื่น (ธีมนี้เจอบ่อยในข้อสอบจริงที่หลอกให้คิดว่าเป็น Routing Bug) |

> **กฎทอง ENARSI**: **ทุก Scenario ต้องเริ่มที่ `show ip eigrp neighbors` เสมอ** ไม่ว่าอาการที่โจทย์
> บอกมาจะดูเหมือนเป็นปัญหาระดับสูงแค่ไหนก็ตาม (เช่น "Traffic ช้า", "เส้นทางไม่ตรงที่คาด") เพราะถ้า
> Neighbor ไม่ขึ้นตั้งแต่แรก ทุกอาการที่เหลือคือผลข้างเคียงทั้งหมด ไม่ต้องไปแก้ที่อื่นเลย

### คำสั่งตั้งต้นที่ต้องรันทุกครั้งก่อนอ่านต่อ (5 คำสั่งจำให้ขึ้นใจ)

```
Router# show ip eigrp neighbors           ! ชั้น 1
Router# show ip eigrp topology            ! ชั้น 2 (เพิ่ม "active" หรือ "all-links" ตามสงสัย)
Router# show ip route eigrp                ! ชั้น 3 (เทียบกับผลชั้น 2)
Router# show ip eigrp topology <prefix>    ! ชั้น 4 (ดู Metric แยกราย Path)
Router# traceroute <destination>            ! ชั้น 5
```

Step 552-559 ที่เหลือของ Part นี้ คือตัวอย่างจริงของแต่ละ "จุดแตก" ในต้นไม้นี้ — Step 560 ปิดท้าย
ด้วยโจทย์ที่ผสม 3 จุดแตกจากคนละชั้นเข้าด้วยกัน เพื่อฝึกไล่ Decision Tree แบบเต็มรูปแบบเหมือนข้อสอบจริง

---

## Step 552 — Scenario 1: Neighbor ไม่ขึ้นจาก K-value Mismatch

### อาการที่รายงานมา

Site BKK-OLD (EIGRP-R1/EIGRP-R2) แจ้งว่า "จู่ๆ Legacy Server LAN (`10.199.200.0/24`) เข้าไม่ได้จาก
Core เลย" — ทีมงานเพิ่งมีวิศวกรใหม่ไปทำ "Performance Tuning" ที่ EIGRP-R1 เมื่อวาน แล้ววันนี้ก็เสีย

### ชั้น 1 — ตรวจ Neighbor

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        13  00:15:33    2    100  0  8
2   10.199.34.14    Gi1/0/24        14  00:15:33    1    100  0  4
```

Neighbor ทาง `Gi1/0/23` (Link1 ไปหา EIGRP-R1) **หายไปจาก List เลย** — ทั้งที่ DIST-SW4 (`.14`) และ
EIGRP-R2 ทาง Link5 (`.18`) ยังขึ้นปกติ นี่คือสัญญาณว่า Bug จำกัดอยู่ที่ Link1 เท่านั้น

### ดู Log ทันทีที่เกิดเหตุ

```
DIST-SW3#
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.2 (GigabitEthernet1/0/23) is down: K-value mismatch
```

Log บอกตรงๆอยู่แล้วว่า **K-value mismatch** — แต่เพื่อฝึกกระบวนการวินิจฉัยแบบเต็ม (ข้อสอบจริงมักไม่
โชว์ Log ให้ตรงๆ) มาเช็คด้วย `show ip protocols` ทั้งสองฝั่งตามชั้น 1 ของ Decision Tree

### เช็คให้แน่ชัดด้วย `show ip protocols`

```
DIST-SW3# show ip protocols | include K1
  Metric weight K1=1, K2=0, K3=1, K4=0, K5=0, K6=0
```

```
EIGRP-R1# show ip protocols | include K1
  Metric weight K1=1, K2=0, K3=2, K4=0, K5=0, K6=0
```

**พบต้นเหตุ**: EIGRP-R1 มี **K3 = 2** (ควรเป็น 1) — ผิดจาก Default

### ดู Running-config ที่เป็นสาเหตุ

```
EIGRP-R1# show running-config | section router eigrp
router eigrp EIGRP-DEMO
 !
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 9.9.9.1
  metric weights 0 1 0 2 0 0        ! <<< บรรทัดนี้คือสาเหตุ — วิศวกรใหม่พิมพ์เข้ามาเมื่อวาน
  network 9.9.9.1 0.0.0.0
  network 10.199.34.0 0.0.0.255
  ...
```

วิศวกรใหม่ตั้งใจจะเพิ่มน้ำหนัก **Delay** ให้มีผลต่อ Metric มากขึ้น (คิดว่ากำลัง "Tune" Path
Selection) แต่ไม่รู้ว่า **K-value ต้องตรงกันทุก Router ในโดเมนเดียวกันเป๊ะ** (ทวนจาก
[Part 13 Step 125](part-013-eigrp-fundamentals.md)) — เพราะ EIGRP-R1 เป็นจุดเชื่อมเดียวของ Link1
การแก้ K-value ที่จุดเดียวนี้ตัด Neighbor กับ DIST-SW3 ทันที และเนื่องจาก DIST-SW3 Path A
(Link5 ตรงไป EIGRP-R2) ยังเป็น Successor ปกติ **Traffic ที่เหลือยังไปได้** แต่ก็เสีย Feasible
Successor Backup (Path B ผ่าน EIGRP-R1) ไปเลย ทำให้ Network เสี่ยงมากขึ้นถ้า Path A ล่มอีกที

> **หมายเหตุ**: ช่วงที่ K-value เพิ่งเปลี่ยน Neighbor จะ Flap ขึ้นๆลงๆสองสามรอบก่อน Log จะสรุปสถานะ
> "down" ค้าง — ผู้ใช้ที่เจอ Traffic สะดุดตอน Flap พอดีอาจรายงานเกินจริงว่า "เข้าไม่ได้เลย" ทั้งที่
> Path A ยังพาไปถึงได้ปกติหลัง Flap สงบ ต้องแยกอาการ "เข้าไม่ได้เลย" กับ "สะดุดชั่วครู่" ให้ออกเสมอ

### แก้ไข

```
EIGRP-R1(config)# router eigrp EIGRP-DEMO
EIGRP-R1(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R1(config-router-af)# no metric weights 0 1 0 2 0 0
EIGRP-R1(config-router-af)# end
```

`no metric weights ...` คืนค่า K-value กลับสู่ Default (K1=1, K3=1, ที่เหลือ=0) — ไม่มี Syntax
"reset to default" เฉพาะ ต้องใช้ `no` ปิดหน้าค่าที่ตั้งไว้เดิมแบบเป๊ะๆเท่านั้น (หรือใช้
`metric weights 0 1 0 1 0 0` ตั้งค่า Default กลับเข้าไปตรงๆก็ได้ผลเหมือนกัน)

### Verify หลังแก้

```
DIST-SW3#
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.2 (GigabitEthernet1/0/23) is up: new adjacency

DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
2   10.199.34.2     Gi1/0/23        14  00:00:12    3    200  0  2
1   10.199.34.18    Gi1/0/25        13  00:22:47    2    100  0  8
0   10.199.34.14    Gi1/0/24        14  00:22:47    1    100  0  4

DIST-SW3# show ip eigrp topology 10.199.200.0/24
P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23     <- Feasible Successor กลับมาแล้ว
```

**บทเรียน ENARSI**: `metric weights` (K-value) เป็นคำสั่งที่ **แทบไม่มีเหตุผลให้แก้ในงานจริงเลย**
(ทวนจาก Part 13) — เมื่อเจอปัญหา Neighbor หลุดพร้อม Log ระบุ K-value mismatch ให้สงสัยว่ามีคนไป
แก้ค่านี้แบบไม่ได้ตั้งใจ (มักมาจากความเข้าใจผิดว่ากำลัง Tune Metric คล้าย `bandwidth`/`delay`)
วิธีแก้ที่ถูกต้องเกือบทุกครั้งคือ **คืนค่า Default ให้ตรงกันทุกตัว ไม่ใช่ไปตั้งค่าใหม่ให้ตรงกันที่ค่า
อื่น**

---

## Step 553 — Scenario 2: Neighbor ไม่ขึ้นจาก Primary Subnet vs Secondary IP Address

### อาการที่รายงานมา

ทีม Network กำลังทำโครงการ Renumbering ย้าย Subnet ของ Link1 จาก `10.199.34.0/30` ไปเป็น
`10.199.35.0/30` แบบ Manual ทีละฝั่งเพื่อลด Downtime (แผนคือใส่ Subnet ใหม่เป็น Secondary ก่อน
ทดสอบ แล้วค่อยสลับ Primary ทีหลัง) — หลังจากแก้ Config บน EIGRP-R1 เสร็จ Neighbor กับ DIST-SW3
หลุดทันที และไม่กลับมาขึ้นอีกเลยแม้ Interface จะ up/up

### ชั้น 1 — ตรวจ Neighbor และ Interface

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        13  00:40:11    2    100  0  8
0   10.199.34.14    Gi1/0/24        14  00:40:11    1    100  0  4
```
*(ไม่มี Neighbor ทาง Gi1/0/23 เหมือนเดิม — แต่ก่อนอื่นต้องแยกให้ออกจาก Step 552: ครั้งนี้ไม่มี Log
`K-value mismatch` เลย)*

```
DIST-SW3# show ip interface brief | include Gi1/0/23
GigabitEthernet1/0/23     10.199.34.1     YES manual up                    up
```
Interface ยัง `up/up` ปกติ — ผ่านชั้น Layer 1/2 แล้ว ต้องดูต่อที่ Subnet

### ดู Running-config ของทั้งสองฝั่ง (จุดที่ Bug ซ่อนอยู่)

```
DIST-SW3# show running-config interface GigabitEthernet1/0/23
interface GigabitEthernet1/0/23
 description ** DEMO-EIGRP: Link1 to EIGRP-R1 (legacy site) **
 no switchport
 ip address 10.199.34.1 255.255.255.252    ! DIST-SW3 ยังอยู่ Subnet เดิม ไม่ได้เปลี่ยนอะไร
 bandwidth 100000
 delay 10
```

```
EIGRP-R1# show running-config interface GigabitEthernet0/0
interface GigabitEthernet0/0
 description ** Link1 to DIST-SW3 **
 ip address 10.199.35.2 255.255.255.252            ! <<< Primary กลายเป็น Subnet ใหม่แล้ว
 ip address 10.199.34.2 255.255.255.252 secondary  ! <<< Subnet เดิม (ที่ DIST-SW3 ใช้จริง) ถูกลดเป็น secondary
 bandwidth 100000
 delay 10
```

**พบต้นเหตุ**: วิศวกรพิมพ์ลำดับคำสั่งผิด — ตั้งใจจะ "เพิ่ม Secondary ก่อน แล้วค่อยสลับ Primary
ทีหลังตอนพร้อม Cutover จริง" แต่ดันพิมพ์ **IP ใหม่เป็นตัวแรก (กลายเป็น Primary ทันที)** แล้วค่อยพิมพ์
IP เดิมพร้อม Keyword `secondary` ตามหลัง — ผลคือ Primary Address ของ EIGRP-R1 กลายเป็น
`10.199.35.2/30` ซึ่ง**ไม่ตรง Subnet กับ Primary Address ของ DIST-SW3 เลย** (`10.199.34.1/30`)
แม้ Secondary Address (`10.199.34.2/30`) จะอยู่ Subnet เดียวกับ DIST-SW3 พอดีก็ตาม

> **กฎสำคัญที่ข้อสอบ ENARSI ชอบถาม**: **EIGRP (และ Routing Protocol อื่นๆแทบทั้งหมด) ใช้ Primary
> Address เป็น Source ของ Hello Packet เสมอ ไม่สนใจ Secondary Address ในการสร้าง Neighbor
> เด็ดขาด** — แม้ Secondary Address จะอยู่ Subnet ที่ถูกต้องพอดีกับอีกฝั่งก็ไม่มีผลอะไรกับการตั้ง
> Adjacency เลย (ทวนหลักการจาก [Part 13 Step 125](part-013-eigrp-fundamentals.md) ข้อ 3:
> "Secondary Address ไม่ถูกใช้ตั้ง Adjacency")

### ยืนยันด้วย `debug` (ชั่วคราว)

```
DIST-SW3# debug eigrp packets hello
EIGRP: Received HELLO on GigabitEthernet1/0/23 nbr 10.199.35.2
EIGRP: Discarding: not in Common subnet
DIST-SW3# undebug all
```

Hello มาจาก `10.199.35.2` (Primary ใหม่ของ EIGRP-R1) ซึ่งไม่อยู่ใน Subnet `10.199.34.0/30` ของ
DIST-SW3 เลย — ถูก Discard ทันที เหมือน Pattern ที่ Part 13 Step 125 อธิบายไว้ (คนละสาเหตุ แต่ Log
เดียวกัน: "not in Common subnet")

### แก้ไข — ต้องเข้าใจลำดับการลบ Secondary Address ก่อน

**ข้อควรระวังสำคัญของ IOS**: ห้าม `no ip address <primary>` ตรงๆทันทีในขณะที่ยังมี Secondary
Address ผูกอยู่ (IOS จะ Error หรือลบ Secondary ทิ้งไปด้วยแบบไม่ได้ตั้งใจ) ต้อง**ลบ Secondary ก่อน
เสมอ** แล้วค่อยตั้ง Primary ใหม่ให้ถูก:

```
EIGRP-R1(config)# interface GigabitEthernet0/0
EIGRP-R1(config-if)# no ip address 10.199.34.2 255.255.255.252 secondary
EIGRP-R1(config-if)# no ip address 10.199.35.2 255.255.255.252
EIGRP-R1(config-if)# ip address 10.199.34.2 255.255.255.252
EIGRP-R1(config-if)# end
```

*(เนื่องจากโครงการ Renumbering ยังไม่พร้อม Cutover ทั้งระบบ วิธีแก้ที่ถูกต้องตอนนี้คือ**คืนกลับไปที่
Subnet เดิมก่อน** แล้วค่อยวางแผน Cutover ทั้งสองฝั่งพร้อมกันใหม่ในช่วง Maintenance Window — ไม่ใช่
แก้โดยไปเปลี่ยน Primary ของ DIST-SW3 ตามให้ตรงกับ EIGRP-R1 เพราะ DIST-SW3 อาจมี Neighbor อื่น
(เช่น DIST-SW4 ทาง Link4 ที่อยู่ Subnet Range เดียวกัน `10.199.34.0/24`) ที่ต้อง Migrate พร้อมกันเป็น
ชุดเดียว ไม่ใช่แก้ทีละ Link*)

### Verify หลังแก้

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
2   10.199.34.2     Gi1/0/23        14  00:00:09    2    100  0  2
1   10.199.34.18    Gi1/0/25        13  00:52:03    2    100  0  8
0   10.199.34.14    Gi1/0/24        14  00:52:03    1    100  0  4
```

**บทเรียน ENARSI**: เมื่อเจอ Neighbor ไม่ขึ้นแบบไม่มี Log ชัดเจน (ต่างจาก K-value/Authentication ที่
มี Log บอกตรงๆ) ให้สงสัย **Primary/Secondary Address ผิดที่** เป็นอันดับแรกๆ — คำสั่งที่ช่วยตรวจ
ได้เร็วสุดคือ `show running-config interface` เทียบทั้งสองฝั่ง (ดูว่า Primary Address จริงๆอยู่
Subnet เดียวกันหรือไม่ ไม่ใช่แค่ดูว่ามี IP ไหนอยู่ Subnet ที่ถูกในนั้นบ้าง)

---

## Step 554 — Scenario 3: Route หายจาก `passive-interface` ที่ใส่ผิดบน Transit Link

### อาการที่รายงานมา

ทีม Security ขอให้ Harden EIGRP Config ตามมาตรฐาน "Deny by Default, Allow by Exception" ที่หลักสูตร
สอนไว้ตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md) (สำหรับ OSPF) — วิศวกรเห็นว่าหลักการเดียวกัน
น่าจะใช้กับ EIGRP ได้ จึงไปเพิ่ม `passive-interface default` บน DIST-SW4 โดยตั้งใจจะเปิดเฉพาะ Link
ที่จำเป็นด้วย `no passive-interface` แต่ลืมเปิดกลับให้ครบ — ผลคือ DIST-SW4 หายไปจาก EIGRP Domain
ทั้งตัว

### ชั้น 1 — ตรวจ Neighbor ที่ DIST-SW3 (มุมมองจากอีกฝั่ง)

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
1   10.199.34.18    Gi1/0/25        13  01:10:22    2    100  0  8
0   10.199.34.2     Gi1/0/23        14  01:10:22    1    100  0  2
```
*(Neighbor ทาง Gi1/0/24 ไปหา DIST-SW4 หายไป — สังเกตว่ารูปแบบคล้าย Step 552/553 แต่คราวนี้ไม่มี
Log `%DUAL-5-NBRCHANGE ... down` เลยแม้แต่บรรทัดเดียว เพราะ Neighbor **ไม่เคยขึ้นตั้งแต่ต้น**
หลัง Reload/Apply Config ใหม่ — สัญญาณสำคัญที่บอกว่าไม่ใช่ K-value/Authentication ที่ Flap แล้ว
ค้าง down แต่เป็นสาเหตุที่ทำให้ Hello ไม่ถูกส่งออกไปเลยตั้งแต่แรก)*

### ชั้น 1 (ต่อ) — เช็คที่ DIST-SW4 ด้วย `show ip eigrp interfaces` (คำสั่งเฉพาะที่ต้องใช้กรณีนี้)

```
DIST-SW4# show ip eigrp interfaces
EIGRP-IPv4 Interfaces for AS(100)
                        Xmit Queue   PeerQ        Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  Un/Reliable   SRTT   Un/Reliable   Flow Timer   Routes
```

**นี่คือสัญญาณสำคัญที่สุดของปัญหานี้**: List **ว่างเปล่าโดยสิ้นเชิง** — ไม่มี Interface ใดปรากฏใน
`show ip eigrp interfaces` เลยแม้แต่ตัวเดียว ทั้งที่ DIST-SW4 มี 2 Interface ที่ควรอยู่ใน EIGRP
(Link3 ไปหา EIGRP-R2, Link4 ไปหา DIST-SW3) — ทวนจาก [Part 13 Step 125](part-013-eigrp-fundamentals.md)
ข้อ 5: **Interface ที่ไม่ปรากฏใน `show ip eigrp interfaces` เลย = Passive Interface อยู่แน่นอน**
(ต่าง จาก Interface ที่ไม่ตรง `network` statement ซึ่งไม่ปรากฏเหมือนกันแต่ด้วยเหตุผลคนละอย่าง —
ต้องเช็ค `network` statement คู่กันเสมอเพื่อแยกสองกรณีนี้)

### ยืนยันด้วย Running-config

```
DIST-SW4# show running-config | section router eigrp
router eigrp EIGRP-DEMO
 !
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.14
  passive-interface default             ! <<< ต้นเหตุ — Harden ตามแนวคิด OSPF ของ Part 11
  network 10.199.34.0 0.0.0.255
  exit-address-family
```

พบ `passive-interface default` แต่**ไม่มีบรรทัด `no passive-interface` ตามมาแม้แต่บรรทัดเดียว** —
ต่างจาก OSPF ที่ DIST-SW ทุกตัวใน Lab หลักมักมี SVI จำนวนมากที่ควร Passive จริง (เช่น Vlan10/20/30)
และมี Transit Link ไม่กี่เส้นที่ต้อง `no passive-interface` แต่ในกรณีของ EIGRP Demo Site นี้
**ทุก Interface ที่เปิด EIGRP ล้วนเป็น Transit Link ทั้งหมด ไม่มี Interface ใดที่ควร Passive เลย** —
การ Copy แนวคิดจาก OSPF มาใช้ตรงๆโดยไม่พิจารณาว่า Topology ต่างกันคือรากของปัญหานี้

> **ข้อสังเกตสำคัญ (ตอบคำถาม ENARSI ที่ชอบถามเปรียบเทียบ)**: `passive-interface` มีผลเหมือนกัน
> ทั้ง OSPF และ EIGRP ในหลักการ (หยุดส่ง Hello แต่ยัง Advertise Subnet ของ Interface นั้นต่อไป
> ตามปกติ — ทวนจาก Part 11 Step 107.2) **แต่ผลลัพธ์ต่างกันโดยสิ้นเชิงเมื่อใช้ผิดที่บน Transit Link**:
> OSPF ที่ไม่มี Neighbor บน Interface Passive ยังคง Advertise Subnet ของ Interface นั้นเป็น Stub
> Network ใน LSA Type 1 ได้ปกติ (เพราะ OSPF รู้ Topology ผ่าน LSDB ทั้งก้อน) แต่ **EIGRP ถ้าไม่มี
> Neighbor ขึ้นบน Transit Link เลย จะไม่มีทาง Advertise Prefix ที่อยู่ "อีกฝั่ง" ของ Link นั้นได้เลย**
> เพราะ EIGRP ไม่มี Topology Database กลางแบบ OSPF — ผลคือ DIST-SW4 กลายเป็น "เกาะ" ที่ตัดขาดจาก
> EIGRP Domain ทั้งหมดทันที ไม่ใช่แค่เสีย Route เดียวแบบ OSPF

### แก้ไข

```
DIST-SW4(config)# router eigrp EIGRP-DEMO
DIST-SW4(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW4(config-router-af)# no passive-interface GigabitEthernet1/0/23   ! Link4 ไปหา DIST-SW3
DIST-SW4(config-router-af)# no passive-interface GigabitEthernet1/0/24   ! Link3 ไปหา EIGRP-R2
DIST-SW4(config-router-af)# end
```

### Verify หลังแก้

```
DIST-SW4# show ip eigrp interfaces
EIGRP-IPv4 Interfaces for AS(100)
                        Xmit Queue   PeerQ        Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  Un/Reliable   SRTT   Un/Reliable   Flow Timer   Routes
Gi1/0/23              1        0/0     0/0            1       0/1          50           0
Gi1/0/24              1        0/0     0/0            2       0/1          50           0

DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
2   10.199.34.14    Gi1/0/24        14  00:00:14    2    100  0  2
1   10.199.34.18    Gi1/0/25        13  01:15:09    2    100  0  8
0   10.199.34.2     Gi1/0/23        14  01:15:09    1    100  0  2
```

**บทเรียน ENARSI**: `show ip eigrp interfaces` (ไม่ใช่แค่ `show ip eigrp neighbors`) คือคำสั่งเดียว
ที่บอก Passive Interface ได้ตรงที่สุด — ถ้าเจอ Neighbor ไม่ขึ้นแบบไม่มี Log ใดๆเลย (ต่างจาก
K-value/Authentication ที่มี Log ชัดเจน) ให้เช็คคำสั่งนี้เป็นอันดับแรกก่อนสงสัยเรื่องอื่น

---

## Step 555 — Scenario 4: SIA จาก Link Flapping ร่วมกับ Summarization ที่ไม่ครบ

### อาการที่รายงานมา

Link3 (DIST-SW4 ↔ EIGRP-R2, จำลอง Serial WAN ความเร็ว 1.544 Mbps) มีปัญหา Physical Layer เป็น
ระยะ (สายเก่า, Connector หลวม) ทำให้ Interface Flap ขึ้นๆลงๆหลายรอบต่อชั่วโมง — ตั้งแต่เริ่มมีปัญหา
นี้ ผู้ใช้ที่ Legacy Server LAN รายงานว่า "บางครั้งเข้าเว็บไม่ได้อยู่นานหลายนาทีทั้งที่ Link น่าจะกลับ
มาแล้ว" และ Log เต็มไปด้วย SIA

### ชั้น 1-2 — ตรวจ Neighbor และ Log

```
DIST-SW4#
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/24, changed state to down
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/24, changed state to up
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.9 (GigabitEthernet1/0/24) is down: interface down
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.9 (GigabitEthernet1/0/24) is up: new adjacency
...
%DUAL-3-SIA: Route 10.199.200.0/24 stuck-in-active state in IPv4-EIGRP(100) 100. Cleaning up
```

Neighbor ขึ้นๆลงๆตาม Physical Flap ปกติ — แต่มี Log `%DUAL-3-SIA` ปรากฏด้วย ตรงกับ Decision Tree
ชั้น 2 (Topology Table ผิดปกติ) — ทวนกลไกจาก [Part 28 Step 275](part-028-eigrp-advanced.md):
SIA เกิดเมื่อ Route เข้าสถานะ Active (Successor ล่มไม่มี FS สำรอง) และ Neighbor อย่างน้อยหนึ่งตัว
ไม่ตอบ Reply ภายใน Active Timer (Default 3 นาที)

### วินิจฉัยด้วย `show ip eigrp topology active` ระหว่างเกิดเหตุ

```
DIST-SW3# show ip eigrp topology active
EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.13)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status

A 10.199.200.0/24, 1 successors, FD is 156160, Q
        1 replies, active 00:01:52, query-origin: Local origin
        via 10.199.34.14 (2298112/2297856), r, GigabitEthernet1/0/24
             Waiting for query replies from
```

พบว่า DIST-SW3 กำลังรอ Reply จาก DIST-SW4 (ทาง Gi1/0/24) ค้างอยู่ — เหตุผลคือทุกครั้งที่ Link3 Flap
Route `10.199.200.0/24` ที่ DIST-SW4 เรียนรู้ (ผ่าน Link3 ไปหา EIGRP-R2) เข้าสถานะ Active ที่ตัว
DIST-SW4 เอง แล้ว **DIST-SW4 ต้อง Query ต่อไปยัง DIST-SW3 ด้วย** เพราะไม่มี FS สำรองสำหรับ Prefix
เจาะจงนี้ (Query "ลาม" ต่อเป็นทอด — ตรงกับสาเหตุ SIA ข้อ "Query Chaining" ใน Part 28 Step 275)

### ทำไมถึงลามไปถึง DIST-SW3 — ตรวจ Summarization ปัจจุบัน

```
DIST-SW4# show running-config interface GigabitEthernet1/0/23
interface GigabitEthernet1/0/23
 description ** DEMO-EIGRP: Link4 to DIST-SW3 **
 no switchport
 ip address 10.199.34.14 255.255.255.252
! ไม่มีบรรทัด ip summary-address eigrp เลย
```

**พบต้นเหตุเชิงโครงสร้าง**: [Part 13 Step 130](part-013-eigrp-fundamentals.md) ตั้ง Summarization
ไว้ที่ DIST-SW3 (Link1 ไปหา EIGRP-R1) เท่านั้น — DIST-SW4 **ไม่เคย** ทำ Summary Route ขาออกทาง
Link4 (ไปหา DIST-SW3) เลย ดังนั้นทุกครั้งที่ Prefix เจาะจงอย่าง `10.199.200.0/24` เปลี่ยนแปลง
สถานะที่ Link3 (Flap) การเปลี่ยนแปลงนั้น**ไม่ถูกกันไว้ที่ Query Boundary ใดๆ** ก่อนจะไปถึง DIST-SW3
เลย — ตรงกับหลักการจาก [Part 28 Step 275](part-028-eigrp-advanced.md): "Summarization สร้าง Query
Boundary" ที่ยังขาดอยู่ตรงจุดนี้

### แก้ไข — เพิ่ม Summarization ที่ DIST-SW4 ปิด Query Boundary ให้ครบ

```
DIST-SW4(config)# interface GigabitEthernet1/0/23
DIST-SW4(config-if)# ip summary-address eigrp 100 10.199.34.0 255.255.255.0
DIST-SW4(config-if)# end
```

และเสริมมาตรการหลักตาม Part 28 Step 275 อีกชั้น — ยืนยันว่า EIGRP-R2 (ปลาย Link3 ที่ Flap)
ยังเป็น Stub ตามที่ตั้งใจไว้ตั้งแต่ Part 28 Step 272 (ไม่ได้หายไป แต่ตรวจซ้ำเพื่อความชัวร์):

```
EIGRP-R2# show running-config | section router eigrp
router eigrp EIGRP-DEMO
 !
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 9.9.9.2
  eigrp stub connected summary          ! ยังอยู่ครบตามที่ควร
```

### Verify หลังแก้ — ทดสอบ Flap ซ้ำ

```
DIST-SW4#
%LINK-3-UPDOWN: Interface GigabitEthernet1/0/24, changed state to down
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.9 (GigabitEthernet1/0/24) is down: interface down
! ไม่มี %DUAL-3-SIA ปรากฏอีกเลย

DIST-SW3# show ip eigrp topology active
! ไม่มี Output ใดๆ — ไม่มี Route ค้าง Active
```

Query จาก DIST-SW4 ตอนนี้ถูกสกัดไว้ที่ Query Boundary ใหม่ (Summary Route ทาง Link4) — DIST-SW3
เห็นแค่ Summary `10.199.34.0/24` ที่ยังเสถียรอยู่ (ตราบใดที่ยังมี Component Subnet เหลืออย่างน้อย
1 เส้น) จึงไม่ต้องถูก Query เลยไม่ว่า Link3 จะ Flap กี่รอบก็ตาม

**บทเรียน ENARSI**: SIA ที่เกิดจาก Link Flapping ไม่ได้แก้ด้วยการซ่อม Physical Layer อย่างเดียว
(ซึ่งควรทำคู่กันแน่นอน) แต่ **ต้นเหตุเชิง Design ที่ทำให้ SIA ลามไปถึง Router อื่นคือ Summarization/
Stub Boundary ที่ไม่ครบ** — ข้อสอบชอบให้โจทย์ที่ดู "เหมือนปัญหา Hardware" แต่คำตอบที่ถูกต้องคือ
แก้ที่ Config ระดับ Summarization ไม่ใช่แค่บอกให้ไปเปลี่ยนสาย

---

## Step 556 — Scenario 5: Asymmetric/Suboptimal Routing จาก `bandwidth` ที่ไม่ตรงกับ Physical จริง

### อาการที่รายงานมา

ทีม WAN แจ้งว่า **ISP อัปเกรด Circuit ของ Link3 จาก 1.544 Mbps (T1) เป็น 45 Mbps (DS3) เมื่อเดือน
ที่แล้ว** และเปลี่ยน Physical Port จาก Serial เป็น Ethernet Hand-off (ยังคงจำลองด้วย GigE Interface
เหมือนเดิมตาม Part 13 Step 122 ที่อธิบายไว้ว่า Lab นี้ใช้ `bandwidth`/`delay` จำลอง WAN บน Physical
GigE) — แต่ผู้ใช้ที่ Legacy Server LAN ยังรายงานว่า Traffic ช้าและ **Path ที่ Traceroute ได้ไม่ตรง
กับที่คาด** (คาดว่าน่าจะเร็วขึ้นแต่กลับไม่เห็นความเปลี่ยนแปลง)

### ชั้น 1-3 — Neighbor และ RIB ปกติดีทั้งหมด

```
DIST-SW3# show ip eigrp neighbors
! ครบทั้ง 3 Neighbor ปกติ (ไม่แสดงซ้ำ)

DIST-SW3# show ip route eigrp
D        10.199.200.0/24 [90/156160] via 10.199.34.18, 00:40:11, GigabitEthernet1/0/25
```

Neighbor ขึ้นครบ, Route อยู่ใน RIB ปกติ — ตรง Decision Tree ชั้น 1-3 ผ่านหมด ต้องไปดูต่อที่ **ชั้น 4
(Metric/Path ที่เลือกสมเหตุสมผลหรือไม่)**

### ชั้น 4 — ตรวจ Metric อย่างละเอียดด้วย `show ip eigrp topology all-links`

```
DIST-SW3# show ip eigrp topology 10.199.200.0/24 all-links
P 10.199.200.0/24, 1 successors, FD is 156160, serno 12
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
        via 10.199.34.14 (2298112/2297856), GigabitEthernet1/0/24     <- Path ผ่าน Link3 ยังแพงมาก!
```

Path ที่ผ่าน DIST-SW4/Link3 (ตัวที่ ISP บอกว่าอัปเกรดเป็น 45 Mbps แล้ว) ยัง **แพงเท่าเดิมกับก่อน
อัปเกรด** (metric 2,298,112 — ตัวเลขเดียวกับที่คำนวณไว้ใน Part 13 Step 123 ตอน Link3 ยังเป็น
1.544 Mbps) — ทั้งที่ Physical Bandwidth เพิ่มขึ้นเกือบ 30 เท่าแล้ว

### ตรวจ `bandwidth`/`delay` ที่ Config จริงเทียบกับ Physical Circuit

```
DIST-SW4# show interfaces GigabitEthernet1/0/24 | include BW|DLY
  MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,
```

**พบต้นเหตุ**: `BW 1544 Kbit/sec` — ค่านี้คือค่าที่**ตั้งด้วยคำสั่ง `bandwidth`** (ทวนจาก
[Part 13 Step 122](part-013-eigrp-fundamentals.md): EIGRP ใช้ค่าที่ Config ไว้ ไม่ใช่ Physical
Speed จริง) ทีม WAN อัปเกรด Physical Circuit จริงแล้ว **แต่ไม่มีใครไปอัปเดตคำสั่ง `bandwidth`/
`delay` บน Interface ตามให้ตรงกับความเร็วใหม่** — Interface ยังคง "จำลอง" ตัวเองว่าเป็น Serial
1.544 Mbps ต่อไปทั้งที่จริงเป็น 45 Mbps ไปแล้ว

```
DIST-SW4# show running-config interface GigabitEthernet1/0/24
interface GigabitEthernet1/0/24
 description ** DEMO-EIGRP: Link3 to EIGRP-R2 (legacy WAN sim) **
 no switchport
 ip address 10.199.34.10 255.255.255.252
 bandwidth 1544        ! <<< ค่าเก่าที่ไม่ได้อัปเดตตามการอัปเกรด Circuit จริง
 delay 20000            ! <<< ค่าเก่าเช่นกัน
```

> **ผลกระทบที่เกิดขึ้นจริง**: เพราะ Metric ของ Path ผ่าน Link3 ยังสูงมากจนไม่ผ่าน Feasibility
> Condition (RD = 2,297,856 ≥ FD = 156,160 เหมือนที่คำนวณไว้ตั้งแต่ Part 13 Step 123) Path นี้จึง
> **ไม่มีทางถูกใช้เลยไม่ว่าจะตั้ง `variance` สูงแค่ไหน** — Bandwidth จริง 45 Mbps ที่จ่ายเงินซื้อมา
> ถูกปล่อยว่างเปล่าไม่ได้ใช้ประโยชน์อะไรเลย ในขณะที่ Path หลัก (ผ่าน Link5/EIGRP-R2 ตรง หรือ Link1/
> EIGRP-R1) ต้องรับ Traffic ทั้งหมดไปเต็มๆ — นี่คือ **Suboptimal Routing แบบเงียบ** ที่ไม่มี Log
> เตือนใดๆเลย เพราะในมุมของ EIGRP มันไม่ใช่ "ปัญหา" (DUAL ทำงานถูกต้องตาม Metric ที่มันเห็น)
> เป็นปัญหาจาก **ข้อมูล Input ที่ผิด** ไม่ใช่ Algorithm ผิด

### แก้ไข — อัปเดต `bandwidth`/`delay` ให้ตรงกับ Circuit ใหม่

DS3 = 44,736 Kbps (มาตรฐานความเร็ว DS3 จริง) และ Delay ของ Fiber/DS3 ทั่วไปประมาณ 200 usec:

```
DIST-SW4(config)# interface GigabitEthernet1/0/24
DIST-SW4(config-if)# bandwidth 44736
DIST-SW4(config-if)# delay 20
DIST-SW4(config-if)# end
```
*(ต้องแก้ทั้งสองฝั่ง — EIGRP-R2 ด้านตรงข้าม Link3 ก็ต้องอัปเดตค่าเดียวกันนี้ด้วย ไม่เช่นนั้น Metric
จะคำนวณไม่ตรงกันระหว่างสองฝั่ง แม้จะดีขึ้นกว่าเดิมก็ตาม)*

```
EIGRP-R2(config)# interface GigabitEthernet0/1
EIGRP-R2(config-if)# bandwidth 44736
EIGRP-R2(config-if)# delay 20
EIGRP-R2(config-if)# end
```

### Verify หลังแก้ — คำนวณ Metric ใหม่

```
DIST-SW3# show ip eigrp topology 10.199.200.0/24 all-links
P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
        via 10.199.34.14 (259840/259584), GigabitEthernet1/0/24     <- ลดลงมหาศาลจาก 2,298,112!
```

Metric ของ Path ผ่าน Link3 ลดจาก 2,298,112 เหลือ 259,840 — ยังไม่ผ่าน FC เทียบกับ FD ปัจจุบัน
(156,160) จึงยังไม่กลายเป็น FS โดยอัตโนมัติ แต่ตอนนี้ **ตัวเลขสมเหตุสมผลกับความเร็วจริงแล้ว** —
ถ้าต้องการดึง Path นี้มาช่วยแบ่ง Load จริง (เพราะ Bandwidth มากถึง 45 Mbps) ทีมสามารถพิจารณาปรับ
`variance` เพิ่มเป็น Step ถัดไปได้อย่างมีเหตุผลรองรับ (ต่างจากก่อนแก้ที่ต่อให้ตั้ง variance สูงแค่ไหน
ก็ไม่มีทางใช้ Path นี้ได้เลย)

**บทเรียน ENARSI**: `bandwidth`/`delay` ไม่ใช่ค่าที่ "ตั้งครั้งเดียวแล้วลืมได้" — ทุกครั้งที่มีการ
เปลี่ยน Physical Circuit จริง (อัปเกรด/ลดระดับ/เปลี่ยนผู้ให้บริการ) ต้องตรวจสอบและอัปเดตค่านี้เสมอ
คู่กัน มิฉะนั้น Routing Protocol ที่พึ่งค่านี้ (ทั้ง EIGRP และ OSPF Cost) จะยังคงตัดสินใจ Path ผิด
ต่อไปเรื่อยๆ แม้ Infrastructure จริงจะเปลี่ยนไปแล้วก็ตาม — ข้อสอบชอบใช้คำว่า "อัปเกรด Link แล้วแต่ยัง
ช้า" เป็นสัญญาณของ Scenario แบบนี้เสมอ

---

## Step 557 — Scenario 6: Route อยู่ใน Topology Table แต่ไม่อยู่ใน Routing Table

### อาการที่รายงานมา

ผู้ใช้ Legacy Server LAN (`10.199.200.0/24`) แจ้งว่าเข้าใช้งานไม่ได้เลยจาก Core แม้ EIGRP Neighbor
ทุกตัวขึ้นปกติ — ทีมงานเคยมี Test Lab ชั่วคราวที่ใช้ Subnet Range ใกล้เคียงกันเมื่อหลายเดือนก่อนและ
ตั้ง Filter ไว้กันไม่ให้ Subnet ทดสอบนั้นหลุดเข้า Production แต่ไม่มีใครถอด Filter ออกหลังเลิกใช้
Test Lab

### ชั้น 1-2 — Neighbor และ Topology Table ปกติดี

```
DIST-SW3# show ip eigrp neighbors
! ครบทั้ง 3 Neighbor ปกติ

DIST-SW3# show ip eigrp topology 10.199.200.0/24
P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
```

**Topology Table มี Successor และ Feasible Successor ครบถูกต้องทุกประการ** ตรงกับที่คำนวณไว้ใน
Part 13 Step 123 เป๊ะ — ตาม Decision Tree ต้องไปเช็คชั้น 3 ต่อ (Route ถูก Install ลง RIB จริงหรือไม่)

### ชั้น 3 — ตรวจ RIB

```
DIST-SW3# show ip route eigrp
      10.0.0.0/8 is variably subnetted, ...
D        10.199.34.12/30 [90/158720] via 10.199.34.14, 00:02:11, GigabitEthernet1/0/24
D        10.199.34.16/30 [90/128512] via 10.199.34.18, 00:02:11, GigabitEthernet1/0/25
! ไม่มี 10.199.200.0/24 ปรากฏเลย
```

**พบความผิดปกติตรงตาม Decision Tree ชั้น 3**: Topology Table มี Route (พร้อม Successor ชัดเจน)
แต่ **RIB ไม่มี Route นี้เลย** — นี่คือลักษณะเฉพาะของปัญหาการ Filter Route แบบ EIGRP ที่
[Part 28 Step 273](part-028-eigrp-advanced.md) เตือนไว้: `distribute-list` ของ EIGRP กรองที่
**Network ปลายทาง** และมีผลต่อ**การ Install ลง RIB** — Router ยังคงรับ Update ผ่าน RTP และเก็บลง
Topology Table ตามปกติ (เพราะ DUAL ยังต้องใช้ข้อมูลนี้คำนวณ Feasibility Condition ให้ Neighbor อื่น
ต่อได้) **แต่ปฏิเสธไม่ Install ลง Routing Table** ถ้ามี Distribute-list ขวางอยู่ที่ทิศทาง `in`
ระดับ Process (ไม่ระบุ Interface) — นี่คือจุดที่ต่างจาก OSPF ที่กรองด้วย `distribute-list` แบบเดียวกัน
(กรอง RIB เท่านั้น เหมือนกัน) แต่ผู้เรียนมักสับสนคิดว่า EIGRP ก็ต้องกรองออกจาก Topology Table ไปด้วย

### หาต้นเหตุใน `show ip protocols` และ Distribute-list

```
DIST-SW3# show ip protocols | include distribute
  Incoming update filter list for all interfaces is OLD-TESTLAB-BLOCK
```

```
DIST-SW3# show ip prefix-list OLD-TESTLAB-BLOCK
ip prefix-list OLD-TESTLAB-BLOCK: 2 entries
   seq 5 deny 10.199.192.0/18 le 32
   seq 10 permit 0.0.0.0/0 le 32
```

**พบต้นเหตุ**: Prefix-list `OLD-TESTLAB-BLOCK` ถูกเขียนไว้เมื่อหลายเดือนก่อนเพื่อกัน Subnet ของ
Test Lab ชั่วคราว (`10.199.192.0/18` — ครอบคลุมตั้งแต่ `10.199.192.0` ถึง `10.199.255.255`)
**แต่ผู้เขียนตอนนั้นเขียน Prefix Range กว้างเกินความจำเป็นไปมาก** เพื่อความสะดวก (คิดว่า "กันหมด
ทีเดียวเผื่อไว้") — Range `/18` นี้ **ครอบคลุม `10.199.200.0/24` (Legacy Server LAN จริง) เข้าไป
ด้วยโดยไม่ตั้งใจ** และไม่มีใครถอด Distribute-list นี้ออกหลังเลิกใช้ Test Lab นั้นแล้ว

```
DIST-SW3# show run | include distribute-list
 distribute-list prefix OLD-TESTLAB-BLOCK in
```

*(สังเกตว่าไม่ระบุ Interface ต่อท้าย — เท่ากับมีผลกับ Update ที่เข้ามาจาก**ทุก**Interface/Neighbor
ของ Process นี้)*

### แก้ไข

เนื่องจาก Test Lab เลิกใช้ไปแล้วจริง วิธีแก้ที่ตรงที่สุดคือถอด Distribute-list ทิ้งทั้งหมด (ไม่ใช่
แค่แก้ Prefix-list ให้แคบลง เพราะไม่มีความจำเป็นต้องกัน Subnet ใดๆแล้ว):

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# no distribute-list prefix OLD-TESTLAB-BLOCK in
DIST-SW3(config-router-af-topology)# end
DIST-SW3(config)# no ip prefix-list OLD-TESTLAB-BLOCK
```

### Verify หลังแก้

```
DIST-SW3# show ip route eigrp
D        10.199.200.0/24 [90/156160] via 10.199.34.18, 00:00:04, GigabitEthernet1/0/25   <- กลับมาแล้ว
D        10.199.34.12/30 [90/158720] via 10.199.34.14, 00:15:22, GigabitEthernet1/0/24
D        10.199.34.16/30 [90/128512] via 10.199.34.18, 00:15:22, GigabitEthernet1/0/25
```

**บทเรียน ENARSI**: เมื่อเจอ Route ที่**อยู่ใน Topology Table แต่หายจาก RIB** ให้ตัดปัญหาเรื่อง
Neighbor/DUAL ออกไปได้ทันที (เพราะพิสูจน์แล้วว่า DUAL คำนวณถูก) แล้วมุ่งไปที่ **Filtering
เท่านั้น**: `distribute-list` (Step นี้), Administrative Distance ที่ถูก Static Route แย่งไป
(ตรวจด้วย `show ip route <prefix>` ดู AD ของ Route ที่แข่งกัน), หรือ `maximum-paths`/`variance`
ที่ตั้งผิดจนดูเหมือน Route หาย ทั้งหมดนี้คือปัญหาระดับ **RIB Installation** ไม่ใช่ปัญหาระดับ DUAL/
Topology อีกต่อไป — และ Distribute-list เก่าที่ "ลืมถอด" หลังเลิกใช้งานชั่วคราวคือสาเหตุที่พบบ่อย
ที่สุดในข้อสอบจริงเพราะมันไม่ทิ้ง Log อะไรเลย เงียบสนิทจนกว่าจะไปเทียบ Topology Table กับ RIB
ด้วยตัวเอง

---

## Step 558 — Scenario 7: Authentication Mismatch หลัง Key Rotation

### อาการที่รายงานมา

ทีม Security วางแผน Key Rotation ของ EIGRP MD5 Authentication ตามกำหนดการที่ตั้งไว้ตั้งแต่
[Part 28 Step 277](part-028-eigrp-advanced.md) — คือให้ Key 2 (`N3wK3y#2026Q4`) เริ่มมีผลตั้งแต่
**1 ตุลาคม 2026** (Overlap กับ Key 1 ไว้ 24 ชั่วโมงตามแผน) วันนี้คือ **26 กันยายน 2026** ทีมงาน
กำลัง Pre-stage Key 2 ไว้ในทุกอุปกรณ์ล่วงหน้าตามกำหนดการ แต่ระหว่าง Deploy Key-chain ไปที่
EIGRP-R2 มีคนพิมพ์วันที่ผิดพลาด — ทีมอยากให้ทดสอบตรวจสอบก่อนถึงวัน Cutover จริงว่า Config ทุกตัว
พร้อมหรือยัง

### ตรวจ Key-chain ทุกตัวก่อนวัน Cutover (Proactive Check — ไม่ต้องรอให้ Neighbor หลุดก่อน)

```
DIST-SW3# show key chain
Key-chain EIGRP-KEY:
    key 1 -- text "C1sco123!EIGRP"
        accept lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 2 2026), valid now
        send lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 1 2026), valid now
    key 2 -- text "N3wK3y#2026Q4"
        accept lifetime (00:00:00 UTC Oct 1 2026) - (infinite), valid now       <- valid now?! ยังไม่ถึง 1 ต.ค.
        send lifetime (00:00:00 UTC Oct 1 2026) - (infinite), valid now
```

*(หมายเหตุ: `valid now` ของ Key 2 ในผลลัพธ์ตัวอย่างนี้อาจดูขัดกับวันที่ปัจจุบันเพราะ IOS แสดงสถานะ
ตาม Boundary ที่ตั้งไว้เทียบกับนาฬิกาปัจจุบันของอุปกรณ์นั้นๆ — สิ่งที่ต้องเทียบจริงๆคือ Boundary ของ
DIST-SW3 กับ EIGRP-R2 ต้อง**ตรงกันเป๊ะ**ไม่ใช่แค่ดูคำว่า valid now เฉยๆ)*

```
EIGRP-R2# show key chain
Key-chain EIGRP-KEY:
    key 1 -- text "C1sco123!EIGRP"
        accept lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 2 2026), valid now
        send lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 1 2026), valid now
    key 2 -- text "N3wK3y#2026Q4"
        accept lifetime (00:00:00 UTC Oct 1 2027) - (infinite), valid not yet    <- ปีผิด! 2027 ไม่ใช่ 2026
        send lifetime (00:00:00 UTC Oct 1 2027) - (infinite), valid not yet
```

**พบต้นเหตุ**: EIGRP-R2 พิมพ์ปี `Oct 1 **2027**` แทน `Oct 1 2026` ตอน Pre-stage Key 2 — Key-chain
ของ EIGRP-R2 จะยังไม่ยอมรับหรือส่ง Key 2 เลยจนกว่าจะถึงปี 2027 จริง ทั้งที่ทุกอุปกรณ์อื่นในระบบ
(DIST-SW3, DIST-SW4, EIGRP-R1) ตั้งปี 2026 ถูกต้องทั้งหมด — ถ้าไม่มีใครจับได้ก่อนถึงวัน Cutover
(1 ต.ค. 2026) เหตุการณ์ที่จะเกิดคือ: ทุกอุปกรณ์เริ่มส่งด้วย Key 2 ตาม `send-lifetime` ที่ถูกต้อง
พร้อมกัน แต่ **EIGRP-R2 ยังไม่รับ Key 2** (เพราะ `accept-lifetime` ของมันยังไม่ Valid จนถึงปี
2027) ในขณะที่ **Key 1 ก็หมด `send-lifetime` ไปแล้วทุกตัวยกเว้น EIGRP-R2 เอง** — สรุปคือไม่มี Key
คู่ใดที่ทั้งสองฝั่งใช้งานร่วมกันได้พร้อมกันเลยในวันที่ 1 ต.ค. — Neighbor ทุกตัวที่ติดกับ EIGRP-R2
(Link2 ไปหา EIGRP-R1, Link3 ไปหา DIST-SW4, Link5 ไปหา DIST-SW3) จะหลุดพร้อมกันทั้งหมด

### จำลองผลลัพธ์ถ้าไม่จับได้ทันเวลา (Log ที่จะเห็นวันที่ 1 ต.ค.)

```
DIST-SW3#
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.18 (GigabitEthernet1/0/25) is down: Auth failure
```

```
DIST-SW3# debug eigrp packets
EIGRP: Received HELLO on GigabitEthernet1/0/25 nbr 10.199.34.18
EIGRP: Authentication mismatch/failure on GigabitEthernet1/0/25, key 2 not valid
DIST-SW3# undebug all
```

### แก้ไข — แก้ไขก่อนถึงวัน Cutover (Proactive Fix)

```
EIGRP-R2(config)# key chain EIGRP-KEY
EIGRP-R2(config-keychain)# key 2
EIGRP-R2(config-keychain-key)# accept-lifetime 00:00:00 Oct 1 2026 infinite
EIGRP-R2(config-keychain-key)# send-lifetime 00:00:00 Oct 1 2026 infinite
EIGRP-R2(config-keychain-key)# end
```
*(IOS ยอมให้พิมพ์ `accept-lifetime`/`send-lifetime` ซ้ำทับของเดิมได้ตรงๆ ไม่ต้อง `no` ก่อน —
ค่าใหม่จะ Overwrite ค่าเดิมของ Key เดียวกันทันที)*

### Verify หลังแก้

```
EIGRP-R2# show key chain
Key-chain EIGRP-KEY:
    key 1 -- text "C1sco123!EIGRP"
        accept lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 2 2026), valid now
        send lifetime (00:00:00 UTC Jan 1 2026) - (00:00:00 UTC Oct 1 2026), valid now
    key 2 -- text "N3wK3y#2026Q4"
        accept lifetime (00:00:00 UTC Oct 1 2026) - (infinite), valid not yet
        send lifetime (00:00:00 UTC Oct 1 2026) - (infinite), valid not yet
```

ตอนนี้ Boundary ของ EIGRP-R2 ตรงกับทุกอุปกรณ์อื่นแล้ว (`Oct 1 2026` ทั้งหมด) — เมื่อถึงวันที่ 1 ต.ค.
2026 จริง ทุกฝั่งจะสลับไปใช้ Key 2 พร้อมกันตามแผน Overlap เดิมที่ Part 28 Step 277 ออกแบบไว้ โดย
ไม่มี Adjacency หลุดแม้แต่วินาทีเดียว

> **ข้อควรระวังเพิ่มเติมที่ควรเช็คคู่กันเสมอ**: Key Rotation ทั้งระบบพึ่งพา **นาฬิกาของ Router ที่
> ตรงกันผ่าน NTP** (ตามที่ Part 28 Step 277 เตือนไว้) — การตรวจ `show key chain` เพียงอย่างเดียว
> ไม่พอ ต้องเช็ค `show ntp status` ของทุกอุปกรณ์ควบคู่กันด้วยเสมอก่อนวัน Cutover จริง เพราะ Clock
> Drift แม้เพียงไม่กี่นาทีก็ทำให้เกิด "Key Rotation Gap" ได้เหมือนกันแม้ Config วันที่จะถูกต้องเป๊ะ
> ก็ตาม

**บทเรียน ENARSI**: Authentication Mismatch ไม่ได้เกิดจาก Key-string พิมพ์ผิดเท่านั้น (แบบที่
Part 13 Step 130 สาธิตไว้แบบง่ายๆ) — ในระบบที่ทำ Key Rotation ตาม `accept-lifetime`/
`send-lifetime` แบบ Enterprise จริง **การพิมพ์วันที่/ปีผิดแม้แค่ตัวเดียวใน Key-chain ก็ทำให้เกิด
ปัญหาเดียวกันได้** และอันตรายกว่าตรงที่**ไม่มีอาการอะไรเลยจนกว่าจะถึงเวลาที่ Boundary เปลี่ยน**
(Time Bomb) — การตรวจ `show key chain` ทุกตัวแบบ Proactive ก่อนวัน Cutover จริงคือวิธีป้องกันที่
ดีที่สุด ดีกว่าการรอให้ Neighbor หลุดแล้วมาแก้ทีหลัง

---

## Step 559 — Scenario 8: Stub Site กลายเป็น Transit Path เพราะลืม `eigrp stub`

### อาการที่รายงานมา

EIGRP-R2 เพิ่งผ่านการ RMA (Return Merchandise Authorization) เปลี่ยนตัวเครื่องใหม่เนื่องจาก
Hardware เสีย — ทีมงาน Restore Config จาก Backup ที่มีอยู่เพื่อความรวดเร็ว แต่ Backup ที่ใช้ดันเป็น
ไฟล์เก่าก่อนที่ [Part 28 Step 272](part-028-eigrp-advanced.md) จะเพิ่ม Stub Routing เข้าไป —
หลังจากนั้นทีม Network Monitoring สังเกตว่า **Latency ระหว่าง DIST-SW3 กับ DIST-SW4 สูงขึ้นผิดปกติ**
ทั้งที่ Link4 (การเชื่อมตรงระหว่างสองตัว) ยังทำงานปกติดี

### ชั้น 1-3 — Neighbor/Topology/RIB ดูปกติหมด (นี่คือจุดยากของ Scenario นี้)

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
2   10.199.34.14    Gi1/0/24        14  02:10:44    1    100  0  4
1   10.199.34.18    Gi1/0/25        13  00:05:12    2    100  0  9
0   10.199.34.2     Gi1/0/23        14  02:10:44    1    100  0  2
```

ทุก Neighbor ขึ้นปกติ ไม่มี Log ผิดปกติใดๆ — Scenario นี้ **ไม่มี Bug ที่ Layer Neighbor/RIB เลย**
Route ทุกเส้นถูก Install อย่างถูกต้องตาม Metric ที่คำนวณได้จริง **นี่คือเหตุผลที่ Decision Tree ต้อง
มีชั้น 5 (Traffic วิ่งจริงตาม Path ที่ควรหรือไม่) เพราะบางปัญหาไม่ได้ Fail แบบชัดเจน แต่เป็น
"ทำงานถูกทุกอย่างตาม Metric ที่มันเห็น" ทั้งที่ Design จริงไม่ได้ตั้งใจให้ Metric เป็นแบบนี้**

### ชั้น 4-5 — ตรวจ Path จริงด้วย `traceroute` และเทียบ Metric

```
DIST-SW3# traceroute 10.199.34.14
Type escape sequence to abort.
Tracing the route to 10.199.34.14
  1 10.199.34.14 2 msec *  2 msec
```

Traceroute ไปหา DIST-SW4 โดยตรง (ผ่าน Link4) ยังปกติ (1 Hop) — แต่ทีม Monitoring รายงานปัญหาที่
**Traffic ปลายทางอื่นที่ต้องผ่าน DIST-SW4** (เช่น Subnet ฝั่ง Lab หลักที่ Redistribute ผ่าน
DIST-SW4 ออกไป — ในที่นี้จำลองด้วย Test Prefix `172.16.99.0/24` ที่ประกาศไว้ทดสอบ) ไม่วิ่งตรงผ่าน
Link4 อย่างที่ควรจะเป็น:

```
DIST-SW3# show ip route 172.16.99.0
Routing entry for 172.16.99.0/24
  Known via "eigrp 100", distance 90, metric 158720, type internal
  Last update from 10.199.34.18 on GigabitEthernet1/0/25, 00:03:02 ago
  Routing Descriptor Blocks:
  * 10.199.34.18, from 10.199.34.18, 00:03:02 ago, via GigabitEthernet1/0/25
      Route metric is 158720, traffic share count is 1
```

**พบความผิดปกติ**: Prefix `172.16.99.0/24` ที่ประกาศจาก DIST-SW4 ถูกเรียนรู้ผ่าน **EIGRP-R2
(Gi1/0/25, Link5)** แทนที่จะผ่าน **Link4 ตรงไปยัง DIST-SW4 เลย** — เส้นทางที่ควรจะเป็นคือ
DIST-SW3 → DIST-SW4 ตรงๆ (1 Hop, GigE เต็มสปีด) แต่ Metric ที่คำนวณได้กลับเลือก DIST-SW3 → EIGRP-R2
→ DIST-SW4 (2 Hop, ผ่าน Link3 ที่จำลอง WAN ช้าด้วย) ซึ่ง**ไม่มีทางเร็วกว่า Path ตรงได้เลยในทาง
กายภาพ** แต่ EIGRP กลับเลือกเป็น Successor

### ตรวจ Topology Table เพื่อดูว่า EIGRP-R2 เข้ามาเป็น Path ได้อย่างไร

```
DIST-SW3# show ip eigrp topology 172.16.99.0/24
P 172.16.99.0/24, 1 successors, FD is 158720
        via 10.199.34.18 (158720/156416), GigabitEthernet1/0/25
        via 10.199.34.14 (161280/158720), GigabitEthernet1/0/24
```

Metric ผ่าน EIGRP-R2 (158,720) ต่ำกว่าผ่าน Link4 ตรง (161,280) เล็กน้อย — เป็นไปได้ในทางคณิตศาสตร์
เพราะ Link5 (DIST-SW3↔EIGRP-R2) ตั้ง `bandwidth 100000` (จำลอง FastE) ซึ่งบวก Delay รวมแล้วอาจ
ต่ำกว่า Path ที่ผ่าน Link4 จริงเล็กน้อยตามค่าที่ Config ไว้ (นี่คือผลข้างเคียงที่ไม่ตั้งใจจากค่า
Bandwidth/Delay จำลองที่ตั้งไว้ตอนสร้าง Lab) — **ในทาง Design ที่ถูกต้อง EIGRP-R2 ไม่ควรเป็น Path
ตัวเลือกสำหรับ Traffic ระหว่าง DIST-SW3/DIST-SW4 เลยไม่ว่า Metric จะออกมาเท่าไหร่ก็ตาม** เพราะ
EIGRP-R2 คือ Legacy Site Router ที่ควรเป็น **Stub** (รับ Route เท่านั้น ไม่ควรถูกใช้เป็น Transit
ระหว่าง Router อื่นสองตัว) — นี่คือสิ่งที่ Stub Routing มีไว้ป้องกันโดยเฉพาะ (ทวนจาก
[Part 28 Step 272](part-028-eigrp-advanced.md))

### ยืนยันต้นเหตุ — ตรวจ Running-config ของ EIGRP-R2 หลัง RMA

```
EIGRP-R2# show running-config | section router eigrp
router eigrp EIGRP-DEMO
 !
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 9.9.9.2
  network 9.9.9.2 0.0.0.0
  network 10.199.34.0 0.0.0.255
  network 10.199.200.0 0.0.0.255
  ! ไม่มีบรรทัด eigrp stub เลย — หายไปหลัง Restore จาก Backup เก่า
```

**พบต้นเหตุยืนยันชัดเจน**: ไม่มี `eigrp stub connected summary` เลยในเครื่องใหม่หลัง RMA — Backup
ที่ใช้ Restore เป็น Snapshot จากก่อน Part 28 Step 272 จะถูกเพิ่มเข้ามา ทำให้ EIGRP-R2 กลับไปเป็น
Router ธรรมดาที่ยอมรับ Query และถูกเลือกเป็น Successor/Transit ได้เต็มรูปแบบเหมือนเดิม

### แก้ไข

```
EIGRP-R2(config)# router eigrp EIGRP-DEMO
EIGRP-R2(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R2(config-router-af)# eigrp stub connected summary
EIGRP-R2(config-router-af)# end
```

### Verify หลังแก้

```
DIST-SW3# show ip eigrp neighbors detail | include Stub
    10.199.34.18    Gi1/0/25     Foreign router, DIST-SW3 sees it as Stub Peer

DIST-SW3# show ip eigrp topology 172.16.99.0/24
P 172.16.99.0/24, 1 successors, FD is 161280
        via 10.199.34.14 (161280/158720), GigabitEthernet1/0/24     <- เหลือ Path เดียว ผ่าน Link4 ตรง
```

หลังตั้ง Stub, EIGRP-R2 **หยุด Advertise Route ที่ตัวเองไม่ได้เป็นเจ้าของ** (ทวนจาก Part 28
Step 272: ค่า Default `connected + summary` = Advertise เฉพาะ Connected Network กับ Manual
Summary ของตัวเองเท่านั้น ไม่ Advertise Route ที่เรียนรู้มาจาก Neighbor อื่นต่อ) — DIST-SW3 จึงไม่
เห็น Path ผ่าน EIGRP-R2 สำหรับ `172.16.99.0/24` อีกต่อไปเลย เหลือ Path เดียวที่ถูกต้องคือผ่าน Link4
ตรง และ Neighbor ที่เป็น Stub จะไม่ถูก Query อีกต่อไปด้วย (ลดความเสี่ยง SIA ตาม Step 555)

**บทเรียน ENARSI**: `eigrp stub` เป็น Config ที่ **ไม่ทิ้ง Log อะไรเลยตอนหายไป** และไม่ทำให้
Neighbor หลุดหรือ Route หายจาก RIB (Scenario นี้ผ่าน Decision Tree ชั้น 1-3 ทั้งหมด) — วิธีเดียวที่
จะจับได้คือ **สังเกตพฤติกรรม Traffic จริง (ชั้น 5) ที่วิ่งผ่าน Site ที่ไม่ควรเป็น Transit** แล้ว
ไล่กลับไปดู Metric และ Running-config Stub Configuration บน Router นั้น — เป็นเหตุผลสำคัญที่ต้อง
**Document ไว้เสมอว่า Site ใดควรเป็น Stub** และตรวจสอบ Stub Configuration เป็นส่วนหนึ่งของ
Checklist หลัง RMA/Restore Config ทุกครั้ง ไม่ใช่แค่เรื่อง IP Address/Routing Protocol พื้นฐาน

---

## Step 560 — Lab เต็มรูปแบบ: 3 Bug ซ้อนกันในโจทย์เดียว

### สถานการณ์ — Maintenance Window ที่ผิดพลาด

ทีม Network ทำ Maintenance Window กลางดึกเพื่อ (1) Standardize K-value Documentation ทุกตัว
(2) เพิ่ม Security Hardening แบบ Passive-interface ตามมาตรฐานใหม่ และ (3) ทำความสะอาด Legacy
Distribute-list ที่ค้างมานาน — เช้าวันรุ่งขึ้น Legacy Server LAN เข้าไม่ได้เลย และทีมงานที่ไม่ได้
อยู่ใน Maintenance Window เมื่อคืนต้อง Troubleshoot โดยไม่รู้ว่ามีการเปลี่ยนแปลงอะไรไปบ้าง — นี่คือ
สถานการณ์จำลองที่ใกล้เคียงข้อสอบ ENARSI จริงที่สุด: **ไม่รู้ล่วงหน้าว่ามี Bug กี่จุด อยู่ตรงไหน**
ต้องไล่ตาม Decision Tree ของ Step 551 ทีละชั้นจนครบ

### สภาพก่อนแก้ (Baseline พัง 3 จุด)

| จุดที่พัง | ตำแหน่ง | ประเภท Bug | ตรงกับ |
|---|---|---|---|
| 1 | EIGRP-R1 (Link1) | K-value ผิด (`metric weights 0 1 0 3 0 0`) | Step 552 |
| 2 | DIST-SW4 (Link3, Link4) | `passive-interface default` ไม่มี Exception | Step 554 |
| 3 | DIST-SW3 (Process-wide) | `distribute-list` เก่าบล็อก `10.199.200.0/24` | Step 557 |

### เริ่มไล่ตาม Decision Tree — ชั้น 1: Neighbor

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
```

**Neighbor ว่างเปล่าทั้งหมด!** — ไม่มีตัวใดขึ้นเลยแม้แต่ตัวเดียว รวมถึง DIST-SW4 (Link4) ที่ไม่มี
เหตุผลอะไรจะหลุดจาก Bug จุดที่ 1 หรือ 3 เลย บ่งบอกว่ามี **มากกว่า 1 จุดที่กระทบ Neighbor Layer**
— ต้องแยกวิเคราะห์ทีละ Link

```
DIST-SW3#
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.199.34.2 (GigabitEthernet1/0/23) is down: K-value mismatch
```

Link1 (ไปหา EIGRP-R1) มี Log ชัดเจน — ตรงกับ Bug จุดที่ 1 ทันที ส่วน Link4 (ไปหา DIST-SW4) และ
Link5 (ไปหา EIGRP-R2) **ไม่มี Log ใดๆเลย** — ต้องแยกเช็คเป็นกรณีคนละแบบ (Link5/EIGRP-R2 ไม่ได้อยู่
ใน Baseline ที่ตั้งใจให้พังของ Lab นี้ ให้ตรวจดูว่ายังปกติหรือไม่ก่อน)

```
DIST-SW3# show ip interface brief | include Gi1/0/2
GigabitEthernet1/0/23     10.199.34.1     YES manual up                    up
GigabitEthernet1/0/24     10.199.34.13    YES manual up                    up
GigabitEthernet1/0/25     10.199.34.17    YES manual up                    up
```

ทุก Interface ที่ DIST-SW3 up/up ปกติหมด — Layer 1/2 ไม่มีปัญหา ปัญหาอยู่ที่ EIGRP Layer ล้วนๆ
ตรวจ EIGRP-R2 (ปลาย Link5) แยกออกไปก่อนเพื่อยืนยันว่าไม่มี Bug ที่ 4:

```
EIGRP-R2# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
```
*(ว่างเช่นกัน — แต่เพราะ EIGRP-R2 เป็น Stub ที่พึ่งพา DIST-SW3 เป็นหลัก ถ้า DIST-SW3 มีปัญหาฝั่งตรง
ข้าม การไม่มี Neighbor ที่นี่อาจเป็นผลลัพธ์ปลายทาง ไม่ใช่ต้นเหตุอีกจุด — ต้องไปดูที่ DIST-SW3/DIST-SW4
ก่อนเป็นหลัก)*

### แก้ Bug จุดที่ 1 — K-value (เหมือน Step 552)

```
EIGRP-R1(config)# router eigrp EIGRP-DEMO
EIGRP-R1(config-router)# address-family ipv4 unicast autonomous-system 100
EIGRP-R1(config-router-af)# no metric weights 0 1 0 3 0 0
EIGRP-R1(config-router-af)# end
```

**Before/After จุดที่ 1:**

| | ก่อนแก้ | หลังแก้ |
|---|---|---|
| `show ip protocols \| include K1` (EIGRP-R1) | `K1=1, K2=0, K3=3, K4=0, K5=0` | `K1=1, K2=0, K3=1, K4=0, K5=0` |
| `show ip eigrp neighbors` (DIST-SW3, Link1) | ไม่มี Neighbor `.2` | `10.199.34.2 Gi1/0/23` ขึ้นแล้ว |

### ตรวจ Link4 ต่อ — ไปที่ DIST-SW4

```
DIST-SW4# show ip eigrp interfaces
EIGRP-IPv4 Interfaces for AS(100)
                        Xmit Queue   PeerQ        Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  Un/Reliable   SRTT   Un/Reliable   Flow Timer   Routes
```
ว่างเปล่า — ตรงกับลักษณะของ Passive Interface ทันที (เหมือน Step 554 เป๊ะ)

```
DIST-SW4# show running-config | section router eigrp
router eigrp EIGRP-DEMO
 !
 address-family ipv4 unicast autonomous-system 100
  eigrp router-id 1.1.1.14
  passive-interface default
  network 10.199.34.0 0.0.0.255
```

### แก้ Bug จุดที่ 2 — Passive-interface (เหมือน Step 554)

```
DIST-SW4(config)# router eigrp EIGRP-DEMO
DIST-SW4(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW4(config-router-af)# no passive-interface GigabitEthernet1/0/23
DIST-SW4(config-router-af)# no passive-interface GigabitEthernet1/0/24
DIST-SW4(config-router-af)# end
```

**Before/After จุดที่ 2:**

| | ก่อนแก้ | หลังแก้ |
|---|---|---|
| `show ip eigrp interfaces` (DIST-SW4) | ว่างเปล่า (ไม่มี Interface ใดปรากฏ) | `Gi1/0/23`, `Gi1/0/24` ปรากฏครบ |
| `show ip eigrp neighbors` (DIST-SW3, Link4) | ไม่มี Neighbor `.14` | `10.199.34.14 Gi1/0/24` ขึ้นแล้ว |

### ชั้น 1 ผ่านครบแล้ว — ไปชั้น 2-3: Topology/RIB

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
2   10.199.34.14    Gi1/0/24        14  00:01:02    2    100  0  2
1   10.199.34.18    Gi1/0/25        13  06:40:15    2    100  0  8
0   10.199.34.2     Gi1/0/23        14  00:03:41    1    100  0  2
```
Neighbor ครบ 3 ตัวแล้ว — ไปชั้น 2:

```
DIST-SW3# show ip eigrp topology 10.199.200.0/24
P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
```
Topology Table ถูกต้องสมบูรณ์ (Successor + FS ตรงตาม Part 13 Step 123) — ไปชั้น 3:

```
DIST-SW3# show ip route eigrp
D        10.199.34.12/30 [90/158720] via 10.199.34.14, 00:01:44, GigabitEthernet1/0/24
! ไม่มี 10.199.200.0/24 ปรากฏเลย ทั้งที่ Topology Table มี Successor ชัดเจน
```

**พบ Bug จุดที่ 3** — ตรงตาม Pattern Step 557 เป๊ะ (Topology มี, RIB ไม่มี)

```
DIST-SW3# show ip protocols | include distribute
  Incoming update filter list for all interfaces is LEGACY-CLEANUP-BLOCK

DIST-SW3# show ip prefix-list LEGACY-CLEANUP-BLOCK
ip prefix-list LEGACY-CLEANUP-BLOCK: 2 entries
   seq 5 deny 10.199.200.0/22 le 32
   seq 10 permit 0.0.0.0/0 le 32
```

ทีม Maintenance ตั้งใจ "ทำความสะอาด" Filter เก่าแต่ดันสร้าง Filter ใหม่ที่เขียน Prefix Range ผิด
(`/22` กว้างเกินไปครอบคลุม `10.199.200.0/24` เข้าไปด้วยโดยไม่ตั้งใจ — คนละ Bug จาก Step 557 แต่
Pattern เดียวกัน: เขียน Summary Prefix กว้างเกินจำเป็น)

### แก้ Bug จุดที่ 3 — Distribute-list (เหมือน Step 557)

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# no distribute-list prefix LEGACY-CLEANUP-BLOCK in
DIST-SW3(config-router-af-topology)# end
DIST-SW3(config)# no ip prefix-list LEGACY-CLEANUP-BLOCK
```

**Before/After จุดที่ 3:**

| | ก่อนแก้ | หลังแก้ |
|---|---|---|
| `show ip eigrp topology 10.199.200.0/24` | Successor/FS ปกติ (ไม่กระทบ) | เหมือนเดิม (ไม่เปลี่ยน) |
| `show ip route eigrp \| include 10.199.200` | ไม่มี Output | `D 10.199.200.0/24 [90/156160] via 10.199.34.18 ...` |

### Verify สุดท้าย — ครบทั้ง 5 ชั้นของ Decision Tree

```
DIST-SW3# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface     Hold Uptime   SRTT   RTO  Q  Seq
2   10.199.34.14    Gi1/0/24        14  00:05:33    2    100  0  4
1   10.199.34.18    Gi1/0/25        13  06:44:46    2    100  0  8
0   10.199.34.2     Gi1/0/23        14  00:08:12    1    100  0  4

DIST-SW3# show ip eigrp topology
P 10.199.200.0/24, 1 successors, FD is 156160
        via 10.199.34.18 (156160/128256), GigabitEthernet1/0/25
        via 10.199.34.2 (156416/130816), GigabitEthernet1/0/23
P 10.199.34.0/24, 1 successors, FD is 128256
        via Summary (128256/0), Null0

DIST-SW3# show ip route eigrp
D        10.199.200.0/24 [90/156160] via 10.199.34.18, 00:06:02, GigabitEthernet1/0/25
D        10.199.34.12/30 [90/158720] via 10.199.34.14, 00:06:02, GigabitEthernet1/0/24

DIST-SW3# ping 10.199.200.1 source Loopback0
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

ชั้น 1 (Neighbor), ชั้น 2 (Topology), ชั้น 3 (RIB), และชั้น 5 (Traffic จริงผ่าน `ping`) ผ่านครบทุก
ชั้น — Lab นี้ไม่มีปัญหา Metric ผิดปกติ (ชั้น 4) จึงข้ามได้จริงในกรณีนี้ตาม Decision Tree

### สรุปกระบวนการที่ใช้ (Checklist ย้อนดู)

```
1. show ip eigrp neighbors           → พบว่าง 100% → มากกว่า 1 จุดพังแน่นอน
2. แยกวิเคราะห์ทีละ Link ด้วย Log/show ip eigrp interfaces
   - Link1: Log ชัดเจน "K-value mismatch"       → แก้ด้วย no metric weights
   - Link4: ไม่มี Log, interfaces list ว่างเปล่า  → แก้ด้วย no passive-interface
3. หลัง Neighbor ครบ → show ip eigrp topology ถูกต้อง → ไปเช็ค RIB
4. show ip route eigrp ขาด Prefix ที่ Topology Table มี → สงสัย distribute-list
   → show ip protocols ยืนยัน → no distribute-list ... in
5. Verify ครบ 3 จุด ด้วย show ip route + ping จริงปิดท้าย
```

**บทเรียนสำคัญที่สุดของ Part นี้**: เมื่อเจอโจทย์ที่มี Bug มากกว่า 1 จุดซ้อนกัน (แบบข้อสอบ ENARSI จริง
ที่มักให้ Complex Scenario เดียวมากกว่าแยกเป็นข้อย่อย) **ห้ามพยายามหา Bug ทั้งหมดจาก Symptom เดียว**
— ให้ไล่ Decision Tree ทีละชั้น **แก้ปัญหาที่ชั้นล่างสุดที่เจอก่อนเสมอ แล้ว Verify ซ้ำจากชั้น 1 ใหม่
ทุกครั้ง** เพราะการแก้ Bug จุดหนึ่งอาจทำให้เห็น Bug จุดถัดไปที่ซ่อนอยู่ข้างหลังชัดเจนขึ้น (เหมือนที่
Lab นี้ต้องแก้ K-value และ Passive-interface ให้ Neighbor ขึ้นครบก่อน จึงจะเห็น Bug จุดที่ 3 ที่
ซ่อนอยู่ในชั้น RIB ได้อย่างชัดเจน)

---

## แบบฝึกหัดทวนความเข้าใจ Part 56

1. ตาม Decision Tree ของ Step 551 — ถ้า `show ip eigrp neighbors` ว่างเปล่า ควรไปเช็คชั้นไหนต่อ
   (Topology Table หรือ RIB) และเพราะเหตุใดการข้ามไปเช็คชั้นที่สูงกว่าก่อนจึงเป็นการเสียเวลา?
2. เพราะเหตุใด Secondary Address ที่อยู่ Subnet ตรงกับอีกฝั่งพอดี จึงยังไม่ทำให้ EIGRP Neighbor
   ขึ้นได้ ถ้า Primary Address ไม่ตรง Subnet กัน?
3. อธิบายความแตกต่างระหว่างผลลัพธ์ของ `passive-interface` บน Transit Link ของ **OSPF** เทียบกับ
   **EIGRP** — เพราะเหตุใดผลลัพธ์จึงต่างกัน (ทวนจากหลักการ Part 11 และ Step 554)?
4. Route หนึ่งปรากฏใน `show ip eigrp topology` พร้อม Successor ชัดเจน แต่ไม่ปรากฏใน
   `show ip route eigrp` เลย — สาเหตุที่เป็นไปได้มากที่สุดคืออะไร และคำสั่งใดที่ควรตรวจสอบก่อน?
5. เพราะเหตุใดการลืมตั้ง `eigrp stub` บน Legacy Site Router (หลัง RMA/Restore Config) จึงไม่ทำให้
   เกิด Log หรือ Error ใดๆเลย ทั้งที่ส่งผลกระทบต่อ Path Selection จริงในเครือข่าย?

**เฉลย:**

1. ควรไปเช็คชั้น **ต่ำกว่า** (คือกลับไปที่การวินิจฉัยว่าทำไม Neighbor ไม่ขึ้น — Layer 1/2, Primary
   Subnet, AS Number, K-value, Passive-interface, Authentication) **ไม่ใช่ไปเช็ค Topology/RIB
   ก่อน** เพราะถ้า Neighbor ไม่ขึ้นเลย Topology Table และ RIB จะไม่มี Route จาก Neighbor นั้นอยู่แล้ว
   เป็นผลลัพธ์ปลายทางของปัญหาที่ Neighbor ไม่ใช่ต้นเหตุอีกจุด — เสียเวลาไล่ดู Metric/Filtering ของ
   Route ที่ไม่มีทางมีอยู่ตั้งแต่ต้น
2. เพราะ EIGRP Hello Packet ใช้ **Primary Address เป็น Source เท่านั้น** ไม่สนใจ Secondary Address
   ในการสร้าง Adjacency เด็ดขาด (ทวนจาก Part 13 Step 125) ถ้า Primary Address ไม่ตรง Subnet กับ
   อีกฝั่ง Hello จะถูก Discard ด้วยเหตุผล "not in Common subnet" เหมือนกันไม่ว่า Secondary Address
   จะตรงหรือไม่ก็ตาม
3. OSPF ที่ Passive บน Transit Link ยัง Advertise Subnet ของ Interface นั้นเป็น Stub Network ใน
   LSA Type 1 ได้ปกติ เพราะ OSPF มี Topology Database กลาง (LSDB) ที่ทุก Router สร้าง SPF Tree
   เอง — ส่วน EIGRP ไม่มี Topology Database กลางแบบนั้น ถ้าไม่มี Neighbor ขึ้นบน Transit Link เลย
   จะไม่มีทาง Advertise Prefix ที่อยู่ฝั่งตรงข้าม Link นั้นได้เลย ทำให้ Router ฝั่งนั้นกลายเป็น "เกาะ"
   ที่ตัดขาดจาก EIGRP Domain ทั้งหมด ต่างจาก OSPF ที่เสียแค่การเป็น Neighbor แต่ยังเห็น Topology
   ผ่านทางอื่นได้
4. สาเหตุที่เป็นไปได้มากที่สุดคือ **`distribute-list` ที่กรอง Route นั้นไม่ให้ Install ลง RIB**
   (EIGRP `distribute-list` มีผลต่อการ Install RIB ไม่ใช่การเก็บลง Topology Table ซึ่งยังเก็บไว้
   ตามปกติเพื่อให้ DUAL คำนวณ Feasibility Condition ต่อไปได้) — ควรตรวจ `show ip protocols` ดูบรรทัด
   "Incoming/Outgoing update filter list" ก่อน แล้วตรวจ ACL/Prefix-list/Route-map ที่ผูกอยู่
5. เพราะ `eigrp stub` ไม่กระทบการสร้าง Neighbor เลย (Neighbor ยังขึ้นปกติ), ไม่กระทบ Route ของ
   ตัวเองที่ Advertise ออกไป (Connected/Summary ยัง Advertise ตามเดิม), และไม่ทำให้ Route ใดหายไป
   จาก RIB ของ Router อื่น (มีแต่ Path ที่ "ไม่ควรถูกใช้" กลับมาเป็นตัวเลือกได้อีกครั้ง) — ผลกระทบ
   จึงปรากฏเฉพาะที่ **พฤติกรรม Path Selection จริง** (Decision Tree ชั้น 4-5) ซึ่งต้องสังเกตจาก
   Traffic/Latency ผิดปกติ ไม่มี Log หรือ Error ใดๆแจ้งเตือนเลยแม้แต่บรรทัดเดียว

---

## สรุป Part 56

Part นี้เปิดฉากระดับ **CCNP ENARSI (300-410)** ด้วยการนำ EIGRP Demo Topology ที่วางฐานไว้ตั้งแต่
Part 13 และขยายเพิ่มใน Part 28 กลับมาสร้างเป็น **8 Scenario Troubleshooting เต็มรูปแบบ** ครอบคลุม
ทุกชั้นของ Decision Tree ที่เสนอไว้ใน Step 551: K-value Mismatch, Primary/Secondary Subnet
ไม่ตรงกัน, Passive-interface ผิดที่บน Transit Link, SIA จาก Summarization ที่ไม่ครบ, Bandwidth
ไม่ตรงกับ Physical จริง, Distribute-list ที่ลืมถอด, Authentication Mismatch จาก Key Rotation
ที่พลาดวันที่, และ Stub Configuration ที่หายไปหลัง Restore Config — ปิดท้ายด้วย Lab เต็มรูปแบบที่
รวม 3 Bug จากคนละชั้นของ Decision Tree เข้าด้วยกันในโจทย์เดียว เพื่อฝึกกระบวนการไล่ปัญหาแบบเป็น
ระบบตามที่ข้อสอบ ENARSI ต้องการจริง — **Decision Tree ทั้ง 5 ชั้นจาก Step 551 จะถูกนำกลับมาใช้ซ้ำ
ตลอด 14 Part ที่เหลือของระดับ ENARSI** (Part 57-70) เพียงแค่เปลี่ยน Routing Protocol/Technology
ที่ใช้วินิจฉัยไปเรื่อยๆ

✅ **พร้อมสำหรับ Part 57**: ย้ายจาก EIGRP ไปสู่ **OSPF Troubleshooting** บน Lab Topology หลักของ
หลักสูตร (CORE-SW1/2, DIST-SW1-4) — นำ Decision Tree แบบเดียวกันมาประยุกต์กับ Adjacency States,
LSA Type ผิดปกติ, Area Design Bug, และ Route Summarization ที่ ABR/ASBR ตั้งผิด

**ไปต่อ:** [Part 57 — OSPF Troubleshooting →](part-057-ospf-troubleshooting.md)
