# Part 99 — Capstone: Build an Enterprise Network From Scratch
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 981–990 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 98 — CCIE Lab Scenario #4: Optimize Module](part-098-ccie-lab-scenario-4-optimize.md)
> ที่ปิดท้าย CCIE Full Lab Scenario ทั้ง 4 Module (Diagnose/Design/Deploy/Optimize) บน **Enterprise
> Lab เดิม**ที่เราสร้างต่อเนื่องมาตั้งแต่ [Part 1](part-001-networking-fundamentals.md) — **Part นี้
> เปลี่ยนบริบทโดยสิ้นเชิงเป็นครั้งแรกในรอบ 98 Part**: เราจะ**ไม่แตะ Enterprise Lab เดิมอีกแล้ว**
> แต่จะรับบทวิศวกรที่ได้รับมอบหมายให้สร้าง**เครือข่ายใหม่ทั้งก้อนให้บริษัทสมมติ "Siam Innovation
> Corp" (SIC)** ตั้งแต่ **กระดาษเปล่า (Blank Slate)** — 2 สาขา (HQ + Branch), งบประมาณจริง,
> Requirement ทางธุรกิจจริง, และไม่มี "Lab ที่มีอยู่แล้วให้ต่อ" เหมือนทุก Part ก่อนหน้า
>
> นี่คือความต่างที่สำคัญที่สุดจาก Capstone ก่อนหน้าทั้ง 3 ครั้ง — [Part 25](part-025-ccna-capstone-lab.md)
> ทวน CCNA, [Part 55](part-055-encor-capstone-lab.md) ทวน ENCOR, [Part 70](part-070-enarsi-capstone-mock-exam.md)
> ทวน ENARSI ล้วนเป็นการ **"มองภาพเดิมที่สร้างมาแล้วให้ครบ"** แต่ Part 99 คือ **"สร้างภาพใหม่ทั้งภาพ
> ด้วยมือตัวเองครั้งเดียว โดยใช้ทักษะเดิมทั้งหมดจาก Part 1-98"** — ตั้งแต่ OSI Model ใน Step 1
> ไปจนถึง Zero Trust Architecture ใน Part 94 นี่คือบทพิสูจน์ว่าเนื้อหา 98 Part ที่ผ่านมา **ไม่ใช่
> ความรู้แยกส่วน** แต่เป็น**ทักษะเดียวที่ประกอบกันเป็นวิศวกรเครือข่ายตัวจริง** ที่ต้องเลือกใช้เทคโนโลยี
> ให้ **"พอดี" กับองค์กรจริง** ไม่ใช่ "ใหญ่ที่สุดที่เคยเรียนมา" — ทักษะที่ไม่มี Certification ใบไหน
> สอนตรงๆ แต่เป็นทักษะที่แยกวิศวกรระดับ Senior ออกจากวิศวกรที่ท่อง Config ได้อย่างชัดเจนที่สุด

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 981 | Project Kickoff — Business Requirements Document ของ "Siam Innovation Corp" |
| 982 | Phase 1 — Design: Topology, IP/VLAN Plan, และการเลือกเทคโนโลยีพร้อมเหตุผล |
| 983 | Phase 2 — Foundation Build: Initial Setup, VLAN/Trunk/STP/EtherChannel, IPv4/IPv6, Static Bootstrap |
| 984 | Phase 3 — Routing & Redundancy: OSPF, HSRP, และทำไมไม่ใช้ StackWise Virtual |
| 985 | Phase 4 — WAN & Remote Site: DMVPN Hub-Spoke + AWS Site-to-Site VPN ด้วย BGP |
| 986 | Phase 5 — Security Hardening: Security Stack ที่ Scale ให้พอดีกับงบประมาณ SMB |
| 987 | Phase 6 — Services & QoS: DHCP/DNS/NTP, QoS สำหรับ Call Center, และทำไมไม่ต้องมี Multicast |
| 988 | Phase 7 — Automation & Monitoring: ชุดเครื่องมือที่ Scale ให้พอดีกับทีม IT 2 คน |
| 989 | Phase 8 — Testing & Go-Live: Pre-Production Checklist, Cutover Plan, Post-Go-Live Validation |
| 990 | Full Lab — Running-Config ครบทุกอุปกรณ์ของ Siam Innovation Corp + Retrospective เชื่อมกลับทุก Part |

---

## Step 981 — Project Kickoff: Business Requirements Document ของ "Siam Innovation Corp"

### บริบทของโปรเจกต์

คุณคือวิศวกรเครือข่ายที่ได้รับมอบหมายให้เป็นผู้ออกแบบและสร้างเครือข่ายใหม่ทั้งหมดให้ **Siam
Innovation Corp (SIC)** บริษัท E-commerce และ Logistics-Tech สัญชาติไทยที่เพิ่งปิดรอบระดมทุน
Series A และกำลังย้ายจาก Co-working Space (ที่มีแค่ Router Wi-Fi ตัวเดียวแจก Internet ให้ทุกคน)
ไปยัง **สำนักงานใหญ่ (HQ) แห่งใหม่ในกรุงเทพฯ** และเปิด **คลังสินค้า/ศูนย์กระจายสินค้า (Branch)
แห่งใหม่ที่ชลบุรี** พร้อมกัน — ไม่มีเครือข่ายเดิมให้ Migrate นี่คือ **Greenfield Project เต็มรูปแบบ**

### ตาราง Business Requirements Document (BRD) ฉบับสรุป

| หัวข้อ | รายละเอียด |
|---|---|
| ธุรกิจ | E-commerce (ขายสินค้า Home & Lifestyle ออนไลน์) + บริการ Last-mile Delivery ของตัวเอง |
| สถานะบริษัท | ก่อตั้ง 2 ปี, ปิดรอบทุน Series A แล้ว, เติบโตเร็ว (YoY Revenue +180%) |
| Site 1 (HQ) | อาคารสำนักงานเช่าใหม่ ย่านสุขุมวิท กรุงเทพฯ — 1 ชั้น พื้นที่ ~800 ตร.ม. |
| Site 2 (Branch) | คลังสินค้า + สำนักงานภูมิภาคใหม่ ชลบุรี (ใกล้ท่าเรือ, เข้าถึงเส้นทางขนส่งหลัก) |
| จำนวนผู้ใช้งาน HQ | 150 คน (รวม Call Center 40 ที่นั่ง) |
| จำนวนผู้ใช้งาน Branch | 50 คน (Warehouse Operations + สำนักงานขาย/บัญชีภูมิภาค) |
| งบประมาณ Capex ปีแรก | ~7,850,000 บาท (ดูรายละเอียดตารางงบประมาณด้านล่าง) |
| งบประมาณ Opex/เดือน | ~165,000 บาท/เดือน (Internet Circuit + 4G Backup + Support Contract + SaaS Subscription) |
| ทีม IT ที่ดูแลเครือข่าย | 2 คน (Network/Infra Engineer) — **นี่คือข้อจำกัดที่สำคัญที่สุดของการออกแบบทั้ง Part นี้** |

### แผนก/จำนวนผู้ใช้งาน (สำหรับ VLAN/Voice/QoS Planning)

| แผนก | HQ | Branch | หมายเหตุ |
|---|---|---|---|
| Engineering/Product | 25 | 0 | ต้องการ Bandwidth สูงสำหรับ CI/CD ไป AWS |
| Marketing/Content | 15 | 0 | |
| Finance/Accounting | 10 | 5 | เข้าถึงระบบ ERP/บัญชี |
| HR/Admin | 5 | 2 | |
| Executive | 5 | 0 | |
| **Customer Service / Call Center** | **40** | 0 | Softphone (UCaaS), ต้องการ QoS สูงสุด |
| IT | 5 | 0 | |
| Warehouse Operations (Picker/Packer) | 0 | 35 | ใช้ Barcode Scanner ผ่าน Wi-Fi ตลอดเวลา |
| สำนักงานขาย/บัญชีภูมิภาค | 0 | 8 | |
| Reception/Security Guard | 5 | 0 | ใช้ Cisco IP Phone จริง (ไม่ใช่ Softphone) |
| **รวม** | **150** | **50** | **รวมทั้งบริษัท 200 คน** |

### แอปพลิเคชันหลักที่เครือข่ายต้องรองรับ

| แอปพลิเคชัน | Host ที่ไหน | ผลต่อการออกแบบเครือข่าย |
|---|---|---|
| E-commerce Platform (Web + Microservices) | AWS (ap-southeast-1, สิงคโปร์) | ต้องมี Reliable Path ไป AWS — เลือก VPN แบบมี Dynamic Routing |
| WMS (Warehouse Management System) | AWS Backend + Local Edge Cache ที่ Branch | Branch ต้องมี Server ท้องถิ่นกันสัญญาณ Internet สะดุดขณะ Scan สินค้า |
| Microsoft 365 (Email/Teams/SharePoint) | Cloud SaaS (Global) | Internet Bandwidth/QoS ปกติ ไม่ต้องมี Dedicated Link |
| ERP/บัญชี (Cloud SaaS ไทย) | Cloud SaaS | เหมือนข้างต้น |
| Contact Center (UCaaS, Softphone 40 ที่นั่ง) | Cloud SaaS + Softphone บน PC | **ต้องมี QoS/LLQ เฉพาะสำหรับ Voice Traffic ของ Softphone** |
| CCTV/NVR คลังสินค้า | Local NVR ที่ Branch + Cloud Backup | ต้องมี VLAN แยกสำหรับกล้อง (IoT Segment) |
| Barcode Scanner/Handheld | Wi-Fi ที่ Branch | ต้องมี Wi-Fi Coverage ครอบคลุมคลังสินค้าทั้งหมด |

### แผนการเติบโต 3 ปี (Growth Plan) — **ทุกการออกแบบใน Part นี้ต้องรองรับตารางนี้ได้โดยไม่ต้อง Rebuild**

| ปี | Site | Headcount | ผลต่อสถาปัตยกรรม |
|---|---|---|---|
| ปีที่ 1 (ปัจจุบัน) | HQ + Branch (2 Site) | 200 คน | สร้างตามที่ออกแบบใน Part นี้ |
| ปีที่ 2 | เพิ่ม Regional Sales Office เชียงใหม่ (Site 3, ~30 คน) | 260 คน | ต้องเพิ่ม DMVPN Spoke ตัวที่ 2 ได้ทันทีโดยไม่แก้ Hub — เหตุผลหลักที่เลือก DMVPN แทน Point-to-Point |
| ปีที่ 3 | พิจารณาคลังสินค้าที่ 2 (ระยอง) + สำรวจตลาดเวียดนาม | 350+ คน | จุดที่ต้อง Re-evaluate: SD-WAN Overlay (ถ้า Site > 5), Direct Connect ไป AWS (ถ้า Traffic โตเกิน VPN รองรับ), WLC รวมศูนย์ (Part 36) แทน Embedded Controller |

### Security & Compliance Requirements — ตัวกำหนดสำคัญของ Phase 5 (Step 986)

| ข้อกำหนด | รายละเอียด | ผลต่อการออกแบบ |
|---|---|---|
| **PDPA** (Personal Data Protection Act ของไทย, มีผลบังคับใช้ตั้งแต่ปี 2565) | ต้องปกป้องข้อมูลลูกค้า (ชื่อ, ที่อยู่, เบอร์โทร, ประวัติการสั่งซื้อ) ตลอด Lifecycle | ต้อง Encrypt Data in Transit (VPN ทุกเส้นทาง), Log การเข้าถึงข้อมูล, จำกัด Access ตาม Least Privilege |
| **PCI-DSS** (Payment Card Industry) Scope แบบจำกัด (SAQ A-EP) | ไม่เก็บเลขบัตรเครดิตเองบน Server บริษัท (ใช้ Payment Gateway Tokenization ของ Omise/2C2P) แต่ทีม Call Center อาจเห็นข้อมูลบางส่วน (Masked) ผ่านเครื่องมือ Support | ต้อง Segment VLAN ของ Call Center ให้แยกจาก VLAN อื่นอย่างชัดเจน (CDE-adjacent Segment) |
| Data Residency | ลูกค้าส่วนใหญ่อยู่ในไทย | เลือก AWS Region **ap-southeast-1 (สิงคโปร์)** — ใกล้ที่สุดที่มี Full AWS Service, สอดคล้องกับเงื่อนไข Cross-border Transfer ของ PDPA ผ่านสัญญา Data Processing Agreement กับ AWS |
| ระดับความเสี่ยงที่ยอมรับได้ | บริษัทขนาดกลาง งบจำกัด — **ไม่ใช่ธนาคารหรือหน่วยงานรัฐ** | **นี่คือเหตุผลสำคัญที่สุดที่ Part นี้จะ "Scale down" Security Stack จากที่สอนใน Part 43-46/78/94 อย่างมีเหตุผล ไม่ใช่ทำน้อยเพราะขี้เกียจ** |

### งบประมาณ Capex ปีแรก (โดยประมาณ)

| หมวด | HQ (บาท) | Branch (บาท) | รวม (บาท) |
|---|---|---|---|
| WAN Router (ISR4331 + ISR4321) | 280,000 | 190,000 | 470,000 |
| Firewall (ASA 5516-X ที่ HQ เท่านั้น) | 420,000 | 0 | 420,000 |
| Core/Distribution Switch (Catalyst 9300 คู่) | 900,000 | 0 | 900,000 |
| L3 Switch แบบ Standalone (Branch) | 0 | 320,000 | 320,000 |
| Access Switch (Catalyst 9200/9200L) | 380,000 | 260,000 | 640,000 |
| Access Point (Catalyst 9120AXI, Mobility Express) | 180,000 | 140,000 | 320,000 |
| ระบบไฟฟ้าสำรอง/Rack/Cabling | 650,000 | 480,000 | 1,130,000 |
| ค่าติดตั้ง Circuit (ISP1/ISP2 Fiber + LTE Backup) | 120,000 | 90,000 | 210,000 |
| Cisco Umbrella (Subscription ปีแรก, DNS Security) | 180,000 | 60,000 | 240,000 |
| ค่าแรง Professional Services/Implementation | 1,100,000 | 600,000 | 1,700,000 |
| Contingency (10%) | — | — | 715,000 |
| **รวม Capex ปีแรก** | | | **~7,850,000 บาท** |

> **ข้อสังเกตสำคัญที่สุดของตารางนี้**: ไม่มีรายการ ISE, DNA Center, StackWise Virtual License,
> หรือ SD-WAN Controller เลยแม้แต่รายการเดียว — ไม่ใช่เพราะทีมงานไม่รู้จักเทคโนโลยีเหล่านี้ (Part
> 34, 38-39, 43-44, 50, 76-77 สอนไว้ครบแล้ว) แต่เพราะ**งบประมาณและขนาดองค์กร (200 คน, 2 Site,
> ทีม IT 2 คน) ไม่สมเหตุสมผลกับการลงทุนนั้น** — ทักษะที่ Part นี้จะฝึกมากที่สุดคือ **การรู้ว่าเมื่อไหร่
> ไม่ควรใช้เทคโนโลยีระดับ CCIE เต็มรูปแบบ** ไม่น้อยไปกว่าการรู้วิธี Config มัน

---

## Step 982 — Phase 1: Design — Topology, IP/VLAN Plan, และการเลือกเทคโนโลยีพร้อมเหตุผล

