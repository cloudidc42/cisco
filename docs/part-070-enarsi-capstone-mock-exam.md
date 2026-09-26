# Part 70 — ENARSI Capstone & Mock Exam
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 691–700 จาก 1000 | ปิดระดับ CCNP ENARSI (300-410)**

> ต่อจาก [Part 69 — Full Troubleshooting Mega-Lab #1](part-069-mega-lab-1.md) นี่คือ **Part สุดท้าย
> ของระดับ CCNP ENARSI (300-410)** — และเพราะ ENARSI คือใบสุดท้ายที่ต้องมีคู่กับ ENCOR
> เพื่อได้ **CCNP Enterprise เต็มใบ**, Part นี้จึงเป็น **Part สุดท้ายของทั้ง CCNP Certification**
> ด้วย เช่นเดียวกับที่ [Part 25](part-025-ccna-capstone-lab.md) ปิดระดับ CCNA และ
> [Part 55](part-055-encor-capstone-lab.md) ปิดระดับ ENCOR, Part นี้**ไม่มีเทคโนโลยีใหม่**ให้เรียน
> แต่เป็น **Capstone + Mock Exam เต็มรูปแบบ** ที่รวบรวมทุก Troubleshooting Domain จาก Part 56-69
> (14 Part เต็ม, Step 551-690) เข้าเป็นภาพเดียว, Mapping เข้ากับ Blueprint ข้อสอบ **Cisco 300-410
> ENARSI** จริงทั้ง 4 Domain, จำลองข้อสอบ 20 ข้อแบบ Multiple-Choice ครบทุก Domain, ฝึก Lab
> Simulation จับเวลาแบบ Ticket-based ที่ ENARSI ขึ้นชื่อเรื่องนี้มากที่สุด และปิดท้ายด้วย
> Final Running-Config Appendix ที่เป็นเอกสารอ้างอิงสมบูรณ์ที่สุดของ**หลักสูตร CCNP เต็มใบ**
> (รวม Part 1-70 ทั้งหมด) ก่อนก้าวสู่ระดับ **CCIE Enterprise Infrastructure** ใน Part 71

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 691 | ENARSI 300-410 Exam Blueprint Mapping — 4 Domain, น้ำหนักคะแนนจริง, Part ที่ครอบคลุม |
| 692 | Full CCNP (ENCOR + ENARSI) Skill-Map Recap — ตาราง Part 1-69 ครบทั้งหลักสูตร |
| 693 | Mock Exam Section A — Layer 3 Technologies (EIGRP/OSPF/BGP/Redistribution) 5 ข้อ พร้อมเฉลย |
| 694 | Mock Exam Section B — VPN Technologies (DMVPN/IPsec/GRE) 5 ข้อ พร้อมเฉลย |
| 695 | Mock Exam Section C — Infrastructure Security (AAA/802.1X/CoPP/ZBFW) 5 ข้อ พร้อมเฉลย |
| 696 | Mock Exam Section D — Infrastructure Services (DHCP/NAT/QoS/Multicast) 5 ข้อ พร้อมเฉลย |
| 697 | Full Timed Lab Simulation — โจทย์ 90 นาที "เครือข่ายพังหลายจุด" พร้อม Model Answer เต็มรูปแบบ |
| 698 | เคล็ดลับวันสอบ ENARSI — Ticket-based Lab Format, จัดสรรเวลา, กัน Tunnel Vision |
| 699 | Career Context — CCNP เต็มใบปลดล็อกอะไรได้บ้าง และเส้นทางสู่ CCIE Enterprise Infrastructure |
| 700 | Final Running-Config Appendix — CORE-SW1 ฉบับสมบูรณ์ที่สุดของทั้งหลักสูตร CCNP (Part 1-70) |

---

## Step 691 — ENARSI 300-410 Exam Blueprint Mapping

Cisco แบ่งข้อสอบ ENARSI 300-410 เป็น **4 Domain** เท่านั้น (น้อยกว่า ENCOR ที่มี 6 Domain) แต่แต่ละ
Domain **เจาะลึกเฉพาะเรื่อง Troubleshooting** มากกว่า ENCOR ที่กว้างครอบคลุมทั้ง Design/Deploy —
ตารางนี้ Mapping แต่ละ Domain เข้ากับน้ำหนักคะแนนจริงและ Part ที่หลักสูตรนี้สอนไปแล้ว (Part 56-69):

| # | Domain (ชื่อทางการ) | น้ำหนัก | Part หลักที่ครอบคลุม |
|---|---|---|---|
| 1.0 | Layer 3 Technologies | **35%** | Part 56 (EIGRP), Part 57 (OSPF), Part 58 (BGP), Part 59 (Redistribution), Part 64 (PBR) |
| 2.0 | VPN Technologies | **20%** | Part 60 (DMVPN/IPsec/GRE Troubleshooting), Part 51-53 (พื้นฐาน ENCOR ที่ใช้ต่อ) |
| 3.0 | Infrastructure Security | **20%** | Part 61 (AAA/802.1X/TrustSec/ZBFW/CoPP Troubleshooting) |
| 4.0 | Infrastructure Services | **25%** | Part 62 (DHCP/NTP/Syslog/SNMP), Part 63 (NAT), Part 65 (Multicast), Part 68 (QoS), Part 67 (Automation/NETCONF-RESTCONF) |
| — | **รวม** | **100%** | Part 56-69 ครบทุก Domain (Part 66 Wireless เป็นเนื้อหาเสริมนอก Blueprint หลักของ 300-410) |

> **ข้อสังเกตสำคัญ**: Domain 1.0 (Layer 3 Technologies, 35%) มีน้ำหนักสูงสุดในข้อสอบ ENARSI —
> มากกว่า Domain อื่นรวมกันครึ่งหนึ่งของคะแนนทั้งหมด สอดคล้องกับความจริงที่ว่า **EIGRP/OSPF/BGP/
> Redistribution คือรากฐานที่ Infrastructure Service และ Security Feature อื่นทั้งหมดต้องพึ่งพา
> Routing ที่ถูกต้องก่อนเสมอ** — ถ้า Routing ผิด ทุก Service ข้างบน (DHCP Relay, NAT, QoS Policy,
> VPN Overlay) ก็พังไปด้วยไม่ว่าจะ Config ถูกแค่ไหน **ควรจัดสรรเวลาทวน Part 56-59 มากกว่า Part อื่น
> อย่างมีนัยสำคัญก่อนสอบจริง**

### รายละเอียดหัวข้อย่อยที่ Cisco ระบุใน Blueprint จริง เทียบกับ Part ที่สอน

| Blueprint Sub-topic (ตัวอย่างสำคัญ) | Part ที่ตรงกัน |
|---|---|
| 1.1 Troubleshoot administrative distance | Part 9, 13 (CCNA), ทวนใน Part 59 |
| 1.2 Troubleshoot routing protocol authentication (EIGRP/OSPF) | Part 56 (EIGRP MD5/SHA), Part 57 (OSPF MD5) |
| 1.3 Troubleshoot loop prevention/redistribution mechanisms (filtering, tagging, split-horizon, route-map) | Part 59 |
| 1.4 Troubleshoot manual/auto-summarization | Part 28 (ENCOR พื้นฐาน), Part 56, Part 57 |
| 1.6-1.9 Troubleshoot EIGRP neighbor, topology table, path selection, Stuck-In-Active | Part 56 |
| 1.10 Troubleshoot EIGRP for IPv6 | Part 56 (Step 554-555) |
| 1.11-1.15 Troubleshoot OSPF neighbor, OSPFv3, path selection, LSA, fast convergence | Part 57 |
| 1.16-1.19 Troubleshoot BGP peer, path selection, PBR, fast convergence | Part 58, Part 64 |
| 2.1 Troubleshoot site-to-site VPN using IPsec | Part 60 |
| 2.2 Troubleshoot MPLS operations (Enterprise WAN) | Part 19 (พื้นฐาน CCNA), แนวคิดต่อยอดใน Part 87 (CCIE) |
| 2.3 Describe DMVPN (single hub) | Part 52, Part 60 |
| 2.4 Describe GET VPN | Part 60 (Step 600, Exam Awareness) |
| 2.5 Describe LISP | Part 38 (SD-Access พื้นฐาน) |
| 3.1 Troubleshoot AAA (TACACS+/RADIUS/local) | Part 22 (พื้นฐาน), Part 61 |
| 3.2-3.3 Troubleshoot router security features, CoPP | Part 46 (พื้นฐาน), Part 61 |
| 3.4 Troubleshoot IPv4/IPv6 ACL | Part 14 (พื้นฐาน), Part 61 |
| 3.5 Troubleshoot Unicast RPF | Part 61 (Step 608 บริบท CoPP ใกล้เคียง) |
| 3.6 Troubleshoot device hardening (control/mgmt/data plane) | Part 22, Part 46, Part 61 |
| 3.7 Describe IPv6 first-hop security | Part 61 (Exam Awareness) |
| 4.1 Troubleshoot device management | Part 2 (พื้นฐาน), Part 62 |
| 4.2 Troubleshoot SNMP | Part 17 (พื้นฐาน), Part 62 (Step 617) |
| 4.3 Troubleshoot logging | Part 17, Part 62 (Step 616) |
| 4.4 Troubleshoot IPv4/IPv6 DHCP | Part 16 (พื้นฐาน), Part 62 (Step 612-614) |
| 4.5 Troubleshoot NAT (IPv4) | Part 15 (พื้นฐาน), Part 63 |
| 4.6 Describe NAT66/NPTv6 | Part 63 (Step 626, Exam Awareness) |
| 4.7 Troubleshoot NETCONF/RESTCONF | Part 49 (พื้นฐาน), Part 67 |
| 4.8 Describe QoS | Part 24, Part 40 (พื้นฐาน), Part 68 |
| 4.9 Describe multicast | Part 41-42 (พื้นฐาน), Part 65 |

---

## Step 692 — Full CCNP (ENCOR + ENARSI) Skill-Map Recap

นี่คือตารางสรุปที่ใหญ่ที่สุดในหลักสูตรจนถึงจุดนี้ — **ทุก Part ตั้งแต่ 1 ถึง 69** ถูก Mapping เข้ากับ
4 Domain ของ ENARSI (ซึ่งเป็น Domain ที่ปิดคอร์ส CCNP) พร้อมระบุว่าแต่ละ Domain มีรากฐานมาจาก
CCNA Part ไหน, ต่อยอดด้วย ENCOR Part ไหน, และเจาะลึก Troubleshoot ด้วย ENARSI Part ไหน

### ตารางที่ 1 — 4 Domain ของ ENARSI × Full-Stack Part Coverage (Part 1-69)

| ENARSI Domain | น้ำหนัก | รากฐาน CCNA (Part) | ต่อยอด ENCOR (Part) | เจาะลึก ENARSI Troubleshooting (Part) |
|---|---|---|---|---|
| 1.0 Layer 3 Technologies | 35% | Part 9 (Static), 11 (OSPF เบื้องต้น), 13 (EIGRP เบื้องต้น) | Part 26-28 (OSPF/EIGRP Advanced), 29-32 (BGP/Redistribution/PBR) | Part 56 (EIGRP), 57 (OSPF), 58 (BGP), 59 (Redistribution), 64 (PBR) |
| 2.0 VPN Technologies | 20% | Part 19 (WAN/VPN Intro) | Part 51 (VRF/GRE/IPsec), 52-53 (DMVPN) | Part 60 (DMVPN/IPsec/GRE Troubleshooting) |
| 3.0 Infrastructure Security | 20% | Part 14 (ACL), 21-22 (Switch Security/AAA) | Part 43-46 (ISE/802.1X/TrustSec/ZBFW/CoPP) | Part 61 (Infra Security Troubleshooting) |
| 4.0 Infrastructure Services | 25% | Part 15-17 (NAT/DHCP/NTP/Syslog/SNMP), 24 (QoS เบื้องต้น) | Part 40 (Advanced QoS), 41-42 (Multicast), 47-50 (Automation/DNAC) | Part 62 (Infra Services), 63 (NAT), 65 (Multicast), 67 (Automation), 68 (QoS) |

### ตารางที่ 2 — 6 Domain ของ ENCOR (จาก Part 55 Step 542) ↔ เสริมความเข้มด้วย Part ENARSI ไหน

| ENCOR Domain (350-401) | น้ำหนัก ENCOR | Part ENCOR หลัก | Part ENARSI ที่กลับมาเจาะลึกด้าน Troubleshooting |
|---|---|---|---|
| 1.0 Architecture | 15% | Part 33-34, 38-39, 51-53 | Part 60 (DMVPN Troubleshooting) |
| 2.0 Virtualization | 10% | Part 51-53 | Part 60 |
| 3.0 Infrastructure | 30% | Part 26-32, 35-37, 40-42 | Part 56-59 (Routing), 64 (PBR), 65 (Multicast), 68 (QoS), 66 (Wireless) |
| 4.0 Network Assurance | 10% | Part 54 | Part 62 (Syslog/SNMP), 67 (Automation Assurance) |
| 5.0 Security | 20% | Part 43-46 | Part 61 |
| 6.0 Automation | 15% | Part 47-50 | Part 67 |

> **ข้อสังเกตสำคัญที่สุดของ Full-Stack Skill Map นี้**: สังเกตว่า **ไม่มี Part ไหนใน ENARSI ที่ไม่มี
> รากฐานมาจาก CCNA/ENCOR มาก่อน** — ENARSI ไม่ได้สอน Protocol/Feature ใหม่แม้แต่ตัวเดียว มันคือการ
> เอา Feature ที่ Config สำเร็จไปแล้วทั้งหมด (ตั้งแต่ Part 1) มา **"ทำให้พัง" อย่างมีเหตุผล แล้วฝึก
> วินิจฉัยกลับ** — นี่คือเหตุผลที่ข้อสอบ ENARSI มักถูกมองว่า "ยากกว่า ENCOR ทั้งที่เนื้อหาน้อยกว่า"
> เพราะมันวัด**ทักษะการวินิจฉัย (Diagnostic Skill)** ไม่ใช่แค่ความรู้ Syntax — ทักษะที่สร้างจาก
> การฝึกซ้ำๆใน Part 56-69 เท่านั้น ไม่มีทางลัดจากการอ่านทฤษฎีเพียวๆ

### Checklist ทักษะที่ต้องมั่นใจ 100% ก่อนสอบจริง (Self-Assessment)

| ทักษะ | มั่นใจหรือยัง? | ถ้ายังไม่มั่นใจ ให้กลับไปที่ |
|---|---|---|
| อ่าน `show ip eigrp topology` แล้วบอก Successor/Feasible Successor ได้ทันที | [ ] | Part 13, 56 |
| แยกอาการ OSPF ExStart ค้าง (MTU Mismatch) ออกจาก Init ค้าง (Auth/Area Mismatch) ได้ | [ ] | Part 11-12, 57 |
| อธิบาย BGP Best Path Selection Algorithm ได้ครบ 13 ขั้นตอนตามลำดับ | [ ] | Part 29-30, 58 |
| เขียน Route-map + Tag ป้องกัน Redistribution Loop ได้โดยไม่ดู Note | [ ] | Part 31, 59 |
| วินิจฉัย DMVPN Phase 3 ที่ไม่ Shortcut ได้ภายใน 5 นาที | [ ] | Part 52-53, 60 |
| อธิบาย AAA Fallback Chain และจุดที่ Lock-out ตัวเองได้ | [ ] | Part 22, 61 |
| อ่าน CoPP Drop Counter แล้วรู้ว่า Class ไหนกำลังกิน Control-Plane Traffic จริง | [ ] | Part 46, 61 |
| ไล่ Order of Operations ของ NAT ระหว่าง ACL/Routing/NAT ได้ | [ ] | Part 15, 63 |

---

## Step 693 — Mock Exam Section A: Layer 3 Technologies (5 ข้อ)

**คำถามที่ 1 — EIGRP Stuck-In-Active**

โจทย์: `EIGRP-R1` รายงาน Log ต่อไปนี้ และ Neighbor กับ `DIST-SW4` หลุดทันทีหลังจากนั้น

```
%DUAL-3-SIA: Route 10.10.60.0/24 stuck-in-active state in IP-EIGRP(0) 100.
```

พิจารณาสาเหตุที่ **เป็นไปได้มากที่สุด**:

A. `EIGRP-R1` มี Passive Interface ตั้งผิดฝั่ง
B. Neighbor ปลายทางที่ต้อง Query ไม่ตอบ Reply กลับมาภายในเวลาที่กำหนด (Query ค้างอยู่ในสถานะ Active นานเกินไป)
C. Administrative Distance ของ EIGRP สูงกว่า OSPF
D. `EIGRP-R1` ไม่มี `router-id` ตั้งไว้

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

SIA (Stuck-In-Active) เกิดขึ้นเมื่อ Router ส่ง **Query** ไปหา Neighbor เพื่อหา Feasible Successor
ใหม่ (หลัง Successor เดิมหาย) แต่ **ไม่ได้รับ Reply กลับมาภายใน Active Timer (Default 3 นาที)**
— สาเหตุที่พบบ่อยที่สุดคือ Neighbor ปลายทาง (หรือ Neighbor ของ Neighbor ต่อไปเป็นทอดๆ) มีปัญหา
เช่น CPU สูงมากจนตอบ Reply ไม่ทัน, Link มี Congestion รุนแรง, หรือ Neighbor Chain ยาวเกินไป
(Query Scoping ไม่ดี) ทำให้ Query กระจายไปไกลเกินจำเป็น

ตัวเลือก A ทำให้ Neighbor ไม่ขึ้นเลยตั้งแต่แรก ไม่ใช่ SIA (ซึ่งเกิดหลัง Neighbor Up แล้ว), C ผิด
เพราะ AD ไม่เกี่ยวกับ SIA เลย (SIA เป็นปัญหาภายใน EIGRP Query/Reply Process), D ทำให้ Neighbor
ไม่ Form ตั้งแต่ต้นเช่นกัน — วิธีแก้จริงคือใช้ `ip summary-address eigrp` ที่ขอบ (Query Boundary)
เพื่อจำกัดขนาด Query Domain ไม่ให้ Query กระจายไปทั้ง Topology (ดู Part 56 Step 556)
</details>

---

**คำถามที่ 2 — OSPF Neighbor ค้างที่ ExStart**

โจทย์: `CORE-SW1` และ `DIST-SW1` เชื่อมผ่าน Port-channel11 — `show ip ospf neighbor` แสดง State
`EXSTART/BDR` ค้างอยู่ไม่ขยับ ทั้งที่ Physical Link Up/Up ปกติ

พิจารณาสาเหตุที่ **เป็นไปได้มากที่สุด**:

A. Area Number ทั้งสองฝั่งไม่ตรงกัน
B. MTU ของทั้งสองฝั่งไม่ตรงกัน ทำให้ DBD (Database Description) Packet แรกที่มีขนาดใหญ่ส่งไม่ผ่าน
C. Router-ID ของทั้งสองฝั่งซ้ำกัน
D. Authentication Key ไม่ตรงกัน

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

State **ExStart** คือขั้นตอนที่ทั้งสอง Router กำลังตกลงกันว่าใครเป็น Master/Slave สำหรับการ
แลกเปลี่ยน DBD — ถ้าค้างที่นี่ (ไม่ขยับไป Exchange) และ Neighbor **ขึ้นมาได้ถึง 2-Way/ExStart
สำเร็จแล้ว** (ตัดข้อ A, C, D ออกได้ทันที เพราะ Area/Router-ID ซ้ำ/Auth ผิด จะทำให้ค้างที่ Down/Init
ไม่ใช่ ExStart) **สาเหตุคลาสสิกที่สุดคือ MTU Mismatch** — DBD Packet ที่มีขนาดใหญ่กว่า MTU ของ
อีกฝั่งจะถูก Drop เงียบๆ (ไม่มี ICMP Fragmentation Needed เพราะ OSPF ไม่ใช้ Path MTU Discovery
แบบ TCP) ทำให้ Router ทั้งสองคุยกันไม่จบสักที

วิธี Verify: `show interface Port-channel11 | include MTU` เทียบทั้งสองฝั่ง — ถ้าไม่ตรงกัน แก้ด้วย
`ip mtu <ค่าที่ตรงกัน>` บน Interface (หรือใช้ `ip ospf mtu-ignore` เป็นทางลัด ซึ่งข้อสอบมักถามควบคู่
กันว่าเป็น Workaround ไม่ใช่การแก้ Root Cause จริง)
</details>

---

**คำถามที่ 3 — BGP Path Selection**

โจทย์: `WAN-EDGE-1` เห็น 2 Path ไปยัง Prefix `8.8.8.0/24` ในตาราง BGP ดังนี้:

```
Path 1: via 203.0.113.1 (AS 65000), Weight 0, LocalPref 100, AS-Path: 65000, Origin IGP
Path 2: via 1.1.1.22 (AS 65001, iBGP), Weight 0, LocalPref 200, AS-Path: 65002, Origin IGP
```

`WAN-EDGE-1` จะเลือก Path ใดเป็น Best Path?

A. Path 1 เพราะเป็น eBGP (ชนะ iBGP เสมอในทุกกรณี)
B. Path 2 เพราะ LocalPref สูงกว่า (200 > 100)
C. Path 1 เพราะ AS-Path สั้นกว่า
D. เท่ากันทั้งคู่ Router จะ Load-balance อัตโนมัติ

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

BGP Best Path Algorithm เทียบ Attribute ตามลำดับความสำคัญ (ทวนจาก Part 30): **1) Weight →
2) Local Preference → 3) Locally Originated → 4) AS-Path Length → ...** — ทั้งสอง Path มี
**Weight เท่ากัน (0)** จึงต้องเทียบขั้นถัดไปคือ **Local Preference ก่อน AS-Path เสมอ** — Path 2
มี LocalPref 200 สูงกว่า Path 1 (100) จึง **ชนะทันทีตรงนี้ ไม่ต้องพิจารณา AS-Path Length เลย**

