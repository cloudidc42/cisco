# Part 51 — VRF-Lite, GRE, IPsec VPN
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 501–510 จาก 1000 | ระดับ CCNP ENCOR**

> ต่อจาก [Part 50 — Cisco DNA Center](part-050-dna-center.md) ที่ยกระดับการจัดการเครือข่ายทั้งหมด
> ขึ้นสู่ Intent-Based Networking ผ่าน Controller กลาง Part นี้จะพากลับมาที่ระดับ CLI/Protocol
> เพื่อเจาะลึกสองเรื่องที่ [Part 19 — WAN Technologies](part-019-wan-technologies.md) และ
> [Part 32 — Policy-Based Routing](part-032-policy-based-routing.md) เคยแนะนำไว้แบบผ่านๆ แล้ว
> "ค้างไว้อย่างตั้งใจ": **VRF-Lite** (แยก Routing Table หลายชุดบนอุปกรณ์เดียว โดยไม่ต้องมี MPLS)
> และ **IPsec แบบเต็มรูปแบบ** (ISAKMP/IKE Phase 1-2, Crypto Map, VTI) ที่จะมาปกป้อง GRE Tunnel
> ระหว่าง **WAN-EDGE-1** กับ **BRANCH-RTR** ที่สร้างไว้ตั้งแต่ Part 19 ให้กลายเป็น Site-to-Site
> VPN ที่ปลอดภัยจริงในทางปฏิบัติ — Part นี้คือก้าวสุดท้ายก่อนเข้าสู่ **DMVPN** ใน Part 52-53
> ซึ่งจะนำ GRE-over-IPsec ไปสู่ Hub-and-Spoke แบบ Dynamic เต็มรูปแบบ

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 501 | VRF (Virtual Routing and Forwarding) พื้นฐาน — หลาย Routing Table บนอุปกรณ์เดียว |
| 502 | คอนฟิก VRF-Lite พื้นฐาน — สร้าง GUEST VRF แยก WIFI VLAN 40 ออกจาก Global Table |
| 503 | Routing ภายใน VRF — รัน OSPF Process แยกต่อ VRF |
| 504 | Inter-VRF Communication — Route Leaking และการเปิดทางแบบควบคุมได้ |
| 505 | GRE Tunnel เชิงลึก — Keepalive, PMTUD, และ Redundancy หลาย Path |
| 506 | IPsec พื้นฐานเชิงลึก — IKEv1 vs IKEv2, ISAKMP Phase 1-2 เต็มรูปแบบ |
| 507 | GRE over IPsec — ปกป้อง GRE Tunnel ของ Part 19 ด้วย IKEv2 + Crypto Map |
| 508 | VTI (Virtual Tunnel Interface) — ทางเลือกที่เรียบง่ายกว่า Crypto Map |
| 509 | FlexVPN — แนวคิดเบื้องต้นก่อนเข้าสู่ DMVPN ใน Part 52-53 |
| 510 | Lab เต็มรูปแบบ: GUEST VRF (VLAN 40) + GRE-over-IPsec (VTI-style) WAN-EDGE-1 ↔ BRANCH-RTR |

---

## Step 501 — VRF (Virtual Routing and Forwarding) พื้นฐาน

### 501.1 ปัญหาที่ VRF แก้ไข

ตั้งแต่ [Part 9](part-009-static-routing.md) จนถึง Part ล่าสุด ทุก Router/Switch ในหลักสูตรนี้มี
**Routing Table เดียว (Global Routing Table)** — ทุก Interface, ทุก Static Route, ทุก Dynamic
Routing Protocol (OSPF/EIGRP/BGP) ต่างเติมข้อมูลลง RIB (Routing Information Base) ชุดเดียวกัน
ทั้งหมด นี่คือพฤติกรรม Default ที่ถูกต้องสำหรับ Enterprise ทั่วไปที่ Site เดียวมีเจ้าของเดียว

แต่ในทางปฏิบัติมีสถานการณ์ที่ **ต้องการ Routing Table แยกจากกันโดยสมบูรณ์บนอุปกรณ์ตัวเดียวกัน**:

| สถานการณ์ | เหตุผลที่ต้องแยก Routing Table |
|---|---|
| **Multi-Tenancy** | ผู้เช่าหลายราย (Tenant) ใช้อุปกรณ์เครือข่ายเดียวกัน (Data Center, MSP) — Tenant A ต้องมองไม่เห็น Route ของ Tenant B เลย แม้ IP Subnet จะซ้อนทับกันได้ (Overlapping IP) |
| **Guest/Wi-Fi Isolation** | Traffic ของ Guest Wi-Fi ต้องไม่สามารถเข้าถึง Internal Network ได้เลย แม้ผ่าน Switch ตัวเดียวกันกับ VLAN ภายใน — ต้องการ **Hard Isolation ระดับ Routing** ไม่ใช่แค่ ACL |
| **Compliance / Regulatory Segregation** | ธุรกิจบางประเภท (การเงิน, สุขภาพ) ต้องแยก Network ของระบบที่อยู่ภายใต้ Compliance (PCI-DSS, HIPAA) ออกจาก Network ทั่วไปอย่างพิสูจน์ได้ (Auditable Separation) |
| **Overlapping IP Address** | Merger & Acquisition ที่สอง Site ใช้ Subnet เดียวกันโดยไม่ได้ตั้งใจ — VRF ทำให้ทั้งสอง Subnet อยู่ร่วมกันบนอุปกรณ์เดียวได้โดยไม่ชนกัน |

**ACL (Part 21, 32) แก้ปัญหานี้ได้เพียงบางส่วน** — ACL กรอง Traffic ที่ Layer 3/4 ตาม Rule ที่
ตั้งไว้ แต่ **Routing Table ยังเป็นชุดเดียวกันอยู่ดี** ถ้า Admin ลืมเขียน ACL สักบรรทัด หรือมี
Route ผิดพลาดหลุดเข้ามา Traffic ก็ยัง**มีเส้นทางไปถึงกันได้ในทางเทคนิค** — VRF ให้การ Isolation
ที่แน่นกว่า: **ถ้าไม่มี VRF ไหนรู้จัก Route ของอีก VRF เลย ต่อให้ไม่มี ACL ก็ไปไม่ถึงกันโดย
Default** เพราะ Routing Table แยกกันตั้งแต่ต้น

### 501.2 นิยามของ VRF

**VRF (Virtual Routing and Forwarding)** คือเทคโนโลยีที่ทำให้ Router/Switch ตัวเดียว **รัน
Routing Table (RIB) และ Forwarding Table (FIB) ได้หลายชุดพร้อมกัน โดยแต่ละชุดเป็นอิสระจากกัน
อย่างสมบูรณ์** — เสมือนมี Router เสมือน (Virtual Router) หลายตัวซ้อนอยู่ในอุปกรณ์ Physical
ตัวเดียว — เช่น Global Routing Table เดิม (Interface VLAN10/30, OSPF Process 1) กับ VRF
"GUEST" (Interface VLAN40, Static/OSPF ของ GUEST เท่านั้น) แยกกันสมบูรณ์บนอุปกรณ์เดียวกัน
ไม่มี Route รั่วไหลข้ามกันโดย Default

- แต่ละ VRF มี **RIB ของตัวเอง, FIB ของตัวเอง, และสามารถรัน Routing Protocol Instance ของตัวเอง
  แยกจาก Global Table ได้อย่างสมบูรณ์** (เช่น OSPF Process 1 ใน Global, OSPF Process 2 ใน VRF)
- Interface ใดๆ (Physical, SVI, Sub-interface) **เป็นสมาชิกของ VRF ได้เพียง VRF เดียวเท่านั้น**
  (หรืออยู่ใน Global Table ถ้าไม่ได้กำหนด VRF)
- Traffic ที่เข้ามาทาง Interface ของ VRF หนึ่ง **จะถูก Lookup กับ Routing Table ของ VRF นั้น
  เท่านั้น** — ไม่มีทางมองเห็น Route ของ VRF อื่นหรือ Global Table โดย Default (ต้องเปิดทาง
  ด้วย Route Leaking — Step 504 เท่านั้น)

### 501.3 VRF-Lite เทียบกับ MPLS VRF (Full VRF) — จุดที่ต้องเข้าใจให้ชัดตั้งแต่ต้น

จาก [Part 19 Step 186](part-019-wan-technologies.md) เราเห็นว่า SP (Service Provider) ใช้ VRF
บน PE Router ร่วมกับ MPLS Label Switching และ MP-BGP เพื่อสร้าง L3VPN ให้ลูกค้าหลายราย — นั่นคือ
"VRF" เวอร์ชันเต็มรูปแบบที่ผูกกับ MPLS แต่ VRF ที่เราจะเรียนใน Part นี้ **ไม่เกี่ยวกับ MPLS เลย**
เรียกว่า **VRF-Lite**

| คุณสมบัติ | VRF-Lite | Full/MPLS VRF (SP-grade) |
|---|---|---|
| ต้องมี MPLS หรือไม่ | ❌ ไม่ต้อง — ทำงานบน IP ปกติ | ✅ ต้องมี MPLS Label Switching |
| การแลก Route ระหว่างอุปกรณ์ | ต้องคอนฟิก VRF **ทีละอุปกรณ์ ทีละ Hop** (Hop-by-Hop) ด้วย Routing Protocol ธรรมดาต่อ VRF | ใช้ **MP-BGP ตัวเดียว** แจก VPN Route ข้าม PE ทั้งหมดพร้อม Route-Distinguisher/Route-Target อัตโนมัติ |
| Scale (จำนวน Site/Hop) | เหมาะกับจำนวน Hop น้อย (Campus, Branch เดียว) — เพิ่ม Hop ต้องคอนฟิกซ้ำทุกจุด | เหมาะกับ Backbone ขนาดใหญ่ระดับ SP (หลายสิบ-หลายพัน PE) |
| ความซับซ้อนการคอนฟิก | ต่ำกว่า — ไม่ต้องรู้จัก MPLS/MP-BGP/Route-Distinguisher เชิงลึก | สูงกว่า — ต้องมี LDP/MPLS Label + MP-BGP + RD/RT ครบ |
| ใครใช้งานจริง | Enterprise Campus, Data Center ขนาดกลาง (Multi-Tenancy ภายในองค์กรเดียว) | Service Provider ขาย L3VPN ให้ลูกค้าหลายองค์กร |
| สอนใน Part ไหนของหลักสูตรนี้ | **Part 51 (Part นี้)** | **Part 86-87** (ระดับ CCIE Service Provider) |

> **สรุปสั้นที่สุด**: VRF-Lite คือการ "ยืม" แนวคิด Routing Table หลายชุดของ MPLS VRF มาใช้แบบ
> **IP-only ไม่มี Label** เหมาะกับ Enterprise/Campus ที่ต้องการแยก Traffic ภายในองค์กรตัวเอง
> (เช่น Guest Wi-Fi) — Part นี้จะใช้ VRF-Lite เพื่อแยก **WIFI VLAN 40** ออกจาก Routing Table
> หลักของ Campus โดยสมบูรณ์

### 501.4 คำศัพท์ที่ต้องรู้ก่อนเริ่มคอนฟิก

| คำศัพท์ | ความหมาย |
|---|---|
| **VRF Definition** | นิยามชื่อ VRF และคุณสมบัติของมันบน Router/Switch (`vrf definition <name>`) |
| **RD (Route Distinguisher)** | ค่าที่ทำให้ Prefix ของแต่ละ VRF ไม่ชนกันตอนแลกผ่าน MP-BGP (จำเป็นเฉพาะ Full VRF/MPLS — VRF-Lite ไม่บังคับต้องมี แต่ยังตั้งได้เพื่อ Future-proofing) |
| **RT (Route Target)** | ควบคุมว่า Prefix จาก VRF ไหน "Import/Export" เข้า VRF ไหนได้ผ่าน MP-BGP (จำเป็นเฉพาะ Full VRF — ใน VRF-Lite ไม่มีผลจริงถ้าไม่มี MP-BGP แต่คอนฟิกไว้ล่วงหน้าได้) |
| **VRF Forwarding** | คำสั่งผูก Interface เข้ากับ VRF (`vrf forwarding <name>` บน Interface) |
| **Global Table / Default Routing Table** | Routing Table ปกติที่ไม่ได้ผูกกับ VRF ใดๆ (พฤติกรรม Default ของทุก Router ตั้งแต่ Part 9) |

---

## Step 502 — คอนฟิก VRF-Lite พื้นฐาน: สร้าง GUEST VRF แยก WIFI VLAN 40

### 502.1 บริบท: ทำไมต้องแยก WIFI VLAN 40

ตาม [00-ip-address-plan.md](00-ip-address-plan.md) **VLAN 40 (WIFI)** ใช้ Subnet `10.10.40.0/24`
เชื่อมต่อผ่าน **ACCESS-SW3/ACCESS-SW4 → DIST-SW3/DIST-SW4** (ตาม Access-Layer Mapping) พร้อม
**WLC-1 + AP-1/AP-2** ให้บริการ Wi-Fi (ตาม [Part 20](part-020-wireless-fundamentals.md)) — ใน
สถานการณ์จริง SSID ของ VLAN 40 มักถูกแบ่งเป็น 2 ประเภท: **Corporate Wi-Fi** (พนักงานใช้ เข้าถึง
Internal Resource ได้) และ **Guest Wi-Fi** (ผู้มาติดต่อใช้ ต้องเข้าถึงได้แค่ Internet เท่านั้น)

