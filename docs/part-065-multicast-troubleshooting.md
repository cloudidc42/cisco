# Part 65 — Multicast Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 641–650 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 64 — Path Control & PBR Troubleshooting](part-064-path-control-pbr-troubleshooting.md)
> ที่เจาะ Troubleshoot PBR/Path Control บน WAN Edge Part นี้วกกลับมาที่หัวข้อ **Multicast** ที่
> [Part 41](part-041-multicast-fundamentals.md) (IGMP, PIM-DM/SM, RPF, `show ip mroute`) และ
> [Part 42](part-042-multicast-advanced.md) (Auto-RP, BSR, **Anycast RP + MSDP**, SSM) ปูพื้นฐาน
> และสร้าง Production Design ไว้ครบแล้ว — STREAM-SRV (`10.10.30.30`) ส่ง All-Hands Video ไปที่
> `239.1.1.1` ผ่าน **Anycast RP (`1.1.1.100` บน CORE-SW1/CORE-SW2) + MSDP**, MARKET-SRV
> (`10.10.30.40`) ส่ง Market-Data Feed ผ่าน **SSM** (`232.50.1.1`) โดยไม่มี RP เกี่ยวข้องเลย และ
> Access Switch ทุกตัวเปิด **IGMP Snooping** ไว้ป้องกัน Flood — Design นี้ "ถูกต้องตามทฤษฎี" ทุก
> ประการ แต่ Multicast เป็นระบบที่มี **หลาย Layer ซ้อนกัน** (IGMP → PIM → RPF → mroute → Forwarding
> จริง) ทำให้จุดพังมีได้หลากหลายมาก และส่วนใหญ่ **ไม่มี Syslog เตือนตรงๆ เหมือน OSPF/EIGRP เลย**
> ([Part 57](part-057-ospf-troubleshooting.md), [Part 56](part-056-eigrp-troubleshooting.md)) —
> Part นี้จะจงใจทำให้ Multicast Design ที่สมบูรณ์นี้พังทีละจุดตามแนวข้อสอบ **CCNP ENARSI
> (300-410)** แล้วฝึกไล่ตาม Layer จนเจอ Root Cause จริง ปิดท้ายด้วย Lab รวมที่ปัญหา 2 จุดเกิดพร้อม
> กันหลัง Network Redesign

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 641 | Multicast Troubleshooting Methodology — ไล่ทีละ Layer จาก IGMP ถึง Forwarding จริง |
| 642 | Scenario 1 — Client Join แล้วไม่ได้รับ Stream เพราะ IGMP Snooping ไม่เรียนรู้ Router Port |
| 643 | Scenario 2 — RPF Failure จาก Static Route ทดสอบที่ลืมถอด (Asymmetric Routing) |
| 644 | Scenario 3 — RP Mapping ไม่ตรงกันหลัง Migrate ไป Anycast RP ไม่ครบทุก Router |
| 645 | Scenario 4 — MSDP Peer ใช้ Anycast Address ผิดพลาดแทน Loopback0 Unique |
| 646 | Scenario 5 — ลืม `ip pim sparse-mode` บน SVI ใหม่ของ VLAN Receiver ที่เพิ่งเพิ่ม |
| 647 | Scenario 6 — SSM ใช้งานไม่ได้เพราะ IGMPv3 ไม่ถึงปลายทาง หรือ `ip pim ssm range` ไม่ครอบคลุม |
| 648 | Scenario 7 — Storm-Control เข้าใจผิดว่า Video Stream ถูกต้องคือ "Storm" |
| 649 | Scenario 8 — Auto-RP/BSR ไม่ถูก Flood ข้าม Transit Interface หลังปรับ ACL ที่ไม่เกี่ยวข้อง |
| 650 | Lab เต็มรูปแบบ: RPF Failure + MSDP Misconfiguration เกิดพร้อมกันหลัง Network Redesign |

---

## Step 641 — Multicast Troubleshooting Methodology

### 641.1 ทำไม Multicast ต้องมี Methodology ที่ต่างจาก Unicast Routing

Multicast ไม่เหมือน OSPF/EIGRP ที่ Neighbor ไม่ขึ้นแล้ว **หยุดเดินหน้าไปเลยชัดเจน** — Multicast
มี Layer ที่ทำงาน**อิสระจากกันแต่ต่อกันเป็นสาย (Chain)** ตั้งแต่ Host จนถึง Router ปลายทาง ถ้า Layer
กลางๆ พังแบบ "เงียบ" (ไม่มี Syslog, ไม่มี Neighbor Down) วิศวกรที่ไม่มี Methodology จะเสียเวลาไล่ดู
Layer ที่ถูกต้องอยู่แล้วซ้ำไปซ้ำมา — Part นี้บังคับให้ไล่ตามลำดับที่ตายตัวเสมอ

```
Receiver ไม่เห็น Video เลย
        │
        ▼
[1] IGMP Membership   show ip igmp snooping groups / show ip igmp groups
        │ ไม่เห็น Group ──► Snooping ไม่เรียนรู้ Router Port (642), IGMP Version ไม่ตรง (647),
        │                    ลืม pim sparse-mode บน SVI (646)
        ▼ เห็นแล้ว
[2] PIM Neighbor      show ip pim neighbor / show ip pim interface
        │ ไม่ขึ้น ──► L1/L2 ไม่ขึ้น, ip pim sparse-mode หายไปบาง Interface (646)
        ▼ ขึ้นครบ
[3] RP Mapping        show ip pim rp mapping (เทียบทุก Router)
        │ ไม่ตรงกัน ──► RP Mapping Inconsistency (644), Auto-RP/BSR ไม่ถูก Flood (649)
        ▼ ตรงกัน
[4] RPF Check         show ip rpf <source> / show ip mroute count
        │ RPF Fail ──► Asymmetric Routing, Static Route ผิด (643, 650)
        ▼ Pass
[5] mroute/OIL State  show ip mroute <group>
        │ OIL ไม่มี Interface ที่ต้องการ ──► MSDP SA ไม่ Sync (645, 650)
        ▼ มีครบ
[6] Forwarding จริง   show interfaces counters, show storm-control (648)
```

### 641.2 ตาราง Layer → คำสั่ง Verify → Step ที่สอนใน Part นี้

| Layer | คำถามหลัก | คำสั่งตรวจสอบ | Step ที่เกี่ยวข้อง |
|---|---|---|---|
| 1. IGMP Membership | Host Join ถูกบันทึกที่ Switch/Router หรือไม่ | `show ip igmp snooping groups`, `show ip igmp groups`, `show ip igmp snooping mrouter` | 642, 646, 647 |
| 2. PIM Neighbor | Router เห็น PIM Neighbor ครบทุกทิศทางหรือไม่ | `show ip pim neighbor`, `show ip pim interface` | 646 |
| 3. RP Mapping | ทุก Router รู้จัก RP ตัวเดียวกันหรือไม่ | `show ip pim rp mapping`, `show ip pim rp` | 644, 649 |
| 4. RPF Check | Path กลับไป Source ตรงกับทาง Data เข้าจริงหรือไม่ | `show ip rpf <source>`, `show ip mroute count` | 643, 650 |
| 5. mroute/OIL State | `(S,G)`/`(*,G)` มี Interface ที่ต้องการใน OIL ครบหรือไม่ | `show ip mroute <group>` | 644, 645, 650 |
| 6. MSDP (ถ้ามี Anycast RP) | RP ทั้งสองตัวรู้จัก Source เดียวกันหรือไม่ | `show ip msdp summary`, `show ip msdp sa-cache` | 645, 650 |
| 7. Forwarding จริง | Traffic ถูก Drop ที่ Interface/Storm-control หรือไม่ | `show interfaces`, `show storm-control` | 648 |

> **กฎเหล็ก**: ไล่จาก Layer 1 ขึ้นไปเสมอ **ห้ามข้ามขั้น** — ถ้า Layer 1 (IGMP) ไม่เห็น Group เลย
> การไปดู `show ip mroute` (Layer 5) จะเสียเวลาเปล่า เพราะ Entry อาจจะไม่มีเลยด้วยซ้ำถ้า IGMP ยังไม่
> ทำงาน ในทางกลับกัน ถ้า Layer 1-3 ผ่านหมดแต่ Traffic ยังไม่ไหล ต้องข้ามไปเช็ค RPF (Layer 4) และ
> MSDP (Layer 6) โดยตรง — Multicast ต่างจาก OSPF (Part 57) ตรงที่ **RPF Failure และ MSDP SA ไม่
> Sync แทบไม่มี Syslog เตือนเลย** ต้องอ่านจาก Counter ใน `show ip mroute count` และ
> `show ip msdp summary` เชิงรุกเสมอ ไม่ใช่รอ Log แจ้ง

### 641.3 คำเตือนซ้ำจาก Part 41: `debug ip mpacket`

