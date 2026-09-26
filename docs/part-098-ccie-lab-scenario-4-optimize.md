# Part 98 — CCIE Full Lab Scenario #4 — Optimize/Operate Module
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 971–980 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 97 — CCIE Full Lab Scenario #3 (Deploy Module)](part-097-ccie-lab-scenario-3-deploy.md)
> ที่ปิดงาน Deploy ไปแล้ว — วิศวกรได้รับ Design จาก Part 96 (Design Module) มาสร้างจริงภายในหน้าต่าง
> เวลาที่จำกัดของ Lab Exam และผ่าน Functional Requirement ทุกข้อ (Connectivity ครบ, Routing ขึ้น,
> Security พื้นฐานมี, Automation Script Deploy สำเร็จ) แต่ **"ผ่าน Baseline" ไม่ได้แปลว่า "Optimal"**
> — Deploy ภายใต้ Time Pressure มักทิ้ง Technical Debt ไว้เสมอในส่วนที่ไม่ใช่ Pass/Fail ตรงๆ
> (Area Design, QoS Differentiation, BGP Scalability, Security Granularity, Automation Maturity)
>
> **Part นี้คือ Module ที่ 4 และปิดท้ายของ CCIE Full Lab Scenario Block (Part 95-98)**:
> **Diagnose** (Part 95, Broken → Fixed), **Design** (Part 96, โจทย์ → Blueprint), **Deploy**
> (Part 97, Blueprint → Working จริง), และ **Optimize** (Part 98, Working → Working ดีขึ้น **โดย
> ไม่พังของเดิม**) — ทักษะทั้ง 4 นี้คือทักษะเดียวกันที่ CCIE Lab Exam จริงแบ่งสัดส่วนคะแนนให้แต่ละ
> Module แยกกันชัดเจน ตามที่ [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) แนะนำภาพรวมไว้
>
> **สถานการณ์ของ Part นี้**: REGION-DC คือ Site ใหม่ที่ Part 97 Deploy สำเร็จเสร็จแล้ว — ประกอบด้วย
> `REGION-EDGE1`/`REGION-EDGE2` (WAN Edge Router เชื่อมกลับ HQ ผ่าน `WAN-EDGE-1`/`WAN-EDGE-2`),
> `REGION-CORE1`/`REGION-CORE2` (Core/Distribution รวมตัวเดียวของ Site ขนาดเล็ก), และ `REGION-ACC1`
> (Access Switch) ทุกอย่าง **ทำงานได้ 100%** ตาม Baseline ที่ Part 97 ตั้งไว้ แต่มี 5 จุดที่ยังไม่
> Optimal ซึ่งเป็นโจทย์ของ Step 972-976 ในภาคนี้

## ตารางอ้างอิง Address Plan ของ REGION-DC (เฉพาะ Part นี้ ส่วนต่อขยายจาก [00-ip-address-plan.md](00-ip-address-plan.md))

| รายการ | ค่า |
|---|---|
| Loopback0: REGION-EDGE1 / REGION-EDGE2 | 1.1.1.31/32 / 1.1.1.32/32 |
| Loopback0: REGION-CORE1 / REGION-CORE2 | 1.1.1.33/32 / 1.1.1.34/32 |
| Transit WAN-EDGE-1 ↔ REGION-EDGE1 | 10.255.91.0/30 |
| Transit WAN-EDGE-2 ↔ REGION-EDGE2 | 10.255.92.0/30 |
| Transit REGION-EDGE1 ↔ REGION-EDGE2 (cross-link) | 10.255.93.0/30 |
| Transit REGION-EDGE1 ↔ REGION-CORE1 | 10.255.94.0/30 |
| Transit REGION-EDGE2 ↔ REGION-CORE2 | 10.255.95.0/30 |
| Transit REGION-CORE1 ↔ REGION-CORE2 (Po1 routed, LACP) | 10.255.96.0/30 |
| VLAN 910 REGION-DATA | 10.90.10.0/24 (GW 10.90.10.1 HSRP VIP) |
| VLAN 920 REGION-VOICE | 10.90.20.0/24 (GW 10.90.20.1 HSRP VIP) |
| VLAN 999 MGMT (ใช้ร่วมกับ HQ) | 10.90.99.0/24 (GW 10.90.99.1 HSRP VIP) |
| AS BGP | 65001 เดียวกับทั้งองค์กร (REGION ไม่ใช่ AS แยก) |
| OSPF Process | 1 เดียวกับทั้งองค์กร |

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 971 | Optimize Module คืออะไร — ทักษะ "ทำของที่ดีอยู่แล้วให้ดีขึ้น" ต่างจาก Diagnose/Deploy อย่างไร |
| 972 | Optimization Task 1 — OSPF: จาก Area 0 เดียวทั้งหมด สู่ Area Boundary ที่ถูกต้อง พร้อม Summarization |
| 973 | Optimization Task 2 — QoS Single-Class → MQC 4-Class Model และปรับ EtherChannel Load-Balance |
| 974 | Optimization Task 3 — BGP: จาก Full-Mesh iBGP สู่ Route-Reflector Hierarchy แบบไม่ Drop Session |
| 975 | Optimization Task 4 — CoPP Class เดียวเกินไป และ ACL กว้างเกินไป → Granular CoPP + TrustSec SGACL |
| 976 | Optimization Task 5 — จาก Ad-hoc Script (Part 23-style) สู่ Pipeline โตเต็มที่ (Part 47/48/79) |
| 977 | วินัยการยืนยันว่า "Optimize แล้วไม่พัง" — Checklist ก่อน/ระหว่าง/หลัง Deploy Change |
| 978 | การวัดผลก่อน/หลังจริง — Convergence Time, CPU, Throughput ด้วย Telemetry (Part 54/80) |
| 979 | Anti-Pattern ของการ Optimize — เสียเวลากับสิ่งที่ไม่ใช่ Bottleneck จริง และ Over-Optimization |
| 980 | Full Lab — Optimize 3 จุดทั่วทั้ง Enterprise Lab (HQ + REGION-DC) พร้อม Evidence เชิงตัวเลข |

---

## Step 971 — Optimize Module คืออะไร

### 971.1 นิยามที่แม่นยำ: "ทำงานอยู่แล้ว" ≠ "ดีที่สุดแล้ว"

