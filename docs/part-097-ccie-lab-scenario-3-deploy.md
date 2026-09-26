# Part 97 — CCIE Full Lab Scenario #3 — Deploy Module
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 961–970 จาก 1000 | CCIE Enterprise Infrastructure**

> ต่อจาก [Part 96 — CCIE Full Lab Scenario #2 (Design Module)](part-096-ccie-lab-scenario-2-design.md)
> ที่ฝึกทักษะ **"อ่าน Requirement แล้วออกแบบ Solution"** แบบไม่ต้อง Config จริงมากนัก, Part นี้คือ
> ขั้นถัดไปที่ตรงกับ **Deploy, Operate Module (~5 ชั่วโมงจาก 8 ชั่วโมง)** ของข้อสอบจริงตามที่ประกาศไว้
> ใน [Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) — โจทย์จะมาในรูปแบบ **Numbered Task
> List แบบข้อสอบจริง** (เช่น "1.1 Configure OSPF บน R1 และ R2 ให้ R1 เป็น DR ห้ามใช้ priority")
> ที่บังคับ **วิธีทำ** ไม่ใช่แค่ผลลัพธ์ — วัดว่าคุณรู้จัก Feature เดียวกัน **หลายวิธี** พอจะเลือกวิธีที่
> ตรงกับข้อจำกัดที่โจทย์ล็อกไว้หรือไม่ Part นี้ใช้ความรู้ Routing/Switching/Security/Services จาก
> Part 11-31, 56-65, 73-87 (Routing), Part 3-6, 18, 33-35 (Switching/HA), Part 14, 21-22, 44-46
> (Security), และ Part 15, 24, 40-41, 47-49 (Services/Automation) ทั้งหมดพร้อมกันในโจทย์เดียว
> ตามธรรมชาติของข้อสอบ CCIE จริง

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 961 | รูปแบบ Deploy Module ของข้อสอบจริง — Numbered Task List, ข้อจำกัดที่บังคับวิธีทำ, และวิธีอ่านโจทย์ |
| 962 | Task Group 1 (Routing) — 5 โจทย์สไตล์ข้อสอบจริงบน OSPF/BGP Backbone |
| 963 | Task Group 1 Model Answer — Config เต็มรูปแบบ พร้อมอธิบายกับดักของแต่ละข้อ |
| 964 | Task Group 2 (Switching/HA) — 5 โจทย์ STP/EtherChannel/FHRP |
| 965 | Task Group 2 Model Answer — Config เต็มรูปแบบ พร้อมอธิบายกับดักของแต่ละข้อ |
| 966 | Task Group 3 (Security) — 5 โจทย์ ACL/AAA/802.1X/TrustSec/CoPP |
| 967 | Task Group 3 Model Answer — Config เต็มรูปแบบ พร้อมอธิบายกับดักของแต่ละข้อ |
| 968 | Task Group 4 (Services/Automation) — 5 โจทย์ NAT/Multicast/EEM/RESTCONF/QoS |
| 969 | Task Group 4 Model Answer — Config เต็มรูปแบบ พร้อมอธิบายกับดักของแต่ละข้อ |
| 970 | Full Lab: Task Group 5 (โจทย์ผูกกันข้ามข้อ) + Model Answer + Deploy Module Exam Strategy |

---

## Step 961 — รูปแบบ Deploy Module ของข้อสอบจริง

### 961.1 ทำไมข้อสอบ CCIE ต้อง "บังคับวิธีทำ"

ข้อสอบ CCNP ถามแค่ "Config X ให้ทำงาน" — มีคำตอบเดียวที่ถูก ข้อสอบ CCIE Deploy Module **ถามแบบเดียวกัน
แต่ผูกเงื่อนไขเพิ่มเข้ามาเสมอ** เพราะ Feature ส่วนใหญ่ในโลก Cisco IOS-XE มีมากกว่า 1 วิธีที่ทำให้ผล
ลัพธ์ปลายทางเหมือนกัน (เช่น "ทำให้ Router A เป็น DR" ทำได้ทั้งจาก `ip ospf priority`, จาก OSPF
Router-ID, หรือจากลำดับการขึ้น Adjacency) — Cisco ออกแบบ Grading Rubric ของ Deploy Module ให้ตรวจ
ทั้ง **(1) ผลลัพธ์ปลายทางถูกไหม** และ **(2) วิธีที่ใช้ตรงกับข้อจำกัดที่ระบุไหม** เพราะในโลกจริง
Production Network มักมีข้อจำกัดที่ทำให้ "วิธีที่ง่ายที่สุด" ใช้ไม่ได้จริง (เช่น interface นั้นมี Track
Object ผูกอยู่แล้ว ห้ามแก้ Cost เพราะกระทบ Path อื่น, ห้ามแก้ Priority เพราะมี Automation Script อื่น
Monitor ค่านั้นอยู่) — โจทย์แบบนี้จึงวัด **ความรู้กว้างของวิธีทำ (Breadth of Mechanism)** ไม่ใช่แค่ความรู้
ลึกของ Feature เดียว

### 961.2 รูปแบบ Numbered Task ตามข้อสอบจริง

ข้อสอบจริงมาเป็น Task Group ใหญ่ (เช่น "1.x" = กลุ่ม Routing, "2.x" = กลุ่ม Switching) แต่ละ Task
ย่อยมีโครงสร้างประโยคที่ต้องแยกส่วนให้ออกทุกครั้ง:

```
┌─────────────────────────────────────────────────────────────────────┐
│  1.1  Configure OSPF on R1 and R2 such that R1 becomes the DR.        │
│       Do not use priority.                                            │
│                                                                        │
│       ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐    │
│       │ ผลลัพธ์ที่ต้องการ│   │ Scope/อุปกรณ์  │   │ ข้อจำกัดวิธีทำ (Trap)│   │
│       │ R1 = DR         │   │ R1, R2 เท่านั้น│   │ ห้ามใช้ priority     │   │
│       └──────────────┘   └──────────────┘   └───────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

ทุก Task ในข้อสอบจริงแยกเป็น 3 ส่วนเสมอแม้จะไม่ได้เขียนแยกบรรทัดให้เห็นชัด: **(1) ผลลัพธ์สุดท้ายที่
ต้อง Verify ได้จริง**, **(2) ขอบเขตอุปกรณ์/Interface ที่อนุญาตให้แก้ไข**, และ **(3) ข้อจำกัดของวิธีทำ**
— ผู้สอบที่พลาดบ่อยที่สุดคือคนที่อ่านแค่ส่วน (1) แล้วรีบ Config ด้วยวิธีที่คุ้นเคยที่สุดโดยไม่เช็ค
ส่วน (3) เลย

### 961.3 กับดักคลาสสิก 4 แบบที่ Deploy Module ชอบใช้

| แบบกับดัก | ตัวอย่างข้อจำกัด | เจตนาที่วัด |
|---|---|---|
| **"ห้ามใช้ Command ที่คุ้นที่สุด"** | "ห้ามใช้ `ip ospf priority`" ทั้งที่เป็นวิธีปกติสุดในการคุม DR | รู้ Mechanism อื่นที่ให้ผลเดียวกันไหม (เช่น Router-ID) |
| **"ห้ามแก้ Parameter ที่ดูเหมือนเกี่ยวข้องที่สุด"** | "ห้ามแก้ OSPF Cost/AD" ทั้งที่เป็นวิธีปกติสุดในการคุม Path Preference | รู้ทางเลือกอ้อม (เช่น `bandwidth` statement) และเข้าใจ Trade-off ของมันไหม |
| **"จำกัด Scope ที่แก้ได้"** | "ห้ามแก้ฝั่ง Neighbor" / "ต้องทำใน ACL เดิม ห้ามลบ/สร้างใหม่" | รู้จักแก้ปัญหาจากจุดเดียวที่อนุญาตไหม ไม่ใช่ทางลัดที่ต้องแก้สองฝั่ง |
| **"ผูกเงื่อนไข Failure/Trigger"** | "ต้องกลับมาทำงานได้เองภายใน X วินาที โดยไม่ใช้ Static Route" | รู้ Feature ระดับ HA/Automation จริง ไม่ใช่แค่ Static Workaround |

> **หลักอ่านโจทย์ที่ต้องท่องจำ**: เจอคำว่า **"ห้าม" (do not)** หรือ **"ต้อง" (must)** ให้ขีดเส้นใต้
> ทันทีและถือเป็น **Hard Constraint ที่ Grading Rubric ตรวจแบบ Binary** (ทำผิดข้อจำกัดข้อเดียว
> = คะแนน Task นั้นเป็น 0 ทั้งข้อ แม้ผลลัพธ์ปลายทาง Ping ผ่านก็ตาม) — นี่ตรงกับหลักที่สอนไว้แล้วใน
> [Part 71 Step 708](part-071-ccie-lab-blueprint-overview.md) เรื่องกับดักคำศัพท์เฉพาะ

### 961.4 โครงสร้างของ Part นี้ — Baseline Topology ร่วม

ทุก Task Group ในเนื้อหานี้ใช้ **Enterprise Lab Topology หลักของหลักสูตร** (อ้างอิงจาก
[00-ip-address-plan.md](00-ip-address-plan.md)) แต่แต่ละกลุ่มจะระบุ **"Baseline ที่มีอยู่แล้วก่อนเริ่ม
ทำโจทย์"** ไว้ชัดเจนก่อนโจทย์เสมอ — เหมือนข้อสอบจริงที่ให้ Running-config เดิมมาก่อนแล้วค่อยถามงาน
เพิ่ม ผู้เรียนต้อง **สมมติว่า Baseline ที่ระบุนั้น Config ไว้แล้วสมบูรณ์** ก่อนเริ่มอ่าน Task แต่ละข้อ

แต่ละ Task Group จัดสรรเวลาไว้ที่ **45-60 นาที** (รวม 4 กลุ่ม ≈ 3.5-4 ชั่วโมง จากเวลา Deploy Module
เต็ม ~5 ชั่วโมง เหลือเวลาให้ Diagnose Ticket ที่มักฝังมาด้วยตามที่บอกไว้ใน
[Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md)) — จับเวลาตัวเองทุกครั้งที่ฝึกจริง

---

## Step 962 — Task Group 1 (Routing): 5 โจทย์บน OSPF/BGP Backbone

### Baseline ที่มีอยู่แล้ว (ห้ามแก้ไขส่วนนี้ นอกจากโจทย์จะสั่งให้แก้)

```
- OSPF Process 1, Router-ID = Loopback0 (ตาม 00-ip-address-plan.md) รันอยู่บน:
    Area 0   : CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2
    Area 20  : DIST-SW3, DIST-SW4  (ABR ของ Area 20 = CORE-SW2, ออกแบบตาม Part 73)
- Dual-homed DIST-SW1 ↔ CORE (เพิ่มเข้ามาตาม Part 33-35):
    DIST-SW1 ↔ CORE-SW1 : 10.255.10.0/30 (Te1/1/1 ↔ Te1/0/1)   <- Path เดิม
    DIST-SW1 ↔ CORE-SW2 : 10.255.15.0/30 (Te1/1/2 ↔ Te1/0/3)   <- Path ใหม่ (ECMP เท่ากันตอนนี้)
- WAN-EDGE-1 (Gi0/0/2 → CORE-SW1) และ WAN-EDGE-2 (Gi0/0/2 → CORE-SW2) อยู่ใน Area 0 เดียวกัน
- eBGP: WAN-EDGE-1 (AS 65001) ↔ ISP-RTR (AS 65000) ผ่าน 203.0.113.0/30
        WAN-EDGE-2 (AS 65001) ↔ ISP-RTR (AS 65000) ผ่าน 203.0.113.4/30
        ISP-RTR ส่ง 0.0.0.0/0 มาให้ทั้งสอง Session จริง (ISP อนุญาต Default Route ตามปกติ)
- iBGP: WAN-EDGE-1 ↔ WAN-EDGE-2 ผ่าน Cross-link 10.10.254.8/30
```

### โจทย์

**1.1** Configure OSPF ระหว่าง `CORE-SW1` และ `DIST-SW1` บน Link `10.255.10.0/30` ให้ `CORE-SW1`
เป็น **DR เสมอ** และ `DIST-SW1` **ต้องไม่เป็น DR แม้หลัง `CORE-SW1` Reload** — **ห้ามใช้คำสั่ง
`ip ospf priority` บนอุปกรณ์ทั้งสองตัว**

**1.2** Redistribute Default Route ที่ `WAN-EDGE-1` และ `WAN-EDGE-2` เรียนรู้ผ่าน eBGP จาก `ISP-RTR`
เข้าสู่ OSPF Area 0 บนทั้งสองตัว ให้ `CORE-SW1`/`CORE-SW2` เห็น Default Route 2 เส้นทาง Equal-cost
เพื่อ Redundancy — **ห้ามใช้ `default-information originate always`** และ **ห้าม Config Static
Default Route บน `WAN-EDGE-1`/`WAN-EDGE-2`**

**1.3** ให้ `DIST-SW1` เลือก Path ผ่าน `CORE-SW1` เป็นหลักเมื่อทุกอย่างปกติ แต่ Failover ไป `CORE-SW2`
โดยอัตโนมัติทันทีที่ Link ไป `CORE-SW1` ล้ม — **ห้ามแก้ไข OSPF Cost บน Interface ใดๆ**
(`ip ospf cost`) และ **ห้ามแก้ไข Administrative Distance**

**1.4** บน `WAN-EDGE-1` ให้ Route ที่เรียนรู้จาก OSPF (จาก Core/Distribution) มีความสำคัญเหนือ Route
เดียวกันที่เรียนรู้จาก eBGP ของ `ISP-RTR` เสมอ (กรณีมี Backdoor Prefix ซ้ำกัน) — **ห้ามแก้ไข Local
Preference, Weight, หรือ AS-Path ของ BGP** และ **ห้าม Redistribute OSPF เข้า BGP**

**1.5** ป้องกันไม่ให้ `DIST-SW3`/`DIST-SW4` (Area 20) เห็น Host Route ของ Loopback0 ของ `WAN-EDGE-1`
และ `WAN-EDGE-2` โดยเฉพาะ (`1.1.1.21/32`, `1.1.1.22/32`) แต่ `DIST-SW1`/`DIST-SW2` (Area 0) ยังต้อง
เห็นครบทุก Route ตามปกติ — **ห้ามใช้ `area range` บน `CORE-SW1`** (เพราะไม่ใช่ ABR ของ Area 20)
และ **ห้ามใช้ `distribute-list` บน `DIST-SW3`/`DIST-SW4`**

---

## Step 963 — Task Group 1 Model Answer

### 1.1 — DR ด้วย Router-ID ไม่ใช่ Priority

**กับดักที่ต้องเลี่ยง**: คำตอบที่ "ดูเหมือนถูก" ที่สุดคือ `ip ospf priority 255` บน `CORE-SW1`
(หรือ `ip ospf priority 0` บน `DIST-SW1`) — **ทั้งสองวิธีนี้ต้องห้ามเด็ดขาดเพราะโจทย์ระบุ "ห้ามใช้
priority" ตรงตัว** แม้จะได้ผลลัพธ์ DR ที่ถูกต้องก็ตาม Grading Rubric จะให้ 0 คะแนนทันทีถ้าเจอ
`ip ospf priority` ปรากฏในบรรทัดใดของ Config ที่ Deploy ไป

**หลักที่ใช้แก้จริง**: DR Election ตัดสินจาก **Priority ก่อน (ค่า Default = 1 เท่ากันทั้งคู่ ถ้าไม่แก้)
แล้วถึงจะดู Router-ID สูงสุดเป็นตัวตัดสิน** — ค่า Default Router-ID ของ `DIST-SW1` (`1.1.1.11`
จาก Loopback0) **สูงกว่า** ของ `CORE-SW1` (`1.1.1.1`) เมื่อเทียบเป็นเลขฐาน 32-bit ล้วน
(`0x0101010B` > `0x01010101`) ทำให้ปกติแล้ว `DIST-SW1` จะชนะ DR ไปโดยไม่มีการแก้ไขอะไร —
วิธีแก้คือกำหนด **OSPF Router-ID แบบ Manual บน `CORE-SW1` ให้สูงกว่า Router-ID ของ `DIST-SW1`**

```
! บน CORE-SW1
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# router-id 1.1.1.100
CORE-SW1(config-router)# end
CORE-SW1# clear ip ospf process
Reset ALL OSPF processes? [no]: yes
```

**Verify:**
```
CORE-SW1# show ip ospf interface TenGigabitEthernet1/0/1 | include DR|Router ID
  Process ID 1, Router ID 1.1.1.100, Network Type BROADCAST, Cost: 1
  DR is 10.255.10.1, Interface address 10.255.10.1
```

**ทำไมข้อนี้ทนต่อ CORE-SW1 Reload**: `router-id` เป็นค่าที่ Manual กำหนดไว้ใน Running/Startup-config
ไม่ผูกกับ Loopback0 อีกต่อไป ดังนั้นหลัง Reload OSPF Process จะ Initialize ด้วย Router-ID `1.1.1.100`
เดิมทุกครั้ง — เมื่อ Adjacency ขาดแล้วขึ้นใหม่ (DR เดิมหายไปจาก Segment ชั่วขณะ) จะเกิด DR Election
รอบใหม่ และ `CORE-SW1` ชนะซ้ำเสมอเพราะ Router-ID ยังสูงกว่า `DIST-SW1` เหมือนเดิม

### 1.2 — Conditional Default-Information Originate

**กับดักที่ต้องเลี่ยง**: `default-information originate always` จะ Advertise Default Route เข้า OSPF
**เสมอไม่ว่า `WAN-EDGE-1` จะมี Default Route ของตัวเองอยู่จริงหรือไม่** — ถ้า eBGP Session ไป
`ISP-RTR` ล้ม `WAN-EDGE-1` จะยังหลอกประกาศ Default Route เข้า Core ต่อไป ทำให้ Core ส่ง Traffic ไป
Black Hole ที่ `WAN-EDGE-1` แทนที่จะ Fail Over ไปทาง `WAN-EDGE-2` — นี่คือเหตุผลที่โจทย์ห้าม `always`
และห้าม Static Route ตัวช่วย (ซึ่งจะเกิดปัญหาเดียวกัน คือประกาศ Default แม้ Upstream ตายแล้ว)

**หลักที่ใช้แก้จริง**: `default-information originate` (ไม่มี `always`) จะประกาศ Default Route เข้า
OSPF **ก็ต่อเมื่อ Router นั้นมี Route `0.0.0.0/0` อยู่ใน Routing Table จริงของตัวเองเท่านั้น**
— เพราะ Baseline ระบุว่า `ISP-RTR` ส่ง `0.0.0.0/0` มาทาง eBGP จริง จึงไม่ต้องทำอะไรเพิ่มเพื่อให้มี
Default Route ใน RIB (BGP เติมให้เอง) เหลือแค่เปิด Conditional Origination:

```
! บน WAN-EDGE-1 และ WAN-EDGE-2 (Config เหมือนกัน)
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# default-information originate
WAN-EDGE-1(config-router)# end
```

**Verify:**
```
WAN-EDGE-1# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "bgp 65001", distance 20, metric 0, candidate default path

CORE-SW1# show ip route ospf | include ^O\*E2
O*E2  0.0.0.0/0 [110/1] via 10.255.10.1, 00:02:11, TenGigabitEthernet1/0/1
O*E2  0.0.0.0/0 [110/1] via 10.255.15.1, 00:02:09, TenGigabitEthernet1/0/3
```

ถ้า eBGP Session ล้ม Route `0.0.0.0/0` จะหลุดจาก RIB ของ `WAN-EDGE-1` โดยธรรมชาติ ทำให้
`default-information originate` **หยุดประกาศเองโดยอัตโนมัติ** — ตรงตามเจตนาของโจทย์เป๊ะ โดยไม่ต้อง
เขียน Logic เพิ่มเลย

### 1.3 — Path Preference ด้วย Bandwidth Statement ไม่ใช่ Cost/AD

**กับดักที่ต้องเลี่ยง**: วิธีปกติสุดคือ `ip ospf cost` บน Interface ไป `CORE-SW2` ให้สูงกว่า
หรือปรับ `distance ospf` — **ทั้งคู่ถูกห้ามตรงตัว**

**หลักที่ใช้แก้จริง**: OSPF คำนวณ Cost อัตโนมัติจากสูตร `Cost = Reference-Bandwidth ÷ Interface
Bandwidth` — ค่า `bandwidth` ของ Interface เป็นแค่ **สถิติที่ Router รายงาน/ใช้คำนวณ** (ไม่ใช่
`ip ospf cost` โดยตรง) แก้ที่ตรงนี้จึงไม่ผิดข้อจำกัดของโจทย์ตามตัวอักษร แม้จะได้ผลลัพธ์เดียวกันกับ
การแก้ Cost ตรงๆ:

```
! บน DIST-SW1 — ลด Bandwidth ที่ประกาศบน Interface ไป CORE-SW2 (Path สำรอง)
DIST-SW1(config)# interface TenGigabitEthernet1/1/2
DIST-SW1(config-if)# bandwidth 1000
DIST-SW1(config-if)# end
```

**Verify:**
```
DIST-SW1# show ip ospf interface TenGigabitEthernet1/1/1 | include Cost
  Process ID 1, Router ID 1.1.1.11, Network Type BROADCAST, Cost: 1
DIST-SW1# show ip ospf interface TenGigabitEthernet1/1/2 | include Cost
  Process ID 1, Router ID 1.1.1.11, Network Type BROADCAST, Cost: 100

DIST-SW1# show ip route 0.0.0.0
O*E2  0.0.0.0/0 [110/2] via 10.255.10.1, TenGigabitEthernet1/1/1   <- Path เดียวที่เลือก (Cost ต่ำกว่า)
```

ถ้า Link ไป `CORE-SW1` ล้ม OSPF จะ Converge ไปใช้ Path ผ่าน `CORE-SW2` (Cost 100 ยังต่ำกว่า Infinity)
โดยอัตโนมัติ — Failover สำเร็จตามโจทย์ **หมายเหตุสำคัญที่ต้องพูดถ้าถูกซัก**: การแก้ `bandwidth`
ยังกระทบค่าที่แสดงใน `show interfaces` (Utilization %), EIGRP Metric ถ้ามีรันคู่กัน, และ Shape/Police
เชิง QoS ที่อ้างอิง Bandwidth — ต้อง Disclose Trade-off นี้เสมอเมื่อ Deploy วิธีนี้จริงใน Production
(แม้ในข้อสอบจะ "ถูกตามข้อจำกัด" ก็ตาม)

### 1.4 — Distance BGP เพื่อคุม Preference ข้าม Protocol

**กับดักที่ต้องเลี่ยง**: แก้ Local Preference/Weight/AS-Path หรือ Redistribute OSPF เข้า BGP
เพื่อให้ Path OSPF ดู "ดีกว่า" — ทุกทางถูกห้ามตรงตัว

**หลักที่ใช้แก้จริง**: โจทย์ไม่ได้ห้าม **Administrative Distance** — ค่า Default AD ของ eBGP (20)
ต่ำกว่า OSPF (110) มาก ทำให้ Route จาก eBGP ชนะเสมอแม้ OSPF จะมี Route เดียวกัน แก้ได้ตรงจุดด้วย
`distance bgp`:

```
! บน WAN-EDGE-1
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# distance bgp 200 200 200
WAN-EDGE-1(config-router)# end
```

พารามิเตอร์ 3 ตัวคือ `<external-AD> <internal-AD> <local-AD>` — ตั้งทั้งสามเป็น 200 (สูงกว่า OSPF
110) ทำให้ Route ใดๆที่ Router มีทั้งจาก BGP และ OSPF พร้อมกัน OSPF จะชนะเสมอ โดยไม่กระทบ Attribute
ของ BGP เลยแม้แต่ตัวเดียว

**Verify:**
```
WAN-EDGE-1# show ip protocols | section bgp
  Distance: external 200 internal 200 local 200
WAN-EDGE-1# show ip route 10.10.10.0
Routing entry for 10.10.10.0/24
  Known via "ospf 1", distance 110, metric 3   <- ชนะ แม้มี BGP Route ซ้ำ Prefix เดียวกัน
```

> **คำเตือนที่ต้องพูดถึงถ้าถูกถามในห้องสอบ**: `distance bgp` มีผลกับ**ทุก Prefix**ของ BGP Process
> นั้น ไม่ใช่เฉพาะ Prefix ที่ซ้ำกับ OSPF เท่านั้น ถ้ามี Prefix ที่ BGP เรียนรู้แล้วไม่มีใน OSPF เลย
> (เช่น Internet Full Table) ค่า AD 200 จะไม่กระทบเพราะไม่มี Route คู่แข่งอยู่ดี จึงปลอดภัยสำหรับ
> Use Case นี้

### 1.5 — Area Filter-List ที่ ABR ที่ถูกต้อง

**กับดักที่ต้องเลี่ยง**: ลอง `area 20 range ...` บน `CORE-SW1` (ผิดทันทีเพราะ `CORE-SW1` ไม่ใช่ ABR
ของ Area 20 เลย — ABR ตัวจริงคือ `CORE-SW2` ตาม Baseline) หรือลอง `distribute-list` ที่
`DIST-SW3`/`DIST-SW4` (ผิดเพราะโจทย์ห้ามตรงตัว และในทางเทคนิค `distribute-list` ที่ Internal Router
กรองได้แค่ Routing Table ของตัวเอง ไม่ได้กรอง LSA ที่ Flood เข้ามาใน Area จริง — Router อื่นใน
Area 20 ยังเห็น LSA เดิมอยู่ดี)

**หลักที่ใช้แก้จริง**: คำสั่ง `area <id> filter-list prefix <name> in` ใช้ได้เฉพาะที่ **ABR ตัวจริง**
ของ Area นั้น เพื่อกรอง Type-3 Summary LSA ที่กำลังจะถูก Inject เข้าไปใน Area — ต้องทำที่ `CORE-SW2`:

```
! บน CORE-SW2 (ABR ตัวจริงของ Area 20)
CORE-SW2(config)# ip prefix-list WAN-LOOPBACKS-DENY seq 10 deny 1.1.1.21/32
CORE-SW2(config)# ip prefix-list WAN-LOOPBACKS-DENY seq 20 deny 1.1.1.22/32
CORE-SW2(config)# ip prefix-list WAN-LOOPBACKS-DENY seq 30 permit 0.0.0.0/0 le 32
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# area 20 filter-list prefix WAN-LOOPBACKS-DENY in
CORE-SW2(config-router)# end
```

**Verify:**
```
DIST-SW3# show ip route 1.1.1.21
% Network not in table                          <- ถูกกรองแล้ว

DIST-SW3# show ip route 1.1.1.1
O IA  1.1.1.1/32 [110/2] via 10.255.30.1, TenGigabitEthernet1/1/1   <- Route อื่นยังเห็นปกติ

DIST-SW1# show ip route 1.1.1.21
O IA  1.1.1.21/32 [110/3] via 10.255.10.1, TenGigabitEthernet1/1/1  <- Area 0 เห็นครบตามโจทย์
```

---

## Step 964 — Task Group 2 (Switching/HA): 5 โจทย์ STP/EtherChannel/FHRP

### Baseline ที่มีอยู่แล้ว

```
- ACCESS-SW1 --(2x Gi trunk)--> DIST-SW1   VLAN 10,20,99  (ยังเป็น Individual Link ไม่ Bundle)
- ACCESS-SW2 --(1x Gi trunk)--> DIST-SW2   VLAN 10,20,99
- ACCESS-SW3/4 --(trunk)--> DIST-SW3/4      VLAN 30,40,99
- DIST-SW1 = HSRP Active, DIST-SW2 = HSRP Standby สำหรับ VLAN10/VLAN99 (ตั้งมาตั้งแต่ Part 18)
- DIST-SW3 = จะถูกตั้งเป็น GLBP AVG สำหรับ VLAN40 (WIFI) ในโจทย์นี้ (ยังไม่มี FHRP บน VLAN40 มาก่อน)
```

### โจทย์

**2.1** สร้าง EtherChannel ระหว่าง `ACCESS-SW1` (`Gi1/0/1-2`) กับ `DIST-SW1` (`Gi1/1/1-2`) โดยต้อง
**Detect ได้ถ้า Neighbor Software ค้าง/Crash แต่ Physical Link ยัง Up อยู่ (Unidirectional)** แล้ว
Shutdown Port นั้นอัตโนมัติ — **ห้ามใช้ `channel-group mode on`** (Static Mode)

**2.2** ป้องกัน `ACCESS-SW2` ไม่ให้กลายเป็น STP Root Bridge ของ VLAN 10/20/99 **ได้เด็ดขาดไม่ว่ากรณี
ใด แม้จะมีคนเผลอตั้ง `spanning-tree vlan 10 priority 0` บน `ACCESS-SW2` เอง** — **ห้ามใช้
`spanning-tree vlan X root primary/secondary`** และ **ห้ามใช้ AAA Command Authorization จำกัดสิทธิ์
พิมพ์คำสั่ง**

**2.3** Config HSRP VLAN10 ให้ `DIST-SW2` (Standby) ยึด Active คืนอัตโนมัติเมื่อ Uplink ของ
`DIST-SW1` กลับมาใช้งานได้ แต่ `DIST-SW1` **ต้องรอ 60 วินาทีพอดีหลัง Uplink กลับมา ก่อนเริ่ม
Preempt** (กัน Black Hole ระหว่าง OSPF Reconverge) — **ห้ามปรับค่า `decrement` ของ `standby track`
ให้สูงกว่า Default (10)**

**2.4** จำกัด Trunk ระหว่าง `ACCESS-SW1` และ `DIST-SW1` ให้ผ่านได้เฉพาะ VLAN 10, 20, 99 (บล็อก
VLAN 30/40 และ VLAN ใหม่ใดๆที่อาจถูกสร้างขึ้นมาโดยไม่ตั้งใจ) — **ห้ามปิด DTP ทั้งหมด** (ต้องให้
Trunk Encapsulation Renegotiate ได้ถ้า Hardware เปลี่ยน) และ **ห้าม Config
`switchport trunk allowed vlan` ที่ฝั่ง `DIST-SW1`**

**2.5** Config GLBP สำหรับ VLAN40 (WIFI) ระหว่าง `DIST-SW3` (AVG) และ `DIST-SW4` ให้ทั้งสองตัว
Forward Traffic จริงพร้อมกัน (Load-Balance) แต่ `DIST-SW4` **ต้องรับโหลดไม่เกิน 25% ของ Client
ทั้งหมด** (เพราะ Uplink ไป `CORE-SW2` เล็กกว่า) — **ห้าม Map MAC Address รายตัวเอง** และ **ห้ามปิด
Load-Balancing ทั้งหมด**

---

## Step 965 — Task Group 2 Model Answer

### 2.1 — LACP + UDLD Aggressive สำหรับ Unidirectional Detection

**กับดักที่ต้องเลี่ยง**: `channel-group mode on` ผิดตรงตัวเพราะโจทย์ห้าม แต่ที่สำคัญกว่าคือ **แม้จะ
เปลี่ยนไปใช้ LACP (`mode active`) เพียงอย่างเดียว ก็ยังไม่ตอบโจทย์ครบ** — LACP ตรวจจับได้แค่ "ไม่มี
LACPDU เข้ามาเลย" (จะถอด Port ออกจาก Channel เฉยๆ) แต่ **ไม่ได้ออกแบบมาเพื่อตรวจ Unidirectional
Traffic โดยเฉพาะ** ต้องเสริมด้วย Feature ที่ทำหน้าที่นี้ตรงตัวคือ **UDLD**

```
! บน ACCESS-SW1 และ DIST-SW1 (Config เหมือนกันทั้งสองฝั่ง)
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1-2
ACCESS-SW1(config-if-range)# channel-group 1 mode active
ACCESS-SW1(config-if-range)# udld port aggressive
ACCESS-SW1(config-if-range)# exit
ACCESS-SW1(config)# interface Port-channel1
ACCESS-SW1(config-if)# switchport mode trunk
ACCESS-SW1(config-if)# switchport trunk allowed vlan 10,20,99
```

**Verify:**
```
ACCESS-SW1# show etherchannel 1 summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)        LACP        Gi1/0/1(P)  Gi1/0/2(P)

ACCESS-SW1# show udld GigabitEthernet1/0/1 | include Bidirectional|Aggressive
  Bidirectional state: Bidirectional
  Aggressive mode: Enabled
```

ถ้า Neighbor Software ค้างจนไม่ตอบ UDLD Echo กลับ (แต่ Physical Link ยัง Up) — UDLD Aggressive Mode
จะ Error-disable Port นั้นทันทีหลัง Retry ครบ (ต่างจาก UDLD Normal Mode ที่แค่ Log Warning เฉยๆ) —
ตรงตามเจตนา "Detect แล้ว Shutdown อัตโนมัติ" ของโจทย์

### 2.2 — Root Guard ที่ Upstream Port ไม่ใช่ที่ตัว ACCESS-SW2 เอง

**กับดักที่ต้องเลี่ยง**: `spanning-tree vlan X root primary` ผิดตรงตัวเพราะโจทย์ห้าม และที่สำคัญคือ
**Priority-based Protection ใดๆก็ตามที่พึ่งพา Priority ยังเอาชนะได้เสมอถ้ามีคนตั้ง Priority เป็น 0
บนอุปกรณ์นั้นจริง** (Priority 0 คือค่าต่ำสุดที่เป็นไปได้ ชนะทุก Priority Config อื่น) จุดที่ต้อง
เข้าใจคือ **การป้องกันต้องทำที่ Port ของอุปกรณ์ต้นน้ำ (`DIST-SW2`) ที่หัน "เข้า" `ACCESS-SW2`
เพราะเป็นจุดเดียวที่ไม่ต้องพึ่ง Priority ของ `ACCESS-SW2` เลย**

```
! บน DIST-SW2 — Port ที่ต่อลง ACCESS-SW2
DIST-SW2(config)# interface GigabitEthernet1/1/5
DIST-SW2(config-if)# spanning-tree guard root
```

**Verify (จำลองว่ามีคนตั้ง Priority 0 บน ACCESS-SW2):**
```
ACCESS-SW2(config)# spanning-tree vlan 10 priority 0

DIST-SW2# show spanning-tree interface GigabitEthernet1/1/5 detail | include Root Inconsistent
%SPANTREE-2-ROOTGUARD_CONFIG_CHANGE: Root guard enabled on port GigabitEthernet1/1/5.
%SPANTREE-2-ROOTGUARDBLOCK: Port GigabitEthernet1/1/5 tried to become non-designated in VLAN 10.
Moving to root-inconsistent state
```

Root Guard **ไม่สนใจค่า Priority ที่ Superior BPDU อ้างเลย** แค่เห็นว่า Port ที่ตั้ง Root Guard
กำลังจะได้รับ BPDU ที่ "ดีกว่า" Root ปัจจุบัน ก็จะ Block ทันทีเข้า Root-Inconsistent State — ตรงตาม
โจทย์ "ป้องกันได้แม้ `ACCESS-SW2` ตั้ง Priority 0 เอง"

### 2.3 — Preempt Delay Minimum โดยไม่แก้ Decrement

**กับดักที่ต้องเลี่ยง**: คำว่า "ต้องรอ 60 วินาทีก่อน Preempt" ทำให้คนจำนวนมากคิดว่าต้องปรับ
`decrement` ของ Tracking ให้สูงพอที่จะทำให้ Priority ตกช้าลง (ผิดเจตนา และโจทย์ห้ามตรงตัวด้วย) —
Cisco มี Command ที่ทำสิ่งนี้**ตรงตัว**อยู่แล้วโดยไม่เกี่ยวกับ Tracking Decrement เลย

```
! บน DIST-SW1
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# standby 10 priority 110
DIST-SW1(config-if)# standby 10 preempt delay minimum 60
DIST-SW1(config-if)# standby 10 track 1 decrement 10
DIST-SW1(config-if)# end
```

**Verify:**
```
DIST-SW1# show standby vlan 10 | include Preempt
  Preempt enabled, delay min 60 secs
```

`preempt delay minimum <seconds>` สั่งให้ Router รอตามวินาทีที่กำหนด **นับจากตอนที่ Track Object
กลับมา Up** ก่อนเริ่ม Preempt เข้ายึด Active — ค่า `decrement 10` เป็นค่า Default อยู่แล้ว (ไม่ได้
ปรับสูงขึ้น) ตรงตามข้อจำกัดทั้งสองข้อของโจทย์

### 2.4 — Allowed VLAN ฝั่งเดียวก็เพียงพอ

**กับดักที่ต้องเลี่ยง**: หลายคนเข้าใจผิดว่า `switchport trunk allowed vlan` ต้อง Config ทั้งสองฝั่ง
ของ Trunk ให้ตรงกันเสมอ (Best Practice จริงแนะนำแบบนั้น) แต่โจทย์ห้าม Config ฝั่ง `DIST-SW1` — ต้อง
เข้าใจว่า Command นี้เป็น **Local, Bidirectional Filter ต่อ Port เดียว** ไม่ใช่ Negotiation ที่ต้อง
ตกลงกันสองฝั่งแบบ DTP

```
! บน ACCESS-SW1 เท่านั้น
ACCESS-SW1(config)# interface Port-channel1
ACCESS-SW1(config-if)# switchport trunk allowed vlan 10,20,99
```

**Verify:**
```
ACCESS-SW1# show interfaces trunk | include Po1
Po1       10,20,99

! ทดสอบ: สร้าง VLAN 30 บน ACCESS-SW1 แบบไม่ตั้งใจ แล้วดูว่ารั่วไป DIST-SW1 หรือไม่
ACCESS-SW1(config)# vlan 30
DIST-SW1# show mac address-table vlan 30 interface Port-channel1
          (ไม่มีรายการ — Frame VLAN 30 ไม่เคยข้าม Trunk นี้ทั้งสองทิศทาง)
```

เพราะ `allowed vlan` กรองทั้ง Traffic ที่ `ACCESS-SW1` **จะส่งออก** (VLAN 30 จะไม่ถูก Tag ส่งออก
Port นี้เลย) และ Traffic ที่ **รับเข้า** (ถ้า `DIST-SW1` เผลอส่ง VLAN 30 ลงมา `ACCESS-SW1` ก็จะ Drop
เอง) — ผลคือ VLAN 30/40 ไม่ข้าม Trunk นี้ในทั้งสองทิศทาง โดยไม่ต้อง Config ที่ `DIST-SW1` เลย DTP
ก็ยัง Negotiate Encapsulation ได้ปกติเพราะไม่ได้แตะ `switchport nonegotiate`

### 2.5 — GLBP Weighted Load-Balancing

**กับดักที่ต้องเลี่ยง**: GLBP Default Load-Balancing Mode คือ `host-dependent` (ผูก Client ตาม
MAC Hash คงที่ 50/50 โดยประมาณ) — การพยายาม Map MAC รายตัวเพื่อคุมสัดส่วนผิดเจตนาโจทย์ (และโจทย์
ห้ามตรงตัว) วิธีที่ถูกต้องคือเปลี่ยนเป็น **Weighted Mode** แล้วถ่วงน้ำหนักตามสัดส่วนที่ต้องการ

```
! บน DIST-SW3 (ตั้งใจให้เป็น AVG)
DIST-SW3(config)# interface Vlan40
DIST-SW3(config-if)# glbp 40 ip 10.10.40.1
DIST-SW3(config-if)# glbp 40 priority 150
DIST-SW3(config-if)# glbp 40 weighting 150
DIST-SW3(config-if)# glbp 40 load-balancing weighted

! บน DIST-SW4
DIST-SW4(config)# interface Vlan40
DIST-SW4(config-if)# glbp 40 ip 10.10.40.1
DIST-SW4(config-if)# glbp 40 priority 100
DIST-SW4(config-if)# glbp 40 weighting 50
DIST-SW4(config-if)# glbp 40 load-balancing weighted
```

**Verify:**
```
DIST-SW3# show glbp vlan40 | include Weighting|Load
  Load balancing: weighted
  Weighting 150 (configured 150)

DIST-SW4# show glbp vlan40 | include Weighting|Load
  Load balancing: weighted
  Weighting 50 (configured 50)
```

สัดส่วน Weight `150:50` = `3:1` ทำให้ AVG แจก Client ประมาณ **75% ให้ `DIST-SW3`, 25% ให้
`DIST-SW4`** — ตรงตามเพดาน "ไม่เกิน 25%" ของโจทย์พอดี โดยไม่ต้อง Map MAC เองแม้แต่ตัวเดียว และ
Load-Balancing ยังทำงานอยู่ (ไม่ได้ปิด) เพียงแค่สัดส่วนไม่เท่ากัน

---

## Step 966 — Task Group 3 (Security): 5 โจทย์ ACL/AAA/802.1X/TrustSec/CoPP

### Baseline ที่มีอยู่แล้ว

```
- ACL Numbered "SALES-TO-SERVERS" บน DIST-SW1 (Vlan10 SVI, inbound) มีอยู่แล้ว (ตาม Part 25 Step 245):
    seq 10  permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    seq 20  permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
    seq 999 deny   ip  any any log
- ACCESS-SW3 (SERVERS VLAN30) ยังไม่มี 802.1X/MAB Config เลย
- WAN-EDGE-1 ยังใช้ Local Login เท่านั้น (ยังไม่มี TACACS+)
- TrustSec/SGT ยังไม่ได้เปิดใช้งานที่ใดในเครือข่ายเลย (ไม่มี ISE ใน Lab — ใช้ Static/Manual SGT)
- CoPP บน CORE-SW1 ยังเป็นค่า Default (ไม่มี Custom Policy)
```

### โจทย์

**3.1** เพิ่ม Rule ให้ SALES เข้าถึง Server2 (`10.10.30.20`) ผ่าน RDP (TCP 3389) ได้ โดย Rule ใหม่
ต้องถูกประเมิน**ก่อน** Line `deny log` เดิม — **ห้ามลบหรือแก้ไข Sequence Number เดิมที่มีอยู่แล้ว**
และ **ห้ามใช้ `ip access-list resequence`**

**3.2** เปิด 802.1X บน Port ของ `ACCESS-SW3` (VLAN30 SERVERS) ให้ Server ที่ไม่มี 802.1X Supplicant
ยังเข้าเครือข่ายได้ผ่านการยืนยันด้วย MAC Address แทน แต่ยังต้องกัน Device ที่ไม่ผ่านการยืนยันใดๆเลย
— และถ้า RADIUS Server ใช้งานไม่ได้เลย Port ต้อง **Fail-Open ไปยัง Guest VLAN ที่จำกัดสิทธิ์**
(ไม่ใช่ Fail-Closed/Block ทั้งหมด) — **ห้ามปิด 802.1X ทั้งระบบ** (`no dot1x`) และ **ห้ามปิด
`authentication port-control auto`**

**3.3** Config AAA Login บน VTY ของ `WAN-EDGE-1` ให้ใช้ TACACS+ เป็นหลัก และ Fallback ไป Local
Database **เฉพาะกรณี TACACS+ Server ติดต่อไม่ได้เลย** (ไม่ใช่กรณี Username/Password ผิด) —
**ห้ามใช้ `login local` ระดับ `line vty`** และ **ห้ามระบุ TACACS+ Server มากกว่า 1 ตัวในคำสั่ง
Method List เอง** (สมมติว่ามี Server-group ที่รวม Server ไว้แล้ว)

**3.4** Implement TrustSec SGT ให้ SALES (VLAN10, SGT10) เข้า SERVERS (VLAN30, SGT30) ได้เฉพาะ
HTTPS แต่ VOICE (VLAN20, SGT20) เข้า SERVERS ได้ทุก Protocol — Tag SGT ต้องติดที่ **ใกล้ต้นทางที่สุด
(Access Layer)** — **ห้ามใช้ Traditional VLAN ACL (`ip access-list` บน SVI) สำหรับ Policy นี้เลย**
(ต้องใช้ SGACL เป็นกลไกเดียว)

**3.5** Config CoPP บน `CORE-SW1` จำกัด SSH Management Traffic ที่เข้าตัวอุปกรณ์เองไม่เกิน 500kbps
โดย OSPF Hello/Dead ระหว่าง `CORE-SW1` กับ Neighbor **ต้องไม่ถูก Rate-limit หรือ Drop เด็ดขาด**
แม้ระหว่างมี Management Traffic Flood หนัก — **ห้ามวาง Class ที่ Match OSPF ไว้หลัง Class SSH ใน
Policy-map** และ **ห้ามใช้ Class-default สำหรับ OSPF**

---

## Step 967 — Task Group 3 Model Answer

### 3.1 — แทรก Sequence Number ในช่องว่าง

**กับดักที่ต้องเลี่ยง**: `ip access-list resequence` ดูสะดวกแต่โจทย์ห้ามตรงตัว (และในความเป็นจริง
Resequence ทั้งชุดมีความเสี่ยงถ้า ACL นั้นถูกใช้อ้างอิงจากที่อื่นด้วย Sequence Number)

```
DIST-SW1(config)# ip access-list extended SALES-TO-SERVERS
DIST-SW1(config-ext-nacl)# 21 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.20 eq 3389
DIST-SW1(config-ext-nacl)# end
```

**Verify:**
```
DIST-SW1# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
    21 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.20 eq 3389    <- แทรกใหม่ ก่อน deny เดิม
    999 deny ip any any log
```

ช่องว่างระหว่าง `20` ถึง `999` เปิดพอให้แทรก Sequence Number `21` ได้โดยไม่กระทบ Line เดิมแม้แต่
บรรทัดเดียว — นี่คือเหตุผลที่ Best Practice สอนให้เว้นช่อง Sequence Number ไว้เผื่อเสมอ (ตามที่สอน
ใน Part 14)

### 3.2 — MAB + Critical VLAN (IAB)

```
ACCESS-SW3(config)# aaa new-model
ACCESS-SW3(config)# aaa authentication dot1x default group radius
ACCESS-SW3(config)# aaa authorization network default group radius
ACCESS-SW3(config)# dot1x system-auth-control

! กำหนด Guest VLAN สำหรับกรณี RADIUS ตายสนิท
ACCESS-SW3(config)# vlan 900
ACCESS-SW3(config-vlan)# name CRITICAL-GUEST
ACCESS-SW3(config)# exit

ACCESS-SW3(config)# interface range GigabitEthernet1/0/10-20
ACCESS-SW3(config-if-range)# switchport mode access
ACCESS-SW3(config-if-range)# switchport access vlan 30
ACCESS-SW3(config-if-range)# authentication port-control auto
ACCESS-SW3(config-if-range)# authentication order dot1x mab
ACCESS-SW3(config-if-range)# authentication priority dot1x mab
ACCESS-SW3(config-if-range)# mab
ACCESS-SW3(config-if-range)# dot1x pae authenticator
ACCESS-SW3(config-if-range)# authentication event fail action next-method
ACCESS-SW3(config-if-range)# authentication event server dead action authorize vlan 900
ACCESS-SW3(config-if-range)# authentication event server alive action reinitialize
```

**Verify:**
```
ACCESS-SW3# show authentication sessions interface GigabitEthernet1/0/10
            Method status list:
            Method    State
            mab       Authc Success
```

`authentication order dot1x mab` + `priority dot1x mab` ทำให้ Port ลอง 802.1X ก่อน ถ้า Server
(Endpoint) ไม่ตอบ EAPOL เลยภายใน Timeout จะร่วงไปลอง MAB (ยืนยันด้วย MAC) แทนโดยอัตโนมัติ — Device
ที่ไม่มี MAC ลงทะเบียนไว้ใน RADIUS เลยจะยังคง**ไม่ผ่าน**ทั้งสองวิธี (ไม่ได้ Bypass ไปเฉยๆ) ส่วน
`authentication event server dead action authorize vlan 900` คือ **Critical VLAN / Inaccessible
Authentication Bypass** — เปิดใช้เฉพาะตอน RADIUS Server ทุกตัวไม่ตอบสนองจริงๆเท่านั้น (Fail-Open
แบบจำกัดสิทธิ์ ไม่ใช่ Fail-Closed)

### 3.3 — AAA Method List Fallback แบบ Native

**กับดักที่ต้องเลี่ยง**: หลายคนพยายามหาวิธี "บอก IOS ชัดๆ" ว่าให้ Fallback เฉพาะกรณี Server ไม่ตอบ
— แต่ความจริงคือ **นี่คือ Behavior Default ของ AAA Method List Chaining อยู่แล้วโดยไม่ต้อง Config
เพิ่ม**: IOS จะข้ามไป Method ถัดไปในลิสต์ **ก็ต่อเมื่อ Method ก่อนหน้าไม่ตอบสนอง (Timeout/
Unreachable)** เท่านั้น ถ้า Server ตอบกลับมาว่า Reject (Username/Password ผิด) IOS จะเชื่อคำตอบ
นั้นทันทีและ**ไม่ไป Fallback ต่อ**

```
WAN-EDGE-1(config)# aaa new-model
WAN-EDGE-1(config)# aaa authentication login VTY-AUTH group TACACS-SERVERS local
WAN-EDGE-1(config)# line vty 0 15
WAN-EDGE-1(config-line)# login authentication VTY-AUTH
```

**Verify:**
```
WAN-EDGE-1# show run | section aaa authentication login
aaa authentication login VTY-AUTH group TACACS-SERVERS local

WAN-EDGE-1# show run | section line vty
line vty 0 15
 login authentication VTY-AUTH
```

ไม่มี `login local` แยกที่ `line vty` เลย (ตามข้อจำกัด) และไม่ได้ระบุ IP ของ TACACS+ Server ซ้ำใน
Method List (อ้างผ่าน Server-group `TACACS-SERVERS` ที่มีอยู่แล้วเพียงชื่อเดียว ตามข้อจำกัด) —
พฤติกรรม Fallback-on-unreachable-only เกิดขึ้นเองจาก AAA Chain Logic มาตรฐานของ IOS

### 3.4 — SGT ที่ Access Layer + SGACL ที่ Enforcement Point

```
! บน ACCESS-SW1 และ ACCESS-SW2 — Tag SGT ที่ใกล้ต้นทางที่สุด (Manual, ไม่มี ISE)
ACCESS-SW1(config)# cts role-based sgt-map vlan 10 sgt 10
ACCESS-SW1(config)# cts role-based sgt-map vlan 20 sgt 20

! Trunk ขึ้น DIST ต้อง Manual Tag SGT แบบ Inline (CTS Manual Mode)
ACCESS-SW1(config)# interface Port-channel1
ACCESS-SW1(config-if)# cts manual
ACCESS-SW1(config-if-cts-manual)# policy static sgt 0 trusted

! บน DIST-SW3 — Enforcement Point หน้า SERVERS VLAN30 (SGT30)
DIST-SW3(config)# cts role-based enforcement
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# cts role-based enforcement

DIST-SW3(config)# cts role-based sgt-map vlan 30 sgt 30
DIST-SW3(config)# ip access-list role-based SALES-TO-SERVERS-HTTPS-ONLY
DIST-SW3(config-rb-acl)# permit tcp dst eq 443
DIST-SW3(config-rb-acl)# deny ip
DIST-SW3(config)# ip access-list role-based VOICE-TO-SERVERS-ANY
DIST-SW3(config-rb-acl)# permit ip

DIST-SW3(config)# cts role-based permissions from 10 to 30 SALES-TO-SERVERS-HTTPS-ONLY
DIST-SW3(config)# cts role-based permissions from 20 to 30 VOICE-TO-SERVERS-ANY
```

**Verify:**
```
DIST-SW3# show cts role-based permissions from 10 to 30
IPv4 Role-based permissions from group 10 to group 30:
        SALES-TO-SERVERS-HTTPS-ONLY

DIST-SW3# show cts role-based permissions from 20 to 30
IPv4 Role-based permissions from group 20 to group 30:
        VOICE-TO-SERVERS-ANY
```

SGT ถูก Map ตั้งแต่ `ACCESS-SW1`/`ACCESS-SW2` (Access Layer, ใกล้ Host ที่สุดตามโจทย์) แล้วเดินทาง
ผ่าน Trunk แบบ Inline Tagging (`cts manual` + `policy static sgt`) จนถึง Enforcement Point ที่
`DIST-SW3` ซึ่งใช้ **SGACL (Security Group ACL) ล้วนๆ** ผ่าน `cts role-based permissions` — ไม่มี
`ip access-list` แบบ VLAN ACL ธรรมดาเข้ามาเกี่ยวข้องกับ Policy นี้เลยตามข้อจำกัด

### 3.5 — CoPP: Class OSPF ต้องมาก่อน Class SSH เสมอ

**กับดักที่ต้องเลี่ยง**: การจัด Class ผิดลำดับใน Policy-map คือกับดักที่อันตรายที่สุดของ CoPP —
Policy-map จับคู่ Class **แบบ First-match ตามลำดับบนลงล่าง** ถ้า Class ที่ Match OSPF ถูกวางไว้
หลัง Class SSH (หรือใช้ Class-default ที่ครอบคลุมกว้างเกินไป) OSPF Packet บางส่วนอาจถูกจับเข้า Class
ผิดไปโดยไม่ตั้งใจถ้า ACL ของ Class ก่อนหน้าเขียนกว้างเกินคาด

```
CORE-SW1(config)# ip access-list extended COPP-OSPF
CORE-SW1(config-ext-nacl)# permit ospf any any
CORE-SW1(config-ext-nacl)# permit ip any host 224.0.0.5
CORE-SW1(config-ext-nacl)# permit ip any host 224.0.0.6

CORE-SW1(config)# ip access-list extended COPP-SSH
CORE-SW1(config-ext-nacl)# permit tcp any any eq 22

CORE-SW1(config)# class-map match-all CLASS-OSPF
CORE-SW1(config-cmap)# match access-group name COPP-OSPF
CORE-SW1(config)# class-map match-all CLASS-SSH
CORE-SW1(config-cmap)# match access-group name COPP-SSH

CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)# class CLASS-OSPF
CORE-SW1(config-pmap-c)# police 8000000 8000 conform-action transmit exceed-action transmit
CORE-SW1(config-pmap)# class CLASS-SSH
CORE-SW1(config-pmap-c)# police 500000 8000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap)# class class-default
CORE-SW1(config-pmap-c)# police 1000000 8000 conform-action transmit exceed-action drop

