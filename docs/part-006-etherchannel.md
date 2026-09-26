# Part 6 — EtherChannel (LACP/PAgP)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 51–60 จาก 1000**

> ต่อจาก [Part 5 — STP / RSTP / MSTP](part-005-stp-rstp-mstp.md) ที่เราเรียนวิธีที่ Spanning Tree
> Protocol ป้องกัน Loop โดย **Block พอร์ตส่วนเกินทิ้ง** — ซึ่งแปลว่า Bandwidth ของ Link สำรองถูก
> เสียเปล่าไปครึ่งหนึ่ง (หรือมากกว่า) เสมอ Part นี้เราจะแก้ปัญหานั้นด้วย **EtherChannel**
> เทคโนโลยีที่รวม Physical Link หลายเส้นให้กลายเป็น Logical Link เดียว ทำให้ STP มองเห็นเป็นแค่
> Link เดียว (ไม่ Block) แต่ได้ Bandwidth และ Redundancy ของทุกเส้นจริง เราจะใช้ Link ระหว่าง
> **CORE-SW1 ↔ CORE-SW2** ที่ประกาศไว้ตั้งแต่ Part 1 (`CORE-SW1 <===LACP Po1===> CORE-SW2`)
> เป็น Lab หลักของ Part นี้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 51 | ทำไมต้องใช้ EtherChannel — Bandwidth Aggregation, Redundancy โดยไม่ถูก STP Block |
| 52 | EtherChannel Protocol — PAgP vs LACP และ Mode ต่างๆ |
| 53 | Negotiation Mode Combination — Combo ไหนรวม Channel ได้ ไหนไม่ได้ |
| 54 | Config LACP EtherChannel บน CORE-SW1 ↔ CORE-SW2 (Po1) ทีละขั้นตอน |
| 55 | Load-Balancing Algorithm ของ EtherChannel และการ Hashing |
| 56 | ข้อกำหนดการรวมพอร์ตเข้า EtherChannel และผลของการตั้งค่าไม่ตรงกัน (Err-Disable) |
| 57 | Layer 3 EtherChannel (Routed Port-Channel) เทียบกับ Layer 2 EtherChannel |
| 58 | LACP Fast Rate และ `port-channel min-links` |
| 59 | Troubleshooting EtherChannel — คำสั่ง Verify และการอ่าน Flag |
| 60 | Lab เต็มรูปแบบ: EtherChannel บน CORE-SW1 ↔ CORE-SW2 พร้อม Verify |

---

## Step 51 — ทำไมต้องใช้ EtherChannel

### ปัญหาที่ EtherChannel แก้

ใน Part 5 เราเห็นว่าเมื่อมี Link คู่ขนานระหว่างสอง Switch (เพื่อ Redundancy) STP จะเลือก Link
ที่ดีที่สุดเป็น **Forwarding** และ Block อีก Link ที่เหลือทันที เพราะ STP มองแต่ละสาย Physical
เป็น Port แยกกัน และกฎของ STP คือ "ห้ามมี Loop เด็ดขาด" — ผลคือถ้าใช้ Cable 2 เส้น ความเร็วรวม 2 Gbps
(เช่น 2x GigabitEthernet) แต่ใช้งานได้จริงแค่ 1 Gbps เพราะอีกเส้น Block ทิ้งเปล่าๆ

```
ไม่มี EtherChannel (STP มองเห็น 2 Port แยกกัน)          มี EtherChannel (STP มองเห็น 1 Logical Port)

   CORE-SW1                CORE-SW2                    CORE-SW1                CORE-SW2
   Gi0/1 ─────Forwarding───── Gi0/1                     Gi0/1 ┐                ┌ Gi0/1
   Gi0/2 ─────BLOCKED ──────  Gi0/2                     Gi0/2 ┴──Po1 (2Gbps)───┴ Gi0/2
                                                              (STP เห็นเป็น Port เดียว = ไม่ Block)

   Bandwidth ใช้งานจริง = 1 Gbps (สูญเปล่า 1 Gbps)         Bandwidth ใช้งานจริง = 2 Gbps เต็ม
```

### สามประโยชน์หลักของ EtherChannel

| ประโยชน์ | คำอธิบาย |
|---|---|
| **Bandwidth Aggregation** | รวม Bandwidth ของ Physical Link หลายเส้นเป็น Logical Link เดียว เช่น 2x1Gbps = 2Gbps, 4x10Gbps = 40Gbps |
| **Redundancy โดยไม่เสีย STP Block** | ถ้า Link สมาชิกเส้นใดเส้นหนึ่งขาด Traffic จะไหลไปที่เส้นที่เหลือทันที (Sub-second, ไม่ต้องรอ STP Convergence เพราะ STP มองเห็นแค่ Logical Interface เดียวที่ยัง Up อยู่) |
| **ลดจำนวน STP Instance/Port ที่ต้องคำนวณ** | STP คำนวณ Port-Channel เป็น 1 Interface ไม่ใช่คำนวณทุก Physical Link แยกกัน ลดภาระ CPU และความซับซ้อนของ Topology |

### หลักการพื้นฐาน

EtherChannel รวม Physical Interface (เรียกว่า **Member Link** หรือ **Member Port**) จำนวน 2-8 เส้น
(ขึ้นกับ Platform) ให้กลายเป็น **1 Logical Interface** เรียกว่า **Port-Channel (Po)** ทุก Layer
ที่อยู่เหนือ Physical Layer ขึ้นไป (Layer 2 Switching, Spanning-Tree, Layer 3 Routing) จะเห็นแค่
Interface `Port-channelX` เพียงตัวเดียว ไม่เห็น Physical Member Link ด้านล่างเลย

```
                    ┌─────────────────────────┐
    Layer 2/3       │   Interface Port-channel1 │   <- STP, VLAN, IP เห็นแค่นี้
    มองเห็นแค่นี้    │      (Logical Interface)   │
                    └────────────┬────────────┘
                                  │  รวมจาก
                    ┌─────────────┴─────────────┐
                    │                            │
              GigabitEthernet0/1          GigabitEthernet0/2
              (Member Link 1)              (Member Link 2)
                    │                            │
              ─────────────────  Physical Cable ─────────────────
```

> **ข้อกำหนดสำคัญ:** EtherChannel ปกติ (Standard/Regular EtherChannel) ทุก Member Link
> **ต้องเชื่อมระหว่าง Switch คู่เดียวกันเท่านั้น** เช่น Port-Channel บน CORE-SW1 ทุกเส้นต้องไปจบที่
> CORE-SW2 เพียงตัวเดียว จะเอาสายหนึ่งไปจบที่ CORE-SW2 อีกสายไปจบที่ DIST-SW1 แบบ Standard
> EtherChannel **ทำไม่ได้** — เรื่องนี้สำคัญมาก จะอธิบายลึกใน Step 53 และ Step 56

---

## Step 52 — EtherChannel Protocol: PAgP vs LACP

EtherChannel ต้องมี **Protocol เจรจา (Negotiation Protocol)** ระหว่างสองปลาย เพื่อให้ทั้งสองฝั่ง
ตกลงกันว่า "พอร์ตกลุ่มนี้จะรวมเป็น Channel เดียวกันนะ" และเพื่อตรวจสอบว่า Setting ของสมาชิกตรงกัน
ก่อนจะรวม (ป้องกัน Loop จาก Misconfiguration) มี 2 Protocol หลัก:

| Protocol | เจ้าของ | มาตรฐาน | คำย่อชื่อเต็ม | รองรับ Link สูงสุด |
|---|---|---|---|---|
| **PAgP** | Cisco (Proprietary) | ไม่ใช่มาตรฐานเปิด — ใช้ได้แค่ระหว่างอุปกรณ์ Cisco | Port Aggregation Protocol | 8 Active + 8 Hot-standby |
| **LACP** | IEEE | เปิด (Open Standard) — ใช้ข้าม Vendor ได้ | Link Aggregation Control Protocol, มาตรฐาน **IEEE 802.3ad** (ปัจจุบันรวมใน 802.1AX) | 8 Active + 8 Hot-standby |

