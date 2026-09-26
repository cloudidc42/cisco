# Part 46 — Control Plane Policing & Infrastructure Security
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 451–460 จาก 1000**

> ต่อจาก [Part 45 — Advanced ACL & Zone-Based Firewall](part-045-advanced-acl-zbfw.md) ที่เราปิด
> ช่องโหว่ของ **Data Plane** ด้วย ACL ขั้นสูงและ ZBFW (Zone-Based Firewall) อย่างละเอียดแล้ว
> Part นี้จะทำตามสัญญาที่ [Part 22 Step 219](part-022-aaa-device-hardening.md#step-219--control-plane-security-ภาพรวม-preview-และการจำกัด-management-plane)
> ค้างไว้ — ตอนนั้นเราแค่ **Preview** แนวคิด **Control Plane Policing (CoPP)** และทำ VTY
> Access-Class เป็นด่านแรกของ Management Plane เท่านั้น Part 46 นี้คือ Part ที่เจาะลึกเต็มรูปแบบ:
> โมเดล 3-Plane ของอุปกรณ์ Cisco, CoPP ด้วย MQC เดียวกับที่เรียนใน [Part 40](part-040-advanced-qos.md)
> แต่ Apply เข้า **CPU ของอุปกรณ์เอง** แทน Interface, Management Plane Protection (MPP),
> uRPF ป้องกัน Spoofed Source IP, และ BGP TTL Security (GTSM) — ปิดท้ายด้วย Lab ที่ Deploy
> ทุกกลไกพร้อมกันทั้ง CORE-SW1/CORE-SW2/WAN-EDGE-1/WAN-EDGE-2

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 451 | โมเดล 3-Plane ของ Router/Switch (Data/Control/Management) และทำไม ACL/ZBFW เพียงอย่างเดียวไม่พอ |
| 452 | CoPP (Control Plane Policing) พื้นฐาน — MQC เดิมจาก Part 40 แต่ Apply บน `control-plane` |
| 453 | Classify Control-Plane Traffic — Routing Protocol / Management / ICMP / Unknown |
| 454 | Policy-map CoPP เต็มรูปแบบ — `police` แต่ละ Class บน CORE-SW1/CORE-SW2 |
| 455 | CoPPr (Control Plane Protection) — Sub-interface Host/Transit/CEF-Exception |
| 456 | Management Plane Protection (MPP) — `management-interface ... allow` |
| 457 | uRPF (Unicast Reverse Path Forwarding) — ป้องกัน Spoofed Source IP |
| 458 | BGP TTL Security (GTSM) — ป้องกัน BGP Session จาก Off-path Spoofing |
| 459 | Verification & Troubleshooting — CoPP Drop Counter, CPU Spike, OSPF Hello ถูก Drop โดยไม่ตั้งใจ |
| 460 | Lab เต็มรูปแบบ: CoPP + MPP + uRPF + BGP TTL-Security ทั้ง Topology |

---

## Step 451 — โมเดล 3-Plane ของ Router/Switch และทำไม ACL/ZBFW เพียงอย่างเดียวไม่พอ

### ทบทวนภาพรวมจาก Part 22 Step 219 แบบละเอียดขึ้น

ทุกอุปกรณ์ Cisco (Router, Switch, Firewall) แบ่งการทำงานภายในออกเป็น **3 Plane ที่แยกกันทาง
สถาปัตยกรรม** แม้จะทำงานอยู่ในกล่องเดียวกันก็ตาม:

```
┌───────────────────────────────────────────────────────────────────────┐
│                         MANAGEMENT PLANE                                │
│  "บริหารจัดการตัวอุปกรณ์เอง"                                              │
│  SSH, Telnet, SNMP, HTTPS, TACACS+/RADIUS, Syslog, NTP                  │
│  ป้องกันด้วย: AAA (Part 22), VTY ACL (Part 22), MPP (Step 456 Part นี้)   │
├───────────────────────────────────────────────────────────────────────┤
│                         CONTROL PLANE                                   │
│  "สมองของอุปกรณ์ — ตัดสินใจว่าจะ Forward Traffic ไปทางไหน"                  │
│  OSPF/EIGRP/BGP Hello & Update, STP BPDU, ARP, CDP/LLDP, HSRP/VRRP,      │
│  ICMP ที่ต้องสร้างขึ้นเอง (TTL Exceeded, Unreachable)                     │
│  ป้องกันด้วย: CoPP (Step 452-455 Part นี้)                                │
├───────────────────────────────────────────────────────────────────────┤
│                         DATA PLANE                                       │
│  "Traffic ของ User ที่ Forward ผ่านอุปกรณ์เฉยๆ (Transit Traffic)"          │
│  HTTP/HTTPS, DNS Query, File Transfer, VoIP RTP Stream ฯลฯ               │
│  ป้องกันด้วย: ACL, ZBFW (Part 43-45), uRPF (Step 457 Part นี้)             │
└───────────────────────────────────────────────────────────────────────┘
```

| Plane | คำถามที่ตอบ | ประมวลผลที่ไหน | ตัวอย่าง Protocol |
|---|---|---|---|
| **Data Plane** | "Traffic นี้ต้องไปทางไหน" | **Hardware** (ASIC/TCAM, CEF) — เร็วมาก ไม่แตะ CPU หลัก | Traffic ของ User ทั่วไป |
| **Control Plane** | "จะรู้ Topology/Forwarding Table ได้อย่างไร" | **CPU/Route Processor** (Software, Process-Switched) | OSPF, EIGRP, BGP, STP, ARP, HSRP |
| **Management Plane** | "Admin จะเข้าคุมอุปกรณ์อย่างไร" | **CPU/Route Processor** เช่นกัน (มักถูกจัดกลุ่มรวมกับ Control Plane ในทางเทคนิค) | SSH, SNMP, TACACS+, Syslog, NTP |

> **จุดสำคัญที่ต้องเข้าใจ**: ในทางสถาปัตยกรรมจริง Cisco IOS/IOS-XE มี Physical Interface
> เพียงหนึ่งเดียวชื่อ **`control-plane`** ที่รวบรวม Traffic ที่ต้องขึ้นไปประมวลผลที่ CPU
> **ทั้ง Control Plane และ Management Plane** เข้าไว้ในจุดเดียวกัน — เหตุผลที่หลักสูตรนี้แยก
> อธิบาย 2 Plane นี้ต่างหากเพราะ **แหล่งที่มาของ Traffic ต่างกัน** (Control Plane มาจาก
> Router/Switch ตัวอื่นในเครือข่าย ส่วน Management Plane มาจาก Admin/NMS) และ**เครื่องมือป้องกัน
> เพิ่มเติม**ก็ต่างกัน (MPP ใน Step 456 จำกัดเฉพาะ Management Plane ที่ระดับ Interface)

### ทำไม Traffic ที่ถูก Forward "ผ่านๆ" ถึงไม่แตะ CPU

```
                    Router/Switch (Cisco Catalyst 9500 — CORE-SW1)
┌─────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   ┌───────────────────────────────────────────────────────┐         │
│   │              CPU / Route Processor (Software)            │  <--- Control + Management Plane
│   │   OSPF Process, BGP Process, SSH Daemon, SNMP Agent       │       (Process Switching, ช้ากว่ามาก)
│   └───────────────────────▲──────────────────────▲──────────┘         │
│                            │ Punt (เฉพาะ packet ที่│ "ต้องขึ้น CPU")     │
│   ┌────────────────────────┴──────────────────────┴──────────┐        │
│   │         ASIC / TCAM (Hardware — CEF Forwarding)             │  <--- Data Plane
│   │   Ingress Gi1/0/1 ──► [Lookup FIB/ACL/QoS ใน Hardware]      │       (Hardware Switching,
│   │                    ──► Egress Gi1/0/5 (ไม่ผ่าน CPU เลย)     │        เร็วระดับ Wire-speed)
│   └────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
```

Traffic ของ User ทั่วไป (เช่น PC1 คุยกับ Server1) ถ้า Destination ไม่ใช่ตัว Router/Switch เอง
— CEF จะ Lookup FIB Table ที่สร้างไว้ล่วงหน้าใน Hardware (ASIC/TCAM) แล้ว Forward ออก
Egress Interface ทันที **ไม่ผ่าน CPU เลยแม้แต่ Cycle เดียว** นี่คือเหตุผลที่ Switch/Router
รุ่นใหม่ Forward Traffic ได้ระดับ Wire-speed (หลาย Gbps/Tbps) แม้ CPU ตัวจะช้ากว่ามาก

แต่ Packet บางประเภท **ต้องถูก "Punt" ขึ้นไปที่ CPU เสมอ** ไม่ว่า Hardware จะเร็วแค่ไหน เพราะ
ASIC ไม่มี Logic พอที่จะประมวลผล Protocol เหล่านี้:

| ประเภท Packet ที่ต้อง Punt ขึ้น CPU | เหตุผล |
|---|---|
| Routing Protocol Packet (OSPF Hello, BGP Update) | ต้องรัน State Machine ของ Protocol ทั้งหมด (Neighbor FSM, Best Path Selection) |
| ARP Request/Reply | ต้องสร้าง/อัพเดต ARP Table เอง |
| Packet ที่ Destination IP = IP ของอุปกรณ์เอง | SSH, SNMP, Ping มาที่ตัวอุปกรณ์ต้องให้ Process ที่ CPU ตอบ |
| Packet ที่ TTL หมดอายุ (TTL=1 แล้วต้อง Decrement) | ต้องสร้าง ICMP TTL Exceeded ส่งกลับ (Traceroute ใช้ประโยชน์จากจุดนี้) |
| Packet ที่มี IP Options หรือต้อง Fragment | ASIC รุ่นเก่า/กลางจัดการ Options ซับซ้อนไม่ได้ ต้อง Punt |
| CEF Miss (ไม่มี FIB Entry ตรง — Glean) | ต้อง Process Switch เพื่อ Trigger ARP หา Next-hop ใหม่ |

### ทำไม ACL/ZBFW (Data Plane, Part 43-45) เพียงอย่างเดียวไม่พอที่จะปกป้อง Control Plane

นี่คือคำถามที่คนเพิ่งเรียน ACL/ZBFW เข้าใจผิดบ่อยที่สุด: *"ในเมื่อผมมี Interface ACL/ZBFW
กรอง Traffic เข้า-ออกทุก Interface อยู่แล้ว ทำไมยังต้องมี CoPP อีก?"*

| เหตุผล | รายละเอียด |
|---|---|
| **1. Permit ≠ ปลอดภัยจาก CPU** | Attacker ปลอม OSPF Hello ที่ Source/Dest ตรง Protocol จริงทุกอย่างจะผ่าน ACL/ZBFW "Permit" ได้สบายๆ (ACL เช็คแค่ IP/Port ไม่เช็คความถี่) แล้วถูกส่งต่อให้ CPU รัน SPF ซ้ำๆ จนโหลดเต็ม — ACL "ทำงานถูกต้องตามที่ตั้งใจ" แต่ป้องกันเหตุการณ์นี้ไม่ได้ |
| **2. ACL เองไม่มี Concept "จำกัดอัตรา"** | ACL ตอบได้แค่ "ผ่านหรือไม่ผ่าน" ไม่มีแนวคิด "ผ่านได้ แต่จำกัดแค่ X packet/sec" ซึ่งเป็นสิ่งที่ CoPP (`police`) ทำได้โดยเฉพาะ |
| **3. Data Plane ปกป้อง "Traffic ที่ผ่าน" ไม่ใช่ "อุปกรณ์เอง"** | ZBFW มี "Self Zone" ครอบคลุม Traffic ถึงตัวอุปกรณ์ได้บ้าง แต่ยังเป็นการกรองแบบ Permit/Deny ไม่ใช่ Rate-Limit — Attacker ที่ปลอม Source ให้ตรง Policy "Permit" (เช่น ปลอมเป็น Neighbor Router ที่ถูกต้อง) จะยังผ่าน Self Zone ได้สบายๆ เช่นกัน |

**สรุปหลักการสำคัญที่สุดของ Part นี้**: **ACL/ZBFW ตอบคำถาม "ใคร/อะไรได้รับอนุญาต"
ส่วน CoPP ตอบคำถาม "แม้จะได้รับอนุญาตแล้ว จะให้ผ่านได้เร็วแค่ไหน"** — ทั้งสองเป็นแนวป้องกันที่
**เสริมกัน ไม่ใช่แทนกัน** และ Enterprise ที่ทำ Security-in-Depth จริงต้องมีทั้งคู่พร้อมกัน

---

## Step 452 — CoPP (Control Plane Policing) พื้นฐาน

### CoPP ใช้ MQC เดียวกับ Part 40 ทุกประการ — ต่างกันแค่ "จุดที่ Apply"

ข่าวดีสำหรับ Part นี้: เราไม่ต้องเรียนภาษา Config ใหม่เลย — CoPP ใช้ **MQC (Modular QoS CLI)**
model 3 ส่วนเดียวกับที่เรียนเต็มรูปแบบใน [Part 40](part-040-advanced-qos.md) ทุกประการ
(`class-map` → `policy-map` → `service-policy`) ต่างกันเพียงจุดเดียว:

| | QoS ปกติ (Part 40) | CoPP (Part นี้) |
|---|---|---|
| Apply `service-policy` ที่ | **Physical/Logical Interface** เช่น `interface GigabitEthernet0/0/0` | **`control-plane`** — Virtual Interface ที่แทน CPU ของอุปกรณ์เอง |
| ทิศทาง (`input`/`output`) | ควบคุม Traffic ที่ผ่าน Interface นั้นจริงๆ | `input` = Traffic ที่ "เข้าสู่" CPU (Punt) — ทิศทางที่ใช้เกือบทั้งหมด |
| Action ที่ใช้บ่อย | `priority`, `bandwidth`, `shape`, `police`, `random-detect` | **`police` เท่านั้นในทางปฏิบัติ** — CoPP ไม่มีแนวคิด Queue/Shape เพราะ CPU ไม่ใช่ Physical Wire ที่มี Bandwidth ให้ Shape |
| Traffic ที่ถูกควบคุม | Traffic รวมของทุก Flow ที่ผ่าน Interface นั้น | Traffic **รวมจากทุก Interface ของอุปกรณ์** ที่ถูก Punt ขึ้น CPU (Aggregate ทั้งเครื่อง) |

> **จุดที่ต้องจำ**: `control-plane` เป็น **Interface เดียวสำหรับทั้งอุปกรณ์** (ไม่ใช่ต่อ Physical
> Interface) — Policy ที่ Apply เข้าไปจะมีผลกับ Traffic ที่ Punt ขึ้น CPU จาก**ทุก**
> Physical Interface พร้อมกัน (Aggregate CoPP) ต่างจาก QoS Interface ปกติที่แยก Policy ได้
> ต่อ Interface อิสระ

### Syntax พื้นฐาน

```
Router(config)# control-plane
Router(config-cp)# service-policy input POLICY-NAME
```

- `control-plane` — เข้าสู่ Config Mode ของ Control-Plane Interface (ไม่ต้องระบุชื่อ/เลข
  เพราะมีแค่ตัวเดียวต่ออุปกรณ์ ยกเว้นอุปกรณ์ Modular Chassis ขนาดใหญ่ที่รองรับ
  `control-plane slot <slot>` แยกตาม Route Processor แต่ละ Slot — เกินขอบเขต CCNP ENCOR)
- `service-policy input POLICY-NAME` — ผูก Policy-map เข้ากับทิศทาง **input** เท่านั้น
  (Traffic ที่วิ่ง**เข้า**สู่ CPU) ซึ่งเป็นทิศทางที่ใช้เกือบ 100% ของการ Deploy CoPP จริง
  (`service-policy output` มีในบางแพลตฟอร์มสำหรับควบคุม Traffic ที่ CPU **สร้างขึ้นเอง**
  ส่งออกไป แต่ใช้น้อยมากในทางปฏิบัติ — ไม่ครอบคลุมใน Part นี้)

### ตัวอย่างขั้นต่ำที่สุด (Skeleton — ยังไม่สมบูรณ์ รายละเอียดเต็มอยู่ Step 453-454)

```
CORE-SW1(config)# class-map match-any COPP-CRITICAL
CORE-SW1(config-cmap)#  match access-group name COPP-ACL-ROUTING
CORE-SW1(config-cmap)# exit
!
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)#  class COPP-CRITICAL
CORE-SW1(config-pmap-c)#   police cir 10000000 bc 512000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap-c)# exit
CORE-SW1(config-pmap)# exit
!
CORE-SW1(config)# control-plane
CORE-SW1(config-cp)#  service-policy input COPP-POLICY
```

โครงสร้างนี้เหมือน MQC ทุกตัวอักษรจาก Part 40 — สิ่งที่ต้องออกแบบเพิ่มคือ **จะแยก Class อะไร
บ้าง** (Step 453) และ **จะตั้ง `police` Rate เท่าไหร่ต่อ Class** (Step 454)

---

## Step 453 — Classify Control-Plane Traffic: Routing Protocol / Management / ICMP / Unknown

### หลักการออกแบบ Class สำหรับ CoPP — เรียงตาม "ความสำคัญต่อความอยู่รอดของเครือข่าย"

```
┌─────────────────────────────────────────────────────────────────────┐
│  COPP-CRITICAL     — Routing Protocol (OSPF/BGP/EIGRP), HSRP           │  ห้าม Drop เกือบทั้งหมด
│  (สำคัญที่สุด)        ถ้า Drop = Network แตก/Reconverge/Session หลุด       │  Rate สูงสุด, Burst ใหญ่
├─────────────────────────────────────────────────────────────────────┤
│  COPP-IMPORTANT    — Management (SSH, SNMP, TACACS+, NTP)               │  สำคัญรองลงมา
│  (สำคัญรองลงมา)      ถ้า Drop = Admin เข้าอุปกรณ์ไม่ได้ ลำบากตอน Incident    │  Rate ปานกลาง
├─────────────────────────────────────────────────────────────────────┤
│  COPP-NORMAL       — ICMP (Ping, Traceroute)                            │  Vector โจมตี DoS ที่พบบ่อย
│  (ปานกลาง)           มีประโยชน์ต่อ Troubleshoot แต่เป็นเป้าโจมตี DoS สุดคลาสสิก│  Rate จำกัดพอสมควร
├─────────────────────────────────────────────────────────────────────┤
│  class-default     — Traffic อื่นทั้งหมดที่ไม่ Match Class ข้างบน           │  น่าสงสัยที่สุด
│  (ต่ำสุด/ไม่ทราบที่มา) รวม Traffic แปลกปลอม, Scan, Spoofed Packet ที่พลาด ACL │  Rate ต่ำที่สุด บังคับ
└─────────────────────────────────────────────────────────────────────┘
```

### 1) ACL สำหรับ Routing Protocol + HSRP (COPP-CRITICAL)

```
CORE-SW1(config)# ip access-list extended COPP-ACL-ROUTING
CORE-SW1(config-ext-nacl)# remark ** OSPF Multicast Hello/LSA (224.0.0.5 = AllSPFRouters, .6 = AllDRouters) **
CORE-SW1(config-ext-nacl)# permit ospf any host 224.0.0.5
CORE-SW1(config-ext-nacl)# permit ospf any host 224.0.0.6
CORE-SW1(config-ext-nacl)# remark ** OSPF Unicast (เช่น Virtual-link บางกรณี) **
CORE-SW1(config-ext-nacl)# permit ospf any any
CORE-SW1(config-ext-nacl)# remark ** EIGRP Multicast/Unicast (224.0.0.10 = AllEIGRPRouters) — เผื่อ Segment ที่รัน EIGRP **
CORE-SW1(config-ext-nacl)# permit eigrp any host 224.0.0.10
CORE-SW1(config-ext-nacl)# permit eigrp any any
CORE-SW1(config-ext-nacl)# remark ** BGP TCP/179 ทั้งสองทิศทาง (Passive/Active Open) **
CORE-SW1(config-ext-nacl)# permit tcp any eq 179 any
CORE-SW1(config-ext-nacl)# permit tcp any any eq 179
CORE-SW1(config-ext-nacl)# remark ** HSRP — UDP/1985 ไปยัง Multicast Group (Part 18) **
CORE-SW1(config-ext-nacl)# permit udp any host 224.0.0.2 eq 1985
CORE-SW1(config-ext-nacl)# permit udp any host 224.0.0.102 eq 1985
CORE-SW1(config-ext-nacl)# exit
```

> **ข้อจำกัดที่ต้องรู้ — ARP ไม่สามารถ Classify ผ่าน `match access-group` ได้**: ARP เป็น
> Ethernet Frame ที่ใช้ EtherType `0x0806` ไม่ใช่ IP Packet เลย (ไม่มี IP Header ให้ ACL
> ตรวจสอบ) ดังนั้น `ip access-list` **จับ ARP ไม่ได้** — CoPP ในรูปแบบพื้นฐานนี้จึงไม่ครอบคลุม
> การป้องกัน ARP Flood โดยตรง ต้องพึ่ง Port Security + DAI (Dynamic ARP Inspection) จาก
> [Part 21](part-021-switch-security.md) แทน — นี่คือตัวอย่างที่ดีว่าทำไม Security-in-Depth
> ต้องใช้**หลายกลไกประกอบกัน** ไม่มีกลไกใดกลไกเดียวครอบคลุมทุกภัยคุกคาม

### 2) ACL สำหรับ Management Plane (COPP-IMPORTANT)

```
CORE-SW1(config)# ip access-list extended COPP-ACL-MGMT
CORE-SW1(config-ext-nacl)# remark ** จำกัดเฉพาะ Traffic จาก MGMT VLAN 99 เท่านั้น (เสริม Defense-in-Depth กับ VTY ACL Part 22) **
CORE-SW1(config-ext-nacl)# permit tcp 10.10.99.0 0.0.0.255 any eq 22
CORE-SW1(config-ext-nacl)# permit tcp 10.10.99.0 0.0.0.255 any eq 49
CORE-SW1(config-ext-nacl)# permit udp 10.10.99.0 0.0.0.255 any eq 161
CORE-SW1(config-ext-nacl)# permit udp any eq 162 10.10.99.0 0.0.0.255
CORE-SW1(config-ext-nacl)# permit udp 10.10.99.0 0.0.0.255 any eq 123
CORE-SW1(config-ext-nacl)# exit
```

| Port | Protocol | หมายเหตุ |
|---|---|---|
| TCP/22 | SSH | Admin login (Part 2, 22) |
| TCP/49 | TACACS+ | AAA Query ไป AAA-SRV (Part 22) |
| UDP/161 | SNMP GET/SET | NMS Poll สถานะ (Part 17) |
| UDP/162 | SNMP Trap | Router ส่ง Trap **ออกไป** ยัง NMS — ทิศทาง Source/Dest จึงสลับกับบรรทัดอื่น |
| UDP/123 | NTP | Time Sync (Part 17) |

### 3) ACL สำหรับ ICMP (COPP-NORMAL)

```
CORE-SW1(config)# ip access-list extended COPP-ACL-ICMP
CORE-SW1(config-ext-nacl)# permit icmp any any echo
CORE-SW1(config-ext-nacl)# permit icmp any any echo-reply
CORE-SW1(config-ext-nacl)# permit icmp any any ttl-exceeded
CORE-SW1(config-ext-nacl)# permit icmp any any unreachable
CORE-SW1(config-ext-nacl)# permit icmp any any port-unreachable
CORE-SW1(config-ext-nacl)# exit
```

### Class-map เต็มรูปแบบ (Match-any ทุกตัว — Class ใดก็ตามที่ Match ACL จะเข้า Class นั้น)

```
CORE-SW1(config)# class-map match-any COPP-CRITICAL
CORE-SW1(config-cmap)# match access-group name COPP-ACL-ROUTING
CORE-SW1(config-cmap)# exit
!
CORE-SW1(config)# class-map match-any COPP-IMPORTANT
CORE-SW1(config-cmap)# match access-group name COPP-ACL-MGMT
CORE-SW1(config-cmap)# exit
!
CORE-SW1(config)# class-map match-any COPP-NORMAL
CORE-SW1(config-cmap)# match access-group name COPP-ACL-ICMP
CORE-SW1(config-cmap)# exit
```

> **`class-default` ไม่ต้องสร้างเอง** — เหมือนที่เรียนใน [Part 40 Step 392](part-040-advanced-qos.md)
> IOS มีให้อัตโนมัติในทุก `policy-map` เสมอ และจะจับ Traffic **ทุกตัวที่ไม่ Match Class ใดข้างบน
> เลย** — สำหรับ CoPP นี่คือ Class ที่ **น่าสงสัยที่สุด** เพราะควรมี Traffic น้อยมากหรือไม่มีเลย
> ที่ต้อง Punt ขึ้น CPU โดยไม่ตรงกับ Routing Protocol/Management/ICMP ที่รู้จัก — ถ้า
> `class-default` มี Traffic จำนวนมากผิดปกติ = สัญญาณเตือนว่ามีอะไรแปลกปลอมกำลังพยายาม
> คุยกับ CPU ของอุปกรณ์อยู่ (Reconnaissance Scan, Malformed Packet, หรือ Attack ที่กำลังเกิดขึ้น)

---

## Step 454 — Policy-map CoPP เต็มรูปแบบ: `police` แต่ละ Class

### หลักคิดการตั้ง Rate — "Generous แต่ยังมีเพดาน" สำหรับ Critical, "เข้มงวดขึ้นเรื่อยๆ" สำหรับ Class ที่เหลือ

```
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)# class COPP-CRITICAL
CORE-SW1(config-pmap-c)#  police cir 10000000 bc 512000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap-c)# exit
CORE-SW1(config-pmap)# class COPP-IMPORTANT
CORE-SW1(config-pmap-c)#  police cir 2000000 bc 256000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap-c)# exit
CORE-SW1(config-pmap)# class COPP-NORMAL
CORE-SW1(config-pmap-c)#  police cir 1000000 bc 128000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap-c)# exit
CORE-SW1(config-pmap)# class class-default
CORE-SW1(config-pmap-c)#  police cir 32000 bc 8000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap-c)# exit
CORE-SW1(config-pmap)# exit
```

| Class | CIR (Rate) | Bc (Burst) | เหตุผลของค่านี้ |
|---|---|---|---|
| **COPP-CRITICAL** | 10 Mbps | 512,000 bytes | Routing Protocol Hello/Update ปริมาณจริงน้อยมาก (หลัก KB/s) แต่ตั้งเพดานสูงมากเพื่อรองรับ **LSA Flooding ช่วง Reconverge ครั้งใหญ่** (Link Flap พร้อมกันหลายเส้น) ไม่ให้โดน Drop โดยไม่ตั้งใจ — Rate นี้แทบไม่มีทางถูกใช้เต็มในสภาวะปกติเลย มันคือ **เพดานฉุกเฉิน** กันไว้เผื่อ Spoofed Flood จริงๆ เท่านั้น |
| **COPP-IMPORTANT** | 2 Mbps | 256,000 bytes | SSH Session/SNMP Poll ปริมาณปานกลาง เพียงพอสำหรับ Admin หลายคน Login พร้อมกัน + NMS Poll ถี่ |
| **COPP-NORMAL** | 1 Mbps | 128,000 bytes | Ping/Traceroute ปกติใช้น้อยมาก แต่ ICMP เป็น Vector DoS คลาสสิก (Ping Flood, Smurf รุ่นเก่า) จึงตั้งเพดานไว้ปานกลาง-ต่ำ |
| **class-default** | 32 kbps | 8,000 bytes | Traffic ไม่รู้จักควรมีน้อยที่สุดหรือไม่มีเลย ตั้ง Rate ต่ำสุดเพื่อไม่ให้ Traffic แปลกปลอมจำนวนมากไปแย่ง CPU Cycle จาก Class อื่น |

### Apply เข้า Control-Plane

```
CORE-SW1(config)# control-plane
CORE-SW1(config-cp)# service-policy input COPP-POLICY
CORE-SW1(config-cp)# exit
```

ทำ Pattern เดียวกันนี้กับ **CORE-SW2** (เปลี่ยนแค่ Hostname prompt) — Policy-map/Class-map/
ACL ชื่อเดียวกันทั้งหมด ทำให้ Config **Portable** ข้ามอุปกรณ์ที่มี Role เดียวกันได้ทันที
(รายละเอียดเต็มรูปแบบทั้ง Topology อยู่ใน Step 460)

### `conform-action transmit` แม้กับ `exceed-action drop` ไม่ได้แปลว่า "ปลอดภัย 100%"

> **ข้อควรระวังสำคัญที่สุดของ CoPP**: การตั้ง `exceed-action drop` กับ Class Routing Protocol
> หมายความว่า **ถ้า Rate เกินเพดานจริง (แม้จะเป็น Hello ของจริงไม่ใช่ Attack) Hello เหล่านั้น
> จะถูก Drop ด้วย** — นี่คือเหตุผลที่ Step 459 จะสอนวิธี Diagnose สถานการณ์ที่ CoPP Policy
> เข้มงวดเกินไปจนกระทบ OSPF/BGP ของจริง ต้องหาจุดสมดุลระหว่าง "เข้มงวดพอจะกัน Attack"
> กับ "หลวมพอจะไม่กระทบ Traffic ที่ถูกต้อง" เสมอ — Rate ในตารางข้างบนเป็น**จุดเริ่มต้นที่สมเหตุ
> สมผล** ไม่ใช่ค่าตายตัวที่ใช้ได้กับทุก Topology (Topology ใหญ่กว่านี้มาก อาจมี LSA จำนวนมากกว่า
> ต้องปรับ Bc ให้ใหญ่ขึ้น)

---

## Step 455 — CoPPr (Control Plane Protection): Sub-interface Host/Transit/CEF-Exception

### ข้อจำกัดของ CoPP แบบพื้นฐาน (Aggregate CoPP)

CoPP ที่เรียนใน Step 452-454 เป็น **Aggregate CoPP** — Policy เดียวควบคุม Traffic **ทุกประเภท
รวมกัน** ที่ Punt ขึ้น CPU ปัญหาคือ Traffic ที่ Punt ขึ้น CPU ไม่ได้มีสาเหตุเดียวเสมอไป
บางแพลตฟอร์ม (โดยเฉพาะ ISR/ASR Router รุ่นที่รองรับ IOS เต็มรูปแบบ) จึงมี
**CoPPr (Control Plane Protection)** ที่แบ่ง Control-Plane Interface ออกเป็น **Sub-interface
ย่อย 3 ประเภท** เพื่อควบคุมแยกกันละเอียดขึ้น:

```
control-plane
 !
 control-plane host              ← Traffic ที่ Destination = ตัวอุปกรณ์เองโดยตรง (SSH, Ping to device)
 control-plane transit           ← Traffic ที่ต้อง Punt เพราะเหตุผลอื่น แต่ไม่ใช่ Destination ตัวเอง
                                    (เช่น IP Options, ต้อง Fragment) แล้วยังต้อง Forward ต่อ
 control-plane cef-exception     ← Traffic ที่ CEF ประมวลผล Hardware ไม่ได้ (Protocol แปลกๆ,
                                    Malformed Packet) ต้อง Punt ไป Software Path เท่านั้น
```

| ประเภท Sub-interface | ตัวอย่าง Traffic | ใช้ Policy แยกเพื่อ |
|---|---|---|
| `control-plane host` | SSH/SNMP มาที่ Interface IP ของอุปกรณ์, Routing Protocol Hello | ป้องกัน DoS ที่มุ่งเป้าโจมตี "ตัวอุปกรณ์" ตรงๆ |
| `control-plane transit` | Packet ที่มี IP Options ที่กำลัง Transit ผ่านไปที่อื่น | ป้องกันไม่ให้ Transit Traffic แปลกๆ ไปแย่ง CPU จาก Traffic ที่ Destination คือตัวอุปกรณ์ |
| `control-plane cef-exception` | Traffic ที่ Hardware Forward ไม่ได้เลยต้อง Software Switch | จำกัดไม่ให้ Traffic ประเภทนี้ (ซึ่งกิน CPU ต่อ Packet สูงกว่ามาก) ท่วม CPU |

### ความพร้อมใช้งานตามแพลตฟอร์ม — สิ่งที่ต้องรู้ก่อนเลือกใช้

| แพลตฟอร์มใน Lab หลักสูตรนี้ | รองรับ CoPPr (Granular) หรือไม่ |
|---|---|
| **WAN-EDGE-1/2 (ISR4451)** | รองรับเต็มรูปแบบ — เป็น Router ที่ออกแบบมาสำหรับ CoPPr โดยเฉพาะ |
| **CORE-SW1/2 (Catalyst 9500)** | รองรับเฉพาะ **Aggregate CoPP** ผ่าน `control-plane` ตัวเดียว — Catalyst Switch ส่วนใหญ่ที่เน้น Hardware Forwarding ไม่มี Sub-interface แบบ Host/Transit/CEF-Exception ให้ใช้ (ดีไซน์ ASIC ต่างจาก Router ที่เน้น Software Feature เต็มรูปแบบ) |

> **ข้อสรุปเชิงปฏิบัติ**: Lab หลักสูตรนี้จะใช้ **Aggregate CoPP** (Step 452-454) เป็นหลักบน
> **ทั้ง CORE-SW1/2 และ WAN-EDGE-1/2** เพื่อความสอดคล้องและง่ายต่อการดูแล (Config เดียวกันใช้
> ได้ทุกอุปกรณ์) — CoPPr แบบ Sub-interface เหมาะกับ Environment ที่ต้องการควบคุมละเอียดสุดขีด
> จริงๆ บน Router ขนาดใหญ่ (Service Provider Edge, Internet Peering Router) ซึ่งเกินขอบเขต
> CCNP ENCOR แต่ควรรู้จักไว้สำหรับ CCIE Enterprise Infrastructure ที่อาจถูกถามถึง

---

## Step 456 — Management Plane Protection (MPP)

### แนวคิด: จำกัด "Interface ที่ Management Protocol ยอมรับ Traffic" ไม่ใช่แค่ "ใครมี Password"

VTY Access-Class จาก [Part 22 Step 219](part-022-aaa-device-hardening.md#step-219--control-plane-security-ภาพรวม-preview-และการจำกัด-management-plane)
ทำงานที่ **Line VTY** — กรอง Source IP ก่อนเข้าสู่ AAA Authentication แต่ยังเป็นการกรองที่
"ระดับ Line" (Layer สูงกว่า Interface) **MPP (Management Plane Protection)** เพิ่มแนวป้องกัน
อีกชั้นที่ **ระดับ Interface โดยตรง** — บอก IOS ว่า **"Protocol การบริหารจัดการนี้ (SSH/SNMP/
HTTPS ฯลฯ) จะยอมรับ Traffic จาก Interface ไหนเท่านั้น"** แม้ Interface อื่นจะ Routing ไปถึง
อุปกรณ์ได้ทาง Network ก็ตาม Traffic ที่มาจาก Interface ที่ไม่ได้รับอนุญาตจะถูก**ปฏิเสธตั้งแต่
ระดับ Listener ของ Protocol นั้น** — ไม่ต้องพึ่ง ACL ที่ต้องเขียนแยกทุก Line/Protocol เอง

### Syntax

```
Router(config)# control-plane host
Router(config-cp-host)# management-interface <interface> allow <protocol1> [protocol2] ...
```

Protocol ที่รองรับ (ขึ้นกับ IOS/IOS-XE Version): `ssh`, `telnet`, `http`, `https`, `snmp`,
`tftp` — เลือกได้หลายตัวในบรรทัดเดียว

### Worked Config — WAN-EDGE-1 (จำกัด Management ให้เฉพาะทาง Loopback0/MGMT เท่านั้น)

MPP เหมาะกับ Router-class Platform (ISR/ASR) มากกว่า Catalyst Switch ที่เน้น SVI เป็นหลัก
— Lab นี้จะสาธิตบน **WAN-EDGE-1** ซึ่งมี Interface สู่ Internet (`GigabitEthernet0/0/0`,
Internet-facing) และ Interface สู่ CORE-SW1 (`GigabitEthernet0/0/1`, LAN-facing) — เราต้องการ
ให้ SSH/SNMP ยอมรับ Traffic **จากฝั่ง LAN (ไปยัง CORE-SW1) เท่านั้น** ไม่ยอมรับแม้จะมี Traffic
มาจากฝั่ง Internet-facing Interface ก็ตาม (ซึ่งไม่ควรมีใครพยายาม SSH เข้ามาจากทางนั้นอยู่แล้ว
แต่ MPP ทำให้ **แม้ ACL ฝั่งนั้นพลาดไปจริงๆ ก็ยังมีแนวป้องกันซ้อนอีกชั้น**):

```
WAN-EDGE-1(config)# control-plane host
WAN-EDGE-1(config-cp-host)# management-interface GigabitEthernet0/0/1 allow ssh snmp
WAN-EDGE-1(config-cp-host)# exit
```

ผลลัพธ์: แม้ Attacker จะพยายาม SSH เข้ามาทาง `GigabitEthernet0/0/0` (Internet-facing) และ
รู้ Username/Password ที่ถูกต้องทุกตัวอักษร (สมมติว่า TACACS+/Local Auth หลุดไปแล้วด้วย) —
**Connection จะไม่มีทางถูกตอบรับเลยแม้แต่ระดับ TCP SYN-ACK** เพราะ SSH Daemon **ไม่ Listen**
บน Interface นั้นตามที่ `management-interface` กำหนดไว้

### เปรียบเทียบ MPP กับ VTY ACL (Part 22) — ใช้คู่กันเสมอ ไม่ใช่แทนกัน

| คุณสมบัติ | VTY Access-Class (Part 22) | MPP (Part นี้) |
|---|---|---|
| ระดับที่ทำงาน | Line (VTY 0-15) | Interface โดยตรง |
| กรองด้วย | Source IP (ACL) | **Interface ที่ Packet เข้ามา** ไม่ใช่ Source IP |
| ครอบคลุม Protocol | เฉพาะที่ผ่าน Line (SSH/Telnet) | SSH, SNMP, HTTP/HTTPS, TFTP (กว้างกว่า) |
| Config ที่ | `line vty 0 15` | `control-plane host` |
| จุดแข็ง | ควบคุม "ใคร" (Source Subnet) ได้ละเอียด | ควบคุม "ทางไหน" (Interface) ได้แม้ Source IP จะปลอมยังไงก็ผ่าน Interface ผิดไม่ได้ |

> **หลักการ Defense-in-Depth**: ใช้ **VTY ACL** เพื่อจำกัด Source Subnet (`10.10.99.0/24`
> เท่านั้น) **ร่วมกับ MPP** เพื่อจำกัด Interface ทางเข้า (เฉพาะ LAN-facing Interface) — สอง
> ชั้นนี้ต่างมุมกัน แม้ชั้นหนึ่งถูก Bypass (เช่น Attacker อยู่ใน Subnet ที่ถูกต้องแต่เข้ามาผิดทาง
> Interface, หรือ Source IP ถูก Spoof ให้ตรงกับ Subnet ที่อนุญาต) อีกชั้นยังทำงานอยู่

---

## Step 457 — uRPF (Unicast Reverse Path Forwarding)

### แนวคิด: ถ้า Return Path ไปยัง Source IP ไม่สมเหตุสมผล = Packet นี้น่าจะปลอม

**Source IP Spoofing** คือเทคนิคพื้นฐานที่สุดของการโจมตี DoS/DDoS — Attacker ปลอม Source IP
ของ Packet ให้เป็น IP อะไรก็ได้ (มักปลอมเป็น IP ของ Victim อื่นเพื่อทำ **Reflection Attack**
หรือปลอมเป็น IP ที่ไม่มีจริงเพื่อซ่อนตัวตนจริง) **uRPF** แก้ปัญหานี้ด้วยการเช็คว่า **"ถ้าจะ
ส่ง Packet กลับไปหา Source IP นี้ Router จะต้อง Route ออกทาง Interface เดียวกันกับที่ Packet
นี้เข้ามาหรือไม่"** — ถ้าคำตอบคือ "ไม่" (Asymmetric อย่างไม่สมเหตุสมผล) = Packet นี้น่าสงสัย
ว่า Source IP ถูกปลอม และจะถูก Drop ทันทีที่ Interface (ก่อนเข้า CPU/Control Plane ด้วยซ้ำ
ในหลายแพลตฟอร์ม — เป็นการป้องกันทั้ง Data Plane และลด Load ที่จะไปกระทบ Control Plane
พร้อมกันในตัว)

```
                    uRPF Strict Mode — ตรวจสอบทิศทางที่ Packet เข้ามา
┌────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   Packet เข้ามาทาง Gi0/0/0    Source IP = 8.8.8.8                     │
│         │                                                             │
│         ▼                                                             │
│   Router เช็ค FIB: "ถ้าจะส่งกลับไปหา 8.8.8.8 จะ Route ออกทาง Interface   │
│                     ไหน?"                                             │
│         │                                                             │
│         ├─ คำตอบ = Gi0/0/0 (Interface เดียวกับที่เข้ามา) → PASS (ปกติ)  │
│         └─ คำตอบ = Interface อื่น (หรือไม่มี Route เลย)  → FAIL (Drop!)  │
│                                                                       │
└────────────────────────────────────────────────────────────────────┘
```

### Strict Mode vs Loose Mode

| Mode | คำสั่ง | เงื่อนไข Pass | ใช้เมื่อ |
|---|---|---|---|
| **Strict** | `ip verify unicast source reachable-via rx` | Return Path ต้องออกทาง **Interface เดียวกัน** ที่ Packet เข้ามาเท่านั้น (`rx` = Receive Interface) | Single-homed Link ที่ Traffic เข้า-ออกทางเดียวแน่นอน (เช่น WAN Edge ไป ISP เดียว ไม่มี Asymmetric Routing) |
| **Loose** | `ip verify unicast source reachable-via any` | Return Path ออกทาง **Interface ใดก็ได้** ของ Router (ขอแค่มี Route ไปหา Source ใน FIB จริง) | Environment ที่มี Asymmetric Routing ปกติ (Multihomed, ECMP, Traffic Engineering ที่ Path ขาไป/กลับไม่ตรงกัน) |

### Worked Config — WAN-EDGE-1: Internet-facing Interface (Strict Mode)

Interface `GigabitEthernet0/0/0` ของ **WAN-EDGE-1** เชื่อมต่อไปยัง **ISP-RTR**
(`203.0.113.2/30`, ดู [00-ip-address-plan.md](00-ip-address-plan.md)) — เป็นจุดที่ Traffic
จาก Internet เข้าสู่องค์กร และเป็นจุดที่เหมาะสมที่สุดที่จะเช็ค Source IP Spoofing เพราะ
Traffic ขาเข้าทาง Interface นี้ **ควรมี Source IP ที่เป็น Public IP จาก Internet เท่านั้น**
(ไม่ควรมี Source IP เป็น Private IP ภายในองค์กรของเราเองปลอมมาจากทาง Internet):

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description ** eBGP Link to ISP-RTR (AS 65000) - Primary Internet **
WAN-EDGE-1(config-if)# ip verify unicast source reachable-via rx
WAN-EDGE-1(config-if)# exit
```

> **ทำไมใช้ Strict Mode ได้บน Interface นี้**: `GigabitEthernet0/0/0` เป็น **Point-to-Point
> Link เดียวไปยัง ISP-RTR ตัวเดียว** (ไม่มี ECMP หรือ Path อื่นสำหรับ Traffic ขาเข้าทาง
> Internet ไปยัง WAN-EDGE-1 ตัวนี้โดยเฉพาะ) — Return Path ไปหา Source IP ใดๆ จาก Internet
> ก็ต้องออกทาง Interface เดียวกันนี้เสมอ (Default Route ชี้ไปทาง ISP-RTR) ทำให้ Strict Mode
> ใช้ได้โดยไม่กระทบ Traffic ที่ถูกต้อง

### เมื่อ Strict Mode ใช้ไม่ได้ — WAN-EDGE-2 กับ Asymmetric Routing จาก Cross-link

ตาม [00-ip-address-plan.md](00-ip-address-plan.md) **WAN-EDGE-1 ↔ WAN-EDGE-2 มี Cross-link**
(`10.10.254.8/30`) และทั้งสองต่อ ISP คนละ Link (Dual-homed Internet Edge จาก
[Part 29-30](part-029-bgp-fundamentals.md)) — เป็นไปได้ที่ Traffic ขาเข้า Internet บางเส้น
จะเข้าทาง WAN-EDGE-1 แต่ BGP Best Path เลือกตอบกลับผ่าน WAN-EDGE-2 (Asymmetric ตามการ
ออกแบบ Redundancy ที่ตั้งใจ ไม่ใช่ความผิดปกติ) — ถ้าใช้ Strict Mode ที่นี่จะ Drop Traffic
ที่ถูกต้องโดยไม่ตั้งใจ (False Positive) จึงต้องใช้ **Loose Mode** แทน:

```
WAN-EDGE-2(config)# interface GigabitEthernet0/0/0
WAN-EDGE-2(config-if)# description ** eBGP Link to ISP-RTR (AS 65000) - Secondary Internet **
WAN-EDGE-2(config-if)# ip verify unicast source reachable-via any allow-default
WAN-EDGE-2(config-if)# exit
```

- `allow-default` — อนุญาตให้ uRPF ยอมรับ Packet ได้แม้ Route ที่เจอใน FIB สำหรับ Source
  นั้นเป็นแค่ **Default Route (0.0.0.0/0)** ไม่ใช่ Route เฉพาะเจาะจง — จำเป็นสำหรับ WAN Edge
  ที่รับแค่ Default Route จาก ISP (ไม่ได้รับ Full BGP Table) เพราะถ้าไม่มี Option นี้ Packet
  จาก Public IP ส่วนใหญ่ในโลกอินเทอร์เน็ตที่ไม่มี Route เฉพาะใน FIB จะถูก Drop หมด (เกินความ
  ตั้งใจของ Loose Mode)

### ตรวจสอบผลลัพธ์

```
WAN-EDGE-1# show cef interface GigabitEthernet0/0/0 | include Unicast RPF
  IP unicast RPF check is enabled

WAN-EDGE-1# show ip interface GigabitEthernet0/0/0 | include RPF
  IP verify source reachable-via RX, allow default
  0 verification drops on this interface
```

---

## Step 458 — BGP TTL Security (GTSM — Generalized TTL Security Mechanism)

### ทบทวน: ทำไม BGP Session ถึงถูกโจมตีด้วย Off-path Spoofing ได้

ย้อนกลับไป [Part 29-30](part-029-bgp-fundamentals.md) เราคอนฟิก eBGP ระหว่าง **WAN-EDGE-1**
(`203.0.113.2`, AS 65001) กับ **ISP-RTR** (`203.0.113.1`, AS 65000) — Session BGP นี้ใช้
TCP Port 179 ธรรมดา ซึ่งมีช่องโหว่คลาสสิกคือ **Attacker ที่ไม่ได้อยู่บน Link เดียวกันจริง
(Off-path)** อาจพยายามปลอม Source IP เป็น `203.0.113.1` (ISP-RTR) ส่ง TCP RST หรือ Packet
ปลอมไปยัง `203.0.113.2` เพื่อพยายาม Reset/Hijack BGP Session — แม้ Attacker จะทำแบบนี้ได้
ยากขึ้นมากถ้ามี TCP Sequence Number ที่คาดเดาไม่ได้ (Modern TCP Stack) แต่ยังเป็น Attack
Vector ที่ต้องป้องกันเพิ่มสำหรับ BGP โดยเฉพาะ เพราะ BGP Session ที่หลุดหมายถึง **สูญเสียเส้นทาง
Internet ทั้งหมดขององค์กรทันที**

### หลักการ GTSM (RFC 5082): ใช้ TTL เป็นเครื่องพิสูจน์ "ระยะทางทางกายภาพ"

```
                     GTSM — ใช้ TTL พิสูจน์ว่า Packet มาจาก "ใกล้ๆ" จริง
┌──────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   ISP-RTR (Directly Connected — 1 Hop)          Off-path Attacker         │
│   ส่ง Packet TTL=255 ────────►                   (อยู่ไกลออกไปหลาย Hop)     │
│                              WAN-EDGE-1           ส่ง Packet ปลอม           │
│                              ตรวจ TTL ที่ได้รับ      Source IP = 203.0.113.1  │
│                              ต้อง >= (255-hops+1)  แต่ TTL จะถูก Router      │
│                                                    ระหว่างทาง Decrement      │
│                              TTL=255 (1 hop)       จนเหลือน้อยกว่าเกณฑ์       │
│                              ──► PASS ✅            ──► FAIL ❌ Drop ทันที    │
└──────────────────────────────────────────────────────────────────────┘
```

หลักการคือ Neighbor ที่ **อยู่ใกล้กันจริง (Directly Connected หรือกี่ Hop ตามที่ Config)**
จะส่ง Packet ด้วย TTL ที่ **แทบไม่ถูก Decrement เลย** (เพราะยังไม่ผ่าน Router คั่นกลางกี่ตัว)
ในขณะที่ Attacker ที่อยู่ **ไกลออกไปหลาย Hop** (แม้จะปลอม Source IP ได้ แต่ **ปลอมค่า TTL
ที่ Router ตัวกลางแต่ละตัวจะ Decrement ไม่ได้**) Packet ของ Attacker จะมาถึงด้วย TTL ที่
ต่ำกว่าเกณฑ์ที่ตั้งไว้เสมอ — GTSM จึง Drop Packet ที่ TTL ต่ำเกินไปตั้งแต่ก่อนถึง TCP Stack
ของ BGP Process เลย

### Syntax และการคำนวณเกณฑ์ TTL

```
Router(config-router)# neighbor <ip> ttl-security hops <N>
```

- `<N>` = จำนวน Hop สูงสุดที่ยอมรับระหว่าง Router ตัวนี้กับ Neighbor (ปกติ `1` สำหรับ eBGP
  Directly Connected เหมือนกรณีของเรา)
- IOS จะคำนวณเกณฑ์ TTL ต่ำสุดที่ยอมรับ = **`255 - N + 1`** — Packet ที่มี TTL ต่ำกว่านี้จะถูก
  Drop ทันที (ถ้า `N=1` → เกณฑ์ = 255 ซึ่งหมายความว่า**ต้องเป็น TTL สูงสุดเท่านั้น**เท่ากับ
  Neighbor ต้อง Directly Connected จริง ไม่มี Router คั่นกลางแม้แต่ตัวเดียว)

### Worked Config — WAN-EDGE-1 ↔ ISP-RTR

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 ttl-security hops 1
WAN-EDGE-1(config-router)# exit
```

ทำเช่นเดียวกันฝั่ง **ISP-RTR** (ต้อง Config **ทั้งสองฝั่ง** เพื่อให้ Session ทำงานถูกต้อง —
GTSM เป็น Feature ที่ต้อง Symmetric):

```
ISP-RTR(config)# router bgp 65000
ISP-RTR(config-router)# neighbor 203.0.113.2 ttl-security hops 1
ISP-RTR(config-router)# exit
```

ทำ Pattern เดียวกันกับ **WAN-EDGE-2 ↔ ISP-RTR** (`203.0.113.6` ↔ `203.0.113.5`,
AS 65001 ↔ AS 65000) ตาม [Part 29 Step 287](part-029-bgp-fundamentals.md)

### ข้อจำกัดสำคัญ: `ttl-security` กับ `ebgp-multihop` ใช้พร้อมกันไม่ได้

```
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 ebgp-multihop 2
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 ttl-security hops 1
% ttl-security ไม่สามารถใช้ร่วมกับ ebgp-multihop ได้ (Conflict — ต้องเลือกใช้อย่างใดอย่างหนึ่ง)
```

เหตุผลคือทั้งสองคำสั่งควบคุมพฤติกรรม TTL ของ BGP Session ไปในทิศทางตรงกันข้าม —
`ebgp-multihop` **ผ่อนคลาย** ให้ Session ทำงานได้แม้ TTL ต่ำ (สำหรับ Neighbor ที่อยู่ไกล
หลาย Hop โดยชอบธรรม เช่น eBGP Multihop จาก [Part 29 Step 287](part-029-bgp-fundamentals.md))
ในขณะที่ `ttl-security` **เข้มงวด** ให้ต้อง TTL สูงเท่านั้น — ถ้า Design ต้องการ eBGP Multihop
จริง (Neighbor ไกลหลาย Hop โดยตั้งใจ) ให้ตั้ง `ttl-security hops <N>` ให้ตรงกับจำนวน Hop จริง
นั้นแทน `ebgp-multihop` (GTSM ครอบคลุมทั้งกรณี 1-hop และ multi-hop ในตัวอยู่แล้ว)

### ตรวจสอบผลลัพธ์

```
WAN-EDGE-1# show ip bgp neighbors 203.0.113.1 | include TTL
  External BGP neighbor may be up to 1 hops away.
  TTL Security, expected TTL: 255
```

---

## Step 459 — Verification & Troubleshooting

### 1) `show policy-map control-plane` — ดู Counter ต่อ Class

```
CORE-SW1# show policy-map control-plane

 Control Plane

  Service-policy input: COPP-POLICY

    Class-map: COPP-CRITICAL (match-any)
      184320 packets, 23412480 bytes
      5 minute offered rate 9000 bps, drop rate 0000 bps
      Match: access-group name COPP-ACL-ROUTING
        184320 packets, 23412480 bytes
        5 minute rate 9000 bps
      police:
          cir 10000000 bps, bc 512000 bytes
        conformed 184320 packets, 23412480 bytes; actions:
          transmit
        exceeded 0 packets, 0 bytes; actions:
          drop
        conformed 9000 bps, exceed 0000 bps

    Class-map: COPP-IMPORTANT (match-any)
      2140 packets, 342400 bytes
      5 minute offered rate 1000 bps, drop rate 0000 bps
      Match: access-group name COPP-ACL-MGMT
      police:
          cir 2000000 bps, bc 256000 bytes
        conformed 2140 packets, 342400 bytes; actions:
          transmit
        exceeded 0 packets, 0 bytes; actions:
          drop

    Class-map: COPP-NORMAL (match-any)
      850 packets, 71400 bytes
      police:
          cir 1000000 bps, bc 128000 bytes
        conformed 850 packets, 71400 bytes; actions:
          transmit
        exceeded 0 packets, 0 bytes; actions:
          drop

    Class-map: class-default (match-any)
      12 packets, 960 bytes
      police:
          cir 32000 bps, bc 8000 bytes
        conformed 12 packets, 960 bytes; actions:
          transmit
        exceeded 0 packets, 0 bytes; actions:
          drop
```

**สิ่งที่ต้องดูเป็นประจำ**: คอลัมน์ `exceeded ... packets` ของแต่ละ Class — ถ้า
`COPP-CRITICAL` มี `exceeded` เพิ่มขึ้นเรื่อยๆ = OSPF/BGP/HSRP ของจริงกำลังถูก Drop
(ต้องรีบสอบสวน — อาจต้องเพิ่ม `cir`/`bc` หรือกำลังถูกโจมตีจริง) ในขณะที่ `class-default`
มี `exceeded` มาก = มี Traffic แปลกปลอมจำนวนมากกำลังถูกกันไว้ตามที่ออกแบบ (ทำงานถูกต้อง)

### 2) `show ip traffic` — สถิติ Protocol ระดับอุปกรณ์ทั้งหมด

```
CORE-SW1# show ip traffic
IP statistics:
  Rcvd:  4502190 total, 4498012 local destination
         0 format errors, 0 checksum errors, 0 bad hop count
         12 unknown protocol, 0 not a gateway
  ICMP statistics:
    Rcvd: 850 echo, 0 echo reply, 0 unreachable
    Sent: 0 echo, 850 echo reply, 2 unreachable
  OSPF statistics:
    Rcvd: 184102 total
    Sent: 183998 total
  BGP statistics:
    Rcvd: 4520 total, Sent: 4488 total
```

ใช้เทียบกับ Baseline ปกติของ Network — ถ้า `ICMP Rcvd: echo` พุ่งสูงผิดปกติ (เช่นจากหลักสิบ
เป็นหลักหมื่นใน 5 นาที) เป็นสัญญาณของ ICMP Flood กำลังเกิดขึ้น (ตรงกับที่ CoPP `COPP-NORMAL`
ควรกำลังจำกัดอยู่ — ไปดู `exceeded` ของ Class นั้นควบคู่กัน)

### 3) `show control-plane host open-ports` — Protocol ไหนกำลัง Listen อยู่บ้าง

```
CORE-SW1# show control-plane host open-ports
Active internet connections (servers and established)
Prot   Local Address        Foreign Address      Service    State
tcp    *.22                 *.*                  SSH        LISTEN
tcp    *.23                 *.*                  Telnet     LISTEN
tcp    *.49                 10.10.99.60.*        TACACS     ESTABLISHED
udp    *.161                *.*                  SNMP       LISTEN
udp    *.123                *.*                  NTP        LISTEN
```

ใช้ตรวจสอบว่า **ไม่มี Service ที่ไม่ได้ตั้งใจเปิดอยู่** — ถ้าเห็น Port ที่ไม่คุ้นเคย (เช่น
Telnet ยัง Listen ทั้งที่ Hardening Checklist Part 22 บอกให้ปิด) ให้กลับไปตรวจสอบ Config
ทันที (`no ip http server` ไม่ครอบคลุม Telnet — ต้องปิดที่ `line vty` แยก)

### 4) จำลอง CPU Spike และดู CoPP ทำงานจริง

```
! บนเครื่องทดสอบ (จำลอง Attacker) ยิง ICMP Flood ไปที่ CORE-SW1
$ ping 10.10.99.10 -f -s 1400 -c 100000        ! Linux flood ping (ต้องใช้ root)

! ระหว่างนั้น สังเกตที่ CORE-SW1
CORE-SW1# show processes cpu sorted | exclude 0.00%  0.00%   0.00%
CPU utilization for five seconds: 8%/3%; one minute: 6%; five minutes: 5%
   PID Runtime(ms)     Invoked      uSecs   5Sec   1Min   5Min TTY Process
   142       48210       98212        491   3.20%  2.80%  2.10%   0 IP Input

CORE-SW1# show policy-map control-plane class COPP-NORMAL
    Class-map: COPP-NORMAL (match-any)
      98812 packets, 138400000 bytes
      5 minute offered rate 980000 bps, drop rate 850000 bps
      police:
          cir 1000000 bps, bc 128000 bytes
        conformed 12400 packets, 17360000 bytes; actions:
          transmit
        exceeded 86412 packets, 121040000 bytes; actions:
          drop
```

ผลลัพธ์ที่ควรเห็น: **CPU ยังอยู่ในระดับต่ำ (5-8%) แม้มี ICMP Flood จำนวนมหาศาลเข้ามา** เพราะ
CoPP `COPP-NORMAL` Drop Packet ส่วนเกิน (`exceeded 86412 packets`) **ก่อน**ที่จะไปกิน CPU
Cycle เต็มที่ — ในขณะเดียวกัน `show ip ospf neighbor` และ `show ip bgp summary` ควรยังแสดง
State `FULL`/`Established` ตามปกติ **ไม่ได้รับผลกระทบเลย** เพราะอยู่ Class `COPP-CRITICAL`
ที่แยกออกจาก `COPP-NORMAL` โดยสมบูรณ์

### 5) Diagnose: CoPP เข้มงวดเกินไปจนกระทบ OSPF Hello จริง

**อาการที่พบ**: OSPF Neighbor บน CORE-SW1 หลุดเป็นพักๆ (Flap) โดยไม่มีปัญหา Physical Link
หรือ MTU

```
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11          1   INIT/  -        00:00:32    10.255.10.2     TenGigabitEthernet1/0/1
!            ↑ ค้างที่ INIT ไม่ขึ้น FULL — สัญญาณว่า Hello ไปไม่ครบ หรือ Hello ขากลับหาย
```

**ขั้นตอนตรวจสอบ**:

```
! ขั้น 1: ดูว่า COPP-CRITICAL มี exceeded/drop เพิ่มขึ้นหรือไม่
CORE-SW1# show policy-map control-plane class COPP-CRITICAL
    ...
        exceeded 4820 packets, 612000 bytes; actions:
          drop
!            ↑ พบ! Class ที่ควรรับประกัน OSPF กลับมี Drop จริง

! ขั้น 2: ตรวจสอบว่า Traffic OSPF จริงถูกจัดเข้า COPP-CRITICAL หรือหลุดไป class-default
CORE-SW1# show access-list COPP-ACL-ROUTING
Extended IP access list COPP-ACL-ROUTING
    10 permit ospf any host 224.0.0.5 (612000 matches)
    20 permit ospf any host 224.0.0.6 (0 matches)
    ...
!            ↑ ปกติดี ACL Match ถูกต้อง แต่ Rate เกินเพดานจริงในช่วงที่มี Link Flap หลายเส้น
!              พร้อมกัน (LSA Flooding พุ่งสูงชั่วขณะ) — CIR ที่ตั้งไว้ต่ำเกินไปสำหรับขนาด
!              Topology จริง ไม่ใช่ปัญหาจาก ACL/Class-map ผิด
```

**วิธีแก้**: เพิ่ม `cir`/`bc` ของ `COPP-CRITICAL` ให้สูงขึ้น (เช่นจาก 10 Mbps เป็น 20 Mbps)
หรือถ้าพบว่า `matches` ของบรรทัด ACL ที่ควร Match กลับเป็น 0 ทั้งที่มี OSPF วิ่งจริง (คนละ
อาการ) แสดงว่า **ACL เขียนผิด** (เช่น พิมพ์ Multicast Group ผิด, ลืม `permit ospf any any`
สำหรับ Virtual-link) ทำให้ Hello หลุดไปเข้า `class-default` ที่มี Rate ต่ำมาก (32 kbps)
แทน — ต้องแก้ที่ ACL ไม่ใช่ที่ Rate ในกรณีนี้

```
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)# class COPP-CRITICAL
CORE-SW1(config-pmap-c)# police cir 20000000 bc 1024000 conform-action transmit exceed-action drop
CORE-SW1(config-pmap-c)# end
```

> **บทเรียนสำคัญ**: **อย่าตั้ง CoPP แล้วไม่ Monitor ต่อ** — Traffic Pattern ของเครือข่ายจริง
> เปลี่ยนไปตามการเติบโต (จำนวน Router เพิ่มขึ้น = LSA/Update มากขึ้น) Rate ที่เหมาะสมวันนี้
> อาจเข้มงวดเกินไปในอีก 1-2 ปีข้างหน้า ต้องตรวจสอบ `show policy-map control-plane` เป็นระยะ
> (ควรผูกกับ SNMP/Syslog Alert เมื่อ `exceeded` ของ `COPP-CRITICAL` เพิ่มขึ้นผิดปกติ)

---

## Step 460 — Lab เต็มรูปแบบ: CoPP + MPP + uRPF + BGP TTL-Security ทั้ง Topology

### ภาพรวม Lab

```
                         INTERNET
                            │
              ┌─────────────┴─────────────┐
              │           ISP-RTR           │  AS 65000
              └──────┬───────────────┬──────┘
      203.0.113.1/30 │               │ 203.0.113.5/30
       (GTSM hops=1)  │               │  (GTSM hops=1)
      203.0.113.2/30  │               │ 203.0.113.6/30
              ┌───────┴──────┐ ┌──────┴───────┐
              │  WAN-EDGE-1   │ │  WAN-EDGE-2   │  AS 65001 (iBGP Mesh — Part 30)
              │  Gi0/0/0: uRPF strict            │  Gi0/0/0: uRPF loose+allow-default
              │  MPP: allow ssh/snmp on Gi0/0/1  │  (Asymmetric จาก Cross-link)
              │  CoPP: COPP-POLICY               │  CoPP: COPP-POLICY
              └───────┬──────┘ └──────┬───────┘
       10.10.254.0/30 │                │ 10.10.254.4/30
              ┌───────┴──────┐ ┌──────┴───────┐
              │   CORE-SW1    │ │   CORE-SW2    │  OSPF Area 0 + HSRP (Part 18)
              │   CoPP: COPP-POLICY               │  CoPP: COPP-POLICY
              │   Vlan99 .10  │ │   Vlan99 .11  │
              └───────────────┘ └───────────────┘
```

### Full Running-Config — CORE-SW1 (ส่วนที่เพิ่มจาก Part นี้ — ต่อยอด Part 22)

```
! ============================================================
!  CONTROL PLANE POLICING & INFRASTRUCTURE SECURITY — CORE-SW1
!  ต่อยอดจาก AAA/Hardening (Part 22) — ห้ามลบ Config เดิม
! ============================================================
configure terminal

! --- ACL สำหรับ CoPP Classification (Step 453) ---
ip access-list extended COPP-ACL-ROUTING
 remark ** OSPF + EIGRP + BGP + HSRP — Control Plane วิกฤต **
 permit ospf any host 224.0.0.5
 permit ospf any host 224.0.0.6
 permit ospf any any
 permit eigrp any host 224.0.0.10
 permit eigrp any any
 permit tcp any eq 179 any
 permit tcp any any eq 179
 permit udp any host 224.0.0.2 eq 1985
 permit udp any host 224.0.0.102 eq 1985
 exit
!
ip access-list extended COPP-ACL-MGMT
 remark ** Management Plane — เฉพาะจาก MGMT VLAN 99 **
 permit tcp 10.10.99.0 0.0.0.255 any eq 22
 permit tcp 10.10.99.0 0.0.0.255 any eq 49
 permit udp 10.10.99.0 0.0.0.255 any eq 161
 permit udp any eq 162 10.10.99.0 0.0.0.255
 permit udp 10.10.99.0 0.0.0.255 any eq 123
 exit
!
ip access-list extended COPP-ACL-ICMP
 permit icmp any any echo
 permit icmp any any echo-reply
 permit icmp any any ttl-exceeded
 permit icmp any any unreachable
 permit icmp any any port-unreachable
 exit

! --- Class-map (Step 453) ---
class-map match-any COPP-CRITICAL
 match access-group name COPP-ACL-ROUTING
 exit
class-map match-any COPP-IMPORTANT
 match access-group name COPP-ACL-MGMT
 exit
class-map match-any COPP-NORMAL
 match access-group name COPP-ACL-ICMP
 exit

! --- Policy-map (Step 454) ---
policy-map COPP-POLICY
 class COPP-CRITICAL
  police cir 20000000 bc 1024000 conform-action transmit exceed-action drop
  exit
 class COPP-IMPORTANT
  police cir 2000000 bc 256000 conform-action transmit exceed-action drop
  exit
 class COPP-NORMAL
  police cir 1000000 bc 128000 conform-action transmit exceed-action drop
  exit
 class class-default
  police cir 32000 bc 8000 conform-action transmit exceed-action drop
  exit
 exit

! --- Apply เข้า Control-Plane (Step 452, 454) ---
control-plane
 service-policy input COPP-POLICY
 exit

end
write memory
```

> **หมายเหตุ CORE-SW1/CORE-SW2**: MPP (`management-interface`) และ uRPF ไม่ Apply บน
> CORE-SW1/CORE-SW2 ในตัวอย่างนี้ เพราะทั้งสองเป็น Internal Core Switch ไม่มี Interface
> เชื่อมตรงกับ Internet — ป้องกันด้วย VTY ACL (Part 22) + CoPP (Part นี้) เพียงพอสำหรับ
> ตำแหน่งนี้ใน Topology (MPP/uRPF มีประโยชน์สูงสุดที่ WAN Edge ซึ่งเป็น Boundary กับ
> Network ที่ไม่น่าเชื่อถือ)

### Full Running-Config — WAN-EDGE-1 (ส่วนที่เพิ่มจาก Part นี้ — ต่อยอด Part 15, 22, 29-30)

ACL/Class-map/Policy-map (Step 453-454) ใช้ **Pattern เดียวกับ CORE-SW1** ทั้งชื่อ ACL
(`COPP-ACL-ROUTING`/`-MGMT`/`-ICMP`) และชื่อ Class/Policy-map (`COPP-CRITICAL/IMPORTANT/
NORMAL` + `COPP-POLICY`) — ต่างกันแค่ 2 จุด: (1) ACL `COPP-ACL-ROUTING` ของ WAN-EDGE ไม่ต้องมี
`permit eigrp`/HSRP เพราะไม่ได้รัน EIGRP/HSRP ที่ตำแหน่งนี้ (2) `police cir` ของ
`COPP-CRITICAL` ใช้ **10 Mbps** (ต่ำกว่า Core เพราะจำนวน OSPF/BGP Neighbor น้อยกว่า) — สิ่งที่
**เพิ่มใหม่เฉพาะที่ WAN Edge** คือ MPP, uRPF, และ BGP TTL-Security:

```
! ============================================================
!  CONTROL PLANE POLICING & INFRASTRUCTURE SECURITY — WAN-EDGE-1
!  (ACL/Class-map/Policy-map/control-plane: Pattern เดียวกับ CORE-SW1 ด้านบน)
! ============================================================
configure terminal

! --- MPP: จำกัด SSH/SNMP ให้เฉพาะทาง LAN-facing Interface (Step 456) ---
control-plane host
 management-interface GigabitEthernet0/0/1 allow ssh snmp
 exit

! --- uRPF Strict Mode บน Internet-facing Interface (Step 457) ---
interface GigabitEthernet0/0/0
 ip verify unicast source reachable-via rx
 exit

! --- BGP TTL-Security / GTSM (Step 458) ---
router bgp 65001
 neighbor 203.0.113.1 ttl-security hops 1
 exit

end
write memory
```

### Full Running-Config — WAN-EDGE-2 (ต่างจาก WAN-EDGE-1 เฉพาะ uRPF Mode)

Pattern เดียวกันกับ WAN-EDGE-1 ทั้งหมด **ยกเว้น** uRPF ที่ต้องใช้ **Loose Mode** ตามที่
อธิบายใน Step 457 (Asymmetric Routing จาก Cross-link):

```
interface GigabitEthernet0/0/0
 ip verify unicast source reachable-via any allow-default
 exit
!
router bgp 65001
 neighbor 203.0.113.5 ttl-security hops 1
 exit
```

### Verification สุดท้าย: ยิง ICMP Flood จำลอง แล้วยืนยันว่า OSPF/BGP ไม่กระทบ

```
! ก่อนยิง Flood — Baseline
CORE-SW1# show ip ospf neighbor | include FULL
1.1.1.11          1   FULL/  -        00:00:38    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          1   FULL/  -        00:00:33    10.255.20.2     TenGigabitEthernet1/0/2

WAN-EDGE-1# show ip bgp summary | include 203.0.113.1
203.0.113.1     4 65000    18402   18391        5    0    0 04:12:08        3

! --- ระหว่างยิง ICMP Flood จากเครื่องทดสอบไปที่ Vlan99 SVI ของ CORE-SW1 ---
$ hping3 --flood --icmp 10.10.99.10

! หลังยิง Flood ต่อเนื่อง 60 วินาที — ตรวจสอบผลลัพธ์
CORE-SW1# show policy-map control-plane class COPP-NORMAL
    Class-map: COPP-NORMAL (match-any)
      612480 packets, 857472000 bytes
      5 minute offered rate 11400000 bps, drop rate 10380000 bps
      police:
          cir 1000000 bps, bc 128000 bytes
        conformed 53420 packets, 74788000 bytes; actions:
          transmit
        exceeded 559060 packets, 782684000 bytes; actions:
          drop

! OSPF Neighbor ยังคง FULL ทั้งคู่ — ไม่ได้รับผลกระทบจาก ICMP Flood เลย
CORE-SW1# show ip ospf neighbor | include FULL
1.1.1.11          1   FULL/  -        00:00:34    10.255.10.2     TenGigabitEthernet1/0/1
1.1.1.12          1   FULL/  -        00:00:31    10.255.20.2     TenGigabitEthernet1/0/2

! BGP Session ที่ WAN-EDGE-1 ยังคง Established เช่นกัน (สังเกต Uptime ไม่ Reset)
WAN-EDGE-1# show ip bgp summary | include 203.0.113.1
203.0.113.1     4 65000    18512   18498        5    0    0 04:13:09        3

! CPU ของ CORE-SW1 ยังอยู่ในระดับปกติตลอดการทดสอบ
CORE-SW1# show processes cpu | include five seconds
CPU utilization for five seconds: 11%/6%; one minute: 9%; five minutes: 7%
```

**สรุปผลการทดสอบ**: แม้จะยิง ICMP Flood ด้วย Rate สูงกว่าเพดานของ `COPP-NORMAL` มากกว่า
10 เท่า (Offered rate 11.4 Mbps เทียบกับ CIR 1 Mbps) — CoPP Drop ส่วนเกินได้ทันที
(`exceeded 559060 packets`) ทำให้ **CPU ไม่พุ่งเกินระดับปกติเลย (11%)** และที่สำคัญที่สุด
**OSPF Neighbor และ BGP Session ไม่ได้รับผลกระทบแม้แต่วินาทีเดียว** เพราะอยู่ Class
`COPP-CRITICAL` ที่ถูกแยก Policer ออกจาก `COPP-NORMAL` โดยสมบูรณ์ตามที่ออกแบบไว้ — นี่คือ
ผลลัพธ์ที่ต้องการจาก Security-in-Depth ครบทั้ง 3 Plane (Data Plane จาก Part 43-45,
Control Plane จาก Part นี้)

---

## แบบฝึกหัดทวนความเข้าใจ Part 46

1. Data Plane, Control Plane, และ Management Plane ของ Router/Switch ต่างกันอย่างไร และทำไม
   ACL/ZBFW ที่ Apply บน Physical Interface (Data Plane) เพียงอย่างเดียวไม่สามารถปกป้อง CPU
   ของอุปกรณ์จาก Control Plane DoS ได้อย่างสมบูรณ์?

2. CoPP ใช้ MQC Model เดียวกับ QoS ปกติใน Part 40 แต่ต่างกันที่จุดใดบ้าง — และเพราะเหตุใด
   ACL แบบ `ip access-list` จึงไม่สามารถใช้ Classify ARP Traffic สำหรับ CoPP ได้?

3. อธิบายความแตกต่างระหว่าง uRPF Strict Mode กับ Loose Mode และยกตัวอย่างสถานการณ์ในหลักสูตร
   นี้ที่ WAN-EDGE-1 ใช้ Strict Mode ได้ แต่ WAN-EDGE-2 ต้องใช้ Loose Mode พร้อม `allow-default`

4. BGP TTL Security (GTSM) ป้องกัน Attack ประเภทใด และทำไมจึงใช้ร่วมกับ `ebgp-multihop`
   ไม่ได้ในเวลาเดียวกัน?

5. ถ้าพบว่า OSPF Neighbor บน CORE-SW1 Flap เป็นพักๆ หลัง Deploy CoPP และตรวจสอบ
   `show policy-map control-plane class COPP-CRITICAL` พบว่ามี `exceeded` เพิ่มขึ้นต่อเนื่อง
   ในขณะที่ `show access-list COPP-ACL-ROUTING` แสดง `matches` เพิ่มขึ้นตามปกติ (ไม่ใช่ 0)
   ปัญหาน่าจะมาจากอะไร และควรแก้อย่างไร?

### เฉลย

1. **Data Plane** คือ Traffic ของ User ที่ Forward ผ่านอุปกรณ์ (Transit) ประมวลผลด้วย Hardware
   (ASIC/CEF) ไม่แตะ CPU **Control Plane** คือ Process ภายในที่ทำให้อุปกรณ์ "รู้ทาง" เช่น
   Routing Protocol, STP, ARP ประมวลผลที่ CPU **Management Plane** คือช่องทางบริหารจัดการ
   ตัวอุปกรณ์เอง (SSH/SNMP) ก็ประมวลผลที่ CPU เช่นกัน — ACL/ZBFW บน Data Plane กรองได้แค่
   "Permit/Deny" ไม่มี Concept ของการจำกัด "อัตรา (Rate)" ของ Traffic ที่ถูกอนุญาตแล้วว่าจะ
   วิ่งเข้า CPU ได้เร็วแค่ไหน — Attacker ที่ปลอม Traffic ให้ตรงตาม Policy "Permit" ทุกอย่าง
   (เช่น ปลอมเป็น OSPF Hello ที่ถูกต้องตาม Protocol) จะยังผ่าน ACL/ZBFW ได้และไปโอเวอร์โหลด
   CPU ได้อยู่ดี ต้องมี CoPP ที่ทำงานเฉพาะที่ Punt Path เข้า CPU มาเสริม

2. CoPP ต่างจาก QoS ปกติที่ (1) Apply `service-policy` บน `control-plane` (Virtual Interface
   แทน CPU) ไม่ใช่ Physical Interface (2) ควบคุม Traffic ที่ Punt ขึ้น CPU **รวมจากทุก
   Interface** ของอุปกรณ์พร้อมกัน (Aggregate) ไม่ใช่แยกต่อ Interface (3) ใช้ Action หลักคือ
   `police` เท่านั้น ไม่มี Queue/Shape เพราะ CPU ไม่ใช่ Physical Wire ที่มี Bandwidth ให้ Shape
   ส่วน ARP ใช้ EtherType `0x0806` ไม่มี IP Header ทำให้ `ip access-list` ที่ตรวจ IP
   Header ไม่สามารถ Match ARP ได้เลย ต้องป้องกัน ARP Flood ด้วย Port Security/DAI จาก
   Part 21 แทน

3. **Strict Mode** (`reachable-via rx`) ต้อง Return Path ออกทาง Interface เดียวกับที่
   Packet เข้ามาเท่านั้น เหมาะกับ Link ที่ Traffic เข้า-ออกทางเดียวแน่นอน **Loose Mode**
   (`reachable-via any`) แค่ต้องมี Route ไปหา Source ใน FIB (Interface ใดก็ได้) เหมาะกับ
   Asymmetric Routing — WAN-EDGE-1 ใช้ Strict ได้เพราะเป็น Point-to-Point เดียวไปยัง
   ISP-RTR โดยไม่มี Path อื่น ส่วน WAN-EDGE-2 ต้องใช้ Loose + `allow-default` เพราะมี
   Cross-link ไปยัง WAN-EDGE-1 ทำให้ Traffic ขาเข้า/ขาออกอาจไม่สมมาตรตามการออกแบบ Redundancy
   ของ Dual-homed Internet Edge (Part 29-30)

4. GTSM ป้องกัน **Off-path Spoofing Attack** ที่ Attacker ปลอม Source IP เป็น BGP Neighbor
   ที่ถูกต้องแต่อยู่ไกลออกไปหลาย Hop (ไม่ได้อยู่บน Link เดียวกันจริง) — ใช้ค่า TTL เป็นเครื่อง
   พิสูจน์ระยะทางทางกายภาพ เพราะ Router คั่นกลางแต่ละตัวจะ Decrement TTL จริงเสมอแม้ Attacker
   จะปลอม Source IP ได้ ใช้ร่วมกับ `ebgp-multihop` ไม่ได้เพราะทั้งสองคำสั่งขัดแย้งกันโดยตรง —
   `ebgp-multihop` ผ่อนคลายให้ Session ทำงานได้แม้ TTL ต่ำ (สำหรับ Neighbor ไกลหลาย Hop
   โดยชอบธรรม) ส่วน `ttl-security` เข้มงวดให้ต้อง TTL สูงเท่านั้น

5. ปัญหาไม่ได้มาจาก ACL ผิด (เพราะ `matches` เพิ่มขึ้นตามปกติ แสดงว่า Classification ถูกต้อง)
   แต่มาจาก **CIR/Bc ของ Class `COPP-CRITICAL` ต่ำเกินไปสำหรับ Volume จริงของ Routing
   Protocol Traffic ในช่วงที่มี Event ผิดปกติ** (เช่น LSA Flooding ตอน Link Flap หลายเส้น
   พร้อมกัน) วิธีแก้คือเพิ่มค่า `cir`/`bc` ของ Class นี้ให้สูงขึ้นให้เพียงพอกับ Burst สูงสุดที่
   เป็นไปได้จริงของ Topology (ไม่ใช่แก้ ACL เพราะ ACL ทำงานถูกต้องอยู่แล้ว)

---

## สรุป Part 46

Part นี้ทำตามสัญญาที่ให้ไว้ใน [Part 22 Step 219](part-022-aaa-device-hardening.md) อย่าง
เต็มรูปแบบ: อธิบาย **โมเดล 3-Plane** (Data/Control/Management) และเหตุผลเชิงลึกว่าทำไม
Data Plane Security (ACL/ZBFW จาก Part 43-45) เพียงอย่างเดียวไม่พอที่จะปกป้อง Control Plane,
**CoPP** ด้วย MQC เดียวกับ Part 40 (Classify Routing Protocol/Management/ICMP/Unknown แล้ว
`police` แยก Rate ตามความสำคัญ), **CoPPr** สำหรับความเข้าใจเชิงลึกขึ้นไปอีกระดับ (Host/
Transit/CEF-Exception), **MPP** ที่จำกัด Management Protocol ให้ Listen เฉพาะ Interface
ที่กำหนด (เสริม VTY ACL จาก Part 22), **uRPF** ป้องกัน Source IP Spoofing ทั้ง Strict/Loose
Mode, และ **BGP TTL-Security (GTSM)** ป้องกัน BGP Session จาก Off-path Spoofing — ปิดท้าย
ด้วย Lab เต็มรูปแบบที่ Deploy ทุกกลไกพร้อมกันบน CORE-SW1/CORE-SW2/WAN-EDGE-1/WAN-EDGE-2
และพิสูจน์ผลจริงด้วยการจำลอง ICMP Flood ที่ CoPP Rate-Limit ได้สำเร็จโดย OSPF/BGP ไม่ได้รับ
ผลกระทบเลย

ตอนนี้เครือข่ายของเราได้รับการป้องกันครบทั้ง **3 Plane**: Data Plane (ACL/ZBFW, Part 43-45),
Control Plane (CoPP/CoPPr, Part นี้), และ Management Plane (AAA/VTY ACL จาก Part 22 +
MPP จาก Part นี้) — นี่คือรากฐาน **Infrastructure Security** ระดับ Enterprise ที่สมบูรณ์
ตามมาตรฐาน CCNP ENCOR

✅ **พร้อมสำหรับ Part 47**: เราจะเปลี่ยนหัวข้อไปที่ **Automation ด้วย Python ระดับสูง**
(ต่อยอดจาก Automation Basics ใน [Part 23](part-023-automation-basics.md)) — ใช้ Netmiko/
NAPALM จัดการอุปกรณ์หลายตัวพร้อมกัน, Parse Output ด้วย TextFSM, และเขียน Script ที่ Deploy
Config (รวมถึง CoPP Policy จาก Part นี้) ให้ทั้ง Topology โดยอัตโนมัติแทนการพิมพ์ CLI
ทีละอุปกรณ์แบบที่ทำมาตลอด 46 Part

**ไปต่อ:** [Part 47 — Automation with Python (Advanced) →](part-047-automation-python-advanced.md)