> Phase นี้ใช้ทักษะจาก **Part 96 (CCIE Lab Scenario #2: Design Module)** — เริ่มจาก Requirement
> (Step 981) แล้ว Derive สถาปัตยกรรมที่เหมาะสมที่สุด ไม่ใช่เริ่มจากเทคโนโลยีที่อยากใช้แล้วหา
> เหตุผลมารองรับทีหลัง

### 982.1 — Topology ระดับสูง (High-Level Diagram)

```
                                    ┌────────────────────┐
                                    │   AWS ap-southeast-1 │
                                    │  (E-commerce/WMS)    │  <- Part 93 (Cloud Connectivity)
                                    └──────────┬───────────┘
                              IPsec VPN + eBGP │ (Tunnel1/Tunnel2, AS 65010 <-> AS 64512)
   ┌─────────────────────────────────────────┴─────────────────────────────────────────┐
   │                                    INTERNET                                         │
   └──────┬──────────────────────────────────────────────────────────────────┬──────────┘
    ISP1/ISP2 (Dual, Fiber)                                          ISP3 (Fiber) + LTE Backup
   ┌───────┴────────┐                                                  ┌───────┴────────┐
   │   HQ-RTR1        │  <- DMVPN Hub, NAT, ISP Failover (Part 9,15,19,51-53)   │   BR-RTR1        │  <- DMVPN Spoke (Part 52-53)
   └───────┬────────┘◄══════════ DMVPN mGRE Tunnel0 (172.20.253.0/24) ══════════►└───────┬────────┘
   ┌───────┴────────┐                                                            ┌───────┴────────┐
   │   HQ-FW1 (ASA)   │  <- Internet Edge Firewall (Part 43-46)                    │   BR-SW1 (L3)    │  <- Collapsed Dist+Access (No StackWise Virtual)
   └───────┬────────┘                                                            └───┬────────┬───┘
┌──────────┴──────────┐                                                        ┌──────┴──┐  ┌──┴───────┐
│ HQ-CORE-SW1 ⇄ HQ-CORE-SW2 │  <- Collapsed Core+Dist, HSRP, Po1 LACP (Part 6,18,33-35)   │BR-ACC-SW2│  │ BR-AP1   │
└───┬──────────────┬───┘                                                        └────┬────┘  └──────────┘
┌───┴───┐      ┌────┴────┐                                                           │
│HQ-ACC-SW1│    │HQ-ACC-SW2│                                                    Warehouse Floor:
└───┬───┘      └────┬────┘                                                     Scanner Wi-Fi + CCTV + NVR
    │                │                                                          + Regional Sales Office
 Office Floor 1   Office Floor 2 + Call Center                                  (Chonburi, 50 users)
 (HQ, 150 users)  + HQ-AP1/HQ-AP2 (Mobility Express Wi-Fi)
```

### 982.2 — IP Address Plan ฉบับใหม่ของ SIC (แยกจาก `00-ip-address-plan.md` โดยเจตนา)

> **หมายเหตุสำคัญ**: `00-ip-address-plan.md` คือ Canonical Reference ของ **Enterprise Lab เดิม**
> (Part 1-98) ที่ใช้ `10.10.x.x` / `10.255.x.x` / `1.1.1.x` — Part นี้สร้างเครือข่าย**ใหม่ทั้งก้อน**
> ให้บริษัท**ใหม่**จึงต้องมี IP Plan **ของตัวเอง** ไม่ปนกับของเดิม (บทเรียนจาก Part 7: IP Planning
> ต้องคิดเผื่อ M&A/การรวมเครือข่ายในอนาคตเสมอ — เลือกช่วง IP ที่ไม่ทับซ้อนตั้งแต่ Day 1)

| รายการ | ช่วงที่ใช้ | เหตุผล |
|---|---|---|
| HQ Internal | `172.20.0.0/16` | แยกจาก `10.10.x.x` ของ Lab เดิมอย่างสิ้นเชิง |
| Branch Internal | `172.21.0.0/16` | |
| Loopback0 (Router-ID) HQ | `172.20.255.0/24` | |
| Loopback0 (Router-ID) Branch | `172.21.255.0/24` | |
| Public-facing (จำลอง ISP) | `198.51.100.0/24` (TEST-NET-2, RFC 5737) | Lab เดิมใช้ `203.0.113.0/24` (TEST-NET-3) — เลือกช่วงคนละช่วงเพื่อไม่ให้สับสนตอนอ่าน Config คู่กัน |
| DMVPN Tunnel Network | `172.20.253.0/24` | |
| AWS VPN Tunnel (BGP link-local ตามมาตรฐาน AWS) | `169.254.10.0/30`, `169.254.11.0/30` | ค่ามาตรฐานที่ AWS กำหนดให้ตอนสร้าง Site-to-Site VPN จริง |
| AWS VPC (ฝั่ง Cloud, Reference เท่านั้น) | `10.50.0.0/16` | นอกขอบเขต Cisco IOS Lab (เหมือนแนวทาง Part 93) |

### 982.3 — VLAN Plan: HQ

| VLAN | ชื่อ | Subnet (IPv4) | Subnet (IPv6 ULA) | Default Gateway |
|---|---|---|---|---|
| 110 | DATA-STAFF | 172.20.10.0/24 | fd20:20:110::/64 | 172.20.10.1 (HSRP VIP) |
| 120 | VOICE (Cisco IP Phone จริง — Reception เท่านั้น) | 172.20.20.0/24 | fd20:20:120::/64 | 172.20.20.1 (HSRP VIP) |
| 130 | SERVERS (AD/DC, NPS/RADIUS, LibreNMS, DNS Forwarder) | 172.20.30.0/24 | fd20:20:130::/64 | 172.20.30.1 (HSRP VIP) |
| 140 | WIFI-CORP | 172.20.40.0/24 | fd20:20:140::/64 | 172.20.40.1 (HSRP VIP) |
| 141 | WIFI-GUEST (Isolated, Internet-only) | 172.20.41.0/24 | fd20:20:141::/64 | 172.20.41.1 (HSRP VIP) |
| 150 | CALLCENTER (Softphone 40 ที่นั่ง — CDE-adjacent) | 172.20.50.0/24 | fd20:20:150::/64 | 172.20.50.1 (HSRP VIP) |
| 199 | MGMT | 172.20.99.0/24 | fd20:20:199::/64 | 172.20.99.1 (HSRP VIP) |

### 982.4 — VLAN Plan: Branch (Chonburi Warehouse + Regional Office)

| VLAN | ชื่อ | Subnet (IPv4) | Subnet (IPv6 ULA) | Default Gateway |
|---|---|---|---|---|
| 210 | DATA-OFFICE | 172.21.10.0/24 | fd21:20:210::/64 | 172.21.10.1 (BR-SW1, No HSRP) |
| 220 | VOICE | 172.21.20.0/24 | fd21:20:220::/64 | 172.21.20.1 |
| 230 | WMS-EDGE (Local Cache Server กันสัญญาณ Internet สะดุด) | 172.21.30.0/24 | fd21:20:230::/64 | 172.21.30.1 |
| 240 | WIFI-WAREHOUSE (Barcode Scanner/Handheld) | 172.21.40.0/24 | fd21:20:240::/64 | 172.21.40.1 |
| 241 | WIFI-GUEST | 172.21.41.0/24 | fd21:20:241::/64 | 172.21.41.1 |
| 250 | CCTV-IOT (กล้อง/NVR/Sensor — Segment แยกเด็ดขาด) | 172.21.50.0/24 | fd21:20:250::/64 | 172.21.50.1 |
| 299 | MGMT | 172.21.99.0/24 | fd21:20:299::/64 | 172.21.99.1 |

> **สังเกตการออกแบบที่ตั้งใจ**: HQ มี HSRP VIP (`.2`/`.3` เป็น Real IP ก่อนขึ้น `.1` VIP —
> Convention เดียวกับ `00-ip-address-plan.md` ของ Part 18 เพราะ HQ มี Core Switch 2 ตัว) แต่
> Branch **ไม่มี HSRP** เพราะมี L3 Switch แค่ตัวเดียว (`BR-SW1` เป็น Gateway ตรง `.1` เลย) — นี่คือ
> การยอมรับ Single Point of Failure ที่ Branch อย่างมีเหตุผล (ดูเหตุผลเต็มใน Step 984)

### 982.5 — Transit Link Plan

| Link | Subnet | หมายเหตุ |
|---|---|---|
| ISP1 ↔ HQ-RTR1 | 198.51.100.0/30 | ISP1 .1 / HQ-RTR1 .2 |
| ISP2 ↔ HQ-RTR1 | 198.51.100.4/30 | ISP2 .1 / HQ-RTR1 .6 (Backup, Floating Static Route) |
| HQ-RTR1 ↔ HQ-FW1 | 172.20.254.0/30 | RTR1 .1 (outside ของ FW1 คือ .2) |
| HQ-FW1 ↔ HQ-CORE-SW1 | 172.20.254.4/30 | FW1 .5 / CORE-SW1 .6 |
| HQ-CORE-SW1 ↔ HQ-CORE-SW2 (Po1, Routed, LACP) | 172.20.254.8/30 | CORE-SW1 .9 / CORE-SW2 .10 |
| ISP3 ↔ BR-RTR1 | 198.51.100.8/30 | ISP3 .9 / BR-RTR1 .10 |
| BR-RTR1 ↔ BR-SW1 | 172.21.254.0/30 | RTR1 .1 / BR-SW1 .2 |

### 982.6 — การเลือกเทคโนโลยีพร้อมเหตุผล (Design Decision Log)

| การตัดสินใจ | ตัวเลือกที่พิจารณา | ตัวเลือกที่เลือก | เหตุผล |
|---|---|---|---|
| Campus Architecture ที่ HQ | 3-Tier (Core/Dist/Access), 2-Tier Collapsed | **2-Tier Collapsed Core+Distribution** | 150 คน อาคารเดียวชั้นเดียว — ไม่มี East-West Traffic ปริมาณมากพอที่ต้องแยก Core ออกจาก Distribution ตาม Part 33 (Advanced Campus Design เตือนไว้ว่า 3-Tier เหมาะกับ Multi-building/Multi-floor ขนาดใหญ่เท่านั้น) |
| Redundancy ที่ HQ Core | StackWise Virtual (Part 34), Standalone Pair + HSRP | **Standalone Pair + HSRP** | ดูรายละเอียดเต็มใน Step 984 — สรุปสั้น: SVL ต้องมี Dedicated Fiber Link คู่ + License เพิ่ม ไม่คุ้มกับ Traffic ระดับ 150 คน |
| Routing Protocol หลัก | OSPF, EIGRP | **OSPF Single-Area (Area 0)** | Vendor-neutral (เผื่ออนาคตซื้ออุปกรณ์ยี่ห้ออื่น), อุปกรณ์ L3 มีแค่ 4 ตัว (HQ-RTR1, CORE-SW1/2, BR-RTR1, BR-SW1) ไม่จำเป็นต้อง Multi-Area เลย |
| Routing สำหรับ AWS VPN | Static Route, eBGP | **eBGP** | AWS **กำหนด**ให้ Site-to-Site VPN แบบ Redundant ต้องใช้ BGP เพื่อ Failover ระหว่าง 2 Tunnel อัตโนมัติ (ไม่ใช่ทางเลือกทางวิศวกรรมของเรา แต่เป็น Requirement จาก Cloud Provider — เพิ่ม BGP เข้ามาเฉพาะจุดที่จำเป็นจริงเท่านั้น ไม่ใช้ทั่วทั้ง Network) |
| WAN ไป Branch | MPLS L3VPN, Commercial SD-WAN, Leased Line, **DMVPN** | **DMVPN Phase 3 (Hub-Spoke, mGRE)** | ดูตารางเปรียบเทียบ 982.7 |
| Wireless Controller | Dedicated WLC (9800-CL/Appliance), **Embedded/Mobility Express** | **Embedded Controller บน AP (Mobility Express)** | รวม AP ทั้ง 2 Site แค่ ~6 ตัว — Dedicated WLC (Part 36) คุ้มค่าเมื่อมี AP หลักสิบตัวขึ้นไปหรือหลาย Site ที่ต้องบริหารรวมศูนย์ ไม่ใช่ตอนนี้ |
| Perimeter Security | NGFW ทั้ง 2 Site, **NGFW ที่ HQ + Router ZBFW ที่ Branch** | ผสมกัน | ดูรายละเอียดเต็มใน Step 986 |
| Micro-Segmentation | TrustSec/SGT + ISE, **VLAN + ACL แบบ Macro-Segmentation** | **Macro-Segmentation** | ต้องมี ISE License + Infrastructure เพิ่ม (Part 44/78) — Deferred ไปปีที่ 2-3 ถ้า Compliance บังคับเพิ่ม |
| Automation Platform | Full CI/CD (Jenkins), DNA Center, **Ansible + Python Script เบา** | **Ansible + Python** | ทีม IT มี 2 คน อุปกรณ์ 11 ตัว — Pipeline แบบ Part 79 คุ้มค่าที่ Fleet ขนาดหลักสิบ-ร้อยอุปกรณ์ |

### 982.7 — ตารางเปรียบเทียบ WAN Technology (เหตุผลที่เลือก DMVPN)

| คุณสมบัติ | Leased Line/P2P | MPLS L3VPN (Part 86-87) | Commercial SD-WAN (Part 39,76) | **DMVPN (Part 51-53) — เลือก** |
|---|---|---|---|---|
| ค่าใช้จ่ายเริ่มต้น | สูง (ต้องเดินสายเฉพาะ) | สูง (Provider Setup Fee) | สูงมาก (Controller License/Subscription) | ต่ำ (ใช้ Internet ที่มีอยู่แล้ว) |
| ค่าใช้จ่ายรายเดือน | สูงมาก | สูง | สูง (License ต่อ Site) | ต่ำ (Internet Circuit ปกติ) |
| Encryption ในตัว | ไม่มี (ต้องเพิ่ม IPsec เอง) | ไม่มี (Private แต่ไม่ได้ Encrypt) | มี (Built-in) | **มี (IPsec ในตัว Design)** |
| เพิ่ม Spoke ตัวใหม่ (Site 3 ปีที่ 2) | ต้องเดินสายใหม่ทั้งหมด (ช้า) | ต้องรอ Provider Provision (2-6 สัปดาห์) | เร็ว (Zero-touch) | **เร็ว (แค่ Config Spoke เพิ่ม ไม่ต้องแก้ Hub)** |
| ต้องมี Controller กลาง | ไม่ต้อง | ไม่ต้อง (Provider จัดการ) | **ต้องมี (Cost เพิ่ม)** | ไม่ต้อง |
| เหมาะกับจำนวน Site | เท่าไหร่ก็ได้แต่แพง | 3+ Site ที่ต้องการ SLA สูง | 5+ Site ที่ต้องการ Central Policy | **2-5 Site ที่งบจำกัด** |
| ความเห็นสำหรับ SIC | ตัดออก — ไม่ยืดหยุ่น, แพงเกินความจำเป็นสำหรับ 2 Site | ตัดออก — SLA สูงเกินความจำเป็น, Lead Time ไม่ทันกำหนด Go-Live | ตัดออกตอนนี้ — **พิจารณาใหม่ปีที่ 3 ถ้า Site ≥ 5** (ดู Growth Plan Step 981) | **เลือก — คุ้มค่าที่สุดสำหรับ 2 Site วันนี้ พร้อมขยายเป็น Multi-Spoke ปีที่ 2 ได้ทันที** |

---

## Step 983 — Phase 2: Foundation Build — Initial Setup, Layer 2, IPv4/IPv6, Static Bootstrap

> Phase นี้ใช้ทักษะจาก **Part 1-10 (Networking Fundamentals ถึง Inter-VLAN Routing)** โดยตรง —
> ทุกอุปกรณ์เริ่มจาก Factory-Default จริง

### 983.1 — Initial Setup Template (ใช้กับทุกอุปกรณ์ Layer 2/3 ของ SIC — อ้าง Part 2)

```
enable
configure terminal
!
hostname HQ-CORE-SW1
no ip domain-lookup
ip domain-name sic.co.th
!
enable secret S1cCap5tone!2026
service password-encryption
!
banner motd #
================================================================
  Siam Innovation Corp - Authorized Access Only
  Unauthorized access is prohibited and monitored.
================================================================
#
!
username sic-admin privilege 15 secret S1cAdm1n!Str0ng
!
line console 0
 password S1cCap5tone!2026
 login
 exec-timeout 10 0
 logging synchronous
!
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0
!
interface Loopback0
 description ** Management/Router-ID Loopback **
 ip address 0.0.0.0 255.255.255.255
!
no cdp run
lldp run
!
end
copy running-config startup-config
```

### 983.2 — Layer 2 Foundation ที่ HQ-CORE-SW1/2 และ HQ-ACC-SW1/2 (VLAN, Trunk, STP, EtherChannel)

```
! ==== HQ-CORE-SW1: สร้าง VLAN Database ====
HQ-CORE-SW1(config)# vlan 110
HQ-CORE-SW1(config-vlan)# name DATA-STAFF
HQ-CORE-SW1(config-vlan)# vlan 120
HQ-CORE-SW1(config-vlan)# name VOICE
HQ-CORE-SW1(config-vlan)# vlan 130
HQ-CORE-SW1(config-vlan)# name SERVERS
HQ-CORE-SW1(config-vlan)# vlan 140
HQ-CORE-SW1(config-vlan)# name WIFI-CORP
HQ-CORE-SW1(config-vlan)# vlan 141
HQ-CORE-SW1(config-vlan)# name WIFI-GUEST
HQ-CORE-SW1(config-vlan)# vlan 150
HQ-CORE-SW1(config-vlan)# name CALLCENTER
HQ-CORE-SW1(config-vlan)# vlan 199
HQ-CORE-SW1(config-vlan)# name MGMT
HQ-CORE-SW1(config-vlan)# exit

! ==== EtherChannel Po1 ระหว่าง CORE-SW1 <-> CORE-SW2 (Routed, LACP) — เหมือน Part 6 ====
HQ-CORE-SW1(config)# interface range TenGigabitEthernet1/1/1-2
HQ-CORE-SW1(config-if-range)# channel-group 1 mode active
HQ-CORE-SW1(config-if-range)# no shutdown
HQ-CORE-SW1(config-if-range)# exit
HQ-CORE-SW1(config)# interface Port-channel1
HQ-CORE-SW1(config-if)# no switchport
HQ-CORE-SW1(config-if)# description ** Po1 to HQ-CORE-SW2 - Routed EtherChannel **
HQ-CORE-SW1(config-if)# ip address 172.20.254.9 255.255.255.252
HQ-CORE-SW1(config-if)# no shutdown
HQ-CORE-SW1(config-if)# exit

! ==== Trunk ไปยัง ACC-SW1/2 ====
HQ-CORE-SW1(config)# interface GigabitEthernet1/0/1
HQ-CORE-SW1(config-if)# description ** Trunk to HQ-ACC-SW1 **
HQ-CORE-SW1(config-if)# switchport mode trunk
HQ-CORE-SW1(config-if)# switchport trunk allowed vlan 110,120,130,140,141,150,199
HQ-CORE-SW1(config-if)# spanning-tree guard root
HQ-CORE-SW1(config-if)# no shutdown
HQ-CORE-SW1(config-if)# exit
HQ-CORE-SW1(config)# interface GigabitEthernet1/0/2
HQ-CORE-SW1(config-if)# description ** Trunk to HQ-ACC-SW2 (Call Center Floor) **
HQ-CORE-SW1(config-if)# switchport mode trunk
HQ-CORE-SW1(config-if)# switchport trunk allowed vlan 110,120,130,140,141,150,199
HQ-CORE-SW1(config-if)# spanning-tree guard root
HQ-CORE-SW1(config-if)# no shutdown
HQ-CORE-SW1(config-if)# exit

! ==== STP: HQ-CORE-SW1 เป็น Root ทุก VLAN (มีแค่คู่เดียว ไม่ต้องแบ่งเหมือน Lab เดิมที่มี 4 DIST) ====
HQ-CORE-SW1(config)# spanning-tree mode rapid-pvst
HQ-CORE-SW1(config)# spanning-tree vlan 110,120,130,140,141,150,199 root primary

! ==== HQ-ACC-SW1: Access Port ตัวอย่าง ====
HQ-ACC-SW1(config)# interface range GigabitEthernet1/0/1-40
HQ-ACC-SW1(config-if-range)# switchport mode access
HQ-ACC-SW1(config-if-range)# switchport access vlan 110
HQ-ACC-SW1(config-if-range)# spanning-tree portfast
HQ-ACC-SW1(config-if-range)# spanning-tree bpduguard enable
HQ-ACC-SW1(config-if-range)# no shutdown
HQ-ACC-SW1(config-if-range)# exit

! ==== HQ-ACC-SW2 (Call Center Floor): Voice VLAN บนพอร์ต Softphone PC ====
HQ-ACC-SW2(config)# interface range GigabitEthernet1/0/1-40
HQ-ACC-SW2(config-if-range)# switchport mode access
HQ-ACC-SW2(config-if-range)# switchport access vlan 150
HQ-ACC-SW2(config-if-range)# spanning-tree portfast
HQ-ACC-SW2(config-if-range)# spanning-tree bpduguard enable
HQ-ACC-SW2(config-if-range)# no shutdown
HQ-ACC-SW2(config-if-range)# exit

! ==== Reception (Cisco IP Phone จริงตัวเดียวที่บริษัท) ====
HQ-ACC-SW1(config)# interface GigabitEthernet1/0/41
HQ-ACC-SW1(config-if)# description ** Reception - Cisco IP Phone **
HQ-ACC-SW1(config-if)# switchport mode access
HQ-ACC-SW1(config-if)# switchport access vlan 110
HQ-ACC-SW1(config-if)# switchport voice vlan 120
HQ-ACC-SW1(config-if)# spanning-tree portfast
HQ-ACC-SW1(config-if)# no shutdown
```

### 983.3 — Layer 3 Foundation: SVI + IPv4/IPv6 + Static Bootstrap (ก่อนมี OSPF)

```
! ==== เปิด Routing บน HQ-CORE-SW1 ====
HQ-CORE-SW1(config)# ip routing
HQ-CORE-SW1(config)# ipv6 unicast-routing

! ==== SVI ตัวอย่าง VLAN110 (Real IP .2 ก่อนมี HSRP - แบบเดียวกับ Convention ของ 00-ip-address-plan.md) ====
HQ-CORE-SW1(config)# interface Vlan110
HQ-CORE-SW1(config-if)# description ** DATA-STAFF SVI **
HQ-CORE-SW1(config-if)# ip address 172.20.10.2 255.255.255.0
HQ-CORE-SW1(config-if)# ipv6 address fd20:20:110::2/64
HQ-CORE-SW1(config-if)# no shutdown
HQ-CORE-SW1(config-if)# exit
! (ทำซ้ำสำหรับ VLAN 120,130,140,141,150,199 ตามตาราง Step 982.3)

! ==== Static Default Route ชั่วคราวชี้ไปยัง FW1 (จะถูกแทนที่ด้วย OSPF ใน Phase 3) ====
HQ-CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 172.20.254.5

! ==== HQ-RTR1: Bootstrap Static Route ไปยัง Internal Network ก่อนมี OSPF ====
HQ-RTR1(config)# ip route 172.20.0.0 255.255.0.0 172.20.254.2

! ==== BR-SW1 (Branch, L3 Standalone): SVI ตรง .1 ไม่ต้องรอ HSRP ====
BR-SW1(config)# ip routing
BR-SW1(config)# interface Vlan210
BR-SW1(config-if)# ip address 172.21.10.1 255.255.255.0
BR-SW1(config-if)# ipv6 address fd21:20:210::1/64
BR-SW1(config-if)# no shutdown
BR-SW1(config-if)# exit
! (ทำซ้ำสำหรับ VLAN 220,230,240,241,250,299)
BR-SW1(config)# ip route 0.0.0.0 0.0.0.0 172.21.254.1
```

> **Verify ก่อนไป Phase ถัดไป**: `show vlan brief`, `show interfaces trunk`, `show spanning-tree
> summary`, `show etherchannel summary`, `show ip interface brief` ทุกอุปกรณ์ — Ping ข้าม VLAN
> ภายใน Site เดียวกันต้องผ่านหมดก่อนเริ่ม Phase 3 (กฎเหล็กเดียวกับ Part 25 Step 243 PHASE 3)

---

## Step 984 — Phase 3: Routing & Redundancy — OSPF, HSRP, และทำไมไม่ใช้ StackWise Virtual

> Phase นี้ใช้ทักษะจาก **Part 11-12 (OSPF), Part 18 (HSRP), Part 34 (StackWise Virtual), Part 35
> (Advanced FHRP Campus HA)**

### 984.1 — เหตุผลที่ไม่ใช้ StackWise Virtual (SVL) ที่ HQ Core

| ปัจจัย | StackWise Virtual (Part 34) | Standalone Pair + HSRP (เลือกใช้จริง) |
|---|---|---|
| Hardware Requirement | ต้องมี Dedicated SVL Link (สาย Fiber คู่ 10G/40G เฉพาะ) | ใช้ Link ปกติที่มีอยู่แล้ว (Po1) |
| ความซับซ้อนในการ Troubleshoot | สูง — Failure Domain เดียวมองเป็น Switch ตัวเดียว, Debug ยากกว่าเมื่อ Split-Brain | ต่ำ — สอง Switch อิสระ, Debug ทีละตัวได้ตรงไปตรงมา (สำคัญมากเมื่อทีม IT มี 2 คน) |
| ประโยชน์หลัก | Loop-free Topology แบบ Active/Active เต็มรูปแบบ, ลด STP Blocking Link | ไม่จำเป็นเมื่อมี Access Switch แค่ 2 ตัว (Traffic ไม่มากพอที่ Blocking Link จะเป็นปัญหา) |
| ค่าใช้จ่ายเพิ่ม | License + สาย SVL เฉพาะ | ไม่มี |
| **สรุปสำหรับ SIC** | **ไม่คุ้มค่า** — ประโยชน์ (Active/Active Bandwidth เต็ม, ไม่มี Blocking Link) ไม่คุ้มกับความเสี่ยงด้าน Troubleshooting ที่เพิ่มขึ้นสำหรับทีมขนาดนี้ | **เลือกใช้** — HSRP ธรรมดาให้ Redundancy พอเพียง, Failure Domain ชัดเจน, ทีม 2 คน Debug ได้เร็วกว่า |

### 984.2 — OSPF Single-Area Configuration

```
! ==== HQ-CORE-SW1 ====
HQ-CORE-SW1(config)# router ospf 1
HQ-CORE-SW1(config-router)# router-id 172.20.255.11
HQ-CORE-SW1(config-router)# passive-interface default
HQ-CORE-SW1(config-router)# no passive-interface Port-channel1
HQ-CORE-SW1(config-router)# network 172.20.0.0 0.0.255.255 area 0
HQ-CORE-SW1(config-router)# area 0 authentication message-digest
HQ-CORE-SW1(config-router)# exit
HQ-CORE-SW1(config)# interface Port-channel1
HQ-CORE-SW1(config-if)# ip ospf message-digest-key 1 md5 S1cOspfKey!
HQ-CORE-SW1(config-if)# exit
!
! ==== IPv6: OSPFv3 คู่ขนาน (Part 27) ====
HQ-CORE-SW1(config)# interface Vlan110
HQ-CORE-SW1(config-if)# ipv6 ospf 1 area 0
HQ-CORE-SW1(config-if)# exit
HQ-CORE-SW1(config)# ipv6 router ospf 1
HQ-CORE-SW1(config-router)# router-id 172.20.255.11
HQ-CORE-SW1(config-router)# exit

! ==== HQ-RTR1: OSPF บน Loopback0 + Tunnel0 (DMVPN, ดู Step 985) ====
HQ-RTR1(config)# router ospf 1
HQ-RTR1(config-router)# router-id 172.20.255.1
HQ-RTR1(config-router)# network 172.20.253.0 0.0.0.255 area 0
HQ-RTR1(config-router)# network 172.20.255.1 0.0.0.0 area 0
HQ-RTR1(config-router)# network 172.20.254.0 0.0.0.3 area 0
```

### 984.3 — HSRP ที่ HQ-CORE-SW1/2 (Load-Distribute Active Role ระหว่าง 2 ตัว — เทคนิคจาก Part 35)

```
! ==== HQ-CORE-SW1: Active สำหรับ VLAN110/120/130/199 ====
HQ-CORE-SW1(config)# interface Vlan110
HQ-CORE-SW1(config-if)# standby version 2
HQ-CORE-SW1(config-if)# standby 110 ip 172.20.10.1
HQ-CORE-SW1(config-if)# standby 110 priority 150
HQ-CORE-SW1(config-if)# standby 110 preempt delay minimum 60
HQ-CORE-SW1(config-if)# standby 110 track Port-channel1 decrement 60
HQ-CORE-SW1(config-if)# exit
! (ทำซ้ำ Priority 150 สำหรับ VLAN 120,130,199 บน CORE-SW1)

! ==== HQ-CORE-SW2: Active สำหรับ VLAN140/141/150 (กระจายภาระ) ====
HQ-CORE-SW2(config)# interface Vlan140
HQ-CORE-SW2(config-if)# standby version 2
HQ-CORE-SW2(config-if)# standby 140 ip 172.20.40.1
HQ-CORE-SW2(config-if)# standby 140 priority 150
HQ-CORE-SW2(config-if)# standby 140 preempt delay minimum 60
HQ-CORE-SW2(config-if)# standby 140 track Port-channel1 decrement 60
HQ-CORE-SW2(config-if)# exit
! (ทำซ้ำ Priority 150 สำหรับ VLAN 141,150 บน CORE-SW2 — และ Priority 100 กลับด้านบน CORE-SW1 สำหรับ VLAN เดียวกัน)
```

> **เหตุผลของ `preempt delay minimum 60`**: เรียนจากบทเรียนราคาแพงของ Part 25 Step 246
> (Transient Black Hole ที่ HSRP Preempt เร็วกว่า OSPF Convergence) — ใส่ตั้งแต่ Day 1 ไม่ต้องรอให้
> เกิดปัญหาจริงก่อนแล้วมาแก้ทีหลังแบบใน Part 25

### 984.4 — Branch: ไม่มี HSRP (ยอมรับความเสี่ยงอย่างมีเหตุผล)

Branch มี L3 Switch แค่ตัวเดียว (`BR-SW1`) — **ไม่มี FHRP ให้ทำ** เพราะไม่มี Switch ตัวที่สองให้
Standby ด้วย นี่คือความเสี่ยงที่ **รับรู้และยอมรับไว้ในเอกสารโครงการอย่างชัดเจน** (ไม่ใช่ความ
ผิดพลาดที่มองไม่เห็น) โดยมี Mitigation ดังนี้: (1) สัญญา Maintenance Contract แบบ 4-Hour
Hardware Replacement กับ Cisco/Partner (2) มี Switch สำรอง (Cold Spare) วางไว้ที่ Warehouse
เอง (3) แผนปีที่ 2 (Growth Plan) จะพิจารณาเพิ่ม Switch ตัวที่สองที่ Branch เมื่อ Headcount โตถึง
80+ คนตามที่ระบุใน Step 981

---

## Step 985 — Phase 4: WAN & Remote Site — DMVPN Hub-Spoke + AWS Site-to-Site VPN ด้วย BGP

> Phase นี้ใช้ทักษะจาก **Part 19 (WAN เบื้องต้น), Part 29-30 (BGP), Part 51 (GRE/IPsec), Part
> 52-53 (DMVPN), Part 93 (Cloud Connectivity)**

### 985.1 — DMVPN Hub Configuration (HQ-RTR1)

```
HQ-RTR1(config)# crypto isakmp policy 10
HQ-RTR1(config-isakmp)# encryption aes 256
HQ-RTR1(config-isakmp)# hash sha256
HQ-RTR1(config-isakmp)# authentication pre-share
HQ-RTR1(config-isakmp)# group 14
HQ-RTR1(config-isakmp)# exit
HQ-RTR1(config)# crypto isakmp key S1cDmvpnPSK! address 0.0.0.0 0.0.0.0
!
HQ-RTR1(config)# crypto ipsec transform-set SIC-TSET esp-aes 256 esp-sha256-hmac
HQ-RTR1(cfg-crypto-trans)# mode transport
HQ-RTR1(cfg-crypto-trans)# exit
HQ-RTR1(config)# crypto ipsec profile SIC-DMVPN-PROFILE
HQ-RTR1(ipsec-profile)# set transform-set SIC-TSET
HQ-RTR1(ipsec-profile)# exit
!
HQ-RTR1(config)# interface Tunnel0
HQ-RTR1(config-if)# description ** DMVPN Hub - Phase 3, mGRE **
HQ-RTR1(config-if)# ip address 172.20.253.1 255.255.255.0
HQ-RTR1(config-if)# no ip redirects
HQ-RTR1(config-if)# ip mtu 1400
HQ-RTR1(config-if)# ip nhrp authentication S1cNhrpKey
HQ-RTR1(config-if)# ip nhrp network-id 99
HQ-RTR1(config-if)# ip nhrp redirect
HQ-RTR1(config-if)# ip ospf network point-to-multipoint
HQ-RTR1(config-if)# tunnel source GigabitEthernet0/0/0
HQ-RTR1(config-if)# tunnel mode gre multipoint
HQ-RTR1(config-if)# tunnel key 2026
HQ-RTR1(config-if)# tunnel protection ipsec profile SIC-DMVPN-PROFILE
HQ-RTR1(config-if)# no shutdown
```

### 985.2 — DMVPN Spoke Configuration (BR-RTR1) — พร้อมขยาย Spoke ที่ 2 ปีถัดไปโดยไม่แก้ Hub

```
BR-RTR1(config)# crypto isakmp policy 10
BR-RTR1(config-isakmp)# encryption aes 256
BR-RTR1(config-isakmp)# hash sha256
BR-RTR1(config-isakmp)# authentication pre-share
BR-RTR1(config-isakmp)# group 14
BR-RTR1(config-isakmp)# exit
BR-RTR1(config)# crypto isakmp key S1cDmvpnPSK! address 0.0.0.0 0.0.0.0
BR-RTR1(config)# crypto ipsec transform-set SIC-TSET esp-aes 256 esp-sha256-hmac
BR-RTR1(cfg-crypto-trans)# mode transport
BR-RTR1(cfg-crypto-trans)# exit
BR-RTR1(config)# crypto ipsec profile SIC-DMVPN-PROFILE
BR-RTR1(ipsec-profile)# set transform-set SIC-TSET
BR-RTR1(ipsec-profile)# exit
!
BR-RTR1(config)# interface Tunnel0
BR-RTR1(config-if)# description ** DMVPN Spoke to HQ-RTR1 (Hub) **
BR-RTR1(config-if)# ip address 172.20.253.2 255.255.255.0
BR-RTR1(config-if)# ip mtu 1400
BR-RTR1(config-if)# ip nhrp authentication S1cNhrpKey
BR-RTR1(config-if)# ip nhrp network-id 99
BR-RTR1(config-if)# ip nhrp nhs 172.20.253.1 nbma 198.51.100.2
BR-RTR1(config-if)# ip nhrp shortcut
BR-RTR1(config-if)# ip ospf network point-to-multipoint
BR-RTR1(config-if)# tunnel source GigabitEthernet0/0/0
BR-RTR1(config-if)# tunnel mode gre multipoint
BR-RTR1(config-if)# tunnel key 2026
BR-RTR1(config-if)# tunnel protection ipsec profile SIC-DMVPN-PROFILE
BR-RTR1(config-if)# no shutdown
!
BR-RTR1(config)# router ospf 1
BR-RTR1(config-router)# router-id 172.21.255.1
BR-RTR1(config-router)# network 172.20.253.0 0.0.0.255 area 0
BR-RTR1(config-router)# network 172.21.254.0 0.0.0.3 area 0
BR-RTR1(config-router)# network 172.21.255.1 0.0.0.0 area 0
```

### 985.3 — ISP Failover ที่ HQ-RTR1 ด้วย IP SLA + Track (Part 9, 19, 32)

```
HQ-RTR1(config)# ip sla 1
HQ-RTR1(config-ip-sla)# icmp-echo 198.51.100.1 source-interface GigabitEthernet0/0/0
HQ-RTR1(config-ip-sla-echo)# frequency 5
HQ-RTR1(config-ip-sla-echo)# exit
HQ-RTR1(config)# ip sla schedule 1 life forever start-time now
HQ-RTR1(config)# track 1 ip sla 1
HQ-RTR1(config-track)# exit
HQ-RTR1(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.1 track 1
HQ-RTR1(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.5 200
HQ-RTR1(config)# ip nat inside source list SIC-NAT-ACL interface GigabitEthernet0/0/0 overload
```

### 985.4 — AWS Site-to-Site VPN ด้วย eBGP (Part 29-30, 93)

```
! ==== VTI Tunnel 1/2 ไปยัง AWS Virtual Private Gateway (2 Tunnel ตาม Best Practice ของ AWS) ====
HQ-RTR1(config)# interface Tunnel1
HQ-RTR1(config-if)# ip address 169.254.10.2 255.255.255.252
HQ-RTR1(config-if)# tunnel source GigabitEthernet0/0/0
HQ-RTR1(config-if)# tunnel destination 52.74.xx.xx1
HQ-RTR1(config-if)# tunnel mode ipsec ipv4
HQ-RTR1(config-if)# tunnel protection ipsec profile AWS-VPN-PROFILE
HQ-RTR1(config-if)# no shutdown
HQ-RTR1(config)# interface Tunnel2
HQ-RTR1(config-if)# ip address 169.254.11.2 255.255.255.252
HQ-RTR1(config-if)# tunnel source GigabitEthernet0/0/0
HQ-RTR1(config-if)# tunnel destination 52.74.xx.xx2
HQ-RTR1(config-if)# tunnel mode ipsec ipv4
HQ-RTR1(config-if)# tunnel protection ipsec profile AWS-VPN-PROFILE
HQ-RTR1(config-if)# no shutdown
!
! ==== eBGP: SIC AS 65010 <-> AWS VGW ASN 64512 (ค่า Default ของ AWS) ====
HQ-RTR1(config)# router bgp 65010
HQ-RTR1(config-router)# neighbor 169.254.10.1 remote-as 64512
HQ-RTR1(config-router)# neighbor 169.254.11.1 remote-as 64512
HQ-RTR1(config-router)# network 172.20.0.0 mask 255.255.0.0
HQ-RTR1(config-router)# exit
!
! ==== Redistribute BGP -> OSPF พร้อม Tag ป้องกัน Loop (Part 31 skill) ====
HQ-RTR1(config)# route-map BGP-TO-OSPF permit 10
HQ-RTR1(config-route-map)# set tag 65010
HQ-RTR1(config-route-map)# exit
HQ-RTR1(config)# router ospf 1
HQ-RTR1(config-router)# redistribute bgp 65010 subnets route-map BGP-TO-OSPF
```

> **ทำไมไม่ Redistribute OSPF เข้า BGP กลับ (ทิศทางเดียว)**: HQ ไม่ต้องการให้ AWS เห็น Route
> ภายในทั้งหมดของบริษัท (Server, Call Center) เกินความจำเป็น — ประกาศแค่ Summary `172.20.0.0/16`
> เดียวพอ (Part 12 Summarization) ตาม Principle "Least Information Disclosure" ซึ่งเป็นหลักการ
> เดียวกับ Zero Trust Data Pillar ที่ Part 94 สอนไว้

---

## Step 986 — Phase 5: Security Hardening — Security Stack ที่ Scale ให้พอดีกับงบประมาณ SMB

> Phase นี้ใช้ทักษะจาก **Part 14 (ACL), 21 (Switch Security), 22 (AAA), 37 (Wireless Security),
> 43-46 (ZBFW/CoPP), 78 (Umbrella), 94 (Zero Trust)** — แต่ **Scale ลง** อย่างมีเหตุผลทุกจุด

### 986.1 — Security Right-Sizing Decision Table

| Feature ระดับ CCIE ที่สอนใน Part 43-46/78/94 | ใช้ที่ SIC หรือไม่ | เหตุผล |
|---|---|---|
| Cisco ISE (Identity Services Engine) | **ไม่ใช้** | License + Server เพิ่มไม่คุ้มกับ Endpoint <300 ตัว — ใช้ Windows NPS (RADIUS) ที่มีอยู่แล้วจาก AD/M365 แทน |
| TrustSec/SGT (Micro-Segmentation) | **ไม่ใช้** | ต้องพึ่ง ISE — Deferred ไปพร้อมกับ ISE |
| Firepower Threat Defense (FTD) เต็มรูปแบบ + IPS Signature | **ไม่ใช้เต็มรูปแบบ** — ใช้ ASA พื้นฐาน + Umbrella แทน IPS Appliance | ASA เพียงพอสำหรับ Stateful Inspection ระดับนี้ Umbrella (DNS-layer) ให้ Threat Intelligence ในราคา SaaS ที่ต่ำกว่า IPS Appliance มาก |
| Zone-Based Firewall (ZBFW) เต็มรูปแบบทุกจุด | **ใช้เฉพาะที่ Branch** (ไม่มี ASA ที่ Branch) | HQ มี ASA ทำหน้าที่แทนแล้ว — ไม่ต้องซ้ำ 2 ชั้นทุก Site |
| 802.1X ทุก Wired Port | **ใช้เฉพาะ VLAN150 (Call Center) + Corp Wi-Fi** | Port Security + DHCP Snooping/DAI เพียงพอสำหรับ Office VLAN ทั่วไปที่ความเสี่ยงต่ำกว่า |
| Control Plane Policing (CoPP) | **ใช้แบบง่าย** (Rate พื้นฐานป้องกัน DoS เบื้องต้น) | ไม่ต้อง Baseline ซับซ้อนแบบ Part 46 เพราะปริมาณอุปกรณ์/Traffic น้อยกว่า Lab เดิมมาก |
| Cisco Umbrella (DNS Security) | **ใช้เต็มรูปแบบ** | SaaS ราคาไม่แพง ROI สูงมากสำหรับทีม IT เล็ก — ป้องกัน Malware/Phishing โดยไม่ต้องมี Appliance เพิ่ม |

### 986.2 — HQ-FW1 (ASA 5516-X): Perimeter Firewall

```
HQ-FW1(config)# hostname HQ-FW1
HQ-FW1(config)# interface GigabitEthernet1/1
HQ-FW1(config-if)# nameif outside
HQ-FW1(config-if)# security-level 0
HQ-FW1(config-if)# ip address 172.20.254.2 255.255.255.252
HQ-FW1(config-if)# no shutdown
HQ-FW1(config)# interface GigabitEthernet1/2
HQ-FW1(config-if)# nameif inside
HQ-FW1(config-if)# security-level 100
HQ-FW1(config-if)# ip address 172.20.254.5 255.255.255.252
HQ-FW1(config-if)# no shutdown
!
HQ-FW1(config)# route outside 0.0.0.0 0.0.0.0 172.20.254.1
HQ-FW1(config)# route inside 172.20.0.0 255.255.0.0 172.20.254.6
!
HQ-FW1(config)# object network SIC-INTERNAL
HQ-FW1(config-network-object)# subnet 172.20.0.0 255.255.0.0
HQ-FW1(config)# object network CALLCENTER-VLAN
HQ-FW1(config-network-object)# subnet 172.20.50.0 255.255.255.0
!
! ==== ACL: Guest Wi-Fi ออก Internet ได้เท่านั้น ไม่แตะ Internal ====
HQ-FW1(config)# access-list OUTSIDE-IN extended deny ip any any log
HQ-FW1(config)# access-group OUTSIDE-IN in interface outside
!
! ==== AAA/Logging/NTP ====
HQ-FW1(config)# aaa authentication ssh console LOCAL
HQ-FW1(config)# username sic-admin password S1cAdm1n!Str0ng privilege 15
HQ-FW1(config)# logging host inside 172.20.30.20
HQ-FW1(config)# logging enable
HQ-FW1(config)# ntp server 172.20.255.1
```

### 986.3 — Access-Layer Baseline (ทุก Access Switch, Part 21)

```
! ==== Port Security + DHCP Snooping + DAI — Template มาตรฐานทุก Access Port ====
HQ-ACC-SW1(config)# interface range GigabitEthernet1/0/1-40
HQ-ACC-SW1(config-if-range)# switchport port-security
HQ-ACC-SW1(config-if-range)# switchport port-security maximum 2
HQ-ACC-SW1(config-if-range)# switchport port-security violation restrict
HQ-ACC-SW1(config-if-range)# switchport port-security mac-address sticky
HQ-ACC-SW1(config-if-range)# exit
!
HQ-ACC-SW1(config)# ip dhcp snooping vlan 110,120,130,140,141,150,199
HQ-ACC-SW1(config)# interface GigabitEthernet1/0/48
HQ-ACC-SW1(config-if)# description ** Uplink to HQ-CORE-SW1 - Trusted **
HQ-ACC-SW1(config-if)# ip dhcp snooping trust
HQ-ACC-SW1(config-if)# ip arp inspection trust
HQ-ACC-SW1(config-if)# exit
HQ-ACC-SW1(config)# ip arp inspection vlan 110,120,130,140,141,150,199
```

### 986.4 — 802.1X เฉพาะ VLAN150 (Call Center — CDE-Adjacent) และ Corp Wi-Fi

```
! ==== HQ-ACC-SW2 (Call Center Floor): 802.1X + MAB Fallback ====
HQ-ACC-SW2(config)# aaa new-model
HQ-ACC-SW2(config)# radius server SIC-NPS
HQ-ACC-SW2(config-radius-server)# address ipv4 172.20.30.5 auth-port 1812 acct-port 1813
HQ-ACC-SW2(config-radius-server)# key S1cRadiusKey!
HQ-ACC-SW2(config-radius-server)# exit
HQ-ACC-SW2(config)# aaa authentication dot1x default group radius
HQ-ACC-SW2(config)# dot1x system-auth-control
HQ-ACC-SW2(config)# interface range GigabitEthernet1/0/1-40
HQ-ACC-SW2(config-if-range)# switchport mode access
HQ-ACC-SW2(config-if-range)# switchport access vlan 150
HQ-ACC-SW2(config-if-range)# authentication port-control auto
HQ-ACC-SW2(config-if-range)# dot1x pae authenticator
HQ-ACC-SW2(config-if-range)# mab
HQ-ACC-SW2(config-if-range)# authentication event fail action next-method
HQ-ACC-SW2(config-if-range)# authentication event server dead action authorize vlan 199
```

> **Critical Authentication (`server dead action authorize vlan 199`)**: บทเรียนตรงจาก Part 61/
> Part 70 Step 695 คำถามที่ 2 — ถ้า NPS ล่ม ไม่ให้ Call Center Agent ทำงานไม่ได้ทั้งหมด แต่ให้เข้า
> VLAN ฉุกเฉิน (MGMT, มี ACL จำกัดเข้มงวดกว่า) ชั่วคราวจนกว่า NPS กลับมา

### 986.5 — Zero Trust ที่ Scale (Part 94) — Macro-Segmentation แทน Micro-Segmentation

```
! ==== ACL บน HQ-CORE-SW1: Guest Wi-Fi ห้ามคุยกับ VLAN ภายในใดๆเลย ====
HQ-CORE-SW1(config)# ip access-list extended GUEST-ISOLATION
HQ-CORE-SW1(config-ext-nacl)# deny ip 172.20.41.0 0.0.0.255 172.20.0.0 0.0.255.255
HQ-CORE-SW1(config-ext-nacl)# permit ip 172.20.41.0 0.0.0.255 any
HQ-CORE-SW1(config-ext-nacl)# exit
HQ-CORE-SW1(config)# interface Vlan141
HQ-CORE-SW1(config-if)# ip access-group GUEST-ISOLATION in
!
! ==== ACL บน BR-SW1: CCTV/IoT VLAN ออก Internet ได้เฉพาะปลายทาง NVR Cloud Backup เท่านั้น ====
BR-SW1(config)# ip access-list extended CCTV-RESTRICTED
BR-SW1(config-ext-nacl)# permit tcp 172.21.50.0 0.0.0.255 host 52.xx.xx.xx eq 443
BR-SW1(config-ext-nacl)# deny ip 172.21.50.0 0.0.0.255 172.21.0.0 0.0.255.255 log
BR-SW1(config-ext-nacl)# deny ip any any log
BR-SW1(config-ext-nacl)# exit
BR-SW1(config)# interface Vlan250
BR-SW1(config-if)# ip access-group CCTV-RESTRICTED in
```

---

## Step 987 — Phase 6: Services & QoS — DHCP/DNS/NTP, QoS สำหรับ Call Center, และทำไมไม่ต้องมี Multicast

> Phase นี้ใช้ทักษะจาก **Part 15-17 (NAT/DHCP-DNS/NTP), Part 24/40 (QoS), Part 41-42 (Multicast)**

### 987.1 — DHCP: ใช้ IOS DHCP Pool บน Switch เอง (ไม่ซื้อ Windows DHCP Server เพิ่ม)

```
HQ-CORE-SW1(config)# ip dhcp excluded-address 172.20.10.1 172.20.10.20
HQ-CORE-SW1(config)# ip dhcp pool DATA-STAFF
HQ-CORE-SW1(dhcp-config)# network 172.20.10.0 255.255.255.0
HQ-CORE-SW1(dhcp-config)# default-router 172.20.10.1
HQ-CORE-SW1(dhcp-config)# dns-server 172.20.30.10 208.67.222.222
HQ-CORE-SW1(dhcp-config)# domain-name sic.co.th
HQ-CORE-SW1(dhcp-config)# lease 7
HQ-CORE-SW1(dhcp-config)# exit
! (ทำซ้ำ pool WIFI-CORP, WIFI-GUEST, CALLCENTER ตาม Subnet ของแต่ละ VLAN)
```

> **`208.67.222.222`**: DNS Anycast Resolver ของ Cisco Umbrella — แจกเป็น DNS Server สำรองให้
> Client ทุกตัวโดยอัตโนมัติผ่าน DHCP Option 6 นี่คือวิธี "ผูก" Security Control (Umbrella DNS
> Filtering จาก Step 986.1) เข้ากับ Infrastructure พื้นฐาน (DHCP) โดยไม่ต้องมี Agent ติดตั้งเพิ่ม
> ในเครื่อง Endpoint ทุกเครื่อง — ROI สูงมากสำหรับทีม IT เล็ก

### 987.2 — DNS/NTP/Syslog/SNMP

```
HQ-CORE-SW1(config)# ip name-server 172.20.30.10 208.67.222.222
HQ-CORE-SW1(config)# ntp server 172.20.255.1
HQ-CORE-SW1(config)# clock timezone ICT 7
HQ-CORE-SW1(config)# logging host 172.20.30.20
HQ-CORE-SW1(config)# logging buffered 16384
HQ-CORE-SW1(config)# snmp-server group SIC-NOC v3 priv
HQ-CORE-SW1(config)# snmp-server user noc-svc SIC-NOC v3 auth sha AuthP@ss123 priv aes 128 PrivP@ss123
!
! ==== HQ-RTR1 คือ NTP Server หลักของบริษัท (Stratum อ้างอิง NECTEC — NTP สาธารณะของไทย) ====
HQ-RTR1(config)# ntp server 203.185.69.60
HQ-RTR1(config)# ntp master 5
```

### 987.3 — QoS สำหรับ Call Center Softphone (40 ที่นั่ง — Business-Critical Traffic)

Reception มี Cisco IP Phone จริง (Trust ผ่าน CDP อัตโนมัติ) แต่ **Call Center 40 ที่นั่งใช้
Softphone บน PC** ซึ่ง OS/Application **ไม่น่าเชื่อถือพอที่จะ Trust DSCP Marking โดยตรง** —
ต้องใช้ Class-Map จับ Traffic Pattern (Port Range ของ UCaaS Provider) แล้ว Mark ใหม่ที่ Switch
เอง (Conditional Trust Boundary ตาม Part 40)

```
! ==== HQ-ACC-SW2: Classify + Mark RTP Traffic จาก UCaaS Softphone ====
HQ-ACC-SW2(config)# ip access-list extended UCAAS-RTP
HQ-ACC-SW2(config-ext-nacl)# permit udp 172.20.50.0 0.0.0.255 any range 16384 32767
HQ-ACC-SW2(config-ext-nacl)# exit
HQ-ACC-SW2(config)# class-map match-all UCAAS-VOICE
HQ-ACC-SW2(config-cmap)# match access-group name UCAAS-RTP
HQ-ACC-SW2(config-cmap)# exit
HQ-ACC-SW2(config)# policy-map CALLCENTER-INGRESS-MARK
HQ-ACC-SW2(config-pmap)# class UCAAS-VOICE
HQ-ACC-SW2(config-pmap-c)# set dscp ef
HQ-ACC-SW2(config-pmap-c)# police 128000
HQ-ACC-SW2(config-pmap-c)# exit
HQ-ACC-SW2(config)# interface range GigabitEthernet1/0/1-40
HQ-ACC-SW2(config-if-range)# service-policy input CALLCENTER-INGRESS-MARK
!
! ==== HQ-RTR1 (WAN Edge): LLQ ให้ EF ก่อน Traffic อื่นทุกชนิดตอนออก Internet ====
HQ-RTR1(config)# class-map match-all VOICE-EF
HQ-RTR1(config-cmap)# match dscp ef
HQ-RTR1(config-cmap)# exit
HQ-RTR1(config)# policy-map WAN-EDGE-QOS
HQ-RTR1(config-pmap)# class VOICE-EF
HQ-RTR1(config-pmap-c)# priority percent 20
HQ-RTR1(config-pmap-c)# class class-default
HQ-RTR1(config-pmap-c)# fair-queue
HQ-RTR1(config-pmap-c)# exit
HQ-RTR1(config)# interface GigabitEthernet0/0/0
HQ-RTR1(config-if)# service-policy output WAN-EDGE-QOS
```

### 987.4 — ทำไม SIC ไม่ต้องมี Multicast (Part 41-42)

| Use Case ที่ปกติต้องใช้ Multicast | มีที่ SIC หรือไม่ |
|---|---|
| Video Conferencing แบบ Multicast ภายในองค์กร | ไม่มี — ใช้ Microsoft Teams (Unicast ผ่าน Cloud) |
| IPTV/Digital Signage | ไม่มีในปีแรก |
| CCTV Feed ไปยัง NVR | **Unicast** (กล้อง IP แต่ละตัวส่งตรงไปยัง NVR โดยตรง ไม่ใช่ Multicast Group) |
| Financial Market Data Feed | ไม่เกี่ยวข้องกับธุรกิจ E-commerce |

> **สรุป**: การไม่ Deploy Multicast **ไม่ใช่การมองข้าม** แต่เป็นผลจากการตรวจสอบ Requirement
> จริงแล้วพบว่าไม่มี Use Case ใดต้องใช้เลย — ถ้าปีที่ 3 บริษัทเริ่มทำ Live Streaming สินค้า
> (Live Commerce) ภายในสำนักงานจริงจัง จะกลับมาพิจารณา PIM-SM ตาม Part 41-42 ทันที นี่คือ
> ตัวอย่างที่ชัดที่สุดของ "รู้ว่าเมื่อไหร่ไม่ควรใช้เทคโนโลยีที่เรียนมา"

---

## Step 988 — Phase 7: Automation & Monitoring — ชุดเครื่องมือที่ Scale ให้พอดีกับทีม IT 2 คน

> Phase นี้ใช้ทักษะจาก **Part 23 (Automation พื้นฐาน), Part 47-48 (Python/Ansible), Part 49
> (NETCONF/RESTCONF), Part 54 (Assurance/Telemetry), Part 79-80 (Full CI/CD, Model-Driven
> Telemetry)** — เลือกเฉพาะส่วนที่ "คุ้มค่าแรงตั้งระบบ" สำหรับ Fleet 11 อุปกรณ์

### 988.1 — เหตุผลที่ไม่ทำ Full CI/CD Pipeline แบบ Part 79

| องค์ประกอบของ Part 79 Pipeline | ใช้ที่ SIC หรือไม่ | เหตุผล |
|---|---|---|
| Git Repository เก็บ Config (Source of Truth) | **ใช้** | ROI สูงมาก ตั้งค่าง่าย ไม่มีข้อเสีย |
| Pre-commit Validation (Batfish/pyATS) | ไม่ใช้ในปีแรก | ต้องมีทีม/เวลาเขียน Test Case — เกินกำลังทีม 2 คนตอนนี้ |
| Digital Twin/Test Lab แยก | ไม่ใช้ | ค่า Infra เพิ่มไม่คุ้มกับ Fleet 11 อุปกรณ์ |
| Jenkins Pipeline อัตโนมัติเต็มรูปแบบ | ไม่ใช้ | Ansible รันตรงจาก Laptop/Jump Host ของ Engineer พอเพียง |
| Canary/Staged Rollout อัตโนมัติ | ไม่ใช้ (ทำ Manual Staged Rollout แทน — ดู Step 989) | อุปกรณ์น้อยเกินกว่าจะคุ้มค่าเขียน Automation Rollback |
| Automated Rollback | **ทำแบบง่าย** (`archive config` + Manual Restore) | เพียงพอสำหรับ Fleet ขนาดนี้ |

### 988.2 — Config Backup อัตโนมัติด้วย Python (Netmiko) — Part 47

```python
#!/usr/bin/env python3
# sic-config-backup.py — รันทุกวันผ่าน cron บน Jump Host ของทีม IT
from netmiko import ConnectHandler
from datetime import date
import os

DEVICES = [
    {"host": "172.20.99.11", "device_type": "cisco_ios", "hostname": "HQ-CORE-SW1"},
    {"host": "172.20.99.12", "device_type": "cisco_ios", "hostname": "HQ-CORE-SW2"},
    {"host": "172.20.99.1",  "device_type": "cisco_ios", "hostname": "HQ-RTR1"},
    {"host": "172.20.99.2",  "device_type": "cisco_asa", "hostname": "HQ-FW1"},
    {"host": "172.21.99.1",  "device_type": "cisco_ios", "hostname": "BR-RTR1"},
    {"host": "172.21.99.11", "device_type": "cisco_ios", "hostname": "BR-SW1"},
]
USERNAME, PASSWORD = "sic-automation", os.environ["SIC_AUTOMATION_PASS"]
BACKUP_DIR = f"/opt/sic-backups/{date.today().isoformat()}"
os.makedirs(BACKUP_DIR, exist_ok=True)

for dev in DEVICES:
    conn = ConnectHandler(username=USERNAME, password=PASSWORD, **{k: v for k, v in dev.items() if k != "hostname"})
    output = conn.send_command("show running-config")
    with open(f"{BACKUP_DIR}/{dev['hostname']}.cfg", "w") as f:
        f.write(output)
    conn.disconnect()
    print(f"[OK] Backed up {dev['hostname']}")
```

### 988.3 — Ansible Playbook: Push VLAN ใหม่ให้ทุก Access Switch พร้อมกัน (Part 48)

```yaml
---
- name: Deploy new VLAN to all SIC access switches
  hosts: sic_access_switches
  gather_facts: no
  tasks:
    - name: Create VLAN 160 (future DMZ segment)
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 160
            name: DMZ-RESERVED
        state: merged
    - name: Save running-config
      cisco.ios.ios_config:
        save_when: always
```

### 988.4 — Monitoring: LibreNMS (Open-Source) แทน DNA Center

| ความสามารถ | DNA Center (Part 50) | LibreNMS + Syslog Server (เลือกใช้จริง) |
|---|---|---|
| ค่า License | สูง (ต่อจำนวนอุปกรณ์) | ฟรี (Open-Source, ติดตั้งเองบน VM เล็ก) |
| SNMP/Syslog Monitoring | มี | **มี** (เพียงพอสำหรับ Fleet 11 อุปกรณ์) |
| Assurance/AI Network Insight | มี (Advanced) | ไม่มี — ใช้ Manual Threshold Alert แทน |
| Automation Provisioning ผ่าน GUI | มี | ไม่มี — ใช้ Ansible (Step 988.3) แทน |
| **เหมาะกับ SIC** | Overkill สำหรับ 11 อุปกรณ์ | **เพียงพอ** — Fleet ขนาดนี้ยังไม่ถึงจุดคุ้มทุนของ DNA Center |

```
! ==== SNMP + Syslog ให้ LibreNMS (172.20.30.20) เก็บข้อมูลจากทุกอุปกรณ์ — เหมือน 987.2 ====
HQ-RTR1(config)# snmp-server community S1cR3adOnly! RO
HQ-RTR1(config)# snmp-server host 172.20.30.20 version 2c S1cR3adOnly!
HQ-RTR1(config)# logging host 172.20.30.20
```

> **แผนอนาคต**: เมื่อ Fleet โตถึง Growth Plan ปีที่ 3 (350+ คน, Site เพิ่ม) ให้ทวน Part 50 (DNA
> Center) และ Part 79 (Full CI/CD) ใหม่อีกครั้ง — จุดตัดสินใจคือ **จำนวนอุปกรณ์ที่ทีม IT ต้องดูแล
> เกิน ~30-40 ตัว หรือความถี่การเปลี่ยน Config เกินสัปดาห์ละหลายครั้ง** ณ จุดนั้น Automation
> แบบเบาที่ใช้อยู่จะเริ่มไม่คุ้มแรงคนเมื่อเทียบกับการลงทุน Platform เต็มรูปแบบ

---

## Step 989 — Phase 8: Testing & Go-Live — Pre-Production Checklist, Cutover Plan, Post-Go-Live Validation

> Phase นี้ใช้ทักษะจาก **Part 91 (Advanced Troubleshooting Methodology)** และแนวคิด **Staged
> Rollout** จาก **Part 79 Step 786** — ประยุกต์กับการ Go-Live ทางกายภาพจริง (ไม่ใช่ Push Config
> อัตโนมัติ)

### 989.1 — Pre-Production Test Checklist (ก่อนวัน Go-Live)

```
LAYER 1-2 (Physical/Data Link)
  [ ] ทุก Uplink/Trunk Up/Up, ไม่มี Interface Error Counter (show interfaces | include error)
  [ ] show spanning-tree summary — ไม่มี VLAN ค้างที่ Blocking ผิดที่ที่คาด
  [ ] show etherchannel summary — Po1 ที่ HQ ขึ้น (SU) ครบ 2 Member

LAYER 3 (Routing)
  [ ] show ip ospf neighbor — ครบทุกคู่ FULL (HQ-RTR1, CORE-SW1/2, BR-RTR1, BR-SW1)
  [ ] show ip route — เห็น Route ของทั้ง 2 Site ในตารางของกันและกัน
  [ ] show dmvpn — Tunnel HQ<->Branch ขึ้น NHRP Mapping สมบูรณ์
  [ ] show ip bgp summary (HQ-RTR1) — Neighbor AWS ทั้ง 2 Tunnel State = Established
  [ ] Ping/Traceroute ข้าม Site และไป AWS Test Instance ให้ผ่านทุกเส้นทาง

REDUNDANCY
  [ ] show standby brief — HSRP ทุก VLAN มี Active/Standby ถูกต้องตามแผน Load-Distribution
  [ ] ทดสอบ Fail Primary ISP ที่ HQ-RTR1 (shutdown Gi0/0/0 ชั่วคราว) — ยืนยัน Failover ไป ISP2 ภายใน 10 วินาที
  [ ] ทดสอบดึงสาย Po1 Member 1 เส้น — Traffic ยังไหลผ่าน Member ที่เหลือโดยไม่ Drop

SECURITY
  [ ] show ip access-lists — Match Counter ของ GUEST-ISOLATION/CCTV-RESTRICTED ทำงานถูกทิศทาง
  [ ] ทดสอบ Login ผ่าน 802.1X ที่ VLAN150 ด้วย Test Account จริง — ต้องได้ VLAN 150 ไม่ใช่ VLAN อื่น
  [ ] ทดสอบ Critical Authentication: ปิด NPS ชั่วคราว — Call Center ต้องได้ VLAN 199 (Fallback) ไม่ใช่ค้าง Unauthorized
  [ ] ยืนยัน Guest Wi-Fi เข้า Internal Subnet ใดๆไม่ได้เลย (Ping จาก Guest ไป Server VLAN130 ต้อง Fail)

SERVICES/QoS
  [ ] DHCP ทุก VLAN แจก IP + DNS (208.67.222.222) ถูกต้อง, Lease Time ตรงตาม Plan
  [ ] ทดสอบ Softphone Call จริงจาก Call Center ระหว่างโหลด Traffic อื่นสูง — ยืนยัน DSCP EF ไม่ถูก Drop
  [ ] NTP Sync ตรงกันทุกอุปกรณ์ (show ntp status)

AUTOMATION/MONITORING
  [ ] รัน sic-config-backup.py แบบ Manual 1 ครั้ง — ยืนยัน Backup ครบทุกอุปกรณ์
  [ ] LibreNMS เห็นทุกอุปกรณ์ Up และรับ Syslog เข้ามาจริง
```

### 989.2 — Cutover Plan (Go-Live Sequencing) — Staged Rollout แบบ Part 79 ประยุกต์กับกายภาพ

```
CUTOVER PLAN — Siam Innovation Corp (Greenfield, ไม่มี Legacy ให้ Migrate)

STAGE 1 (สัปดาห์ -2): HQ Infrastructure Bring-up ตาม Phase 2-7 (Step 983-988) ในเวลาว่าง
         (ไม่มี User จริงใช้งานยัง — เสี่ยงต่ำที่สุด, แก้ Bug ได้เต็มที่)

STAGE 2 (สัปดาห์ -1): ย้าย Pilot Group 15 คน (ทีม IT + Engineering) เข้า HQ Network ก่อน
         — เลือกกลุ่มที่ Debug เก่งที่สุดเป็น "Canary" ตามหลัก Part 79 Step 786 แต่ทำแบบ Manual
         [ ] เก็บ Feedback ต่อเนื่อง 5 วันทำงาน ก่อนขยาย

STAGE 3 (วัน Go-Live HQ): ย้าย User ที่เหลือทั้งหมด 135 คนของ HQ พร้อมกัน
         — เลือกทำ**เช้าวันจันทร์** (Low-Risk Window ตาม Part 91 — มีเวลาทั้งสัปดาห์แก้ปัญหา
         ก่อนวันหยุด ไม่ใช่ทำวันศุกร์บ่ายที่ไม่มีใครช่วยแก้ถ้าพัง)
         [ ] ทีม IT Standby เต็มเวลาวันนั้น, มี War-Room ชั่วคราว

STAGE 4 (สัปดาห์ +1): Branch (Chonburi) Bring-up ตาม Phase 2-7 — **ก่อนแคมเปญ Non-สำคัญ**
         (หลีกเลี่ยงการ Go-Live ก่อน Big Sale Campaign ที่คลังสินค้าต้องทำงานเต็มกำลัง — เลือก
         ช่วงที่ Order Volume ต่ำที่สุดของเดือนเป็น Go-Live Window)

STAGE 5 (วัน Go-Live Branch): ย้าย Warehouse Operations 35 คน + Regional Office 8 คน + CCTV/NVR
         [ ] ทดสอบ WMS Scanner จริงกับ Order จริงปริมาณน้อยก่อน (Soft Launch ครึ่งวัน)
         [ ] ขยายเป็น Full Operations เมื่อยืนยัน Scanner→WMS→AWS Latency ปกติ

ROLLBACK PLAN ทุก Stage: เก็บ Co-working Space Internet เดิมไว้เป็น Fallback อีก 2 สัปดาห์
         หลัง Go-Live เต็มรูปแบบ (ค่าใช้จ่ายน้อยกว่าความเสี่ยง Business หยุดชะงัก)
```

### 989.3 — Post-Go-Live Validation (7 วันแรกหลัง Go-Live เต็มรูปแบบ)

| วัน | สิ่งที่ต้องตรวจสอบ |
|---|---|
| Day 1-2 | Monitor LibreNMS ต่อเนื่อง — CPU/Memory/Interface Error ของทุกอุปกรณ์ต้องอยู่ใน Baseline ปกติ |
| Day 3 | ตรวจ `show ip ospf neighbor`/`show dmvpn`/BGP ทุกอุปกรณ์ซ้ำอีกรอบ — ยืนยัน Adjacency เสถียร ไม่ Flap |
| Day 4 | สัมภาษณ์ Call Center Agent สุ่ม 5 คน — คุณภาพเสียง Softphone เป็นอย่างไรช่วง Peak Hour |
| Day 5 | ตรวจ `show ip access-lists` Match Counter ของทุก Security ACL อีกครั้ง — หา False Positive/Negative |
| Day 6-7 | Full Failover Drill จริง (ตัด ISP1 จริง ไม่ใช่ shutdown จำลอง) — ยืนยัน Business ไม่หยุดชะงัก |
| End of Week 1 | จัด Retrospective Meeting กับผู้บริหาร SIC — เปรียบเทียบผลจริงกับ Requirement ใน Step 981 ทุกข้อ |

---

## Step 990 — Full Lab: Running-Config ครบทุกอุปกรณ์ + Retrospective เชื่อมกลับทุก Part

### 990.1 — Running-Config: HQ-RTR1 (ISR4331, WAN Edge — ฉบับสมบูรณ์)

```
hostname HQ-RTR1
no ip domain-lookup
ip domain-name sic.co.th
enable secret S1cCap5tone!2026
service password-encryption
username sic-admin privilege 15 secret S1cAdm1n!Str0ng
username sic-automation privilege 15 secret S1cAut0Bkp!2026
!
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0
line console 0
 exec-timeout 10 0
 logging synchronous
!
interface Loopback0
 description ** Router-ID / NTP Master **
 ip address 172.20.255.1 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** ISP1 - Primary Fiber **
 ip address 198.51.100.2 255.255.255.252
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** ISP2 - Backup Fiber **
 ip address 198.51.100.6 255.255.255.252
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** to HQ-FW1 outside **
 ip address 172.20.254.1 255.255.255.252
 ip nat inside
 no shutdown
!
interface Tunnel0
 description ** DMVPN Hub - Phase 3 mGRE **
 ip address 172.20.253.1 255.255.255.0
 ip mtu 1400
 ip nhrp authentication S1cNhrpKey
 ip nhrp network-id 99
 ip nhrp redirect
 ip ospf network point-to-multipoint
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 2026
 tunnel protection ipsec profile SIC-DMVPN-PROFILE
 no shutdown
!
interface Tunnel1
 description ** AWS VPN Tunnel 1 **
 ip address 169.254.10.2 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 52.74.10.10
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-VPN-PROFILE
 no shutdown
!
interface Tunnel2
 description ** AWS VPN Tunnel 2 (Redundant) **
 ip address 169.254.11.2 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 52.74.10.11
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-VPN-PROFILE
 no shutdown
!
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key S1cDmvpnPSK! address 0.0.0.0 0.0.0.0
crypto ipsec transform-set SIC-TSET esp-aes 256 esp-sha256-hmac
 mode transport
crypto ipsec profile SIC-DMVPN-PROFILE
 set transform-set SIC-TSET
crypto ipsec profile AWS-VPN-PROFILE
 set transform-set SIC-TSET
!
ip access-list extended SIC-NAT-ACL
 permit ip 172.20.0.0 0.0.255.255 any
!
ip nat inside source list SIC-NAT-ACL interface GigabitEthernet0/0/0 overload
!
ip sla 1
 icmp-echo 198.51.100.1 source-interface GigabitEthernet0/0/0
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1
!
ip route 0.0.0.0 0.0.0.0 198.51.100.1 track 1
ip route 0.0.0.0 0.0.0.0 198.51.100.5 200
!
router ospf 1
 router-id 172.20.255.1
 redistribute bgp 65010 subnets route-map BGP-TO-OSPF
 network 172.20.253.0 0.0.0.255 area 0
 network 172.20.254.0 0.0.0.3 area 0
 network 172.20.255.1 0.0.0.0 area 0
!
router bgp 65010
 neighbor 169.254.10.1 remote-as 64512
 neighbor 169.254.11.1 remote-as 64512
 network 172.20.0.0 mask 255.255.0.0
!
route-map BGP-TO-OSPF permit 10
 set tag 65010
!
ntp server 203.185.69.60
ntp master 5
logging host 172.20.30.20
snmp-server community S1cR3adOnly! RO
snmp-server host 172.20.30.20 version 2c S1cR3adOnly!
!
no cdp run
lldp run
end
```

### 990.2 — Running-Config: HQ-FW1 (ASA 5516-X — ฉบับสมบูรณ์)

```
hostname HQ-FW1
domain-name sic.co.th
enable secret S1cCap5tone!2026
!
interface GigabitEthernet1/1
 nameif outside
 security-level 0
 ip address 172.20.254.2 255.255.255.252
 no shutdown
interface GigabitEthernet1/2
 nameif inside
 security-level 100
 ip address 172.20.254.5 255.255.255.252
 no shutdown
!
object network SIC-INTERNAL
 subnet 172.20.0.0 255.255.0.0
object network CALLCENTER-VLAN
 subnet 172.20.50.0 255.255.255.0
!
access-list OUTSIDE-IN extended deny ip any any log
access-group OUTSIDE-IN in interface outside
!
route outside 0.0.0.0 0.0.0.0 172.20.254.1
route inside 172.20.0.0 255.255.0.0 172.20.254.6
!
aaa authentication ssh console LOCAL
aaa authentication enable console LOCAL
username sic-admin password S1cAdm1n!Str0ng privilege 15
!
ssh 172.20.99.0 255.255.255.0 inside
ssh timeout 10
!
logging enable
logging host inside 172.20.30.20
logging trap informational
ntp server 172.20.255.1
!
end
```

### 990.3 — Running-Config: HQ-CORE-SW1 (Catalyst 9300 — ฉบับสมบูรณ์)

```
hostname HQ-CORE-SW1
no ip domain-lookup
ip domain-name sic.co.th
enable secret S1cCap5tone!2026
service password-encryption
username sic-admin privilege 15 secret S1cAdm1n!Str0ng
username sic-automation privilege 15 secret S1cAut0Bkp!2026
!
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
 exec-timeout 10 0
line console 0
 exec-timeout 10 0
 logging synchronous
!
ip routing
ipv6 unicast-routing
!
vlan 110
 name DATA-STAFF
vlan 120
 name VOICE
vlan 130
 name SERVERS
vlan 140
 name WIFI-CORP
vlan 141
 name WIFI-GUEST
vlan 150
 name CALLCENTER
vlan 199
 name MGMT
!
interface Loopback0
 ip address 172.20.255.11 255.255.255.255
!
interface Port-channel1
 description ** Po1 to HQ-CORE-SW2 - Routed LACP **
 no switchport
 ip address 172.20.254.9 255.255.255.252
 ip ospf message-digest-key 1 md5 S1cOspfKey!
 no shutdown
!
interface range TenGigabitEthernet1/1/1-2
 channel-group 1 mode active
 no shutdown
!
interface TenGigabitEthernet1/1/3
 description ** to HQ-FW1 inside **
 no switchport
 ip address 172.20.254.6 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/0/1
 description ** Trunk to HQ-ACC-SW1 **
 switchport mode trunk
 switchport trunk allowed vlan 110,120,130,140,141,150,199
 spanning-tree guard root
 no shutdown
interface GigabitEthernet1/0/2
 description ** Trunk to HQ-ACC-SW2 (Call Center Floor) **
 switchport mode trunk
 switchport trunk allowed vlan 110,120,130,140,141,150,199
 spanning-tree guard root
 no shutdown
!
interface Vlan110
 description ** DATA-STAFF SVI **
 ip address 172.20.10.2 255.255.255.0
 ipv6 address fd20:20:110::2/64
 ip ospf message-digest-key 1 md5 S1cOspfKey!
 ip helper-address 172.20.30.10
 standby version 2
 standby 110 ip 172.20.10.1
 standby 110 priority 150
 standby 110 preempt delay minimum 60
 standby 110 track Port-channel1 decrement 60
 no shutdown
interface Vlan120
 ip address 172.20.20.2 255.255.255.0
 ipv6 address fd20:20:120::2/64
 standby version 2
 standby 120 ip 172.20.20.1
 standby 120 priority 150
 standby 120 preempt delay minimum 60
 standby 120 track Port-channel1 decrement 60
 no shutdown
interface Vlan130
 ip address 172.20.30.2 255.255.255.0
 ipv6 address fd20:20:130::2/64
 standby version 2
 standby 130 ip 172.20.30.1
 standby 130 priority 150
 standby 130 preempt delay minimum 60
 standby 130 track Port-channel1 decrement 60
 no shutdown
interface Vlan140
 ip address 172.20.40.2 255.255.255.0
 ipv6 address fd20:20:140::2/64
 standby version 2
 standby 140 ip 172.20.40.1
 standby 140 priority 100
 standby 140 preempt delay minimum 60
 standby 140 track Port-channel1 decrement 60
 no shutdown
interface Vlan141
 ip address 172.20.41.2 255.255.255.0
 ipv6 address fd20:20:141::2/64
 ip access-group GUEST-ISOLATION in
 standby version 2
 standby 141 ip 172.20.41.1
 standby 141 priority 100
 standby 141 preempt delay minimum 60
 standby 141 track Port-channel1 decrement 60
 no shutdown
interface Vlan150
 ip address 172.20.50.2 255.255.255.0
 ipv6 address fd20:20:150::2/64
 standby version 2
 standby 150 ip 172.20.50.1
 standby 150 priority 100
 standby 150 preempt delay minimum 60
 standby 150 track Port-channel1 decrement 60
 no shutdown
interface Vlan199
 ip address 172.20.99.2 255.255.255.0
 ipv6 address fd20:20:199::2/64
 standby version 2
 standby 199 ip 172.20.99.1
 standby 199 priority 150
 standby 199 preempt delay minimum 60
 standby 199 track Port-channel1 decrement 60
 no shutdown
!
ip access-list extended GUEST-ISOLATION
 deny ip 172.20.41.0 0.0.0.255 172.20.0.0 0.0.255.255
 permit ip 172.20.41.0 0.0.0.255 any
!
ip dhcp excluded-address 172.20.10.1 172.20.10.20
ip dhcp pool DATA-STAFF
 network 172.20.10.0 255.255.255.0
 default-router 172.20.10.1
 dns-server 172.20.30.10 208.67.222.222
 domain-name sic.co.th
 lease 7
ip dhcp excluded-address 172.20.40.1 172.20.40.20
ip dhcp pool WIFI-CORP
 network 172.20.40.0 255.255.255.0
 default-router 172.20.40.1
 dns-server 172.20.30.10 208.67.222.222
 lease 1
ip dhcp excluded-address 172.20.41.1 172.20.41.10
ip dhcp pool WIFI-GUEST
 network 172.20.41.0 255.255.255.0
 default-router 172.20.41.1
 dns-server 208.67.222.222
 lease 0 4
ip dhcp excluded-address 172.20.50.1 172.20.50.20
ip dhcp pool CALLCENTER
 network 172.20.50.0 255.255.255.0
 default-router 172.20.50.1
 dns-server 172.20.30.10 208.67.222.222
 lease 7
!
router ospf 1
 router-id 172.20.255.11
 passive-interface default
 no passive-interface Port-channel1
 network 172.20.0.0 0.0.255.255 area 0
 area 0 authentication message-digest
!
ipv6 router ospf 1
 router-id 172.20.255.11
!
ip route 172.20.253.0 255.255.255.0 172.20.254.5
!
ip access-list standard MGMT-ONLY-VTY
 permit 172.20.99.0 0.0.0.255
 deny any log
line vty 0 15
 access-class MGMT-ONLY-VTY in
!
ip dhcp snooping vlan 110,120,130,140,141,150,199
ip arp inspection vlan 110,120,130,140,141,150,199
!
ntp server 172.20.255.1
logging host 172.20.30.20
snmp-server community S1cR3adOnly! RO
snmp-server host 172.20.30.20 version 2c S1cR3adOnly!
!
no cdp run
lldp run
end
```

### 990.4 — Running-Config: HQ-CORE-SW2 (Delta Config — เหมือน HQ-CORE-SW1 ทุกส่วน ยกเว้นด้านล่างนี้)

```
hostname HQ-CORE-SW2
! Loopback0 ต่างจาก CORE-SW1
interface Loopback0
 ip address 172.20.255.12 255.255.255.255
! Po1 real IP อีกฝั่งของ /30
interface Port-channel1
 ip address 172.20.254.10 255.255.255.252
! Real IP ของทุก SVI ใช้ .3 (ตาม Convention ก่อน HSRP VIP)
interface Vlan110
 ip address 172.20.10.3 255.255.255.0
 standby 110 priority 100          ! Standby รอง VLAN110 (Active อยู่ที่ CORE-SW1)
interface Vlan140
 ip address 172.20.40.3 255.255.255.0
 standby 140 priority 150          ! Active จริงของ VLAN140 อยู่ที่ CORE-SW2 (Load-Distribute)
interface Vlan141
 ip address 172.20.41.3 255.255.255.0
 standby 141 priority 150
interface Vlan150
 ip address 172.20.50.3 255.255.255.0
 standby 150 priority 150
! (VLAN120,130,199 ใช้ real IP .3 แต่ Priority 100 คงเดิม - Active อยู่ CORE-SW1)
! ส่วน VLAN Database, DHCP Pool, ACL, AAA, NTP, Logging, SNMP — เหมือน HQ-CORE-SW1 ทุกตัวอักษร
```

### 990.5 — Running-Config: HQ-ACC-SW1 (Catalyst 9200 — ฉบับสมบูรณ์)

```
hostname HQ-ACC-SW1
no ip domain-lookup
ip domain-name sic.co.th
enable secret S1cCap5tone!2026
service password-encryption
username sic-admin privilege 15 secret S1cAdm1n!Str0ng
!
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
line console 0
 logging synchronous
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
ip dhcp snooping vlan 110,120,130,140,141,150,199
!
interface range GigabitEthernet1/0/1-40
 switchport mode access
 switchport access vlan 110
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
!
interface GigabitEthernet1/0/41
 description ** Reception - Cisco IP Phone **
 switchport mode access
 switchport access vlan 110
 switchport voice vlan 120
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/48
 description ** Uplink to HQ-CORE-SW1 **
 switchport mode trunk
 switchport trunk allowed vlan 110,120,130,140,141,150,199
 ip dhcp snooping trust
 ip arp inspection trust
 no shutdown
!
ip arp inspection vlan 110,120,130,140,141,150,199
!
ntp server 172.20.255.1
logging host 172.20.30.20
no cdp run
lldp run
end
```

### 990.6 — Running-Config: BR-RTR1 (ISR4321, Branch WAN Edge — ฉบับสมบูรณ์)

```
hostname BR-RTR1
no ip domain-lookup
ip domain-name sic.co.th
enable secret S1cCap5tone!2026
service password-encryption
username sic-admin privilege 15 secret S1cAdm1n!Str0ng
username sic-automation privilege 15 secret S1cAut0Bkp!2026
!
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
line console 0
 logging synchronous
!
interface Loopback0
 ip address 172.21.255.1 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** ISP3 - Primary Fiber **
 ip address 198.51.100.10 255.255.255.252
 ip nat outside
 no shutdown
!
interface Cellular0/1/0
 description ** LTE Backup - Failover Only **
 ip address negotiated
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** to BR-SW1 **
 ip address 172.21.254.1 255.255.255.252
 ip nat inside
 no shutdown
!
interface Tunnel0
 description ** DMVPN Spoke to HQ-RTR1 **
 ip address 172.20.253.2 255.255.255.0
 ip mtu 1400
 ip nhrp authentication S1cNhrpKey
 ip nhrp network-id 99
 ip nhrp nhs 172.20.253.1 nbma 198.51.100.2
 ip nhrp shortcut
 ip ospf network point-to-multipoint
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 2026
 tunnel protection ipsec profile SIC-DMVPN-PROFILE
 no shutdown
!
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
crypto isakmp key S1cDmvpnPSK! address 0.0.0.0 0.0.0.0
crypto ipsec transform-set SIC-TSET esp-aes 256 esp-sha256-hmac
 mode transport
crypto ipsec profile SIC-DMVPN-PROFILE
 set transform-set SIC-TSET
!
ip access-list extended SIC-NAT-ACL
 permit ip 172.21.0.0 0.0.255.255 any
ip nat inside source list SIC-NAT-ACL interface GigabitEthernet0/0/0 overload
!
ip sla 1
 icmp-echo 198.51.100.9 source-interface GigabitEthernet0/0/0
 frequency 5
ip sla schedule 1 life forever start-time now
track 1 ip sla 1
ip route 0.0.0.0 0.0.0.0 198.51.100.9 track 1
ip route 0.0.0.0 0.0.0.0 Cellular0/1/0 200
!
router ospf 1
 router-id 172.21.255.1
 network 172.20.253.0 0.0.0.255 area 0
 network 172.21.254.0 0.0.0.3 area 0
 network 172.21.255.1 0.0.0.0 area 0
!
! ==== Zone-Based Firewall (แทน ASA เพราะ Branch ไม่มี Dedicated Firewall - Part 45 ====
zone security BR-INSIDE
zone security BR-OUTSIDE
zone-pair security BR-IN-TO-OUT source BR-INSIDE destination BR-OUTSIDE
 service-policy type inspect BR-INSPECT-POLICY
class-map type inspect match-any BR-INSPECT-CLASS
 match protocol tcp
 match protocol udp
 match protocol icmp
policy-map type inspect BR-INSPECT-POLICY
 class type inspect BR-INSPECT-CLASS
  inspect
 class class-default
  drop
interface GigabitEthernet0/0/0
 zone-member security BR-OUTSIDE
interface GigabitEthernet0/0/2
 zone-member security BR-INSIDE
!
ntp server 172.20.255.1
logging host 172.21.30.20
no cdp run
lldp run
end
```

### 990.7 — Running-Config: BR-SW1 (Catalyst 9200L, Branch L3 Standalone — ฉบับสมบูรณ์)

```
hostname BR-SW1
no ip domain-lookup
ip domain-name sic.co.th
enable secret S1cCap5tone!2026
service password-encryption
username sic-admin privilege 15 secret S1cAdm1n!Str0ng
!
crypto key generate rsa modulus 2048
ip ssh version 2
line vty 0 15
 transport input ssh
 login local
line console 0
 logging synchronous
!
ip routing
ipv6 unicast-routing
!
vlan 210
 name DATA-OFFICE
vlan 220
 name VOICE
vlan 230
 name WMS-EDGE
vlan 240
 name WIFI-WAREHOUSE
vlan 241
 name WIFI-GUEST
vlan 250
 name CCTV-IOT
vlan 299
 name MGMT
!
interface Loopback0
 ip address 172.21.255.11 255.255.255.255
!
interface GigabitEthernet1/0/1
 description ** to BR-RTR1 **
 no switchport
 ip address 172.21.254.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/0/48
 description ** Trunk to BR-ACC-SW2 (Warehouse Floor) **
 switchport mode trunk
 switchport trunk allowed vlan 210,220,230,240,241,250,299
 no shutdown
!
interface Vlan210
 ip address 172.21.10.1 255.255.255.0
 ipv6 address fd21:20:210::1/64
 no shutdown
interface Vlan220
 ip address 172.21.20.1 255.255.255.0
 ipv6 address fd21:20:220::1/64
 no shutdown
interface Vlan230
 ip address 172.21.30.1 255.255.255.0
 ipv6 address fd21:20:230::1/64
 no shutdown
interface Vlan240
 ip address 172.21.40.1 255.255.255.0
 ipv6 address fd21:20:240::1/64
 no shutdown
interface Vlan241
 ip address 172.21.41.1 255.255.255.0
 ipv6 address fd21:20:241::1/64
 ip access-group GUEST-ISOLATION-BR in
 no shutdown
interface Vlan250
 ip address 172.21.50.1 255.255.255.0
 ipv6 address fd21:20:250::1/64
 ip access-group CCTV-RESTRICTED in
 no shutdown
interface Vlan299
 ip address 172.21.99.1 255.255.255.0
 ipv6 address fd21:20:299::1/64
 no shutdown
!
ip access-list extended GUEST-ISOLATION-BR
 deny ip 172.21.41.0 0.0.0.255 172.21.0.0 0.0.255.255
 permit ip 172.21.41.0 0.0.0.255 any
ip access-list extended CCTV-RESTRICTED
 permit tcp 172.21.50.0 0.0.0.255 host 52.76.20.20 eq 443
 deny ip 172.21.50.0 0.0.0.255 172.21.0.0 0.0.255.255 log
 deny ip any any log
!
ip dhcp excluded-address 172.21.10.1 172.21.10.20
ip dhcp pool DATA-OFFICE
 network 172.21.10.0 255.255.255.0
 default-router 172.21.10.1
 dns-server 172.21.30.10 208.67.222.222
 domain-name sic.co.th
ip dhcp excluded-address 172.21.40.1 172.21.40.20
ip dhcp pool WIFI-WAREHOUSE
 network 172.21.40.0 255.255.255.0
 default-router 172.21.40.1
 dns-server 172.21.30.10 208.67.222.222
 lease 1
!
router ospf 1
 router-id 172.21.255.11
 passive-interface default
 no passive-interface GigabitEthernet1/0/1
 network 172.21.0.0 0.0.255.255 area 0
!
ip dhcp snooping vlan 210,220,230,240,241,250,299
ip arp inspection vlan 210,220,230,240,241,250,299
interface range GigabitEthernet1/0/2-40
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 spanning-tree portfast
 spanning-tree bpduguard enable
!
ntp server 172.20.255.1
logging host 172.21.30.20
no cdp run
lldp run
end
```

> **BR-ACC-SW2 (Warehouse Floor Access Switch)**: Config เหมือน `HQ-ACC-SW1` (990.5) ทุกส่วน
> เปลี่ยนเพียง Hostname, VLAN Allowed (210,220,230,240,241,250,299), และ Uplink ไปยัง `BR-SW1`
> — ไม่ขอเขียนซ้ำตามหลัก DRY (Don't Repeat Yourself) ที่ Part 47-48 สอนเรื่อง Template ไว้แล้ว

### 990.8 — AP Configuration Note (HQ-AP1/2, BR-AP1 — Mobility Express Embedded Controller)

Access Point ที่ใช้ Mobility Express ตั้งค่าหลักผ่าน Web GUI ของ AP เอง (ไม่ใช่ Running-Config
แบบ IOS) — ค่าที่สำคัญที่ต้องตั้งตรงกับ Design ของ Part นี้:

```
SSID: SIC-Corp        -> Map to VLAN 140, Security: WPA2/WPA3-Enterprise (802.1X ผ่าน NPS 172.20.30.5)
SSID: SIC-Guest        -> Map to VLAN 141, Security: WPA2-PSK (Rotate ทุก 90 วัน), Client Isolation: Enabled
SSID: SIC-Warehouse    -> Map to VLAN 240 (Branch เท่านั้น), Security: WPA2-PSK, Band: 2.4GHz+5GHz
QoS Profile: Platinum (Voice) ไม่ได้ Map ให้ Wi-Fi SSID ใดเลย — Softphone วิ่งบน Wired Data VLAN
             ของ Call Center เท่านั้น (ตาม Design Step 987.3 ไม่ใช่ Wireless Voice)
```

### 990.9 — Retrospective: เชื่อมทุกการตัดสินใจของ Siam Innovation Corp กลับไปยัง Part ที่สอนทักษะนั้น

| การตัดสินใจ/สิ่งที่สร้างใน Part 99 | Part ที่สอนทักษะพื้นฐาน | Part ที่สอนทักษะขั้นสูงที่ใช้ตัดสินใจ "Scale ลง" |
|---|---|---|
| OSI/TCP-IP Model ที่ใช้อธิบาย Troubleshoot ตลอด Part นี้ | **Part 1** | — |
| Initial Setup, SSH, Password Security ทุกอุปกรณ์ | **Part 2** | — |
| VLAN, Access Port, Voice VLAN (Reception Phone) | **Part 3** | — |
| Trunk 802.1Q ระหว่าง Core↔Access | **Part 4** | — |
| STP Root ที่ HQ-CORE-SW1, PortFast/BPDU Guard | **Part 5** | Part 33 (ตัดสินใจไม่ทำ Multi-Instance ซับซ้อนเพราะ Topology เล็ก) |
| EtherChannel Po1 (LACP) ระหว่าง Core Pair | **Part 6** | — |
| IPv4 Addressing/Subnetting (IP Plan ใหม่ทั้งหมดของ SIC) | **Part 7** | — |
| IPv6 ULA คู่กับทุก VLAN (Dual-Stack) | **Part 8** | Part 88 (ตัดสินใจ Deploy Internal ก่อน แต่ยังไม่ทำ WAN Dual-Stack เพราะ ISP ไทยส่วนใหญ่ยังไม่รองรับ) |
| Static Route Bootstrap ก่อนมี OSPF | **Part 9** | — |
| Inter-VLAN Routing: SVI บน Core, Routed Port ไปยัง FW1 | **Part 10** | — |
| OSPF Single-Area (Area 0) ทั้งเครือข่าย | **Part 11-12** | (ตัดสินใจไม่ทำ Multi-Area เพราะอุปกรณ์ L3 มีแค่ 4-5 ตัว) |
| (พิจารณา EIGRP แล้วไม่เลือก — เหตุผล Vendor-Neutral) | **Part 13** | — |
| ACL: GUEST-ISOLATION, CCTV-RESTRICTED, MGMT-ONLY-VTY | **Part 14** | Part 45 (พิจารณา ZBFW เต็มรูปแบบที่ Branch แทน ACL ธรรมดา) |
| NAT/PAT Overload ที่ HQ-RTR1/BR-RTR1 | **Part 15** | — |
| DHCP Pool บน Switch เอง, DNS ผ่าน Umbrella | **Part 16** | — |
| NTP (NECTEC), Syslog/SNMP ไปยัง LibreNMS | **Part 17** | Part 54 (พิจารณา Full Telemetry Streaming แล้วเลือก SNMP/Syslog ธรรมดาพอ) |
| HSRP พร้อม Load-Distribution + Preempt Delay | **Part 18** | Part 35 (Advanced FHRP HA — ใช้เทคนิค Track/Preempt Delay ตรง) |
| WAN Technology Comparison (Step 982.7), IP SLA Failover | **Part 19** | Part 39/76 (พิจารณา SD-WAN แล้วไม่เลือกตอนนี้) |
| Wireless: SSID/VLAN Mapping | **Part 20** | Part 36 (พิจารณา Dedicated WLC แล้วเลือก Mobility Express) |
| Port Security, DHCP Snooping, DAI ทุก Access Port | **Part 21** | — |
| AAA (Local + NPS RADIUS), Device Hardening | **Part 22** | Part 78 (พิจารณา ISE แล้วเลือกใช้ NPS ที่มีอยู่แล้วแทน) |
| Config Backup Script (Python/Netmiko) | **Part 23** | Part 47 (Python ขั้นสูง) |
| QoS: Conditional Trust Boundary, DSCP Marking, LLQ | **Part 24** | Part 40 (NBAR/Class-Based Marking ขั้นสูง) |
| แนวทาง Capstone/Full Topology Recap | **Part 25** | (แรงบันดาลใจของโครงสร้าง Part นี้เอง) |
| (พิจารณา Advanced OSPF Feature เพิ่มแล้วไม่จำเป็น) | Part 26 | — |
| OSPFv3 สำหรับ IPv6 คู่ขนาน | **Part 27** | — |
| BGP Fundamentals/Advanced สำหรับ AWS VPN | **Part 29-30** | Part 74 (WAN Edge BGP ขั้นสูง — ใช้แค่บางส่วน) |
| Route Redistribution + Route-map + Tag (BGP→OSPF) | **Part 31** | — |
| (พิจารณา PBR แล้วไม่จำเป็นตอนนี้ — Traffic Pattern ไม่ซับซ้อนพอ) | Part 32 | — |
| Campus Design: เลือก 2-Tier Collapsed แทน 3-Tier | Part 1 | **Part 33** (Advanced Campus Design — ให้เหตุผลไม่ใช้ 3-Tier) |
| StackWise Virtual — ประเมินแล้วไม่เลือกใช้ | — | **Part 34** (เหตุผลเต็มใน Step 984.1) |
| HSRP Load-Distribution/Preempt Delay | Part 18 | **Part 35** |
| SSID/AP Design พื้นฐาน | Part 20 | **Part 36** (ประเมินแล้วไม่ทำ Dedicated WLC) |
| WPA2/WPA3-Enterprise บน Corp Wi-Fi | Part 20 | **Part 37** |
| (พิจารณา SD-Access แล้วไม่เลือก — Overkill สำหรับ 2 Site) | — | Part 38, 77 |
| SD-WAN — ประเมินเปรียบเทียบแล้วไม่เลือกตอนนี้ | Part 19 | **Part 39, 76** |
| Advanced QoS (NBAR-style Classification) | Part 24 | **Part 40** |
| Multicast — ประเมินแล้วไม่ต้องมี Use Case | — | **Part 41-42, 75, 83** |
| Security Architecture Framework (ใช้แนวคิด ไม่ใช้ ISE จริง) | — | **Part 43** |
| 802.1X + MAB เฉพาะ VLAN150/Corp Wi-Fi | Part 22 | **Part 44** (TrustSec/SGT ประเมินแล้วไม่จำเป็น) |
| ZBFW ที่ Branch (แทน ASA ที่ไม่มี) | Part 14 | **Part 45** |
| CoPP พื้นฐานที่ WAN Edge | — | **Part 46** |
| Python (Netmiko) Backup Script | Part 23 | **Part 47** |
| Ansible Playbook Push Config | — | **Part 48** |
| (พิจารณา NETCONF/RESTCONF แล้วยังไม่จำเป็นสำหรับ Fleet เล็ก) | — | Part 49 |
| DNA Center — ประเมินแล้วใช้ LibreNMS แทน | — | **Part 50** |
| GRE/IPsec สำหรับ AWS VTI Tunnel | — | **Part 51** |
| DMVPN Hub-Spoke Phase 3 (mGRE) | — | **Part 52-53** |
| Network Assurance — ใช้ LibreNMS แบบเบา | — | **Part 54** |
| แนวทาง Capstone/Full-Topology, Design Decision Log | Part 55 | (แรงบันดาลใจของโครงสร้าง Part นี้เอง) |
| Troubleshooting Layer-by-Layer ตลอด Pre-Production Checklist | Part 56-59 | Part 91 |
| DMVPN/VPN Troubleshooting Mindset ที่ใช้ตรวจสอบ Step 989.1 | Part 60 | — |
| Infra Security/Services Troubleshooting Checklist | Part 61-62 | — |
| NAT Troubleshooting Mindset | Part 63 | — |
| (PBR Troubleshooting — ไม่ได้ใช้ PBR จึงไม่จำเป็น) | Part 64 | — |
| Multicast Troubleshooting (ไม่จำเป็นเพราะไม่ Deploy) | Part 65 | — |
| Wireless Troubleshooting Mindset | Part 66 | — |
| Automation Troubleshooting Mindset | Part 67 | — |
| QoS Troubleshooting Mindset (Call Center Voice Quality) | Part 68 | — |
| Mega-Lab Mindset (รวมหลาย Domain พร้อมกัน) | Part 69 | (แรงบันดาลใจ Testing Phase) |
| Mock Exam/Full Skill-Map Mindset | Part 70 | (แรงบันดาลใจของโครงสร้าง Part นี้เอง) |
| CCIE Lab Blueprint แนวคิด Design/Deploy/Optimize | Part 71 | ใช้เป็นกรอบคิดของ Step 982-990 ทั้งหมด |
| Large Campus Fabric — ประเมินแล้วไม่จำเป็น (2 Site เล็ก) | — | Part 72 |
| Large-Scale Routing — ประเมินแล้ว Single-Area พอ | — | Part 73 |
| Advanced BGP WAN Edge — ใช้บางส่วนสำหรับ AWS | Part 29-30 | Part 74 |
| Advanced Multicast — ไม่จำเป็น | — | Part 75 |
| SD-WAN Advanced Design — ประเมินแล้วไม่เลือกตอนนี้ | Part 39 | Part 76 |
| SD-Access Advanced — ประเมินแล้วไม่เลือก | Part 38 | Part 77 |
| Security Integration (ISE/TrustSec/FTD/Umbrella) — เลือกเฉพาะ Umbrella | Part 43-46 | **Part 78** |
| Full-Stack Automation/CI-CD — ใช้แนวคิด Staged Rollout เท่านั้น | Part 47-49 | **Part 79** |
| Model-Driven Telemetry — ประเมินแล้ว SNMP/Syslog พอ | Part 54 | Part 80 |
| High Availability (NSF/SSO/ISSU) — ประเมินแล้วไม่จำเป็นระดับนี้ | — | Part 81 |
| End-to-End QoS Design Mindset | Part 24, 40 | **Part 82** |
| Enterprise Multicast Deep-Dive — ไม่จำเป็น | — | Part 83 |
| VXLAN/EVPN — ประเมินแล้ว Overkill สำหรับ Campus เล็ก | — | Part 84-85 |
| MPLS L3VPN — ประเมินเปรียบเทียบแล้วไม่เลือก (Step 982.7) | — | **Part 86-87** |
| IPv6 Enterprise Deployment มุมมอง Dual-Stack ภายใน | Part 8, 27 | **Part 88** |
| Enterprise Wireless High-Density — ไม่จำเป็น (AP น้อย) | — | Part 89 |
| Catalyst 9000 Advanced Features — ใช้พื้นฐานเท่านั้น | — | Part 90 |
| Troubleshooting Methodology ที่ใช้ตลอด Step 989 | — | **Part 91** |
| Disaster Recovery มุมมอง Rollback Plan/Cold Spare | — | **Part 92** |
| Cloud Connectivity (AWS VPN/BGP) | — | **Part 93** |
| Zero Trust — Macro-Segmentation ที่ Scale ให้พอดี | — | **Part 94** |
| CCIE Lab Scenario ทักษะ Diagnose/Design/Deploy/Optimize | — | Part 95-98 |

> **ข้อสังเกตสำคัญที่สุดของตารางนี้**: มี Part จำนวนมาก (32, 38, 41-42, 49-50, 65, 72-73, 75-77,
> 80-81, 83-85, 89-90) ที่ Siam Innovation Corp **"ประเมินแล้วไม่เลือกใช้"** — นี่**ไม่ใช่ช่องโหว่
> ของ Part 99** แต่คือ**หลักฐานว่าวิศวกรที่ผ่านหลักสูตรนี้ครบ 98 Part สามารถบอกได้อย่างมีเหตุผลว่า
> "ทำไมไม่ใช้"** ทุกเทคโนโลยีเหล่านั้น ไม่ใช่แค่ "ไม่รู้จัก" — ความแตกต่างระหว่างสองคำนี้คือ
> เส้นแบ่งที่ชัดเจนที่สุดระหว่างวิศวกรที่ท่อง Certification มาสอบผ่าน กับวิศวกรที่พร้อมออกแบบเครือข่าย
> จริงให้ธุรกิจจริงที่มีงบประมาณจริงและ Requirement จริง — ซึ่งคือเป้าหมายแท้จริงของทั้งหลักสูตรนี้

---

## แบบฝึกหัดทวนความเข้าใจ Part 99

1. เพราะเหตุใด Siam Innovation Corp จึงเลือก DMVPN Phase 3 แทน MPLS L3VPN หรือ Commercial
   SD-WAN สำหรับเชื่อมต่อ Branch ทั้งที่หลักสูตรนี้สอนทั้ง 3 เทคโนโลยีอย่างละเอียด?
2. เพราะเหตุใด HQ Core จึงเลือกใช้ Standalone Pair + HSRP แทน StackWise Virtual ทั้งที่ SVL
   ให้ Active/Active Bandwidth เต็มรูปแบบมากกว่า?
3. ทำไม eBGP จึงถูกใช้เฉพาะจุดเชื่อมต่อ AWS Site-to-Site VPN เท่านั้น ไม่ใช้แทน OSPF ทั้งเครือข่าย?
4. อธิบายว่า `ip nhrp shortcut` ที่ตั้งไว้ที่ `BR-RTR1` (Step 985.2) ทั้งที่ปัจจุบันมี Spoke แค่ตัว
   เดียว มีประโยชน์อย่างไรกับ Growth Plan ของบริษัทในปีที่ 2?
5. เพราะเหตุใด Call Center (VLAN150) จึงได้รับ Security Control ที่เข้มกว่า VLAN อื่น (802.1X,
   ACL เข้มงวดกว่า) ทั้งที่ SIC ไม่ได้เก็บเลขบัตรเครดิตบน Server ของตัวเองเลย?

**เฉลย:**

1. เพราะ SIC มีแค่ 2 Site และงบประมาณจำกัด — MPLS L3VPN มีค่าใช้จ่ายรายเดือนสูงและ Lead Time
   นานเกินกำหนด Go-Live ในขณะที่ Commercial SD-WAN ต้องมี Controller License ที่ไม่คุ้มค่ากับ
   จำนวน Site เพียง 2 แห่ง DMVPN ให้ Encryption ในตัว, ต้นทุนต่ำ (ใช้ Internet Circuit ที่มีอยู่แล้ว),
   และขยาย Spoke ใหม่ได้ทันทีโดยไม่ต้องแก้ Hub — ตรงกับ Growth Plan ที่จะเพิ่ม Site ที่ 3 ในปีที่ 2
   พอดี (ดูตารางเปรียบเทียบ Step 982.7)
2. StackWise Virtual ต้องมี Dedicated SVL Link เฉพาะและ License เพิ่ม แต่ที่สำคัญกว่าคือ**ความ
   ซับซ้อนในการ Troubleshoot** — ทีม IT ของ SIC มีแค่ 2 คน การมี 2 Switch อิสระที่ Debug แยกกัน
   ได้ตรงไปตรงมามีค่ามากกว่าประโยชน์ของ Active/Active Bandwidth เต็มรูปแบบที่ Traffic ระดับ
   150 คนไม่ต้องการอยู่แล้ว (ดู Step 984.1)
3. เพราะ BGP ถูก**บังคับใช้โดย AWS เอง** (Requirement จาก Cloud Provider ไม่ใช่ทางเลือกทาง
   วิศวกรรมของเรา) สำหรับ Site-to-Site VPN แบบ Redundant 2 Tunnel — ส่วนเครือข่ายภายในทั้งหมด
   (แค่ 4-5 อุปกรณ์ L3) ไม่มีความจำเป็นต้องใช้ Protocol ที่ซับซ้อนกว่า OSPF Single-Area เลย
   หลักการคือ "เพิ่มความซับซ้อนเฉพาะจุดที่มี Requirement บังคับจริงเท่านั้น"
4. `ip nhrp shortcut` เป็นองค์ประกอบที่จำเป็นสำหรับ DMVPN Phase 3 Spoke-to-Spoke Direct
   Communication — ถ้าปีที่ 2 เพิ่ม Spoke ตัวที่ 2 (Chiang Mai Regional Office) Traffic ระหว่าง
   Branch ทั้งสองจะสามารถวิ่งตรงถึงกัน (ผ่าน NHRP Redirect จาก Hub) โดยไม่ต้องผ่าน Hub HQ-RTR1
   เสมอ ลด Latency และภาระของ Hub — ตั้งไว้ตั้งแต่ Day 1 แม้ยังใช้ประโยชน์ไม่ได้ทันที เพราะการ
   เพิ่มทีหลังต้องไป Config ทุก Spoke ใหม่อยู่ดี ในขณะที่ตั้งไว้ก่อนไม่มีต้นทุนเพิ่มเลย
5. เพราะ Call Center Agent **เห็นข้อมูลบางส่วนของบัตรเครดิตลูกค้าแบบ Masked** ผ่าน Support Tool
   แม้ SIC จะไม่เก็บเลขบัตรเต็มบน Server ของตัวเอง (ใช้ Tokenization ผ่าน Payment Gateway) ตาม
   PCI-DSS Scope SAQ A-EP ยังกำหนดให้ต้องมี Network Segmentation ที่ชัดเจนสำหรับ Segment ที่
   "อยู่ติดกับ" Cardholder Data Environment (CDE-Adjacent) — นี่คือหลักการ Risk-Based Security
   Investment: ใส่ Control ที่แข็งแรงที่สุดตรงจุดที่ Compliance/ความเสี่ยงจริงบังคับ ไม่ใช่ใส่ทุกจุด
   เท่ากันหมดโดยไม่แยกแยะระดับความเสี่ยง (ดู Step 986.1)

---

## สรุป Part 99

Part นี้พา Enterprise Network ของ **Siam Innovation Corp** จาก **กระดาษเปล่า** ไปสู่เครือข่าย
2 Site ที่ใช้งานได้จริง ครบ 8 Phase: **Business Requirements (981) → Design (982) → Foundation
Build (983) → Routing & Redundancy (984) → WAN & Remote Site (985) → Security Hardening
(986) → Services & QoS (987) → Automation & Monitoring (988) → Testing & Go-Live (989) →
Full Lab + Retrospective (990)** ✅ โดยใช้ทักษะจาก **Part 1 ถึง Part 94** ทุก Domain — Switching,
Routing, WAN, Wireless, Security, QoS, Automation, Cloud — แต่สิ่งที่สำคัญกว่า Config ทุกบรรทัด
คือ**ทักษะการเลือกและปฏิเสธเทคโนโลยีอย่างมีเหตุผล** ให้พอดีกับองค์กรขนาดกลางที่มีงบประมาณจำกัด
และทีม IT เล็ก — ทักษะที่ไม่มี Certification ใบไหนสอนตรงๆ แต่วัดผลได้จาก Retrospective Table
ใน Step 990.9 ที่เชื่อมทุกการตัดสินใจกลับไปยัง Part ที่สอนทักษะนั้นจริง

**Part นี้คือบทพิสูจน์สุดท้ายก่อนจบหลักสูตร**: จาก Part 1 ที่เริ่มด้วย "เครือข่ายคอมพิวเตอร์คืออะไร"
มาถึงจุดที่คุณสามารถรับ Requirement จากธุรกิจจริง แปลงเป็น Design ที่มีเหตุผลรองรับทุกข้อ,
Deploy ด้วยมือตัวเองครบทุก Layer, และอธิบายได้ว่าทำไมไม่เลือกเทคโนโลยีบางตัวที่เรียนมา — เหลือ
เพียง **Part 100 — Career Mastery: Exam Strategy & Real-World Scenarios** ที่จะปิดหลักสูตรทั้ง
100 Part ด้วยการเตรียมตัวสอบ CCIE Enterprise Infrastructure จริง และเส้นทางอาชีพหลังจากนี้

**ไปต่อ:** [Part 100 — Career Mastery: Exam Strategy & Real-World Scenarios →](part-100-career-mastery-exam-strategy.md)