CORE-SW1(config)# control-plane
CORE-SW1(config-cp)# service-policy input COPP-POLICY
```

**Verify:**
```
CORE-SW1# show policy-map control-plane | begin CLASS-OSPF
  Class-map: CLASS-OSPF (match-all)
    police:
        cir 8000000 bps, bc 8000 bytes
      conformed 0 packets, ...  exceed-action: transmit    <- ไม่มีการ Drop เลยไม่ว่าจะเกิน Rate แค่ไหน

  Class-map: CLASS-SSH (match-all)
    police:
        cir 500000 bps, bc 8000 bytes
      conformed ... exceed-action: drop
```

`CLASS-OSPF` ถูกวางเป็น Class **แรก** ในลำดับ Policy-map (ก่อน `CLASS-SSH`) และตั้ง
`exceed-action transmit` (ไม่ Drop แม้เกิน Rate ที่ Police ไว้ — ใส่ Police สูงมากแทบเป็น Unlimited
เพื่อยังคง Track ปริมาณได้แต่ไม่ Drop จริง) ตรงตามข้อจำกัด "ต้องไม่ถูก Rate-limit หรือ Drop เด็ดขาด"
— Packet OSPF ที่ Match `CLASS-OSPF` ได้ก่อนจะไม่ตกไปถึงการพิจารณา `CLASS-SSH` หรือ `class-default`
อีกเลยไม่ว่ากรณีใด

---

## Step 968 — Task Group 4 (Services/Automation): 5 โจทย์ NAT/Multicast/EEM/RESTCONF/QoS

### Baseline ที่มีอยู่แล้ว

```
- WAN-EDGE-1 ยังไม่มี NAT Config เลย (Internet Egress ยังไม่ทำงานสำหรับ SERVERS/VOICE)
- Multicast (PIM) ยังไม่ได้เปิดที่ใดในเครือข่ายเลย — Video Server บน VLAN30 (10.10.30.50) ต้องส่ง
  Stream ไปกลุ่ม 239.1.1.1 ให้ WIFI Client บน VLAN40 (ผ่าน DIST-SW3/DIST-SW4) รับได้