ตัวเลือก A ผิด เพราะ "eBGP ชนะ iBGP" เป็นเกณฑ์ที่ใช้เทียบใน **ขั้นตอนหลัง** Local Preference
(ลำดับที่ 8 ของ Algorithm) — Local Preference มาก่อนเสมอถ้าค่าไม่เท่ากัน ตัวเลือก C ผิดด้วยเหตุผล
เดียวกัน (AS-Path Length อยู่ลำดับที่ 4 ซึ่งมาหลัง LocalPref) — นี่คือกับดักคลาสสิกที่ข้อสอบ ENARSI
ชอบออกมากที่สุดข้อหนึ่ง: **จำลำดับ Algorithm ผิด = ตอบผิดทันทีแม้จะรู้ Attribute ทุกตัว**
</details>

---

**คำถามที่ 4 — Redistribution Loop**

โจทย์: `DIST-SW3` (EIGRP↔OSPF Mutual Redistribution) มี Config ดังนี้:

```
DIST-SW3(config)# route-map OSPF-TO-EIGRP permit 10
DIST-SW3(config-route-map)# set tag 110
DIST-SW3(config)# route-map EIGRP-TO-OSPF deny 10
DIST-SW3(config-route-map)# match tag 110
DIST-SW3(config)# route-map EIGRP-TO-OSPF permit 20
```

Engineer สังเกตว่า Route บางตัวที่ Originate จาก OSPF เอง (ไม่ได้มาจาก EIGRP) กลับ **ถูก
Redistribute วนกลับเข้า OSPF ซ้ำอีกรอบผ่านทาง EIGRP** ทำให้เห็น Route เดิม 2 ครั้งจาก 2 Source
สาเหตุที่เป็นไปได้มากที่สุดคือ?

A. `route-map OSPF-TO-EIGRP` ไม่มี `deny` ปิดท้าย จึงยอมให้ทุก Route ผ่านเข้า EIGRP รวมถึง Route
   ที่ควรกันไว้ไม่ให้ Redistribute กลับ
B. `set tag 110` ผิด ต้องเป็น `set tag 90`
C. EIGRP AS Number ผิด
D. OSPF Process ID ผิด

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: A**

หลักการ Anti-Redistribution-Loop ด้วย Route-map + Tag (ทวนจาก Part 31, 59) ต้องทำ **2 ทิศทาง
พร้อมกันเสมอ**: ทิศทางหนึ่ง `set tag` เพื่อ "ปั๊มตราประทับ" ว่า Route นี้มาจากไหน, อีกทิศทาง
`deny ... match tag` เพื่อ "กันไม่ให้ Route ที่มีตราประทับนั้นวนกลับเข้าไปที่เดิม" — ในโจทย์นี้
`OSPF-TO-EIGRP` (ทิศทาง OSPF→EIGRP) มีแค่ `permit 10` ที่ `set tag 110` **แต่ไม่มี Sequence ที่
`deny` Route ที่มี Tag จาก EIGRP อยู่แล้ว (เช่น tag 90) ไม่ให้เข้า EIGRP ซ้ำ** — ทำให้ Route ที่
เพิ่งออกจาก EIGRP เข้า OSPF (มี tag 90 ติดมา) ถูกดึงกลับเข้า EIGRP อีกครั้งผ่าน Sequence 10 นี้
(เพราะไม่มีเงื่อนไข `match` ใดๆกรองเลย แค่ `set tag` เฉยๆ) เกิดเป็น Loop สมบูรณ์

การแก้: เพิ่ม `route-map OSPF-TO-EIGRP deny 5 / match tag 90` **ก่อน** Sequence 10 เสมอ
(Sequence Number ต้องน้อยกว่า เพราะ Route-map ประมวลผลจากบนลงล่าง)
</details>

---

**คำถามที่ 5 — OSPF Area Type Mismatch**

