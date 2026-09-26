# Part 71 — CCIE Enterprise Infrastructure Lab Blueprint Overview
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 701–710 จาก 1000 | เริ่มต้นระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 70 — ENARSI Capstone & Mock Exam](part-070-enarsi-capstone-mock-exam.md) ที่ปิด
> **CCNP Enterprise เต็มใบ** (ENCOR 350-401 + ENARSI 300-410) อย่างสมบูรณ์ — Part นี้คือจุดเริ่มต้น
> ของ **ระดับสุดท้ายและยากที่สุดของหลักสูตร: CCIE Enterprise Infrastructure (Part 71-100,
> Step 701-1000)** ก่อนจะลงมือ Config เทคโนโลยีใหม่ใดๆ (VXLAN/EVPN, MPLS L3VPN, SD-Access/SD-WAN
> ระดับ Scale, Zero Trust, Cloud) ตั้งแต่ Part 72 เป็นต้นไป Part นี้จะ **ปูภาพรวมทั้งหมดของ CCIE
> Lab Exam ก่อน** — รูปแบบข้อสอบจริง, Blueprint 5 Domain แบบเป็นทางการ, 4 รูปแบบงานที่ต้องทำในห้อง
> สอบ, ความต่างของ "วิธีคิด" ระหว่าง CCNP กับ CCIE, สภาพแวดล้อม Lab จริง, กลยุทธ์การฝึกฝน, บทบาทของ
> Automation ในข้อสอบเวอร์ชันปัจจุบัน, จุดพลาดที่ผู้สอบส่วนใหญ่เจอ, แผนเนื้อหา Part 72-100 ทั้งหมด
> และปิดท้ายด้วย **Gap Analysis Lab** ที่ Mapping ทุกหัวข้อใน Blueprint เข้ากับ Part ที่สอนไปแล้ว
> (1-70) หรือ Part ที่กำลังจะสอน (72-100) เพื่อให้คุณมีตาราง Self-Tracking ของตัวเองใช้ตลอดเส้นทาง

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 701 | CCIE Enterprise Infrastructure คืออะไร — รูปแบบข้อสอบ Lab 8 ชั่วโมง และความต่างจาก CCNP |
| 702 | CCIE EI v1.1 Lab Exam Blueprint — 5 Domain อย่างเป็นทางการ พร้อมตาราง Sub-topic เต็มรูปแบบ |
| 703 | 4 รูปแบบงานในห้องสอบ CCIE — Design, Deploy, Optimize/Operate, Diagnose |
| 704 | CCIE คิดต่างจาก CCNP อย่างไร — ตัวอย่างเปรียบเทียบคำถาม OSPF Area Design แบบ CCNP vs CCIE |
| 705 | สภาพแวดล้อม Lab Exam จริง — อุปกรณ์ IOS-XE/NX-OS/WLC, จำนวน Device, ความจริงเรื่องเวลา |
| 706 | กลยุทธ์การฝึกฝนสำหรับ CCIE — ทำไม End-to-End Lab ดีกว่า Isolated Feature Practice |
| 707 | บทบาท Automation ใน CCIE ยุคใหม่ — ทำไม Part 47-49/79 ไม่ใช่แค่ "Nice to Have" อีกต่อไป |
| 708 | จุดพลาดคลาสสิกของผู้สอบ CCIE — Time Allocation, การอ่านโจทย์, การ Verify |
| 709 | แผนเนื้อหา Part 72-100 ของหลักสูตรนี้ — เทคโนโลยีเจาะลึกและ Full Lab Scenario |
| 710 | Full Lab: Gap Analysis — Mapping ทุกหัวข้อ Blueprint เข้ากับ Part 1-70 (ทำแล้ว) / 72-100 (กำลังจะทำ) |

---

## Step 701 — CCIE Enterprise Infrastructure คืออะไร และรูปแบบข้อสอบ Lab 8 ชั่วโมง

### CCIE คืออะไร ในบริบทของ Cisco Certification Track

**CCIE (Cisco Certified Internetwork Expert)** คือ Certification ระดับสูงสุดของสาย Track ปกติของ
Cisco (เหนือกว่า CCNA และ CCNP) วัดความสามารถ**ระดับ Expert ในการออกแบบ, Deploy, ปรับแต่ง,
และวินิจฉัยปัญหา**เครือข่ายองค์กรที่ซับซ้อนระดับ Production จริง ไม่ใช่แค่ "รู้วิธี Config Feature"
แบบ CCNP แต่ต้อง**ประกอบ Feature หลายสิบตัวเข้าด้วยกันให้ทำงานร่วมกันได้ภายใต้ข้อจำกัดทางธุรกิจ**
และทำทั้งหมดนี้**ด้วยมือตัวเองสดๆในห้องสอบ ไม่มีตัวช่วย ไม่มี Google**

CCIE Enterprise Infrastructure (ย่อ **CCIE EI**) คือ Track ที่สืบทอดมาจาก CCIE Routing & Switching
เดิม (ปรับชื่อและ Scope ใหม่ตั้งแต่ปี 2020) ครอบคลุมเทคโนโลยี Enterprise Campus/WAN/Automation
ทั้งหมด — เป็น Track ที่ "ตรงสาย" ที่สุดกับหลักสูตรนี้ที่เดินทางมาตั้งแต่ Part 1 (Networking
Fundamentals) ผ่าน CCNA และ CCNP Enterprise เต็มรูปแบบ

### CCIE ประกอบด้วย 2 ส่วนสอบ — Written Exam + Lab Exam

| ส่วนสอบ | รูปแบบ | ระยะเวลา | ผ่านแล้วได้อะไร |
|---|---|---|---|
| **Qualification (Written) Exam** — CCIE Enterprise Infrastructure v1.1 | Multiple-choice, Drag-drop, Simulation แบบจำกัด บน Computer-based Testing Center | 120 นาที | สิทธิ์สมัครสอบ **Lab Exam** (มีอายุ 18 เดือน) |
| **Lab Exam** — CCIE Enterprise Infrastructure v1.1 | **Hands-on เต็มรูปแบบ ไม่มี Multiple-choice เลย** บนอุปกรณ์ Cisco จริง/Virtual ผ่าน Remote Lab | **8 ชั่วโมง** ต่อเนื่อง | ใบ **CCIE Enterprise Infrastructure** (Active Certification, ต่ออายุทุก 3 ปี) |

> **จุดสำคัญที่สุดที่ต้องเข้าใจก่อนอ่านต่อ**: Written Exam ของ CCIE **มีเนื้อหาเนื้อหาเดียวกันโดย
> พื้นฐานกับที่คุณเรียนมาแล้วใน Part 1-70** (ขยายลึกขึ้นอีกระดับ) แต่ **Lab Exam คือด่านที่แท้จริง**
> ของ CCIE — Candidate ส่วนใหญ่ที่ "รู้ทฤษฎีครบ" กลับสอบ Lab ไม่ผ่านซ้ำแล้วซ้ำเล่า เพราะ Lab ไม่ได้
> วัดว่า "รู้อะไร" แต่วัดว่า "**ทำได้ภายใต้ความกดดันของเวลาและความซับซ้อนของ Integration หรือไม่**"

### Lab Exam ต่างจาก CCNP (Written + Practical) อย่างไร — ตารางเปรียบเทียบ

| มิติ | CCNP (ENCOR + ENARSI) | CCIE EI Lab Exam |
|---|---|---|
| รูปแบบคำถาม | Multiple-choice + Simulation สั้นๆ (ไม่กี่นาทีต่อข้อ) | **ไม่มี Multiple-choice เลย** — เป็นโจทย์ Config/Design/Troubleshoot จริงต่อเนื่อง |
| ระยะเวลา | ENCOR 120 นาที, ENARSI 90 นาที (แยกกัน 2 รอบสอบ) | **8 ชั่วโมงต่อเนื่องในวันเดียว** ไม่มีการหยุดพักยาว |
| อุปกรณ์ที่ทดสอบ | Simulator จำลอง (Packet Tracer-style) จำนวนจำกัด | **Topology จริงหลายสิบอุปกรณ์** (IOS-XE, NX-OS, WLC) เชื่อมโยงกันทั้งหมด |
| ขอบเขตโจทย์ | โจทย์แยกเป็น Feature เดี่ยว (เช่น "Config OSPF area นี้") | **โจทย์เป็น Business Requirement** ที่ต้องแปลงเป็น Design แล้ว Implement ให้ครบ Chain |
| การอ้างอิงเอกสาร | ไม่มีเอกสารช่วยระหว่างสอบ (แต่โจทย์ตรงไปตรงมา) | ไม่มีเอกสารช่วยเช่นกัน **แต่โจทย์ซับซ้อนหลายชั้น ต้องอ่านและตีความ Requirement เอง** |
| ผลกระทบของ Config ผิด | คะแนนข้อนั้นเสียไปข้อเดียว | **Config ผิดจุดเดียวอาจทำให้โจทย์ต่อเนื่องอีกหลายข้อพังไปด้วย** (Cascading Failure) |
| การให้คะแนน | ถูก/ผิดต่อข้อ ชัดเจน | ให้คะแนนจาก **State ปลายทางของเครือข่ายจริง** ผ่าน Automated + Proctor Verification |