> **ทำไมส่วนใหญ่เลือก LACP ในปัจจุบัน:** เพราะเป็น Open Standard ใช้ได้กับ Switch ต่าง Vendor
> (เช่น Cisco ↔ Arista, Cisco ↔ Juniper) และ Cisco เองก็แนะนำ LACP เป็น Default ในอุปกรณ์รุ่นใหม่
> ส่วน PAgP ยังมีใช้ในองค์กรที่มีอุปกรณ์ Cisco ทั้งหมดและ Config มาแบบเดิมนานแล้ว

### Mode ของแต่ละ Protocol

การรวม Channel เกิดจาก Mode ที่ตั้งไว้บน Interface ด้วยคำสั่ง `channel-group <number> mode <mode>`

| Protocol | Mode | คำสั่ง | ความหมาย |
|---|---|---|---|
| PAgP | **auto** | `channel-group 1 mode auto` | Passive — รอให้อีกฝั่งเริ่มเจรจาก่อน ตัวเองไม่ส่ง PAgP packet เริ่มต้น |
| PAgP | **desirable** | `channel-group 1 mode desirable` | Active — ส่ง PAgP packet เริ่มเจรจาเองทันที |
| PAgP | **on** | `channel-group 1 mode on` | บังคับรวม Channel แบบ Static ไม่มีการเจรจา Protocol เลย (ไม่ตรวจสอบความเข้ากันได้อัตโนมัติ) |
| LACP | **active** | `channel-group 1 mode active` | Active — ส่ง LACPDU (LACP Data Unit) เริ่มเจรจาเองทันที |
| LACP | **passive** | `channel-group 1 mode passive` | Passive — รอให้อีกฝั่งเริ่มเจรจาก่อน |
| LACP | **on** | `channel-group 1 mode on` | บังคับรวม Channel แบบ Static ไม่มีการเจรจา Protocol เลย |

### หลักจำง่ายๆ

- **auto/passive** = "รอ" (Passive) — ถ้าทั้งสองฝั่งเป็น Passive พร้อมกัน จะไม่มีใครเริ่มเจรจา → **ไม่รวม Channel**
- **desirable/active** = "เริ่มเอง" (Active)
- **on** = ไม่มี Protocol เจรจาเลย เป็นการบังคับด้วยมือ (Static) — เสี่ยงเกิด Loop สูงกว่า
  ถ้าอีกฝั่ง Config ผิดจะไม่มีการตรวจสอบใดๆ ก่อนเปิดใช้งาน (รายละเอียด Step 53)

> **Best Practice ของ Cisco:** ใช้ **LACP active-active** เสมอในทุก Production Network เพราะได้ทั้ง
> การตรวจสอบความเข้ากันได้อัตโนมัติ (Consistency Check) และการ Failover ที่รวดเร็วเมื่อ Link
> มีปัญหาแบบ Unidirectional (Protocol เจรจาต่อเนื่องตลอดเวลา ไม่ใช่แค่ตอนเริ่มต้น)

---

## Step 53 — Negotiation Mode Combination

นี่คือตารางที่ต้องจำให้ได้สำหรับข้อสอบ CCNA/CCNP: **Mode ฝั่ง A + Mode ฝั่ง B คู่ไหนรวม Channel
สำเร็จ คู่ไหนไม่สำเร็จ**

### ตาราง Combination ของ PAgP

| ฝั่ง A ＼ ฝั่ง B | auto | desirable | on |
|---|---|---|---|
| **auto** | ❌ ไม่รวม (ทั้งคู่ Passive รอกัน) | ✅ รวม | ❌ ไม่รวม |
| **desirable** | ✅ รวม | ✅ รวม | ❌ ไม่รวม |
| **on** | ❌ ไม่รวม | ❌ ไม่รวม | ✅ รวม (Static, ไม่มี Protocol ตรวจสอบ) |

### ตาราง Combination ของ LACP

| ฝั่ง A ＼ ฝั่ง B | passive | active | on |
|---|---|---|---|
| **passive** | ❌ ไม่รวม (ทั้งคู่ Passive รอกัน) | ✅ รวม | ❌ ไม่รวม |
| **active** | ✅ รวม | ✅ รวม | ❌ ไม่รวม |
| **on** | ❌ ไม่รวม | ❌ ไม่รวม | ✅ รวม (Static, ไม่มี Protocol ตรวจสอบ) |

### กฎสรุปที่ใช้ได้ทั้ง PAgP และ LACP

1. **Passive + Passive (auto+auto หรือ passive+passive) = ไม่รวม** เพราะไม่มีใครเริ่มส่ง Negotiation
   Packet ก่อน
2. **Active/Desirable อย่างน้อย 1 ฝั่ง = รวมได้** ตราบใดที่อีกฝั่งไม่ใช่ `on`
3. **`on` ต้องคู่กับ `on` เท่านั้น** — ถ้าฝั่งหนึ่งเป็น `on` (Static, ไม่มี Protocol) แต่อีกฝั่งใช้
   Protocol (auto/desirable/active/passive) จะ**ไม่รวม Channel** เพราะฝั่งที่เป็น `on` ไม่ส่ง/ไม่รับ
   Negotiation Packet เลย ส่วนอีกฝั่งที่รอ Packet จะไม่เห็นอะไรตอบมา
4. **ห้ามผสม PAgP กับ LACP ในคู่เดียวกัน** เช่น ฝั่ง A ใช้ PAgP desirable ฝั่ง B ใช้ LACP active
   จะไม่รวม Channel เพราะเป็นคนละ Protocol กันเลย (PAgP packet กับ LACPDU มี Format ต่างกัน)

```
ตัวอย่างที่รวมสำเร็จ:                          ตัวอย่างที่ไม่รวม (ควรระวัง):

CORE-SW1                CORE-SW2               CORE-SW1                CORE-SW2
mode active     <---->  mode active            mode auto       <---->  mode auto
       ✅ รวม Channel                                  ❌ ทั้งคู่รอกัน ไม่มีใครเริ่ม

CORE-SW1                CORE-SW2               CORE-SW1                CORE-SW2
mode active     <---->  mode passive           mode on          <---->  mode active
       ✅ รวม Channel (active เริ่มเจรจา)              ❌ on ไม่เจรจา Protocol เลย
                                                        → Port อีกฝั่งค้างรอ Negotiation ตลอดไป
```

> **สาเหตุที่ควรเลี่ยง `on` ใน Production:** ถ้า Cable ต่อผิด (เช่น Gi0/2 ของ CORE-SW1 ไปเสียบผิด
> Switch โดยไม่ได้ตั้งใจ) โหมด `on` จะไม่มีการตรวจสอบใดๆ เลยว่าอีกฝั่งคืออุปกรณ์อะไร รวม Channel
> ไปตรงๆ ตามที่ Config ไว้ ซึ่งอาจทำให้เกิด Bridging Loop ได้ทันทีถ้า Physical Topology ผิดจากที่ตั้งใจ
> ส่วน LACP/PAgP โหมด Active/Desirable จะ "คุย" กันก่อนเสมอ และจะไม่รวม Channel ถ้าตรวจพบว่า
> Setting ไม่ตรงกัน (ดู Step 56)

---

## Step 54 — Config LACP EtherChannel บน CORE-SW1 ↔ CORE-SW2 (Po1)

เราจะสร้าง EtherChannel ตัวแรกของหลักสูตรบน Link หลักที่ประกาศไว้ตั้งแต่ Part 1:
**CORE-SW1 ↔ CORE-SW2** โดยใช้ 2 Physical Link (GigabitEthernet0/1 และ GigabitEthernet0/2)
รวมเป็น **Port-channel1 (Po1)** ด้วย LACP mode active ทั้งสองฝั่ง