โจทย์: `DIST-SW4` (Area 2) ตั้งค่า `area 2 nssa` แต่ `CORE-SW2` (ABR ของ Area 2) ยังตั้งเป็น
`area 2` ปกติ (ไม่ใส่ nssa) ผลลัพธ์ที่คาดว่าจะเกิดคือ?

A. ไม่มีผลกระทบใดๆ NSSA เป็นแค่ Cosmetic Setting
B. Neighbor Adjacency จะไม่ขึ้นเลย (ค้างที่ Down หรือ Init) เพราะ Area Option Mismatch
C. Adjacency ขึ้นได้ปกติ แต่ Type-7 LSA จะไม่ถูกแปลงเป็น Type-5 ที่ ABR
D. OSPF จะ Reset ตัวเองอัตโนมัติเพื่อ Sync ค่าให้ตรงกัน

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

Area Type (Normal / Stub / NSSA) ถูกเข้ารหัสอยู่ใน **Options Field ของ Hello Packet** (E-bit
สำหรับ Stub, N-bit สำหรับ NSSA) — OSPF ตรวจสอบ Options Field นี้เป็นหนึ่งในเงื่อนไข **ต้องตรงกัน
ทุกตัวอักษรก่อน Adjacency จะขึ้นได้เลย** เหมือนกับ Area Number, Hello/Dead Timer, Authentication
— ถ้า Router หนึ่งประกาศตัวเป็น NSSA (N-bit=1) แต่อีกฝั่งเป็น Normal Area (N-bit=0) ทั้งสองจะเห็น
Options Mismatch และ**ปฏิเสธ Adjacency ทันที** ไม่ต่างจาก Area Number ไม่ตรงกันเลย

Log ที่จะเห็นคือ `%OSPF-5-ADJCHG` ไม่ขึ้น Full พร้อม Debug (`debug ip ospf adj`) แสดง
`Nbr has different [NSSA/Normal] area option` การแก้คือต้องตั้ง `area 2 nssa` ให้ตรงกัน**ทุกตัว**
ใน Area นั้น (ทั้ง ABR และ Internal Router) พร้อมกัน — ตรงกับกฎ Runbook ของ Part 55 Step 543 ที่
บอกไว้ว่า "ต้องทำที่ ABR ก่อน แล้วค่อยทำที่ Internal Router" เพื่อลดช่วงเวลาที่ Mismatch เกิดขึ้นจริง
</details>

---

## Step 694 — Mock Exam Section B: VPN Technologies (5 ข้อ)

**คำถามที่ 1 — DMVPN Phase 3 ไม่ Shortcut**

โจทย์: `BRANCH-RTR` และ `BRANCH-RTR-2` (Spoke ของ DMVPN Phase 3 Dual-Hub) Traffic ยังวิ่งผ่าน Hub
เสมอ ตรวจสอบ Hub (`WAN-EDGE-1`) พบว่ามี `ip nhrp redirect` ตั้งไว้แล้ว แต่ Spoke ยังไม่ Shortcut
สาเหตุที่เป็นไปได้มากที่สุด?

A. Hub ต้องมี `ip nhrp shortcut` ไม่ใช่ Spoke
B. Spoke ไม่มี `ip nhrp shortcut` — Spoke เพิกเฉยต่อ NHRP Redirect ที่ Hub ส่งมา
C. Tunnel Key ไม่ตรงกัน ทำให้ Redirect ส่งไม่ถึง
D. NHRP Network-ID ต้องเป็นคนละค่ากันระหว่าง Hub/Spoke

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

ทวนจาก Part 53/60: DMVPN Phase 3 Spoke-to-Spoke Shortcut ต้องมี **2 องค์ประกอบพร้อมกันเสมอ**:
`ip nhrp redirect` ที่ **Hub** (แจ้ง Spoke ว่ามีทางลัด) และ `ip nhrp shortcut` ที่ **Spoke**
(ยอมรับ Redirect แล้วสร้าง CEF Entry ใหม่ชี้ตรงไปยัง Spoke ปลายทาง) — โจทย์บอกว่า Hub ตั้ง
`redirect` ไว้แล้ว แต่ไม่ได้บอกว่า Spoke ตั้ง `shortcut` จึงเป็นจุดที่ต้องสงสัยอันดับแรก ถ้า Spoke
ไม่มีคำสั่งนี้ **จะไม่มีทางสร้าง Shortcut Entry ได้เลยไม่ว่า Hub จะส่ง Redirect มาเท่าไหร่ก็ตาม**
(Spoke จะยัง Route ตาม Routing Table เดิมที่ชี้ผ่าน Hub อยู่ดี)

ตัวเลือก C, D ผิด เพราะ Tunnel Key/Network-ID ผิดจะทำให้ **NHRP Mapping พื้นฐาน (Hub-Spoke) ไม่ขึ้น
เลยตั้งแต่แรก** (จะไม่เห็น `show dmvpn` แสดง Peer เป็น UP) ซึ่งขัดกับโจทย์ที่บอกว่า Basic
Connectivity ยังทำงานได้ (แค่ไม่ Shortcut) — ต้องแยกอาการ "ไม่ขึ้นเลย" กับ "ขึ้นแต่ไม่ Shortcut"
ให้ชัด
</details>

---

**คำถามที่ 2 — IPsec Phase 1 ไม่ขึ้น**

โจทย์: `show crypto isakmp sa` บน `WAN-EDGE-1` แสดงผลว่างเปล่า (ไม่มี SA เลย) ทั้งที่ GRE Tunnel
Underlay (`ping` ไปยัง Tunnel Destination) ผ่านปกติ สาเหตุที่เป็นไปได้มากที่สุด?

A. GRE Tunnel MTU ตั้งผิด
B. `crypto isakmp policy` (Encryption/Hash/DH Group/Lifetime) หรือ Pre-shared Key ไม่ตรงกัน
   ระหว่างสองฝั่ง
C. `tunnel mode gre multipoint` ตั้งผิดเป็น `tunnel mode gre ip`
D. OSPF Neighbor ยังไม่ขึ้นบน Tunnel Interface

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

`show crypto isakmp sa` ว่างเปล่าแปลว่า **IKE Phase 1 ยังไม่เริ่มต้นสำเร็จเลยแม้แต่ Negotiation**
— เงื่อนไขที่ทำให้ Phase 1 ล้มเหลวโดยไม่มี SA ใดๆขึ้นมาแม้ชั่วคราวคือ **ISAKMP Policy Parameter
ไม่ Match กัน** (ต้องตรงกันทุกตัว: Encryption Algorithm, Hash, Authentication Method, DH Group,
Lifetime) หรือ **Pre-shared Key ผิด** — ทั้งสองกรณีนี้ Router จะปฏิเสธ Negotiation ตั้งแต่ต้นโดย
ไม่สร้าง SA ใดๆเลย (ต่างจาก Phase 2/IPsec SA ที่ถ้าตั้งผิดจะยังเห็น Phase 1 SA ขึ้นแต่ Phase 2 ล้ม)

ตัวเลือก A, C, D เป็นปัญหาที่ **Layer อื่น** (Underlay/GRE Encapsulation/Routing) ซึ่งโจทย์ระบุ
ชัดว่า GRE Underlay Ping ผ่านปกติอยู่แล้ว (ตัด A, C ออกได้) และ OSPF เป็น Overlay ที่ทำงาน**หลัง**
IPsec ขึ้นสำเร็จเท่านั้น (ตัด D ออกได้ — OSPF ที่ไม่ขึ้นเป็นผลลัพธ์ ไม่ใช่สาเหตุ) วิธี Verify ต้อง
เทียบ `show run | section crypto isakmp policy` ทั้งสองฝั่งให้ตรงกันทุกบรรทัด
</details>

---

**คำถามที่ 3 — GRE Recursive Routing**

โจทย์: `WAN-EDGE-1` Config GRE Tunnel ไปยัง `BRANCH-RTR` แล้วพบ Log:

```
%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing
```

สาเหตุที่เป็นไปได้มากที่สุด?

A. Tunnel Key ไม่ตรงกันทั้งสองฝั่ง
B. Routing Table มี Route ไปยัง Tunnel Destination (NBMA Address) ที่ชี้ผ่าน Tunnel Interface
   เอง (วนกลับเข้าตัวเอง)
C. MTU ของ Tunnel Interface ใหญ่เกินไป
D. `keepalive` บน Tunnel Interface ไม่ได้เปิด

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

**Recursive Routing** เกิดขึ้นเมื่อ Router พยายามหา Path ไปยัง **Tunnel Destination (NBMA/
Physical Address ปลายทาง)** แต่ Best Route ที่เจอใน Routing Table **ชี้ผ่าน Tunnel Interface
เอง** — สร้าง Loop เชิงตรรกะ (ต้องเข้า Tunnel เพื่อไปหา Tunnel Destination ที่ต้องใช้ตอนสร้าง
Tunnel) IOS ตรวจจับกรณีนี้และ **Shutdown Tunnel Interface อัตโนมัติชั่วคราว** เพื่อป้องกัน Loop
จริง — สาเหตุที่พบบ่อยที่สุดคือ **Routing Protocol (เช่น OSPF/EIGRP ที่วิ่งบน Tunnel) ดันประกาศ
Route ครอบคลุม Tunnel Destination กลับเข้าไปโดยไม่ตั้งใจ** (เช่น Redistribute Connected แบบไม่
กรอง หรือ Summarization ที่กว้างเกินไป)

วิธีแก้: ตรวจ `show ip route <tunnel-destination-ip>` — ถ้าเห็น Route ชี้ผ่าน Tunnel Interface
เอง ต้องแก้ Redistribution/Summarization ไม่ให้ครอบคลุม NBMA Address ของ Tunnel Destination
(ทางแก้ทั่วไปคือใส่ Tunnel Destination ผ่าน Static Route หรือ Routing Protocol Underlay ที่**ไม่
วิ่งบน Tunnel** เท่านั้น)
</details>

---

**คำถามที่ 4 — NHRP Registration ล้มเหลว**

โจทย์: `BRANCH-RTR` (Spoke ใหม่) ไม่ปรากฏใน `show dmvpn` ของ `WAN-EDGE-1` (Hub) เลย ตรวจสอบ
`BRANCH-RTR` พบว่า Tunnel Interface Up/Up และ `ip nhrp nhs 172.16.220.1 nbma 203.0.113.2` ตั้งไว้
ถูกต้อง แต่ `WAN-EDGE-1` มี `ip nhrp network-id 2` (ตัวเลขต่างจาก Spoke ที่ตั้ง `network-id 1`)
ผลลัพธ์ที่คาดว่าจะเกิดคือ?

A. NHRP Registration จะสำเร็จ เพราะ Network-ID เป็นแค่ Label ไม่มีผลต่อการทำงาน
B. NHRP Registration จะล้มเหลว เพราะ Network-ID ต้อง**ตรงกันทุกตัว**ในทุก Node ของ DMVPN Cloud
   เดียวกัน — เป็นค่าที่ระบุขอบเขตของ NHRP Domain
C. Tunnel จะ Flap ตลอดเวลาแบบไม่แน่นอน
D. IPsec Phase 1 จะล้มเหลวก่อน NHRP จะได้ทำงานเลย

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

`ip nhrp network-id` **เป็น Local Significant เท่านั้นในเชิง Syntax แต่ต้องตรงกันทุกตัวในทาง
ปฏิบัติ** เพื่อให้ Router มองว่าอยู่ใน NHRP Domain (Cloud) เดียวกัน — แม้ Network-ID จะไม่ได้ส่งค่า
ไปเทียบกันตรงๆแบบ Authentication Key แต่ Cisco IOS ใช้มันเป็น**ตัวแบ่ง NHRP Cache แยกตาม Cloud**
ภายในอุปกรณ์เดียวกันที่อาจมีหลาย DMVPN Interface — ถ้า Hub และ Spoke ตั้งเลขต่างกัน NHRP
Registration Request ที่ Spoke ส่งไปจะไม่ถูกจับคู่เข้ากับ Mapping Entry ที่ถูกต้องของ Hub

ค่านี้ไม่เหมือน `tunnel key` (ซึ่งใส่ไว้ใน GRE Header จริงและ**ต้องตรงกัน**เพื่อให้ Encapsulation/
De-encapsulation อ่านตรงกัน) แต่ผลลัพธ์ทางปฏิบัติเหมือนกันคือ **ถ้าไม่ตรงกัน DMVPN จะไม่ทำงาน**
วิธี Verify: `show run interface Tunnel0 | include network-id` เทียบทุกอุปกรณ์ในกลุ่มเดียวกันให้
ตรงกันทั้งหมด
</details>

---

**คำถามที่ 5 — GET VPN Concept (Exam Awareness)**

โจทย์: ข้อใดอธิบาย **GET VPN (Group Encrypted Transport VPN)** ได้ถูกต้องที่สุด เมื่อเทียบกับ
DMVPN?