เช่นเดียวกับที่เตือนไว้ใน [Part 41 Step 409](part-041-multicast-fundamentals.md#step-409--verification--troubleshooting)
`debug ip mpacket` แสดง Log ระดับ Per-Packet ซึ่งเสี่ยงทำ CPU พุ่งจน Console ค้างได้บน Link ที่มี
Traffic Rate สูงอย่าง Video Stream ของ STREAM-SRV — ทุก Scenario ใน Part นี้จะใช้ `show`
Command เป็นหลักเสมอ และใช้ `debug` เท่าที่จำเป็นจริงๆ พร้อม Access-list จำกัดขอบเขตก่อนเปิดทุกครั้ง

---

## Step 642 — Scenario 1: Client Join แล้วไม่ได้รับ Stream เพราะ IGMP Snooping ไม่เรียนรู้ Router Port

### 642.1 บริบท

ทีมงานเพิ่มผู้ชม All-Hands คนที่สองบน **ACCESS-SW1** (`GigabitEthernet0/9`, VLAN 10 SALES) โดย
PC ตัวใหม่เปิด Media Player ส่ง IGMP Membership Report ไป Join `239.1.1.1` ตามปกติ — แต่ทีม
Network Security ที่กำลังทำ Hardening Baseline ใหม่ตาม [Part 21](part-021-switch-security.md)
คัดลอก Template จาก Switch รุ่นเก่าที่ยังใช้ **CGMP (Cisco Group Management Protocol)** ซึ่งเป็น
กลไก Legacy ก่อนยุค PIM-based Snooping มาใช้กับ ACCESS-SW1 โดยไม่ได้ตรวจสอบ Compatibility
ก่อน

### 642.2 Config ที่ผิด

```
! ACCESS-SW1 — Copy-paste Template เก่าที่สั่งให้ Snooping เรียนรู้ Router Port ผ่าน CGMP
! (Legacy Mechanism ที่ปกติไม่มี Router ตัวใดในหลักสูตรนี้ส่ง CGMP Packet ออกมาเลย)
ACCESS-SW1(config)# ip igmp snooping vlan 10 mrouter learn cgmp
```

### 642.3 อาการที่พบ

```
ACCESS-SW1# show ip igmp snooping groups vlan 10
Vlan  Group            Type    Version  Port List
----  ----------------  ------  -------  -------------
10    239.1.1.1          igmp    v2       Gi0/8, Gi0/9

ACCESS-SW1# show ip igmp snooping mrouter vlan 10
Vlan     ports
----     -----
                                    <-- ไม่มีบรรทัดผลลัพธ์เลย! ไม่มี Router Port ที่เรียนรู้ได้
```

Switch **ยังเห็น Host Join ปกติ** (Gi0/8 ของ PC-SALES1 เดิม และ Gi0/9 ของ PC ใหม่ ปรากฏใน Port
List) แต่ **ไม่มี Router Port (mrouter) เลยแม้แต่ Port เดียว** — ผลคือ Report ของ Host ทั้งสองไม่ถูก
Forward ขึ้นไปยัง DIST-SW1 เลย เพราะ IGMP Snooping จะส่ง Report ต่อไปทาง mrouter Port เท่านั้น

```
! ยืนยันที่ DIST-SW1 — ไม่เคยเห็น Group นี้จาก Interface Vlan10 เลย
DIST-SW1# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface        Uptime    Expires   Last Reporter
                                    <-- ไม่มี 239.1.1.1 ปรากฏเลย
```

### 642.4 การวินิจฉัย

```
ACCESS-SW1# show running-config | include mrouter
ip igmp snooping vlan 10 mrouter learn cgmp
```

ค่า Default ของ IGMP Snooping mrouter Learning คือ **`pim-dvmrp`** — โหมดนี้ทำให้ Switch "แอบดู"
**PIM Hello Packet** (`224.0.0.13`) หรือ **IGMP Query** ที่ DIST-SW1 ส่งออกมาบน Trunk แล้วสรุปเอง
ว่า "Port ที่ Hello/Query เข้ามาต้องเป็นทางไปหา Router" — แต่โหมด **`cgmp`** รอ Packet แบบ CGMP
โดยเฉพาะ (Multicast MAC พิเศษที่ DIST-SW1 **ไม่เคยส่งออกมาเลย** เพราะ Cisco IOS สมัยใหม่ไม่ใช้
CGMP อีกต่อไป) — Switch จึงไม่มีวันเรียนรู้ Router Port ได้เลยตราบใดที่ยังตั้งค่าผิดโหมดอยู่

### 642.5 Root Cause

**IGMP Snooping mrouter Learning Method ถูกตั้งเป็น `cgmp` โดยไม่ตั้งใจ** (Copy-paste จาก
Template Switch รุ่นเก่า) ทำให้ Switch ไม่สามารถเรียนรู้ Router-facing Port แบบอัตโนมัติผ่าน
PIM Hello/IGMP Query ที่ DIST-SW1 ส่งจริงได้ — Membership Report จาก Host ทุกตัวใน VLAN 10 จึง
ไม่ถูก Forward ไปถึง Router เลย แม้ Host จะ Join ถูกต้องทุกประการก็ตาม

> **ทางเลือกที่สองที่ข้อสอบชอบถามคู่กัน**: ถ้า Segment นั้น**ไม่มี PIM Router ทำ Query เลยจริงๆ**
> (เช่น L2-only Switch คั่นกลางไม่มี L3 Gateway ทำ IGMP Querier) Switch จะไม่มีทาง Detect
> Router Port ได้เลยไม่ว่าโหมด Learning จะถูกหรือผิด — ทางแก้ในกรณีนั้นคือเปิด
> **`ip igmp snooping querier`** บน Switch เองเพื่อให้ Switch สวมบทบาท Querier ชั่วคราว (ใช้เฉพาะ
> เมื่อไม่มี L3 Device ทำหน้าที่นี้อยู่แล้วจริงๆ — ในหลักสูตรนี้ DIST-SW1 ทำ Querier อยู่แล้วเสมอเพราะ
> เปิด `ip pim sparse-mode`)

### 642.6 วิธีแก้และ Verify

```
ACCESS-SW1(config)# no ip igmp snooping vlan 10 mrouter learn cgmp
ACCESS-SW1(config)# ip igmp snooping vlan 10 mrouter learn pim-dvmrp
```

```
ACCESS-SW1# show ip igmp snooping mrouter vlan 10
Vlan     ports
----     -----
10       Gi1/0/24                          <-- เรียนรู้ Trunk ไปยัง DIST-SW1 ได้แล้ว

DIST-SW1# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface        Uptime    Expires   Last Reporter
239.1.1.1         Vlan10           00:00:14  00:02:46  10.10.10.12
```

Report ของ Host ใหม่ (`10.10.10.12`) ไปถึง DIST-SW1 ทันทีหลังแก้ — `show ip mroute 239.1.1.1`
บน DIST-SW1 กลับมามี Interface Vlan10 อยู่ใน OIL ตามเดิม และ PC ทั้งสองเครื่องบน ACCESS-SW1
รับ Video ได้พร้อมกัน

---

## Step 643 — Scenario 2: RPF Failure จาก Static Route ทดสอบที่ลืมถอด

### 643.1 บริบท

ระหว่างทำ Lab Path Control/PBR ใน [Part 64](part-064-path-control-pbr-troubleshooting.md)
วิศวกรทดสอบ Static Route ชั่วคราวบน **CORE-SW1** เพื่อบังคับ Traffic บางส่วนออกทาง
**WAN-EDGE-1** สำหรับทดสอบ Failover Scenario แล้ว**ลืมถอด Static Route ตัวนั้นออก** หลังปิด Lab
— Static Route ตัวนี้ดันมี Prefix ที่ครอบคลุม `10.10.30.0/24` ซึ่งเป็น Subnet เดียวกับที่
**STREAM-SRV** (`10.10.30.30`) อยู่จริงโดยไม่ได้ตั้งใจ

### 643.2 Config ที่ผิด (Leftover)

```
! CORE-SW1 — Static Route ทดสอบจาก Lab Part 64 ที่ไม่ได้ถอดออก
CORE-SW1(config)# ip route 10.10.30.0 255.255.255.0 10.10.254.1
! 10.10.254.1 = WAN-EDGE-1 (ตาม 00-ip-address-plan.md) — ไม่เกี่ยวข้องกับ Path ไป STREAM-SRV เลย
```

### 643.3 อาการที่พบ

```
! Receiver ที่เพิ่ง Join ใหม่ (หรือ Session ที่ต้องสร้าง (S,G) State ใหม่) ไม่ได้รับ Video เลย
! แต่ Session เดิมที่ยังมี (S,G) Cache ค้างอยู่ก่อนหน้าอาจยังดูได้ปกติชั่วคราว (Cache ยังไม่ Expire)

CORE-SW1# show ip mroute count
IP Multicast Statistics
3 routes using 2094 bytes of memory
2 groups, 0.50 average sources per group
Forwarding Counts: Pkt Count/Pkts per second/Avg Pkt Size/Kilobits per second
Other counts: Total/RPF failed/Other drops(OIF-null, rate-limit etc)

Group: 239.1.1.1, Source count: 1, Group pkt count: 0
  Source: 10.10.30.30/32, Forwarding Counts: 0/0/0/0, Other counts: 15420/15420/0
```

คอลัมน์ **`Other counts: 15420/15420/0`** คือหลักฐานชัดเจนที่สุด — **Packet ทั้งหมด 15420 ตัวที่
เข้ามาตรง RPF Failed ทั้งหมด (0 ตัวถูก Forward เลย)** — จุดสำคัญที่ต้องรู้: **RPF Failure แบบนี้ไม่มี
Syslog เตือนเลยโดย Default** ต่างจาก OSPF Area ID Mismatch ([Part 57 Step 564](part-057-ospf-troubleshooting.md))
ที่มี `%OSPF-4-ERRRCV` ชัดเจน — ต้องไล่เช็ค Counter ด้วยตัวเองเสมอ

### 643.4 การวินิจฉัย

```
CORE-SW1# show ip rpf 10.10.30.30
RPF information for ? (10.10.30.30)
  RPF interface: TenGigabitEthernet1/0/5
  RPF neighbor: ? (10.10.254.1)
  RPF route/mask: 10.10.30.0/24
  RPF type: Unicast (static)                    <-- มาจาก Static Route ไม่ใช่ OSPF!
  Doing distance-preferred lookups across tables
```

**`RPF type: Unicast (static)`** คือจุดที่ต้องสังเกตทันที — เทียบกับ `show ip route`:

```
CORE-SW1# show ip route 10.10.30.30
Routing entry for 10.10.30.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.10.254.1
      Route metric is 0, traffic share count is 1

CORE-SW1# show ip route ospf | include 10.10.30
                                    <-- OSPF เรียนรู้ Prefix นี้จริง แต่ไม่ถูกใช้เพราะ AD สูงกว่า
```

Static Route มี **Administrative Distance = 1** ซึ่งชนะ OSPF (**AD 110**) เสมอตามกฎ AD มาตรฐาน
— RPF Check จึงถูกบังคับให้ชี้ไปทาง `TenGigabitEthernet1/0/5` (ทางไป WAN-EDGE-1) ทั้งที่ Video
Data ทางจริงส่งเข้ามาทาง **`Port-channel1`** (จาก CORE-SW2 ที่ต่อกับ DIST-SW3 ซึ่งมี STREAM-SRV
อยู่จริง)

### 643.5 Root Cause

**Static Route ทดสอบจาก Path Control Lab ที่ไม่ได้ถอดออก มี Prefix ครอบคลุม Source Subnet ของ
STREAM-SRV โดยไม่ได้ตั้งใจ** — เพราะ Static Route ชนะ OSPF ด้วย AD ที่ต่ำกว่าเสมอ RPF Check จึง
คำนวณ Incoming Interface ผิดไปเป็นทางที่ไม่มีวันมี Multicast Data จริงไหลผ่าน (WAN-facing
Interface) ทำให้ Data ที่มาจริงทาง Port-channel1 ถูก Drop ทันทีตามกลไก RPF ที่สอนใน
[Part 41 Step 408](part-041-multicast-fundamentals.md#step-408--multicast-forwarding-และ-rpf-check-sg-และ-g)
— **นี่คือ RPF Failure จาก Asymmetric Routing แบบ Classic**: Unicast RIB (ที่ RPF อ้างอิง) กับ
ทางเดินจริงของ Multicast Data ไม่ตรงกัน

### 643.6 วิธีแก้และ Verify

```
CORE-SW1(config)# no ip route 10.10.30.0 255.255.255.0 10.10.254.1
```

```
CORE-SW1# show ip rpf 10.10.30.30
RPF information for ? (10.10.30.30)
  RPF interface: Port-channel1
  RPF neighbor: ? (10.255.0.2)
  RPF route/mask: 10.10.30.0/24
  RPF type: Unicast (ospf 1)                    <-- กลับมาใช้ OSPF แล้ว

CORE-SW1# show ip mroute count
Group: 239.1.1.1, Source count: 1, Group pkt count: 3084
  Source: 10.10.30.30/32, Forwarding Counts: 3084/25/1400/280, Other counts: 3084/0/0
```

`RPF failed` กลับเป็น 0 และ Forwarding Counts เริ่มขยับตามจริง — ทุก Receiver กลับมารับ Video
ได้ปกติ

> **หมายเหตุสำหรับกรณีที่ Asymmetric Routing เป็นสิ่งที่ตั้งใจ (ไม่ใช่ Bug)**: ถ้า Design จริงจำเป็น
> ต้องให้ Unicast Path กับ Multicast Path ต่างกัน (เช่น Traffic Engineering ที่ตั้งใจแยกทาง Unicast/
> Multicast ออกจากกัน) Cisco มีคำสั่ง **`ip mroute <source> <mask> <rpf-address-or-interface>`**
> ที่สร้าง "Static Route เฉพาะสำหรับ RPF Check" แยกออกจาก Unicast RIB โดยสิ้นเชิง — ใช้เมื่อ
> ต้องการบังคับ RPF Interface ให้ตรงกับทางที่ Multicast Data มาจริง โดยไม่ไปยุ่งกับ Unicast Routing
> Table เลย (จะใช้เทคนิคนี้จริงใน Lab รวม Step 650)

---

## Step 644 — Scenario 3: RP Mapping ไม่ตรงกันหลัง Migrate ไป Anycast RP ไม่ครบทุก Router

### 644.1 บริบท

ตาม [Part 42 Step 420](part-042-multicast-advanced.md#step-420--lab-เต็มรูปแบบ-anycast-rp--msdp-แทน-static-rp-พร้อมเปิด-ssm)
ทีมงาน Migrate จาก Static RP (`1.1.1.1`) ไปเป็น **Anycast RP (`1.1.1.100`)** โดยต้องรัน
`no ip pim rp-address 1.1.1.1` + `ip pim rp-address 1.1.1.100 10` บนทุก Router ที่ไม่ใช่ Core
(DIST-SW1, DIST-SW3, DIST-SW4) — แต่ Rollout ครั้งนี้ทำผ่าน Change Window ที่มีเวลาจำกัด และ
**DIST-SW4 ถูกข้ามไปโดยไม่ตั้งใจ** (อยู่ในรายการ Rollout แต่ Script Batch หยุดกลางทางเพราะ Session
Timeout ก่อนถึงตัวสุดท้าย)

### 644.2 Config ที่ผิด (ค้างจาก Part 41)

```
! DIST-SW4 — ยังเหลือ Static RP ตัวเก่าอยู่ ไม่ได้ Migrate ตาม Rollout จริง
DIST-SW4# show running-config | include rp-address
ip pim rp-address 1.1.1.1                        <-- ควรเป็น 1.1.1.100 10 ตาม Part 42
```

### 644.3 อาการที่พบ

PC-WIFI1 (`10.10.40.51`, ต่อ DIST-SW4 ผ่าน ACCESS-SW4) เคยดู Video ได้ปกติก่อน Migrate แต่
**หลัง Migrate กลับดูไม่ได้เลย** ทั้งที่ Config อื่นทุกจุด (IGMP Snooping, PIM sparse-mode) ไม่ได้ถูก
แก้ไข

```
DIST-SW1# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via static

DIST-SW3# show ip pim rp mapping
Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via static

DIST-SW4# show ip pim rp mapping
Group(s) 239.0.0.0/8
  RP 1.1.1.1 (?), v2v1                            <-- ไม่ตรงกับ DIST-SW1/DIST-SW3!
    Info source: 1.1.1.1 (?), via static
```

### 644.4 การวินิจฉัย

DIST-SW1 และ DIST-SW3 เห็น RP เป็น `1.1.1.100` (Anycast) ตรงกัน แต่ **DIST-SW4 ตัวเดียวยังเห็น
RP เป็น `1.1.1.1`** ซึ่งเป็น Loopback0 เดิมของ CORE-SW1 ที่ **ไม่ได้ทำหน้าที่ RP อีกต่อไปแล้ว**
(ถูก `no ip pim rp-address 1.1.1.1` ไปแล้วตาม Part 42 Step 420) — เมื่อ DIST-SW4 ส่ง PIM Join
ไปหา `1.1.1.1` (ยังคง Ping ได้ปกติเพราะ Loopback0 ยังมีอยู่จริง แค่ไม่ใช่ RP แล้ว) **CORE-SW1 จะไม่รับ
Join นี้ไปทำอะไรต่อเลย** เพราะตัวมันเองไม่คิดว่าตัวเองเป็น RP สำหรับ Group นี้แล้ว — Join จึงเงียบหาย
ไปโดยไม่มี Error ใดๆ

```
CORE-SW1# show ip pim rp
Group: 239.1.1.1, RP: 1.1.1.100, uptime 01:20:33, expires never
                                    <-- CORE-SW1 เองก็ยืนยันว่า RP คือ 1.1.1.100 ไม่ใช่ 1.1.1.1 แล้ว
```

### 644.5 Root Cause

**RP Mapping Inconsistency จากการ Migrate ไม่ครบทุก Router** — DIST-SW4 ยังมี Static RP Config
เดิม (`ip pim rp-address 1.1.1.1`) ค้างอยู่หลังจากที่ Rollout ไป Anycast RP ทำไม่ครบ ทำให้ DIST-SW4
กลายเป็น "เกาะ" ที่แยกออกจากความเข้าใจร่วมของ Domain — ตรงกับคำเตือนที่ให้ไว้แล้วใน
[Part 42 Step 419](part-042-multicast-advanced.md#step-419--verification--troubleshooting-ขั้นสูง)
("Router บางตัวเห็น RP เป็นตัวเก่า Static บางตัวเห็น RP ใหม่ Dynamic — Migrate ไม่ครบทุก Router")

### 644.6 วิธีแก้และ Verify

```
DIST-SW4(config)# no ip pim rp-address 1.1.1.1
DIST-SW4(config)# ip pim rp-address 1.1.1.100 10
```

```
DIST-SW4# show ip pim rp mapping
Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via static

DIST-SW4# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:00:09/00:02:59, RP 1.1.1.100, flags: SJC
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr 10.255.40.1
  Outgoing interface list:
    Vlan40, Forward/Sparse, 00:00:09/00:02:59
```

RP Mapping ตรงกันทั้ง Domain แล้ว — PC-WIFI1 กลับมารับ Video ได้ทันที **ข้อคิดสำคัญ**: ให้ตรวจ
`show ip pim rp mapping` เทียบกันทุก Router **ทุกครั้ง**หลัง Rollout RP ไม่ว่าจะเป็น Static, Auto-RP,
BSR หรือ Anycast RP — Inconsistency แบบนี้เป็นสาเหตุ Troubleshoot ที่พบบ่อยที่สุดของ PIM-SM
ตามที่ Part 41 Step 407 เตือนไว้แล้วตั้งแต่ต้น

---

## Step 645 — Scenario 4: MSDP Peer ใช้ Anycast Address ผิดพลาดแทน Loopback0 Unique

### 645.1 บริบท

[Part 42 Step 415](part-042-multicast-advanced.md#step-415--anycast-rp-rfc-4610) เตือนไว้อย่าง
ชัดเจนว่า **MSDP Peer ต้องใช้ Loopback0 (Unique Address ของแต่ละ Router) ไม่ใช่ Anycast Address
(`1.1.1.100`)** — วิศวกรใหม่ที่เข้ามาดูแล MSDP ต่อจากทีมเดิมไม่ทันสังเกตคำเตือนนี้ และแก้ Config
MSDP บน CORE-SW1/CORE-SW2 ระหว่างทำ Cleanup Loopback Numbering โดยเปลี่ยน `connect-source`
ไปเป็น Loopback1 (Anycast) แทน Loopback0 โดยเข้าใจผิดว่า "RP ควร Peer กันด้วย RP Address"

### 645.2 Config ที่ผิด

```
! CORE-SW1 — แก้ผิด ใช้ Anycast Address (1.1.1.100) เป็นทั้ง Source และ Peer
CORE-SW1(config)# no ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW1(config)# ip msdp peer 1.1.1.100 connect-source Loopback1

! CORE-SW2 — แก้แบบเดียวกัน (ผิดคู่กัน)
CORE-SW2(config)# no ip msdp peer 1.1.1.1 connect-source Loopback0
CORE-SW2(config)# ip msdp peer 1.1.1.100 connect-source Loopback1
```

### 645.3 อาการที่พบ

```
CORE-SW1# show ip msdp summary
MSDP Peer Status Summary
Peer Address    AS    State       Uptime/   Reset  SA     Peer Name
                                  Downtime  Count   Count
1.1.1.100        -     Connecting  00:12:44  8       0      CORE-SW2-AnycastRP-Peer
```

**`State: Connecting`** ค้างอยู่ตลอด ไม่มีทางขึ้น `Up` เลย และ **`SA Count: 0`** ตลอดเวลา — ผลกระทบ
จริง: Receiver ที่อยู่ใกล้ CORE-SW1 (RP #1) ไม่เห็น Source ที่ Register เข้า CORE-SW2 (RP #2) เลย
และในทางกลับกัน

### 645.4 การวินิจฉัย

```
CORE-SW1# show ip msdp peer 1.1.1.100
MSDP Peer 1.1.1.100 (?), AS -
  Connection status:
    State: Connecting, Resets: 8, Connection source: Loopback1 (1.1.1.100)
  ...
  Local host: 1.1.1.100                              <-- Local และ Remote เป็น Address เดียวกัน!
  Remote host: 1.1.1.100
```

**`Local host` และ `Remote host` เป็น IP Address เดียวกันเป๊ะ (`1.1.1.100`)** คือหลักฐานที่ชี้ตรงถึง
ต้นเหตุ — เพราะ `1.1.1.100` เป็น **Anycast Address ที่มีอยู่จริงบนทั้ง CORE-SW1 และ CORE-SW2
พร้อมกัน** เมื่อ Router สั่ง "เปิด TCP Session (Port 639) จาก Address ของตัวเอง (`1.1.1.100`) ไปยัง
Address ปลายทาง (`1.1.1.100`)" มันไม่มีทางรู้ได้เลยว่ากำลังพยายามคุยกับ**ตัวเอง**หรือ**อีกฝั่ง**
เพราะ Address เดียวกันมีอยู่ทั้งสองที่ — TCP Session จึงไม่มีวันสร้างสำเร็จตามที่ตั้งใจ

### 645.5 Root Cause

**MSDP `peer` Address และ `connect-source` ถูกตั้งเป็น Anycast Address (`1.1.1.100`) แทน
Loopback0 Unique Address ของแต่ละ Router** — ตรงกับคำเตือนที่ [Part 42 Step 415](part-042-multicast-advanced.md#step-415--anycast-rp-rfc-4610)
ระบุไว้ว่า "ถ้าใช้ Anycast Address เป็น MSDP Source Address ทั้งคู่จะพยายาม Peer กับตัวเอง" — MSDP
จำเป็นต้องมี Address ที่**เจาะจงแยกแต่ละ Router จริง**เพื่อระบุตัวตนของ TCP Endpoint ทั้งสองฝั่งให้
ต่างกันเสมอ

### 645.6 วิธีแก้และ Verify

```
CORE-SW1(config)# no ip msdp peer 1.1.1.100 connect-source Loopback1
CORE-SW1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW1(config)# ip msdp description 1.1.1.2 CORE-SW2-AnycastRP-Peer

CORE-SW2(config)# no ip msdp peer 1.1.1.100 connect-source Loopback1
CORE-SW2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
CORE-SW2(config)# ip msdp description 1.1.1.1 CORE-SW1-AnycastRP-Peer
```

```
CORE-SW1# show ip msdp summary
Peer Address    AS    State   Uptime/   Reset  SA     Peer Name
                              Downtime  Count   Count
1.1.1.2          -     Up      00:00:22  9       1      CORE-SW2-AnycastRP-Peer

CORE-SW2# show ip msdp sa-cache
MSDP Source-Active Cache - 1 entries
(10.10.30.30, 239.1.1.1), RP 1.1.1.1, MBGP/AS -, 00:00:18/00:05:42, Peer 1.1.1.1
```

`State: Up` และ SA Cache กลับมาแสดงผลถูกต้อง — Receiver ทั้งสองฝั่งของ Anycast RP กลับมาเห็น
Source เดียวกันครบตามที่ Part 42 ออกแบบไว้

---

## Step 646 — Scenario 5: ลืม `ip pim sparse-mode` บน SVI ใหม่ของ VLAN Receiver ที่เพิ่งเพิ่ม

### 646.1 บริบท

ฝ่าย Facilities ติดตั้ง **Digital Signage** (จอแสดงผลถ่ายทอด All-Hands) ในโซนขายที่ดูแลโดย
**ACCESS-SW2** (Uplink ไปยัง **DIST-SW2** — ตาม `00-ip-address-plan.md` Access-Layer Mapping)
ทีมงานสร้าง **VLAN 50 (`DIGITAL-SIGNAGE`, `10.10.50.0/24`)** ขึ้นใหม่ เนื่องจาก **DIST-SW2 ไม่เคย
ถูกดึงเข้า PIM-SM Domain มาก่อนเลย** (Part 41/42 Config เฉพาะ DIST-SW1, DIST-SW3, DIST-SW4)
ทีมงานจึง Onboard DIST-SW2 เข้า Multicast Domain แบบเต็มรูปแบบ: เปิด `ip multicast-routing`,
`ip pim rp-address 1.1.1.100 10`, และ `ip pim sparse-mode` บน Uplink Interface — **แต่ลืมเปิด
`ip pim sparse-mode` บน SVI ของ VLAN 50 เอง** ซึ่งเป็น Interface ที่ Signage เชื่อมต่ออยู่จริง

### 646.2 Config ที่ผิด (ขาดหายไป)

```
! DIST-SW2 — Onboard เข้า PIM Domain ครบเฉพาะ Global + Uplink แต่ลืม SVI ใหม่
DIST-SW2(config)# ip multicast-routing distributed
DIST-SW2(config)# ip pim rp-address 1.1.1.100 10
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# ip pim sparse-mode
DIST-SW2(config-if)# exit
!
DIST-SW2(config)# interface Vlan50
DIST-SW2(config-if)# ip address 10.10.50.1 255.255.255.0
DIST-SW2(config-if)# ip igmp version 2
! <-- ลืม "ip pim sparse-mode" ตรงนี้!
```

### 646.3 อาการที่พบ

```
DIST-SW2# show ip pim interface
Address          Interface                Ver/  Nbr    Query  DR     DR
                                           Mode  Count  Intvl  Prior  
10.255.20.2       TenGigabitEthernet1/1/1  v2/S  1      30     1      10.255.20.1
                                    <-- Vlan50 ไม่ปรากฏในรายการนี้เลย!

DIST-SW2# show ip igmp groups
IGMP Connected Group Membership
                                    <-- ไม่มีบรรทัดใดๆ เลย แม้ Signage จะส่ง IGMP Join แล้ว
```

### 646.4 การวินิจฉัย

`show ip pim interface` แสดงเฉพาะ Interface ที่เปิด `ip pim sparse-mode` เท่านั้น — Vlan50
**ไม่ปรากฏในรายการเลย** ยืนยันว่าไม่ได้เปิด PIM บน Interface นี้ และเพราะ Cisco IOS ผูก **IGMP
Process ไว้กับ PIM Mode ของ Interface** (IGMP จะทำงานอัตโนมัติก็ต่อเมื่อ Interface เปิด PIM Mode
ใดโหมดหนึ่งแล้ว) `show ip igmp groups` จึงไม่เห็น Group เลยแม้ Signage จะส่ง Report เข้ามาจริง —
ยืนยันด้วย Running-config:

```
DIST-SW2# show running-config interface Vlan50
interface Vlan50
 ip address 10.10.50.1 255.255.255.0
 ip igmp version 2
end
                                    <-- ไม่มีบรรทัด "ip pim sparse-mode" เลย
```

### 646.5 Root Cause

**`ip pim sparse-mode` ถูกเปิดเฉพาะ Global Config และ Uplink Interface แต่ไม่ได้เปิดบน SVI ใหม่
ที่มี Receiver อยู่จริง** — ตรงกับหลักการที่ [Part 41 Step 407](part-041-multicast-fundamentals.md#step-407--basic-pim-sm-configuration)
เน้นย้ำไว้ตั้งแต่ต้น: **PIM ต้องถูกเปิดบน "ทุก Interface ที่เกี่ยวข้องกับ Multicast Forwarding" ไม่ใช่
แค่จุดที่มี Source/RP** — Interface ที่ขาด `ip pim sparse-mode` จะไม่มีทาง Forward Multicast หรือ
รับ IGMP Report ได้เลย ไม่ว่า Global Config หรือ Interface อื่นจะถูกต้องเพียงใดก็ตาม

### 646.6 วิธีแก้และ Verify

```
DIST-SW2(config)# interface Vlan50
DIST-SW2(config-if)# ip pim sparse-mode
```

```
DIST-SW2# show ip pim interface
Address          Interface                Ver/  Nbr    Query  DR     DR
                                           Mode  Count  Intvl  Prior  
10.255.20.2       TenGigabitEthernet1/1/1  v2/S  1      30     1      10.255.20.1
10.10.50.1        Vlan50                   v2/S  0      30     1      10.10.50.1

DIST-SW2# show ip igmp groups
Group Address    Interface        Uptime    Expires   Last Reporter
239.1.1.1         Vlan50           00:00:07  00:02:52  10.10.50.20

DIST-SW2# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:00:07/00:02:59, RP 1.1.1.100, flags: SJC
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr 10.255.20.1
  Outgoing interface list:
    Vlan50, Forward/Sparse, 00:00:07/00:02:52
```

Signage เริ่มรับ Video ได้ทันที **บทเรียนสำคัญของ Scenario นี้**: เมื่อ Onboard Distribution Switch
ตัวใหม่เข้า Multicast Domain ให้ทำ Checklist ครบทั้ง 3 จุดเสมอ — (1) `ip multicast-routing`,
(2) `ip pim rp-address` ให้ตรงกับ Domain, (3) **`ip pim sparse-mode` บนทุก Interface ที่มี Source
หรือ Receiver รวมถึง SVI ทุกตัวที่เพิ่มใหม่ในอนาคต** ไม่ใช่แค่ Uplink

---

## Step 647 — Scenario 6: SSM ใช้งานไม่ได้เพราะ IGMPv3 ไม่ถึงปลายทาง

### 647.1 บริบท

ทีม Trading เพิ่มเครื่อง **PC-TRADE1** บน ACCESS-SW1 (`GigabitEthernet0/9`, VLAN 10) เพื่อรับ
Market-Data Feed จาก **MARKET-SRV** (`10.10.30.40` → `232.50.1.1` ผ่าน **SSM** ตาม
[Part 42 Step 420](part-042-multicast-advanced.md#step-420--lab-เต็มรูปแบบ-anycast-rp--msdp-แทน-static-rp-พร้อมเปิด-ssm))
— DIST-SW1 เปิด `ip igmp version 3` บน Vlan10 SVI ไว้แล้วตาม Lab เดิม แต่ ACCESS-SW1 เพิ่งผ่าน
Hardening Baseline Rollout ที่คัดลอก Template จาก Switch รุ่นเก่าซึ่ง**Fix IGMP Snooping Version
ไว้ที่ v2 อย่างชัดเจน** (เพื่อ Compatibility กับอุปกรณ์ Legacy บาง VLAN อื่น) โดยลืมเว้น Exception ให้
VLAN 10 ที่ต้องใช้ SSM

### 647.2 Config ที่ผิด

```
! ACCESS-SW1 — Hardening Template Fix IGMP Snooping Version ไว้ที่ v2 ทุก VLAN
ACCESS-SW1(config)# ip igmp snooping vlan 10 version 2
```

### 647.3 อาการที่พบ

```
ACCESS-SW1# show ip igmp snooping groups vlan 10
Vlan  Group            Type    Version  Port List
----  ----------------  ------  -------  -------------
10    232.50.1.1         igmp    v2       Gi0/9              <-- Version แสดงเป็น v2 ไม่ใช่ v3!

DIST-SW1# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface        Uptime    Expires   Last Reporter
                                    <-- ไม่มี 232.50.1.1 ปรากฏเลยที่ Router
```

### 647.4 การวินิจฉัย

คอลัมน์ **`Version: v2`** ใน `show ip igmp snooping groups` คือหลักฐานตรงตัว — เพราะ
`ip igmp snooping vlan 10 version 2` บังคับให้ Switch **ตีความ/ปรับลด (Normalize) ทุก Report ในVLAN
นี้ให้เป็นรูปแบบ v2** ก่อนส่งต่อขึ้นไป แม้ตัว Host/PC-TRADE1 จะส่ง Report แบบ IGMPv3 INCLUDE Mode
(พร้อมระบุ Source `10.10.30.40`) มาจริงก็ตาม — Report ที่ไปถึง DIST-SW1 จึงกลายเป็น IGMPv2
"Join แบบไม่ระบุ Source" ซึ่ง **ใช้ไม่ได้เลยกับ Group ในช่วง SSM (`232.0.0.0/8`)** เพราะ IOS ถือว่า
Group ช่วงนี้ **ต้องมี Source ระบุมาด้วยเสมอ ไม่มีแนวคิด `(*,G)` สำหรับช่วงนี้โดย Design** — Router
จึง**เงียบทิ้ง Report นี้ไปโดยไม่มี Error ใดๆ แจ้งเตือน**

### 647.5 Root Cause

**IGMP Snooping Version บน Access Switch ถูกบังคับไว้ที่ v2 ทับ Report ของ Host ที่ส่งมาแบบ v3**
ทำให้ Source Filtering ที่จำเป็นสำหรับ SSM หายไปก่อนถึง Router — ตรงกับหลักการที่
[Part 42 Step 417](part-042-multicast-advanced.md#step-417--ssm-source-specific-multicast) ย้ำไว้ว่า
**SSM ต้องมี IGMPv3 ตลอดเส้นทางตั้งแต่ Host จนถึง Router (End-to-End)** — จุดใดจุดหนึ่งตกกลับไปเป็น
v2 (ไม่ว่าจะ Host, Switch Snooping, หรือ Router Interface) ทำให้ Source Filtering หายไปทันที และ
Group ในช่วง SSM ไม่มีทางสร้าง `(S,G)` State ได้เลย

### 647.6 วิธีแก้และ Verify

```
ACCESS-SW1(config)# no ip igmp snooping vlan 10 version 2
! ค่า Default ของ IGMP Snooping รองรับ v3 อยู่แล้วโดยไม่ต้องระบุ version เพิ่ม
```

```
ACCESS-SW1# show ip igmp snooping groups vlan 10
Vlan  Group            Type    Version  Port List
----  ----------------  ------  -------  -------------
10    232.50.1.1         igmp    v3       Gi0/9

DIST-SW1# show ip mroute 232.50.1.1
(10.10.30.40, 232.50.1.1), 00:00:11/00:03:18, flags: sTI
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr 10.255.10.1
  Outgoing interface list:
    Vlan10, Forward/Sparse, 00:00:11/00:03:18
```

`(S,G)` State สร้างสำเร็จทันที (Flag `s` = SSM Group, ไม่มี `RP` Field ตามที่คาด) — PC-TRADE1
รับ Feed ได้ปกติ

### 647.7 ตัวแปรที่สอง — `ip pim ssm range` ไม่ครอบคลุม Group ที่ใช้จริง

ปัญหาแบบเดียวกันเกิดได้อีกทางถ้าทีมงานเพิ่ม Feed จาก Vendor ตลาดหลักทรัพย์อีกรายที่ประกาศ Group
Address เป็น `238.10.10.10` (คนละช่วงกับ `232.0.0.0/8`) — แม้ Host/Switch/Router จะพูด IGMPv3
ถูกต้องครบทุกจุด แต่ถ้า Router มีแค่:

```
CORE-SW1(config)# ip pim ssm default
! ครอบคลุมเฉพาะ 232.0.0.0/8 (Default Range) เท่านั้น — ไม่ครอบคลุม 238.10.10.10
```

Router จะ**ไม่ปฏิบัติต่อ Group นี้แบบ SSM เลย** (เพราะ IGMPv3 Report ที่มี Source ระบุมา จะถูก
ยอมรับแบบ SSM ก็ต่อเมื่อ Group Address นั้นอยู่ในขอบเขตที่ `ip pim ssm range` ครอบคลุมเท่านั้น) —
ต้องเพิ่มขอบเขตด้วย Access-list ตามที่ [Part 42 Step 417](part-042-multicast-advanced.md#step-417--ssm-source-specific-multicast)
สอนไว้:

```
CORE-SW1(config)# access-list 20 permit 232.0.0.0 0.255.255.255
CORE-SW1(config)# access-list 20 permit 238.10.10.10 0.0.0.0
CORE-SW1(config)# ip pim ssm range 20
```

> **ข้อสังเกตสำหรับข้อสอบ**: ปัญหาทั้งสองแบบใน Step นี้ **ไม่มี Log หรือ Error แจ้งเลย** — ต้องตรวจ
> `show ip igmp snooping groups` (เช็ค Version) และ `show running-config | include ssm` (เช็ค
> ขอบเขต Range) เชิงรุกเสมอเมื่อ SSM Group ใหม่ใช้งานไม่ได้

---

## Step 648 — Scenario 7: Storm-Control เข้าใจผิดว่า Video Stream ถูกต้องคือ "Storm"

### 648.1 บริบท

ทีม Security ทำ Hardening Rollout ตาม [Part 21 Step 209](part-021-switch-security.md) กับทุก
Access Port ของ ACCESS-SW3 (รวมถึง `GigabitEthernet0/5` ที่ **STREAM-SRV เชื่อมต่ออยู่จริง**)
โดยใช้ Template ที่ตั้งใจไว้สำหรับ Endpoint ทั่วไป (PC/Printer) ซึ่งมี Threshold ที่**เข้มงวดเกินไป**
สำหรับ Port ที่มี Traffic โดยธรรมชาติสูงอย่าง Video Server

### 648.2 Config ที่ผิด

```
! ACCESS-SW3 — Template สำหรับ Endpoint ทั่วไป ถูกใช้กับ Port ของ STREAM-SRV โดยไม่ได้แยก Exception
ACCESS-SW3(config)# interface GigabitEthernet0/5
ACCESS-SW3(config-if)# storm-control multicast level 10.00
ACCESS-SW3(config-if)# storm-control action trap
```

### 648.3 อาการที่พบ

พนักงานรายงานว่า Video ใน All-Hands **กระตุก/หลุดเป็นช่วงๆ** เฉพาะช่วงที่ Video Encoder ส่ง
Bitrate สูงขึ้นชั่วขณะ (Motion สูง, VBR Encoding) — ไม่ใช่ตลอดเวลา จึงดูเหมือนเป็นปัญหา Network
ทั่วไปในตอนแรก

```
ACCESS-SW3# show storm-control multicast
Interface     Filter State   Upper       Lower      Current
---------     -------------  ----------  ---------  ----------
Gi0/5          Blocking       10.00%      10.00%     15.80%
```

**`Filter State: Blocking`** และ **`Current: 15.80%` เกิน `Upper: 10.00%`** คือหลักฐานตรงตัว —
Switch กำลัง Drop Multicast Traffic **ที่เข้ามาจาก STREAM-SRV เอง (Ingress)** ทุกครั้งที่ Bitrate
ของ Video เกิน 10% ของ 1 Gbps (~100 Mbps)

### 648.4 การวินิจฉัย

```
ACCESS-SW3# show logging | include storm-control
%STORM_CONTROL-3-FILTERED: A Multicast storm detected on Gi0/5. Multicast traffic is being
suppressed.
```

Log ยืนยันว่า Switch มองว่า Traffic ปกติจาก STREAM-SRV เป็น "Storm" — Threshold `10.00%` ต่ำ
เกินไปสำหรับ Traffic Profile ของ Video Server ที่ Bitrate ปกติอยู่ที่ประมาณ 8-12% แล้วพุ่งเกิน 15%
เป็นช่วงๆ ตามธรรมชาติของ VBR Video ซึ่ง**ไม่ใช่ Storm จริงเลย** (ไม่มี Loop, ไม่มี Broadcast Flood
ผิดปกติ — เป็น Traffic ที่ถูกต้องตาม Design ของ Multicast เอง)

### 648.5 Root Cause

**Storm-control Threshold ที่ตั้งไว้สำหรับ Endpoint ทั่วไปถูกนำมาใช้กับ Port ของ Multicast Video
Source โดยไม่ได้ปรับให้เหมาะกับ Traffic Profile จริง** — Storm-control ไม่มีความสามารถแยกแยะ
"Multicast ที่ถูกต้องตาม Design" ออกจาก "Multicast Storm จริง" ได้เลย มันแค่นับ % ของ Bandwidth
เทียบ Threshold เท่านั้น ถ้า Threshold ตั้งต่ำกว่า Traffic ปกติของ Application ที่ถูกต้อง มันจะ Drop
Traffic ที่ถูกต้องนั้นเสมอ ไม่ต่างจาก False Positive ของระบบ Security อื่นๆ

### 648.6 วิธีแก้และ Verify

มี 2 ทางเลือกตามลักษณะ Port — Port ของ Video Server เป็น Known-good Trusted Source (ไม่มี Host
อื่นเสียบสลับได้ง่าย) จึงเลือกปรับ Threshold ให้เหมาะสมกับ Traffic Profile จริงมากกว่าปิดทั้งหมด:

```
ACCESS-SW3(config)# interface GigabitEthernet0/5
ACCESS-SW3(config-if)# storm-control multicast level 80.00 70.00
! ปรับ Threshold ให้สูงพอสำหรับ Traffic Profile ของ Video Server จริง
! (80% ยังคงป้องกัน Storm จริงระดับ Loop/Broadcast Flood ที่ควรกินเกือบทั้ง Link ได้อยู่)
```

```
ACCESS-SW3# show storm-control multicast
Interface     Filter State   Upper       Lower      Current
---------     -------------  ----------  ---------  ----------
Gi0/5          Forwarding     80.00%      70.00%     15.80%
```

`Filter State: Forwarding` กลับสู่ปกติ — Video ไม่กระตุกอีก **ข้อคิดสำคัญ**: เมื่อ Apply Hardening
Template แบบ Blanket ([Part 21](part-021-switch-security.md)) ให้ตรวจสอบ Traffic Profile จริงของ
แต่ละ Port ก่อนเสมอ โดยเฉพาะ Port ที่เป็น Known Server/Source ที่มี Traffic Pattern สูงกว่า
Endpoint ทั่วไปโดย Design — Storm-control ควรตั้งสูงกว่า Traffic ปกติสูงสุดที่คาดไว้พอสมควร (เผื่อ
Margin สำหรับ Traffic Spike ที่ถูกต้อง) ไม่ใช่ตั้งตาม Template เดียวกันทุก Port แบบไม่แยกแยะ

---

## Step 649 — Scenario 8: Auto-RP/BSR ไม่ถูก Flood ข้าม Transit Interface หลังปรับ ACL ที่ไม่เกี่ยวข้อง

> **หมายเหตุก่อนเริ่ม**: Production Design จริงของหลักสูตรนี้ใช้ **Static Anycast RP** ตาม
> [Part 42 Step 420](part-042-multicast-advanced.md#step-420--lab-เต็มรูปแบบ-anycast-rp--msdp-แทน-static-rp-พร้อมเปิด-ssm)
> เป็นหลักแล้ว — แต่ระหว่าง Migration Window ทีมงานยังเปิด **Auto-RP ทิ้งไว้เป็น Fallback Mechanism**
> คู่ขนานกันบางส่วนของ Domain (ตามที่ [Part 42 Step 412](part-042-multicast-advanced.md#step-412--auto-rp-cisco-proprietary)
> Config ไว้) เพื่อรองรับอุปกรณ์ Legacy บางตัวที่ยังไม่ Support Static RP Reconfiguration ระยะสั้นๆ —
> Step นี้ฝึกวินิจฉัยปัญหา Auto-RP โดยเฉพาะ ซึ่งเป็นแนวข้อสอบ ENARSI ที่ถูกถามแยกจาก Design จริง
> เสมอ

### 649.1 บริบท

ทีม Security ทำ Cleanup ACL บน **CORE-SW2** เพื่อ Block Multicast Group เก่าที่เลิกใช้แล้ว
(`224.2.0.0/16` — Legacy Video Conferencing Range จากระบบเก่าก่อน STREAM-SRV) โดย Apply
ผ่าน `ip multicast boundary` บน Transit Interface ที่ไปยัง DIST-SW4 — วิศวกรเขียน Wildcard Mask
ผิดจนกลืน Group ที่ไม่เกี่ยวข้องไปด้วย

### 649.2 Config ที่ผิด

```
! CORE-SW2 — ACL ตั้งใจ Block แค่ 224.2.0.0/16 (Legacy) แต่ Wildcard Mask เขียนผิด
CORE-SW2(config)# access-list 30 deny 224.2.0.0 15.255.255.255
                                    !          <-- ผิด! ควรเป็น 0.0.255.255 (Mask ของ /16)
                                    !          Wildcard 15.255.255.255 กลืน 224.0.0.0-239.255.255.255
                                    !          เกือบทั้งหมด รวมถึง 224.0.1.39/40 (Auto-RP) ด้วย!
CORE-SW2(config)# access-list 30 permit ip any any
CORE-SW2(config)# interface TenGigabitEthernet1/0/2
CORE-SW2(config-if)# ip multicast boundary 30
```

### 649.3 อาการที่พบ

```
DIST-SW4# show ip pim rp mapping
PIM Group-to-RP Mappings
This system is not a Mapping Agent

Group(s) 239.0.0.0/8
                                    <-- ไม่มี Entry เลย! (ก่อนหน้านี้เคยเห็น RP ปกติ)

DIST-SW1# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via static
```

DIST-SW1 ยังเห็น RP ปกติ (ผ่าน Static Anycast RP ซึ่งไม่ได้พึ่ง Multicast Group พิเศษเลย) แต่
**DIST-SW4 ที่ยังพึ่ง Auto-RP Fallback สำหรับ Sub-domain ของตัวเองไม่เห็น RP Mapping เลย**

### 649.4 การวินิจฉัย

```
CORE-SW2# show access-lists 30
Standard IP access list 30
    10 deny   224.2.0.0, wildcard bits 15.255.255.255 (matched many times)
    20 permit any (matched many times)

CORE-SW2# show ip mroute 224.0.1.40
Group 224.0.1.40 not found                    <-- Auto-RP Discovery Group ไม่มีเลยหลัง Boundary นี้!
```

Wildcard Mask `15.255.255.255` (Binary `00001111.11111111.11111111.11111111`) ทำให้ ACL Match
**ทุก Address ที่ Octet แรกอยู่ในช่วง `224.x.x.x` ถึง `239.x.x.x` เกือบทั้งหมด** (เพราะ 4 bit บนของ
Octet แรกถูก "Don't Care" ไปด้วยความผิดพลาด) รวมถึง **`224.0.1.39` (RP-Announce) และ
`224.0.1.40` (RP-Discovery)** ที่ Auto-RP ต้องพึ่งพาโดยตรง — `ip multicast boundary` จึง Deny
Group เหล่านี้ไม่ให้ข้าม Interface นี้ไปยัง DIST-SW4 เลย ตัด Auto-RP Flooding ขาดสนิทที่จุดนี้

### 649.5 Root Cause

**Wildcard Mask ของ ACL ที่ผูกกับ `ip multicast boundary` เขียนผิดจนกลืน Auto-RP Control Group
(`224.0.1.39`/`224.0.1.40`) ไปด้วยโดยไม่ตั้งใจ** — ตรงกับ Checklist ที่ [Part 42 Step 419](part-042-multicast-advanced.md#step-419--verification--troubleshooting-ขั้นสูง)
เตือนไว้แล้ว: "Auto-RP ทำงานได้บางส่วน (Local Segment) แต่ไม่กระจายไปทั้ง Domain — Group
`224.0.1.39/40` ถูก Deny โดย Multicast Boundary/ACL ที่ตั้งไว้โดยไม่ตั้งใจ" — ผลลัพธ์เหมือนกับการรัน
`no ip pim autorp listener` บน Router นั้นเลย เพราะทั้งสองกรณีทำให้ Router ปลายทางไม่มีทาง
ได้รับ RP-Discovery Message ข้าม Interface นี้ได้อีกต่อไป

### 649.6 วิธีแก้และ Verify

```
CORE-SW2(config)# no access-list 30 deny 224.2.0.0 15.255.255.255
CORE-SW2(config)# ip access-list standard 30
CORE-SW2(config-std-nacl)# no 10
CORE-SW2(config-std-nacl)# deny   224.2.0.0 0.0.255.255
CORE-SW2(config-std-nacl)# permit any
```

```
CORE-SW2# show ip mroute 224.0.1.40
(*, 224.0.1.40), ..., RP 0.0.0.0, flags: DC
  Incoming interface: Null, RPF nbr 0.0.0.0
  Outgoing interface list:
    TenGigabitEthernet1/0/2, Forward/Dense, ...

DIST-SW4# show ip pim rp mapping
Group(s) 239.0.0.0/8
  RP 1.1.1.100 (?), v2v1
    Info source: 1.1.1.100 (?), via Auto-RP
```

RP Mapping กลับมาปกติผ่าน Auto-RP อีกครั้ง **ข้อคิดสำคัญ**: เมื่อเขียน ACL ผูกกับ
`ip multicast boundary` หรือ Filter ใดๆ ที่ครอบคลุม Class D Range ให้ตรวจ Wildcard Mask ให้
เจาะจงที่สุดเสมอ (`show access-lists` ก่อน Apply จริง) และเผื่อพื้นที่ยกเว้น `224.0.1.39`/`224.0.1.40`
ไว้เสมอถ้า Domain ยังพึ่งพา Auto-RP อยู่ไม่ว่าจะเป็น Mechanism หลักหรือ Fallback ก็ตาม

---

## Step 650 — Lab เต็มรูปแบบ: RPF Failure + MSDP Misconfiguration เกิดพร้อมกันหลัง Network Redesign

### 650.1 บริบท

ทีม Network ทำ **Redesign Project** เพื่อรองรับ Traffic ที่โตขึ้นบน Backbone — งานนี้มี 2 ส่วนที่
ทำในหน้าต่าง Maintenance เดียวกัน:

1. **ปรับ OSPF Cost บนลิงก์ Dual-homed จาก [Part 33](part-033-advanced-stp-campus-design.md)**
   เพื่อ Traffic-Engineer ให้ CORE-SW1 ใช้ลิงก์ตรงไป DIST-SW3 (`TenGigabitEthernet1/0/3`,
   `10.255.31.0/30`) เป็นทางหลักสำหรับ Unicast Traffic บางประเภท
2. **Re-peer MSDP** ระหว่าง CORE-SW1/CORE-SW2 เพราะทีมงานกำลังทำความสะอาด Loopback
   Numbering Plan ไปพร้อมกัน

ทั้งสองงานทำโดยวิศวกรคนละคนในตารางเดียวกัน ไม่ได้ Cross-check กันก่อน Deploy

### 650.2 Config ที่ผิดทั้งสองจุด

```
! ==== จุดที่ 1: CORE-SW1 — ปรับ OSPF Cost เพื่อ Traffic Engineering ====
CORE-SW1(config)# interface TenGigabitEthernet1/0/3
CORE-SW1(config-if)# ip ospf cost 5
! ทำให้เส้นทางตรงไป DIST-SW3 (1 Hop) ถูกเลือกสำหรับ Unicast RIB ของ 10.10.30.0/24 เสมอ
! แต่ STREAM-SRV Register/(S,G) Data เดิมยังไหลผ่าน Port-channel1 (ทาง CORE-SW2) อยู่
! เพราะ DIST-SW3 เชื่อมกับ CORE-SW2 โดยตรง (ลิงก์เดิม, Cost ต่ำกว่าสำหรับตัว DIST-SW3 เอง)

! ==== จุดที่ 2: MSDP Re-peer ผิดพลาด (ซ้ำรูปแบบ Step 645) ====
CORE-SW1(config)# no ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW1(config)# ip msdp peer 1.1.1.100 connect-source Loopback1

CORE-SW2(config)# no ip msdp peer 1.1.1.1 connect-source Loopback0
CORE-SW2(config)# ip msdp peer 1.1.1.100 connect-source Loopback1
```

### 650.3 อาการที่พบ

หลัง Maintenance Window ปิดงาน — Helpdesk ได้รับ Ticket ว่า **PC-SALES1 (ใกล้ CORE-SW1) ดูวิดีโอ
ไม่ได้เลย** ในขณะที่ **PC-WIFI1 (ใกล้ CORE-SW2) ยังดูได้ปกติ** — อาการไม่สมมาตรแบบนี้เป็นสัญญาณว่า
มีมากกว่า 1 ปัญหาเกิดพร้อมกัน

### 650.4 การวินิจฉัย — จุดที่ 1: RPF Failure

```
CORE-SW1# show ip rpf 10.10.30.30
RPF information for ? (10.10.30.30)
  RPF interface: TenGigabitEthernet1/0/3
  RPF neighbor: ? (10.255.31.2)
  RPF route/mask: 10.10.30.0/24
  RPF type: Unicast (ospf 1)

CORE-SW1# show ip mroute count
Group: 239.1.1.1, Source count: 1, Group pkt count: 0
  Source: 10.10.30.30/32, Forwarding Counts: 0/0/0/0, Other counts: 8820/8820/0
```

CORE-SW1 คำนวณ RPF Interface เป็น `TenGigabitEthernet1/0/3` (ทางตรงไป DIST-SW3) ตาม Cost ใหม่
— แต่ DIST-SW3 เอง (First-Hop Router ของ STREAM-SRV) ยังคง Register/ส่ง Data เข้าสู่ Core ทาง
**CORE-SW2** อยู่ (เพราะ Cost จากมุมของ DIST-SW3 เองไม่ได้เปลี่ยน — ลิงก์เดิมไป CORE-SW2 ยังถูก
เลือกอยู่) ทำให้ Data จริงไหลเข้า CORE-SW1 ทาง **Port-channel1** ไม่ใช่ `Te1/0/3` ที่ RPF คาดไว้ —
`Other counts: 8820/8820/0` ยืนยัน RPF Fail 100%

### 650.5 วิธีแก้ — จุดที่ 1

Traffic Engineering Requirement (ต้องการให้ Unicast ไปทาง `Te1/0/3`) ยังต้องคงอยู่ — ทางแก้ที่ถูก
ต้องคือใช้ **`ip mroute`** บังคับ RPF Interface ของ Multicast โดยเฉพาะ แยกออกจาก Unicast RIB ที่
เพิ่งปรับ Cost ไป โดยไม่ต้อง Rollback Traffic Engineering ทั้งหมด:

```
CORE-SW1(config)# ip mroute 10.10.30.0 255.255.255.0 Port-channel1
```

```
CORE-SW1# show ip rpf 10.10.30.30
RPF information for ? (10.10.30.30)
  RPF interface: Port-channel1
  RPF neighbor: ? (10.255.0.2)
  RPF route/mask: 10.10.30.0/24
  RPF type: Multicast (static)                    <-- ใช้ ip mroute แยกจาก Unicast RIB แล้ว

CORE-SW1# show ip mroute count
Group: 239.1.1.1, Source count: 1, Group pkt count: 4210
  Source: 10.10.30.30/32, Forwarding Counts: 4210/26/1400/291, Other counts: 4210/0/0
```

RPF กลับมา Pass — `Other counts` เหลือ `0` RPF Failed

### 650.6 การวินิจฉัย — จุดที่ 2: MSDP

```
CORE-SW1# show ip msdp summary
Peer Address    AS    State       Uptime/   Reset  SA     Peer Name
                                  Downtime  Count   Count
1.1.1.100        -     Connecting  00:18:02  11      0

CORE-SW1# show ip msdp peer 1.1.1.100
  Local host: 1.1.1.100
  Remote host: 1.1.1.100                           <-- ซ้ำรูปแบบเดียวกับ Step 645 เป๊ะ
```

Root Cause เดียวกับ Step 645: `connect-source` และ `peer` ใช้ Anycast Address (`1.1.1.100`) ทั้งคู่

### 650.7 วิธีแก้ — จุดที่ 2

```
CORE-SW1(config)# no ip msdp peer 1.1.1.100 connect-source Loopback1
CORE-SW1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0

CORE-SW2(config)# no ip msdp peer 1.1.1.100 connect-source Loopback1
CORE-SW2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
```

### 650.8 ตารางสรุปทั้ง 2 ปัญหาและ Verification สุดท้าย

| # | ปัญหา | จุดที่เจอ | Root Cause | คำสั่ง Fix หลัก |
|---|---|---|---|---|
| 1 | RPF Failure บน `239.1.1.1` | CORE-SW1 | OSPF Cost Traffic-Engineering ทำให้ RPF Interface ไม่ตรงกับทาง Data มาจริง | `ip mroute 10.10.30.0 255.255.255.0 Port-channel1` |
| 2 | MSDP Peer ค้างที่ `Connecting`, SA Cache ว่าง | CORE-SW1 ↔ CORE-SW2 | `connect-source`/`peer` ใช้ Anycast Address (`1.1.1.100`) ซ้ำรูปแบบ Step 645 | เปลี่ยนกลับเป็น Loopback0 Unique (`1.1.1.1`/`1.1.1.2`) |

```
! Verification สุดท้าย — ก่อนแก้ (Before)
CORE-SW1# show ip mroute count | include Other counts
  Source: 10.10.30.30/32, Forwarding Counts: 0/0/0/0, Other counts: 8820/8820/0
CORE-SW1# show ip msdp sa-cache
MSDP Source-Active Cache - 0 entries

! Verification สุดท้าย — หลังแก้ทั้ง 2 จุด (After)
CORE-SW1# show ip mroute count | include Other counts
  Source: 10.10.30.30/32, Forwarding Counts: 6104/26/1400/291, Other counts: 6104/0/0

CORE-SW1# show ip msdp summary
Peer Address    AS    State   Uptime/   Reset  SA     Peer Name
                              Downtime  Count   Count
1.1.1.2          -     Up      00:00:31  12      1      CORE-SW2-AnycastRP-Peer

CORE-SW1# show ip msdp sa-cache
MSDP Source-Active Cache - 1 entries
(10.10.30.30, 239.1.1.1), RP 1.1.1.1, MBGP/AS -, 00:00:24/00:05:36, Peer 1.1.1.2

PC-SALES1> ping 239.1.1.1
! Video เริ่มไหลปกติ ยืนยันจาก Helpdesk Ticket ปิด
```

> **บทเรียนของ Lab สุดท้ายนี้**: Network Redesign ที่แก้ Unicast Routing (OSPF Cost) และ
> Multicast Control-Plane (MSDP) พร้อมกันในหน้าต่างเดียว **มีความเสี่ยงซ้อนกันสูงมาก** เพราะทั้งสอง
> ระบบดูเหมือนไม่เกี่ยวกัน แต่ RPF Check ของ Multicast **ผูกกับ Unicast RIB โดยตรงเสมอ** — ทุกครั้ง
> ที่ปรับ OSPF Cost/Path Control ในเครือข่ายที่มี Multicast ทำงานอยู่ ต้องตรวจ `show ip rpf` และ
> `show ip mroute count` ควบคู่ไปด้วยเสมอ ไม่ใช่แค่ตรวจ `show ip route`/`show ip ospf neighbor`
> ตามปกติ — และ `ip mroute` คือเครื่องมือสำคัญที่ทำให้ Traffic Engineering กับ Multicast RPF
> อยู่ร่วมกันได้โดยไม่ต้องเลือกอย่างใดอย่างหนึ่ง

---

## แบบฝึกหัดทวนความเข้าใจ Part 65

1. ตาม Methodology ของ Step 641 ถ้า Receiver ไม่เห็น Video เลย และ `show ip igmp groups` บน
   Router **ไม่เห็น Group นั้นเลย** ควรไปตรวจ Layer ไหนต่อก่อน และเพราะเหตุใดการไปดู
   `show ip mroute` ทันทีจึงเสียเวลาเปล่าในกรณีนี้?
2. IGMP Snooping mrouter Learning Method `pim-dvmrp` กับ `cgmp` ต่างกันอย่างไร และทำไมการตั้ง
   `cgmp` ผิดพลาดจึงทำให้ Host Join ไม่ถึง Router เลยแม้ Host จะ Join ถูกต้องทุกประการ (Step 642)?
3. RPF Failure จาก Static Route (Step 643) กับ RPF Failure จาก Traffic Engineering/OSPF Cost
   (Step 650) ต่างกันอย่างไรในมุมของ "ทางแก้ที่ถูกต้อง" — ทำไม Step 650 เลือกใช้ `ip mroute` แทน
   การ Rollback OSPF Cost?
4. อธิบายว่าทำไม MSDP Peer ที่ใช้ Anycast Address (`1.1.1.100`) เป็นทั้ง `connect-source` และ
   Peer Address จึงทำให้ `show ip msdp peer` แสดง `Local host` และ `Remote host` เป็น Address
   เดียวกัน และเหตุใด TCP Session (Port 639) จึงไม่มีวันสร้างสำเร็จ?
5. ทำไม SSM Group (`232.0.0.0/8`) ถึงไม่มีแนวคิด `(*,G)` เลย และการที่ IGMP Snooping บังคับ
   Version เป็น v2 (Step 647) ทำให้ Join ของ SSM Group หายไปอย่างไร โดยไม่มี Error ใดๆ แจ้งเตือน?

**เฉลย:**

1. ควรตรวจ **Layer 1 (IGMP Membership)** ต่อทันที — ดูว่า `show ip igmp snooping groups` บน
   Access Switch เห็น Host Join หรือไม่ ถ้าเห็นแต่ Router ไม่เห็น แปลว่าปัญหาอยู่ที่ IGMP Snooping
   ไม่ Forward Report ขึ้นไป (เช่น mrouter Port ไม่ถูกเรียนรู้ ตาม Step 642) — การไปดู
   `show ip mroute` ทันทีเสียเวลาเปล่าเพราะถ้า Router ไม่เคยเห็น Group นี้เลย Entry ของ `(*,G)`
   ที่มี Interface นั้นอยู่ใน OIL ก็จะไม่มีทางถูกสร้างขึ้นมาได้ตั้งแต่ต้น ไม่ว่า PIM/RPF จะถูกต้องเพียงใด
2. `pim-dvmrp` (Default) ทำให้ Switch เรียนรู้ Router-facing Port โดยอัตโนมัติจากการ "แอบดู" PIM
   Hello Packet (`224.0.0.13`) หรือ IGMP Query ที่ Router ส่งออกมาจริงบน Trunk ส่วน `cgmp` รอ
   Packet แบบ Cisco Group Management Protocol ซึ่งเป็น Legacy Mechanism ที่ Cisco IOS สมัยใหม่
   ไม่ส่งออกมาอีกแล้ว — เมื่อตั้ง `cgmp` ผิดพลาด Switch จะไม่มีวันเห็น Packet ที่คาดไว้เลย จึงไม่เรียนรู้
   mrouter Port ได้เลย และ Report ของ Host ทุกตัวจะไม่ถูก Forward ไปหา Router (Report ถูกส่งออก
   ทาง mrouter Port เท่านั้นตาม Design ของ IGMP Snooping) แม้ Host จะ Join ถูกต้องทุกประการ
3. Step 643 เป็น **Bug ที่ไม่มีเหตุผลอันสมควร** (Static Route ทดสอบที่ลืมถอด) ทางแก้ที่ถูกต้องคือ
   **ลบ Static Route ตัวนั้นออก** ให้ Unicast RIB กลับไปใช้ OSPF ตามปกติ ส่วน Step 650 การเปลี่ยน
   OSPF Cost เป็น **Requirement ทาง Business ที่ต้องคงอยู่จริง** (Traffic Engineering) การ Rollback
   Cost จะแก้ปัญหา Multicast ได้ก็จริง แต่ทำให้เสีย Requirement ของ Unicast ไปด้วย — จึงใช้
   `ip mroute` สร้าง RPF Path แยกเฉพาะสำหรับ Multicast โดยไม่ไปยุ่งกับ Unicast RIB ที่ตั้งใจปรับ
   Cost ไว้แล้ว ทำให้ทั้งสอง Requirement อยู่ร่วมกันได้
4. เพราะ Anycast Address (`1.1.1.100`) มีอยู่จริงบนทั้ง CORE-SW1 และ CORE-SW2 พร้อมกัน เมื่อ
   Router เปิด TCP Session (Port 639) จาก Address ของตัวเอง (`1.1.1.100`) ไปยัง Address ปลายทาง
   ที่ Config ไว้ (`1.1.1.100` เหมือนกัน) ทั้ง Local และ Remote Field จึงเป็นค่าเดียวกันเสมอ — TCP
   ไม่สามารถแยกแยะได้ว่ากำลังพยายามเชื่อมต่อกับตัวเองหรือกับอีกฝั่งจริง เพราะ Address เดียวกันมีอยู่
   สองที่พร้อมกันโดย Design ของ Anycast เอง ทำให้ Session ไม่มีวันสร้างสำเร็จตามที่ตั้งใจ
5. SSM ถูกออกแบบให้ **ทุก Join ต้องระบุ Source มาด้วยเสมอ** (INCLUDE Mode ของ IGMPv3) เพราะ
   ตัด RP/Shared Tree ออกจากสมการทั้งหมด — ไม่มี RP ให้ Join แบบไม่ระบุ Source เข้าไปหาได้เลย
   จึงไม่มีแนวคิด `(*,G)` สำหรับช่วงนี้ ส่วน IGMP Snooping ที่บังคับ Version เป็น v2 จะปรับ Report
   ของ Host ให้อยู่ในรูปแบบ v2 (ไม่มี Source Field) ก่อนส่งต่อขึ้นไป — เมื่อ Router ได้รับ Report แบบ
   ไม่มี Source สำหรับ Group ในช่วง SSM มันจะไม่รู้จะสร้าง `(S,G)` ตัวไหนเลย และเพราะ Group นี้ไม่มี
   `(*,G)` ให้ Fallback ไปใช้ได้ Router จึงเงียบทิ้ง Report นั้นไปโดยไม่มี Error หรือ Log ใดๆ แจ้งเตือน

---

## สรุป Part 65

Part นี้พา Multicast Design ที่สมบูรณ์จาก **Part 41-42** (PIM-SM, Anycast RP + MSDP, SSM,
IGMP Snooping) กลับมาทดสอบผ่าน **8 Broken Scenario จริง** ตามแนวข้อสอบ **CCNP ENARSI**:
IGMP Snooping ไม่เรียนรู้ Router Port จากการตั้ง mrouter Learning Method ผิด, RPF Failure จาก
Static Route ทดสอบที่ลืมถอด, RP Mapping ไม่ตรงกันหลัง Migrate ไป Anycast RP ไม่ครบ, MSDP Peer
ใช้ Anycast Address ผิดพลาดแทน Loopback0, ลืม `ip pim sparse-mode` บน SVI ใหม่, SSM พังจาก
IGMP Version ที่ไม่ครบ End-to-End, Storm-control เข้าใจผิดว่า Video ปกติคือ Storm, และ Auto-RP
ถูกตัดขาดจาก ACL Wildcard Mask ที่เขียนผิด ปิดท้ายด้วย **Lab รวม 2 ปัญหาซ้อนกัน** (RPF Failure
จาก Traffic Engineering + MSDP Misconfiguration) หลัง Network Redesign ที่สอนให้ใช้
**`ip mroute`** แยก RPF Path ของ Multicast ออกจาก Unicast RIB เมื่อทั้งสอง Requirement ต้องอยู่
ร่วมกัน ✅

**สิ่งที่ต้องจำที่สุดจาก Part นี้**: Multicast แทบไม่มี Syslog เตือนเมื่อ RPF Fail หรือ MSDP SA ไม่
Sync ต่างจาก OSPF/EIGRP ที่มี Log ชัดเจนเสมอ ([Part 57](part-057-ospf-troubleshooting.md),
[Part 56](part-056-eigrp-troubleshooting.md)) — ต้องฝึกอ่าน **Counter** (`show ip mroute count`,
`show ip msdp summary`) เชิงรุกและไล่ตาม Layer ของ Step 641 อย่างมีระบบเสมอ ไม่ใช่รอ Log แจ้ง
หรือเดา Config สุ่มๆ

**Part ถัดไป** จะเปลี่ยนโฟกัสจาก Wired Infrastructure ไปสู่ **Wireless** เต็มรูปแบบเป็นครั้งแรกของสาย
Troubleshooting — **Wireless Troubleshooting** ที่ดึง WLC-1/AP-1/AP-2 จาก
[Part 20](part-020-wireless-fundamentals.md), [Part 36](part-036-wireless-architecture-deep-dive.md)
และ [Part 37](part-037-wireless-security-advanced.md) กลับมาวินิจฉัยปัญหาที่ Wireless มีลักษณะ
เฉพาะตัวไม่เหมือน Wired เลย (RF Interference, Roaming, AP Join Failure, Client Authentication)

**ไปต่อ:** [Part 66 — Wireless Troubleshooting →](part-066-wireless-troubleshooting.md)