### Topology ของ Lab นี้

```
                 ┌─────────────────────┐        ┌─────────────────────┐
                 │      CORE-SW1        │        │      CORE-SW2        │
                 │                      │        │                      │
                 │   Gi0/1  ────────────┼────────┼────────  Gi0/1       │
                 │   Gi0/2  ────────────┼────────┼────────  Gi0/2       │
                 │                      │        │                      │
                 │   interface Po1      │        │   interface Po1      │
                 │   (LACP active)      │        │   (LACP active)      │
                 └─────────────────────┘        └─────────────────────┘
                        รวม Gi0/1 + Gi0/2 = Po1 (2 Gbps, Trunk ทุก VLAN)
```

### ขั้นตอนที่ 1 — ปิด (Shutdown) Physical Interface ก่อน Config เสมอ

Cisco แนะนำให้ `shutdown` Interface สมาชิกก่อน Config EtherChannel ทุกครั้ง เพื่อป้องกัน
Transient Loop ระหว่างที่ Config ยังไม่ครบทุกเส้น (โดยเฉพาะกรณีที่ Trunk Setting ต่างกันชั่วขณะ)

```
CORE-SW1(config)# interface range GigabitEthernet0/1 - 2
CORE-SW1(config-if-range)# shutdown
```

### ขั้นตอนที่ 2 — ตั้งค่า Trunk Setting บน Physical Interface ให้ตรงกันก่อนรวม (แนะนำ)

แม้ Trunk Setting จะสามารถตั้งที่ Port-channel Interface ทีเดียวได้ (แล้ว IOS จะ Copy ลงสมาชิก
อัตโนมัติ) แต่การตั้งที่ Physical Interface ก่อนช่วยให้เห็นภาพชัดเจนกว่าตอนเรียน:

```
CORE-SW1(config)# interface range GigabitEthernet0/1 - 2
CORE-SW1(config-if-range)# switchport trunk encapsulation dot1q
CORE-SW1(config-if-range)# switchport mode trunk
CORE-SW1(config-if-range)# switchport trunk allowed vlan 10,20,30,40,99
CORE-SW1(config-if-range)# switchport trunk native vlan 999
```

### ขั้นตอนที่ 3 — สร้าง Channel-Group ด้วย LACP mode active

```
CORE-SW1(config)# interface range GigabitEthernet0/1 - 2
CORE-SW1(config-if-range)# channel-group 1 mode active
Creating a port-channel interface Port-channel 1
CORE-SW1(config-if-range)# no shutdown
```

คำสั่ง `channel-group 1 mode active` จะสร้าง Interface `Port-channel1` ให้อัตโนมัติถ้ายังไม่มี
(สังเกต Log "Creating a port-channel interface Port-channel 1")

### ขั้นตอนที่ 4 — ตั้งค่า Interface Port-channel1 (ตั้งซ้ำอีกครั้งเพื่อความชัวร์ — Best Practice)

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# switchport trunk encapsulation dot1q
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99
CORE-SW1(config-if)# switchport trunk native vlan 999
CORE-SW1(config-if)# description ### LINK-TO-CORE-SW2-Po1 ###
CORE-SW1(config-if)# no shutdown
```

### Config เต็มฝั่ง CORE-SW1

```
CORE-SW1(config)# interface range GigabitEthernet0/1 - 2
CORE-SW1(config-if-range)# description ### MEMBER-OF-Po1-TO-CORE-SW2 ###
CORE-SW1(config-if-range)# switchport trunk encapsulation dot1q
CORE-SW1(config-if-range)# switchport mode trunk
CORE-SW1(config-if-range)# switchport trunk allowed vlan 10,20,30,40,99
CORE-SW1(config-if-range)# switchport trunk native vlan 999
CORE-SW1(config-if-range)# channel-group 1 mode active
CORE-SW1(config-if-range)# no shutdown
CORE-SW1(config-if-range)# exit
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# description ### LINK-TO-CORE-SW2-Po1 ###
CORE-SW1(config-if)# switchport trunk encapsulation dot1q
CORE-SW1(config-if)# switchport mode trunk
CORE-SW1(config-if)# switchport trunk allowed vlan 10,20,30,40,99
CORE-SW1(config-if)# switchport trunk native vlan 999
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# end
CORE-SW1# copy running-config startup-config
```

### Config เต็มฝั่ง CORE-SW2 (Mirror กัน — ต้องตรงทุก Parameter)

```
CORE-SW2(config)# interface range GigabitEthernet0/1 - 2
CORE-SW2(config-if-range)# description ### MEMBER-OF-Po1-TO-CORE-SW1 ###
CORE-SW2(config-if-range)# switchport trunk encapsulation dot1q
CORE-SW2(config-if-range)# switchport mode trunk
CORE-SW2(config-if-range)# switchport trunk allowed vlan 10,20,30,40,99
CORE-SW2(config-if-range)# switchport trunk native vlan 999
CORE-SW2(config-if-range)# channel-group 1 mode active
CORE-SW2(config-if-range)# no shutdown
CORE-SW2(config-if-range)# exit
CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# description ### LINK-TO-CORE-SW1-Po1 ###
CORE-SW2(config-if)# switchport trunk encapsulation dot1q
CORE-SW2(config-if)# switchport mode trunk
CORE-SW2(config-if)# switchport trunk allowed vlan 10,20,30,40,99
CORE-SW2(config-if)# switchport trunk native vlan 999
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# end
CORE-SW2# copy running-config startup-config
```

### Verify ทันทีหลัง Config

```
CORE-SW1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        M - not in use, minimum links not met
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:            1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P)    Gi0/2(P)
```

`Po1(SU)` แปลว่า Port-channel1 อยู่ในสถานะ **S**witched (Layer 2) และ **U**se (กำลังใช้งานอยู่จริง)
— นี่คือสถานะที่ถูกต้องสมบูรณ์ ส่วน `Gi0/1(P)` และ `Gi0/2(P)` แปลว่าทั้งสอง Physical Interface
**P**ถูก Bundle เข้า Port-channel เรียบร้อยแล้ว (รายละเอียด Flag ทั้งหมดอยู่ใน Step 59)

---

## Step 55 — Load-Balancing Algorithm ของ EtherChannel

### ทำไมต้องมี Load-Balancing Algorithm

EtherChannel **ไม่ได้** กระจาย Traffic แบบ Round-Robin ทีละ Packet ไปตาม Member Link สลับกัน
(เพราะจะทำให้ Packet มาถึงปลายทางไม่เรียงลำดับ - Out-of-Order Delivery ซึ่งเป็นปัญหาใหญ่กับ TCP)
แต่ใช้วิธี **Hashing** — นำค่าบางอย่างใน Header ของ Frame/Packet (เช่น MAC Address, IP Address,
Port Number) มาคำนวณเป็นค่า Hash แล้ว "ยัด" (Map) Flow นั้นไปที่ Member Link เส้นเดียวคงที่ตลอด
Session ทำให้ทุก Packet ของ Flow เดียวกันไปทางเดียวกันเสมอ (รักษาลำดับ Packet)

### คำสั่งตั้งค่า Algorithm (Global Config)

```
CORE-SW1(config)# port-channel load-balance ?
  dst-ip        Dst IP Addr
  dst-mac       Dst Mac Addr
  dst-port      Dst TCP/UDP Port
  src-dst-ip    Src XOR Dst IP Addr
  src-dst-mac   Src XOR Dst Mac Addr
  src-dst-port  Src XOR Dst TCP/UDP Port
  src-ip        Src IP Addr
  src-mac       Src Mac Addr
  src-port      Src TCP/UDP Port