A. GET VPN ใช้ mGRE เหมือน DMVPN แต่เพิ่ม NHRP เข้าไป
B. GET VPN ไม่ใช้ Tunnel Interface เลย — ใช้ Group Domain of Interpretation (GDOI) แจก Group
   Key ให้สมาชิกทุกตัวใช้ Encrypt/Decrypt Traffic บน Native Routing Path เดิม จึงรักษา Full-Mesh
   และ QoS/Multicast ตาม Original Routing ได้โดยไม่ต้องผ่าน Hub
C. GET VPN ใช้ได้เฉพาะ Point-to-Point เท่านั้น ไม่รองรับ Full-Mesh
D. GET VPN คือชื่อรุ่นใหม่ของ DMVPN Phase 3

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

Blueprint ENARSI (2.4) ระบุแค่ "Describe" GET VPN ไม่ต้อง Config จริง แต่ข้อสอบชอบถามแนวคิดเทียบกับ
DMVPN — ความต่างหลักคือ **GET VPN ไม่สร้าง Overlay Tunnel Topology ใหม่เลย** มันใช้ **GDOI
(Group Domain of Interpretation)** ให้ Key Server แจก Group Key เดียวกันให้สมาชิกทุกตัวในกลุ่ม
(Group Member) แล้ว Encrypt Traffic **บน Native IP Routing Path เดิม** (มักเป็น MPLS/Enterprise
WAN ที่ Routing อยู่แล้ว) — ข้อดีคือรักษาคุณสมบัติ Full-Mesh, Any-to-Any, QoS Marking แบบเดิมของ
Underlay ได้เต็มที่ (เพราะไม่มี Tunnel Encapsulation Overhead แบบ DMVPN) เหมาะกับ MPLS L3VPN ที่
Provider ให้ Full-Mesh Underlay อยู่แล้ว และต้องการ Encrypt เพิ่มโดยไม่เปลี่ยน Routing Topology

ตัวเลือก A, C, D สลับสับสนระหว่าง DMVPN (ใช้ mGRE+NHRP, Hub-Spoke ก่อนมี Phase 3) กับ GET VPN
(ไม่มี Tunnel Interface เลย) ซึ่งเป็นกับดักคลาสสิกของคำถาม Exam Awareness แบบนี้
</details>

---

## Step 695 — Mock Exam Section C: Infrastructure Security (5 ข้อ)

**คำถามที่ 1 — AAA Lockout**

โจทย์: `AAA-SRV` (10.10.99.51) ล่มกะทันหัน Engineer พบว่าไม่สามารถ SSH เข้า `DIST-SW1` ได้เลย
แม้ Config มี `aaa authentication login default group tacacs+ local` สาเหตุที่เป็นไปได้มากที่สุด?

A. `local` ใน Fallback Chain ใช้งานไม่ได้เพราะไม่มี `username ... secret` ตั้งไว้ล่วงหน้า
B. TACACS+ Key ผิด
C. `line vty` ไม่มี `login authentication default` เรียกใช้ Method List
D. Access-class บล็อก Source IP ของ Engineer

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: A**

