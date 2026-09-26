# Part 84 — VXLAN/EVPN Fundamentals
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 831–840 จาก 1000**

> ต่อจาก [Part 83 — Enterprise Multicast Deep Dive](part-083-enterprise-multicast-deep-dive.md)
> ที่ปิดกลุ่มเนื้อหา Multicast ระดับ CCIE ด้วยการนำ PIM-SM/MVPN/IGMP ที่เรียนมาตั้งแต่
> [Part 41–42](part-041-multicast-fundamentals.md) ไปออกแบบและ Scale ทั้ง Campus แล้ว Part นี้จะ
> เปลี่ยนโจทย์ไปอีกทางหนึ่งโดยสิ้นเชิง — เราทิ้งคำสัญญาไว้ 2 จุดที่ต้องทำตามในวันนี้:
>
> 1. [Part 38](part-038-sd-access-fundamentals.md) Step 374 แนะนำ **VXLAN** แบบสรุปย่อในฐานะ
>    Data Plane ของ SD-Access เท่านั้น และบอกไว้ว่า **"เจาะลึกเต็มรูปแบบที่ Part 84–85 ระดับ CCIE"**
> 2. [Part 72](part-072-large-campus-fabric-design.md) Step 717 แนะนำ **Spine-Leaf** ในฐานะ
>    Topology ทางเลือกสำหรับพื้นที่ East-West Traffic หนัก และบอกไว้ว่า **"เจาะลึกที่ Part 84-85"**
>
> **Part นี้คือ Part ที่ทำตามสัญญาทั้งสองข้อ** — เราจะเรียน **VXLAN** (Data Plane Encapsulation)
> และ **EVPN** (Ethernet VPN, Control Plane ที่ใช้ MP-BGP) แบบเต็มรูปแบบ ซึ่งเป็นเทคโนโลยีที่ถูก
> ออกแบบมาสำหรับ **Data Center Fabric** เป็นหลัก (แม้ SD-Access จะหยิบ VXLAN มาใช้เป็น Data Plane
> ด้วยก็ตาม — รายละเอียดความต่างอยู่ที่ Step 839) เพื่อให้เห็นภาพ Lab ที่เป็นรูปธรรมชัดเจน **Part นี้
> จะประกาศอุปกรณ์ใหม่ 4 ตัว**: **DC-SPINE-1, DC-SPINE-2, DC-LEAF-1, DC-LEAF-2** — Data Center
> Fabric ขนาดเล็กที่ต่อพ่วงเข้ากับ Campus Lab เดิมผ่าน **CORE-SW1/CORE-SW2** (ดู Diagram เต็มที่
> Step 831 และ 840)
>
> **ข้อสังเกตสำคัญ**: DC Fabric ใหม่นี้ **ไม่ใช่** "Building 10" ที่ [Part 72](part-072-large-campus-fabric-design.md)
> Step 714 เคยพูดถึง (Building 10 เป็น Case Study ระดับ Capstone ขนาด Spine×2→Leaf×8 สำหรับฝึก
> ออกแบบ Campus ทั้งองค์กร) — ที่นี่เราสร้าง Fabric ขนาดเล็กมาก (Spine×2, Leaf×2) แยกเป็นเอกเทศ
> เพื่อใช้เป็น "ห้องทดลอง" สอนกลไก VXLAN/EVPN แบบจับต้องได้ทุกบรรทัด Config ไม่ปนกับความซับซ้อน
> ของ Capstone Design ใน Part 72
>
> **สำคัญสำหรับข้อสอบ CCIE Enterprise Infrastructure**: VXLAN/EVPN อยู่ใน Exam Blueprint หัวข้อ
> Data Center Interconnect และเป็นพื้นฐานที่ต้องเข้าใจก่อนเรียน **Multi-Site EVPN** (Border
> Gateway, Multi-Site Overlay) ที่ [Part 85](part-085-vxlan-evpn-multisite-advanced.md)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 831 | ทำไมต้องมี VXLAN — ข้อจำกัด VLAN (4094) เทียบกับ VNI (16 ล้าน) และทำไม STP-based L2 ไม่ scale ใน Spine-Leaf Fabric |
| 832 | VXLAN Encapsulation Format — Header, UDP Port 4789, MTU Overhead (50 byte) |
| 833 | VXLAN Flood-and-Learn (โมเดลดั้งเดิม) — Multicast Underlay สำหรับ BUM Traffic และข้อจำกัด |
| 834 | EVPN (Ethernet VPN) Control Plane — MP-BGP กระจาย MAC/IP Reachability แทน Flood-and-Learn |
| 835 | EVPN Route Type — Type 2 (MAC/IP), Type 3 (Inclusive Multicast), Type 5 (IP Prefix) |
| 836 | Basic VXLAN/EVPN Configuration บน DC-SPINE-1/2, DC-LEAF-1/2 — Underlay OSPF + `nve` + BGP EVPN |
| 837 | Anycast Gateway ใน VXLAN/EVPN — Gateway IP/MAC เดียวกันทุก Leaf |
| 838 | VXLAN/EVPN vs VLAN+STP+HSRP แบบดั้งเดิม — ตารางเปรียบเทียบเต็มรูปแบบ |
| 839 | จุดเชื่อมกับ SD-Access — ยืนยันว่า Data Plane คือ VXLAN ตัวเดียวกัน, LISP vs EVPN Control Plane |
| 840 | Lab เต็มรูปแบบ: VXLAN/EVPN L2 Fabric + Anycast Gateway + Host Mobility Test |

---

## Step 831 — ทำไมต้องมี VXLAN: ข้อจำกัดของ VLAN และ STP-based L2 ใน Spine-Leaf Fabric

### ปัญหาที่ 1: VLAN ID มีแค่ 12-bit

ย้อนกลับไป [Part 3](part-003-ethernet-switching-vlan.md) เราเรียนว่า 802.1Q Tag มีฟิลด์ VLAN ID
ขนาด **12-bit** เท่านั้น:

```
802.1Q Tag (4 byte):
┌───────────────┬─────┬─────┬────────────────┐
│ TPID (16 bit)  │ PCP │ DEI │ VLAN ID (12 bit)│
│ (0x8100)       │(3bit)│(1bit)│  0 - 4095       │
└───────────────┴─────┴─────┴────────────────┘
```

12-bit = 4096 ค่า แต่ VLAN 0 และ VLAN 4095 ถูกสงวนไว้ตาม Standard จึงเหลือใช้งานได้จริง
**4094 VLAN** — ฟังดูเยอะสำหรับ Campus ทั่วไป (หลักสูตรนี้ใช้แค่ 5 VLAN ตาม
[`00-ip-address-plan.md`](00-ip-address-plan.md)) แต่ **ไม่พอเลย** สำหรับ Data Center ที่ให้บริการ
**Multi-Tenant** จริง (Cloud Provider, Hosting, Enterprise DC ที่มีหลาย Business Unit/ลูกค้าเช่า
พื้นที่) เพราะ:

- ลูกค้า/Tenant 1 รายอาจต้องการหลาย VLAN (Web/App/DB Tier ละ VLAN)
- ถ้ามี Tenant หลักร้อย-หลักพันราย (Cloud-scale) → ต้องการ Segment มากกว่า 4094 แน่นอน
- แม้แต่ Enterprise DC ขนาดกลางที่ไม่ใช่ Cloud ก็มักชน Limit นี้เมื่อรวม VLAN ของทุก
  Application/Environment (Prod/Dev/Test/DR) เข้าด้วยกันในที่เดียว

**VXLAN (Virtual Extensible LAN)** แก้ปัญหานี้ด้วยฟิลด์ **VNI (VXLAN Network Identifier)**
ขนาด **24-bit** แทน VLAN ID:

| | VLAN (802.1Q) | VXLAN (VNI) |
|---|---|---|
| ขนาดฟิลด์ | 12-bit | 24-bit |
| จำนวน Segment สูงสุด | 4096 (ใช้จริง 4094) | 16,777,216 (16 ล้าน) |
| ขอบเขตการทำงาน | ต้อง Trunk ทะลุทุก Switch ที่เกี่ยวข้อง (L2 Adjacency จริง) | ห่อหุ้มเป็น IP Packet ธรรมดา ส่งข้าม Routed Underlay ได้ (L2-over-L3) |

### ปัญหาที่ 2: STP-based L2 ไม่ scale ใน Spine-Leaf Fabric

[Part 72](part-072-large-campus-fabric-design.md) Step 717 แนะนำ **Spine-Leaf** ไว้แบบสรุปสั้นๆ
ว่าเหมาะกับ Traffic แบบ East-West หนัก (Server-to-Server) — เหตุผลเชิงลึกที่ต้องเข้าใจตอนนี้คือ
**ถ้า Spine-Leaf ยังใช้ L2 แบบดั้งเดิม (VLAN Trunk + STP) มันจะ "เสีย" ข้อดีของ Topology นี้ไปเกือบ
ทั้งหมด**:

```
Spine-Leaf แบบ L2 ดั้งเดิม (VLAN Trunk ทุก Link + STP):

        DC-SPINE-1              DC-SPINE-2
         /        \              /        \
        /          \            /          \
   DC-LEAF-1 ══════════════ DC-LEAF-2
   (ทุก Link เป็น Trunk, STP มองเป็น Loop เดียวกันหมด)

STP เลือก Root Bridge 1 ตัว แล้ว "Block" Link ส่วนเกินเพื่อตัด Loop:
   - ถ้า DC-SPINE-1 เป็น Root → Link ไปยัง DC-SPINE-2 (หรือเส้นใดเส้นหนึ่ง) ถูก Block
   - แบนด์วิดท์ที่ซื้อมา (4 Uplink ระหว่าง 2 Spine + 2 Leaf) ใช้งานจริงได้แค่ครึ่งเดียว
   - Traffic ทุกเส้นที่ไม่ผ่าน Root ต้อง "เดินอ้อม" เพิ่ม Hop โดยไม่จำเป็น
```

ปัญหานี้คือปัญหาเดียวกับที่ [Part 5](part-005-stp-rstp-mstp.md) สอนไว้ (STP ต้อง Block Port ส่วนเกิน
เพื่อป้องกัน Loop) — แต่ใน Campus ทั่วไปที่ Uplink มีไม่กี่เส้น ผลกระทบยังพอรับได้ ส่วนใน **Spine-Leaf
DC Fabric ที่ต้องการ Bandwidth เต็มทุก Link แบบ Active-Active (ECMP) เสมอ** STP กลายเป็นตัวถ่วง
สถาปัตยกรรมทันที

**ทางออกที่ VXLAN/EVPN เลือก**: เปลี่ยน Spine-Leaf Underlay ทั้งหมดให้เป็น **Routed L3 Network
ธรรมดา** (ไม่มี VLAN Trunk ระหว่าง Spine-Leaf เลย ไม่มี STP เกี่ยวข้องเลย) แล้วใช้ **VXLAN
ห่อหุ้ม Layer 2 Frame เดิมของ Endpoint ให้เดินทางเป็น IP Packet ธรรมดาข้าม Routed Underlay
นี้แทน** — Underlay จึงใช้ **ECMP เต็มรูปแบบ** (ทุก Link Active พร้อมกัน, Load-balance ด้วย
Routing Protocol ธรรมดาอย่าง OSPF ตาม [Part 11](part-011-ospfv2-fundamentals.md)) โดยไม่ต้องมี
STP อยู่ในสมการเลยแม้แต่บรรทัดเดียว

### ประกาศ Lab ใหม่: Small Data Center Fabric

จากนี้ Part 84–85 จะใช้อุปกรณ์ใหม่ 4 ตัวนี้เป็น Lab หลัก:

| Hostname | Role | Platform แนะนำ | เชื่อมต่อ |
|---|---|---|---|
| DC-SPINE-1 | Spine Switch (Underlay Route Reflector ด้วย) | Cisco Nexus 9500 (NX-OS) | DC-LEAF-1, DC-LEAF-2, CORE-SW1 |
| DC-SPINE-2 | Spine Switch (Underlay Route Reflector ด้วย) | Cisco Nexus 9500 (NX-OS) | DC-LEAF-1, DC-LEAF-2, CORE-SW2 |
| DC-LEAF-1 | Leaf Switch (VTEP) | Cisco Nexus 9300 (NX-OS) | DC-SPINE-1, DC-SPINE-2, Server(s) |
| DC-LEAF-2 | Leaf Switch (VTEP) | Cisco Nexus 9300 (NX-OS) | DC-SPINE-1, DC-SPINE-2, Server(s) |

> **หมายเหตุเรื่อง Platform**: ตลอด Part 1–83 หลักสูตรนี้ใช้ Cisco IOS/IOS-XE เป็นหลัก (ตามที่
> [Part 1](part-001-networking-fundamentals.md) Step 10 ประกาศไว้) เพราะ Fabric VXLAN/EVPN ในโลก
> จริงส่วนใหญ่ Deploy บน **Cisco Nexus (NX-OS)** Part 84–85 จะใช้ **NX-OS CLI** สำหรับอุปกรณ์ DC
> ทั้ง 4 ตัวนี้โดยเฉพาะ — Syntax ต่างจาก IOS เล็กน้อย (เช่นต้องเปิด `feature` ก่อนใช้ฟีเจอร์เสมอ)
> จะอธิบายควบคู่ไปกับ Config จริงในแต่ละ Step

### Diagram ภาพรวม: DC Fabric ต่อพ่วงเข้ากับ Campus Lab เดิม

```
                    ┌─────────────────────────────────────────────┐
                    │        CAMPUS LAB เดิม (Part 1–83)             │
                    │   CORE-SW1 <===LACP Po1===> CORE-SW2           │
                    └──────┬─────────────────────────────┬─────────┘
                           │ 10.255.131.0/30              │ 10.255.131.4/30
                           │ (ใหม่ Part 84)                │ (ใหม่ Part 84)
                    ┌──────┴──────┐                 ┌──────┴──────┐
                    │  DC-SPINE-1  │═════════════════│  DC-SPINE-2  │  <- Spine (Underlay
                    └───┬─────┬───┘   (ไม่มี Link      └───┬─────┬───┘     + BGP Route
                        │     │        ตรงระหว่าง Spine                    Reflector)
                        │     │        ในดีไซน์นี้ — Leaf ทุกตัว
              ┌─────────┘     └─────────┐  ต่อทั้ง 2 Spine แทน = ECMP)
              │                          │
        ┌──────┴──────┐            ┌──────┴──────┐
        │  DC-LEAF-1   │            │  DC-LEAF-2   │  <- Leaf (VTEP)
        └──────┬───────┘            └──────┬───────┘
               │                            │
         SRV-DC-01                    SRV-DC-02
      (VLAN 3010/DC-SERVERS)      (VLAN 3010/DC-SERVERS)
```

สังเกตว่า **DC-LEAF-1 และ DC-LEAF-2 ไม่มี Link ตรงถึงกัน และ DC-SPINE-1/2 ก็ไม่มี Link ตรงถึงกัน
เช่นกัน** — นี่คือลักษณะเฉพาะของ Spine-Leaf แท้ๆ (Clos Fabric): ทุก Leaf ต่อกับ **ทุก** Spine
เสมอ (Full-Mesh ระหว่าง 2 Layer) แต่ไม่มี Link ระดับเดียวกัน (Leaf-to-Leaf หรือ Spine-to-Spine)
เลย — ทำให้ทุกคู่ Leaf ห่างกันแค่ **2 Hop เสมอ** (Leaf → Spine → Leaf) ไม่ว่า Fabric จะใหญ่ขึ้นแค่ไหน

---

## Step 832 — VXLAN Encapsulation Format: Header, UDP Port 4789, MTU Overhead

### แนวคิดหลัก: ห่อหุ้ม Frame เดิมทั้งดวงไว้ใน UDP/IP Packet ใหม่

VXLAN คือเทคนิค **MAC-in-UDP Encapsulation** — นำ Ethernet Frame เดิมของ Endpoint ทั้งดวง
(Destination MAC, Source MAC, 802.1Q Tag ถ้ามี, EtherType, Payload, ไม่รวม FCS เดิม) มาห่อหุ้ม
ไว้ภายใน UDP Datagram ใหม่ แล้วส่งข้าม **Underlay ที่เป็น Routed IP Network ธรรมดา**

### Full Packet Diagram

```
Frame เดิมของ Endpoint (Inner Frame — ไม่ถูกแก้ไขเลยแม้แต่ Byte เดียว):
┌────────────┬────────────┬─────────────┬──────────┬─────────────┐
│ Inner Dst   │ Inner Src   │ (802.1Q Tag  │ EtherType │  Payload    │
│ MAC (6B)    │ MAC (6B)    │  ถ้ามี, 4B)  │  (2B)     │ (46-1500B) │
└────────────┴────────────┴─────────────┴──────────┴─────────────┘

หลัง VXLAN Encapsulate ที่ VTEP ต้นทาง (เช่น DC-LEAF-1):
┌───────────┬──────────┬───────────┬───────────────────────┬─────────────────────────────┐
│ Outer Eth  │ Outer IP  │ Outer UDP  │      VXLAN Header      │   Inner Frame เดิมทั้งดวง    │
│ Header     │ Header    │ Header     │      (8 byte)           │   (ไม่แก้ไขเลย)              │
│ (14 byte)  │ (20 byte) │ (8 byte)   │                          │                              │
└───────────┴──────────┴───────────┴───────────────────────┴─────────────────────────────┘
  Dst/Src MAC   Src/Dst IP   Src Port=    Flags(8bit) +           = Ethernet Frame ต้นฉบับของ
  ของ Underlay  = VTEP↔VTEP  Random(Hash) VNI(24bit) + Reserved     Endpoint 100%
  Hop-by-hop    (Loopback0)  Dst Port=    (สังเกต: VNI 24-bit
                              4789/UDP     คือคำตอบของ Step 831)
```

### VXLAN Header (8 byte) แบบละเอียด

| ฟิลด์ | ขนาด | ความหมาย |
|---|---|---|
| Flags | 8 bit | Bit ที่สำคัญคือ **I-bit** (VXLAN Network ID present) ต้องเป็น 1 เสมอในการใช้งานจริง |
| Reserved | 24 bit | สงวนไว้ ต้องเป็น 0 |
| **VNI (VXLAN Network Identifier)** | **24 bit** | เลข Segment ID (เทียบเท่า VLAN ID แต่รองรับ 16 ล้านค่า) — ดู Step 831 |
| Reserved | 8 bit | สงวนไว้ ต้องเป็น 0 |

### UDP Port 4789 และเหตุผลที่ Source Port เป็นค่า "สุ่ม"

- **Destination Port = 4789/UDP** เสมอ (ตาม RFC 7348, IANA จองไว้ให้ VXLAN โดยเฉพาะ — ห้ามสับสน
  กับ Port ทดลองเก่าที่บาง Vendor เคยใช้ก่อนมี RFC)
- **Source Port** ของ Outer UDP **ไม่คงที่** — VTEP คำนวณจาก Hash ของ Inner Frame (เช่น Inner
  Src/Dst MAC, Inner Src/Dst IP, L4 Port ถ้ามี) เพื่อให้ Flow เดียวกันได้ Source Port เดิมเสมอ
  (Flow-consistent) แต่ Flow ต่างกันได้ Source Port ต่างกัน — **นี่คือกลไกที่ทำให้ Underlay ทำ
  ECMP Load-balance ได้ต่อ Flow** เพราะ Router/Switch ระหว่างทาง Hash 5-Tuple (Src/Dst IP,
  Protocol, Src/Dst Port) ตามปกติ ซึ่ง Source Port ที่สุ่มตาม Flow ทำให้แต่ละ Flow กระจายไปคนละ
  ECMP Path ได้ (ตรงกับที่ Step 831 ต้องการ — ECMP เต็มรูปแบบ)

### MTU Overhead: ทำไมต้องเปิด Jumbo Frame บน Underlay

รวม Overhead ที่ VXLAN เพิ่มเข้ามา:

| ส่วนที่เพิ่ม | ขนาด |
|---|---|
| Outer Ethernet Header | 14 byte |
| Outer IP Header | 20 byte |
| Outer UDP Header | 8 byte |
| VXLAN Header | 8 byte |
| **รวม Overhead** | **50 byte** |

ถ้า Endpoint ส่ง Frame ขนาดมาตรฐาน 1500 byte (MTU ปกติตาม [Part 1](part-001-networking-fundamentals.md)
Step 7) เท่าเดิม แต่ Underlay ยังตั้ง MTU ไว้ที่ 1500 byte เหมือนเดิม — Packet ที่ถูก VXLAN
Encapsulate แล้วจะมีขนาด **1550 byte** ซึ่ง**เกิน MTU ของ Underlay** ทำให้ถูก **Fragment**
(ถ้า Router ระหว่างทางยอมทำ) หรือถูก **Drop** ทันที (ถ้าตั้ง DF-bit หรือ Underlay ไม่รองรับ
Fragmentation ของ Encapsulated Traffic) — ทั้งสองกรณีคือ Performance Killer ที่ไม่ควรเกิดใน DC
Fabric จริง

**Best Practice**: ตั้งค่า **Jumbo MTU** (แนะนำ 9216 byte บน Nexus) ให้กับ**ทุก Interface ของ
Underlay** (Spine↔Leaf, Leaf↔Server ถ้าจะใช้ Jumbo กับ Server ด้วย) ก่อนเปิด VXLAN ใช้งานจริง:

```
! NX-OS: ตั้ง Jumbo MTU ระดับ System (ทำครั้งเดียว ต้องมีก่อน Interface MTU ถึงมีผล)
DC-SPINE-1(config)# system jumbomtu 9216

! ตั้ง MTU ต่อ Interface (ตัวอย่างที่ DC-SPINE-1 ขา DC-LEAF-1)
DC-SPINE-1(config)# interface Ethernet1/1
DC-SPINE-1(config-if)# mtu 9216
DC-SPINE-1(config-if)# no shutdown
```

> **สำหรับข้อสอบ CCIE**: จำเลข **50 byte** และ **1550 byte** ให้แม่น — เป็นตัวเลขที่ข้อสอบ
> Design ชอบถามในรูปแบบ "ถ้า Endpoint MTU 1500 และ Underlay ไม่ได้ปรับ MTU จะเกิดอะไรขึ้น"

---

## Step 833 — VXLAN Flood-and-Learn (โมเดลดั้งเดิม) — Multicast Underlay สำหรับ BUM Traffic

### VXLAN ดั้งเดิมตาม RFC 7348 ไม่มี Control Plane

VXLAN เวอร์ชันดั้งเดิม (ก่อนมี EVPN) **ไม่มี Control Plane แยกต่างหากเลย** — VTEP (VXLAN Tunnel
Endpoint คือ Fabric Edge/Leaf ที่ทำ Encapsulate/Decapsulate) เรียนรู้ตำแหน่งของ MAC Address
ปลายทางด้วยวิธีเดียวกับ Switch L2 ธรรมดาที่เรียนใน [Part 3](part-003-ethernet-switching-vlan.md):
**Data-Plane Learning** (ดูจาก Source MAC ของ Frame ที่ไหลผ่านเข้ามา) — เรียกโมเดลนี้ว่า
**Flood-and-Learn**