### Prerequisite Path — Part 1-70 คือฐานความรู้ ส่วน CCIE Lab คือการเอาไปใช้จริงภายใต้ความกดดัน

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     เส้นทางความรู้ของหลักสูตรนี้ (Part 1-100)                │
│                                                                            │
│  Part 1-25 (CCNA)          Part 26-55 (ENCOR)        Part 56-70 (ENARSI)  │
│  ┌────────────────┐        ┌────────────────┐        ┌────────────────┐  │
│  │ พื้นฐานทุกอย่าง   │  ───►  │ Enterprise-scale│  ───►  │ วินิจฉัยเมื่อพัง   │  │
│  │ Switch/Route/    │        │ Feature ลึก     │        │ ทุก Domain      │  │
│  │ Security/Wireless│        │ + Design        │        │ (Diagnostic     │  │
│  │ พื้นฐาน           │        │ พื้นฐาน         │        │  Skill)         │  │
│  └────────────────┘        └────────────────┘        └────────────────┘  │
│         │                          │                          │          │
│         └──────────────────────────┴──────────────────────────┘          │
│                                     │                                     │
│                                     ▼                                     │
│                     = ฐานความรู้ (Knowledge Base) ครบ 100%                 │
│                                     │                                     │
│                                     ▼                                     │
│  Part 71-100 (CCIE EI)  ──►  ไม่ใช่ความรู้ใหม่ล้วนๆ แต่คือ:                  │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ 1) เทคโนโลยีต่อยอดที่ลึกกว่า CCNP (VXLAN/EVPN, MPLS L3VPN เต็มรูป) │    │
│  │ 2) การ "ใช้ความรู้เดิมทั้งหมดพร้อมกัน" ภายใต้ Time Pressure จริง    │    │
│  │ 3) การ Integrate ข้าม Domain (Routing+Security+Automation+Wireless)│   │
│  │    ในโจทย์เดียวกัน แบบที่ CCNP ไม่เคยบังคับให้ทำพร้อมกัน            │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

พูดให้ชัดที่สุด: **CCIE Lab ไม่ได้ถามอะไรที่คุณไม่เคยเรียน** (ยกเว้นเทคโนโลยีเสริมบางตัวที่ Part
72-94 จะเติมให้) แต่มันถามให้คุณ **เอาทุกอย่างที่เรียนมาตั้งแต่ Part 1 มาใช้พร้อมกัน ถูกต้อง
ครบถ้วน ภายในเวลาจำกัด และพร้อมอธิบาย/แก้ไขได้ทันทีเมื่อมีคนถามว่า "ทำไมเลือกวิธีนี้"** — นี่คือ
เหตุผลที่ CCIE ถูกมองว่าเป็น Certification ที่ยากที่สุดในสายเครือข่ายระดับโลก ไม่ใช่เพราะเนื้อหา
เพียวๆยากเกินเข้าใจ แต่เพราะ**การ Integrate ทุกอย่างให้ทำงานร่วมกันได้จริงภายใต้ความกดดันคือทักษะ
ที่ฝึกยากที่สุด**

---

## Step 702 — CCIE EI v1.1 Lab Exam Blueprint: 5 Domain อย่างเป็นทางการ

Cisco แบ่ง CCIE Enterprise Infrastructure Lab Exam v1.1 เป็น **5 Domain** ที่มีน้ำหนักคะแนนต่างกัน
อย่างมีนัยสำคัญ — สังเกตว่าแตกต่างจาก ENARSI (4 Domain, เน้น Troubleshoot อย่างเดียว) และ ENCOR
(6 Domain, เน้น Feature กว้าง) เพราะ CCIE เพิ่ม **Domain Automation ขึ้นมาเป็นสัดส่วนใหญ่ที่สุด
เป็นอันดับ 2 (25%)** สะท้อนทิศทางอุตสาหกรรมที่ Automation ไม่ใช่ทักษะเสริมอีกต่อไป

### ตารางน้ำหนักคะแนนหลัก 5 Domain

| # | Domain | น้ำหนัก | สิ่งที่วัดโดยสรุป |
|---|---|---|---|
| 1.0 | **Architecture** | **20%** | อ่าน Business Requirement แล้วออกแบบ Solution ที่เหมาะสม, วิเคราะห์ Fault Domain, เลือก Routing Protocol/Topology ที่ถูกต้อง |
| 2.0 | **Infrastructure** | **30%** | Config Layer 2/Layer 3/Wireless/Network Services ระดับ Enterprise เต็มรูปแบบ — Domain ที่มีน้ำหนักสูงสุด |
| 3.0 | **Network Assurance** | **10%** | Telemetry (Model-driven), Monitoring, Device Management, Visibility |
| 4.0 | **Security** | **15%** | Infrastructure Security, AAA, Zero Trust Workflow, Segmentation |
| 5.0 | **Automation** | **25%** | Python, YANG/NETCONF/RESTCONF, EEM, Git, Ansible/Puppet/Chef/SaltStack — น้ำหนักสูงเป็นอันดับ 2 |
| — | **รวม** | **100%** | — |

> **ข้อสังเกตสำคัญที่สุด**: Infrastructure (30%) + Automation (25%) = **55% ของคะแนนทั้งหมด**
> รวมกันมากกว่าครึ่ง — นี่คือสัญญาณชัดเจนว่า CCIE EI ยุคปัจจุบัน **ไม่ใช่แค่ "Router/Switch Config
> เก่งๆ" อีกต่อไป แต่ต้อง Automate การทำงานนั้นได้ด้วยความคล่องแคล่วเทียบเท่ากับ Manual CLI**
> Automation ที่มีน้ำหนัก 25% สูงกว่า Security (15%) และสูงกว่า Network Assurance (10%) รวมกันเสีย
> อีก — Candidate ที่มองข้าม Domain นี้เพราะคิดว่า "ไม่ใช่ของจริง เหมือนแค่ Bonus" มักเสียคะแนน
> มากกว่าที่คาดไว้มาก (ดู Step 707 และ Step 708 ต่อไป)

### รายละเอียด Sub-topic เต็มรูปแบบต่อ Domain

**Domain 1.0 — Architecture (20%)**

| Sub-topic | รายละเอียด |
|---|---|
| 1.1 | วิเคราะห์ Requirement ทางธุรกิจ/เทคนิค แล้วเลือก Solution ที่เหมาะสม (รวม Operational Constraint: คน/กระบวนการ/เครื่องมือที่ลูกค้ามีอยู่) |
| 1.2 | ออกแบบ Solution ให้ตรงกับ Requirement — เลือกใช้ Feature/Protocol ที่ "พอดี" ไม่ใช่ที่ "ซับซ้อนที่สุด" |
| 1.3 | ออกแบบ Layer 3 Infrastructure (IGP Selection, Summarization Boundary, Route Filtering Strategy) |
| 1.4 | วิเคราะห์ Design แล้วระบุ Fault Domain / Failure Point ที่อาจเกิดขึ้น |
| 1.5 | ออกแบบ Solution สำหรับการบริหารจัดการเครือข่าย (Management Plane Design) |
| 1.6 | ออกแบบ Mechanism สำหรับควบคุมทิศทาง Traffic (Traffic Engineering, PBR, Route Manipulation) |
| 1.7 | ออกแบบ Solution สำหรับ High Availability (Redundancy ทุกระดับ, NSF/SSO) |
| 1.8 | เปรียบเทียบ Routing Protocol Technology ให้เหมาะกับ Use Case (OSPF vs EIGRP vs BGP vs Static) |

**Domain 2.0 — Infrastructure (30%)**

| Sub-topic | รายละเอียด |
|---|---|
| 2.1 | Layer 2 — VLAN, Trunking, STP/RSTP/MST, EtherChannel, StackWise/VSS |
| 2.2 | Layer 3 — OSPF, EIGRP, BGP (Full-scale), Route Redistribution, VRF-Lite, Policy-Based Routing |
| 2.3 | Wireless — CAPWAP, Roaming, RRM, WLC High Availability, Wireless Security ระดับ Enterprise |
| 2.4 | Network Services — FHRP, DHCP, NTP, QoS End-to-End, Multicast (PIM/IGMP/MSDP) |
| 2.5 | Infrastructure Automation & Orchestration — ใช้ Automation Tool ตั้งค่า Infrastructure จริงในโจทย์ |

**Domain 3.0 — Network Assurance (10%)**

| Sub-topic | รายละเอียด |
|---|---|
| 3.1 | วินิจฉัยปัญหาเครือข่ายด้วย Traditional (SNMP/Syslog) และ Model-driven Telemetry |
| 3.2 | Config Data Collection สำหรับ Network Telemetry (YANG Push, gRPC, Streaming Telemetry) |
| 3.3 | Implement Device Management และ Monitoring (NETCONF/RESTCONF-based) |
| 3.4 | Implement Visibility Solution (NetFlow/Flexible NetFlow, Application Visibility) |

**Domain 4.0 — Security (15%)**

| Sub-topic | รายละเอียด |
|---|---|
| 4.1 | ออกแบบ Network Security ให้เหมาะกับ Threat Model และ Requirement |
| 4.2 | Implement Security บน Router/Switch (ACL, CoPP, Device Hardening) |
| 4.3 | Config และ Verify Infrastructure Security Feature (Port Security, DHCP Snooping, 802.1X) |
| 4.4 | Implement AAA (TACACS+/RADIUS, Method List, Authorization Policy) |
| 4.5 | Implement Zero Trust Workflow (Micro-segmentation, TrustSec/SGT, Identity-based Policy) |

**Domain 5.0 — Automation (25%)**

