# Part 57 — Advanced OSPF Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 561–570 จาก 1000**

> ต่อจาก [Part 56 — Advanced EIGRP Troubleshooting](part-056-eigrp-troubleshooting.md) ที่ปูพื้น
> **Troubleshooting Methodology** ให้กับ EIGRP Demo Site ของหลักสูตร Part นี้พา Methodology
> แบบเดียวกันมาใช้กับ **OSPF 3-Area Design** ที่วางไว้ตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md)
> (Neighbor State Machine, RID, Network Type), [Part 12](part-012-ospfv2-advanced.md) (Multi-Area,
> ABR, `area range`, Virtual-Link, NSSA) และ [Part 26](part-026-advanced-ospf.md) (BFD, SPF/LSA
> Throttling, `auto-cost reference-bandwidth`, `area filter-list`) — Lab หลักของหลักสูตรใช้
> **Area 0** (CORE-SW1↔CORE-SW2 ผ่าน Port-channel1), **Area 1** (CORE-SW1 เป็น ABR ให้ DIST-SW1/
> DIST-SW2), และ **Area 2** (CORE-SW2 เป็น ABR ให้ DIST-SW3/DIST-SW4) พร้อม MD5 Authentication,
> Passive-Interface, BFD และ Filter ที่ Config ไว้ครบแล้วทุกจุด — Part นี้จะ**จงใจทำให้พังทีละ
> จุด** ด้วยสถานการณ์ที่สมจริงตามแนวข้อสอบ **CCNP ENARSI (300-410)** แล้วฝึกวินิจฉัยจนเจอ Root
> Cause และแก้ไขให้ถูกต้องทีละ Scenario ปิดท้ายด้วย Lab รวมที่ปัญหา 3 จุดเกิดพร้อมกันในคราวเดียว

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 561 | OSPF Troubleshooting Methodology — Decision Tree จาก Neighbor State ถึง Route Installed |
| 562 | Scenario 1 — Neighbor ค้างที่ 2-Way เพราะ Priority 0 ทั้งสองฝั่ง (ไม่มี DR/BDR ให้เลือก) |
| 563 | Scenario 2 — Neighbor ค้างที่ ExStart เพราะ MTU Mismatch (Classic Gotcha) |
| 564 | Scenario 3 — Area ID ไม่ตรงกันบน Link เดียวกัน (Adjacency ไม่ก่อตัวเลย) |
| 565 | Scenario 4 — Route หายเพราะ `area range` กว้างเกินไป (และ `filter-list` กว้างเกินไป) |
| 566 | Scenario 5 — Authentication Type ไม่ตรงกันหลัง Rollout ครึ่งเดียว |
| 567 | Scenario 6 — Suboptimal Routing จาก `auto-cost reference-bandwidth` ไม่ครบทุกตัว |
| 568 | Scenario 7 — NSSA Mismatch ตัด Area 2 ขาดจาก Backbone ทั้ง Area |
| 569 | Scenario 8 — LSA Flooding Storm / CPU พุ่งจาก Interface Flap แม้มี Throttle Timer |
| 570 | Lab เต็มรูปแบบ: 3 ปัญหาซ้อนกันพร้อมกัน (MTU + Area Range + Authentication) |

---

## Step 561 — OSPF Troubleshooting Methodology: Decision Tree จาก Neighbor State ถึง Route Installed

### 561.1 ทำไมต้องมี Methodology ก่อนแตะ Config

ข้อสอบ ENARSI และงาน Production จริงไม่เคยบอกตรงๆ ว่า "ปัญหาคืออะไร" — สิ่งที่ได้มาคือ
อาการปลายทาง เช่น "PC ปิง Server ไม่ได้" แล้วต้องไล่ลงไปเรื่อยๆ จนเจอ Layer ที่พัง วิธีที่ผิดที่สุด
คือการเดา Config สุ่มๆ — วิธีที่ถูกต้องคือ **ไล่ตามลำดับ Layer ของ OSPF เอง** เพราะ OSPF มี
ลำดับการทำงานที่ตายตัว (Neighbor → LSDB → SPF → RIB → FIB) ถ้า Layer ใดพัง Layer ถัดไปก็จะ
ไม่มีทางถูกต้องได้เลย ไม่ว่าจะไปแก้อะไรที่ Layer หลัง

```
                          ┌─────────────────────────┐
                          │  ping/traceroute ไม่ผ่าน  │
                          └────────────┬────────────┘
                                       ▼
                     ┌─────────────────────────────────┐
                     │ show ip ospf neighbor            │
                     │ Neighbor ขึ้น FULL หรือไม่?        │
                     └───────────────┬───────────────────┘
                    ไม่ FULL │                        │ FULL แล้ว
                             ▼                        ▼
              ┌───────────────────────────┐  ┌──────────────────────────┐
              │ State ค้างอยู่ State ไหน?    │  │ show ip ospf database    │
              └──────────────┬─────────────┘  │ LSA ที่ควรมีอยู่ครบไหม?    │
        ┌─────────┬─────────┼─────────┬───────┘  └────────────┬─────────┘
        ▼         ▼         ▼         ▼                    ครบ│    │ขาด/ผิด
     DOWN      INIT     2-WAY    EXSTART/                     ▼    ▼
        │         │     ค้าง     EXCHANGE                (ไปข้อถัดไป) area range/
   L1/L2,ACL  Timer/    ตลอด     ค้างตลอด                       filter-list/
   Passive   Unidirect  (Step562) MTU (Step563)                 NSSA (Step565,568)
   ทั้งคู่,   Hello,                Duplicate RID
   Area ID   Auth (Step566)
   (Step564)
                                       │ ครบ
                                       ▼
                        ┌───────────────────────────┐
                        │ show ip route ospf         │
                        │ Route ปรากฏใน RIB หรือไม่?  │
                        └───────────────┬─────────────┘
                     ไม่ปรากฏ│                        │ปรากฏแล้ว
                             ▼                        ▼
                  area range/filter-list      ┌──────────────────────────┐
                  กว้างเกินไป (Step565)         │ Path ที่เลือกเหมาะสมไหม?   │
                  Path-Type ไม่ตรงที่คาด          │ Cost สมเหตุสมผลไหม?       │
                                               └────────────┬─────────────┘
                                                   ไม่เหมาะ │            │เหมาะแล้ว
                                                            ▼            ▼
                                              auto-cost reference-      Traffic ควร
                                              bandwidth ไม่ครบ (Step567)  ผ่านได้ปกติ
                                                                        เช็ค CPU/Flap
                                                                        (Step569) แทน
```

### 561.2 ตาราง Neighbor State ที่ค้าง → สาเหตุที่เป็นไปได้มากที่สุด

| State ที่ค้าง | สาเหตุที่พบบ่อยที่สุด | Step ที่สอนใน Part นี้ |
|---|---|---|
| **ไม่ปรากฏใน Neighbor Table เลย** | L1/L2 ไม่ขึ้น, `passive-interface` ทั้งสองฝั่งหรือฝั่งเดียว, ACL Block IP Protocol 89, **Area ID ไม่ตรงกัน** | Step 564 |
| **Down → Init สลับตลอด (Flapping)** | Hello/Dead Timer Mismatch (Part 11 Step 106), Unidirectional Link (สายส่ง-รับใช้ Fiber คนละคู่ผิด) | อ้างอิง Part 11 |
| **Init ค้าง** | ฝั่งหนึ่งไม่เห็น RID ตัวเองใน Hello ของอีกฝั่ง (มักเป็น ACL หรือ Unicast-only ผิด) | อ้างอิง Part 11 |
| **2-Way ค้างตลอด (บน Broadcast Segment)** | ทั้งสองฝั่ง (หรือทุกฝั่ง) ตั้ง `ip ospf priority 0` จนไม่มีใครมีสิทธิ์เป็น DR/BDR | **Step 562** |
| **ExStart/Exchange ค้างตลอด** | **MTU Mismatch**, Duplicate Router-ID ระหว่าง Router สองตัว | **Step 563** |
| **Neighbor ขึ้น Full แล้วหลุดกะทันหัน** | Authentication Mismatch หลัง Rollout ครึ่งเดียว, Dead Timer หมดจาก BFD/Flap จริง | **Step 566**, Step 569 |

### 561.3 คำสั่ง Verify ที่ต้องใช้ตามลำดับ (Troubleshooting Toolkit)

| ลำดับ | คำสั่ง | ใช้เช็คอะไร |
|---|---|---|
| 1 | `show ip ospf neighbor` | Adjacency ขึ้น Full หรือไม่ ค้าง State ไหน |
| 2 | `show ip ospf interface <if>` | Network Type, Cost, Timer, Area, Authentication ราย Interface |
| 3 | `debug ip ospf adj` | ดู State Machine แบบ Real-time ตอนสร้าง/พัง Adjacency |
| 4 | `show ip ospf database` | LSDB มี LSA ที่ควรมีครบไหม (ต้องเทียบทุก Router ใน Area เดียวกันให้เหมือนกัน) |
| 5 | `show ip route ospf` | Route ติดตั้งใน RIB จริงหรือไม่ Path-Type อะไร (O, O IA, O E1/E2, O N1/N2) |
| 6 | `show ip ospf border-routers` | ABR/ASBR ที่ Router เห็น และ Cost ไปถึง — เช็ค Backbone Reachability |
| 7 | `show ip ospf statistics` | ความถี่ SPF Run — ผิดปกติถ้าถี่กว่า 1 ครั้ง/วินาทีต่อเนื่อง |
| 8 | `show processes cpu sorted` | ยืนยันว่า CPU พุ่งจาก OSPF Process จริงหรือ Process อื่น |

> **กฎเหล็กของการ Troubleshoot ทุกครั้ง**: ไล่ตามลำดับตารางข้างบนเสมอ **ห้ามข้ามขั้น** — ถ้า
> Neighbor ยังไม่ Full อย่าเสียเวลาไปดู `show ip route` เพราะ Route จะไม่มีทางถูกต้องได้เลยถ้า
> Adjacency พื้นฐานยังไม่สมบูรณ์ (ยกเว้นกรณี Neighbor Full แล้วแต่ Route หาย ซึ่งต้องข้ามไปเช็ค
> LSDB/Filter โดยตรงตาม Decision Tree ด้านบน)

---

## Step 562 — Scenario 1: Neighbor ค้างที่ 2-Way เพราะ Priority 0 ทั้งสองฝั่ง

### 562.1 บริบท

Link ระหว่าง **CORE-SW1 (GigabitEthernet1/0/1) ↔ WAN-EDGE-1 (GigabitEthernet0/0/1)** เป็น Link
เดียวในหลักสูตรนี้ที่ยังเป็น **OSPF Network Type = Broadcast จริง** (ตามที่ Part 11 Step 109.1
แสดงไว้ — Neighbor 1.1.1.21 State `FULL/BDR`) เพราะไม่ได้ถูกบังคับเป็น Point-to-Point เหมือน
Backbone Link อื่นๆ ทีมงานความปลอดภัยเพิ่งทำ Hardening Guideline ใหม่ ต้องการ "ปิดสิทธิ์การเป็น
DR ของทุก Edge Device ที่ไม่ใช่ Core" จึงไปตั้ง `ip ospf priority 0` ไว้ที่ WAN-EDGE-1 — แต่ในการ
Deploy ครั้งเดียวกัน วิศวกรอีกคนไปตั้ง `ip ospf priority 0` ไว้ที่ CORE-SW1 ด้วยเหตุผลว่า "ให้
WAN-EDGE-1 เป็น DR แทน" โดยไม่ได้คุยกันก่อน

