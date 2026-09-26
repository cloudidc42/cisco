# Part 59 — Route Redistribution Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 581–590 จาก 1000**

> ย้อนกลับ: [Part 58 — BGP Troubleshooting](part-058-bgp-troubleshooting.md)

> Part 58 ปิดท้ายการ Troubleshoot BGP ที่ WAN Edge ไปแล้ว Part นี้เดินหน้าเข้าสู่จุดที่ CCNP
> ENARSI (300-410) ถือว่า**ยากที่สุด**ของทั้งข้อสอบ: **Route Redistribution Troubleshooting**
> — เพราะ Redistribution ไม่ใช่ "Protocol" ตัวหนึ่ง แต่เป็น**จุดเชื่อมระหว่าง 2 Protocol**
> ทำให้การ Troubleshoot ต้องเข้าใจทั้งสองฝั่งพร้อมกัน และเข้าใจว่า "จุดเชื่อม" นั้นทำงานถูกต้อง
> หรือไม่ Part นี้จะใช้ Lab เดิมจาก [Part 31](part-031-route-redistribution.md) ทั้งหมด — Mutual
> Redistribution OSPF↔EIGRP ที่ DIST-SW3/DIST-SW4 พร้อม Route Tag (90/110) ป้องกัน Loop, และ
> Controlled Redistribution BGP↔OSPF ที่ WAN-EDGE-1/WAN-EDGE-2 — แล้ว**จงใจทำให้พังทีละจุด**
> ตาม Fault Pattern ที่ออกข้อสอบจริงและเกิดขึนบ่อยที่สุดในงาน Production เพื่อฝึก
> การวินิจฉัยแบบเป็นระบบ (Systematic Diagnosis) ทุก Scenario ในนี้ให้ Config ที่พังจริง,
> อาการที่เห็นจาก `show` command จริง, กระบวนการวินิจฉัยทีละขั้น, Root Cause, และ Fix ที่ Verify
> แล้วว่าใช้ได้จริง — ตรงตามแนวข้อสอบ ENARSI ที่มักถาม "Router นี้ควรเห็น Route ใด แต่ไม่เห็น
> เพราะเหตุใด"

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 581 | Methodology การ Troubleshoot Redistribution — ความซับซ้อนที่เพิ่มมาจากการมี 2 Protocol |
| 582 | Scenario 1 — ลืม Seed Metric: Route หายไปเงียบๆ (EIGRP) หรือใช้ Default โดยไม่รู้ตัว (OSPF) |
| 583 | Scenario 2 — Routing Loop จาก Route Tag Filter หายไปที่ ASBR จุดเดียว |
| 584 | Scenario 3 — Administrative Distance ผิด ทำให้ Route ที่ควรชนะกลับแพ้ |
| 585 | Scenario 4 — Prefix-list `ge`/`le` ผิด บล็อก Route ที่ควร Redistribute ออกไปโดยไม่ตั้งใจ |
| 586 | Scenario 5 — BGP Full Table รั่วเข้า IGP เพราะ Route-map หลุดออกจากคำสั่ง Redistribute |
| 587 | Scenario 6 — Metric-type ไม่ตรงกัน (E1 ปน E2) ทำให้เลือก Path ผิดที่มี ASBR หลายจุด |
| 588 | Scenario 7 — `default-metric` แบบ Global ดันไปกระทบ Redistribute Statement ที่ไม่ควรถูกกระทบ |
| 589 | Scenario 8 — Asymmetric Routing จาก Redistribution ที่มีอยู่แค่จุดเดียว (ควรมี 2 จุด) |
| 590 | Lab เต็มรูปแบบ: 2 บั๊กซ้อนกัน — Tag Filter หาย + Seed Metric หาย ในเวลาเดียวกัน |

---

## Step 581 — Methodology การ Troubleshoot Redistribution: ความซับซ้อนที่เพิ่มมาจากการมี 2 Protocol

### ทำไม Troubleshoot Redistribution ยากกว่า Troubleshoot Protocol เดียว

เมื่อ Troubleshoot OSPF อย่างเดียว (Part ก่อนๆ) หรือ BGP อย่างเดียว (Part 58) ปัญหาจะอยู่**ใน
Protocol เดียว** — Neighbor ขึ้นไม่ติด, Route ไม่ถูกเลือก, Filter ผิด ฯลฯ ทุกอย่างวินิจฉัยได้ด้วย
`show` command ของ Protocol นั้นตัวเดียว แต่ Redistribution มี Router ตัวหนึ่ง (ASBR) ที่**ยืนอยู่
กึ่งกลางระหว่าง 2 โลก** ทำให้เกิดตัวแปรที่ต้องตรวจสอบเพิ่มขึ้นอีกชั้นเสมอ:

| คำถามที่ต้องตอบก่อนเสมอ | เหตุผลที่ต้องถาม |
|---|---|
| **ฝั่งไหนพัง** — ต้นทาง (Source Protocol ที่ ASBR "ดึง" Route มา) หรือปลายทาง (Destination Protocol ที่ ASBR "ฉีด" Route เข้าไป)? | Route อาจมีอยู่จริงในฝั่ง Source แต่ไม่ถูกฉีดเข้า Destination เลย ต้องแยกให้ออกว่าปัญหาอยู่คนละฝั่งกับที่มีอาการ |
| **ทิศทางไหน** — A→B, B→A, หรือทั้งสองทิศทาง (Mutual)? | Redistribution เกือบทุกจุดเป็นคนละคำสั่งกันคนละทิศทาง (`redistribute eigrp ... under router ospf` ≠ `redistribute ospf ... under router eigrp`) ทิศทางหนึ่งพังไม่ได้แปลว่าอีกทิศทางพังด้วย |
| **Seed Metric ถูกต้องหรือไม่** — มีอยู่จริงไหม ค่าตรงกับที่ตั้งใจไหม | Route ที่หายไปทั้งหมดมักเป็นเรื่อง Seed Metric (Step 582) ในขณะที่ Route ที่อยู่แต่ Path ผิดมักเป็นเรื่องค่า Metric ไม่ตรงกับที่ตั้งใจ (Step 587) |
| **Filter (Route-map/Prefix-list/Tag) ทำงานตามที่ตั้งใจหรือไม่** | Filter ที่ตั้งใจกันไว้ (selective redistribution, loop prevention) อาจ Match ผิด Sequence หรือหายไปโดยไม่มีใครรู้ (Step 583, 585) |
| **AD ของ Route ที่ถูก Redistribute ชนกับ Route อื่นในตารางเดียวกันหรือไม่** | ต่อให้ Redistribute ถูกทุกอย่าง Router ปลายทางอาจยังเลือก Route ผิดเพราะ AD (Step 584) |

### Flowchart การวินิจฉัยแบบเป็นระบบ (ใช้กับทุก Scenario ใน Part นี้)

```
1. ยืนยันอาการก่อน: Router ปลายทางเห็น Route หรือไม่?
   ├─ ไม่เห็นเลย ────────────► ไปข้อ 2 (สงสัย Seed Metric / Filter deny ทั้งหมด)
   └─ เห็น แต่ Path/Metric ผิดที่คาด ─► ไปข้อ 4 (สงสัย Metric-type / AD / Tag ผิดจุด)

2. ที่ ASBR: show ip route <prefix> ในฝั่ง Source Protocol ก่อน
   ├─ ไม่มีใน Source เลย ──► ปัญหาไม่ใช่ Redistribution เลย (เป็นปัญหาฝั่ง Source Protocol เอง
   │                          — ย้อนกลับไป Part ที่เกี่ยวกับ Protocol นั้นแทน)
   └─ มีใน Source ────────► ไปข้อ 3 (ปัญหาอยู่ที่ตัว Redistribution แน่นอน)

3. ที่ ASBR: show ip protocols + show run | section router <dest-protocol>
   ├─ ไม่มีบรรทัด "Redistributing External Routes from" ─► Redistribute Statement หายไป/ผิด Process-ID
   ├─ มี แต่ไม่มี metric/default-metric ────────────────► Seed Metric หาย (Step 582)
   └─ มี route-map ผูกอยู่ ──► show route-map ดู Sequence ไหน Match/Deny (Step 583, 585, 586)

4. ที่ Router ปลายทาง: show ip ospf database external <prefix> หรือ
   show ip eigrp topology <prefix> ดู ADV Router/Originating Router, Metric-type, Tag
   ├─ มี ASBR มากกว่า 1 ตัว Advertise Prefix เดียวกัน ─► เทียบ Metric-type/Metric ระหว่าง ASBR (Step 587)
   ├─ Metric โต/Flap ขึ้นเรื่อยๆ เมื่อ Poll ซ้ำ ─► สงสัย Loop จาก Tag Filter หาย (Step 583)
   └─ Path ที่เลือกไม่ตรง Protocol ที่คาด (เช่นคาด O E2 แต่ได้ S) ─► เทียบ AD จริงในตาราง (Step 584)
```

> **หลักคิดสำคัญที่สุดของ Part นี้**: อย่าเริ่มแก้ที่ Router ปลายทางที่เห็นอาการทันที — ให้เริ่มที่
> **ASBR เสมอ** เพราะ ASBR คือจุดเดียวที่ Config ทั้งสอง Protocol อยู่ด้วยกัน การแก้ที่ปลายทาง
> (เช่นเพิ่ม Static Route ปะ) เป็นแค่ Workaround ที่จะสร้างปัญหา AD/Loop ซ้อนเพิ่มขึ้นไปอีกเสมอ

---

## Step 582 — Scenario 1: ลืม Seed Metric — Route หายไปเงียบๆ (EIGRP) หรือใช้ Default โดยไม่รู้ตัว (OSPF)

### Case A (พังจริง): Redistribute OSPF เข้า EIGRP โดยไม่มี Seed Metric

**สถานการณ์**: ทีม Network เพิ่ง Migrate DIST-SW3 ไปใช้ Template Config ใหม่ (Automation จาก
Part 47/48) และมือ Copy ผิดบรรทัด ทำให้ Redistribute Statement ที่เคยมี `metric` ครบ (จาก
Part 31 Step 304) หายไปครึ่งหนึ่ง:

```
! ===== Config ที่พังบน DIST-SW3 (หลัง Migrate) =====
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 route-map OSPF-TO-EIGRP
  exit-af-topology
 exit-address-family
```

สังเกตว่าบรรทัด `metric 1000000 100 255 1 1500` ที่ควรอยู่ต่อท้าย `redistribute ospf 1` (ตาม
Part 31 Step 304) **หายไปทั้งหมด** เหลือแค่ `route-map OSPF-TO-EIGRP` — และไม่มี `default-metric`
อยู่ใต้ `router eigrp` เลยด้วย