- ACCESS-SW3 มี Port Security เปิดแล้ว (`violation shutdown`) แต่ไม่มี Automation ช่วย Recovery
- ACCESS-SW3 รองรับ RESTCONF อยู่แล้ว (`restconf` เปิดใน Global Config), Gi1/0/1 คือ Uplink ไป DIST-SW3
- QoS Trust Boundary ยังไม่ได้ Config ที่ ACCESS-SW1/ACCESS-SW2 เลย (Part 24 ยังไม่ Deploy จริงใน Lab นี้)
```

### โจทย์

**4.1** Config NAT บน `WAN-EDGE-1` ให้ SERVERS (VLAN30, `10.10.30.0/24`) NAT ออก Internet ด้วย
Public IP เดียว `203.0.113.2` เสมอ ส่วน VOICE (VLAN20, `10.10.20.0/24`) **ห้าม NAT เมื่อคุยกับ SIP
Trunk Provider Block `203.0.113.128/25` โดยเฉพาะ** แต่ยัง NAT ปกติเมื่อออก Internet ทั่วไป —
**ห้ามใช้ Route-map ในการทำ NAT นี้**

**4.2** เปิด PIM Sparse-Mode ให้ WIFI Client (VLAN40) รับ Stream จาก Video Server (VLAN30,
`239.1.1.1`) ได้ โดยใช้ Static RP ที่ `CORE-SW1` เป็นหลัก แต่ถ้า `CORE-SW1` ล้ม `CORE-SW2` ต้องทำหน้าที่
RP แทนได้ **โดยอัตโนมัติ ไม่ต้องแก้ Config ใดๆบน Router ตัวอื่น** — **ห้ามใช้ Auto-RP หรือ BSR**
(ต้องเป็น Static RP Config เท่านั้นทุกตัว)

**4.3** เขียน EEM Applet บน `ACCESS-SW3` ตรวจ Port ที่ Err-disable จาก Port Security Violation แล้ว
Log Syslog รายละเอียด (ชื่อ Interface, จำนวน Violation) — ถ้า Violation Count = 1 ครั้ง ให้เปิด Port
คืนอัตโนมัติหลัง 5 นาที แต่ถ้า **≥ 2 ครั้งในวันเดียวกัน ต้องคงปิดไว้รอ Manual** — **ห้ามใช้
`errdisable recovery cause psecure-violation` + `errdisable recovery interval` แบบ Global**

**4.4** ใช้ RESTCONF (ไม่ใช่ NETCONF/SSH) แก้ Description ของ `GigabitEthernet1/0/1` บน `ACCESS-SW3`
เป็น `UPLINK-TO-DIST-SW3-VERIFIED` พร้อมเปิด `logging event link-status` บน Interface เดียวกัน (Leaf
นี้มีเฉพาะใน `Cisco-IOS-XE-native`, ไม่มีใน `ietf-interfaces`) — ทั้งหมดต้องทำในคำขอ **PATCH เดียว**
และ **ห้ามกระทบ Parameter อื่นของ Interface** (VLAN, Trunk, Speed/Duplex เดิมต้องไม่เปลี่ยน)

**4.5** Config QoS ให้ VOICE (VLAN20) Mark เป็น DSCP EF ที่ `ACCESS-SW1`/`ACCESS-SW2` (ใกล้ต้นทาง
ที่สุด), Police เกิน 128kbps ต่อ Phone ให้ Drop ส่วนเกิน (ไม่ Remark ลง), ได้ Strict-Priority Queue
ทุก Hop จนถึง WAN Edge — พร้อมกันนั้น Background File Transfer ที่ Mark CS1 บน VLAN10 (SALES) ต้อง
**ไม่ถูก WIFI (VLAN40) แย่ง Bandwidth จนเหลือ 0% ระหว่าง Congestion** (ต้องมี Guaranteed Minimum) —
**ห้ามใช้ Policy-map มากกว่า 1 ตัวสำหรับงาน Classify+Mark+Police ที่ Ingress Access Port** (Nested
Policy-map ส่วนนี้ทำไม่ได้ — Queuing ที่ Egress แต่ละ Hop แยกเป็นคนละ Policy-map ได้ตามปกติ)

---

## Step 969 — Task Group 4 Model Answer

### 4.1 — NAT ACL แบบ Deny-Before-Permit ไม่ต้องพึ่ง Route-map

**กับดักที่ต้องเลี่ยง**: สถานการณ์ "NAT ยกเว้นบาง Destination" มักถูกสอนด้วย Route-map เพราะรองรับ
เงื่อนไขซับซ้อนกว่า — แต่กรณีนี้ทำได้ง่ายกว่าด้วย ACL ธรรมดาที่ใช้ `deny` เพื่อ **"ยกเว้นจากการ
Match"** (ACL ที่ใช้กับ `ip nat inside source list` ไม่ใช่ Security ACL — `deny` แปลว่า "ไม่ให้ NAT"
ส่วน `permit` แปลว่า "ให้ NAT")

```
WAN-EDGE-1(config)# ip access-list extended NAT-EXCEPTION-ACL
WAN-EDGE-1(config-ext-nacl)# deny   ip 10.10.20.0 0.0.0.255 203.0.113.128 0.0.0.127
WAN-EDGE-1(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 any
WAN-EDGE-1(config-ext-nacl)# permit ip 10.10.20.0 0.0.0.255 any

WAN-EDGE-1(config)# ip nat pool SERVERS-POOL 203.0.113.2 203.0.113.2 netmask 255.255.255.252
WAN-EDGE-1(config)# ip nat inside source list NAT-EXCEPTION-ACL pool SERVERS-POOL overload
```

**Verify:**
```
WAN-EDGE-1# show ip nat translations | include 10.10.30
tcp 203.0.113.2:24501  10.10.30.10:443  ...

WAN-EDGE-1# ping 203.0.113.130 source 10.10.20.15
! traceroute แสดง Source ยังเป็น 10.10.20.15 จริง (ไม่ถูก NAT) เมื่อปลายทางอยู่ใน 203.0.113.128/25
```

Line `deny` แถวแรกดักจับ Traffic VOICE→SIP Provider Block ก่อน ACL หยุดที่บรรทัดนั้นเลยและ**ไม่ทำ
NAT ให้ Traffic นี้** ส่วน Traffic อื่นทั้งหมด (SERVERS ทุกปลายทาง, VOICE ไปปลายทางอื่น) ยังถูก
Permit และ NAT ตามปกติ — ไม่ต้องมี Route-map เข้ามาเกี่ยวข้องเลย

### 4.2 — Anycast RP ด้วย Static RP + MSDP (ไม่ใช่ Auto-RP/BSR)

**กับดักที่ต้องเลี่ยง**: หลายคนเข้าใจผิดว่า Static RP "ไม่มีทาง Redundant ได้" เพราะเป็นค่าคงที่ —
ความจริงคือ Cisco รองรับ **Anycast RP** ซึ่งยังเป็น Static RP Config ล้วนๆบนทุก Router (ไม่มี
Auto-RP/BSR Message วิ่งเลย) โดยให้ RP ทั้งสองตัวใช้ IP เดียวกัน (ผ่าน Loopback ที่ Advertise เข้า
IGP) แล้วซิงค์ Source-Active State กันด้วย MSDP

```
! บน CORE-SW1 และ CORE-SW2 — สร้าง Anycast RP Loopback ที่มี IP เดียวกันทั้งคู่
CORE-SW1(config)# interface Loopback99
CORE-SW1(config-if)# ip address 1.1.1.99 255.255.255.255
CORE-SW1(config-if)# ip pim sparse-mode

CORE-SW2(config)# interface Loopback99
CORE-SW2(config-if)# ip address 1.1.1.99 255.255.255.255
CORE-SW2(config-if)# ip pim sparse-mode

! ทุก Router ในเครือข่าย (CORE-SW1/2, DIST-SW1-4, WAN-EDGE-1/2) — Static RP ชี้ไปที่ Anycast IP เดียวกัน
CORE-SW1(config)# ip pim rp-address 1.1.1.99
CORE-SW2(config)# ip pim rp-address 1.1.1.99
DIST-SW1(config)# ip pim rp-address 1.1.1.99
DIST-SW3(config)# ip pim rp-address 1.1.1.99
! (ทำเหมือนกันทุกตัวที่รัน PIM)

! MSDP Peering ระหว่าง CORE-SW1 กับ CORE-SW2 ด้วย Loopback0 จริง (ไม่ใช่ Anycast IP)
CORE-SW1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
```

**Verify:**
```
DIST-SW3# show ip pim rp mapping
Group(s): 224.0.0.0/4, Static
    RP: 1.1.1.99
      Info source: static

! จำลอง CORE-SW1 ล้ม
CORE-SW1# reload

DIST-SW3# show ip mroute 239.1.1.1 | include RP
   RP 1.1.1.99                     <- ยังเป็น RP เดิม แต่ IGP ตอนนี้ Route ไปหา CORE-SW2 แทน (Loopback99 Anycast)
CORE-SW2# show ip msdp sa-cache
MSDP Source-Active Cache - 1 entries
(10.10.30.50, 239.1.1.1), RP 1.1.1.99, ...       <- CORE-SW2 รับรู้ Source ต่อเนื่องผ่าน MSDP
```

เพราะ `1.1.1.99` Advertise เข้า OSPF จากทั้ง `CORE-SW1` และ `CORE-SW2` พร้อมกัน (Anycast) ทุก Router
ในเครือข่ายจะ Route ไปยัง RP ตัวที่ **ใกล้ที่สุดตาม IGP Metric ในขณะนั้นเสมอ** — ถ้า `CORE-SW1` ล้ม
IGP จะ Converge ให้ทุกคนเห็น `1.1.1.99` ผ่าน `CORE-SW2` แทนโดยอัตโนมัติ ไม่มีใครต้องแก้
`ip pim rp-address` เลยแม้แต่ตัวเดียว ส่วน MSDP ทำหน้าที่ให้ RP ทั้งสองรับรู้ Active Source เดียวกัน
ตลอดเวลา (ไม่ใช่แค่ตอน Failover) เพื่อไม่ให้เกิด Miss Source Information ช่วง Transition

### 4.3 — EEM Applet ตรวจ Violation Count แบบมีเงื่อนไข

```
ACCESS-SW3(config)# event manager applet PSEC-CONDITIONAL-RECOVERY
ACCESS-SW3(config-applet)# event syslog pattern "PM-4-ERR_DISABLE.*psecure-violation"
ACCESS-SW3(config-applet)# action 1.0 regexp "Interface ([A-Za-z0-9/]+)," "$_syslog_msg" match intf
ACCESS-SW3(config-applet)# action 2.0 cli command "enable"
ACCESS-SW3(config-applet)# action 2.1 cli command "show port-security interface $intf | include Violation Count"
ACCESS-SW3(config-applet)# action 2.2 regexp "Violation Count *: ([0-9]+)" "$_cli_result" match vcount
ACCESS-SW3(config-applet)# action 3.0 syslog msg "PSEC-RECOVERY: $intf err-disabled, violation count=$vcount"
ACCESS-SW3(config-applet)# action 4.0 if $vcount eq "1"
ACCESS-SW3(config-applet)# action 4.1  wait 300
ACCESS-SW3(config-applet)# action 4.2  cli command "configure terminal"
ACCESS-SW3(config-applet)# action 4.3  cli command "interface $intf"
ACCESS-SW3(config-applet)# action 4.4  cli command "shutdown"
ACCESS-SW3(config-applet)# action 4.5  cli command "no shutdown"
ACCESS-SW3(config-applet)# action 4.6  syslog msg "PSEC-RECOVERY: $intf auto-recovered after single violation"
ACCESS-SW3(config-applet)# action 5.0 else
ACCESS-SW3(config-applet)# action 5.1  syslog msg "PSEC-RECOVERY: $intf left disabled — violation count $vcount requires manual review"
ACCESS-SW3(config-applet)# action 6.0 end
```

**Verify (จำลอง Violation ครั้งที่ 1):**
```
%PM-4-ERR_DISABLE: psecure-violation error detected on Gi1/0/25, putting Gi1/0/25 in err-disable state
%HA_EM-6-LOG: PSEC-CONDITIONAL-RECOVERY: PSEC-RECOVERY: Gi1/0/25 err-disabled, violation count=1
... (รอ 300 วินาที) ...
%HA_EM-6-LOG: PSEC-CONDITIONAL-RECOVERY: PSEC-RECOVERY: Gi1/0/25 auto-recovered after single violation
```

Applet ใช้ `regexp` ดึงชื่อ Interface จาก Syslog Message เอง แล้ว Query `show port-security
interface` เพื่ออ่าน Violation Count จริงจากอุปกรณ์ (ไม่ใช่ Static Threshold ที่ตั้งไว้ล่วงหน้า)
เงื่อนไข `if $vcount eq "1"` ทำให้ Auto-recovery เกิดขึ้นเฉพาะครั้งแรก — ถ้า Port เดิมโดน Violation
ซ้ำอีกในวันเดียวกัน (Count เพิ่มเป็น 2 เพราะ Cisco ไม่รีเซ็ต Counter จนกว่าจะ `clear port-security`)
Applet จะเข้า Branch `else` แล้วปล่อยให้ Port ค้าง Disable ไว้รอ Manual — ตรงตามเงื่อนไขทั้งสองข้อ
โดยไม่แตะ `errdisable recovery` แบบ Global เลย

### 4.4 — RESTCONF PATCH บน Native Model

```
PATCH https://ACCESS-SW3/restconf/data/Cisco-IOS-XE-native:native/interface/GigabitEthernet=1%2F0%2F1
Content-Type: application/yang-data+json
Authorization: Basic <base64>

{
  "Cisco-IOS-XE-native:GigabitEthernet": {
    "name": "1/0/1",
    "description": "UPLINK-TO-DIST-SW3-VERIFIED",
    "logging": {
      "event": {
        "link-status": [null]
      }
    }
  }
}
```

**Verify:**
```
GET https://ACCESS-SW3/restconf/data/Cisco-IOS-XE-native:native/interface/GigabitEthernet=1%2F0%2F1

HTTP/1.1 200 OK
{
  "Cisco-IOS-XE-native:GigabitEthernet": {
    "name": "1/0/1",
    "description": "UPLINK-TO-DIST-SW3-VERIFIED",
    "switchport": { "trunk": { ... เดิมไม่เปลี่ยน ... } },
    "logging": { "event": { "link-status": [null] } }
  }
}
```

ใช้ **PATCH** (merge Partial) ไม่ใช่ **PUT** (replace ทั้ง Object) — Body ส่งแค่ 2 Leaf ที่ต้องแก้
(`description`, `logging/event/link-status`) ทำให้ Leaf อื่นทั้งหมดของ Interface (VLAN, Trunk,
Speed/Duplex) **ไม่ถูกแตะเลย** — ต้องใช้ Model `Cisco-IOS-XE-native` เพราะ Leaf
`logging/event/link-status` เป็น Cisco-Proprietary ที่ไม่มีอยู่ใน `ietf-interfaces` มาตรฐาน การรวม
สอง Leaf จากสอง Sub-tree เดียวกัน (`native/interface/GigabitEthernet`) ไว้ใน Body เดียวทำให้เป็น
1 PATCH Request ตามข้อจำกัด

### 4.5 — Single Ingress Policy-map (Mark+Police) + Egress Queuing แยก Hop

```
! ACCESS-SW1 / ACCESS-SW2 — Ingress: Classify + Mark + Police ในตัวเดียว (ตามข้อจำกัด)
ACCESS-SW1(config)# class-map match-all VOICE-RTP
ACCESS-SW1(config-cmap)# match access-group name VOICE-RTP-ACL
ACCESS-SW1(config)# class-map match-all SALES-BULK
ACCESS-SW1(config-cmap)# match access-group name SALES-BULK-ACL

ACCESS-SW1(config)# policy-map INGRESS-MARK-POLICE
ACCESS-SW1(config-pmap)# class VOICE-RTP
ACCESS-SW1(config-pmap-c)# set dscp ef
ACCESS-SW1(config-pmap-c)# police cir 128000 bc 8000 conform-action transmit exceed-action drop
ACCESS-SW1(config-pmap)# class SALES-BULK
ACCESS-SW1(config-pmap-c)# set dscp cs1
ACCESS-SW1(config-pmap)# class class-default
ACCESS-SW1(config-pmap-c)# set dscp default

ACCESS-SW1(config)# interface range GigabitEthernet1/0/1-24
ACCESS-SW1(config-if-range)# service-policy input INGRESS-MARK-POLICE

! ทุก Uplink Hop (ACCESS→DIST→CORE→WAN-EDGE) — Egress Queuing: คนละ Policy-map แยกจากตัว Mark
ACCESS-SW1(config)# class-map match-any EF-QUEUE
ACCESS-SW1(config-cmap)# match dscp ef
ACCESS-SW1(config)# class-map match-any CS1-QUEUE
ACCESS-SW1(config-cmap)# match dscp cs1

ACCESS-SW1(config)# policy-map EGRESS-QUEUING
ACCESS-SW1(config-pmap)# class EF-QUEUE
ACCESS-SW1(config-pmap-c)# priority
ACCESS-SW1(config-pmap)# class CS1-QUEUE
ACCESS-SW1(config-pmap-c)# bandwidth remaining percent 15
ACCESS-SW1(config-pmap)# class class-default
ACCESS-SW1(config-pmap-c)# fair-queue

ACCESS-SW1(config)# interface Port-channel1
ACCESS-SW1(config-if)# service-policy output EGRESS-QUEUING
! (Config เดียวกันทำซ้ำบน DIST-SW1 uplink→CORE-SW1, CORE-SW1 uplink→WAN-EDGE-1, ...)
```

**Verify:**
```
ACCESS-SW1# show policy-map interface GigabitEthernet1/0/5 input
  Class-map: VOICE-RTP (match-all)
    police:
        cir 128000 bps, bc 8000 bytes
      exceed-action: drop

ACCESS-SW1# show policy-map interface Port-channel1 output
  Class-map: EF-QUEUE (match-any)
    Priority: ... (strict priority, no bandwidth limit shown as absolute)
  Class-map: CS1-QUEUE (match-any)
    bandwidth remaining 15% ...
```

ข้อจำกัดของโจทย์พูดถึง**เฉพาะ Policy-map ที่ทำ Classify+Mark+Police ที่ Ingress Access Port**
ว่าห้ามเกิน 1 ตัว (`INGRESS-MARK-POLICE` ตัวเดียวจริงตามข้อจำกัด) — **ไม่ได้ห้าม Policy-map ตัวอื่น
ที่ทำหน้าที่ Queuing ที่ Egress ของแต่ละ Hop** (`EGRESS-QUEUING` เป็นคนละหน้าที่ คนละทิศทาง
คนละ Interface โดยสิ้นเชิง) — นี่คือจุดที่ต้องอ่านขอบเขตของข้อจำกัดให้แม่นตามตัวอักษร ไม่ตีความ
กว้างเกินจนไปบล็อกสิ่งที่โจทย์ไม่ได้ห้าม `priority` (Strict Priority Queue, ไม่มี Starve) รับประกัน
EF ทุก Hop ส่วน `bandwidth remaining percent 15` รับประกัน CS1 มี Queue Share ขั้นต่ำเสมอแม้ WIFI
(Class-default) จะพยายามใช้ Bandwidth ที่เหลือทั้งหมดก็ตาม

---

## Step 970 — Full Lab: Task Group 5 (โจทย์ผูกกันข้ามข้อ) + Deploy Module Exam Strategy

### 970.1 บริบท — ทำไมกลุ่มนี้ยากกว่ากลุ่มก่อน

ข้อสอบจริงในช่วงท้าย Deploy Module มักมี Task Group ที่ **Task ข้อหลังอ้างอิงผลลัพธ์เฉพาะเจาะจงจาก
Task ข้อก่อนหน้า** (ไม่ใช่แค่ "ใช้ Topology เดียวกัน" แต่ **ตัวเลข/ขอบเขตที่เลือกใน Task แรกจะกลาย
เป็น Input ของ Task ถัดไปโดยตรง**) — ถ้าเลือกวิธีทำ Task แรกต่างไปจาก Model Answer (แต่ยังถูกตาม
Requirement ของ Task แรกเอง) Task ถัดไปที่ผูกกันจะ **ต้องปรับตามให้สอดคล้อง** ไม่มีคำตอบตายตัวเดียว
สำหรับ Task ที่ผูกกัน — Grading Rubric จะตรวจ **ความสอดคล้องภายใน (Internal Consistency)** ระหว่าง
คำตอบของ Task ที่ผูกกันเป็นหลัก มากกว่าตรวจแค่ค่าตายตัว

### โจทย์

**5.1** Config BGP Aggregation บน `WAN-EDGE-1` และ `WAN-EDGE-2` ให้ Enterprise Prefix Block ทั้งหมด
(VLAN/Loopback รวมทั้งเครือข่าย) ถูกส่งไป `ISP-RTR` เป็น Aggregate ที่น้อยที่สุดเท่าที่เป็นไปได้
โดย **ต้องระงับ (Suppress) Prefix ย่อยทั้งหมดที่เป็นส่วนประกอบ ไม่ให้หลุดไปที่ ISP** — ใช้
BGP Aggregation เท่านั้น (ห้าม Redistribution Filtering)

**5.2** เพราะ Task 5.1 เปลี่ยนจำนวน Prefix ที่ `ISP-RTR` เห็นจาก `WAN-EDGE-1`/`WAN-EDGE-2` (จาก
หลายสิบ Prefix เหลือ Aggregate จำนวนน้อย) ปรับ Inbound Prefix-list ที่ `ISP-RTR` ให้ **รับเฉพาะ
Aggregate ที่ตรงกับที่ Task 5.1 สร้างไว้แบบเป๊ะๆ (Exact Match)** และปฏิเสธ Prefix ย่อยใดๆที่อาจรั่ว
ออกมาถ้า Aggregation ใน 5.1 พังหรือถูก Rollback บางส่วนในอนาคต

**5.3** Config OSPF Area Range บน ABR ของ Area 20 (`CORE-SW2`) ให้ขอบเขต Summarization ที่ประกาศ
เข้า Area 0 **ตรงกับขอบเขต Aggregate เดียวกันที่เลือกไว้ใน Task 5.1 เป๊ะ** (เพื่อให้ Design ของ
Internal Area Boundary กับ External BGP Aggregation Boundary สอดคล้องกันเป็นภาพเดียว ไม่ใช่คนละ
ขอบเขต)

**5.4** เขียน Ansible Playbook (Idempotent) Deploy Config ของ Task 5.1 ไปยัง `WAN-EDGE-1` และ
`WAN-EDGE-2` พร้อมกันจาก Playbook เดียว ด้วย Template เดียว (Parameterize ด้วย Host Variable) —
รันซ้ำครั้งที่สองต้องได้ `changed=0` ทั้งสอง Host — **ห้ามใช้ `ios_config` โหมด `lines:` แบบไม่มี
`parents:`** (เสี่ยง Duplicate Line ทุกครั้งที่รัน)

### Model Answer

**5.1 — เลือก Aggregate Boundary (จุดตัดสินใจที่ Task 5.3 จะต้องตามให้ตรง)**

Enterprise มี 2 Block หลักที่ไม่ต่อเนื่องกัน (Non-contiguous) คือ `10.10.0.0/16` (ทุก VLAN Campus)
และ `1.1.1.0/24` (ทุก Loopback0) — **ไม่สามารถรวมเป็น Aggregate เดียวได้** (Supernet คนละ Major
Block กัน) จึงต้องใช้ **2 Statement** นี่คือ "การเลือก Implementation" ของ Task 5.1 ที่ Task 5.3
ต้องอ้างอิงตาม:

```
! บน WAN-EDGE-1 และ WAN-EDGE-2 (Config เหมือนกัน)
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# aggregate-address 10.10.0.0 255.255.0.0 summary-only
WAN-EDGE-1(config-router)# aggregate-address 1.1.1.0 255.255.255.0 summary-only
```

**Verify:**
```
WAN-EDGE-1# show bgp ipv4 unicast summary | include Aggreg
WAN-EDGE-1# show ip route bgp
B        10.10.0.0/16 [200/0] via 0.0.0.0, ... , summarized                <- ส่งออกตัวนี้
B        1.1.1.0/24 [200/0] via 0.0.0.0, ..., summarized                   <- และตัวนี้

ISP-RTR# show bgp ipv4 unicast | include 65001
*> 10.10.0.0/16       ...  65001 i
*> 1.1.1.0/24         ...  65001 i
                       (ไม่มี /24, /30 ย่อยใดๆของ Enterprise หลุดมาอีกเลย)
```

**5.2 — Prefix-list ที่ ISP-RTR ต้อง "ตรงกับที่ 5.1 เลือก" เป๊ะ**

เพราะ Task 5.1 เลือก **สอง** Aggregate (`10.10.0.0/16`, `1.1.1.0/24`) — Prefix-list ที่ `ISP-RTR`
ต้องมี**สองบรรทัด Exact Match ตรงกับตัวเลขนี้เท่านั้น** ถ้า Task 5.1 เคยเลือกวิธีอื่น (เช่น สมมติ
มีคนเลือกทำ Aggregate แยกย่อยเป็น `10.10.0.0/17` + `10.10.128.0/17` โดยไม่จำเป็น) Prefix-list ตรงนี้
ก็ต้องเปลี่ยนตามให้ตรงเช่นกัน — นี่คือจุดผูกที่ Grading จะเช็คว่า **สอดคล้องกับสิ่งที่ Deploy จริง
ใน 5.1 หรือไม่** ไม่ใช่เช็คกับค่าคงที่ที่ตั้งไว้ล่วงหน้าเพียงค่าเดียว

```
! บน ISP-RTR
ISP-RTR(config)# ip prefix-list ENTERPRISE-EXACT-ONLY seq 10 permit 10.10.0.0/16
ISP-RTR(config)# ip prefix-list ENTERPRISE-EXACT-ONLY seq 20 permit 1.1.1.0/24
ISP-RTR(config)# ip prefix-list ENTERPRISE-EXACT-ONLY seq 30 deny 0.0.0.0/0 le 32

ISP-RTR(config)# router bgp 65000
ISP-RTR(config-router)# neighbor 203.0.113.1 prefix-list ENTERPRISE-EXACT-ONLY in
ISP-RTR(config-router)# neighbor 203.0.113.5 prefix-list ENTERPRISE-EXACT-ONLY in
```

หมายเหตุ: prefix-list ที่ไม่ระบุ `ge`/`le` ต่อจาก Prefix ที่ให้มา = **Exact Match เท่านั้น**
(`10.10.0.0/16` ตรงนี้จะไม่ Match `10.10.10.0/24` เลย) — ทำให้ต่อให้ Aggregation ที่ `WAN-EDGE-1`
พังจนหลุด Prefix ย่อยออกมาจริง `ISP-RTR` ก็จะปฏิเสธ Prefix ย่อยเหล่านั้นเป็นชั้น Defense-in-Depth
ชั้นที่สอง

**5.3 — Area Range ต้องตรงกับ Boundary เดียวกับ 5.1**

เพราะ 5.1 เลือก `10.10.0.0/16` เป็น Boundary ของฝั่ง Campus VLAN — `area range` ที่ `CORE-SW2`
(ABR ของ Area 20) ก็ต้องใช้ **Network/Mask เดียวกันเป๊ะ** เพื่อให้ Design สอดคล้องเป็นภาพเดียวตามที่
โจทย์ต้องการ:

```
! บน CORE-SW2
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# area 20 range 10.10.0.0 255.255.0.0
```

**Verify:**
```
CORE-SW0# show ip ospf database summary | include 10.10.0.0
CORE-SW1# show ip route ospf | include 10.10.0.0
O IA  10.10.0.0/16 [110/2] via 10.255.30.1, TenGigabitEthernet1/0/1     <- Summary เดียว แทนของ VLAN30/40 แยกย่อย
```

> **จุดผูกที่สำคัญที่สุดของ Task Group นี้**: ถ้า Task 5.1 (การเลือกของผู้สอบเอง) ไม่ได้เลือก
> `10.10.0.0/16` เป็น Boundary แต่เลือกวิธีอื่นที่ยัง Valid ตาม Requirement ของ 5.1 เอง (เช่น สมมติ
> ผู้สอบเลือกทำ Aggregate แยกเฉพาะ VLAN Subnet ที่มีจริงโดยไม่รวม Unused Space) `area range` ใน 5.3
> จะต้องเปลี่ยนตามให้ตรงกับ Boundary ใหม่นั้น **ไม่ใช่ก๊อปคำตอบนี้มาใช้ตรงๆ** — นี่คือแก่นของ
> Task ที่ "ผูกกันข้ามข้อ" ตามที่ข้อสอบจริงทำ

**5.4 — Ansible Playbook Idempotent ด้วย `parents:`**

```yaml
# inventory/hosts.yml
all:
  children:
    wan_edge:
      hosts:
        WAN-EDGE-1:
          ansible_host: 10.10.254.1
          bgp_as: 65001
        WAN-EDGE-2:
          ansible_host: 10.10.254.5
          bgp_as: 65001
```

```yaml
# deploy-bgp-aggregate.yml
- name: Deploy BGP Aggregation (Task 5.1) — Idempotent
  hosts: wan_edge
  gather_facts: no
  tasks:
    - name: Ensure aggregate-address statements exist under router bgp
      cisco.ios.ios_config:
        lines:
          - "aggregate-address 10.10.0.0 255.255.0.0 summary-only"
          - "aggregate-address 1.1.1.0 255.255.255.0 summary-only"
        parents: "router bgp {{ bgp_as }}"
        match: line
```

**Verify (รันครั้งที่ 1 แล้วครั้งที่ 2):**
```
$ ansible-playbook -i inventory/hosts.yml deploy-bgp-aggregate.yml
PLAY RECAP
WAN-EDGE-1    : ok=1  changed=1   <- ครั้งแรก Deploy จริง
WAN-EDGE-2    : ok=1  changed=1

$ ansible-playbook -i inventory/hosts.yml deploy-bgp-aggregate.yml
PLAY RECAP
WAN-EDGE-1    : ok=1  changed=0   <- รันซ้ำ ไม่มีอะไรเปลี่ยน (Idempotent จริง)
WAN-EDGE-2    : ok=1  changed=0
```

`parents: "router bgp {{ bgp_as }}"` บอก Module ว่า Line เหล่านี้ต้อง Match **ภายใน Context ของ
`router bgp` Section เท่านั้น** ทำให้ Module เช็ค Running-config ปัจจุบันก่อนว่ามี Line นี้อยู่แล้ว
ในบริบทที่ถูกต้องหรือไม่ (`match: line`) — ถ้ามีอยู่แล้วจะไม่ส่ง Command ซ้ำเลย (`changed=0`) ต่างจาก
`lines:` แบบไม่มี `parents:` ที่ Module จะมองแค่ว่า "Line นี้มีอยู่ที่ไหนก็ได้ใน Running-config
หรือไม่" ซึ่งบางกรณี (เช่น Syntax ที่ IOS แสดงผลต่างจากที่ป้อนเล็กน้อย) อาจ False-negative แล้ว Push
ซ้ำซ้อนทุกครั้งที่รัน

### 970.2 Deploy Module Exam Strategy — สรุปกลยุทธ์ทั้ง Part

**หลักที่ 1 — อ่านทุกคำในโจทย์ก่อนเปิด Terminal**: ขีดเส้นใต้คำว่า "ห้าม"/"ต้อง"/"เฉพาะ"/"โดยไม่"
ทุกครั้งก่อนพิมพ์ Command แรก (ตามที่ฝึกซ้ำมาทั้ง Task Group 1-5 ใน Part นี้)

**หลักที่ 2 — ทำ List ข้อจำกัดแยกจาก Requirement**: เขียนสองคอลัมน์บนกระดาษ/Note จริงในห้องสอบ —
"ผลลัพธ์ที่ต้อง Verify ได้" กับ "สิ่งที่ห้ามใช้" แยกกันชัดๆ ก่อนตัดสินใจว่าจะ Config อย่างไร

**หลักที่ 3 — Verify ทุกข้อจำกัดแยกจากกัน ไม่ใช่แค่ "Ping ผ่านไหม"**: ตัวอย่างจาก Task 1.1 — Ping
ระหว่าง `CORE-SW1`/`DIST-SW1` ผ่านได้ไม่ได้บอกอะไรเรื่อง DR เลย ต้อง `show ip ospf interface` แยก
เพื่อยืนยัน DR ตัวจริง และต้องเช็คด้วยว่า Running-config **ไม่มี** `ip ospf priority` หลงเหลืออยู่
— การ Verify ที่ถูกต้องคือ Verify **แยกทีละ Requirement และทีละข้อจำกัด** ไม่ใช่ Verify แบบรวบ
ผลลัพธ์เดียว

**หลักที่ 4 — จัดการ Task ที่ผูกกัน (Dependency Chain) ด้วยการจดบันทึกการเลือกของตัวเอง**: อย่างที่
เห็นใน Task Group 5 — ถ้า Task ข้อแรกมีทางเลือกได้หลายวิธี (Aggregate Boundary แบบไหนก็ได้ที่ Valid)
**ต้องจดว่าตัวเองเลือกอะไรไปแบบไหน** เพื่อให้ Task ที่ผูกกันทำได้สอดคล้องกัน — Candidate จำนวนมาก
เสียคะแนน Task ที่ผูกกันเพราะ**จำไม่ได้ว่าตัวเอง Deploy อะไรไปใน Task ก่อนหน้า** ทั้งที่ Task แรก
ทำถูกทุกอย่าง

**หลักที่ 5 — บริหารเวลาต่อ Task Group ไม่ใช่ต่อ Task เดี่ยว**: ถ้า Task Group ใช้เวลาไปแล้วเกิน
Budget ที่ตั้งไว้ (45-60 นาทีต่อกลุ่มตามที่แนะนำใน Step 961) ให้ข้าม Task ที่เหลือของกลุ่มนั้นไปทำ
กลุ่มถัดไปก่อน แล้วกลับมาทำ Task ที่ข้ามในช่วงท้ายถ้ามีเวลาเหลือ — Task ยากที่สุดในกลุ่มไม่ควรทำให้
เสียเวลาของกลุ่มถัดไปที่มี Task ง่ายกว่าและได้คะแนนเท่ากัน (ตามที่เตือนไว้แล้วใน
[Part 71 Step 705](part-071-ccie-lab-blueprint-overview.md))

---

## แบบฝึกหัดทวนความเข้าใจ Part 97

1. ใน Task 1.1 เพราะเหตุใดการใช้ `ip ospf priority 255` บน `CORE-SW1` จึงได้คะแนน 0 ทั้งข้อ แม้ผล
   ลัพธ์ปลายทาง (`CORE-SW1` เป็น DR จริง) จะถูกต้อง?
2. ใน Task 2.4 เพราะเหตุใดการ Config `switchport trunk allowed vlan` ที่ฝั่งเดียว (`ACCESS-SW1`)
   ก็เพียงพอที่จะบล็อก VLAN 30/40 ไม่ให้ข้าม Trunk ไปยัง `DIST-SW1` ได้ทั้งสองทิศทาง?
3. ใน Task 3.5 เพราะเหตุใดลำดับ Class ใน Policy-map ของ CoPP ถึงสำคัญกว่าค่า Police Rate ที่ตั้งไว้?
4. ใน Task Group 5 ทำไม Task 5.3 (`area range` บน `CORE-SW2`) จึงถูกอธิบายว่า "ไม่มีคำตอบตายตัวเดียว"
   ทั้งที่ Model Answer ให้ค่า `10.10.0.0/255.255.0.0` มาชัดเจน?
5. ตามหลักที่ 3 ของ Deploy Module Exam Strategy (Step 970.2) เพราะเหตุใด "Ping ผ่าน" ถึงไม่เพียงพอ
   ต่อการยืนยันว่า Task ในข้อสอบ Deploy Module ทำถูกต้องครบถ้วน?

**เฉลย:**

1. เพราะ Task 1.1 ระบุข้อจำกัดตรงตัวว่า **"ห้ามใช้คำสั่ง `ip ospf priority`"** — Grading Rubric ของ
   Deploy Module ตรวจข้อจำกัดของวิธีทำแบบ Binary (ใช้ = ผิดทันที) แยกจากการตรวจผลลัพธ์ปลายทาง
   เพราะโจทย์แบบนี้จงใจวัดว่าผู้สอบรู้จัก Mechanism อื่น (Router-ID) ที่ให้ผลเหมือนกันหรือไม่ ไม่ใช่
   แค่รู้ว่า Priority ควบคุม DR ได้ (ซึ่งเป็นความรู้ระดับ CCNA)
2. เพราะ `switchport trunk allowed vlan` เป็น Filter แบบ Local ต่อ Port เดียว ที่กรองทั้ง Frame
   ที่ Port นั้น**จะส่งออก**ไปยัง Neighbor (VLAN ที่ไม่อยู่ใน List จะไม่ถูก Tag ส่งออกเลย) และ Frame
   ที่**รับเข้ามา**จาก Neighbor (Frame ที่ Tag เป็น VLAN ที่ไม่อยู่ใน List จะถูก Drop ทันทีที่ Ingress)
   — ทำงานครบทั้งสองทิศทางโดยไม่ต้องพึ่ง Config ฝั่ง `DIST-SW1` เลย
3. เพราะ Policy-map จับคู่ Class แบบ **First-match ตามลำดับบนลงล่างเสมอ** Packet หนึ่งตัวจะถูกจัด
   เข้า Class แรกที่ Match เจอเท่านั้นแล้วหยุด ไม่พิจารณา Class ที่เหลือต่อ — ถ้าวาง Class ที่ Match
   OSPF ไว้หลัง Class SSH (หรือ Class-default ที่ครอบคลุมกว้าง) OSPF Packet บางส่วนอาจถูกจัดเข้า Class
   ผิดไปก่อนถึง Class ของตัวเองจริง ไม่ว่า Police Rate ของ Class OSPF เองจะสูงแค่ไหนก็ช่วยไม่ได้
   ถ้า Packet ไม่ได้ไปถึง Class นั้น
4. เพราะขอบเขต Aggregate ที่เลือกใน Task 5.1 (`10.10.0.0/16`) เป็น**การตัดสินใจของผู้สอบเอง**ท่ามกลาง
   ทางเลือกที่ Valid ได้หลายแบบ (เช่น อาจเลือก Aggregate แยกย่อยกว่านี้ก็ยังผ่าน Requirement ของ 5.1
   เองได้) — ค่าที่ใช้ใน Task 5.3 จึงต้อง **สอดคล้องกับสิ่งที่ผู้สอบ Deploy จริงใน 5.1** ไม่ใช่ค่า
   ตายตัวสากล ถ้าเลือก Boundary ต่างไปใน 5.1 ค่าใน 5.3 ก็ต้องเปลี่ยนตามเพื่อให้ยัง Consistent กัน
5. เพราะ Ping ยืนยันได้แค่ **Reachability ระดับ Layer 3** เท่านั้น ไม่ได้บอกอะไรเกี่ยวกับ **วิธีที่
   ใช้ทำให้ Reachability นั้นเกิดขึ้น** ซึ่งเป็นส่วนที่ข้อจำกัดของโจทย์ (เช่น "ห้ามใช้ Priority",
   "ห้ามแก้ Cost") ควบคุมอยู่ — ต้อง Verify แยกด้วยคำสั่งที่ตรงกับ Requirement ตัวจริงเสมอ (เช่น
   `show ip ospf interface` เพื่อดู DR, `show run` เพื่อยืนยันไม่มี Command ที่ต้องห้ามหลงเหลืออยู่)

---

## สรุป Part 97

Part นี้ฝึกทักษะ **Deploy Module** เต็มรูปแบบผ่าน 24 Task สไตล์ข้อสอบจริง (Task Group 1-5) ที่ทุกข้อ
ผูกข้อจำกัดวิธีทำเข้ากับ Requirement เสมอ ตามธรรมชาติของข้อสอบ CCIE จริงที่วัด **ความรู้กว้างของ
Mechanism** ไม่ใช่แค่ความรู้ลึกของ Feature เดียว — ครอบคลุม Routing (OSPF Router-ID/Bandwidth-cost
trick/Area filter-list, BGP Distance), Switching/HA (UDLD, Root Guard, HSRP Preempt Delay, GLBP
Weighted), Security (ACL Sequence Insert, MAB/Critical VLAN, AAA Fallback Chain, TrustSec SGACL,
CoPP Class Ordering), Services/Automation (NAT ACL Exception, Anycast RP, Conditional EEM,
RESTCONF PATCH, End-to-end QoS) และปิดท้ายด้วย Task Group ที่ผูกกันข้ามข้อจริงเหมือนสถานการณ์สอบจริง
ที่สุด พร้อม Deploy Module Exam Strategy 5 หลักที่ต้องนำไปใช้ทุกครั้งที่ฝึก

✅ **พร้อมสำหรับ Part 98**: เราจะเข้าสู่ Task Type สุดท้ายของ 4 แบบตามที่ประกาศไว้ใน
[Part 71 Step 703](part-071-ccie-lab-blueprint-overview.md) — **Optimize/Operate Module** ที่โจทย์
ให้เครือข่ายที่ **ทำงานอยู่แล้วจริง** (บางส่วนมาจาก Config ที่ Deploy ไปแล้วใน Part นี้) แล้วต้องปรับ
ปรุง Performance/Scale/HA แบบ **Non-disruptive** โดยกระทบ Traffic เดิมให้น้อยที่สุด — ทักษะที่ต่างจาก
Deploy Module อย่างสิ้นเชิงเพราะจุดเริ่มต้นไม่ใช่ "เครือข่ายเปล่า" แต่เป็น "เครือข่ายที่มี History
และ Production Traffic ต้องรักษาไว้"

**ไปต่อ:** [Part 98 — CCIE Full Lab Scenario #4 (Optimize Module) →](part-098-ccie-lab-scenario-4-optimize.md)