### 562.2 Config ที่ผิด (ทั้งสองฝั่งพร้อมกัน)

```
! CORE-SW1
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# ip ospf priority 0

! WAN-EDGE-1
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# ip ospf priority 0
```

### 562.3 อาการที่พบ

```
CORE-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.21          0   2WAY/DROTHER    00:00:31    10.10.254.1     GigabitEthernet1/0/1
```

Neighbor ไม่หลุด ไม่ Flap แต่ **ค้างที่ 2WAY/DROTHER ตลอดเวลาไม่ไป Full เลย** — ผลกระทบจริง:
`show ip route ospf` บน WAN-EDGE-1 **ไม่เห็น Route จาก CORE-SW1 เลยแม้แต่เส้นเดียว** เพราะ
Router ที่คุยกันแค่ 2-Way จะไม่แลก LSA กัน (ตาม State Machine ที่สอนใน Part 11 Step 102.2)

### 562.4 การวินิจฉัย

```
CORE-SW1# show ip ospf interface GigabitEthernet1/0/1
GigabitEthernet1/0/1 is up, line protocol is up
  Internet Address 10.10.254.2/30, Area 0, Attached via Network Statement
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DROTHER, Priority 0        <--- Priority = 0 ยืนยัน
  Designated Router (ID) 0.0.0.0, Interface address 0.0.0.0   <--- ไม่มี DR เลย!
  No backup designated router on this network
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
```

บรรทัด **`Designated Router (ID) 0.0.0.0`** และ **`No backup designated router`** คือหลักฐาน
ชัดเจนที่สุด — Segment นี้**ไม่มี DR และไม่มี BDR เลย** เพราะ Router ทุกตัวบน Segment (ในกรณีนี้
มีแค่ 2 ตัว) ประกาศ Priority 0 พร้อมกันทั้งคู่ ตามกฎ DR/BDR Election ที่สอนใน Part 11 Step
105.3 (ข้อ 3): "Router ที่ตั้ง Priority = 0 จะไม่มีสิทธิ์เป็น DR **หรือ** BDR เลย" — เมื่อไม่มีใคร
มีสิทธิ์เลยแม้แต่ตัวเดียว การเลือกตั้งจึงไม่มีผลลัพธ์ ทุกตัวเลยกลายเป็น DROTHER ถาวร และ DROTHER
คุยกับ DROTHER ตัวอื่นได้แค่ 2-Way ตามกฎเดียวกัน (Step 105.3)

```
! ยืนยันอีกชั้นด้วย debug — เห็นวนซ้ำที่ 2-Way ไม่ขยับไปไหนเลย
CORE-SW1# debug ip ospf adj
OSPF-1 ADJ  GigabitEthernet1/0/1: 2 Way Communication to 1.1.1.21, state 2WAY
OSPF-1 ADJ  GigabitEthernet1/0/1: Neighbor change Event on interface GigabitEthernet1/0/1
OSPF-1 ADJ  GigabitEthernet1/0/1: DR/BDR election
OSPF-1 ADJ  GigabitEthernet1/0/1: Elect BDR 0.0.0.0
OSPF-1 ADJ  GigabitEthernet1/0/1: Elect DR 0.0.0.0
OSPF-1 ADJ  GigabitEthernet1/0/1: No DR on network GigabitEthernet1/0/1
```

### 562.5 Root Cause

**ทั้งสองฝั่งของ Broadcast Segment ตั้ง `ip ospf priority 0` พร้อมกัน** ทำให้ไม่มี Router ตัวใด
มีสิทธิ์เป็น DR/BDR เลย — Segment จึงไม่มี DR/BDR ถาวร และ Neighbor ทุกคู่บน Segment นั้นจะค้างที่
2-Way ตลอดไป (ไม่ใช่ Bug เป็นพฤติกรรมที่ถูกต้องตาม RFC 2328 แต่เป็น**ผลจาก Misconfiguration**)

### 562.6 วิธีแก้และ Verify

ต้องเหลือ Router ที่ Priority ≥ 1 อย่างน้อย **1 ตัว** บน Segment เสมอ (แนะนำให้ Core Device เป็น
DR เพื่อควบคุม Flood Traffic ให้อยู่ฝั่งที่มี Bandwidth/CPU สูงกว่า):

```
CORE-SW1(config)# interface GigabitEthernet1/0/1
CORE-SW1(config-if)# ip ospf priority 1          ! คืนค่า Default ให้ CORE-SW1 มีสิทธิ์เป็น DR
```

> **ข้อสังเกต**: ไม่จำเป็นต้อง `clear ip ospf process` หรือ Shutdown Interface — DR/BDR Election
> จะ Trigger ใหม่อัตโนมัติทันทีที่ Priority เปลี่ยน (ต่างจาก Router-ID ที่ต้อง Restart Process)

```
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.21          0   FULL/BDR        00:00:38    10.10.254.1     GigabitEthernet1/0/1

CORE-SW1# show ip ospf interface GigabitEthernet1/0/1 | include Designated|Backup
  Designated Router (ID) 1.1.1.1, Interface address 10.10.254.2
  Backup Designated Router (ID) 1.1.1.21, Interface address 10.10.254.1
```

CORE-SW1 กลายเป็น DR (Priority 1 > 0 ของ WAN-EDGE-1), WAN-EDGE-1 กลายเป็น BDR (เพราะเป็น Router
ตัวเดียวที่เหลือให้เลือกและยังไม่ Priority 0) — Neighbor ขึ้น **FULL/BDR** และ `show ip route
ospf` บน WAN-EDGE-1 กลับมาเห็น Route จาก CORE-SW1 ปกติทันที

---

## Step 563 — Scenario 2: Neighbor ค้างที่ ExStart เพราะ MTU Mismatch

### 563.1 บริบท

ทีม Data Center เพิ่งทดลองเปิด **Jumbo Frame** บน DIST-SW1 เพื่อเตรียมรองรับ Storage Traffic ใน
อนาคต โดยตั้งใจจะเปิดแค่บาง Interface ที่เชื่อม Server แต่พิมพ์ผิด Interface ไปโดนพอร์ต Uplink
ไปยัง CORE-SW1 (`TenGigabitEthernet1/1/1`) เข้าด้วย — ฝั่ง CORE-SW1 (`TenGigabitEthernet1/0/1`)
ไม่มีใครไปแก้ ยังเป็นค่า Default 1500 Byte

### 563.2 Config ที่ผิด

```
! DIST-SW1 — ตั้งใจจะเปิด Jumbo Frame แค่ฝั่ง Server แต่พิมพ์ผิด Interface
DIST-SW1(config)# interface TenGigabitEthernet1/1/1     ! <-- ที่จริงควรเป็น Interface อื่น
DIST-SW1(config-if)# mtu 9216
```

### 563.3 อาการที่พบ

```
DIST-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   EXSTART/  -     00:00:38    10.255.10.1     TenGigabitEthernet1/1/1
```

Neighbor ไม่ Down ไปเลย แต่ **ค้างที่ EXSTART** (บางครั้งจะเห็นสลับ EXSTART ↔ DOWN ↔ INIT ↔
2WAY ↔ EXSTART วนไปเรื่อยๆ ทุกๆ Retransmit Interval เพราะ DBD ถูกปฏิเสธซ้ำแล้วซ้ำอีก) — ไม่มี
วันไปถึง Exchange/Loading/Full

### 563.4 การวินิจฉัย

```
DIST-SW1# debug ip ospf adj
OSPF-1 ADJ  Te1/1/1: 2 Way Communication to 1.1.1.1, state 2WAY
OSPF-1 ADJ  Te1/1/1: Nbr 1.1.1.1: Prepare dbase exchange
OSPF-1 ADJ  Te1/1/1: Send DBD to 1.1.1.1 seq 0x20A1 opt 0x52 flag 0x7 len 32
OSPF-1 ADJ  Te1/1/1: Rcv DBD from 1.1.1.1 seq 0x20A1 opt 0x52 flag 0x7 len 32  state EXSTART
OSPF-1 ADJ  Te1/1/1: NBR Negotiation Done. We are the MASTER
OSPF-1 ADJ  Te1/1/1: Nbr 1.1.1.1 has smaller interface MTU
%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on TenGigabitEthernet1/1/1 from EXSTART to DOWN, Neighbor Down: Too many retransmissions
```

บรรทัด **`Nbr 1.1.1.1 has smaller interface MTU`** คือจุดสำคัญ — เทียบ MTU สองฝั่งตรงๆ:

```
DIST-SW1# show interfaces TenGigabitEthernet1/1/1 | include MTU
  MTU 9216 bytes, BW 10000000 Kbit/sec, DLY 10 usec,

CORE-SW1# show interfaces TenGigabitEthernet1/0/1 | include MTU
  MTU 1500 bytes, BW 10000000 Kbit/sec, DLY 10 usec,
```

DIST-SW1 ประกาศ MTU 9216 ใน DBD Packet ของตัวเอง — CORE-SW1 (MTU 1500) ตรวจพบว่า **MTU ที่
ประกาศมาใหญ่กว่า MTU ของตัวเอง** จึง**ปฏิเสธ DBD Packet นั้นทันที** (เพราะกลัวรับ LSA ที่มีขนาด
เกิน MTU ตัวเองไม่ได้ในขั้นตอนถัดไป) — Retransmit วนซ้ำจนเกิน Threshold แล้ว Reset Adjacency
กลับไป Down เริ่มใหม่ วนเป็น Loop ไม่จบ

### 563.5 Root Cause

**MTU ไม่ตรงกันระหว่างสองฝั่งของ Link เดียวกัน** — OSPF ตรวจสอบค่า MTU ที่แจ้งมาใน DBD Packet
เทียบกับ MTU ของ Interface ตัวเอง โดย Default (ไม่มี `ip ospf mtu-ignore`) ถ้าค่าไม่ตรงกัน (ไม่ใช่
แค่ "ใหญ่กว่า" ต้องเท่ากันเป๊ะ) Router จะปฏิเสธ DBD และไม่ยอมเดินหน้าไปเกิน ExStart/Exchange เด็ดขาด
— นี่คือ **Classic Gotcha ที่ข้อสอบ ENARSI ชอบออกที่สุดข้อหนึ่ง** เพราะ Neighbor ไม่ Down ไปเลย
(ทำให้เข้าใจผิดว่าปัญหาอยู่ที่ Physical/Timer) แต่จริงๆ ค้างอยู่ที่ State การแลก Database เท่านั้น

### 563.6 วิธีแก้ — 2 ทางเลือก พร้อมข้อดี/ข้อเสีย

| วิธีแก้ | คำสั่ง | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **แก้ MTU ให้ตรงกัน (แนะนำ)** | `mtu 1500` กลับที่ DIST-SW1 (หรือปรับทั้งสองฝั่งให้เท่ากันตามความตั้งใจจริง) | แก้ที่ต้นเหตุจริง ไม่มี Risk เรื่อง Fragmentation ของ LSA/Data Traffic จริงที่ผ่าน Link | ต้องเสีย Jumbo Frame Benefit ถ้าจำเป็นต้องใช้จริง |
| **`ip ospf mtu-ignore` (Workaround)** | สั่งบน Interface ที่ MTU ต่ำกว่า (หรือทั้งสองฝั่งเพื่อความชัวร์) | Adjacency ขึ้นได้ทันทีแม้ MTU ไม่ตรง ไม่ต้องแก้ Physical Design | **ข้ามการตรวจสอบไปเฉยๆ** ไม่ได้แก้ MTU จริง ถ้ามี LSA ขนาดใหญ่จริงเกิด Fragmentation ที่ Layer ต่ำกว่าอาจทำให้ Database Sync ผิดเพี้ยนแบบเงียบๆ ไม่มี Error แจ้ง |