### อาการที่เห็น

```
EIGRP-R1# show ip route eigrp
% (ไม่มี Output เลยสักบรรทัด สำหรับ Prefix ของ Campus)

EIGRP-R1# show ip route | include 10.10
% (ไม่พบ Prefix ใดของ VLAN 10/20/30/40/99 เลยแม้แต่เส้นเดียว)
```

**ไม่มี Error Message ใดๆ ทั้งสิ้น** — คำสั่ง `redistribute ospf 1 route-map OSPF-TO-EIGRP` ถูก
ยอมรับเข้า Running-config ปกติทุกประการ (ผ่าน `show run` ตรวจสอบก็ดูเหมือนใช้งานได้)

### กระบวนการวินิจฉัย

```
! Step 1: ยืนยันว่าฝั่ง Source (OSPF) มี Route จริง
DIST-SW3# show ip route ospf | include 10.10.10.0
O    10.10.10.0/24 [110/11] via 10.255.30.1, ...

! Step 2: OSPF มี Route จริง -> ตรวจ show ip protocols ฝั่ง EIGRP
DIST-SW3# show ip protocols
Routing Protocol is "eigrp EIGRP-DEMO"
  Redistributing: eigrp EIGRP-DEMO
    Redistributing External Routes from,
      ospf 1 (route-map OSPF-TO-EIGRP)

! Statement มีอยู่จริงแต่ไม่มีบรรทัดบอก Metric -> เจาะ Running-config โดยตรง
DIST-SW3# show run | section router eigrp
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 route-map OSPF-TO-EIGRP
  exit-af-topology
```

**พบ Root Cause ทันที** — ไม่มีคำว่า `metric` ต่อท้ายเลย และไม่มี `default-metric` อยู่ในบล็อกนี้
เลยด้วย ตรงกับหลักการ Step 302 ของ Part 31 พอดี: **EIGRP ไม่มี Default Seed Metric ให้ใช้แทน —
ถ้าไม่ระบุ Route จะไม่ถูกติดตั้งเข้า EIGRP เลยแบบเงียบๆ ไม่มี Error**

### Fix

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# no redistribute ospf 1 route-map OSPF-TO-EIGRP
DIST-SW3(config-router-af-topology)# redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
```

### Verify

```
EIGRP-R1# show ip route eigrp
D EX    10.10.10.0/24 [170/28160256] via 10.199.34.1, 00:00:08, GigabitEthernet0/0
D EX    10.10.20.0/24 [170/28160256] via 10.199.34.1, 00:00:08, GigabitEthernet0/0
D EX    10.10.30.0/24 [170/28160256] via 10.199.34.1, 00:00:08, GigabitEthernet0/0
```

Route กลับมาทันทีที่เพิ่ม Seed Metric — ยืนยัน Root Cause ถูกต้อง

### Case B (ไม่พัง แต่เป็นกับดัก): Redistribute EIGRP เข้า OSPF โดยไม่ระบุ `metric`

ตรงข้ามกับ Case A — ถ้าลืม `metric` ตอน Redistribute เข้า **OSPF** แทน (เช่น
`redistribute eigrp 100 subnets tag 90` โดยไม่มี `metric 20`) คำสั่งนี้**ไม่พัง** เพราะ OSPF มี
Default Seed Metric = 20 ให้ใช้แทนเสมอ (ตามตาราง Step 302 ของ Part 31) Route จะยังโผล่ปกติเป็น
`O E2 ... [110/20]` — **แต่ปัญหาคือมันอาจไม่ใช่ค่าที่ Design ตั้งใจไว้จริง** เช่นถ้า Design
ต้องการให้ Path นี้แพงกว่า Path อื่นโดยตั้งใจ (Metric 30) แต่ลืมใส่ ผลคือ Router จะเลือก Path
นี้เป็น Best Path โดยไม่ตรงกับที่ Design ไว้ ทั้งที่ไม่มี Error หรืออาการชัดเจนให้เห็นเลย

> **ข้อสังเกตสำหรับข้อสอบ**: คำถามแนว "Route ไม่โผล่เลย" → มักเป็น EIGRP (Case A) ส่วนคำถามแนว
> "Route โผล่ แต่ Path/Metric ไม่ตรงกับที่ Design ตั้งใจ" → มักเป็น OSPF ที่ตกไปใช้ Default 20
> (Case B) วิธีตรวจสอบทั้งสอง Case ให้ใช้คำสั่งเดียวกันเสมอคือ `show run | section router` เทียบ
> กับ Design Document ว่า `metric` ที่ตั้งใจไว้จริงๆคือค่าอะไร

---

## Step 583 — Scenario 2: Routing Loop จาก Route Tag Filter หายไปที่ ASBR จุดเดียว

### สถานการณ์

DIST-SW3 และ DIST-SW4 ต่างเป็น ASBR ที่ Redistribute Mutual OSPF↔EIGRP พร้อมกัน (ตาม Part 31
Step 307 — ป้องกัน Loop ด้วย Route-map + Tag 90/110 ที่ต้อง**เหมือนกันทุกประการทั้งสองตัว**)
วิศวกรคนหนึ่งเข้าไปแก้ Route-map บน **DIST-SW4 เพียงตัวเดียว** เพื่อ "แก้ปัญหา" ที่เข้าใจผิดว่า
Legacy Server บางเครื่องเข้า Campus ไม่ได้ (จริงๆเป็นปัญหาอื่น) แล้วลบ Sequence `deny` ที่กัน Loop
ออกไปโดยไม่รู้ผลกระทบ:

```
! ===== Route-map ที่พังบน DIST-SW4 (deny 10 ถูกลบออกไป) =====
DIST-SW4(config)# no route-map OSPF-TO-EIGRP deny 10
```

Route-map `OSPF-TO-EIGRP` บน DIST-SW4 ตอนนี้เหลือแค่:

```
DIST-SW4# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, permit, sequence 20
  Match clauses:
  Set clauses:
    tag 110
  Policy routing matches: 214 packets, ...
```

**ไม่มี Sequence 10 (`deny 10 match tag 90`) อีกต่อไป** — ทุก Route ที่ผ่านมาจาก OSPF
(รวมถึง Route ที่มี Tag 90 ซึ่งแปลว่า**เพิ่ง Redistribute มาจาก EIGRP ที่ DIST-SW3 มาแล้ว**) จะถูก
`permit 20` รับหมด แล้ว `set tag 110` ทับ Tag เดิมทิ้งไปด้วย — Loop จาก Step 307 ของ Part 31
กลับมาเกิดซ้ำทันที แต่ครั้งนี้ทางเดียว (ทิศ OSPF→EIGRP ที่ DIST-SW4 เท่านั้น)

### อาการที่เห็น: Metric "โต" ขึ้นทุกครั้งที่ Poll ซ้ำ

```
EIGRP-R2# show ip eigrp topology 10.10.10.0/24
EIGRP-IPv4 (AS 100): Topology entry for 10.10.10.0/24
  State is Passive, 1 Successor(s), FD is 28160256

! -- รอ 2 นาที แล้ว Poll ซ้ำคำสั่งเดิม --

EIGRP-R2# show ip eigrp topology 10.10.10.0/24
EIGRP-IPv4 (AS 100): Topology entry for 10.10.10.0/24
  State is Passive, 1 Successor(s), FD is 30720512
```

FD (Feasible Distance) ของ Prefix เดิม**เปลี่ยนค่าไปเรื่อยๆ** ทั้งที่ไม่มีการเปลี่ยน Topology
จริงเลย — สัญญาณคลาสสิกของ Route ที่กำลัง "วิ่งวน" ผ่านการ Redistribute ซ้ำๆ (แต่ละรอบผ่าน
EIGRP→OSPF→EIGRP จะสะสม Cost ภายใน OSPF เพิ่มเข้าไปในรอบใหม่ก่อนแปลงกลับเป็น Composite Metric
รอบต่อไป) เมื่อเวลาผ่านไปนานพอ อาจเห็น `%DUAL-5-NBRCHANGE` Flap ถี่ขึ้นเรื่อยๆ ด้วย

### กระบวนการวินิจฉัย

```
! Step 1: ดูที่ OSPF Database ว่า Prefix เดียวกันถูก Advertise จากกี่ ASBR
CORE-SW2# show ip ospf database external 10.10.10.0

            OSPF Router with ID (1.1.1.2) (Process ID 1)
                Type-5 AS External Link States

  LS age: 12
  Advertising Router: 1.1.1.14
  LS Seq Number: 8000004A     <--- Sequence Number สูงและวิ่งขึ้นเร็วผิดปกติ
  Network Mask: /24
        Metric Type: 2
        Metric: 20
        External Route Tag: 110
```

**สิ่งที่ผิดปกติทันที**: `10.10.10.0/24` เป็น Subnet ของ Campus เอง (VLAN 10, Native OSPF
Intra-area) แต่กลับมี **LSA Type-5 (External)** จาก ASBR `1.1.1.14` (DIST-SW4) ด้วย! และ
`LS Seq Number` วิ่งขึ้นเร็วมาก (บ่งบอกว่า LSA นี้ถูก Re-originate ใหม่ซ้ำๆ ต่อเนื่อง)

```
! Step 2: ยืนยันที่ DIST-SW4 ว่า Route-map ตัวไหน "รับ" Route Tag 90 เข้ามาโดยไม่ควร
DIST-SW4# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, permit, sequence 20
  Match clauses:
  Set clauses:
    tag 110
  Policy routing matches: 3841 packets, ...    <--- ตัวเลขวิ่งเร็วมาก ยืนยันว่า Match ทุก Route จริง
```

ไม่มี Sequence `deny 10 match tag 90` เหลืออยู่เลย — **Root Cause ยืนยันชัดเจน**: DIST-SW4
รับ Route ที่มี Tag 90 (มาจาก EIGRP ผ่าน DIST-SW3 อยู่แล้ว) กลับเข้า EIGRP อีกรอบ แล้ว Tag
ใหม่เป็น 110 ทำให้รอบต่อไปมันจะถูก Redistribute กลับเข้า OSPF ที่ DIST-SW3 ได้อีก (เพราะ
Route-map ฝั่ง `EIGRP-TO-OSPF` ของ DIST-SW3 เช็ค `deny match tag 110` เท่านั้น — Route ที่มี
Tag 110 ใหม่จาก DIST-SW4 จะไม่ผ่าน DIST-SW3 กลับเข้า OSPF ก็จริง แต่ที่ผ่านไปแล้วฝั่ง EIGRP
ก็ยังคงวิ่งวนไปมาระหว่าง DIST-SW3↔EIGRP↔DIST-SW4↔OSPF↔DIST-SW3 ได้เรื่อยๆ ตามรูปใน Step 307)

### Fix

```
DIST-SW4(config)# route-map OSPF-TO-EIGRP deny 10
DIST-SW4(config-route-map)# match tag 90
DIST-SW4(config-route-map)# exit
DIST-SW4(config)# route-map OSPF-TO-EIGRP permit 20
DIST-SW4(config-route-map)# set tag 110
```

`route-map <name> <action> <seq>` ที่ระบุ Sequence Number ที่หายไปกลับเข้าไปใหม่ (10) จะ
**สร้าง Sequence นั้นกลับมา** ไม่กระทบ Sequence 20 ที่มีอยู่แล้ว (ต่างจากการลบทั้ง Route-map แล้ว
พิมพ์ใหม่ทั้งหมดซึ่งเสี่ยงพิมพ์ผิดมากกว่า)

### Verify

```
DIST-SW4# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, deny, sequence 10
  Match clauses:
    tag 90
  Set clauses:
  Policy routing matches: 0 packets, 0 bytes