Part นี้จะจำลองว่า **VLAN 40 ทั้ง Subnet ถูกกำหนดให้เป็น Guest Wi-Fi โดยสมบูรณ์** (สถานการณ์
ง่ายที่สุดสำหรับการสอน VRF-Lite) — เป้าหมายคือ **แยก VLAN 40 ออกจาก Global Routing Table ของ
Campus โดยสิ้นเชิง** ก่อนที่ Step 504 จะเปิดทางให้ออก Internet ได้แบบควบคุม

### 502.2 Syntax คอนฟิก VRF-Lite (IOS-XE สมัยใหม่ — `vrf definition`)

Cisco IOS-XE รุ่นใหม่ (ตั้งแต่ 15.x เป็นต้นไป ซึ่งเป็น Baseline ของ Catalyst 9000 ตาม
[00-ip-address-plan.md](00-ip-address-plan.md)) ใช้ Syntax **`vrf definition`** แทน Syntax เก่า
`ip vrf` ที่เลิกใช้แล้ว (Legacy — พบใน IOS รุ่นเก่ามาก ไม่ใช้ในหลักสูตรนี้)

```
DIST-SW3(config)# vrf definition GUEST
DIST-SW3(config-vrf)# description ** Guest Wi-Fi Isolation VRF - VLAN 40 **
DIST-SW3(config-vrf)# rd 65001:40
DIST-SW3(config-vrf)# address-family ipv4
DIST-SW3(config-vrf-af)# route-target export 65001:40
DIST-SW3(config-vrf-af)# route-target import 65001:40
DIST-SW3(config-vrf-af)# exit-address-family
DIST-SW3(config-vrf)# exit
```

| คำสั่ง | ความหมาย |
|---|---|
| `vrf definition GUEST` | สร้าง VRF ชื่อ `GUEST` (ชื่อกำหนดเองได้ ไม่บังคับตัวเลข) |
| `rd 65001:40` | **Route Distinguisher** — รูปแบบ `<ASN>:<หมายเลขใดๆ>` หรือ `<IP>:<หมายเลขใดๆ>` — **ไม่มีผลใช้งานจริงใน VRF-Lite ที่ไม่มี MP-BGP** แต่ **ควรตั้งไว้เสมอเพื่อ Future-proofing**: ถ้าองค์กรขยายไปใช้ MPLS L3VPN จริงในอนาคต (Part 86-87) หรือทำ Route Leaking ผ่าน MP-BGP (Step 504) ค่านี้จำเป็นต้องมีอยู่แล้ว — เลข `65001` มาจาก Private AS ที่ใช้ในหลักสูตร (อ้างอิง [Part 29](part-029-bgp-fundamentals.md)), `40` มาจากเลข VLAN เพื่อให้จำง่าย |
| `address-family ipv4` | ระบุว่า VRF นี้จะรองรับ Address-Family ใด (IPv4 ในกรณีนี้ — รองรับ `address-family ipv6` แยกได้ด้วยถ้าต้องการ Dual-Stack VRF) |
| `route-target export/import 65001:40` | **Route Target** — เหมือน `rd` คือ **ไม่มีผลจริงถ้าไม่มี MP-BGP** แต่ตั้งไว้ล่วงหน้าเพื่อ Future-proofing เช่นกัน — ถ้าในอนาคตต้องทำ Route Leaking แบบ MP-BGP (Step 504.3) ค่านี้จะถูกใช้จริงทันที |

> **จุดสำคัญที่สุดของ Step นี้**: `rd`/`route-target` ใน VRF-Lite ล้วนๆ (ไม่มี MPLS/MP-BGP) เป็น
> แค่ **ค่าที่ "จอง" ไว้ล่วงหน้าเพื่อ Future-proofing** — จะยังไม่มีผลจริงจนกว่าจะมี MP-BGP หรือ
> Route Leaking มาใช้ (ดู Step 504)

### 502.3 ผูก Interface เข้ากับ VRF (`vrf forwarding`)

```
DIST-SW3(config)# interface Vlan40
DIST-SW3(config-if)# description ** GUEST Wi-Fi SVI - Isolated VRF **
DIST-SW3(config-if)# vrf forwarding GUEST
DIST-SW3(config-if)# ip address 10.10.40.2 255.255.255.0
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit
```

> **ข้อสังเกตสำคัญที่สุด**: การพิมพ์ `vrf forwarding GUEST` บน Interface ที่มี `ip address`
> อยู่แล้ว **จะลบ `ip address` เดิมออกโดยอัตโนมัติทันที** (เพราะ IP เดิมผูกกับ Global Table)
> พร้อม Warning `% Interface Vlan40 IPv4 disabled and address(es) removed due to enabling VRF
> GUEST` — **ต้องพิมพ์ `ip address` ใหม่อีกครั้งเสมอ** นี่คือข้อผิดพลาดที่พบบ่อยที่สุดของ
> วิศวกรที่ทำ VRF ครั้งแรก

DIST-SW4 (คู่ HSRP ของ VLAN 40 ตาม [Part 18](part-018-fhrp.md)) ต้องคอนฟิก VRF `GUEST`
(RD/RT เดียวกัน) และ `vrf forwarding` บน `Vlan40` แบบเดียวกันทุกประการ (IP `10.10.40.3/24`)
เพราะ VRF เป็น Concept ระดับอุปกรณ์ — ทุกอุปกรณ์ที่มี Interface ของ VLAN 40 ต้องมี VRF `GUEST`
เป็นของตัวเอง ส่วน **HSRP (`standby ...`)** ยังใช้ Syntax เดิมทุกประการไม่ต้องแก้ไขอะไร
เพราะทำงานที่ Layer ต่ำกว่า VRF (ดูแล Virtual MAC/IP บน Interface โดยไม่สนใจว่าอยู่ VRF ไหน):

```
DIST-SW3(config)# interface Vlan40
DIST-SW3(config-if)# standby 40 ip 10.10.40.1
DIST-SW3(config-if)# standby 40 priority 110
DIST-SW3(config-if)# standby 40 preempt
DIST-SW3(config-if)# exit
```

### 502.4 Verify VRF พื้นฐาน

```
DIST-SW3# show vrf
  Name                             Default RD            Protocols   Interfaces
  GUEST                            65001:40               ipv4        Vl40

DIST-SW3# show vrf detail GUEST
VRF GUEST (VRF Id = 1); default RD 65001:40; default VPNID <not set>
  New CLI format, supports multiple address-families
  Flags: 0x1808
  Interfaces:
    Vl40
  Address family ipv4 (Table ID = 0x1):
    Export VPN route-target communities
      RT:65001:40
    Import VPN route-target communities
      RT:65001:40
    No import route-map
    No export route-map
    VRF label distribution protocol: not configured
    VRF label allocation mode: per-prefix

DIST-SW3# show ip interface brief | include Vlan40
Vlan40                 10.10.40.2      YES manual up                    up
```

`VRF Id = 1` คือ Internal Index ที่ IOS สร้างให้อัตโนมัติ (ไม่ต้องกำหนดเอง) — ยืนยันว่า VRF
`GUEST` ถูกสร้างขึ้นจริงและมี Interface `Vl40` เป็นสมาชิกแล้ว

### 502.5 ทดสอบว่า Isolation ทำงานจริง

```
DIST-SW3# ping 10.10.30.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.30.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5)         <- ping จาก Global Table หา SERVERS VLAN30 ผ่านได้ปกติ

DIST-SW3# ping vrf GUEST 10.10.30.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.30.1, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)           <- ping จาก VRF GUEST หา SERVERS VLAN30 ไม่ผ่าน!

DIST-SW3# show ip route vrf GUEST
Routing Table: GUEST
Codes: L - local, C - connected, S - static, ...
Gateway of last resort is not set

C    10.10.40.0/24 is directly connected, Vlan40
L    10.10.40.2/32 is directly connected, Vlan40
```

`show ip route vrf GUEST` แสดง**เฉพาะ** Subnet ที่เป็นสมาชิกของ VRF `GUEST` เท่านั้น (ในตอนนี้
มีแค่ `10.10.40.0/24` ที่ Connected) — **ไม่มี Route ไปยัง VLAN อื่นเลยแม้แต่บรรทัดเดียว**
ทั้งที่ Global Table ของอุปกรณ์เดียวกันมี Route ไปทั่ว Campus ครบถ้วน — นี่คือ **VRF Isolation
ที่ทำงานตามที่ตั้งใจออกแบบ**

---

## Step 503 — Routing ภายใน VRF: รัน OSPF Process แยกต่อ VRF

### 503.1 ทำไม Static Route เดียวไม่พอ

Step 502 แสดง VRF `GUEST` ที่มีแค่ Subnet Connected เดียว (`10.10.40.0/24`) — ในสถานการณ์จริง
DIST-SW3 และ DIST-SW4 (คู่ HSRP) ต้อง **แลก Route ของ VRF `GUEST` ระหว่างกัน** ผ่าน Link
Uplink ไป CORE-SW1/CORE-SW2 เพื่อให้ Guest Wi-Fi Client ที่เชื่อมกับ AP ฝั่ง DIST-SW3 คุยกับ
Client ฝั่ง DIST-SW4 ได้ (Redundant Path เดียวกับที่ Global Table ทำมาตลอดตั้งแต่
[Part 11-14](part-011-ospfv2-fundamentals.md))

**หลักการสำคัญที่สุดของ Step นี้**: Routing Protocol ทุกตัว (OSPF, EIGRP, BGP, Static) **ต้อง
รันแยก Instance ต่อ VRF อย่างชัดเจน** — Global OSPF Process (เช่น `router ospf 1` ที่ใช้กับ
Campus หลัก) **ไม่มีทางรู้จัก Route ของ VRF `GUEST` ได้เลย** แม้จะเป็น Process เดียวกันบน
อุปกรณ์เดียวกันก็ตาม ต้องประกาศ `vrf` เข้าไปในคำสั่ง Routing Protocol โดยตรง

### 503.2 คอนฟิก OSPF แยก Process สำหรับ VRF GUEST

```
DIST-SW3(config)# router ospf 2 vrf GUEST
DIST-SW3(config-router)# router-id 10.10.40.2
DIST-SW3(config-router)# network 10.10.40.0 0.0.0.255 area 0
DIST-SW3(config-router)# exit
```

| คำสั่ง | ความหมาย |
|---|---|
| `router ospf 2 vrf GUEST` | สร้าง OSPF Process **หมายเลข 2** (**ต้องเป็นเลขคนละตัวกับ Process หลัก** เช่น `router ospf 1` ของ Global Table เพื่อไม่ให้สับสน) ผูกกับ VRF `GUEST` โดยตรง — Process นี้จะ**เห็นเฉพาะ Interface ที่อยู่ใน VRF `GUEST` เท่านั้น** |
| `router-id 10.10.40.2` | ตั้ง Router-ID ของ OSPF Process นี้เอง (แนะนำให้ตั้งชัดเจนเสมอ ไม่ปล่อยให้ IOS เลือกอัตโนมัติ — เหมือนหลักการที่สอนมาตลอดตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md)) — Process ของ VRF แยก Router-ID จาก Process หลักได้อย่างอิสระ |
| `network 10.10.40.0 0.0.0.255 area 0` | ประกาศ Interface `Vlan40` (ซึ่งอยู่ใน VRF `GUEST`) เข้า OSPF Area 0 — Syntax เหมือน OSPF ปกติทุกประการ ต่างกันแค่ Process นี้ผูกกับ VRF |

คอนฟิกฝั่ง **DIST-SW4** แบบเดียวกัน (`router ospf 2 vrf GUEST`, `router-id 10.10.40.3`,
`network 10.10.40.0 0.0.0.255 area 0`) — เลข Process (`ospf 2`) **ไม่จำเป็นต้องตรงกันระหว่าง
DIST-SW3 กับ DIST-SW4** เพราะเป็นค่า Local ต่ออุปกรณ์เท่านั้น สิ่งที่ต้องตรงกันคือ **Area
Number** และการที่ Interface อยู่ใน VRF ชื่อเดียวกัน (`GUEST`) เท่านั้น

### 503.3 กรณีต้องการ Uplink VRF ผ่าน CORE-SW1/CORE-SW2 (Hop-by-Hop)

ถ้าต้องการให้ VRF `GUEST` มี Reachability ข้าม CORE-SW1/CORE-SW2 ด้วย (เช่น Guest Wi-Fi ของทั้ง
DIST-SW1/DIST-SW2 และ DIST-SW3/DIST-SW4 คุยกันได้ผ่าน Core) **ต้องสร้าง VRF `GUEST` บน
CORE-SW1/CORE-SW2 ด้วย** และผูก Interface Uplink (เช่น `TenGigabitEthernet1/0/1` ที่ต่อไป
DIST-SW3) เข้า VRF `GUEST` เช่นกัน มิฉะนั้น Traffic ของ VRF `GUEST` จะหลุดเข้า Global Table ของ
CORE-SW1 ทันทีที่ข้าม Uplink (ผิดหลักการ Isolation) — นี่คือข้อจำกัดสำคัญของ VRF-Lite ตามที่
อธิบายใน Step 501.3: **ต้องคอนฟิก VRF ซ้ำทุก Hop ที่ Traffic ของ VRF นั้นต้องเดินทางผ่าน**
(Hop-by-Hop)