```
! วิธีที่แนะนำ — แก้ MTU ให้ตรงกันจริง (Rollback ความผิดพลาด)
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# mtu 1500

DIST-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:00:33    10.255.10.1     TenGigabitEthernet1/1/1
```

> **ข้อสอบชอบถาม**: `ip ospf mtu-ignore` เป็น Workaround ที่ใช้ได้จริงเฉพาะกรณีที่**รู้แน่ชัด**ว่า
> MTU ต่างกันด้วยเหตุผลที่ยอมรับได้ (เช่น GRE Tunnel ที่ MTU ต่ำกว่า Physical เสมอโดยธรรมชาติ)
> ไม่ควรใช้เป็นทางแก้ปัญหา Production ทั่วไปที่เกิดจาก Human Error แบบนี้ — ควรแก้ MTU ให้ตรงกันเสมอ
> เมื่อทำได้จริง

---

## Step 564 — Scenario 3: Area ID ไม่ตรงกันบน Link เดียวกัน

### 564.1 บริบท

DIST-SW2 เพิ่งผ่านการ RMA (เปลี่ยนอุปกรณ์ใหม่) วิศวกรที่ Restore Config กลับเข้าไปพิมพ์คำสั่ง
`ip ospf area` ผิดเลข Area บน Uplink Interface ที่เชื่อมไป CORE-SW1 — ตาม [Part 12 Step
114.4](part-012-ospfv2-advanced.md) DIST-SW2 ต้องอยู่ **Area 1** ทั้งอุปกรณ์ (VLAN10/20/99 +
Uplink) แต่วิศวกรพิมพ์ `area 2` ไปโดยไม่ตั้งใจ (สลับกับความจำ Area 2 ของ DIST-SW3/DIST-SW4)

### 564.2 Config ที่ผิด

```
! DIST-SW2 (ควรเป็น area 1 ทุก Interface ตาม Part 12 Step 114.4)
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# ip ospf 1 area 2                 ! <-- พิมพ์ผิด ควรเป็น area 1
```

### 564.3 อาการที่พบ

```
CORE-SW1# show ip ospf neighbor | include 1.1.1.12
                                                        <-- ไม่มีบรรทัดนี้เลย! Neighbor หายไปทั้งหมด
```

Neighbor ของ DIST-SW2 **ไม่ปรากฏใน Neighbor Table ของ CORE-SW1 เลยแม้แต่ State Down** — ต่างจาก
Timer Mismatch (Part 11 Step 106) ที่ยังเห็นค้างที่ Init ได้ — Area ID Mismatch ทำให้ Hello Packet
ถูก**ปฏิเสธตั้งแต่ต้นทาง ก่อนสร้าง Neighbor Entry ด้วยซ้ำ**

### 564.4 การวินิจฉัย

```
CORE-SW1# show logging | include ERRRCV
%OSPF-4-ERRRCV: Received invalid packet: mismatch area ID from 10.255.20.2, TenGigabitEthernet1/0/2
%OSPF-4-ERRRCV: Received invalid packet: mismatch area ID from 10.255.20.2, TenGigabitEthernet1/0/2
%OSPF-4-ERRRCV: Received invalid packet: mismatch area ID from 10.255.20.2, TenGigabitEthernet1/0/2
```

Syslog `mismatch area ID` คือหลักฐานตรงตัวที่สุด — ยืนยันด้วยการเทียบ Area ทั้งสองฝั่งของ Link
เดียวกัน (ต้องเช็คทั้งสองฝั่งเสมอ ไม่ใช่แค่ฝั่งเดียว):

```
CORE-SW1# show ip ospf interface TenGigabitEthernet1/0/2 | include Area
  Internet Address 10.255.20.1/30, Area 1, Attached via Network Statement

DIST-SW2# show ip ospf interface TenGigabitEthernet1/1/1 | include Area
  Internet Address 10.255.20.2/30, Area 2, Attached via Network Statement    <-- ไม่ตรงกัน!
```

หรือดูภาพรวมเร็วกว่าด้วย `show ip ospf interface brief`:

```
DIST-SW2# show ip ospf interface brief
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Vl10         1     1               10.10.10.3/24       1     P2P   0/0
Vl20         1     1               10.10.20.3/24       1     P2P   0/0
Vl99         1     1               10.10.99.3/24       1     P2P   0/0
Te1/1/1      1     2               10.255.20.2/30      1     P2P   0/0    <-- ตัวเดียวที่ Area 2
```

### 564.5 Root Cause

**Area ID ไม่ตรงกันระหว่างสองฝั่งของ Link เดียวกัน** — Hello Packet ของ OSPF บรรจุ Area ID ไปด้วย
เสมอ (ต่างจาก Router ID ที่ไม่ต้องตรงกัน) ถ้า Area ID ที่ระบุมาไม่ตรงกับ Area ที่ Interface รับ
ฝั่งตนตั้งไว้ Router จะ**ปฏิเสธ Hello Packet ทั้งใบทันที** ไม่มีการสร้าง Neighbor Entry ขึ้นมาเลย
แม้แต่ State Down — ต่างจาก Timer/Authentication Mismatch ที่มักยังเห็น Neighbor ค้างอยู่ที่ State
ต่ำๆ ได้บ้าง

### 564.6 วิธีแก้และ Verify

```
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# ip ospf 1 area 1                 ! แก้ให้ตรงกับ CORE-SW1 (Area 1)
```

> **หมายเหตุ**: คำสั่ง `ip ospf 1 area 1` ทับคำสั่งเดิมได้ตรงๆ ไม่ต้อง `no ip ospf 1 area 2` ก่อน
> IOS จะย้าย Interface จาก Area เดิมไป Area ใหม่ให้อัตโนมัติทันทีที่พิมพ์ค่าใหม่ทับ

```
CORE-SW1# show ip ospf neighbor | include 1.1.1.12
1.1.1.12          0   FULL/  -        00:00:39    10.255.20.2     TenGigabitEthernet1/0/2
```

Neighbor ขึ้น FULL ทันที และ `show ip route ospf` บน CORE-SW1 กลับมาเห็น VLAN10/20/99 ของ
DIST-SW2 ผ่าน Path-Type `O` (Intra-area) ปกติทุกประการ ยืนยันว่า DIST-SW2 กลับเข้ามาเป็น Internal
Router ของ Area 1 อย่างถูกต้องแล้ว

---

## Step 565 — Scenario 4: Route หายเพราะ `area range` กว้างเกินไป

### 565.1 บริบท

ทีม Security เพิ่มเครือข่ายเล็กๆ สำหรับ IoT Sensor ชุดใหม่บน DIST-SW1 ชื่อ **VLAN 90
(10.10.90.0/24)** โดยตั้งใจให้ CORE-SW2/DIST-SW3/DIST-SW4 (ฝั่ง Area 2) เห็น Subnet นี้เป็น
**Prefix เฉพาะเจาะจง** (ไม่ใช่แค่รวมอยู่ใน Summary) เพราะทีม Firewall ต้องใช้ Prefix นี้ตรงๆ ใน
`distribute-list`/ACL เพื่อจำกัดสิทธิ์ Sensor เหล่านี้ไม่ให้ปนกับ VLAN อื่นของ Area 1 — แต่ CORE-SW1
มี `area range` ที่ตั้งไว้ตั้งแต่ [Part 12 Step 115.2](part-012-ospfv2-advanced.md) อยู่แล้ว:

```
CORE-SW1# show running-config | section router ospf
router ospf 1
 router-id 1.1.1.1
 area 1 range 10.10.0.0 255.255.0.0          ! <-- ครอบคลุม 10.10.0.0 - 10.10.255.255 ทั้งหมด
 area 1 range 10.255.10.0 255.255.255.252 not-advertise
 area 1 range 10.255.20.0 255.255.255.252 not-advertise
```

`10.10.90.0/24` อยู่ในขอบเขตของ `10.10.0.0/16` เป๊ะ — ทันทีที่เพิ่ม VLAN90 เข้า Area 1 มันจะถูก
Summarize เข้าไปในก้อนเดียวกับ VLAN10/20/99 โดยอัตโนมัติทันที **โดยไม่มีใครต้องสั่งอะไรเพิ่ม**

### 565.2 อาการที่พบ

```
! ที่ DIST-SW1 (Area 1) — เห็น VLAN90 เป็น Intra-area ปกติ ไม่มีปัญหาฝั่งนี้
DIST-SW1# show ip route ospf | include 10.10.90
O       10.10.90.0/24 [110/1] via 0.0.0.0, Vlan90, directly connected

! แต่ที่ CORE-SW2 (Area 2 — ฝั่งที่ต้องการเห็น Prefix เฉพาะเจาะจง)
CORE-SW2# show ip route ospf | include 10.10.90
                                                     <-- ไม่มีบรรทัดนี้เลย!
CORE-SW2# show ip route ospf | include 10.10.0.0
O IA    10.10.0.0/16 [110/11] via 10.255.30.2, TenGigabitEthernet1/0/1
                                                     <-- เห็นแค่ก้อน /16 เท่านั้น ไม่เห็น /24 แยก
```

Firewall Team Deploy ACL ที่คาดหวัง Match `10.10.90.0/24` ตรงๆ แล้วพบว่า Traffic ทั้งก้อน
`10.10.0.0/16` (รวมถึง VLAN อื่นที่ไม่ใช่ IoT) ถูกจัดการด้วย Rule เดียวกันหมด — เป็นปัญหาด้าน
Security Policy ไม่ใช่ Connectivity (Ping ยังผ่านได้ปกติเพราะ Aggregate Route ก็ชี้ไปถูกทาง)
แต่ก็ยังนับเป็น "Route ที่ต้องการหาย" ตามที่ ENARSI ชอบทดสอบ

### 565.3 การวินิจฉัย

```
! เช็คที่ต้นตอ — CORE-SW1 (ABR) เห็น VLAN90 ใน LSDB ของ Area 1 (Type 1) จริงไหม
CORE-SW1# show ip ospf database router 1.1.1.11 | include 10.10.90
      Link connected to: a Transit Network
       (Link ID) Designated Router address: 10.10.90.1
                                                    <-- มีอยู่จริงใน Area 1 LSDB (Type 1)

! แต่ Summary LSA (Type 3) ที่ CORE-SW1 Originate ออกไป Area 0/Area 2 มีแค่ก้อนใหญ่
CORE-SW1# show ip ospf database summary | include 10.10
Link ID         ADV Router      Age         Seq#       Checksum
10.10.0.0       1.1.1.1         120         0x80000015  0x4A2B
                                                    <-- ไม่มี 10.10.90.0 เป็น Type 3 แยกเลย
```

