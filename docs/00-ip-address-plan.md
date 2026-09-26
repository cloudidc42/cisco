# IP Address Plan — เอกสารอ้างอิงกลางของหลักสูตร (Canonical Reference)

> เอกสารนี้คือ **แหล่งความจริงเดียว (Single Source of Truth)** สำหรับ IP Addressing, Hostname,
> และ Interface numbering ที่ใช้ตลอดหลักสูตร Part 1-100 ทุก Part ที่เขียนต่อจากนี้ต้องอ้างอิง
> เอกสารนี้เพื่อความสอดคล้องกัน หากมีความขัดแย้งระหว่าง Part ก่อนหน้ากับเอกสารนี้ ให้ถือเอกสารนี้
> เป็นค่าที่ถูกต้อง (บาง transit link ในช่วง Part 9-10 มีการปรับเลขที่อยู่ระหว่างการพัฒนาเนื้อหา
> — เอกสารนี้คือค่าสุดท้ายที่ใช้ตั้งแต่ Part 11 เป็นต้นไป)

## VLAN & Subnet Plan

| VLAN | ชื่อ | Subnet (IPv4) | Subnet (IPv6 ULA) | Default Gateway (สุดท้ายหลัง Part 18 HSRP) |
|---|---|---|---|---|
| 10 | SALES | 10.10.10.0/24 | fd00:10:10:10::/64 | 10.10.10.1 (HSRP VIP) |
| 20 | VOICE | 10.10.20.0/24 | fd00:10:10:20::/64 | 10.10.20.1 (HSRP VIP) |
| 30 | SERVERS | 10.10.30.0/24 | fd00:10:10:30::/64 | 10.10.30.1 (HSRP VIP) |
| 40 | WIFI | 10.10.40.0/24 | fd00:10:10:40::/64 | 10.10.40.1 (HSRP VIP) |
| 99 | MGMT | 10.10.99.0/24 | fd00:10:10:99::/64 | 10.10.99.1 (HSRP VIP) |

ก่อน Part 18 (HSRP): DIST-SW1/DIST-SW3 ใช้ `.2` เป็น real IP ของ SVI, DIST-SW2/DIST-SW4 ใช้ `.3`
(ดู Part 10) — ค่า `.1` (HSRP Virtual IP) จะถูกเพิ่มเข้ามาใน Part 18

## Loopback0 Plan (Router-ID สำหรับทุก Routing Protocol)

| Device | Loopback0 |
|---|---|
| CORE-SW1 | 1.1.1.1/32 |
| CORE-SW2 | 1.1.1.2/32 |
| DIST-SW1 | 1.1.1.11/32 |
| DIST-SW2 | 1.1.1.12/32 |
| DIST-SW3 | 1.1.1.13/32 |
| DIST-SW4 | 1.1.1.14/32 |
| WAN-EDGE-1 | 1.1.1.21/32 |
| WAN-EDGE-2 | 1.1.1.22/32 |
| ISP-RTR | 1.1.1.254/32 (จำลอง ISP เท่านั้น ไม่ใช่ของจริงในหลักสูตร) |

## Transit / Backbone Link Plan (Point-to-Point, /30) — **Canonical ตั้งแต่ Part 11**

### WAN Edge

| Link | Subnet | IP ปลาย A | IP ปลาย B |
|---|---|---|---|
| ISP-RTR ↔ WAN-EDGE-1 | 203.0.113.0/30 | ISP-RTR .1 | WAN-EDGE-1 .2 |
| ISP-RTR ↔ WAN-EDGE-2 | 203.0.113.4/30 | ISP-RTR .5 | WAN-EDGE-2 .6 |
| WAN-EDGE-1 ↔ WAN-EDGE-2 (cross-link) | 10.10.254.8/30 | WAN-EDGE-1 .9 | WAN-EDGE-2 .10 |
| WAN-EDGE-1 ↔ CORE-SW1 | 10.10.254.0/30 | WAN-EDGE-1 .1 | CORE-SW1 .2 |
| WAN-EDGE-2 ↔ CORE-SW2 | 10.10.254.4/30 | WAN-EDGE-2 .5 | CORE-SW2 .6 |

### Core / Distribution Backbone

| Link | Subnet | IP ปลาย A | IP ปลาย B | Interface (ตัวอย่าง) |
|---|---|---|---|---|
| CORE-SW1 ↔ CORE-SW2 (Po1 routed, LACP) | 10.255.0.0/30 | CORE-SW1 .1 | CORE-SW2 .2 | Port-channel1 (no switchport) |
| CORE-SW1 ↔ DIST-SW1 | 10.255.10.0/30 | CORE-SW1 .1 | DIST-SW1 .2 | Te1/0/1 ↔ Te1/1/1 |
| CORE-SW1 ↔ DIST-SW2 | 10.255.20.0/30 | CORE-SW1 .1 | DIST-SW2 .2 | Te1/0/2 ↔ Te1/1/1 |
| CORE-SW2 ↔ DIST-SW3 | 10.255.30.0/30 | CORE-SW2 .1 | DIST-SW3 .2 | Te1/0/1 ↔ Te1/1/1 |
| CORE-SW2 ↔ DIST-SW4 | 10.255.40.0/30 | CORE-SW2 .1 | DIST-SW4 .2 | Te1/0/2 ↔ Te1/1/1 |

> **หมายเหตุ Redundancy**: Lab ตั้งแต่ Part 10 ใช้ single-homed DIST↔CORE (DIST แต่ละตัวต่อ CORE
> ตัวเดียว) เพื่อความง่ายในการสอน Static Route/OSPF เบื้องต้น — การทำ **Dual-homed DIST↔CORE**
> (ต่อทั้ง CORE-SW1 และ CORE-SW2 พร้อม ECMP/redundancy เต็มรูปแบบ) จะเพิ่มเข้ามาใน **Part 33-35**
> (Advanced Campus Design & HA) ของระดับ CCNP ENCOR

## Device Platform Reference

| Hostname | Role | Platform |
|---|---|---|
| ISP-RTR | จำลอง ISP | CSR1000v / IOSv |
| WAN-EDGE-1, WAN-EDGE-2 | WAN Edge Router | ISR4451 / CSR1000v |
| FW-1 | Firewall | ASAv / FTDv |
| CORE-SW1, CORE-SW2 | Core Switch (L3, routed) | Catalyst 9500 |
| DIST-SW1–4 | Distribution Switch (L3, SVI) | Catalyst 9300 |
| ACCESS-SW1–4 | Access Switch (L2 only) | Catalyst 9200/2960 |
| WLC-1 | Wireless Controller | Catalyst 9800-CL |
| AP-1, AP-2 | Access Point | Catalyst 9130 |

## Access-Layer Mapping

| Switch | VLANs | Uplink (Trunk) ไปยัง |
|---|---|---|
| ACCESS-SW1 | 10, 20, 99 | DIST-SW1 |
| ACCESS-SW2 | 10, 20, 99 | DIST-SW2 |
| ACCESS-SW3 | 30, 40, 99 | DIST-SW3 |
| ACCESS-SW4 | 30, 40, 99 | DIST-SW4 |