Part นี้เน้นสอนแนวคิดและคอนฟิกที่ **DIST-SW3 เป็นจุดเดียว** เพื่อความชัดเจน (Guest Wi-Fi ของ
DIST-SW3/DIST-SW4 เพียงพอสำหรับ Lab การสอน) — การขยาย VRF ให้ครอบคลุมทั้ง Campus หลาย Hop
เต็มรูปแบบจะกลับมาเรียนอีกครั้งเมื่อเข้าสู่ MPLS L3VPN ระดับ Provider ใน **Part 86-87** ที่
MP-BGP จะมาแทนที่การคอนฟิก Hop-by-Hop แบบนี้ทั้งหมด

### 503.4 Verify Routing ภายใน VRF

```
DIST-SW3# show ip route vrf GUEST
Routing Table: GUEST
Codes: L - local, C - connected, O - OSPF, ...
Gateway of last resort is not set

     10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C       10.10.40.0/24 is directly connected, Vlan40
L       10.10.40.2/32 is directly connected, Vlan40
O       10.10.41.0/24 [110/2] via 10.10.40.3, 00:04:12, Vlan40   <- ตัวอย่าง Subnet Guest เพิ่มเติมที่เรียนรู้จาก DIST-SW4 ผ่าน OSPF

DIST-SW3# show ip ospf 2
 Routing Process "ospf 2" with ID 10.10.40.2 running in VRF GUEST
 ...
DIST-SW3# show ip ospf neighbor vrf GUEST
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.10.40.3        1   FULL/BDR        00:00:35    10.10.40.3      Vlan40
```

`show ip ospf 2` และ `show ip ospf neighbor vrf GUEST` ยืนยันว่า OSPF Process ของ VRF `GUEST`
ทำงานสมบูรณ์แยกจาก Global OSPF Process (`router ospf 1` ที่ใช้กับ Campus หลัก) — คำสั่ง
`show` ส่วนใหญ่ที่เคยใช้กับ Global Table **ทุกตัวรองรับ Keyword `vrf <name>` เพิ่มเข้ามาเสมอ**
เพื่อระบุว่าต้องการดูข้อมูลของ VRF ไหน

---

## Step 504 — Inter-VRF Communication: Route Leaking

### 504.1 ทำไม VRF ถูกออกแบบให้ Isolate โดย Default

จาก Step 502-503 เราพิสูจน์แล้วว่า VRF `GUEST` **มองไม่เห็น Route ของ Global Table เลย** และ
ในทางกลับกัน Global Table ก็มองไม่เห็น Route ของ VRF `GUEST` — นี่คือพฤติกรรม **Default ที่
ถูกต้องตามเจตนาการออกแบบ VRF**: ถ้า VRF สองตัวมองเห็นกันได้ง่ายๆ โดย Default ก็จะไม่มีประโยชน์
อะไรที่จะสร้าง VRF ขึ้นมาแยกกันตั้งแต่ต้น (เหมือนสร้าง Vault แล้วไม่ล็อกกุญแจ)

แต่ในทางปฏิบัติ VRF ที่ Isolate สมบูรณ์ 100% มักไม่มีประโยชน์ — **Guest Wi-Fi ก็ยังต้องออก
Internet ได้** เพียงแต่ **ต้องไม่สามารถเข้าถึง Internal Resource ของ Campus ได้เลย** — นี่คือ
โจทย์ของ Step นี้: **เปิดทางแบบควบคุมได้ ("มีรูรั่วที่ตั้งใจ") ไม่ใช่เปิดทางทั้งหมด**

### 504.2 วิธีที่ 1 — Route Leaking ผ่าน Route-Target Import/Export (แนวคิด MP-BGP)

วิธีมาตรฐานของ SP (ตามที่กล่าวถึงใน [Part 19 Step 186](part-019-wan-technologies.md)) คือใช้
**MP-BGP (Multiprotocol BGP)** เป็นตัวกลางแลก Route ระหว่าง VRF โดยอ้างอิงค่า **Route-Target**
ที่ Step 502.2 ได้ตั้ง Future-proof ไว้แล้ว (`route-target export/import 65001:40`)

แนวคิด: ทุก Prefix ที่ VRF หนึ่ง **Export** ออกไปด้วย RT ที่ตรงกับ RT ที่อีก VRF **Import** ไว้
จะถูกนำเข้าอัตโนมัติผ่าน MP-BGP VPNv4 Table ที่อยู่กลาง — Syntax แนวคิด (ต้องมี MP-BGP
Address-Family `vrf` ทำงานอยู่ — จะสอนเต็มรูปแบบใน Part 86-87):

```
DIST-SW3(config)# router bgp 65001
DIST-SW3(config-router)# address-family ipv4 vrf GUEST
DIST-SW3(config-router-af)# redistribute ospf 2
DIST-SW3(config-router-af)# exit-address-family
DIST-SW3(config-router)# address-family ipv4
DIST-SW3(config-router-af)# exit-address-family
```

> **ทำไมยังไม่ลงคอนฟิกเต็มรูปแบบใน Part นี้**: ต้องมี **Underlying BGP Session ระหว่าง VRF กับ
> Global Table** ผ่าน Address-Family ซ้อนกันหลายชุดบน Process BGP เดียว ซึ่งลึกซึ้งกว่า BGP
> พื้นฐานที่เรียนใน [Part 29-30](part-029-bgp-fundamentals.md) มาก และเป็นก้าวแรกสู่ MPLS L3VPN
> — หลักสูตรนี้จะสอน MP-BGP VPNv4 Address-Family แบบเต็มรูปแบบใน **Part 86-87**

### 504.3 วิธีที่ 2 — Static Route Leaking ด้วย Keyword `global`/`vrf` (ทางเลือกที่ง่ายกว่า)

สำหรับ Guest Wi-Fi ที่ต้องการแค่ **"ออก Internet ได้ทางเดียว"** โดยไม่ต้องมี MP-BGP เต็มรูปแบบ
IOS มี Static Route Leaking แบบง่ายที่ทำงานได้บนอุปกรณ์**ตัวเดียว** (Local Leaking ระหว่าง VRF
กับ Global Table บนอุปกรณ์เดียวกัน) โดยใช้ Keyword `global` และ `vrf`:

```
ขาไป (VRF GUEST → Internet ผ่าน Global Table):
DIST-SW3(config)# ip route vrf GUEST 0.0.0.0 0.0.0.0 10.255.10.1 global

ขากลับ (Global Table รู้จักทางกลับไปยัง Subnet ของ VRF GUEST):
DIST-SW3(config)# ip route 10.10.40.0 255.255.255.0 Vlan40 vrf GUEST
```

| คำสั่ง | ความหมาย |
|---|---|
| `ip route vrf GUEST 0.0.0.0 0.0.0.0 10.255.10.1 global` | สร้าง **Default Route ภายใน VRF `GUEST`** ที่ชี้ไปยัง Next-Hop `10.255.10.1` (ทิศทางไป CORE-SW1 ตาม [00-ip-address-plan.md](00-ip-address-plan.md)) — Keyword **`global`** ท้ายคำสั่งบอกว่า **Next-Hop นี้อยู่ใน Global Table** (ไม่ใช่ VRF `GUEST`) นี่คือจุดที่ "รั่ว" ออกจาก VRF ไปยัง Global Table โดยตั้งใจ |
| `ip route 10.10.40.0 255.255.255.0 Vlan40 vrf GUEST` | สร้าง Static Route **ใน Global Table** ที่ชี้ไปยัง Subnet `10.10.40.0/24` โดยใช้ Interface `Vlan40` เป็นทางออก — Keyword **`vrf GUEST`** ท้ายคำสั่งบอกว่า Interface ปลายทาง (`Vlan40`) อยู่ใน VRF `GUEST` (คนละ VRF กับ Static Route นี้เอง ซึ่งอยู่ใน Global Table) — นี่คือ**ทางกลับ**ที่ทำให้ Traffic ตอบกลับจาก Internet หา Guest Client เจอทาง |

> **ข้อดีของวิธีนี้เทียบกับ MP-BGP (504.2)**: ไม่ต้องมี BGP เลย ไม่ต้องเข้าใจ VPNv4/RT เชิงลึก
> — ใช้ Static Route ธรรมดาที่ทุกคนคุ้นเคยตั้งแต่ [Part 9](part-009-static-routing.md) เพียงแค่
> เพิ่ม Keyword `global`/`vrf <name>` เท่านั้น **ข้อจำกัด**: ทำงานได้แค่ **บนอุปกรณ์ตัวเดียว
> (Local Leaking)** — ถ้าต้องการ Leak ข้ามหลายอุปกรณ์ ต้องใช้ MP-BGP (504.2) หรือ Redistribute
> ระหว่าง Routing Process ของแต่ละ VRF ผ่าน Route-map ที่ Filter อย่างเข้มงวด

### 504.4 Worked Example เต็มรูปแบบ: GUEST VRF ออก Internet ได้ทางเดียวผ่าน WAN-EDGE-1

**เป้าหมาย**: Guest Wi-Fi (VRF `GUEST`) ต้องออก Internet ได้ (ผ่าน WAN-EDGE-1 ที่อยู่ใน Global
Table) แต่**ต้องเข้าถึง Internal Subnet อื่น (VLAN 10/20/30/99) ไม่ได้เลย**

```
DIST-SW3(config)# ip route vrf GUEST 0.0.0.0 0.0.0.0 10.255.10.1 global
DIST-SW3(config)# ip route 10.10.40.0 255.255.255.0 Vlan40 vrf GUEST
```

Global Table ของ DIST-SW3 ต้องมี Default Route ไปยัง Internet ผ่าน CORE-SW1 → WAN-EDGE-1 อยู่
แล้ว (มาจาก OSPF Default-Route Origination ปกติของ Campus ตาม [Part 11-14](part-011-ospfv2-fundamentals.md)) — Traffic ของ Guest จะไหลตามลำดับ:

```
Guest Client (10.10.40.x) ──► DIST-SW3 VRF GUEST ──leak (global)──► DIST-SW3 Global Table
       ──► CORE-SW1 ──► WAN-EDGE-1 ──► ISP-RTR ──► Internet
```

**ทำไม Guest ยังเข้า Internal Subnet ไม่ได้**: เพราะ Static Route ที่ Leak ไว้เป็น **Default
Route (`0.0.0.0/0`) เท่านั้น** — VRF `GUEST` **ไม่มี Route เจาะจงไปยัง `10.10.10.0/24`,
`10.10.20.0/24`, `10.10.30.0/24` หรือ `10.10.99.0/24` เลย** (Route เหล่านี้อยู่ใน Global Table
เท่านั้น ไม่ได้ถูก Leak เข้ามา) ดังนั้นแม้ Traffic ปลายทางเป็น Internal Subnet จะ Match กับ
Default Route `0.0.0.0/0` ของ VRF `GUEST` และถูกส่งออกไปทาง `10.255.10.1` (CORE-SW1) เหมือน
Internet Traffic — **แต่ CORE-SW1 (Global Table) จะ Route Traffic นั้นไปยัง VLAN ปลายทางจริง
ตามปกติ เพราะ CORE-SW1 มองไม่เห็นว่า Traffic นี้มาจาก VRF ไหน (แค่เห็น Global Table ปลายทาง)**

> **ข้อควรระวังสำคัญ**: Static Route Leaking แบบนี้ **มี "ช่องโหว่" ทางทฤษฎี**: เพราะ Traffic
> ทั้งหมดจาก VRF `GUEST` ไหลเข้า Global Table ของ CORE-SW1 **โดยไม่มีการกรองใดๆ** — ถ้า Guest
> Client ส่ง Packet ที่ Destination เป็น `10.10.30.5` (Server ภายใน) ตรงๆ Traffic นั้นก็จะไหล
> ผ่าน Route Leaking ไปถึง Global Table แล้ว **CORE-SW1 จะ Route ไปหา Server จริงได้!** —
> **การ Leak แค่ Default Route ไม่ได้ป้องกัน Internal Access โดยอัตโนมัติ** ต้องเพิ่ม **ACL
> กรอง Internal Subnet บน Interface ขาเข้าของ VRF `GUEST` (Vlan40) หรือบน Uplink ของ CORE-SW1**
> ควบคู่ไปด้วยเสมอ (Defense-in-Depth — ผสม VRF Isolation + ACL ตาม [Part 21](part-021-switch-security.md)
> ไม่ใช่พึ่งพา VRF เพียงอย่างเดียว):
> ```
> DIST-SW3(config)# ip access-list extended GUEST-INTERNET-ONLY
> DIST-SW3(config-ext-nacl)# deny   ip any 10.10.0.0 0.0.255.255
> DIST-SW3(config-ext-nacl)# permit ip any any
> DIST-SW3(config-ext-nacl)# exit
> DIST-SW3(config)# interface Vlan40
> DIST-SW3(config-if)# ip access-group GUEST-INTERNET-ONLY in
> DIST-SW3(config-if)# exit
> ```
> **บทเรียนสำคัญที่สุดของ Step นี้**: VRF ให้ Isolation ที่แน่นกว่า ACL เพียงอย่างเดียว แต่
> **ทันทีที่ทำ Route Leaking (เปิดรูรั่วโดยตั้งใจ) ต้องกลับมาพึ่ง ACL เพื่อกรอง Traffic ที่ไหล
> ผ่านรูรั่วนั้นอีกที** — VRF และ ACL เป็น**เครื่องมือที่เสริมกัน ไม่ใช่แทนกัน**