ยืนยันได้ว่า **VLAN90 มีอยู่จริงใน Area 1 (Type 1 Router LSA)** แต่ **ไม่ถูก Originate เป็น Type 3
แยกออกจาก Area 1** เพราะ `area range` ที่ CORE-SW1 ครอบคลุมมันไปแล้ว — ต่างจาก Scenario ที่ Route
หายเพราะ `distribute-list` (ซึ่งจะยังเห็น Type 3 LSA ปรากฏอยู่แต่แค่ไม่ถูกติดตั้งเข้า RIB — อ้างอิง
[Part 26 Step 251.1-251.3](part-026-advanced-ospf.md)) กรณีนี้ **Type 3 LSA เฉพาะของ VLAN90 ไม่
ถูกสร้างขึ้นมาตั้งแต่ต้นเลย** เพราะ Summarization ทำงานที่ตัว ABR ก่อนที่จะไปถึงขั้น Filter ด้วยซ้ำ

### 565.4 Root Cause

**`area range` มีขอบเขตกว้างกว่าที่ตั้งใจ** — คำสั่ง `area 1 range 10.10.0.0 255.255.0.0` ถูกตั้ง
ไว้ตั้งแต่ Part 12 เพื่อสรุป VLAN10/20/99 (ซึ่งกระจัดกระจายอยู่ใน Octet ที่ 3 คนละค่า: .10, .20,
.99) — Mask `/16` จึงต้องกว้างพอที่จะครอบทั้งสามค่านี้พร้อมกัน แต่ผลข้างเคียงคือ **Subnet ใหม่ใดๆ
ที่ถูกเพิ่มเข้ามาใน Area 1 ในอนาคตที่อยู่ในขอบเขต 10.10.0.0/16 จะถูกดูดเข้า Summary นี้โดย
อัตโนมัติเสมอ ไม่มีทางแยกออกมาได้ตราบใดที่ยังใช้ Subnet อยู่ในขอบเขตเดียวกัน** — IOS ไม่มีคำสั่ง
"Exclude Subnet ออกจาก Range" แบบตรงๆ (`area range` ไม่รองรับ Prefix-list Exception)

### 565.5 วิธีแก้ — Address Planning คือทางแก้ที่ถูกต้อง ไม่ใช่การถอด `area range`

| ตัวเลือก | รายละเอียด | ควรใช้ไหม |
|---|---|---|
| **ถอด `area range` ทั้งหมด** | VLAN90 จะกลับมาเห็นเป็น /24 แยก แต่ VLAN10/20/99 ก็จะแยกกลับไปเป็น 3 เส้นเหมือนก่อน Part 12 Step 115 | ❌ เสีย Benefit ของ Summarization ทั้งหมดเพื่อแก้ปัญหาแค่ 1 Subnet |
| **ย้าย VLAN90 ไปใช้ Subnet นอกขอบเขต 10.10.0.0/16** (เช่น `10.20.90.0/24`) | Subnet ใหม่จะไม่ตรงกับ `area range` เดิมเลย จึงถูก Originate เป็น Type 3 แยกโดยอัตโนมัติ | ✅ วิธีที่ถูกต้อง — แก้ที่ผังการจ่าย Subnet ล่วงหน้า |
| **สร้าง `area range` ก้อนใหม่แยกที่แคบกว่าเฉพาะ VLAN10/20/99** | ทำไม่ได้ในกรณีนี้เพราะ 3 VLAN กระจาย Octet ไม่ติดกัน ต้องใช้ Mask กว้างเท่าเดิมอยู่ดี | ❌ ไม่ช่วยอะไร |

```
! Fix จริงที่ Deploy — Renumber VLAN90 ให้อยู่นอกขอบเขต Summary เดิม
DIST-SW1(config)# interface Vlan90
DIST-SW1(config-if)# ip address 10.20.90.1 255.255.255.0
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan90
DIST-SW1(config-if)# ip ospf 1 area 1
```

```
CORE-SW2# show ip route ospf | include 10.20.90
O IA    10.20.90.0/24 [110/11] via 10.255.30.2, TenGigabitEthernet1/0/1
                                                    <-- ปรากฏเป็น Prefix เฉพาะเจาะจงแล้ว
```

### 565.6 ตัวแปรที่สอง — `filter-list` กว้างเกินไปให้ผลลัพธ์เดียวกัน

ปัญหาแบบเดียวกันเกิดได้จาก **`area filter-list`** ที่สอนใน [Part 26 Step 251.4](part-026-advanced-ospf.md)
ถ้า Prefix-list ที่ใช้เขียนกว้างเกินไป เช่น ตั้งใจ Block แค่ `10.10.40.0/24` (VLAN40) แต่เขียน
Prefix-list ผิดเป็น:

```
! ผิด — le 32 ทำให้ Match ทุก Subnet ย่อยของ 10.10.0.0/16 ทั้งหมด ไม่ใช่แค่ VLAN40
ip prefix-list BLOCK-VLAN40 seq 5 permit 10.10.0.0/16 le 32
```

ผลคือ **Type 3 ทุกก้อนที่ตรงกับ 10.10.0.0/16 ถูก Block ทั้งหมด** (รวมถึง Summary 10.10.0.0/16
เองที่ CORE-SW1 พยายาม Re-advertise กลับเข้า Area 1 ตามที่สอนใน Part 12 Step 120.5) — วิธีตรวจคือ
`show ip prefix-list detail BLOCK-VLAN40` เทียบกับ Requirement จริง แล้วแก้ให้เจาะจงแค่ Subnet
ที่ต้องการ:

```
! ถูก — เจาะจงเฉพาะ VLAN40 เท่านั้น
ip prefix-list BLOCK-VLAN40 seq 5 permit 10.10.40.0/24
```

> **บทเรียนรวมของ Step นี้**: ทั้ง `area range` และ `area filter-list` เป็นเครื่องมือที่ทรงพลัง
> ระดับ LSA แต่ก็เป็นดาบสองคม — **ยิ่งกว้างยิ่งเสี่ยงกลืน Subnet ที่ไม่ตั้งใจ** ต้องออกแบบ Address
> Plan ให้สอดคล้องกับขอบเขต Summary ตั้งแต่ต้น (เช่น เผื่อ "Reserved Block" ไว้นอกขอบเขต Summary
> สำหรับ Subnet ที่ต้องมองเห็นแบบเจาะจงในอนาคต)

---

## Step 566 — Scenario 5: Authentication Type ไม่ตรงกันหลัง Rollout ครึ่งเดียว

### 566.1 บริบท

ทีม Network Security สั่งให้หมุน (Rotate) MD5 Key ของ Backbone Link ทั้งหมดตามนโยบายรอบปี — ตาม
[Part 11 Step 108.5](part-011-ospfv2-fundamentals.md) เดิมทุก Link ใช้ Key ID 1 (`OSPF-BACKBONE-K3y!`)
วิศวกรเริ่ม Rollout ที่ **CORE-SW2 ↔ DIST-SW3** ก่อน โดยตั้งใจจะเพิ่ม Key ID 2 คู่กับ Key ID 1
แบบ Rolling (ตามที่ Part 11 Step 108.3 อธิบายไว้) แต่ระหว่างพิมพ์คำสั่งบน **DIST-SW3** เผลอกด
`no ip ospf authentication message-digest` (ตั้งใจจะลบ Key ID เก่าแต่ไปลบคำสั่งเปิด Authentication
Mode ทั้งเส้นแทน) ทำให้ DIST-SW3 เหลือแค่ `message-digest-key` อยู่ในระบบเฉยๆ โดยไม่มีคำสั่งเปิด
ใช้งาน Authentication Mode เลย (เทียบเท่ากับไม่มี Authentication ใดๆ)

### 566.2 Config ที่ผิด

```
! DIST-SW3 — คำสั่งเปิด Authentication Mode หายไป (เหลือ Key ทิ้งไว้เฉยๆ ไม่มีผล)
DIST-SW3(config)# interface TenGigabitEthernet1/1/1
DIST-SW3(config-if)# no ip ospf authentication message-digest
! (message-digest-key 1 md5 ... ยังอยู่ใน running-config แต่ไม่ถูกใช้งานแล้ว)

! CORE-SW2 (ฝั่งตรงข้าม) — ยังคง MD5 ปกติทุกประการ ไม่มีการแก้ไข
```

### 566.3 อาการที่พบ

```
CORE-SW2# show ip ospf neighbor | include 1.1.1.13
                                                     <-- Neighbor หลุดหายไปจาก Table ทันที

CORE-SW2# show logging | include ADJCHG|ERRRCV
%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.13 on TenGigabitEthernet1/0/1 from FULL to DOWN, Neighbor Down: Dead timer expired
%OSPF-4-ERRRCV: Received invalid packet: mismatch authentication type from 10.255.30.2, TenGigabitEthernet1/0/1
%OSPF-4-ERRRCV: Received invalid packet: mismatch authentication type from 10.255.30.2, TenGigabitEthernet1/0/1
```

Neighbor ที่เดิมเป็น **FULL มานานแล้ว** หลุดกะทันหันไม่มีสาเหตุ Physical ใดๆ — Syslog ระบุตรงตัว
`mismatch authentication type`

### 566.4 การวินิจฉัย

```
CORE-SW2# show ip ospf interface TenGigabitEthernet1/0/1 | include authentication
  Cryptographic authentication enabled
    Youngest key id is 1

DIST-SW3# show ip ospf interface TenGigabitEthernet1/1/1 | include authentication
                                                     <-- ไม่มีบรรทัดใดเกี่ยวกับ Authentication เลย!
```

CORE-SW2 ยังยืนยัน `Cryptographic authentication enabled` (Type 2 = MD5) แต่ DIST-SW3 **ไม่มี
บรรทัด Authentication ปรากฏเลย** ยืนยันว่า Authentication Type ของ DIST-SW3 ตกกลับไปเป็น
**Type 0 (None)** — ยิ่งชัดเจนขึ้นด้วย `debug ip ospf adj` ที่ DIST-SW3:

```
DIST-SW3# debug ip ospf adj
OSPF-1 ADJ  Te1/1/1: Rcv pkt from 10.255.30.1, TenGigabitEthernet1/1/1: Mismatch Authentication type. Input packet specified type 2, we use type 0
```

ข้อความ **"Input packet specified type 2, we use type 0"** คือคำอธิบายตรงที่สุด — CORE-SW2 ส่ง
Packet มาแบบ Type 2 (MD5) แต่ DIST-SW3 คาดหวัง Type 0 (None) เพราะคำสั่งเปิด MD5 Mode หายไปแล้ว

### 566.5 Root Cause

**Authentication Type ไม่ตรงกันหลัง Rollout ครึ่งเดียว** — คำสั่ง `ip ospf authentication
message-digest` (ตัวกำหนด Authentication **Mode/Type**) ถูกลบไปโดยไม่ตั้งใจ ทำให้ Interface ที่
ยังมี `message-digest-key` เหลืออยู่**ไม่มีผลอะไรเลย** เพราะ Key ใช้งานได้ก็ต่อเมื่อ Mode ถูกเปิด
อยู่เท่านั้น — ตรงกับที่ [Part 11 Step 108.3](part-011-ospfv2-fundamentals.md) เน้นย้ำว่า "ต้องมี
ทั้งสองคำสั่งจึงทำงาน" (ตั้ง Key + เปิด Mode) ขาดอย่างใดอย่างหนึ่งไม่ได้

### 566.6 วิธีแก้และ Verify

```
DIST-SW3(config)# interface TenGigabitEthernet1/1/1
DIST-SW3(config-if)# ip ospf authentication message-digest      ! คืนคำสั่งเปิด MD5 Mode กลับมา
```

