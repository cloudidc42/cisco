# Part 27 — OSPFv3 for IPv6
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 261–270 จาก 1000 (ระดับ CCNP ENCOR)**

> ต่อจาก [Part 26 — Advanced OSPF](part-026-advanced-ospf.md) ที่ขยาย OSPFv2 (IPv4) ด้วยหัวข้อระดับ
> CCNP เช่น Route Filtering, Redistribution และ Fast Convergence ให้กับ Topology 3-Area เดิมจาก
> [Part 12](part-012-ospfv2-advanced.md) (Area 0 = CORE↔CORE, Area 1 = ฝั่ง CORE-SW1/DIST-SW1-2,
> Area 2 = ฝั่ง CORE-SW2/DIST-SW3-4) — Part นี้จะพา OSPF ตัวเดียวกันไป **routing IPv6** ด้วย
> **OSPFv3 (RFC 5340)** ซึ่งเป็น Link-State Protocol ตระกูลเดียวกับ OSPFv2 ทุกประการในเชิง
> Algorithm (Dijkstra SPF, Area, ABR/ASBR, LSA Flooding) แต่ถูกออกแบบใหม่หลายจุดเพื่อให้ทำงานกับ
> IPv6 ได้อย่างเป็นธรรมชาติ เราจะสร้าง OSPFv3 3-Area ที่ **มี Topology และเลข Area ตรงกับ OSPFv2
> เป๊ะ** บน **CORE-SW1, CORE-SW2, DIST-SW1–4** โดยใช้ ULA Address Plan `fd00:10:10::/48` จาก
> [Part 8](part-008-ipv6-fundamentals.md) และ [00-ip-address-plan.md](00-ip-address-plan.md)
> ปิดท้ายด้วย Lab เต็มรูปแบบที่รัน **OSPFv2 (IPv4) และ OSPFv3 (IPv6) พร้อมกันแบบ Dual-Stack**
> บนอุปกรณ์ชุดเดียวกัน

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 261 | OSPFv3 vs OSPFv2 — ความแตกต่างเชิงโครงสร้างและ Packet Header |
| 262 | OSPFv3 Router-ID และแนวคิด Instance ID |
| 263 | คอนฟิก OSPFv3 พื้นฐาน — สร้าง 3-Area Design เดียวกับ Part 12 บน IPv6 |
| 264 | LSA Type ของ OSPFv3 — Link-LSA และ Intra-Area-Prefix-LSA ที่เพิ่มเข้ามาใหม่ |
| 265 | OSPFv3 Authentication ผ่าน IPsec (AH/ESP) |
| 266 | OSPFv3 Address-Family — รวม IPv4 และ IPv6 ไว้ใน Process เดียว |
| 267 | Route Summarization (`area range`) และ Stub Area บน OSPFv3 |
| 268 | ออกแบบ Dual-Stack Routing — แยก Process vs รวม Address-Family |
| 269 | คำสั่ง Verify/Troubleshoot OSPFv3 |
| 270 | Lab เต็มรูปแบบ: OSPFv3 3-Area พร้อม Authentication และ Summarization |

---

## Step 261 — OSPFv3 vs OSPFv2: ความแตกต่างเชิงโครงสร้างและ Packet Header

### 261.1 สิ่งที่ "เหมือนเดิมทุกประการ" กับ OSPFv2

ก่อนไล่ความต่าง ต้องย้ำก่อนว่า OSPFv3 **ไม่ใช่ Protocol ใหม่ทั้งหมด** — มันคือ OSPF ตัวเดียวกับที่
เรียนมาตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md) เกือบทุกแนวคิดยังใช้ได้เหมือนเดิม:

- ยังเป็น **Link-State**, ยังใช้ **Dijkstra SPF Algorithm** ตัวเดียวกัน
- ยังมี **Neighbor State Machine** 7 State เดิม (Down → Init → 2-Way → ExStart → Exchange →
  Loading → Full) และยัง Election **DR/BDR** บน Broadcast Segment เหมือนเดิม
- ยังมีแนวคิด **Area, ABR, ASBR, Backbone Area 0 Rule** เหมือนเดิมทุกประการ (Part 12)
- ยังใช้ **Administrative Distance = 110** เท่ากัน และ Cost คำนวณจาก Reference Bandwidth ด้วยสูตร
  เดียวกัน (Part 11 Step 101.4)

สิ่งที่ต่างคือ **วิธีที่ OSPF คุยกับ Layer 3** ต้องเปลี่ยนใหม่ทั้งหมด เพราะ IPv6 มีธรรมชาติต่างจาก
IPv4 อยู่หลายจุด (RFC 5340 แทนที่ RFC 2740/2328 เดิม)

### 261.2 ตารางเปรียบเทียบหลัก