### 504.5 Verify Route Leaking

```
DIST-SW3# show ip route vrf GUEST
Routing Table: GUEST
Codes: L - local, C - connected, S - static, ...
Gateway of last resort is 10.255.10.1 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 10.255.10.1 (Global)
C    10.10.40.0/24 is directly connected, Vlan40
L    10.10.40.2/32 is directly connected, Vlan40

DIST-SW3# show ip route | include 10.10.40.0
S    10.10.40.0/24 [1/0] via Vlan40 (GUEST)
```

`via 10.255.10.1 (Global)` และ `via Vlan40 (GUEST)` คือจุดยืนยันสำคัญที่สุด — IOS ระบุชัดเจน
ในวงเล็บว่า Next-Hop/Interface ของ Static Route นี้ **อ้างอิงไปยัง VRF อื่น** (Cross-VRF
Reference) ซึ่งเป็นผลจาก Keyword `global`/`vrf` ที่ตั้งไว้ใน 504.3

---

## Step 505 — GRE Tunnel เชิงลึก: Keepalive, PMTUD, และ Redundancy

### 505.1 ทวนพื้นฐาน GRE จาก Part 19

[Part 19 Step 188](part-019-wan-technologies.md) สร้าง GRE Tunnel พื้นฐานระหว่าง **WAN-EDGE-1**
และ **BRANCH-RTR** (`Tunnel0`, Subnet `172.16.200.0/30`) — Tunnel นี้ทำงานได้แล้ว แต่ยังมี
จุดอ่อน 2 อย่างที่ Production จริงต้องแก้ไข:

1. **ไม่มี Keepalive**: ถ้า Path กลางทาง (Internet) ขาดโดยไม่มีการแจ้งเตือน Layer 2/3 ปกติ
   (เช่น Provider drop Traffic เงียบๆ แต่ Physical Interface ยัง `up/up`) **Tunnel0 จะยังคง
   แสดง `up/up` ตลอดไป** แม้ Traffic จริงจะไปไม่ถึงปลายทางแล้ว — Routing Protocol ที่วิ่งบน
   Tunnel (เช่น OSPF ที่ตั้งไว้ใน Part 19 Step 190) จะไม่รู้ว่า Tunnel ตายจนกว่า Dead Timer
   ของ Routing Protocol เองจะหมด (ช้ากว่าที่ควรจะเป็น)
2. **ไม่มีการจัดการ MTU/Fragmentation**: GRE เพิ่ม Header 24 byte (IP 20 + GRE 4) เข้าไปใน
   Packet เดิม — ถ้า Path Physical มี MTU 1500 byte เต็ม Packet ที่มีขนาดใกล้ 1500 byte
   (ก่อนห่อ GRE) จะเกิน MTU ของ Physical Link ทันทีหลังห่อ GRE ทำให้ต้อง Fragment ซึ่งลด
   Performance และอาจถูก Drop ถ้า Provider ปิด Fragmentation ไว้

### 505.2 GRE Keepalive

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# keepalive 5 3
WAN-EDGE-1(config-if)# exit
```

- **Syntax**: `keepalive <interval-seconds> <retry-count>`
- `keepalive 5 3` = ส่ง Keepalive Packet ทุก **5 วินาที**, ถ้าไม่ได้รับ Response ครบ **3 ครั้ง
  ติดต่อกัน** (รวม 15 วินาที) — IOS จะเปลี่ยนสถานะ Tunnel Interface เป็น **`down`** ทันที
- **หลักการทำงาน**: WAN-EDGE-1 ส่ง GRE Keepalive Packet ที่ห่อ GRE ซ้อนกัน 2 ชั้น (GRE-in-GRE)
  ไปยัง BRANCH-RTR — ถ้า BRANCH-RTR ยังมี Tunnel0 อยู่และ Routing ไปกลับได้ปกติ มันจะ Echo
  Packet นั้นกลับมาโดยอัตโนมัติ (ไม่ต้องคอนฟิกอะไรเพิ่มที่ฝั่ง BRANCH-RTR — **GRE Keepalive
  เป็น One-Sided ทำงานได้แม้อีกฝั่งไม่ได้ตั้ง `keepalive` เลย** ต่างจาก Protocol อื่นที่ต้อง
  Symmetric ทั้งสองฝั่ง)
- **แนะนำให้ตั้งทั้งสองฝั่งเสมอ** เพื่อให้ทั้งสอง Endpoint รู้สถานะ Tunnel พร้อมกัน:

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# keepalive 5 3
BRANCH-RTR(config-if)# exit
```

> **ผลลัพธ์ที่ได้**: ถ้า Path กลางทางขาดจริง (แม้ Physical Interface ทั้งสองฝั่งยัง `up/up`)
> Tunnel0 จะเปลี่ยนเป็น `down` ภายใน ~15 วินาที ทำให้ Routing Protocol (OSPF) Reconverge ไป
> ใช้ Path สำรอง (Leased Line PPP จาก Part 19) **เร็วกว่า**การรอ OSPF Dead Timer เดิม
> (Default 40 วินาที) มาก

### 505.3 Tunnel Path-MTU-Discovery (PMTUD)

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# tunnel path-mtu-discovery
WAN-EDGE-1(config-if)# exit
```

- **หลักการทำงาน**: เปิดให้ Router **Discover ค่า MTU ที่เล็กที่สุดตลอดเส้นทาง (Path MTU)**
  โดยอัตโนมัติผ่านการรับ ICMP "Fragmentation Needed" message จาก Router กลางทางที่ Drop
  Packet เพราะเกิน MTU ของ Interface ตัวเอง — เมื่อ WAN-EDGE-1 ได้รับ ICMP นี้ มันจะ**ปรับ
  MTU ของ Tunnel0 ให้เล็กลงอัตโนมัติ**และตั้งค่า **DF (Don't Fragment) Bit** ใน GRE Header
  ให้ Packet ขนาดใหญ่ที่เกิน MTU ใหม่ **ไม่ต้อง Fragment ที่ตัว Router เอง** (ให้ Host ต้นทาง
  เป็นคน Fragment เองตั้งแต่ต้น หรือใช้ TCP MSS ที่เล็กลงพอดี)
- **ข้อจำกัดในทางปฏิบัติ**: ต้องพึ่งพา ICMP Type 3 Code 4 (Fragmentation Needed but DF Set)
  ไหลผ่าน Internet กลับมาได้จริง — Firewall/Provider จำนวนมากปิด ICMP ทั้งหมดหรือกรอง ICMP
  Type นี้ (เพื่อป้องกัน Attack) ทำให้ PMTUD **ทำงานไม่สมบูรณ์ในทางปฏิบัติจริง** (ปัญหา
  "PMTUD Black-Hole" ที่รู้จักกันดีในวงการ Network Engineering)
- **วิธีแก้ที่แนะนำกว่าในทางปฏิบัติ**: ปรับ **TCP MSS (Maximum Segment Size) Clamping**
  บน Tunnel Interface โดยตรง ซึ่งไม่พึ่งพา ICMP เลย:

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# ip tcp adjust-mss 1360
WAN-EDGE-1(config-if)# exit
```

`ip tcp adjust-mss 1360` บังคับให้ TCP Segment ที่วิ่งผ่าน Tunnel0 มีขนาดเล็กพอที่จะไม่ต้อง
Fragment แม้ห่อ GRE Header (และ IPsec Header ใน Step 507-508) เพิ่มแล้วก็ตาม (ค่าที่แนะนำ
ทั่วไปสำหรับ GRE-over-IPsec คือ `1360` เพื่อเผื่อ Overhead ของทั้ง GRE + ESP)

### 505.4 GRE Redundancy: หลาย Tunnel ผ่านหลาย Path

Design ที่ Part 19 Step 190 ทำไว้ (Leased Line + GRE Tunnel เดี่ยว) มีจุดอ่อน: **ถ้า
WAN-EDGE-1 เชื่อมต่อ Internet ผ่าน ISP เดียว** (`GigabitEthernet0/0/0` → ISP-RTR) การที่ Link
นี้ล่มจะทำให้ **ทั้ง Path หลัก (PPP) และ Path สำรอง (GRE) ล่มพร้อมกันไม่ได้** เพราะ PPP ไม่ได้
พึ่ง Internet แต่ **ถ้าองค์กรมี ISP 2 เจ้า (ตาม WAN-EDGE-1 และ WAN-EDGE-2 ของ Campus)** ควรสร้าง
**GRE Tunnel คู่ที่สอง** ผ่าน WAN-EDGE-2 ด้วย (Tunnel0 Primary ผ่าน WAN-EDGE-1
`172.16.200.0/30`, Tunnel1 Secondary ผ่าน WAN-EDGE-2 `172.16.202.0/30`) เพื่อ Redundancy ระดับ
ISP อย่างสมบูรณ์:

**คอนฟิก Tunnel1 เพิ่มบน BRANCH-RTR** (ฝั่ง WAN-EDGE-2 คอนฟิก Tunnel0 ของตัวเองแบบเดียวกันกับ
ที่ WAN-EDGE-1 ทำไว้ใน Part 19):

```
BRANCH-RTR(config)# interface Tunnel1
BRANCH-RTR(config-if)# description ** GRE Tunnel Backup via WAN-EDGE-2 **
BRANCH-RTR(config-if)# ip address 172.16.202.2 255.255.255.252
BRANCH-RTR(config-if)# tunnel source GigabitEthernet0/0/1
BRANCH-RTR(config-if)# tunnel destination 203.0.113.6
BRANCH-RTR(config-if)# keepalive 5 3
BRANCH-RTR(config-if)# ip tcp adjust-mss 1360
BRANCH-RTR(config-if)# exit
!
BRANCH-RTR(config)# router ospf 1
BRANCH-RTR(config-router)# network 172.16.202.0 0.0.0.3 area 0
BRANCH-RTR(config-router)# exit
```

ให้ OSPF ที่รันอยู่บน Tunnel0/Tunnel1 ทั้งคู่เลือก Path ที่ Metric ดีที่สุดเองตามปกติ (หรือปรับ
`ip ospf cost` บน Tunnel1 ให้สูงกว่า Tunnel0 เพื่อบังคับให้เป็น Backup-Only อย่างชัดเจน — หลัก
การเดียวกับที่เรียนใน [Part 26](part-026-advanced-ospf.md)):

```
BRANCH-RTR(config)# interface Tunnel1
BRANCH-RTR(config-if)# ip ospf cost 2000
BRANCH-RTR(config-if)# exit
```

> **ผลลัพธ์**: ได้ Redundancy ครบ 3 ชั้น — **Leased Line (PPP)** เป็น Primary, **GRE Tunnel0
> ผ่าน WAN-EDGE-1** เป็น Backup ชั้นที่ 1, **GRE Tunnel1 ผ่าน WAN-EDGE-2** เป็น Backup ชั้นที่ 2
> — ทนต่อความล้มเหลวได้ทั้งระดับ Leased Line, ระดับ ISP ตัวเดียว, และ Keepalive ยังช่วยให้
> ตรวจจับความล้มเหลวได้เร็วกว่าการรอ Routing Protocol Dead Timer เดิม

---

## Step 506 — IPsec พื้นฐานเชิงลึก: IKEv1 vs IKEv2, Phase 1-2 เต็มรูปแบบ

### 506.1 ทวนพื้นฐานจาก Part 19 และภาพรวม Step นี้

[Part 19 Step 189](part-019-wan-technologies.md) แนะนำ **ISAKMP/IKE Phase 1-2** แบบแนวคิด
เท่านั้น (ยังไม่มีคอนฟิกจริง) — Step นี้จะลงคอนฟิกเต็มรูปแบบทั้ง **IKEv1 (Legacy)** และ
**IKEv2 (มาตรฐานปัจจุบัน)** พร้อมเปรียบเทียบ แล้วเลือก **IKEv2** เป็นหลักสำหรับ Step 507-510
เพราะเป็นมาตรฐานที่ Cisco แนะนำสำหรับ Deployment ใหม่ทั้งหมด (รวมถึง FlexVPN/DMVPN Phase 3 ที่
จะเรียนใน Part 52-53)

### 506.2 IKEv1 vs IKEv2 — ตารางเปรียบเทียบ

| คุณสมบัติ | IKEv1 (RFC 2409, Legacy) | IKEv2 (RFC 7296, มาตรฐานปัจจุบัน) |
|---|---|---|
| จำนวน Message ในการเจรจา Phase 1 | 6 Message (Main Mode) หรือ 3 Message (Aggressive Mode) | **4 Message เท่านั้น** (เร็วกว่า) |
| การรวม Phase 1+2 | แยกกันชัดเจน (ISAKMP SA ก่อน แล้วค่อย Quick Mode สร้าง IPsec SA) | รวม Concept เข้าด้วยกันมากขึ้น — เรียก IKE_SA และ CHILD_SA |
| NAT-Traversal | ต้องเพิ่มเติม (NAT-T เป็น RFC เสริมแยกออกไป) | **มีในตัวมาตรฐานตั้งแต่ต้น** |
| Dead Peer Detection (DPD) | ต้องเปิดแยกเป็น Feature เสริม | **Built-in** เป็นส่วนหนึ่งของ Protocol |
| รองรับ EAP Authentication (สำหรับ Remote Access VPN) | จำกัด (ต้องพึ่ง Xauth แยก) | **รองรับ EAP โดยตรงในมาตรฐาน** — เหมาะกับ FlexVPN/AnyConnect |
| ความซับซ้อนของ Configuration | ใช้ `crypto isakmp policy` | ใช้ `crypto ikev2 proposal/policy` (Syntax คล้ายกันแต่ปรับใหม่) |
| Cisco แนะนำสำหรับ Deployment ใหม่ | ❌ ไม่แนะนำ (Legacy — คงไว้เพื่อ Backward Compatibility เท่านั้น) | ✅ **แนะนำเสมอ** — เป็นพื้นฐานของ FlexVPN/DMVPN Phase 3 |