Method List `group tacacs+ local` ควร Fallback ไปที่ `local` ได้เมื่อ TACACS+ Server **ไม่
Response เลย (Timeout)** — แต่ Fallback นี้จะใช้งานได้จริงก็ต่อเมื่อมี **`username <ชื่อ> secret
<รหัส> privilege 15`** ตั้งไว้ในอุปกรณ์ล่วงหน้าแล้วเท่านั้น (ตามกฎเหล็กของ Part 22/25: "ต้องมี local
เป็น Fallback เสมอ ไม่งั้นถ้า AAA-SRV ล่ม จะ Login เข้าอุปกรณ์ไม่ได้เลย") — ถ้าไม่มี Local User
ตั้งไว้ (หรือถูกลบไปโดยไม่ตั้งใจ) Fallback Chain จะไปถึง `local` แล้ว **ไม่มี User ให้ Authenticate
ผ่านได้เลย** เกิด Lockout ทันทีที่ TACACS+ Server ล่ม

ตัวเลือก B, C, D เป็นปัญหาที่ **ป้องกัน TACACS+ ไม่ให้ทำงาน** ซึ่งควรทำให้ Fallback ไปที่ `local`
ได้ปกติ (ถ้า Local User มีอยู่จริง) — จึงไม่ใช่สาเหตุของ "Lockout สมบูรณ์" แบบในโจทย์
</details>

---

**คำถามที่ 2 — 802.1X ค้างที่ Unauthorized**

โจทย์: Port บน `ACCESS-SW1` แสดง `show authentication sessions interface Gi1/0/5` เป็น
`Authz Success = No`, `Method State = Authc Failed` ต่อเนื่อง และ ISE (`ISE-1`, 10.10.99.65)
ไม่ตอบสนอง Request เลย (`show radius statistics` เห็น Timeout สูงมาก) แต่ Config มี
`authentication event fail action next-method` และ `mab` ตั้งไว้ สาเหตุที่ Port ยังใช้งานไม่ได้
เลยคือ?

A. MAB ควรทำงานแทนได้ แต่ MAC Address ของ PC ไม่ได้อยู่ใน Endpoint Database ของ ISE เช่นกัน
   (ISE ล่ม MAB ก็ Fail ด้วยเหตุผลเดียวกัน)
B. `dot1x pae authenticator` ไม่ได้ตั้งไว้บน Port
C. VLAN ที่ Port อยู่ไม่มีใน Trunk ฝั่ง Uplink
D. Switchport Mode ตั้งเป็น Trunk ไม่ใช่ Access

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: A**

โจทย์บอกชัดว่า **RADIUS Server (ISE-1) Timeout ทั้งหมด** — แปลว่าปัญหาไม่ใช่ที่ Local Config
ของ Switch (dot1x/MAB Config ถูกต้องอยู่แล้วตามที่ระบุ) แต่เป็นที่ **RADIUS Server ไม่ Reachable
เลย** — ทั้ง 802.1X (Authc) และ MAB (ซึ่งก็ส่ง RADIUS Request ไปยัง Server เดียวกัน) จะ **Fail
ด้วยเหตุผลเดียวกันทั้งคู่** เพราะไม่มี Server ตอบให้ Authenticate ผ่านได้เลย ไม่เกี่ยวกับว่า MAC
Address อยู่ใน Endpoint DB หรือไม่ (ISE ยังไม่ได้รับ Request เลยด้วยซ้ำ)

ทางแก้ที่ถูกต้องคือ **Critical VLAN / Critical Authentication** (`authentication event server
dead action authorize vlan <critical-vlan>`) — Feature ที่ออกแบบมาเฉพาะสำหรับกรณี RADIUS Server
ตายทั้งหมด ให้ Port ยัง Authorize เข้า VLAN ฉุกเฉินที่กำหนดไว้ล่วงหน้าได้ ไม่ต้องรอ Server กลับมา
(ทวนจาก Part 61 Step 604 ที่ตรงกับ Scenario นี้เป๊ะ)
</details>

---

**คำถามที่ 3 — CoPP บล็อก Routing Protocol**

โจทย์: ระหว่างเกิด Network Event (Link Flap จำนวนมาก) `CORE-SW1` เริ่ม Drop OSPF Hello และ BGP
Update Packet ทำให้ Neighbor หลุดเป็นระยะ ตรวจสอบพบ:

```
CORE-SW1# show policy-map control-plane class COPP-CRITICAL
  police:
      2000000 bps, 31250 limit, 31250 extended limit
    conformed 1850000 bps, exceed 620000 bps
```

สาเหตุที่เป็นไปได้มากที่สุด?

A. `COPP-ROUTING-ACL` ไม่ครอบคลุม OSPF/BGP Protocol
B. Rate ที่ตั้งไว้สำหรับ Class `COPP-CRITICAL` (2 Mbps) ต่ำเกินไปสำหรับปริมาณ Control-Plane
   Traffic จริงในช่วง Network Event ทำให้ Traffic ส่วนที่เกิน (`exceed`) ถูก Drop
C. ACL Sequence ผิดลำดับ
D. CoPP ไม่รองรับ OSPF/BGP โดย Design

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

Output แสดง `conformed 1850000 bps, exceed 620000 bps` — แปลว่า ACL **จับ Traffic ถูกกลุ่มแล้ว**
(ไม่ใช่ปัญหา A เพราะถ้า ACL ไม่ครอบคลุม Traffic จะไม่ขึ้นใน Class นี้เลยตั้งแต่ต้น) และ Policy
กำลังทำงานตามที่ตั้งไว้ (Police 2 Mbps) แต่ **ปริมาณ Traffic จริงในช่วง Network Event สูงกว่า Rate
ที่ตั้งไว้** ทำให้ Traffic ส่วนเกิน (620 kbps) ถูก `exceed-action` (ซึ่งตาม Config เดิมของ Part 46/
55 คือ `conform-action transmit exceed-action transmit` สำหรับ CRITICAL — แต่ถ้า Engineer
เปลี่ยนเป็น `drop` โดยไม่ได้ตั้งใจ หรือ Class Rate ตั้งไว้ต่ำเกินจริง ผลคือ Control-Plane Traffic
ที่ควร Transmit ทั้งหมดถูก Drop บางส่วน)

นี่คือกับดักคลาสสิกของ CoPP ที่ Part 55 Step 546 เตือนไว้: **CoPP Policy Update ที่ตั้ง Rate ไม่
สัมพันธ์กับ Traffic จริงของ Network ขนาดนั้น** ทำให้ Protocol ที่ควรได้รับความสำคัญสูงสุด (OSPF/
BGP อยู่ใน Class CRITICAL) กลับถูกกระทบเสียเอง — วิธีแก้คือ Baseline ปริมาณ Control-Plane
Traffic จริงช่วง Peak (เช่นด้วย NetFlow จาก Part 54) แล้วปรับ Police Rate ให้มี Headroom เพียงพอ
</details>

---

**คำถามที่ 4 — ZBFW บล็อก Traffic ที่ควรผ่าน**

โจทย์: เพิ่ม Subnet ใหม่ `10.10.35.0/24` (Server Farm ใหม่) เข้า Interface ที่เป็น Zone `INSIDE`
บน `WAN-EDGE-1` แล้ว Traffic จาก Subnet นี้ไป Internet ถูก Drop ทั้งหมด ทั้งที่ Zone-pair
`INSIDE-TO-OUTSIDE` และ Policy `INSIDE-TO-OUTSIDE-POLICY` (`inspect` Action) มีอยู่แล้วปกติ และ
ใช้งานได้กับ Subnet เดิมทุกตัว สาเหตุที่เป็นไปได้มากที่สุด?

A. Subnet ใหม่ต้องสร้าง Zone แยกของตัวเอง
B. Interface (VLAN/SVI) ของ Subnet ใหม่ยังไม่ได้ถูกกำหนดด้วย `zone-member security INSIDE` —
   Interface ที่ไม่ได้ Assign เข้า Zone จะถูกมองเป็น "ไม่มี Zone" ซึ่ง ZBFW จะ **Drop Traffic
   ทั้งหมด** ที่ผ่านจาก/เข้า Interface นั้นโดย Default (Implicit Deny ระหว่าง Zoned และ Non-Zoned)
C. Policy-map ต้องเพิ่ม Class ใหม่เฉพาะ Subnet นี้
D. NAT ยังไม่ได้ตั้งให้ Subnet นี้

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

หลักการพื้นฐานที่สุดของ ZBFW (ทวนจาก Part 45): **Interface ที่ยังไม่ได้ `zone-member security
<zone>` จะถูกมองว่าอยู่นอกระบบ Zone ทั้งหมด** — และกฎ Default ของ ZBFW คือ **Traffic ระหว่าง
Interface ที่อยู่ใน Zone กับ Interface ที่ไม่มี Zone (หรือคนละ Zone ที่ไม่มี Zone-pair) จะถูก
Deny โดย Implicit เสมอ** ต่างจาก ACL ทั่วไปที่ไม่มีแนวคิด Zone — เมื่อเพิ่ม SVI/Interface ใหม่
สำหรับ Subnet `10.10.35.0/24` เข้าไป **ต้องไม่ลืมสั่ง `zone-member security INSIDE`** บน
Interface นั้นด้วยเสมอ ไม่ใช่แค่เพิ่ม IP Address แล้วคิดว่า "อยู่ใน Network เดียวกับ INSIDE
อัตโนมัติ"

Zone-pair และ Policy ที่มีอยู่แล้วไม่ต้องแก้อะไรเลย (ตัวเลือก A, C ผิด เพราะ Zone `INSIDE` มีอยู่
แล้วและ Policy เดิมครอบคลุม Traffic Pattern เดียวกันอยู่แล้ว) และ NAT (D) เป็นคนละ Feature ที่
ทำงานอิสระจาก ZBFW — Server Farm ใหม่ต้องผ่าน ZBFW ก่อน (Drop ที่นี่) NAT จึงยังไม่มีโอกาสถูก
ตรวจสอบเลยด้วยซ้ำ
</details>

---

**คำถามที่ 5 — Unicast RPF บล็อก Asymmetric Traffic**

โจทย์: `WAN-EDGE-1` เปิด `ip verify unicast source reachable-via rx` (Strict Mode uRPF) บน
Interface ขาเข้าจาก ISP แล้วพบว่า Traffic ขากลับของ Session ที่ใช้ **PBR** (Policy-Based
Routing ส่ง Traffic ไปทาง WAN-EDGE-2 แทน แม้ Traffic ขาเข้ามาทาง WAN-EDGE-1) ถูก Drop จำนวนมาก
สาเหตุคือ?

A. uRPF Strict Mode คาดหวังว่า Return Path (ตาม Routing Table ปกติ) ต้องเป็น Interface เดียวกัน
   กับที่ Traffic เข้ามา — Asymmetric Routing (ที่ PBR หรือ Dual-homed WAN ทำให้เกิดได้ปกติ) จะ
   ถูกมองว่าเป็น Spoofed Traffic เสมอ
B. Access-list ผิด
C. uRPF ใช้ไม่ได้กับ PBR โดย Design ต้อง Disable ทั้งคู่
D. Loopback Interface ต้องเปิด uRPF ด้วยเสมอ

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: A**

**uRPF Strict Mode** ตรวจสอบว่า **Source Address ของ Packet ที่เข้ามา ต้องมี Best Route (ตาม
Routing Table ปัจจุบัน) ที่ชี้กลับออกไปทาง Interface เดียวกันกับที่ Packet เข้ามาจริง** — เป็นการ
ป้องกัน Spoofing ที่ได้ผลดีมากใน**เครือข่ายที่ Routing Symmetric** แต่จะกลายเป็นปัญหาใน
**เครือข่าย Asymmetric Routing** (Dual-homed WAN, PBR, BGP Multi-homing ที่ Traffic เข้าทางหนึ่ง
ออกอีกทาง) — เพราะ Best Path ตาม Routing Table อาจไม่ตรงกับ Interface ที่ Traffic เข้ามาจริง แม้
Traffic นั้นถูกต้องตามปกติ (ไม่ใช่ Spoofed) ก็ตาม

วิธีแก้ที่ถูกต้องตาม Blueprint (3.5): เปลี่ยนไปใช้ **uRPF Loose Mode**
(`ip verify unicast source reachable-via any`) ซึ่งเช็คแค่ว่า **มี Route ไปยัง Source นั้นอยู่ใน
Routing Table หรือไม่ (Interface ไหนก็ได้)** ไม่บังคับว่าต้องเป็น Interface เดียวกัน — เหมาะกับ
Environment ที่มี Asymmetric Routing ตามปกติของ Design (เช่น Topology หลักสูตรนี้ที่มี
WAN-EDGE-1/2 คู่กันแบบ Dual-ISP)
</details>

---

## Step 696 — Mock Exam Section D: Infrastructure Services (5 ข้อ)

**คำถามที่ 1 — DHCP Relay Missing**

โจทย์: เพิ่ม VLAN ใหม่ `VLAN 50 (IOT)` บน `DIST-SW2` พร้อม SVI `10.10.50.2/24` แล้ว Client ใน
VLAN นี้ไม่ได้รับ IP Address เลย (`DHCP Discover` ส่งออกแต่ไม่มี `Offer` กลับมา) DHCP Server อยู่ที่
`10.10.30.20` (คนละ Subnet) สาเหตุที่เป็นไปได้มากที่สุด?

A. VLAN 50 ไม่ได้เพิ่มเข้า VTP Database
B. SVI `Vlan50` ไม่มี `ip helper-address 10.10.30.20` — DHCP Discover (Broadcast) จะไม่ถูก
   Forward ข้าม Subnet ไปยัง Server โดย Default
C. DHCP Server ไม่มี Pool สำหรับ VLAN 50
D. ทั้ง B และ C ถูกต้อง เป็นไปได้ทั้งคู่ ต้องตรวจสอบทั้งสองจุด

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: D**

นี่คือคำถามแบบที่ ENARSI ชอบออก — ให้เลือก "สาเหตุที่เป็นไปได้มากที่สุด" เมื่อ**อาการเดียวกันมีได้
มากกว่า 1 Root Cause ที่เป็นไปได้จริง** และข้อสอบจริงมักให้ Diagnostic Output เพิ่มเพื่อตัดตัวเลือก
— ในกรณีนี้ทั้ง B (Router ไม่รู้จะ Forward Broadcast ไปทางไหน) และ C (Server รู้ Request แต่ไม่มี
Pool ให้ Offer) ทำให้เกิดอาการเดียวกันได้ทั้งคู่ (ไม่มี Offer กลับมา) — ขั้นตอน Troubleshoot ที่
ถูกต้องคือแยกให้ออกด้วย `debug ip dhcp server packet` ที่ Server (ถ้าไม่เห็น Discover มาถึงเลย =
ปัญหา B ที่ Router; ถ้าเห็น Discover มาถึงแต่ Server ตอบ `NAK`/ไม่ตอบเพราะไม่มี Pool = ปัญหา C)

บทเรียนสำหรับข้อสอบ: **อย่าฟันธงสาเหตุเดียวเร็วเกินไป** — ตรวจสอบ Evidence ที่มีอยู่จริงก่อนเสมอ
(`show ip interface vlan 50 | include helper` เป็นคำสั่งแรกที่เร็วที่สุดในกรณีนี้ เพราะเช็คได้จาก
Router โดยไม่ต้องรอ Debug ที่ Server)
</details>

---

**คำถามที่ 2 — NAT Port Exhaustion**

โจทย์: `WAN-EDGE-1` ใช้ `ip nat inside source list NAT-INTERNAL-ACL interface
GigabitEthernet0/0/0 overload` ผู้ใช้งานรายงานว่า Internet ใช้งานได้ปกติตอนเช้า แต่ **ช้าและ
Connection ใหม่ Timeout บ่อยตอนบ่ายที่คนใช้งานมากที่สุด** ตรวจสอบ `show ip nat translations |
count` พบจำนวน Entry สูงมากใกล้เพดาน สาเหตุที่เป็นไปได้มากที่สุด?

A. PAT Port Range (Ephemeral Port ที่ใช้ได้ต่อ 1 Global IP, ~64,000 Port) เริ่มไม่พอสำหรับจำนวน
   Session พร้อมกันของ Internal User ทั้งหมดที่ใช้ Global IP Address เดียว (Overload)
B. DNS Server ล่ม
C. Routing Table เต็ม
D. ACL `NAT-INTERNAL-ACL` Deny Traffic บางส่วน

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: A**

`overload` (PAT) แปลง Many-to-One โดยใช้ **Port Number แยกแต่ละ Session** ภายใต้ Global IP
Address เดียว — ทฤษฎีสูงสุดคือประมาณ 65,536 Port ต่อ Global IP แต่ในทางปฏิบัติ IOS จะเริ่มมีปัญหา
Performance ก่อนถึงเพดานจริงมาก (เพราะ Session ที่ยังไม่ถูก Age Out ทัน + TCP/UDP Timeout Default
ค่อนข้างนาน) — อาการ "ช้าตอนที่คนใช้งานมากที่สุด" ตรงกับรูปแบบ **Port/Translation Exhaustion**
แบบคลาสสิก: Session เก่าที่ค้างอยู่ (ยังไม่ Timeout) กิน Port ไปเรื่อยๆ จนไม่มี Port เหลือให้
Session ใหม่ Allocate ได้ทันเวลา ทำให้ดูเหมือน Timeout/ช้า

วิธีแก้ที่ ENARSI คาดหวัง (Part 63): เพิ่ม **Global IP Address มากกว่า 1 ตัวในการทำ Overload**
(`ip nat pool` ที่มีหลาย IP + `overload`) เพื่อขยาย Port Pool ทั้งหมดให้มากกว่า 1 เท่าของ 65,536,
หรือปรับ `ip nat translation timeout`/`tcp-timeout` ให้ Age Out Session ที่ไม่ใช้งานแล้วเร็วขึ้น
เพื่อคืน Port ให้ระบบเร็วกว่าเดิม
</details>

---

**คำถามที่ 3 — QoS Policy ผิด Direction**

โจทย์: `DIST-SW1` Apply `service-policy input DIST-QOS-POLICY` บน Port-channel1 (Uplink ไป Core)
โดย Policy มี `class VOICE / priority percent 15` (LLQ) แต่ Voice Quality ยังแย่ตอน Congestion
สาเหตุที่เป็นไปได้มากที่สุด?

A. LLQ (`priority`) ใช้ได้กับ `input` Direction เท่านั้น
B. Queuing Feature (LLQ/CBWFQ/Shaping) ทำงานได้เฉพาะ **ขาออก (Output/Egress)** เท่านั้น — Apply
   เป็น `input` ทำให้ Policing ทำงาน (ถ้ามี) แต่ Queuing/Priority ไม่มีผลใดๆเลยกับ Congestion
   ที่เกิดขึ้นตอนส่งออกจาก Interface
C. Class-map `VOICE` Match DSCP ผิด Value
D. Bandwidth ของ Policy ตั้งน้อยเกินไป

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

หลักการพื้นฐานที่สุดของ QoS (ทวนจาก Part 24/40): **Queuing (CBWFQ, LLQ/Priority Queue,
Shaping) แก้ปัญหา Congestion ที่เกิด "ตอนส่ง Packet ออกจาก Interface" เท่านั้น** — เพราะ
Congestion (Buffer เต็ม, ต้องเลือกว่า Packet ไหนออกก่อน) เกิดขึ้นที่ **Output Queue ของ
Interface** ไม่ใช่ตอนรับเข้ามา ดังนั้น Feature กลุ่มนี้ **ใช้ได้กับ `service-policy output`
เท่านั้น** — ถ้า Apply เป็น `input` (ใช้ได้กับ Policing/Marking/Classification เท่านั้น) Priority
Queue ที่ตั้งไว้จะ **ไม่มีผลอะไรกับการจัดลำดับ Traffic ขาออกเลย** เสียง Voice จึงยังแย่เหมือนเดิม
ตอน Uplink Congest

นี่คือกับดักที่ Part 55 Step 543 เตือนไว้ตรงๆว่า "Apply (`service-policy`) ที่ Interface **ขาออก**
เสมอสำหรับ Queuing/Shaping" — ตัวเลือก A ผิดตรงข้ามกับความจริง (LLQ ใช้ได้กับ Output เท่านั้น
ไม่ใช่ Input), C, D อาจเป็นปัญหาเสริมได้แต่ไม่ตรงกับอาการ "Voice แย่ทุกครั้งที่ Congest" แบบตายตัว
เท่ากับ Direction ผิด
</details>

---

**คำถามที่ 4 — Multicast RP ไม่ Reachable**

โจทย์: `DIST-SW3` (Receiver ของ Multicast Group `239.1.1.1`) ไม่ได้รับ Multicast Traffic เลย
ตรวจสอบ `show ip pim rp mapping` พบว่า RP คือ `1.1.1.100` (Anycast RP) ปกติ แต่
`show ip route 1.1.1.100` แสดง **ไม่มี Route ไปยัง RP Address นี้เลย** สาเหตุที่เป็นไปได้มากที่สุด?

A. IGMP Version ไม่ตรงกันระหว่าง Host กับ Switch
B. `ip pim rp-address 1.1.1.100` ตั้งไว้ แต่ Anycast RP Address (`1.1.1.100/32`) ไม่ได้ถูก
   Advertise เข้า Routing Protocol (เช่น ไม่มีบน Loopback ที่ Redistribute หรือไม่มี `network`
   Statement ครอบคลุม) ทำให้ไม่มี Router ตัวใดรู้จัก Path ไปยัง RP เลย
C. MSDP Peer ระหว่าง CORE-SW1/CORE-SW2 ไม่ขึ้น
D. PIM Mode ตั้งเป็น Dense Mode ผิดที่ Interface บางจุด

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

Static RP (`ip pim rp-address`) **ไม่ได้สร้าง Route ไปยัง RP ให้อัตโนมัติ** — มันแค่ "บอก" ทุก
Router ว่า RP อยู่ที่ IP ไหน แต่ **การไปถึง RP Address นั้นจริงยังต้องพึ่ง Unicast Routing
(OSPF/EIGRP/BGP) ตามปกติทุกประการ** เหมือน Destination ทั่วไป — ถ้า RP Address (มักเป็น Loopback
ที่ทำ Anycast) ไม่ได้ถูก Advertise เข้า Routing Protocol อย่างถูกต้อง (ลืม `network` Statement,
ลืม Redistribute, หรือ ACL/Filter บล็อกไว้) Router ที่ต้องส่ง PIM Join ไปยัง RP จะ **ไม่มี Path
ไปถึงเลย** — ผลคือ (*,G) Join ไม่สามารถสร้าง Shared Tree ไปยัง RP ได้ Multicast Traffic จึงไม่มี
ทางไหลลงมาถึง Receiver