| Sub-topic | รายละเอียด |
|---|---|
| 5.1 | อ่านและแก้ไข Python Script พื้นฐานที่เกี่ยวกับ Network Automation |
| 5.2 | สร้าง/แก้ไข YANG Data Model ให้ถูกต้อง |
| 5.3 | อ่านและแปลงข้อมูลรูปแบบ XML, JSON, YAML |
| 5.4 | อธิบายแนวคิด API (NETCONF, RESTCONF, gNMI) และใช้งานจริงผ่าน Postman/Script |
| 5.5 | สร้าง EEM Applet เพื่อ Automate การตอบสนองต่อ Event บนอุปกรณ์ |
| 5.6 | ใช้ Git พื้นฐาน (clone/commit/push/branch) จัดการ Configuration as Code |
| 5.7 | เปรียบเทียบ Agent-based vs Agentless Orchestration (Ansible vs Puppet/Chef/SaltStack) |
| 5.8 | อธิบายความสามารถของ Network Configuration Management Tool ในภาพรวม |

---

## Step 703 — 4 รูปแบบงานในห้องสอบ CCIE: Design, Deploy, Optimize/Operate, Diagnose

ข้อสอบ CCIE EI Lab Exam อย่างเป็นทางการแบ่งเป็น 2 Module ใหญ่ตามเวลา (**Design Module ~3 ชั่วโมง**
และ **Deploy, Operate Module ~5 ชั่วโมง**) แต่ภายใน Module หลังนั้น**งานที่ Candidate ต้องทำจริง**
แยกออกเป็น**ลักษณะงาน (Task Type) 4 แบบที่ต่างกันโดยพื้นฐาน** — หลักสูตรนี้จะสอนทักษะทั้ง 4 แบบ
แยกกันให้ชัดเจนที่สุดใน Part 95-98 (CCIE Full Lab Scenario #1-4) เพราะแต่ละแบบต้องใช้ "กระบวนการ
คิด" ที่ต่างกันโดยสิ้นเชิง แม้จะใช้ความรู้ Protocol เดียวกันก็ตาม

### ตารางสรุป 4 รูปแบบงาน

| Task Type | อยู่ใน Module ทางการ | คำถามหลักที่โจทย์ถาม | ทักษะที่วัด | จำลองใน |
|---|---|---|---|---|
| **1. Design** | Design Module (3 ชม.) | "ถ้าลูกค้าต้องการ X ภายใต้ข้อจำกัด Y จะออกแบบอย่างไร" | อ่าน Requirement → เลือก Topology/Protocol → อธิบายเหตุผล (มักตอบผ่าน Diagram/เอกสาร ไม่ต้อง Config จริง) | Part 96 |
| **2. Deploy** | Deploy, Operate Module (5 ชม.) | "Implement Design ที่กำหนดให้ Config จริงบนอุปกรณ์" | แปลง Design เป็น Running-config ที่ถูกต้อง ครบ Requirement ทุกข้อ ภายในเวลาจำกัด | Part 97 |
| **3. Optimize / Operate** | Deploy, Operate Module (5 ชม.) | "เครือข่ายทำงานอยู่แล้ว แต่ต้องปรับให้ดีขึ้น/รองรับ Requirement ใหม่ โดยกระทบ Traffic เดิมน้อยที่สุด" | แก้ไข Config ที่มีอยู่แบบ **Non-disruptive**, ปรับ Performance/HA/Scale | Part 98 |
| **4. Diagnose (Troubleshoot)** | ฝังอยู่ใน Deploy, Operate Module | "เครือข่ายมีปัญหาอยู่แล้ว หาสาเหตุและแก้ไขให้ตรง Root Cause" | วินิจฉัยปัญหาข้าม Layer/Domain ภายในเวลาจำกัดต่อ Ticket | Part 95 |

> **หมายเหตุความถูกต้องเชิงเอกสาร**: โครงสร้าง Module อย่างเป็นทางการของ CCIE EI Lab Exam มีเพียง
> **2 Module ตามเวลา** (Design 3 ชม. + Deploy,Operate 5 ชม.) — Cisco ไม่ได้แยก "Diagnose" เป็น
> Module เวลาต่างหากแบบที่ CCIE R&S รุ่นเก่าเคยทำ (Troubleshooting Module แยก 2 ชม.) แต่ **งาน
> Diagnose ถูกฝังอยู่ในรูปของ Ticket ภายใน Deploy, Operate Module** — หลักสูตรนี้แยกมันออกมาเป็น
> "4 รูปแบบงาน" (ไม่ใช่ "4 Module เวลา") เพื่อจุดประสงค์ **การสอน** เท่านั้น ให้ผู้เรียนฝึกแต่ละทักษะ
> แบบเข้มข้นทีละตัวก่อนไปเจอโจทย์ผสมจริงที่รวมทั้ง 4 อย่างไว้ในเวลาเดียวกัน (ตามที่ข้อสอบจริงทำ)

### ไทม์ไลน์ 8 ชั่วโมง (ภาพประกอบตามโครงสร้างทางการ)

```
เวลา 0h ────────────────── 3h ──────────────────────────────────────── 8h
     │                      │                                          │
     │   DESIGN MODULE      │           DEPLOY, OPERATE MODULE          │
     │   (~3 ชั่วโมง)         │              (~5 ชั่วโมง)                  │
     │                      │                                          │
     │ - อ่าน Requirement    │ - Deploy: Config ตาม Design ที่ให้มา       │
     │ - เลือก Topology/     │ - Operate: ปรับ Config เดิมแบบ            │
     │   Protocol ที่เหมาะสม │   Non-disruptive                          │
     │ - ตอบผ่าน Diagram/    │ - Optimize: ปรับ Performance/Scale        │
     │   คำอธิบาย (ไม่ Config│ - Diagnose: Ticket แก้ปัญหาที่ฝังอยู่ใน     │
     │   จริงมากนัก)         │   Topology (มักอ้างอิงงานที่ทำใน Deploy   │
     │                      │   ก่อนหน้าในห้องสอบเดียวกัน)                │
     └──────────────────────┴──────────────────────────────────────────┘
```

จุดที่ Candidate มักพลาดคือ **มองว่า Design Module "เบา" เพราะไม่ต้อง Config** แล้วรีบข้ามไปโฟกัส
Deploy — แต่ในความจริง **คำตอบที่ผิดใน Design Module มักส่งผลต่อคะแนนใน Deploy, Operate Module
ด้วย** เพราะบางข้อสอบให้ Candidate ต้อง Deploy ตาม Design ของตัวเองที่ตอบไปแล้วในช่วงแรก — ถ้า
Design ผิดตั้งแต่ต้น การ Deploy ที่ "ถูกตาม Design ผิด" ก็จะไม่ตรง Requirement จริงไปด้วย

---

## Step 704 — CCIE คิดต่างจาก CCNP อย่างไร: ตัวอย่าง OSPF Area Design

### หลักการความต่างโดยสรุป

| | CCNP ถาม | CCIE ถาม |
|---|---|---|
| รูปแบบคำถาม | **"จะ Config X อย่างไร"** | **"ให้ Business Requirement + ข้อจำกัดมา จะออกแบบและ Implement Solution ที่ดีที่สุดอย่างไร แล้วพร้อมอธิบาย/ป้องกันเหตุผลของ Design ที่เลือก"** |
| จุดโฟกัส | ความถูกต้องของ Syntax/Feature เดี่ยว | ความเหมาะสมของ **ทางเลือก** ท่ามกลางหลายทางที่เป็นไปได้ ภายใต้ Constraint จริง |
| คำตอบที่ถูก | มักมีคำตอบเดียวที่ถูกต้องชัดเจน | มัก**มีหลายคำตอบที่ "ใช้งานได้"** แต่มีคำตอบที่ "ดีที่สุดตาม Requirement ที่ให้" เพียงคำตอบเดียว/ไม่กี่คำตอบ |
| การประเมิน | ถูก/ผิด | ถูก/ผิด **+ เหตุผลสนับสนุนที่ตรงกับ Requirement + Trade-off ที่ยอมรับได้** |

### ตัวอย่างเปรียบเทียบจริง — หัวข้อเดียวกัน: OSPF Area Design

**คำถามสไตล์ CCNP (แบบที่เคยเจอใน Part 11-12, 26)**

> "Router `DIST-SW3` ต้องเข้าร่วม OSPF Area 20 กับ `CORE-SW2` จง Config `router ospf 1` พร้อม
> ประกาศ Network Statement ที่ถูกต้องให้ Adjacency ขึ้นเป็น FULL"

โจทย์แบบนี้มี**คำตอบเดียว**: ใส่ `network 10.255.30.0 0.0.0.3 area 20` (หรือเทียบเท่าด้วย
`ip ospf 1 area 20` บน Interface) แค่นั้น — วัดว่า "รู้ Syntax ถูกไหม"

**คำถามสไตล์ CCIE (ระดับที่จะเจอใน Part 73, 96)**

> "องค์กรมี Site ทั้งหมด 45 แห่งทั่วประเทศ แบ่งเป็น 5 ภูมิภาค ภูมิภาคละ 9 Site โดยเฉลี่ย แต่ละ Site
> มี Subnet ท้องถิ่นของตัวเอง ผู้บริหารกำหนดว่า **การเปลี่ยนแปลง Topology ในภูมิภาคหนึ่ง (เช่น Site
> ใหม่เข้ามา, Link ล้ม) ต้องไม่ทำให้เกิด SPF Recalculation ที่กระทบ Router ในภูมิภาคอื่น** และทีม
> Operation ที่ดูแลแต่ละภูมิภาคมีความเชี่ยวชาญด้าน Routing ไม่เท่ากัน (บางทีมเพิ่งเริ่มงาน) จง
> ออกแบบ OSPF Area Structure ที่เหมาะสม พร้อม Config บนอุปกรณ์ Backbone ตามที่ออกแบบ และเตรียม
> คำอธิบายพร้อมป้องกันเหตุผลของ Design ที่เลือก"

โจทย์แบบนี้**ไม่มีคำตอบสำเร็จรูปให้ท่องจำ** — ต้องผ่านกระบวนการคิดหลายชั้น:

1. **จับ Requirement ที่ซ่อนอยู่**: "ต้องไม่กระทบ SPF ข้ามภูมิภาค" = ต้องมี **Area Boundary ที่
   ตรงกับขอบภูมิภาค** (Multi-area Design, ไม่ใช่ Single Area ทั้งองค์กร) เพราะ SPF Recalculation
   เกิดเฉพาะภายใน Area เดียวกันเท่านั้น (Inter-area เปลี่ยนแค่ Distance Vector-like ผ่าน ABR)
2. **จับ Constraint ที่ซ่อนอยู่**: "ทีม Operation ความเชี่ยวชาญไม่เท่ากัน" = Design ต้อง**ง่ายพอให้
   ทีมที่เพิ่งเริ่มงานดูแลได้** → เลี่ยง Design ที่ซับซ้อนเกินจำเป็น (เช่น ไม่ควรทำ Area แบบ
   Virtual-link ซับซ้อนถ้าไม่จำเป็นจริงๆ)
3. **ตัดสินใจ**: แบ่งเป็น **Area 0 (Backbone) + Area ต่อภูมิภาคๆละ 1 Area (Area 10-50)** โดยให้
   Router ระดับภูมิภาคทำหน้าที่ ABR เชื่อม Area ภูมิภาคของตัวเองเข้ากับ Backbone — Site ใหม่ที่
   เข้ามาในภูมิภาคใด จะ Trigger SPF แค่ใน Area ของภูมิภาคนั้น ไม่กระทบภูมิภาคอื่น (ตรง Requirement
   ข้อ 1 เป๊ะ)
4. **เพิ่ม Safeguard**: ใส่ **Route Summarization ที่ ABR ทุกตัว** (`area X range ...`) เพื่อลด
   จำนวน LSA ที่ส่งเข้า Backbone และลด Router ที่ต้อง SPF ซ้ำเมื่อมีการเปลี่ยนแปลงระดับ Subnet
   ย่อยภายใน Area
5. **เตรียมป้องกันคำถามตาม**: ถ้าถูกถามว่า "ทำไมไม่ใช้ EIGRP ที่มี Query Boundary อยู่แล้ว" ต้อง
   ตอบได้ว่า Requirement ไม่ได้บอกว่าใช้ Protocol อะไรอยู่แล้ว แต่ถ้าองค์กรมี Multi-vendor Router
   ปนอยู่ (ซึ่งบางโจทย์จะแฝง Constraint นี้มาด้วย) OSPF (Open Standard) จะเหมาะกว่า EIGRP (Cisco
   Proprietary เดิม แม้จะเปิดบางส่วนแล้ว)

> **สรุปหลักการ**: คำถาม CCNP วัดว่า **"คุณ Config ถูกไหม"** ส่วนคำถาม CCIE วัดว่า **"คุณอ่านออก
> ไหมว่า Requirement ที่ให้มาบอกอะไรเป็นนัย, เลือกทางที่เหมาะสมที่สุดได้ไหม (ไม่ใช่แค่ทางที่ใช้งาน
> ได้), Implement มันถูกต้องไหม, และอธิบาย/ป้องกันการเลือกนั้นได้ไหมเมื่อถูกถามซ้ำ"** — ทักษะนี้
> ฝึกจากการทำ Design Exercise ซ้ำๆ ไม่ใช่จากการท่อง Syntax เพิ่ม

---

## Step 705 — สภาพแวดล้อม Lab Exam จริง: อุปกรณ์, จำนวน Device, ความจริงเรื่องเวลา

### Platform ที่ใช้ในห้องสอบจริง (Remote Lab ผ่าน Cisco)

| Platform | ใช้แทนอุปกรณ์ | หมายเหตุ |
|---|---|---|
| **Cisco IOS-XE** (Catalyst 9000 Series, ISR/ASR) | Router, Switch หลัก ส่วนใหญ่ของ Topology | เป็น OS หลักที่ใช้ตลอดทั้งข้อสอบ (ตรงกับที่หลักสูตรนี้ Config มาตั้งแต่ Part 2) |
| **Cisco NX-OS** (Nexus Series) | Data Center-facing Device บางจุด (โจทย์ Cloud/DC Interconnect) | Syntax ต่างจาก IOS-XE พอสมควร ต้องฝึกแยก (จะแนะนำใน Part 84-87, 93) |
| **Cisco Catalyst 9800 WLC (IOS-XE Wireless)** | Wireless Controller | ตรงกับที่เรียนใน Part 20, 36-37 ต่อยอดใน Part 89 |
| **Cisco DNA Center / Catalyst Center** | Automation/SD-Access Controller (บางโจทย์) | ต่อยอดจาก Part 50, 77 |

### จำนวนอุปกรณ์ใน Topology จริง

Topology ของ CCIE EI Lab Exam โดยทั่วไปประกอบด้วยอุปกรณ์ **หลายสิบตัว** (มักอยู่ในช่วง 20-40+
Device ตามธีมของ Task แต่ละชุด) ครอบคลุม:

```
┌─────────────────────────────────────────────────────────────────┐
│                  ตัวอย่างสัดส่วน Device ใน Lab Topology จริง         │
├─────────────────────────────────────────────────────────────────┤
│  Core / Backbone Router-Switch      : 4-8  เครื่อง                │
│  Distribution / Aggregation Switch  : 6-10 เครื่อง                │
│  Access / Edge Switch               : 6-10 เครื่อง                │
│  WAN Edge Router (Site-to-Site)     : 4-8  เครื่อง                │
│  Wireless LAN Controller            : 1-2  เครื่อง                │
│  Firewall / Security Appliance      : 1-2  เครื่อง (บางโจทย์)      │
│  End-host / Server จำลอง             : 2-6  เครื่อง                │
├─────────────────────────────────────────────────────────────────┤
│  รวมโดยประมาณ                        : 25-45+ อุปกรณ์               │
└─────────────────────────────────────────────────────────────────┘
```

เทียบกับ Enterprise Lab ของหลักสูตรนี้ (2 Core + 4 Dist + 4-8 Access + 2 WAN Edge + 1 FW + 1
WLC+AP ≈ 15 Device หลัก) จะเห็นว่า **CCIE Lab ใหญ่กว่าประมาณ 2-3 เท่า** และมี**การเชื่อมต่อ
ข้าม Domain มากกว่า** (Routing+Wireless+Security+Automation ทำงานพร้อมกันในโจทย์เดียว) — นี่คือ
เหตุผลที่ Part 99 (Capstone: Build Enterprise Network From Scratch) จะขยาย Lab ของเราให้ใกล้เคียง
Scale นี้มากที่สุดก่อนสอบจริง

### ความจริงเรื่องเวลา — "ผู้สอบส่วนใหญ่ไม่ได้แพ้เพราะไม่รู้ แต่แพ้เพราะเวลาหมด"

สถิติและ Feedback จาก Candidate ที่สอบ CCIE Lab ทั่วโลก (ทุก Track ไม่จำกัดแค่ EI) ชี้ไปทางเดียวกัน
อย่างสม่ำเสมอ:

- คนที่สอบตกจำนวนมาก **ไม่ใช่เพราะไม่รู้วิธี Config Feature ที่โจทย์ถาม** — ส่วนใหญ่รู้วิธีทำ
  ถูกต้องเป๊ะถ้าให้เวลาไม่จำกัด
- สาเหตุที่แท้จริงคือ **บริหารเวลาผิด**: ติดอยู่กับ Task ข้อเดียวนานเกินไป (มักเป็น Task ที่ตัวเอง
  "อยากทำให้สมบูรณ์แบบ" มากกว่าที่จำเป็น) จนไม่มีเวลาเหลือให้ Task ข้ออื่นที่ง่ายกว่าแต่มีคะแนน
  เท่ากัน
- Task บางข้อ**ถูกออกแบบมาให้ "ยอมรับ Good Enough"** ไม่ใช่ "Perfect" — Candidate ที่พยายาม
  Optimize เกินความจำเป็น (เช่น ปรับ QoS Policy ให้สมบูรณ์แบบทุกรายละเอียดทั้งที่โจทย์แค่ขอ
  Baseline) จะเสียเวลาไปกับ Task ที่ให้ผลตอบแทนต่อนาทีต่ำ

> **บทเรียนที่ต้องจำ**: ทักษะการบริหารเวลาต้องฝึกคู่กับทักษะเทคนิคเสมอ — Part 95-98 (Full Lab
> Scenario) ของหลักสูตรนี้จะจับเวลาทุกครั้งเหมือนสอบจริง เพื่อฝึกสัญชาตญาณ **"เมื่อไหร่ต้องปล่อย
> Task นี้ไปก่อนแล้วกลับมาทำทีหลัง"** ซึ่งเป็นทักษะที่วัดจากการฝึกซ้ำเท่านั้น ไม่มีทางลัดจากการอ่าน
> ทฤษฎี

---

## Step 706 — กลยุทธ์การฝึกฝนสำหรับ CCIE: ทำไม End-to-End Lab ดีกว่า Isolated Feature Practice

### ปัญหาของการฝึกแบบ Isolated Feature Practice

วิธีฝึกที่ CCNP Candidate ส่วนใหญ่คุ้นเคยคือ **ฝึกทีละ Feature แบบแยกส่วน**: เปิด Topology เล็กๆ
2-3 อุปกรณ์ ฝึก Config OSPF ให้ขึ้น FULL, ปิด Lab, เปิด Topology ใหม่ฝึก BGP, ปิด Lab, ฝึก DMVPN
ต่อ — วิธีนี้**ใช้ได้ดีสำหรับ CCNP** เพราะข้อสอบ CCNP ก็ถามแบบแยก Feature เช่นกัน

แต่วิธีนี้ **ใช้ไม่ได้ผลกับ CCIE** เพราะเหตุผลสำคัญ 3 ข้อ:

1. **CCIE Lab ไม่เคยถาม Feature แบบแยกโดด** — ทุก Task เชื่อมกับ Task อื่นเสมอ (เช่น ถ้า OSPF
   Design ผิดตั้งแต่ Task แรก Task ที่ 15 ที่ต้องใช้ Route Redistribution บน Topology เดียวกันจะ
   พังไปด้วย) — ฝึกแบบแยกส่วนจะไม่เคยเจอปัญหา "Interaction ข้าม Feature" ที่เป็นด่านจริงของข้อสอบ
2. **ทักษะการวินิจฉัยปัญหาข้าม Layer ต้องมี Topology ที่ซับซ้อนพอ** — Topology 2-3 อุปกรณ์ไม่มี
   ทางจำลองอาการ "Query Scope กว้างเกินไปเพราะ Topology ใหญ่" หรือ "SPF ในภูมิภาคหนึ่งกระทบ Timer
   ของอีกภูมิภาค" ได้เลย
3. **ทักษะบริหารเวลา (Step 705) ฝึกไม่ได้ถ้า Lab เล็กเกินไป** — Topology เล็กทำเสร็จเร็วเกินไป
   จนไม่รู้สึกถึงความกดดันของเวลาจริง

### ทำไม Enterprise Lab ต่อเนื่องของหลักสูตรนี้ (Part 1 → Part 100) คือแนวทางที่ถูกต้อง

หลักสูตรนี้ออกแบบมาให้ **Lab Topology เดียวกันขยายต่อเนื่องตั้งแต่ Part 1** (ตามที่ประกาศไว้ใน
[Part 1 Step 10](part-001-networking-fundamentals.md)) ไม่ใช่เรื่องบังเอิญ — นี่คือ**การฝึกแบบ
End-to-End Lab ที่ CCIE ต้องการโดยธรรมชาติ**:

| ข้อดีของ Lab ต่อเนื่อง | เทียบกับ Isolated Practice |
|---|---|
| ทุก Feature ใหม่ต้อง**ทำงานร่วมกับของเดิมที่มีอยู่แล้ว** (เช่น เพิ่ม MPLS L3VPN ใน Part 86 ต้องไม่พัง OSPF/BGP ที่ตั้งมาตั้งแต่ Part 11-30) | Isolated Practice ไม่เคยเจอ Conflict แบบนี้ เพราะ Lab ถูกลบทุกครั้ง |
| ฝึกความรู้สึก "เครือข่ายที่มีอายุและ History" — Config เก่าที่อาจไม่ใช่ Best Practice ล่าสุดแต่ยังใช้งานอยู่ | ตรงกับสภาพเครือข่าย Production จริงที่ CCIE ต้อง Operate เสมอ |
| Topology ขยายใหญ่ขึ้นเรื่อยๆ (Part 99 จะใกล้เคียง Scale ข้อสอบจริง 25-45+ Device) | จำลองความกดดันเรื่องเวลาและความซับซ้อนได้จริง |
| จำ IP Plan/Hostname เดิมได้ (ดู [00-ip-address-plan.md](00-ip-address-plan.md)) จนคล่องเหมือนเครือข่ายของตัวเอง | ตรงกับที่ CCIE Candidate ต้อง "รู้จัก" Topology ข้อสอบภายในเวลาไม่กี่นาทีแรก |

### Recommended Practice Platform

| Platform | ข้อดี | เหมาะกับ |
|---|---|---|
| **Cisco Modeling Labs (CML)** | Cisco Official, รองรับ IOS-XE/NX-OS/WLC Image เต็มรูปแบบ, ใกล้เคียง Environment ข้อสอบจริงที่สุด | การฝึกช่วงท้าย (Part 90+) ก่อนสอบจริง |
| **EVE-NG (Professional/Community)** | รองรับ Image จริงหลากหลาย (ต้องมี Image เอง), Community ใหญ่, ราคาถูกกว่า CML | การฝึกตลอดหลักสูตรตั้งแต่ Part 26 เป็นต้นไป (ระดับ ENCOR/ENARSI/CCIE) |
| **GNS3** | ฟรี, รองรับ Real IOS Image | ทางเลือกสำหรับ Part ที่ไม่ต้องใช้ NX-OS/WLC เฉพาะทาง |
| **dCloud / Cisco Sandbox** | Lab สำเร็จรูปบางหัวข้อ (Automation, ISE, DNA Center) | เสริมสำหรับหัวข้อที่ Setup เองยาก (Part 78, 79, 50) |

> **คำแนะนำจากหลักสูตรนี้**: ใช้ **EVE-NG หรือ CML พร้อม Image จริง** ตั้งแต่เริ่ม Part 71 เป็นต้น
> ไป (ถ้ายังไม่ได้ใช้มาตั้งแต่ ENCOR) เพราะ Packet Tracer **ไม่รองรับ Feature ระดับ CCIE ส่วนใหญ่**
> (VXLAN/EVPN, MPLS L3VPN เต็มรูปแบบ, YANG/NETCONF บางส่วน) — และให้คง Lab เดิมของหลักสูตรไว้
> ต่อเนื่อง อย่าลบแล้วสร้างใหม่ทุก Part

---

## Step 707 — บทบาท Automation ใน CCIE ยุคใหม่: ไม่ใช่แค่ "Nice to Have" อีกต่อไป

### ทำไม Automation ถึงมีน้ำหนัก 25% — สูงเป็นอันดับ 2 ของ Blueprint

CCIE EI v1.1 เพิ่มน้ำหนัก Domain Automation ขึ้นอย่างมีนัยสำคัญเทียบกับรุ่นก่อน สะท้อนความจริงของ
อุตสาหกรรมที่ Network Engineer ระดับ Expert ในปัจจุบัน**ไม่ได้ Config อุปกรณ์ทีละตัวด้วยมือ**อีก
ต่อไปเมื่อ Scale ของเครือข่ายใหญ่ขึ้น — องค์กรระดับที่จ้าง CCIE คาดหวังให้คนระดับนี้**ออกแบบ
กระบวนการ Automate การ Deploy/Verify/Remediate** ได้ ไม่ใช่แค่ Config เก่งคนเดียว

### เชื่อมโยงกับ Part ที่เรียนมาแล้วในหลักสูตรนี้

| Part ที่เรียนมาแล้ว | ทักษะที่ได้ | ความเกี่ยวข้องกับ CCIE Domain 5.0 |
|---|---|---|
| [Part 23](part-023-automation-basics.md) — Automation Basics | NETCONF/RESTCONF/Ansible เบื้องต้น | ฐานของ Sub-topic 5.4, 5.7 |
| [Part 47](part-047-automation-python-advanced.md) — Python for Network Engineers | เขียน/แก้ไข Python Script จริง (Netmiko, NAPALM) | ตรงกับ Sub-topic **5.1 โดยตรง** |
| [Part 48](part-048-automation-ansible-advanced.md) — Ansible for Network Engineers | Playbook, Inventory, Idempotency | ตรงกับ Sub-topic **5.7 โดยตรง** |
| [Part 49](part-049-netconf-restconf-yang-deep-dive.md) — YANG/NETCONF/RESTCONF Deep Dive | YANG Model, NETCONF `<edit-config>`, RESTCONF CRUD | ตรงกับ Sub-topic **5.2, 5.3, 5.4 โดยตรง** |
| [Part 50](part-050-dna-center.md) — DNA Center / Catalyst Center | Intent-based Automation, Template, API | เสริม Sub-topic 5.8 |
| [Part 67](part-067-automation-troubleshooting.md) — Automation Troubleshooting | Debug Script/API ที่ทำงานผิดปกติ | เสริม Domain 3.0 (Assurance) + 5.0 |
| **Part 79 (กำลังจะเรียน)** — Full-Stack Automation (Python+Ansible+CI/CD) | รวม Python+Ansible+Git+CI/CD Pipeline เป็นระบบเดียว | ครอบคลุม Sub-topic **5.1, 5.5, 5.6, 5.7 ครบวง** |

### ทำไม Part 47-49 (ที่เรียนไปแล้วใน ENCOR) และ Part 79 (ที่กำลังจะเรียน) ถึง "สอบได้จริง" ไม่ใช่แค่ทฤษฎี

จุดที่ Candidate จำนวนมากเข้าใจผิดคือคิดว่า Automation ใน CCIE Lab เป็นแค่ "หัวข้อเสริมที่ถามน้อย"
แต่ความจริงคือ:

- โจทย์ Deploy/Optimize บางข้อ**บังคับให้ต้องใช้ Ansible Playbook หรือ Python Script**ในการ
  Config อุปกรณ์จำนวนมากพร้อมกัน (ทำด้วยมือทันเวลาไม่ได้จริงถ้า Device เยอะและเวลาน้อย)
- โจทย์ Network Assurance (Domain 3.0) มักผูกกับ Domain 5.0 โดยตรง — ต้อง Config **YANG Push
  Telemetry** ผ่าน NETCONF แล้วอ่านผลลัพธ์ JSON/XML กลับมาตีความ
- EEM Applet (Sub-topic 5.5) มักถูกใช้ใน Task ประเภท **"ให้ Config ตอบสนอง Event อัตโนมัติโดยไม่
  ต้องมีคนเข้าไปกดคำสั่งเอง"** ซึ่งเป็นรูปแบบ Automation ที่ฝังอยู่ใน Device เอง ไม่ต้องพึ่ง
  External Controller

> **ข้อสรุปสำคัญ**: ทักษะ Automation ที่หลักสูตรนี้สอนใน Part 47-49 (ENCOR) และกำลังจะขยายเพิ่มใน
> Part 79 (CCIE) **ไม่ใช่ทักษะเสริมนอกสายอีกต่อไป** แต่เป็น **1 ใน 4 เสาหลักของคะแนนสอบ CCIE
> เทียบเท่า Infrastructure Domain (30%)** — Candidate ที่ยังมองว่า "ข้ามได้ ไปโฟกัส Routing/
> Switching อย่างเดียวพอ" กำลังยอมเสียคะแนนไปเกือบ 1 ใน 4 ของข้อสอบทั้งฉบับโดยไม่จำเป็น

---

## Step 708 — จุดพลาดคลาสสิกของผู้สอบ CCIE

### 1. Over-focusing บน Domain ที่ตัวเองถนัด แล้วละเลย Domain อื่น

Candidate ที่มีพื้นฐาน Routing/Switching แข็งมาก (ผ่าน ENCOR/ENARSI มาอย่างเข้มข้น) มักมีแนวโน้ม
**ทุ่มเวลาฝึกซ้อม Routing Protocol Design ซ้ำๆ** เพราะรู้สึกมั่นใจและสนุกกับมัน แต่ **ละเลย Domain
Automation (25%) และ Network Assurance (10%)** เพราะรู้สึกไม่คุ้นเคย — ผลคือเสียคะแนนสะสมจาก
Domain ที่ไม่ถนัด**มากกว่า**คะแนนที่ทำได้เกินจาก Domain ที่ถนัดอยู่แล้ว (เพราะ Domain ที่ถนัดมัก
ทำคะแนนเต็มหรือใกล้เต็มอยู่แล้ว ทำเพิ่มไม่ได้อีกมาก)

### 2. Time Allocation ที่ไม่สัมพันธ์กับน้ำหนักคะแนน

จุดพลาดที่พบบ่อยที่สุดคือ **จัดสรรเวลาตาม "ความยากที่รู้สึก" ไม่ใช่ตาม "น้ำหนักคะแนนจริง"** —
เช่น ใช้เวลา 2 ชั่วโมงกับ Task Security ที่มีน้ำหนักรวมในโจทย์นั้นแค่ไม่กี่คะแนน เพราะ "รู้สึกว่า
ยาก อยากทำให้ถูก" ทั้งที่ Task Infrastructure ข้ออื่นที่ยังไม่แตะเลยมีน้ำหนักคะแนนสูงกว่ามาก
— วิธีแก้คือ **อ่านโจทย์ทั้งหมดก่อน แล้วประเมินน้ำหนักคะแนนต่อ Task คร่าวๆก่อนตัดสินใจลำดับการทำ**
ไม่ใช่ทำตามลำดับที่โจทย์เรียงมา

### 3. อ่าน Requirement ไม่ละเอียดพอ — กับดักคำศัพท์เฉพาะที่ต้องระวัง

นี่คือจุดพลาดที่ทำให้เสียคะแนนโดยไม่รู้ตัวมากที่สุด เพราะ **Config ที่เขียนไปนั้น "ทำงานได้" และ
"ดูสมเหตุสมผล" แต่ไม่ตรงกับ Requirement ที่ระบุไว้เป๊ะๆ** ตัวอย่างกับดักคำศัพท์ที่พบบ่อย:

| คำในโจทย์ | ความหมายที่ถูก | ความเข้าใจผิดที่พบบ่อย |
|---|---|---|
| "**ต้อง**ใช้ OSPF Area 0 เท่านั้น" | ห้ามสร้าง Area อื่นเด็ดขาด แม้จะดูเหมาะกว่าในมุมมอง Design | Candidate มักสร้าง Multi-area เพราะ "คิดว่าดีกว่า" ทั้งที่โจทย์ห้ามชัดเจน |
| "โดย**ไม่กระทบ Traffic ที่มีอยู่**" | Task ต้องทำแบบ Non-disruptive (ใช้ Technique เช่น Graceful, Maximum-paths ค่อยๆปรับ) | Candoendite มักใช้ `shutdown`/`no router ospf` แล้ว Config ใหม่ทั้งหมด ซึ่งกระทบ Traffic ทันที |
| "Router X **ควร**เป็น Primary" (ใช้คำว่า "ควร" ไม่ใช่ "ต้อง") | เป็น Preference ไม่ใช่ Hard Requirement — ให้ Design ตาม Best Practice ทั่วไปได้ ถ้ามีเหตุผลต่างจากนี้ต้องพร้อมอธิบาย | Candidate มักตีความ "ควร" เป็น "ต้อง" แบบเข้มงวดเกินจำเป็น เสียเวลา Design ซับซ้อนเกิน |
| "**เฉพาะ** VLAN 10 และ 20" | ห้ามลืมและห้ามเกิน — ทั้งสองคำ (เฉพาะ + รายชื่อ) ต้องตรงกันแบบเป๊ะ | Candidate มักลืมอ่านคำว่า "เฉพาะ" แล้ว Config ครอบคลุม VLAN อื่นด้วย "เผื่อไว้" ซึ่งกลายเป็นผิด Requirement |

> **บทเรียน**: คำสั่งเล็กๆอย่าง "ต้อง" vs "ควร", "เฉพาะ" vs "รวมถึง", "ไม่กระทบ" vs "ปรับปรุง" คือ
> จุดที่ Cisco Grading Rubric ใช้แยกคำตอบถูก/ผิดอย่างเข้มงวด — ต้องฝึกอ่านโจทย์แบบ **Legal
> Document ที่ทุกคำมีน้ำหนัก** ไม่ใช่อ่านแบบเร็วๆแล้วรีบ Implement ตามความเข้าใจคร่าวๆ

### 4. ไม่ Verify/Test หลัง Implement ก่อนย้าย Task ถัดไป

จุดพลาดสุดท้ายที่พบบ่อยมากคือ **Config แล้วเชื่อว่าถูก โดยไม่ทดสอบจริง** ก่อนย้ายไป Task ถัดไป —
ปัญหาคือ CCIE Lab หลาย Task **ต่อเนื่องกัน** (Task ที่ 10 อาจสร้างบนสมมติฐานว่า Task ที่ 5 ทำงาน
ถูกต้องแล้ว) ถ้า Task ที่ 5 มี Bug เล็กๆที่ไม่ได้ Verify ผลกระทบจะลุกลามไปหลาย Task ข้างหน้าโดยไม่รู้
ตัวจนกว่าจะเสียเวลาไปมากแล้ว — วินัยที่ต้องฝึกให้เป็นอัตโนมัติคือ **`show`/`ping`/`traceroute`
Verify ทันทีหลังทุก Config Block สำคัญ** ไม่ใช่รอ Verify รวมทีเดียวท้ายชั่วโมง (ตรงกับหลักการ
Runbook ที่สอนมาตั้งแต่ Part 55-70 ระดับ ENARSI)

---

## Step 709 — แผนเนื้อหา Part 72-100 ของหลักสูตรนี้

Part 71 (Part นี้) วางภาพรวมเสร็จแล้ว — ต่อจากนี้หลักสูตรจะเดินหน้าตามแผน 2 ช่วงใหญ่:

### ช่วงที่ 1: Part 72-94 — เจาะลึกเทคโนโลยีที่มีน้ำหนักสูงใน Blueprint (23 Part)

```
┌────────────────────────────────────────────────────────────────────────┐
│         Part 72-94: เติมเทคโนโลยีที่ Blueprint ต้องการแต่ยังไม่ลึกพอ         │
├────────────────────────────────────────────────────────────────────────┤
│ Architecture (20%)   →  Part 72,73,76,77,84-87,92  (Design เชิงลึก)      │
│ Infrastructure (30%) →  Part 72-75,81-83,84-90     (Feature เชิงลึก)     │
│ Assurance (10%)      →  Part 80,91                 (Telemetry/วิธีวินิจฉัย)│
│ Security (15%)       →  Part 78,94                 (ISE/TrustSec/ZeroTrust)│
│ Automation (25%)     →  Part 79,80                 (Full-Stack/Telemetry) │
└────────────────────────────────────────────────────────────────────────┘
```

| Part | หัวข้อ | เติมเต็ม Domain |
|---|---|---|
| 72 | Large Campus Fabric Design (L2) | Architecture, Infrastructure |
| 73 | Large-Scale Routing Design (OSPF+BGP+EIGRP) | Architecture, Infrastructure — ต่อยอดโดยตรงจาก Step 704 |
| 74 | Advanced BGP for Enterprise WAN Edge | Infrastructure |
| 75 | Advanced Multicast for Enterprise | Infrastructure |
| 76 | SD-WAN Advanced Design (Viptela Deep Dive) | Architecture |
| 77 | SD-Access Advanced Design & Deployment | Architecture, Infrastructure |
| 78 | Security Integration (ISE, TrustSec, FTD, Umbrella) | Security |
| 79 | Full-Stack Automation (Python+Ansible+CI/CD) | Automation |
| 80 | Model-Driven Telemetry & Streaming | Automation, Network Assurance |
| 81 | High Availability (NSF/SSO, ISSU) | Architecture, Infrastructure |
| 82 | End-to-End Enterprise QoS Design | Infrastructure |
| 83 | Enterprise Multicast Deep Dive Advanced | Infrastructure |
| 84 | VXLAN/EVPN Fundamentals | Architecture, Infrastructure |
| 85 | VXLAN/EVPN Multisite Advanced | Architecture, Infrastructure |
| 86 | MPLS L3VPN Fundamentals | Architecture, Infrastructure |
| 87 | MPLS L3VPN Advanced & WAN Integration | Architecture, Infrastructure |
| 88 | IPv6 Enterprise Deployment Deep Dive | Infrastructure |
| 89 | Enterprise Wireless at Scale (High Density) | Infrastructure |
| 90 | Catalyst 9000 Advanced Features | Infrastructure |
| 91 | Advanced Troubleshooting Methodology | Network Assurance |
| 92 | Disaster Recovery & Network Resilience | Architecture |
| 93 | Cloud Connectivity (AWS/Azure/GCP, Cloud onRamp) | Architecture, Infrastructure |
| 94 | Zero Trust Network Architecture | Security |

### ช่วงที่ 2: Part 95-100 — Full CCIE-style Integrated Lab (6 Part)

```
┌────────────────────────────────────────────────────────────────────────┐
│    Part 95-99: ใช้ Enterprise Lab เดิม (สร้างต่อเนื่องจาก Part 1) เต็มรูปแบบ │
├────────────────────────────────────────────────────────────────────────┤
│  Part 95  →  Diagnose Module  (จำลอง Task Type "Diagnose" จาก Step 703) │
│  Part 96  →  Design Module    (จำลอง Task Type "Design")                │
│  Part 97  →  Deploy Module    (จำลอง Task Type "Deploy")                │
│  Part 98  →  Optimize Module  (จำลอง Task Type "Optimize/Operate")      │
│  Part 99  →  Capstone: Build Enterprise Network From Scratch           │
│              (รวมทั้ง 4 Task Type ในโจทย์เดียว เหมือนข้อสอบจริง 8 ชั่วโมง)  │
│  Part 100 →  Career Mastery: Exam Strategy & Real-World Scenarios      │
└────────────────────────────────────────────────────────────────────────┘
```

หลักการสำคัญของช่วงที่ 2: Part 95-98 จะฝึก**แต่ละ Task Type แยกกันแบบเข้มข้นก่อน** (ตาม Step 703)
เพื่อสร้างความคุ้นเคยกับ "วิธีคิด" ของแต่ละแบบ จากนั้น Part 99 จะ**รวมทั้ง 4 แบบเข้าด้วยกันในโจทย์
เดียว จับเวลาจริงแบบ 8 ชั่วโมง** บน Topology ที่ขยายมาจาก Part 1 จนครบทุก Technology ที่เรียนมา
ทั้ง 98 Part ก่อนหน้า — เป็นการซ้อมที่ใกล้เคียงสภาพข้อสอบจริงที่สุดที่หลักสูตรนี้จะให้ได้

---

## Step 710 — Full Lab: Gap Analysis Exercise

โจทย์ของ Full Lab นี้คือให้คุณสร้าง **ตาราง Self-Tracking ส่วนตัว** ที่ Mapping ทุกหัวข้อใน CCIE EI
v1.1 Blueprint (จาก Step 702) เข้ากับ Part ที่ครอบคลุมหัวข้อนั้น — แยกให้เห็นชัดว่าอะไร**ทำสำเร็จ
แล้ว (Part 1-70)** และอะไร**กำลังจะเรียน (Part 72-100)** ตารางนี้คือเครื่องมือที่ Candidate จริง
ใช้วางแผนการอ่านทวนก่อนสอบ — ให้ Print หรือ Copy ตารางนี้ไปทำ Checklist ของตัวเองได้ทันที

### ตาราง Gap Analysis เต็มรูปแบบ — Domain 1.0 Architecture (20%)

| Blueprint Sub-topic | ครอบคลุมจาก Part | สถานะ |
|---|---|---|
| 1.1 วิเคราะห์ Requirement/Constraint ทางธุรกิจ | Part 33-39 (ENCOR Design พื้นฐาน) | ✅ Part 1-70 |
| 1.2 ออกแบบ Solution ให้ตรง Requirement | Part 33-39, ทวนเข้มข้นใน Step 704 ของ Part นี้ | ✅ (พื้นฐาน) / ⏳ (เจาะลึกต่อ Part 96) |
| 1.3 ออกแบบ Layer 3 Infrastructure | Part 11-13, 26-32 (Routing พื้นฐาน-Advanced) | ✅ Part 1-70 / ⏳ Part 73 (Large-scale) |
| 1.4 วิเคราะห์ Fault Domain | Part 33-35 (Campus HA), Part 91 | ✅ (พื้นฐาน) / ⏳ Part 91-92 |
| 1.5 ออกแบบ Management Plane | Part 17, 22 (NTP/Syslog/SNMP/AAA) | ✅ Part 1-70 |
| 1.6 Traffic Engineering / Steering | Part 32 (PBR), Part 64 (PBR Troubleshoot) | ✅ Part 1-70 / ⏳ Part 74 (BGP TE) |
| 1.7 High Availability Design | Part 18, 34-35 (FHRP, StackWise/VSS) | ✅ (พื้นฐาน) / ⏳ Part 81 (NSF/SSO/ISSU) |
| 1.8 เปรียบเทียบ Routing Protocol | Part 11-13, 29-31 | ✅ Part 1-70 / ⏳ Part 73 |
| SD-WAN/SD-Access Design | Part 38-39 (พื้นฐาน) | ⏳ Part 76-77 (Advanced) |
| VXLAN/EVPN, MPLS L3VPN Design | ยังไม่ครอบคลุม | ⏳ Part 84-87 |
| Cloud/Disaster Recovery Design | ยังไม่ครอบคลุม | ⏳ Part 92-93 |

### ตาราง Gap Analysis — Domain 2.0 Infrastructure (30%)

| Blueprint Sub-topic | ครอบคลุมจาก Part | สถานะ |
|---|---|---|
| 2.1 Layer 2 (VLAN/Trunk/STP/EtherChannel/StackWise) | Part 3-6, 33-34 | ✅ Part 1-70 |
| 2.2 Layer 3 (OSPF/EIGRP/BGP/Redistribution/VRF/PBR) | Part 11-13, 26-32, 51, 56-59, 64 | ✅ Part 1-70 อย่างสมบูรณ์ |
| 2.3 Wireless (CAPWAP/Roaming/RRM/WLC HA) | Part 20, 36-37, 66 | ✅ (พื้นฐาน-กลาง) / ⏳ Part 89 (Scale) |
| 2.4 Network Services (FHRP/DHCP/NTP/QoS/Multicast) | Part 16-18, 24, 40-42, 62-63, 65, 68 | ✅ Part 1-70 อย่างสมบูรณ์ / ⏳ Part 82-83 (Advanced) |
| 2.5 Infrastructure Automation | Part 23, 47-50, 67 | ✅ (พื้นฐาน-กลาง) / ⏳ Part 79 (Full-Stack) |
| Large-scale Campus Fabric (Multi-site) | ยังไม่ครอบคลุมระดับ Scale | ⏳ Part 72 |
| Multi-Region BGP WAN Edge | Part 29-30 (พื้นฐาน) | ⏳ Part 74 |
| IPv6 Enterprise เต็มรูปแบบ | Part 8, 27 (พื้นฐาน) | ⏳ Part 88 |
| Catalyst 9000 Advanced Feature-set | Part 33-35 (บางส่วน) | ⏳ Part 90 |

### ตาราง Gap Analysis — Domain 3.0 Network Assurance (10%)

| Blueprint Sub-topic | ครอบคลุมจาก Part | สถานะ |
|---|---|---|
| 3.1 วินิจฉัยด้วย Traditional Monitoring | Part 17, 54, 62 | ✅ Part 1-70 |
| 3.1 วินิจฉัยด้วย Model-driven Telemetry | Part 54 (แนะนำเบื้องต้น) | ⏳ Part 80 (เจาะลึก) |
| 3.2 Config Data Collection (YANG Push/gRPC) | Part 49 (พื้นฐาน NETCONF/YANG) | ⏳ Part 80 |
| 3.3 Device Management ผ่าน NETCONF/RESTCONF | Part 49, 67 | ✅ (พื้นฐาน) / ⏳ Part 80 |
| 3.4 Visibility (NetFlow) | Part 54 | ✅ Part 1-70 |
| วิธีวินิจฉัยข้าม Domain แบบ CCIE (Methodology) | Part 56-69 (ระดับ Feature เดี่ยว) | ⏳ Part 91 (ข้าม Domain เต็มรูปแบบ) |

### ตาราง Gap Analysis — Domain 4.0 Security (15%)

| Blueprint Sub-topic | ครอบคลุมจาก Part | สถานะ |
|---|---|---|
| 4.1 ออกแบบ Network Security | Part 43 (Security Architecture พื้นฐาน) | ✅ (พื้นฐาน) / ⏳ Part 94 (Zero Trust) |
| 4.2 Security บน Router/Switch (ACL/CoPP/Hardening) | Part 14, 22, 45-46, 61 | ✅ Part 1-70 อย่างสมบูรณ์ |
| 4.3 Infrastructure Security Feature (Port Security/DHCP Snooping/802.1X) | Part 21, 44 | ✅ Part 1-70 |
| 4.4 AAA (TACACS+/RADIUS) | Part 22, 61 | ✅ Part 1-70 |
| 4.5 Zero Trust Workflow / Micro-segmentation | Part 44 (TrustSec พื้นฐาน) | ⏳ Part 78 (ISE Integration), Part 94 (Zero Trust เต็มรูปแบบ) |
| Firepower/FTD, Umbrella Integration | Part 43 (แนะนำ) | ⏳ Part 78 |

### ตาราง Gap Analysis — Domain 5.0 Automation (25%)

| Blueprint Sub-topic | ครอบคลุมจาก Part | สถานะ |
|---|---|---|
| 5.1 Python Script | Part 47 | ✅ Part 1-70 |
| 5.2 YANG Data Model | Part 49 | ✅ Part 1-70 |
| 5.3 XML/JSON/YAML | Part 23, 49 | ✅ Part 1-70 |
| 5.4 NETCONF/RESTCONF/API | Part 23, 49 | ✅ Part 1-70 |
| 5.5 EEM Applet | ยังไม่ครอบคลุมเจาะลึก | ⏳ Part 79 |
| 5.6 Git พื้นฐาน | ยังไม่ครอบคลุมเจาะลึก | ⏳ Part 79 (CI/CD Pipeline) |
| 5.7 Ansible/Puppet/Chef/SaltStack | Part 48 (Ansible เจาะลึก), เปรียบเทียบ Tool อื่น | ✅ (Ansible) / ⏳ Part 79 (เปรียบเทียบ+รวมระบบ) |
| 5.8 Configuration Management Tool ภาพรวม | Part 50 (DNA Center) | ✅ (พื้นฐาน) / ⏳ Part 79 |

### สรุปภาพรวม Gap Analysis — ตารางเดียวจบ

| Domain | น้ำหนัก | % ที่ Part 1-70 ครอบคลุมแล้ว (ประมาณ) | Part ที่จะเติมส่วนที่เหลือ |
|---|---|---|---|
| 1.0 Architecture | 20% | ~50% (พื้นฐาน Design ครบ, Design ระดับ Scale/VXLAN/MPLS ยังไม่มี) | 72,73,76,77,84-87,92-93 |
| 2.0 Infrastructure | 30% | ~70% (Feature หลักครบเกือบทั้งหมด, ขาด Scale/VXLAN/MPLS/IPv6 เจาะลึก) | 72-75,81-90 |
| 3.0 Network Assurance | 10% | ~40% (Traditional Monitoring ครบ, Model-driven Telemetry ยังบางส่วน) | 80,91 |
| 4.0 Security | 15% | ~75% (Infra Security ครบมาก, ขาด ISE Integration/Zero Trust เต็มรูปแบบ) | 78,94 |
| 5.0 Automation | 25% | ~55% (พื้นฐาน API/Python/Ansible ครบ, ขาด EEM/Git/CI-CD รวมระบบ) | 79,80 |

> **วิธีใช้ตารางนี้จริง**: ก่อนเข้าสู่ Part 72 ให้ทำ Checklist ส่วนตัวจากตารางข้างบน — Tick ทุก
> Sub-topic ที่ "มั่นใจ 100%" แล้ว (ควรเป็นแถวที่มาจาก Part 1-70) ถ้ามีแถวไหนที่ Tick ไม่ได้ ให้
> ย้อนกลับไปทวน Part ที่อ้างอิงก่อนเดินหน้า — ตารางนี้จะแม่นยำขึ้นเรื่อยๆทุกครั้งที่ Part ใหม่ใน
> ช่วง 72-100 ถูกเขียนเสร็จ (Column "สถานะ" ของแต่ละแถวจะเปลี่ยนจาก ⏳ เป็น ✅ ตามลำดับ)

---

## แบบฝึกหัดทวนความเข้าใจ Part 71

1. CCIE EI Lab Exam ใช้เวลาสอบทั้งหมดกี่ชั่วโมง และแบ่งเป็น Module ทางการกี่ Module อะไรบ้าง?
2. Domain ใดใน CCIE EI v1.1 Blueprint ที่มีน้ำหนักคะแนนสูงที่สุด และ Domain ใดมีน้ำหนักสูงเป็น
   อันดับ 2?
3. อธิบายความต่างของคำถามสไตล์ CCNP กับสไตล์ CCIE โดยใช้หลักการ "how to configure X" เทียบกับ
   อะไร?
4. เพราะเหตุใดการฝึกด้วย End-to-End Lab ต่อเนื่อง (แบบที่หลักสูตรนี้ทำมาตั้งแต่ Part 1) จึงเหมาะกับ
   การเตรียมสอบ CCIE มากกว่า Isolated Feature Practice?
5. ยกตัวอย่างกับดักคำศัพท์ในโจทย์ CCIE ที่ทำให้ Candidate ตีความ Requirement ผิดพลาดได้ พร้อม
   อธิบายว่าทำไมถึงพลาดง่าย

**เฉลย:**

1. **8 ชั่วโมงต่อเนื่อง** แบ่งเป็น 2 Module ทางการ: **Design Module (~3 ชั่วโมง)** และ
   **Deploy, Operate Module (~5 ชั่วโมง)** — ส่วน "Diagnose" เป็นรูปแบบงาน (Task Type) ที่ฝังอยู่
   ใน Deploy, Operate Module ไม่ใช่ Module เวลาแยก
2. **Infrastructure (30%)** สูงที่สุด และ **Automation (25%)** สูงเป็นอันดับ 2 — รวมกัน 55%
   ของคะแนนทั้งฉบับ
3. คำถามสไตล์ CCNP ถามว่า **"จะ Config X อย่างไร"** (มีคำตอบเดียวชัดเจน วัด Syntax) ส่วนคำถาม
   สไตล์ CCIE ถามว่า **"ให้ Business Requirement + ข้อจำกัดมา จะออกแบบและ Implement Solution ที่
   ดีที่สุดอย่างไร แล้วพร้อมอธิบาย/ป้องกันเหตุผลของ Design ที่เลือก"** (มักมีหลายทางที่ใช้งานได้
   แต่มีทางที่ดีที่สุดตาม Requirement ไม่กี่ทาง วัดการวิเคราะห์และการตัดสินใจ)