> **หลักสูตรนี้จะสอน IKEv1 Syntax ไว้เพื่อให้อ่าน Config เก่า/สอบ CCNP ได้ (Step 506.3) แต่จะ
> ใช้ IKEv2 จริงในทุก Worked Example ตั้งแต่ Step 507 เป็นต้นไป** ตามแนวทางที่ Cisco แนะนำและ
> ตรงกับพื้นฐานของ FlexVPN ที่จะเรียนใน Step 509

### 506.3 IKEv1 Phase 1-2 (Syntax อ้างอิง — Legacy, ให้เห็นความคล้ายกับ IKEv2)

```
WAN-EDGE-1(config)# crypto isakmp policy 10
WAN-EDGE-1(config-isakmp)# encryption aes 256
WAN-EDGE-1(config-isakmp)# hash sha256
WAN-EDGE-1(config-isakmp)# authentication pre-share
WAN-EDGE-1(config-isakmp)# group 14
WAN-EDGE-1(config-isakmp)# exit
WAN-EDGE-1(config)# crypto isakmp key Cisco@IPsec123 address 198.51.100.2
!
WAN-EDGE-1(config)# crypto ipsec transform-set TSET-LEGACY esp-aes 256 esp-sha256-hmac
WAN-EDGE-1(cfg-crypto-trans)# mode tunnel
WAN-EDGE-1(cfg-crypto-trans)# exit
WAN-EDGE-1(config)# ip access-list extended VPN-TRAFFIC
WAN-EDGE-1(config-ext-nacl)# permit ip host 203.0.113.2 host 198.51.100.2
WAN-EDGE-1(config-ext-nacl)# exit
WAN-EDGE-1(config)# crypto map CMAP-LEGACY 10 ipsec-isakmp
WAN-EDGE-1(config-crypto-map)# set peer 198.51.100.2
WAN-EDGE-1(config-crypto-map)# set transform-set TSET-LEGACY
WAN-EDGE-1(config-crypto-map)# match address VPN-TRAFFIC
WAN-EDGE-1(config-crypto-map)# exit
```

| คำสั่งหลัก | ความหมาย |
|---|---|
| `crypto isakmp policy 10` + `encryption`/`hash`/`group` | Algorithm ของ Phase 1 (Management Channel) — เลข Policy น้อยกว่า = Priority สูงกว่า |
| `authentication pre-share` + `crypto isakmp key ... address ...` | ยืนยันตัวตนด้วย Pre-Shared Key ผูกกับ IP Peer ตรงๆ (ทางเลือกอื่นคือ `rsa-sig` ผ่าน PKI) |
| `crypto ipsec transform-set` + `crypto map ... match address` | Phase 2 — Transform-Set + Crypto ACL + Crypto Map แบบเดียวกับที่จะอธิบายเต็มรูปแบบใน Step 507 (ด้วย IKEv2) |

### 506.5 IKEv2 Phase 1: Proposal + Policy + Keyring

IKEv2 แยกส่วน Algorithm (`proposal`) ออกจากส่วนที่ระบุว่าจะใช้ Proposal ไหนกับ Peer ไหน
(`policy`) ทำให้ยืดหยุ่นกว่า IKEv1 (ที่รวม Algorithm ทั้งหมดไว้ใน Policy เดียว)

```
WAN-EDGE-1(config)# crypto ikev2 proposal IKEV2-PROPOSAL
WAN-EDGE-1(config-ikev2-proposal)# encryption aes-cbc-256
WAN-EDGE-1(config-ikev2-proposal)# integrity sha256
WAN-EDGE-1(config-ikev2-proposal)# group 14
WAN-EDGE-1(config-ikev2-proposal)# exit
!
WAN-EDGE-1(config)# crypto ikev2 policy IKEV2-POLICY
WAN-EDGE-1(config-ikev2-policy)# proposal IKEV2-PROPOSAL
WAN-EDGE-1(config-ikev2-policy)# exit
!
WAN-EDGE-1(config)# crypto ikev2 keyring IKEV2-KEYRING
WAN-EDGE-1(config-ikev2-keyring)# peer BRANCH-RTR
WAN-EDGE-1(config-ikev2-keyring-peer)# address 198.51.100.2
WAN-EDGE-1(config-ikev2-keyring-peer)# pre-shared-key Cisco@IPsec123
WAN-EDGE-1(config-ikev2-keyring-peer)# exit
WAN-EDGE-1(config-ikev2-keyring)# exit
!
WAN-EDGE-1(config)# crypto ikev2 profile IKEV2-PROFILE
WAN-EDGE-1(config-ikev2-profile)# match identity remote address 198.51.100.2 255.255.255.255
WAN-EDGE-1(config-ikev2-profile)# authentication local pre-share
WAN-EDGE-1(config-ikev2-profile)# authentication remote pre-share
WAN-EDGE-1(config-ikev2-profile)# keyring local IKEV2-KEYRING
WAN-EDGE-1(config-ikev2-profile)# exit
```

| Object | หน้าที่ |
|---|---|
| **`crypto ikev2 proposal`** | ชุด Algorithm ที่ Router ยินดีใช้ (Encryption/Integrity/DH Group) — เทียบเท่า `crypto isakmp policy` ของ IKEv1 แต่แยกออกมาเป็น Object อิสระ ใช้ซ้ำได้หลาย Policy |
| **`crypto ikev2 policy`** | จับคู่ Proposal เข้ากับเงื่อนไข (เช่น match ตาม VRF หรือ Local Address — ในตัวอย่างนี้ใช้แบบพื้นฐานไม่เจาะจงเงื่อนไข) |
| **`crypto ikev2 keyring`** | เก็บ Pre-Shared Key **แยกตาม Peer แต่ละราย** (ต่างจาก IKEv1 ที่ผูก Key กับ IP ตรงๆ ใน Global Config) — รองรับหลาย Peer ในไฟล์เดียวได้ดีกว่า |
| **`crypto ikev2 profile`** | **Object ใหม่ที่ IKEv1 ไม่มี** — รวม Identity Matching + Authentication Method + Keyring เข้าด้วยกันเป็นชุดเดียว ทำให้ 1 Router รองรับ Peer หลายรายที่ใช้ Authentication คนละแบบได้ในเวลาเดียวกัน (จุดนี้คือรากฐานสำคัญของ FlexVPN ใน Step 509) |

### 506.6 IKEv2 Phase 2: IPsec Transform-Set (Syntax เหมือน IKEv1)

ข่าวดี: **IPsec Transform-Set (Phase 2) ใช้ Syntax เดียวกันทั้ง IKEv1 และ IKEv2** — ไม่มี
"IKEv2 Transform-Set" แยกต่างหาก เพราะ Transform-Set อธิบาย Algorithm ของ **Data Traffic
Encryption** ซึ่งเป็นแนวคิดที่ไม่เปลี่ยนไม่ว่า Phase 1 จะเจรจาด้วย IKE เวอร์ชันไหน:

```
WAN-EDGE-1(config)# crypto ipsec transform-set TSET-IKEV2 esp-aes 256 esp-sha256-hmac
WAN-EDGE-1(cfg-crypto-trans)# mode tunnel
WAN-EDGE-1(cfg-crypto-trans)# exit
```

- **`esp-aes 256`**: ใช้ ESP (Encapsulating Security Payload) เข้ารหัสด้วย AES-256
- **`esp-sha256-hmac`**: ใช้ SHA-256 HMAC สำหรับ Integrity/Authentication ของแต่ละ Packet
- **`mode tunnel`**: **Tunnel Mode** (Default) — เข้ารหัสทั้ง IP Packet เดิม (Header + Payload)
  แล้วห่อด้วย IP Header ใหม่ ต่างจาก **Transport Mode** ที่เข้ารหัสแค่ Payload เดิม (ใช้ IP
  Header เดิม) — Site-to-Site VPN ใช้ **Tunnel Mode เสมอ** (Transport Mode ใช้กับ Host-to-Host
  โดยตรงเท่านั้น ซึ่งพบน้อยกว่ามาก)

### 506.7 ภาพรวมกระบวนการเจรจา IKEv2

IKEv2 เจรจาแค่ 2 คู่ Message: **`IKE_SA_INIT`** (เจรจา Algorithm + แลก DH Key — เทียบเท่า
IKEv1 Main Mode Message 1-2) ตามด้วย **`IKE_AUTH`** (ยืนยันตัวตนด้วย PSK **พร้อมสร้าง
CHILD_SA แรกในคราวเดียว** — รวม Phase 1 Authentication + Phase 2/Quick Mode เข้าด้วยกัน) จบแล้ว
ได้ **IKE_SA** (เทียบเท่า ISAKMP SA) และ **CHILD_SA** (เทียบเท่า IPsec SA) พร้อมใช้งานทันที —
นี่คือเหตุผลที่ IKEv2 ใช้แค่ 4 Message รวม เทียบกับ 6+ Message ของ IKEv1 Main Mode ที่ต้องแยก
Quick Mode อีกชุด

---

## Step 507 — GRE over IPsec: ปกป้อง GRE Tunnel ของ Part 19 ด้วย IKEv2 + Crypto Map

### 507.1 เป้าหมายของ Step นี้

นำ GRE Tunnel0 ที่สร้างไว้ใน [Part 19 Step 188](part-019-wan-technologies.md) (WAN-EDGE-1 ↔
BRANCH-RTR, `172.16.200.0/30`) มา**เข้ารหัสด้วย IPsec ผ่านวิธีดั้งเดิมที่สุด: Crypto Map ผูก
กับ Physical Interface** — วิธีนี้**ไม่แก้ Tunnel0 เลย** เพียงแค่เพิ่ม Crypto Map เข้าไปที่
Interface ทางออกจริง (`GigabitEthernet0/0/0` ของ WAN-EDGE-1) เพื่อเข้ารหัส **Traffic ทั้งหมดที่
เป็น GRE ระหว่างสอง Endpoint นี้โดยเฉพาะ**

```
                         ก่อน Step นี้ (Part 19):
   [ Original Packet ] → GRE encapsulate → [ GRE[ Original Packet ] ] → Cleartext บน Internet

                         หลัง Step นี้ (GRE over IPsec ด้วย Crypto Map):
   [ Original Packet ] → GRE encapsulate → [ GRE[ Original Packet ] ]
                                                    │ IPsec เข้ารหัส (Crypto Map ที่ Gi0/0/0)
                                                    ▼
                          [ New IP Header [ ESP [ GRE[ Original Packet ] ] ] ]
```

### 507.2 คอนฟิกเต็มรูปแบบบน WAN-EDGE-1

**ขั้นที่ 1 — IKEv2 Phase 1** (ใช้ Object จาก Step 506.5 ตรงตัว):

```
WAN-EDGE-1(config)# crypto ikev2 proposal IKEV2-PROPOSAL
WAN-EDGE-1(config-ikev2-proposal)# encryption aes-cbc-256
WAN-EDGE-1(config-ikev2-proposal)# integrity sha256
WAN-EDGE-1(config-ikev2-proposal)# group 14
WAN-EDGE-1(config-ikev2-proposal)# exit
!
WAN-EDGE-1(config)# crypto ikev2 keyring IKEV2-KEYRING
WAN-EDGE-1(config-ikev2-keyring)# peer BRANCH-RTR
WAN-EDGE-1(config-ikev2-keyring-peer)# address 198.51.100.2
WAN-EDGE-1(config-ikev2-keyring-peer)# pre-shared-key Cisco@IPsec123
WAN-EDGE-1(config-ikev2-keyring-peer)# exit
WAN-EDGE-1(config-ikev2-keyring)# exit
!
WAN-EDGE-1(config)# crypto ikev2 profile IKEV2-PROFILE
WAN-EDGE-1(config-ikev2-profile)# match identity remote address 198.51.100.2 255.255.255.255
WAN-EDGE-1(config-ikev2-profile)# authentication local pre-share
WAN-EDGE-1(config-ikev2-profile)# authentication remote pre-share
WAN-EDGE-1(config-ikev2-profile)# keyring local IKEV2-KEYRING
WAN-EDGE-1(config-ikev2-profile)# exit
```

**ขั้นที่ 2 — IPsec Transform-Set (Phase 2)**:

```
WAN-EDGE-1(config)# crypto ipsec transform-set TSET-GRE esp-aes 256 esp-sha256-hmac
WAN-EDGE-1(cfg-crypto-trans)# mode tunnel
WAN-EDGE-1(cfg-crypto-trans)# exit
```

**ขั้นที่ 3 — Crypto ACL: ระบุ Traffic ที่ต้องเข้ารหัส (คือ GRE Traffic ระหว่างสอง Endpoint)**:

```
WAN-EDGE-1(config)# ip access-list extended CRYPTO-GRE-TRAFFIC
WAN-EDGE-1(config-ext-nacl)# permit gre host 203.0.113.2 host 198.51.100.2
WAN-EDGE-1(config-ext-nacl)# exit
```