ตัวเลือก A จะทำให้ Host ไม่ Join Group เลยตั้งแต่ระดับ Local Segment (คนละปัญหา), C เป็นปัญหาของ
**Source ที่อยู่ RP ตัวอื่น** (Anycast RP ข้าม Domain ผ่าน MSDP) ไม่เกี่ยวกับ Receiver ไปหา RP
ตัวเองไม่ได้, D ทำให้พฤติกรรมต่างออกไปแต่ไม่ตรงกับอาการ "ไม่มี Route ไปยัง RP" ที่ระบุตรงๆในโจทย์
— ต้องอ่าน Evidence ที่ให้มา (`show ip route`) ให้ตรงกับ Layer ที่ถูกถามเสมอ
</details>

---

**คำถามที่ 5 — SNMPv3 Authentication ล้มเหลว**

โจทย์: `NOC-SRV` Poll `CORE-SW1` ผ่าน SNMPv3 แล้วได้ Error `authenticationFailure` ตรวจสอบ
Config ทั้งสองฝั่งพบว่า **Username และ Group ตรงกัน** แต่ Poll ยังล้มเหลว สาเหตุที่เป็นไปได้มาก
ที่สุด?

A. SNMP Community String ผิด
B. Auth Password (และ/หรือ Priv Password ถ้าใช้ `priv`) ที่ตั้งไว้บน `CORE-SW1`
   (`snmp-server user ... auth sha <password> priv aes 128 <password>`) ไม่ตรงกับที่ NOC-SRV
   ใช้ Poll — SNMPv3 ไม่ใช้ Community String แต่ใช้ User-based Security Model (USM) ที่ต้อง
   Auth/Priv Key ตรงกันแทน
C. SNMP Version ที่ NOC-SRV ใช้จริงคือ v2c ไม่ใช่ v3
D. ACL บล็อก UDP/161

<details>
<summary>เฉลยและคำอธิบาย</summary>

**คำตอบที่ถูก: B**

โจทย์ระบุชัดว่า Error เป็น `authenticationFailure` (Error Code เฉพาะของ SNMPv3 USM) **และ**
Username/Group ตรงกันแล้ว — ตัวแปรที่เหลือของ SNMPv3 USM ที่ต้องตรงกันคือ **Auth
Password/Algorithm (MD5/SHA) และ Priv Password/Algorithm (DES/AES) ที่ใช้สร้าง Key จริง** ซึ่ง
Config ทั้งสองฝั่งต้องตรงกันทุกตัวอักษร (Password ผิดแม้ 1 ตัวอักษร = Key ที่คำนวณได้ต่างกันสิ้นเชิง
เพราะเป็น Hash-based Key Derivation)

ตัวเลือก A ผิด เพราะ Community String เป็นแนวคิดของ SNMPv1/v2c เท่านั้น ไม่มีใน v3 เลย (ข้อสอบชอบ
ใส่ตัวเลือกที่ผสมแนวคิดระหว่าง Version มาให้สับสน), C ขัดกับ Error Code ที่ให้มา (v2c ที่ Community
ผิดจะได้ Error คนละแบบ ไม่ใช่ `authenticationFailure`), D จะทำให้ **ไม่มี Response กลับมาเลย
(Timeout)** ไม่ใช่ Error `authenticationFailure` ที่ต้องได้รับ Response กลับมาก่อนถึงจะรู้ว่า Auth
ล้มเหลว — การอ่าน Error Code ให้ตรงกับ Layer ที่มันบอกคือทักษะสำคัญของข้อ 4.2 ใน Blueprint
</details>

---

## Step 697 — Full Timed Lab Simulation (90 นาที)

### โจทย์: "The Morning After" — เครือข่ายพังหลายจุดพร้อมกันหลัง Maintenance Window

> **สถานการณ์**: เมื่อคืนมี Maintenance Window ที่ทีมอื่นเข้าไปทำงานกับหลายอุปกรณ์พร้อมกัน (Upgrade
> Firmware, ปรับ ACL, เพิ่ม VLAN ใหม่) เช้านี้ NOC รายงานปัญหาหลายจุดพร้อมกัน — คุณมีเวลา **90
> นาที** ในการวินิจฉัยและแก้ไขให้ครบทุก Ticket (มี Weight คะแนนต่างกันตามความซับซ้อน)

| Ticket | อาการที่รายงาน | Domain | Point | เวลาแนะนำ |
|---|---|---|---|---|
| 1 | `DIST-SW2` ↔ `CORE-SW1` OSPF Neighbor หลุด (Area 1) | Layer 3 | 20 | 12 นาที |
| 2 | EIGRP-DEMO Redistribution เกิด Suboptimal Route กลับไปยัง `EIGRP-R1` | Layer 3 | 15 | 10 นาที |
| 3 | `BRANCH-RTR` (DMVPN Spoke) Register กับ Hub ไม่สำเร็จตั้งแต่เช้า | VPN | 20 | 15 นาที |
| 4 | Server Farm ใหม่ VLAN 35 คุย Internet ไม่ได้ผ่าน ZBFW | Security | 15 | 10 นาที |
| 5 | VLAN 50 (IOT ใหม่) ไม่มี Client ได้ IP เลย | Services | 10 | 8 นาที |
| 6 | `CORE-SW1` Drop OSPF/BGP Packet เป็นระยะช่วง Traffic สูง | Security | 20 | 15 นาที |
| — | **สำรองเวลาตรวจทาน + แก้ Ticket ที่ค้าง** | — | — | **20 นาที** |

> รวม 90 นาที (6 Ticket ~70 นาที + สำรอง 20 นาที) — ลำดับการทำ**ไม่จำเป็นต้องตามเลข Ticket**
> ควรเริ่มจาก Ticket ที่กระทบ Layer ต่ำสุดก่อนเสมอ (Ticket 1 ควรทำก่อน เพราะ Ticket 3, 6 อาจได้รับ
> ผลกระทบจาก Underlay Routing ที่ยังไม่เสถียร)

---

### Model Answer — Walkthrough แบบเต็มทุก Ticket

**Ticket 1 — OSPF Neighbor หลุด (Area 1)**

```
DIST-SW2# show ip ospf neighbor
                                            <- ไม่เห็น CORE-SW1 (1.1.1.1) เลย

DIST-SW2# show ip ospf interface Port-channel1 | include Hello|Dead
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
CORE-SW1# show ip ospf interface Port-channel11 | include Hello|Dead
  Timer intervals configured, Hello 5, Dead 20, Wait 20, Retransmit 5
```

**Root Cause**: ทีม Maintenance แก้ Hello/Dead Timer ที่ `CORE-SW1` ไม่ตรงกับ `DIST-SW2` อีกต่อไป
(Timer Mismatch เป็นหนึ่งในเงื่อนไขที่ Adjacency ต้องตรงกันเหมือน Area/Auth/MTU)

**Fix**:
```
CORE-SW1(config)# interface Port-channel11
CORE-SW1(config-if)# ip ospf hello-interval 10
CORE-SW1(config-if)# ip ospf dead-interval 40
```
Verify: `show ip ospf neighbor` เห็น `1.1.1.12` State `FULL` กลับมาภายใน ~15 วินาที

---

**Ticket 2 — EIGRP Redistribution Suboptimal Route**

```
DIST-SW3# show route-map OSPF-TO-EIGRP
route-map OSPF-TO-EIGRP, permit, sequence 10
  Match clauses: (none)
  Set clauses: tag 110
```

**Root Cause**: เหมือนคำถามที่ 4 ของ Step 693 เป๊ะ — ไม่มี `deny` Sequence กัน Route ที่มี `tag 90`
(มาจาก EIGRP อยู่แล้ว) ไม่ให้วนกลับเข้า EIGRP อีกครั้ง

**Fix**:
```
DIST-SW3(config)# route-map OSPF-TO-EIGRP deny 5
DIST-SW3(config-route-map)# match tag 90
DIST-SW3(config-route-map)# exit
```
Verify: `show ip eigrp topology` ที่ `EIGRP-R1` ไม่เห็น Route ที่ Advertise ผ่าน Path ผิดอีกต่อไป

---

**Ticket 3 — DMVPN Spoke Register ไม่สำเร็จ**

```
BRANCH-RTR# show dmvpn
% No entries found       <- ไม่มี Peer เลยแม้แต่ Hub

BRANCH-RTR# show crypto isakmp sa
                                            <- ว่างเปล่า
```

**Root Cause**: IKE Phase 1 ไม่ขึ้นเลย (เหมือนคำถามที่ 2 ของ Step 694) — ตรวจ
`show run | section crypto isakmp policy` พบว่าทีม Maintenance เปลี่ยน Hash จาก `sha` เป็น
`sha256` ที่ `WAN-EDGE-1` แต่ `BRANCH-RTR` ยังเป็น `sha` เดิม

**Fix**:
```
WAN-EDGE-1(config)# crypto isakmp policy 10
WAN-EDGE-1(config-isakmp)# hash sha
```
(หรือกลับเป็นค่าเดิมให้ตรงกันทั้งสองฝั่ง — เลือกทางที่กระทบ Peer อื่นน้อยที่สุด เพราะ Policy นี้ใช้
กับ Spoke อื่นด้วย) Verify: `show crypto isakmp sa` เห็น `QM_IDLE`, `show dmvpn` เห็น Hub State
`UP`

---

**Ticket 4 — ZBFW บล็อก VLAN 35 ใหม่**

```
WAN-EDGE-1# show zone-pair security
                                            <- INSIDE-TO-OUTSIDE มีอยู่ปกติ
CORE-SW1# show run interface vlan 35
interface Vlan35
 ip address 10.10.35.2 255.255.255.0
                                            <- ไม่มี zone-member เลย (แต่ ZBFW อยู่ที่ WAN-EDGE ไม่ใช่ CORE)
```

**Root Cause**: เหมือนคำถามที่ 4 ของ Step 695 — Interface ที่เกี่ยวข้องกับ VLAN 35 บน
`WAN-EDGE-1` (ฝั่งที่ Traffic ผ่านจริงก่อนออก Internet) ไม่ได้ `zone-member security INSIDE`

**Fix**:
```
WAN-EDGE-1(config)# interface TenGigabitEthernet0/0/1.35
WAN-EDGE-1(config-subif)# zone-member security INSIDE
```
Verify: `ping 8.8.8.8` จาก Server ใน VLAN 35 ผ่านได้ปกติ, `show policy-map type inspect zone-pair
INSIDE-TO-OUTSIDE` เห็น Session Count เพิ่มขึ้น

---

**Ticket 5 — VLAN 50 ไม่มี Client ได้ IP**

```
DIST-SW2# show ip interface vlan 50 | include helper
                                            <- ไม่มีบรรทัด helper-address เลย
```

**Root Cause**: เหมือนคำถามที่ 1 ของ Step 696 — ไม่มี `ip helper-address` (สาเหตุ B ในคำตอบเดิม
ยืนยันตรงนี้ ตัด C ออกได้เพราะ Server มี Pool `IOT-DEVICES` เตรียมไว้แล้วล่วงหน้าตามข้อมูลจริงของ
Ticket นี้)

**Fix**:
```
DIST-SW2(config)# interface vlan 50
DIST-SW2(config-if)# ip helper-address 10.10.30.20
```
Verify: `show ip dhcp binding` ที่ Server เห็น Lease ใหม่ของ Client ใน `10.10.50.0/24`

---

**Ticket 6 — CoPP Drop OSPF/BGP**

```
CORE-SW1# show policy-map control-plane class COPP-CRITICAL
  police:
      1000000 bps, 15625 limit, 15625 extended limit
    conformed 900000 bps, exceed 450000 bps, drop 450000 bps
```

**Root Cause**: เหมือนคำถามที่ 3 ของ Step 695 — ทีม Maintenance ปรับ Rate ของ `COPP-CRITICAL`
ลงจาก 2 Mbps เดิมเป็น 1 Mbps โดยไม่ได้ตั้งใจ (หรือลืมคืนค่าหลัง Test) ทำให้ Traffic ส่วนเกินถูก
Drop จริงตอน Traffic สูง

**Fix**:
```
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)# class COPP-CRITICAL
CORE-SW1(config-pmap-c)# police 8000000 conform-action transmit exceed-action transmit
```
(ปรับ Rate ให้มี Headroom สูงกว่าปริมาณ Control-Plane Traffic จริงตาม Baseline ที่วัดได้ ไม่ใช่
คืนเป็น 2 Mbps เดิมเฉยๆ เพราะ Network ปัจจุบันโตขึ้นกว่าตอน Part 46/55 แล้ว) Verify: OSPF/BGP
Neighbor เสถียรระหว่าง Generate Traffic สูงจำลอง Event เดิม

> **บทเรียนของ Simulation นี้**: สังเกตว่า **4 จาก 6 Ticket มี Root Cause ที่ตรงกับคำถาม Mock
> Exam Section A-D ทุกตัวอักษร** — นี่คือรูปแบบการฝึกที่ถูกต้องที่สุดสำหรับ ENARSI: ทำ Mock Exam
> ให้แม่นจนจำ "ลายเซ็นของ Root Cause" แต่ละแบบได้ แล้วนำมาประยุกต์กับ Ticket ที่ Wording ต่างออกไป
> — ข้อสอบจริงไม่ถามซ้ำคำเดิม แต่ Pattern ของ Root Cause ซ้ำกันเสมอ
</details>