CORE-SW1(config)# port-channel load-balance src-dst-ip
```

> **สำคัญ**: คำสั่งนี้เป็น **Global Config** มีผลกับ **ทุก Port-channel ใน Switch ตัวนั้น**
> ไม่สามารถตั้งแยกราย Port-channel ได้ (ต่างจาก Trunk Setting ที่ตั้งได้ต่อ Interface)

### ตาราง Algorithm ที่รองรับ (แตกต่างตาม Platform)

| Algorithm | คำอธิบาย | เหมาะกับ |
|---|---|---|
| `src-mac` | Hash จาก Source MAC Address | Traffic หลาย Source ไป Destination เดียว (เช่น Client หลายเครื่องคุยกับ Server ตัวเดียว) |
| `dst-mac` | Hash จาก Destination MAC Address | Traffic Source เดียวไปหลาย Destination |
| `src-dst-mac` | Hash จาก Source XOR Destination MAC | ดีกว่า 2 ตัวข้างต้นเพราะ Entropy สูงกว่า — **Default ของ Switch หลายรุ่น** |
| `src-ip` | Hash จาก Source IP Address | เหมาะกับ Routed Traffic ที่ Source หลากหลาย |
| `dst-ip` | Hash จาก Destination IP Address | เหมาะกับ Routed Traffic ที่ Destination หลากหลาย |
| `src-dst-ip` | Hash จาก Source XOR Destination IP | นิยมใช้บน Layer 3 Switch/Router — Entropy สูง |
| `src-port` / `dst-port` / `src-dst-port` | Hash จาก TCP/UDP Port Number | Entropy สูงสุด เหมาะกับ Traffic ที่มี Flow จำนวนมากระหว่าง IP คู่เดิม (เช่น Load Balancer, Data Center) |

### หลักการทำงานของ Hashing (แนวคิด)

```
Frame/Packet เข้ามา
      │
      ▼
คำนวณค่า Hash จาก Field ที่เลือก (เช่น src-ip XOR dst-ip)
      │
      ▼
ผลลัพธ์ Hash mod จำนวน Member Link ที่ Active
      │
      ▼
ได้เลข Index ของ Member Link ที่จะใช้ส่ง Flow นี้ → คงที่ตลอด Session
```

ตัวอย่างจำลอง (Switch มี 2 Member Link, ใช้ src-dst-ip):

| Flow | Source IP | Destination IP | ผล Hash (สมมติ) | ไปทาง Link |
|---|---|---|---|---|
| PC1 → Server1 | 10.10.10.11 | 10.10.30.11 | 0 | Gi0/1 |
| PC2 → Server1 | 10.10.10.12 | 10.10.30.11 | 1 | Gi0/2 |
| PC3 → Server1 | 10.10.10.13 | 10.10.30.11 | 0 | Gi0/1 |

### ทำไม Traffic "ดูไม่ Balance" เวลามีแค่ 2 Flow

นี่คือจุดที่ Engineer มือใหม่มักเข้าใจผิดว่า EtherChannel "เสีย" หรือ "Config ผิด" — ในความเป็นจริง
**Hashing เป็น Deterministic** (ผลลัพธ์คงที่สำหรับ Input เดิม) ถ้ามี Member Link 2 เส้น แต่มี Flow
Active อยู่แค่ 2 Flow เท่านั้น มีโอกาสสูงที่ค่า Hash ของทั้ง 2 Flow จะตกลงที่ Member Link **เส้นเดียวกัน**
พอดี ทำให้ดูเหมือน Link เส้นหนึ่งไม่มี Traffic วิ่งเลยแม้ Channel จะ Up ครบ 2 เส้นก็ตาม —
**ยิ่งมีจำนวน Flow ที่หลากหลายมากเท่าไหร่ (Source/Destination/Port ต่างกันมาก) การกระจายจะยิ่ง
สมดุลมากขึ้นตามสัดส่วนทางสถิติ** นี่คือเหตุผลที่แนะนำใช้ `src-dst-port` (ถ้า Platform รองรับ) ในระบบที่
มี Flow จำนวนมากอย่าง Data Center เพราะให้ Entropy สูงสุด

```
CORE-SW1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        src-dst-ip

EtherChannel Load-Balancing Addresses Used Per-Protocol:
Non-IP: Source XOR Destination MAC address
  IPv4: Source XOR Destination IP address
  IPv6: Source XOR Destination IP address
```

---

## Step 56 — ข้อกำหนดการรวมพอร์ตเข้า EtherChannel

ก่อน Physical Interface จะถูกรวมเข้า Port-channel ได้สำเร็จ **ทุก Member Link ต้อง Config
ตรงกันในพารามิเตอร์สำคัญ** ทั้งหมด ไม่ใช่แค่ทั้งสองฝั่งของ Link (CORE-SW1 กับ CORE-SW2) แต่รวมถึง
**Member Link เส้นอื่นในกลุ่มเดียวกันของ Switch ตัวเดียวกันด้วย** (Gi0/1 ต้อง Match กับ Gi0/2 บน
CORE-SW1 เอง)

### ตารางพารามิเตอร์ที่ต้องตรงกันทุกเส้น

| พารามิเตอร์ | ต้องตรงกันหรือไม่ | หมายเหตุ |
|---|---|---|
| **Speed** | ✅ ต้องตรงกัน | 1Gbps กับ 1Gbps เท่านั้น ห้ามผสม 1G กับ 10G ในกลุ่มเดียวกัน |
| **Duplex** | ✅ ต้องตรงกัน | ต้อง Full-Duplex ทั้งหมด (Half-Duplex ใช้ EtherChannel ไม่ได้เลยในทางปฏิบัติ) |
| **Switchport Mode** (Access/Trunk) | ✅ ต้องตรงกัน | ถ้าเส้นหนึ่งเป็น Access อีกเส้นเป็น Trunk จะไม่รวม |
| **Trunk Encapsulation** (dot1q/ISL) | ✅ ต้องตรงกัน | ปัจจุบันแทบไม่มี ISL แล้ว แต่ยังเป็นเหตุผลข้อสอบ CCNA |
| **Allowed VLAN List บน Trunk** | ✅ ต้องตรงกัน | ต่างกันได้ EtherChannel ก็ยังรวมได้ แต่ VLAN ที่ไม่ตรงกันจะไม่ Forward Traffic ข้าม (Best Practice ให้ตรงกันเป๊ะ) |
| **Native VLAN** | ✅ ต้องตรงกัน | ต่างกัน = เกิด Native VLAN Mismatch, IOS จะ Log Warning และเสี่ยง VLAN Leak |
| **Access VLAN** (ถ้าเป็น Access Port) | ✅ ต้องตรงกัน | ถ้าจะรวม Access Port เข้า Channel ต้องอยู่ VLAN เดียวกันทุกเส้น |
| **STP Port Cost/Priority ของสมาชิก** | ⚠️ ไม่จำเป็นต้องตรง (ตั้งได้เฉพาะที่ Port-channel Interface) | STP มองแค่ Port-channel Interface เดียว การตั้งที่ Physical Member จะถูกมองข้าม |
| **Channel-group Mode** (PAgP/LACP/on) | ✅ ต้องเข้ากันได้ตามตาราง Step 53 | ผิดตารางนี้ = ไม่รวม Channel เลย |

### สิ่งที่เกิดขึ้นเมื่อ Config ไม่ตรงกัน

**กรณีที่ 1 — ไม่ตรงกันแบบ Protocol ตรวจจับได้ (LACP/PAgP mode Active/Desirable):**
Port ที่ Config ผิดจะไม่ถูกรวมเข้า Channel และแสดงสถานะ `(s)` = **suspended** หรือ `(I)` =
**stand-alone** ใน `show etherchannel summary` (ยังทำงานเป็น Access/Trunk port เดี่ยวๆ ต่อไปได้
ไม่ Down ทั้ง Interface) พร้อม Syslog เตือน:

```
%EC-5-CANNOT_BUNDLE2: Gi0/2 is not compatible with Gi0/1 and will be suspended
(trunk mode of Gi0/2 is access, Gi0/1 is trunk)
```

**กรณีที่ 2 — Loop Guard ตรวจพบความเสี่ยง Loop จาก Misconfiguration (EtherChannel Misconfig
Guard):** ฟีเจอร์นี้ (Default เปิดอยู่บน Catalyst Switch ส่วนใหญ่) จะตรวจจับกรณีที่ฝั่งหนึ่ง Config
เป็น EtherChannel (เช่น mode on หรือ desirable) แต่อีกฝั่งไม่ได้ Config เป็น EtherChannel เลย
(เสียบสายผิด Switch, หรือลืม Config อีกฝั่ง) ซึ่งเสี่ยงเกิด Loop สูงมาก — Switch จะสั่ง
**Err-Disable ทั้ง Physical Port** ทันที:

```
%PM-4-ERR_DISABLE: channel-misconfig (STP) error detected on Gi0/2, putting Gi0/2 in err-disable state
%SPANTREE-2-CHNL_MISCFG: Detected an EtherChannel misconfiguration on Gi0/2
```

### ตรวจสอบและกู้คืน Port ที่ Err-Disable

```
CORE-SW1# show interfaces status err-disabled
Port      Name                    Status       Reason
Gi0/2                             err-disabled channel-misconfig (STP)