ปัญหาคือ: การ "Flood" ปกติของ Switch L2 (ส่ง Unknown-Unicast/Broadcast/Multicast ออกทุก Port
ยกเว้น Port ต้นทาง) **ใช้งานไม่ได้ตรงๆ** ข้าม VXLAN Fabric เพราะ Underlay เป็น Routed L3 Network
(ไม่มี Broadcast Domain เดียวกันข้าม Subnet) — VTEP จึงต้องหาวิธี "จำลอง Flood" ข้าม Underlay
ให้ได้ ซึ่งวิธีที่ RFC 7348 เลือกคือ **ใช้ IP Multicast**

### กลไก: 1 VNI = 1 Multicast Group ใน Underlay

```
VNI 10010 (DC-SERVERS) ผูกกับ Multicast Group 239.1.1.10 ใน Underlay

           ┌──────────────────────────────────────────┐
           │         Underlay ต้องรัน PIM-SM (Part 41-42) │
           │         เพื่อสร้าง Multicast Distribution Tree │
           │         ให้ครอบคลุมทุก VTEP ที่มี VNI 10010     │
           └──────────────────────────────────────────┘
                    ▲                              ▲
                    │ Join 239.1.1.10               │ Join 239.1.1.10
              ┌─────┴──────┐                  ┌─────┴──────┐
              │ DC-LEAF-1   │                  │ DC-LEAF-2   │
              │ (VTEP)      │                  │ (VTEP)      │
              └─────────────┘                  └─────────────┘

BUM Traffic (Broadcast/Unknown-Unicast/Multicast) จาก SRV-DC-01 (หลัง DC-LEAF-1):
1. DC-LEAF-1 Encapsulate BUM Frame ด้วย VXLAN, Outer Dst IP = 239.1.1.10 (Multicast)
2. Underlay (PIM-SM) Replicate Packet นี้ไปให้ทุก VTEP ที่ Join Group 239.1.1.10 ไว้
   (ในที่นี้คือ DC-LEAF-2) — เสมือนการ "Flood" แบบเดิมของ L2 Switch
3. DC-LEAF-2 รับ, Decapsulate, เห็น Source MAC ของ SRV-DC-01 → "เรียนรู้" ว่า
   MAC นี้อยู่หลัง VTEP ที่มี Outer Src IP = Loopback ของ DC-LEAF-1 (Data-Plane Learning
   เหมือน Switch L2 ปกติ แค่ "แหล่งที่มา" กลายเป็น Remote VTEP IP แทน Physical Port)
```

นี่คือเหตุผลที่ [Part 41–42](part-041-multicast-fundamentals.md), [Part 75](part-075-advanced-multicast-enterprise.md),
และ [Part 83](part-083-enterprise-multicast-deep-dive.md) ที่เรียน PIM-SM/IGMP/MVPN มาอย่าง
ละเอียด **มีประโยชน์ตรงนี้พอดี** — VXLAN Flood-and-Learn แบบดั้งเดิม **ต้องมี PIM-SM ทำงานอยู่
เต็มรูปแบบใน Underlay ทุกจุด** ก่อนจะเปิด VXLAN ใช้งานได้เลย

ตัวอย่าง Config NX-OS แบบ Flood-and-Learn (ใช้ Multicast ใน Underlay — **แสดงเพื่อการศึกษาเท่านั้น
Lab จริงของ Part นี้ (Step 836 เป็นต้นไป) จะไม่ใช้วิธีนี้**):

```
DC-LEAF-1(config)# feature pim
DC-LEAF-1(config)# interface nve1
DC-LEAF-1(config-if-nve)# member vni 10010
DC-LEAF-1(config-if-nve-vni)# mcast-group 239.1.1.10    ! <- ผูก VNI กับ Multicast Group
```

### ข้อจำกัดของ Flood-and-Learn ที่ทำให้ EVPN ถูกพัฒนาขึ้นมา

| ข้อจำกัด | รายละเอียด |
|---|---|
| **ต้องมี PIM-SM ทั่ว Underlay** | เพิ่ม Protocol ที่ต้อง Design/Deploy/Troubleshoot อีกชุด (RP Placement, RPF Check ฯลฯ ตามที่ [Part 75](part-075-advanced-multicast-enterprise.md) สอน) ทั้งที่ Underlay ควร "โง่และเรียบง่ายที่สุด" |
| **Unknown-Unicast ยังต้อง Flood ทุกครั้ง** | ถ้า VTEP ยังไม่เคย "เห็น" MAC ปลายทางมาก่อน (เช่นพึ่ง Reboot, Table หมดอายุ) Frame นั้นต้อง Flood ไปทุก VTEP ใน VNI เสมอ แม้ VTEP ปลายทางจริงมีแค่ 1 ตัว — สิ้นเปลือง Bandwidth ทั่ว Fabric |
| **ไม่มี ARP Suppression** | ARP Request (เป็น Broadcast) ต้อง Flood ข้าม Fabric ทุกครั้งที่ Host คุยกันครั้งแรก ไม่มีกลไกให้ VTEP "ตอบแทน" ท้องถิ่นได้เลย |
| **Convergence ช้าเมื่อ Endpoint ย้าย** | เมื่อ MAC ย้าย VTEP (VM Migration) VTEP เก่าไม่รู้ตัวจนกว่า MAC Aging Timer หมดอายุ และ VTEP อื่นต้องรอ BUM/Flood รอบใหม่ถึงจะ "เรียนรู้ใหม่" — ไม่มีการแจ้งเตือนแบบ Proactive |
| **ไม่มี Prefix-based Routing ในตัว** | Flood-and-Learn ทำได้แค่ L2 MAC Reachability เท่านั้น ไม่มีกลไกกระจาย IP Prefix สำหรับงาน Routed/L3 VXLAN (ที่ Step 835 Type 5 จะแก้) |
| **Scale จำกัดตามขนาด Multicast Tree** | จำนวน VNI ที่ Deploy ได้ผูกกับจำนวน Multicast Group ที่ Underlay รองรับได้จริง (Hardware Multicast Route Table มี Limit) |

> **สรุป**: Flood-and-Learn ใช้งานได้จริงและยังพบใน Deployment เก่า แต่มีข้อจำกัดทั้งด้าน
> Operational Complexity (ต้องมี Multicast) และ Scale/Convergence — **EVPN (Step 834)** ถูก
> พัฒนาขึ้นมาเพื่อแก้ปัญหาเหล่านี้โดยเปลี่ยนวิธี "เรียนรู้ MAC/IP" จาก Data Plane ไปเป็น **Control
> Plane ที่ใช้ BGP** แทน

---

## Step 834 — EVPN (Ethernet VPN) Control Plane: MP-BGP กระจาย MAC/IP Reachability

### แนวคิดหลัก: เปลี่ยนจาก "รอ Flood มาเรียนรู้" เป็น "ประกาศ Proactive ทันทีที่รู้"

**EVPN (Ethernet VPN)** คือการนำ **MP-BGP (Multiprotocol BGP)** ที่เรียนมาตั้งแต่
[Part 29–30](part-029-bgp-fundamentals.md) มาใช้เป็น **Control Plane ของ VXLAN Fabric** โดยเพิ่ม
Address Family ใหม่คือ **L2VPN EVPN** — แทนที่จะรอให้ Frame จริงไหลผ่านมาก่อนแล้วค่อย "เรียนรู้"
(Flood-and-Learn) VTEP แต่ละตัวจะ:

1. เรียนรู้ MAC (และ IP ถ้ามี) ของ Endpoint ท้องถิ่นด้วย Data-Plane Learning ตามปกติ (ไม่เปลี่ยน)
2. **ประกาศ (Advertise) MAC/IP นั้นออกไปทันทีผ่าน BGP UPDATE Message** ให้ VTEP อื่นทุกตัวใน
   Fabric รู้ **ก่อนที่จะมี Traffic จริงไหลไปหาปลายทางนั้นด้วยซ้ำ**

นี่คือความต่างพื้นฐานที่สุด: Flood-and-Learn เป็นกลไก **Reactive/Data-Plane-Driven** (ต้องมี
Traffic ก่อนถึงเรียนรู้) ส่วน EVPN เป็นกลไก **Proactive/Control-Plane-Driven** (ประกาศล่วงหน้า
ผ่าน BGP โดยไม่ต้องพึ่ง Traffic จริงเลย) — เป็นกลไกที่ **แตกต่างโดยสิ้นเชิง** จาก Flood-and-Learn
และดีกว่าในแทบทุกมิติตามที่ Step 833 สรุปข้อจำกัดไว้

### สิ่งที่ EVPN "ตัดออก" ได้: Multicast ใน Underlay (สำหรับ Control-Plane Learning)

เพราะ MAC/IP Reachability ทั้งหมดกระจายผ่าน BGP UPDATE (ซึ่งวิ่งบน TCP Port 179 แบบ Unicast
ปกติที่ [Part 29](part-029-bgp-fundamentals.md) สอนไว้) **Underlay ไม่จำเป็นต้องรัน PIM-SM เพื่อ
การเรียนรู้ MAC อีกต่อไป** — ตัด Protocol ทั้งชุดที่ Step 833 ระบุว่าเป็นภาระออกไปได้ทันที (ส่วน
BUM Traffic ที่ยังต้อง "กระจาย" ไปหลายจุดจริงๆ เช่น ARP/DHCP Broadcast จะใช้กลไกใหม่ที่ไม่ต้องพึ่ง
PIM ก็ได้ — ดู Type 3 Route ที่ Step 835 และ Ingress Replication ที่ Step 836)

### Full Architecture Diagram

```
                        ┌──────────────────────────────────────────┐
                        │   iBGP AS 65001 (ใช้ AS เดียวกับ Enterprise │
                        │   ตาม Part 29 Step "WAN-EDGE ใช้ AS 65001") │
                        └──────────────────────────────────────────┘

              DC-SPINE-1                              DC-SPINE-2
           (BGP Route Reflector)                  (BGP Route Reflector)
           Loopback0: 1.1.1.31                     Loopback0: 1.1.1.32
                 │        ▲                              │        ▲
                 │        │  iBGP L2VPN EVPN              │        │  iBGP L2VPN EVPN
                 │        │  (Route Reflector Client)     │        │
                 ▼        │                              ▼        │
           ┌─────┴────────┴──┐                      ┌─────┴────────┴──┐
           │   DC-LEAF-1      │◄──── VXLAN Tunnel ──►│   DC-LEAF-2      │
           │   Loopback0:     │      (Data Plane,    │   Loopback0:     │
           │   1.1.1.41       │       Step 832)       │   1.1.1.42       │
           │   (VTEP)         │                      │   (VTEP)         │
           └────────┬─────────┘                      └────────┬─────────┘
                    │                                          │
              SRV-DC-01                                  SRV-DC-02
         (MAC 0050.56aa.1001)                        (MAC 0050.56aa.1002)

ขั้นตอน:
1. SRV-DC-01 ส่ง Frame แรกเข้า DC-LEAF-1 → DC-LEAF-1 เรียนรู้ MAC 0050.56aa.1001 (Local, Data-Plane)
2. DC-LEAF-1 ประกาศ EVPN Type 2 Route: "MAC 0050.56aa.1001 อยู่หลังฉัน (Next-Hop=1.1.1.41)"
   ผ่าน iBGP ไปยัง DC-SPINE-1/2 (Route Reflector)
3. DC-SPINE-1/2 Reflect Route นี้ต่อไปยัง DC-LEAF-2 ทันที (ตาม Route Reflector concept
   ที่เรียนใน Part 29-30)
4. DC-LEAF-2 ได้รู้จัก MAC 0050.56aa.1001 (Remote, ผ่าน BGP) "ก่อน" ที่ SRV-DC-02 จะส่ง Frame
   ไปหา SRV-DC-01 ด้วยซ้ำ — เมื่อ SRV-DC-02 ส่ง Frame จริง DC-LEAF-2 รู้ทันทีว่าต้อง Encapsulate
   VXLAN ส่งตรงไปยัง Outer Dst IP = 1.1.1.41 (RLOC/VTEP ของ DC-LEAF-1) แบบ Unicast ล้วนๆ
   ไม่ต้อง Flood เลย
```