Optimize Module ในภาษาของ CCIE Lab Exam ให้โจทย์ในรูปแบบเฉพาะที่ต่างจาก Module อื่นทั้งหมด:
**เครือข่ายที่ให้มาทำงานได้ 100% ตาม Baseline Requirement ทุกข้ออยู่แล้ว** — Ping ผ่านหมด, Routing
Converge ได้, Security Policy บล็อกสิ่งที่ควรบล็อก — แต่โจทย์จะให้ **เป้าหมายเพิ่มเติมที่เจาะจง**
(เช่น "ลด Convergence Time ลงต่ำกว่า X วินาที", "ลด CPU Utilization ช่วง Peak", "แยก Traffic Voice
ออกจาก Data ให้ชัดเจน") แล้วให้ผู้สอบ **แก้ไขเพื่อบรรลุเป้าหมายนั้น โดยที่ทุก Requirement เดิมที่ผ่าน
อยู่แล้วต้องยังผ่านเหมือนเดิมทุกข้อ**

### 971.2 เปรียบเทียบ 3 ทักษะที่ดูคล้ายกันแต่ต่างกันโดยสิ้นเชิง

| มิติ | Diagnose (Part 95) | Deploy (Part 97) | **Optimize (Part 98)** |
|---|---|---|---|
| จุดเริ่มต้น (Starting State) | เสีย/ผิดปกติ (Broken) | ไม่มีอะไรเลย (Nothing) | **ทำงานได้แล้ว (Working)** |
| เป้าหมาย | กลับไปสู่สภาพที่ควรทำงานได้ | สร้างให้ทำงานได้ตาม Design | **ทำงานได้ดีขึ้นกว่าเดิมตามเป้าที่กำหนด** |
| ความเสี่ยงหลัก | วินิจฉัยผิดจุด เสียเวลา | Design ผิด/ลืม Requirement | **Regression — แก้แล้วของเดิมพัง** |
| หลักฐานว่า "เสร็จงาน" | อาการเดิมหายไป | Requirement ทุกข้อผ่านครั้งแรก | **Metric เป้าหมายดีขึ้น + Requirement เดิมทุกข้อยังผ่าน** |
| ทักษะที่ถูกวัดจริง | Methodology การหา Root Cause (Part 91) | ความครบถ้วนตาม Design + วินัย Change | **การเปลี่ยนแปลงอย่างระมัดระวังบนระบบที่ "มีอะไรให้เสีย"** |

**ข้อสังเกตสำคัญที่สุด**: Diagnose และ Deploy ทั้งคู่ทำงานกับระบบที่ **"ยังไม่มี Baseline ที่ต้องรักษา"**
— Diagnose มี Baseline แต่ Baseline นั้น **เสียไปแล้ว** (ไม่มีอะไรจะพังเพิ่ม เพราะพังอยู่แล้ว) ส่วน Deploy
ยังไม่มี Production Traffic วิ่งอยู่ (ยังไม่ Go-Live เต็มรูปแบบ) **Optimize คือ Module เดียวที่ต้องแก้ไข
ระบบที่ "มี Production Traffic วิ่งอยู่จริงและใช้งานได้ดีอยู่แล้ว"** ความเสี่ยงจึงไม่เหมือนกันเลย — ทุก
คำสั่งที่พิมพ์มีโอกาสทำให้สิ่งที่ทำงานอยู่แล้วหยุดทำงาน (Regression) ซึ่งไม่มีใน 2 Module ก่อนหน้า

### 971.3 ทำไม Regression Risk คือแกนกลางของทุก Step ใน Part นี้

```
Diagnose:  [ผิดอยู่แล้ว] ──(แก้)──► [ถูก]                     ไม่มีอะไรจะเสียเพิ่ม
Deploy:    [ไม่มีอะไร]   ──(สร้าง)──► [ใช้งานได้]               ยังไม่มี Production พึ่งพา
Optimize:  [ใช้งานได้ดี] ──(แก้)──► [ใช้งานได้ดีขึ้น] หรือ [พัง] ◄── มี Production พึ่งพาอยู่แล้ว!
```

ทุก Step ตั้งแต่ 972 ถึง 976 จะมีโครงสร้างเดียวกันเสมอ: **สถานะเดิม (Before) ที่ทำงานได้** →
**เป้าหมายที่ระบุมา (Stated Goal)** → **การเปลี่ยนแปลง (Change)** → **การพิสูจน์ว่าของเดิมไม่พัง
(Non-Regression Verification)** → **การพิสูจน์ว่าเป้าหมายบรรลุจริง (Goal Verification)** — ขาดขั้นตอน
ใดขั้นตอนหนึ่งไปถือว่า **ยังไม่จบงาน Optimize** ไม่ว่าการเปลี่ยนแปลงทางเทคนิคจะถูกต้องแค่ไหนก็ตาม

### 971.4 สถานะ REGION-DC หลัง Part 97 Deploy — 5 จุดที่ยังไม่ Optimal

| # | โดเมน | สถานะปัจจุบัน (ผ่าน Baseline) | จุดที่ยังไม่ Optimal | Step ที่แก้ |
|---|---|---|---|---|
| 1 | OSPF | Routing ครบ, Ping ผ่านทุกปลายทาง | ทุกอุปกรณ์อยู่ Area 0 เดียวกับ HQ ทั้งหมด, Reference-bandwidth ยังเป็น Default | 972 |
| 2 | QoS/EtherChannel | Data Traffic วิ่งได้ปกติ | Voice ไม่มี Priority, WAN Link ไม่มี Shaping, Po1 Load-balance ใช้ src-dst-mac | 973 |
| 3 | BGP | Prefix ครบทุกเส้นทาง | Full-Mesh iBGP 4 ตัวใหม่ ซ้ำรอย HQ ก่อน Part 30 | 974 |
| 4 | Security | Traffic ที่ควรบล็อกถูกบล็อกแล้ว | CoPP Class เดียวรวมทุกอย่าง, ACL กว้างเกินจำเป็น | 975 |
| 5 | Automation | Config ถูก Push สำเร็จตอน Deploy | Script Ad-hoc รันจาก Laptop มือเดียว ไม่มี Git/Test/Rollback | 976 |

---

## Step 972 — Optimization Task 1: OSPF Area Design & Reference-Bandwidth

### 972.1 โจทย์ที่ระบุมา (Stated Requirement)

> "REGION-DC เชื่อมกลับ HQ ผ่าน WAN Link ที่ Latency สูงกว่า Backbone ภายในมาก การเปลี่ยนแปลงเล็กๆ
> ภายใน REGION-DC (เช่น Port Access User เข้า-ออกตามปกติ) **ต้องไม่ทำให้ CORE-SW1/CORE-SW2 ของ HQ
> ต้องคำนวณ SPF ใหม่ทุกครั้ง** และ REGION-DC เองก็ไม่จำเป็นต้องเห็นทุก Backbone Link ย่อยของ HQ
> แบบ /30 — ให้ปรับ OSPF Design ให้บรรลุเป้าหมายนี้ **โดยที่ Reachability ทุกปลายทางต้องไม่เปลี่ยน
> แม้แต่เส้นทางเดียว**"

### 972.2 Before — ตรวจสภาพปัจจุบันก่อนแก้อะไรเลย

```
REGION-EDGE1# show running-config | section router ospf
router ospf 1
 router-id 1.1.1.31
 network 10.255.91.0 0.0.0.3 area 0
 network 10.255.93.0 0.0.0.3 area 0
 network 10.255.94.0 0.0.0.3 area 0

REGION-CORE1# show running-config | section router ospf
router ospf 1
 router-id 1.1.1.33
 network 10.255.94.0 0.0.0.3 area 0
 network 10.255.96.0 0.0.0.3 area 0
 network 10.90.10.0 0.0.0.255 area 0
 network 10.90.20.0 0.0.0.255 area 0
 network 10.90.99.0 0.0.0.255 area 0
```

**ปัญหาที่เห็นชัด**: ไม่มี `auto-cost reference-bandwidth` เลย (ใช้ Default 100 Mbps) ทั้งที่ Link
ภายใน REGION-DC เป็น 10G ทั้งหมด และทุก `network` statement อยู่ใน `area 0` เหมือน HQ Backbone
เป๊ะๆ — เท่ากับ REGION-DC ทั้ง Site เป็นส่วนขยายของ Area 0 โดยตรง ไม่มี Boundary ใดๆ กั้นเลย

```
CORE-SW1# show ip ospf database database-summary | include Area 0|Total

  Area 0 database summary
    Total          142

CORE-SW1# show ip ospf statistics | include 10.90
   SPF triggered by change in LSA type 1, LSA ID 1.1.1.33, area 0.0.0.0
```

**หลักฐานยืนยัน Root Cause**: `show ip ospf statistics` บน `CORE-SW1` (HQ) แสดงว่า SPF ล่าสุดถูก
Trigger โดย LSA ของ `REGION-CORE1` (1.1.1.33) — **HQ Core ต้องรัน SPF ใหม่จาก REGION-DC โดยตรง**
ยืนยันปัญหาตามที่โจทย์ระบุ

### 972.3 After — สร้าง Area 90 และ Summarization ที่ ABR (`REGION-EDGE1`/`REGION-EDGE2`)

```
! ทุกอุปกรณ์ใน REGION-DC (รวม auto-cost ให้ตรง HQ Standard จาก Part 26 Step 257 ก่อน)
REGION-EDGE1(config)# router ospf 1
REGION-EDGE1(config-router)# auto-cost reference-bandwidth 100000
REGION-EDGE2(config-router)# auto-cost reference-bandwidth 100000
REGION-CORE1(config-router)# auto-cost reference-bandwidth 100000
REGION-CORE2(config-router)# auto-cost reference-bandwidth 100000

! REGION-EDGE1 = ABR: WAN-facing (ไปทาง HQ) อยู่ area 0 เดิม, REGION-facing ย้ายไป area 90
REGION-EDGE1(config-router)# no network 10.255.93.0 0.0.0.3 area 0
REGION-EDGE1(config-router)# no network 10.255.94.0 0.0.0.3 area 0
REGION-EDGE1(config-router)# network 10.255.93.0 0.0.0.3 area 90
REGION-EDGE1(config-router)# network 10.255.94.0 0.0.0.3 area 90
! (10.255.91.0/30 ไปทาง WAN-EDGE-1 ยังคง area 0 เดิม ไม่แตะ)

! Summarize เส้นทางภายใน Area 90 ทั้งหมดก่อนส่งเข้า Area 0 (ทิศทางออกจาก REGION สู่ HQ)
REGION-EDGE1(config-router)# area 90 range 10.90.0.0 255.255.0.0
! Summarize เส้นทาง Backbone ของ HQ (Area 0) ก่อนส่งเข้า Area 90 (ทิศทางจาก HQ สู่ REGION)
REGION-EDGE1(config-router)# area 0 range 10.255.0.0 255.255.0.0
REGION-EDGE1(config-router)# area 0 range 10.10.0.0 255.255.0.0

! REGION-EDGE2 config เดียวกัน (ABR ตัวที่สอง — เผื่อ Redundancy)
REGION-EDGE2(config-router)# no network 10.255.93.0 0.0.0.3 area 0
REGION-EDGE2(config-router)# no network 10.255.95.0 0.0.0.3 area 0
REGION-EDGE2(config-router)# network 10.255.93.0 0.0.0.3 area 90
REGION-EDGE2(config-router)# network 10.255.95.0 0.0.0.3 area 90
REGION-EDGE2(config-router)# area 90 range 10.90.0.0 255.255.0.0
REGION-EDGE2(config-router)# area 0 range 10.255.0.0 255.255.0.0
REGION-EDGE2(config-router)# area 0 range 10.10.0.0 255.255.0.0

! REGION-CORE1/CORE2 (Internal Router ล้วน) ย้ายทุก network statement ไป area 90
REGION-CORE1(config-router)# no network 10.255.94.0 0.0.0.3 area 0
REGION-CORE1(config-router)# no network 10.255.96.0 0.0.0.3 area 0
REGION-CORE1(config-router)# no network 10.90.10.0 0.0.0.255 area 0
REGION-CORE1(config-router)# no network 10.90.20.0 0.0.0.255 area 0
REGION-CORE1(config-router)# no network 10.90.99.0 0.0.0.255 area 0
REGION-CORE1(config-router)# network 10.255.94.0 0.0.0.3 area 90
REGION-CORE1(config-router)# network 10.255.96.0 0.0.0.3 area 90
REGION-CORE1(config-router)# network 10.90.10.0 0.0.0.255 area 90
REGION-CORE1(config-router)# network 10.90.20.0 0.0.0.255 area 90
REGION-CORE1(config-router)# network 10.90.99.0 0.0.0.255 area 90
! REGION-CORE2 config รูปแบบเดียวกัน (สลับ interface ตามจริง)
```

### 972.4 ลำดับการเปลี่ยนแปลงที่ปลอดภัย — ทำไม OSPF Area Mismatch จึง "Fail Loud" ไม่ใช่ "Fail Silent"

**ข้อดีที่ต้องรู้ก่อนกลัวการเปลี่ยน Area**: OSPF บังคับให้ Area ID ตรงกันทั้ง 2 ฝั่งของ Link ก่อนขึ้น
Adjacency — ถ้าฝั่งหนึ่งเปลี่ยนเป็น `area 90` แล้วอีกฝั่งยังเป็น `area 0` **Adjacency จะ Down ทันที
อย่างเห็นได้ชัด** (ไม่ใช่ Routing ผิดแบบเงียบๆ) นี่คือความปลอดภัยในตัวเองของ Feature นี้ — ผลกระทบ
จำกัดอยู่แค่ Link นั้น ไม่ลามไปที่อื่น

**ลำดับที่แนะนำ** (ใช้ประโยชน์จาก REGION-EDGE1/EDGE2 ที่มี 2 เส้นทางไปกลับ HQ):

```
1. เปลี่ยนฝั่ง REGION-EDGE1 ก่อน (ระหว่างนี้ Traffic REGION↔HQ ทั้งหมดวิ่งผ่าน REGION-EDGE2 ชั่วคราว)
2. ตรวจ REGION-EDGE1 ขึ้น Adjacency กับ area 90 ใหม่เรียบร้อย + area range ทำงาน (ดู 972.5)
3. เปลี่ยนฝั่ง REGION-EDGE2 (ตอนนี้ REGION-EDGE1 พร้อมรับ Traffic แทนแล้ว)
4. เปลี่ยน REGION-CORE1 แล้ว REGION-CORE2 ตามลำดับ (Internal Router ไม่ใช่ ABR ผลกระทบแคบกว่า)
```

### 972.5 Verification — ทั้ง Non-Regression และ Goal

```
! Non-Regression: Reachability ทุกปลายทางต้องเหมือนเดิม
REGION-ACC1# ping 10.10.30.10 source Vlan910    ! ไปหา Server ที่ HQ
Success rate is 100 percent (5/5)

! Goal 1: HQ เห็น Summary Route เดียว ไม่เห็นทุก /24 ของ REGION
CORE-SW1# show ip route ospf | include 10.90
O IA  10.90.0.0/16 [110/11] via 10.255.91.2, 00:02:14, GigabitEthernet0/0/1

! Goal 2: REGION เห็น Summary ของ HQ Backbone แทนทุก /30
REGION-CORE1# show ip route ospf | include 10.255|10.10
O IA  10.255.0.0/16 [110/12] via 10.255.94.1, 00:02:10, TenGigabitEthernet1/1
O IA  10.10.0.0/16 [110/13] via 10.255.94.1, 00:02:10, TenGigabitEthernet1/1

! Goal 3: Flap ที่ REGION-ACC1 ไม่ทำให้ HQ รัน SPF อีกต่อไป
REGION-ACC1# shutdown ... (จำลอง Flap ที่ Access Layer)
CORE-SW1# show ip ospf statistics | include triggered
   (ไม่มีบรรทัดใหม่ปรากฏ — HQ ไม่รัน SPF จากเหตุการณ์นี้อีกต่อไป)
REGION-CORE1# show ip ospf statistics | include triggered
   SPF triggered by change in LSA type 1 ... area 0.0.0.90   (รันแค่ใน Area 90 ตามคาด)
```

ผลลัพธ์ครบทั้ง 2 ด้าน: **Reachability เดิมไม่เปลี่ยน (Non-Regression)** และ **Blast Radius ของ SPF
ถูกจำกัดอยู่ใน Area 90 เท่านั้น ไม่ลามไป HQ อีกต่อไป (Goal บรรลุ)**

---

## Step 973 — Optimization Task 2: QoS Single-Class → MQC 4-Class และ EtherChannel Load-Balance

### 973.1 โจทย์ที่ระบุมา

> "IP Phone ที่ REGION-DC เพิ่งเปิดใช้งานมีเสียงแตก/ก้องเป็นระยะ โดยเฉพาะช่วงที่ Data Traffic (Backup,
> File Transfer) วิ่งพร้อมกัน ให้ปรับ QoS ให้ Voice ได้ Priority ตามมาตรฐานเดียวกับ HQ (Part 24/40)
> **โดยไม่ทำให้ Data Traffic เดิม Throughput ลดลงเกิน 5%**"