! หลังแก้ไข Config ให้ตรงกันแล้ว กู้คืน Port ด้วยวิธีใดวิธีหนึ่ง:
CORE-SW1(config)# interface GigabitEthernet0/2
CORE-SW1(config-if)# shutdown
CORE-SW1(config-if)# no shutdown

! หรือตั้ง Auto-Recovery (แนะนำใน Production):
CORE-SW1(config)# errdisable recovery cause channel-misconfig
CORE-SW1(config)# errdisable recovery interval 300
```

---

## Step 57 — Layer 3 EtherChannel (Routed Port-Channel) vs Layer 2 EtherChannel

### ความแตกต่างหลัก

| คุณสมบัติ | Layer 2 EtherChannel | Layer 3 EtherChannel |
|---|---|---|
| ใช้กับ | Switchport (Access/Trunk) | Routed Port (ไม่มี VLAN, ไม่ใช่ Switchport) |
| คำสั่งบน Physical Member | `switchport ...` ปกติ | `no switchport` ก่อน แล้วค่อย `channel-group` |
| IP Address | ตั้งที่ SVI (VLAN Interface) แยกหาก ต้องการ L3 | ตั้ง IP Address ตรงที่ Interface Port-channel เลย |
| ใช้งานเมื่อ | เชื่อมระหว่าง Switch ที่ต้องพก VLAN ข้าม (Trunk) | เชื่อมแบบ Point-to-Point ล้วนๆ ระหว่างอุปกรณ์ L3 สองตัว ไม่ต้องพก VLAN |
| ตัวอย่างในหลักสูตรนี้ | Po1 ระหว่าง CORE-SW1 ↔ CORE-SW2 (พก VLAN 10,20,30,40,99 ข้าม Core) | Link แบบ Routed ล้วนๆ เช่น ระหว่าง Core Switch กับ Router ที่ไม่ต้องการ Trunk |

### แนวคิด

Layer 3 EtherChannel คือการรวม Physical Link หลายเส้นเป็น Port-channel แบบเดียวกัน แต่ Interface
Port-channel ที่ได้ **ไม่ใช่ Switchport** — มันทำตัวเหมือน Routed Interface ของ Router ทั่วไป
คือรับ `ip address` ได้ตรงๆ ไม่ต้องผ่าน VLAN/SVI เหมาะกับ Link แบบ Point-to-Point ล้วนๆ ระหว่าง
อุปกรณ์ Layer 3 สองตัว (เช่นในอนาคตเมื่อเราต่อ CORE-SW ไปยัง WAN-EDGE Router แบบ Routed แทน Trunk)

### ตัวอย่าง Config (สมมติ Link Routed แยกต่างหาก บน Interface Gi0/3-Gi0/4 เป็น Po2)

**ขั้นตอนที่ 1 — ปิดสถานะ Switchport บน Physical Member ก่อน:**

```
CORE-SW1(config)# interface range GigabitEthernet0/3 - 4
CORE-SW1(config-if-range)# no switchport
CORE-SW1(config-if-range)# channel-group 2 mode active
CORE-SW1(config-if-range)# no shutdown
```

**ขั้นตอนที่ 2 — ตั้ง IP Address ที่ Port-channel Interface ตรงๆ:**

```
CORE-SW1(config)# interface Port-channel2
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.10.250.1 255.255.255.252
CORE-SW1(config-if)# description ### L3-EtherChannel-Routed-Link ###
CORE-SW1(config-if)# no shutdown
```

**Verify:**

```
CORE-SW1# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P)    Gi0/2(P)
2      Po2(RU)         LACP      Gi0/3(P)    Gi0/4(P)

CORE-SW1# show ip interface brief | include Port-channel
Port-channel1          unassigned      YES unset   up                    up
Port-channel2          10.10.250.1     YES manual  up                    up
```

สังเกต Flag `Po2(RU)` — **R** = Layer**3 (Routed)**, **U** = In Use ต่างจาก `Po1(SU)` ที่เป็น
**S**witched (Layer 2) ชัดเจน — คำสั่ง `show etherchannel summary` แยกให้เห็นทันทีว่า Port-channel
ไหนเป็น L2 หรือ L3

> **หมายเหตุสำหรับหลักสูตรนี้:** Link หลักระหว่าง CORE-SW1 ↔ CORE-SW2 (Po1) ในหลักสูตรนี้ยังคง
> เป็น **Layer 2 Trunk EtherChannel** ตามที่ประกาศไว้ตั้งแต่ Part 1 (ต้องพก VLAN ทั้งหมดข้าม Core
> เพื่อให้ DIST-SW ทุกตัวเห็น VLAN เดียวกันได้) ตัวอย่าง L3 EtherChannel ข้างต้นเป็นแนวคิดเสริมที่จะ
> นำไปใช้จริงตอนออกแบบ WAN Edge แบบ Routed ใน Part หลังๆ ของหลักสูตร

---

## Step 58 — LACP Fast Rate และ `port-channel min-links`

### LACP Rate — ความเร็วในการตรวจจับ Link ล่ม

Default ของ LACP คือส่ง **LACPDU (LACP Data Unit)** ทุก **30 วินาที** (Slow Rate) และจะถือว่า
Link ตายถ้าขาดการรับ LACPDU ไป 3 รอบติดกัน (= 90 วินาทีถึงจะรู้ตัว) ซึ่งช้าเกินไปสำหรับ Production
Network ที่ต้องการ Failover รวดเร็ว จึงมีคำสั่งเร่งความเร็วเป็น **Fast Rate = ส่งทุก 1 วินาที**
(ถือว่าตายถ้าขาด 3 รอบ = 3 วินาทีถึงจะรู้ตัว)

```
CORE-SW1(config)# interface range GigabitEthernet0/1 - 2
CORE-SW1(config-if-range)# lacp rate fast
```

> **ข้อสำคัญ:** ต้องตั้ง `lacp rate fast` ที่**ทั้งสองฝั่ง**ของ Link (CORE-SW1 และ CORE-SW2)
> เพราะแต่ละฝั่งควบคุมความเร็วการ**ส่ง**ของตัวเองเท่านั้น ถ้าตั้งฝั่งเดียว อีกฝั่งจะยังส่งช้าที่ 30
> วินาทีต่อไป

**Verify:**

```
CORE-SW1# show lacp internal
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode       P - Device is in Passive mode

Channel group 1
                            LACP port     Admin     Oper    Port        Port