### ตารางเปรียบเทียบ Flood-and-Learn vs EVPN

| หัวข้อ | Flood-and-Learn (Step 833) | EVPN (MP-BGP) |
|---|---|---|
| วิธีเรียนรู้ MAC/IP | Data-Plane (ต้องมี Traffic ไหลผ่านก่อน) | Control-Plane (BGP ประกาศล่วงหน้า) |
| Multicast ใน Underlay | **ต้องมี** (PIM-SM ต่อ VNI) | **ไม่ต้องมี** (ใช้ Ingress Replication ได้ — Step 836) |
| ARP Suppression | ไม่มี | **มี** (Type 2 Route พก IP ไปด้วย — Step 835) |
| ความเร็วในการรู้ตำแหน่ง Endpoint ใหม่ | รอ Flood รอบถัดไป | ทันทีที่ BGP UPDATE ไปถึง (วินาทีเดียว) |
| Multi-Tenancy / Route-Target Isolation | ไม่มีในตัว | มีในตัว (RD/RT ตามมาตรฐาน BGP VPN — คล้าย MPLS L3VPN) |
| Protocol ที่ต้อง Deploy เพิ่ม | PIM-SM ทั่ว Underlay | BGP (ซึ่ง Enterprise ส่วนใหญ่มีทักษะอยู่แล้วจาก Part 29-30) |

> **สรุปสำหรับข้อสอบ CCIE**: คำถามแนว "VXLAN กับ EVPN สัมพันธ์กันอย่างไร" ให้ตอบว่า **VXLAN คือ
> Data Plane (วิธี Encapsulate), EVPN คือ Control Plane (วิธีกระจาย Reachability)** — ใช้
> VXLAN โดยไม่มี EVPN ได้ (Flood-and-Learn) แต่ Deployment สมัยใหม่แทบทั้งหมดใช้คู่กัน

---

## Step 835 — EVPN Route Type: Type 2 (MAC/IP), Type 3 (Inclusive Multicast), Type 5 (IP Prefix)

EVPN NLRI (Network Layer Reachability Information) ที่วิ่งอยู่ใน BGP UPDATE ไม่ได้มีแบบเดียว —
RFC 7432 กำหนด **Route Type** หลายแบบ แต่ละแบบสื่อความหมายต่างกัน ตาราง 3 Type ที่สำคัญที่สุด
สำหรับ VXLAN/EVPN Fabric พื้นฐาน (Type 1 และ Type 4 สำหรับ Multi-Homing/ESI จะเรียนที่
[Part 85](part-085-vxlan-evpn-multisite-advanced.md)):

| Route Type | ชื่อเต็ม | หน้าที่ | บรรจุข้อมูลอะไร | ใช้เมื่อ |
|---|---|---|---|---|
| **Type 2** | MAC/IP Advertisement | ประกาศ MAC Address (และ IP ถ้ามี) ของ Endpoint แต่ละตัวให้ VTEP อื่นรู้ | RD, ESI, Ethernet Tag, MAC Address, IP Address (optional), L2 VNI, L3 VNI (optional), Route-Target | ทุกครั้งที่ VTEP เรียนรู้ MAC/IP ใหม่ของ Endpoint ท้องถิ่น — เป็น Route Type ที่ใช้บ่อยที่สุด |
| **Type 3** | Inclusive Multicast Ethernet Tag | ประกาศว่า "ฉันเป็นสมาชิกของ VNI นี้ พร้อมรับ BUM Traffic" — ใช้สร้างรายชื่อ VTEP ปลายทางสำหรับ Ingress Replication (หรือ Multicast Group ถ้าเลือกใช้) | RD, Ethernet Tag, Originating Router IP, PMSI Tunnel Attribute (บอกวิธีส่ง BUM: Ingress Replication หรือ Multicast) | ทุก VTEP ที่มี VNI นั้น Active ต้องประกาศเสมอ (แม้ยังไม่มี Endpoint จริง) เพื่อให้ VTEP อื่นรู้ว่าต้องส่ง BUM มาหาด้วย |
| **Type 5** | IP Prefix Route | ประกาศ IP Prefix (Subnet) แทนที่จะประกาศทีละ Host — ใช้ในดีไซน์ **Routed VXLAN / L3-only** ที่ไม่ต้องการ Stretch L2 ข้าม Site | RD, ESI, Ethernet Tag, IP Prefix, GW IP, L3 VNI, Route-Target | ดีไซน์ที่ Routing ล้วน (ไม่มี L2 Adjacency ข้าม Leaf) หรือ Summarize Host Route จำนวนมากเป็น Prefix เดียวเพื่อ Scale (Type-5-only Design) |

### ทำไม Type 3 ยังจำเป็นแม้จะเลิกใช้ Multicast ของ Underlay ไปแล้ว (Step 834)

จุดที่มักสับสน: EVPN "ตัด Multicast **ของ Underlay** ออกจากการเรียนรู้ MAC" ได้ (Step 834) แต่
**BUM Traffic จริง** (Broadcast เช่น ARP Request, Unknown-Unicast ตอน Table ยังไม่รู้จัก MAC,
Multicast Application Traffic) **ยังต้องมีวิธี "กระจาย" ไปยัง VTEP ที่เกี่ยวข้องอยู่ดี** — เพียงแต่
เปลี่ยนวิธีจาก "Multicast Tree ใน Underlay" (Step 833) ไปเป็นหนึ่งใน 2 ทางเลือกนี้:

| วิธีจัดการ BUM ใน EVPN | อาศัย Type 3 อย่างไร | ต้องมี PIM ใน Underlay ไหม |
|---|---|---|
| **Ingress Replication (Head-End Replication)** | VTEP ต้นทางอ่านรายชื่อ VTEP ทั้งหมดที่ประกาศ Type 3 ของ VNI เดียวกัน แล้ว **Unicast ซ้ำ (Replicate) BUM Frame ไปให้ทุกตัวทีละ Copy** ด้วยตัวเอง | **ไม่ต้อง** — เป็นวิธีที่ Step 836 ของ Part นี้ใช้ |
| **Underlay Multicast** | Type 3 บอก VNI นั้นผูกกับ Multicast Group ไหน (ผ่าน PMSI Tunnel Attribute) — เหมือน Step 833 แต่ Type 3 ทำให้ VTEP รู้ Mapping ผ่าน BGP แทนการ Config Manual ทุกตัว | **ต้องมี** (เหมาะกับ Fabric ขนาดใหญ่มาก ที่ Ingress Replication จะสร้าง Load สูงเกินไปที่ VTEP ต้นทาง) |

> **สรุปสำหรับข้อสอบ CCIE**: **Type 2 = "ใครอยู่ที่ไหน" (Unicast Reachability)**, **Type 3 =
> "ใครอยู่ใน VNI ไหนบ้าง สำหรับ BUM"**, **Type 5 = "Subnet นี้อยู่หลัง VTEP ไหน" (Routed/L3-only)**
> — Type 1 (Ethernet A-D Route) และ Type 4 (Ethernet Segment Route) ใช้กับ **Multi-Homing**
> (Endpoint ต่อ 2 Leaf พร้อมกันแบบ Active-Active) ซึ่งเป็นเนื้อหาระดับ Multi-Site ที่
> [Part 85](part-085-vxlan-evpn-multisite-advanced.md) จะสอนต่อ

---

## Step 836 — Basic VXLAN/EVPN Configuration บน DC-SPINE-1/2, DC-LEAF-1/2

Config ใน Step นี้เป็น **L2-only VXLAN/EVPN Fabric** พื้นฐาน (VNI เดียว, ยังไม่มี Anycast
Gateway/Routing ระหว่าง VNI — ส่วนนั้นอยู่ Step 837) แบ่งเป็น 3 ชั้นตามลำดับที่ต้อง Config จริง

### ชั้นที่ 1: Underlay Routing (OSPF) — ต่อยอดจาก Part 11

Loopback0 และ Point-to-Point Link ทั้งหมดของ DC Fabric ใหม่ (ไม่กระทบ `00-ip-address-plan.md`
เดิม เพราะเป็น Block ใหม่ที่ยังไม่ถูกใช้):

| Device | Loopback0 |
|---|---|
| DC-SPINE-1 | 1.1.1.31/32 |
| DC-SPINE-2 | 1.1.1.32/32 |
| DC-LEAF-1 | 1.1.1.41/32 |
| DC-LEAF-2 | 1.1.1.42/32 |

| Link | Subnet | ปลาย A | ปลาย B |
|---|---|---|---|
| DC-SPINE-1 ↔ DC-LEAF-1 | 10.255.130.0/30 | DC-SPINE-1 Eth1/1 = .1 | DC-LEAF-1 Eth1/1 = .2 |
| DC-SPINE-1 ↔ DC-LEAF-2 | 10.255.130.4/30 | DC-SPINE-1 Eth1/2 = .5 | DC-LEAF-2 Eth1/1 = .6 |
| DC-SPINE-2 ↔ DC-LEAF-1 | 10.255.130.8/30 | DC-SPINE-2 Eth1/1 = .9 | DC-LEAF-1 Eth1/2 = .10 |
| DC-SPINE-2 ↔ DC-LEAF-2 | 10.255.130.12/30 | DC-SPINE-2 Eth1/2 = .13 | DC-LEAF-2 Eth1/2 = .14 |
| CORE-SW1 ↔ DC-SPINE-1 | 10.255.131.0/30 | CORE-SW1 Te1/0/3 = .1 | DC-SPINE-1 Eth1/3 = .2 |
| CORE-SW2 ↔ DC-SPINE-2 | 10.255.131.4/30 | CORE-SW2 Te1/0/3 = .5 | DC-SPINE-2 Eth1/3 = .6 |

Config DC-LEAF-1 (NX-OS — ต้องเปิด `feature ospf` ก่อนใช้เสมอ ต่างจาก IOS ที่เปิดใช้ได้ทันที):

```
DC-LEAF-1(config)# feature ospf
DC-LEAF-1(config)# router ospf 1
DC-LEAF-1(config-router)# router-id 1.1.1.41
DC-LEAF-1(config-router)# auto-cost reference-bandwidth 100000    ! ตาม Part 33 Step 322
DC-LEAF-1(config-router)# exit

DC-LEAF-1(config)# interface loopback0
DC-LEAF-1(config-if)# ip address 1.1.1.41/32
DC-LEAF-1(config-if)# ip router ospf 1 area 0.0.0.0

DC-LEAF-1(config)# interface Ethernet1/1
DC-LEAF-1(config-if)# description ** Uplink to DC-SPINE-1 **
DC-LEAF-1(config-if)# no switchport
DC-LEAF-1(config-if)# mtu 9216
DC-LEAF-1(config-if)# ip address 10.255.130.2/30
DC-LEAF-1(config-if)# ip router ospf 1 area 0.0.0.0
DC-LEAF-1(config-if)# no shutdown

DC-LEAF-1(config)# interface Ethernet1/2
DC-LEAF-1(config-if)# description ** Uplink to DC-SPINE-2 **
DC-LEAF-1(config-if)# no switchport
DC-LEAF-1(config-if)# mtu 9216
DC-LEAF-1(config-if)# ip address 10.255.130.10/30
DC-LEAF-1(config-if)# ip router ospf 1 area 0.0.0.0
DC-LEAF-1(config-if)# no shutdown
```