route-map OSPF-TO-EIGRP, permit, sequence 20
  Match clauses:
  Set clauses:
    tag 110
  Policy routing matches: 12 packets, ...

! รอ 5 นาที ตรวจสอบ FD อีกครั้ง
EIGRP-R2# show ip eigrp topology 10.10.10.0/24
EIGRP-IPv4 (AS 100): Topology entry for 10.10.10.0/24
  State is Passive, 1 Successor(s), FD is 28160256    <--- ค่าคงที่แล้ว ไม่โตอีก
```

> **บทเรียนสำคัญของ Scenario นี้**: Tag ที่ใช้ป้องกัน Loop ต้อง Config **เหมือนกันทุกประการทั้ง
> 2 จุด** เสมอ — การแก้ Route-map แค่จุดเดียวโดยไม่รู้ว่ามันเป็นกลไก Loop-prevention คือสาเหตุ
> Outage จริงในระบบ Production ที่พบบ่อยที่สุดของหัวข้อ Redistribution ทั้งหมด ควรมี Comment
> (`!` บน IOS ไม่รองรับ Inline Comment ใน Route-map แต่ใช้ Description หรือ Naming ที่ชัดเจน
> เช่น `OSPF-TO-EIGRP-LOOP-GUARD` แทนได้) เพื่อเตือนว่า Sequence นี้**ห้ามลบ**

---

## Step 584 — Scenario 3: Administrative Distance ผิด ทำให้ Route ที่ควรชนะกลับแพ้

### ทบทวนตาราง AD ที่เกี่ยวข้อง (จาก Part 9 Step 85)

| Source | AD |
|---|---|
| Connected | 0 |
| Static | 1 |
| eBGP | 20 |
| EIGRP (Internal) | 90 |
| OSPF (Internal + External) | 110 |
| EIGRP (External, `D EX`) | 170 |
| iBGP | 200 |

### สถานการณ์

ระหว่างการทดสอบ Failover ของ Legacy Server LAN ก่อน Part 31 จะ Migrate ไปใช้ OSPF↔EIGRP
Redistribution เต็มรูปแบบ วิศวกรเคยเพิ่ม Static Route ชั่วคราวไว้ที่ CORE-SW1 เพื่อทดสอบเส้นทาง
สำรองผ่าน WAN Edge cross-link (`10.10.254.8/30`) — แล้ว**ลืมลบทิ้ง**หลังเลิกทดสอบ:

```
! ===== Static Route ตกค้างบน CORE-SW1 (ควรลบไปนานแล้ว) =====
CORE-SW1(config)# ip route 10.199.200.0 255.255.255.0 10.10.254.9
```

(`10.10.254.9` คือ IP ฝั่ง WAN-EDGE-2 บน Cross-link — เส้นทางนี้ไม่มี Route ต่อไปยัง Legacy
Server LAN จริงเลย เป็นแค่ปลายตันที่เหลือจากการทดสอบเก่า)

### อาการที่เห็น

```
CORE-SW1# show ip route 10.199.200.0
Routing entry for 10.199.200.0/24
  Known via "static", distance 1, metric 0     <--- AD=1 ชนะทุก Protocol เสมอ
  Routing Descriptor Blocks:
  * 10.10.254.9
      Route metric is 0, traffic share count is 1
```

```
PC-Legacy(บนวง 10.199.200.0/24)> ping 10.10.10.55
Success rate is 0 percent (0/5)     <--- Ping ไม่ผ่านเลย
```

Traffic จาก Campus ไปยัง Legacy Server LAN ถูกส่งไปทาง WAN-EDGE-2 (ปลายตัน) แทนที่จะไปทาง
DIST-SW3 (เส้นทางจริงที่ถูกต้องผ่าน OSPF E2 ที่ Redistribute มาจาก Step 303/307)

### กระบวนการวินิจฉัย

```
! Step 1: เทียบว่า Route ที่ "ควรจะ" ถูกเลือกคืออะไร โดยดู OSPF Database ตรงๆ (ไม่ผ่าน RIB)
CORE-SW1# show ip ospf database external 10.199.200.0
  Advertising Router: 1.1.1.13
        Metric Type: 2
        Metric: 20
        External Route Tag: 90
```

ยืนยันว่า OSPF **มี** External Route สำหรับ Prefix นี้อยู่จริงใน Database (Redistribution ที่
DIST-SW3 ทำงานถูกต้อง 100%) — แต่ `show ip route` กลับไม่แสดง Route นี้เป็น Best Path เลย

```
! Step 2: ตรวจสอบ RIB ตรงๆ ว่า Route ไหนถูกเลือกจริงและด้วย AD เท่าไหร่
CORE-SW1# show ip route 10.199.200.0 255.255.255.0 longer-prefixes
Routing entry for 10.199.200.0/24
  Known via "static", distance 1, metric 0
```

**Root Cause ยืนยัน**: `show ip route` แสดง AD=1 (Static) ชนะ OSPF E2 (AD=110) ตามกฎ AD ปกติ
ของ IOS — เมื่อมี Route ปลายทางเดียวกันจากหลาย Source, IOS เลือกด้วย **AD ต่ำที่สุดเสมอ**
ไม่สนใจ Metric เลยแม้แต่น้อย (Metric ใช้เทียบก็ต่อเมื่อ AD เท่ากันเท่านั้น) Static Route ที่ตกค้าง
จึงชนะ Route ที่ Redistribute มาอย่างถูกต้องแบบไม่มีทางสู้

### ตารางเทียบ AD จริงของ Scenario นี้

| Route | Source | AD | ผลลัพธ์ |
|---|---|---|---|
| `10.199.200.0/24 via 10.10.254.9` | Static (ตกค้าง) | **1** | **ชนะ (ผิด — ตั้งใจให้เป็น Backup ชั่วคราวเท่านั้น)** |
| `10.199.200.0/24 via 10.255.30.1` | OSPF E2 (Redistributed จาก EIGRP) | 110 | แพ้ (แต่คือ Path ที่ถูกต้องจริง) |

### Fix

```
CORE-SW1(config)# no ip route 10.199.200.0 255.255.255.0 10.10.254.9
```

### Verify

```
CORE-SW1# show ip route 10.199.200.0
Routing entry for 10.199.200.0/24
  Known via "ospf 1", distance 110, metric 20, type extern 2
  Last update from 10.255.10.2 on TenGigabitEthernet1/0/1, 00:00:03 ago

PC-Legacy> ping 10.10.10.55
Success rate is 100 percent (5/5)
```

> **ข้อสังเกตสำหรับข้อสอบ**: คำถามแนว AD ใน Redistribution มี 2 แบบเสมอ — (1) **Native Route
> ที่ดีกว่าแพ้ Route Redistribute ที่แย่กว่า** เพราะ AD ของฝั่ง External สูงกว่าโดยไม่ตั้งใจ
> (พบใน Part 31 กรณี `D EX`=170 แพง กว่า `D`=90 ซึ่งเป็นพฤติกรรม**ที่ตั้งใจ**อยู่แล้ว) กับ
> (2) แบบใน Scenario นี้ — **Route ที่ควรชนะ (Redistributed) กลับแพ้ Native Route ที่แย่กว่า
> (Static ตกค้าง)** เพราะ AD ต่ำผิดที่ผิดทาง ทั้งสองแบบแก้ด้วยการตรวจ `show ip route <prefix>`
> ดูฟิลด์ `distance` เทียบกับตาราง AD มาตรฐานเสมอ ไม่ใช่เดา

---

## Step 585 — Scenario 4: Prefix-list `ge`/`le` ผิด บล็อก Route ที่ควร Redistribute ออกไปโดยไม่ตั้งใจ

### ทบทวน Selective Redistribution จาก Part 31 Step 306

Design เดิมอนุญาตแค่ `10.199.200.0/24` (Legacy Server LAN) เท่านั้นเข้า OSPF ผ่าน:

```
ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24
```

### สถานการณ์

ทีม Legacy Site เพิ่ม VLAN ใหม่บน EIGRP-R2 สำหรับ Server กลุ่มที่สอง (`10.199.201.0/24`) และ
ขอให้ Campus มองเห็น Subnet นี้ด้วย วิศวกรพยายามแก้ Prefix-list ให้ "ครอบคลุมกว้างขึ้น" โดยหวังว่า
`ge`/`le` จะช่วยจับ Subnet ใหม่ที่จะมีในอนาคตด้วย แต่เขียนผิดจุด:

```
! ===== Prefix-list ที่พัง (ตั้งใจจะให้ครอบคลุม /24 เดิม บวก Subnet ย่อยในอนาคต) =====
DIST-SW3(config)# no ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24
DIST-SW3(config)# ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24 ge 25 le 32
```

### ทำไมคำสั่งนี้ถึงพัง — ทบทวนความหมายของ `ge`/`le`

`ip prefix-list <name> permit <network>/<len> [ge <ge-len>] [le <le-len>]` มีความหมายว่า
"Match เฉพาะ Prefix ที่อยู่ภายใต้ `<network>/<len>` **และ** มี Subnet Mask Length อยู่ในช่วง
`ge-len` ถึง `le-len` เท่านั้น" — **ถ้าใส่ `ge 25` แปลว่า Prefix ที่มี Mask `/24` เอง (Mask
Length = 24, น้อยกว่า 25) จะ**ไม่ Match**อีกต่อไป** เพราะ 24 ไม่อยู่ในช่วง 25-32! วิศวกรตั้งใจจะ
"เพิ่ม" การครอบคลุม แต่กลับ**เอา Subnet เดิมที่ต้องการอยู่แล้วออกไปจากการ Match โดยไม่รู้ตัว**
และก็ยังไม่ได้เพิ่ม `10.199.201.0/24` เข้าไปเลยด้วย (ลืม Entry ใหม่)

### อาการที่เห็น

```
CORE-SW1# show ip route ospf
      10.0.0.0/8 is variably subnetted, 5 subnets, 3 masks