```
DIST-SW3# show ip ospf interface TenGigabitEthernet1/1/1 | include authentication
  Cryptographic authentication enabled
    Youngest key id is 1

CORE-SW2# show ip ospf neighbor | include 1.1.1.13
1.1.1.13          0   FULL/  -        00:00:35    10.255.30.2     TenGigabitEthernet1/0/1
```

> **เทคนิคป้องกันไม่ให้เกิดซ้ำ**: เมื่อทำ Key Rotation แบบ Rolling ตาม Part 11 Step 108.3 ให้
> **เพิ่ม Key ID ใหม่ก่อนเสมอ แล้วค่อยลบ Key ID เก่าทีหลังสุด** ไม่ควรไปแก้คำสั่งเปิด/ปิด
> Authentication Mode เลยในระหว่างกระบวนการ Rotation — จะช่วยตัดความเสี่ยงที่จะพิมพ์ผิดคำสั่งจน
> ปิด Mode ไปโดยไม่ตั้งใจแบบ Scenario นี้ได้เกือบทั้งหมด

---

## Step 567 — Scenario 6: Suboptimal Routing จาก `auto-cost reference-bandwidth` ไม่ครบทุกตัว

### 567.1 บริบท

[Part 26 Step 257](part-026-advanced-ospf.md) สั่งให้ตั้ง `auto-cost reference-bandwidth 100000`
(100 Gbps) บน **ทุก Router ในหลักสูตร** เพื่อแก้ปัญหา Cost Collision ระหว่าง Gig/10G — แต่ทีม WAN
มักดูแล **WAN-EDGE-1/WAN-EDGE-2 แยกจากทีม Campus** (ทีม Campus ดูแล CORE-SW1/CORE-SW2/DIST-SW1-4)
ในการ Rollout จริง ทีม Campus ปรับค่าครบทุกตัวของตัวเอง แต่ไม่ได้แจ้งทีม WAN ให้ปรับ WAN-EDGE-1
ตามด้วย — WAN-EDGE-1 จึงยังเหลือค่า Default (`Reference bandwidth unit is 100 mbps`)

### 567.2 ทำไมค่านี้ทำให้เลือก Path ผิด — คำนวณให้ดูเป็นตัวเลข

Topology ที่เกี่ยวข้อง (ทั้งหมดอยู่ Area 0): CORE-SW1 มี **2 เส้นทาง** ไปถึง Loopback0 ของ
WAN-EDGE-2 (`1.1.1.22/32`):

```
Path A: CORE-SW1 --Gi1/0/1--> WAN-EDGE-1 --Gi0/0/2(cross-link)--> WAN-EDGE-2
Path B: CORE-SW1 --Po1(10G)--> CORE-SW2 --Gi1/0/1--> WAN-EDGE-2
```

| Router ที่ Advertise Cost ของ Link ขาออกตัวเอง | Ref-BW ที่ใช้จริง | Cost ที่คำนวณได้ (GigE) | Cost ที่คำนวณได้ (Po1 10G) |
|---|---|---|---|
| CORE-SW1 (Gi1/0/1, Po1) | 100000 Mbps (ตั้งถูกต้อง) | 100 | 10 |
| CORE-SW2 (Gi1/0/1) | 100000 Mbps (ตั้งถูกต้อง) | 100 | — |
| **WAN-EDGE-1 (Gi0/0/2 cross-link)** | **100 Mbps (Default — ไม่ได้ตั้ง!)** | **1** (ปัดขั้นต่ำ ผิดปกติ) | — |

Cost ของแต่ละ Hop ถูก**กำหนดโดย Router ที่ Advertise Link ขาออกนั้นเอง** (ไม่ใช่ผู้รับคำนวณใหม่) —
ดังนั้น:

```
Cost รวม Path A (มุมมอง CORE-SW1) = CORE-SW1's Gi1/0/1 (100, ถูกต้อง)
                                   + WAN-EDGE-1's cross-link (1, ผิดปกติเพราะ Ref-BW เพี้ยน)
                                   = 101

Cost รวม Path B (มุมมอง CORE-SW1) = CORE-SW1's Po1 (10, ถูกต้อง)
                                   + CORE-SW2's Gi1/0/1 (100, ถูกต้อง)
                                   = 110
```

**101 < 110 → CORE-SW1 เลือก Path A** (ผ่าน 2 เส้น GigE ต่อกัน คอขวดจริงคือ 1 Gbps) **ทั้งที่ Path
B ผ่าน Backbone 10G ควรถูกเลือกมากกว่า** — สาเหตุทั้งหมดมาจาก WAN-EDGE-1 Advertise Cost ของ
Cross-link ต่ำผิดปกติเพราะ Ref-BW ของตัวเองยังเป็นค่า Default

### 567.3 อาการที่พบจริงบน CLI

```
CORE-SW1# show ip route 1.1.1.22
Routing entry for 1.1.1.22/32
  Known via "ospf 1", distance 110, metric 101, type intra area
  Last update from 10.10.254.1 on GigabitEthernet1/0/1, 00:12:03 ago
  Routing Descriptor Blocks:
  * 10.10.254.1, from 1.1.1.22, 00:12:03 ago, via GigabitEthernet1/0/1
      Route metric is 101, traffic share count is 1
                                          <-- ไปทาง WAN-EDGE-1 (คอขวด 1G) ทั้งที่มี Backbone 10G
```

### 567.4 การวินิจฉัย

```
WAN-EDGE-1# show ip ospf | include Reference
    Reference bandwidth unit is 100 mbps

CORE-SW1# show ip ospf | include Reference
    Reference bandwidth unit is 100000 mbps

CORE-SW2# show ip ospf | include Reference
    Reference bandwidth unit is 100000 mbps
```

เทียบ `show ip ospf | include Reference` ทีละตัวทั้งโดเมนคือวิธีที่เร็วที่สุดในการหา Router ที่
"ตกหล่น" จากการ Rollout — ยืนยันอีกชั้นด้วย `show ip ospf interface` เทียบ Cost ที่แต่ละตัว
คำนวณสำหรับ Interface ความเร็วเดียวกัน (ทุก GigE ควรมี Cost 100 เท่ากันหมดถ้า Ref-BW ตรงกัน):

```
WAN-EDGE-1# show ip ospf interface GigabitEthernet0/0/2 | include Cost
  Process ID 1, Router ID 1.1.1.21, Network Type BROADCAST, Cost: 1        <-- ผิดปกติ! ควรเป็น 100
```

### 567.5 Root Cause

**`auto-cost reference-bandwidth` ไม่ได้ตั้งให้ครบทุก Router ในโดเมนเดียวกัน** — ตามกฎเหล็กที่
[Part 26 Step 257](part-026-advanced-ospf.md) ย้ำไว้ ค่านี้**ไม่จำเป็นต้อง Match กันทางโปรโตคอล**
(OSPF จะไม่ปฏิเสธ Adjacency แม้ Ref-BW ต่างกัน — ไม่เหมือน Hello/Dead Timer หรือ Area ID) แต่
**ต้อง Match กันเพื่อให้ Cost มีความหมายตรงกันทั้งโดเมน** เพราะ Cost ของแต่ละ Link ถูกกำหนดขึ้นที่
Router ต้นทางของ Link นั้นเพียงฝ่ายเดียว ถ้า Router ใดตัวหนึ่งยังใช้ Ref-BW เดิม การเปรียบเทียบ Cost
ข้าม Router จะไม่สมเหตุสมผลอีกต่อไป — และเพราะ Adjacency ยังขึ้น Full ปกติทุกประการ **ปัญหานี้จะ
ไม่มี Log เตือนใดๆ เลย** ต้องอาศัยการเทียบค่าด้วยมือเท่านั้นจึงจะพบ

### 567.6 วิธีแก้และ Verify

```
WAN-EDGE-1(config)# router ospf 1
WAN-EDGE-1(config-router)# auto-cost reference-bandwidth 100000
% OSPF: Reference bandwidth is changed.
        Please ensure reference bandwidth is consistent across all routers.
```

> IOS เตือนประโยคนี้ให้ทุกครั้งที่เปลี่ยนค่า Reference Bandwidth — เป็นการยืนยันจาก Cisco เองว่า
> ค่านี้**ต้อง Consistent ทั้งโดเมน** ตรงตามที่สอนไว้

```
WAN-EDGE-1# show ip ospf interface GigabitEthernet0/0/2 | include Cost
  Process ID 1, Router ID 1.1.1.21, Network Type BROADCAST, Cost: 100      <-- ถูกต้องแล้ว

CORE-SW1# show ip route 1.1.1.22
Routing entry for 1.1.1.22/32
  Known via "ospf 1", distance 110, metric 110, type intra area
  * 10.255.0.2, from 1.1.1.22, 00:00:04 ago, via Port-channel1
      Route metric is 110, traffic share count is 1
                                          <-- Path เปลี่ยนไปทาง Backbone 10G (Po1) ถูกต้องแล้ว
```

Cost ของ Path A กลับมาเป็น 100(CORE-SW1)+100(WAN-EDGE-1 cross-link ที่แก้แล้ว) = 200 ซึ่งมากกว่า
Path B (110) แล้ว SPF จึงเลือก Path B ผ่าน Backbone 10G ให้ถูกต้องตามที่ควรจะเป็น

---

## Step 568 — Scenario 7: NSSA Mismatch ตัด Area 2 ขาดจาก Backbone ทั้ง Area

### 568.1 บริบท

[Part 26 Step 253.5](part-026-advanced-ospf.md) แปลง **Area 2 เป็น NSSA** ทั้ง Area (CORE-SW2,
DIST-SW3, DIST-SW4 ทุกตัวตั้ง `area 2 nssa` ให้ตรงกัน) เพื่อรองรับ ASBR ที่ Redistribute Route
ภายนอก (`172.16.30.0/24`) เข้ามาโดยไม่ต้องรับ External Type 5 เต็มโดเมน — ทีม Operation ทำความ
สะอาด Config ประจำเดือน (Config Audit) เข้าใจผิดว่า `area 2 nssa` บน CORE-SW2 เป็น Config
ทดสอบที่ Part 26 หลงเหลือไว้และไม่จำเป็นแล้ว จึงลบออกจาก CORE-SW2 ตัวเดียว (ไม่ได้แก้ DIST-SW3/
DIST-SW4)

### 568.2 Config ที่ผิด

```
! CORE-SW2 — ถูกถอด NSSA ออกโดยเข้าใจผิดว่าเป็น Config ทดสอบ (DIST-SW3/DIST-SW4 ยังเป็น NSSA เดิม)
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# no area 2 nssa
```

### 568.3 อาการที่พบ

```
CORE-SW2# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:38    10.255.0.1      Port-channel1
                                                    <-- เหลือแค่ Neighbor ฝั่ง Area 0 (CORE-SW1)
                                                        Neighbor ฝั่ง Area 2 (DIST-SW3/DIST-SW4) หายไปหมด!
```

**Area 2 ทั้ง Area ขาดการเชื่อมต่อกับ CORE-SW2 (ABR ตัวเดียวของ Area 2) ทันที** — DIST-SW3 และ
DIST-SW4 ยัง Up อยู่ปกติแต่ไม่มี Full Adjacency กับ CORE-SW2 เลย ซึ่งหมายความว่า **Area 2 ทั้งก้อน
ขาดจาก Backbone (Area 0) โดยสมบูรณ์** เพราะ CORE-SW2 เป็น ABR ตัวเดียวที่เชื่อม Area 2 เข้ากับ
Area 0 (ไม่มี ABR สำรองในสถาปัตยกรรม Single-homed ปัจจุบันตามที่ [Part 12 Step
114.2](part-012-ospfv2-advanced.md) เคยเตือนไว้)