### 973.2 Before — REGION-ACC1 ไม่มี Trust Boundary, REGION-EDGE ไม่มี Shaping

```
REGION-ACC1# show running-config interface GigabitEthernet1/0/5
interface GigabitEthernet1/0/5
 switchport access vlan 910
 switchport voice vlan 920
 spanning-tree portfast
! ไม่มี mls qos trust เลย — DSCP จาก IP Phone ถูก Trust ตาม Default (untrusted = remark เป็น 0)

REGION-EDGE1# show running-config interface GigabitEthernet0/0/1
interface GigabitEthernet0/0/1
 description WAN Circuit 20 Mbps to WAN-EDGE-1 (Physical port = 1 Gbps)
 ip address 10.255.91.2 255.255.255.252
! Physical 1 Gbps แต่ Contract จริงแค่ 20 Mbps — ไม่มี shape ใดๆ เลย
! Provider Policer จะ Tail-drop ทุกอย่างปนกันเมื่อเกิน 20 Mbps รวม Voice ด้วย
```

### 973.3 After — Trust Boundary + MQC 4-Class ตามมาตรฐาน Part 24/40, Shaping ที่ WAN Edge

```
! 1) Trust Boundary ที่ Access Port (Part 24 Step 246 pattern)
REGION-ACC1(config)# interface GigabitEthernet1/0/5
REGION-ACC1(config-if)# mls qos trust device cisco-phone

! 2) MQC 4-Class Model เดียวกับ WAN-EDGE-1/2 ของ HQ (Part 40 Step 392)
REGION-EDGE1(config)# class-map match-any VOICE
REGION-EDGE1(config-cmap)# match dscp ef
REGION-EDGE1(config)# class-map match-any VIDEO
REGION-EDGE1(config-cmap)# match dscp af41 af42 af43
REGION-EDGE1(config)# class-map match-any CRITICAL-DATA
REGION-EDGE1(config-cmap)# match dscp af31 af21 cs3

REGION-EDGE1(config)# policy-map REGION-QOS
REGION-EDGE1(config-pmap)# class VOICE
REGION-EDGE1(config-pmap-c)#  priority percent 10
REGION-EDGE1(config-pmap)# class VIDEO
REGION-EDGE1(config-pmap-c)#  bandwidth percent 20
REGION-EDGE1(config-pmap)# class CRITICAL-DATA
REGION-EDGE1(config-pmap-c)#  bandwidth percent 30
REGION-EDGE1(config-pmap)# class class-default
REGION-EDGE1(config-pmap-c)#  bandwidth percent 40
REGION-EDGE1(config-pmap-c)#  random-detect dscp-based

! 3) Hierarchical: Shape ก่อน (ให้ตรงกับ Contract 20 Mbps จริง) แล้วค่อย Nest Policy ด้านบน
REGION-EDGE1(config)# policy-map REGION-SHAPE
REGION-EDGE1(config-pmap)# class class-default
REGION-EDGE1(config-pmap-c)#  shape average 20000000
REGION-EDGE1(config-pmap-c)#  service-policy REGION-QOS

REGION-EDGE1(config)# interface GigabitEthernet0/0/1
REGION-EDGE1(config-if)# service-policy output REGION-SHAPE
```

### 973.4 EtherChannel Load-Balance ที่ Po1 (REGION-CORE1 ↔ REGION-CORE2)

```
REGION-CORE1# show etherchannel load-balance
EtherChannel Load-Balancing Method: src-dst-mac

REGION-CORE1(config)# port-channel load-balance src-dst-ip
REGION-CORE2(config)# port-channel load-balance src-dst-ip
```

**เหตุผล**: ระหว่าง `REGION-CORE1` กับ `REGION-CORE2` มีแค่ MAC Address คู่เดียวเสมอ (เป็น Router
ทั้งคู่ ไม่ใช่ Switch ที่มี MAC หลากหลาย) — `src-dst-mac` จึง Hash ได้ผลลัพธ์เดิมซ้ำเสมอ ทำให้ Traffic
ทั้งหมดวิ่งลง Physical Link เดียวใน Po1 แม้จะมี 2 Link ก็ตาม เปลี่ยนเป็น `src-dst-ip` ทำให้กระจาย
ตาม IP Pair ของ Traffic ปลายทางจริง (Server/Client จำนวนมาก) กระจายได้ดีกว่ามาก

> **คำเตือนเรื่อง Non-Regression**: การเปลี่ยน `port-channel load-balance` **มีผลย้อนหลังกับ Flow ที่
> วิ่งอยู่แล้วบางส่วน** เพราะ Hash ใหม่อาจย้าย Flow ที่กำลังวิ่งอยู่ไปคนละ Physical Link (สำหรับ TCP
> Session ที่วิ่งอยู่ผลกระทบมักไม่มาก เพราะเป็นแค่ Load-balance ภายใน Channel เดียวกัน ไม่ใช่ Routing
> Path เปลี่ยน) แนะนำเปลี่ยนช่วง Low-traffic Window และ Monitor `show interfaces port-channel1
> etherchannel` ก่อน/หลังเทียบ Packet Count ต่อ Physical Member

### 973.5 Verification

```
! Non-Regression: Data Throughput ต้องไม่ลดลงเกิน 5% ตามโจทย์
REGION-ACC1# iperf3 -c 10.10.30.10 -t 30   ! Before: 18.2 Mbps / After: 17.6 Mbps (ลดลง 3.3% — ผ่านเกณฑ์)

! Goal: Voice Jitter ต้องลดลงชัดเจน
REGION-ACC1# iperf3 -u -c 10.10.20.10 -b 100k -t 30
! Before shaping+priority: Jitter 42.8 ms, Loss 3.1%
! After shaping+priority:  Jitter 6.4 ms,  Loss 0.0%
```