Port      Flags   State     Priority      Key       Key     Number      State
Gi0/1     SA      bndl      32768         0x1       0x1     0x102       10101100
Gi0/2     SA      bndl      32768         0x1       0x1     0x103       10101100
```

หากตั้ง `lacp rate fast` แล้ว Flag จะเปลี่ยนจาก `S` (Slow) เป็น `F` (Fast) ทันที

### `port-channel min-links` — กำหนด Bandwidth ขั้นต่ำที่ยอมรับได้

โดย Default EtherChannel จะยังทำงานต่อไปได้แม้เหลือ Member Link แค่เส้นเดียว (Bandwidth ลดลง
แต่ไม่ Down ทั้ง Port-channel) ซึ่งในบางกรณี (เช่น SLA กำหนดว่าต้องมี Bandwidth ขั้นต่ำเท่านี้เสมอ)
เราต้องการให้ **Port-channel ทั้งเส้น Down ไปเลย** ถ้า Bandwidth ต่ำกว่าเกณฑ์ เพื่อให้ Routing
Protocol หรือ HSRP เปลี่ยนไปใช้เส้นทางสำรองอื่นแทนที่จะพยายามส่ง Traffic ผ่าน Link ที่เหลือน้อยเกินไป

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# port-channel min-links 2
```

Config นี้แปลว่า **Po1 ต้องมี Member Link Active อย่างน้อย 2 เส้นเสมอ** ถ้าเหลือ 1 เส้น (เช่น
Gi0/2 ขาด) Port-channel1 ทั้งเส้นจะเปลี่ยนสถานะเป็น **Down** ทันที (แม้ Gi0/1 จะยัง Up อยู่ก็ตาม)

```
CORE-SW1# show etherchannel port-channel

                Channel-group listing:
                -----------------------

Group: 1
----------
Port-channels in the group:
----------------------

Port-channel: Po1    (Primary Aggregator)

Age of the Port-channel   = 02d:14h:33m:12s
Logical slot/port   = 2/1          Number of ports = 1
GC                 = 0x00000000      HotStandBy port = null
Port state          = Port-channel Ag-Not-Inuse
Protocol            =   LACP
Port security        = Disabled

Ports in the Port-channel:
Index   Load   Port     EC state        No of bits
------+------+------+------------------+-----------
  0     00     Gi0/1    Active            0

Time since last port bundled:    00d:00h:00m:05s    Gi0/2
Time since last port Un-bundled: 00d:00h:00m:03s    Gi0/2

! ด้านบนคือตัวอย่างเมื่อ Gi0/2 หลุด และ min-links 2 ไม่ผ่าน
! สถานะ "Ag-Not-Inuse" = Aggregator ถูกสร้างแล้วแต่ไม่ได้ใช้งาน เพราะไม่ผ่านเกณฑ์ min-links
```

> **หมายเหตุ:** `min-links` มีประโยชน์มากที่สุดเมื่อ Bundle มี Member Link จำนวนมาก (4-8 เส้น)
> เพื่อตั้งเกณฑ์ Bandwidth ขั้นต่ำ เช่น Bundle 4 เส้น 10Gbps ตั้ง `min-links 2` (ยอมให้เหลือได้ถึง
> 2 เส้น = 20Gbps ขั้นต่ำ ถ้าต่ำกว่านั้น Down ทั้งหมด)

---

## Step 59 — Troubleshooting EtherChannel

### คำสั่ง Verify หลักที่ต้องใช้เป็นประจำ

| คำสั่ง | ใช้ตรวจสอบ |
|---|---|
| `show etherchannel summary` | ภาพรวมทุก Port-channel ใน Switch, สถานะ Bundle ของสมาชิกแต่ละเส้น |
| `show etherchannel port-channel` | รายละเอียดเจาะจง Port-channel ตัวเดียว รวม Timestamp การ Bundle/Un-bundle ล่าสุด |
| `show etherchannel <group> detail` | รายละเอียดเจาะลึกทุก Field ของ Group นั้น |
| `show lacp neighbor` | ดูข้อมูล LACP ของ**อุปกรณ์ฝั่งตรงข้าม** (Partner) ที่เจรจาด้วย |
| `show lacp internal` | ดูสถานะ LACP ของ**พอร์ตตัวเอง** (Rate, Priority, State bitmap) |
| `show pagp neighbor` | เทียบเท่า `show lacp neighbor` แต่สำหรับ PAgP |
| `show interfaces port-channel1` | สถานะ, Counter, Error ของ Interface Port-channel เหมือน Physical Interface ทั่วไป |
| `show spanning-tree interface port-channel1` | ดูว่า STP มองเห็น Po1 เป็น Port เดียว และ Role/State เป็นอะไร |

### อ่าน Flag ใน `show etherchannel summary` ให้เป็น (สำคัญที่สุดสำหรับข้อสอบ)

```
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        M - not in use, minimum links not met
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port
```

| Flag | ความหมาย | มักเจอเมื่อ |
|---|---|---|
| **D** | Down — Interface ตัวนี้ Down ทางกายภาพ | สาย Cable หลุด/พอร์ตปิด |
| **P** | Bundled — พอร์ตนี้ถูกรวมเข้า Port-channel เรียบร้อยแล้ว (สถานะปกติที่ต้องการเห็น) | Config ถูกต้องทั้งหมด |
| **I** | Stand-alone — พอร์ตทำงานแบบพอร์ตเดี่ยว ไม่ได้ถูก Bundle (แต่ยัง Up อยู่) | LACP/PAgP เจรจาไม่สำเร็จ (เช่น อีกฝั่งไม่ได้ Config Channel-group เลย) |
| **s** | Suspended — พอร์ตถูกพักไว้เพราะตรวจพบ Setting ไม่ตรงกับสมาชิกเส้นอื่น | Trunk mode/VLAN/Speed ไม่ตรงกัน (ดู Step 56) |
| **H** | Hot-standby — พอร์ตสำรอง (เกินจำนวน Active สูงสุด 8 เส้น) รอในกรณี Active หลุด | Bundle มีสมาชิกเกิน 8 เส้น (Max Active) |
| **S** | Layer2 (Switched) | Port-channel เป็น Switchport ปกติ |
| **R** | Layer3 (Routed) | Port-channel ตั้งแบบ `no switchport` |
| **U** | In use — Port-channel ตัวนี้กำลังทำงานอยู่จริง | สถานะปกติสมบูรณ์ |
| **M** | Not in use, minimum links not met | Member Link ที่ Active เหลือน้อยกว่าที่ตั้งใน `port-channel min-links` |
| **u** | Unsuitable for bundling | Hardware ไม่รองรับ (เช่น Speed ของพอร์ตนี้ผิดประเภทกับ ASIC ที่ใช้ทำ Bundle) |

**ตัวอย่างที่ต้องแปลผลให้ได้:**

```
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P)    Gi0/2(D)
```

→ แปลว่า Po1 ยังทำงานได้ (S=Layer2, U=In use) แต่ **Gi0/2 หลุด (D=Down)** ตอนนี้เหลือแค่ Gi0/1
รับ Traffic ทั้งหมด — ต้องไปเช็ค Physical Layer ของ Gi0/2 ต่อ (สาย, SFP, พอร์ตอีกฝั่ง)

```
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SD)         LACP      Gi0/1(I)    Gi0/2(I)
```

→ Po1 มีสถานะ **D = Down ทั้ง Port-channel**, สมาชิกทั้งสองเส้นเป็น **I = Stand-alone** (ไม่ถูก
Bundle เลย) — นี่คือสัญญาณว่า LACP เจรจากับฝั่งตรงข้ามไม่สำเร็จเลย ต้องไปเช็ค `show lacp neighbor`
ว่าเห็น Partner หรือไม่

### `show lacp neighbor` — ดูฝั่งตรงข้าม