```
DIST-SW3# show ip ospf neighbor
                                                    <-- ไม่มี Neighbor เลยแม้แต่ตัวเดียว

DIST-SW3# show ip ospf border-routers
                                                    <-- ว่างเปล่า ไม่เห็น ABR/ASBR ตัวใดเลย
OSPF Process 1 internal Routing Table
Codes: i - Intra-area route, I - Inter-area route

! เทียบกับก่อนพัง (ค่าอ้างอิงปกติ) ที่ต้องเห็นแบบนี้:
! i   1.1.1.2  [10] via 10.255.30.1, TenGigabitEthernet1/1/1, ABR, Area 2
```

`show ip ospf border-routers` ที่ว่างเปล่าคือหลักฐานตรงตัวที่สุดของคำว่า **"ไม่มี Full Adjacency
ไปถึง Backbone"** — DIST-SW3 มองไม่เห็น ABR ตัวใดเลยในระบบ (ไม่ใช่แค่มองไม่เห็น Route ปลายทาง แต่
มองไม่เห็นเส้นทางไปถึง ABR ตั้งแต่ต้น)

### 568.4 การวินิจฉัย

```
CORE-SW2# show ip ospf | section Area 2
    Area 2
        Number of interfaces in this area is 2
        It is a normal area                              <-- ผิด! ควรเป็น NSSA
        Area has no authentication

DIST-SW3# show ip ospf | section Area 2
    Area 2
        Number of interfaces in this area is 4
        It is a NSSA area                                 <-- ยังเป็น NSSA เหมือนเดิม
        Perform type-7/type-5 LSA translation
```

`show ip ospf | section Area 2` เทียบสองฝั่งเห็นชัดเจนที่สุด — CORE-SW2 บอกว่า **"It is a normal
area"** ในขณะที่ DIST-SW3/DIST-SW4 ยังบอกว่า **"It is a NSSA area"** — นี่คือ Area Type Mismatch
ระดับเดียวกับ Stub/NSSA/Normal ที่ Hello Packet บรรจุ Option Bit (N-bit) แจ้งประเภท Area ไปด้วย
ถ้า N-bit ไม่ตรงกัน Router จะปฏิเสธ Hello ทันทีเหมือนกับ Area ID Mismatch (Step 564) เพียงแต่
สาเหตุคือ **Type ของ Area ไม่ตรง** ไม่ใช่ **หมายเลข Area ไม่ตรง**:

```
DIST-SW3# debug ip ospf adj
OSPF-1 ADJ  Te1/1/1: Rcv Hello from 1.1.1.2 10.255.30.1 area 2, wrong Area type NSSA/Normal mismatch
```

### 568.5 Root Cause

**NSSA Flag ไม่ตรงกันระหว่าง ABR (CORE-SW2) กับ Internal Router (DIST-SW3, DIST-SW4) ของ Area
เดียวกัน** — [Part 12 Step 116](part-012-ospfv2-advanced.md) ระบุไว้ชัดว่า Area Type (Standard/
Stub/NSSA) **ต้องตั้งให้ตรงกันทุกตัวในทุก Router ของ Area นั้น** เพราะ Area Type ไม่ใช่แค่
Local Config เฉยๆ แต่ถูกประกาศผ่าน Hello Packet ด้วย — เมื่อ ABR ตัวเดียวของ Area 2 ไม่สามารถ
สร้าง Adjacency กับ Internal Router ใดๆ ใน Area 2 ได้เลย ผลลัพธ์คือ **Area 2 ทั้ง Area สูญเสีย
Backbone Reachability ทันที** แม้ Physical Link จะยัง Up ทุกจุดก็ตาม

> **เทียบกับ Virtual-Link**: [Part 12 Step 117](part-012-ospfv2-advanced.md) สอน Virtual-Link
> เป็นเครื่องมือกู้สถานการณ์เมื่อ ABR ตัวหลักเสียการเชื่อมต่อ Area 0 **โดยมี Physical Path สำรอง
> ผ่าน Area ตัวกลางอื่น (Transit Area)** — แต่ในสถาปัตยกรรม Single-homed ปัจจุบันของหลักสูตรนี้
> Area 2 มี ABR เดียว (CORE-SW2) และไม่มี Physical Path สำรองไปยัง Area 0 เส้นอื่นเลย Virtual-Link
> จึง **ช่วยอะไรไม่ได้ในกรณีนี้** (ไม่มี Physical Path ให้ผูก Virtual-Link ทับ) — ต้องรอ
> **Dual-homed CORE-DIST Redundancy ใน Part 33-35** ตามที่ IP Plan ระบุไว้ ก่อน Virtual-Link จะมี
> ประโยชน์จริงกับ Area 2 ได้ ปัญหานี้จึงต้องแก้ที่ **Root Cause ของ NSSA Mismatch ตรงๆ เท่านั้น**

### 568.6 วิธีแก้และ Verify

```
CORE-SW2(config)# router ospf 1
CORE-SW2(config-router)# area 2 nssa
```

```
CORE-SW2# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:35    10.255.0.1      Port-channel1
1.1.1.13          0   FULL/  -        00:00:31    10.255.30.2     TenGigabitEthernet1/0/1
1.1.1.14          0   FULL/  -        00:00:38    10.255.40.2     TenGigabitEthernet1/0/2

DIST-SW3# show ip ospf border-routers
OSPF Process 1 internal Routing Table
Codes: i - Intra-area route, I - Inter-area route

i    1.1.1.2   [10] via 10.255.30.1, TenGigabitEthernet1/1/1, ABR, Area 2
                                          <-- กลับมาเห็น ABR แล้ว ยืนยัน Backbone Reachability คืนมา
```

---

## Step 569 — Scenario 8: LSA Flooding Storm / CPU พุ่งจาก Interface Flap แม้มี Throttle Timer

### 569.1 บริบท

DIST-SW4 มี SFP เสื่อมสภาพบน Uplink `TenGigabitEthernet1/1/1` (ไป CORE-SW2) ทำให้ Link **Flap
ทุก 2-3 วินาทีต่อเนื่องนานกว่า 20 นาที** — [Part 26 Step 260](part-026-advanced-ospf.md) เปิด
**BFD (150ms)** และ **`timers throttle spf/lsa 10 100 5000`** ไว้ครบทุกตัวแล้วเพื่อป้องกัน CPU
พังจากเหตุการณ์แบบนี้โดยเฉพาะ — คำถามคือ Throttle Timer ที่มีอยู่แล้วเพียงพอไหม

### 569.2 อาการที่พบ

```
CORE-SW2# show processes cpu sorted | exclude 0.00%  0.00%  0.00%
CPU utilization for five seconds: 38%/12%; one minute: 41%; five minutes: 37%
 PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process
 142      812340      184213       4409  22.15% 24.88% 21.03%   0 OSPF-1 Router
 118       94210       88213       1068   3.02%  3.11%  2.98%   0 BFD Process
```

CPU สูงกว่าปกติต่อเนื่อง (ปกติของ Lab นี้ CPU ควรอยู่ต่ำกว่า 5%) และ Process ที่กิน CPU มากที่สุด
คือ **OSPF-1 Router** ยืนยันว่าไม่ใช่ Process อื่นที่ผิดปกติ

```
CORE-SW2# show ip ospf statistics
  SPF calculation executed 187 times, latest at 00:00:04 ago
  Area 2: SPF algorithm executed 187 times
```

**187 ครั้งใน 20 นาที** เฉลี่ยเกือบ 1 ครั้งทุก 6-7 วินาที — สูงกว่าปกติมาก (Lab ปกติควรเห็นเลข
หลักหน่วยถึงสิบต้นๆ ต่อวัน ไม่ใช่ต่อนาที)

### 569.3 การวินิจฉัย — ไล่ตาม Workflow จาก Part 26 Step 259.4

```
1. show ip ospf statistics                 → พบ SPF Run ถี่ผิดปกติ (ยืนยันแล้วข้างบน)

2. show logging | include LINEPROTO
%LINEPROTO-5-UPDOWN: Line protocol on Interface TenGigabitEthernet1/1/1, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface TenGigabitEthernet1/1/1, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface TenGigabitEthernet1/1/1, changed state to down
   (ซ้ำเป็นสิบๆ รอบ ทุก 2-3 วินาที บน DIST-SW4)

3. show interfaces TenGigabitEthernet1/1/1 | include error
   Input queue: 0/375/1421/0 (size/max/drops/flushes); Total output drops: 0
   1847 input errors, 1802 CRC, 0 frame, 0 overrun, 0 ignored
                                          <-- CRC Error สูงมาก ยืนยัน Physical Layer เสีย

4. show ip ospf statistics detail
   1.1.1.14 originated 94 Router-LSAs in the last 20 minutes
                                          <-- DIST-SW4 คือต้นตอ LSA ซ้ำๆ ตัวจริง
```

### 569.4 ทำไม BFD ยิ่งทำให้ Trigger ถี่ขึ้น (ไม่ใช่ Bug — เป็น Trade-off ที่ต้องเข้าใจ)

[Part 26 Step 254](part-026-advanced-ospf.md) เปิด BFD 150ms เพื่อให้ตรวจจับ Failure **เร็วกว่า
Dead Timer (40s) มาก** — แต่ในเคสที่ Physical Layer Flap จริงถี่ระดับวินาที BFD ก็ยิ่งตรวจจับและ
แจ้ง OSPF ให้ตัด/สร้าง Neighbor ใหม่**ถี่ตามไปด้วย** (ถ้าไม่มี BFD, Dead Timer 40s อาจจะ "กรอง"
การ Flap สั้นๆ บางรอบออกไปได้เองโดยไม่ทัน Trigger เพราะ Flap แต่ละรอบสั้นกว่า Dead Interval) —
นี่คือ Trade-off ที่ต้องเข้าใจสำหรับ ENARSI: **BFD แลก Convergence เร็วขึ้นกับความถี่ของ Event ที่
มากขึ้นในกรณี Flapping จริง** ส่วน `timers throttle spf/lsa 10 100 5000` ทำหน้าที่ "เพดานเบรก"
ไม่ให้ SPF/LSA วิ่งเร็วกว่า Exponential Backoff ที่กำหนด (สูงสุดคือรอ 5000ms ต่อรอบหลัง Backoff
เต็มที่) — แต่ถ้า Flap ยังไม่หยุดต่อเนื่อง SPF ก็ยัง**วิ่งซ้ำที่เพดาน 5 วินาที/รอบตลอด 20 นาที**
ซึ่งยังกิน CPU สะสมได้พอสมควรอยู่ดี (Throttle ช่วย "ลดอัตรา" แต่ไม่ได้ "หยุด" การเกิดซ้ำ)

### 569.5 Root Cause

**Physical Layer จริง (SFP เสื่อม) ทำให้ Interface Flap ต่อเนื่อง** — Throttle Timer และ BFD ทำงาน
ถูกต้องตามที่ Config ไว้ทุกประการ (ไม่ใช่ Bug ของ Config) แต่**ไม่สามารถแก้ปัญหาที่ต้นเหตุทาง
Hardware ได้** เป็นเพียง Mitigation ที่ช่วยให้ระบบยังพอใช้งานได้ระหว่างรอซ่อมเท่านั้น ตรงตามบทเรียน
ที่ [Part 26 Step 259.4](part-026-advanced-ospf.md) เตือนไว้ล่วงหน้าแล้ว