---

## Step 974 — Optimization Task 3: BGP Full-Mesh → Route-Reflector Hierarchy

### 974.1 โจทย์ที่ระบุมา

> "REGION-EDGE1/2 และ REGION-CORE1/2 ถูก Deploy แบบ iBGP Full-Mesh ทั้ง 4 ตัวเข้าหากันและเข้าหา
> `CORE-SW1`/`CORE-SW2` ของ HQ ตรงๆ (ซ้ำรูปแบบที่ HQ เองเคยมีก่อน [Part 30 Step 293](part-030-bgp-advanced.md))
> ให้ปรับเป็น Route-Reflector Hierarchy ที่สอดคล้องกับ HQ **โดยห้าม Drop Session ระหว่าง
> Transition แม้แต่ครั้งเดียว** (Prefix ต้องพร้อมส่งต่อได้ตลอดเวลาระหว่างการเปลี่ยน)"

### 974.2 Before — Full-Mesh 4 ตัวใหม่ ซ้อนบน RR เดิมของ HQ

```
REGION-CORE1# show ip bgp summary | begin Neighbor
Neighbor        V    AS  MsgRcvd  MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
1.1.1.1         4 65001     8842     8840        0    0    0 3w2d           412   ! CORE-SW1 (HQ RR)
1.1.1.2         4 65001     8839     8841        0    0    0 3w2d           412   ! CORE-SW2 (HQ RR)
1.1.1.31        4 65001     3201     3199        0    0    0 2w0d           412   ! REGION-EDGE1
1.1.1.32        4 65001     3198     3200        0    0    0 2w0d           412   ! REGION-EDGE2
1.1.1.34        4 65001     3202     3201        0    0    0 2w0d           412   ! REGION-CORE2
```

REGION-CORE1 มี Session ทั้งหมด 5 เส้น (ไปหา HQ RR ทั้ง 2 ตัว + REGION ตัวอื่นอีก 3 ตัว) — ทุก Router
ใหม่ทั้ง 4 ตัวเป็นแบบเดียวกันหมด รวม 10 Session ใหม่ในการเพิ่ม REGION-DC เข้ามาเพียง Site เดียว และ
ยังไม่ได้ใช้ประโยชน์จาก RR ที่ HQ ลงทุนสร้างไว้ตั้งแต่ Part 30/74 เลย

### 974.3 Design ใหม่ — REGION-CORE1/2 เป็น Tier-2 RR

```
Tier 1 (HQ RR):        CORE-SW1 ◄──────► CORE-SW2                (Non-Client กันเอง, มีอยู่แล้ว)
                             ▲                  ▲
                     RR-Client          RR-Client
                             │                  │
Tier 2 (REGION RR):  REGION-CORE1 ◄────► REGION-CORE2            (Non-Client กันเอง — ใหม่)
                             ▲                  ▲
                     RR-Client          RR-Client
                             │                  │
                      REGION-EDGE1      REGION-EDGE2
```

### 974.4 Transition แบบไม่ Drop Session — ขั้นตอนที่ปลอดภัย

**หลักการสำคัญที่สุด**: การเพิ่ม `route-reflector-client` บน Session ที่ Established อยู่แล้ว **ไม่
ต้อง Reset Session** — เป็นแค่การเปลี่ยนว่า Router จะ Reflect Route ต่ออย่างไร ต้องการแค่ Soft
Refresh ขาออกเท่านั้น

```
! ขั้นที่ 1 — เพิ่ม route-reflector-client บน Session ที่มีอยู่แล้ว (ไม่ Drop)
REGION-CORE1(config)# router bgp 65001
REGION-CORE1(config-router)# address-family ipv4
REGION-CORE1(config-router-af)# neighbor 1.1.1.31 route-reflector-client
REGION-CORE1(config-router-af)# neighbor 1.1.1.32 route-reflector-client
REGION-CORE1(config-router-af)# end
REGION-CORE1# clear ip bgp 1.1.1.31 soft out
REGION-CORE1# clear ip bgp 1.1.1.32 soft out
! REGION-CORE2 ทำแบบเดียวกัน

! ขั้นที่ 2 — CORE-SW1/CORE-SW2 (HQ) รับ REGION-CORE1/2 เป็น RR-Client ของตัวเอง
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# address-family ipv4
CORE-SW1(config-router-af)# neighbor 1.1.1.33 route-reflector-client
CORE-SW1(config-router-af)# neighbor 1.1.1.34 route-reflector-client
CORE-SW1(config-router-af)# end
CORE-SW1# clear ip bgp 1.1.1.33 soft out
CORE-SW1# clear ip bgp 1.1.1.34 soft out
! CORE-SW2 ทำแบบเดียวกัน

! ขั้นที่ 3 — ยืนยัน RR-Path มาแทนที่ Full-Mesh Path ก่อนถอด Session (สำคัญที่สุด!)
REGION-EDGE1# show bgp ipv4 unicast summary | include 1.1.1.32
1.1.1.32        4 65001     3205     3203        0    0    0 00:00:12       412   ! PfxRcd เท่าเดิม (412) หลัง Soft Refresh

! ขั้นที่ 4 — เมื่อยืนยัน PfxRcd เท่าเดิมทุกคู่แล้ว จึงค่อยถอด Session Full-Mesh ที่ซ้ำซ้อนทีละคู่
REGION-EDGE1(config-router)# no neighbor 1.1.1.32 remote-as 65001      ! ถอด Direct Session ไป EDGE2
REGION-EDGE1# show bgp ipv4 unicast summary | include 1.1.1.34
                                                                       ! PfxRcd ต้องยังเท่าเดิม (ได้จาก REGION-CORE1 RR แทน)
```

**กฎเหล็ก**: ห้ามถอด Session Full-Mesh คู่ไหนเลย **จนกว่าจะยืนยันแล้วว่า RR-Path ส่ง Prefix จำนวน
เท่าเดิมมาให้แล้วจริง** (ขั้นที่ 3 ก่อนขั้นที่ 4 เสมอ) — ถอดทีละคู่ ตรวจ `PfxRcd` ทุกครั้งก่อนถอดคู่ถัดไป

### 974.5 After — Session Count ลดลง และ Scalability ดีขึ้นเมื่อ Site เพิ่ม

| | Before (Full-Mesh) | After (RR Hierarchy) |
|---|---|---|
| Session ทั้งหมดที่เกี่ยวกับ REGION-DC | 10 | 9 (ลดลงเล็กน้อยที่ขนาดปัจจุบัน) |
| Session ต่อ REGION-EDGE ตัวใหม่ที่เพิ่มเข้ามาในอนาคต | +4 ต่อตัว (Full-Mesh) | **+1 ต่อตัว** (แค่ต่อ RR) |
| Update Replication ที่ REGION-EDGE ต้องทำ | ส่งให้ Neighbor ทุกตัวเอง | RR (REGION-CORE) จัดการให้ |

**ข้อสังเกตตรงไปตรงมา**: ที่ขนาด 4 ตัวปัจจุบัน Session ลดลงแค่ 1 เส้น — ประโยชน์จริงของ RR ไม่ได้อยู่
ที่ตัวเลขวันนี้ แต่อยู่ที่ **Scalability เมื่อ REGION-DC ขยาย Site ในอนาคต** (Full-Mesh โต O(n²),
RR โตแค่ O(n)) — Step 979 จะเตือนเรื่องการคาดหวังผลลัพธ์ที่เกินจริงจากการ Optimize แบบนี้

---

## Step 975 — Optimization Task 4: CoPP Granularity และ TrustSec SGACL

### 975.1 โจทย์ที่ระบุมา

> "ช่วงที่ NMS ทำ SNMP Polling REGION-DC แบบ Full Sweep พบว่า OSPF/BGP Neighbor หลุดเป็นระยะสั้นๆ
> พร้อมกัน ให้แก้ไข CoPP ให้ SNMP Burst ไม่กระทบ Routing Protocol อีก และให้ปรับ ACL ที่กว้างเกินไป
> ของ REGION ให้เจาะจงตาม Least-Privilege **โดยไม่บล็อก Traffic ที่ Legitimate อยู่แม้แต่ Session
> เดียว**"

### 975.2 Root Cause ของปัญหา CoPP — Class เดียวรวมทุกอย่าง

```
REGION-EDGE1# show running-config | section class-map|policy-map|control-plane
class-map match-any COPP-DEFAULT
 match any
policy-map COPP-POLICY
 class COPP-DEFAULT
  police cir 5000000 bc 512000 conform-action transmit exceed-action drop
control-plane
 service-policy input COPP-POLICY
```

**ปัญหา**: OSPF Hello, BGP Keepalive, SSH Management, และ SNMP Polling **ทั้งหมดแบ่ง Policer
Bucket เดียวกัน** (`cir 5000000` = 5 Mbps รวมทุกอย่าง) เมื่อ SNMP Full Sweep ส่ง GET-BULK จำนวนมาก
พร้อมกันจนเกิน 5 Mbps รวม Bucket เดียวนี้ **Exceed-Action Drop มีสิทธิ์ Drop OSPF Hello/BGP Keepalive
ที่มาในช่วงเวลาเดียวกันไปด้วย** ทั้งที่ไม่เกี่ยวอะไรกับ SNMP เลย — Routing Protocol จึงหลุดเป็น
Collateral Damage

### 975.3 After — Granular CoPP ตามมาตรฐาน HQ (Part 46 Step 453/454)