```
CORE-SW1# show lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode       P - Device is in Passive mode

Channel group 1 neighbors

Partner's information:

          Partner             LACP Partner      Partner        Partner
Port       System ID           Port Number       Age          Flags     State
Gi0/1  32768,0011.2233.4455    0x102             150s          SA        bndl

          LACP Partner         Partner         Partner
Port      Port Priority        Oper Key        Port State
Gi0/1     32768                0x1              0x3D
```

ถ้า `show lacp neighbor` **ไม่แสดงข้อมูลอะไรเลย (Empty)** แปลว่า Switch ตัวนี้ไม่เคยได้รับ LACPDU
จากฝั่งตรงข้ามเลย — สาเหตุที่พบบ่อยที่สุดคือ **ฝั่งตรงข้ามยังไม่ได้ Config Channel-group เลย** หรือ
**Physical Link ไม่ Up** (ตรวจสอบด้วย `show interfaces status` ควบคู่กันเสมอ)

### `show pagp neighbor` — เทียบเท่าฝั่ง PAgP

```
CORE-SW1# show pagp neighbor
                Flags:  S - Device is sending Slow hello.  C - Device is in Consistent state.
                        A - Device is in Auto mode.         P - Device learns on physical port.

Channel group 1 neighbors

              Partner              Partner            Partner        Partner
Port          Name                 Device ID           Port           Age    Flags
Gi0/1         CORE-SW2             0011.2233.4455       Gi0/1          15s    SC
```

### Checklist การไล่ปัญหา EtherChannel ไม่รวม Channel (เรียงตามลำดับที่ควรเช็ค)

1. `show interfaces status` — Physical Link Up จริงหรือไม่ (connected/notconnect/disabled)
2. Mode Combination ตรงตาม Step 53 หรือไม่ (auto+auto จะไม่รวมเสมอ)
3. `show running-config interface <if>` เทียบทั้งสองฝั่ง — Speed/Duplex/Trunk/VLAN ตรงกันไหม
4. `show etherchannel summary` — Flag ของสมาชิกคืออะไร (P/I/s/D)
5. `show lacp neighbor` หรือ `show pagp neighbor` — เห็น Partner หรือไม่
6. `show logging | include EC-|CHNL|ERR_DISABLE` — มี Error Message อะไรบ้าง

---

## Step 60 — Lab เต็มรูปแบบ: EtherChannel บน CORE-SW1 ↔ CORE-SW2

Lab นี้สรุปทุกอย่างจาก Step 51-59 เข้าด้วยกัน สร้าง **Po1 แบบ LACP, 2 Member Link, Trunk พก
ทุก VLAN** ระหว่าง CORE-SW1 ↔ CORE-SW2 ให้สมบูรณ์พร้อม Verify แบบครบวงจร — Config นี้จะเป็น
Baseline ที่ Part ถัดๆ ไป (Part 10 Inter-VLAN Routing, Part 18 FHRP, Part 33-34 Advanced Campus)
จะใช้อ้างอิงต่อ

### Topology สรุป

```
              ┌────────────────────────────────────────────────────────┐
              │                                                          │
   ┌──────────┴───────────┐                                  ┌──────────┴───────────┐
   │      CORE-SW1          │                                  │      CORE-SW2          │
   │                        │                                  │                        │
   │  Gi0/1 ────────────────┼──────────────────────────────────┼──────────── Gi0/1     │
   │  Gi0/2 ────────────────┼──────────────────────────────────┼──────────── Gi0/2     │
   │                        │                                  │                        │
   │  interface Port-channel1                                   │  interface Port-channel1
   │  - LACP active                                              │  - LACP active
   │  - Trunk: VLAN 10,20,30,40,99                              │  - Trunk: VLAN 10,20,30,40,99
   │  - Native VLAN 999                                          │  - Native VLAN 999
   └────────────────────────┘                                  └────────────────────────┘
                Po1 = 2x GigabitEthernet = 2 Gbps รวม, Redundant, ไม่ถูก STP Block
```

### Full Running-Config — CORE-SW1

```
hostname CORE-SW1
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
vlan 999
 name NATIVE_UNUSED
!
interface GigabitEthernet0/1
 description ### MEMBER-OF-Po1-TO-CORE-SW2 ###
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
!
interface GigabitEthernet0/2
 description ### MEMBER-OF-Po1-TO-CORE-SW2 ###
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description ### LINK-TO-CORE-SW2-Po1 ###
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99
 switchport mode trunk
 port-channel min-links 1
 no shutdown
!
end
```

### Full Running-Config — CORE-SW2

```
hostname CORE-SW2
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
vlan 999
 name NATIVE_UNUSED
!
interface GigabitEthernet0/1
 description ### MEMBER-OF-Po1-TO-CORE-SW1 ###
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
!
interface GigabitEthernet0/2
 description ### MEMBER-OF-Po1-TO-CORE-SW1 ###
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99
 switchport mode trunk
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description ### LINK-TO-CORE-SW1-Po1 ###
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30,40,99
 switchport mode trunk
 port-channel min-links 1
 no shutdown
!
end
```

### Verification เต็มชุด — รันบน CORE-SW1

**1. `show etherchannel summary` — ต้องเห็น `Po1(SU)` และสมาชิกเป็น `(P)` ทั้งคู่**

```
CORE-SW1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        M - not in use, minimum links not met
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:            1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P)    Gi0/2(P)
```

**2. `show interfaces Port-channel1` — ดู Bandwidth รวมและสถานะ**

```
CORE-SW1# show interfaces Port-channel1
Port-channel1 is up, line protocol is up (connected)
  Hardware is EtherChannel, address is 0011.2233.4401 (bia 0011.2233.4401)
  Description: ### LINK-TO-CORE-SW2-Po1 ###
  MTU 1500 bytes, BW 2000000 Kbit/sec, DLY 10 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive set (10 sec)
  Full-duplex, 1000Mb/s, media type is unknown
  input flow-control is off, output flow-control is unsupported
  Members in this channel: Gi0/1 Gi0/2
  ARP type: ARPA, ARP Timeout 04:00:00
  Last input 00:00:02, output 00:00:01, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/2000/0/0 (size/max/drops/flushes); Total output drops: 0
  Queueing strategy: fifo
  5 minute input rate 128000 bits/sec, 95 packets/sec
  5 minute output rate 96000 bits/sec, 80 packets/sec
```

สังเกต **`BW 2000000 Kbit/sec` = 2 Gbps** — ยืนยันว่า Bandwidth ของ 2 Member Link (1Gbps x 2)
ถูกรวมกันจริงที่ Interface Port-channel1

**3. `show spanning-tree interface port-channel1` — ยืนยันว่า STP มองเห็นเป็น Port เดียว ไม่ Block**

```
CORE-SW1# show spanning-tree interface port-channel1

Vlan             Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
VLAN0010         Root FWD 3         128.1665 P2p
VLAN0020         Root FWD 3         128.1665 P2p
VLAN0030         Root FWD 3         128.1665 P2p
VLAN0040         Root FWD 3         128.1665 P2p
VLAN0099         Root FWD 3         128.1665 P2p
```

ทุก VLAN ที่ผ่าน Trunk มี Role = **Root**, State = **FWD (Forwarding)** — ไม่มี VLAN ไหนถูก
**Blocking** เลย เพราะ STP มองเห็น Po1 เป็น Interface เดียว (ไม่ใช่ 2 Physical Link แยกกัน)

**4. `show lacp neighbor` — ยืนยันเจรจา LACP กับ CORE-SW2 สำเร็จ**

```
CORE-SW1# show lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode       P - Device is in Passive mode

Channel group 1 neighbors

Partner's information:

          Partner             LACP Partner      Partner        Partner
Port       System ID           Port Number       Age          Flags     State
Gi0/1  32768,0011.2233.4402    0x101             320s          SA        bndl
Gi0/2  32768,0011.2233.4402    0x102             320s          SA        bndl
```

