# Part 88 — IPv6 Enterprise Deployment Deep Dive
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 871–880 จาก 1000 (ระดับ CCIE Enterprise Infrastructure)**

> ต่อจาก [Part 87 — MPLS L3VPN Advanced & WAN Integration](part-087-mpls-l3vpn-advanced-wan-integration.md)
> ที่ยกระดับ MPLS L3VPN ขึ้นสู่ Production (Route Reflector, Inter-AS VPN, Traffic Engineering,
> SD-WAN Integration) Part นี้จะพา **IPv6** ที่ [Part 8](part-008-ipv6-fundamentals.md) ปูพื้นฐาน
> ไว้ (Address Type, EUI-64, NDP, Static Routing) และ [Part 27](part-027-ospfv3-ipv6.md) ต่อยอด
> ด้วย OSPFv3 กลับมาอีกครั้ง — คราวนี้ในมุม **Production-Grade Enterprise Deployment** ที่ CCIE
> Blueprint คาดหวัง: จะเลือก Global Unicast Address จริงจาก ISP/RIR เมื่อไหร่, ACL/Security
> Policy สำหรับ IPv6 ที่ต้องระวัง ICMPv6, First-Hop Security (RA Guard, DHCPv6 Guard, IPv6
> Snooping), MLD Snooping, DHCPv6 แบบเต็มรูปแบบรวม Prefix Delegation, BGP สำหรับ IPv6, NAT66/
> NPTv6, สรุป Transition Mechanism ทั้งหมด และท้ายที่สุด — เครือข่ายจะไปสู่ IPv6-Only ได้จริงไหม
> ปิดท้ายด้วย Lab ใหญ่ที่รวมทุกฟีเจอร์เข้าด้วยกันบน ACCESS-SW1-4, DIST-SW1-4 และ WAN Edge ของ
> หลักสูตร

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 871 | IPv6 Addressing Strategy — GUA จาก ISP/RIR จริง vs ULA-only แบบที่ Lab ใช้มาตั้งแต่ Part 8 |
| 872 | IPv6 ใน ACL และ Security Policy — `ipv6 access-list`, ข้อควรระวังเรื่อง ICMPv6 |
| 873 | IPv6 First-Hop Security — RA Guard, DHCPv6 Guard, IPv6 Snooping/ND Inspection |
| 874 | IPv6 Multicast ระดับ Switch — MLD Snooping (คู่กับ IGMP Snooping จาก Part 41) |
| 875 | DHCPv6 เต็มรูปแบบ — Stateful DHCPv6 Server และ DHCPv6-PD (Prefix Delegation) |
| 876 | IPv6 ใน BGP — `address-family ipv6`, Dual-Stack BGP Peering แบบ Session เดียว vs แยก Session |
| 877 | IPv6 และ NAT — NAT66/NPTv6 (แนวคิดย่อ) |
| 878 | สรุป IPv6 Transition Mechanism — Dual-Stack, Tunneling, NAT64/DNS64 พร้อมตารางตัดสินใจ |
| 879 | IPv6-Only Network — อะไรพังถ้าถอด IPv4 ออกจริง และควรทำเมื่อไหร่ |
| 880 | Lab เต็มรูปแบบ: IPv6 Security Hardening + MLD Snooping + DHCPv6-PD + Dual-Stack BGP |

---

## Step 871 — IPv6 Addressing Strategy: GUA จาก ISP/RIR จริง vs ULA-only

### 871.1 ทวนสิ่งที่ Lab ทำมาตั้งแต่ Part 8

Lab ของหลักสูตรนี้เลือกใช้ **ULA (`fd00:10:10::/48`)** สำหรับทุก VLAN ภายใน และใช้ **GUA
Documentation Prefix (`2001:db8:203:113::/64`, RFC 3849)** จำลอง Public IPv6 เฉพาะที่ WAN Link
ระหว่าง WAN-EDGE-1 ↔ ISP-RTR เท่านั้น — โครงสร้างนี้ตั้งใจเลียนแบบการออกแบบ Enterprise จริง
**ทุกประการ** เพียงแต่สลับ Prefix จำลองเป็น Prefix ปลอมเพื่อความปลอดภัยในการสอน (เหมือนที่ IPv4
Lab ใช้ `203.0.113.0/24` และ `10.10.x.0/24` แทน Public/Private จริง)

### 871.2 การขอ GUA จริงจาก ISP หรือ RIR — 2 เส้นทาง

| เส้นทาง | เรียกว่า | ใครเป็นเจ้าของ Prefix | เหมาะกับ |
|---|---|---|---|
| **PA (Provider Aggregatable)** | ขอผ่าน ISP โดยตรง | ISP (Prefix นี้เป็นส่วนย่อยของ Block ใหญ่ของ ISP) | องค์กรทั่วไปที่ Single-Homed หรือ Multi-Homed แบบไม่ต้องการ Portability สูง |
| **PI (Provider Independent)** | ขอตรงจาก RIR (APNIC/ARIN/RIPE ฯลฯ) ผ่านการเป็น LIR/Member เอง | องค์กรเอง (Portable — เปลี่ยน ISP ได้โดยไม่ต้อง Renumber) | องค์กรที่ Multi-Homed ไป 2+ ISP และต้องมี ASN ของตัวเอง (ตรงกับ BGP Design ของ [Part 29/30](part-029-bgp-fundamentals.md)) |

**ขนาดที่ ISP/RIR จ่ายจริงตามคำแนะนำ RFC 6177** ("IPv6 Address Assignment to End Sites"):
มาตรฐานคือ **`/48` ต่อ Site** (ให้ 65,536 Subnet `/64` ต่อองค์กร — มากเกินพอสำหรับ Enterprise
ขนาดใหญ่ที่สุด) แม้บาง ISP ในทางปฏิบัติจะจ่ายแค่ `/56` หรือ `/60` ให้ลูกค้า SME ขนาดเล็กก็ตาม

```
ตัวอย่าง Global Routing Prefix ที่ ISP จ่ายจริง (สมมติ):  2001:db8:abcd::/48
                                                            └──────┬───────┘
                                                          Global Routing Prefix (48 bit)
แบ่ง Subnet ID ภายในองค์กร (16 bit) เหมือนที่ Lab ทำกับ ULA:
  2001:db8:abcd:0010::/64   ← เทียบเท่า fd00:10:10:10::/64 (VLAN 10 SALES)
  2001:db8:abcd:0020::/64   ← เทียบเท่า fd00:10:10:20::/64 (VLAN 20 VOICE)
  2001:db8:abcd:0030::/64   ← เทียบเท่า fd00:10:10:30::/64 (VLAN 30 SERVERS)
```

สังเกตว่า**โครงสร้างการแบ่ง Subnet เหมือนกับที่ Lab ทำกับ ULA เป๊ะ** — ถ้าจะย้าย Lab นี้ขึ้น
Production จริง สิ่งที่ต้องทำคือ **สลับ Prefix 48 bit แรก** จาก `fd00:10:10::/48` เป็น Prefix
ที่ ISP/RIR จ่ายมา โดย Subnet ID (`0010`, `0020`, ...) และ Interface ID ยังคงเดิมทุกประการ

### 871.3 เมื่อไหร่ต้องมี Public GUA จริง (ไม่พอแค่ ULA)

| สถานการณ์ | เหตุผลที่ต้องมี GUA |
|---|---|
| มี Service ที่ต้อง Internet-Facing โดยตรงแบบ Dual-Stack (Web/Mail/API Server ที่ต้องการให้ Client จาก Internet เข้าถึงผ่าน IPv6 ได้จริง) | IPv6 **ไม่มี NAT แบบ Many-to-One เหมือน IPv4 PAT** โดยปรัชญาการออกแบบ — Server ที่ต้อง Routable บน Internet ต้องมี GUA จริง จะใช้ ULA แล้วหวังพึ่ง NAT66 (Step 877) ก็ทำได้แต่ขัดกับเจตนาเดิมของ IPv6 และเพิ่มความซับซ้อนไม่จำเป็น |
| Multi-Homed ไป ISP 2 รายเพื่อ Redundancy ผ่าน BGP (ตรงกับ [Part 29/30](part-029-bgp-fundamentals.md) Design) | ต้องมี PI Space + ASN ของตัวเองเพื่อประกาศ Prefix เดียวกันออกทั้ง 2 ISP — ULA **Advertise ผ่าน BGP สู่ Internet ไม่ได้** เพราะไม่ Routable โดยนิยาม |
| ต้องเชื่อมต่อ Extranet/Partner ที่ต้องการ Address ที่ Global-Unique แน่นอน (ไม่ชนกับ ULA ของ Partner รายอื่นที่อาจสุ่ม Global ID ชนกัน) | แม้ ULA ออกแบบให้ชนกันยากด้วย Global ID 40 bit สุ่ม แต่ **ไม่มีการันตี Uniqueness แบบ Global เหมือน GUA** ที่มี IANA/RIR ควบคุมการจ่ายแบบรวมศูนย์ |
| ต้องรองรับ Cloud Provider สมัยใหม่ที่ Peer ผ่าน IPv6 GUA ตรง (Direct Connect/ExpressRoute บางประเภท) | Cloud Provider ส่วนใหญ่กำหนดให้ Peering ทำผ่าน GUA เท่านั้น ไม่รับ ULA |

### 871.4 เมื่อไหร่ ULA-only เพียงพอ (เหมือนที่ Lab ใช้)

| สถานการณ์ | เหตุผล |
|---|---|
| Lab/Test Environment ที่ไม่ต้องออก Internet จริงเลย | ตรงกับหลักสูตรนี้ — ไม่มีความเสี่ยงเรื่อง Uniqueness เพราะไม่เคย Merge กับ Network อื่น |
| Segment ที่ตั้งใจให้ไม่ Routable ออกนอกองค์กรโดยเด็ดขาด (เช่น OT/ICS, Backup/DR Replication Link ภายใน) | ULA ทำหน้าที่เหมือน RFC 1918 ของ IPv4 — เป็น "Air Gap ทาง Routing" ที่ตรวจสอบง่าย (เห็น Prefix `fd00::/8` แปลว่าไม่ควรออก Internet โดยธรรมชาติของ Design) |
| Internal-Only Infrastructure Link (Loopback, Backbone P2P) ที่ไม่มี Host เกาะเลย | ไม่มีเหตุผลต้อง "เปลือง" GUA ของ ISP กับ Link ที่ไม่มีใครต้องเข้าถึงจากนอกองค์กร |

> **ข้อควรรู้ระดับ CCIE — RFC 7381 (IPv6 Enterprise Network Deployment Guidelines)**: แนวทาง
> ของ Cisco/IETF **แนะนำให้ Enterprise ใช้ GUA ทั่วทั้งเครือข่ายเป็นค่าเริ่มต้น** (ไม่ใช่ ULA-only)
> แล้วควบคุม Reachability ด้วย **Firewall/ACL ที่ Perimeter** เหมือนที่ทำกับ IPv4 Public
> Address ภายใน DMZ — เพราะ ULA-only บังคับให้ต้องมี NPTv6 ทุกจุดที่ต้องออก Internet ซึ่งเพิ่ม
> State และความซับซ้อนโดยไม่ได้ประโยชน์ด้าน Security เพิ่มขึ้นจริง (Security มาจาก Firewall/ACL
> ไม่ใช่จาก NAT — บทเรียนเดียวกับที่ [Part 45](part-045-advanced-acl-zbfw.md) สอนเรื่อง ZBFW)
> — หลักสูตรนี้เลือก ULA เพื่อความง่ายในการสอนที่ไม่ต้องพึ่ง Prefix จริงจาก RIR เท่านั้น
> ในงาน Production จริงควรพิจารณา GUA ทั่วทั้งองค์กรตาม RFC 7381 เป็นทางเลือกแรก

---

## Step 872 — IPv6 ใน ACL และ Security Policy

### 872.1 Syntax `ipv6 access-list` — ต่างจาก IPv4 ACL อย่างไร