4. เพราะ CCIE Lab ไม่เคยถาม Feature แบบแยกโดด ทุก Task เชื่อมกับ Task อื่นเสมอ (Interaction ข้าม
   Feature), ต้องมี Topology ใหญ่พอที่จะจำลองปัญหาข้าม Layer/Domain ได้จริง, และต้องมี Scale พอที่
   จะฝึกความกดดันเรื่องเวลาได้จริง — Isolated Practice บน Topology เล็กที่ลบทิ้งทุกครั้งไม่สามารถ
   จำลอง 3 อย่างนี้ได้เลย
5. ตัวอย่าง: คำว่า "**ต้อง**ใช้ OSPF Area 0 เท่านั้น" ถ้าอ่านไม่ละเอียดอาจไปสร้าง Multi-area Design
   เพราะคิดว่า "ดีกว่าในมุมมอง Design" ทั้งที่โจทย์ห้ามชัดเจน — พลาดง่ายเพราะ Candidate มักอ่านโจทย์
   เร็วและตีความตาม Best Practice ที่คุ้นเคย มากกว่าตาม Requirement ที่ระบุตรงหน้าจริงๆ (ดูตาราง
   กับดักคำศัพท์เต็มรูปแบบใน Step 708)

---

## สรุป Part 71

Part นี้ปูภาพรวมทั้งหมดของ **CCIE Enterprise Infrastructure** ก่อนเข้าสู่เนื้อหาเทคนิคเจาะลึก:
รูปแบบข้อสอบ Lab 8 ชั่วโมงที่ไม่มี Multiple-choice, Blueprint v1.1 ทั้ง 5 Domain
(Architecture 20% / Infrastructure 30% / Network Assurance 10% / Security 15% / Automation 25%),
4 รูปแบบงานในห้องสอบ (Design/Deploy/Optimize-Operate/Diagnose), ความต่างพื้นฐานของ "วิธีคิด"
ระดับ CCIE เทียบกับ CCNP ผ่านตัวอย่าง OSPF Area Design, สภาพแวดล้อม Lab จริงที่มีอุปกรณ์หลายสิบตัว,
กลยุทธ์การฝึกแบบ End-to-End Lab ที่หลักสูตรนี้ใช้มาตั้งแต่ Part 1, บทบาทของ Automation ที่ยกระดับ
เป็น 1 ใน 4 เสาหลักของคะแนน, จุดพลาดคลาสสิกของผู้สอบจริง, แผนเนื้อหา Part 72-100 ทั้งหมด และปิดท้าย
ด้วย Gap Analysis ตารางเต็มที่ Mapping ทุกหัวข้อ Blueprint เข้ากับ Part ที่ทำสำเร็จแล้วหรือกำลังจะ
ทำ — ตารางนี้คือเครื่องมือ Self-Tracking ที่ควรกลับมาเปิดดูซ้ำทุกครั้งที่ Part ใหม่ในช่วง 72-100
เขียนเสร็จ

**พร้อมสำหรับ Part 72**: เราจะเริ่มเจาะลึกเทคโนโลยีตัวแรกของช่วง CCIE — **Large Campus Fabric
Design (Layer 2)** ขยาย Enterprise Lab เดิมของเราให้รองรับ Campus Design ระดับ Multi-site/
Multi-building ตาม Architecture Domain (20%) ของ Blueprint ที่วางไว้ใน Step 702

**ไปต่อ:** [Part 72 — Large Campus Fabric Design (L2) →](part-072-large-campus-fabric-design.md)