> **จุดสำคัญที่สุดของ Step นี้**: Crypto ACL ระบุ `permit gre host 203.0.113.2 host
> 198.51.100.2` คือ **Match ที่ Protocol Number 47 (GRE)** ระหว่าง **Physical (Tunnel
> Source/Destination) IP** ของทั้งสอง Endpoint **ไม่ใช่** Tunnel IP (`172.16.200.x`) — เพราะ
> IPsec ทำงานบน Physical Interface (Gi0/0/0) โดยเข้ารหัส Packet ที่ออกจาก Interface นั้น**หลัง
> จาก**ที่ GRE ห่อหุ้มเสร็จแล้ว (Traffic ตอนออก Physical Interface จะเห็นเป็น IP Packet ที่มี
> Protocol = GRE, Source/Destination = Physical IP ทั้งคู่ ไม่ใช่ Tunnel IP อีกต่อไป)

**ขั้นที่ 4 — Crypto Map (ผูก Peer + Transform-Set + ACL เข้าด้วยกัน, ระบุ IKEv2 Profile)**:

```
WAN-EDGE-1(config)# crypto map CMAP-GRE 10 ipsec-isakmp
WAN-EDGE-1(config-crypto-map)# set peer 198.51.100.2
WAN-EDGE-1(config-crypto-map)# set transform-set TSET-GRE
WAN-EDGE-1(config-crypto-map)# set ikev2-profile IKEV2-PROFILE
WAN-EDGE-1(config-crypto-map)# match address CRYPTO-GRE-TRAFFIC
WAN-EDGE-1(config-crypto-map)# exit
```

**ขั้นที่ 5 — Apply Crypto Map เข้า Physical Interface ที่แท้จริง (ไม่ใช่ Tunnel0!)**:

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# crypto map CMAP-GRE
WAN-EDGE-1(config-if)# exit
```

```
%CRYPTO-6-ISAKMP_ON_OFF: ISAKMP is ON
```

### 507.3 คอนฟิกฝั่ง BRANCH-RTR (ภาพสะท้อนกลับทุกค่า)

โครงสร้างเหมือน 507.2 ทุกประการ สลับ Local/Remote IP: `crypto ikev2 keyring` ชี้ `peer
WAN-EDGE-1 address 203.0.113.2`, `crypto ikev2 profile` ใช้ `match identity remote address
203.0.113.2`, Crypto ACL เป็น `permit gre host 198.51.100.2 host 203.0.113.2`, และ `crypto
map CMAP-GRE` ใช้ `set peer 203.0.113.2` แล้ว Apply เข้า `interface GigabitEthernet0/0/1`
(Physical Interface ที่ต่อ Internet ของ BRANCH-RTR) — Transform-Set (`TSET-GRE`) และ IKEv2
Proposal (`IKEV2-PROPOSAL`) ใช้ Algorithm เดียวกันทั้งสองฝั่งเสมอ (ตาม 507.2)

### 507.4 Verify: `show crypto isakmp sa` และ `show crypto ipsec sa`

```
WAN-EDGE-1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
198.51.100.2    203.0.113.2     QM_IDLE           1001 ACTIVE

WAN-EDGE-1# show crypto ikev2 sa
IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         203.0.113.2/500       198.51.100.2/500      none/none            READY
      Encr: AES-CBC, keysize: 256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/240 sec

WAN-EDGE-1# show crypto ipsec sa