```
REGION-EDGE1(config)# class-map match-any COPP-CRITICAL
REGION-EDGE1(config-cmap)# match access-group name ROUTING-PROTOCOLS   ! OSPF, BGP
REGION-EDGE1(config)# class-map match-any COPP-IMPORTANT
REGION-EDGE1(config-cmap)# match access-group name MGMT-TRAFFIC        ! SSH, NTP
REGION-EDGE1(config)# class-map match-any COPP-NORMAL
REGION-EDGE1(config-cmap)# match access-group name MONITORING          ! SNMP, ICMP

REGION-EDGE1(config)# policy-map COPP-POLICY-V2
REGION-EDGE1(config-pmap)# class COPP-CRITICAL
REGION-EDGE1(config-pmap-c)#  police cir 10000000 bc 512000 conform-action transmit exceed-action transmit
REGION-EDGE1(config-pmap)# class COPP-IMPORTANT
REGION-EDGE1(config-pmap-c)#  police cir 2000000 bc 256000 conform-action transmit exceed-action transmit
REGION-EDGE1(config-pmap)# class COPP-NORMAL
REGION-EDGE1(config-pmap-c)#  police cir 1000000 bc 128000 conform-action transmit exceed-action transmit
REGION-EDGE1(config-pmap)# class class-default
REGION-EDGE1(config-pmap-c)#  police cir 500000 bc 64000 conform-action transmit exceed-action drop
```

**ข้อสังเกต**: ทุก Class **ตั้ง `exceed-action transmit` ก่อน** (ไม่ใช่ `drop`) — นี่คือ Monitor Mode
ตามวินัยของ Step 977 ต้อง Soak Period ดูค่าจริงก่อนเปลี่ยนเป็น `drop`

```
! Soak Period — สังเกต Counter จริง 24-48 ชม. ก่อนเปลี่ยนเป็น Enforce
REGION-EDGE1# show policy-map control-plane | include Class|matched|dropped
  Class-map: COPP-CRITICAL
    412000 packets, 34608000 bytes    ! Rate สูงสุดสังเกตได้ 8.2 Mbps — ต่ำกว่า CIR 10M ไม่มี Drop
  Class-map: COPP-NORMAL
    89000 packets, 7565000 bytes      ! Burst SNMP สูงสุด 1.4 Mbps ระหว่าง Sweep — ต่ำกว่า CIR แล้ว

! เมื่อมั่นใจว่า CIR แต่ละ Class ครอบคลุม Traffic จริงแล้ว จึงเปลี่ยนเป็น Enforce จริง
REGION-EDGE1(config)# policy-map COPP-POLICY-V2
REGION-EDGE1(config-pmap-c)#  police cir 1000000 bc 128000 conform-action transmit exceed-action drop
```

### 975.4 TrustSec SGACL แทน ACL กว้าง

```
! Before: ACL กว้างเกินไปที่ REGION-EDGE1 (Deploy Shortcut)
ip access-list extended REGION-OUTBOUND
 permit ip 10.90.0.0 0.0.255.255 any

! After: จำแนก SGT ตาม VLAN (Part 44 pattern) แล้วสร้าง SGACL Matrix เจาะจง
REGION-CORE1(config)# cts role-based sgt-map vlan 910 sgt 210    ! REGION-DATA
REGION-CORE1(config)# cts role-based sgt-map vlan 920 sgt 220    ! REGION-VOICE
REGION-CORE1(config)# cts role-based permissions from 210 to 120 sgacl REGION-TO-HQ-SERVERS-SGACL
REGION-CORE1(config)# cts role-based permissions from 220 to 120 sgacl REGION-VOICE-TO-SERVERS-SGACL
REGION-CORE1(config)# cts role-based permissions from 210 to 220 sgacl DENY-ALL-SGACL

! เปิด Enforcement เฉพาะหลัง Dry-run ด้วย Counter ยืนยันว่าไม่มี Traffic Legitimate ที่จะถูกบล็อก
REGION-CORE1# show cts role-based counters | include 210|220
REGION-CORE1(config-if)# cts role-based enforcement   ! เปิดจริงหลัง Dry-run ผ่าน
```

---

## Step 976 — Optimization Task 5: จาก Ad-hoc Script สู่ Pipeline โตเต็มที่

### 976.1 สถานะปัจจุบัน (มรดกจาก Part 97 Deploy)

Part 97 ใช้ `deploy_region_dc.py` (Netmiko แบบ Part 23 Step 227) รันครั้งเดียวจาก Laptop วิศวกรตอน
Deploy Weekend — **ไม่มี Git, ไม่มี Backup ก่อน Push, ไม่มี Idempotency Check, ไม่มี Test/Verify
Stage อัตโนมัติ** — ตั้งแต่ Go-Live ทุกการเปลี่ยนแปลงบน REGION-DC เป็นการ Login เข้าไปแก้ Manual
ตามแต่ Engineer ที่ On-call — Configuration Drift สั่งสมตั้งแต่วันแรก

### 976.2 ตารางเปรียบเทียบก่อน/หลัง

| คุณสมบัติ | Before (Ad-hoc Script) | After (Pipeline โตเต็มที่) |
|---|---|---|
| Version Control | ไม่มี — Config อยู่บนอุปกรณ์เท่านั้น | Git Repo เดียวกับ HQ ([Part 47 Step 468](part-047-automation-python-advanced.md)) |
| Idempotency | Push คำสั่งซ้ำทุกครั้งไม่ว่าสถานะปัจจุบันเป็นอย่างไร | Ansible `ios_config` เทียบสถานะก่อน Push ([Part 48](part-048-automation-ansible-advanced.md)) |
| Pre-change Backup | ไม่มี | Auto-backup ก่อน Deploy ทุกครั้ง ([Part 79 Step 782](part-079-fullstack-automation-cicd.md)) |
| Blast Radius Control | Push ทั้ง 5 อุปกรณ์พร้อมกันเสมอ | Canary Group 1 อุปกรณ์ก่อน ([Part 79 Step 786](part-079-fullstack-automation-cicd.md)) |
| Rollback | Manual ล้วน จำ Config เดิมเอง | `git revert` + Re-deploy Config เดิมอัตโนมัติ |
| Audit Trail | ไม่มี — ไม่รู้ว่าใครเปลี่ยนอะไรเมื่อไหร่ | Jenkins Pipeline Log + Git Commit History |

### 976.3 Migration Plan — ไม่ทำให้ NOC ที่คุ้นเคย Manual Runbook เดิมติดขัดกลางทาง

```
1. Commit Running-config ปัจจุบันของ REGION-DC ทั้ง 5 ตัวเข้า Git Repo เดิม (ตั้ง Baseline แรก)
2. เขียน Ansible Role "region_dc" (โครงเดียวกับ Part 48) แปลง Logic จาก deploy_region_dc.py เดิม
3. เพิ่ม Group "region_canary" ใน Inventory = REGION-EDGE1 ตัวเดียว (Part 79 Step 786 pattern)
4. รัน Pipeline ใหม่แบบ --check (Dry-run) เทียบผลกับ Manual Runbook เดิมคู่กันไปก่อน (Parallel Run)
5. ต่อ health_check.py --group region_canary หลัง Deploy จริงทุกครั้งเป็น Gate ก่อนขยายไป region_all
6. หลัง Pipeline ผ่าน Maintenance Window จริงติดต่อกัน 2 รอบไม่มีปัญหา จึงเลิกใช้ Script เดิมถาวร
```

```yaml
# inventory/region.yml (ต่อยอดจากโครง Part 79 Step 786)
region_canary:
  hosts:
    REGION-EDGE1:
region_all:
  hosts:
    REGION-EDGE1:
    REGION-EDGE2:
    REGION-CORE1:
    REGION-CORE2:
    REGION-ACC1:
```

> **บทเรียนสำคัญ**: การ Optimize ในงานนี้ **ไม่ได้แก้ Config บนอุปกรณ์เลยแม้แต่บรรทัดเดียว** — สิ่งที่
> ถูก Optimize คือ **กระบวนการดำเนินงาน (Operational Process)** เอง ซึ่งเป็นเป้าหมาย Optimize ที่
> จับต้องยากกว่า Config แต่ส่งผลกระทบยาวนานกว่ามากในระยะยาว (ทุกการเปลี่ยนแปลงบน REGION-DC หลังจากนี้
> ได้ประโยชน์จาก Version Control/Rollback ตลอดไป ไม่ใช่แค่ครั้งเดียว)

---

## Step 977 — วินัยการยืนยันว่า "Optimize แล้วไม่พัง"

### 977.1 Checklist หลัก — ใช้กับทุก Optimization Task ใน Part นี้

| ลำดับ | คำถามที่ต้องตอบได้ | เครื่องมือ/วิธี | เกณฑ์ผ่าน |
|---|---|---|---|
| 1 | เก็บ Baseline ก่อนแก้แล้วหรือยัง? | `show run`, Config Backup, Perf Counter Snapshot | ไฟล์ Baseline มี Timestamp ชัดเจนก่อนแก้ไข |
| 2 | นิยาม "ไม่พัง" ไว้ชัดเจนหรือยัง? | เขียน List Requirement เดิมทุกข้อที่ต้องยังผ่าน | Review/Sign-off List ก่อนเริ่มแก้จริง |
| 3 | มี Staged/Canary Rollout Plan? | เลือก 1 อุปกรณ์/Site นำก่อน (Part 79 Canary) | Canary ผ่าน Health-check ก่อนขยาย |
| 4 | Rollback ทดสอบจริงแล้วหรือแค่เขียนไว้? | ลองรันจริงใน Maintenance Window/Lab ก่อน | Rollback คืนสภาพ Baseline ได้ภายใน MTTR ที่ตั้งไว้ |
| 5 | วัดผล After แล้วเทียบ Before เชิงตัวเลขหรือยัง? | ตาม Step 978 | Metric เป้าหมายดีขึ้นจริง **และ** Metric อื่นไม่ถดถอย |
| 6 | บันทึก Change Record ไว้เป็น Baseline รอบต่อไปหรือยัง? | Commit + Audit Trail (Part 79) | พร้อมใช้เป็น Baseline Comparison รอบถัดไป ([Part 91 Step 904](part-091-advanced-troubleshooting-methodology.md)) |

