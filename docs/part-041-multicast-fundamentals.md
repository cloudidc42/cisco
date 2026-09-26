# Part 41 — Multicast Fundamentals (IGMP, PIM-DM/SM)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 401–410 จาก 1000**

> ต่อจาก [Part 40 — Advanced QoS](part-040-advanced-qos.md) ที่ปิดท้ายเรื่อง Traffic Shaping,
> Policing และ QoS Policy บน Campus Network ของเราแล้ว Part นี้จะเปิดหัวข้อใหม่ทั้งหมดของ
> CCNP ENCOR คือ **IP Multicast** — เทคโนโลยีที่ทำให้ Router/Switch ส่งข้อมูลชุดเดียวไปถึงผู้รับ
> จำนวนมากพร้อมกันได้อย่างมีประสิทธิภาพ โดยไม่ต้องทำสำเนาข้อมูลซ้ำที่ต้นทาง (Unicast) และไม่ต้อง
> flood ไปทุกที่ (Broadcast) เราจะเรียนตั้งแต่ทฤษฎีพื้นฐานของ Multicast, IGMP (โปรโตคอลระหว่าง
> Host กับ Router แรก), ไปจนถึง PIM (โปรโตคอล Routing ของ Multicast ระหว่าง Router) ทั้ง
> Dense Mode และ Sparse Mode พร้อม Lab เต็มรูปแบบบน Topology หลักของหลักสูตร

## Use Case ที่จะขับเคลื่อน Part นี้: All-Hands Video Streaming

ทีม HR ของบริษัทต้องการจัดงาน **All-Hands Meeting** ทุกเดือน โดยถ่ายทอดสดวิดีโอจาก Server
ตัวใหม่ที่ชื่อ **STREAM-SRV** (`10.10.30.30`, อยู่ใน **VLAN 30 SERVERS** ตาม
[`00-ip-address-plan.md`](00-ip-address-plan.md)) ไปให้พนักงานในหลายแผนกรับชมพร้อมกันแบบ
Real-time — พนักงานฝ่ายขาย (VLAN 10 SALES) และพนักงานที่เชื่อมต่อผ่าน Wi-Fi (VLAN 40 WIFI)
ต้องดูวิดีโอตัวเดียวกันในเวลาเดียวกัน

ทีมงานลองคิด 2 วิธีที่มีอยู่แล้วในหลักสูตร (Part 1-40) แต่พบว่าทั้งคู่มีปัญหา:

```
วิธีที่ 1: Unicast (ส่งทีละคน)                วิธีที่ 2: Broadcast (ส่งทุกคนในทุก Subnet)

   STREAM-SRV                                     STREAM-SRV
   10.10.30.30                                     10.10.30.30
      │  ├──► Stream ไป PC-SALES1  (copy 1)           │
      │  ├──► Stream ไป PC-SALES2  (copy 2)            ▼  broadcast (255.255.255.255)
      │  ├──► Stream ไป PC-WIFI1   (copy 3)      ┌─────────────────┐
      │  └──► Stream ไป PC-WIFI2   (copy 4)      │ ไปได้แค่ VLAN30  │  ← ไม่ข้าม VLAN/Subnet
      │                                          │ เท่านั้น         │     เลยด้วยซ้ำ! และถึงข้าม
   ปัญหา: NIC/Uplink ของ Server ต้องส่ง          └─────────────────┘     ได้ก็จะ flood ไปหา
   ข้อมูลซ้ำ N ชุด (N = จำนวนผู้ชม) — ถ้ามี                                     ทุกเครื่องแม้เครื่อง
   ผู้ชม 500 คนดู Video 4 Mbps = ต้องการ                                       ที่ไม่ได้ดู Video เลย
   Bandwidth ขาออกถึง 2 Gbps จาก Server เดียว
```

**Multicast คือคำตอบ**: STREAM-SRV ส่งข้อมูลออกไปเพียง **ชุดเดียว** ไปยัง Group Address พิเศษ
(`239.1.1.1` ที่เราจะเลือกใช้ใน Lab) แล้วปล่อยให้ **Router/Switch ในเครือข่ายเป็นผู้ทำสำเนา**
เฉพาะที่จุดแยกสาย (Fork Point) ที่จำเป็นเท่านั้น — ส่งไปถึงทุก VLAN ที่มีผู้ต้องการดูจริง โดยไม่แตะ
VLAN ที่ไม่มีใครดูเลย นี่คือหัวใจของ Part นี้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 401 | Multicast คืออะไร — Unicast vs Broadcast vs Multicast และ Class D Address Range |
| 402 | การแปลง Multicast IP เป็น Ethernet Multicast MAC Address (01:00:5E:xx:xx:xx) |
| 403 | IGMP (Internet Group Management Protocol) v1/v2/v3 และ IGMP Snooping บน Switch |
| 404 | PIM คืออะไร — ทำไมต้องมี Multicast Routing Protocol แยกจาก IGMP |
| 405 | PIM Dense Mode (PIM-DM) — Flood-and-Prune Model |
| 406 | PIM Sparse Mode (PIM-SM) — Explicit-Join, Rendezvous Point, Shared Tree vs SPT |
| 407 | Basic PIM-SM Configuration — `ip multicast-routing`, `ip pim sparse-mode`, Static RP |
| 408 | Multicast Forwarding และ RPF Check — (S,G) และ (*,G) State |
| 409 | Verification & Troubleshooting — `show ip mroute`, `show ip pim neighbor`, `show ip igmp groups` |
| 410 | Lab เต็มรูปแบบ: PIM-SM Deployment ส่ง All-Hands Video Stream ข้าม Campus จริง |

---

## Step 401 — Multicast คืออะไร: Unicast vs Broadcast vs Multicast

### เปรียบเทียบ 3 รูปแบบการส่งข้อมูล (ทบทวนแนวคิดจาก [Part 1](part-001-networking-fundamentals.md))

| รูปแบบ | ผู้รับ | Address ที่ใช้ | ประสิทธิภาพเมื่อมีผู้รับหลายคน |
|---|---|---|---|
| **Unicast** | 1 ต้นทาง → 1 ปลายทาง | Host address ปกติ | แย่ — ต้นทางต้องส่งซ้ำทีละสำเนาต่อผู้รับ (N ผู้รับ = N เท่าของ Bandwidth) |
| **Broadcast** | 1 ต้นทาง → ทุกเครื่องใน Subnet/VLAN | `255.255.255.255` หรือ Directed Broadcast | สิ้นเปลือง — ส่งไปแม้เครื่องที่ไม่ต้องการข้อมูลก็ต้องประมวลผล และไม่ข้าม Subnet/VLAN โดย Design (Router ไม่ forward broadcast ข้าม Interface) |
| **Multicast** | 1 ต้นทาง → **กลุ่มผู้รับที่สมัครเป็นสมาชิก (Group)** | Class D (`224.0.0.0` – `239.255.255.255`) | ดีที่สุด — ต้นทางส่งครั้งเดียว, Network ทำสำเนาเฉพาะจุดแยกสายที่จำเป็น, ข้าม Subnet/VLAN ได้ถ้ามี Multicast Routing (PIM) |

```
Unicast (N=3 ผู้รับ)          Broadcast (VLAN เดียว)         Multicast (Group Member เท่านั้น)

  SRV                            SRV                              SRV
   │╲                             │                                │
   │ ╲___R1                       ▼ (flood ทุก port ใน VLAN)        ▼  (ส่งครั้งเดียว)
   │  ╲__R2                  ┌─────────┐                       ┌─────────┐
   │   ╲_R3                  │ Switch  │                       │ Router  │──copy──► Group Member A
  3 สำเนาจาก SRV              └─┬─┬─┬───┘                       └────┬────┘
  (Bandwidth = 3x)          ทุกเครื่องรับหมด                        │ (copy เฉพาะจุดที่มี
                            แม้ไม่ได้ต้องการ                         │  member หรือ router
                                                                     ▼  downstream เท่านั้น)
                                                              ┌─────────┐
                                                              │non-memberไม่ได้รับ│
                                                              └─────────┘
```

### Class D Address Range — พื้นที่อยู่ของ Multicast