interface: GigabitEthernet0/0/0
    Crypto map tag: CMAP-GRE, local addr 203.0.113.2

   local  ident (addr/mask/prot/port): (203.0.113.2/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (198.51.100.2/255.255.255.255/47/0)
   current_peer 198.51.100.2 port 500
    #pkts encaps: 1284, #pkts encrypt: 1284, #pkts digest: 1284
    #pkts decaps: 1279, #pkts decrypt: 1279, #pkts verify: 1279

     inbound esp sas:
      spi: 0x8C41D2E0(2352087264)
        transform: esp-256-aes esp-sha256-hmac, in use settings ={Tunnel, }
     outbound esp sas:
      spi: 0x3A2F1B90(974601104)
        transform: esp-256-aes esp-sha256-hmac, in use settings ={Tunnel, }
```

| ค่าที่ต้องดู | ความหมาย |
|---|---|
| `state: QM_IDLE` / `Status: READY` | ISAKMP/IKEv2 SA (Phase 1) เจรจาสำเร็จ พร้อมใช้งาน |
| `local/remote ident ... prot/port: .../47/0` | ยืนยันว่า Crypto ACL Match Protocol 47 (GRE) ถูกต้องตามที่ตั้งใจ |
| `#pkts encaps` / `#pkts decaps` เพิ่มขึ้นเรื่อยๆ | ยืนยันว่า Traffic จริงถูกเข้ารหัส/ถอดรหัสอยู่ (ตัวเลขนิ่งค้าง = ไม่มี Traffic ไหลผ่าน หรือ Tunnel มีปัญหา) |
| `inbound/outbound esp sas` มีค่า `spi` | IPsec SA (Phase 2) สร้างสำเร็จทั้งสองทิศทาง |

### 507.5 ข้อจำกัดของวิธี Crypto Map แบบดั้งเดิม

- ต้องดูแล **Crypto ACL ให้ตรงกับ Physical IP คู่ Peer ทุกคู่แยกกันเอง** — ถ้าจำนวน Site เพิ่ม
  ขึ้นมาก (Hub-and-Spoke หลาย Spoke) ต้องเขียน ACL + Crypto Map Entry แยกทุกคู่ **ไม่ Scale**
- **ผูกกับ Physical Interface ตรงๆ** — ถ้าต้องการเปลี่ยน ISP/Physical Interface ในอนาคต ต้อง
  ย้าย Crypto Map ทั้งชุดตาม
- **ไม่มี Interface เดี่ยวที่เป็นตัวแทนของ Tunnel ที่เข้ารหัสแล้ว** — การเขียน Routing Policy,
  QoS, หรือ ACL แยกสำหรับ Traffic ที่เข้ารหัสแล้วทำได้ยาก เพราะทุกอย่างเกิดที่ Physical
  Interface ปนกับ Traffic อื่น

ข้อจำกัดเหล่านี้คือเหตุผลที่ Cisco พัฒนา **VTI (Virtual Tunnel Interface)** ขึ้นมาแทนที่ —
รายละเอียดใน Step 508

---

## Step 508 — VTI (Virtual Tunnel Interface): ทางเลือกที่เรียบง่ายกว่า

### 508.1 แนวคิดของ VTI

**VTI (Virtual Tunnel Interface)** คือ Interface เสมือนที่ **รวม GRE-like Tunnel Interface
เข้ากับ IPsec Encryption ไว้ในที่เดียว** — ไม่ต้องมี Crypto Map, ไม่ต้องมี Crypto ACL แยก
ต่างหาก **การเข้ารหัสผูกอยู่กับ Interface ตรงๆ ผ่านคำสั่งเดียว: `tunnel protection ipsec
profile`**

### 508.2 เปรียบเทียบ VTI กับ Crypto Map แบบละเอียด

| คุณสมบัติ | Crypto Map (Step 507) | VTI (Step นี้) |
|---|---|---|
| ต้องมี Crypto ACL แยก | ✅ ต้องมี (`match address`) | ❌ ไม่ต้อง — Traffic เข้า Interface = เข้ารหัสหมด |
| จำนวน Object ที่ต้องจัดการ | มาก (ACL + Crypto Map + Interface ผูก) | น้อย (แค่ IPsec Profile + Tunnel Interface) |
| Routing Protocol ผ่าน Tunnel ได้ไหม | ต้องพึ่ง GRE (Step 505-507) เพราะ Crypto Map ล้วนไม่รองรับ Multicast | ✅ ได้เลยในตัว (VTI Interface เป็น Point-to-Point ที่รองรับ Routing Protocol/Multicast ผ่านมันได้ตามปกติ — ไม่ต้องพึ่ง GRE ซ้อนอีกชั้น) |
| ใช้ QoS/ACL/Firewall Zone แยกสำหรับ Traffic ที่เข้ารหัสแล้ว | ยาก (ปนกับ Physical Interface) | ✅ ง่าย — Apply ตรงที่ Tunnel Interface ได้เหมือน Interface ปกติทุกประการ |
| Scale สำหรับหลาย Peer (Hub-and-Spoke) | ยาก — ต้องเพิ่ม ACL/Crypto Map Entry ทุกคู่ | ดีกว่า — แต่ละ Peer มี VTI ของตัวเอง จัดการอิสระ (และเป็นพื้นฐานของ DMVPN แบบ Multipoint GRE ใน Part 52) |
| ความนิยมใน Deployment ใหม่ | ลดลง (Legacy แต่ยังพบใน Config เก่า) | ✅ **แนะนำเป็นค่า Default สำหรับ Site-to-Site VPN ใหม่ทุกกรณี** |

### 508.3 คอนฟิกเต็มรูปแบบ: Pure IPsec VTI (ไม่มี GRE)

**ขั้นที่ 1 — IKEv2 Phase 1 + IPsec Transform-Set** (ใช้ Object เดิมจาก Step 506-507 ได้เลย
ถ้าต้องการรันคู่กับ Tunnel0 พร้อมกัน แนะนำให้สร้าง IKEv2 Profile ใหม่แยกเพื่อความชัดเจน):

```
WAN-EDGE-1(config)# crypto ikev2 profile IKEV2-PROFILE-VTI
WAN-EDGE-1(config-ikev2-profile)# match identity remote address 198.51.100.2 255.255.255.255
WAN-EDGE-1(config-ikev2-profile)# authentication local pre-share
WAN-EDGE-1(config-ikev2-profile)# authentication remote pre-share
WAN-EDGE-1(config-ikev2-profile)# keyring local IKEV2-KEYRING
WAN-EDGE-1(config-ikev2-profile)# exit
```

**ขั้นที่ 2 — IPsec Profile (Object ใหม่ที่ VTI ใช้แทน Crypto Map)**:

```
WAN-EDGE-1(config)# crypto ipsec profile IPSEC-PROFILE-VTI
WAN-EDGE-1(ipsec-profile)# set transform-set TSET-GRE
WAN-EDGE-1(ipsec-profile)# set ikev2-profile IKEV2-PROFILE-VTI
WAN-EDGE-1(ipsec-profile)# exit
```

> **`crypto ipsec profile` คือ Object ใหม่ที่แทนที่ `crypto map` สำหรับ VTI** — ทำหน้าที่
> เหมือนกัน (ผูก Transform-Set + IKEv2 Profile) แต่**ไม่มี `match address` (Crypto ACL) เลย**
> เพราะไม่จำเป็น — Traffic ทั้งหมดที่วิ่งเข้า Tunnel Interface ที่ผูก Profile นี้จะถูกเข้ารหัส
> โดยอัตโนมัติ 100% (Interface ทำหน้าที่เป็น "ACL" อยู่แล้วในตัวมันเอง)

**ขั้นที่ 3 — สร้าง Tunnel Interface โหมด IPsec (ไม่ใช่ GRE):**

```
WAN-EDGE-1(config)# interface Tunnel1
WAN-EDGE-1(config-if)# description ** IPsec VTI to BRANCH-RTR **
WAN-EDGE-1(config-if)# ip address 172.16.201.1 255.255.255.252
WAN-EDGE-1(config-if)# tunnel source GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# tunnel destination 198.51.100.2
WAN-EDGE-1(config-if)# tunnel mode ipsec ipv4
WAN-EDGE-1(config-if)# tunnel protection ipsec profile IPSEC-PROFILE-VTI
WAN-EDGE-1(config-if)# ip tcp adjust-mss 1360
WAN-EDGE-1(config-if)# exit
```

| คำสั่ง | ความหมาย |
|---|---|
| `tunnel mode ipsec ipv4` | **จุดต่างหลักจาก GRE** — บอกว่า Tunnel Interface นี้ **ไม่ใช้ GRE Encapsulation เลย** ห่อหุ้ม Packet ด้วย IPsec (ESP) ตรงๆ (ประหยัด Overhead 4 byte ของ GRE Header ที่ไม่จำเป็นถ้าไม่ต้องการ Multicast/Multiprotocol) |
| `tunnel protection ipsec profile IPSEC-PROFILE-VTI` | ผูก IPsec Profile เข้ากับ Tunnel Interface นี้โดยตรง — **นี่คือคำสั่งเดียวที่แทนที่ Crypto Map + Crypto ACL ทั้งชุดของ Step 507** |

**คอนฟิกฝั่ง BRANCH-RTR (ภาพสะท้อนกลับ)**: `crypto ikev2 profile IKEV2-PROFILE-VTI` ใช้
`match identity remote address 203.0.113.2`, `crypto ipsec profile IPSEC-PROFILE-VTI` ผูก
`transform-set TSET-GRE` + `ikev2-profile IKEV2-PROFILE-VTI` เหมือนกัน แล้วสร้าง
`interface Tunnel1` (`ip address 172.16.201.2/30`, `tunnel source GigabitEthernet0/0/1`,
`tunnel destination 203.0.113.2`, `tunnel mode ipsec ipv4`, `tunnel protection ipsec profile
IPSEC-PROFILE-VTI`, `ip tcp adjust-mss 1360`)

### 508.4 Verify VTI

```
WAN-EDGE-1# show interface Tunnel1
Tunnel1 is up, line protocol is up
  Internet address is 172.16.201.1/30
  Tunnel source 203.0.113.2, destination 198.51.100.2
  Tunnel protocol/transport IPSEC/IP
  Tunnel protection via IPSec (profile "IPSEC-PROFILE-VTI")

WAN-EDGE-1# ping 172.16.201.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.201.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/7/11 ms

WAN-EDGE-1# show crypto ipsec sa

interface: Tunnel1
    Crypto map tag: Tunnel1-head-0, local addr 203.0.113.2
   protected vrf: (none)
   local  ident (addr/mask/prot/port): (203.0.113.2/255.255.255.255/0/0)
   remote ident (addr/mask/prot/port): (198.51.100.2/255.255.255.255/0/0)
   current_peer 198.51.100.2 port 500
    #pkts encaps: 87, #pkts encrypt: 87, #pkts digest: 87
    #pkts decaps: 87, #pkts decrypt: 87, #pkts verify: 87
```

> **ข้อสังเกต**: `Tunnel protocol/transport IPSEC/IP` (ไม่มีคำว่า GRE เลย) และ `local/remote
> ident ... prot/port: .../0/0` (Protocol `0` = Any, ไม่จำกัดที่ GRE เหมือน Step 507 อีกแล้ว)
> ยืนยันว่า Tunnel1 เป็น **Pure IPsec Tunnel** — Traffic ทั้งหมดที่ Router ตัดสินใจ Route ผ่าน
> Tunnel1 (ตาม Routing Table ปกติ) จะถูกเข้ารหัสอัตโนมัติโดยไม่ต้องมี ACL ใดๆ มาระบุเพิ่มเติม

### 508.5 ข้อจำกัดที่ยังเหลืออยู่ของ Pure IPsec VTI

`tunnel mode ipsec ipv4` **รองรับเฉพาะ Unicast IP Traffic** — **ไม่รองรับ Multicast/Broadcast
โดยตรง** เหมือนที่อธิบายไว้ใน [Part 19 Step 189](part-019-wan-technologies.md) ดังนั้นถ้า
ต้องการรัน **OSPF/EIGRP ผ่าน Tunnel1 แบบนี้ตรงๆ จะไม่ทำงาน** (เพราะ Hello Packet เป็น
Multicast) — ทางเลือกคือ:

1. ใช้ **Static Route** ผ่าน Tunnel1 แทน Dynamic Routing Protocol (เพียงพอสำหรับ Site-to-Site
   คู่เดียวที่ไม่ซับซ้อน)
2. หรือกลับไปใช้ **GRE Tunnel (`tunnel mode gre ip`) + `tunnel protection ipsec profile`**
   แบบผสม — ได้ทั้ง GRE ที่รองรับ Multicast/Routing Protocol **และ** ความเรียบง่ายของ
   `tunnel protection` (ไม่ต้องมี Crypto Map/ACL) ในเวลาเดียวกัน — **นี่คือสิ่งที่ Step 510
   (Lab เต็มรูปแบบ) จะใช้จริง** เพื่อคง OSPF over Tunnel จาก Part 19 ไว้ พร้อมได้ความเรียบง่าย
   ของ VTI-style Configuration

```
Router(config)# interface Tunnel0
Router(config-if)# tunnel mode gre ip              ! <- ยังเป็น GRE (รองรับ Multicast/Routing Protocol)
Router(config-if)# tunnel protection ipsec profile IPSEC-PROFILE-VTI   ! <- แต่เข้ารหัสแบบ VTI-style
```

---

## Step 509 — FlexVPN: แนวคิดเบื้องต้นก่อนเข้าสู่ DMVPN

### 509.1 FlexVPN คืออะไร

**FlexVPN** คือ Framework ของ Cisco ที่**รวม IPsec VPN ทุกรูปแบบ** (Site-to-Site, Hub-and-
Spoke, Remote Access) **ไว้ภายใต้ชุดคำสั่งเดียวกันบนพื้นฐานของ IKEv2** — สร้างขึ้นมาแทนที่
ชุดคำสั่งเก่าที่แตกกระจาย (Classic Crypto Map, Easy VPN, DMVPN แบบ IKEv1 เดิม) ให้เหลือ
**Syntax เดียว (IKEv2 Profile/Keyring/Authorization) ที่ใช้ได้กับทุก VPN Topology** ทั้ง
**Site-to-Site VPN** (Step 507-508 ที่เรียนไปแล้วคือรากฐานเดียวกันนี้), **Hub-and-Spoke VPN**
(DMVPN Phase 3 ใน Part 52-53 ใช้ mGRE + NHRP + IKEv2 บน Spoke แต่ละตัว) และ **Remote Access
VPN** (AnyConnect + EAP — เกินขอบเขต ENCOR)

### 509.2 ทำไม Step 507-508 ที่เรียนไปแล้วคือ "FlexVPN เบื้องต้น" อยู่แล้ว

จุดสำคัญที่สุดที่ต้องเข้าใจ: **`crypto ikev2 profile` + `crypto ipsec profile` + `tunnel
protection ipsec profile` ที่คอนฟิกใน Step 508 คือ Building Block เดียวกันกับที่ FlexVPN
ใช้ทั้งหมด** — Cisco ไม่ได้แยก "FlexVPN Command Set" ออกมาเป็นคำสั่งใหม่ต่างหาก แต่ **FlexVPN
คือชื่อเรียกของสถาปัตยกรรมที่ประกอบขึ้นจาก Object เดิมเหล่านี้** เพียงแต่นำไปประยุกต์กับ
Topology ที่ซับซ้อนกว่า (Multipoint GRE, Dynamic Spoke-to-Spoke, RADIUS-based Authorization)

| สิ่งที่เรียนใน Step 508 (VTI พื้นฐาน) | สิ่งที่ FlexVPN/DMVPN Phase 3 เพิ่มเข้ามา (Part 52-53) |
|---|---|
| `crypto ikev2 profile` ต่อ Peer เดียว (Point-to-Point) | `crypto ikev2 profile` ที่ Match ได้กับ **Peer หลายตัวพร้อมกัน** (Wildcard/Certificate-based) |
| `interface Tunnel1` แบบ Point-to-Point (`tunnel destination` ระบุตายตัว) | **mGRE (Multipoint GRE)** — Tunnel Interface เดียวรองรับหลาย Spoke พร้อมกันโดยไม่ต้องระบุ `tunnel destination` ตายตัว |
| Routing แบบ Static หรือ OSPF ปกติ | **NHRP (Next Hop Resolution Protocol)** — ให้ Spoke ค้นหา Physical IP ของ Spoke อื่นแบบ Dynamic เพื่อสร้าง Spoke-to-Spoke Tunnel ได้เอง |
| Pre-Shared Key ต่อ Peer (Manual) | **IKEv2 Authorization + RADIUS/AAA** — แจก Policy ให้ Spoke ใหม่โดยอัตโนมัติตอน Join (Zero-Touch สำหรับ Spoke จำนวนมาก) |

### 509.3 ภาพรวมที่จะเรียนต่อใน Part 52-53

Part 51 (Point-to-Point GRE/VTI + IKEv2/IPsec, 1 Hub ↔ 1 Spoke) → Part 52 (DMVPN Phase 1-2:
mGRE + NHRP, Hub-and-Spoke แบบ Dynamic Spoke) → Part 53 (DMVPN Phase 3 + FlexVPN Server/Client:
Spoke-to-Spoke Dynamic, IKEv2 Authorization)

> **สรุป Step นี้**: FlexVPN ไม่ใช่เทคโนโลยีใหม่ที่ต้องเรียนจากศูนย์ — มันคือ**การขยาย Building
> Block ของ IKEv2 Profile/IPsec Profile ที่เรียนไปแล้วใน Step 506-508** ให้ทำงานกับ Topology
> ที่ซับซ้อนกว่า (Multipoint) เท่านั้น การเข้าใจ Step 506-508 อย่างแม่นยำคือกุญแจสำคัญที่สุด
> ก่อนเข้าสู่ DMVPN ใน Part 52-53

---

## Step 510 — Lab เต็มรูปแบบ: GUEST VRF + GRE-over-IPsec (VTI-Style) WAN-EDGE-1 ↔ BRANCH-RTR

### 510.1 ภาพรวม Lab

Lab นี้รวมทั้งสองหัวข้อหลักของ Part 51 เข้าด้วยกัน:

1. **VRF-Lite**: DIST-SW3/DIST-SW4 สร้าง VRF `GUEST` แยก **VLAN 40 (WIFI)** ออกจาก Global
   Routing Table โดยสมบูรณ์ พร้อม Route Leaking ให้ออก Internet ได้ทางเดียว (Step 502-504)
2. **Secured Tunnel**: WAN-EDGE-1 ↔ BRANCH-RTR ใช้ **GRE Tunnel0 เดิมจาก Part 19 (`tunnel mode
   gre ip`) ผสมกับ `tunnel protection ipsec profile`** (แนวทางที่แนะนำไว้ใน Step 508.5) เพื่อ
   ได้ทั้ง **Routing Protocol Support (OSPF)** และ **IPsec Encryption ที่คอนฟิกง่ายแบบ VTI**
   ในคำสั่งเดียว — ไม่ต้องมี Crypto Map/ACL แยกแบบ Step 507 อีกต่อไป

```
ISP-RTR ── 203.0.113.0/30 ── WAN-EDGE-1 (Lo0 1.1.1.21/32) ══ Tunnel0 (GRE+IPsec, 172.16.200.0/30) ══
    BRANCH-RTR (Lo0 172.20.0.1/32) via Gi0/0/1 198.51.100.2

CORE-SW1 (Global) ── 10.255.10.0/30 ── DIST-SW3 (Global + VRF GUEST: VLAN 40) ── ACCESS-SW3
    ── Guest Wi-Fi Client (10.10.40.x ผ่าน AP-1/AP-2) — Leak default-route ออกไปทาง CORE-SW1 → WAN-EDGE-1
```

### 510.2 IP Addressing สรุปทั้งหมดของ Lab นี้

| ส่วน | Device | Interface | IP/VRF | หมายเหตุ |
|---|---|---|---|---|
| VRF-Lite | DIST-SW3 | Vlan40 | 10.10.40.2/24 (VRF GUEST) | Guest Wi-Fi SVI |
| VRF-Lite | DIST-SW4 | Vlan40 | 10.10.40.3/24 (VRF GUEST) | คู่ HSRP |
| VRF-Lite | ทั้งคู่ | — | HSRP VIP 10.10.40.1 | ตาม [00-ip-address-plan.md](00-ip-address-plan.md) |
| Secured Tunnel | WAN-EDGE-1 | Tunnel0 | 172.16.200.1/30 | GRE + IPsec Protection |
| Secured Tunnel | WAN-EDGE-1 | Gi0/0/0 | 203.0.113.2/30 | Tunnel source (ตาม IP Plan หลัก) |
| Secured Tunnel | BRANCH-RTR | Tunnel0 | 172.16.200.2/30 | GRE + IPsec Protection |
| Secured Tunnel | BRANCH-RTR | Gi0/0/1 | 198.51.100.2/30 | Tunnel destination (จำลอง Public IP สาขา) |

### 510.3 Running-Config เต็มรูปแบบ — DIST-SW3 (ส่วน VRF-Lite)

```
hostname DIST-SW3
!
vrf definition GUEST
 description ** Guest Wi-Fi Isolation VRF - VLAN 40 **
 rd 65001:40
 address-family ipv4
  route-target export 65001:40
  route-target import 65001:40
 exit-address-family
!
ip access-list extended GUEST-INTERNET-ONLY
 deny   ip any 10.10.0.0 0.0.255.255
 permit ip any any
!
interface Vlan40
 description ** GUEST Wi-Fi SVI - Isolated VRF **
 vrf forwarding GUEST
 ip address 10.10.40.2 255.255.255.0
 ip access-group GUEST-INTERNET-ONLY in
 standby 40 ip 10.10.40.1
 standby 40 priority 110
 standby 40 preempt
 no shutdown
!
router ospf 2 vrf GUEST
 router-id 10.10.40.2
 network 10.10.40.0 0.0.0.255 area 0
!
ip route vrf GUEST 0.0.0.0 0.0.0.0 10.255.10.1 global
ip route 10.10.40.0 255.255.255.0 Vlan40 vrf GUEST
```

### 510.4 Running-Config เต็มรูปแบบ — WAN-EDGE-1 (ส่วน Secured Tunnel)

```
hostname WAN-EDGE-1
!
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 keyring IKEV2-KEYRING
 peer BRANCH-RTR
  address 198.51.100.2
  pre-shared-key Cisco@IPsec123
!
crypto ikev2 profile IKEV2-PROFILE-VTI
 match identity remote address 198.51.100.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local IKEV2-KEYRING
!
crypto ipsec transform-set TSET-GRE esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile IPSEC-PROFILE-VTI
 set transform-set TSET-GRE
 set ikev2-profile IKEV2-PROFILE-VTI
!
interface Loopback0
 description ** Router-ID / Management **
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** Link to ISP-RTR (Internet path for GRE/IPsec) **
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface Tunnel0
 description ** GRE Tunnel to BRANCH-RTR - secured via IPsec profile (VTI-style) **
 ip address 172.16.200.1 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 198.51.100.2
 tunnel mode gre ip
 tunnel protection ipsec profile IPSEC-PROFILE-VTI
 keepalive 5 3
 ip tcp adjust-mss 1360
!
router ospf 1
 router-id 1.1.1.21
 network 172.16.200.0 0.0.0.3 area 0
 passive-interface default
 no passive-interface Tunnel0
```

### 510.5 Running-Config เต็มรูปแบบ — BRANCH-RTR (ส่วน Secured Tunnel)

```
hostname BRANCH-RTR
!
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 keyring IKEV2-KEYRING
 peer WAN-EDGE-1
  address 203.0.113.2
  pre-shared-key Cisco@IPsec123
!
crypto ikev2 profile IKEV2-PROFILE-VTI
 match identity remote address 203.0.113.2 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local IKEV2-KEYRING
!
crypto ipsec transform-set TSET-GRE esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile IPSEC-PROFILE-VTI
 set transform-set TSET-GRE
 set ikev2-profile IKEV2-PROFILE-VTI
!
interface Loopback0
 description ** จำลอง LAN ของสาขา **
 ip address 172.20.0.1 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Link to Internet (จำลอง Public IP สาขา) **
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Tunnel0
 description ** GRE Tunnel to WAN-EDGE-1 (HQ) - secured via IPsec profile (VTI-style) **
 ip address 172.16.200.2 255.255.255.252
 tunnel source GigabitEthernet0/0/1
 tunnel destination 203.0.113.2
 tunnel mode gre ip
 tunnel protection ipsec profile IPSEC-PROFILE-VTI
 keepalive 5 3
 ip tcp adjust-mss 1360
!
router ospf 1
 router-id 172.20.0.1
 network 172.16.200.0 0.0.0.3 area 0
 network 172.20.0.1 0.0.0.0 area 0
 passive-interface default
 no passive-interface Tunnel0
```

### 510.6 Verification เต็มรูปแบบ

**1) ตรวจ VRF Isolation:**