### 977.2 ทำไม Canary ถึงสำคัญกับ Optimize มากกว่า Deploy

Deploy สร้างของใหม่ที่ยังไม่มี User พึ่งพา — Canary ช่วยแค่จับ Bug ก่อน Go-Live เต็มรูป ส่วน
**Optimize แก้ของที่ User พึ่งพาอยู่แล้วจริง** — Canary ที่นี่คือ **เส้นสุดท้ายที่กันไม่ให้ Blast
Radius ของ Regression ลามไปทั้ง Site** เช่น Step 972 เลือกเปลี่ยน `REGION-EDGE1` ก่อน `REGION-EDGE2`
(มี Path สำรองให้ Traffic วิ่งระหว่างเปลี่ยน) และ Step 976 เลือก `region_canary` เป็นอุปกรณ์เดียว
ก่อนขยาย — ทั้งคู่คือหลักการเดียวกัน

### 977.3 กับดัก: "ทดสอบผ่านใน Lab" ≠ "ปลอดภัยใน Production"

Lab Environment ไม่มี Production Traffic จริงวิ่งขนาน (Real BGP Table เต็ม, Real User Load) — การ
Verify ต้องทำใน Production จริงเสมอ (ผ่าน Canary ที่จำกัด Blast Radius ไว้แล้ว) ไม่ใช่แค่ Lab จำลอง
เพราะ **Regression บางชนิดโผล่มาเฉพาะภายใต้ Load จริงเท่านั้น** (เช่น CoPP Rate ที่ตั้งจาก Lab Traffic
เบาบางอาจ Drop จริงเมื่อเจอ SNMP Sweep ของจริงที่ Production — ตรงกับเหตุผลที่ Step 975 ใช้ Monitor
Mode สังเกต Production Traffic จริงก่อน Enforce)

---

## Step 978 — การวัดผลก่อน/หลังจริง: Convergence, CPU, Throughput

### 978.1 หลักการ: "ฉันทำการเปลี่ยนแปลงแล้ว" ไม่ใช่หลักฐาน — ต้องมีตัวเลข

Optimize ที่ไม่มีตัวเลขก่อน/หลังเทียบกัน **พิสูจน์ไม่ได้เลยว่าดีขึ้นจริง** — อาจดีขึ้น, อาจเท่าเดิม,
หรือแม้แต่แย่ลงในบางมิติที่ไม่ได้สังเกต (ตรงกับคำเตือนของ Step 979) หลักฐานเชิงตัวเลขจึงเป็นส่วนบังคับ
ของงาน Optimize ทุกงาน ไม่ใช่ทางเลือก

### 978.2 Convergence Time — ตัวอย่างจริงจาก Task 1 (Step 972)

```
! ก่อนแก้ — Flap ที่ REGION-ACC1 วัดผลกระทบที่ CORE-SW1 (HQ)
CORE-SW1# show ip ospf statistics
   Intra-area SPF calculations: 218     ! ก่อน Flap ทดสอบ
! ... จำลอง Flap ที่ REGION-ACC1 ...
CORE-SW1# show ip ospf statistics
   Intra-area SPF calculations: 219     ! เพิ่มขึ้น 1 ครั้ง — HQ ถูกกระทบจาก REGION โดยตรง

! หลังแก้ (Area 90 + Summarization) — ทำ Flap เดียวกันซ้ำ
CORE-SW1# show ip ospf statistics
   Intra-area SPF calculations: 340     ! ก่อน Flap ทดสอบ
! ... จำลอง Flap ที่ REGION-ACC1 (เหตุการณ์เดียวกัน) ...
CORE-SW1# show ip ospf statistics
   Intra-area SPF calculations: 340     ! ไม่เปลี่ยน — HQ ไม่ถูกกระทบอีกต่อไป
```

### 978.3 CPU Utilization

```
CORE-SW1# show processes cpu history
                                            50
                                            40
                            #               30
                        #   #   #           20   ! ก่อนแก้: Spike สั้นๆ ทุกครั้งที่ REGION Flap
                    #   #   #   #   #       10
        0....5....1....1....2....2....3
                  0    5    0    5    0

! หลังแก้: ไม่มี Spike สัมพันธ์กับ REGION Flap อีกต่อไป (Baseline เดิมของ HQ เองล้วนๆ)
```

### 978.4 Throughput/Jitter — สรุปตัวเลขจาก Task 2 (Step 973)

| Metric | Before | After | หมายเหตุ |
|---|---|---|---|
| Voice Jitter (REGION↔HQ) | 42.8 ms | 6.4 ms | ต่ำกว่า Threshold MOS ที่ยอมรับได้ (30 ms) |
| Voice Packet Loss | 3.1% | 0.0% | |
| Data Throughput (Backup Job) | 18.2 Mbps | 17.6 Mbps (-3.3%) | ยังอยู่ในเกณฑ์ที่โจทย์กำหนด (< 5%) |

### 978.5 ทำไม Telemetry (Part 54/80) ดีกว่าการวัดแบบ Manual Snapshot

การรัน `show` คำสั่งเดียวก่อน/หลังคือ **จุดข้อมูล 2 จุดเท่านั้น** — ไม่รู้ว่าค่าระหว่างทางเสถียรหรือ
แค่บังเอิญดีตอนที่วัด [Part 54](part-054-network-assurance-telemetry.md) และ
[Part 80](part-080-model-driven-telemetry-streaming.md) สร้าง Pipeline Streaming Telemetry ที่เก็บ
`process-cpu-oper`, OSPF SPF Event, Interface Counter แบบต่อเนื่องอยู่แล้ว — การ Subscribe Path
เดิมที่มีอยู่แล้วช่วงก่อน/หลัง Optimize ให้ **กราฟ Time-series เต็มรูป** แทน Snapshot 2 จุด ทำให้เห็น
ว่าผลลัพธ์เสถียรจริงตลอดช่วง Soak Period ไม่ใช่ Fluke — และเป็นหลักฐานแนบ Change Record ตาม Step
977.1 ลำดับที่ 6 ได้โดยตรง

---

## Step 979 — Anti-Pattern ของการ Optimize

### 979.1 Anti-Pattern #1 — Optimize สิ่งที่ไม่ใช่ Bottleneck จริง

**ตัวอย่าง**: ผู้ใช้รายงาน "เสียงแตกตอนคุยโทรศัพท์" วิศวกรที่รีบสรุปไปที่ "OSPF Convergence ช้า" แล้ว
ใช้เวลาทั้งวันทำ Area Redesign (Step 972) ทั้งที่ Root Cause จริงคือ QoS ไม่มี Priority (Step 973)
— งาน OSPF Redesign อาจถูกต้องทางเทคนิค 100% แต่ **ไม่แก้ปัญหาที่รายงานมาเลย** เสียเวลาและ Risk
จาก Regression ไปโดยไม่ได้ประโยชน์ตรงจุด

> **ป้องกันได้ด้วยอะไร**: วินัย Root-Cause ตาม [Part 91](part-091-advanced-troubleshooting-methodology.md)
> ต้องมาก่อนการเลือกว่า "จะ Optimize อะไร" เสมอ — Optimize ที่ไม่ผ่าน Diagnosis ก่อนคือการเดา

### 979.2 Anti-Pattern #2 — Over-Optimization จนสูญเสีย Maintainability

| ตัวอย่าง | ผลลัพธ์ที่ดูดีบนกระดาษ | ต้นทุนจริงที่มองไม่เห็น |
|---|---|---|
| สร้าง QoS 12 Class ตาม Application ทุกตัวแยกกัน (Port-based) | Classify ละเอียดสุดๆ | ทุก Application ใหม่ต้องแก้ Class-map — ไม่มีใครกล้าแก้เพราะซับซ้อนเกินไป |
| สร้าง OSPF Area แยกทุก Access Switch (ไม่ใช่ทุก Site) | SPF Blast Radius เล็กสุด | จำนวน Area เยอะเกินจัดการ, ABR ต้องทำ LSA Translation มหาศาล, Debug ยากขึ้นกว่าประโยชน์ที่ได้ |
| ใช้ Route-Reflector 3 Tier สำหรับ Site ที่มีแค่ 4 Router | "Scalable ในอนาคต" | ซับซ้อนเกินความจำเป็นปัจจุบัน วิศวกรรุ่นถัดไป Debug ยากขึ้นโดยไม่มี Site ที่ 5 มาสมทบจริง |

### 979.3 Anti-Pattern #3 — Premature Optimization ที่สร้างปัญหาใหม่ (Cure Worse Than Disease)

