# Part 75 — Advanced Multicast for Enterprise (CCIE-level)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 741–750 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 74 — Advanced BGP WAN Edge](part-074-advanced-bgp-wan-edge.md) ที่ปิดท้ายเรื่อง
> WAN Edge Routing Policy ระดับ CCIE Part นี้วกกลับมาที่ **Multicast** ที่ [Part 41](part-041-multicast-fundamentals.md)
> (IGMP, PIM-DM/SM, RPF) และ [Part 42](part-042-multicast-advanced.md) (Auto-RP, BSR, Anycast RP +
> MSDP, SSM, Bidir-PIM) ปูพื้นฐานไว้ครบ และ [Part 65](part-065-multicast-troubleshooting.md) ฝึก
> Troubleshoot จุดพังที่พบบ่อยไปแล้ว — Design ทั้งหมดที่เรียนมาใช้งานได้ดีบน Campus เดียว (CORE-SW1/
> CORE-SW2 + DIST-SW1/3/4) ที่มี Group อยู่ไม่กี่ตัว (`239.1.1.1` All-Hands, `232.50.1.1` Market-Data
> SSM) แต่คำถามระดับ **CCIE Enterprise Infrastructure** ไม่ใช่ "PIM-SM ทำงานอย่างไร" อีกต่อไป —
> คำถามคือ **"ถ้า Campus นี้ขยายเป็น 10 อาคารตาม [Part 72](part-072-campus-fabric-scale-design.md)
> พร้อม Video/Signage/Training Feed หลายร้อย Group จะเกิดอะไรขึ้นกับ (S,G) State, RP, WAN Link,
> DMVPN Spoke และ QoS Queue"** — Part นี้จะตอบคำถามนั้นให้ครบทุกมุม ตั้งแต่ Scaling Theory,
> MVPN Concept, PIM Design Pattern ระดับ Decision Matrix, Boundary Filtering, QoS Integration,
> Multicast over DMVPN, IGMP/MLD Scaling, High Availability ระดับ Multi-site และปิดท้ายด้วย Lab
> Design เต็มรูปแบบของทั้ง Enterprise

## Use Case ที่จะขับเคลื่อน Part นี้: "จาก 1 Campus สู่ Enterprise ระดับ CCIE"

ทีม Network ของบริษัทเพิ่งได้รับอนุมัติ Budget ขยาย Campus HQ จาก 4 Distribution Block (DIST-SW1,
DIST-SW2, DIST-SW3, DIST-SW4 เดิม) ไปเป็น **10 อาคาร** ตามแผน [Part 72](part-072-campus-fabric-scale-design.md)
พร้อมกันนั้นฝ่าย Corporate Communications ก็ต้องการเพิ่ม Use Case ใหม่ที่ใช้ Multicast ทุกอาคาร:

```
Building 1-4 (เดิม)                         Building 5-10 (ใหม่ตาม Part 72)
  - All-Hands Video    239.1.1.1               - Lobby Digital Signage ต่ออาคาร  239.50.<bldg>.1
  - Market-Data (SSM)  232.50.1.1               - Training Room Video Wall ต่อห้อง 239.60.<bldg>.<room>
                                                 - Security Camera Multicast Feed   239.70.<bldg>.<cam>

รวมทั้ง Campus (ประมาณการ): 250+ Active Group, 3,000+ Receiver Port, Sender กระจายทั้ง 10 อาคาร
```

นอกจากนี้บริษัทยังเพิ่ง Acquire **Regional Data Center** (เชื่อมผ่าน MPLS/DMVPN Core ที่จะเรียนเต็ม
ใน [Part 86-87](part-086-mpls-l3vpn-fundamentals.md)) ที่ต้องรับ All-Hands Video เดียวกันข้าม Site
และมี **Branch Office** ที่เชื่อมผ่าน DMVPN ([Part 52-53](part-052-dmvpn-fundamentals.md)) ที่ต้องรับ
Training Video บางรายการด้วย — Design เดิมจาก Part 41/42 (Static/Anycast RP เดี่ยว, ไม่มี Boundary,
ไม่มี QoS Marking เฉพาะ Multicast) **จะไม่ Scale และไม่ปลอดภัยพอ** สำหรับสถานการณ์นี้เลย — นี่คือ
ช่องว่างที่ Part นี้จะปิดให้ครบ

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 741 | Multicast Scaling ระดับ Enterprise — หลักร้อย Group, หลักพัน Receiver และปัญหา (S,G) State Scaling |
| 742 | Multicast VPN (MVPN) — Default MDT / Data MDT เมื่อ Multicast ต้องข้าม VRF-Lite/MPLS VPN Core |
| 743 | PIM Design Pattern ที่ Scale ได้ — Static RP vs Anycast RP vs BSR พร้อม Decision Matrix ระดับ CCIE |
| 744 | Multicast Boundary/Filtering Design — `ip multicast boundary` ควบคุม Group ข้าม Administrative Boundary |
| 745 | Multicast + QoS Integration — Marking/Queuing วิดีโอ Multicast Bandwidth สูงไม่ให้แย่ง/โดน Drop Traffic อื่น |
| 746 | Multicast over DMVPN — NBMA Challenge, `ip pim nbma-mode`, RP Reachability ข้าม Spoke |
| 747 | IGMP/MLD Scaling — IGMPv3/MLDv2 Source Filtering และ `ip igmp limit` ป้องกัน Router ล้น |
| 748 | Multicast High Availability — Anycast RP Failure ระดับ Scale และ MSDP Mesh-group ป้องกัน SA Loop |
| 749 | Multicast Troubleshooting ระดับ Scale — `show ip mroute summary`/`count`, Filter ตาม Group Range |
| 750 | Lab เต็มรูปแบบ: Enterprise Multicast Design ทั้ง Campus 10 อาคาร + Regional DC + Branch |

---

## Step 741 — Multicast Scaling ระดับ Enterprise

### 741.1 ทบทวน: (S,G)/(*,G) State คืออะไร (จาก [Part 41 Step 408](part-041-multicast-fundamentals.md#step-408--multicast-forwarding-และ-rpf-check-sg-และ-g))