---

## Step 698 — เคล็ดลับวันสอบ ENARSI: Ticket-based Lab Format

ENARSI 300-410 (เหมือน CCNP อื่นๆ) มี Lab Section แบบ **Ticket-based Troubleshooting** เป็นส่วนที่
กินเวลามากที่สุดและตัดสินผลสอบมากที่สุด — นี่คือเคล็ดลับเฉพาะสำหรับ Format นี้ (ต่างจาก
Multiple-Choice ทั่วไป):

### 698.1 จัดสรรเวลาต่อ Ticket แบบ Weighted ไม่ใช่ Equal Split

| หลักการ | เหตุผล |
|---|---|
| **อ่าน Point Value ของทุก Ticket ก่อนเริ่มแม้แต่ Ticket แรก** | Ticket ที่ Point สูงไม่ได้แปลว่าซับซ้อนกว่าเสมอไป — บาง Ticket ง่ายแต่ Point สูงเพราะกระทบ Business Impact มาก ควรทำก่อนถ้าเจอ Root Cause เร็ว |
| **ตั้ง "เพดานเวลา" ต่อ Ticket ล่วงหน้า (เช่น 12-15 นาที)** | ถ้าเกินเพดานแล้วยังหา Root Cause ไม่เจอ ให้ **Flag แล้วข้ามไป Ticket ถัดไปทันที** — อย่าฝืนต่อ |
| **เผื่อเวลา 15-20% ท้ายสุดเสมอ** | สำหรับกลับมาทำ Ticket ที่ Flag ไว้ — สมองที่ผ่านการแก้ Ticket อื่นมาก่อนมักเห็น Pattern ที่พลาดไปตอนแรกได้ |

### 698.2 กัน Tunnel Vision — วิธีสังเกตตัวเองว่ากำลังติดกับดัก

- **สัญญาณเตือน**: แก้ Config เดิมซ้ำมากกว่า 2 ครั้งแล้วอาการไม่เปลี่ยน, หรือกำลัง Debug ที่อุปกรณ์
  เดียวกันนานเกิน 5 นาทีโดยไม่ขยับไปดูอุปกรณ์ข้างเคียง
- **วิธีแก้**: กลับไปที่ **Layer ต่ำกว่า** ที่เคยข้ามไปเสมอ (เช่น กำลังไล่ BGP Attribute อยู่นาน
  ให้ถอยกลับไปเช็ค `show ip route` ธรรมดาก่อนว่า Underlay ไป Peer ได้จริงหรือไม่) — ทวนหลักการ
  Layer-by-Layer จาก Part 1/25 Step 245 เสมอเมื่อรู้สึกว่าเริ่มหลงทาง
- **กฎ "2-Command Rule"**: ถ้า Show Command 2 คำสั่งแรกที่คิดออกไม่นำไปสู่เบาะแสอะไรใหม่เลย
  ให้เปลี่ยนสมมติฐาน Root Cause ทันที อย่าไล่ Command เดิมซ้ำๆ

### 698.3 Show Command Cheat-Sheet ที่ต้องนึกออกภายใน 5 วินาที (ไม่ต้องเปิด Note)

| อาการเริ่มต้น | คำสั่งแรกที่ควรนึกถึง |
|---|---|
| Neighbor ไม่ขึ้น (Routing Protocol ใดก็ได้) | `show <protocol> neighbor`, `show run interface <link>` เทียบ Timer/Auth/Area ทั้งสองฝั่ง |
| Route หายไปจาก Table | `show ip route <prefix>` → ถ้าไม่มีเลย ไล่ `show ip protocols`, `show route-map` |
| Traffic บาง Protocol ไม่ผ่าน แต่บาง Protocol ผ่าน | `show access-lists` ดู Match Counter ก่อนเสมอ (ลายเซ็น ACL/ZBFW) |
| VPN/Tunnel ไม่ขึ้น | `show crypto isakmp sa` → `show crypto ipsec sa` → `show dmvpn` ไล่ตามลำดับ Layer |
| Client ไม่ได้ IP | `show ip interface <vlan> \| include helper` ก่อนไปดู Server เสมอ |
| Control-Plane Drop | `show policy-map control-plane class <name>` ดู `drop`/`exceed` Counter |

### 698.4 บันทึกทุกครั้งที่แก้ Config

Ticket-based Lab มักให้คะแนนแยกทีละ Ticket — **`copy running-config startup-config` (หรือ
Save ตามที่ระบบสอบกำหนด) ทุกครั้งหลังแก้ Ticket เสร็จแต่ละอัน** ไม่ใช่รอทำครบทุก Ticket แล้ว Save
รวดเดียว เพราะถ้าระบบ Crash หรือหมดเวลากลางที่ยังไม่ Save Ticket ที่แก้ไปแล้วอาจไม่ถูกนับ

### 698.5 อย่าลืม "Baseline ก่อนแก้" แม้ในห้องสอบ

ก่อนแก้ Config ใดๆ ให้จด (ในกระดาษที่สอบให้ หรือ Comment ใน Note) **Baseline สถานะปัจจุบัน**
สั้นๆ (เช่น "OSPF Neighbor 1.1.1.1 = ไม่ขึ้น, Timer Hello 5/Dead 20") — เพื่อให้ตรวจสอบได้ชัดเจนว่า
การแก้ไขที่ทำ **แก้ปัญหาได้จริง ไม่ใช่แค่เปลี่ยนอาการ** ก่อนย้ายไป Ticket ถัดไป

---

## Step 699 — Career Context: CCNP เต็มใบปลดล็อกอะไรได้บ้าง

### 699.1 ตำแหน่งงานที่ CCNP Enterprise (ENCOR + ENARSI) ครบใบเปิดทางได้จริง

| ระดับตำแหน่ง | บทบาทที่คาดหวัง | ทำไม CCNP เต็มใบถึงตรงกับตำแหน่งนี้ |
|---|---|---|
| Network Engineer II / Senior Network Engineer | ดูแล Enterprise Network ระดับ Production เต็มรูปแบบ, Design เบื้องต้น, Troubleshoot ระดับสูง | ENCOR ให้ Design/Architecture Awareness, ENARSI ให้ทักษะ Troubleshoot ที่ต้องใช้จริงหน้างานทุกวัน |
| Network Operations (NOC) Team Lead | ควบคุมทีม NOC, ตัดสินใจ Escalation Path, เขียน Runbook | Ticket-based Troubleshooting ที่ฝึกมาตลอด Part 56-70 ตรงกับงาน Incident Response โดยตรง |
| Presales Systems Engineer (SE) | ออกแบบ Solution ให้ลูกค้าองค์กรขนาดกลาง-ใหญ่ | ต้องเข้าใจทั้ง Design (ENCOR) และรู้ว่า Feature ไหน "พังง่าย" ในสนามจริง (ENARSI) เพื่อออกแบบให้แข็งแรงตั้งแต่ต้น |
| Network Consultant (Independent/MSP) | รับงาน Implement/Troubleshoot ให้หลายองค์กร | CCNP เต็มใบเป็น Credential มาตรฐานขั้นต่ำที่ลูกค้าองค์กรใหญ่คาดหวังเห็นในโปรไฟล์ |

### 699.2 CCNP เต็มใบ = จุดเริ่มต้นที่แข็งแรงที่สุดสำหรับ CCIE Enterprise Infrastructure

Cisco ออกแบบ CCIE Lab Exam ให้ **สมมติว่าผู้เข้าสอบมีความรู้ระดับ CCNP Enterprise (ทั้ง ENCOR และ
ENARSI) เป็นพื้นฐานอยู่แล้ว 100%** — ไม่มีการทวนพื้นฐานใน CCIE Lab เลย โจทย์ CCIE Lab จะเข้าสู่
Complexity ระดับ "Design + Deploy + Optimize" ทันทีโดยไม่อธิบาย Concept พื้นฐานซ้ำ

> **ข่าวดีที่สำคัญ**: การผ่าน **CCIE Lab Exam** (ไม่ว่า Track ใด) จะทำให้ได้ **CCNP ในสายเดียวกัน
> โดยอัตโนมัติ** (Cisco Recertification Policy) — แปลว่าเส้นทางที่หลักสูตรนี้ออกแบบไว้ (CCNA →
> ENCOR+ENARSI → CCIE Lab) เป็นเส้นทางที่ **Investment ทุกใบไม่มีทางเสียเปล่า** ไม่ว่าจะหยุดที่
> CCNP หรือไปต่อถึง CCIE

### 699.3 Timeline แนะนำสำหรับก้าวสู่ CCIE Enterprise Infrastructure (Part 71 เป็นต้นไป)

| ช่วงเวลา (หลังผ่าน ENARSI) | สิ่งที่ควรทำ |
|---|---|
| เดือนที่ 1-2 | สอบ **Cisco 300-410 ENARSI** จริงให้ผ่านก่อน (ถ้ายังไม่สอบ) — อย่าลากยาวความรู้สึก "พร้อมสอบ" ให้นานเกินไป ยิ่งรอยิ่งลืม |
| เดือนที่ 2-4 | เริ่ม Part 71-83 (CCIE EI Written Exam Domain: Design/Deploy/Optimize) — เนื้อหาเชิง Design ที่ลึกกว่า ENCOR มาก |
| เดือนที่ 4-8 | สอบ **CCIE Enterprise Infrastructure Written Exam** ให้ผ่าน (อายุ Pass Written = 18 เดือนก่อนต้องสอบ Lab) |
| เดือนที่ 5-14 (คู่ขนานกับ Written) | Part 84-98 (VXLAN/EVPN, MPLS L3VPN, Full-Stack Automation, CCIE Full Lab Scenario x4) — ฝึก Lab จริงบน EVE-NG/CML อย่างต่อเนื่อง สัปดาห์ละหลายชั่วโมง |
| เดือนที่ 12-18 | จองสอบ **CCIE Lab Exam** (8 ชั่วโมง) — ควรฝึก Full Mock Lab จับเวลาจริงอย่างน้อย 5-10 รอบก่อนจอง |

> **คำแนะนำที่ตรงไปตรงมาที่สุด**: อย่าข้าม Part 71-98 ไปตรงเข้า CCIE Lab Mock ทันทีเพราะ "รู้สึกว่า
> เก่งแล้ว" — CCIE Lab วัดความลึกที่ ENARSI ไม่แตะเลย เช่น VXLAN/EVPN Control-Plane, MPLS L3VPN
> PE-CE Design เต็มรูปแบบ, และ Automation ระดับ CI/CD จริง — ความมั่นใจจาก CCNP เต็มใบคือ
> **จุดเริ่มต้นที่ดี ไม่ใช่จุดจบของการเตรียมตัว**

---

## Step 700 — Final Running-Config Appendix: CORE-SW1 ฉบับสมบูรณ์ที่สุดของทั้งหลักสูตร CCNP