IPv6 รองรับ **เฉพาะ Named ACL เท่านั้น** (ไม่มี Numbered ACL แบบ `access-list 100` ของ
[Part 14](part-014-access-control-lists.md)) และ Apply ด้วยคำสั่ง `ipv6 traffic-filter`
(เทียบเท่า `ip access-group` ของ IPv4):

```
Router(config)# ipv6 access-list <name>
Router(config-ipv6-acl)# permit|deny <protocol> <source>/<len> [operator port] <destination>/<len> [operator port] [sequence <n>]
Router(config-ipv6-acl)# exit
Router(config)# interface <if>
Router(config-if)# ipv6 traffic-filter <name> {in|out}
```

| จุดต่าง | IPv4 ACL (Part 14) | IPv6 ACL |
|---|---|---|
| ประเภท | Numbered หรือ Named | **Named เท่านั้น** |
| คำสั่ง Apply บน Interface | `ip access-group <name/num> {in\|out}` | `ipv6 traffic-filter <name> {in\|out}` |
| Wildcard Mask | ใช้ Wildcard Mask (`0.0.0.255`) | ใช้ **Prefix Length** (`/64`) แทน ไม่มี Wildcard Mask |
| Implicit Deny ปลาย ACL | `deny ip any any` | `deny ipv6 any any` **แต่มี Implicit Permit สำหรับ NDP ก่อนถึงบรรทัดนี้เสมอ** (ดู 872.2) |

### 872.2 กับดักที่สำคัญที่สุด: ห้าม Block ICMPv6 แบบเหมาเหมือน ICMPv4