```
DIST-SW3# show vrf
  Name                             Default RD            Protocols   Interfaces
  GUEST                            65001:40               ipv4        Vl40

DIST-SW3# show ip route vrf GUEST
Routing Table: GUEST
Gateway of last resort is 10.255.10.1 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 10.255.10.1 (Global)
C    10.10.40.0/24 is directly connected, Vlan40
L    10.10.40.2/32 is directly connected, Vlan40

DIST-SW3# ping vrf GUEST 10.10.30.1
Success rate is 0 percent (0/5)          <- ยืนยัน Isolation จาก Internal Subnet
```

**2) ตรวจ IKEv2/IPsec SA ของ Tunnel:**

```
WAN-EDGE-1# show crypto ikev2 sa
Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         203.0.113.2/500       198.51.100.2/500      none/none            READY

WAN-EDGE-1# show crypto ipsec sa

interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 203.0.113.2
   local  ident (addr/mask/prot/port): (203.0.113.2/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (198.51.100.2/255.255.255.255/47/0)
   current_peer 198.51.100.2 port 500
    #pkts encaps: 452, #pkts encrypt: 452, #pkts digest: 452
    #pkts decaps: 448, #pkts decrypt: 448, #pkts verify: 448
```

> **ข้อสังเกตสำคัญ**: `prot/port: .../47/0` (Protocol 47 = GRE) กลับมาปรากฏอีกครั้งเหมือน Step
> 507 — เพราะ Tunnel0 นี้ยังใช้ `tunnel mode gre ip` (มี GRE Header อยู่) เพียงแต่ **ไม่ต้อง
> เขียน Crypto ACL เอง** — IOS สร้าง Identity นี้ให้อัตโนมัติจาก `tunnel source`/`tunnel
> destination` ที่ตั้งไว้บน Interface โดยตรง (Interface `Tunnel0-head-0` แทนที่ Crypto Map
> ชื่อที่ตั้งเองใน Step 507 ทั้งหมด) — นี่คือ**ข้อพิสูจน์ที่ชัดเจนที่สุด**ว่า `tunnel protection
> ipsec profile` บน GRE Tunnel คือ "VTI-style Simplicity" ที่ยังคง GRE Encapsulation ไว้ครบ

**3) ตรวจ OSPF ยังทำงานผ่าน Tunnel ที่เข้ารหัสแล้ว:**

```
WAN-EDGE-1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
172.20.0.1        0   FULL/  -        00:00:38    172.16.200.2    Tunnel0

WAN-EDGE-1# show ip route ospf
     172.20.0.0/32 [110/1001] via 172.16.200.2, 00:12:04, Tunnel0
```

OSPF Neighbor ขึ้น `FULL` ผ่าน Tunnel0 ที่ตอนนี้ **เข้ารหัสด้วย IPsec เต็มรูปแบบแล้ว** — พิสูจน์
ว่าการเพิ่ม `tunnel protection` ไม่ได้กระทบ Routing Protocol ที่ Part 19 ตั้งไว้เลยแม้แต่น้อย

**4) Encrypted Ping Test ข้าม Tunnel (End-to-End):**

```
WAN-EDGE-1# ping 172.20.0.1 source Loopback0

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.20.0.1, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.21
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/9/14 ms

WAN-EDGE-1# show crypto ipsec sa | include pkts encaps
    #pkts encaps: 457, #pkts encrypt: 457, #pkts digest: 457
```

`#pkts encaps` เพิ่มขึ้นจาก `452` เป็น `457` (เพิ่ม 5 พอดีตามจำนวน ICMP Echo ที่ส่ง) —
ยืนยันแบบ 100% ว่า Ping ที่ผ่านไปสำเร็จนั้น **ถูกเข้ารหัสจริงทุก Packet** ไม่ใช่วิ่งผ่าน Path
อื่นที่ไม่เข้ารหัส

### 510.7 สรุปผลลัพธ์ Lab

| เป้าหมาย | ผลลัพธ์ที่พิสูจน์แล้ว |
|---|---|
| Guest Wi-Fi (VLAN 40) แยก Routing Table จาก Campus หลัก | ✅ `show ip route vrf GUEST` ไม่มี Route ของ Internal Subnet เลย, ping ตรงไม่ผ่าน |
| Guest Wi-Fi ยังออก Internet ได้ | ✅ Default Route ถูก Leak ผ่าน Keyword `global` ไปยัง Global Table สำเร็จ |
| GRE Tunnel (Part 19) ยังรองรับ OSPF/Routing Protocol | ✅ `show ip ospf neighbor` ขึ้น FULL ผ่าน Tunnel0 ตามปกติ |
| Traffic บน Tunnel เข้ารหัสจริงด้วย IKEv2/IPsec | ✅ `show crypto ipsec sa` แสดง SA พร้อม Packet Counter เพิ่มขึ้นตรงกับ Traffic จริง |
| Configuration เรียบง่ายกว่า Crypto Map แบบ Step 507 | ✅ ไม่มี Crypto Map/Crypto ACL แยก — ใช้ `tunnel protection ipsec profile` คำสั่งเดียว |

---

## แบบฝึกหัดทวนความเข้าใจ Part 51

1. VRF-Lite แตกต่างจาก Full/MPLS VRF อย่างไร และทำไม Enterprise Campus ส่วนใหญ่จึงใช้ VRF-Lite
   พอเพียงโดยไม่ต้องมี MPLS?
2. ทำไมหลังพิมพ์คำสั่ง `vrf forwarding <name>` บน Interface ที่มี `ip address` อยู่แล้ว ต้อง
   พิมพ์ `ip address` ซ้ำใหม่อีกครั้งเสมอ?
3. Static Route Leaking ด้วย Keyword `global`/`vrf` (Step 504.3) ต่างจาก Route Leaking ผ่าน
   Route-Target/MP-BGP (Step 504.2) อย่างไร และมีข้อจำกัดอะไรบ้าง?
4. อธิบายความแตกต่างสำคัญระหว่าง `tunnel mode gre ip` กับ `tunnel mode ipsec ipv4` — แบบไหน
   รองรับ Routing Protocol ผ่าน Tunnel ได้โดยตรง และทำไม?
5. `tunnel protection ipsec profile` ทำให้ Configuration ง่ายกว่า Crypto Map แบบดั้งเดิม
   (Step 507) อย่างไร และยังต้องมี Object ใดบ้างที่จำเป็นต้องคอนฟิกก่อนใช้คำสั่งนี้ได้?

**เฉลย:**

1. **VRF-Lite** ทำงานบน IP ล้วนๆ ไม่ต้องมี MPLS Label Switching และต้องคอนฟิกทีละ Hop
   (Hop-by-Hop) ด้วย Routing Protocol ธรรมดาต่อ VRF ส่วน **Full/MPLS VRF** ต้องมี MPLS
   Backbone และใช้ MP-BGP แจก VPN Route ข้าม PE อัตโนมัติพร้อม RD/RT — Enterprise Campus
   ส่วนใหญ่มีจำนวน Hop ไม่มาก (ไม่ใช่ระดับ SP Backbone) จึงไม่จำเป็นต้องแบก Complexity ของ
   MPLS/MP-BGP เพียงเพื่อแยก Routing Table ภายในองค์กรตัวเอง
2. เพราะ IP Address เดิมผูกอยู่กับ **Global Routing Table** — เมื่อสั่ง `vrf forwarding`
   IOS จะ**ลบ IP Address เดิมออกโดยอัตโนมัติทันที** (เพราะ IP นั้นเป็นของ Global Table
   ไม่ใช่ของ VRF ใหม่) และแสดง Warning ให้เห็น ต้องพิมพ์ `ip address` ใหม่เพื่อผูก IP เข้ากับ
   VRF ใหม่อย่างชัดเจน
3. **Static Route Leaking ด้วย `global`/`vrf`**: ทำงานได้แค่ **บนอุปกรณ์ตัวเดียว (Local)**
   ไม่ต้องมี MP-BGP ง่ายกว่าแต่ Scale ได้จำกัด — **Route-Target/MP-BGP**: ทำงานข้ามหลาย
   อุปกรณ์ได้ (Full Route Leaking Architecture) แต่ต้องมี BGP Address-Family ต่อ VRF ที่
   ซับซ้อนกว่า และเป็นรากฐานของ MPLS L3VPN เต็มรูปแบบที่จะสอนใน Part 86-87
4. **`tunnel mode gre ip`** ห่อหุ้ม Packet ด้วย GRE Header ก่อน ซึ่งรองรับ **Multicast/
   Broadcast** ทำให้ Routing Protocol อย่าง OSPF/EIGRP ที่ต้องส่ง Hello แบบ Multicast ทำงาน
   ผ่าน Tunnel ได้โดยตรง — **`tunnel mode ipsec ipv4`** ไม่มี GRE Header เข้ารหัสด้วย IPsec
   ตรงๆ ซึ่งรองรับแค่ **Unicast IP Traffic** เท่านั้น จึงต้องใช้ Static Route หรือรวมกับ GRE
   (`tunnel protection` บน GRE Tunnel) ถ้าต้องการ Dynamic Routing Protocol ผ่าน Tunnel
5. `tunnel protection ipsec profile` ผูก IPsec Profile เข้ากับ Tunnel Interface โดยตรง ทำให้
   **ไม่ต้องสร้าง Crypto ACL (`match address`) หรือ Crypto Map แยกต่างหาก** เหมือน Step 507
   — Traffic ทั้งหมดที่วิ่งเข้า Tunnel Interface นั้นถูกเข้ารหัสอัตโนมัติ อย่างไรก็ตาม ยังต้อง
   คอนฟิก **`crypto ikev2 profile`** (Phase 1: Identity + Authentication + Keyring) และ
   **`crypto ipsec profile`** (ผูก Transform-Set + IKEv2 Profile เข้าด้วยกัน) ให้เสร็จก่อน
   จึงจะนำไปอ้างอิงในคำสั่ง `tunnel protection` บน Interface ได้

---

## สรุป Part 51

Part นี้เติมเต็มสองหัวข้อที่ [Part 19](part-019-wan-technologies.md) และ
[Part 32](part-032-policy-based-routing.md) ค้างไว้อย่างตั้งใจ: **VRF-Lite** (แยก Routing Table
หลายชุดบนอุปกรณ์เดียวโดยไม่ต้องมี MPLS พร้อม Route Leaking แบบควบคุมได้) และ **IPsec แบบเต็ม
รูปแบบ** (IKEv1/IKEv2, ISAKMP Phase 1-2, Crypto Map แบบดั้งเดิม, และ VTI ที่เรียบง่ายกว่า) ✅
Lab สุดท้ายรวมทั้งสองเรื่องเข้าด้วยกัน: **GUEST VRF แยก Wi-Fi ออกจาก Campus** และ **GRE Tunnel
ของ Part 19 ที่ปกป้องด้วย IPsec แบบ VTI-Style** ระหว่าง WAN-EDGE-1 กับ BRANCH-RTR ที่ใช้งานได้
จริงและ Verify ผ่านทุกขั้นตอน

Building Block ของ IKEv2 Profile / IPsec Profile / Tunnel Protection ที่เรียนใน Part นี้คือ
**รากฐานที่จำเป็นที่สุด** ก่อนเข้าสู่ **DMVPN (Dynamic Multipoint VPN)** ใน Part ถัดไป ซึ่งจะ
ขยาย Point-to-Point Tunnel เดี่ยวๆ ที่เรียนมาให้กลายเป็น **Hub-and-Spoke แบบ Dynamic** ที่รองรับ
หลาย Spoke พร้อมกันโดยไม่ต้องคอนฟิก Tunnel คู่ต่อคู่ทีละ Site

**ไปต่อ:** [Part 52 — DMVPN Fundamentals →](part-052-dmvpn-fundamentals.md)