| หัวข้อ | OSPFv2 (IPv4) | OSPFv3 (IPv6) |
|---|---|---|
| RFC | RFC 2328 | RFC 5340 |
| วิ่งอยู่บน | IPv4 โดยตรง (IP Protocol 89) | IPv6 โดยตรง (Next Header 89) |
| หน่วยที่สร้าง Adjacency | **Per-Network** — ต้อง match Subnet ที่ Interface Configured ตรงๆ (`network` statement ต้องพิมพ์ IP/Wildcard ให้ตรง Subnet) | **Per-Link** — เปิด OSPFv3 บน Interface ได้เลยไม่ต้องสนใจว่ามี Prefix อะไร Configured อยู่ (1 Link รองรับได้หลาย IPv6 Prefix พร้อมกัน) |
| Address ที่ใช้เป็น Next-Hop/สร้าง Adjacency | Global/Primary IPv4 Address ของ Interface | **Link-Local Address (`fe80::/10`) เสมอ** ไม่ว่าจะมี Global/ULA Configured หรือไม่ |
| Router ID / Area ID | 32-bit แต่มาจาก IPv4 Address โดยตรง | ยังเป็น 32-bit Dotted-Decimal เหมือนเดิม แต่**ไม่ผูกกับ IPv6 Address เลย** (ดู Step 262) |
| Authentication | ฝังอยู่ใน OSPF Packet Header เอง (Null/Plaintext/MD5 — Part 11 Step 108) | **ถูกถอดออกจาก Header ทั้งหมด** ใช้กลไก IPsec (AH/ESP) ของ IPv6 แทน (Step 265) |
| รองรับหลาย Instance บน Link เดียว | ไม่รองรับ | รองรับผ่าน **Instance ID** field (Step 262) — ใช้แยก IPv4-AF/IPv6-AF บน Link เดียวกันได้ (Step 266) |
| LSA ที่บรรจุ Prefix | Router-LSA(1)/Network-LSA(2) มี Prefix ปนอยู่กับข้อมูล Topology | **แยกออกจากกัน** — Router/Network LSA เหลือแค่ข้อมูล Topology, Prefix ย้ายไปอยู่ LSA ใหม่ (Type 9, Step 264) |
| Multicast Address | `224.0.0.5` (All-SPF), `224.0.0.6` (All-DR) | `ff02::5` (All-SPF), `ff02::6` (All-DR) — ตามที่กล่าวถึงแล้วใน [Part 8 Step 73](part-008-ipv6-fundamentals.md#step-73--well-known-multicast-address-และ-solicited-node-multicast) |
| รองรับ Multiple Address-Family ใน Process เดียว | ไม่ (ต้องแยก Process ตาม AF เสมอ) | รองรับผ่าน **OSPFv3 Address-Family** (RFC 5838, Step 266) — รัน IPv4 unicast **และ** IPv6 unicast พร้อมกันใน Process เดียว |

### 261.3 Packet Header เปลี่ยนไปอย่างไร

```
OSPFv2 Common Header (24 byte)                    OSPFv3 Common Header (16 byte)
┌─────────┬─────────┬────────────────┐            ┌─────────┬─────────┬────────────────┐
│ Version │  Type   │ Packet Length  │            │ Version │  Type   │ Packet Length  │
├─────────┴─────────┴────────────────┤            ├─────────┴─────────┴────────────────┤
│           Router ID (32 bit)        │            │           Router ID (32 bit)        │
├──────────────────────────────────────┤            ├──────────────────────────────────────┤
│           Area ID (32 bit)          │            │           Area ID (32 bit)          │
├────────────────┬─────────────────────┤            ├────────┬───────┬─────────────────────┤
│   Checksum     │      AuType         │            │Checksum│Instance│      Reserved       │
├────────────────┴─────────────────────┤            │        │  ID    │                     │
│      Authentication (64 bit)         │            └────────┴───────┴─────────────────────┘
└──────────────────────────────────────┘            ไม่มี AuType/Authentication field เหลืออยู่เลย!
```

จุดที่ต้องจำ 3 อย่าง:

1. **Header สั้นลง** จาก 24 byte เหลือ 16 byte เพราะตัด Authentication field (16 byte รวม AuType)
   ออกทั้งหมด แล้วแทนที่ด้วย **Instance ID (8 bit)** + Reserved (8 bit) เท่านั้น
2. **ไม่มี Network Mask ใน Hello Packet อีกต่อไป** — OSPFv2 Hello บรรจุ Network Mask ของ Interface
   มาด้วยเสมอ (ใช้เช็คว่า Neighbor อยู่ Subnet เดียวกันไหมก่อนจะสร้าง Adjacency) แต่ OSPFv3 ตัดออก
   เพราะแนวคิด Per-Link (261.2) ทำให้ไม่จำเป็นต้องมี Subnet เดียวที่ตรงกันอีกต่อไป — สิ่งที่ยังเช็ค
   เหมือนเดิมคือ Hello/Dead Interval และ Area ID ต้องตรงกัน
3. **Options field** ของ OSPFv3 เปลี่ยนความหมาย bit บางตัว (เช่น เพิ่ม bit "AF" สำหรับบอกว่า Router
   รองรับ Address-Family Capability ตาม RFC 5838 หรือไม่) แต่รายละเอียด bit-level นี้ไม่จำเป็นต้อง
   จำสำหรับข้อสอบ CCNP — จำแค่ภาพรวมว่า Header เบาลงเพราะยกภาระ Authentication ไปให้ IPv6 เอง

### 261.4 IPv4-over-OSPFv3 — เกริ่นนำ (รายละเอียดเต็มใน Step 266)

RFC 5838 นิยาม **OSPFv3 Address Families** ที่ทำให้ OSPFv3 Process เดียวสามารถ Route **ทั้ง IPv4
และ IPv6** พร้อมกันได้ (ไม่ใช่แค่ IPv6 อย่างที่ชื่อ "for IPv6" อาจชวนให้เข้าใจผิด) โดยใช้ Instance
ID เป็นตัวแยก Address-Family ออกจากกันบน Link เดียวกัน — แนวคิดนี้เป็นทางเลือกสำหรับองค์กรที่อยาก
รวม Routing Protocol ของ IPv4/IPv6 ให้เหลือ Process เดียวบริหารจัดการง่ายขึ้น เราจะกลับมาดูโดย
ละเอียดพร้อม Config จริงใน Step 266

---

## Step 262 — OSPFv3 Router-ID และแนวคิด Instance ID

### 262.1 Router ID ยังเป็น 32-bit Dotted-Decimal เหมือนเดิม

แม้จะ Route IPv6 แต่ **OSPFv3 Router ID ยังคงเป็นเลข 32-bit เขียนแบบ Dotted-Decimal เหมือน OSPFv2
ทุกประการ** (เช่น `1.1.1.1`) เพราะ Router ID เป็น Field ที่นิยามโดย **ตัว OSPF Protocol เอง** ไม่ได้
ผูกกับ IP Version ที่ Route อยู่ — เป็นแค่เลข 32-bit ที่บังเอิญนิยมเขียนในรูปแบบ IPv4 เพื่อให้อ่าน
และจดจำง่าย (ไม่มีความหมายเป็น "IP Address" จริงๆ ในบริบทของ OSPFv3)

### 262.2 ทำไมต้อง Config Router ID แบบ Explicit เสมอ

กฎการเลือก RID อัตโนมัติของ OSPFv2 (Part 11 Step 103.2) อาศัย **IPv4 Address ของ Loopback/Physical
Interface** ที่สูงที่สุดเป็นตัวเลือก — แต่ปัญหาคือ **Router/Switch ที่รัน IPv6-only ล้วนๆ (ไม่มี
IPv4 Address อยู่บนอุปกรณ์เลยแม้แต่ใบเดียว) จะไม่มี Candidate ให้ OSPFv3 เลือกอัตโนมัติได้เลย**

```
CORE-SW1(config)# ipv6 router ospf 1
% OSPFv3: Could not pick a router-id, please configure manually
```

ต่างจาก Lab ของหลักสูตรนี้ที่เป็น **Dual-Stack** (มี IPv4 Loopback0 อยู่แล้วทุกตัวตาม
[00-ip-address-plan.md](00-ip-address-plan.md)) ซึ่ง IOS ยังพอจะยืม IPv4 Address มาเลือก RID ให้
อัตโนมัติได้ในทางเทคนิค — แต่ **Best Practice ยังคงเหมือนเดิมกับ Part 11 Step 103.3–103.4**: อย่า
พึ่งพา Auto-Election เด็ดขาด เพราะพฤติกรรมเปลี่ยนแปลงได้โดยไม่ตั้งใจ (RID ไม่เปลี่ยนเองจนกว่าจะ
Restart Process) และในโลก IPv6-only ที่นับวันจะพบมากขึ้น (Enterprise ที่เลิกใช้ IPv4 ภายในแล้ว)
**การไม่มี IPv4 Address ให้ยืมเลยเป็นสถานการณ์จริงที่เกิดขึ้นได้** — หลักสูตรนี้จึงตั้ง `router-id`
แบบ Explicit ให้ตรงกับค่า OSPFv2 RID เดิมทุกตัว (Part 11 Step 103.5) เพื่อให้อ่านคู่กันได้ทันที

```
CORE-SW1(config)# ipv6 router ospf 1
CORE-SW1(config-rtr)# router-id 1.1.1.1
```

### 262.3 Instance ID — แนวคิดที่ไม่มีใน OSPFv2

**Instance ID** คือ Field 8-bit ใน OSPFv3 Header (Step 261.3) ที่ทำให้ **Link เดียวกันวิ่ง OSPFv3
ได้มากกว่า 1 "รอบ" พร้อมกัน โดยไม่ปนกัน** — Router ที่เห็น Instance ID ไม่ตรงกันจะ**มองข้าม Hello
Packet นั้นไปเลย ราวกับไม่เคยได้รับ** (เทียบเท่ากับการอยู่คนละ Network ที่ไม่มีวันเป็น Neighbor กัน)

ตาม RFC 5838 มีการแบ่งช่วง Instance ID เป็นธรรมเนียม (Convention) สำหรับใช้กับ Address-Family:

| Instance ID Range | ใช้กับ Address-Family |
|---|---|
| 0–31 | IPv6 Unicast (ค่า Default ของ OSPFv3 คลาสสิกคือ **0**) |
| 32–63 | IPv4 Unicast (Step 266 ใช้ช่วงนี้เมื่อเปิด Address-Family IPv4 บน Process เดียวกับ IPv6) |
| 64–95 | IPv6 Multicast (สำรองไว้สำหรับ Multicast Routing ในอนาคต) |
| 96–127 | IPv4 Multicast |
| 128–255 | สำรอง/กำหนดเองได้ |

**ประโยชน์ที่จับต้องได้ 2 อย่าง**:

1. **แยก Address-Family บน Link เดียวกัน** (Step 266) — IOS จัดการ Instance ID ให้อัตโนมัติเวลาใช้
   Address-Family Mode ไม่ต้องตั้งเองปกติ
2. **แยก "Topology" บน Link เดียวกันด้วยมือ** — กรณีขั้นสูง (เช่น Traffic Engineering หรือ Lab
   ทดสอบ) ที่ต้องการรัน OSPFv3 สองชุดที่ **ไม่คุยกันเลย** บน Physical Link เดียวกัน ทำได้โดยตั้ง
   Instance ID ต่างกันตรงๆ:

```
Router(config-if)# ipv6 ospf 1 area 0 instance 5
```

> หลักสูตรนี้ใช้ **Instance ID = 0 (Default)** ตลอดทั้ง Lab ของ Part นี้ เพราะไม่มีความจำเป็นต้อง
> แยก Topology ซ้อนกันบน Link เดียว — Step นี้ให้เข้าใจ Concept ไว้สำหรับข้อสอบ CCNP และเพื่อปูทาง
> ไปสู่ Step 266 ที่ Instance ID จะทำงานเบื้องหลังโดยอัตโนมัติ

---

## Step 263 — คอนฟิก OSPFv3 พื้นฐาน: สร้าง 3-Area Design เดียวกับ Part 12 บน IPv6

### 263.1 คำสั่งหลัก 2 คำสั่ง (Syntax แบบคลาสสิก)

| คำสั่ง | Mode | หน้าที่ |
|---|---|---|
| `ipv6 router ospf <process-id>` | Global Config | สร้าง OSPFv3 Process (เทียบเท่า `router ospf` ของ IPv4) |
| `ipv6 ospf <process-id> area <area-id>` | Interface Config | เปิด OSPFv3 บน Interface นั้น เข้า Area ที่ระบุ (เทียบเท่า `ip ospf area` — **ไม่มี `network` statement แบบ OSPFv2** เพราะ OSPFv3 เป็น Per-Link ล้วน ต้อง Config ที่ Interface เท่านั้น) |

> **ข้อกำหนดเบื้องต้น**: ทุกอุปกรณ์ต้องเปิด `ipv6 unicast-routing` ไว้แล้ว (Global Config) ตามที่
> วางไว้ตั้งแต่ [Part 8 Step 76](part-008-ipv6-fundamentals.md#step-76--การ-config-ipv6-บน-cisco-ios)
> — ถ้าลืมเปิด OSPFv3 จะสร้าง Neighbor ไม่ได้เลยแม้ Config OSPFv3 ครบทุกบรรทัด

### 263.2 ส่วนขยาย IPv6 Addressing Plan สำหรับ Backbone/Transit Link

[00-ip-address-plan.md](00-ip-address-plan.md) กำหนด ULA เฉพาะ VLAN Subnet ของผู้ใช้งานไว้แล้ว
(`fd00:10:10:<VLAN-ID>::/64`) แต่ยังไม่มี IPv6 สำหรับ Backbone/Transit Link (`10.255.x.0/30` ฝั่ง
IPv4) — Part นี้จึง **ขยายส่วน ULA เพิ่มเติม** โดยใช้ Hextet ที่ 4 ขึ้นต้นด้วย `ff` (แยกให้ชัดจาก
เลข VLAN ที่ใช้ค่า 2 หลักธรรมดา) ตามด้วยเลขเดียวกับ Octet ที่ 3 ของ IPv4 Backbone เพื่อจำง่าย
คู่กัน — Host Bit ท้ายสุดยึดรูปแบบเดียวกับ IPv4 เดิม (`.1`=ฝั่ง CORE, `.2`=ฝั่ง DIST):

| Link | IPv4 (Part 12 อ้างอิง) | IPv6 ULA (ส่วนขยายของ Part นี้) | Area |
|---|---|---|---|
| CORE-SW1 ↔ CORE-SW2 (Po1) | 10.255.0.0/30 | `fd00:10:10:ff00::/64` | Area 0 |
| CORE-SW1 ↔ DIST-SW1 (Te1/0/1) | 10.255.10.0/30 | `fd00:10:10:ff10::/64` | Area 1 |
| CORE-SW1 ↔ DIST-SW2 (Te1/0/2) | 10.255.20.0/30 | `fd00:10:10:ff20::/64` | Area 1 |
| CORE-SW2 ↔ DIST-SW3 (Te1/0/1) | 10.255.30.0/30 | `fd00:10:10:ff30::/64` | Area 2 |
| CORE-SW2 ↔ DIST-SW4 (Te1/0/2) | 10.255.40.0/30 | `fd00:10:10:ff40::/64` | Area 2 |

Host Address ของแต่ละ Link (`::1` = ฝั่ง CORE, `::2` = ฝั่ง DIST) และ VLAN SVI (`::2`/`::3` ตาม
ธรรมเนียม Real-IP ก่อน HSRP ของ [00-ip-address-plan.md](00-ip-address-plan.md)) สรุปครบใน Step 270

### 263.3 คอนฟิก CORE-SW1 (ABR: Area 0 + Area 1) — OSPFv3 พื้นฐาน

```
CORE-SW1(config)# ipv6 router ospf 1
CORE-SW1(config-rtr)# router-id 1.1.1.1
CORE-SW1(config-rtr)# passive-interface Loopback0
CORE-SW1(config-rtr)# exit
!
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# description ** Routed backbone link to CORE-SW2 - Area 0 (v6) **
CORE-SW1(config-if)# ipv6 address fd00:10:10:ff00::1/64
CORE-SW1(config-if)# ipv6 ospf 1 area 0
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# description ** Routed downlink to DIST-SW1 - Area 1 (v6) **
CORE-SW1(config-if)# ipv6 address fd00:10:10:ff10::1/64
CORE-SW1(config-if)# ipv6 ospf 1 area 1
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# description ** Routed downlink to DIST-SW2 - Area 1 (v6) **
CORE-SW1(config-if)# ipv6 address fd00:10:10:ff20::1/64
CORE-SW1(config-if)# ipv6 ospf 1 area 1
CORE-SW1(config-if)# exit
```

สังเกตว่าโครงสร้างเหมือน OSPFv2 เป๊ะ (Part 12 Step 114.2) เพียงแค่เปลี่ยน `ip ospf` เป็น
`ipv6 ospf` และใช้ `ipv6 router ospf` แทน `router ospf` — **Area Number ตรงกันทุกจุด** (Po1 =
Area 0, Te1/0/1 และ Te1/0/2 = Area 1) ยืนยันหลักการที่โจทย์ต้องการ: OSPFv3 ใช้ Topology/Area
Design เดียวกับ OSPFv2 ทุกประการ เพียงแค่เป็นคนละ Protocol/LSDB ที่วิ่งขนานกัน

### 263.4 คอนฟิก DIST-SW1 (Internal Router, Area 1 ล้วน) — OSPFv3 พื้นฐาน

```
DIST-SW1(config)# ipv6 router ospf 1
DIST-SW1(config-rtr)# router-id 1.1.1.11
DIST-SW1(config-rtr)# passive-interface Loopback0
DIST-SW1(config-rtr)# exit
!
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ipv6 address fd00:10:10:10::2/64
DIST-SW1(config-if)# ipv6 ospf 1 area 1
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# interface Vlan20
DIST-SW1(config-if)# ipv6 address fd00:10:10:20::2/64
DIST-SW1(config-if)# ipv6 ospf 1 area 1
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# interface Vlan99
DIST-SW1(config-if)# ipv6 address fd00:10:10:99::2/64
DIST-SW1(config-if)# ipv6 ospf 1 area 1
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# description ** Routed uplink to CORE-SW1 - Area 1 (v6) **
DIST-SW1(config-if)# ipv6 address fd00:10:10:ff10::2/64
DIST-SW1(config-if)# ipv6 ospf 1 area 1
DIST-SW1(config-if)# exit
```

### 263.5 Verify เร็ว — Neighbor ขึ้น FULL แล้ว

```
CORE-SW1# show ipv6 ospf neighbor

Neighbor ID     Pri   State           Dead Time   Interface ID   Interface
1.1.1.2           0   FULL/  -        00:00:38    6              Port-channel1
1.1.1.11          0   FULL/  -        00:00:33    6              TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:39    6              TenGigabitEthernet1/0/2
```

สังเกตว่า `show ipv6 ospf neighbor` แสดง **Interface ID** (เลขภายในที่ IOS ใช้ระบุ Interface แทน
IP Address) แทนที่จะแสดง Neighbor Address ตรงๆ แบบ `show ip ospf neighbor` ของ IPv4 — นี่คือผล
จากแนวคิด Per-Link ใน Step 261.2 ที่ OSPFv3 ไม่ได้ผูก Adjacency กับ IP Address ใดๆ โดยตรง (ใช้
Link-Local เป็น Next-Hop เบื้องหลัง แต่ตัว Neighbor Table อ้างอิงด้วย Router ID + Interface ID
เป็นหลัก)

---

## Step 264 — LSA Type ของ OSPFv3: Link-LSA และ Intra-Area-Prefix-LSA ที่เพิ่มเข้ามาใหม่

### 264.1 หลักการสำคัญ: แยก "Topology" ออกจาก "Prefix"

จุดเปลี่ยนที่สำคัญที่สุดของ LSA ใน OSPFv3 คือการ **แยกข้อมูล Topology (ใครต่อกับใคร, Cost เท่าไหร่)
ออกจากข้อมูล Prefix (Subnet/Address อะไรอยู่บน Link นั้น) โดยสิ้นเชิง** — OSPFv2 รวมสองอย่างนี้ไว้
ใน LSA เดียวกัน (Router-LSA มีทั้ง "ต่อกับใคร" และ "Subnet อะไร") แต่ OSPFv3 แยกเป็นคนละ LSA:
Router-LSA/Network-LSA เหลือแค่ Topology ล้วนๆ ส่วน Prefix ทั้งหมดย้ายไปอยู่ **Intra-Area-Prefix-LSA
(Type 9)** ต่างหาก

### 264.2 ตาราง LSA Type ของ OSPFv3 (เทียบกับ OSPFv2)

| Type | ชื่อ (OSPFv3) | เทียบเท่า OSPFv2 | Originate โดย | Scope การ Flood | หน้าที่ |
|---|---|---|---|---|---|
| 1 | Router-LSA | Router-LSA (Type 1) | ทุก Router | Area | บอก Topology ล้วนๆ (ต่อกับใคร, Cost เท่าไหร่) — **ไม่มี Prefix ปนอยู่แล้ว** |
| 2 | Network-LSA | Network-LSA (Type 2) | DR เท่านั้น (Broadcast/NBMA) | Area | บอกว่า Router ตัวไหนอยู่บน Multi-access Segment เดียวกันบ้าง — **ไม่มี Prefix ปนอยู่แล้วเช่นกัน** |
| 3 | Inter-Area-Prefix-LSA | Summary LSA (Type 3) | ABR | ข้าม Area (Area 0 ↔ Area อื่น) | สรุป Prefix จาก Area หนึ่งส่งให้อีก Area รู้จัก (ทำหน้าที่เดียวกับ Type 3 เดิมทุกประการ แค่เปลี่ยนชื่อให้ตรงความหมาย) |
| 4 | Inter-Area-Router-LSA | ASBR Summary LSA (Type 4) | ABR | ข้าม Area | บอกตำแหน่ง/Cost ไปถึง ASBR ให้ Area อื่นรู้ |
| 5 | AS-External-LSA | AS-External LSA (Type 5) | ASBR | ทั้ง OSPF Domain (ยกเว้น Stub/NSSA) | Route จากนอก OSPF (Redistribute) |
| 7 | NSSA-External-LSA | NSSA External LSA (Type 7) | ASBR ภายใน NSSA | ภายใน NSSA Area นั้น | เหมือน Type 5 แต่ใช้เฉพาะใน NSSA (ABR แปลงเป็น Type 5 ก่อนส่งออก) |
| **8** | **Link-LSA** | **ไม่มีใน OSPFv2 (ใหม่)** | ทุก Router, ต่อ 1 ใบ/Interface | **Link-Local เท่านั้น** (ไม่ข้าม Router แม้แต่ Hop เดียว) | ประกาศ Link-Local Address ของตัวเอง + รายการ IPv6 Prefix ที่ Configure บน Link นั้น + Option bit สำหรับ DR Election |
| **9** | **Intra-Area-Prefix-LSA** | **ไม่มีใน OSPFv2 (ใหม่)** | ทุก Router (และ DR สำหรับ Broadcast Segment) | Area | บรรจุ **Prefix ทั้งหมด** ที่เคยฝังอยู่ใน Router-LSA/Network-LSA ของ OSPFv2 — แยกออกมาเป็น LSA เดี่ยว |

### 264.3 ทำไมต้องแยก Link-LSA ออกมาต่างหาก

**Link-LSA (Type 8)** ทำ 2 หน้าที่พร้อมกัน:

1. **บอก Link-Local Address ของตัวเองให้ Neighbor บน Link เดียวกันรู้** — จำเป็นเพราะ Adjacency ของ
   OSPFv3 ใช้ Link-Local เป็น Next-Hop เสมอ (Step 261.2) Router ต้องรู้ Link-Local ของ Neighbor
   ก่อนจะ Route ผ่านมันได้จริง
2. **ประกาศรายการ IPv6 Prefix ที่ Interface นั้นมี** ให้ DR (ถ้าเป็น Broadcast Segment) นำไปรวมเข้า
   Intra-Area-Prefix-LSA ต่อ

จุดสำคัญคือ Link-LSA มี **Scope แคบที่สุดในบรรดา LSA ทั้งหมด** — **Flood แค่บน Link เดียวเท่านั้น
ไม่ข้าม Router แม้แต่ Hop เดียว** (ต่างจาก Area-scope ของ Router/Network/Prefix LSA) เพราะ
Link-Local Address ไม่มีความหมายอะไรนอก Link นั้นอยู่แล้ว การ Flood ไปไกลกว่านั้นจึงไม่มีประโยชน์

### 264.4 ตัวอย่าง `show ipv6 ospf database`

```
CORE-SW1# show ipv6 ospf database

            OSPFv3 1 address-family ipv6 (router-id 1.1.1.1)

                Router Link States (Area 0)
ADV Router      Age         Seq#         Fragment ID  Link count  Bits
1.1.1.1         205         0x80000006   0            1           B
1.1.1.2         198         0x80000005   0            1           B

                Net Link States (Area 0)
ADV Router      Age         Seq#         Link ID
1.1.1.1         205         0x80000002   6

                Inter Area Prefix Link States (Area 0)
ADV Router      Age         Seq#         Prefix
1.1.1.1         30          0x80000003   FD00:10:10::/48
1.1.1.2         28          0x80000003   FD00:10:10::/48

                Link (Type-8) Link States (Area 0)
ADV Router      Age         Seq#         Link ID  Interface
1.1.1.1         205         0x80000001   6        Port-channel1
1.1.1.2         198         0x80000001   6        Port-channel1

                Intra Area Prefix Link States (Area 0)
ADV Router      Age         Seq#         Link ID  Ref-lstype  Ref-LSID
1.1.1.1         205         0x80000002   0        0x2001      0
1.1.1.2         198         0x80000002   0        0x2001      0
```

เทียบกับ `show ip ospf database` ของ OSPFv2 (Part 11 Step 109.3) จะเห็นว่ามี Section เพิ่มมา 2
ส่วนคือ **Link (Type-8)** และ **Intra Area Prefix** — ส่วน **Router Link States** ตอนนี้ไม่มีคอลัมน์
"Link count" ที่หมายถึง Subnet เหมือนเดิมอีกแล้ว (แปลว่า Prefix ถูกแยกออกไปตามที่อธิบายใน 264.1)

---

## Step 265 — OSPFv3 Authentication ผ่าน IPsec (AH/ESP)

### 265.1 ทำไมต้องเปลี่ยนวิธี Authentication ทั้งหมด

Step 261.3 กล่าวไปแล้วว่า OSPFv3 **ตัด Authentication Field ออกจาก Packet Header ทั้งหมด** —
เหตุผลคือทีมออกแบบ OSPFv3 มองว่า **IPv6 มีกลไก Security ของตัวเองอยู่แล้วในระดับ Network Layer**
(IPsec ถูกออกแบบให้เป็นส่วนหนึ่งของ IPv6 มาตั้งแต่ต้น ตามที่กล่าวถึงใน
[Part 8 Step 71](part-008-ipv6-fundamentals.md#step-71--ทำไมต้อง-ipv6-และรูปแบบ-address)) จึงไม่
จำเป็นต้องมี Authentication Mechanism ซ้ำซ้อนอยู่ในตัว Routing Protocol เองแบบ OSPFv2 (MD5 ใน
Part 11 Step 108) — ให้ OSPFv3 Packet ทุกใบถูกห่อหุ้ม (หรือยืนยันตัวตน) ด้วย **IPsec AH
(Authentication Header)** หรือ **ESP (Encapsulating Security Payload)** ที่ระดับ IPv6 แทน (RFC 4552)

| | IPsec AH | IPsec ESP |
|---|---|---|
| ยืนยันตัวตน (Authentication) | ✅ | ✅ |
| เข้ารหัสเนื้อหา (Encryption) | ❌ | ✅ (Optional) |
| ใช้กับ OSPFv3 ส่วนใหญ่ | ✅ (นิยมกว่า เพราะ OSPF ไม่ต้องการปกปิดเนื้อหา Routing Info) | ใช้ได้เช่นกันถ้าต้องการเข้ารหัสด้วย |

### 265.2 Manual Keying — ไม่ต้องมี IKE/ISAKMP เต็มรูปแบบ

ข่าวดีคือ Cisco IOS **ไม่บังคับให้ต้องตั้ง IPsec Tunnel/IKE Phase 1-2 เต็มรูปแบบ** เหมือนงาน VPN
ทั่วไป — สำหรับ OSPFv3 ใช้วิธี **Manual Keying** (กำหนด SPI และ Key ตรงๆ ไม่มีการแลกเปลี่ยน Key
อัตโนมัติ) ทำให้ Config สั้นและใกล้เคียงกับ MD5 Authentication ของ OSPFv2 มาก:

```
Router(config-if)# ipv6 ospf authentication ipsec spi <spi> {md5 | sha1} <key>
```

| Parameter | ความหมาย |
|---|---|
| `spi` | **Security Parameter Index** — เลข 256 ถึง 4294967295 ใช้ระบุว่า Key ชุดนี้คือชุดไหน ต้อง**ตรงกันทั้งสองฝั่งของ Link** |
| `md5 <key>` | ใช้ MD5 เป็น Hash Algorithm — Key ต้องเป็น **Hex String ยาว 32 ตัวอักษร (128 bit)** |
| `sha1 <key>` | ใช้ SHA-1 (แข็งแรงกว่า MD5) — Key ต้องเป็น **Hex String ยาว 40 ตัวอักษร (160 bit)** |

### 265.3 คอนฟิกจริง — MD5 ผ่าน IPsec บน Backbone Link

```
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
CORE-SW1(config-if)# exit

CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
CORE-SW2(config-if)# exit
```

> **ข้อสำคัญเหมือน MD5 ของ OSPFv2 (Part 11 Step 108.5) เป๊ะ**: SPI และ Key **ต้องตรงกันทั้งสองฝั่ง
> ของ Link เดียวกัน** (Case-sensitive สำหรับ Key) ถ้าไม่ตรง Adjacency จะไม่ก่อตัวเลย พร้อม Syslog
> แจ้งเตือน:
> ```
> %OSPFv3-4-ERRRCV: Received invalid packet: Auth Failed from FE80::21A:2FFF:FE3C:0102, Port-channel1
> ```

### 265.4 Area-Wide Authentication — ตั้งครั้งเดียวทั้ง Area

เหมือน OSPFv2 (Part 11 Step 108.4) OSPFv3 ก็รองรับการตั้ง Authentication แบบ Area-wide เพื่อบังคับ
ทุก Interface ใน Area นั้นให้ต้องมี Key (ยังต้องตั้ง Key ต่อ Interface อยู่ดี แค่ "บังคับเปิดใช้"
จาก Process แทนที่จะเปิดทีละ Interface):

```
CORE-SW1(config)# ipv6 router ospf 1
CORE-SW1(config-rtr)# area 0 authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
```

> **หลักสูตรนี้เลือกใช้วิธี Per-Interface** เหมือนที่เลือกใช้กับ OSPFv2 มาตลอด เพื่อความชัดเจนว่า
> Interface ไหนเปิด Authentication อยู่จริงบ้าง — Verify ด้วย
> `show ipv6 ospf interface Port-channel1 | include Auth`

---

## Step 266 — OSPFv3 Address-Family: รวม IPv4 และ IPv6 ไว้ใน Process เดียว

### 266.1 ที่มา — RFC 5838

Step 261.4 เกริ่นไว้ว่า OSPFv3 สามารถ Route **ทั้ง IPv4 และ IPv6** ได้ในตัวเดียวกัน — Cisco IOS
Implement แนวคิดนี้ผ่าน Syntax ใหม่ **`router ospfv3`** (ต่างจาก `ipv6 router ospf` แบบคลาสสิกที่ใช้
มาตั้งแต่ Step 263) ซึ่งรองรับ **Address-Family Sub-mode** คล้ายกับที่ BGP ใช้ (`address-family
ipv4`/`address-family vpnv4` ที่จะเรียนใน Part 30) — ต้องใช้ IOS 15.1(3)S / 15.2(1)T ขึ้นไป (หรือ
IOS-XE รุ่นปัจจุบันของ Lab นี้) จึงจะมีคำสั่งชุดนี้

### 266.2 Syntax เต็ม

```
Router(config)# router ospfv3 <process-id>
Router(config-router)# router-id <router-id>
Router(config-router)#  address-family ipv4 unicast
Router(config-router-af)# exit-address-family
Router(config-router)#  address-family ipv6 unicast
Router(config-router-af)# exit-address-family
!
Router(config)# interface <interface-id>
Router(config-if)# ospfv3 <process-id> ipv4 area <area-id>
Router(config-if)# ospfv3 <process-id> ipv6 area <area-id>
```

สังเกต 2 จุดที่ต่างจาก Step 263:

1. คำสั่งใต้ Global Config เปลี่ยนจาก `ipv6 router ospf` เป็น **`router ospfv3`** (ไม่มี `ipv6`
   นำหน้าแล้ว เพราะ Process เดียวรองรับทั้งสอง AF ไม่ใช่ของ IPv6 ล้วนอีกต่อไป)
2. คำสั่งใต้ Interface เปลี่ยนจาก `ipv6 ospf <process> area <id>` เป็น **`ospfv3 <process> ipv4|ipv6
   area <id>`** — ต้องระบุ AF ให้ชัดว่าเปิดสำหรับ IPv4 หรือ IPv6 (เปิดได้ทั้งสองบรรทัดพร้อมกันบน
   Interface เดียว ถ้า Interface นั้น Dual-Stack)

### 266.3 ตัวอย่าง — CORE-SW1 แบบ Unified Address-Family (ทางเลือกสมัยใหม่)

```
CORE-SW1(config)# router ospfv3 1
CORE-SW1(config-router)# router-id 1.1.1.1
CORE-SW1(config-router)#  address-family ipv4 unicast
CORE-SW1(config-router-af)#   passive-interface Loopback0
CORE-SW1(config-router-af)#  exit-address-family
CORE-SW1(config-router)#  address-family ipv6 unicast
CORE-SW1(config-router-af)#   passive-interface Loopback0
CORE-SW1(config-router-af)#  exit-address-family
CORE-SW1(config-router)# exit
!
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# ip address 10.255.0.1 255.255.255.252
CORE-SW1(config-if)# ipv6 address fd00:10:10:ff00::1/64
CORE-SW1(config-if)# ospfv3 1 ipv4 area 0
CORE-SW1(config-if)# ospfv3 1 ipv6 area 0
```

**บรรทัดเดียว `ospfv3 1 ipv4 area 0` + `ospfv3 1 ipv6 area 0`** แทนที่ทั้ง `ip ospf 1 area 0` (จาก
Part 12) และ `ipv6 ospf 1 area 0` (จาก Step 263) รวมกัน — Process เดียว, Router-ID เดียว, จุด
บริหารจัดการเดียว

### 266.4 Verify แบบ Unified

```
CORE-SW1# show ospfv3 1 ipv4 neighbor
CORE-SW1# show ospfv3 1 ipv6 neighbor
CORE-SW1# show ospfv3 1 ipv6 database
```

คำสั่ง Verify ต้องระบุ AF (`ipv4`/`ipv6`) ต่อท้าย Process ID เสมอ เพราะแต่ละ AF ยังคง **แยก LSDB
กันโดยสมบูรณ์** (การรวม Process ไม่ได้แปลว่า Routing Table ของ IPv4/IPv6 ถูก "รวม" กันเป็นอันเดียว
— ยังคำนวณ SPF แยกกันสองชุดเหมือนเดิม เพียงแค่บริหารจัดการ Config/Router-ID ร่วมกันเท่านั้น)

---

## Step 267 — Route Summarization (`area range`) และ Stub Area บน OSPFv3

### 267.1 `area range` — Syntax เหมือน OSPFv2 ทุกประการ

```
Router(config)# ipv6 router ospf <process-id>
Router(config-rtr)# area <area-id> range <ipv6-prefix>/<prefix-length> [not-advertise]
```

หลักการเหมือน Part 12 Step 115 เป๊ะ: Config บน **ABR เท่านั้น**, สรุป Prefix ที่เป็น Intra-area
ของ Area ที่ระบุก่อนที่จะสร้าง Inter-Area-Prefix-LSA (Type 3) ข้ามไปยัง Area อื่น

### 267.2 Worked Example — สรุป VLAN ของแต่ละ Area เข้า Area 0

เนื่องจาก VLAN ทั้งหมดของหลักสูตรนี้ (VLAN 10/20/30/40/99) อยู่ภายใต้ Super-block เดียวกันคือ
`fd00:10:10::/48` (เหมือนกับที่ IPv4 ใช้ `10.10.0.0/16` เป็น Super-block ครอบทุก VLAN) การสรุปจึง
ทำได้ด้วย Prefix เดียวกันทั้งสอง ABR:

```
! CORE-SW1 (ABR ของ Area 1)
CORE-SW1(config)# ipv6 router ospf 1
CORE-SW1(config-rtr)# area 1 range fd00:10:10::/48

! CORE-SW2 (ABR ของ Area 2)
CORE-SW2(config)# ipv6 router ospf 1
CORE-SW2(config-rtr)# area 2 range fd00:10:10::/48
```

```
CORE-SW1# show ipv6 ospf database

                Inter Area Prefix Link States (Area 0)
ADV Router      Age         Seq#         Prefix
1.1.1.1         12          0x80000004   FD00:10:10::/48       <- Summary จาก Area 1
1.1.1.2         10          0x80000004   FD00:10:10::/48       <- Summary จาก Area 2
```

> **คำเตือนเดียวกับ Part 12 Step 115.2**: ทั้ง Area 1 และ Area 2 ต่างก็ Advertise **VLAN 99
> (MGMT)** อยู่ (ทั้ง DIST-SW1/2 และ DIST-SW3/4 มี Vlan99) และ Prefix `fd00:10:10::/48` ที่ทั้งสอง
> ABR ใช้สรุปนั้น**ครอบคลุมทับซ้อนกันทั้งสอง Area โดยตั้งใจ** เพื่อความง่ายในการ Config (เช่นเดียว
> กับที่ IPv4 ใช้ `10.10.0.0/16` ครอบทั้ง Area 1/2 ทับกัน) — Traffic ยังคง Forward ถูกทางเพราะ
> Longest-Prefix-Match ยังใช้ Intra-Area Route ที่แม่นยำกว่าเสมอเมื่ออยู่ใน Area เดียวกัน ปัญหานี้
> จะเกิดเฉพาะตอน Debug/อ่าน LSDB ที่อาจดู "ซ้ำ" กว่าที่ควรเท่านั้น ไม่กระทบ Data Plane

### 267.3 Backbone Link — สรุปแบบซ่อนไม่ให้ Advertise เลย

เหมือน Part 12 Step 115.3 ถ้าต้องการซ่อน Backbone Transit Prefix ไม่ให้รั่วไหลข้าม Area (ไม่มี
ประโยชน์ที่ Area อื่นจะรู้จัก Prefix ของ Link ภายใน Area 1/2 โดยตรง) ใช้ `not-advertise`:

```
CORE-SW1(config-rtr)# area 1 range fd00:10:10:ff10::/64 not-advertise
CORE-SW1(config-rtr)# area 1 range fd00:10:10:ff20::/64 not-advertise
```

### 267.4 Stub Area และ Totally Stub — คำสั่งเดียวกับ OSPFv2

```
! Stub Area ธรรมดา (ต้องตั้งทุก Router ใน Area นั้นให้ตรงกัน — เหมือน OSPFv2 Part 12 Step 116)
Router(config-rtr)# area 2 stub

! Totally Stub — เพิ่ม no-summary ที่ฝั่ง ABR เท่านั้น
CORE-SW2(config-rtr)# area 2 stub no-summary

! NSSA
Router(config-rtr)# area 2 nssa
```

พฤติกรรมเหมือน OSPFv2 ทุกประการ (ตาราง Stub/Totally-Stub/NSSA/Totally-NSSA ใน Part 12 Step
116.1) — ต่างแค่ว่า Type ของ LSA ที่ถูก Block คือ Type 5 (AS-External) / Type 3 (Inter-Area-Prefix)
ของ OSPFv3 แทน — เหตุผลเดียวกับ Part 12 ที่บอกว่า **Area 1/2 ของ Lab นี้ยังไม่เหมาะเป็น Stub**
(เพราะยังไม่มี ASBR/External Route ให้ต้อง Block) ยังคงใช้ได้กับ OSPFv3 เช่นกัน — Step 270 จึงยัง
ไม่เปิด Stub ใน Lab เต็มรูปแบบ

---

## Step 268 — ออกแบบ Dual-Stack Routing: แยก Process vs รวม Address-Family

เมื่อ Network ต้อง Route ทั้ง IPv4 (ด้วย OSPFv2) และ IPv6 (ด้วย OSPFv3) พร้อมกัน มีสอง Approach
หลักให้เลือก — Step นี้สรุปข้อดี/ข้อเสียของแต่ละแบบเพื่อประกอบการตัดสินใจออกแบบจริง

### 268.1 Approach A — แยก Process อิสระ (OSPFv2 Classic + OSPFv3 Classic คู่ขนาน)

คือสิ่งที่ Step 263 ทำอยู่: `router ospf 1` (IPv4, จาก Part 11/12) และ `ipv6 router ospf 1` (IPv6,
Step 263) รันคู่ขนานกันแบบไม่รู้จักกันเลย

| ข้อดี | ข้อเสีย |
|---|---|
| แต่ละ Protocol Debug/Troubleshoot แยกกันชัดเจน ไม่ปนกัน (`show ip ospf` vs `show ipv6 ospf`) | ต้อง Config 2 ชุดคู่ขนานเสมอ — เสี่ยง **Asymmetric Topology** ถ้าลืมเปิด IPv6 บาง Interface (Neighbor IPv4 ขึ้นแต่ IPv6 ไม่ขึ้น หรือกลับกัน) |
| เสถียร/เป็นที่รู้จักมานาน (Mature — ใช้งานจริงในองค์กรส่วนใหญ่ปัจจุบัน) | LSDB และ SPF Calculation แยกกัน 2 ชุด = **ภาระ CPU/Control-Plane เพิ่มเป็นสองเท่า** บน Router ทุกตัว |
| รองรับบน IOS แทบทุกรุ่น (Feature เก่า, ไม่มีปัญหาความเข้ากันได้กับ Platform) | ต้องจำ Syntax คู่ขนาน 2 ชุด (`network`/`ip ospf area` คู่กับ `ipv6 ospf area`) — เพิ่มโอกาส Human Error |
| แยก Fault Domain — ปัญหาของ IPv6 OSPFv3 (เช่น LSDB Corrupt) ไม่กระทบ IPv4 เลย | Router ID ทั้งสอง Process อาจตั้งไม่ตรงกันได้ถ้าไม่ระวัง (แม้ในหลักสูตรนี้จะตั้งให้ตรงกันเสมอ) |

### 268.2 Approach B — OSPFv3 Address-Family รวม Process เดียว (Step 266)

| ข้อดี | ข้อเสีย |
|---|---|
| Config/บริหารจัดการจุดเดียว — Router-ID เดียว, `router ospfv3 1` เดียวคุมทั้งสอง AF | ต้องใช้ **IOS/IOS-XE รุ่นใหม่เท่านั้น** (15.1(3)S/15.2(1)T ขึ้นไป) — Platform เก่าบางรุ่นไม่รองรับ |
| รับประกัน **Topological Symmetry** ได้ง่ายกว่า — เห็น 2 บรรทัด `ospfv3 1 ipv4/ipv6 area` คู่กันใต้ Interface เดียว ลดโอกาสลืมเปิดฝั่งใดฝั่งหนึ่ง | Syntax ใหม่ (`router ospfv3`, `address-family`) ทีมงานที่คุ้นกับ OSPFv2/OSPFv3 Classic ต้องเรียนรู้เพิ่ม |
| Feature ระดับ Process (เช่น Graceful Restart, BFD ที่จะเรียนใน Part ถัดไป) Config ครั้งเดียวใช้ได้ทั้งสอง AF | **Single Point of Misconfiguration** — คำสั่งผิดพลาดใน Process เดียวอาจกระทบทั้ง IPv4 และ IPv6 พร้อมกัน (เช่น `clear ospfv3 process` ที่ไม่ระบุ AF จะ Restart ทั้งคู่) |
| เตรียมพร้อมสำหรับ Network ที่กำลังจะเลิกใช้ IPv4 ทั้งหมดในอนาคต (ค่อยๆ ปิด AF ipv4 ทีหลังโดยไม่ต้องรื้อ Process) | ยังคง**แยก LSDB/SPF สองชุดเหมือนเดิม** (Step 266.4) — ไม่ได้ลด CPU Overhead จริงๆ แค่ลดภาระ "การบริหารจัดการ Config" เท่านั้น |

### 268.3 คำแนะนำของหลักสูตรนี้

Lab หลักของหลักสูตร (ต่อเนื่องมาจาก Part 11/12 ที่มี OSPFv2 ทำงานอยู่แล้วเต็มรูปแบบ) เลือกใช้
**Approach A (แยก Process)** เป็นแนวทางหลักใน Step 270 เพราะ **ไม่ต้อง Migrate OSPFv2 ที่ทำงานอยู่
แล้วทิ้ง** และ Concept ชัดเจนเหมาะกับการสอนแยกทีละ Protocol — แต่ให้จำ Approach B (Step 266) ไว้
เป็นทางเลือกสำหรับ **Greenfield Design** (Network ที่เริ่มออกแบบใหม่ทั้งหมด ไม่มี OSPFv2 เดิมให้
ต้องรักษาไว้) หรือองค์กรที่ต้องการ Long-term Roadmap ไปสู่ IPv6-only ในอนาคต — ข้อสอบ CCNP ENCOR
มักถามเปรียบเทียบสอง Approach นี้โดยตรง จึงต้องเข้าใจ Trade-off ทั้งสองฝั่งให้แม่น

---

## Step 269 — คำสั่ง Verify/Troubleshoot OSPFv3

### 269.1 ตารางคำสั่งหลัก

| คำสั่ง | ใช้ตรวจสอบอะไร | เทียบเท่า OSPFv2 |
|---|---|---|
| `show ipv6 ospf neighbor` | สถานะ Adjacency ทั้งหมด | `show ip ospf neighbor` |
| `show ipv6 ospf interface [if]` | Timer, Cost, Network Type, Area, Authentication ราย Interface | `show ip ospf interface` |
| `show ipv6 ospf database` | LSDB ทั้งหมด (แยกตาม LSA Type ใน Step 264) | `show ip ospf database` |
| `show ipv6 route ospf` | Route ที่เรียนรู้จาก OSPFv3 เท่านั้น | `show ip route ospf` |
| `show ipv6 protocols` | สรุป Process, Router-ID, Area ที่ Active อยู่ | `show ip protocols` |
| `show ipv6 ospf border-routers` | ABR/ASBR ที่มองเห็น | `show ip ospf border-routers` |
| `debug ipv6 ospf adj` | Debug กระบวนการสร้าง Adjacency Real-time | `debug ip ospf adj` |

### 269.2 ตัวอย่าง `show ipv6 protocols`

```
CORE-SW1# show ipv6 protocols
IPv6 Routing Protocol is "connected"
IPv6 Routing Protocol is "ND"
IPv6 Routing Protocol is "ospf 1"
  Router ID 1.1.1.1
  Number of areas: 2 normal, 0 stub, 0 nssa
  Interfaces (Area 0):
    Port-channel1
  Interfaces (Area 1):
    TenGigabitEthernet1/0/1
    TenGigabitEthernet1/0/2
  Passive interface(s):
    Loopback0
  Redistribution:
    None
```

Field ที่ต้องเช็คทุกครั้งตอน Troubleshoot ภาพรวม: **Router ID** (ต้องตรงกับที่ตั้งใจ), **Number of
areas** (จำนวน Area ตรงตาม Design ไหม — ในที่นี้ CORE-SW1 ควรเห็น 2 Area คือ 0 กับ 1), และ
**Passive interface(s)** (ยืนยันว่า Loopback0 ไม่ได้พยายามสร้าง Neighbor)

### 269.3 ตัวอย่าง `show ipv6 ospf interface`

```
CORE-SW1# show ipv6 ospf interface Port-channel1
Port-channel1 is up, line protocol is up
  Link Local Address FE80::21A:2FFF:FE3C:0101, Interface ID 6
  Area 0, Process ID 1, Instance ID 0, Router ID 1.1.1.1
  Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 1.1.1.2, local address FE80::21A:2FFF:FE3C:0202
  Backup Designated router (ID) 1.1.1.1, local address FE80::21A:2FFF:FE3C:0101
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Hello due in 00:00:03
  Graceful restart helper support enabled
  Index 1/1/1, flag 0x0
  Next 0x0(0)/0x0(0)/0x0(0)
  Last flood scan length is 1, maximum is 1
  Last flood scan time is 0 msec, maximum is 0 msec
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 1.1.1.2  (Backup Designated Router)
  Suppress hello for 0 neighbor(s)
```

สังเกตบรรทัด **Link Local Address** และ **Instance ID** ที่ปรากฏชัดเจน — สองอย่างนี้ไม่มีใน
`show ip ospf interface` ของ OSPFv2 เลย เป็นหลักฐานตรงจากคำสั่งที่ยืนยัน Concept ของ Step 261-262
ได้ทันที นอกจากนี้ยังสังเกตว่า **Backbone Link ที่ไม่ได้บังคับ `network point-to-point` จะยังเป็น
BROADCAST และมี DR/BDR Election** เหมือนพฤติกรรมของ OSPFv2 (Part 11 Step 105) — ถ้าต้องการหลีกเลี่ยง
Overhead เดียวกันนี้ ใช้คำสั่ง `ipv6 ospf network point-to-point` ได้เช่นเดียวกับ OSPFv2

### 269.4 Checklist Troubleshoot OSPFv3 ที่ไม่ขึ้น Full

1. `show run | include ipv6 unicast-routing` — เปิดหรือยัง (ปัญหาที่พบบ่อยที่สุด เหมือน Part 8 Step 76)
2. `show ipv6 interface brief` — Interface มี Link-Local Address หรือไม่ (ถ้าไม่มีเลย OSPFv3 สร้าง Neighbor ไม่ได้แน่นอน)
3. `show ipv6 ospf neighbor` — ค้างที่ State ไหน (ใช้หลักการเดียวกับ Part 11 Step 109.1: ค้างที่ INIT/EXSTART = Timer หรือ Authentication Mismatch)
4. `show ipv6 ospf interface <if>` — เช็ค Area, Instance ID, Timer, Authentication ให้ตรงกันทั้งสองฝั่ง
5. `debug ipv6 ospf adj` — ดู Real-time ว่า Reject เพราะเหตุผลอะไร (Area Mismatch, Instance ID Mismatch, Auth Failed)

---

## Step 270 — Lab เต็มรูปแบบ: OSPFv3 3-Area พร้อม Authentication และ Summarization

Lab สุดท้ายของ Part นี้จะ Deploy OSPFv3 3-Area (Topology เดียวกับ OSPFv2 ทุกประการ) ให้ครบทั้ง
**CORE-SW1, CORE-SW2, DIST-SW1–4** พร้อม **IPsec Authentication** บน Backbone Link ทุกเส้น และ
**Route Summarization** บนทั้งสอง ABR แล้ว Verify ปิดท้ายด้วยการพิสูจน์ว่า **OSPFv2 (IPv4) และ
OSPFv3 (IPv6) วิ่งคู่ขนานกันได้สมบูรณ์แบบ Dual-Stack**

### 270.1 Topology และ Address Plan เต็ม

```
                    Area 1                              Area 0                          Area 2
     (fd00:10:10:10/20/99::/64)              (fd00:10:10:ff00::/64)          (fd00:10:10:30/40/99::/64)
┌──────────┐                          ┌──────────────────────────────┐                          ┌──────────┐
│ DIST-SW1 │──Te1/1/1 (Area1,v6)──┐  │                                │  ┌──Te1/1/1 (Area2,v6)──│ DIST-SW3 │
└──────────┘                       │  │                                │  │                        └──────────┘
                             ┌─────┴──┴┐  Po1 ::1 ═══ Po1 ::2          ┌┴──┴─────┐
                             │ CORE-SW1 │  fd00:10:10:ff00::/64 (Area0) │ CORE-SW2 │
                             │  (ABR)   │  IPsec SPI 500 (MD5)          │  (ABR)   │
                             └─────┬────┘                              └────┬─────┘
┌──────────┐                       │  area 1 range fd00:10:10::/48          │      ┌──────────┐
│ DIST-SW2 │──Te1/1/1 (Area1,v6)──┘  (สรุป VLAN10/20/99 → Area 0)          └──Te1/1/1 (Area2,v6)──│ DIST-SW4 │
└──────────┘                                                          area 2 range fd00:10:10::/48 └──────────┘
                                                                        (สรุป VLAN30/40/99 → Area 0)
```

### 270.2 ตาราง Address เต็ม (IPv6 ULA — ส่วนขยายของ Part นี้)

| Device | Interface | Area | IPv6 Address |
|---|---|---|---|
| CORE-SW1 | Port-channel1 | 0 | `fd00:10:10:ff00::1/64` |
| CORE-SW1 | TenGigabitEthernet1/0/1 (↔DIST-SW1) | 1 | `fd00:10:10:ff10::1/64` |
| CORE-SW1 | TenGigabitEthernet1/0/2 (↔DIST-SW2) | 1 | `fd00:10:10:ff20::1/64` |
| CORE-SW2 | Port-channel1 | 0 | `fd00:10:10:ff00::2/64` |
| CORE-SW2 | TenGigabitEthernet1/0/1 (↔DIST-SW3) | 2 | `fd00:10:10:ff30::1/64` |
| CORE-SW2 | TenGigabitEthernet1/0/2 (↔DIST-SW4) | 2 | `fd00:10:10:ff40::1/64` |
| DIST-SW1 | Vlan10 / Vlan20 / Vlan99 | 1 | `fd00:10:10:10::2/64` / `fd00:10:10:20::2/64` / `fd00:10:10:99::2/64` |
| DIST-SW1 | TenGigabitEthernet1/1/1 (↔CORE-SW1) | 1 | `fd00:10:10:ff10::2/64` |
| DIST-SW2 | Vlan10 / Vlan20 / Vlan99 | 1 | `fd00:10:10:10::3/64` / `fd00:10:10:20::3/64` / `fd00:10:10:99::3/64` |
| DIST-SW2 | TenGigabitEthernet1/1/1 (↔CORE-SW1) | 1 | `fd00:10:10:ff20::2/64` |
| DIST-SW3 | Vlan30 / Vlan40 / Vlan99 | 2 | `fd00:10:10:30::2/64` / `fd00:10:10:40::2/64` / `fd00:10:10:99::2/64` |
| DIST-SW3 | TenGigabitEthernet1/1/1 (↔CORE-SW2) | 2 | `fd00:10:10:ff30::2/64` |
| DIST-SW4 | Vlan30 / Vlan40 / Vlan99 | 2 | `fd00:10:10:30::3/64` / `fd00:10:10:40::3/64` / `fd00:10:10:99::3/64` |
| DIST-SW4 | TenGigabitEthernet1/1/1 (↔CORE-SW2) | 2 | `fd00:10:10:ff40::2/64` |

Router-ID ของ OSPFv3 ใช้ค่าเดียวกับ OSPFv2 ทุกตัว (Part 11 Step 103.5): CORE-SW1=`1.1.1.1`,
CORE-SW2=`1.1.1.2`, DIST-SW1=`1.1.1.11`, DIST-SW2=`1.1.1.12`, DIST-SW3=`1.1.1.13`,
DIST-SW4=`1.1.1.14` — IPsec Authentication ใช้ SPI `500`, MD5 Key `0123456789ABCDEF0123456789ABCDEF`
เหมือนกันทุก Backbone Link (ตามธรรมเนียมเดียวกับ MD5 Key เดียวทั้ง Backbone ใน Part 11 Step 108.5)

### 270.3 Running-config เต็ม — CORE-SW1

```
hostname CORE-SW1
!
ip routing
ipv6 unicast-routing
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW2 - Area 0 **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 ipv6 address fd00:10:10:ff00::1/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Routed downlink to DIST-SW1 - Area 1 **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 1
 ipv6 address fd00:10:10:ff10::1/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 1
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Routed downlink to DIST-SW2 - Area 1 **
 no switchport
 ip address 10.255.20.1 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 1
 ipv6 address fd00:10:10:ff20::1/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 1
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.1 255.255.255.255
!
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 100000
 area 1 range 10.10.0.0 255.255.0.0
 passive-interface Loopback0
!
ipv6 router ospf 1
 router-id 1.1.1.1
 area 1 range fd00:10:10::/48
 passive-interface Loopback0
!
end
```

### 270.4 Running-config เต็ม — CORE-SW2

```
hostname CORE-SW2
!
ip routing
ipv6 unicast-routing
!
interface Port-channel1
 description ** Routed backbone link to CORE-SW1 - Area 0 **
 no switchport
 ip address 10.255.0.2 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 0
 ipv6 address fd00:10:10:ff00::2/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 0
 no shutdown
!
interface TenGigabitEthernet1/0/1
 description ** Routed downlink to DIST-SW3 - Area 2 **
 no switchport
 ip address 10.255.30.1 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 2
 ipv6 address fd00:10:10:ff30::1/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 2
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** Routed downlink to DIST-SW4 - Area 2 **
 no switchport
 ip address 10.255.40.1 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 2
 ipv6 address fd00:10:10:ff40::1/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 2
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.2 255.255.255.255
!
router ospf 1
 router-id 1.1.1.2
 auto-cost reference-bandwidth 100000
 area 2 range 10.10.0.0 255.255.0.0
 passive-interface Loopback0
!
ipv6 router ospf 1
 router-id 1.1.1.2
 area 2 range fd00:10:10::/48
 passive-interface Loopback0
!
end
```

### 270.5 Running-config เต็ม — DIST-SW1 (Area 1, VLAN 10/20/99)

```
hostname DIST-SW1
!
ip routing
ipv6 unicast-routing
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
 ip ospf 1 area 1
 ipv6 address fd00:10:10:10::2/64
 ipv6 ospf 1 area 1
 no shutdown
!
interface Vlan20
 description ** Gateway VLAN20-VOICE **
 ip address 10.10.20.2 255.255.255.0
 ip ospf 1 area 1
 ipv6 address fd00:10:10:20::2/64
 ipv6 ospf 1 area 1
 no shutdown
!
interface Vlan99
 description ** Gateway VLAN99-MGMT **
 ip address 10.10.99.2 255.255.255.0
 ip ospf 1 area 1
 ipv6 address fd00:10:10:99::2/64
 ipv6 ospf 1 area 1
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW1 - Area 1 **
 no switchport
 ip address 10.255.10.2 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 1
 ipv6 address fd00:10:10:ff10::2/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 1
 no shutdown
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.11 255.255.255.255
!
router ospf 1
 router-id 1.1.1.11
 auto-cost reference-bandwidth 100000
 passive-interface Loopback0
!
ipv6 router ospf 1
 router-id 1.1.1.11
 passive-interface Loopback0
!
end
```

### 270.6 DIST-SW2 — ใช้ Pattern เดียวกับ DIST-SW1 ทุกประการ

**DIST-SW2** Config เหมือน DIST-SW1 ทุกบรรทัด (270.5) เปลี่ยนแค่ 4 จุดตามตาราง Address Plan
(270.2) และ Loopback0 Plan (Part 11 Step 103.5):

| จุดที่เปลี่ยน | DIST-SW1 | DIST-SW2 |
|---|---|---|
| `hostname` | DIST-SW1 | DIST-SW2 |
| Host Bit ของทุก VLAN SVI (IPv4 `.2`/IPv6 `::2`) | `.2` / `::2` | `.3` / `::3` |
| Uplink Interface | `fd00:10:10:ff10::2/64` (ไปยัง Te1/0/1 ของ CORE-SW1) | `fd00:10:10:ff20::2/64` (ไปยัง Te1/0/2 ของ CORE-SW1) |
| Router ID (ทั้ง `router ospf` และ `ipv6 router ospf`) | `1.1.1.11` | `1.1.1.12` |

Area, VLAN ที่รับผิดชอบ (10/20/99), MD5/IPsec Key, และโครงสร้างคำสั่งอื่นทั้งหมด**เหมือนกันทุก
ประการ** — Pattern นี้สอดคล้องกับที่ Part 12 Step 120.4 ใช้มาแล้วสำหรับ OSPFv2

### 270.7 Running-config เต็ม — DIST-SW3 (Area 2, VLAN 30/40/99, Host `.2`)

```
hostname DIST-SW3
!
ip routing
ipv6 unicast-routing
!
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
interface Vlan30
 ip address 10.10.30.2 255.255.255.0
 ip ospf 1 area 2
 ipv6 address fd00:10:10:30::2/64
 ipv6 ospf 1 area 2
 no shutdown
!
interface Vlan40
 ip address 10.10.40.2 255.255.255.0
 ip ospf 1 area 2
 ipv6 address fd00:10:10:40::2/64
 ipv6 ospf 1 area 2
 no shutdown
!
interface Vlan99
 ip address 10.10.99.2 255.255.255.0
 ip ospf 1 area 2
 ipv6 address fd00:10:10:99::2/64
 ipv6 ospf 1 area 2
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** Routed uplink to CORE-SW2 - Area 2 **
 no switchport
 ip address 10.255.30.2 255.255.255.252
 ip ospf message-digest-key 1 md5 OSPF-BACKBONE-K3y!
 ip ospf authentication message-digest
 ip ospf 1 area 2
 ipv6 address fd00:10:10:ff30::2/64
 ipv6 ospf authentication ipsec spi 500 md5 0123456789ABCDEF0123456789ABCDEF
 ipv6 ospf 1 area 2
 no shutdown
!
interface Loopback0
 ip address 1.1.1.13 255.255.255.255
!
router ospf 1
 router-id 1.1.1.13
 auto-cost reference-bandwidth 100000
 passive-interface Loopback0
!
ipv6 router ospf 1
 router-id 1.1.1.13
 passive-interface Loopback0
!
end
```

### 270.8 DIST-SW4 — ใช้ Pattern เดียวกับ DIST-SW3 ทุกประการ

**DIST-SW4** Config เหมือน DIST-SW3 ทุกบรรทัด (270.7) เปลี่ยนแค่ 4 จุดเดียวกับที่อธิบายใน 270.6:

| จุดที่เปลี่ยน | DIST-SW3 | DIST-SW4 |
|---|---|---|
| `hostname` | DIST-SW3 | DIST-SW4 |
| Host Bit ของทุก VLAN SVI (IPv4 `.2`/IPv6 `::2`) | `.2` / `::2` | `.3` / `::3` |
| Uplink Interface | `fd00:10:10:ff30::2/64` (ไปยัง Te1/0/1 ของ CORE-SW2) | `fd00:10:10:ff40::2/64` (ไปยัง Te1/0/2 ของ CORE-SW2) |
| Router ID | `1.1.1.13` | `1.1.1.14` |

### 270.9 Verify #1 — Neighbor ทั้งสอง Protocol บน CORE-SW1

```
CORE-SW1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.2           0   FULL/  -        00:00:38    10.255.0.2      Port-channel1
1.1.1.11          0   FULL/  -        00:00:33    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          0   FULL/  -        00:00:39    10.255.20.2     TenGigabitEthernet1/0/2

CORE-SW1# show ipv6 ospf neighbor

Neighbor ID     Pri   State           Dead Time   Interface ID   Interface
1.1.1.2           1   FULL/BDR        00:00:32    6              Port-channel1
1.1.1.11          1   FULL/BDR        00:00:35    6              TenGigabitEthernet1/0/1
1.1.1.12          1   FULL/BDR        00:00:31    6              TenGigabitEthernet1/0/2
```

ทั้งสองคำสั่งเห็น Neighbor RID **ชุดเดียวกันเป๊ะ** (1.1.1.2, 1.1.1.11, 1.1.1.12) เพราะ Topology
เหมือนกันทุกประการ ต่างแค่ Protocol และ LSDB คนละชุด — ยืนยันว่า Design "Mirror" ระหว่าง OSPFv2
กับ OSPFv3 ทำได้สำเร็จตามที่ตั้งใจไว้ตั้งแต่ Step 263

### 270.10 Verify #2 — Route Summarization ทำงานทั้งสอง Protocol

```
DIST-SW1# show ip route ospf
     10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
O IA    10.10.0.0/16 [110/20] via 10.255.10.1, 00:05:12, TenGigabitEthernet1/1/1

DIST-SW1# show ipv6 route ospf
OI  FD00:10:10::/48 [110/20]
     via FE80::21A:2FFF:FE3C:0101, TenGigabitEthernet1/1/1
```

Prefix Summary `10.10.0.0/16` (IPv4) และ `FD00:10:10::/48` (IPv6) ปรากฏเป็น **Route เดียว** แทนที่
จะเห็นแยกทีละ VLAN — ยืนยันว่า `area range` ทำงานถูกต้องทั้งสอง Address-Family ตาม Step 267

### 270.11 Verify #3 — Authentication ทำงาน

```
CORE-SW1# show ip ospf interface Port-channel1 | include Cryptographic
  Cryptographic authentication enabled

CORE-SW1# show ipv6 ospf interface Port-channel1 | include IPsec
  IPsec AH, SPI 500, Algorithm MD5
```

### 270.12 Verify #4 — Dual-Stack Ping/Traceroute ข้าม Area จริง (DIST-SW1 → Server1 บน VLAN30)

```
! IPv4 ผ่าน OSPFv2 — Server1 อยู่ VLAN30 (10.10.30.0/24), Area 2
DIST-SW1# ping 10.10.30.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.30.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms

DIST-SW1# traceroute 10.10.30.10
Type escape sequence to abort.
Tracing the route to 10.10.30.10

  1 10.255.10.1 2 msec 1 msec 1 msec        <- CORE-SW1
  2 10.255.30.2 3 msec 3 msec 2 msec        <- CORE-SW2 (ข้าม Area 0 ผ่าน Po1)
  3 10.10.30.10 4 msec 4 msec 3 msec        <- Server1

! IPv6 ผ่าน OSPFv3 — Server1 มี IPv6 ULA fd00:10:10:30::10 บน VLAN เดียวกัน
DIST-SW1# ping fd00:10:10:30::10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FD00:10:10:30::10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/3/5 ms

DIST-SW1# traceroute ipv6 fd00:10:10:30::10
Type escape sequence to abort.
Tracing the route to FD00:10:10:30::10

  1 FE80::21A:2FFF:FE3C:0101 2 msec 2 msec 1 msec     <- CORE-SW1 (Link-Local เป็น Next-Hop)
  2 FE80::21A:2FFF:FE3D:0202 3 msec 3 msec 3 msec     <- CORE-SW2
  3 FD00:10:10:30::10 5 msec 4 msec 4 msec             <- Server1
```

**ผลลัพธ์ที่ต้องสังเกต**: `traceroute` ของ IPv4 แสดง Hop เป็น Global/ULA Address ตามปกติ แต่
`traceroute ipv6` แสดง Hop กลางทาง (CORE-SW1, CORE-SW2) เป็น **Link-Local Address (`FE80::`)**
ทั้งหมด — เป็นหลักฐานยืนยัน Step 261.2 ตรงๆ ว่า **OSPFv3 ใช้ Link-Local เป็น Next-Hop เสมอ** ต่างจาก
OSPFv2 ที่ใช้ Global IPv4 Address เป็น Next-Hop โดยตรง แม้ Path ที่ Traffic เดินทางจริงจะเป็นเส้น
ทางเดียวกันทุกประการก็ตาม (ผ่าน CORE-SW1 → Po1 → CORE-SW2 → DIST-SW3 เหมือนกัน)

---

## แบบฝึกหัดทวนความเข้าใจ Part 27

1. เพราะเหตุใด OSPFv3 จึงใช้ **Link-Local Address** เป็น Next-Hop เสมอ แทนที่จะใช้ Global/ULA
   Address เหมือนที่ OSPFv2 ใช้ Global IPv4 Address?
2. LSA Type ใหม่ 2 ตัวที่ไม่มีใน OSPFv2 คืออะไร และแต่ละตัวมีหน้าที่ต่างกันอย่างไร?
3. ทำไม OSPFv3 Router ID ยังต้องเป็น 32-bit Dotted-Decimal ทั้งที่ไม่ได้ Route IPv4 เลย และทำไม
   ต้อง Config แบบ Explicit เสมอในองค์กรที่เป็น IPv6-only?
4. OSPFv3 ใช้กลไกอะไรแทน MD5 Authentication ของ OSPFv2 และคำสั่งหลักที่ใช้ Config บน Cisco IOS
   คืออะไร?
5. อธิบายข้อดี-ข้อเสียของการรัน OSPFv2 และ OSPFv3 แบบแยก Process อิสระ เทียบกับการใช้ OSPFv3
   Address-Family รวม Process เดียว

**เฉลย:**

1. เพราะ OSPFv3 ยึดแนวคิด **Per-Link** (Step 261.2) — Link เดียวสามารถมีหลาย Global/ULA Prefix
   พร้อมกันได้ (ต่าง Router อาจใช้คนละ Prefix บน Link เดียวกันในทางทฤษฎี) แต่ **Link-Local Address
   รับประกันว่ามีอยู่เสมอและไม่ซ้ำกันภายใน Link เดียวกัน** จึงเป็นตัวเลือกที่มั่นคงที่สุดสำหรับใช้
   เป็น Next-Hop โดยไม่ขึ้นกับว่า Admin จะ Config Global/ULA Address แบบไหนบน Interface นั้น
2. **Link-LSA (Type 8)** — ประกาศ Link-Local Address ของตัวเอง + รายการ Prefix บน Link นั้น
   Flood แค่ระดับ Link เดียว (ไม่ข้าม Router เลย); **Intra-Area-Prefix-LSA (Type 9)** — บรรจุ
   Prefix ทั้งหมดที่แยกออกมาจาก Router-LSA/Network-LSA เดิม เพื่อแยกข้อมูล Topology ออกจาก Prefix
   อย่างชัดเจน (Step 264)
3. เพราะ Router ID เป็น Field ที่นิยามโดยตัว OSPF Protocol เอง ไม่ได้ผูกกับ IP Version ที่ Route
   อยู่ (แค่นิยมเขียนแบบ Dotted-Decimal เพื่อให้อ่านง่าย) — และองค์กร IPv6-only ที่ไม่มี IPv4
   Address อยู่บนอุปกรณ์เลยจะไม่มี Candidate ให้ IOS เลือก RID อัตโนมัติได้เลย (จะ Error
   "Could not pick a router-id") จึงต้อง Config `router-id` แบบ Explicit เสมอ (Step 262.2)
4. ใช้ **IPsec (AH/ESP)** ตาม RFC 4552 แทนที่ Authentication Field ที่ถูกถอดออกจาก Packet Header
   — คำสั่งหลักคือ `ipv6 ospf authentication ipsec spi <spi> {md5 | sha1} <key>` Config ที่
   Interface (หรือ `area <id> authentication ipsec spi ...` แบบ Area-wide) (Step 265)
5. **แยก Process**: ข้อดีคือ Debug แยกชัดเจน เสถียร รองรับ IOS รุ่นเก่า แยก Fault Domain; ข้อเสีย
   คือเสี่ยง Asymmetric Topology และภาระ CPU เพิ่มเป็นสองเท่า **รวม Address-Family**: ข้อดีคือ
   บริหารจัดการจุดเดียว ลดโอกาสลืมเปิด AF ใดฝั่งหนึ่ง เตรียมพร้อมสำหรับอนาคต IPv6-only; ข้อเสียคือ
   ต้องใช้ IOS รุ่นใหม่ และเป็น Single Point of Misconfiguration ที่กระทบทั้งสอง AF พร้อมกันได้
   (Step 268)

---

## สรุป Part 27

Part นี้พา OSPF ที่คุ้นเคยจาก Part 11/12 ไปสู่โลก IPv6 ด้วย **OSPFv3 (RFC 5340)** โดยคง Algorithm,
Area Design, และ Topology 3-Area เดิมไว้ทุกประการ พร้อมเรียนรู้สิ่งที่เปลี่ยนไปจริง: แนวคิด
**Per-Link** และ **Link-Local เป็น Next-Hop**, **Instance ID**, LSA ใหม่ **Link-LSA/Intra-Area-Prefix-LSA**,
Authentication ผ่าน **IPsec** แทน MD5 ในตัว Protocol, และทางเลือกสมัยใหม่ **OSPFv3 Address-Family**
ที่รวม IPv4/IPv6 ไว้ Process เดียว — ปิดท้ายด้วย Lab ที่พิสูจน์ว่า **OSPFv2 และ OSPFv3 วิ่งคู่ขนาน
กันแบบ Dual-Stack ได้อย่างสมบูรณ์** บน Topology เดียวกัน

**พร้อมสำหรับ Part 28**: เราจะเปลี่ยนไปดู Routing Protocol อีกตระกูลหนึ่งที่ Cisco เป็นเจ้าของเอง
— **EIGRP Advanced** (Named Mode, Unequal-Cost Load Balancing ด้วย Variance, Feasibility
Condition เชิงลึก, EIGRP for IPv6) เพื่อเปรียบเทียบปรัชญาการออกแบบ Distance-Vector สมัยใหม่กับ
Link-State ที่เพิ่งเรียนจบไปใน Part นี้

**ไปต่อ:** [Part 28 — EIGRP Advanced →](part-028-eigrp-advanced.md)