### 569.6 วิธีแก้และ Verify

```
! แก้ที่ต้นเหตุจริง — เปลี่ยน SFP ที่เสื่อมสภาพ (Hardware Fix)
DIST-SW4(config)# interface TenGigabitEthernet1/1/1
DIST-SW4(config-if)# shutdown
! (เปลี่ยน SFP ทางกายภาพ)
DIST-SW4(config-if)# no shutdown
```

```
! Verify หลังเปลี่ยน SFP — CRC Error ไม่เพิ่มขึ้นอีก และ SPF กลับสู่ความถี่ปกติ
DIST-SW4# show interfaces TenGigabitEthernet1/1/1 | include CRC
1802 input errors, 1802 CRC, 0 frame, 0 overrun, 0 ignored
                                          <-- ตัวเลขหยุดนิ่ง (ไม่เพิ่มขึ้นอีกหลังเปลี่ยน SFP)

CORE-SW2# show ip ospf statistics
  SPF calculation executed 188 times, latest at 00:14:22 ago
                                          <-- ไม่มีรอบใหม่เพิ่มเข้ามาอีกเลยตลอด 14 นาทีที่ผ่านมา

CORE-SW2# show processes cpu sorted | include OSPF
 142      812340      184301       4408   1.02%  1.15%  1.98%   0 OSPF-1 Router
                                          <-- CPU กลับสู่ระดับปกติ
```

> **บทเรียนสำคัญที่สุดของ Step นี้**: อย่าไปปรับ `timers throttle spf/lsa` ให้ "หย่อนลง" (Delay
> มากขึ้น) เพื่อ "แก้" อาการ CPU สูงจาก Flap จริง — การทำแบบนั้นแค่ซ่อนอาการและทำให้ Convergence
> ของทั้งระบบช้าลงโดยไม่จำเป็นสำหรับ Event ปกติทั่วไปด้วย ต้องหา Physical Root Cause แล้วแก้ที่
> จุดนั้นเสมอ Throttle Timer ที่ Config ไว้แล้วตาม Part 26 คือค่าที่เหมาะสมสำหรับ Enterprise Core
> อยู่แล้ว ไม่ต้องแก้เพิ่ม

---

## Step 570 — Lab เต็มรูปแบบ: 3 ปัญหาซ้อนกันพร้อมกัน

### 570.1 สถานการณ์

หลัง Maintenance Window เปลี่ยนอุปกรณ์ DIST-SW1 และ DIST-SW2 พร้อมกันในคืนเดียว (RMA อุปกรณ์ทั้ง
คู่จาก Batch เดียวกัน) เช้าวันถัดมามี Ticket แจ้งว่า **"VLAN10 ของ Area 1 ปิงไป VLAN30 ของ Area 2
ไม่ได้เลย"** — โจทย์นี้จะจงใจซ่อนปัญหา **3 จุดที่ไม่เกี่ยวกันโดยตรง** ให้ฝึกไล่ตาม Decision Tree
จาก Step 561 ทีละจุดจนครบ

```
                    Area 1                        Area 0                       Area 2
┌──────────┐                          ┌────────────────────────┐                          ┌──────────┐
│ DIST-SW1 │══[MTU 9216 vs 1500]═════│                          │                          │ DIST-SW3 │
└──────────┘   ❌ ปัญหา 1: EXSTART    │   Po1 .1 ═══════ Po1 .2 │                          └──────────┘
                                      │                          │
                          ┌───────────┴─────┐              ┌────┴──────┐
                          │    CORE-SW1      │              │ CORE-SW2  │
                          │  (ABR Area0/1)   │              │(ABR Area0/2)
                          └────┬─────────────┘              └────┬──────┘
┌──────────┐                    │  ❌ ปัญหา 3: area 1 range        │      ┌──────────┐
│ DIST-SW2 │══[MD5 vs Plaintext]┘  10.10.0.0/16 กว้างเกินไป          └──────│ DIST-SW4 │
└──────────┘   ❌ ปัญหา 2: Auth       กลืน VLAN90 ใหม่                       └──────────┘
   Mismatch
```

### 570.2 ขั้นที่ 1 — เริ่มจาก Neighbor State ตามกฎเหล็กของ Step 561

```
CORE-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:38    10.255.0.2      Port-channel1
1.1.1.11          0   EXSTART/  -     00:00:35    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:31    10.255.20.2     TenGigabitEthernet1/0/2
```

พบทันทีว่า DIST-SW1 (1.1.1.11) ค้างที่ **EXSTART** — ตาม Decision Tree Step 561.2 ต้องสงสัย
**MTU Mismatch หรือ Duplicate RID ก่อน** ส่วน DIST-SW2 (1.1.1.12) ดูเหมือนปกติ (FULL) ในขั้นนี้ —
**ต้องแก้ปัญหานี้ก่อนไปดูปัญหาถัดไป เพราะ Neighbor ต้อง Full ให้ครบก่อนเสมอ**

### 570.3 แก้ปัญหาที่ 1 — MTU Mismatch บน DIST-SW1

```
CORE-SW1# debug ip ospf adj
OSPF-1 ADJ  Te1/0/1: Nbr 1.1.1.11 has smaller interface MTU
%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.11 on TenGigabitEthernet1/0/1 from EXSTART to DOWN, Neighbor Down: Too many retransmissions

DIST-SW1# show interfaces TenGigabitEthernet1/1/1 | include MTU
  MTU 9216 bytes, BW 10000000 Kbit/sec, DLY 10 usec,
```

ยืนยัน Root Cause เดียวกับ Step 563 (MTU ผิดจากการ Restore Config หลัง RMA ไม่ครบตามค่าเดิม)

```
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# mtu 1500

CORE-SW1# show ip ospf neighbor | include 1.1.1.11
1.1.1.11          0   FULL/  -        00:00:39    10.255.10.2     TenGigabitEthernet1/0/1
```

### 570.4 ขั้นที่ 2 — Neighbor Full ครบแล้ว แต่ Ping ยังไม่ผ่าน ไล่ต่อไป LSDB/Route

```
DIST-SW1# ping 10.10.30.1 source vlan10
Success rate is 0 percent (0/5)

DIST-SW1# show ip route ospf | include 10.10.30
                                                     <-- ไม่มี Route ไป VLAN30 (Area 2) เลย
```

Neighbor ทุกตัวตอนนี้ Full หมดแล้ว (เช็คซ้ำ `show ip ospf neighbor` ยืนยันครบทั้ง 3 เส้น) แต่
Route ไป VLAN30 ยังไม่มี — ตาม Decision Tree ต้องข้ามไปเช็ค **LSDB/Filter/Area Range** ตาม Step 565

```
DIST-SW1# show ip ospf database summary | include 10.10
Link ID         ADV Router      Age         Seq#       Checksum
10.10.0.0       1.1.1.1         45          0x80000021  0x8F1A
10.10.0.0       1.1.1.2         52          0x80000019  0x2C4B
                                                     <-- เห็นแค่ 10.10.0.0/16 (Summary เดิมจาก Part 12)
                                                         VLAN30 ไม่ควรหายเพราะ Summary นี้มีมาแต่เดิมอยู่แล้ว
```

ตรงนี้พบว่า `10.10.0.0/16` ยังปรากฏอยู่ปกติ (ไม่ใช่ปัญหา) — ต้องเช็คว่า Type 3 ของ VLAN30 (Area 2)
เดิมทีถูก Summarize เข้าไปในก้อนนี้อยู่แล้วตาม Part 12 Step 120 (ปกติ ไม่ใช่ Bug) แต่ทำไม Ping ยัง
ไม่ผ่าน — ต้องเช็คขั้นถัดไปคือ **Authentication บน Path ที่ Traffic ต้องเดินผ่านจริง**

### 570.5 แก้ปัญหาที่ 2 — Authentication Mismatch บน DIST-SW2 (Path จริงที่ VLAN10 ต้องใช้ออก Area 1)

```
CORE-SW1# show logging | include ERRRCV
%OSPF-4-ERRRCV: Received invalid packet: mismatch authentication type from 10.255.20.2, TenGigabitEthernet1/0/2
```

แม้ `show ip ospf neighbor` ตอนแรกจะแสดง DIST-SW2 เป็น FULL (เพราะจับภาพก่อนที่ Neighbor จะหลุด
จาก Retransmission ของ Authentication ที่ผิด) แต่ Syslog นี้ (และ `show ip ospf neighbor` ซ้ำอีก
รอบ) เผยว่า Neighbor **หลุดๆ ติดๆ อยู่เป็นระยะ** เพราะ DIST-SW2 หลัง RMA ถูก Restore Config ด้วย
Template เก่าที่ยังเป็น **Plaintext Authentication** (ไม่ใช่ MD5 ตาม Baseline ปัจจุบัน):

```
DIST-SW2# show running-config interface TenGigabitEthernet1/1/1 | include authentication
 ip ospf authentication                              <-- Plaintext ไม่ใช่ message-digest!
 ip ospf authentication-key C1sco123
```

```
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# no ip ospf authentication
DIST-SW2(config-if)# no ip ospf authentication-key C1sco123
DIST-SW2(config-if)# ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
DIST-SW2(config-if)# ip ospf authentication message-digest
```

```
CORE-SW1# show ip ospf neighbor | include 1.1.1.12
1.1.1.12          0   FULL/  -        00:00:39    10.255.20.2     TenGigabitEthernet1/0/2
                                                     <-- เสถียรแล้ว ไม่หลุดซ้ำอีก
```

### 570.6 ขั้นที่ 3 — Ping ผ่านบางส่วนแต่ Firewall Team รายงาน ACL ผิดพลาด (ปัญหาที่ 3)

```
DIST-SW1# ping 10.10.30.1 source vlan10
Success rate is 100 percent (5/5)
                                                     <-- ตอนนี้ Ping ผ่านแล้ว! Ticket เดิมปิดได้
```

แต่ระหว่างตรวจสอบ พบ Ticket ที่สองจากทีม Firewall: **"IoT VLAN90 ที่เพิ่งเปิดใหม่บน DIST-SW1
มองไม่เห็น Prefix เฉพาะเจาะจงที่ CORE-SW2 เลย เห็นแต่ก้อน 10.10.0.0/16 รวม"** — นี่คือปัญหาที่ 3
ตรงกับ Root Cause ของ Step 565 เป๊ะ (`area 1 range 10.10.0.0 255.255.0.0` กลืน VLAN90 ที่เพิ่ง
เพิ่มเข้ามาใหม่หลัง RMA):

```
CORE-SW2# show ip route ospf | include 10.10.90
                                                     <-- ไม่มี — ยืนยัน Root Cause เดิมของ Step 565
```

```
! Fix เดียวกับ Step 565.5 — Renumber VLAN90 ออกจากขอบเขต Summary เดิม
DIST-SW1(config)# interface Vlan90
DIST-SW1(config-if)# ip address 10.20.90.1 255.255.255.0
DIST-SW1(config-if)# ip ospf 1 area 1
```

```
CORE-SW2# show ip route ospf | include 10.20.90
O IA    10.20.90.0/24 [110/11] via 10.255.30.2, TenGigabitEthernet1/0/1
                                                     <-- ปรากฏเป็น Prefix เฉพาะเจาะจงแล้ว
```

### 570.7 ตารางสรุปทั้ง 3 ปัญหาและ Verification สุดท้าย