ผู้ดูแลระบบที่คุ้นกับ IPv4 มักมองว่า ICMP คือ "ping ที่ไม่ปลอดภัย ควร Block ทิ้ง" แล้วเขียน ACL
แบบ `deny icmp any any` ทับทุกอย่าง — **พฤติกรรมนี้ทำลาย IPv6 ทั้ง Subnet ทันที** เพราะ ICMPv6
ไม่ใช่แค่ Diagnostic Tool เหมือน ICMPv4 แต่เป็น **Transport ของฟังก์ชันหลักที่ IPv6 ขาดไม่ได้**
(NDP ทั้งหมดจาก [Part 8 Step 77](part-008-ipv6-fundamentals.md#step-77--neighbor-discovery-protocol-ndp)
วิ่งอยู่บน ICMPv6)

**IOS ป้องกันปัญหานี้ไว้ระดับหนึ่งด้วย Implicit Entry อัตโนมัติ** — ทุก IPv6 ACL ที่ Admin สร้าง
จะมี 2 บรรทัดถูกแทรกให้อัตโนมัติ **ก่อน** Implicit Deny เสมอ (มองไม่เห็นใน `show run` แต่เห็นได้
ใน `show ipv6 access-list`):

```
ipv6 access-list <name>
    ...ACE ที่ Admin เขียน...
    permit icmp any any nd-ns          <-- แทรกอัตโนมัติ (Neighbor Solicitation)
    permit icmp any any nd-na          <-- แทรกอัตโนมัติ (Neighbor Advertisement)
    deny ipv6 any any                  <-- Implicit Deny (เหมือน IPv4)
```

> แม้ IOS จะช่วย Permit NS/NA อัตโนมัติ แต่ **ไม่ครอบคลุม ICMPv6 Type อื่นที่สำคัญไม่แพ้กัน**
> เช่น RA/RS (Router Discovery) หรือ Packet-Too-Big (PMTUD) — ยังต้องเขียน Permit เองเสมอ
> ตามตาราง RFC 4890 ด้านล่าง

### 872.3 RFC 4890 — ICMPv6 Type ที่ "ต้องไม่ Block" ที่ Perimeter/Firewall

| ICMPv6 Type | ชื่อ | ทำไมห้าม Block |
|---|---|---|
| 1 | Destination Unreachable | Application จำเป็นต้องรู้ว่าปลายทางไปไม่ถึงเพื่อ Fail Fast |
| **2** | **Packet Too Big** | **สำคัญที่สุด** — คือกลไก PMTUD ทั้งหมดของ IPv6 (Router กลาง**ไม่ Fragment Packet แทน Host เหมือน IPv4**) ถ้า Block Type นี้ = Connection แบบ TCP ที่ใช้ MTU ใหญ่จะ **Silently Blackhole** (เชื่อมต่อได้แต่ค้างไม่มี Error ใดๆ — Troubleshoot ยากมาก) |
| 3 | Time Exceeded (Code 0) | Traceroute และ TTL/Hop-Limit Diagnostic ต้องพึ่ง Type นี้ |
| 4 | Parameter Problem (Code 1, 2) | แจ้ง Header Error ที่ Peer ต้องรู้ |
| 128/129 | Echo Request/Reply | Ping พื้นฐาน — Block ได้เฉพาะที่ Perimeter ขาเข้าจาก Internet ถ้าต้องการ (Policy Choice ไม่ใช่ Requirement) |
| **133/134/135/136** | RS/RA/NS/NA (NDP ทั้งหมด) | **ต้องเปิดเสมอภายใน Link เดียวกัน** — Block = Address Resolution/SLAAC พังทันที |
| 130/131/132/143 | MLD (Multicast Listener Discovery) | จำเป็นสำหรับ MLD Snooping (Step 874) และ PIM6 ([Part 83](part-083-enterprise-multicast-deep-dive.md)) |

> **กฎจำง่าย**: ICMPv6 Type 1-4 (Error Message) และ 133-137 (NDP) **ต้องเปิดเสมอ** ไม่ว่า ACL
> จะเข้มงวดแค่ไหน — สิ่งที่ควร Block ตาม RFC 4890 จริงๆ มีแค่กลุ่มที่ไม่จำเป็นข้าม Perimeter เช่น
> Node Information Query (Type 139/140) หรือ Redirect (Type 137) ที่ควร Block เฉพาะทิศทางเข้า
> จาก Internet เท่านั้น (ไม่ควรมี Router ภายนอกมา Redirect Traffic ของเราเอง)

### 872.4 Worked Example — ขยาย ACL Skill จาก Part 14/45 มาเป็น IPv6 บน WAN-EDGE-1

```
! ===== WAN-EDGE-1 — IPv6 Perimeter ACL (ขาเข้าจาก ISP-RTR) =====
WAN-EDGE-1(config)# ipv6 access-list INTERNET-IN-V6
WAN-EDGE-1(config-ipv6-acl)# permit icmp any any packet-too-big
WAN-EDGE-1(config-ipv6-acl)# permit icmp any any time-exceeded
WAN-EDGE-1(config-ipv6-acl)# permit icmp any any unreachable
WAN-EDGE-1(config-ipv6-acl)# deny icmp any any router-renumbering
WAN-EDGE-1(config-ipv6-acl)# permit tcp any host 2001:DB8:30:30::10 eq 443
WAN-EDGE-1(config-ipv6-acl)# permit tcp any host 2001:DB8:30:30::10 eq 80
WAN-EDGE-1(config-ipv6-acl)# permit tcp any any established
WAN-EDGE-1(config-ipv6-acl)# deny ipv6 any any log
WAN-EDGE-1(config-ipv6-acl)# exit
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# ipv6 traffic-filter INTERNET-IN-V6 in
```

สังเกตว่า **ไม่ต้อง Permit NS/NA เอง** (IOS แทรกให้อัตโนมัติตาม 872.2) แต่ **ต้อง Permit
Packet-Too-Big/Time-Exceeded/Unreachable เอง** เพราะ IOS ไม่ได้แทรกให้อัตโนมัติเหมือน NDP —
นี่คือจุดที่วิศวกรจำนวนมากพลาดตอน Migrate Security Policy จาก IPv4 มา IPv6 แบบรีบเร่ง

### 872.5 ZBFW กับ IPv6 — ข่าวดี: Policy เดียวใช้ได้ทั้งสอง AF

**IOS Zone-Based Firewall** จาก [Part 45](part-045-advanced-acl-zbfw.md) ไม่ได้แยก Data Plane
ระหว่าง IPv4/IPv6 — `zone-member security` ผูกกับ Interface (Dual-Stack) ไม่ใช่ผูกกับ Protocol
และ `class-map type inspect` ที่ Match ด้วย Protocol Keyword (เช่น `match protocol tcp`) จะจับ
ทั้ง IPv4 และ IPv6 Traffic โดยอัตโนมัติในเงื่อนไขเดียว — ไม่ต้องสร้าง Zone-Pair หรือ Policy-Map
แยกชุดสำหรับ IPv6 เลย ถ้า Class-Map ไม่ได้ Match ด้วย ACL ที่ระบุ Address เฉพาะ Family ใด
Family หนึ่ง (ถ้าต้องการ Exception เฉพาะ IPv6 จึงค่อยแนบ `match access-group name <ipv6-acl>`
เพิ่มเป็น Class แยก)

---

## Step 873 — IPv6 First-Hop Security: RA Guard, DHCPv6 Guard, IPv6 Snooping/ND Inspection

### 873.1 ภาพรวม: ทำไมต้องมี First-Hop Security เวอร์ชัน IPv6

[Part 21](part-021-switch-security.md) สอน **DHCP Snooping** (ป้องกัน Rogue DHCP Server) และ
**Dynamic ARP Inspection** (ป้องกัน ARP Spoofing) สำหรับ IPv4 — IPv6 มีปัญหาแบบเดียวกันแต่ผ่าน
Protocol คนละตัว (NDP/DHCPv6 แทน ARP/DHCP) จึงต้องมีฟีเจอร์คู่ขนานชุดใหม่ที่เรียกรวมว่า
**IPv6 First-Hop Security (FHS)**:

| ปัญหา (IPv4, Part 21) | กลไกป้องกัน (IPv4) | ปัญหาเทียบเท่า (IPv6) | กลไกป้องกัน (IPv6) |
|---|---|---|---|
| Rogue DHCP Server แจก IP/Gateway ปลอม | DHCP Snooping | **Rogue Router Advertisement (RA)** — Host/Attacker ปลอมส่ง RA ประกาศตัวเป็น Default Gateway หรือประกาศ Prefix ปลอม ทำให้ Host ทั้ง Subnet คำนวณ Address ผิดหรือส่ง Traffic ผ่าน Attacker | **RA Guard** |
| — (ไม่มีเทียบเท่าตรง เพราะ IPv4 DHCP Snooping คุมทั้ง Server) | — | **Rogue DHCPv6 Server** แจก Address/Option ปลอมผ่าน Stateful DHCPv6 | **DHCPv6 Guard** |
| ARP Spoofing/Poisoning | Dynamic ARP Inspection (DAI) | **NDP Spoofing** — ปลอม Neighbor Advertisement (NA) เพื่อ Poison Neighbor Cache เทียบเท่า ARP Cache | **IPv6 ND Inspection** (ผ่าน IPv6 Snooping Policy) |

### 873.2 RA Guard — ป้องกัน Rogue Router Advertisement

**หลักการ**: กำหนด **Device-Role** ให้แต่ละพอร์ต — พอร์ตที่เป็น **Host** (Access Port ปลาย
User) จะไม่ได้รับอนุญาตให้ส่ง RA เข้ามาเลย (RA ที่มาจากพอร์ตนี้ถูก Drop ทันที ไม่ว่าเนื้อหาจะ
ถูกต้องหรือไม่) มีแค่พอร์ตที่ตั้งเป็น **Router** (Trusted, ปกติคือ Uplink ไป DIST-SW ที่เป็น
IPv6 Gateway จริง) เท่านั้นที่ RA ผ่านได้:

```
! ===== ACCESS-SW1 — RA Guard Policy =====
ACCESS-SW1(config)# ipv6 nd raguard policy HOST-PORT-POLICY
ACCESS-SW1(config-nd-raguard)# device-role host
ACCESS-SW1(config-nd-raguard)# exit
!
ACCESS-SW1(config)# ipv6 nd raguard policy UPLINK-POLICY
ACCESS-SW1(config-nd-raguard)# device-role router
ACCESS-SW1(config-nd-raguard)# trusted-port
ACCESS-SW1(config-nd-raguard)# exit
!
! ----- Apply: Access Port (User) = Host Role -----
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ipv6 nd raguard attach-policy HOST-PORT-POLICY
ACCESS-SW1(config-if-range)# exit
!
! ----- Apply: Uplink ไป DIST-SW1 = Router Role (Trusted) -----
ACCESS-SW1(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW1(config-if-range)# ipv6 nd raguard attach-policy UPLINK-POLICY
```

> **เทียบกับ DHCP Snooping**: `device-role host` บน Access Port ทำหน้าที่เหมือน Port ที่**ไม่ได้
> ตั้ง** `ip dhcp snooping trust` (Default = Untrusted) — ส่วน `trusted-port` บน Uplink ทำหน้าที่
> เหมือน `ip dhcp snooping trust` บน DHCP Snooping ทุกประการ เพียงแค่สลับจาก "DHCP Server ปลอม"
> เป็น "Router ปลอมที่ส่ง RA"

### 873.3 DHCPv6 Guard — ป้องกัน Rogue DHCPv6 Server

หลักการเดียวกับ DHCP Snooping ของ IPv4 แต่ใช้ Syntax แยกชุด (`ipv6 dhcp guard`) เพราะเป็น
Feature คนละตัวจาก RA Guard (คุม DHCPv6 Reply ไม่ใช่ RA):

```
! ===== ACCESS-SW1 — DHCPv6 Guard Policy =====
ACCESS-SW1(config)# ipv6 dhcp guard policy CLIENT-PORT-POLICY
ACCESS-SW1(config-dhcp-guard)# device-role client
ACCESS-SW1(config-dhcp-guard)# exit
!
ACCESS-SW1(config)# ipv6 dhcp guard policy SERVER-UPLINK-POLICY
ACCESS-SW1(config-dhcp-guard)# device-role server
ACCESS-SW1(config-dhcp-guard)# trusted-port
ACCESS-SW1(config-dhcp-guard)# exit
!
! ----- Apply: Access Port = Client Role (ปฏิเสธ DHCPv6 Reply ที่มาจากพอร์ตนี้ทั้งหมด) -----
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ipv6 dhcp guard attach-policy CLIENT-PORT-POLICY
ACCESS-SW1(config-if-range)# exit
!
! ----- Apply: Uplink = Server Role (Trusted, เพราะ DHCPv6 Server จริงอยู่ที่ DIST-SW ตาม Step 875) -----
ACCESS-SW1(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW1(config-if-range)# ipv6 dhcp guard attach-policy SERVER-UPLINK-POLICY
```

### 873.4 IPv6 Snooping Policy — สร้าง Binding Table + ND Inspection (เทียบเท่า DAI)

**IPv6 Snooping** เป็น Framework กลางที่สร้าง **Binding Table** (จับคู่ IPv6 Address ↔ MAC ↔
Port ↔ VLAN, เทียบเท่า DHCP Snooping Binding Table ของ IPv4) จากการดักฟัง DHCPv6 และ NDP —
Binding Table นี้เองที่ **IPv6 ND Inspection** ใช้ตรวจสอบ NS/NA ทุกใบที่ผ่านพอร์ต Untrusted
(เทียบเท่า DAI ที่พึ่ง DHCP Snooping Binding Table):

```
! ===== ACCESS-SW1 — IPv6 Snooping Policy =====
ACCESS-SW1(config)# ipv6 snooping policy ACCESS-PORT-POLICY
ACCESS-SW1(config-ipv6-snooping)# protocol dhcp
ACCESS-SW1(config-ipv6-snooping)# protocol ndp
ACCESS-SW1(config-ipv6-snooping)# security-level guard
ACCESS-SW1(config-ipv6-snooping)# tracking enable reachable-lifetime 30
ACCESS-SW1(config-ipv6-snooping)# exit
!
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ipv6 snooping attach-policy ACCESS-PORT-POLICY
```

| Parameter | ความหมาย |
|---|---|
| `protocol dhcp` / `protocol ndp` | เรียนรู้ Binding จากการดักฟังทั้ง DHCPv6 Exchange และ NDP (NS/NA) |
| `security-level guard` | ระดับสูงสุด — ตรวจสอบเข้มงวดเต็มรูปแบบ (ต่างจาก `inspect` ที่แค่ Log ไม่ Drop) |
| `tracking enable reachable-lifetime <sec>` | อายุของ Binding Entry ก่อนต้อง Re-verify (คล้าย ND Reachable Time จาก [Part 8](part-008-ipv6-fundamentals.md)) |

### 873.5 Verification

```
ACCESS-SW1# show ipv6 nd raguard policy UPLINK-POLICY
Target      RA-guard Feature
Policy      UPLINK-POLICY
Device-role router
Trusted-port

ACCESS-SW1# show ipv6 snooping policy ACCESS-PORT-POLICY

ACCESS-SW1# show ipv6 neighbors binding
Codes: L - Local, S - Static, ND - Neighbor Discovery, DH - DHCP
       R - Recursive, PS - Prefix suppressed, ...

Network Layer Address                     Age      State  Interface  MAC Address     VLAN
FD00:10:10:10::101                        00:12:41 REACH  Gi1/0/1    0050.7966.6801  10

ACCESS-SW1# show ipv6 dhcp guard policy SERVER-UPLINK-POLICY
```

> **ข้อควรระวังเรื่อง Platform**: RA Guard/DHCPv6 Guard/IPv6 Snooping ต้องอาศัย **SISF (Switch
> Integrated Security Features)** ซึ่งมีเฉพาะ IOS-XE รุ่นใหม่บน Catalyst 9000 — ตาม
> [00-ip-address-plan.md](00-ip-address-plan.md) ACCESS-SW1-4 ของ Lab นี้ระบุเป็น "Catalyst
> 9200/2960" ซึ่งถ้าเป็นรุ่น **2960 (Legacy) จะไม่รองรับฟีเจอร์เหล่านี้เลย** — Feature เต็ม
> รูปแบบนี้ใช้ได้เฉพาะเมื่อ ACCESS-SW เป็น **Catalyst 9200** เท่านั้น เป็นเหตุผลสำคัญที่ CCIE
> ต้องระวังตอนเลือก Platform สำหรับ Design ที่ต้องการ IPv6 FHS เต็มรูปแบบ

---

## Step 874 — MLD Snooping: IGMP Snooping เวอร์ชัน IPv6

### 874.1 ทวนจาก Part 41 และ Part 83

[Part 41 Step 403](part-041-multicast-fundamentals.md#step-403--igmp-internet-group-management-protocol)
สอน **IGMP Snooping** — ฟีเจอร์ระดับ Switch ที่ป้องกันไม่ให้ Multicast Traffic ถูก Flood ไปทุก
Port ของ VLAN (ฟัง IGMP Report/Leave แล้วส่ง Traffic ไปแค่ Port ที่มี Receiver Join จริง)
[Part 83 Step 824](part-083-enterprise-multicast-deep-dive.md#step-824--ipv6-multicast-mld-และ-pim6)
เพิ่ม **MLD และ PIM6** ในระดับ **Routing** (Layer 3, เทียบเท่า IGMP+PIM) แต่**ยังไม่เคย Config
MLD Snooping ระดับ Switch (Layer 2) เลย** — Step นี้ปิด Gap นั้น

### 874.2 เปรียบเทียบ IGMP Snooping vs MLD Snooping

| คุณสมบัติ | IGMP Snooping (IPv4, Part 41) | MLD Snooping (IPv6) |
|---|---|---|
| ดักฟัง Message อะไร | IGMP Report/Leave | **MLD Report/Done** (ผ่าน ICMPv6 ตาม [Part 8](part-008-ipv6-fundamentals.md)) |
| คำสั่งเปิด Global | `ip igmp snooping` | `ipv6 mld snooping` |
| คำสั่งเปิดต่อ VLAN | `ip igmp snooping vlan <id>` | `ipv6 mld snooping vlan <id>` |
| Default บน Catalyst รุ่นใหม่ | เปิด Global อยู่แล้วบางรุ่น | **ปิดเป็น Default เสมอ ต้องเปิดเอง** |
| ต้องมี Querier ไหม | ต้องมี (Static หรือ PIM Router) | ต้องมี (Static `ipv6 mld snooping querier` หรือ PIM6 Router) |
| Mrouter Port Learning | ผ่าน PIM Hello หรือกำหนด Static | ผ่าน PIM6 Hello (`ff02::d`) หรือกำหนด Static เหมือนกัน |

### 874.3 Worked Config — เปิด MLD Snooping ให้ STREAM-SRV (VLAN 30, ต่อจาก Part 83)

ต่อยอด Topology เดียวกับ [Part 83 Step 824.3](part-083-enterprise-multicast-deep-dive.md) ที่
STREAM-SRV มี IPv6 Address `fd00:10:10:30::30` และกระจาย Group `ff05::239:1:1` — คราวนี้เปิด
MLD Snooping ที่ **ACCESS-SW3** (ปลาย Server) และ **ACCESS-SW1/ACCESS-SW4** (ปลาย Receiver)
เพื่อไม่ให้ Switch Flood Multicast Frame ไปทุก Port โดยไม่จำเป็น:

```
! ===== ACCESS-SW3 (VLAN 30 SERVERS — ฝั่ง STREAM-SRV) =====
ACCESS-SW3(config)# ipv6 mld snooping
ACCESS-SW3(config)# ipv6 mld snooping vlan 30
ACCESS-SW3(config)# ipv6 mld snooping vlan 40
!
! ----- กำหนด Uplink ไป DIST-SW3 เป็น Mrouter Port ชัดเจน (PIM6 Router อยู่ปลายนั้น) -----
ACCESS-SW3(config)# ipv6 mld snooping vlan 30 mrouter interface GigabitEthernet0/1

! ===== ACCESS-SW1 (VLAN 10 SALES — ฝั่ง Receiver) =====
ACCESS-SW1(config)# ipv6 mld snooping
ACCESS-SW1(config)# ipv6 mld snooping vlan 10
ACCESS-SW1(config)# ipv6 mld snooping vlan 20
ACCESS-SW1(config)# ipv6 mld snooping vlan 10 mrouter interface GigabitEthernet0/1

! ===== ACCESS-SW4 (VLAN 40 WIFI — ฝั่ง Receiver) =====
ACCESS-SW4(config)# ipv6 mld snooping
ACCESS-SW4(config)# ipv6 mld snooping vlan 40
ACCESS-SW4(config)# ipv6 mld snooping vlan 40 mrouter interface GigabitEthernet0/1
```

### 874.4 Verification

```
ACCESS-SW1# show ipv6 mld snooping vlan 10
Vlan 10:
--------
MLD snooping                     : Enabled
MLDv2 immediate leave             : Disabled
Explicit host tracking            : Enabled
Multicast router learning mode    : pim-dvmrp
Source only learning age timer    : 10
Leave mode                        : Fast

ACCESS-SW1# show ipv6 mld snooping mrouter vlan 10
Vlan    Ports
----    -----
10      Gi0/1(dynamic)

ACCESS-SW1# show ipv6 mld snooping groups
Vlan     Group                                    Type    Version  Port List
10       FF05::239:1:1                            D       v2       Gi1/0/1
```

`show ipv6 mld snooping groups` ยืนยันว่า Switch ส่ง Multicast Frame ของ Group `ff05::239:1:1`
ไปแค่ `Gi1/0/1` (Port ที่มี Receiver Join จริง) **ไม่ Flood ไปทุก Port ของ VLAN 10** — ทำงาน
คู่กับ `show ip igmp snooping groups` ของ IPv4 (Part 41) แบบ Dual-Stack สมบูรณ์

---

## Step 875 — DHCPv6 เต็มรูปแบบ: Stateful Server และ DHCPv6-PD

### 875.1 ทวนจาก Part 8 และสิ่งที่ยังไม่ได้ทำ

[Part 8 Step 75](part-008-ipv6-fundamentals.md#step-75--วิธีกำหนด-ipv6-address-บนอุปกรณ์host)
อธิบาย Concept ของ M/O Flag และบอกไว้ว่า "รายละเอียดเต็ม (Server Config, Relay, Pool) จะเรียน
ต่อเนื่องกับ DHCP ของ IPv4 ใน Part 16" — แต่ [Part 16](part-016-dhcp-dns.md) เองก็แค่แนะนำ
Concept SLAAC vs Stateful DHCPv6 (Step 158) โดยไม่ได้ Config Pool จริง — Step นี้ปิด Gap ทั้งสอง
Part ด้วย Config เต็มรูปแบบ

### 875.2 Stateful DHCPv6 Server — Worked Config บน DIST-SW1 (VLAN 10 SALES)

```
! ===== DIST-SW1 — Stateful DHCPv6 Pool สำหรับ VLAN 10 =====
DIST-SW1(config)# ipv6 dhcp pool SALES-V6-POOL
DIST-SW1(config-dhcpv6)# address prefix fd00:10:10:10::/64 lifetime 86400 43200
DIST-SW1(config-dhcpv6)# dns-server fd00:10:10:30::30
DIST-SW1(config-dhcpv6)# domain-name cisco-lab.local
DIST-SW1(config-dhcpv6)# exit
!
DIST-SW1(config)# interface Vlan10
DIST-SW1(config-if)# ipv6 nd managed-config-flag
DIST-SW1(config-if)# ipv6 nd other-config-flag
DIST-SW1(config-if)# ipv6 dhcp server SALES-V6-POOL
```

| Parameter | ความหมาย |
|---|---|
| `address prefix <prefix>/64 lifetime <valid> <preferred>` | Pool ของ Address ที่แจกได้ (Valid Lifetime 86400s = 24 ชม., Preferred 43200s = 12 ชม. — คล้าย DHCP Lease ของ IPv4 แต่มี 2 Timer ต่างจาก IPv4 ที่มี Timer เดียว) |
| `dns-server` / `domain-name` | Option เพิ่มเติมที่ Stateful DHCPv6 แจกได้ (เหมือน DHCPv4) |
| `ipv6 nd managed-config-flag` | ตั้ง **M-flag = 1** ใน RA — บอก Host ว่า "ไปขอ Address แบบ Stateful จาก DHCPv6 Server เท่านั้น อย่าใช้ SLAAC" |
| `ipv6 nd other-config-flag` | ตั้ง **O-flag = 1** — บอก Host ว่ามี Option เพิ่มเติม (DNS ฯลฯ) รอให้ไปขอจาก DHCPv6 |
| `ipv6 dhcp server <pool>` | Bind Pool เข้ากับ Interface นี้ (ทำหน้าที่ DHCPv6 Server ตรง ต่างจาก Relay) |

### 875.3 Verification

```
DIST-SW1# show ipv6 dhcp pool
DHCPv6 pool: SALES-V6-POOL
 Address allocation prefix: FD00:10:10:10::/64 valid 86400 preferred 43200 (2 in use, 0 conflicts)
 DNS server: FD00:10:10:30::30
 Domain name: cisco-lab.local
 Active clients: 2

DIST-SW1# show ipv6 dhcp binding
Client: FE80::A00:27FF:FE12:3456
  DUID: 00030001080027123456
  Address: FD00:10:10:10::1000
    preferred lifetime 43200, valid lifetime 86400
```

### 875.4 DHCPv6-PD (Prefix Delegation) — แจก "Prefix ทั้งก้อน" ไม่ใช่แค่ Address เดียว

**DHCPv6-PD (RFC 8415)** ต่างจาก Stateful DHCPv6 ปกติตรงที่แจก **Prefix ขนาดใหญ่ทั้งก้อน**
(เช่น `/60` ที่มี 16 Subnet `/64` ย่อยอยู่ในนั้น) ให้กับ **Router ปลายทาง** เพื่อให้ Router
ปลายทางเอาไป Subnet ต่อและแจก SLAAC/DHCPv6 ให้ Host ของตัวเองอีกชั้น — ใช้บ่อยที่สุดในสถานการณ์
**ISP → CPE Router** และ **HQ → Branch Router** (Router ที่ Branch ไม่ต้องรอ Admin มา Assign
Prefix ให้ทีละ Site ด้วยมือ)

```
IA_PD Request/Reply (แนวคิด):

Branch Router (PD Client)              WAN-EDGE-1 (PD Server)
        │  Solicit (IA_PD)                     │
        │  "ขอ Prefix มาให้ฉันแจกต่อหน่อย"        │
        │  ─────────────────────────────────►   │
        │                                       │
        │  Advertise/Reply: Prefix = fd00:10:11:20::/60
        │  ◄─────────────────────────────────   │
        │                                       │
  Branch Router เอา /60 นี้ไปแบ่ง /64 ต่อให้ LAN ของตัวเอง
  (fd00:10:11:20::/64, fd00:10:11:21::/64, ...)
```

> **หมายเหตุ Address Plan**: [00-ip-address-plan.md](00-ip-address-plan.md) ไม่ได้กันพื้นที่
> สำหรับ PD ไว้ — Step นี้กำหนดส่วนขยายเฉพาะ Part 88 เพิ่มเติม (Convention เดียวกับที่
> [Part 83](part-083-enterprise-multicast-deep-dive.md) เพิ่ม Loopback0 IPv6): สำรอง
> **`fd00:10:11::/48`** ไว้เป็น Pool กลางสำหรับ PD ไปยัง Branch/Downstream Router โดยเฉพาะ
> แยกจาก `fd00:10:10::/48` ที่ใช้กับ VLAN หลักของ Lab

**Worked Config — WAN-EDGE-1 เป็น PD Server ให้ Branch Router**:

```
! ===== WAN-EDGE-1 — DHCPv6-PD Server =====
WAN-EDGE-1(config)# ipv6 local pool BRANCH-PD-POOL fd00:10:11::/48 60
WAN-EDGE-1(config)# ipv6 dhcp pool PD-DELEGATION-POOL
WAN-EDGE-1(config-dhcpv6)# prefix-delegation pool BRANCH-PD-POOL lifetime 604800 302400
WAN-EDGE-1(config-dhcpv6)# exit
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# description ** Link to BRANCH-RTR (PD Client) **
WAN-EDGE-1(config-if)# ipv6 address fd00:10:11:ffff::1/64
WAN-EDGE-1(config-if)# ipv6 dhcp server PD-DELEGATION-POOL
```

`ipv6 local pool BRANCH-PD-POOL fd00:10:11::/48 60` สร้าง Pool ของ Prefix ขนาด **`/60`** ย่อย
จาก Block ใหญ่ `/48` — คำนวณได้ 4,096 Prefix `/60` ที่แจกให้แต่ละ Branch ได้ (2^(60-48) = 4096)

**Worked Config — Branch Router เป็น PD Client**:

```
! ===== BRANCH-RTR — DHCPv6-PD Client =====
BRANCH-RTR(config)# interface GigabitEthernet0/0
BRANCH-RTR(config-if)# description ** WAN Uplink ไป WAN-EDGE-1 **
BRANCH-RTR(config-if)# ipv6 address autoconfig
BRANCH-RTR(config-if)# ipv6 dhcp client pd BRANCH-PREFIX
BRANCH-RTR(config-if)# exit
!
BRANCH-RTR(config)# interface GigabitEthernet0/1
BRANCH-RTR(config-if)# description ** LAN Branch — ใช้ Prefix ที่ได้รับจาก PD **
BRANCH-RTR(config-if)# ipv6 address BRANCH-PREFIX ::1/64
BRANCH-RTR(config-if)# no shutdown
```

`ipv6 dhcp client pd BRANCH-PREFIX` ขอ Prefix จาก WAN-EDGE-1 แล้วเก็บไว้ในชื่อ Local
`BRANCH-PREFIX` — Interface LAN (`Gi0/1`) จึงอ้างชื่อนี้แทนการพิมพ์ Prefix เต็ม (`ipv6 address
BRANCH-PREFIX ::1/64` = เอา Prefix ที่ได้จาก PD มาต่อ Host Part `::1`) **ถ้า WAN-EDGE-1 แจก
Prefix ใหม่ (เช่นตอน Renew หรือ Failover) Interface LAN จะได้ Address ใหม่ตามอัตโนมัติทันที
โดยไม่ต้องแก้ Config มือเลย** — นี่คือข้อได้เปรียบหลักของ PD เทียบกับ Static Assignment

### 875.5 Verification DHCPv6-PD

```
WAN-EDGE-1# show ipv6 dhcp pool
DHCPv6 pool: PD-DELEGATION-POOL
 Prefix pool: BRANCH-PD-POOL
 Preferred/valid lifetimes: 302400/604800
 Active clients: 1

BRANCH-RTR# show ipv6 dhcp interface GigabitEthernet0/0
GigabitEthernet0/0 is in client mode
 Prefix State is OPEN
 Renew for address will be sent in 3d12h
 Prefix name: BRANCH-PREFIX
 Prefix FD00:10:11:20::/60 valid lifetime 604800 preferred lifetime 302400

BRANCH-RTR# show ipv6 interface brief
GigabitEthernet0/1  [up/up]
    FE80::A8BB:CCFF:FE00:2001
    FD00:10:11:20::1
```

### 875.6 ตารางสรุป Stateful DHCPv6 vs DHCPv6-PD

| คุณสมบัติ | Stateful DHCPv6 (875.2) | DHCPv6-PD (875.4) |
|---|---|---|
| แจกอะไร | Address เดียว (`/128` Effectively) | **Prefix ทั้งก้อน** (เช่น `/60`) |
| ให้ใครใช้ | Host/Client ปลายทาง (End Device) | **Router** ที่จะเอา Prefix ไปแบ่งต่อ |
| ใช้บ่อยที่สุดที่ | LAN Segment ปลาย (VLAN ผู้ใช้) | ISP↔CPE, HQ↔Branch, Enterprise↔Downstream Router |
| Flag ที่ต้องตั้งบน RA | M-flag = 1 | ไม่เกี่ยวกับ RA Flag (Client เรียก PD ผ่าน Solicit/Request แยก ไม่ผ่าน SLAAC) |

---

## Step 876 — IPv6 ใน BGP: `address-family ipv6` และ Dual-Stack Peering

### 876.1 ทวนโครงสร้าง BGP ของ Lab จาก Part 29/30/74

[Part 74](part-074-advanced-bgp-wan-edge.md) สร้าง **iBGP AS 65001** เต็มรูปแบบผ่าน **Route
Reflector** (CORE-SW1/CORE-SW2 เป็น RR, DIST-SW1-4 เป็น RR-Client ผ่าน Peer-Group
`RR-CLIENT-DIST`) และ **eBGP ไป ISP-RTR (AS 65000)** ที่ WAN-EDGE-1/2 — ทุก Session Peer ผ่าน
**Loopback0 (IPv4)** เป็นหลัก การเพิ่ม IPv6 เข้าไปมี 2 แนวทางที่ CCIE ต้องเลือกให้ถูกกับสถานการณ์

### 876.2 แนวทางที่ 1 — Session เดียว Carry ทั้งสอง AF (เหมาะกับ iBGP/RR)

MP-BGP (RFC 4760) อนุญาตให้ **1 TCP Session ที่สร้างผ่าน Address ตระกูลหนึ่ง (เช่น IPv4)
Carry NLRI ของอีกตระกูลได้ (IPv6)** ผ่านการ `activate` Neighbor ตัวเดิมใน `address-family
ipv6 unicast` เพิ่ม — เหมาะกับ **iBGP ผ่าน Route Reflector** เพราะ `next-hop-self` ทำให้ RR
เขียน Next-Hop เป็น Address ของตัวเองใน AF นั้นๆ เสมอ (ไม่ต้องพึ่ง Next-Hop ของ Neighbor เดิม)

```
! ===== CORE-SW1 — RR ขยาย Peer-Group RR-CLIENT-DIST ให้ Carry ทั้ง IPv4 และ IPv6 =====
CORE-SW1(config)# interface Loopback0
CORE-SW1(config-if)# ipv6 address fd00:1:1:1::1/128
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# router bgp 65001
CORE-SW1(config-router)# neighbor RR-CLIENT-DIST peer-group
CORE-SW1(config-router)# neighbor RR-CLIENT-DIST remote-as 65001
CORE-SW1(config-router)# neighbor RR-CLIENT-DIST update-source Loopback0
CORE-SW1(config-router)# neighbor 1.1.1.11 peer-group RR-CLIENT-DIST
CORE-SW1(config-router)# neighbor 1.1.1.12 peer-group RR-CLIENT-DIST
CORE-SW1(config-router)# neighbor 1.1.1.13 peer-group RR-CLIENT-DIST
CORE-SW1(config-router)# neighbor 1.1.1.14 peer-group RR-CLIENT-DIST
!
CORE-SW1(config-router)# address-family ipv4 unicast
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST activate
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST next-hop-self
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST route-reflector-client
CORE-SW1(config-router-af)# exit-address-family
!
CORE-SW1(config-router)# address-family ipv6 unicast
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST activate
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST next-hop-self
CORE-SW1(config-router-af)# neighbor RR-CLIENT-DIST route-reflector-client
CORE-SW1(config-router-af)# network fd00:10:10:10::/64
CORE-SW1(config-router-af)# exit-address-family
```

> **จุดสำคัญ**: Session ยังเป็น **TCP เดียวไปที่ IPv4 Address ของ Loopback0** ทุกประการ (ไม่มี
> Session ที่ 2) — สิ่งที่เปลี่ยนคือมี**การ Activate NLRI เพิ่มอีก 1 AF** บน Session เดิม ทำให้
> Update Message ที่ส่งไปมามี AFI/SAFI 2 ชุดปนกันได้ (IPv4 Unicast + IPv6 Unicast) — วิธีนี้
> **ประหยัด Session/Overhead ที่สุด** เหมาะกับ iBGP ภายในองค์กรที่ Router ทุกตัว Reachability
> กันได้แน่นอนผ่าน IGP อยู่แล้ว (OSPFv2 สำหรับ IPv4-Next-Hop, OSPFv3 จาก [Part 27](part-027-ospfv3-ipv6.md)
> สำหรับ IPv6-Next-Hop)

**DIST-SW1 (RR-Client) ฝั่งตรงข้าม**:

```
DIST-SW1(config)# interface Loopback0
DIST-SW1(config-if)# ipv6 address fd00:1:1:11::11/128
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# router bgp 65001
DIST-SW1(config-router)# neighbor 1.1.1.1 remote-as 65001
DIST-SW1(config-router)# neighbor 1.1.1.1 update-source Loopback0
DIST-SW1(config-router)# neighbor 1.1.1.2 remote-as 65001
DIST-SW1(config-router)# neighbor 1.1.1.2 update-source Loopback0
!
DIST-SW1(config-router)# address-family ipv4 unicast
DIST-SW1(config-router-af)# neighbor 1.1.1.1 activate
DIST-SW1(config-router-af)# neighbor 1.1.1.2 activate
DIST-SW1(config-router-af)# network 10.10.10.0 mask 255.255.255.0
DIST-SW1(config-router-af)# exit-address-family
!
DIST-SW1(config-router)# address-family ipv6 unicast
DIST-SW1(config-router-af)# neighbor 1.1.1.1 activate
DIST-SW1(config-router-af)# neighbor 1.1.1.2 activate
DIST-SW1(config-router-af)# network fd00:10:10:10::/64
DIST-SW1(config-router-af)# exit-address-family
```

สังเกตว่า `neighbor 1.1.1.1 activate` (IPv4 Address) ถูกใช้ **ซ้ำ** ทั้งใน `address-family
ipv4` และ `address-family ipv6` — DIST-SW1 ไม่จำเป็นต้องรู้จัก IPv6 Address ของ CORE-SW1 เลย
ก็ Peer แลก IPv6 Route กันได้ เพราะ Session สร้างผ่าน IPv4 Loopback ตัวเดียวตั้งแต่แรก

### 876.3 แนวทางที่ 2 — แยก Session ตาม AF (เหมาะกับ eBGP ไปยัง ISP)

สำหรับ eBGP ที่ WAN-EDGE-1 ↔ ISP-RTR ([Part 29](part-029-bgp-fundamentals.md)) แนวทางที่นิยม
ใช้จริงคือ**แยก Session ตาม Address Family อย่างสมบูรณ์** แม้จะวิ่งบน Physical Link เดียวกัน —
เหตุผลคือ **ควบคุม/Troubleshoot ง่ายกว่า แยก Fault Domain ได้** (Session IPv6 ล้มไม่กระทบ
Session IPv4 และในทางกลับกัน) และ**ไม่ต้องพึ่ง Next-Hop Rewrite ข้าม AF** ที่อาจสร้างความ
สับสนตอน Debug:

```
! ===== WAN-EDGE-1 — eBGP แยก Session ตาม AF ไปยัง ISP-RTR =====
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# neighbor 203.0.113.1 remote-as 65000
WAN-EDGE-1(config-router)# neighbor 2001:DB8:203:113::1 remote-as 65000
!
WAN-EDGE-1(config-router)# address-family ipv4 unicast
WAN-EDGE-1(config-router-af)# neighbor 203.0.113.1 activate
WAN-EDGE-1(config-router-af)# exit-address-family
!
WAN-EDGE-1(config-router)# address-family ipv6 unicast
WAN-EDGE-1(config-router-af)# neighbor 2001:DB8:203:113::1 activate
WAN-EDGE-1(config-router-af)# network 2001:DB8:30:30::/64
WAN-EDGE-1(config-router-af)# exit-address-family
```

**ISP-RTR ฝั่งตรงข้าม**:

```
ISP-RTR(config)# router bgp 65000
ISP-RTR(config-router)# neighbor 203.0.113.2 remote-as 65001
ISP-RTR(config-router)# neighbor 2001:DB8:203:113::2 remote-as 65001
!
ISP-RTR(config-router)# address-family ipv4 unicast
ISP-RTR(config-router-af)# neighbor 203.0.113.2 activate
ISP-RTR(config-router-af)# exit-address-family
!
ISP-RTR(config-router)# address-family ipv6 unicast
ISP-RTR(config-router-af)# neighbor 2001:DB8:203:113::2 activate
```

`neighbor 203.0.113.1` และ `neighbor 2001:DB8:203:113::1` เป็น **2 Neighbor Statement แยกกัน
ของ Peer เดียวกัน (ISP-RTR)** — เกิด **2 TCP Session** แม้จะเป็น Physical Link เดียวกันทั้งคู่

### 876.4 ตารางตัดสินใจ: เลือกแนวทางไหน

| ปัจจัย | Session เดียว Carry 2 AF (876.2) | แยก Session ตาม AF (876.3) |
|---|---|---|
| เหมาะกับ | iBGP ภายในองค์กร (RR/RR-Client) | eBGP ข้ามองค์กร/ISP |
| จำนวน TCP Session | น้อยกว่า (Overhead ต่ำ) | มากกว่า (Overhead สูงขึ้นเป็นเงาตามตัว) |
| Fault Isolation | AF หนึ่งมีปัญหาอาจกระทบ Session รวม | แยก Fault Domain ชัดเจนต่อ AF |
| ความซับซ้อนเรื่อง Next-Hop | ต้องพึ่ง `next-hop-self`/IGP ให้ Resolve Next-Hop ข้าม AF ได้ | ไม่มีปัญหา Next-Hop ข้าม AF เลย (แต่ละ Session ใช้ Next-Hop ของ AF ตัวเองอยู่แล้ว) |
| ตัวอย่างในหลักสูตรนี้ | CORE-SW1 (RR) ↔ DIST-SW1-4 | WAN-EDGE-1 ↔ ISP-RTR |

### 876.5 Verification

```
CORE-SW1# show bgp ipv6 unicast summary
BGP router identifier 1.1.1.1, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.11        4 65001      42      40        8    0    0 00:31:05        1
1.1.1.12        4 65001      41      39        8    0    0 00:29:52        1

CORE-SW1# show bgp ipv6 unicast
BGP table version is 8, local router ID is 1.1.1.1
   Network              Next Hop            Metric LocPrf Weight Path
*> FD00:10:10:10::/64   FD00:1:1:1::1             0    100      0 i
```

`Next Hop` เป็น `FD00:1:1:1::1` (Loopback0 IPv6 ของ CORE-SW1 เอง) ยืนยันว่า `next-hop-self`
บน RR ทำงานถูกต้อง — DIST-SW อื่นไม่จำเป็นต้องรู้จัก Next-Hop เดิม (Loopback0 IPv6 ของ DIST-SW1)
เลย เหมือนกับพฤติกรรม IPv4 RR ที่ [Part 74](part-074-advanced-bgp-wan-edge.md) สอนไว้ทุกประการ

---

## Step 877 — IPv6 และ NAT: NAT66/NPTv6

### 877.1 ทำไม IPv6 (ในทางทฤษฎี) ไม่ต้องมี NAT

IPv6 ออกแบบมาให้มี Address เหลือเฟือ ([Part 8 Step 71](part-008-ipv6-fundamentals.md#step-71--ทำไมต้อง-ipv6-และรูปแบบ-address))
จึง**ไม่มีเหตุผลด้าน "ประหยัด Address" ที่ต้องทำ NAT แบบ PAT ของ IPv4** ([Part 15](part-015-nat-pat.md))
— ปรัชญาดั้งเดิมของ IPv6 คือ End-to-End Reachability เต็มรูปแบบ ไม่มี NAT มาบัง Address ระหว่าง
ทาง แต่ในทางปฏิบัติยังมีเหตุผลอื่นที่ Enterprise ใช้ Prefix Translation อยู่บ้าง

### 877.2 NPTv6 (Network Prefix Translation, RFC 6296) — ทางเลือกที่ IETF แนะนำ

NPTv6 เป็นการแปลง **แค่ Prefix (Network Part) แบบ 1:1 เชิงพีชคณิต ไม่แตะ Interface ID
(Host Part) เลย** และ **ไม่ทำ Port Translation แบบ PAT** — ต่างจาก NAT44 ที่ Map หลาย Internal
Address เข้า 1 Public Address (Many-to-One ผ่าน Port) โดยพื้นฐาน:

```
Internal (ULA):        fd00:10:10:10::100/64
                              │
                    NPTv6 แปลงแค่ 64 bit แรก (Prefix)
                    Interface ID (::100) คงเดิมทุก bit
                              ▼
External (GUA):        2001:db8:203:113:0:10:10:100/64   (ตัวอย่างแนวคิด)
```

**ใช้เมื่อไหร่**: สถานการณ์ที่ต้องการ **Multihoming แบบไม่ Renumber** — องค์กรมี ULA ภายในคงที่
แล้ว Map ออกเป็น GUA ของ ISP ที่ Active อยู่ตอนนั้น ผ่าน NPTv6 ที่ Edge Router สองจุด (จุดละ
1 ISP) — ถ้า ISP หลักล้ม สลับไปใช้ GUA ของ ISP รองได้โดย **ไม่ต้อง Renumber Host ภายในเลย**
(Host ยังใช้ ULA เดิม, NPTv6 Rule แค่เปลี่ยน Mapping เป็น GUA ก้อนใหม่)

```
! แนวคิด Syntax (Platform-Dependent — IOS-XE บางรุ่น/ASA รองรับ ipv6 nat prefix)
WAN-EDGE-1(config)# ipv6 nat prefix fd00:10:10::/48 2001:db8:203:113::/48
```

เพราะเป็นการแปลงแค่ Prefix แบบ Checksum-Neutral (RFC 6296 ออกแบบ Algorithm ให้ Checksum ของ
Transport Layer ไม่เปลี่ยนแม้ Prefix เปลี่ยน) จึง **ไม่ต้องคำนวณ Checksum ใหม่เหมือน NAT44**
และไม่ทำลาย End-to-End Transparency ของ Port Number เหมือนที่ PAT ทำ

### 877.3 NAT66 (Traditional, Stateful) — ทำไมไม่แนะนำ

**NAT66** หมายถึง NAT แบบดั้งเดิม (Stateful, ซ่อน Address จริงเหมือน NAT44/PAT) ที่ Apply กับ
IPv6 ตรงๆ — แม้บาง Firewall (ASA/FTD) จะรองรับในบางรูปแบบ แต่ **IETF ไม่แนะนำ** เพราะ:

| ปัญหา | รายละเอียด |
|---|---|
| ทำลาย End-to-End Principle | ขัดกับเจตนาการออกแบบ IPv6 ที่ต้องการ Global Reachability โดยตรง |
| ซับซ้อนกับ IPsec | Encrypted Payload ทำให้ Stateful NAT ไม่สามารถแก้ Address/Port ข้างในได้ (ปัญหาเดียวกับ NAT44+IPsec แต่รุนแรงกว่าเพราะ IPv6 คาดหวัง IPsec เป็นมาตรฐาน) |
| ไม่มีประโยชน์ด้าน Address Conservation | ต่างจาก NAT44/PAT ที่เกิดจากความจำเป็นจริง (IPv4 Exhaustion) — IPv6 ไม่มีปัญหานี้ตั้งแต่ต้น |

> **สรุปสำหรับข้อสอบ CCIE**: ถ้าเจอโจทย์ "Multihoming โดยไม่ Renumber" หรือ "แปลง Prefix แบบ
> ไม่กระทบ Port/Checksum" → คำตอบคือ **NPTv6** ถ้าเจอคำว่า "ซ่อน Internal Address แบบ Stateful
> เหมือน PAT" (ซึ่งไม่ใช่ Best Practice) → นั่นคือ **NAT66** ที่ควรหลีกเลี่ยงในการออกแบบใหม่

---

## Step 878 — สรุป IPv6 Transition Mechanism ทั้งหมด: ตารางตัดสินใจ

### 878.1 ทวนแต่ละเทคนิคที่หลักสูตรนี้แตะมาแล้ว

| เทคนิค | เรียนที่ | สถานะปัจจุบัน (2026) |
|---|---|---|
| **Dual-Stack** | [Part 8](part-008-ipv6-fundamentals.md) และตลอดหลักสูตรนี้ | **มาตรฐานหลักที่ Enterprise ใช้จริง** — แนวทางที่แนะนำเป็นอันดับแรกเสมอ |
| **6to4 / ISATAP (Automatic Tunneling)** | ใหม่ในส่วนนี้ (878.2) | **Deprecated/Legacy** — เกือบไม่มีใครใช้แล้วในปี 2026 |
| **Manual/GRE Tunnel (รวม 6PE/6VPE บน MPLS)** | เทียบกับ [Part 86/87](part-086-mpls-l3vpn-fundamentals.md) | ยังมีใช้ในบาง Core ที่ P Router ไม่รองรับ IPv6 Native |
| **NAT64/DNS64** | [Part 63 Step 626](part-063-advanced-nat-scenarios.md#step-626--nat64--nat-pt-แนวคิดสำหรับ-ipv4-to-ipv6-transition) | ใช้จริงในเครือข่าย Mobile/Carrier ที่เป็น **IPv6-Only** |
| **NAT-PT** | [Part 63 Step 626.3](part-063-advanced-nat-scenarios.md) | **Deprecated อย่างเป็นทางการ (RFC 4966)** — รู้จักชื่อไว้สอบเท่านั้น |

### 878.2 6to4 และ ISATAP — ทำไมกลายเป็น Legacy

| เทคนิค | หลักการ | ทำไมเลิกใช้ |
|---|---|---|
| **6to4** | Encapsulate IPv6 ใน IPv4 Header อัตโนมัติผ่าน Prefix พิเศษ `2002::/16` (ฝัง IPv4 Address ไว้ใน Prefix) ส่งผ่าน **6to4 Relay** สาธารณะ | พึ่งพา Relay ของคนอื่นที่ควบคุมคุณภาพไม่ได้, มีปัญหา Asymmetric Routing บ่อย, ชนกับ NAT44 ที่ปลายทาง (6to4 ออกแบบมาก่อนที่ NAT บ้าน/องค์กรจะแพร่หลาย) |
| **ISATAP** | Tunnel IPv6 ผ่าน IPv4-only Intranet Backbone โดยสร้าง Interface ID จาก IPv4 Address (`::5EFE:w.x.y.z`) | ออกแบบมาเป็น "สะพานชั่วคราว" ระหว่างที่ Backbone ยังไม่รองรับ Dual-Stack เต็มที่ — ปัจจุบัน Backbone ส่วนใหญ่ Native Dual-Stack แล้ว (เหมือนที่ Lab นี้ทำมาตั้งแต่ Part 8) จึงไม่มีความจำเป็นต้อง Tunnel ผ่าน IPv4 อีก |

### 878.3 ตารางตัดสินใจฉบับสมบูรณ์: เลือกเทคนิคไหนเมื่อไหร่

| สถานการณ์ | เทคนิคที่เหมาะสม | หมายเหตุ |
|---|---|---|
| Enterprise LAN ทั่วไปที่ยังมี Legacy App/Device เยอะ (สถานการณ์ของ Lab นี้) | **Dual-Stack** | ปลอดภัยที่สุด ไม่ต้องพึ่ง Translation ใดๆ — ตรงกับที่ Lab นี้ใช้ตลอด 88 Part |
| Backbone ภายในยังไม่รองรับ IPv6 Native แต่ Site ปลายทั้งสองฝั่งต้องคุยกันด้วย IPv6 | Manual GRE Tunnel หรือ 6PE/6VPE (ถ้าเป็น MPLS Core, [Part 86/87](part-086-mpls-l3vpn-fundamentals.md)) | เป็นทางออกชั่วคราวจนกว่า Backbone จะ Upgrade เป็น Native Dual-Stack |
| Mobile/Carrier Network ที่เป็น **IPv6-Only ล้วน** แต่ยังต้องเข้าเว็บ IPv4-Only บาง Site | **NAT64 + DNS64** | ต้องมี DNS64 Resolver คู่กันเสมอ — ไม่ทำงานถ้า Client ใช้ DNSSEC เข้มงวดหรือ Hardcode IP Literal |
| Multihomed ต้องการสลับ ISP โดยไม่ Renumber Host ภายใน | **NPTv6** | ดู [Step 877.2](#step-877--ipv6-และ-nat-nat66nptv6) |
| Network ใหม่ทั้งหมด (Greenfield) ไม่มี Legacy Baggage เลย | พิจารณา **IPv6-Only ตรง** (ไม่ต้อง Transition เลย) | ดู [Step 879](#step-879--ipv6-only-network--อะไรพังถ้าถอด-ipv4-ออกจริง) |

---

## Step 879 — IPv6-Only Network: อะไรพังถ้าถอด IPv4 ออกจริง

### 879.1 แรงผลักดันสู่ IPv6-Only

แรงกดดันหลักคือ **IPv4 Address Exhaustion** ที่ [Part 8 Step 71](part-008-ipv6-fundamentals.md#step-71--ทำไมต้อง-ipv6-และรูปแบบ-address)
พูดถึงตั้งแต่ต้น — RIR ทุกแห่งหมด Free Pool แล้ว การได้ IPv4 Address ใหม่ (แม้แค่ Private/CGNAT
Layer เพิ่ม) มีต้นทุนสูงขึ้นทุกปี ในขณะที่ IPv6-Only ตัด **ต้นทุน Dual-Stack ทั้งหมด**:
ไม่ต้องดูแล ACL/Routing Table/DHCP Server สองชุด, ไม่ต้อง Troubleshoot ปัญหาที่เกิดจาก IPv4
กับ IPv6 ไม่ตรงกัน (เช่น Application Prefer IPv6 แต่ Firewall Rule ตกหล่นแค่ฝั่ง IPv6)

### 879.2 อะไรพังจริงถ้าถอด IPv4 ออกทั้งหมด

| สิ่งที่พัง | รายละเอียด |
|---|---|
| **Legacy IoT/OT Device** | เครื่อง Sensor/PLC/Printer รุ่นเก่าจำนวนมากมี Firmware ที่**ไม่รองรับ IPv6 เลย** และ Vendor เลิกอัพเดต Firmware ไปแล้ว — ต้องเปลี่ยนอุปกรณ์ทั้งหมดซึ่งมีต้นทุนสูงมากในโรงงาน/ICS |
| **Application ที่ Hardcode IPv4 Literal** | Software ภายในบางตัว (มักเป็น In-House App เก่า) เขียน Logic ผูกกับ Format `a.b.c.d` ตรงๆ (เช่น Regex Validate IP, Firewall Rule ฝัง IP ในโค้ด) — ต้องไล่แก้ Code ทุกจุดก่อน Migrate ได้ |
| **VPN Client/Remote Access รุ่นเก่า** | Client VPN บางตัว (โดยเฉพาะระดับ Consumer/SME) ยังไม่รองรับ IPv6 Transport เต็มรูปแบบ |
| **Partner/Extranet ที่ยังเป็น IPv4-Only** | องค์กรควบคุม Network ตัวเองได้ แต่ควบคุม Partner ไม่ได้ — ถ้า Partner ยังไม่รองรับ IPv6 ต้องมี NAT64 หรือ Dual-Stack ที่จุดเชื่อมต่อนั้นอยู่ดี |
| **Monitoring/Management Tool เก่า** | NMS/Syslog/SNMP Trap Receiver บางระบบ Legacy ที่ผูก Address Family ไว้กับ IPv4 ตั้งแต่ต้น |
| **License Server แบบผูก MAC/IP** | ระบบ License บางตัวยังตรวจสอบผ่าน IPv4 Address ที่ Register ไว้ |

### 879.3 การประเมินตามจริง: ควร IPv6-Only เมื่อไหร่ vs Dual-Stack ต่อไป

| ระดับ Network | ความเป็นไปได้ของ IPv6-Only |
|---|---|
| **Service Provider Access Network (Mobile/Cellular)** | **สูงมาก** — Carrier ใหญ่หลายรายทำ IPv6-Only + NAT64 มานานแล้วในโครง Core เพราะ Client เป็น Sim Card ใหม่ควบคุมได้เต็มที่ |
| **Cloud-Native/Container Workload ภายใน (Internal-Only Segment เช่น Kubernetes Pod Network)** | **สูง** — Workload สร้างขึ้นใหม่ทั้งหมด ไม่มี Legacy Baggage เลย |
| **Data Center/Server Farm ที่ควบคุม Software Stack เองทั้งหมด** | ปานกลางถึงสูง — ถ้า Audit แล้วไม่มี Legacy App เหลือจริง |
| **Enterprise Campus LAN ทั่วไป (สถานการณ์ของ Lab นี้)** | **ต่ำในระยะสั้น-กลาง** — มี Printer/Badge Reader/Legacy Peripheral ปนอยู่เสมอ Audit เต็มรูปแบบมีต้นทุนสูง |
| **OT/ICS Network** | **ต่ำมาก** — Lifecycle ของอุปกรณ์อุตสาหกรรมยาว 15-20 ปี เปลี่ยนยากที่สุด |

> **บทสรุปสำหรับ CCIE**: IPv6-Only ไม่ใช่ "อนาคตที่มาแทน Dual-Stack ทุกที่" แต่เป็น **เครื่องมือ
> ที่เหมาะกับบาง Segment เท่านั้น** — Enterprise ส่วนใหญ่ (รวม Lab ของหลักสูตรนี้) จะยังคง
> **Dual-Stack ไปอีกนาน** โดยค่อยๆ ผลัก Segment ที่ Audit แล้วสะอาดจริง (เช่น Internal Cloud
> Workload ใหม่) ไปเป็น IPv6-Only แบบ Incremental ไม่ใช่ Big-Bang ทั้งองค์กรพร้อมกัน

---

## Step 880 — Lab เต็มรูปแบบ: IPv6 Security Hardening + MLD Snooping + DHCPv6-PD + Dual-Stack BGP

### 880.1 Scope ของ Lab นี้

รวม 4 องค์ประกอบจาก Step 873-876 เข้าเป็น Lab เดียวบน Topology หลักของหลักสูตร:

1. **IPv6 First-Hop Security เต็มรูปแบบ** (RA Guard + DHCPv6 Guard + IPv6 Snooping/ND
   Inspection) บน **ACCESS-SW1-4** ทั้งหมด
2. **MLD Snooping ทั่วทั้งองค์กร** บน ACCESS-SW1-4 (ทุก VLAN Data)
3. **DHCPv6-PD จาก DIST Switch** — ขยายจาก Step 875.4 (WAN-EDGE) มาเป็น **DIST-SW1/DIST-SW3**
   แจก Prefix ให้ Downstream Router จำลอง (IOT-GW-1/IOT-GW-3)
4. **Dual-Stack BGP เต็มรูปแบบ** ต่อจาก Step 876 บน CORE-SW1/CORE-SW2 (RR) และ DIST-SW1-4
   (RR-Client) พร้อม eBGP ที่ WAN-EDGE-1/2 ไป ISP-RTR

```
                              ┌───────────────┐
                              │   ISP-RTR      │  AS 65000
                              │ (v4+v6 eBGP,   │
                              │  2 Session)    │
                              └───┬───────┬───┘
                    203.0.113.0/30│       │203.0.113.4/30
                   2001:db8:203:113::/64  │
                  ┌────────┴──┐       ┌──┴────────┐
                  │WAN-EDGE-1 │       │WAN-EDGE-2 │  AS 65001 (eBGP: Session แยกตาม AF)
                  │(PD Server │       └─────┬─────┘
                  │ ไป IOT-GW)│             │
                  └─────┬─────┘             │
              ┌──────────┴──────┐   ┌────────┴─────────┐
              │   CORE-SW1 (RR) │===│   CORE-SW2 (RR)   │  iBGP: Session เดียว Carry v4+v6
              └───┬─────────┬───┘   └────┬─────────┬────┘
          ┌────────┴──┐ ┌────┴───┐ ┌───────┴──┐ ┌────┴───────┐
          │ DIST-SW1   │ │DIST-SW2│ │ DIST-SW3 │ │ DIST-SW4   │  RR-Client (iBGP v4+v6)
          │ (PD Server │ │        │ │(PD Server│ │            │
          │ →IOT-GW-1) │ │        │ │→IOT-GW-3)│ │            │
          └─────┬──────┘ └───┬────┘ └────┬─────┘ └─────┬──────┘
          ┌──────┴─────┐┌────┴───┐  ┌─────┴────┐  ┌──────┴─────┐
          │ ACCESS-SW1 ││ACCESS-SW2│ │ACCESS-SW3│  │ACCESS-SW4  │  RA Guard+DHCPv6 Guard+
          │ FHS+MLD    ││FHS+MLD   │ │FHS+MLD   │  │FHS+MLD     │  ND Inspection+MLD Snoop
          │ Snooping   ││Snooping  │ │Snooping  │  │Snooping    │  ทุกตัว
          └────────────┘└──────────┘ └──────────┘  └────────────┘
```

### 880.2 ส่วนที่ 1 — IPv6 First-Hop Security เต็มรูปแบบบน ACCESS-SW1-4

**ACCESS-SW1** (ตัวแทนแนวคิดที่ใช้ซ้ำกับ ACCESS-SW2-4 ทุกประการ เปลี่ยนแค่เลข VLAN ตาม
[00-ip-address-plan.md](00-ip-address-plan.md) — ACCESS-SW1/2 คุม VLAN 10,20,99, ACCESS-SW3/4
คุม VLAN 30,40,99):

```
! ===== ACCESS-SW1 — RA Guard =====
ACCESS-SW1(config)# ipv6 nd raguard policy HOST-PORT-POLICY
ACCESS-SW1(config-nd-raguard)# device-role host
ACCESS-SW1(config-nd-raguard)# exit
ACCESS-SW1(config)# ipv6 nd raguard policy UPLINK-POLICY
ACCESS-SW1(config-nd-raguard)# device-role router
ACCESS-SW1(config-nd-raguard)# trusted-port
ACCESS-SW1(config-nd-raguard)# exit
!
! ===== ACCESS-SW1 — DHCPv6 Guard =====
ACCESS-SW1(config)# ipv6 dhcp guard policy CLIENT-PORT-POLICY
ACCESS-SW1(config-dhcp-guard)# device-role client
ACCESS-SW1(config-dhcp-guard)# exit
ACCESS-SW1(config)# ipv6 dhcp guard policy SERVER-UPLINK-POLICY
ACCESS-SW1(config-dhcp-guard)# device-role server
ACCESS-SW1(config-dhcp-guard)# trusted-port
ACCESS-SW1(config-dhcp-guard)# exit
!
! ===== ACCESS-SW1 — IPv6 Snooping (ND Inspection + Binding Table) =====
ACCESS-SW1(config)# ipv6 snooping policy ACCESS-PORT-POLICY
ACCESS-SW1(config-ipv6-snooping)# protocol dhcp
ACCESS-SW1(config-ipv6-snooping)# protocol ndp
ACCESS-SW1(config-ipv6-snooping)# security-level guard
ACCESS-SW1(config-ipv6-snooping)# tracking enable reachable-lifetime 30
ACCESS-SW1(config-ipv6-snooping)# exit
!
! ===== ACCESS-SW1 — MLD Snooping =====
ACCESS-SW1(config)# ipv6 mld snooping
ACCESS-SW1(config)# ipv6 mld snooping vlan 10
ACCESS-SW1(config)# ipv6 mld snooping vlan 20
!
! ----- Apply ทั้งหมดบน Access Port (Host/Client Role) -----
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 20
ACCESS-SW1(config-if-range)# ipv6 nd raguard attach-policy HOST-PORT-POLICY
ACCESS-SW1(config-if-range)# ipv6 dhcp guard attach-policy CLIENT-PORT-POLICY
ACCESS-SW1(config-if-range)# ipv6 snooping attach-policy ACCESS-PORT-POLICY
ACCESS-SW1(config-if-range)# exit
!
! ----- Apply ทั้งหมดบน Uplink ไป DIST-SW1 (Router/Server/Trusted Role) -----
ACCESS-SW1(config)# interface range GigabitEthernet0/1 - 2
ACCESS-SW1(config-if-range)# ipv6 nd raguard attach-policy UPLINK-POLICY
ACCESS-SW1(config-if-range)# ipv6 dhcp guard attach-policy SERVER-UPLINK-POLICY
ACCESS-SW1(config-if-range)# ipv6 mld snooping vlan 10 mrouter interface GigabitEthernet0/1
```

**ACCESS-SW2, ACCESS-SW3, ACCESS-SW4** ใช้ Policy Name เดียวกันทุกตัว (Reusable Policy ตาม
แนวทาง Peer-Group/Object-Group ที่ [Part 45](part-045-advanced-acl-zbfw.md) สอนไว้) และ Apply
คำสั่งเดียวกันทั้งชุด (Policy Attach + MLD Snooping) เปลี่ยนแค่เลข VLAN ให้ตรงกับ Mapping ของ
[00-ip-address-plan.md](00-ip-address-plan.md) — ACCESS-SW2 ใช้ VLAN 10,20 เหมือน ACCESS-SW1,
ACCESS-SW3/ACCESS-SW4 ใช้ VLAN 30,40 แทน (`ipv6 mld snooping vlan 30`, `vlan 40`)

### 880.3 ส่วนที่ 2 — DHCPv6-PD จาก DIST-SW1 และ DIST-SW3

```
! ===== DIST-SW1 — PD Server ให้ IOT-GW-1 (Downstream Gateway จำลอง) =====
DIST-SW1(config)# ipv6 local pool IOTGW1-PD-POOL fd00:10:11:10::/60 64
DIST-SW1(config)# ipv6 dhcp pool IOTGW1-PD-POOL-CFG
DIST-SW1(config-dhcpv6)# prefix-delegation pool IOTGW1-PD-POOL lifetime 604800 302400
DIST-SW1(config-dhcpv6)# exit
DIST-SW1(config)# interface GigabitEthernet1/0/24
DIST-SW1(config-if)# description ** Link to IOT-GW-1 (PD Client) **
DIST-SW1(config-if)# ipv6 address fd00:10:11:1::1/64
DIST-SW1(config-if)# ipv6 dhcp server IOTGW1-PD-POOL-CFG
DIST-SW1(config-if)# no shutdown
```

**DIST-SW3** ใช้โครงสร้างเดียวกันทุกประการ เปลี่ยนแค่ชื่อ Pool/Prefix ให้ไม่ชนกับ DIST-SW1:
`ipv6 local pool IOTGW3-PD-POOL fd00:10:11:30::/60 64` และ Interface `fd00:10:11:3::1/64`
ไป `IOT-GW-3`

> สังเกตว่า Pool `fd00:10:11:10::/60` และ `fd00:10:11:30::/60` แบ่งจาก Block กลาง
> `fd00:10:11::/48` ที่กันไว้ตั้งแต่ [Step 875.4](#875-4-dhcpv6-pd-prefix-delegation--แจก-prefix-ทั้งก้อน-ไม่ใช่แค่-address-เดียว)
> — แยก Sub-Block ตาม DIST-SW เพื่อไม่ให้ Pool ของ DIST-SW1 และ DIST-SW3 ชนกัน (เหมือนหลักการ
> แบ่ง VLAN Subnet ตาม [00-ip-address-plan.md](00-ip-address-plan.md))

**IOT-GW-1 (PD Client)**:

```
IOT-GW-1(config)# interface GigabitEthernet0/0
IOT-GW-1(config-if)# ipv6 address autoconfig
IOT-GW-1(config-if)# ipv6 dhcp client pd IOTGW1-PREFIX
IOT-GW-1(config-if)# exit
IOT-GW-1(config)# interface GigabitEthernet0/1
IOT-GW-1(config-if)# ipv6 address IOTGW1-PREFIX ::1/64
IOT-GW-1(config-if)# no shutdown
```

### 880.4 ส่วนที่ 3 — Dual-Stack BGP เต็มรูปแบบ (สรุปจาก Step 876 ใส่ครบทุกอุปกรณ์)

| อุปกรณ์ | Role | AF ที่เปิด | Session Model |
|---|---|---|---|
| CORE-SW1, CORE-SW2 | Route Reflector (iBGP AS 65001) | ipv4 unicast + ipv6 unicast | Session เดียวต่อ RR-Client Carry ทั้ง 2 AF |
| DIST-SW1-4 | RR-Client | ipv4 unicast + ipv6 unicast | Session เดียวไปยัง CORE-SW1/CORE-SW2 |
| WAN-EDGE-1, WAN-EDGE-2 | eBGP Edge (ไป ISP-RTR, AS 65000) | ipv4 unicast + ipv6 unicast | **แยก Session ตาม AF** (Neighbor คนละ Address) |
| ISP-RTR | eBGP Peer (AS 65000) | ipv4 unicast + ipv6 unicast | แยก Session ตาม AF เช่นกัน |

### 880.5 Verification รวมทั้ง Lab

```
! ----- ตรวจสอบ First-Hop Security -----
ACCESS-SW1# show ipv6 nd raguard policy UPLINK-POLICY
ACCESS-SW1# show ipv6 dhcp guard policy SERVER-UPLINK-POLICY
ACCESS-SW1# show ipv6 neighbors binding
Network Layer Address        Age      State  Interface  MAC Address     VLAN
FD00:10:10:10::101            00:02:14 REACH  Gi1/0/1    0050.7966.6801  10

! ----- ตรวจสอบ MLD Snooping -----
ACCESS-SW3# show ipv6 mld snooping groups
Vlan     Group                                    Type    Version  Port List
30       FF05::239:1:1                            D       v2       Gi1/0/1

! ----- ตรวจสอบ DHCPv6-PD -----
DIST-SW1# show ipv6 dhcp binding
Client: FE80::A8BB:CCFF:FE00:3001
  DUID: 00030001AABBCC003001
  IA PD: IA ID 0x0C270001, T1 302400, T2 483840
    Prefix: FD00:10:11:10::/60
      preferred lifetime 302400, valid lifetime 604800

IOT-GW-1# show ipv6 interface brief
GigabitEthernet0/1  [up/up]
    FE80::A8BB:CCFF:FE00:3101
    FD00:10:11:10::1

! ----- ตรวจสอบ Dual-Stack BGP -----
CORE-SW1# show bgp ipv6 unicast summary
BGP router identifier 1.1.1.1, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
1.1.1.11        4 65001      55      53        9    0    0 00:40:12        1
1.1.1.12        4 65001      54      52        9    0    0 00:39:58        1
1.1.1.13        4 65001      53      51        9    0    0 00:38:44        1
1.1.1.14        4 65001      52      50        9    0    0 00:38:20        1

WAN-EDGE-1# show bgp ipv6 unicast summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor              V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
2001:DB8:203:113::1   4 65000      30      28        4    0    0 00:20:05        1

WAN-EDGE-1# ping ipv6 fd00:10:10:10::10 source Loopback0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FD00:10:10:10::10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/4/6 ms
```

ผลลัพธ์ยืนยันครบทุกองค์ประกอบที่ Lab นี้ตั้งใจพิสูจน์: **FHS Binding Table** จับคู่ Address/MAC
ถูกต้องบน Access Port, **MLD Snooping Group Table** จำกัด Multicast ไปแค่ Port ที่มี Receiver,
**DHCPv6-PD Binding** แจก Prefix `/60` ให้ IOT-GW จริง, และ **Dual-Stack BGP** แลก Prefix IPv6
ผ่านทั้ง iBGP RR (Session เดียว Carry 2 AF) และ eBGP ไป ISP (แยก Session) ได้สำเร็จ — Ping
ข้าม WAN-EDGE-1 ไปถึง VLAN 10 ยืนยัน End-to-End Reachability ของ IPv6 ทั้ง Path

---

## แบบฝึกหัดทวนความเข้าใจ Part 88

1. ทำไม RFC 7381 จึงแนะนำให้ Enterprise ใช้ GUA ทั่วทั้งเครือข่ายเป็นค่าเริ่มต้น แทนที่จะใช้
   ULA-only แล้วพึ่ง NPTv6 ที่ Perimeter?
2. เพราะเหตุใดการเขียน `deny icmp any any` แบบเหมาแบบเดียวกับที่บางทีมทำกับ IPv4 จึงเป็นความ
   ผิดพลาดร้ายแรงกว่ามากถ้าทำกับ IPv6 ACL? ยกตัวอย่าง ICMPv6 Type ที่ต้องเปิดเสมอ
3. อธิบายความแตกต่างระหว่าง RA Guard, DHCPv6 Guard, และ IPv6 Snooping/ND Inspection — แต่ละตัว
   ป้องกัน Attack แบบไหน และเทียบเคียงได้กับฟีเจอร์ IPv4 ตัวใดใน Part 21?
4. Stateful DHCPv6 กับ DHCPv6-PD แจก "อะไร" ต่างกันอย่างไร และทำไม DHCPv6-PD จึงเหมาะกับ
   HQ→Branch มากกว่า Stateful DHCPv6 ปกติ?
5. เพราะเหตุใด Dual-Stack BGP ระหว่าง CORE-SW1 (RR) กับ DIST-SW1 จึงใช้ Session เดียว Carry
   ทั้ง IPv4/IPv6 ได้สบายๆ ในขณะที่ WAN-EDGE-1↔ISP-RTR เลือกแยก Session ตาม AF?

**เฉลย:**

1. เพราะ ULA-only บังคับให้ทุกจุดที่ต้องออก Internet ต้องมี NPTv6 คอย Map Prefix ซึ่งเพิ่ม
   State และจุด Failure โดยไม่ได้ประโยชน์ด้าน Security เพิ่มขึ้นจริง — Security ที่ต้องการ
   (ควบคุมว่าใครเข้าถึงอะไรได้) ทำได้ด้วย Firewall/ACL ที่ Perimeter อยู่แล้วโดยไม่ต้องพึ่ง NAT
   เป็นกลไก Security เหมือนที่คนเข้าใจผิดกับ NAT44 บางครั้ง — GUA ทั่วเครือข่ายจึงง่ายกว่า
   ในการ Operate และ Troubleshoot โดยไม่เสีย Security ไปเลย
2. เพราะ ICMPv6 ไม่ใช่แค่ Diagnostic Tool เหมือน ICMPv4 (Ping/Traceroute) แต่เป็น **Transport
   ของ NDP ทั้งหมด** (Neighbor Solicitation/Advertisement ที่แทนที่ ARP, Router Solicitation/
   Advertisement ที่ขับเคลื่อน SLAAC) — Block ICMPv6 ทั้งหมด = Address Resolution และ Default
   Gateway Discovery พังทันทีทั้ง Subnet ไม่ใช่แค่ Ping ใช้ไม่ได้เหมือน IPv4 Type ที่ต้องเปิด
   เสมอคือ 133 (RS), 134 (RA), 135 (NS), 136 (NA), และ 2 (Packet Too Big ที่ขับเคลื่อน PMTUD)
3. RA Guard ป้องกัน **Rogue Router Advertisement** (เทียบเคียง DHCP Snooping ที่ป้องกัน Rogue
   DHCP Server) โดย Drop RA ที่มาจาก Port ที่ตั้ง Device-Role เป็น `host` — DHCPv6 Guard ป้องกัน
   **Rogue DHCPv6 Server** (ไม่มีเทียบเคียงตรงใน Part 21 เพราะ IPv4 DHCP Snooping ครอบคลุมทั้ง
   Server ในฟีเจอร์เดียว) โดย Drop DHCPv6 Reply ที่มาจาก Port ที่ตั้งเป็น `client` — IPv6
   Snooping/ND Inspection สร้าง Binding Table จาก DHCPv6/NDP แล้วตรวจสอบ NS/NA เทียบกับ Binding
   Table นั้น เทียบเคียงได้กับ **Dynamic ARP Inspection (DAI)** ของ Part 21 ที่พึ่ง DHCP
   Snooping Binding Table ตรวจสอบ ARP
4. Stateful DHCPv6 แจก **Address เดียว** (คล้าย `/128`) ให้ **Host/Client ปลายทาง** ใช้งานตรง
   ส่วน DHCPv6-PD แจก **Prefix ทั้งก้อน** (เช่น `/60`) ให้ **Router** เอาไปแบ่ง Subnet ต่อเอง —
   HQ→Branch เหมาะกับ PD มากกว่าเพราะ Branch Router ต้องมี Subnet ของตัวเองหลายวง (LAN, Guest,
   ฯลฯ) ไม่ใช่แค่ Address เดียว และ Admin ที่ HQ ไม่ต้องนั่ง Assign Prefix ให้ Branch ใหม่ทุก
   Site ด้วยมือ — PD ทำให้ Process นี้เป็น Automatic
5. เพราะ CORE-SW1↔DIST-SW1 เป็น **iBGP ภายในองค์กรเดียวกัน** ที่ Reachability ระหว่าง Loopback0
   ของทั้งสองฝั่งรับประกันได้แน่นอนผ่าน IGP ภายใน (OSPFv2/OSPFv3) — ทำให้ `next-hop-self` บน
   RR ทำงานได้อย่างไว้ใจได้เต็มที่ ไม่ต้องกังวลเรื่อง Next-Hop Resolution ข้าม AF เลย ในขณะที่
   WAN-EDGE-1↔ISP-RTR เป็น **eBGP ข้ามองค์กร** ที่ต้องการ Fault Isolation ระหว่าง AF ชัดเจน
   (ถ้า Session IPv6 มีปัญหาที่ฝั่ง ISP ไม่ควรกระทบ Session IPv4 ที่ยังใช้งานได้) และไม่ต้อง
   พึ่งพา Behavior ของ Next-Hop-Self ข้าม AF ที่ซับซ้อนกว่าการแยก Session ตรงไปตรงมา

---

## สรุป Part 88

Part นี้ยก IPv6 จากความรู้พื้นฐานที่ [Part 8](part-008-ipv6-fundamentals.md) และ OSPFv3 ที่
[Part 27](part-027-ospfv3-ipv6.md) ปูไว้ ขึ้นสู่ระดับ **Production-Grade Enterprise Deployment**
เต็มรูปแบบ — เริ่มจากกลยุทธ์ **Addressing** (เมื่อไหร่ต้องมี GUA จริง เมื่อไหร่ ULA พอ), ต่อด้วย
**Security** ทั้ง ACL/ICMPv6 ระดับ Layer 3 (Step 872) และ **First-Hop Security** ระดับ Layer 2
เต็มรูปแบบ — RA Guard, DHCPv6 Guard, IPv6 Snooping/ND Inspection (Step 873) ซึ่งเป็นคู่ขนานของ
DHCP Snooping/DAI จาก [Part 21](part-021-switch-security.md) ที่ IPv6 ขาดไปตลอด 87 Part ที่
ผ่านมา ตามด้วย **MLD Snooping** ที่ปิด Gap ระดับ Switch ของ Multicast IPv6 ต่อจาก PIM6/MLD
Routing ที่ [Part 83](part-083-enterprise-multicast-deep-dive.md) ทำไว้ระดับ Layer 3 (Step 874)
**DHCPv6 เต็มรูปแบบ** ทั้ง Stateful Server และ Prefix Delegation (Step 875) **BGP สำหรับ IPv6**
ทั้ง 2 รูปแบบ Session (Step 876) **NAT66/NPTv6** (Step 877) จนถึงการมองภาพรวม **Transition
Mechanism ทั้งหมด** และประเมินความเป็นไปได้ของ **IPv6-Only** อย่างสมเหตุสมผล (Step 878-879)
ปิดท้ายด้วย Lab ใหญ่ที่รวมทุกฟีเจอร์เข้าด้วยกันบน Topology จริงของหลักสูตร (Step 880) —
IPv6 ของ Lab นี้จึงไม่ใช่แค่ "Dual-Stack ที่ Ping ได้" อีกต่อไป แต่เป็น IPv6 ที่ **Secure,
Multicast-Aware, Auto-Provisioned ผ่าน PD, และ Route ผ่าน BGP** ครบทุกมุมเทียบเท่า IPv4 ที่
สร้างมาตลอด 87 Part ก่อนหน้า

✅ **พร้อมสำหรับ Part 89**: เราจะย้ายจาก Wired/Routing Infrastructure ไปสู่ **Enterprise
Wireless ระดับ High-Density** — การออกแบบ WLAN สำหรับพื้นที่ผู้ใช้หนาแน่น (Auditorium,
Stadium, Open-Office) ที่ WLC-1/AP-1/AP-2 ของหลักสูตรนี้ต้องรองรับ RF Design, Channel/Power
Planning, Band Steering, และ Client Load Balancing ในระดับที่ Part ก่อนหน้ายังไม่ได้แตะ

**ไปต่อ:** [Part 89 — Enterprise Wireless High-Density Design →](part-089-enterprise-wireless-high-density.md)