Config DC-SPINE-1 (Underlay ฝั่ง Spine — DC-LEAF-2 และ CORE-SW1 ทำ Pattern เดียวกัน):

```
DC-SPINE-1(config)# feature ospf
DC-SPINE-1(config)# router ospf 1
DC-SPINE-1(config-router)# router-id 1.1.1.31
DC-SPINE-1(config-router)# auto-cost reference-bandwidth 100000
DC-SPINE-1(config-router)# exit

DC-SPINE-1(config)# interface loopback0
DC-SPINE-1(config-if)# ip address 1.1.1.31/32
DC-SPINE-1(config-if)# ip router ospf 1 area 0.0.0.0

DC-SPINE-1(config)# interface Ethernet1/1
DC-SPINE-1(config-if)# description ** to DC-LEAF-1 **
DC-SPINE-1(config-if)# no switchport
DC-SPINE-1(config-if)# mtu 9216
DC-SPINE-1(config-if)# ip address 10.255.130.1/30
DC-SPINE-1(config-if)# ip router ospf 1 area 0.0.0.0
DC-SPINE-1(config-if)# no shutdown

DC-SPINE-1(config)# interface Ethernet1/2
DC-SPINE-1(config-if)# description ** to DC-LEAF-2 **
DC-SPINE-1(config-if)# no switchport
DC-SPINE-1(config-if)# mtu 9216
DC-SPINE-1(config-if)# ip address 10.255.130.5/30
DC-SPINE-1(config-if)# ip router ospf 1 area 0.0.0.0
DC-SPINE-1(config-if)# no shutdown

DC-SPINE-1(config)# interface Ethernet1/3
DC-SPINE-1(config-if)# description ** Uplink to CORE-SW1 (Campus) **
DC-SPINE-1(config-if)# no switchport
DC-SPINE-1(config-if)# mtu 9216
DC-SPINE-1(config-if)# ip address 10.255.131.2/30
DC-SPINE-1(config-if)# ip router ospf 1 area 0.0.0.0
DC-SPINE-1(config-if)# no shutdown
```

ฝั่ง Campus (CORE-SW1, IOS-XE — เพิ่มแค่ Interface ใหม่ 1 เส้น ไม่กระทบ Config เดิมที่มีอยู่แล้ว
ตาม `00-ip-address-plan.md`, process OSPF ยังเป็น `router ospf 1` เดิม):

```
CORE-SW1(config)# interface TenGigabitEthernet1/0/3
CORE-SW1(config-if)# description ** Uplink to DC-SPINE-1 (New DC Fabric, Part 84) **
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# mtu 9216
CORE-SW1(config-if)# ip address 10.255.131.1 255.255.255.252
CORE-SW1(config-if)# ip ospf 1 area 0
CORE-SW1(config-if)# no shutdown
```

### ชั้นที่ 2: NVE Interface (VTEP) — DC-LEAF-1/2 เท่านั้น

Spine ใน Design นี้ทำหน้าที่ **Underlay Router + BGP Route Reflector** เท่านั้น (ไม่ใช่ VTEP —
ไม่ต้องมี `interface nve1`) — VTEP มีแค่ที่ Leaf:

```
DC-LEAF-1(config)# feature nv overlay
DC-LEAF-1(config)# feature vn-segment-vlan-based
DC-LEAF-1(config)# feature bgp
DC-LEAF-1(config)# feature interface-vlan

! ผูก VLAN 3010 (DC-SERVERS) เข้ากับ VNI 10010
DC-LEAF-1(config)# vlan 3010
DC-LEAF-1(config-vlan)# name DC-SERVERS
DC-LEAF-1(config-vlan)# vn-segment 10010
DC-LEAF-1(config-vlan)# exit

! ประกาศ VNI ใน EVPN พร้อม RD/RT (ใช้ auto เพื่อความง่าย เหมาะกับ Fabric ขนาดเล็ก)
DC-LEAF-1(config)# evpn
DC-LEAF-1(config-evpn)# vni 10010 l2
DC-LEAF-1(config-evpn-vni)# rd auto
DC-LEAF-1(config-evpn-vni)# route-target import auto
DC-LEAF-1(config-evpn-vni)# route-target export auto
DC-LEAF-1(config-evpn-vni)# exit

! NVE Interface = VTEP ตัวจริง
DC-LEAF-1(config)# interface nve1
DC-LEAF-1(config-if-nve)# no shutdown
DC-LEAF-1(config-if-nve)# source-interface loopback0
DC-LEAF-1(config-if-nve)# host-reachability protocol bgp
DC-LEAF-1(config-if-nve)# member vni 10010
DC-LEAF-1(config-if-nve-vni)# ingress-replication protocol bgp    ! <- ไม่ใช้ Multicast (Step 833/835)

! Access Port สำหรับ Server (SRV-DC-01)
DC-LEAF-1(config)# interface Ethernet1/10
DC-LEAF-1(config-if)# description ** to SRV-DC-01 **
DC-LEAF-1(config-if)# switchport
DC-LEAF-1(config-if)# switchport mode access
DC-LEAF-1(config-if)# switchport access vlan 3010
DC-LEAF-1(config-if)# no shutdown
```

DC-LEAF-2 ใช้ Config Pattern เดียวกันทั้งหมด (VNI/VLAN เดียวกัน เพราะอยู่ Broadcast Domain
เดียวกัน) เปลี่ยนแค่ `source-interface loopback0` (ยังเป็น loopback0 เหมือนกัน แต่ IP ต่างกันตาม
ตัวเครื่อง คือ 1.1.1.42) และ Interface ขา Server เป็น SRV-DC-02

### ชั้นที่ 3: BGP EVPN Address-Family

ใช้ **iBGP ภายใน AS 65001 เดียวกับที่ [Part 29](part-029-bgp-fundamentals.md) กำหนดให้เป็น AS
ขององค์กร** — DC-SPINE-1/2 ทำหน้าที่ **Route Reflector** (RR) ให้ DC-LEAF-1/2 เป็น RR-Client
(หลักการ Route Reflector เดียวกับที่เรียนใน Part 29-30 นำมาใช้ที่นี่ตรงๆ):

```
! ==== DC-SPINE-1 (Route Reflector) ====
DC-SPINE-1(config)# feature bgp
DC-SPINE-1(config)# router bgp 65001
DC-SPINE-1(config-router)# router-id 1.1.1.31
DC-SPINE-1(config-router)# neighbor 1.1.1.41 remote-as 65001
DC-SPINE-1(config-router-neighbor)# update-source loopback0
DC-SPINE-1(config-router-neighbor)# address-family l2vpn evpn
DC-SPINE-1(config-router-neighbor-af)# send-community both
DC-SPINE-1(config-router-neighbor-af)# route-reflector-client
DC-SPINE-1(config-router-neighbor-af)# exit
DC-SPINE-1(config-router-neighbor)# exit
DC-SPINE-1(config-router)# neighbor 1.1.1.42 remote-as 65001
DC-SPINE-1(config-router-neighbor)# update-source loopback0
DC-SPINE-1(config-router-neighbor)# address-family l2vpn evpn
DC-SPINE-1(config-router-neighbor-af)# send-community both
DC-SPINE-1(config-router-neighbor-af)# route-reflector-client

! ==== DC-LEAF-1 (RR-Client, peer ทั้ง 2 Spine) ====
DC-LEAF-1(config)# router bgp 65001
DC-LEAF-1(config-router)# router-id 1.1.1.41
DC-LEAF-1(config-router)# neighbor 1.1.1.31 remote-as 65001
DC-LEAF-1(config-router-neighbor)# update-source loopback0
DC-LEAF-1(config-router-neighbor)# address-family l2vpn evpn
DC-LEAF-1(config-router-neighbor-af)# send-community both
DC-LEAF-1(config-router-neighbor-af)# exit
DC-LEAF-1(config-router-neighbor)# exit
DC-LEAF-1(config-router)# neighbor 1.1.1.32 remote-as 65001
DC-LEAF-1(config-router-neighbor)# update-source loopback0
DC-LEAF-1(config-router-neighbor)# address-family l2vpn evpn
DC-LEAF-1(config-router-neighbor-af)# send-community both
```

DC-SPINE-2 และ DC-LEAF-2 ใช้ Pattern เดียวกัน (DC-SPINE-2 เป็น RR ให้ DC-LEAF-1/2 เช่นกัน —
ทำให้มี RR 2 ตัวเพื่อ Redundancy, DC-LEAF-1/2 จึง Peer กับทั้ง DC-SPINE-1 และ DC-SPINE-2 พร้อมกัน)

### Verification เบื้องต้น

```
DC-LEAF-1# show nve peers
Interface  Peer-IP          State  LearnType  Uptime    Router-Mac
---------  ---------------  -----  ---------  --------  -----------------
nve1       1.1.1.42         Up     BGP        00:02:15  n/a

DC-LEAF-1# show nve vni
Codes: CP - Control Plane        DP - Data Plane
Interface VNI      Multicast-group   VNI State  Mode  Type [BD/VRF]  Flags
--------- -------- ----------------  ---------- ----  -------------  -----
nve1      10010    n/a (IR)          Up         CP    L2 [3010]
```

---

## Step 837 — Anycast Gateway ใน VXLAN/EVPN

### แนวคิด: ทุก Leaf มี Gateway IP/MAC "เดียวกัน" พร้อมกัน

Traditional Design ([Part 18](part-018-fhrp.md) HSRP) ให้ Switch หลายตัวแข่งกันเป็น **Active/
Standby** — มี Switch จริงแค่ 1 ตัวที่ "ตอบ" ในเวลาหนึ่งๆ (Standby รอสำรอง) VXLAN/EVPN เลือก
วิธีที่ต่างไปโดยสิ้นเชิงเรียกว่า **Distributed IP Anycast Gateway**: ตั้งค่า IP Address **และ**
MAC Address ของ Default Gateway ให้ **เหมือนกันทุกตัวอักษร** บน **ทุก Leaf ที่มี VNI นั้น** —
ทุก Leaf เป็น **Active พร้อมกันทั้งหมด** (True Active-Active, ไม่มี Standby)

นี่คือแนวคิดเดียวกันกับ **Anycast Gateway ของ SD-Access** ที่ [Part 77](part-077-sd-access-advanced-design.md)
Step 763 สอนไว้ (Edge Node ทุกตัวมี Gateway เดียวกัน) — **เพราะ SD-Access ยืม VXLAN มาเป็น Data
Plane ตรงๆ (Step 839) กลไก Anycast Gateway จึงเป็นกลไกเดียวกันในระดับ Data Plane ทุกประการ**
ต่างกันแค่ว่าใครเป็นคนสั่ง (SD-Access ให้ DNAC Provision ผ่าน LISP, ส่วนที่นี่เรา Config ผ่าน
NX-OS CLI ตรงๆ ผ่าน EVPN)

### ทำไมไม่ชนกันเอง (ARP ไม่งง แม้ MAC ซ้ำหลายที่)

Endpoint ส่ง ARP Request หา Gateway IP เดียวกันได้จากทุก Leaf เพราะ:

- Anycast Gateway MAC **ไม่ใช่ MAC จริงของ Interface ใดๆ** — เป็น MAC เสมือนที่ตั้งค่าเหมือนกัน
  ทุก Leaf โดยตั้งใจ (คล้ายแนวคิด Virtual MAC ของ HSRP แต่ไม่มี Active/Standby Election เลย)
- Traffic ที่ Endpoint ส่งไปหา Gateway **ไม่เคยต้องเดินทางข้าม VXLAN Tunnel ไปยัง Leaf อื่นเลย**
  เพราะ Leaf ที่ Endpoint เสียบอยู่ **ทำหน้าที่ Routing (L3) ให้ทันทีในเครื่องตัวเอง** — Traffic
  จึงไม่มีโอกาส "เจอ" MAC ซ้ำจาก Leaf อื่นในสถานการณ์ปกติ

### Config: Anycast Gateway บน DC-LEAF-1 และ DC-LEAF-2

ต้องตั้งค่า **Global Anycast Gateway MAC ให้ตรงกันทุก Leaf** ก่อน (ทำครั้งเดียวต่อเครื่อง
ค่าต้องเหมือนกันทุกตัวในทั้ง Fabric):

```
! ==== ทำเหมือนกันทั้ง DC-LEAF-1 และ DC-LEAF-2 ====
DC-LEAF-1(config)# fabric forwarding anycast-gateway-mac 2020.0000.00aa
```

จากนั้นตั้งค่า SVI ของ VLAN 3010 (DC-SERVERS) — **IP Address เดียวกัน** บนทั้ง DC-LEAF-1 และ
DC-LEAF-2:

```
! ==== DC-LEAF-1 ====
DC-LEAF-1(config)# feature interface-vlan
DC-LEAF-1(config)# interface Vlan3010
DC-LEAF-1(config-if)# no shutdown
DC-LEAF-1(config-if)# ip address 10.200.30.1/24
DC-LEAF-1(config-if)# fabric forwarding mode anycast-gateway

! ==== DC-LEAF-2 (IP เดียวกันเป๊ะ ไม่ใช่ .2/.3 แบบ HSRP Part 18) ====
DC-LEAF-2(config)# feature interface-vlan
DC-LEAF-2(config)# interface Vlan3010
DC-LEAF-2(config-if)# no shutdown
DC-LEAF-2(config-if)# ip address 10.200.30.1/24
DC-LEAF-2(config-if)# fabric forwarding mode anycast-gateway
```

### Verification

```
DC-LEAF-1# show interface vlan 3010
Vlan3010 is up, line protocol is up
  Hardware is EtherSVI, address is 2020.0000.00aa    <- Anycast Gateway MAC (เหมือนกันทุก Leaf)
  Internet Address is 10.200.30.1/24                  <- Anycast Gateway IP (เหมือนกันทุก Leaf)

DC-LEAF-2# show interface vlan 3010
Vlan3010 is up, line protocol is up
  Hardware is EtherSVI, address is 2020.0000.00aa    <- ตรงกับ DC-LEAF-1 ทุกตัวอักษร
  Internet Address is 10.200.30.1/24                  <- ตรงกับ DC-LEAF-1 ทุกตัวอักษร
```

> **สำหรับข้อสอบ CCIE**: Anycast Gateway **ไม่ได้ยกเลิกแนวคิด Default Gateway** เพียงแค่เปลี่ยน
> จาก "Gateway 1 ตัวที่ Active" (HSRP/VRRP) เป็น "Gateway ทุกตัวพร้อม Active เท่ากันหมด" — ข้อสอบ
> ชอบถามว่าทำไมวิธีนี้ถึง "ไม่ต้องมี Election" ต่างจาก HSRP ที่ต้องมี Active/Standby เสมอ

---

## Step 838 — VXLAN/EVPN vs VLAN+STP+HSRP แบบดั้งเดิม

### ตารางเปรียบเทียบเต็มรูปแบบ

| หัวข้อ | Traditional (VLAN + STP + HSRP) | VXLAN/EVPN |
|---|---|---|
| Loop Prevention | STP/RSTP/MSTP ([Part 5](part-005-stp-rstp-mstp.md)) — Block Port ส่วนเกิน | **ไม่มี Loop ให้ป้องกันเลย** — Underlay เป็น Routed L3 ล้วน (ไม่มี L2 Loop โดยธรรมชาติ) |
| การใช้ Bandwidth ของ Uplink | ~50% (Link ที่ถูก STP Block ใช้งานไม่ได้ปกติ) | **100% ทุก Link** (ECMP Active-Active เต็มรูปแบบ) |
| First-Hop Gateway | HSRP/VRRP ([Part 18](part-018-fhrp.md)) — Active 1 ตัว, Standby รอสำรอง | **Anycast Gateway (Step 837)** — Active ทุกตัวพร้อมกัน |
| Convergence เมื่อ Link/Node ล้ม | STP ต้อง Recalculate Topology (Sub-second ถึงหลายวินาทีตาม Timer) | Routing Protocol (OSPF/ECMP) Reroute ทันที — ไม่มี STP Recalculation เลย |
| Endpoint/VM Mobility ข้าม Rack | ต้องอยู่ VLAN เดิม/Subnet เดิมเสมอ (ผูกกับ Physical Location ของ VLAN Trunk) | ย้ายไป Leaf ไหนก็ได้ที่มี VNI เดียวกัน Provision ไว้ — Gateway/IP เดิมใช้ได้ทันที |
| จำนวน Segment สูงสุด | 4094 (VLAN 12-bit, Step 831) | 16,777,216 (VNI 24-bit) |
| Multi-Tenancy Isolation | ต้องพึ่ง VRF-Lite/ACL แยกเพิ่มเอง | มีในตัวผ่าน RD/Route-Target (คล้าย MPLS L3VPN) |
| MAC Learning | Data-Plane ล้วน (Flood ตาม Broadcast Domain) | Control-Plane ผ่าน BGP (Type 2, Step 835) — เร็วกว่า, แม่นกว่า |
| Protocol ที่ต้องรันเพิ่มจาก Routing พื้นฐาน | STP (ทุก L2 Segment), HSRP (ทุก SVI) | BGP EVPN (Address-Family เดียว ครอบคลุมทั้ง Fabric) |
| Hardware ที่ต้องรองรับ | Switch L2/L3 ทั่วไป (Catalyst ทุกรุ่นตาม `00-ip-address-plan.md`) | ต้องรองรับ VXLAN/NVE (Nexus 9000 เป็นต้นไป, หรือ Catalyst 9000 บางรุ่น) |
| Troubleshooting | `show spanning-tree`, `show standby` | `show nve peers`, `show bgp l2vpn evpn`, `show mac address-table` |

### สรุปทำไม VXLAN/EVPN กำลังกลายเป็นมาตรฐานใหม่

1. **East-West Traffic ในยุค Cloud/Virtualization โตกว่า North-South มาก** — Spine-Leaf +
   ECMP เต็มรูปแบบตอบโจทย์นี้โดยตรง (STP-blocked Link คือสิ่งที่ยอมรับไม่ได้อีกต่อไป)
2. **VM/Container Mobility ต้องไร้รอยต่อ** — Anycast Gateway ทำให้ Endpoint ย้าย Location ทาง
   กายภาพได้โดยไม่ต้อง Renumber (เหมือนที่ [Part 77](part-077-sd-access-advanced-design.md) Step
   763 แก้ปัญหาเดียวกันในโลก Campus)
3. **BGP เป็น Protocol ที่ Enterprise ส่วนใหญ่มีทักษะอยู่แล้ว** ([Part 29-30](part-029-bgp-fundamentals.md))
   — การใช้ EVPN ต่อยอดจาก BGP เดิมง่ายกว่าเรียน Protocol ใหม่ทั้งชุด
4. **Multi-Tenancy ในตัว** — DC สมัยใหม่ (Private Cloud, Hybrid Cloud) ต้องการ Isolation
   ระดับ RD/Route-Target ไม่ใช่แค่ VLAN

> **ข้อเตือนใจสำหรับข้อสอบ CCIE Design**: VXLAN/EVPN **ไม่ได้แทนที่ Traditional Campus ทุกที่**
> — สำหรับ Access Layer ที่ End-user เสียบสายตรง (ไม่ใช่ DC) [Part 33](part-033-advanced-stp-campus-design.md)/[Part 35](part-035-advanced-fhrp-campus-ha.md)
> ยังเป็นคำตอบที่เหมาะสมกว่าในหลายกรณี — ต้องเลือกตาม Traffic Pattern และ Scale requirement จริง
> (หลักการเดียวกับที่ [Part 72](part-072-large-campus-fabric-design.md) Step 717 สอนไว้)

---

## Step 839 — จุดเชื่อมกับ SD-Access: Data Plane เดียวกัน, Control Plane คนละแบบ

### ยืนยันสิ่งที่ [Part 38](part-038-sd-access-fundamentals.md) บอกไว้แบบย่อ

[Part 38](part-038-sd-access-fundamentals.md) Step 374 สรุปไว้สั้นๆ ว่า SD-Access ใช้ **VXLAN
เป็น Data Plane** และ **LISP เป็น Control Plane** — ตอนนี้เราเรียน VXLAN Header เต็มรูปแบบแล้ว
(Step 832) สามารถยืนยันได้ตรงๆ ว่า:

- **VXLAN Header ที่ SD-Access ใช้ห่อหุ้ม Traffic ของ Endpoint คือ Header ตัวเดียวกันเป๊ะ**
  กับที่เรียนใน Step 832 (UDP 4789, VNI 24-bit, Overhead 50 byte) — ไม่มีความต่างทาง Technical
  ระดับ Data Plane เลยแม้แต่นิดเดียว
- ความต่างอยู่ที่ **ใครเป็นคนบอก VTEP ว่า "MAC/IP นี้อยู่ที่ไหน"** — SD-Access ใช้ **LISP
  Map-Server/Map-Resolver** ([Part 38](part-038-sd-access-fundamentals.md) Step 373) ส่วน
  VXLAN/EVPN Fabric (Part นี้) ใช้ **MP-BGP EVPN**

### LISP vs EVPN: กลไก Control Plane ที่ "เป็นลูกพี่ลูกน้องกัน" แต่ไม่ใช่ตัวเดียวกัน

ทั้งสองแก้ปัญหาเดียวกัน (บอกตำแหน่งปัจจุบันของ Endpoint แบบไม่ผูกกับ Subnet Boundary) ด้วย
แนวคิดพื้นฐานคล้ายกัน (แยก Identity ออกจาก Location) แต่ **กลไกการทำงานต่างกันโดยพื้นฐาน**:

| หัวข้อ | LISP (SD-Access, [Part 38](part-038-sd-access-fundamentals.md) Step 373) | EVPN (VXLAN/EVPN Fabric, Part นี้) |
|---|---|---|
| Protocol Family | LISP (Protocol เฉพาะทาง ออกแบบมาสำหรับ ID/Location Separation โดยตรง) | ต่อยอดจาก **BGP** ([Part 29-30](part-029-bgp-fundamentals.md)) ผ่าน Address-Family ใหม่ |
| รูปแบบการกระจายข้อมูล | **Pull Model** — ITR **ถาม** Map-Resolver เมื่อต้องการ (On-Demand Query) | **Push Model** — VTEP **ประกาศ** ผ่าน BGP UPDATE ทันทีที่รู้ (Proactive) |
| ที่เก็บฐานข้อมูลกลาง | Map-Server/Map-Resolver (มักรวมกับ Border Node) | ไม่มี "ฐานข้อมูลกลาง" เดี่ยว — ทุก BGP Speaker มี RIB ของตัวเอง (Route Reflector ช่วยกระจาย ไม่ใช่ที่เก็บข้อมูลแบบ Query) |
| ศัพท์ที่ใช้เรียก Location/Identity | EID (Identity) / RLOC (Location) | MAC/IP Address (Identity) / VTEP Next-Hop (Location) — ไม่มีศัพท์เฉพาะ ใช้ภาษา BGP ปกติ |
| Route Type/Message หลัก | Map-Register, Map-Request, Map-Reply | Type 2 (MAC/IP), Type 3 (BUM), Type 5 (Prefix) — ดู Step 835 |
| Multi-Tenancy | Instance-ID (คล้าย VRF) | Route-Distinguisher + Route-Target (มาตรฐาน BGP VPN) |
| Origin/History | พัฒนาโดย Cisco+IETF สำหรับ Campus Fabric/Mobility โดยเฉพาะ | พัฒนาต่อจาก MPLS L3VPN/BGP VPN (RFC 4364) ให้รองรับ L2 ผ่าน RFC 7432 |
| ใครควบคุมใน Lab นี้ | Cisco DNA Center (GUI Workflow) | CLI ตรง (NX-OS) หรือ Cisco ACI APIC ในงานจริงระดับ Enterprise DC |

### ทำไม Cisco เลือกคนละ Protocol สำหรับ 2 บริบท

- **SD-Access (Campus)**: โจทย์หลักคือ **Wireless Client Roaming** และ **Wired Endpoint ย้าย
  ที่นั่ง** ที่เกิดขึ้น**บ่อยและกระจัดกระจาย** (Client จำนวนมาก, เปลี่ยนตำแหน่งตลอดเวลา) —
  LISP ออกแบบมาให้ Query แบบ On-Demand เหมาะกับ Pattern นี้ (ไม่ต้องกระจาย Full Table ไปทุกที่
  ล่วงหน้า ประหยัด Control-Plane Overhead เมื่อมี Endpoint จำนวนมากแต่ Active พร้อมกันไม่หมด)
- **VXLAN/EVPN Fabric (Data Center)**: โจทย์หลักคือ **VM/Container Mobility ระหว่าง Rack**
  และ **Multi-Tenancy ขนาดใหญ่** — จำนวน "Location" (Leaf/Rack) น้อยกว่า Campus มาก แต่ต้องการ
  ความแม่นยำ/เร็วในการ Converge สูง และ Ecosystem ทั้ง Data Center (Firewall, Load Balancer,
  Orchestration) คุ้นเคยกับ BGP อยู่แล้ว — Push Model ของ BGP ให้ Convergence เร็วกว่าและ
  Integrate เข้ากับ MPLS L3VPN ที่มีอยู่แล้วง่ายกว่า

> **สรุปสำหรับข้อสอบ CCNP ENCOR/CCIE**: เจอคำว่า **"SD-Access"** ให้นึก **LISP**, เจอคำว่า
> **"VXLAN/EVPN Fabric"** หรือ **"Data Center Fabric"** เฉยๆ ให้นึก **MP-BGP EVPN** — ทั้งสอง
> **ใช้ VXLAN Header เดียวกันเป็น Data Plane** (Step 832) แต่ **Control Plane คนละ Protocol
> Family กันโดยสิ้นเชิง** แม้แนวคิดพื้นฐาน (แยก Identity จาก Location) จะคล้ายกันก็ตาม

---

## Step 840 — Lab เต็มรูปแบบ: VXLAN/EVPN L2 Fabric + Anycast Gateway + Host Mobility Test

### Topology สุดท้ายของ Part นี้

```
                    ┌─────────────────────────────────────────────┐
                    │              CAMPUS LAB เดิม                   │
                    │   CORE-SW1 <===LACP Po1===> CORE-SW2           │
                    └──────┬─────────────────────────────┬─────────┘
                    10.255.131.0/30              10.255.131.4/30
                           │ .1            .2 │              │ .5            .6 │
                    ┌──────┴──────┐                   ┌──────┴──────┐
                    │  DC-SPINE-1  │                   │  DC-SPINE-2  │
                    │ 1.1.1.31/32  │                   │ 1.1.1.32/32  │
                    │ (BGP RR)     │                   │ (BGP RR)     │
                    └───┬──────┬──┘                   └───┬──────┬──┘
              10.255.130.0/30│  │10.255.130.4/30 10.255.130.8/30│  │10.255.130.12/30
                        .1│  │.5                        .9│  │.13
                          │  └───────────────┐   ┌────────┘  │
                          │  ┌───────────────┘   └───────────┼──┐
                        .2│.6│                             .10│.14
                    ┌──────┴──┴───┐                   ┌──────┴──┴───┐
                    │  DC-LEAF-1   │══VXLAN(VNI 10010)══│  DC-LEAF-2   │
                    │ 1.1.1.41/32  │   Tunnel ผ่าน Underlay │ 1.1.1.42/32  │
                    │ (VTEP)       │                   │ (VTEP)       │
                    └──────┬───────┘                   └──────┬───────┘
                           │ Eth1/10                            │ Eth1/10
                     SRV-DC-01                              SRV-DC-02
              MAC 0050.56aa.1001                      MAC 0050.56aa.1002
              VLAN 3010 / VNI 10010 / Subnet 10.200.30.0/24
              Anycast GW: 10.200.30.1 (MAC 2020.0000.00aa บนทั้ง 2 Leaf)
```

### Config เต็มรูปแบบ (รวมทุก Step 836-837 เข้าด้วยกัน)

**DC-SPINE-1 (สรุปเต็ม):**

```
feature ospf
feature bgp
!
router ospf 1
  router-id 1.1.1.31
  auto-cost reference-bandwidth 100000
!
interface loopback0
  ip address 1.1.1.31/32
  ip router ospf 1 area 0.0.0.0
!
interface Ethernet1/1
  description ** to DC-LEAF-1 **
  no switchport
  mtu 9216
  ip address 10.255.130.1/30
  ip router ospf 1 area 0.0.0.0
  no shutdown
!
interface Ethernet1/2
  description ** to DC-LEAF-2 **
  no switchport
  mtu 9216
  ip address 10.255.130.5/30
  ip router ospf 1 area 0.0.0.0
  no shutdown
!
interface Ethernet1/3
  description ** Uplink to CORE-SW1 **
  no switchport
  mtu 9216
  ip address 10.255.131.2/30
  ip router ospf 1 area 0.0.0.0
  no shutdown
!
router bgp 65001
  router-id 1.1.1.31
  neighbor 1.1.1.41 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
      route-reflector-client
  neighbor 1.1.1.42 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
      route-reflector-client
```

**DC-SPINE-2**: Config เหมือน DC-SPINE-1 ทุกประการ เปลี่ยนแค่ IP ตาม Loopback0 `1.1.1.32`,
Ethernet1/1→DC-LEAF-1 (`10.255.130.9/30`), Ethernet1/2→DC-LEAF-2 (`10.255.130.13/30`),
Ethernet1/3→CORE-SW2 (`10.255.131.6/30`)

**DC-LEAF-1 (สรุปเต็ม):**

```
feature ospf
feature bgp
feature nv overlay
feature vn-segment-vlan-based
feature interface-vlan
!
router ospf 1
  router-id 1.1.1.41
  auto-cost reference-bandwidth 100000
!
interface loopback0
  ip address 1.1.1.41/32
  ip router ospf 1 area 0.0.0.0
!
interface Ethernet1/1
  description ** to DC-SPINE-1 **
  no switchport
  mtu 9216
  ip address 10.255.130.2/30
  ip router ospf 1 area 0.0.0.0
  no shutdown
!
interface Ethernet1/2
  description ** to DC-SPINE-2 **
  no switchport
  mtu 9216
  ip address 10.255.130.10/30
  ip router ospf 1 area 0.0.0.0
  no shutdown
!
vlan 3010
  name DC-SERVERS
  vn-segment 10010
!
evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target export auto
!
interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp
  member vni 10010
    ingress-replication protocol bgp
!
fabric forwarding anycast-gateway-mac 2020.0000.00aa
!
interface Vlan3010
  no shutdown
  ip address 10.200.30.1/24
  fabric forwarding mode anycast-gateway
!
interface Ethernet1/10
  description ** to SRV-DC-01 **
  switchport
  switchport mode access
  switchport access vlan 3010
  no shutdown
!
router bgp 65001
  router-id 1.1.1.41
  neighbor 1.1.1.31 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
  neighbor 1.1.1.32 remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community both
```

**DC-LEAF-2**: Config เหมือน DC-LEAF-1 ทุกประการ (VLAN/VNI/Anycast GW IP/MAC **เหมือนกันเป๊ะ**
ตามหลัก Step 837) เปลี่ยนแค่ Loopback0 `1.1.1.42`, Ethernet1/1 (`10.255.130.6/30`), Ethernet1/2
(`10.255.130.14/30`), และ Ethernet1/10 description เป็น `to SRV-DC-02`

**CORE-SW1/CORE-SW2 (เพิ่มเติมจาก Config เดิม)**: ตามที่แสดงไว้แล้วใน Step 836 (เพิ่ม
`TenGigabitEthernet1/0/3` พร้อม `ip ospf 1 area 0`)

### Verification: ก่อนเริ่ม Host Mobility Test

```
DC-LEAF-1# show nve peers
Interface  Peer-IP     State  LearnType  Uptime    Router-Mac
---------  ----------  -----  ---------  --------  -----------------
nve1       1.1.1.42    Up     BGP        00:05:40  n/a

DC-LEAF-1# show bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 1.1.1.41:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[0050.56aa.1001]:[0]:[0.0.0.0]/216
                      1.1.1.41                        100      32768 i
Route Distinguisher: 1.1.1.42:32777    (L2VNI 10010, เรียนรู้ผ่าน BGP จาก DC-LEAF-2)
*>l[2]:[0]:[0]:[48]:[0050.56aa.1002]:[0]:[0.0.0.0]/216
                      1.1.1.42          100             0 65001 i

DC-LEAF-1# show mac address-table vlan 3010
   VLAN     Mac Address       Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
   3010     0050.56aa.1001    dynamic   0         F    F    Ethernet1/10
   3010     0050.56aa.1002    dynamic   0         F    F    nve1(1.1.1.42)
```

สังเกต: `0050.56aa.1002` (SRV-DC-02 หลัง DC-LEAF-2) ถูกเรียนรู้ **ผ่าน `nve1` ชี้ไปที่ Peer
`1.1.1.42`** — DC-LEAF-1 **ไม่เคยเห็น Frame จริงจาก SRV-DC-02 เลยแม้แต่ Frame เดียว** แต่รู้จัก
MAC นี้ล่วงหน้าแล้วผ่าน BGP EVPN Type 2 (Step 834) — นี่คือข้อพิสูจน์ที่ชัดเจนที่สุดของความต่าง
จาก Flood-and-Learn (Step 833)

### Host Mobility Test: ย้าย SRV-DC-01 จาก DC-LEAF-1 ไป DC-LEAF-2

จำลองสถานการณ์ VM Migration/ย้ายสาย — SRV-DC-01 (MAC `0050.56aa.1001`) ถูกย้ายจาก
`DC-LEAF-1 Eth1/10` ไปเสียบที่ `DC-LEAF-2 Eth1/11` (Port ใหม่ที่เตรียม VLAN 3010 ไว้แล้ว)

**ขั้นที่ 1 — ก่อนย้าย (Baseline):**