นี่คือ **Running-Config ฉบับ "Graduate-Level"** ของ `CORE-SW1` — ต่อยอดโดยตรงจาก Config ฉบับ
ENCOR ใน [Part 55 Step 550](part-055-encor-capstone-lab.md#step-550) โดยเพิ่ม **Refinement ระดับ
ENARSI** ที่ได้เรียนรู้จาก Part 56-69 เข้าไปทับ (Fast Convergence Tuning, CoPP Rate ที่ปรับตาม
Baseline จริง, uRPF, IPv6 First-Hop Security, Logging/SNMP ที่แข็งแรงขึ้น) — คือเอกสารอ้างอิงที่
สมบูรณ์ที่สุดของทั้ง **หลักสูตร CCNP (Part 1-70) รวมกัน**

> **หมายเหตุ Scope**: เช่นเดียวกับ Part 25/55, Feature ที่เป็นของอุปกรณ์อื่นโดยเฉพาะ (EIGRP-DEMO,
> DMVPN/IPsec/ZBFW/NAT ที่ WAN-EDGE-1/2, 802.1X/TrustSec ที่ ACCESS-SW, DHCP Pool ที่ DIST-SW2)
> **ไม่อยู่ใน Config ชุดนี้** — ดู Step 692 ตารางที่ 1-2 สำหรับ Mapping ว่า Feature ไหนอยู่อุปกรณ์ใด
> Config ชุดนี้แสดงเฉพาะสิ่งที่ตรงกับบทบาทจริงของ Core Switch ตลอดทั้งหลักสูตร

### CORE-SW1 (Catalyst 9500, StackWise Virtual Pair — Full Curriculum Reference Config)

```
hostname CORE-SW1
!
no ip domain-lookup
ip domain-name lab.local
enable secret Cisco123!
service password-encryption
!
! ===== Part 22/61: AAA (พื้นฐาน + Lockout-safe Fallback ที่ทวนซ้ำใน Part 61 Step 606) =====
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
!
username admin privilege 15 secret StrongP@ssw0rd!
tacacs server AAA-SRV
 address ipv4 10.10.99.51
 key CCNA-LAB-KEY
!
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
! ===== Part 34: StackWise Virtual (SVL Domain 100) =====
stackwise-virtual
 domain 100
!
ip routing
ip multicast-routing
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
!
spanning-tree mode rapid-pvst
!
interface Loopback0
 description ** Router-ID / Anycast RP Source / Fabric Border+CP Node **
 ip address 1.1.1.1 255.255.255.255
 ip pim sparse-mode
!
! ===== Part 34: SVL Link + Dual-Active Detection =====
interface TenGigabitEthernet1/0/1
 description ** SVL Link 1 to CORE-SW2 **
 stackwise-virtual link 1
 no shutdown
!
interface TenGigabitEthernet1/0/2
 description ** SVL Link 2 to CORE-SW2 (redundant) **
 stackwise-virtual link 1
 no shutdown
!
interface TenGigabitEthernet1/0/3
 description ** Dual-Active Detection Link **
 stackwise-virtual dual-active-detection
 no shutdown
!
! ===== Part 26/57: OSPF Backbone (Area 0) + BFD (Fast Convergence, ตรวจ Timer ให้ตรงทุกฝั่ง — Step 697 Ticket 1) =====
interface Port-channel1
 description ** Routed backbone link to CORE-SW2 (Area 0) **
 no switchport
 ip address 10.255.0.1 255.255.255.252
 ip ospf 1 area 0
 ip ospf message-digest-key 1 md5 Ar3aZeroKey!
 ip ospf hello-interval 10
 ip ospf dead-interval 40
 bfd interval 150 min_rx 150 multiplier 3
 ip ospf bfd
 ip pim sparse-mode
 ip verify unicast source reachable-via any
 no shutdown
!
! ===== Part 34: MEC ไปยัง DIST-SW1 (Dual-homed, Area 1) =====
interface Port-channel11
 description ** MEC to DIST-SW1 (Dual-homed, spans SVL Pair) - Area 1 **
 no switchport
 ip address 10.255.10.1 255.255.255.252
 ip ospf 1 area 1
 ip ospf message-digest-key 1 md5 Area1Key!
 ip ospf network point-to-point
 ip ospf hello-interval 10
 ip ospf dead-interval 40
 bfd interval 150 min_rx 150 multiplier 3
 ip ospf bfd
 ip pim sparse-mode
 ip verify unicast source reachable-via any
 service-policy output CORE-QOS-POLICY
 no shutdown
!
interface TenGigabitEthernet1/1/1
 description ** MEC member 1 - to DIST-SW1 **
 no switchport
 channel-group 11 mode active
 no shutdown
!
interface TenGigabitEthernet1/1/2
 description ** MEC member 2 - to DIST-SW1 (ผ่าน CORE-SW2 Chassis ทาง SVL) **
 no switchport
 channel-group 11 mode active
 no shutdown
!
! ===== Part 27: OSPFv3 (IPv6) - Per-link, Manual Router-ID =====
ipv6 unicast-routing
ipv6 router ospf 1
 router-id 1.1.1.1
!
interface Port-channel1
 ipv6 address fd00:255:0::1/64
 ipv6 ospf 1 area 0
!
interface Port-channel11
 ipv6 address fd00:255:10::1/64
 ipv6 ospf 1 area 1
!
! ===== Part 61 (Step 61x): IPv6 First-Hop Security บน Interface ที่มี Host ปลายทาง (แนวคิดคู่กับ ipv6 nd) =====
ipv6 nd raguard policy CORE-RAGUARD
 device-role router
!
! ===== Part 40/68: MQC QoS (3-Class: VOICE/VIDEO/CRITICAL-DATA) - ตรวจ Direction เสมอ (Step 696 คำถามที่ 3) =====
class-map match-any VOICE
 match dscp ef
class-map match-any VIDEO
 match dscp af41
class-map match-any CRITICAL-DATA
 match dscp af31
!
policy-map CORE-QOS-POLICY
 class VOICE
  priority percent 15
 class VIDEO
  bandwidth remaining percent 25
 class CRITICAL-DATA
  bandwidth remaining percent 30
 class class-default
  fair-queue
!
mls qos
mls qos trust dscp
!
! ===== Part 42/65: Multicast Advanced - Anycast RP + MSDP (RP ต้อง Reachable ใน Routing เสมอ — Step 696 คำถามที่ 4) =====
ip pim rp-address 1.1.1.100 10
ip msdp peer 1.1.1.2 connect-source Loopback0
ip msdp originator-id Loopback0
!
! ===== Part 30/58: BGP iBGP AS65001 =====
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 update-source Loopback0
 neighbor 1.1.1.21 remote-as 65001
 neighbor 1.1.1.21 update-source Loopback0
 neighbor 1.1.1.22 remote-as 65001
 neighbor 1.1.1.22 update-source Loopback0
 !
 address-family ipv4 unicast
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.21 activate
  neighbor 1.1.1.22 activate
 exit-address-family
!
! ===== Part 26/57/59: OSPF Advanced + Redistribution ปลอดภัย (ไม่มี "always" ผูกกับ BGP Dynamic - Part 55 Step 544) =====
router ospf 1
 router-id 1.1.1.1
 area 0 authentication message-digest
 area 1 authentication message-digest
 area 1 range 10.10.0.0 255.255.0.0
 area 2 nssa
 passive-interface default
 no passive-interface Port-channel1
 no passive-interface Port-channel11
 default-information originate metric 1 metric-type 1
!
! ===== Part 46/61: Control Plane Policing (Rate ปรับตาม Baseline จริง - Step 697 Ticket 6) =====
class-map match-any COPP-CRITICAL
 match access-group name COPP-ROUTING-ACL
class-map match-any COPP-IMPORTANT
 match access-group name COPP-MGMT-ACL
class-map match-any COPP-NORMAL
 match access-group name COPP-ICMP-ACL
!
policy-map COPP-POLICY
 class COPP-CRITICAL
  police 8000000 conform-action transmit exceed-action transmit
 class COPP-IMPORTANT
  police 1000000 conform-action transmit exceed-action drop
 class COPP-NORMAL
  police 500000 conform-action transmit exceed-action drop
 class class-default
  police 250000 conform-action transmit exceed-action drop
!
ip access-list extended COPP-ROUTING-ACL
 permit ospf any any
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit pim any any
 permit udp any any eq 3784
 permit udp any any eq 3785
!
ip access-list extended COPP-MGMT-ACL
 permit tcp any any eq 22
 permit udp host 10.10.99.51 any eq tacacs
 permit udp any any eq snmp
 permit udp any any eq 830
!
ip access-list extended COPP-ICMP-ACL
 permit icmp any any
!
control-plane
 service-policy input COPP-POLICY
!
! ===== Part 49/67: NETCONF/RESTCONF (จำกัดด้วย ACL Management ผ่าน AAA เดิม) =====
netconf-yang
restconf
ip http secure-server
!
! ===== Part 54/62: Model-Driven Telemetry + Logging/SNMP ที่แข็งแรงขึ้น =====
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization
 source-address 1.1.1.1
 stream yang-push
 update-policy periodic 3000
 receiver ip address 10.10.99.50 57500 protocol grpc-tcp
!
ntp server 10.10.99.50
ntp authenticate
ntp authentication-key 1 md5 NTPKey123 7
ntp trusted-key 1
logging host 10.10.99.50
logging trap informational
logging source-interface Loopback0
snmp-server host 10.10.99.50 version 3 priv noc-svc
snmp-server group NOC-GROUP v3 priv
snmp-server user noc-svc NOC-GROUP v3 auth sha AuthPass123! priv aes 128 PrivPass123!
!
ip access-list standard MGMT-ONLY-VTY
 permit 10.10.99.0 0.0.0.255
 deny any log
!
line console 0
 password Cisco123!
 login authentication default
 exec-timeout 10 0
 logging synchronous
line vty 0 15
 access-class MGMT-ONLY-VTY in
 transport input ssh
 login authentication default
 exec-timeout 10 0
!
no cdp run
lldp run
!
end
```

### เกณฑ์ Self-Check สุดท้ายก่อนปิดหลักสูตร CCNP ทั้งใบ

| Feature Group | คำสั่ง Verify หลัก | ควรเห็นอะไร |
|---|---|---|
| OSPF Timer/Auth ตรงกันทุกฝั่ง | `show ip ospf neighbor`, `show ip ospf interface` | FULL ทุกคู่, Hello/Dead ตรงกัน |
| BFD | `show bfd neighbors detail` | Session UP, Timer ตรงตาม Config |
| BGP iBGP Full-Mesh | `show ip bgp summary` | ทุก Neighbor มี State เป็นตัวเลข Prefix |
| Redistribution ปลอดภัย | `show ip route ospf \| include ^O\*E` | เห็น O*E1 เดียว ไม่มี Loop |
| uRPF | `show cef interface Port-channel1` | uRPF Mode = Loose (ไม่ใช่ Strict บน Link ที่มี Asymmetric Routing) |
| CoPP Headroom | `show policy-map control-plane class COPP-CRITICAL` | `drop` Counter = 0 แม้ตอน Traffic สูง |
| Anycast RP + MSDP | `show ip pim rp mapping`, `show ip msdp summary` | RP = 1.1.1.100, MSDP Peer = Established |
| AAA Fallback ปลอดภัย | `show run \| include username` | มี Local User Privilege 15 เสมอ |
| Telemetry/NETCONF | `show telemetry ietf subscription all` | Subscription State = Active |

---

## จบหลักสูตร CCNP เต็มรูปแบบ!

**ยินดีด้วยอย่างที่สุด — คุณเรียนครบ Part 1-70 (Step 1-700 จาก 1000) และผ่านทั้ง ENCOR + ENARSI
ครบทั้งหลักสูตร CCNP Enterprise แล้ว**

ทวนภาพการเดินทางทั้งหมดตั้งแต่ต้น:

- **Part 1-25 (CCNA)**: ปูพื้นฐานทุกอย่างตั้งแต่ OSI Model จนถึง Switching/Routing/Services/
  Security/Wireless พื้นฐาน — จบด้วย Capstone แรกที่สอน "การมองภาพรวม" เป็นครั้งแรก
- **Part 26-55 (CCNP ENCOR)**: ยกระดับทุก Protocol เดิมให้ "ลึกและซับซ้อนแบบ Enterprise จริง"
  (Multi-area OSPF, BGP, Redistribution, StackWise Virtual/MEC, SD-Access/SD-WAN, Security
  Architecture เต็มรูปแบบ, Automation) — จบด้วย Capstone ที่สอน Troubleshooting ข้าม Layer
  เป็นครั้งแรก
- **Part 56-70 (CCNP ENARSI)**: เปลี่ยนมุมมองจาก "สร้างให้ทำงาน" เป็น **"วินิจฉัยเมื่อพัง"**
  อย่างเป็นระบบทุก Domain (Layer 3/VPN/Security/Services) — จบด้วย Capstone ที่รวม Mock Exam
  เต็มรูปแบบ, Lab Simulation จับเวลาจริง, และเอกสารอ้างอิงระดับ Graduate ของทั้งหลักสูตร

จาก Enterprise Lab เดิมที่เริ่มต้นด้วย 2 Core Switch ต่อกันด้วย EtherChannel ธรรมดาใน **Part 1**
ตอนนี้กลายเป็นเครือข่ายที่ **ทนต่อความผิดพลาดได้จริง**: OSPF/BGP ที่ Converge เร็วด้วย BFD, CoPP ที่
ปรับ Rate ตาม Baseline จริงไม่ใช่ค่าเดาสุ่ม, uRPF ที่เข้าใจ Asymmetric Routing ของ Design ตัวเอง,
และทีมงาน (คุณ) ที่ผ่านการฝึกวินิจฉัยปัญหามากกว่า 80 Scenario ตลอด Part 56-70 — นี่คือระดับ
ความสามารถที่ตรงกับตำแหน่ง **Senior Network Engineer / NOC Lead** ในตลาดจริง และเป็นใบเบิกทางที่
แข็งแรงที่สุดสู่ระดับ **CCIE**

**CCNP Enterprise เต็มใบ (ENCOR 350-401 + ENARSI 300-410) จบสมบูรณ์ที่ Part นี้**

**Part 71 จะเริ่มระดับ CCIE Enterprise Infrastructure** — ไฟล์ถัดไปคือ
[part-071-ccie-lab-blueprint-overview.md](part-071-ccie-lab-blueprint-overview.md) หัวข้อ
**CCIE Lab Blueprint Overview** ซึ่งจะแนะนำภาพรวมข้อสอบ CCIE Enterprise Infrastructure ทั้ง
Written Exam และ Lab Exam (8 ชั่วโมง), Design/Deploy/Optimize Module, และวางแผน Part 72-100
ที่จะพา Enterprise Lab เดิมของเราไปสู่ระดับ **VXLAN/EVPN, MPLS L3VPN, Full-Stack Automation, และ
CCIE Full Lab Scenario เต็มรูปแบบ 4 Module**

**เจอกันที่ Part 71 — CCIE Enterprise Infrastructure เริ่มต้นแล้ว**