| # | ปัญหา | จุดที่เจอ | Root Cause | คำสั่ง Fix หลัก |
|---|---|---|---|---|
| 1 | Neighbor ค้าง EXSTART | CORE-SW1 ↔ DIST-SW1 | MTU 9216 vs 1500 หลัง RMA | `mtu 1500` |
| 2 | Neighbor Flap ไม่เสถียร | CORE-SW1 ↔ DIST-SW2 | Authentication Type: Plaintext vs MD5 | `ip ospf authentication message-digest` + Key ให้ตรง |
| 3 | Route หายเฉพาะ Prefix ใหม่ | VLAN90 (DIST-SW1) → Area 2 | `area 1 range 10.10.0.0/16` กว้างเกินไป | Renumber VLAN90 ออกนอกขอบเขต Summary |

```
! Verification สุดท้าย — ครบทั้ง 3 จุดพร้อมกัน
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:36    10.255.0.2      Port-channel1
1.1.1.11          0   FULL/  -        00:00:32    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:38    10.255.20.2     TenGigabitEthernet1/0/2

DIST-SW1# ping 10.10.30.1 source vlan10
Success rate is 100 percent (5/5)

CORE-SW2# show ip route ospf | include 10.20.90
O IA    10.20.90.0/24 [110/11] via 10.255.30.2, TenGigabitEthernet1/0/1
```

> **บทเรียนของ Lab สุดท้ายนี้**: ปัญหาจริงในองค์กรมักไม่มาทีละจุด — Ticket เดียวอาจซ่อนหลาย Root
> Cause ที่ไม่เกี่ยวกันเลยไว้พร้อมกัน **วิธีเดียวที่จะไม่หลงทาง** คือไล่ตาม Decision Tree ของ Step
> 561 อย่างมีระบบทุกครั้ง (Neighbor State ก่อนเสมอ → LSDB/Filter → Route → Path) แก้ทีละจุดแล้ว
> Verify ซ้ำก่อนไปจุดถัดไป ไม่ใช่พยายามแก้ทุกอย่างพร้อมกันในครั้งเดียวจนแยกไม่ออกว่า Fix ไหนแก้
> อะไรจริง

---

## แบบฝึกหัดทวนความเข้าใจ Part 57

1. บน Broadcast Segment ที่มี Router 2 ตัว ถ้าตั้ง `ip ospf priority 0` ทั้งสองฝั่งพร้อมกัน
   Neighbor State สุดท้ายที่ค้างอยู่คือ State ใด และทำไมถึงไม่ไปต่อ Full ได้?
2. เพราะเหตุใด MTU Mismatch จึงทำให้ Neighbor ค้างที่ ExStart/Exchange แทนที่จะ Down ไปเลย และ
   `ip ospf mtu-ignore` ต่างจากการแก้ MTU ให้ตรงกันจริงอย่างไร?
3. Area ID Mismatch กับ Authentication Type Mismatch ต่างกันอย่างไรในมุมของ Neighbor Table —
   ทำไม Area ID Mismatch ทำให้ Neighbor ไม่ปรากฏขึ้นมาเลยแม้แต่ State Down?
4. `area range` ที่กว้างเกินไปกับ `area filter-list` ที่กว้างเกินไป ให้ผลลัพธ์คล้ายกัน (Route
   หายไปจาก Area อื่น) แต่ Mechanism ที่ทำให้เกิดผลนั้นต่างกันอย่างไร (ดูจาก `show ip ospf
   database` เทียบกัน)?
5. `auto-cost reference-bandwidth` ที่ไม่ตรงกันระหว่าง Router ทำให้เกิด Suboptimal Routing ได้
   โดยไม่มี Log เตือนใดๆ เลย เพราะเหตุใด และควรใช้คำสั่งใดตรวจสอบเจอได้เร็วที่สุด?

**เฉลย:**

1. ค้างที่ **2WAY/DROTHER** ทั้งสองฝั่งตลอดไป เพราะ Router ที่ตั้ง Priority = 0 ประกาศตัวว่า "ไม่มี
   สิทธิ์เป็น DR หรือ BDR เลย" ตามกฎ Part 11 Step 105.3 — เมื่อไม่มี Router ใดมีสิทธิ์แม้แต่ตัวเดียว
   บน Segment นั้น การเลือกตั้ง DR/BDR จะไม่มีผลลัพธ์ (ไม่มี DR ไม่มี BDR) และ DROTHER คุยกับ
   DROTHER ตัวอื่นได้แค่ 2-Way ตามกฎเดิมเท่านั้น จึงไม่มีวันไป Full
2. OSPF ตรวจสอบ MTU ที่ประกาศมาใน DBD Packet เทียบกับ MTU ของ Interface ตัวเอง — ถ้าไม่ตรงกัน
   (โดย Default ไม่มี `mtu-ignore`) Router จะปฏิเสธ DBD ทันทีและ Retransmit วนซ้ำจนเกิน Threshold
   แล้ว Reset State กลับ ทำให้ค้างวนอยู่แถว ExStart/Exchange/Down สลับกัน ไม่ไป Full — `ip ospf
   mtu-ignore` เป็นการ**ข้ามการตรวจสอบไปเฉยๆ** ทำให้ Adjacency ขึ้นได้แม้ MTU ต่างกันจริง แต่ไม่ได้
   แก้ MTU จริง หากมี LSA/Data ขนาดใหญ่จริงอาจเกิด Fragmentation ที่ Layer ต่ำกว่าแบบเงียบๆ ในขณะ
   ที่การแก้ MTU ให้ตรงกันจริงคือการแก้ที่ต้นเหตุแท้จริงไม่มีความเสี่ยงนี้เลย
3. Area ID เป็นข้อมูลที่บรรจุอยู่ใน **Hello Packet ทุกใบ** — ถ้า Area ID ที่ระบุมาไม่ตรงกับ Area
   ที่ Interface ตนตั้งไว้ Router จะปฏิเสธ Hello Packet ทั้งใบตั้งแต่ต้นทาง **ก่อนสร้าง Neighbor
   Entry ขึ้นมาด้วยซ้ำ** จึงไม่ปรากฏใน Neighbor Table เลยแม้แต่ State Down ส่วน Authentication
   Type Mismatch มักเกิดกับ Neighbor ที่**เคย Full มาก่อนแล้ว** (Rollout ครึ่งเดียว) ทำให้เห็น
   ร่องรอยการหลุดจาก FULL ไป DOWN ใน Log (`%OSPF-5-ADJCHG ... from FULL to DOWN`) ต่างจาก Area
   ID ที่มักไม่เคยมี Adjacency เกิดขึ้นเลยตั้งแต่แรก
4. `area range` ทำงานที่ตัว **ABR ก่อนสร้าง Type 3 LSA** — ถ้ากว้างเกินไป Subnet ที่ตรงกับขอบเขตจะ
   ไม่ถูกสร้างเป็น Type 3 แยกตั้งแต่ต้น (`show ip ospf database summary` จะไม่เห็น Prefix นั้นแยก
   เลย เห็นแต่ก้อน Summary ใหญ่) ส่วน `area filter-list` ทำงาน**หลัง Type 3 LSA ถูกสร้างแล้ว**
   เป็นการกรองไม่ให้ Type 3 LSA ที่มีอยู่แล้วข้าม Area (ถ้าเช็ค LSDB ฝั่งต้นทางก่อนกรองจะยังเห็น
   Type 3 ของ Prefix นั้นอยู่ปกติ แต่ฝั่งปลายทางที่ถูก Filter จะไม่เห็นเลย)
5. เพราะ Reference Bandwidth ไม่ใช่ค่าที่ OSPF ใช้ตรวจสอบความเข้ากันได้ระหว่าง Neighbor (ไม่เหมือน
   Hello/Dead Timer หรือ Area ID/Authentication) — Adjacency จะขึ้น Full ได้ปกติทุกประการแม้ค่านี้
   ต่างกัน เพราะ Cost ของแต่ละ Link ถูกคำนวณและ Advertise โดย Router ต้นทางของ Link นั้นฝ่ายเดียว
   ไม่มีการเทียบค่ากับ Neighbor เลย จึงไม่มี Log เตือนใดๆ เกิดขึ้น — วิธีตรวจสอบเจอเร็วที่สุดคือ
   ไล่คำสั่ง `show ip ospf | include Reference` ทีละ Router ทั้งโดเมนเทียบกัน หรือเทียบ Cost ของ
   Interface ความเร็วเดียวกัน (`show ip ospf interface`) ระหว่าง Router ที่สงสัยว่าค่าตกหล่น

---

## สรุป Part 57

Part นี้พา **OSPF 3-Area Design** ที่วางฐานไว้ตั้งแต่ Part 11/12/26 กลับมาเจาะลึกเป็น Lab
Troubleshooting เข้มข้นตามแนวข้อสอบ **CCNP ENARSI** — เริ่มจาก **Decision Tree Methodology**
ที่ไล่ตามลำดับ Neighbor State → LSDB → Route → Path เสมอ ไม่เดา Config สุ่มๆ จากนั้นฝึกวินิจฉัย
และแก้ **8 Broken Scenario จริง**: DR/BDR Priority 0 ทั้งคู่, MTU Mismatch แบบ Classic Gotcha,
Area ID พิมพ์ผิด, `area range`/`filter-list` กว้างเกินจนกลืน Prefix ที่ควรเห็นแยก, Authentication
Rollout ครึ่งเดียว, `auto-cost reference-bandwidth` ตกหล่นจนเลือก Path ผิด, NSSA Mismatch ตัด
Area ทั้ง Area ขาดจาก Backbone, และ LSA Flooding Storm จาก SFP เสื่อมที่ยืนยันว่า Throttle Timer
เป็นแค่ Mitigation ไม่ใช่การแก้ต้นเหตุ ปิดท้ายด้วย **Lab รวม 3 ปัญหาซ้อนกันในคราวเดียว** ที่ต้อง
ใช้ Methodology จาก Step 561 ไล่แก้ทีละจุดอย่างมีระบบจนครบ ✅

**สิ่งที่ต้องจำที่สุดจาก Part นี้**: OSPF แทบไม่มี "Silent Failure" จริงๆ เลย — ทุกปัญหาทิ้ง
ร่องรอยไว้เสมอ ไม่ว่าจะเป็น Syslog (`%OSPF-4-ERRRCV`), Debug Output (`debug ip ospf adj`), หรือ
ตัวเลขที่ผิดปกติใน `show` command ต่างๆ สิ่งที่ต้องฝึกคือ**อ่านร่องรอยเหล่านั้นให้ไวและแม่นยำ**
ไม่ใช่การจำ Solution แต่ละกรณีแบบท่องจำ

**Part ถัดไป** จะย้ายจาก Link-State (OSPF) ไปสู่ **Path-Vector Protocol** เต็มรูปแบบ — **BGP
Troubleshooting** ที่มี Failure Mode ต่างจาก OSPF โดยสิ้นเชิง (Session ไม่ขึ้นจาก TCP/AS-Number/
Peer-Group Mismatch, Route ไม่ถูกเลือกจาก Path Attribute/Best-Path Algorithm ที่ซับซ้อนกว่า Cost
เดียวของ OSPF มาก) พร้อม Demo Site BGP ที่วางไว้ตั้งแต่ Part 29-30

**ไปต่อ:** [Part 58 — BGP Troubleshooting →](part-058-bgp-troubleshooting.md)