```
DC-SPINE-1# show bgp l2vpn evpn 0050.56aa.1001
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.56aa.1001]:[0]:[0.0.0.0]/216
Paths: (1 available, best #1)
  Advertised to peers: 1.1.1.42
  65001
    1.1.1.41 (metric 0) from 1.1.1.41 (1.1.1.41)
      Origin IGP, localpref 100, valid, external, best
      Extcommunity: RT:65001:10010 ENCAP:8 Router MAC:2020.0000.00aa
```

**ขั้นที่ 2 — ย้ายสายจริง:** ถอด SRV-DC-01 ออกจาก `DC-LEAF-1 Eth1/10` แล้วเสียบที่
`DC-LEAF-2 Eth1/11`:

```
DC-LEAF-2(config)# interface Ethernet1/11
DC-LEAF-2(config-if)# description ** to SRV-DC-01 (Migrated from DC-LEAF-1) **
DC-LEAF-2(config-if)# switchport
DC-LEAF-2(config-if)# switchport mode access
DC-LEAF-2(config-if)# switchport access vlan 3010
DC-LEAF-2(config-if)# no shutdown
```

ทันทีที่ SRV-DC-01 ส่ง Frame แรก (หรือ Gratuitous ARP ถ้าเป็น VM Migration จริง) ออกมาที่
Eth1/11 ของ DC-LEAF-2:

```
! DC-LEAF-2 เรียนรู้ MAC ท้องถิ่นทันที (Data-Plane Learning ปกติ)
DC-LEAF-2# show mac address-table vlan 3010
   VLAN     Mac Address       Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
   3010     0050.56aa.1001    dynamic   0         F    F    Ethernet1/11    <- เปลี่ยนจาก nve1 เป็น Local
   3010     0050.56aa.1002    dynamic   3         F    F    Ethernet1/10

! DC-LEAF-2 ประกาศ Type 2 Route ใหม่ พร้อม MAC Mobility Extended Community
! (Sequence Number เพิ่มขึ้น เพื่อบอกว่า "นี่คือข้อมูลใหม่กว่า" ป้องกัน Route เก่า-ใหม่ชนกัน)
DC-LEAF-2# show bgp l2vpn evpn 0050.56aa.1001
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.56aa.1001]:[0]:[0.0.0.0]/216
Paths: (1 available, best #1)
  Advertised to peers: 1.1.1.31, 1.1.1.32
  Local
    1.1.1.42 (metric 0) from 0.0.0.0 (1.1.1.42)
      Origin IGP, localpref 100, valid, sourced, best
      Extcommunity: RT:65001:10010 ENCAP:8 Router MAC:2020.0000.00aa
                    Mac Mobility:0x00 (Sequence 1)          <- Sequence Number ใหม่ > เดิม (0)
```

**ขั้นที่ 3 — DC-LEAF-1 รับ Update ผ่าน BGP และปรับตัวทันที (ไม่ต้องรอ Flood/Age-out):**

```
DC-LEAF-1# show mac address-table vlan 3010
   VLAN     Mac Address       Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
   3010     0050.56aa.1001    dynamic   0         F    F    nve1(1.1.1.42)    <- เปลี่ยนจาก Local เป็น Remote ทันที
   3010     0050.56aa.1002    dynamic   5         F    F    nve1(1.1.1.42)
```

**ขั้นที่ 4 — ยืนยันความเร็วที่ DC-SPINE (Route Reflector):**

```
DC-SPINE-1# show bgp l2vpn evpn 0050.56aa.1001
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.56aa.1001]:[0]:[0.0.0.0]/216
Paths: (2 available, best #2)
  Path 1: withdrawn (received from 1.1.1.41, superseded by higher Sequence Number)
  Path 2: (best)
    1.1.1.42 (metric 0) from 1.1.1.42 (1.1.1.42)
      Origin IGP, localpref 100, valid, external, best
      Extcommunity: RT:65001:10010 ENCAP:8 Router MAC:2020.0000.00aa
                    Mac Mobility:0x00 (Sequence 1)
```

### สรุปผลการทดสอบ

| ขั้นตอน | Flood-and-Learn (ถ้าไม่มี EVPN) | EVPN (Lab จริงของ Part นี้) |
|---|---|---|
| DC-LEAF-1 รู้ว่า MAC ย้ายไปแล้ว | ต้องรอ MAC Aging Timer หมดอายุก่อน (Default หลักนาที) แล้ว "ลืม" Entry เก่าไปเฉยๆ ไม่รู้ว่าย้ายไปไหน | รับ BGP UPDATE Route ใหม่ (Sequence Number สูงกว่า) — **หลักวินาที** |
| DC-LEAF-2 บอก Fabric อื่นว่า "MAC นี้อยู่ที่ฉันแล้ว" | ไม่มีกลไก Proactive เลย ต้องรอ Traffic จริงจากปลายทางอื่นมาถามซ้ำ (Flood ใหม่) | ประกาศ Type 2 Route ทันทีที่เรียนรู้ Local (Push Model) |
| ความเสี่ยง MAC Flapping ระหว่าง 2 Leaf ชั่วครู่ | เป็นไปได้สูง (ทั้ง 2 Leaf อาจ "คิดว่า" ตัวเองมี MAC นี้พร้อมกันชั่วขณะ) | ป้องกันด้วย **MAC Mobility Sequence Number** — Route ที่ Sequence สูงกว่าชนะเสมอ ไม่มี Flapping |

> **สรุปสำหรับข้อสอบ CCIE**: การทดสอบนี้คือข้อพิสูจน์เชิงปฏิบัติของทุกแนวคิดที่เรียนมาทั้ง Part —
> **VXLAN (Step 832)** คือสิ่งที่ห่อหุ้ม Traffic จริงให้เดินทางข้าม Underlay, **EVPN Type 2
> (Step 835)** คือสิ่งที่ทำให้ DC-LEAF-1 รู้เรื่องการย้ายโดยไม่ต้องรอ Flood, **MAC Mobility
> Sequence Number** คือกลไกป้องกัน Race Condition ระหว่าง Route เก่ากับใหม่ — ทั้งหมดนี้คือเหตุผล
> ที่ VXLAN/EVPN เหมาะกับ VM/Container Mobility มากกว่า Flood-and-Learn แบบดั้งเดิมอย่างชัดเจน

---

## แบบฝึกหัดทวนความเข้าใจ Part 84

1. VLAN ID มีขนาดกี่ Bit และรองรับได้กี่ Segment เทียบกับ VNI ของ VXLAN?
2. VXLAN Encapsulation เพิ่ม Overhead ให้ Packet กี่ Byte และทำไมต้องเปิด Jumbo MTU บน Underlay?
3. VXLAN Flood-and-Learn แบบดั้งเดิมต้องพึ่งพา Protocol อะไรใน Underlay สำหรับจัดการ BUM Traffic
   และมีข้อจำกัดอะไรบ้าง?
4. EVPN Route Type 2, Type 3, และ Type 5 แต่ละแบบใช้ทำอะไร แตกต่างกันอย่างไร?
5. LISP (SD-Access) กับ EVPN (VXLAN/EVPN Fabric) เหมือนและต่างกันอย่างไร?

**เฉลย:**

1. VLAN ID มีขนาด 12-bit รองรับ 4096 ค่า (ใช้จริง 4094) ส่วน VNI ของ VXLAN มีขนาด 24-bit
   รองรับ 16,777,216 (16 ล้าน) Segment
2. เพิ่ม Overhead 50 byte (Outer Eth 14 + Outer IP 20 + Outer UDP 8 + VXLAN Header 8) — ถ้า
   Underlay ยังใช้ MTU 1500 byte เดิม Packet ที่ Endpoint ส่งขนาดพอดี 1500 byte จะกลายเป็น 1550
   byte หลัง Encapsulate แล้วถูก Fragment หรือ Drop จึงต้องเปิด Jumbo MTU (แนะนำ 9216 byte)
   รองรับล่วงหน้า
3. ต้องพึ่ง **PIM-SM** (IP Multicast) เพื่อสร้าง Multicast Distribution Tree ต่อ VNI สำหรับ
   กระจาย BUM Traffic ข้าม VTEP — ข้อจำกัดคือต้อง Deploy/Design Multicast เพิ่มทั้ง Underlay,
   Unknown-Unicast ยังต้อง Flood ทุกครั้ง, ไม่มี ARP Suppression, Convergence ช้าเมื่อ Endpoint
   ย้าย, และ Scale ถูกจำกัดตาม Multicast Route Table ของ Hardware
4. **Type 2 (MAC/IP Advertisement)** ประกาศตำแหน่งของ MAC/IP แต่ละ Host, **Type 3 (Inclusive
   Multicast Ethernet Tag)** ประกาศรายชื่อ VTEP ที่ต้องรับ BUM Traffic ของ VNI นั้น (ใช้สร้าง
   Ingress Replication List หรือผูก Multicast Group), **Type 5 (IP Prefix Route)** ประกาศ Subnet
   ทั้งก้อนสำหรับดีไซน์ Routed/L3-only ที่ไม่ต้องการประกาศทีละ Host Route
5. เหมือนกันในแนวคิดพื้นฐาน (แยก Identity ออกจาก Location เพื่อรองรับ Mobility) แต่กลไกต่างกัน:
   LISP เป็น **Pull Model** (ITR Query Map-Resolver แบบ On-Demand ใช้ EID/RLOC) เหมาะกับ SD-Access
   ที่มี Endpoint จำนวนมากเคลื่อนที่บ่อย ส่วน EVPN เป็น **Push Model** (BGP UPDATE ประกาศ Proactive
   ใช้ MAC/IP กับ VTEP Next-Hop) เหมาะกับ Data Center ที่ต้องการ Convergence เร็วและ Integrate
   กับ BGP/MPLS L3VPN ที่มีอยู่แล้ว — ทั้งสองใช้ **VXLAN Header เดียวกัน** เป็น Data Plane

---

## สรุป Part 84

Part นี้ทำตามสัญญาที่ [Part 38](part-038-sd-access-fundamentals.md) และ
[Part 72](part-072-large-campus-fabric-design.md) ทิ้งไว้ — เจาะลึก **VXLAN** (Data Plane
Encapsulation, UDP 4789, VNI 24-bit, Overhead 50 byte) และ **EVPN** (Control Plane ที่ใช้
MP-BGP กระจาย MAC/IP ผ่าน Route Type 2/3/5) แบบเต็มรูปแบบ พร้อมประกาศ **Lab Data Center Fabric
ใหม่** (DC-SPINE-1/2, DC-LEAF-1/2) ที่ต่อพ่วงเข้ากับ Campus Lab เดิมผ่าน CORE-SW1/CORE-SW2 —
Config จริงตั้งแต่ Underlay OSPF, `nve` Interface, BGP EVPN Address-Family, Anycast Gateway,
ไปจนถึงการทดสอบ Host Mobility ที่พิสูจน์ว่า EVPN Converge เร็วกว่า Flood-and-Learn ดั้งเดิมมาก
เพียงพอสำหรับใช้งาน VM/Container Migration จริงในระดับ Enterprise Data Center

✅ **พร้อมสำหรับ Part 85**: เราจะขยาย Fabric นี้ให้ครอบคลุมหลาย Site (**Multi-Site EVPN**) —
เรียน **Ethernet Segment (Type 1/4) สำหรับ Multi-Homing**, **Border Gateway (BGW)** สำหรับเชื่อม
Fabric ข้าม Data Center, และดีไซน์ **Type-5-only Routed VXLAN** สำหรับ Scale ระดับ CCIE เต็มรูปแบบ

**ไปต่อ:** [Part 85 — VXLAN/EVPN Multi-Site Advanced →](part-085-vxlan-evpn-multisite-advanced.md)