O*E1 0.0.0.0/0 [110/1] via 10.10.254.1, 00:05:44, GigabitEthernet1/0/1
! -- 10.199.200.0/24 หายไปจากตารางทั้งหมด --
```

`10.199.200.0/24` ที่เคยอยู่มาตลอดตั้งแต่ Part 31 **หายไปจากตาราง OSPF ของ Campus ทั้งหมด**
(และ `10.199.201.0/24` ก็ไม่โผล่มาเหมือนเดิม เพราะไม่มี Entry ไหนอนุญาตมันอยู่แล้ว)

### กระบวนการวินิจฉัย

```
! Step 1: ยืนยันว่าฝั่ง Source (EIGRP) มี Route ทั้งสองจริง
DIST-SW3# show ip route eigrp | include 10.199.20
D    10.199.201.0/24 [90/156160] via 10.199.34.9, 00:10:02, GigabitEthernet1/0/23
D    10.199.200.0/24 [90/156160] via 10.199.34.9, 00:10:02, GigabitEthernet1/0/23

! มีทั้งคู่ฝั่ง Source แน่นอน -> ปัญหาอยู่ที่ Filter -> ตรวจ Prefix-list ด้วย Detail
DIST-SW3# show ip prefix-list detail LEGACY-SERVER
ip prefix-list LEGACY-SERVER:
   count: 1, range entries: 1, sequences: 5 - 5, refcount: 2
   seq 5 permit 10.199.200.0/24 ge 25 le 32 (hit count: 0, refcount: 1)
```

**`hit count: 0`** คือหลักฐานชี้ตรงที่สุด — Sequence นี้ไม่เคย Match Route ไหนเลยตั้งแต่ถูกสร้าง
(หรือแก้ไข) มา ยืนยันว่า Prefix-list เขียนผิดจนไม่ Match อะไรเลยจริงๆ ไม่ใช่ปัญหาที่จุดอื่น

(Site นี้ยังคง Combine Selective Redistribution ของ Step 306 เข้ากับ Tag Loop-guard ของ
Step 307 ไว้ด้วยกัน — Route-map `EIGRP-TO-OSPF-SELECTIVE` จึงมีทั้ง `match ip address
prefix-list LEGACY-SERVER` และ `set tag 90` อยู่ใน Sequence เดียวกัน ซึ่งเป็นรูปแบบที่พบได้จริง
ในหลาย Production Network ที่ต้องการทั้งกรอง Prefix และป้องกัน Loop ไปพร้อมกัน) **Hit Count = 0**
จาก Step 2 คือหลักฐานที่ชี้ตรงจุดที่สุดแล้วว่า Root Cause อยู่ที่ Prefix-list ไม่ใช่ Route-map

### Fix — แก้ Prefix-list ให้ครอบคลุมทั้ง Subnet เดิมและ Subnet ใหม่อย่างถูกต้อง

```
DIST-SW3(config)# no ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24 ge 25 le 32
DIST-SW3(config)# ip prefix-list LEGACY-SERVER seq 5 permit 10.199.200.0/24
DIST-SW3(config)# ip prefix-list LEGACY-SERVER seq 10 permit 10.199.201.0/24
```

(แยก Entry ชัดเจนสำหรับแต่ละ Subnet ที่**ต้องการจริง** ปลอดภัยกว่าการพยายามเขียน `ge`/`le`
ให้ครอบคลุมกว้างแบบเดารูปแบบ Subnet ในอนาคต — ตาม "หลักการทองคำ" ของ Part 31 Step 301 ที่ว่า
"Redistribute ให้น้อยที่สุดเท่าที่จำเป็น" การระบุ Prefix ตรงๆ ทีละเส้นชัดเจนกว่าเสมอ)

### Verify

```
DIST-SW3# show ip prefix-list detail LEGACY-SERVER
ip prefix-list LEGACY-SERVER:
   count: 2, sequences: 5 - 10
   seq 5 permit 10.199.200.0/24 (hit count: 1, refcount: 1)
   seq 10 permit 10.199.201.0/24 (hit count: 1, refcount: 1)

CORE-SW1# show ip route ospf | include 10.199.20
O E2    10.199.200.0/24 [110/20] via 10.255.30.1, 00:00:09, TenGigabitEthernet1/0/1
O E2    10.199.201.0/24 [110/20] via 10.255.30.1, 00:00:09, TenGigabitEthernet1/0/1
```

> **กับดักข้อสอบเรื่อง `ge`/`le` ที่ต้องจำ**: `permit <network>/<len>` โดยไม่มี `ge`/`le` เลย
> จะ Match **เฉพาะ Prefix ที่มี Mask ตรงกับ `<len>` เป๊ะเท่านั้น** (Exact Match) การเพิ่ม
> `ge <n>` คือ "อนุญาต Mask ตั้งแต่ n ขึ้นไป" — ถ้าอยากให้ Mask เดิม (`<len>`) ยัง Match ได้ด้วย
> ต้องตั้ง `ge` ให้**เท่ากับหรือน้อยกว่า** `<len>` เดิมเสมอ ไม่ใช่มากกว่า

---

## Step 586 — Scenario 5: BGP Full Table รั่วเข้า IGP เพราะ Route-map หลุดออกจากคำสั่ง Redistribute

### สถานการณ์

WAN-EDGE-1 มี Partner เชื่อมต่อพิเศษผ่าน ISP ที่ต้องการให้ Campus เห็น Prefix หนึ่งเพิ่มเติม
นอกจาก Default Route (ตาม Safe Pattern 2 ของ Part 31 Step 308) วิศวกรที่เร่งงานแก้ไข Route-map
`BGP-TO-OSPF-SAFE` แต่ระหว่างทดสอบ **ถอด Route-map ออกจากคำสั่ง `redistribute` ชั่วคราว**
เพื่อเช็คว่า BGP มี Route Partner จริงหรือไม่ (วิธี Debug ที่ผิด) แล้ว**ลืมเอากลับเข้าไป**:

```
! ===== Config ที่พังบน WAN-EDGE-1 =====
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# no redistribute bgp 65001 subnets route-map BGP-TO-OSPF-SAFE
WAN-EDGE-1(config-router)# redistribute bgp 65001 subnets
```

### อาการที่เห็น: Routing Table Campus บวมทันที + CPU พุ่ง

```
CORE-SW1# show ip route summary
IP routing table name is default (0x0)
IP routing table maximum-paths is 32
Route Source    Networks    Subnets     Replicates  Overhead    Memory (bytes)
static          2           0           0           144         320
connected       0           14          0           1008        2240
ospf 1          18          241855      0           17413560    386968000    <--- ระเบิด!
eigrp EIGRP-DEMO 0          8           0           576         1280
internal        22                                              5148
Total           42          241877                  17415288    387297668
```

```
CORE-SW1# show processes cpu sorted | exclude 0.00%  0.00%  0.00%
CPU utilization for five seconds: 97%/85%; one minute: 88%; five minutes: 71%
 PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process
 142    892341       48291      18475  61.23%  55.10%  40.02%   0 OSPF Router
```

Ospf 1 บวมจาก 7 Subnet (ตาม Part 31 Step 310 หลัง Safe Pattern) เป็น **241,855 Subnet**
เกือบทันที และ CPU Process `OSPF Router` พุ่งไปเกิน 60% ต่อเนื่อง — ตรงตามอาการที่ Part 31
Step 308 เตือนไว้ทุกประการ (SPF Recalculation ค้าง, Database ระเบิด)

### กระบวนการวินิจฉัย

```
! Step 1: ยืนยันจำนวน Prefix จริงในตาราง BGP ที่ WAN-EDGE-1
WAN-EDGE-1# show ip bgp summary | include 65000
203.0.113.1     4 65000   48210   48099  241890    0    0 1d02h      241855

! Neighbor ISP ส่ง Prefix มาจริง 241,855 เส้น (ไม่ใช่ปัญหาฝั่ง BGP) -> ตรวจ Redistribute ที่ ASBR
WAN-EDGE-1# show run | section router ospf
router ospf 1
 router-id 1.1.1.21
 default-information originate always metric 1 metric-type 1
 redistribute bgp 65001 subnets
```

**พบ Root Cause ทันที** — `redistribute bgp 65001 subnets` อยู่โดย**ไม่มี `route-map`
ต่อท้ายเลย** ต่างจาก Design เดิมใน Part 31 Step 308 (Safe Pattern 2) ที่ต้องมี
`route-map BGP-TO-OSPF-SAFE` เสมอ — เท่ากับ Redistribute **ทั้ง BGP Table** เข้า OSPF ตรงๆ
ตามที่ Part 31 เตือนไว้ว่า "ห้ามทำแบบนี้เด็ดขาด"

### Fix

```
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# no redistribute bgp 65001 subnets
WAN-EDGE-1(config-router)# redistribute bgp 65001 subnets route-map BGP-TO-OSPF-SAFE
```

> **ข้อควรระวังเพิ่มเติม**: หลังลบ `redistribute bgp ... subnets` (ไม่มี Route-map) แล้ว LSA
> Type-5 ที่เคย Flood ไปทั้ง Domain แล้ว**ไม่หายไปทันที** — ต้องรอ LSA Age ครบ `MaxAge` (3600
> วินาที) หรือบังคับ Clear ด้วย `clear ip ospf process` (Disruptive — ทำให้ Neighbor Reset
> ชั่วขณะ ควรทำนอกเวลา Peak เท่านั้น) เพื่อล้าง Database ให้เร็วขึ้นในกรณีเร่งด่วน

### Verify

```
CORE-SW1# clear ip ospf process
Reset ALL OSPF processes? [no]: yes

! รอ Neighbor Full ใหม่ทั้ง Domain (ปกติ < 1 นาทีสำหรับ Lab นี้)