**ตัวอย่างจริงที่พบบ่อย**: วิศวกรต้องการ "Convergence เร็วขึ้น" จึงปรับ OSPF `hello-interval 1` /
`dead-interval 3` ทั่วทั้ง Enterprise รวม Link REGION-EDGE1↔WAN-EDGE-1 ที่มี RTT WAN จริงประมาณ 40ms
และมี Jitter เป็นระยะ (ธรรมชาติของ WAN Circuit) — ผลลัพธ์: **Adjacency หลุดซ้ำๆ ทุกครั้งที่ Jitter
กระโดดเกิน Dead-Timer ที่แน่นเกินไป** ทำให้ SPF รันถี่กว่าตอนใช้ Timer Default เสียอีก — "การ Optimize"
กลับสร้างความไม่เสถียรมากกว่าปัญหาเดิมที่ไม่มีอยู่จริง (เพราะ Convergence เดิมก็ไม่ได้ช้าจนเป็นปัญหา
ตั้งแต่ต้น — Anti-Pattern #1 ซ้อนอยู่ในตัวอย่างนี้ด้วย)

> **จุดที่ Step 977's Checklist จะจับได้**: การ Soak/Canary ก่อน Rollout เต็มรูป (977.1 ลำดับ 3) จะ
> เห็น Adjacency Flap ที่ REGION-EDGE1 Canary ก่อนขยายไปทั่ว Enterprise — นี่คือเหตุผลที่ทุก Step ใน
> Part นี้ยืนกรานเรื่อง Staged Rollout และ Baseline Comparison ก่อนเสมอ ไม่ใช่พิธีกรรมที่ไม่มีประโยชน์

---

## Step 980 — Full Lab: Optimize 3 จุดทั่วทั้ง Enterprise Lab (HQ + REGION-DC)

### 980.1 โจทย์

> ท่านได้รับมอบหมายให้ตรวจสอบ Enterprise Lab ทั้งระบบ (HQ ตาม [00-ip-address-plan.md](00-ip-address-plan.md)
> รวม REGION-DC จาก Part 97-98) แล้วเลือก Optimize **3 จุดที่มีเหตุผลรองรับชัดเจนที่สุด** — แต่ละจุด
> ต้องมี: (1) เหตุผลว่าทำไมจุดนี้คุ้มค่าที่จะ Optimize จริง (ไม่ใช่ Anti-Pattern ตาม Step 979),
> (2) Before/After Config เต็มรูป, (3) Non-Regression Verification, (4) หลักฐานตัวเลขว่าดีขึ้นจริง

### 980.2 Topology รวม (HQ + REGION-DC)

```
                         ┌───────────┐        ┌───────────┐
                         │ CORE-SW1  │◄──Po1─►│ CORE-SW2  │      (HQ RR Tier 1)
                         └─────┬─────┘        └─────┬─────┘
                    ┌──────────┼──────┐   ┌─────────┼──────────┐
              ┌─────┴───┐ ┌────┴────┐ │   │   ┌──────┴──┐ ┌─────┴───┐
              │DIST-SW1 │ │DIST-SW2 │ │   │   │DIST-SW3 │ │DIST-SW4 │
              └─────────┘ └─────────┘ │   │   └─────────┘ └─────────┘
                                       │   │
                                 ┌─────┴───┴─────┐
                                 │  WAN-EDGE-1/2  │
                                 └───────┬────────┘
                                         │  10.255.91-92.0/30
                                 ┌───────┴────────┐
                                 │REGION-EDGE1/2  │  (RR Tier 2 Client ของ HQ)
                                 └───────┬────────┘
                                 ┌───────┴────────┐
                                 │REGION-CORE1/2  │  (RR Tier 2, Area 90 ABR)
                                 └───────┬────────┘
                                    REGION-ACC1
```

### 980.3 การเลือก 3 จุด — พร้อมเหตุผลว่าไม่ใช่ Anti-Pattern

| # | จุดที่เลือก Optimize | เหตุผลที่คุ้มค่า (ไม่ใช่ Anti-Pattern #1/#2) |
|---|---|---|
| A | Reference-bandwidth ให้ตรงกันทั่ว Enterprise รวม REGION (ต่อยอด Step 972) | ยืนยันด้วย Evidence จริง (`show ip ospf statistics` Step 978) ว่ามี SPF Blast Radius ปัญหาจริงเกิดขึ้น ไม่ใช่การเดา — และแก้แค่จุดเดียวที่มี Evidence ไม่ลามไปสร้างความซับซ้อนเกินจำเป็น (เลี่ยง #2) |
| B | Validate RR Hierarchy รวม HQ+REGION ครบวงจร (ต่อยอด Step 974) | Session Count และ Update Replication มีทิศทางลดลงจริงตามที่วัดได้ ไม่ได้เพิ่ม Tier เกินจำเป็น (คง 2 Tier ไม่ทำ 3 Tier ตาม 979.2) |
| C | Align CoPP Class Boundary ให้ REGION ตรงกับ Threshold จริงของ HQ (ต่อยอด Step 975) | ใช้ค่าจาก Soak Period จริง (975.3) ไม่ใช่ค่าคาดเดา — ตรงตามวินัย Monitor-Mode-ก่อน-Enforce |

### 980.4 Task A — Enterprise-wide Reference-Bandwidth Audit

```
! Script ตรวจสอบทุกอุปกรณ์ที่รัน OSPF Process 1 (ใช้โครง Part 47 compliance_check.py)
def check_reference_bandwidth(devices):
    mismatches = []
    for dev in devices:
        cfg = get_running_config(dev)
        if "auto-cost reference-bandwidth 100000" not in cfg:
            mismatches.append(dev)
    return mismatches

# ผลลัพธ์รันจริงกับ CORE-SW1/2, DIST-SW1-4, WAN-EDGE-1/2, REGION-EDGE1/2, REGION-CORE1/2:
# mismatches = []   ← หลัง Step 972 แก้ REGION-DC แล้ว ทุกอุปกรณ์ตรงกันครบทั้ง Enterprise
```

**Non-Regression**: Full-mesh Ping ระหว่างทุก Loopback0 (HQ 8 ตัว + REGION 4 ตัว) ก่อน/หลัง ต้อง
Success 100% เท่าเดิมทุกคู่ — Reference-bandwidth ไม่เปลี่ยน Reachability เปลี่ยนแค่ Cost/Path
Preference เท่านั้น

### 980.5 Task B — Validate RR Hierarchy ครบวงจร

```
! นับ Session ทั้งหมดใน AS 65001 ก่อน/หลังทั่ว Enterprise
CORE-SW1# show ip bgp summary | include 65001 | count
   Before: 6 (HQ full-mesh เดิมของ Part 30) + 10 (REGION Full-mesh Step 974 ก่อนแก้) = 16 Session รวม
   After:  6 (HQ RR เดิม ไม่เปลี่ยน) + 9 (REGION RR หลัง Step 974) = 15 Session รวม

! ยืนยัน Prefix ที่ REGION-ACC1 เห็นปลายทาง HQ ยังครบ (Non-Regression ระดับ End-to-End)
REGION-ACC1# traceroute 10.10.30.10
   1  10.90.10.1       ! REGION-CORE1 (HSRP)
   2  10.255.94.1      ! REGION-EDGE1
   3  10.255.91.1      ! WAN-EDGE-1
   4  10.10.30.10       ! Server1 ถึงปลายทางสำเร็จ เส้นทางเดิมไม่เปลี่ยน
```

### 980.6 Task C — CoPP Threshold Alignment

```
! เทียบ CIR ที่ REGION-EDGE1 (Step 975) กับ CORE-SW1 (HQ, Part 46 Step 454) — ปรับให้เป็นมาตรฐานเดียว
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap-c)#  police cir 1000000 bc 128000 conform-action transmit exceed-action drop
! (ปรับ COPP-NORMAL ของ HQ ให้ตรงกับค่าที่ REGION พิสูจน์แล้วว่าเหมาะสมจาก Soak Period จริง Step 975.3)
```

### 980.7 Optimization Scorecard สุดท้าย

| Metric | Before | After | Δ |
|---|---|---|---|
| SPF Run ที่ HQ ต่อ REGION Flap 1 ครั้ง | 1 (กระทบ) | 0 (ไม่กระทบ) | -100% |
| iBGP Session รวม (HQ+REGION) | 16 | 15 | -6.25% (แนวโน้มดีขึ้นเมื่อ Site เพิ่ม) |
| Voice Jitter REGION↔HQ | 42.8 ms | 6.4 ms | -85% |
| Data Throughput Regression | — | -3.3% | ผ่านเกณฑ์ (< 5%) |
| CoPP Legit-traffic False Drop (SNMP Sweep) | มี (Routing Flap) | ไม่มี | ผ่าน |
| Rollback ทดสอบจริงก่อน Deploy | — | ผ่านทุก Task | ตาม Step 977 Checklist |

ทั้ง 3 Task ผ่านทั้ง 2 เกณฑ์ของ Optimize Module: **เป้าหมายที่ระบุมาบรรลุจริงเชิงตัวเลข** และ
**Requirement เดิมทุกข้อยังผ่านครบ ไม่มี Regression แม้แต่จุดเดียว**

---

## แบบฝึกหัดทวนความเข้าใจ Part 98

1. อธิบายความแตกต่างของ "จุดเริ่มต้น (Starting State)" และ "ความเสี่ยงหลัก" ระหว่าง Diagnose, Deploy,
   และ Optimize Module — เพราะเหตุใด Optimize จึงเป็น Module เดียวที่ต้องกังวลเรื่อง Regression
   มากเป็นพิเศษ?
2. ใน Step 972 การสร้าง Area 90 ทำไมจึงต้องใช้คำสั่ง `area 90 range` **และ** `area 0 range` ที่
   ABR (`REGION-EDGE1`) พร้อมกันทั้ง 2 คำสั่ง — แต่ละคำสั่งทำหน้าที่ Summarize ทิศทางไหน?
3. ใน Step 974 เพราะเหตุใดการเพิ่ม `route-reflector-client` บน BGP Session ที่ Established อยู่แล้ว
   จึงไม่จำเป็นต้อง Reset Session เต็มรูป และทำไมต้องยืนยัน `PfxRcd` เท่าเดิมก่อนถอด Full-Mesh
   Session แต่ละคู่ (ไม่ใช่ถอดพร้อมกันทั้งหมด)?
4. อธิบายว่าทำไม CoPP ที่รวมทุก Class-map เป็น Class เดียวจึงทำให้ SNMP Polling Burst กลายเป็นสาเหตุ
   ที่ OSPF/BGP Neighbor หลุดได้ ทั้งที่ SNMP ไม่เกี่ยวข้องกับ Routing Protocol โดยตรงเลย?
5. ยกตัวอย่าง Anti-Pattern ทั้ง 3 แบบจาก Step 979 (ไม่ใช่ Bottleneck จริง / Over-Optimization /
   Premature Optimization) และอธิบายว่า Checklist ใน Step 977 ช่วยป้องกัน Anti-Pattern แบบที่ 3
   (Premature Optimization) ได้อย่างไรก่อนที่จะสร้างความเสียหายทั่วทั้ง Enterprise?

**เฉลย:**

1. **Diagnose** เริ่มจากสถานะเสีย/ผิดปกติ ความเสี่ยงหลักคือวินิจฉัยผิดจุดทำให้เสียเวลา ไม่มีอะไรจะ
   "พังเพิ่ม" เพราะพังอยู่แล้ว **Deploy** เริ่มจากไม่มีอะไรเลย ความเสี่ยงหลักคือ Design ผิดหรือลืม
   Requirement แต่ยังไม่มี Production Traffic พึ่งพา **Optimize** เริ่มจากระบบที่ **ทำงานได้ดีอยู่แล้ว
   และมี Production Traffic วิ่งอยู่จริง** ความเสี่ยงหลักคือ Regression — ทุกคำสั่งที่พิมพ์มีโอกาส
   ทำให้สิ่งที่ทำงานอยู่แล้วหยุดทำงาน ซึ่งเป็นความเสี่ยงที่ไม่มีใน 2 Module ก่อนหน้าเลย
2. `area 90 range 10.90.0.0 255.255.0.0` Summarize เส้นทางที่เป็น**สมาชิกภายใน Area 90** (REGION-DC)
   ก่อนส่งเข้า Area 0 — ทิศทาง **REGION → HQ** ส่วน `area 0 range 10.255.0.0/10.10.0.0 ...`
   Summarize เส้นทางที่เป็น**สมาชิกภายใน Area 0** (HQ Backbone) ก่อนส่งเข้า Area 90 — ทิศทาง
   **HQ → REGION** ทั้ง 2 คำสั่งอยู่บน ABR ตัวเดียวกันแต่ทำงานคนละทิศทาง ต้องมีทั้งคู่เพื่อให้ทั้ง 2
   ฝั่งเห็น Summary Route แทน Route ย่อยจำนวนมาก
3. เพราะ `route-reflector-client` เป็นแค่ Attribute ที่กำหนดว่า Router จะ **Reflect** Route ที่ได้รับ
   ต่ออย่างไร ไม่ใช่การเปลี่ยน Session Parameter (AS/IP) ที่ต้องสร้าง TCP Session ใหม่ จึงใช้แค่
   `clear ip bgp <neighbor> soft out` (Refresh ขาออกอย่างเดียว) พอ ส่วนการยืนยัน `PfxRcd` เท่าเดิม
   ก่อนถอด Full-Mesh แต่ละคู่เป็นเพราะต้องพิสูจน์ว่า **เส้นทางใหม่ผ่าน RR ทำงานสมบูรณ์แทนที่ Full-Mesh
   ได้แล้วจริง** ก่อนตัด Path เดิมทิ้ง — ถอดพร้อมกันทั้งหมดเสี่ยงเสีย Reachability ชั่วคราวหากมี Path
   ใดที่ RR ยังไม่ Reflect ครบ ซึ่งจะไม่รู้ตัวจนกว่า Traffic จริงจะขาดไปแล้ว
4. เพราะ CoPP Class เดียวใช้ Policer Bucket เดียวรวมกันสำหรับทุกประเภท Traffic Control-Plane —
   เมื่อ SNMP Full Sweep ส่ง Packet จำนวนมากจนรวมกับ Traffic อื่นเกิน CIR ของ Bucket เดียวนั้น
   `exceed-action drop` จะ Drop Packet ที่เกินโดยไม่สนใจว่า Packet นั้นเป็น SNMP หรือ OSPF Hello —
   OSPF/BGP จึงกลายเป็น Collateral Damage จาก Traffic คนละประเภทที่ไม่เกี่ยวข้องกันเลยในทางตรรกะ
   การแยก Class ตามประเภท Traffic ทำให้แต่ละประเภทมี Bucket อิสระของตัวเอง Burst ของ SNMP กระทบแค่
   Bucket ของ SNMP เท่านั้น
5. **#1**: OSPF Area Redesign ทั้งวันทั้งที่ Root Cause จริงของ "เสียงแตก" คือ QoS ไม่มี Priority
   **#2**: สร้าง QoS 12 Class ตาม Application ทุกตัว ทำให้ดูแลยากเกินความจำเป็นเทียบกับประโยชน์ที่ได้
   **#3**: ปรับ OSPF Timer แน่นเกินไปทั่ว Enterprise จนกระทบ Link WAN ที่มี Jitter ธรรมชาติ ทำให้
   Adjacency หลุดถี่กว่าเดิม — Checklist ของ Step 977 (โดยเฉพาะลำดับ 3 Staged/Canary Rollout) ป้องกัน
   Anti-Pattern แบบที่ 3 ได้เพราะ **จะเห็น Adjacency Flap ที่ Canary Device/Link ตัวแรกก่อน** ที่จะ
   ขยายการเปลี่ยนแปลงไปทั่วทั้ง Enterprise — ความเสียหายจึงถูกจำกัดอยู่แค่ Canary จุดเดียว ตรวจพบได้
   ก่อนสร้างผลกระทบระดับ Enterprise-wide

---

## สรุป Part 98

Part นี้ปิดท้าย CCIE Full Lab Scenario Block (Part 95-98) ด้วยทักษะที่ต่างจาก 3 Module ก่อนหน้า
โดยสิ้นเชิง — **Step 971** วางกรอบว่า Optimize คือการทำสิ่งที่ทำงานอยู่แล้วให้ดีขึ้นโดยมี Regression
Risk เป็นแกนกลาง, **Step 972** Optimize OSPF จาก Area 0 เดียวสู่ Area Boundary ที่ถูกต้องพร้อม
Summarization 2 ทิศทาง, **Step 973** Optimize QoS จาก Single-Class สู่ MQC 4-Class พร้อม Shaping
และปรับ EtherChannel Load-Balance, **Step 974** Optimize BGP จาก Full-Mesh สู่ RR Hierarchy ด้วย
Transition ที่ไม่ Drop Session แม้แต่ครั้งเดียว, **Step 975** Optimize CoPP จาก Class เดียวสู่
Granular Model พร้อม TrustSec SGACL แทน ACL กว้าง, **Step 976** Optimize กระบวนการ Automation จาก
Ad-hoc Script สู่ Pipeline โตเต็มที่ (ตัวอย่างการ Optimize ที่ไม่แก้ Config เลยแม้แต่บรรทัดเดียว),
**Step 977** สรุป Checklist วินัยที่ใช้ยืนยันว่า "ไม่พัง" ก่อน/ระหว่าง/หลัง Deploy Change, **Step
978** สอนวัดผลจริงเชิงตัวเลขด้วย Convergence/CPU/Throughput และ Telemetry แทนการอ้างแบบไม่มีหลักฐาน,
**Step 979** เตือน 3 Anti-Pattern ที่ทำให้ Optimize กลายเป็นการเสียเวลาหรือสร้างความเสียหายใหม่, และ
**Step 980** รวมทุกอย่างในการ Optimize 3 จุดจริงทั่วทั้ง Enterprise Lab (HQ + REGION-DC) พร้อม
Scorecard เชิงตัวเลขปิดท้าย

**บทเรียนที่สำคัญที่สุดของทั้ง Block (Part 95-98)**: ทักษะทั้ง 4 — Diagnose, Design, Deploy,
Optimize — ไม่ใช่ทักษะที่แยกจากกันโดยสิ้นเชิง แต่เป็น**วงจรเดียวกันที่วนซ้ำตลอดอายุของเครือข่ายจริง**
ระบบที่ Deploy วันนี้จะต้องถูก Optimize ในวันข้างหน้า และการ Optimize ที่ผิดพลาดก็สร้างสิ่งที่ต้อง
Diagnose ในวันถัดไปเช่นกัน — Enterprise Lab ที่หลักสูตรนี้สร้างต่อเนื่องมาตั้งแต่ Part 1 คือหลักฐาน
ของวงจรนี้เอง: ทุก Part ใหม่ทั้งแก้ปัญหาที่ Part ก่อนทิ้งไว้ และสร้างสิ่งใหม่ที่ Part ถัดไปจะต้อง
Optimize ต่อ

✅ **พร้อมสำหรับ Part 99**: จาก Enterprise Lab ที่สั่งสม Feature มาตั้งแต่ Part 1 จนถึงจุดที่ซับซ้อน
และผ่านการ Diagnose/Design/Deploy/Optimize มาครบวงจรแล้ว **Part 99 (Capstone: Build Enterprise
Network From Scratch)** จะพลิกกลับไปเริ่มต้นใหม่ทั้งหมด — คราวนี้ไม่มี Topology ที่สร้างไว้ให้แล้ว
ผู้เรียนจะต้องนำความรู้ทั้ง 98 Part มาออกแบบและสร้าง Enterprise Network ขึ้นมาเองตั้งแต่ศูนย์
บทพิสูจน์สุดท้ายว่าทักษะที่สั่งสมมาตลอดหลักสูตรนี้ใช้งานได้จริงโดยไม่มี "ของเดิม" ให้พึ่งพาอีกต่อไป

**ไปต่อ:** [Part 99 — Capstone: Build Enterprise Network From Scratch →](part-099-capstone-build-from-scratch.md)