ทุก Group ที่มี Receiver อยู่จริงบน Router ตัวหนึ่ง จะกิน **State Entry อย่างน้อย 1 รายการ** ใน
Multicast Routing Table เสมอ — และถ้า Group นั้น Switchover ไป SPT แล้ว (Default Behavior ของ
IOS ตาม [Part 41 Step 406](part-041-multicast-fundamentals.md#step-406--pim-sparse-mode-pim-sm-explicit-join-model))
จะมี **ทั้ง `(*,G)` และ `(S,G)` แยกกัน** ต่อ 1 Source ต่อ 1 Group — Router ที่อยู่ตรงกลาง Topology
(เช่น CORE-SW1/CORE-SW2) จะเห็น State ของ **ทุก Group ที่มี Receiver อยู่ที่ใดก็ได้ใน Campus** ไม่ใช่
แค่ Group ที่ตัวเองสนใจ เพราะ Traffic ต้อง Transit ผ่านมันเสมอ

```
สูตรประมาณ (S,G)+(*,G) State บน Core Router ของ Campus

จำนวน State โดยประมาณ  ≈  (จำนวน Active Group ทั้ง Campus)
                          × (1 + จำนวน Active Source เฉลี่ยต่อ Group ที่ Switchover ไป SPT แล้ว)

ตัวอย่าง Campus เดิม (Part 41/42):     2 Group × (1+1)  =   4 State   (เล็กมาก ไม่มีปัญหา)
ตัวอย่าง Campus 10 อาคาร (Part นี้):  250 Group × (1+1) = 500 State  (ต้องคิดเรื่อง Scale จริงจัง)
```

### 741.2 ที่มาของ 250+ Group — ประมาณการจาก Use Case ของ Part นี้

| ประเภท Group | จำนวนต่ออาคาร | 10 อาคาร | รวม |
|---|---|---|---|
| All-Hands / Corporate (ข้าม Site ทั้งหมด) | ใช้ร่วมกัน 1 Group | - | 1 |
| Lobby Digital Signage (`239.50.x.x`) | 1 Group/อาคาร | ×10 | 10 |
| Training Room Video Wall (`239.60.x.x`) | เฉลี่ย 4 ห้อง/อาคาร | ×10 | 40 |
| Security Camera Feed (`239.70.x.x`) | เฉลี่ย 20 กล้อง/อาคาร | ×10 | 200 |
| Market-Data SSM (`232.50.1.1`) | ใช้ร่วมกัน 1 Group | - | 1 |
| **รวมประมาณ** | | | **≈ 252 Group** |

Receiver ต่อ Group ก็ต่างกันมาก — All-Hands อาจมี Receiver หลักร้อยถึงหลักพัน (ทุก Access Switch
ทั้ง 10 อาคาร) แต่ Security Camera Feed 1 กล้องอาจมี Receiver เดียว (NVR Server ตัวเดียวที่ดูภาพ)
— **จำนวน Receiver ไม่ได้เพิ่ม State บน Core Router โดยตรง** (OIL ยาวขึ้นใน Entry เดิม ไม่ใช่ Entry
ใหม่) แต่ **จำนวน Group ที่ต่างกันคือตัวเพิ่ม State จริง** — นี่คือจุดที่ CCIE-level Design ต้องแยกให้
ถูก: **Scaling ปัญหาคือ "จำนวน Group × Source" ไม่ใช่ "จำนวน Receiver"**

### 741.3 ผลกระทบของ (S,G) State ที่มากขึ้นบน Hardware จริง

| ทรัพยากรที่ถูกใช้ | ผลกระทบเมื่อ State เพิ่มจากหลักสิบเป็นหลักร้อย |
|---|---|
| **Memory (Control-Plane)** | แต่ละ mroute Entry ใช้ Memory หลักร้อย Byte (Metadata + Timer + OIL List) — 500 Entry ยังเล็กเทียบกับ RAM ของ Catalyst 9500/9300 สมัยใหม่ (หลัก GB) แต่ถ้าโต **ไม่มีขอบเขต** (เช่น ไม่มี `ip igmp limit` ตาม Step 747) จะกลายเป็นปัญหาจริงได้ |
| **Hardware Forwarding Table (TCAM/ASIC)** | Catalyst 9300/9500 มี Multicast Forwarding Table ขนาดจำกัดต่อ ASIC (มีเพดานตายตัวตาม Platform Datasheet) — ถ้า State เกินเพดาน Router จะ **Software-switch Multicast บางส่วน** (Punt ไป CPU) ซึ่งช้ากว่า Hardware Forwarding มาก และเสี่ยง CPU พุ่งถ้า Traffic Rate สูง |
| **PIM Join/Prune Processing** | ทุก (S,G)/(*,G) มี Join/Prune Message เป็นระยะ (ทุก 60s ตาม Default) — 500 Entry คูณด้วยจำนวน PIM Neighbor ที่ต้องส่ง/รับ Message นี้ทุกทาง คือ CPU Load ที่สะสมได้จริงถ้า Design ไม่ดี |
| **RP Register/Join Processing** | RP ตัวเดียว (Static RP แบบ Part 41) ต้อง Process PIM Register จาก **ทุก First-Hop Router ของทุก Source ใน 10 อาคาร** พร้อมกัน — นี่คือเหตุผลที่ Step 743 จะเสนอ Anycast RP เป็น Default choice สำหรับ Scale ระดับนี้ |

### 741.4 กฎการออกแบบสำหรับ CCIE-level Scale

```
กฎที่ 1: แยก RP Load ด้วย Anycast RP เสมอเมื่อจำนวน Group เกิน ~50 Group หรือมี Source
         กระจายมากกว่า 1 อาคาร (ดู Step 743 สำหรับ Decision Matrix เต็มรูปแบบ)

กฎที่ 2: ใช้ SPT-Threshold อย่างมีสติ — Group ที่มี Rate สูงและ Receiver น้อย (เช่น Security
         Camera 1 ตัวต่อ 1 NVR) ควร Switchover ไป SPT ทันที (Default) เพราะ (S,G) State เดียว
         ไม่กระทบ Scale เท่า Shared Tree ที่ต้อง Fan-out กว้าง — แต่ถ้า Design ต้องการจำกัด
         (S,G) State ให้น้อยที่สุดจริงๆ (เช่น Core Router รุ่นเก่าที่ TCAM เล็ก) ให้พิจารณา
         `ip pim spt-threshold infinity` เฉพาะ Group ที่ไม่ Sensitive ต่อ Latency ผ่าน Group-list ACL

กฎที่ 3: จำกัดจำนวน (*,G)/(S,G) ต่อ Interface ด้วย IGMP Limit (Step 747) เพื่อป้องกัน Endpoint
         ที่ Misconfigured หรือถูกโจมตี Join Group จำนวนมากเกินจริง

กฎที่ 4: Filter Group ที่ไม่จำเป็นต้องข้ามอาคาร/ข้าม Site ด้วย Multicast Boundary (Step 744) —
         Security Camera Feed ของอาคาร 1 ไม่มีเหตุผลต้องไปถึง Core Router ของอาคาร 10 เลย
```

> **จุดสำคัญสำหรับข้อสอบ CCIE Enterprise Infrastructure**: คำถาม Scaling มักไม่ถามตัวเลขที่แน่นอน
> (Platform เปลี่ยน Spec บ่อย) แต่ถามหลักการ **"State ที่ต้องคิดคือ Group × Source ไม่ใช่ Receiver"**
> และ **"RP ตัวเดียวคือ Bottleneck ทั้ง Control-plane (Register/Join Processing) และ Availability
> (Single Point of Failure ตาม Part 42 Step 411)"** — สอง Concept นี้คือรากฐานของทุก Step ที่เหลือ
> ใน Part นี้

---

## Step 742 — Multicast VPN (MVPN): Default MDT / Data MDT

### 742.1 ปัญหา: Multicast ข้าม VRF หรือข้าม MPLS VPN Core ทำงานอย่างไร

จาก [Part 51](part-051-vrf-lite-gre-ipsec.md) เราแยก **VLAN 40 (WIFI)** เข้า VRF ชื่อ `GUEST`
ด้วย VRF-Lite — คำถามที่ Part 51 ไม่ได้ตอบคือ **"ถ้า GUEST VRF ต้องมี Multicast Video ของตัวเอง
(เช่น Guest Portal Advertisement Video) จะ Config อย่างไร"** — คำตอบคือ **PIM ต้องรันแยกต่อ VRF
เหมือนที่ OSPF/EIGRP รันแยกต่อ VRF ได้ (Part 51 Step 503)**:

```
CORE-SW1(config)# ip multicast-routing vrf GUEST
CORE-SW1(config)# ip pim vrf GUEST rp-address 1.1.1.100
CORE-SW1(config)# interface Vlan40
CORE-SW1(config-if)# ip pim sparse-mode
```

**Global Table PIM** (ที่เราใช้มาตั้งแต่ Part 41) และ **VRF GUEST PIM** เป็น **Multicast Domain
คนละอันกันโดยสิ้นเชิง** — RP ของ Global Table (`1.1.1.100`) และ RP ของ VRF GUEST เป็น Address
คนละ Namespace แม้จะเผอิญเลขเดียวกัน (Router มองผ่าน VRF Context แยกจากกันเสมอ) นี่คือ VRF-Lite
ระดับ Hop-by-Hop ตามที่ Part 51 อธิบายไว้ — **Multicast แยก VRF ได้ตรงไปตรงมาไม่มีอะไรซับซ้อนเพิ่ม**
เพราะยังอยู่บน Router/Switch ตัวเดียวกันทั้งหมด (ไม่มี MPLS Core คั่นกลาง)

### 742.2 ปัญหาที่ยากขึ้น: ข้าม MPLS L3VPN Core (Preview ของ [Part 86-87](part-086-mpls-l3vpn-fundamentals.md))

เมื่อ Design ขยายไปมี **Regional Data Center** ที่เชื่อมผ่าน **MPLS L3VPN** (SP-grade VRF ผ่าน
MP-BGP ตามที่ [Part 51 Step 501](part-051-vrf-lite-gre-ipsec.md#step-501--vrf-virtual-routing-and-forwarding-พื้นฐาน)
เกริ่นไว้ว่าจะเรียนเต็มใน Part 86-87) ปัญหาใหม่จะเกิดขึ้น: **P-Router (Provider Core Router ที่ไม่รู้
จัก VRF ของ Customer เลย) ไม่มีทาง Forward Customer Multicast Traffic ข้าม MPLS Core ได้** เพราะ
P-Router เห็นแค่ MPLS Label ไม่เห็น Customer Multicast Group/Source Address ข้างในเลย

```
                  Customer VRF Multicast (Site A)              Customer VRF Multicast (Site B)
                     239.1.1.1 All-Hands                          Receiver ต้องการดู Feed เดียวกัน
                          │                                               │
                    ┌─────┴─────┐    MPLS L3VPN Core (P-Network)   ┌─────┴─────┐
                    │  PE-A      │◄──── P-Router ไม่รู้จัก ────────►│  PE-B      │
                    │ (WAN-EDGE) │      Customer Multicast เลย     │ (Regional  │
                    └───────────┘      (เห็นแค่ MPLS Label)         │  DC Edge)  │
                                                                    └───────────┘
```

### 742.3 MVPN แก้ปัญหานี้อย่างไร: "Tunnel PIM ของ Customer ผ่าน PIM ของ Provider"

**MVPN (Multicast VPN)** แบบดั้งเดิม (Draft-Rosen, GRE-based) แก้ปัญหานี้ด้วยแนวคิดที่ตรงไปตรงมา
มาก: **ให้ P-Network มี PIM-SM Domain ของตัวเอง (Provider Multicast, ไม่เกี่ยวกับ Customer) แล้ว
ให้ PE Router ทุกตัวของ VRF เดียวกัน Join Group พิเศษของ Provider ร่วมกัน** — สร้าง **GRE Tunnel
เสมือน (MDT — Multicast Distribution Tree)** ที่ Encapsulate Customer Multicast Traffic ทั้งหมด
ไว้ข้างใน แล้วส่งผ่าน Provider PIM Tree ข้าม Core ไปโดยที่ P-Router ไม่ต้องรู้จัก Customer Multicast
เลยแม้แต่นิดเดียว (เห็นแค่ Provider Group Address ธรรมดา)

```
Default MDT (Multicast Distribution Tree) — มีเสมอ ต่อ 1 VRF ต่อ 1 Provider Group

  PE-A (VRF CUSTOMER-A)                              PE-B (VRF CUSTOMER-A)
     │  Customer (*,G)/(S,G) State ปกติ                  │  Customer (*,G)/(S,G) State ปกติ
     ▼                                                     ▼
  ┌─────────────────────────────────────────────────────────┐
  │  GRE Tunnel เสมือน (Encapsulate ด้วย Provider Group      │
  │  239.192.1.1 — ทุก PE ของ VRF เดียวกัน Join Group นี้      │
  │  ร่วมกันเสมอ ไม่ว่าจะมี Customer Traffic จริงหรือไม่)       │
  └─────────────────────────────────────────────────────────┘
     ▲                                                     ▲
     └──────────── Provider PIM-SM Domain (P-Network) ─────┘
              (P-Router เห็นแค่ 239.192.1.1 ธรรมดา ไม่รู้จัก
               239.1.1.1/232.50.1.1 ของ Customer เลย)
```

### 742.4 Data MDT — แก้ปัญหา "Flood ทุก PE แม้ไม่มี Receiver"

ปัญหาของ Default MDT คือ: **Provider Group เดียวถูก Join โดยทุก PE ของ VRF นั้นเสมอ** — ถ้า
Customer มี High-Bandwidth Flow (เช่น All-Hands Video 4K) แต่ PE-C (สมมติมี Site C อีกแห่ง) ไม่มี
Receiver ของ Video นี้เลย **Traffic ก็จะยังถูกส่งไปถึง PE-C ผ่าน Default MDT อยู่ดี** (เพราะ Default
MDT เป็น Shared Tree ของทุก PE ใน 1 VRF) สิ้นเปลือง Bandwidth บน Core เหมือนปัญหา Broadcast ที่
[Part 41 Step 401](part-041-multicast-fundamentals.md#step-401--multicast-คืออะไร-unicast-vs-broadcast-vs-multicast)
เคยอธิบายไว้

**Data MDT** แก้ปัญหานี้: เมื่อ Customer (S,G) มี Traffic Rate เกิน Threshold ที่กำหนด PE ต้นทาง
จะสร้าง **Provider Group ใหม่เฉพาะ Flow นี้** (จาก Pool ที่ Config ไว้) แล้วส่ง **MDT Join TLV**
บอก PE อื่นๆ ทุกตัว — เฉพาะ **PE ที่มี Receiver จริงของ Flow นี้เท่านั้น** จะ Join Provider Group
ใหม่ตัวนี้ (PE ที่ไม่มี Receiver จะไม่ Join และไม่ได้รับ Traffic เลย) — ทำหน้าที่เหมือน SPT-Switchover
ใน PIM-SM ปกติ แต่ทำในระดับ "Provider Tree" แทน

### 742.5 Config โดยสรุป (Conceptual Preview — ไม่ลง Lab เต็มใน Part นี้)

```
! ===== PE-A (WAN-EDGE-1 สมมติทำหน้าที่ PE) — VRF ของ Customer ที่ต้องการ MVPN =====
PE-A(config)# ip vrf CUSTOMER-A
PE-A(config-vrf)# rd 65001:100
PE-A(config-vrf)# route-target export 65001:100
PE-A(config-vrf)# route-target import 65001:100
PE-A(config-vrf)# mdt default 239.192.1.1
PE-A(config-vrf)# mdt data 232.1.1.0 0.0.0.255 threshold 500
PE-A(config-vrf)# exit

! ===== เปิด Multicast Routing ทั้ง Global (Provider) และ VRF (Customer) =====
PE-A(config)# ip multicast-routing               ! Provider (Global) PIM Domain
PE-A(config)# ip multicast-routing vrf CUSTOMER-A ! Customer PIM Domain
PE-A(config)# ip pim vrf CUSTOMER-A rp-address 1.1.1.100  ! RP ของ Customer เท่านั้น (คนละ Domain)
```

> **สรุปสำหรับ CCIE Enterprise Infrastructure**: MVPN เป็นหัวข้อที่ **เชื่อมกับ Service Provider
> Track มากกว่า Enterprise Track** โดยตรง — ข้อสอบระดับ Enterprise มักถามแค่ **Concept ที่ต้องรู้**
> (Default MDT มีเสมอต่อ VRF, Data MDT สร้างเมื่อ Traffic เกิน Threshold เพื่อไม่ Flood ทุก PE,
> P-Router ไม่ต้องรู้จัก Customer Multicast เลย) มากกว่า Syntax เชิงลึก — หลักสูตรนี้ให้ภาพรวมที่
> เพียงพอตรงนี้ และจะกลับมาลง Lab เต็มรูปแบบตอนเรียน MPLS L3VPN จริงใน Part 86-87 ที่ Customer
> จะมีทั้ง Regional Data Center และ Branch ที่ต้องรับ Multicast ข้าม Provider Core จริง

---

## Step 743 — PIM Design Pattern ที่ Scale ได้: Decision Matrix ระดับ CCIE

### 743.1 ทวนตัวเลือก RP Discovery ทั้ง 3 แบบจาก Part 41/42

| แบบ | เรียนใน | กลไกหลัก |
|---|---|---|
| Static RP | [Part 41 Step 407](part-041-multicast-fundamentals.md#step-407--basic-pim-sm-configuration) | `ip pim rp-address` Hardcode เหมือนกันทุก Router |
| Auto-RP | [Part 42 Step 412](part-042-multicast-advanced.md#step-412--auto-rp-cisco-proprietary) | Candidate-RP ประกาศผ่าน `224.0.1.39`, Mapping Agent เลือกและกระจายผ่าน `224.0.1.40` |
| BSR | [Part 42 Step 413](part-042-multicast-advanced.md#step-413--bsr-bootstrap-router-rfc-5059) | Candidate-BSR ถูกเลือกเป็น BSR แล้ว Flood Candidate-RP-Set ผ่าน PIM Hello ปกติ |
| Anycast RP + MSDP | [Part 42 Step 415-416](part-042-multicast-advanced.md#step-415--anycast-rp-rfc-4610) | RP มากกว่า 1 ตัว ใช้ IP เดียวกัน (OSPF เลือกทางใกล้สุด) + MSDP Sync Source ระหว่างกัน |

Part 41/42 สอนแต่ละแบบแยกกัน — Part นี้จะรวมเป็น **Decision Matrix เดียว** ที่ใช้ตอบคำถาม CCIE-level
Design จริง: **"เลือกแบบไหนสำหรับ Tier ของ Network ขนาดนี้"**

### 743.2 Decision Matrix ตาม 3 Tier ของขนาด Network

| เกณฑ์ | Tier 1: Small (Lab/Branch เดี่ยว, <10 PIM Router) | Tier 2: Medium (Campus 1 Site, 10-50 PIM Router — เช่น Campus 4-10 อาคารของ Part นี้) | Tier 3: Large (Multi-site Enterprise, 50+ PIM Router, มี WAN/DMVPN/MPLS ข้าม Site) |
|---|---|---|---|
| **RP Redundancy ที่ต้องมี** | ต่ำ — Downtime สั้นยอมรับได้ | สูง — All-Hands ข้ามอาคารต้องไม่มี SPOF | สูงสุด — RP ต้อง Available ทุก Site พร้อมกัน |
| **แนะนำหลัก** | **Static RP** (Config ง่ายที่สุด, พอสำหรับ Scale เล็ก) | **Anycast RP + MSDP ภายใน Site เดียว** (RP 2 ตัว Load-split ตาม Part 42 Step 415) | **Anycast RP ต่อ Site + MSDP Mesh-group ระหว่าง Site** (ดู Step 748) |
| **ทางเลือกรอง** | Auto-RP/BSR ถ้าอยากฝึก Dynamic Discovery ไว้ก่อน Scale ขึ้น | BSR ถ้ามี Mixed-Vendor ปนอยู่ในโดเมนเดียวกัน | BSR ยังใช้ได้ถ้า Mixed-Vendor แต่ต้องคู่กับ Anycast RP เสมอสำหรับ Redundancy จริง — BSR/Auto-RP เองไม่มี Load-splitting |
| **จำนวน RP Active จริงในเวลาเดียวกัน** | 1 (ไม่มี Redundancy) | 2 (Anycast, Load-split ตามความใกล้ผ่าน OSPF) | 2 ต่อ Site (แต่ละ Site มี Anycast RP คู่ของตัวเอง, ทุกคู่ Sync กันผ่าน MSDP) |
| **Operational Complexity** | ต่ำสุด | กลาง — ต้องดูแล OSPF Reachability ของ Anycast Address + MSDP Peer 1 คู่ | สูง — ต้องดูแล MSDP Mesh-group หลายคู่ + RPF Check ของ SA Message (Step 748) |
| **จุดอ่อนที่ต้องระวัง** | Single Point of Failure ตรงตามคำเตือนของ [Part 42 Step 411](part-042-multicast-advanced.md#step-411--จุดอ่อนของ-static-rp-single-point-of-failure) | Rollout ไม่ครบทุก Router ทำ RP Mapping Inconsistency (ตรงกับ [Part 65 Step 644](part-065-multicast-troubleshooting.md#step-644--scenario-3-rp-mapping-ไม่ตรงกันหลัง-migrate-ไป-anycast-rp-ไม่ครบทุก-router)) | MSDP SA Message Loop ถ้าไม่มี Mesh-group ในโครงสร้าง MSDP Full-mesh ขนาดใหญ่ |

### 743.3 กราฟการตัดสินใจแบบง่าย (Quick Decision Flow)

```
เริ่มต้น: ต้อง Design RP สำหรับ Network ใหม่
        │
        ▼
  จำนวน PIM Router ทั้ง Domain > 10 ตัว หรือมี Source/Receiver มากกว่า 1 อาคาร/Site?
        │
   ไม่ ─┤─ ใช่
   │    │
   ▼    ▼
 Static RP        มี WAN/Multi-site เชื่อมกันหรือไม่ (ต้องมี RP มากกว่า 1 Domain ที่ Sync กัน)?
 พอเพียง                │
                    ไม่ ─┤─ ใช่
                    │    │
                    ▼    ▼
              Anycast RP      Anycast RP ต่อ Site
              + MSDP          + MSDP Mesh-group ข้าม Site (Step 748)
              (1 คู่พอ)       + Multicast Boundary กรอง Group ที่ไม่ควรข้าม Site (Step 744)
```

### 743.4 สำหรับ Use Case ของ Part นี้: เลือกแบบไหน

Campus 10 อาคารตาม Part 72 อยู่ใน **Tier 2** (Multi-building, Single Site) — ใช้ **Anycast RP +
MSDP ภายใน Site เดียว** ต่อจาก Part 42 ได้ทันที ไม่ต้องเปลี่ยนกลไก แต่ Regional Data Center และ
Branch ที่เชื่อมผ่าน WAN ทำให้ Design รวมกลายเป็น **Tier 3** — ต้องเพิ่ม **Anycast RP คู่ที่สองที่ DC**
และ **MSDP Mesh-group** เชื่อมทั้งสอง Site เข้าด้วยกัน (จะ Config เต็มรูปแบบใน Step 748 และ Step 750)

---

## Step 744 — Multicast Boundary/Filtering Design

### 744.1 ปัญหาที่ Boundary แก้: "ไม่ใช่ทุก Group ควรข้ามทุกที่"

จาก Step 741 เราเห็นว่า Campus 10 อาคารมี Group ประมาณ 252 ตัว — **แต่ไม่ใช่ทุก Group ควรถูก
Forward ไปทุกที่**: Security Camera Feed ของอาคาร 3 (`239.70.3.x`) ไม่มีเหตุผลทางธุรกิจต้องไปถึง
Regional Data Center หรือ Branch เลย (แค่ NVR Server ในอาคารเดียวกันดูก็พอ) — ถ้าไม่กรอง Traffic นี้
จะไหลข้าม WAN Link ที่มี Bandwidth จำกัดโดยไม่จำเป็น สิ้นเปลืองเหมือนปัญหา Flood ที่ [Part 41 Step
401](part-041-multicast-fundamentals.md#step-401--multicast-คืออะไร-unicast-vs-broadcast-vs-multicast)
เตือนไว้แต่ต้น — และยังเป็นความเสี่ยงด้าน Security ด้วย (Feed ภายในไม่ควรรั่วไหลออกนอก Boundary)

### 744.2 คำสั่งหลัก: `ip multicast boundary`

```
Router(config-if)# ip multicast boundary <access-list> [filter-autorp]
```

| ส่วนประกอบ | ความหมาย |
|---|---|
| ระดับที่ Config | **Interface** เท่านั้น (ไม่ใช่ Global เหมือน `ip pim rp-address`) |
| `<access-list>` | ACL (Standard/Extended) ที่กำหนดว่า Group ไหน **permit** = ให้ข้าม Interface นี้ได้ (ทั้งทิศทาง Join และ Data), Group ไหน **deny** = block ไม่ให้ข้าม |
| `filter-autorp` | Keyword เสริม — ถ้าใส่ จะ **Filter RP-Announce/RP-Discovery Message ของ Auto-RP (`224.0.1.39`/`224.0.1.40`)** ที่พูดถึง Group ในช่วงที่ ACL Deny ด้วย ไม่ให้ Mapping Information ของ Group ที่ควรถูกกันไว้หลุดข้าม Boundary ไปพร้อมกับ Auto-RP |
| ผลกระทบ | Block ทั้ง **PIM Join/Prune ที่จะสร้าง State ข้าม Boundary** และ **Multicast Data Packet ของ Group ที่ Deny** — ทำหน้าที่เป็น "Administrative Scope Boundary" เหมือนแนวคิด TTL-based Scoping ของ Multicast รุ่นเก่า แต่ควบคุมด้วย Group Address ชัดเจนกว่า |

### 744.3 ออกแบบ ACL: Group ไหนควรข้าม WAN ได้ Group ไหนไม่ควร

```
                    ตาราง Group Range ของ Campus (ทบทวนจาก Step 741.2)

  239.1.1.1              All-Hands              ── ต้องข้าม WAN ได้ (Regional DC + Branch ต้องดู)
  239.50.<bldg>.1        Lobby Signage          ── ไม่ควรข้าม WAN (เฉพาะอาคารตัวเอง)
  239.60.<bldg>.<room>   Training Room Video    ── ข้าม WAN ได้เฉพาะ Branch ที่ Subscribe (อนุญาตแบบเจาะจง)
  239.70.<bldg>.<cam>    Security Camera        ── ไม่ควรข้าม WAN เด็ดขาด (Security-sensitive)
  232.50.1.1             Market-Data (SSM)      ── ข้าม WAN ได้ (Trading Floor อาจอยู่หลาย Site)
```

### 744.4 Config เต็มรูปแบบ: Apply Boundary บน WAN-facing Interface ของ CORE-SW1/CORE-SW2

```
! ===== ACL: กำหนดว่า Group ช่วงไหน "ข้าม WAN Boundary ได้" (permit) และช่วงไหน "ต้องกันไว้" (deny) =====
CORE-SW1(config)# ip access-list extended MCAST-WAN-BOUNDARY
CORE-SW1(config-ext-nacl)# remark ==== อนุญาต Group ที่ตั้งใจให้ข้าม Site ====
CORE-SW1(config-ext-nacl)# permit ip any host 239.1.1.1
CORE-SW1(config-ext-nacl)# permit ip any 239.60.0.0 0.0.255.255
CORE-SW1(config-ext-nacl)# permit ip any host 232.50.1.1
CORE-SW1(config-ext-nacl)# remark ==== กันช่วง Signage/Camera ไว้ในอาคาร/Site เท่านั้น ====
CORE-SW1(config-ext-nacl)# deny   ip any 239.50.0.0 0.0.255.255
CORE-SW1(config-ext-nacl)# deny   ip any 239.70.0.0 0.0.255.255
CORE-SW1(config-ext-nacl)# remark ==== Group อื่นๆ ที่ไม่รู้จัก ให้ผ่านตาม Policy ปกติของ Domain ====
CORE-SW1(config-ext-nacl)# permit ip any 239.0.0.0 0.255.255.255
CORE-SW1(config-ext-nacl)# exit

! ===== Apply บน Interface ที่ต่อไปทาง WAN Edge (ทางที่ไปยัง Regional DC/Branch) =====
CORE-SW1(config)# interface Port-channel2
CORE-SW1(config-if)# description ** Uplink to WAN-EDGE-1 (Regional DC / Branch) **
CORE-SW1(config-if)# ip multicast boundary MCAST-WAN-BOUNDARY filter-autorp
CORE-SW1(config-if)# exit

! ===== ทำเหมือนกันบน CORE-SW2 (ทางไป WAN-EDGE-2 — ตาม 00-ip-address-plan.md Dual-homed Design) =====
CORE-SW2(config)# ip access-list extended MCAST-WAN-BOUNDARY
CORE-SW2(config-ext-nacl)# permit ip any host 239.1.1.1
CORE-SW2(config-ext-nacl)# permit ip any 239.60.0.0 0.0.255.255
CORE-SW2(config-ext-nacl)# permit ip any host 232.50.1.1
CORE-SW2(config-ext-nacl)# deny   ip any 239.50.0.0 0.0.255.255
CORE-SW2(config-ext-nacl)# deny   ip any 239.70.0.0 0.0.255.255
CORE-SW2(config-ext-nacl)# permit ip any 239.0.0.0 0.255.255.255
CORE-SW2(config-ext-nacl)# exit
CORE-SW2(config)# interface Port-channel2
CORE-SW2(config-if)# ip multicast boundary MCAST-WAN-BOUNDARY filter-autorp
CORE-SW2(config-if)# exit
```

> **จุดที่ต้องระวังที่สุด**: `deny` ใน ACL ของ `ip multicast boundary` **ไม่เหมือนกับ ACL ปกติที่
> ปิดท้ายด้วย Implicit Deny All** — ควรเขียน ACL ให้ครบทุก Group Range ที่มีจริง (บรรทัดสุดท้าย
> `permit ip any 239.0.0.0 0.255.255.255` ในตัวอย่างนี้กันไม่ให้ Group ใหม่ในอนาคตที่ยังไม่ได้ตั้งใจ
> Block ถูก Implicit-deny ไปโดยไม่ตั้งใจ) — ตรงข้ามกับ Security ACL ทั่วไปที่มักอยากให้ Deny เป็น
> Default (Zero-Trust) กรณี Multicast Boundary ต้อง **คิดให้ชัดว่า Deny เฉพาะ Group ที่รู้ว่าไม่ควร
> ข้ามจริงๆ เท่านั้น** ไม่ใช่ Deny ทุกอย่างแล้วค่อย Permit — เพราะ Group ใหม่ที่ Business เพิ่มมาแบบ
> ไม่ทันแจ้ง Network Team จะโดน Block ไปเงียบๆ (ปัญหาแบบเดียวกับ RPF Silent Failure ใน [Part 65
> Step 643](part-065-multicast-troubleshooting.md#step-643--scenario-2-rpf-failure-จาก-static-route-ทดสอบที่ลืมถอด))

### 744.5 Verification

```
CORE-SW1# show running-config interface Port-channel2 | include boundary
 ip multicast boundary MCAST-WAN-BOUNDARY filter-autorp

! ก่อน Config: Receiver ที่ Regional DC เห็น 239.70.3.5 (Camera อาคาร 3) หลุดเข้ามาทั้งที่ไม่ควร
! หลัง Config: (S,G)/(*,G) ของ Group ที่ถูก Deny จะไม่ปรากฏใน show ip mroute ของฝั่ง WAN เลย

CORE-SW1# show ip mroute 239.70.3.5
Group 239.70.3.5 not found
                                    <-- ถูกก้นไว้ในอาคาร ไม่มี State ฝั่ง WAN-facing เลย ตามที่ตั้งใจ

CORE-SW1# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:12:30/00:02:47, RP 1.1.1.100, flags: SJC
  Outgoing interface list:
    Port-channel2, Forward/Sparse, 00:12:30/00:02:47   <-- All-Hands ยังข้าม WAN ได้ตามที่ตั้งใจ
```

---

## Step 745 — Multicast + QoS Integration

### 745.1 ปัญหา: Video Multicast เป็น Traffic ที่ "แย่งกับตัวเองได้เก่งมาก"

ทบทวนจาก [Part 40](part-040-advanced-qos.md) — MQC (Class-map → Policy-map → Service-policy)
ถูก Config ไว้แล้วบน WAN-EDGE-1 (`policy-map WAN-EDGE-QOS` มี Class VOICE (`priority`), VIDEO,
CRITICAL-DATA, class-default) — แต่ Class `VIDEO` เดิมนั้น Match ด้วย Protocol/Port ทั่วไป **ไม่ได้
แยก Multicast Video Stream ที่มี Rate สูงและ Burst แรง (เช่น All-Hands 4K, Training Room Video Wall)
ออกจาก Unicast Video Call ทั่วไป (เช่น Webex/Teams 1-to-1)** — ทั้งสองแบบมีพฤติกรรม Traffic ต่างกัน
มาก: Unicast Video Call ปรับ Bitrate ได้ตาม Congestion (Adaptive), แต่ **Multicast Video Stream
จากอุปกรณ์ Broadcast/Signage ส่วนใหญ่ส่งที่ Bitrate คงที่ ไม่ปรับตาม Network เลย** — ถ้าใช้ Class
เดียวกันปนกัน Multicast Stream ที่ Rate สูงจะแย่ง Bandwidth ของ Unicast Video Call จนกระตุก

### 745.2 แยก Class เฉพาะ Multicast Video ด้วย ACL Match Group Address

```
! ===== ACL: Match เฉพาะ Multicast Group Range ที่เป็น "Video/Signage สำคัญที่ต้องการันตี Bandwidth" =====
WAN-EDGE-1(config)# ip access-list extended MCAST-VIDEO-CRITICAL
WAN-EDGE-1(config-ext-nacl)# permit ip any host 239.1.1.1
WAN-EDGE-1(config-ext-nacl)# permit ip any 239.60.0.0 0.0.255.255
WAN-EDGE-1(config-ext-nacl)# exit

! ===== Class-map แยกออกจาก VIDEO Class เดิมของ Part 40 =====
WAN-EDGE-1(config)# class-map match-all MULTICAST-VIDEO
WAN-EDGE-1(config-cmap)# match access-group name MCAST-VIDEO-CRITICAL
WAN-EDGE-1(config-cmap)# exit
```

### 745.3 Marking ที่ Source-facing Interface (Trust Boundary)

หลักการ Trust Boundary จาก [Part 40](part-040-advanced-qos.md) ยังใช้เหมือนเดิม — Marking ควรทำ
ที่ **จุดใกล้ Source ที่สุด (Access/Distribution Layer)** ไม่ใช่รอไป Mark ที่ WAN Edge เท่านั้น
เพื่อให้ Priority ถูกเคารพตลอดเส้นทางในแคมปัสด้วย ไม่ใช่แค่ตอนออก WAN:

```
! ===== DIST-SW3 (ทางเข้าของ Building ที่มี Video Encoder/Signage Server) =====
DIST-SW3(config)# ip access-list extended MCAST-VIDEO-CRITICAL
DIST-SW3(config-ext-nacl)# permit ip any host 239.1.1.1
DIST-SW3(config-ext-nacl)# permit ip any 239.60.0.0 0.0.255.255
DIST-SW3(config-ext-nacl)# exit
DIST-SW3(config)# class-map match-all MULTICAST-VIDEO
DIST-SW3(config-cmap)# match access-group name MCAST-VIDEO-CRITICAL
DIST-SW3(config-cmap)# exit
DIST-SW3(config)# policy-map MARK-MULTICAST-VIDEO
DIST-SW3(config-pmap)# class MULTICAST-VIDEO
DIST-SW3(config-pmap-c)# set dscp af41              ! AF41 = Assured Forwarding วิดีโอมาตรฐาน (RFC 4594)
DIST-SW3(config-pmap-c)# exit
DIST-SW3(config-pmap)# exit
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# service-policy input MARK-MULTICAST-VIDEO
```

> **ทำไมไม่ใช้ `EF` (Expedited Forwarding, DSCP 46) แบบ Voice**: `EF` สงวนไว้สำหรับ Traffic ที่
> **Rate ต่ำและคงที่จริง** (Voice RTP ~ 50-100 kbps ต่อ Call) เพราะ LLQ (`priority`) ที่ผูกกับ `EF`
> ทำงานแบบ **Strict-Priority ไม่มีการจำกัด "สัดส่วน" ที่ชัดเจนพอสำหรับ Traffic Bandwidth สูงและ
> Burst แรงอย่าง Video 4K** — ถ้า Video Stream ถูกจัดเป็น `EF`/LLQ เดียวกับ Voice จะเสี่ยง**แย่ง
> Priority Queue จน Voice หลุด/มี Jitter** (Voice Sensitive ต่อ Delay/Jitter มากกว่า Video มาก) —
> `AF41` ใช้ CBWFQ (`bandwidth`/`bandwidth percent`) การันตี "สัดส่วน" ที่กว้างพอสำหรับ Video โดย
> ไม่ไปแย่ง Strict-Priority Queue ของ Voice เลย

### 745.4 Policy-map ที่ WAN Edge: ขยาย `WAN-EDGE-QOS` เดิมจาก Part 40

```
WAN-EDGE-1(config)# policy-map WAN-EDGE-QOS
WAN-EDGE-1(config-pmap)# class VOICE
WAN-EDGE-1(config-pmap-c)#  priority percent 10          ! LLQ เดิมจาก Part 40 — คงไว้ไม่แตะ
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class MULTICAST-VIDEO
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 25          ! CBWFQ การันตีขั้นต่ำ ไม่ใช่ Strict-Priority
WAN-EDGE-1(config-pmap-c)#  random-detect dscp-based      ! WRED ป้องกัน TCP Global Sync (Part 40 Step 394) — สำหรับ Control Traffic ที่ปนมา ไม่ใช่ Video UDP เอง
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class VIDEO
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 20          ! Unicast Video Call เดิมจาก Part 40 — แยก Class ชัดเจนจาก Multicast แล้ว
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class CRITICAL-DATA
WAN-EDGE-1(config-pmap-c)#  bandwidth percent 20
WAN-EDGE-1(config-pmap-c)# exit
WAN-EDGE-1(config-pmap)# class class-default
WAN-EDGE-1(config-pmap-c)#  fair-queue
```

> **หมายเหตุเรื่อง WRED กับ Multicast UDP**: Video Stream ส่วนใหญ่เป็น **UDP** ซึ่ง **ไม่มี
> Retransmission/Congestion Window แบบ TCP** — WRED (`random-detect`) ที่ Design มาเพื่อแก้ TCP
> Global Synchronization ([Part 40 Step 394](part-040-advanced-qos.md)) **แทบไม่ช่วยอะไรกับ UDP
> Video โดยตรง** (Drop UDP Packet ไปเฉยๆ ไม่มี Windowing ให้ปรับ) — เหตุผลที่ยังใส่ `random-detect
> dscp-based` ไว้ใน Class เดียวกันคือเผื่อกรณีมี Control-plane Traffic ปนมาใน Class นี้ (เช่น RTCP)
> — **ถ้า Class มี UDP Video ล้วนๆ 100% ให้ใช้ Tail-drop ปกติแทน WRED และเน้นที่ `bandwidth percent`
> ให้เพียงพอจริงมากกว่า** เพื่อไม่ให้ต้อง Drop เลยเป็นหลัก — คำถามข้อสอบมักถามตรงนี้: **"ทำไม WRED
> ไม่ช่วย UDP Video เท่า TCP Data"**

### 745.5 กฎการ Sizing `bandwidth percent` สำหรับ Multicast Video โดยเฉพาะ

```
กฎการคำนวณ bandwidth percent ของ Class MULTICAST-VIDEO:

  ต้อง >= (Bitrate รวมของ Stream ที่ "จำเป็นต้องข้าม Link นี้จริง" ตาม Boundary ที่ตั้งไว้ใน Step 744)
         ────────────────────────────────────────────────────────────────────────
                          Bandwidth ของ WAN Link

ตัวอย่าง: All-Hands 4K (~15 Mbps) + Training Room Video (สมมติ 2 Stream ข้าม WAN ที่ ~5 Mbps รวม)
         = 20 Mbps ต้องการ บน WAN Link 100 Mbps  →  ต้องการอย่างน้อย 20% (ตั้งไว้ 25% เผื่อ Burst)

ข้อสำคัญ: เพราะ Multicast ส่ง "1 Copy เดียว" ไม่ว่าจะมี Receiver กี่คนที่ปลายทางอีกฝั่ง
         (Fan-out เกิดที่ Router ปลายทางหลัง Data ข้าม WAN Link มาแล้ว) การคำนวณ Bandwidth
         WAN Link จึงคิดจาก "จำนวน Stream ไม่ซ้ำ" ไม่ใช่ "จำนวน Receiver" — นี่คือข้อได้เปรียบ
         สำคัญที่สุดของ Multicast เทียบกับ Unicast ตามที่ Part 41 Step 401 อธิบายไว้แต่ต้น
         ยังคงจริงแม้ในบริบท QoS Sizing ระดับ WAN
```

---

## Step 746 — Multicast over DMVPN

### 746.1 ทบทวน: DMVPN คือ NBMA (Non-Broadcast Multi-Access) โดยธรรมชาติ

จาก [Part 52](part-052-dmvpn-fundamentals.md) Tunnel0/Tunnel10 (mGRE) บน WAN-EDGE-1 (Hub) เป็น
**Interface เดียวที่คุยกับหลาย Spoke พร้อมกัน (BRANCH-RTR, BRANCH2-RTR)** — แต่ physical characteristic
ของมันคือ **NBMA**: ไม่มี Native Broadcast/Multicast แบบ Ethernet Segment จริง ทุก Frame ที่ส่งออก
Tunnel Interface (รวมถึง Multicast) **ต้องถูก Replicate เป็น Unicast แยกไปยังแต่ละ NBMA Address
ของ Spoke ที่ NHRP Registration ไว้** — ต่างจาก LAN Segment ปกติที่ Multicast/Broadcast Frame
เดียวไปถึงทุกคนพร้อมกันด้วย 1 Transmission จริง

```
LAN Segment ปกติ (Ethernet)                    mGRE Tunnel (NBMA)

  Router ส่ง Multicast 1 ครั้ง                   Router (Hub) ต้อง Replicate
  ทุก Host ใน Segment ได้รับพร้อมกัน               ส่ง Unicast (ครอบด้วย GRE) แยกไปทีละ Spoke
  (1 Transmission ทาง Physical Layer จริง)        (N Transmission ทาง Physical ถ้ามี N Spoke)
```

### 746.2 `ip pim nbma-mode` — บอก Router ว่า Interface นี้ต้อง Replicate แบบ Unicast

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip pim nbma-mode
```

| ผลของคำสั่งนี้ | รายละเอียด |
|---|---|
| ไม่มี Command นี้ (Default) | IOS พยายามส่ง PIM Join/Prune และ Multicast Data แบบ "Multi-access" ปกติบน Tunnel Interface — ใช้ได้กับ DMVPN Phase 2/3 ที่มี Spoke-to-Spoke Direct Tunnel จริง แต่ **มีปัญหากับ Phase 1 หรือ Spoke ที่ยังไม่มี Direct Tunnel ต่อกัน** เพราะ PIM คิดว่า Interface นี้เป็น Multi-access Segment เดียวที่ Neighbor ทุกตัวคุยกันตรงได้ (เหมือน LAN) ทั้งที่จริง Spoke คุยกันตรงไม่ได้ถ้ายังไม่มี NHRP Resolve/Tunnel สร้างจริง |
| มี `ip pim nbma-mode` | Router (มักตั้งที่ Hub) เปลี่ยนพฤติกรรมเป็น **ส่ง PIM Join/Prune และ (ถ้าจำเป็น) Data แบบ Replicate เป็น Unicast แยกไปยัง Neighbor แต่ละตัวที่รู้จักจริงบน Tunnel** ไม่ใช่ Multicast แบบ Multi-access — ทำให้ทำงานถูกต้องแม้ Spoke คุยกันโดยตรงไม่ได้ (Hub-and-Spoke Only แบบ DMVPN Phase 1) |
| ข้อจำกัด | Router ต้องรู้จัก **PIM Neighbor List บน Tunnel Interface ทีละตัว** (Static ผ่าน `neighbor` command หรือ Dynamic ผ่าน NHRP Registration ที่เกิดขึ้นแล้ว) — ถ้า Spoke ยังไม่ Register กับ Hub เลย (Tunnel ยัง Down) Router จะไม่มีทาง Replicate ไปให้ได้ |

### 746.3 ทำไม DMVPN Phase ที่ต่างกันกระทบ Multicast ต่างกัน

ทบทวนจาก [Part 53 Step 521](part-053-dmvpn-advanced.md#step-521--dmvpn-phase-3-คืออะไร) — DMVPN
มี 3 Phase ต่างกันที่ **Spoke-to-Spoke Direct Tunnel** — ผลกระทบต่อ Multicast:

| Phase | Spoke-to-Spoke ตรง | ผลกระทบต่อ Multicast |
|---|---|---|
| Phase 1 | ไม่มี — ทุก Traffic ผ่าน Hub เสมอ | **เหมาะกับ `ip pim nbma-mode` ที่สุด** — Hub เป็นจุดเดียวที่ต้อง Replicate Multicast ไปทุก Spoke อยู่แล้วโดย Design เหมือนกับ RP/Source ที่อยู่ตรงกลางพอดี |
| Phase 2 | มี (Manual NHRP Resolve) | ซับซ้อนขึ้น — ถ้า Source อยู่ที่ Spoke ตัวหนึ่งและ Receiver อยู่ Spoke อีกตัว PIM ต้องตัดสินใจว่าจะส่งผ่าน Hub (Shared Tree ปกติ) หรือพยายาม Direct — **ในทางปฏิบัติ Enterprise ส่วนใหญ่ยังคง Force ให้ Multicast ผ่าน Hub เสมอ** (ไม่ทำ Spoke-to-Spoke Multicast Direct) เพราะความซับซ้อนของ NHRP Multicast Mapping ไม่คุ้มกับ Bandwidth ที่ประหยัดได้ |
| Phase 3 | มี (Auto ผ่าน NHRP Redirect/Shortcut) | เหมือน Phase 2 — Unicast Traffic ใช้ Shortcut ได้ตาม [Part 53](part-053-dmvpn-advanced.md) แต่ **Multicast แทบไม่มี Vendor ไหน Implement Spoke-to-Spoke Multicast Shortcut จริงจัง** — Best Practice ยังคงให้ Multicast วิ่งผ่าน Hub เป็น Rendezvous Point ตามธรรมชาติของ PIM-SM Shared Tree อยู่ดี |

> **ข้อสรุปสำคัญสำหรับ CCIE-level Design**: ไม่ว่า DMVPN จะเป็น Phase ไหน **Best Practice ของ
> Multicast over DMVPN คือให้ Hub ทำหน้าที่เป็นทั้ง RP และจุด Replicate เสมอ** — ไม่ต้องพยายามฝืน
> ทำ Spoke-to-Spoke Multicast Direct (Phase 2/3 Shortcut มีไว้สำหรับ Unicast Data Traffic เท่านั้น
> ในทางปฏิบัติส่วนใหญ่) นี่คือเหตุผลที่ `ip pim nbma-mode` ยังจำเป็นแม้ Design จะใช้ Phase 3 ก็ตาม

### 746.4 RP Reachability ข้าม Spoke — ทำไมต้องคิดเป็นพิเศษ

RP (`1.1.1.100` Anycast ของ Campus) อยู่ที่ CORE-SW1/CORE-SW2 **หลัง** WAN-EDGE-1/2 อีกที (ตาม
Topology ของ [00-ip-address-plan.md](00-ip-address-plan.md)) — Spoke ที่ Join Group ต้องส่ง
PIM Join **ผ่าน Tunnel Interface ไปถึง Hub ก่อน แล้ว Hub ต้อง Route ต่อไปถึง RP จริงอีกที** ผ่าน
Unicast Routing (OSPF/EIGRP บน Tunnel — Part 52/53) — จุดที่ต้องตรวจสอบเสมอ:

```
BRANCH-RTR (Spoke)# show ip route 1.1.1.100
Routing entry for 1.1.1.100/32
  Known via "ospf 1", distance 110, metric 1001
  * via 172.16.201.1, Tunnel0                    <-- ต้องเห็น Route ไป RP ผ่าน Tunnel เสมอ
                                                      ถ้าไม่เห็น = PIM Join จะไม่มีทางไปถึง RP ได้เลย

BRANCH-RTR# show ip pim neighbor
Neighbor Address    Interface    Uptime/Expires  Ver   DR Prio/Mode
172.16.201.1          Tunnel0      00:45:12/00:01:32  v2   1 / DR S P G
                                    <-- ต้องเห็น Hub เป็น PIM Neighbor ผ่าน Tunnel0 ด้วย
```

**ถ้า Route ไป RP ไม่มี (เช่น Summarization ที่ Hub ตาม [Part 53 Step 523](part-053-dmvpn-advanced.md#step-523--summarization-ที่-hub-สำหรับ-phase-3)
บังคับ Advertise แค่ Default Route)** — ต้องตรวจสอบว่า Default Route ที่ Advertise ไปยัง Spoke
เพียงพอให้ RPF Check ผ่าน (RPF อ้างอิง Unicast Routing Table เสมอ ตาม [Part 41 Step 408](part-041-multicast-fundamentals.md#step-408--multicast-forwarding-และ-rpf-check-sg-และ-g))
— ถ้า Spoke มีแค่ Default Route ไป Hub RPF ก็ยัง Pass ได้ตราบใดที่ Multicast Data เดินทางเข้ามา
ทาง Interface เดียวกับที่ Default Route ชี้ไป (ปกติคือ Tunnel0 เสมอ ในกรณี Hub-and-Spoke)

---

## Step 747 — IGMP/MLD Scaling

### 747.1 ปัญหา: Endpoint ที่ Join Group มากเกินจริง (ตั้งใจหรือไม่ตั้งใจ)

Campus 10 อาคารมี Endpoint หลายพันตัว — ถ้า Endpoint ตัวใดตัวหนึ่ง **Misconfigured (Bug ของ
Application) หรือถูกโจมตี (Malware ที่พยายาม Join Group จำนวนมากเพื่อสร้าง State Exhaustion บน
Router)** ส่ง IGMP Membership Report Join Group จำนวนมากผิดปกติ — Router (โดยเฉพาะ Last-Hop
Router อย่าง DIST-SW3) จะสร้าง `(*,G)` State ตามจำนวน Group ที่ Join เข้ามาทุกตัวโดยไม่มีขอบเขต
ถ้าไม่ได้ป้องกันไว้

### 747.2 IGMPv3/MLDv2 Source Filtering ที่ Scale — ทบทวนและขยายจาก SSM

จาก [Part 42 Step 417](part-042-multicast-advanced.md#step-417--ssm-source-specific-multicast)
เราใช้ IGMPv3 INCLUDE Mode กับ SSM (`232.0.0.0/8`) เพื่อไม่ต้องมี RP — ที่ Scale ระดับ Enterprise
ประโยชน์ของ IGMPv3 Source Filtering ไม่ได้จำกัดแค่ SSM: **แม้ Group อยู่ใน `239.0.0.0/8` ปกติ (ไม่
SSM) IGMPv3 ก็ยังลด (S,G) State ที่ไม่จำเป็นได้** เพราะ Client ที่ระบุ Source ชัดเจน (`INCLUDE`)
ทำให้ Last-Hop Router **ไม่ต้องสร้าง `(*,G)` แบบเปิดกว้างรับทุก Source** — ลด Attack Surface และ
State ที่ไม่มีประโยชน์จริงลงไปพร้อมกัน

```
IGMPv2 Join (ไม่ระบุ Source)              IGMPv3 INCLUDE Join (ระบุ Source ชัดเจน)

Client บอกแค่ "ฉันต้องการ Group G"          Client บอกว่า "ฉันต้องการ Group G จาก Source S เท่านั้น"
Router ต้องเปิดรับ Source ใดก็ได้           Router จำกัด State ไว้เฉพาะ Source ที่ระบุ
= (*,G) เปิดกว้าง เสี่ยงมี Source            = ลด Attack Surface (Source ปลอมที่ไม่ใช่ S
  ปลอมส่ง Traffic เข้า Group เดียวกันได้        ส่ง Traffic มาที่ Group เดียวกันจะถูก Filter
  โดยไม่มีการกรอง (Source Spoofing)             ออกตั้งแต่ Last-Hop Router เลย)
```

**MLDv2** คือ IGMPv3 เวอร์ชัน IPv6 (ทำหน้าที่เดียวกันทุกประการ แค่เปลี่ยน Transport เป็น ICMPv6
แทน IGMP ตรงตาม Dual-Stack Design ที่หลักสูตรนี้วางไว้ตั้งแต่ [00-ip-address-plan.md](00-ip-address-plan.md))
— Concept การ Scale เหมือนกันทุกประการ ต่างกันแค่ Syntax คำสั่ง (`ip igmp` → `ipv6 mld`)

### 747.3 `ip igmp limit` — จำกัดจำนวน State ต่อ Interface/Router

```
! ===== จำกัดจำนวน (*,G)/(S,G) ต่อ Interface (ระดับ VLAN Receiver) =====
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ip igmp limit 100
! Interface นี้ (VLAN 30 SERVERS) จะไม่ยอมรับ IGMP State เกิน 100 Entry
! ถ้าเกิน Report ใหม่ที่เกิน Limit จะถูก Drop และมี Syslog แจ้งเตือน

! ===== จำกัดจำนวน State รวมทั้ง Router (Global) =====
DIST-SW3(config)# ip igmp limit 500
! รวมทุก Interface บน DIST-SW3 ห้ามเกิน 500 State — ป้องกัน Memory/CPU Exhaustion ระดับ Router ทั้งตัว
```

| ระดับ Config | ผลกระทบ | ใช้เมื่อไหร่ |
|---|---|---|
| Interface-level (`interface` mode) | จำกัดต่อ Segment เดียว (เช่น 1 VLAN) | ป้องกัน Endpoint กลุ่มเดียว (เช่น 1 อาคาร) Join เกินจริง โดยไม่กระทบอาคารอื่น |
| Global-level (`config` mode) | จำกัดรวมทั้งอุปกรณ์ทุก Interface | ป้องกัน Memory/CPU Exhaustion ของ Router ทั้งตัวจาก IGMP State รวม ไม่ว่าจะมาจาก Interface ไหน |

### 747.4 Verification: ตรวจว่า Limit ทำงานจริงและใครถูก Drop

```
DIST-SW3# show ip igmp interface Vlan30 | include limit
IGMP interface has limit 100, currently active joins: 87

! เมื่อมี Endpoint ตัวใหม่พยายาม Join Group ตัวที่ 101 บน VLAN นี้
DIST-SW3#
%IGMP-4-IGMP_GROUP_LIMIT: Group limit exceeded for group 239.99.1.1 on interface Vlan30

DIST-SW3# show ip igmp groups vlan30 count
IGMP Connected Group Membership Count: 87 (Limit: 100)
```

> **จุดสำคัญสำหรับ CCIE-level Design**: `ip igmp limit` เป็นเครื่องมือ **Defense-in-Depth** ไม่ใช่
> ตัวแทนของ Multicast Boundary (Step 744) — Boundary กรองตาม **Group Address ว่าไม่ควรข้ามที่ไหน**
> ส่วน IGMP Limit กรองตาม **จำนวน State ทั้งหมดไม่ว่าจะเป็น Group ใด** ใช้คู่กันเสมอในการ Design
> ระดับ Enterprise ที่ต้องทั้งควบคุมทิศทาง Traffic (Boundary) และปริมาณ State (Limit) พร้อมกัน

---

## Step 748 — Multicast High Availability ระดับ Scale

### 748.1 ทวนกลไก Anycast RP Failure จาก Part 42/65 — และขยายไปหลาย Site

[Part 42 Step 415](part-042-multicast-advanced.md#step-415--anycast-rp-rfc-4610) สอน Anycast RP
คู่เดียว (CORE-SW1 `1.1.1.1` + CORE-SW2 `1.1.1.2` ใช้ Anycast `1.1.1.100`) — เมื่อ Design ขยายไป
มี **Regional Data Center** ที่มี Anycast RP คู่ของตัวเอง (สมมติ `DC-CORE-1`/`DC-CORE-2` ใช้ Anycast
`2.2.2.100`) จะเกิดคำถามใหม่: **"ถ้า Receiver ที่ Regional DC ต้องดู Source ที่อยู่ที่ Campus HQ จะ
เกิดอะไรขึ้น"** — คำตอบเหมือน [Part 42 Step 415-416](part-042-multicast-advanced.md#step-416--msdp-multicast-source-discovery-protocol-เจาะลึก)
ที่อธิบาย MSDP Inter-domain Use Case ไว้แล้ว: **ต้องมี MSDP Peer ระหว่าง RP ของทั้งสอง Site**

```
              Campus HQ (Anycast RP 1.1.1.100)         Regional DC (Anycast RP 2.2.2.100)
                 CORE-SW1 (1.1.1.1) ◄──┐                    DC-CORE-1 (2.2.2.1) ◄──┐
                 CORE-SW2 (1.1.1.2) ◄──┼── MSDP ภายใน Site   DC-CORE-2 (2.2.2.2) ◄──┼── MSDP ภายใน Site
                                       │  (Part 42 เดิม)                            │  (แบบเดียวกัน)
                                       │                                            │
                                       └──────────── MSDP ข้าม Site ─────────────────┘
                                         (CORE-SW1 ↔ DC-CORE-1, CORE-SW2 ↔ DC-CORE-2
                                          — ต้องมี Peer มากกว่า 1 คู่เพื่อ Redundancy)
```

### 748.2 ปัญหาใหม่ที่เกิดตอนมี MSDP Peer มากกว่า 2 ตัว: SA Message Loop

[Part 42 Step 416](part-042-multicast-advanced.md#step-416--msdp-multicast-source-discovery-protocol-เจาะลึก)
เตือนไว้ว่า SA Message ต้องผ่าน **Peer-RPF Check** ก่อนถูก Forward ต่อ — แต่ตอนมี MSDP Peer แค่คู่
เดียว (Point-to-point) กฎข้อ 1 (ผ่านเสมอ) ใช้ได้ตรงไปตรงมา — พอ Design ขยายเป็น **Full-mesh MSDP**
ระหว่าง 4 RP (`1.1.1.1`, `1.1.1.2`, `2.2.2.1`, `2.2.2.2` เพื่อ Redundancy เต็มรูปแบบทุกคู่) ปัญหาใหม่
เกิดขึ้น: **SA Message ตัวเดียวถูก Re-flood ไปมาระหว่าง Peer ที่เห็นกันซ้ำหลายเส้นทาง** สิ้นเปลือง
Bandwidth/CPU โดยไม่จำเป็น (แม้ Peer-RPF Check จะป้องกัน Infinite Loop จริงได้ แต่ Redundant SA
Flooding ระหว่าง Full-mesh ก็ยังเป็นภาระที่ไม่ควรมี)

```
Full-mesh MSDP 4 Peer (ไม่มี Mesh-group) — SA Message ถูก Re-flood ซ้ำซ้อน

  1.1.1.1 ──SA──► 1.1.1.2 ──SA (re-flood)──► 2.2.2.1 ──SA (re-flood)──► 2.2.2.2
     │                                                                     │
     └──────────────────── SA (re-flood ทางตรงอีกเส้น) ─────────────────────┘
  (2.2.2.2 ได้รับ SA เดิมซ้ำ 2 ครั้งจาก 2 เส้นทาง — Bandwidth/CPU เสียเปล่าโดยไม่จำเป็น
   แม้ Peer-RPF จะยัง Filter ไม่ให้เกิด Infinite Loop จริงก็ตาม)
```

### 748.3 `ip msdp mesh-group` — แก้ปัญหา Redundant Flooding ใน Full-mesh

```
Router(config)# ip msdp mesh-group <group-name> <peer-address>
```

**หลักการ**: ถ้า Peer ทุกตัวใน "กลุ่ม" เดียวกัน **Peer กันแบบ Full-mesh กันเองอยู่แล้ว** (ทุกตัวเห็น
ทุกตัวโดยตรง) Router ที่ได้รับ SA Message จาก Peer ตัวหนึ่งใน Mesh-group **จะไม่ Re-flood SA
Message นั้นกลับไปยัง Peer ตัวอื่นใน Mesh-group เดียวกันอีก** (เพราะรู้แน่ชัดว่า Peer อื่นในกลุ่ม
เดียวกันได้รับ SA ตัวนี้ตรงจาก Originator อยู่แล้วโดยไม่ต้อง Re-flood ผ่านตัวกลาง)

### 748.4 Config เต็มรูปแบบ: MSDP Mesh-group ระหว่าง Campus HQ และ Regional DC

```
! ===== CORE-SW1 (Campus HQ) =====
CORE-SW1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
CORE-SW1(config)# ip msdp mesh-group HQ-DC-MESH 1.1.1.2
CORE-SW1(config)# ip msdp peer 2.2.2.1 connect-source Loopback0
CORE-SW1(config)# ip msdp mesh-group HQ-DC-MESH 2.2.2.1
CORE-SW1(config)# ip msdp peer 2.2.2.2 connect-source Loopback0
CORE-SW1(config)# ip msdp mesh-group HQ-DC-MESH 2.2.2.2

! ===== CORE-SW2 (Campus HQ) =====
CORE-SW2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
CORE-SW2(config)# ip msdp mesh-group HQ-DC-MESH 1.1.1.1
CORE-SW2(config)# ip msdp peer 2.2.2.1 connect-source Loopback0
CORE-SW2(config)# ip msdp mesh-group HQ-DC-MESH 2.2.2.1
CORE-SW2(config)# ip msdp peer 2.2.2.2 connect-source Loopback0
CORE-SW2(config)# ip msdp mesh-group HQ-DC-MESH 2.2.2.2

! ===== DC-CORE-1 (Regional DC) =====
DC-CORE-1(config)# ip msdp peer 2.2.2.2 connect-source Loopback0
DC-CORE-1(config)# ip msdp mesh-group HQ-DC-MESH 2.2.2.2
DC-CORE-1(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
DC-CORE-1(config)# ip msdp mesh-group HQ-DC-MESH 1.1.1.1
DC-CORE-1(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
DC-CORE-1(config)# ip msdp mesh-group HQ-DC-MESH 1.1.1.2

! ===== DC-CORE-2 (Regional DC) =====
DC-CORE-2(config)# ip msdp peer 2.2.2.1 connect-source Loopback0
DC-CORE-2(config)# ip msdp mesh-group HQ-DC-MESH 2.2.2.1
DC-CORE-2(config)# ip msdp peer 1.1.1.1 connect-source Loopback0
DC-CORE-2(config)# ip msdp mesh-group HQ-DC-MESH 1.1.1.1
DC-CORE-2(config)# ip msdp peer 1.1.1.2 connect-source Loopback0
DC-CORE-2(config)# ip msdp mesh-group HQ-DC-MESH 1.1.1.2
```

> **ข้อกำหนดสำคัญของ Mesh-group ที่มักออกข้อสอบ**: `mesh-group` ใช้ได้ก็ต่อเมื่อ **Peer ทุกตัวใน
> กลุ่มเดียวกัน Peer กันแบบ Full-mesh จริง (ทุกตัวคุยกับทุกตัวโดยตรง)** — ถ้ามี Router ตัวใดใน
> Mesh-group **ไม่ได้ Peer กับสมาชิกอื่นครบทุกตัว** (Partial-mesh) SA Message บางเส้นทางจะไม่ถูก
> Forward เลย เพราะ Router จะเข้าใจผิดว่า "สมาชิกกลุ่มเดียวกันได้รับ SA นี้แล้วอยู่ดี" ทั้งที่จริง
> Router ปลายทางไม่ได้ Peer กับ Originator ตรงเลย — **Mesh-group ไม่ใช่ Feature ที่ใส่ได้ตามใจ
> ต้องมั่นใจว่า Full-mesh จริงก่อนใช้เสมอ**

### 748.5 Verification

```
CORE-SW1# show ip msdp mesh-group
MSDP Mesh-groups
  HQ-DC-MESH: 1.1.1.2 2.2.2.1 2.2.2.2

CORE-SW1# show ip msdp summary
Peer Address    AS    State   Uptime/   Reset  SA     Peer Name
                              Downtime  Count   Count
1.1.1.2          -     Up      02:15:03  1       12     CORE-SW2 (mesh-group HQ-DC-MESH)
2.2.2.1          -     Up      00:45:20  2       12     DC-CORE-1 (mesh-group HQ-DC-MESH)
2.2.2.2          -     Up      00:45:18  2       12     DC-CORE-2 (mesh-group HQ-DC-MESH)
```

**SA Count เท่ากันทั้ง 3 Peer** (12 ในตัวอย่าง) ยืนยันว่าทุก Source ที่ทั้ง 4 RP รู้จักถูก Sync กันครบ
โดยไม่มี Redundant Flood ปนอยู่ — ถ้า RP ตัวใดตัวหนึ่งใน Site ล่ม (เช่น CORE-SW1) ตัวที่เหลือ
(CORE-SW2) ยังมี SA Cache ครบจาก MSDP Sync เดิม และ Anycast (OSPF) จะ Reroute Register/Join
ไปทาง CORE-SW2 อัตโนมัติทันที — Availability ยังคงอยู่แม้ RP ตัวหนึ่งใน Site ล่มไปจริง

---

## Step 749 — Multicast Troubleshooting ระดับ Scale

### 749.1 ปัญหาของ Methodology เดิมจาก Part 65 ที่ Scale ระดับ Enterprise

[Part 65 Step 641](part-065-multicast-troubleshooting.md#step-641--multicast-troubleshooting-methodology)
สอน Methodology ไล่ Layer ทีละ Group (`show ip mroute <group>`) ซึ่งยังใช้ได้ถูกต้องทุกประการที่
Scale นี้ — แต่คำสั่งที่ระบุ **1 Group เฉพาะเจาะจง** ไม่มีประสิทธิภาพอีกต่อไปเมื่อต้อง Monitor
**250+ Group พร้อมกัน** — วิศวกร CCIE ต้องมีเครื่องมือระดับ "มองทั้ง Domain พร้อมกัน" ก่อนจะเจาะไป
ที่ Group เดียว

### 749.2 `show ip mroute summary` — ภาพรวมทั้ง Domain ในหน้าจอเดียว

```
CORE-SW1# show ip mroute summary
IP Multicast Routing Table
Total number of routes: 254
Total number of (*,G) routes: 127
Total number of (S,G) routes: 125
Total number of (S,G) RPF-failed routes: 0
Total number of (*,G-Ineligible) routes: 2

(*,239.1.1.1), 00:2:15:33/stopped, RP 1.1.1.100, OIF count: 8, flags: SJC
(10.10.30.30,239.1.1.1), 00:2:15:20/00:02:58, flags: JT
(*,239.50.1.1), 00:45:12/stopped, RP 1.1.1.100, OIF count: 1, flags: SJCF
...
```

**สิ่งที่ต้องเช็คก่อนเสมอในภาพรวม**: `Total number of (S,G) RPF-failed routes` — ถ้าค่านี้ไม่เป็น 0
แปลว่ามี Group อย่างน้อย 1 ตัวที่ RPF Fail ตามปัญหาแบบ [Part 65 Step 643](part-065-multicast-troubleshooting.md#step-643--scenario-2-rpf-failure-จาก-static-route-ทดสอบที่ลืมถอด)
— ไม่ต้องไล่ดูทีละ Group 250 ตัว แค่เห็นค่านี้ผิดปกติก็รู้ทันทีว่ามีปัญหาอยู่ที่ไหนสักที่ ต้องไปหาต่อว่า
Group ไหน

### 749.3 `show ip mroute count` — ดู Traffic Rate จริงต่อ Group แยกจาก State

```
CORE-SW1# show ip mroute count
IP Multicast Statistics
254 routes using 178192 bytes of memory
127 groups, 0.98 average sources per group
Forwarding Counts: Pkt Count/Pkts per second/Avg Pkt Size/Kilobits per second
Other counts: Total/RPF failed/Other drops(OIF-null, rate-limit etc)

Group: 239.1.1.1, Source count: 1, Group pkt count: 8452103
  Source: 10.10.30.30/32, Forwarding Counts: 8452103/1420/1350/1533, Other counts: 8452103/0/0
Group: 239.70.3.5, Source count: 1, Group pkt count: 452019
  Source: 10.10.30.15/32, Forwarding Counts: 452019/25/800/0, Other counts: 452019/0/12044
                                    <-- "Other drops" 12044 ตัว ผิดปกติ! (OIF-null หรือ rate-limit)
```

คอลัมน์ **`Other counts: Total/RPF failed/Other drops`** คือจุดที่ Scale บอกอะไรได้เยอะกว่า Group
เดียว — เห็น Group `239.70.3.5` มี **Other drops 12044** ในขณะที่ Group อื่นเป็น 0 หมด บอกได้ทันที
ว่า Group นี้มีปัญหาเฉพาะตัว (เช่น OIL ไม่มี Interface ที่ Active จริง หรือโดน Rate-limit) โดยไม่ต้อง
ไล่ดู 254 Group ทีละตัว — **อ่าน `show ip mroute count` ทั้งหน้าแล้วหา Anomaly เป็นหลัก ไม่ใช่ไล่
ทีละบรรทัดจากบนลงล่าง**

### 749.4 Filter ตาม Group Range ด้วย ACL — เจาะเฉพาะ "กลุ่มปัญหา" ที่สงสัย

```
! สงสัยว่า Security Camera Feed ของอาคาร 3 (239.70.3.x) มีปัญหา — Filter ดูเฉพาะ Range นี้
CORE-SW1# show ip mroute count 239.70.3.0 0.0.0.255
Group: 239.70.3.1, Source count: 1, Group pkt count: 452019
Group: 239.70.3.5, Source count: 1, Group pkt count: 452019, Other counts drop: 12044
Group: 239.70.3.9, Source count: 1, Group pkt count: 0                    <-- Group นี้ไม่มี Traffic เลย! ผิดปกติ

! หรือใช้ show ip mroute เฉยๆ กับ ACL แทนการพิมพ์ Group เดี่ยว
CORE-SW1# access-list 99 permit 239.70.3.0 0.0.0.255
CORE-SW1# show ip mroute list 99
```

### 749.5 ตารางสรุปคำสั่ง Scale-aware Troubleshooting

| คำสั่ง | ใช้ตอบคำถาม | ข้อดีที่ Scale |
|---|---|---|
| `show ip mroute summary` | "มีปัญหาอยู่ที่ไหนในทั้ง Domain หรือไม่" | เห็น RPF-failed Count รวมในบรรทัดเดียว ไม่ต้องไล่ทีละ Group |
| `show ip mroute count` | "Group ไหนมี Drop/Rate ผิดปกติ" | เห็น Forwarding Rate + Drop Count ของทุก Group พร้อมกันในหน้าจอเดียว |
| `show ip mroute count <group> <wildcard>` / `show ip mroute list <acl>` | "เจาะเฉพาะ Range ที่สงสัย" | ลด Noise จาก Group อื่นที่ไม่เกี่ยวข้อง เจาะตรงจุดโดยไม่ต้อง Scroll ผ่าน 250+ Group |
| `show ip msdp summary` (ถ้ามี Anycast RP หลาย Site) | "RP ทุกตัวเห็น SA Count ตรงกันหรือไม่" | เทียบ SA Count ระหว่าง Peer ในบรรทัดเดียว — ไม่ตรงกัน = MSDP Sync มีปัญหา (ตรงกับ [Part 65 Step 645](part-065-multicast-troubleshooting.md#step-645--scenario-4-msdp-peer-ใช้-anycast-address-ผิดพลาดแทน-loopback0-unique)) |
| `show ip pim rp mapping` เทียบทุก Router | "RP Mapping ตรงกันทั้ง Domain หรือไม่" | ยังจำเป็นเหมือนเดิมตาม [Part 65 Step 644](part-065-multicast-troubleshooting.md#step-644--scenario-3-rp-mapping-ไม่ตรงกันหลัง-migrate-ไป-anycast-rp-ไม่ครบทุก-router) แม้ Scale ใหญ่ขึ้น — Automate ด้วย Script/Ansible เมื่อจำนวน Router มากกว่า 10-20 ตัว (ตาม [Part 48](part-048-automation-ansible-advanced.md)) |

> **กฎเหล็กสำหรับ Scale**: เริ่มจาก "ภาพรวม" (`summary`/`count` ไม่ระบุ Group) เสมอ **ก่อน** จะ
> เจาะไปที่ Group เดียว — สลับลำดับ (เจาะ Group เดียวก่อนเห็นภาพรวม) จะเสียเวลามากเมื่อมี Group
> เป็นร้อย เพราะไม่รู้ว่าควรเริ่มเจาะจาก Group ไหนก่อน

---

## Step 750 — Lab เต็มรูปแบบ: Enterprise Multicast Design ทั้ง Campus 10 อาคาร + Regional DC + Branch

### 750.1 สรุป Topology ของ Lab นี้

```
                         ┌───────────────────────────────────────────────────────┐
                         │              CAMPUS HQ (Building 1-10, Part 72)         │
                         │                                                         │
                         │   CORE-SW1 (1.1.1.1)◄──MSDP──►CORE-SW2 (1.1.1.2)         │
                         │        Anycast RP: 1.1.1.100 (Lo1, OSPF Area 0)         │
                         │        Mesh-group: HQ-DC-MESH                           │
                         │            │                    │                      │
                         │      DIST-SW1-4 (เดิม)     DIST-SW5-10 (Part 72 ใหม่)    │
                         │      All-Hands, Market-Data  Signage/Training/Camera    │
                         └───────────────────┬─────────────────────────────────────┘
                                              │ Port-channel2 (ip multicast boundary
                                              │  MCAST-WAN-BOUNDARY filter-autorp — Step 744)
                              ┌───────────────┴────────────────┐
                        ┌─────┴─────┐                    ┌─────┴─────┐
                        │ WAN-EDGE-1 │                    │ WAN-EDGE-2 │
                        │ (Tunnel0   │◄──── DMVPN Cloud ─►│  Backup    │
                        │  Hub, PIM  │      NHRP + PIM     │  Hub       │
                        │  nbma-mode)│      nbma-mode      │            │
                        └─────┬─────┘                    └───────────┘
                              │                                   │
                    ┌─────────┴─────────┐               ┌─────────┴─────────┐
              ┌─────┴─────┐       ┌─────┴─────┐   ┌─────┴─────┐
              │BRANCH-RTR │       │Regional DC │   │BRANCH2-RTR│
              │ (Spoke)   │       │DC-CORE-1/2  │   │ (Spoke)   │
              │Training   │       │Anycast RP:  │   │Training   │
              │Video      │       │2.2.2.100    │   │Video      │
              │Receiver   │       │Mesh-group:  │   │Receiver   │
              └───────────┘       │HQ-DC-MESH   │   └───────────┘
                                  └────────────┘
```

### 750.2 Checklist การ Deploy (ลำดับที่แนะนำสำหรับ CCIE-level Rollout)

```
[1] Anycast RP คู่ Campus HQ (CORE-SW1/CORE-SW2 — ต่อจาก Part 42 เดิม ยังใช้ 1.1.1.100)
[2] Anycast RP คู่ Regional DC (DC-CORE-1/DC-CORE-2 — ใหม่ ใช้ 2.2.2.100)
[3] MSDP Mesh-group HQ-DC-MESH ครบทั้ง 4 RP (Step 748)
[4] Multicast Boundary บน WAN-facing Interface ทั้ง CORE-SW1/CORE-SW2 (Step 744)
[5] ip pim nbma-mode บน Tunnel Interface ของ WAN-EDGE-1/WAN-EDGE-2 (Step 746)
[6] IGMP Limit บน Access-facing Interface ทุก DIST-SW/BRANCH-RTR (Step 747)
[7] QoS Marking/Queuing สำหรับ Multicast Video บน DIST-SW ต้นทางและ WAN-EDGE ปลายทาง (Step 745)
[8] Rollout ให้ครบทุก Router ในครั้งเดียว (ตามคำเตือนของ Part 65 Step 644 — ห้าม Migrate ไม่ครบ)
```

### 750.3 Config ส่วนที่ยังไม่เคยลงใน Step ก่อนหน้า: Regional DC Anycast RP (ใหม่)

```
! ===== DC-CORE-1 =====
DC-CORE-1(config)# ip multicast-routing
DC-CORE-1(config)# interface Loopback0
DC-CORE-1(config-if)# ip address 2.2.2.1 255.255.255.255
DC-CORE-1(config-if)# ip pim sparse-mode
DC-CORE-1(config-if)# exit
DC-CORE-1(config)# interface Loopback1
DC-CORE-1(config-if)# ip address 2.2.2.100 255.255.255.255
DC-CORE-1(config-if)# ip pim sparse-mode
DC-CORE-1(config-if)# exit
DC-CORE-1(config)# router ospf 1
DC-CORE-1(config-router)# network 2.2.2.1 0.0.0.0 area 0
DC-CORE-1(config-router)# network 2.2.2.100 0.0.0.0 area 0
DC-CORE-1(config-router)# exit
DC-CORE-1(config)# ip pim rp-address 2.2.2.100 10

! ===== DC-CORE-2 (ทำแบบเดียวกัน, IP ตัวเอง = 2.2.2.2) =====
DC-CORE-2(config)# ip multicast-routing
DC-CORE-2(config)# interface Loopback0
DC-CORE-2(config-if)# ip address 2.2.2.2 255.255.255.255
DC-CORE-2(config-if)# ip pim sparse-mode
DC-CORE-2(config-if)# exit
DC-CORE-2(config)# interface Loopback1
DC-CORE-2(config-if)# ip address 2.2.2.100 255.255.255.255
DC-CORE-2(config-if)# ip pim sparse-mode
DC-CORE-2(config-if)# exit
DC-CORE-2(config)# router ospf 1
DC-CORE-2(config-router)# network 2.2.2.2 0.0.0.0 area 0
DC-CORE-2(config-router)# network 2.2.2.100 0.0.0.0 area 0
DC-CORE-2(config-router)# exit
DC-CORE-2(config)# ip pim rp-address 2.2.2.100 10
```

### 750.4 Config: DMVPN Hub ทั้งสองตัวเปิด `ip pim nbma-mode` + RP ของฝั่งตัวเองสำหรับ Group ที่ต้อง Local

```
WAN-EDGE-1(config)# ip multicast-routing
WAN-EDGE-1(config)# ip pim rp-address 1.1.1.100 10
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip pim sparse-mode
WAN-EDGE-1(config-if)# ip pim nbma-mode
WAN-EDGE-1(config-if)# ip igmp limit 300
WAN-EDGE-1(config-if)# exit

WAN-EDGE-2(config)# ip multicast-routing
WAN-EDGE-2(config)# ip pim rp-address 1.1.1.100 10
WAN-EDGE-2(config)# interface Tunnel0
WAN-EDGE-2(config-if)# ip pim sparse-mode
WAN-EDGE-2(config-if)# ip pim nbma-mode
WAN-EDGE-2(config-if)# ip igmp limit 300
WAN-EDGE-2(config-if)# exit
```

### 750.5 Config: BRANCH-RTR (Spoke) รับ Training Room Video เท่านั้น (Group ที่ Boundary อนุญาต)

```
BRANCH-RTR(config)# ip multicast-routing
BRANCH-RTR(config)# ip pim rp-address 1.1.1.100 10
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# ip pim sparse-mode
BRANCH-RTR(config-if)# exit
BRANCH-RTR(config)# interface GigabitEthernet0/1
BRANCH-RTR(config-if)# description ** LAN ฝั่ง Branch — Video Wall Receiver **
BRANCH-RTR(config-if)# ip pim sparse-mode
BRANCH-RTR(config-if)# ip igmp limit 50
BRANCH-RTR(config-if)# ip igmp version 3
```

### 750.6 Verification เต็มรูปแบบ — ไล่ตาม Checklist

```
! [1]-[3] Anycast RP + MSDP Mesh-group ครบทุก Site
CORE-SW1# show ip msdp mesh-group
MSDP Mesh-groups
  HQ-DC-MESH: 1.1.1.2 2.2.2.1 2.2.2.2

! [4] Multicast Boundary กันช่วง Signage/Camera ไม่ให้ข้าม WAN
CORE-SW1# show ip mroute 239.70.3.5
Group 239.70.3.5 not found

! [5] Multicast over DMVPN — Spoke เห็น Hub เป็น PIM Neighbor ผ่าน Tunnel0
BRANCH-RTR# show ip pim neighbor
Neighbor Address    Interface    Uptime/Expires  Ver   DR Prio/Mode
172.16.201.1          Tunnel0      01:20:44/00:01:35  v2   1 / DR S P G

! All-Hands Video ไปถึง Branch จริง (ข้าม Boundary ที่อนุญาตไว้)
BRANCH-RTR# show ip mroute 239.1.1.1
(*, 239.1.1.1), 00:05:12/00:02:47, RP 1.1.1.100, flags: SJC
  Incoming interface: Tunnel0, RPF nbr 172.16.201.1
  Outgoing interface list:
    GigabitEthernet0/1, Forward/Sparse, 00:05:12/00:02:47

! Training Room Video ข้าม WAN ถึง Regional DC ด้วย MSDP + Anycast RP คนละคู่
DC-CORE-1# show ip msdp sa-cache 239.60.5.2
MSDP Source-Active Cache - 1 entries
(10.10.60.5, 239.60.5.2), RP 1.1.1.1, MBGP/AS -, 00:12:40/00:05:18, Peer 1.1.1.1
                                    <-- DC-CORE-1 รู้จัก Source ที่ Campus HQ ผ่าน MSDP Mesh-group

! [6] IGMP Limit ทำงานจริง
DIST-SW3# show ip igmp interface Vlan30 | include limit
IGMP interface has limit 100, currently active joins: 92

! [7] QoS Marking ผ่านจริง
WAN-EDGE-1# show policy-map interface Tunnel0 | section MULTICAST-VIDEO
  Class-map: MULTICAST-VIDEO (match-all)
    24857802 packets, 2980936240 bytes
    Queueing
    queue limit 64 packets
    (queue depth/total drops/no-buffer drops) 0/0/0
    bandwidth 25% (25000 kbps)

! [8] show ip mroute summary — ยืนยันภาพรวมทั้ง Domain สะอาด ไม่มี RPF-failed
CORE-SW1# show ip mroute summary
Total number of routes: 254
Total number of (S,G) RPF-failed routes: 0
```

Design นี้ตอบครบทุกปัญหาที่ตั้งไว้ตอนต้น Part: **Scale** (250+ Group ผ่าน Anycast RP กระจาย Load,
IGMP Limit กันไม่ล้น), **Availability** (Anycast RP + MSDP Mesh-group ทนต่อ RP ตัวใดตัวหนึ่งล่ม),
**Security/Efficiency** (Multicast Boundary กัน Traffic ไม่จำเป็นไม่ให้ข้าม WAN), **DMVPN
Compatibility** (`ip pim nbma-mode` ให้ Multicast ทำงานถูกต้องบน NBMA Fabric), และ **QoS** (Video
Stream ได้ Bandwidth การันตีโดยไม่แย่ง Voice/Data อื่น) — ครบทุกมุมตามที่ระดับ CCIE Enterprise
Infrastructure ต้องออกแบบได้จริง

---

## Quiz ทบทวน Part 75

**Q1.** เมื่อ Campus ขยายจาก 4 อาคารเป็น 10 อาคาร ปัจจัยหลักที่ทำให้ (S,G)/(*,G) State บน Core
Router เพิ่มขึ้นคือปัจจัยใด — และปัจจัยใดที่ **ไม่** ทำให้ State เพิ่มขึ้นตรงๆ

**Q2.** Default MDT และ Data MDT ใน MVPN ต่างกันอย่างไร และ Data MDT ถูกสร้างขึ้นเมื่อไหร่

**Q3.** อธิบายว่าทำไม Campus 10 อาคาร (Single Site) เหมาะกับ Anycast RP ภายใน Site เดียว แต่พอมี
Regional Data Center เพิ่มเข้ามาต้องเพิ่ม MSDP Mesh-group — เกิดจากเหตุผลอะไร

**Q4.** `ip multicast boundary` กับ `ip igmp limit` ต่างกันอย่างไร ใช้แก้ปัญหาคนละแบบอย่างไร

**Q5.** ทำไม `ip pim nbma-mode` จึงจำเป็นบน Hub ของ DMVPN Phase 1 มากกว่า Phase 3 ที่มี NHRP
Redirect/Shortcut — และทำไม Best Practice ยังแนะนำให้ Multicast วิ่งผ่าน Hub เสมอแม้จะเป็น Phase 3

---

## เฉลย Quiz

**A1.** ปัจจัยหลักคือ **จำนวน Group ที่ Active (ไม่ซ้ำ) คูณกับจำนวน Source ต่อ Group** (โดยเฉพาะ
Group ที่ Switchover ไป SPT แล้วซึ่งเพิ่ม (S,G) แยกจาก (*,G)) — การเพิ่มอาคารทำให้ Group ใหม่เกิด
ขึ้นจริง (Signage/Training/Camera ต่ออาคาร) จึงเพิ่ม State ตรงๆ ส่วน **จำนวน Receiver ต่อ Group
ไม่ทำให้ State เพิ่ม** เพราะ Receiver เพิ่มแค่ทำให้ OIL (Outgoing Interface List) ของ Entry เดิมยาว
ขึ้น ไม่ได้สร้าง Entry ใหม่

**A2.** **Default MDT** คือ Provider Multicast Group ที่ทุก PE ของ VRF เดียวกัน Join ร่วมกันเสมอ
(Static, มีตลอด) ใช้ Carry Control-plane และ Traffic Rate ต่ำของ Customer ข้าม P-Network — **Data
MDT** ถูกสร้างขึ้น **เมื่อ Customer (S,G) มี Traffic Rate เกิน Threshold ที่ Config ไว้** เพื่อให้
เฉพาะ PE ที่มี Receiver จริงของ Flow นั้น Join Group ใหม่ (แทนที่จะ Flood ผ่าน Default MDT ไปถึง
ทุก PE โดยไม่จำเป็น) — เหมือน SPT-Switchover ของ PIM-SM ปกติแต่ทำในระดับ Provider Tree

**A3.** Campus 10 อาคารเป็น **Single Site** — RP สองตัว (CORE-SW1/CORE-SW2) เข้าถึงกันผ่าน OSPF
ภายใน Site เดียวได้ทันที, MSDP Peer แค่คู่เดียวก็พอ (Peer-RPF Check กฎข้อ 1 "Peer เดียวผ่านเสมอ"
ตาม Part 42 Step 416) — พอเพิ่ม Regional DC ที่มี Anycast RP คู่ของตัวเอง ทำให้มี **RP ทั้งหมด 4
ตัวใน 2 Domain ที่ต้อง Sync กันครบทุกคู่** (เพื่อ Redundancy เต็มรูปแบบ) — ถ้า Peer กันแบบ Full-mesh
ตรงๆ โดยไม่มี Mesh-group จะเกิด SA Message Re-flood ซ้ำซ้อนไม่จำเป็น — Mesh-group จึงจำเป็นเมื่อ
จำนวน MSDP Peer มากกว่า 2 ตัวและ Peer กันแบบ Full-mesh

**A4.** `ip multicast boundary` กรองตาม **Group Address** (ควบคุมว่า Group ไหนข้าม Interface/
Administrative Boundary นี้ได้หรือไม่ — ปัญหาด้าน Traffic Direction/Security Scope) ส่วน
`ip igmp limit` กรองตาม **จำนวน State รวม** ไม่ว่าจะเป็น Group ใด (ปัญหาด้าน Resource Exhaustion/
Denial-of-Service) — ใช้คู่กันเสมอในการ Design: Boundary ควบคุมทิศทาง, Limit ควบคุมปริมาณ

**A5.** DMVPN Phase 1 บังคับให้ Traffic ทุกอย่างผ่าน Hub เสมอ (ไม่มี Spoke-to-Spoke Direct Tunnel
เลย) — Hub จึงต้อง Replicate Multicast เป็น Unicast แยกไปทุก Spoke อยู่แล้วโดย Design ทำให้
`ip pim nbma-mode` เข้ากับพฤติกรรมธรรมชาติของ Phase 1 พอดี — ส่วน Phase 3 มี NHRP Redirect/
Shortcut แต่ Feature นี้ Design มาสำหรับ **Unicast Data Traffic** เท่านั้น ไม่มี Vendor ไหน
Implement Spoke-to-Spoke Multicast Shortcut จริงจัง — Best Practice จึงยังให้ Multicast วิ่งผ่าน
Hub เสมอไม่ว่า Phase ใด เพราะ Hub ทำหน้าที่เป็น Rendezvous Point ตามธรรมชาติของ PIM-SM Shared
Tree อยู่แล้ว และการฝืนทำ Direct Multicast ระหว่าง Spoke ไม่คุ้มกับความซับซ้อนที่เพิ่มขึ้น

---

## สรุป Part 75

Part นี้ยก Multicast จากระดับ CCNP ENCOR/ENARSI ([Part 41](part-041-multicast-fundamentals.md),
[Part 42](part-042-multicast-advanced.md), [Part 65](part-065-multicast-troubleshooting.md)) ไปสู่
ระดับ **CCIE Enterprise Infrastructure** เต็มรูปแบบ — เริ่มจากทำความเข้าใจว่า **State Scaling
Problem คือ Group × Source ไม่ใช่ Receiver** (Step 741), เชื่อมแนวคิดไปยัง **MVPN/Default MDT/Data
MDT** สำหรับอนาคตที่ต้องข้าม VRF-Lite/MPLS VPN Core (Step 742, เชื่อมกับ [Part 51](part-051-vrf-lite-gre-ipsec.md)
และ Preview [Part 86-87](part-086-mpls-l3vpn-fundamentals.md)), สร้าง **Decision Matrix เลือก RP
Pattern ตาม Tier ของ Network** (Step 743), ป้องกัน Traffic ที่ไม่จำเป็นด้วย **Multicast Boundary**
(Step 744), การันตี Bandwidth ให้ Video Stream ด้วย **QoS Integration** (Step 745, เชื่อมกับ
[Part 40](part-040-advanced-qos.md)), แก้ปัญหา NBMA ของ **Multicast over DMVPN** (Step 746, เชื่อม
กับ [Part 52-53](part-052-dmvpn-fundamentals.md)), ป้องกัน Resource Exhaustion ด้วย **IGMP
Scaling/Limit** (Step 747), เพิ่ม Availability ระดับ Multi-site ด้วย **MSDP Mesh-group** (Step
748), เร่งความเร็ว Troubleshooting ด้วยคำสั่งระดับ Domain (Step 749) และปิดท้ายด้วย **Lab Design
เต็มรูปแบบของทั้ง Enterprise** ที่รวมทุก Technique เข้าด้วยกัน (Step 750)

จุดที่สำคัญที่สุดที่ต้องพาไปใช้ต่อ: **Multicast Design ระดับ CCIE ไม่ใช่แค่ "ทำให้ PIM ทำงาน" แต่คือ
การ Balance 4 มิติพร้อมกันเสมอ — Scale (State/RP Load), Availability (Anycast RP/MSDP), Security/
Efficiency (Boundary), และ Performance (QoS)** — ทั้ง 4 มิตินี้จะกลับมาอีกครั้งใน Context ใหม่ที่
[Part 76 — SD-WAN Advanced Design](part-076-sdwan-advanced-design.md) จะเรียน ซึ่ง Multicast
over SD-WAN Fabric มีความซับซ้อนเพิ่มขึ้นไปอีกขั้น เพราะ SD-WAN Overlay เปลี่ยน Transport แบบ
Dynamic ตาม Application Policy — ทำให้ต้องคิดเรื่อง RPF และ Boundary ใหม่ในบริบทของ Overlay ที่ไม่
มี Physical Topology ตายตัวเหมือน DMVPN ที่เพิ่งเรียนใน Part นี้