ทบทวนจาก [Part 1 Step 8](part-001-networking-fundamentals.md#step-8--ip-address-พื้นฐาน) ตาราง
IPv4 Class ระบุไว้แล้วว่า **Class D = Multicast** ครอบคลุมช่วง:

```
224.0.0.0  ────────────────────────────────────────────────  239.255.255.255
   11100000.00000000.00000000.00000000     11101111.11111111.11111111.11111111
   └─┬─┘ 4 bit แรกตายตัว = 1110               28 bit ที่เหลือ = Group ID
```

4 bit แรกของ Octet 1 ถูกกำหนดตายตัวเป็น `1110` เสมอ — ทำให้ Multicast **ไม่มี Subnet Mask
แบบ Unicast** (ไม่มี Network/Host แบ่งแบบ Class A/B/C) เพราะทั้ง 224.0.0.0-239.255.255.255
คือ "Group Address" ล้วนๆ ไม่ใช่ Address ของอุปกรณ์จริงตัวใดตัวหนึ่ง

### แบ่งช่วงย่อยตามการใช้งาน (Multicast Address Scope)

| ช่วง | ชื่อเรียก | การใช้งาน |
|---|---|---|
| `224.0.0.0` – `224.0.0.255` | Local Network Control Block | Protocol ควบคุมภายใน Subnet เดียว (TTL=1 เสมอ, Router ไม่ forward ข้าม Interface) |
| `224.0.1.0` – `224.0.1.255` | Internetwork Control Block | Protocol ที่ต้อง forward ข้าม Network ได้ (เช่น NTP multicast) |
| `232.0.0.0/8` | SSM (Source-Specific Multicast) Block | ใช้กับ IGMPv3/PIM-SSM (จะเรียนใน Part ถัดไป) |
| `233.0.0.0/8` | GLOP Addressing | Map จาก AS Number ขององค์กร (RFC 3180) |
| `239.0.0.0/8` | Administratively Scoped (Private) | เหมือน RFC 1918 ของ Multicast — ใช้ได้เองภายในองค์กร ไม่ routing ออก Internet — **Lab ของเราใช้ `239.1.1.1` จากช่วงนี้** |

### ตาราง Well-Known Reserved Multicast Address ที่ต้องจำ

| Multicast Address | ใช้โดย | อธิบาย |
|---|---|---|
| `224.0.0.1` | All-Hosts | ทุกอุปกรณ์ที่รองรับ Multicast ใน Subnet (Host + Router) |
| `224.0.0.2` | All-Routers | ทุก Router ใน Subnet |
| `224.0.0.5` | OSPF | All OSPF Routers (DR/BDR ใช้รับ Hello/LSA) — [Part 11](part-011-ospfv2-fundamentals.md) |
| `224.0.0.6` | OSPF | All OSPF DR/BDR Routers (ใช้ส่ง LSA ไปหา DR) — [Part 11](part-011-ospfv2-fundamentals.md) |
| `224.0.0.9` | RIPv2 | All RIPv2 Routers |
| `224.0.0.10` | EIGRP | All EIGRP Routers — [Part 13](part-013-eigrp-fundamentals.md) |
| `224.0.0.13` | PIM | All PIM Routers (ใช้ Hello ระหว่าง PIM Neighbor — จะเจอใน Step 407) |
| `224.0.0.102` | HSRPv2 | HSRP Hello Message เวอร์ชัน 2 — ตรงกับที่เรียนไปแล้วใน [Part 18](part-018-fhrp.md#step-175--hsrpv1-vs-hsrpv2) |

> **จุดที่ต้องสังเกต**: Address ในตาราง Well-Known ทั้งหมดอยู่ในช่วง `224.0.0.0/24` (Local Network
> Control Block) และมี TTL=1 เสมอ — Router ปกติ (ไม่ใช่ PIM Router) จะ **ไม่ route** Multicast
> พวกนี้ข้าม Interface โดย Design ซึ่งถูกต้องแล้ว เพราะ Protocol เหล่านี้ (OSPF Hello, EIGRP Hello,
> HSRP) ต้องทำงานเฉพาะ Local Segment เท่านั้น — ต่างจาก Video Stream ของ STREAM-SRV ที่ **ต้อง**
> ข้าม VLAN/Subnet ได้ ซึ่งเป็นเหตุผลที่เราต้องมี PIM (Step 404 เป็นต้นไป)

---

## Step 402 — การแปลง Multicast IP เป็น Ethernet Multicast MAC Address

การจะส่ง Multicast Packet บน Ethernet Segment ได้ Switch/NIC ต้องรู้จัก **Destination MAC
Address** ก่อน — Cisco/IEEE จึงกำหนดสูตรแปลง Class D IP Address ให้เป็น Ethernet Multicast
MAC โดยอัตโนมัติ ไม่ต้องมี ARP (เพราะ Multicast ไม่มีเจ้าของตัวเดียวให้ ARP หา)

### กฎการ Mapping

```
IANA จองช่วง OUI  01:00:5E:00:00:00 – 01:00:5E:7F:FF:FF  ไว้สำหรับ IPv4 Multicast ทั้งหมด
(ครึ่งบนของ 01:00:5E:xx:xx:xx คือ 25 bit คงที่ + 23 bit ที่เหลือมาจาก IP)

IP Multicast Address (32 bit):     1110 XXXX . XXXXXXXX . XXXXXXXX . XXXXXXXX
                                          └──────────── 28 bit Group ID ──────────┘
                                               │
                                     ตัด 5 bit สูงสุดทิ้ง (ไม่ใช้)
                                               │
                                               ▼
                                     เอาแค่ 23 bit ล่างสุด ────► ยัดเข้า MAC

Ethernet Multicast MAC (48 bit):   0000 0001 . 0000 0000 . 0101 1110 . 0??????? . ???????? . ????????
                                    └────────── 25 bit คงที่ (01:00:5E + 1 bit) ──────────┘  └── 23 bit จาก IP ──┘
```

### Worked Example: แปลง `239.1.1.1` เป็น Multicast MAC

| ขั้นตอน | ค่า |
|---|---|
| 1. IP Address | `239.1.1.1` |
| 2. แปลงเป็น Binary | `11101111.00000001.00000001.00000001` |
| 3. เอา 28 bit ท้าย (ตัด 4 bit Class D prefix ทิ้ง) | `1111.00000001.00000001.00000001` |
| 4. เอาแค่ 23 bit ล่างสุด (ตัด 5 bit บนสุดของ 28 bit ทิ้งอีกที) | `0000001.00000001.00000001` |
| 5. ยัดเข้า 23 bit ล่างของ `01:00:5E:00:00:00` | `01:00:5E:` + `01`(บิตบนสุด=0 เสมอ) + `:01:01` |
| **ผลลัพธ์** | **`01:00:5E:01:01:01`** |

เพราะ Octet 2-4 ของ `239.1.1.1` คือ `1.1.1` ซึ่งมีค่าน้อย (ไม่เกิน 127 ในแต่ละ byte แรก) การ Mapping
ในตัวอย่างนี้จึงตรงไปตรงมา — ไม่มี bit ชนกันให้เห็นชัด แต่ในทางทฤษฎี **5 bit ที่ถูกตัดทิ้งไปในขั้นตอน
ที่ 4 ทำให้เกิดปัญหา Overlap**

### ปัญหา 32:1 Ambiguity/Overlap

เนื่องจากใช้แค่ **23 จาก 28 bit** ของ Group ID เท่านั้น (bit ที่เหลือ 5 bit ถูกทิ้งไปเฉยๆ) จึงมี
**2⁵ = 32 IP Multicast Address ที่ต่างกัน แต่แปลงเป็น MAC เดียวกัน** เช่น:

```
224.1.1.1   ┐
225.1.1.1   │
226.1.1.1   ├──►  ทุก IP นี้แปลงเป็น MAC เดียวกันคือ 01:00:5E:01:01:01
...         │     (เพราะ Octet แรกต่างกันเฉพาะ 5 bit ที่ถูกตัดทิ้ง)
239.1.1.1   ┘
```

**ทำไมปัญหานี้แทบไม่ส่งผลกระทบในทางปฏิบัติ**:
1. NIC/Switch ที่ได้รับ Frame ปลายทาง MAC นี้ จะส่งขึ้นไปให้ Layer 3 (IP) ตรวจสอบ **Destination
   IP Address ที่แท้จริง** อีกครั้งเสมอ — ถ้า Host ไม่ได้ Join Group IP นั้นจริง มันจะ drop packet ที่ IP
   Layer แม้ MAC จะตรงก็ตาม (แค่เสีย CPU cycle เพิ่มขึ้นนิดหน่อยในการเปิด packet ดู)
2. Enterprise ส่วนใหญ่ใช้ Address จาก `239.0.0.0/8` (Administratively Scoped) ซึ่งมี Prefix
   ที่ต่างจาก Well-known Protocol Address (`224.0.0.x`) มากพอที่แผน Address ภายในจะไม่ชนกันเอง
   ถ้าวางแผน Group Address ดี (เหมือนวางแผน VLAN/Subnet)
3. IGMP Snooping (Step 403) ทำงานที่ **Layer 2 แต่อ้างอิงเนื้อหา IGMP ที่ Layer 3** อยู่แล้ว จึงไม่ได้
   พึ่งพา MAC Address เพียงอย่างเดียวในการตัดสินใจ forward/drop

---

## Step 403 — IGMP (Internet Group Management Protocol)

**IGMP** คือโปรโตคอลที่ทำงานระหว่าง **Host กับ Router ตัวแรกใน Subnet เดียวกัน (Last-Hop
Router)** เท่านั้น — ใช้บอกว่า "ฉันต้องการรับ Multicast Group นี้" หรือ "ฉันเลิกรับแล้ว"
**IGMP ไม่ใช่ Routing Protocol** และไม่ทำงานข้าม Subnet — งานส่ง Multicast ข้าม Subnet เป็นหน้าที่
ของ PIM (Step 404)

### เปรียบเทียบ IGMPv1 / v2 / v3

| คุณสมบัติ | IGMPv1 | IGMPv2 | IGMPv3 |
|---|---|---|---|
| Membership Report | มี | มี | มี (พร้อม Source Filtering) |
| Leave Group Message | **ไม่มี** (ต้องรอ Timeout ~3 นาที) | **มี** (Leave ทันที, เร็วขึ้นมาก) | มี |
| Query จาก Router | มีแต่ไม่มี Group-Specific Query | มี Group-Specific Query (ถามเจาะเฉพาะ Group) | มี พร้อม Source-Specific Query |
| Source Filtering (SSM) | ไม่รองรับ | ไม่รองรับ | **รองรับ** — เลือกรับจาก Source ที่กำหนดเท่านั้น (`(S,G)` โดยตรงจาก Host) |
| ค่า Default บน Cisco IOS ปัจจุบัน | - | - | **IGMPv2** (ต้องเปิดเองถ้าต้องการ v3/SSM) |

### กระบวนการทำงานของ IGMP (v2)

```
   Host A                          Router (Querier)
     │                                   │
     │──── Membership Report ──────────►│   "ฉันต้องการ Join Group 239.1.1.1"
     │        (Join Group)               │
     │                                   │   Router สร้าง/เพิ่ม Group ใน Local Table
     │                                   │   แล้วแจ้ง PIM (Step 404) ให้เริ่มสร้าง Tree
     │                                   │
     │◄──── General Query (ทุก 60s) ─────│   "ยังมีใครสนใจ Group ไหนอยู่บ้าง?" (Keepalive)
     │──── Membership Report ───────────►│   ตอบกลับยืนยันว่ายังต้องการอยู่
     │       (ตอบ Query)                  │
     │                                   │
     │──── Leave Group ─────────────────►│   "ฉันเลิกรับแล้ว" (IGMPv2 เท่านั้นที่มี message นี้)
     │                                   │   Router ส่ง Group-Specific Query ถามซ้ำก่อนลบ
     │                                   │   (เผื่อมี Host อื่นยังต้องการ Group เดียวกันอยู่)
```

- **Membership Report**: Host ส่งเข้ามาเพื่อ Join Group (Multicast ไปที่ Group เอง หรือ
  `224.0.0.22` สำหรับ v3)
- **Query**: Router (คนที่ได้รับเลือกเป็น **IGMP Querier** — ปกติคือ Router ที่มี IP ต่ำสุดใน Subnet)
  ถามเป็นระยะว่ายังมีใคร Join อยู่หรือไม่ ถ้าไม่มีใครตอบ Report ภายในเวลาที่กำหนด Router จะลบ
  Group นั้นออกจาก Table (Soft-state)
- **Leave**: มีเฉพาะ v2/v3 — ทำให้ Router รู้ทันทีว่ามีคน leave แทนที่ต้องรอ Timeout

### IGMP Snooping — ป้องกัน Switch Flood Multicast ไปทุก Port

ปัญหา: **Switch (Layer 2) โดย Default ไม่รู้จัก IP/IGMP เลย** เห็น Multicast MAC (Step 402)
เป็นแค่ Frame ปลายทางที่ไม่รู้จัก (Unknown Destination) จึง **Flood ไปทุก Port ใน VLAN เดียวกัน**
เหมือน Broadcast — ทำให้ Client ที่ไม่ได้ดู Video ก็โดน Flood รับ Traffic เข้า NIC ฟรีๆ (สิ้นเปลือง
Bandwidth บน Access Port ทุกจุด แม้จะไม่ได้ Join Group ก็ตาม)

**IGMP Snooping** คือ Feature ของ Switch ที่ "แอบดู (Snoop)" เนื้อหา IGMP Message ที่ผ่านตัวมัน
(แม้จะเป็น Layer 2 Device) แล้วสร้าง Table จับคู่ **Multicast Group ↔ Port ที่มี Member จริง**
จากนั้นจะ Forward Multicast เฉพาะ Port ที่มี Member (หรือ Port ที่ต่อไปยัง Router/Switch อื่น)
เท่านั้น — Port ที่ไม่มี Member จะไม่ได้รับ Traffic นี้เลย

```
ไม่มี IGMP Snooping (Flood ทุก Port)         มี IGMP Snooping (Forward เฉพาะ Port ที่ Join)

        STREAM-SRV                                   STREAM-SRV
            │                                             │
       ┌────┴────┐                                   ┌────┴────┐
       │ Switch  │──┬──┬──┬──┬── ทุก Port             │ Switch  │──┬──X──┬──X── เฉพาะ Port
       └─────────┘  │  │  │  │  ได้ Video หมด          └─────────┘  │     │     ที่มี Member
                    PC1 PC2 PC3 PC4                              PC1  PC2   PC3  PC4 Join จริง
                (แม้ PC2,PC4 ไม่ได้ดู)                        (Join)     (Join)
                                                            X = ไม่ได้รับ Traffic เลย
```

### Config: เปิด IGMP Snooping บน Switch (Cisco Catalyst)

```
! IGMP Snooping เปิด Global โดย Default อยู่แล้วบน Catalyst IOS/IOS-XE รุ่นใหม่
! แต่ควรตรวจสอบและระบุให้ชัดเจนเสมอในทุก Config Baseline

ACCESS-SW1(config)# ip igmp snooping
ACCESS-SW1(config)# ip igmp snooping vlan 10
ACCESS-SW1(config)# ip igmp snooping vlan 20
ACCESS-SW1(config)# ip igmp snooping vlan 99

! ตรวจสอบสถานะ
ACCESS-SW1# show ip igmp snooping
Global IGMP Snooping configuration:
-------------------------------------------
IGMP snooping                : Enabled
IGMPv3 snooping               : Enabled
Report suppression            : Enabled
TCN solicit query             : Disabled
Optimised Multicast flood     : Disabled
Vlan 10:
--------
IGMP snooping                : Enabled
IGMP snooping running mode    : IGMP
IGMP snooping querier         : Disabled
Explicit host tracking        : Enabled
```

> **หมายเหตุ**: บาง Platform (เช่น Catalyst 3560/3750 รุ่นเก่า) มี `ip igmp snooping` เปิด Global
> เป็น Default แต่ต้องเปิดใน Config อย่างชัดเจนเพื่อให้อยู่ใน Running-config เสมอ ตาม Best Practice
> ของหลักสูตรนี้ที่จะไม่พึ่งพา Default ที่มองไม่เห็นใน Config

---

## Step 404 — PIM คืออะไร: ทำไมต้องมี Multicast Routing Protocol แยกจาก IGMP

IGMP (Step 403) ตอบได้แค่คำถาม **"Host คนไหนใน Subnet นี้ต้องการ Group นี้บ้าง"** — แต่ไม่ได้
ตอบคำถามที่สำคัญกว่าคือ **"Router ตัวนี้ควร Forward Multicast Packet ออก Interface ไหนบ้าง
เพื่อไปถึง Subnet อื่นๆ ที่มี Member อยู่"** นี่คือช่องว่างที่ **PIM (Protocol Independent Multicast)**
เข้ามาเติมเต็ม

### เปรียบเทียบขอบเขตการทำงาน

```
                    ┌─────────────────────────────────────────────┐
                    │              Campus Network                  │
                    │                                               │
   STREAM-SRV       │   DIST-SW3 ◄──PIM──► CORE-SW1/2 ◄──PIM──► DIST-SW1  │   PC-SALES1
   VLAN 30 ─────────┤       ▲                                    ▲    ├────── VLAN 10
                    │       │ IGMP                          IGMP │    │
                    │   (Host↔Router                    (Router↔Host)│
                    │    Subnet เดียว)                   Subnet เดียว)│
                    └─────────────────────────────────────────────┘

   IGMP  = ทำงานเฉพาะ "ขอบ" ของ Network (Host ↔ Router ตัวแรก) — Local Subnet เท่านั้น
   PIM   = ทำงาน "ใน" Network (Router ↔ Router) — สร้าง Distribution Tree ข้าม Subnet ทั้งหมด
```

**หน้าที่ของ PIM**:
1. เรียนรู้จาก IGMP (ผ่าน Last-Hop Router) ว่า Subnet ใดมี Member ของ Group ใดอยู่บ้าง
2. สร้าง **Multicast Distribution Tree** — เส้นทางที่ข้อมูลจะไหลจาก Source ไปถึงทุก Subnet
   ที่มี Member โดยแยกสำเนา (Replicate) เฉพาะที่จุดแยกสาย (Fork Point)
3. ดูแลการ Forward Packet ตาม Tree นั้น พร้อมทำ **RPF Check** (จะเรียนใน Step 408) เพื่อป้องกัน Loop

ชื่อ "Protocol **Independent**" มาจากการที่ PIM **ไม่สร้าง Routing Table ของตัวเอง** (ไม่เหมือน
OSPF/EIGRP ที่คำนวณ Best Path เอง) แต่ **ยืม Unicast Routing Table ที่มีอยู่แล้ว** (จาก OSPF ใน
Lab ของเรา — [Part 11-12](part-011-ospfv2-fundamentals.md)) มาใช้ตัดสินใจว่า "ทางไหนคือทางกลับ
ไป Source" ซึ่งเป็นกลไกหลักของ RPF Check นั่นเอง

PIM มี 2 Mode หลักที่ต้องรู้สำหรับ CCNP ENCOR: **Dense Mode (Step 405)** และ **Sparse Mode
(Step 406)** — ทั้งสองใช้ Packet Format และ Neighbor Discovery เดียวกัน (Hello ไปที่ `224.0.0.13`)
แต่ต่างกันที่ **สมมติฐานเริ่มต้นว่ามี Receiver อยู่ที่ไหนบ้าง**

---

## Step 405 — PIM Dense Mode (PIM-DM): Flood-and-Prune Model

**PIM-DM** ใช้สมมติฐานตรงข้ามกับความเป็นจริงส่วนใหญ่ในองค์กร: **"สมมติว่ามี Receiver อยู่ทุกที่
จนกว่าจะพิสูจน์ว่าไม่มี"**

### กลไก Flood-and-Prune

```
รอบที่ 1: FLOOD                          รอบที่ 2: PRUNE (หลัง 3 วินาที)

   Source                                    Source
     │                                          │
     ▼ flood ทุกทาง                              ▼ ส่งเฉพาะทางที่มี Receiver
  ┌──┴──┐                                     ┌──┴──┐
  │  R1  │──┬────┬────┬──                     │  R1  │──────┬────────
  └─────┘   │    │    │                       └─────┘       │
           R2   R3   R4                                    R2 (มี Receiver)
        (ไม่มี Receiver)  (มี Receiver)              R3, R4 ส่ง Prune Message
        ส่ง Prune กลับ    ไม่ส่ง Prune              กลับไปบอก R1 ว่า "ไม่ต้องส่งมาแล้ว"
        (ตัดกิ่งทิ้ง)      (ยังรับ Flood ต่อ)
```

1. **Flood**: เมื่อ Source เริ่มส่ง Packet, Router ต้นทางจะ Flood ไปยัง **ทุก PIM Neighbor
   ทุกทิศทาง** (ยกเว้นทางที่ Packet เข้ามา — RPF Interface) โดยไม่ถามก่อนว่ามี Receiver หรือไม่
2. **Prune**: Router ปลายกิ่งที่ **ไม่มี** Receiver (ไม่มี IGMP Report สำหรับ Group นั้น และไม่มี
   PIM Neighbor ที่ต้องการต่อ) จะส่ง **Prune Message** กลับไปยัง Upstream Router เพื่อบอกว่า
   "อย่าส่งมาทางนี้อีก"
3. **Prune Timeout**: Prune State มีอายุ (ปกติ 3 นาที) — เมื่อหมดอายุ Router จะ Flood ใหม่อีกรอบ
   เพื่อเช็คว่ามี Receiver ใหม่เกิดขึ้นหรือยัง (Periodic Re-flood)
4. **Graft**: ถ้ามี Receiver ใหม่เกิดขึ้นก่อน Timeout, Router จะส่ง Graft Message เพื่อขอ "ต่อกิ่ง"
   กลับมาทันทีโดยไม่ต้องรอ Re-flood

### ทำไม PIM-DM ไม่ scale และแทบไม่ถูกใช้ในเครือข่ายสมัยใหม่

| ปัญหา | รายละเอียด |
|---|---|
| **สิ้นเปลือง Bandwidth ตอน Flood** | ทุก Router ทุก Link ต้องรับ Traffic อย่างน้อย 1 รอบเสมอ แม้ไม่มี Receiver เลยในเครือข่ายทั้งหมด |
| **Re-flood ซ้ำทุก 3 นาที** | แม้ Prune ไปแล้ว ก็ต้อง Flood ใหม่วนซ้ำตลอดอายุของ Stream — ยิ่ง Network ใหญ่ยิ่งเปลือง |
| **ไม่เหมาะกับ WAN/Low-Bandwidth Link** | การ Flood ก่อน Prune ใช้ไม่ได้เลยกับ Link ที่ Bandwidth จำกัด (ตรงข้ามกับเป้าหมายของ Multicast ที่ต้องการประหยัด Bandwidth) |
| **ไม่มี Rendezvous Point/Shared Tree** | ไม่มีจุดกลางให้ Join แบบเจาะจง ต้อง Flood ทั้ง Domain เสมอสำหรับทุก Source ใหม่ |

**หลักสูตรนี้ครอบคลุม PIM-DM เพื่อความสมบูรณ์ของทฤษฎีเท่านั้น** — ในทางปฏิบัติ Enterprise/Campus
สมัยใหม่ **ใช้ PIM-SM (Step 406) เกือบ 100%** เพราะ Scale ได้ดีกว่ามาก และ Lab จริงของ Part นี้
(Step 407, 410) จะ Config ด้วย PIM-SM เท่านั้น

---

## Step 406 — PIM Sparse Mode (PIM-SM): Explicit-Join Model

**PIM-SM** ใช้สมมติฐานที่ตรงกับความเป็นจริงในองค์กรส่วนใหญ่: **"สมมติว่าไม่มี Receiver อยู่เลย
จนกว่าจะมีใครขอ (Explicit Join) เข้ามาจริงๆ"** — ตรงข้ามกับ PIM-DM อย่างสิ้นเชิง ทำให้ Traffic
ไม่ไหลไปที่ใดเลยจนกว่าจะมีการ Join จริง ประหยัด Bandwidth กว่ามาก และเป็น Mode ที่ Scale ได้
ในระดับ Enterprise/Service Provider

### Rendezvous Point (RP) — จุดนัดพบกลางของ Domain

ปัญหาของ Explicit-Join คือ: **Router ที่มี Receiver จะรู้ได้อย่างไรว่า Source อยู่ที่ไหน** (ในเมื่อ
ยังไม่มี Traffic ไหลมาให้เห็นเลย)? PIM-SM แก้ปัญหานี้ด้วยการกำหนด Router ตัวหนึ่งในเครือข่ายให้เป็น
**Rendezvous Point (RP)** — จุดกลางที่ทุก Router รู้จัก (ผ่าน Static Config หรือ Auto-RP/BSR) ทำ
หน้าที่เป็น "จุดนัดพบ" ระหว่าง Source กับ Receiver:

- **Source** ที่เริ่มส่ง Traffic จะให้ First-Hop Router (Router ที่ต่อกับ Source โดยตรง) **Register**
  ตัวเองไปที่ RP (บอกว่า "มี Source ตัวนี้กำลังส่ง Group นี้อยู่นะ")
- **Receiver** ที่ Join Group จะให้ Last-Hop Router (Router ที่ต่อกับ Receiver โดยตรง) ส่ง
  **PIM Join** มุ่งไปที่ RP (บอกว่า "ฉันต้องการ Group นี้ ส่งมาทาง RP ให้ที")
- RP จับคู่ทั้งสองฝั่งให้เจอกัน — Traffic จาก Source จะไหลผ่าน RP ไปยัง Receiver ในช่วงแรก

### Shared Tree (RPT) vs Shortest-Path Tree (SPT)

```
STAGE 1 — Shared Tree (RPT) — รากอยู่ที่ RP เสมอ (ทุก Group ใช้ RP เดียวกัน)

     Source ──Register──► RP ◄──Join── Receiver
                            │
                    Traffic ไหลผ่าน RP ก่อนเสมอ (อาจไม่ใช่เส้นทางที่สั้นที่สุด)
     Source ──data──────► RP ──data──► Receiver     (เส้นทาง S→RP→Receiver)


STAGE 2 — Shortest-Path Tree (SPT) — Switchover หลังผ่าน Threshold

     Last-Hop Router (ของ Receiver) เห็น Traffic Rate เกิน Threshold
                            │
                    ส่ง (S,G) Join ตรงไปยัง Source โดยตรง (ไม่ผ่าน RP อีกต่อไป)
                            │
     Source ──data (เส้นทางตรง, สั้นที่สุด)────────────► Receiver
                    RP ไม่ต้องเกี่ยวข้องอีกต่อไปสำหรับ (S,G) นี้
```

| คุณสมบัติ | Shared Tree (RPT) | Shortest-Path Tree (SPT) |
|---|---|---|
| Root ของ Tree | Rendezvous Point (RP) เสมอ | Source โดยตรง |
| แสดงเป็น State | `(*, G)` — ทุก Source ของ Group นี้ | `(S, G)` — เจาะจง Source ตัวเดียว |
| ใช้ตอนไหน | เริ่ม Join ใหม่ (ยังไม่รู้ว่า Source อยู่ที่ไหน) | หลังทราบ Source แล้ว และ Traffic Rate สูงเพียงพอ |
| ข้อดี | Setup ง่าย, RP เดียวรู้ทุก Source | เส้นทางสั้นที่สุด, ไม่ต้องพึ่ง RP (ลด Bottleneck) |
| ข้อเสีย | อาจไม่ใช่เส้นทางที่สั้นที่สุด (Traffic เดินทางผ่าน RP ก่อน) | ต้องมี (S,G) State แยกทุก Source (ใช้ Memory มากขึ้นถ้า Source เยอะ) |

### SPT-Switchover Threshold

Cisco IOS ตั้งค่า Default ให้ Last-Hop Router **Switchover จาก Shared Tree ไป SPT ทันทีที่ได้รับ
Packet แรก** (`ip pim spt-threshold infinity` คือค่าที่ปิดการ Switchover ให้อยู่บน Shared Tree
ตลอดไป ซึ่งไม่ใช่ Default):

```
DIST-SW1(config)# ip pim spt-threshold ?
  <0-4294967>  Traffic rate in kbps
  infinity     Never switch to source tree

! Default บน IOS ปัจจุบันคือ "0" ความหมายคือ switchover ทันทีที่มี packet แรกมาถึง
! (แตกต่างจากค่า Default ของอุปกรณ์ Cisco รุ่นเก่ามากที่อาจตั้งเป็นค่าอื่น ต้องตรวจสอบด้วย
!  show running-config เสมอ)
```

การ Switchover ทันที (Threshold = 0) เหมาะกับ Lab และ Traffic สำคัญที่ต้องการ Latency ต่ำที่สุด
เช่น Video Streaming ของ STREAM-SRV ใน Use Case ของเรา — ในขณะที่การตั้ง `infinity` เหมาะกับ
Design ที่ต้องการควบคุม (S,G) State ให้น้อยที่สุดบน Router ที่มี Memory จำกัด (พบมากใน Core
Router ของ Service Provider ที่มี Source จำนวนมาก)

---

## Step 407 — Basic PIM-SM Configuration

### คำสั่งหลักที่ต้องรู้ (3 คำสั่งเท่านั้น สำหรับ Basic PIM-SM)

| คำสั่ง | ระดับ | ความหมาย |
|---|---|---|
| `ip multicast-routing` | Global Config | **เปิด Multicast Routing Process ทั้งเครื่อง** — ต้องเปิดก่อนคำสั่งอื่นทั้งหมด (คล้าย `ip routing` สำหรับ Unicast) |
| `ip pim sparse-mode` | Interface Config | เปิด PIM-SM บน Interface นั้น — ทำให้ Interface นี้ส่ง PIM Hello (`224.0.0.13`), รับ Join/Prune, และเป็นตัวเลือก RPF Interface ได้ |
| `ip pim rp-address <RP-IP> [group-list]` | Global Config | ประกาศ Static RP ให้ทุก Router ใน Domain รู้จัก IP เดียวกัน — ทุก Router ที่รัน PIM ต้องมี Config บรรทัดนี้เหมือนกันทั้งหมด (Static RP วิธีเดียวที่ไม่มี Auto-RP/BSR) |

### เลือก CORE-SW1 เป็น Static RP

Lab ของเราจะกำหนดให้ **CORE-SW1** (Loopback0 = `1.1.1.1/32` ตาม
[`00-ip-address-plan.md`](00-ip-address-plan.md)) เป็น **Static RP** เพราะ CORE Layer อยู่ตรงกลาง
Topology พอดี (ทุก Distribution Block ไปถึงได้ในจำนวน Hop เท่ากันโดยประมาณ) และ Loopback0 เป็น
Address ที่เสถียรที่สุด (ไม่มีวันล่มพร้อม Physical Interface เส้นใดเส้นหนึ่ง)

```
                              ┌─────────────────┐
                              │  CORE-SW1        │
                              │  Lo0 = 1.1.1.1   │  ◄── Static RP ของทั้ง Domain
                              │  (RP)            │
                              └────────┬────────┘
                                  Po1  │
                              ┌────────┴────────┐
                              │  CORE-SW2        │
                              └───┬─────────┬───┘
                        ┌─────────┘         └─────────┐
                  ┌─────┴─────┐                 ┌─────┴─────┐
                  │ DIST-SW1/2 │                 │ DIST-SW3/4 │
                  │ VLAN 10,20 │                 │ VLAN 30,40 │
                  └───────────┘                 └───────────┘
              ACCESS-SW1/2 (Sales,Voice)     ACCESS-SW3/4 (Servers,WiFi)
```

### Config เต็มรูปแบบ: เปิด PIM-SM บนทุก Interface ที่เกี่ยวข้องใน Campus

```
! ===== CORE-SW1 (Static RP) =====
CORE-SW1(config)# ip multicast-routing
CORE-SW1(config)# ip pim rp-address 1.1.1.1
CORE-SW1(config)# interface Loopback0
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# ip pim sparse-mode
CORE-SW1(config-if)# exit

! ===== CORE-SW2 =====
CORE-SW2(config)# ip multicast-routing
CORE-SW2(config)# ip pim rp-address 1.1.1.1
CORE-SW2(config)# interface TenGigabitEthernet1/0/1
CORE-SW2(config-if)# ip pim sparse-mode
CORE-SW2(config-if)# exit
CORE-SW2(config)# interface TenGigabitEthernet1/0/2
CORE-SW2(config-if)# ip pim sparse-mode
CORE-SW2(config-if)# exit
CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# ip pim sparse-mode
CORE-SW2(config-if)# exit

! ===== DIST-SW1 (VLAN 10 SALES, 20 VOICE, 99 MGMT) =====
DIST-SW1(config)# ip multicast-routing distributed
DIST-SW1(config)# ip pim rp-address 1.1.1.1
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# ip pim sparse-mode
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ip pim sparse-mode
DIST-SW1(config-if)# exit

! ===== DIST-SW3 (VLAN 30 SERVERS, 40 WIFI, 99 MGMT) =====
DIST-SW3(config)# ip multicast-routing distributed
DIST-SW3(config)# ip pim rp-address 1.1.1.1
DIST-SW3(config)# interface TenGigabitEthernet1/1/1
DIST-SW3(config-if)# ip pim sparse-mode
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ip pim sparse-mode
DIST-SW3(config-if)# exit
DIST-SW3(config)# interface Vlan40
DIST-SW3(config-if)# ip pim sparse-mode
DIST-SW3(config-if)# exit

! ===== DIST-SW4 (VLAN 30 SERVERS, 40 WIFI, 99 MGMT) =====
DIST-SW4(config)# ip multicast-routing distributed
DIST-SW4(config)# ip pim rp-address 1.1.1.1
DIST-SW4(config)# interface TenGigabitEthernet1/1/1
DIST-SW4(config-if)# ip pim sparse-mode
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface Vlan40
DIST-SW4(config-if)# ip pim sparse-mode
DIST-SW4(config-if)# exit
```

> **หมายเหตุสำคัญเรื่อง Static RP**: `ip pim rp-address 1.1.1.1` ต้อง Config **เหมือนกันทุกตัวอักษร
> บน Router/Switch ทุกตัวที่รัน PIM ในทั้ง Domain** (CORE-SW1 เองก็ต้องมีบรรทัดนี้ด้วย แม้ตัวมันเอง
> จะเป็น RP อยู่แล้ว) — ถ้า Router ตัวใดไม่มี หรือชี้ไป RP คนละตัว จะทำให้ Domain แตกเป็นสองฝั่งที่
> คุยกันไม่รู้เรื่อง (Inconsistent RP Mapping) เป็นสาเหตุ Troubleshoot ที่พบบ่อยที่สุดของ PIM-SM
>
> **`ip multicast-routing distributed`** บน Platform ระดับ Catalyst 9300/9500 หมายถึงเปิดใช้
> Distributed CEF สำหรับ Multicast (กระจายงาน Forwarding ไปยัง Hardware ASIC ของแต่ละ Module)
> — บน Platform ที่ไม่รองรับ Distributed Switching ให้ใช้ `ip multicast-routing` เฉยๆ ก็เพียงพอ

---

## Step 408 — Multicast Forwarding และ RPF Check: (S,G) และ (*,G)

### Unicast Forwarding vs Multicast Forwarding — ต่างกันที่คำถามพื้นฐาน

```
Unicast Forwarding                          Multicast Forwarding
คำถาม: "Packet นี้จะไป (Destination)         คำถาม: "Packet นี้มาจากทางที่ถูกต้องหรือไม่
        ทางไหน?"                                    (Source) แล้วควรกระจาย (Fan-out)
ดูจาก: Destination IP → Routing Table               ออกไปทางไหนบ้าง?"
       → เลือก 1 Output Interface              ดูจาก: Source IP → RPF Check → ถ้าถูกทาง
                                                      → Forward ออกหลาย Interface (Fan-out)
                                                      ตาม Outgoing Interface List (OIL)
```

นี่คือความแตกต่างพื้นฐานที่สุดที่ต้องเข้าใจ: **Unicast สนใจปลายทาง, Multicast สนใจต้นทาง**
เพราะ Multicast Packet ตัวเดียวอาจต้อง Forward ออกหลาย Interface พร้อมกัน (ไม่ใช่แค่ 1 ทาง
แบบ Unicast) — ถ้าไม่มีการตรวจสอบทางเข้า (RPF) ก่อน จะเกิด Loop ได้ง่ายมาก เพราะ Multicast
Packet สามารถวนกลับมาที่ Router เดิมได้จากหลายทิศทาง

### RPF (Reverse Path Forwarding) Check

**RPF Check** คือกลไกป้องกัน Loop หลักของ Multicast: เมื่อ Multicast Packet มาถึง Interface ใด
Router จะถามคำถามเดียว **"ถ้าฉันจะส่ง Unicast Packet กลับไปหา Source นี้ ฉันจะใช้ Interface
ไหน (ตาม Unicast Routing Table)?"** ถ้าคำตอบ **ตรงกับ Interface ที่ Packet เพิ่ง Arrive เข้ามาจริง**
→ ผ่าน RPF Check → Forward ต่อ ถ้า **ไม่ตรง** → Drop ทิ้งทันที (ถือว่าผิดทาง ป้องกัน Loop)

```
                    ┌─────────┐
   Source ──────────┤ Router A │
                    └────┬────┘
                         │ (ทางที่ควรมาถึงจริง ตาม Unicast Route ไปยัง Source)
                    ┌────┴────┐
                    │ Router B │
                    └────┬────┘
                    Gi0/1│    │Gi0/2
                         │    │
              Packet มาถึงทาง Gi0/1 (ตรงกับ RPF)  →  ✅ ผ่าน RPF Check → Forward
              ถ้า Packet เดียวกันมาถึงทาง Gi0/2 (ผิดทาง) →  ❌ Fail RPF → Drop (ป้องกัน Loop)
```

### State Entry สองแบบใน Multicast Routing Table

| Notation | อ่านว่า | ความหมาย |
|---|---|---|
| `(*, G)` | "Star comma G" | Shared Tree Entry — รับ Traffic ของ Group G **จาก Source ใดก็ได้** (RPF Interface ชี้ไปทาง RP) |
| `(S, G)` | "S comma G" | Source Tree Entry (SPT) — รับ Traffic ของ Group G **จาก Source S ตัวนี้เท่านั้น** (RPF Interface ชี้ไปทาง Source โดยตรง) |

ทั้งสอง Entry มีองค์ประกอบเหมือนกัน 2 ส่วนที่ต้องอ่านให้เป็น:

```
(S, G) หรือ (*, G)
   │
   ├── Incoming Interface (RPF Interface) — ทางเดียวเท่านั้นที่ยอมรับ Traffic เข้ามา
   │   (มาจากทางอื่นจะถูก Drop ทันทีตาม RPF Check)
   │
   └── Outgoing Interface List (OIL) — รายการ Interface ที่จะ Forward Traffic ออกไป
       (อาจมีได้หลาย Interface พร้อมกัน — นี่คือจุดที่ "แยกสำเนา" เกิดขึ้นจริง)
```

ตัวอย่างสำหรับ Use Case ของเรา: หลัง STREAM-SRV เริ่มส่ง Stream ไปที่ `239.1.1.1` และมี Receiver
Join แล้วทั้งใน VLAN 10 และ VLAN 40, CORE-SW1 (ในเส้นทางกลาง) จะมี Entry ประมาณนี้:

```
(10.10.30.30, 239.1.1.1)
   Incoming: interface ทางไป DIST-SW3 (ทางไป Source)
   Outgoing: interface ทางไป DIST-SW1 (Receiver ใน VLAN10)
             interface ทางไป DIST-SW4 (Receiver ใน VLAN40)
```

จะเห็นว่า Router ตัวนี้ "แยกสำเนา" Traffic ออกเป็น 2 ทางพร้อมกันจาก 1 ทางเข้า — **นี่คือหัวใจของ
ประสิทธิภาพ Multicast** ที่ตอบโจทย์ปัญหาจาก Step 401 ได้ตรงจุด

---

## Step 409 — Verification & Troubleshooting

### `show ip igmp groups` — ดูว่า Group ไหนถูก Join อยู่ที่ Interface ไหนบ้าง

```
DIST-SW1# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface        Uptime    Expires   Last Reporter  Group Accounted
239.1.1.1         Vlan10           00:04:12  00:02:48  10.10.10.11
224.0.1.40        Vlan99           02:15:03  00:02:41  10.10.99.5
```

### `show ip pim neighbor` — ตรวจสอบว่า PIM Neighbor ขึ้นครบทุกทิศทาง

```
DIST-SW1# show ip pim neighbor
PIM Neighbor Table
Mode: B - Bidir Capable, DR - Designated Router, N - Default DR Priority,
      P - Proxy Capable, S - State Refresh Capable, G - GenID Capable
Neighbor Address    Interface           Uptime/Expires  Ver   DR Prio/Mode
10.255.10.1          TenGigabitEthernet1/1/1  00:32:10/00:01:41  v2   1 / DR S P G
```

### `show ip mroute` — คำสั่งสำคัญที่สุดของ Multicast Troubleshooting

```
CORE-SW1# show ip mroute 239.1.1.1
IP Multicast Routing Table
Flags: D - Dense, S - Sparse, s - SSM Group, C - Connected, L - Local,
       P - Pruned, R - RP-bit set, F - Register flag, T - SPT-bit set,
       J - Join SPT, M - MSDP created entry, X - Proxy Join Timer Running,
       A - Advertised via MSDP, U - URD, I - Received Source Specific Host
       Report, Z - Multicast Tunnel, z - MDT-data group sender,
       Y - Joined MDT-data group, y - Sending to MDT-data group

(*, 239.1.1.1), 00:05:20/00:02:47, RP 1.1.1.1, flags: SJC
  Incoming interface: Loopback0, RPF nbr 0.0.0.0
  Outgoing interface list:
    TenGigabitEthernet1/0/1, Forward/Sparse, 00:05:20/00:02:47
    TenGigabitEthernet1/0/2, Forward/Sparse, 00:05:20/00:02:47

(10.10.30.30, 239.1.1.1), 00:03:02/00:03:27, flags: FT
  Incoming interface: Port-channel1, RPF nbr 10.255.0.2
  Outgoing interface list:
    TenGigabitEthernet1/0/1, Forward/Sparse, 00:03:02/00:03:27
    TenGigabitEthernet1/0/2, Forward/Sparse, 00:03:02/00:03:27
```

**การอ่าน Output ทีละส่วน**:

| ส่วน | ความหมาย |
|---|---|
| `(*, 239.1.1.1)` | Entry แบบ Shared Tree — ยังรับได้จากทุก Source ของ Group นี้ |
| `RP 1.1.1.1` | RP ของ Entry นี้คือ CORE-SW1 (ตัวมันเอง ในกรณีนี้เพราะ CORE-SW1 คือ RP) |
| `flags: SJC` | S=Sparse Mode, J=Join SPT (พร้อม switchover), C=Connected (มี Receiver ต่อ Local อยู่) |
| `Incoming interface: Loopback0` | เพราะตัวเองเป็น RP — RPF ชี้เข้าตัวเอง |
| `Outgoing interface list` | ทาง Forward ออกไปยัง DIST-SW1 (Te1/0/1) และ DIST-SW3/4 (Te1/0/2) |
| `(10.10.30.30, 239.1.1.1)` | Entry แบบ SPT เจาะจง Source = STREAM-SRV โดยตรง |
| `flags: FT` | F=Register flag (มาจาก Register จาก First-Hop), T=SPT-bit (ใช้ Shortest-Path Tree แล้ว) |
| `Incoming interface: Port-channel1` | ทางเข้าจริงคือทาง CORE-SW2 (เพราะ Source ต่อ DIST-SW3 ซึ่งอยู่ฝั่ง CORE-SW2) |

### `show ip pim rp mapping` — ตรวจสอบว่า Router รับรู้ RP ตัวไหนสำหรับ Group ไหน

```
DIST-SW3# show ip pim rp mapping
PIM Group-to-RP Mappings

Group(s): 224.0.0.0/4, Static
    RP: 1.1.1.1 (?)
```

`224.0.0.0/4` หมายถึง RP นี้ครอบคลุม **ทุก Group** ใน Class D ทั้งหมด (ค่า Default เมื่อ Config
`ip pim rp-address` โดยไม่ระบุ `group-list` ต่อท้าย) — ถ้าต้องการให้ RP ตัวนี้ดูแลเฉพาะบาง Group
ต้องเพิ่ม Access-list ต่อท้ายคำสั่ง เช่น `ip pim rp-address 1.1.1.1 10`

### คำเตือนสำหรับ `debug ip mpacket`

```
CORE-SW1# debug ip mpacket
```

`debug ip mpacket` แสดง Log ของ **ทุก Multicast Packet ที่ถูก Forward/Drop** ในระดับ Per-Packet
— บน Link ที่มี Video Stream หลาย Mbps คำสั่งนี้จะสร้าง Log จำนวนมหาศาลจนอาจทำให้ CPU ของ
Router/Switch พุ่งสูงถึงขั้น Lock ตัวเองจน Console ไม่ตอบสนอง (Classic "debug ที่ทำ Production
ล่มเอง") **ควรใช้เฉพาะใน Lab Environment ที่มี Traffic Rate ต่ำมากๆ เท่านั้น** และควร Config
`no logging console` + จำกัดด้วย Access-list ก่อนเปิด `debug` เสมอถ้าจำเป็นต้องใช้จริงบน Production
— ในทางปฏิบัติ `show ip mroute` ร่วมกับ `show ip mroute count` เพียงพอสำหรับ Troubleshoot
เกือบทุกกรณีโดยไม่ต้องเสี่ยงเปิด `debug` เลย

---

## Step 410 — Lab เต็มรูปแบบ: PIM-SM Deployment ส่ง All-Hands Video Stream ข้าม Campus

### Topology ของ Lab นี้

```
                                   ┌──────────────────┐
                                   │     CORE-SW1       │
                                   │   Lo0: 1.1.1.1      │ ◄── Static RP
                                   └─────────┬─────────┘
                                        Po1  │
                                   ┌─────────┴─────────┐
                                   │     CORE-SW2       │
                                   │   Lo0: 1.1.1.2      │
                                   └────┬───────────┬───┘
                        Te1/0/1 (10.255.30.0/30) │  │ Te1/0/2 (10.255.40.0/30)
                          ┌──────────────┘         └──────────────┐
                    ┌─────┴─────┐                            ┌─────┴─────┐
                    │  DIST-SW3   │                            │  DIST-SW4   │
                    │ Vlan30 SVI  │                            │ Vlan40 SVI  │
                    │ Vlan40 SVI  │                            │ Vlan40 SVI  │
                    └─────┬───────┘                            └─────┬───────┘
                          │ Trunk                                     │ Trunk
                    ┌─────┴───────┐                              ┌─────┴───────┐
                    │  ACCESS-SW3   │                              │  ACCESS-SW4   │
                    │ (Vlan 30, 40) │                              │ (Vlan 30, 40) │
                    └───────┬───────┘                              └───────┬───────┘
                            │ Gi0/5 (access, Vlan30)                       │ Gi0/10 (access, Vlan40)
                      ┌─────┴─────┐                                  ┌─────┴─────┐
                      │ STREAM-SRV  │                                  │ PC-WIFI1    │
                      │10.10.30.30  │  ──── Source ────► 239.1.1.1     │10.10.40.51  │ (Receiver)
                      └────────────┘                                  └────────────┘

   (ฝั่ง CORE-SW1 ↔ DIST-SW1/DIST-SW2 ↔ ACCESS-SW1/ACCESS-SW2 ตาม Topology หลักจาก Part 1/33)
                                   ┌─────────────┐
                                   │  DIST-SW1     │  Vlan10 SVI
                                   └──────┬────────┘
                                          │ Trunk
                                   ┌──────┴────────┐
                                   │  ACCESS-SW1     │ (Vlan 10)
                                   └──────┬────────┘
                                          │ Gi0/8 (access, Vlan10)
                                   ┌──────┴────────┐
                                   │  PC-SALES1      │
                                   │  10.10.10.11    │  (Receiver)
                                   └────────────────┘
```

**สรุป Use Case**: STREAM-SRV (VLAN 30, ต่อ ACCESS-SW3) ส่ง Video Stream ไปที่ Group
`239.1.1.1` — มี Receiver 2 จุดที่ Join Group นี้: **PC-SALES1** (VLAN 10, ต่อ ACCESS-SW1 คนละ
Distribution Block กับ Source) และ **PC-WIFI1** (VLAN 40, ต่อ ACCESS-SW4 คนละ Access Switch
แต่ Distribution Block เดียวกับ Source) — ครอบคลุมทั้งกรณี "ข้าม Distribution Block" และ
"ข้าม Access Switch ใน Block เดียวกัน"

### Running-config ที่เกี่ยวข้อง (ต่อจาก Step 407 — เพิ่มส่วนที่ยังไม่ได้ลง)

```
! ===== DIST-SW4 (เพิ่มเติมจาก Step 407) =====
DIST-SW4(config)# ip multicast-routing distributed
DIST-SW4(config)# ip pim rp-address 1.1.1.1
DIST-SW4(config)# interface TenGigabitEthernet1/1/1
DIST-SW4(config-if)# ip pim sparse-mode
DIST-SW4(config-if)# exit
DIST-SW4(config)# interface Vlan40
DIST-SW4(config-if)# ip pim sparse-mode
DIST-SW4(config-if)# exit

! ===== ACCESS-SW1 (IGMP Snooping) =====
ACCESS-SW1(config)# ip igmp snooping
ACCESS-SW1(config)# ip igmp snooping vlan 10
ACCESS-SW1(config)# interface GigabitEthernet0/8
ACCESS-SW1(config-if)# switchport mode access
ACCESS-SW1(config-if)# switchport access vlan 10
ACCESS-SW1(config-if)# exit

! ===== ACCESS-SW2 (IGMP Snooping — ไม่มี Receiver จริง แต่เปิด Baseline ตาม Best Practice) =====
ACCESS-SW2(config)# ip igmp snooping
ACCESS-SW2(config)# ip igmp snooping vlan 10
ACCESS-SW2(config)# ip igmp snooping vlan 20

! ===== ACCESS-SW3 (IGMP Snooping — พอร์ตของ STREAM-SRV) =====
ACCESS-SW3(config)# ip igmp snooping
ACCESS-SW3(config)# ip igmp snooping vlan 30
ACCESS-SW3(config)# ip igmp snooping vlan 40
ACCESS-SW3(config)# interface GigabitEthernet0/5
ACCESS-SW3(config-if)# switchport mode access
ACCESS-SW3(config-if)# switchport access vlan 30
ACCESS-SW3(config-if)# exit

! ===== ACCESS-SW4 (IGMP Snooping — พอร์ตของ PC-WIFI1) =====
ACCESS-SW4(config)# ip igmp snooping
ACCESS-SW4(config)# ip igmp snooping vlan 40
ACCESS-SW4(config)# interface GigabitEthernet0/10
ACCESS-SW4(config-if)# switchport mode access
ACCESS-SW4(config-if)# switchport access vlan 40
ACCESS-SW4(config-if)# exit
```

### จำลอง Client Join Group (บน PC-SALES1 และ PC-WIFI1)

ในสภาพแวดล้อมจริง Client จะ Join Group ผ่าน Application (เช่น VLC, Media Player ที่เปิด Stream
`239.1.1.1`) ซึ่งจะสั่งให้ OS ส่ง IGMP Membership Report โดยอัตโนมัติ — ในบาง Lab Environment
(เช่น Packet Tracer / IOSv ที่ไม่มี Client จริงรัน Media Player) วิศวกรมักจำลองการ Join ด้วยคำสั่ง
ทดสอบบน Router/Switge เพื่อยืนยันว่า Path การ Forward ถูกต้องก่อน Deploy จริง:

```
! จำลองการ Join แบบ Static เพื่อทดสอบ Forwarding Path (ไม่ต้องพึ่ง IGMP จาก Client จริง)
ACCESS-SW1(config)# interface GigabitEthernet0/8
ACCESS-SW1(config-if)# ip igmp static-group 239.1.1.1
```

### Verification: `show ip mroute` แสดง (S,G) State ครบทั้ง Path

```
CORE-SW1# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:12:40/00:03:10, RP 1.1.1.1, flags: SJC
  Incoming interface: Loopback0, RPF nbr 0.0.0.0
  Outgoing interface list:
    TenGigabitEthernet1/0/1, Forward/Sparse, 00:12:40/00:03:10   (ไปทาง DIST-SW1, VLAN10)

(10.10.30.30, 239.1.1.1), 00:09:15/00:03:22, flags: FT
  Incoming interface: Port-channel1, RPF nbr 10.255.0.2          (ทางกลับไป Source ผ่าน CORE-SW2)
  Outgoing interface list:
    TenGigabitEthernet1/0/1, Forward/Sparse, 00:09:15/00:03:22   (ไปทาง DIST-SW1, VLAN10)

DIST-SW4# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:11:02/00:03:29, RP 1.1.1.1, flags: SJC
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr 10.255.40.1
  Outgoing interface list:
    Vlan40, Forward/Sparse, 00:11:02/00:03:29                     (ไปทาง ACCESS-SW4, PC-WIFI1)

(10.10.30.30, 239.1.1.1), 00:07:48/00:02:58, flags: FT
  Incoming interface: TenGigabitEthernet1/1/1, RPF nbr 10.255.40.1
  Outgoing interface list:
    Vlan40, Forward/Sparse, 00:07:48/00:02:58
```

สังเกตว่า **DIST-SW4 ไม่ต้องผ่าน CORE เลยด้วยซ้ำสำหรับ Traffic Data จริง** เพราะ DIST-SW3
(ที่ต่อกับ Source โดยตรง) และ DIST-SW4 อยู่คนละ Distribution Block แต่ทั้งคู่ต่อกับ CORE-SW2 —
เมื่อ SPT Switchover เกิดขึ้น (Step 406) Traffic จะเดินทางเส้นที่สั้นที่สุดตาม Unicast Topology จริง
ไม่จำเป็นต้องผ่าน RP (CORE-SW1) อีกต่อไปสำหรับ (S,G) Entry นี้

### Verification: `show ip igmp groups` บน Switch ฝั่ง Receiver

```
ACCESS-SW1# show ip igmp snooping groups
Vlan  Group            Type    Version  Port List
----  ----------------  ------  -------  -------------
10    239.1.1.1          igmp    v2       Gi0/8

DIST-SW1# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface        Uptime    Expires   Last Reporter
239.1.1.1         Vlan10           00:12:40  00:02:35  10.10.10.11

ACCESS-SW4# show ip igmp snooping groups
Vlan  Group            Type    Version  Port List
----  ----------------  ------  -------  -------------
40    239.1.1.1          igmp    v2       Gi0/10

DIST-SW4# show ip igmp groups
IGMP Connected Group Membership
Group Address    Interface        Uptime    Expires   Last Reporter
239.1.1.1         Vlan40           00:11:02  00:02:41  10.10.40.51
```

ผลลัพธ์ยืนยันครบทั้ง Chain: **ACCESS-SW เห็น Port ที่ Join ผ่าน IGMP Snooping → DIST-SW เห็น
Group Membership ผ่าน IGMP เต็มรูปแบบ → CORE/DIST เห็น (S,G) และ (*,G) State ผ่าน PIM-SM
ครบทุกจุด** — All-Hands Video Stream จาก STREAM-SRV ไปถึงทั้ง VLAN 10 (SALES) และ VLAN 40
(WIFI) พร้อมกันได้จริง โดยที่ VLAN 20 (VOICE) และ VLAN 99 (MGMT) ที่ไม่มีใคร Join เลย **ไม่ได้
รับ Traffic นี้แม้แต่ Byte เดียว** ตรงตามเป้าหมายที่ตั้งไว้ตั้งแต่ต้น Part

---

## แบบฝึกหัดทวนความเข้าใจ Part 41

1. IP Address `224.0.0.5` และ `224.0.0.102` ใช้โดย Protocol อะไร และทำไม Address ทั้งสองนี้จึง
   ไม่มีวันถูก Forward ข้าม Router ไปยัง Subnet อื่น?
2. แปลง Multicast IP `239.10.20.30` เป็น Ethernet Multicast MAC Address ตามหลักที่เรียนใน
   Step 402 (แสดงวิธีคิดทีละขั้น)
3. ทำไม IGMP เพียงอย่างเดียวไม่เพียงพอที่จะทำให้ STREAM-SRV ส่งวิดีโอไปถึง PC-WIFI1 ที่อยู่คนละ
   VLAN ได้ และ PIM เข้ามาแก้ปัญหานี้อย่างไร?
4. อธิบายความแตกต่างระหว่าง `(*, 239.1.1.1)` กับ `(10.10.30.30, 239.1.1.1)` ที่เห็นใน
   `show ip mroute` — Entry แต่ละแบบถูกใช้ตอนไหน และ RPF Interface ของแต่ละ Entry ต่างกัน
   อย่างไรในตัวอย่าง Lab ของ Part นี้?
5. ทำไม PIM-DM (Flood-and-Prune) จึงไม่เหมาะกับ Campus Network ขนาดใหญ่ที่มี Source/Group
   จำนวนมาก และ PIM-SM แก้ปัญหานี้อย่างไรด้วยแนวคิด Explicit-Join?

**เฉลย:**

1. `224.0.0.5` = OSPF All-Routers ([Part 11](part-011-ospfv2-fundamentals.md)),
   `224.0.0.102` = HSRPv2 Hello ([Part 18](part-018-fhrp.md)) — ทั้งสองอยู่ในช่วง
   `224.0.0.0/24` (Local Network Control Block) ซึ่งถูกกำหนดให้มี **TTL=1 เสมอ** ตั้งแต่ต้นทาง
   ทำให้ Router ตัวแรกที่รับ Packet จะลด TTL เหลือ 0 และ Drop ทิ้งทันทีตาม Design โดยเจตนา
   เพราะ Protocol เหล่านี้ต้องทำงานเฉพาะ Local Segment เดียวเท่านั้น
2. `239.10.20.30` → Binary: `11101111.00001010.00010100.00011110` → เอา 28 bit ท้าย
   ตัด 4 bit Class D prefix ทิ้ง → `1111.00001010.00010100.00011110` → เอาแค่ 23 bit ล่างสุด
   (ตัด 5 bit บนสุดทิ้ง ได้ Octet 2 เหลือ `0001010` 7 bit) → ยัดเข้า `01:00:5E:` ตามลำดับ
   → ผลลัพธ์คือ **`01:00:5E:0A:14:1E`** (0A=10, 14=20, 1E=30 ในฐาน Hex ของ Octet 2-4 หลังตัด
   บิตบนสุดของ Octet 2 ออกซึ่งในกรณีนี้เป็น 0 อยู่แล้ว)
3. IGMP ทำงานเฉพาะระหว่าง Host กับ Router ตัวแรกภายใน Subnet เดียวกันเท่านั้น (Local Scope)
   ไม่มีกลไกส่งข้อมูลข้าม Subnet/VLAN เลย — ถ้าไม่มี PIM, Router แต่ละตัวจะไม่รู้เลยว่าต้อง
   Forward Multicast Packet จาก VLAN30 ไปยัง VLAN40 ทาง Interface ไหน PIM จึงเข้ามาทำหน้าที่
   สร้าง Distribution Tree ระหว่าง Router (ใช้ RP เป็นจุดนัดพบ แล้ว Switchover ไป SPT) ทำให้
   ข้อมูลข้าม VLAN/Subnet ไปถึง Receiver ที่ Join ไว้ได้จริง
4. `(*, 239.1.1.1)` คือ Shared Tree Entry รับ Traffic ของ Group นี้จาก **Source ใดก็ได้** RPF
   Interface ชี้ไปทาง RP (บน CORE-SW1 คือ Loopback0 เพราะตัวเองเป็น RP) ส่วน
   `(10.10.30.30, 239.1.1.1)` คือ SPT Entry เจาะจง Source = STREAM-SRV โดยตรง RPF Interface
   ชี้ไปทางเส้นทางที่สั้นที่สุดไปยัง Source จริง (บน CORE-SW1 คือ Port-channel1 ไปทาง CORE-SW2
   ที่ต่อกับ DIST-SW3 ซึ่งมี Source อยู่) — Entry แบบ (S,G) จะถูกใช้จริงหลัง SPT Switchover เกิดขึ้น
5. PIM-DM ต้อง Flood Traffic ไปทุกทิศทางก่อนเสมอโดยไม่ถามว่ามี Receiver หรือไม่ แล้วค่อย Prune
   กิ่งที่ไม่มี Receiver ทิ้งภายหลัง และต้อง Re-flood ซ้ำทุก 3 นาทีตลอดอายุ Stream — ยิ่ง Campus
   มี Source/Group จำนวนมาก ยิ่งสิ้นเปลือง Bandwidth บนทุก Link ซ้ำไปซ้ำมาไม่จบ ในขณะที่ PIM-SM
   ใช้แนวคิด Explicit-Join คือไม่ส่ง Traffic ไปที่ใดเลยจนกว่าจะมี PIM Join จริงเข้ามา (ผ่าน RP)
   ทำให้ Bandwidth ถูกใช้เฉพาะเส้นทางที่มี Receiver จริงเท่านั้น scale ได้ดีกว่ามากในเครือข่ายขนาดใหญ่

---

## สรุป Part 41

Part นี้เปิดโลกใหม่ของ **IP Multicast** ให้กับหลักสูตร โดยเริ่มจากปัญหาจริงที่ STREAM-SRV
(`10.10.30.30`) ต้องกระจาย All-Hands Video Stream ไปหลาย VLAN พร้อมกันอย่างมีประสิทธิภาพ —
เราเรียนรู้ Class D Address Space, การแปลง IP เป็น Multicast MAC (พร้อมปัญหา 32:1 Overlap),
**IGMP** สำหรับจัดการ Membership ระดับ Local Subnet และ **IGMP Snooping** ที่ทำให้ Switch
Layer 2 ไม่ Flood Multicast ไปทุก Port อีกต่อไป จากนั้นขยับขึ้นไปที่ **PIM** ซึ่งเป็นหัวใจของการ
Routing Multicast ข้าม Subnet ทั้ง **PIM-DM** (Flood-and-Prune ที่ไม่ scale) และ **PIM-SM**
(Explicit-Join ผ่าน Rendezvous Point, Shared Tree, และ SPT Switchover) ปิดท้ายด้วย Lab เต็ม
รูปแบบที่ตั้ง **CORE-SW1 เป็น Static RP** และพิสูจน์ด้วย `show ip mroute` ว่า Video Stream ไปถึง
ทั้ง VLAN 10 และ VLAN 40 จริง โดยไม่แตะ VLAN ที่ไม่มีใคร Join เลย

Part นี้วางพื้นฐาน PIM-SM ไว้ให้แล้ว แต่ยังมีอีกหลายเรื่องที่ Enterprise ระดับ CCNP/CCIE ต้องรู้
เพิ่ม — RP แบบ Static ที่เรา Config ไปนั้นมีจุดอ่อนคือ **Single Point of Failure** (ถ้า CORE-SW1
ล่ม RP ก็หายไปทั้ง Domain) และยังมีเทคนิคขั้นสูงอย่าง **Anycast RP, Auto-RP, BSR (Bootstrap
Router)** ที่ช่วยให้ RP มี Redundancy ได้ รวมถึง **PIM-SSM (Source-Specific Multicast)** ที่ใช้
IGMPv3 เลือก Source เจาะจงได้โดยไม่ต้องมี RP เลย — **Part 42 — Multicast Advanced (RP
Redundancy, Auto-RP/BSR, SSM)** จะพาไปแก้จุดอ่อนนี้และเสริม Multicast Design ให้พร้อมใช้งาน
ระดับ Production เต็มรูปแบบ

**ไปต่อ:** [Part 42 — Multicast Advanced (RP Redundancy, Auto-RP/BSR, SSM) →](part-042-multicast-advanced.md)