CORE-SW1# show ip route summary
Route Source    Networks    Subnets
ospf 1          18          7            <--- กลับมาปกติ (Default Route + 6 Prefix ตาม Design)
```

```
CORE-SW1# show ip route ospf | include ^O
O*E1 0.0.0.0/0 [110/1] via 10.10.254.1, 00:00:44, GigabitEthernet1/0/1
O E2 10.199.200.0/24 [110/20] via 10.255.30.1, 00:12:03, TenGigabitEthernet1/0/1
O E2 10.199.201.0/24 [110/20] via 10.255.30.1, 00:12:03, TenGigabitEthernet1/0/1
O E1 198.51.100.0/24 [110/21] via 10.10.254.1, 00:00:44, GigabitEthernet1/0/1
```

CPU กลับสู่ระดับปกติ (< 5%) และ Routing Table Campus กลับมามีแค่ Prefix ที่จำเป็นจริง

> **บทเรียนสำคัญของ Scenario นี้**: **ห้ามถอด Route-map ออกจากคำสั่ง `redistribute` เพื่อ Debug
> เด็ดขาด** ในระบบที่ Source Protocol มี Prefix จำนวนมาก (BGP โดยเฉพาะ) — ให้ใช้
> `show ip bgp <route-map-filter-check>` หรือทดสอบ Route-map แยกด้วยคำสั่ง `show route-map`
> ก่อนเสมอ โดยไม่ต้องเอา Filter ออกจริงบน Production Config เลยแม้แต่วินาทีเดียว

---

## Step 587 — Scenario 6: Metric-type ไม่ตรงกัน (E1 ปน E2) ทำให้เลือก Path ผิดที่มี ASBR หลายจุด

### ทบทวนกฎการเลือก Path ของ OSPF External Route

OSPF จัดลำดับความสำคัญของ Route Type ก่อนเทียบ Cost เสมอ:

```
Intra-area (O) > Inter-area (O IA) > External Type 1 (O E1) > External Type 2 (O E2)
```

**E1 ชนะ E2 เสมอไม่ว่า Cost ตัวเลขจะเป็นเท่าไหร่ก็ตาม** — Type ต้องเทียบก่อน Cost เสมอ (คนละ
กฎกับ AD ที่เทียบ AD ก่อน Metric — Type ของ External ก็เป็นอีกชั้นก่อนถึง Metric เหมือนกัน)
ความแตกต่างของ Cost แต่ละ Type:

| Type | สูตร Cost | ตัวอย่าง |
|---|---|---|
| **E1** | Seed Metric + Internal OSPF Cost สะสมจาก Router ที่ถามไปถึง ASBR | `20 (seed) + 11 (internal cost ไป ASBR) = 31` |
| **E2** | Seed Metric อย่างเดียว (Fixed, ไม่บวก Internal Cost) | `20 (คงที่เสมอไม่ว่าไปกี่ Hop)` |

### สถานการณ์

Design เดิมของ Part 31 Step 303/307 ตั้งให้ทั้ง DIST-SW3 และ DIST-SW4 ใช้ `metric-type 2`
เหมือนกันทั้งคู่ (เพื่อให้ Cost เท่ากันเสมอและใช้ ECMP/Load-balance ได้ถ้า Cost ไปถึงทั้งสอง ASBR
เท่ากัน) แต่ระหว่างทำ Config Audit วิศวกรพิมพ์คำสั่งใหม่บน DIST-SW4 ผิดโดยยึด Syntax จาก
Protocol อื่นที่คุ้นเคยมากกว่า (ลืมว่า Default ของ Redistribution คือ E2 ไม่ใช่ E1):

```
! ===== Config ที่พังบน DIST-SW4 =====
DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# no redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
DIST-SW4(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 1 route-map EIGRP-TO-OSPF
```

(สังเกต `metric-type 1` — ผิดจาก Design เดิมที่ต้องเป็น `metric-type 2` ทั้ง DIST-SW3 และ
DIST-SW4 เหมือนกัน)

### อาการที่เห็น: Path ที่เลือกอ้อมกว่าที่ควรเป็น

```
CORE-SW1# show ip route 10.199.200.0
Routing entry for 10.199.200.0/24
  Known via "ospf 1", distance 110, metric 31, type extern 1
  Last update from 10.255.40.1 on TenGigabitEthernet1/0/2, 00:02:11 ago    <--- ผ่าน DIST-SW4 (ไกลกว่า)
```

Path ที่ถูกเลือกวิ่งผ่าน `TenGigabitEthernet1/0/2` (ไป DIST-SW4 ผ่าน CORE-SW2 — อ้อมกว่า) ทั้งที่
Path ผ่าน DIST-SW3 (`TenGigabitEthernet1/0/1`) มี Cost ต่ำกว่าและใกล้กว่าโดย Topology จริง

### กระบวนการวินิจฉัย

```
! Step 1: ดู OSPF Database เทียบทั้ง 2 LSA จาก 2 ASBR พร้อมกัน
CORE-SW1# show ip ospf database external 10.199.200.0

            OSPF Router with ID (1.1.1.1) (Process ID 1)
                Type-5 AS External Link States

  LS age: 320
  Advertising Router: 1.1.1.13
        Metric Type: 2 (Larger is better)
        Metric: 20
        External Route Tag: 90

  LS age: 88
  Advertising Router: 1.1.1.14
        Metric Type: 1 (Comparable directly to link state metric)
        Metric: 20
        External Route Tag: 90
```

**พบ Root Cause ทันที**: ASBR ทั้งสองตัว Advertise Prefix เดียวกัน (`10.199.200.0/24`, Tag 90
เท่ากัน — ไม่ใช่ปัญหา Loop) แต่ `Metric Type` **ไม่ตรงกัน**: `1.1.1.13` (DIST-SW3) = Type 2,
`1.1.1.14` (DIST-SW4) = Type 1

```
! Step 2: คำนวณ Cost จริงของทั้งสอง Path เพื่อยืนยันว่า E1 ชนะแม้ Cost จะแพงกว่า
! Path ผ่าน DIST-SW3 (E2): Cost คงที่ = 20 เท่านั้น ไม่ว่า Internal Cost จะเป็นเท่าไหร่
! Path ผ่าน DIST-SW4 (E1): Cost = 20 (seed) + Internal Cost สะสมจาก CORE-SW1 ไปยัง DIST-SW4
! รวม Internal Cost ผ่าน CORE-SW2 (Po1) + Area 2 ตาม show ip ospf interface ทุก Hop = 11
! -> E1 Total Cost = 20 + 11 = 31 (ตรงกับ metric 31 ที่เห็นใน show ip route จริง)
```

ผลคือ **E1 Cost = 31 แพงกว่า E2 Cost = 20 ของ DIST-SW3 อย่างชัดเจน** แต่เพราะกฎ OSPF เทียบ
**Type ก่อน Cost เสมอ** (E1 มีสถานะดีกว่า E2 โดย Design ของ Protocol เอง ไม่ใช่บั๊ก) Router จึง
เลือก Path ผ่าน DIST-SW4 (E1, Cost 31) แทนที่จะเลือก Path ผ่าน DIST-SW3 (E2, Cost 20) ที่ถูกกว่า
และใกล้กว่าจริง — Root Cause คือ Metric-type ที่ไม่ตรงกันระหว่าง 2 ASBR ทำให้กฎ "Type ก่อน
Cost" ไปสร้าง Suboptimal Path โดยไม่ตั้งใจ

### Fix

```
DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# no redistribute eigrp 100 subnets metric 20 metric-type 1 route-map EIGRP-TO-OSPF
DIST-SW4(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
```

### Verify

```
CORE-SW1# show ip route 10.199.200.0
Routing entry for 10.199.200.0/24
  Known via "ospf 1", distance 110, metric 20, type extern 2
  Last update from 10.255.10.2 on TenGigabitEthernet1/0/1, 00:00:05 ago    <--- กลับมาผ่าน DIST-SW3
```

> **ข้อสังเกตสำหรับข้อสอบ**: คำถามแนว "ทำไม Router เลือก Path ที่ Cost แพงกว่า" มักซ่อนคำตอบไว้ที่
> **Type ของ External Route ไม่ใช่ตัวเลข Cost** ให้เช็ค `show ip route <prefix>` ดูฟิลด์
> `type extern 1` vs `type extern 2` เป็นอันดับแรกก่อนไปเทียบตัวเลข Metric เสมอ — ถ้า Design
> ต้องการให้ ASBR หลายจุด Load-balance หรือเทียบ Cost กันตรงๆ ได้ ต้องบังคับ `metric-type` ให้
> **เหมือนกันทุกจุดเสมอ**

---

## Step 588 — Scenario 7: `default-metric` แบบ Global ดันไปกระทบ Redistribute Statement ที่ไม่ควรถูกกระทบ

### สถานการณ์

ทีม Operations ต้องการทดสอบ Backup Path ชั่วคราวที่ DIST-SW3 โดย Redistribute Static Route
ตัวหนึ่งที่ตั้งไว้สำหรับ Test Lab เข้า EIGRP ด้วย (ไม่เกี่ยวกับ Design เดิมของ Part 31 เลย) —
วิศวกรเข้าใจผิดคิดว่าต้องมี `default-metric` แบบ Global ไว้ก่อนเสมอ (เห็นมาจาก Protocol อื่นที่
เคยทำ) จึงเพิ่ม `default-metric` ไว้ที่ระดับ `topology base` แทนที่จะใส่ `metric` เฉพาะจุดที่
`redistribute ospf 1` เพียงอย่างเดียวตามที่ Part 31 Step 304 ทำไว้เดิม:

```
! ===== Config ที่พังบน DIST-SW3 =====
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# no redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
DIST-SW3(config-router-af-topology)# default-metric 1000000 100 255 1 1500
DIST-SW3(config-router-af-topology)# redistribute ospf 1 route-map OSPF-TO-EIGRP
DIST-SW3(config-router-af-topology)# redistribute static
```

(`redistribute static` ถูกเพิ่มเข้ามาใหม่สำหรับ Test Route ตัวหนึ่งที่ DIST-SW3 มีอยู่
— `ip route 172.16.99.0 255.255.255.0 Null0` ที่เคยใช้ Test Aggregation เฉยๆ ไม่ได้ตั้งใจให้
ออกไป EIGRP เลย)

### ทำไม `default-metric` ทำให้เกิดปัญหาที่ไม่ตั้งใจ

`default-metric` มีผล**กับทุก `redistribute` Statement ที่ไม่ได้ระบุ `metric` ของตัวเอง**
(ตามหลักการ Part 31 Step 302) — ตอนนี้มีทั้ง `redistribute ospf 1` **และ**
`redistribute static` อยู่ในบล็อกเดียวกัน ทั้งคู่ไม่มี `metric` เฉพาะของตัวเอง ทำให้ทั้งคู่ดึง
Seed Metric จาก `default-metric` เดียวกันไปหมด — ผลคือ **Static Test Route ที่ไม่ควรถูก
Redistribute ออกไปเลย กลับถูกฉีดเข้า EIGRP ไปด้วยความ "สำเร็จ" อย่างเงียบๆ** (ถ้าไม่มี
`default-metric` มันจะถูกปฏิเสธเงียบๆ เหมือน Step 582 Case A — แต่รอบนี้กลับ "สำเร็จ" เพราะมี
Seed Metric ให้ใช้ร่วมกันโดยไม่ตั้งใจ)

### อาการที่เห็น

```
EIGRP-R1# show ip route eigrp | include 172.16
D EX    172.16.99.0/24 [170/28160256] via 10.199.34.1, 00:03:40, GigabitEthernet0/0
```

Legacy Site เห็น Prefix `172.16.99.0/24` ที่**ไม่มีความหมายอะไรกับ Legacy Site เลย** (เป็น
Null0 Aggregate Route ภายใน Campus สำหรับ Test เฉยๆ) — ถ้ามี Traffic หลุดไปหา Prefix นี้จาก
Legacy Site จะถูกส่งเข้า Null0 ที่ DIST-SW3 (Blackhole) ทำให้เกิด "Silent Packet Loss" ที่
Debug ยากเพราะไม่มี Error ใดๆ Config ก็ดู "ทำงานถูกต้อง" ทุกอย่าง

### กระบวนการวินิจฉัย

```
! Step 1: ตรวจ show ip protocols เพื่อดูว่า Redistribute Source มีตัวไหนที่ไม่ควรมี
DIST-SW3# show ip protocols
Routing Protocol is "eigrp EIGRP-DEMO"
  Redistributing: eigrp EIGRP-DEMO
    Redistributing External Routes from,
      ospf 1 (route-map OSPF-TO-EIGRP)
      static

! พบ static โผล่มาเป็น Source ที่ 2 (Design เดิมไม่มี) -> เจาะ Running-config ดูว่า Metric มาจากไหน
DIST-SW3# show run | section router eigrp
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  topology base
   default-metric 1000000 100 255 1 1500
   redistribute ospf 1 route-map OSPF-TO-EIGRP
   redistribute static
  exit-af-topology
```

**Root Cause ยืนยัน**: `default-metric` ถูกตั้งไว้ระดับ Global ของ Topology และ
`redistribute static` ไม่มี `metric` ของตัวเอง — ทั้งสอง Statement (`ospf 1` และ `static`)
ใช้ `default-metric` ตัวเดียวกันร่วมกันโดยไม่ตั้งใจ

### Fix — ย้ายกลับไปใช้ `metric` เฉพาะจุดตาม Design เดิม และลบ `redistribute static` ที่ไม่ต้องการ

```
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# no redistribute static
DIST-SW3(config-router-af-topology)# no default-metric 1000000 100 255 1 1500
DIST-SW3(config-router-af-topology)# no redistribute ospf 1 route-map OSPF-TO-EIGRP
DIST-SW3(config-router-af-topology)# redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
```

> **ถ้าจำเป็นต้อง Redistribute Static จริง (กรณีอื่นในอนาคต)** ให้ตั้ง `metric` เฉพาะจุดของ
> Statement นั้นเสมอ (`redistribute static metric ... route-map <filter-เฉพาะ>`) **ไม่ใช้
> `default-metric` แบบ Global** เพื่อไม่ให้กระทบ Source อื่นที่ Redistribute อยู่ร่วมกันโดย
> ไม่ตั้งใจ — Local `metric` ที่ระบุเฉพาะ Statement Override `default-metric` เสมอตามหลักการ
> Part 31 Step 302

### Verify

```
DIST-SW3# show ip protocols
Routing Protocol is "eigrp EIGRP-DEMO"
  Redistributing: eigrp EIGRP-DEMO
    Redistributing External Routes from,
      ospf 1 (route-map OSPF-TO-EIGRP)

EIGRP-R1# show ip route eigrp | include 172.16
% (ไม่พบ — Prefix ที่ไม่ควรออกไปหายไปแล้ว)
```

---

## Step 589 — Scenario 8: Asymmetric Routing จาก Redistribution ที่มีอยู่แค่จุดเดียว (ควรมี 2 จุด)

### สถานการณ์

Design เดิมของ Part 31 (Step 307/310) ตั้งใจให้ **ทั้ง DIST-SW3 และ DIST-SW4 ทำ Mutual
Redistribution OSPF↔EIGRP เหมือนกันทั้งคู่** เพื่อให้มี 2 เส้นทางเข้า-ออก Legacy Site
(Redundancy) — ระหว่างทำ Maintenance ที่ DIST-SW4 (เปลี่ยน Line Card) วิศวกรปิดคำสั่ง
Redistribute ทิศทางเดียว (EIGRP→OSPF) ไว้ชั่วคราวเพื่อความปลอดภัยระหว่างงาน แล้ว**ลืมเปิดกลับ**
เมื่องานเสร็จ (ทิศทาง OSPF→EIGRP ยังเปิดอยู่ตามปกติ):

```
! ===== Config ที่พังบน DIST-SW4 (ทิศทาง EIGRP->OSPF ถูกปิดไว้ ไม่ได้เปิดกลับ) =====
DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# no redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
```

(คำสั่ง `redistribute ospf 1 metric ... route-map OSPF-TO-EIGRP` ใต้ `router eigrp` ยังคงอยู่
ปกติ — ปิดไปแค่ทิศทางเดียวเท่านั้น)

### อาการที่เห็น: Traceroute ไป-กลับ "คนละทาง"

```
PC1(10.10.10.55)> traceroute 10.199.200.10
  1  10.10.10.1       <- DIST-SW1 (VLAN10 Gateway)
  2  10.255.10.1       <- CORE-SW1
  3  10.255.30.2       <- DIST-SW3 (ทาง Area2 ปกติ ตาม OSPF Cost ต่ำสุด)
  4  10.199.34.1        <- EIGRP-R1
  5  10.199.200.10      <- Legacy Server
```

```
PC-Legacy(10.199.200.10)> traceroute 10.10.10.55
  1  10.199.200.254     <- EIGRP-R2 (Gateway ของ Legacy Server LAN)
  2  10.199.34.9         <- DIST-SW4 (คนละเส้นทางกับตอนไป!)
  3  10.255.40.1          <- CORE-SW2
  4  10.10.10.1            <- DIST-SW1
  5  10.10.10.55
```

**เส้นทางไปและกลับไม่ใช่เส้นทางเดียวกัน** (Asymmetric Routing) — ไปทาง DIST-SW3, กลับทาง
DIST-SW4 ทั้งที่การเชื่อมต่อพื้นฐานยังทำงานได้ (ยัง Ping ผ่าน) แต่สร้างปัญหาแอบแฝงคือ Stateful
Device ระหว่างทาง (เช่น Firewall ถ้ามีในอนาคต) อาจ Drop Return Traffic เพราะไม่เห็น Session
ตอน Forward ผ่านมาก่อน และการ Troubleshoot อื่นๆ (Latency, Packet Loss) จะสับสนเพราะดู
Interface ผิดฝั่งได้ง่าย

### กระบวนการวินิจฉัย

```
! Step 1: ตรวจ show ip protocols ทั้ง DIST-SW3 และ DIST-SW4 เทียบกัน
DIST-SW3# show ip protocols | begin ospf
Routing Protocol is "ospf 1"
  Redistributing External Routes from,
    eigrp 100, includes subnets in redistribution

DIST-SW4# show ip protocols | begin ospf
Routing Protocol is "ospf 1"
  ! -- ไม่มีบรรทัด "Redistributing External Routes from, eigrp 100" เลย --

! DIST-SW3 ยังมี Redistribute EIGRP->OSPF ปกติ แต่ DIST-SW4 ไม่มีเลย -> เช็คทิศตรงข้ามที่ DIST-SW4
DIST-SW4# show ip protocols | begin eigrp
Routing Protocol is "eigrp EIGRP-DEMO"
  Redistributing: eigrp EIGRP-DEMO
    Redistributing External Routes from,
      ospf 1 (route-map OSPF-TO-EIGRP)
```

**พบ Root Cause ทันที** — ทิศ EIGRP→OSPF หายไปจาก DIST-SW4 จุดเดียว ในขณะที่ทิศ OSPF→EIGRP
ที่จุดเดียวกันยังทำงานปกติ — ทำให้ EIGRP-R2 ยังเห็น Campus ผ่าน DIST-SW4 (Metric ใกล้กว่า) และ
เลือกออกทางนั้นเสมอเมื่อ Legacy Site เริ่ม Traffic แต่ Campus ไม่รู้จัก Legacy Site ผ่าน DIST-SW4
อีกต่อไป จึงต้องอ้อมไปใช้ DIST-SW3 เสมอเมื่อ Campus เป็นฝ่ายเริ่ม Traffic — เกิด Asymmetric Path

| ทิศทาง Traffic | เส้นทางที่ใช้จริง | เหตุผล |
|---|---|---|
| Campus → Legacy | ผ่าน DIST-SW3 เท่านั้น | ASBR เดียวที่ยัง Redistribute OSPF→EIGRP ให้ Legacy Site เห็น Campus |
| Legacy → Campus | ผ่าน DIST-SW4 (Metric ใกล้กว่าจาก R2) | EIGRP-R2 ยังเห็น Campus ผ่าน DIST-SW4 (ทิศ OSPF→EIGRP ยังทำงานอยู่) |

### Fix

```
DIST-SW4(config)# router ospf 1
DIST-SW4(config-router)# redistribute eigrp 100 subnets metric 20 metric-type 2 route-map EIGRP-TO-OSPF
```

### Verify

```
DIST-SW4# show ip protocols | begin ospf
Routing Protocol is "ospf 1"
  Redistributing External Routes from,
    eigrp 100, includes subnets in redistribution

PC-Legacy> traceroute 10.10.10.55
  1  10.199.200.254
  2  10.199.34.9        <- DIST-SW4
  3  10.255.40.1
  4  10.10.10.1
  5  10.10.10.55
```

(สิ่งสำคัญที่สุดที่ต้อง Verify คือ**ทั้งสองทิศทาง Redistribute กลับมาสมบูรณ์ทั้ง 2 จุดเหมือนกัน
แล้ว** ตาม Design — ส่วน Path จะสมมาตรสนิทหรือไม่ขึ้นกับ Cost จริงของ Topology อีกที)

> **กรณีตรงข้าม (ควรมีจุดเดียว แต่กลับมี 2 จุด)**: ถ้า Design ตั้งใจให้ Redistribute เกิดที่
> จุดเดียวเท่านั้น แต่มีคนเผลอเพิ่ม Statement ที่จุดที่สองเข้ามาโดยไม่ตั้งใจ (เช่น Copy Template
> ผิด) จะเกิดปัญหาย้อนกลับคือ **Loop แบบ Step 583** ทันทีถ้าไม่มี Tag ป้องกันไว้ล่วงหน้า —
> ต้องตรวจสอบเสมอว่า "จำนวนจุด Redistribute ที่มีอยู่จริงตรงกับที่ Design ต้องการหรือไม่"

---

## Step 590 — Lab เต็มรูปแบบ: 2 บั๊กซ้อนกัน — Tag Filter หาย + Seed Metric หาย ในเวลาเดียวกัน

### สถานการณ์เริ่มต้น

หลัง Maintenance Window ใหญ่ในคืนวันเสาร์ (Upgrade IOS-XE บน DIST-SW3 และ DIST-SW4 พร้อมกัน)
ทีม Network พบว่าเช้าวันจันทร์ Helpdesk ได้รับ Ticket 2 เรื่องพร้อมกัน:

1. "Legacy Server บางเครื่องเข้าถึงไม่ได้จาก Campus บางครั้ง เดี๋ยวได้เดี๋ยวไม่ได้"
2. "EIGRP-R1 บ่นใน Log ว่า Neighbor Flap และ CPU Router สูงกว่าปกติตลอดคืน"

ทั้งสอง Ticket มีสาเหตุมาจาก**การพังพร้อมกัน 2 จุดคนละแบบ** ที่เกิดขึ้นระหว่าง Restore Config
หลัง Upgrade (บาง Section ของ Config ถูก Restore จาก Backup คนละเวอร์ชันกัน):

```
! ===== บั๊กที่ 1: DIST-SW4 — Route-map Loop-guard ขาด Sequence (เหมือน Step 583) =====
DIST-SW4# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, permit, sequence 20
  Match clauses:
  Set clauses:
    tag 110
  Policy routing matches: 4102 packets, ...

! ===== บั๊กที่ 2: DIST-SW3 — Seed Metric ขาดหายตอน Redistribute OSPF->EIGRP (เหมือน Step 582) =====
DIST-SW3# show run | section router eigrp
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 route-map OSPF-TO-EIGRP
  exit-af-topology
```

### อาการรวมที่เห็นจริง

```
EIGRP-R1# show ip route eigrp | include 10.10
% (ไม่พบ Prefix ของ Campus เลยแม้แต่เส้นเดียว — ตรงกับ Ticket 1: "เข้าไม่ได้")

EIGRP-R1# show ip eigrp topology 10.199.200.0/24
EIGRP-IPv4 (AS 100): Topology entry for 10.199.200.0/24
  State is Passive, 1 Successor(s), FD is 156160     <-- Poll ซ้ำแล้วค่าเปลี่ยนไม่หยุด (ตรงกับ Ticket 2)

CORE-SW1# show processes cpu sorted | exclude 0.00%  0.00%  0.00%
CPU utilization for five seconds: 42%/30%; one minute: 38%; five minutes: 35%
 PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process
 142    412200       28291      14567  22.10%  19.87%  15.02%   0 OSPF Router
```

สังเกตว่าอาการทั้งสองเรื่องดู "ไม่เกี่ยวกัน" ในตอนแรก (Ticket 1 เป็นเรื่อง Route หาย, Ticket 2
เป็นเรื่อง Flap/CPU) — นี่คือเหตุผลที่ Redistribution Troubleshooting ต้อง**แยกวิเคราะห์แต่ละ
จุด ASBR อย่างเป็นระบบ** ไม่ใช่พยายามหา "สาเหตุเดียว" ที่อธิบายทุกอาการ

### กระบวนการวินิจฉัยแบบเป็นระบบ (ใช้ Flowchart จาก Step 581)

**Ticket 1 (OSPF→EIGRP)**: ยืนยันฝั่ง Source (OSPF) มี Route จริงก่อน แล้วไล่ตรวจ ASBR
DIST-SW3:

```
DIST-SW3# show ip route ospf | include 10.10.10.0
O    10.10.10.0/24 [110/11] via 10.255.30.1, 00:20:04, TenGigabitEthernet1/0/1

! Source มี Route แน่นอน -> ปัญหาอยู่ที่ตัว Redistribution เอง ตรวจ show ip protocols ต่อ
DIST-SW3# show ip protocols | begin eigrp
Routing Protocol is "eigrp EIGRP-DEMO"
  Redistributing: eigrp EIGRP-DEMO
    Redistributing External Routes from,
      ospf 1 (route-map OSPF-TO-EIGRP)

! Statement มีอยู่จริงแต่ไม่มีบรรทัดบอก Metric -> เจาะ Running-config
DIST-SW3# show run | section router eigrp
router eigrp EIGRP-DEMO
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 route-map OSPF-TO-EIGRP
```

**ไม่มี `metric` เลย** — ยืนยัน Root Cause ของ Ticket 1: Seed Metric หายไปจาก Config ตอน
Restore Backup (เหมือน Step 582 Case A เป๊ะ)

**Ticket 2 (Loop จาก Tag Filter)**: ตรวจ Route-map Loop-guard ที่ DIST-SW4 และยืนยันด้วย
OSPF LSA Sequence Number:

```
DIST-SW4# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, permit, sequence 20
  Match clauses:
  Set clauses:
    tag 110
  Policy routing matches: 4102 packets, ...

! ไม่มี Sequence deny 10 match tag 90 เลย -> ยืนยันด้วย LSA Sequence Number ที่ CORE-SW2
CORE-SW2# show ip ospf database external 10.10.20.0
  Advertising Router: 1.1.1.14
  LS Seq Number: 80000037
! -- Poll ซ้ำอีก 3 นาทีต่อมา --
CORE-SW2# show ip ospf database external 10.10.20.0
  Advertising Router: 1.1.1.14
  LS Seq Number: 8000004C     <--- วิ่งขึ้นเร็วมาก ยืนยัน Loop จริง
```

**ไม่มี Sequence `deny 10 match tag 90` เลย** — ยืนยัน Root Cause ของ Ticket 2: Loop-guard
หายไปจาก DIST-SW4 (เหมือน Step 583 เป๊ะ) ทำให้ Route ที่มี Tag 90 (มาจากฝั่ง EIGRP ผ่าน
DIST-SW3 อยู่แล้ว) วิ่งวนกลับเข้า EIGRP อีกรอบผ่าน DIST-SW4 สร้าง Flap และภาระ CPU เพิ่ม

### สรุป Root Cause ทั้ง 2 จุด

| # | Router | บั๊ก | เกี่ยวข้องกับ Ticket |
|---|---|---|---|
| 1 | DIST-SW3 | `redistribute ospf 1 route-map OSPF-TO-EIGRP` ขาด `metric 1000000 100 255 1 1500` | Ticket 1 — Legacy Server เข้าไม่ได้จาก Campus (Route ไม่ถูกฉีดเข้า EIGRP เลย) |
| 2 | DIST-SW4 | Route-map `OSPF-TO-EIGRP` ขาด `deny 10 match tag 90` | Ticket 2 — Neighbor Flap/CPU สูง (Loop จาก Route วิ่งวนผ่าน Redistribution) |

### Fix ทั้ง 2 จุดพร้อมกัน

```
! ===== Fix บั๊กที่ 1 บน DIST-SW3 =====
DIST-SW3(config)# router eigrp EIGRP-DEMO
DIST-SW3(config-router)# address-family ipv4 unicast autonomous-system 100
DIST-SW3(config-router-af)# topology base
DIST-SW3(config-router-af-topology)# no redistribute ospf 1 route-map OSPF-TO-EIGRP
DIST-SW3(config-router-af-topology)# redistribute ospf 1 metric 1000000 100 255 1 1500 route-map OSPF-TO-EIGRP
DIST-SW3(config-router-af-topology)# exit-af-topology
DIST-SW3(config-router-af)# exit-address-family
```

```
! ===== Fix บั๊กที่ 2 บน DIST-SW4 =====
DIST-SW4(config)# route-map OSPF-TO-EIGRP deny 10
DIST-SW4(config-route-map)# match tag 90
DIST-SW4(config-route-map)# exit
```

> **ลำดับการ Fix สำคัญ**: แก้บั๊กที่ 2 (Loop-guard) ก่อนหรือพร้อมกับบั๊กที่ 1 เสมอ — ถ้าแก้บั๊ก
> ที่ 1 (เปิด Seed Metric กลับมา) ก่อนโดยยังไม่แก้ Loop-guard ที่ DIST-SW4 จะทำให้ Route ที่
> เพิ่งฉีดกลับเข้า EIGRP จาก DIST-SW3 ใหม่ **มีโอกาสวิ่งวนผ่าน DIST-SW4 ทันทีที่เปิดกลับมา**
> (เพราะ Loop-guard ที่นั่นยังพังอยู่) ทำให้ปัญหา CPU/Flap ของ Ticket 2 แย่ลงชั่วขณะก่อนจะดีขึ้น
> — ควร Fix ทั้ง 2 จุดในหน้าต่าง Maintenance เดียวกันเสมอเพื่อป้องกันผลกระทบระหว่างช่วง Fix

### Verification เต็มรูปแบบหลัง Fix

```
! Verify 1: Ticket 1 - Legacy Server เข้าถึงได้จาก Campus แล้ว
EIGRP-R1# show ip route eigrp | include 10.10
D EX    10.10.10.0/24 [170/28160256] via 10.199.34.1, 00:01:12, GigabitEthernet0/0
D EX    10.10.20.0/24 [170/28160256] via 10.199.34.1, 00:01:12, GigabitEthernet0/0
D EX    10.10.30.0/24 [170/28160256] via 10.199.34.1, 00:01:12, GigabitEthernet0/0

! Verify 2: Ticket 2 - LSA Sequence Number คงที่ไม่วิ่งขึ้นอีก (รอ 5 นาทีแล้ว Poll ซ้ำ)
CORE-SW2# show ip ospf database external 10.10.20.0
  Advertising Router: 1.1.1.14
  LS Seq Number: 80000050
! -- รอ 5 นาที Poll ซ้ำ --
CORE-SW2# show ip ospf database external 10.10.20.0
  Advertising Router: 1.1.1.14
  LS Seq Number: 80000050     <--- ค่าเดิม ไม่โตอีก ยืนยัน Loop หมดแล้ว

! Verify 3: CPU กลับสู่ระดับปกติ
CORE-SW1# show processes cpu sorted | exclude 0.00%  0.00%  0.00%
CPU utilization for five seconds: 3%/1%; one minute: 4%; five minutes: 5%
 PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process
 142    412900       28299       1023   0.50%   0.61%   0.55%   0 OSPF Router

! Verify 4: Route-map Loop-guard เหมือนกันทั้ง DIST-SW3/DIST-SW4 แล้ว
DIST-SW3# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, deny, sequence 10
  Match clauses:
    tag 90
route-map OSPF-TO-EIGRP, permit, sequence 20
  Set clauses:
    tag 110
DIST-SW4# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, deny, sequence 10
  Match clauses:
    tag 90
route-map OSPF-TO-EIGRP, permit, sequence 20
  Set clauses:
    tag 110

! Verify 5: Traceroute End-to-End ปกติ ไม่มี Loop
PC1(10.10.10.55)> traceroute 10.199.200.10
  1  10.10.10.1
  2  10.255.10.1
  3  10.255.30.2       <- DIST-SW3
  4  10.199.34.1
  5  10.199.200.10
Trace complete.
```

### บทเรียนจาก Lab นี้

| หลักการ | เหตุผล |
|---|---|
| **Backup/Restore Config ต้อง Verify ทุกครั้งหลัง Maintenance ไม่ใช่แค่เช็คว่า Boot ขึ้น** | บั๊กทั้ง 2 จุดเกิดจาก Restore Config จาก Backup คนละเวอร์ชันกัน ซึ่งเป็นสาเหตุ Production Outage จริงที่พบบ่อยมาก |
| **อาการที่ดู "ไม่เกี่ยวกัน" อาจมี Root Cause คนละจุดแต่เกิดพร้อมกันได้จริง** | อย่าพยายามหาสาเหตุเดียวที่อธิบายทุก Ticket เสมอไป — แยกวิเคราะห์แต่ละ ASBR ตาม Flowchart Step 581 อย่างเป็นระบบ |
| **Redistribution ที่ทำงานถูกต้องต้อง Verify ทั้ง 2 มุม**: (1) Route ถูกฉีดเข้าจริงหรือไม่ (Seed Metric) (2) Route ไม่วิ่งวนกลับ (Tag/Loop-guard) | ทั้งสองมุมเป็นอิสระจากกัน — ถูกมุมหนึ่งไม่ได้แปลว่าอีกมุมถูกด้วย ต้องเช็คคู่กันเสมอทุกจุด ASBR |

---

## แบบฝึกหัดทวนความเข้าใจ Part 59

1. เมื่อ Router ปลายทางไม่เห็น Route ที่ถูก Redistribute เลย ตาม Methodology ของ Step 581 ควร
   เริ่มตรวจสอบที่จุดใดก่อน และทำไมถึงไม่ควรเริ่มแก้ที่ Router ปลายทางที่เห็นอาการทันที?
2. อธิบายว่าทำไมการลืม Seed Metric ตอน Redistribute เข้า **EIGRP** ทำให้ Route หายไปแบบไม่มี
   Error แต่การลืม Seed Metric ตอน Redistribute เข้า **OSPF** กลับไม่ทำให้ Route หายไปแต่อาจ
   ได้ Metric ที่ไม่ตรงกับที่ Design ตั้งใจไว้แทน?
3. ในกรณี Routing Loop จาก Route Tag Filter หายไปที่ ASBR จุดเดียว (Step 583) จะใช้คำสั่งใด
   ยืนยัน Root Cause ได้เร็วที่สุด และเพราะเหตุใด `show route-map` ถึงเป็นเครื่องมือสำคัญใน
   Scenario นี้?
4. เพราะเหตุใด OSPF External Type 1 (E1) จึงถูกเลือกเป็น Best Path เสมอเมื่อเทียบกับ External
   Type 2 (E2) ของ Prefix เดียวกัน แม้ Cost ตัวเลขของ E1 จะแพงกว่า E2 ก็ตาม?
5. อธิบายว่า Asymmetric Routing ใน Step 589 เกิดขึ้นได้อย่างไรจากการที่ Redistribution มีอยู่
   แค่ทิศทางเดียวที่ ASBR จุดหนึ่ง ทั้งที่อีกทิศทางยังทำงานปกติ?

**เฉลย:**

1. ควรเริ่มตรวจสอบที่ **ASBR** ก่อนเสมอ (จุดที่ Config ทั้ง 2 Protocol อยู่ด้วยกัน) โดยไล่ตาม
   Flowchart: ยืนยันก่อนว่าฝั่ง Source Protocol มี Route จริงหรือไม่ (`show ip route <source>`)
   จากนั้นตรวจ `show ip protocols` และ `show run | section router <dest>` ว่า Redistribute
   Statement/Seed Metric/Route-map ถูกต้องหรือไม่ — ไม่ควรแก้ที่ Router ปลายทางเพราะนั่นเป็นแค่
   ที่ "เห็นอาการ" ไม่ใช่ที่ "เกิดสาเหตุ" การเพิ่ม Static Route ปะที่ปลายทางจะสร้างปัญหา AD/Loop
   ซ้อนเพิ่มขึ้นอีกโดยไม่แก้ปัญหาที่ต้นตอจริง
2. เพราะ **EIGRP ไม่มี Default Seed Metric ใดๆ ให้ใช้แทน** ถ้าไม่ระบุ `metric`/`default-metric`
   Route จะถูกปฏิเสธเงียบๆ ตั้งแต่ต้นไม่ถูกติดตั้งเข้า Topology Table เลย ในขณะที่ **OSPF มี
   Default Seed Metric = 20** ให้ใช้แทนเสมอเมื่อไม่ได้ระบุ (ยกเว้น BGP ที่ Default = 1) ทำให้
   Route ยังถูก Redistribute เข้าไปได้ปกติ แต่ค่า Metric ที่ได้อาจไม่ตรงกับที่ Design ต้องการ
   จริง (เช่นต้องการให้แพงกว่า Path อื่นแต่ลืมตั้ง กลายเป็นค่า Default ที่ทำให้ถูกเลือกเป็น
   Best Path แทน)
3. ใช้ `show route-map <name>` ดูว่า Sequence ที่เป็น `deny` พร้อม `match tag <value>` (ตัว
   Loop-guard) หายไปหรือไม่ และดูตัวเลข `Policy routing matches` ของ Sequence `permit` ที่วิ่ง
   สูงผิดปกติ (ยืนยันว่า Route ที่ควรถูก Deny กลับผ่านเข้ามาที่ Sequence Permit แทน) — สำคัญ
   เพราะ `show route-map` เป็นจุดเดียวที่แสดงทั้ง**โครงสร้าง Sequence จริงที่ใช้งานอยู่** และ
   **สถิติการ Match จริง** พร้อมกัน ทำให้ยืนยัน Root Cause ได้โดยไม่ต้องเดา
4. เพราะ OSPF จัดลำดับ Route Type ก่อนเทียบ Cost เสมอ ตามลำดับ Intra-area > Inter-area >
   External Type 1 > External Type 2 — E1 มีสถานะดีกว่า E2 โดย Design ของ Protocol เอง
   (ไม่ใช่ตัดสินจากตัวเลข Cost) ดังนั้นถึง E1 จะมี Cost สะสม (Seed Metric + Internal OSPF
   Cost) แพงกว่า E2 (Seed Metric คงที่อย่างเดียว) ระบบก็ยังเลือก E1 เป็น Best Path เสมอ ทำให้
   ถ้า ASBR หลายจุด Redistribute Prefix เดียวกันด้วย `metric-type` ต่างกัน อาจได้ Suboptimal
   Path โดยไม่ตั้งใจ
5. เกิดขึ้นเพราะ Redistribution ระหว่าง OSPF↔EIGRP เป็นคนละคำสั่งกันคนละทิศทางเสมอ — ถ้าทิศทาง
   หนึ่ง (เช่น EIGRP→OSPF) ถูกปิดไปที่ ASBR จุดหนึ่ง แต่อีกทิศทาง (OSPF→EIGRP) ที่จุดเดียวกันยัง
   เปิดอยู่ปกติ Legacy Site จะยังมองเห็น Campus ผ่าน ASBR จุดนั้นได้ (เพราะทิศทาง OSPF→EIGRP
   ยังทำงาน) และเลือกเป็นเส้นทางออกตาม Metric ของตัวเอง แต่ Campus จะไม่รู้จัก Legacy Site ผ่าน
   ASBR จุดนั้นอีกต่อไป (เพราะทิศทาง EIGRP→OSPF ถูกปิด) จึงต้องใช้ ASBR อีกจุดแทนเสมอ ทำให้
   เส้นทางไปและเส้นทางกลับกลายเป็นคนละเส้นทางกัน (Asymmetric)

---

## สรุป Part 59

Part นี้ต่อยอดจาก Lab Mutual Redistribution OSPF↔EIGRP และ Controlled Redistribution BGP↔OSPF
ของ [Part 31](part-031-route-redistribution.md) มาสร้างเป็น**คู่มือ Troubleshoot 8 Fault
Pattern ที่พบบ่อยที่สุดในโลกจริงและในข้อสอบ ENARSI**: เริ่มจาก Methodology ที่แยกให้เห็นว่า
Redistribution Troubleshooting ต้องตอบ 3 คำถามเสมอ (ฝั่งไหน, ทิศทางไหน, Seed Metric ถูกไหม)
ตามด้วย Seed Metric หายแบบเงียบ (EIGRP) เทียบกับใช้ Default โดยไม่รู้ตัว (OSPF), Routing Loop
จาก Tag Filter หายที่จุดเดียว, Administrative Distance ที่ทำให้ Route ผิด Priority ชนะ,
Prefix-list `ge`/`le` ที่เขียนผิดจนบล็อก Route ที่ควรผ่าน, BGP Full Table รั่วเข้า IGP จาก
Route-map หลุด, Metric-type E1/E2 ปนกันจนเลือก Path ผิด, `default-metric` แบบ Global ที่กระทบ
Source อื่นที่ไม่ควรกระทบ, และ Asymmetric Routing จาก Redistribution ที่มีแค่จุดเดียว — ปิดท้าย
ด้วย Lab เต็มรูปแบบที่รวม 2 บั๊กเข้าด้วยกันพร้อมกันตามสถานการณ์ Production จริงหลัง Maintenance
Window พร้อมการวินิจฉัยและ Verification แบบครบวงจร ✅

**พร้อมสำหรับ Part 60**: เมื่อ Routing Domain ทั้งหมด (OSPF, EIGRP, BGP) และจุดเชื่อมระหว่างกัน
ผ่านการ Troubleshoot ครบแล้ว ขั้นต่อไปคือการดูแล **WAN Overlay** ที่ซับซ้อนที่สุดของหลักสูตร —
**DMVPN และ Site-to-Site VPN Troubleshooting** ซึ่งผสมทั้งปัญหา Routing (ผ่าน mGRE/NHRP),
Tunnel Encapsulation, และ IPsec เข้าด้วยกันในเวลาเดียวกัน ต่อยอดจาก Foundation ของ
[Part 52](part-052-dmvpn-fundamentals.md) และ [Part 53](part-053-dmvpn-advanced.md)

**ไปต่อ:** [Part 60 — DMVPN & VPN Troubleshooting →](part-060-dmvpn-vpn-troubleshooting.md)