**5. ทดสอบ Redundancy — ถอด Cable Gi0/1 ออก (จำลอง Link ล่ม)**

```
! Log ที่เห็นบน CORE-SW1 ทันทีที่ Gi0/1 หลุด
%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to down
%EC-5-UNBUNDLE: Interface GigabitEthernet0/1 left the port-channel Port-channel1

CORE-SW1# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(D)    Gi0/2(P)
```

Port-channel1 **ยังคง Up (SU)** ต่อไปแม้ Gi0/1 หลุด (เพราะ min-links ตั้งไว้แค่ 1) — Traffic
ที่กำลังไหลอยู่ย้ายไปที่ Gi0/2 ทันทีในระดับ Sub-second โดยที่ STP หรือ Routing Protocol ชั้นบน
**ไม่รู้สึกตัวด้วยซ้ำ** ว่ามี Link หลุด เพราะเห็นแค่ Interface Port-channel1 เพียงตัวเดียวที่ยัง Up
อยู่ — นี่คือประโยชน์ของ EtherChannel ตามที่อธิบายไว้ใน Step 51 ที่พิสูจน์ได้จริงในการทดสอบนี้

**6. เสียบ Cable กลับคืน — ตรวจสอบว่า Gi0/1 กลับเข้า Channel อัตโนมัติ**

```
%LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to up
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to up
%EC-5-BUNDLE: Interface GigabitEthernet0/1 joined port-channel Port-channel1

CORE-SW1# show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P)    Gi0/2(P)
```

Lab นี้ยืนยันครบทั้ง 3 คุณสมบัติหลักของ EtherChannel: **Bandwidth Aggregation** (2Gbps), **STP
ไม่ Block** (ทุก VLAN Forwarding), และ **Redundancy อัตโนมัติ** (Failover/Recovery โดยไม่ต้อง
Config เพิ่ม)

---

## แบบฝึกหัดทวนความเข้าใจ Part 6

1. อธิบายว่าทำไม EtherChannel ทำให้ STP ไม่ Block Link สำรอง ในขณะที่ปกติ STP จะ Block Link
   คู่ขนานระหว่าง Switch สองตัวเสมอ
2. ถ้า CORE-SW1 ตั้ง `channel-group 1 mode passive` (LACP) และ CORE-SW2 ตั้ง `channel-group 1
   mode passive` เช่นกัน จะเกิดอะไรขึ้น เพราะเหตุใด
3. คำสั่งใดใช้เปลี่ยน Load-Balancing Algorithm ของ EtherChannel เป็น Source XOR Destination IP
   Address และคำสั่งนี้มีผลกับ Port-channel กี่ตัวในเครื่อง
4. Flag `Gi0/2(s)` ใน `show etherchannel summary` หมายถึงอะไร และสาเหตุที่พบบ่อยที่สุดคืออะไร
5. ทำไม EtherChannel มาตรฐาน (Regular EtherChannel) ระหว่าง DIST-SW1 จึงไม่สามารถมี Member
   Link เส้นหนึ่งไปจบที่ CORE-SW1 และอีกเส้นไปจบที่ CORE-SW2 ได้ ต้องใช้เทคโนโลยีอะไรถึงจะทำได้
   และจะได้เรียนใน Part ไหน

**เฉลย:**

1. เพราะ EtherChannel รวม Physical Link หลายเส้นให้เป็น Logical Interface (Port-channel) เดียว
   STP มองเห็นแค่ Interface เดียวที่เชื่อมระหว่าง Switch สองตัว ไม่เห็น Physical Link ย่อยด้านใน
   จึงไม่มี Loop ให้ต้อง Block เลยตั้งแต่แรก (Bandwidth ของทุกเส้นถูกใช้งานจริงพร้อมกัน)
2. จะ**ไม่รวม Channel** เพราะ Passive Mode คือ "รอให้อีกฝั่งเริ่มเจรจาก่อน" ถ้าทั้งสองฝั่งเป็น
   Passive พร้อมกัน จะไม่มีใครส่ง LACPDU เริ่มต้นเลย ทำให้ค้างอยู่ที่สถานะ Stand-alone (I) ตลอดไป
   ต้องมีอย่างน้อยฝั่งหนึ่งเป็น `active`
3. คำสั่ง `port-channel load-balance src-dst-ip` — เป็นคำสั่ง Global Config จึงมีผลกับ
   **ทุก Port-channel ในเครื่องนั้น** ไม่สามารถตั้งแยกเฉพาะ Port-channel ใด Port-channel หนึ่งได้
4. `(s)` = Suspended แปลว่า Interface นี้ถูกพักไว้ ไม่ได้ถูกรวมเข้า Port-channel เพราะตรวจพบว่า
   Setting ไม่ตรงกับสมาชิกเส้นอื่นในกลุ่ม (เช่น Trunk mode, Allowed VLAN, Native VLAN, Speed/Duplex
   ไม่ตรงกัน) — สาเหตุที่พบบ่อยที่สุดคือ Trunk Setting หรือ Allowed VLAN List ไม่เหมือนกันทั้งสองฝั่ง
5. เพราะ Standard/Regular EtherChannel กำหนดว่า Member Link ทุกเส้นต้องเชื่อมไปจบที่ **Switch
   ตัวเดียวกันเท่านั้น** (มองจาก Physical Layer เป็นสาย 2 เส้นที่ต้องไปสิ้นสุดที่ Chassis เดียวกัน
   ของอีกฝั่ง) การกระจาย Member Link ไปคนละ Switch (CORE-SW1 และ CORE-SW2) ต้องใช้เทคโนโลยีที่ทำให้
   สอง Switch นั้นถูกมองเป็น "Chassis เดียวกัน" ในทาง Control Plane เช่น **StackWise Virtual (SVL)**
   หรือ **VSS (Virtual Switching System)** ซึ่งจะทำให้เกิด **Multichassis EtherChannel (MEC)** ได้
   — เรื่องนี้จะเรียนละเอียดใน **Part 34 — StackWise / VSS / Multichassis EtherChannel**

---

## สรุป Part 6

Part นี้เราเรียนรู้ **EtherChannel** อย่างครบวงจร ตั้งแต่เหตุผลที่ต้องใช้ (แก้ปัญหา Bandwidth
สูญเปล่าจาก STP Block), Protocol เจรจาทั้ง PAgP และ LACP พร้อม Mode Combination ที่ต้องจำ,
การ Config จริงบน CORE-SW1 ↔ CORE-SW2 ด้วย LACP active-active, Load-Balancing Algorithm และ
หลักการ Hashing, ข้อกำหนดการรวมพอร์ตและผลของการตั้งค่าไม่ตรงกัน (Err-Disable), ความแตกต่างของ
Layer 2 กับ Layer 3 EtherChannel, การปรับแต่ง LACP Rate และ min-links, จนถึงการ Troubleshoot
ด้วย `show etherchannel summary`/`show lacp neighbor` และ Lab เต็มรูปแบบที่พิสูจน์ Redundancy
จริงด้วยการถอด-เสียบ Cable ✅ Link หลักของ Topology (CORE-SW1 ↔ CORE-SW2) ตอนนี้เป็น **Po1
แบบ LACP 2x1Gbps** พร้อมใช้งานเป็น Backbone ให้ Part ถัดๆ ไปแล้ว

**พร้อมสำหรับ Part 7**: เราจะวางพื้นฐานที่สำคัญที่สุดของ Layer 3 ทั้งหมด — **IPv4 Addressing
& Subnetting** ตั้งแต่โครงสร้าง Binary/Decimal, VLSM, Subnetting แบบ Fast Technique, จนถึงการ
วางแผน IP Address ให้ครบทุก VLAN ใน Lab Topology ของหลักสูตรนี้

**ไปต่อ:** [Part 7 — IPv4 Addressing & Subnetting →](part-007-ipv4-addressing-subnetting.md)
