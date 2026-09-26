# Part 92 — Disaster Recovery & Network Resilience
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 911–920 จาก 1000**

> ต่อจาก [Part 91 — Advanced Troubleshooting Methodology](part-091-advanced-troubleshooting-methodology.md)
> ที่สอนวิธี**คิดอย่างเป็นระบบ**เมื่อ Network มีปัญหาระดับ Incident เดียว (Sev-1/Sev-2 ตาม War-Room
> Process ของ [Part 69](part-069-mega-lab-1.md)) — Part นี้ยกระดับคำถามขึ้นไปอีกชั้น: **ถ้าปัญหา
> ไม่ใช่ "อุปกรณ์ตัวหนึ่งเสีย" แต่เป็น "Data Center ทั้งแห่งใช้งานไม่ได้" (ไฟไหม้, น้ำท่วม, ไฟดับ
> ทั้งอาคาร, หรือ Ransomware ที่ต้องตัดขาดทั้ง Site) องค์กรจะทำอะไรได้บ้าง?** เราเรียน **High
> Availability ระดับ Chassis/Device** มาแล้วอย่างละเอียดใน [Part 81](part-081-high-availability-nsf-sso-issu.md)
> (SSO/NSF/ISSU — ทำให้ "อุปกรณ์ตัวเดียวเสีย ไม่กระทบ"), **Redundancy ระดับ Link/FHRP** มาแล้วใน
> [Part 18/35](part-035-advanced-fhrp-campus-ha.md), และ [Part 85 Step 849](part-085-vxlan-evpn-multisite-advanced.md#step-849--multisite-และ-disaster-recovery-design)
> ได้ Preview ไว้ชัดเจนว่า **EVPN Multisite ที่สร้าง DC1/DC2 ขึ้นมานั้นเป็นแค่ "Network-layer
> enabler" ของ DR ไม่ใช่ DR Solution เต็มรูปแบบในตัวเอง** — Part นี้คือ Part ที่มาเติมส่วนที่เหลือ
> ให้ครบ: **RTO/RPO สำหรับ Network โดยเฉพาะ, DR Site Model (Hot/Warm/Cold), Config DR (ต่อยอด
> Part 17/47/79), WAN/DCI Circuit Diversity (ต่อยอด Part 72 Step 716), DNS Failover (ต่อยอด
> Part 16), Runbook-driven DR, DR Testing Discipline, War-Room ระดับ Business Continuity (ต่อยอด
> Part 69), และเส้นแบ่งบางๆระหว่าง DR กับ Security Incident Response (ต่อยอด Part 78 Rapid Threat
> Containment)** ปิดท้ายด้วย Lab ที่เป็น **DR Runbook Document ฉบับสมบูรณ์** สำหรับสถานการณ์ "DC1
> ล่มทั้ง Site" บน DC1/DC2 Multisite Fabric ที่ Part 84-85 สร้างไว้

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 911 | DR คืออะไรสำหรับ Network — RTO/RPO ที่ใช้กับ Network Config/State ไม่ใช่แค่ Data |
| 912 | DR Site Model — Hot / Warm / Cold Site: ตารางเปรียบเทียบ Cost vs RTO |
| 913 | Network Configuration DR — ทำให้ "อุปกรณ์ที่ถูกทำลาย" สร้างใหม่ได้เร็ว ไม่ใช่แค่ "Network ไม่ล่ม" |
| 914 | WAN/DCI Circuit Diversity — Single Point of Failure ที่ซ่อนอยู่ใน Physical Path และบทเรียนจากของจริง |
| 915 | DNS และ Resilience — TTL Tuning, Health-check-based Failover ที่มักถูกมองข้าม |
| 916 | Runbook-driven DR — โครงสร้าง Runbook มาตรฐานสำหรับสถานการณ์ "Primary DC ล่มทั้ง Site" |
| 917 | DR Testing — Tabletop Exercise vs Live Failover Test และความขัดแย้งกับความเสี่ยงต่อ Production |
| 918 | การสื่อสารและสายบังคับบัญชาระหว่างเกิด DR Event — War-Room ระดับ Business Continuity |
| 919 | Cyber-incident-driven "Disaster" — เมื่อ Ransomware ต้องใช้ Network Isolation แบบ DR |
| 920 | Lab เต็มรูปแบบ: DR Runbook Document สำหรับสถานการณ์ "DC1 ล่มทั้ง Site" บน DC1/DC2 Multisite |

---

## Step 911 — DR คืออะไรสำหรับ Network: RTO/RPO ที่ใช้กับ Network Config/State

### 911.1 ทำไม Network DR ถูกมองข้ามเสมอเมื่อเทียบกับ Server/Data DR

องค์กรส่วนใหญ่ลงทุนกับ **Disaster Recovery (DR)** ในมุม Server/Application/Database อย่างจริงจัง —
มี Backup Database ทุกคืน, มี Database Replication ข้าม Site, มี Runbook กู้คืน Application ที่
ทดสอบทุกไตรมาส แต่พอถามว่า **"ถ้า Core Switch/Router ที่ทำให้ Application เหล่านั้นคุยกับใครได้
เลยพังไปทั้ง Site ล่ะ?"** คำตอบที่ได้บ่อยที่สุดคือความเงียบ หรือ "ก็ Network มัน Redundant อยู่แล้ว"
ซึ่งเป็นคำตอบที่สับสนระหว่าง **Redundancy ระดับอุปกรณ์/ลิงก์ (Part 81/35)** กับ **DR ระดับ Site**
สองเรื่องนี้แก้ปัญหาคนละชั้นกันอย่างสิ้นเชิง:

```
Redundancy (Part 18/35/81)                    DR (Part นี้)
─────────────────────────                     ─────────────
แก้: "อุปกรณ์/ลิงก์ตัวหนึ่งเสีย                  แก้: "ทั้ง Site ใช้งานไม่ได้เลย"
     ในระบบเดียวกันยังทำงานต่อได้"                  (ไฟไหม้, น้ำท่วม, ไฟดับทั้งอาคาร,
Scope: ภายใน Site เดียว                            แผ่นดินไหว, Ransomware ทั้ง Fabric)
เวลา Recover: Sub-second ถึงไม่กี่วินาที          Scope: ข้าม Site (ต้องมี Site ที่ 2)
(HSRP/SSO/NSF/ECMP ทำงานอัตโนมัติ)                เวลา Recover: นาทีถึงชั่วโมง (ต้องมี
                                                    การตัดสินใจของมนุษย์ + Failover Procedure)
```

เหตุผลที่ Network DR ถูกมองข้ามบ่อยกว่า Server/Data DR มีอยู่ 3 ข้อหลัก:

| เหตุผล | รายละเอียด |
|---|---|
| **"มองไม่เห็น" เท่า Data** | ผู้บริหารเข้าใจง่ายว่า "ถ้า Database หายไป = ข้อมูลลูกค้าหายไปด้วย" แต่ไม่ค่อยเข้าใจว่า "ถ้า Core Switch พังทั้ง Site = ต่อให้ Database ยัง Sync อยู่ที่ DR Site แต่ไม่มีใครไปถึงมันได้เลย" |
| **มักถูกฝังไว้ใน Redundancy ที่มีอยู่แล้ว** | ทีม Network มักคิดว่า HA ระดับ Chassis/Link (Part 81) "พอแล้ว" โดยไม่แยกให้ชัดว่านั่นคือคนละ Failure Domain กับการเสีย Site ทั้งหมด |
| **Network เป็น "ท่อ" ไม่ใช่ "จุดหมายปลายทาง"** | Application/Data Team คิดถึง Application ของตัวเองเป็นหลักในการทำ DR Plan และมักสมมติเอาเองว่า "Network จะพร้อมให้ Failover ไปเสมอ" โดยไม่เคย Verify กับทีม Network จริง |

### 911.2 RTO และ RPO — นิยามที่ใช้กับ Data DR แบบเดิม

**RTO (Recovery Time Objective)** = เวลาสูงสุดที่ยอมรับได้ตั้งแต่เกิด Disaster จนกว่าระบบจะ
กลับมาใช้งานได้ **RPO (Recovery Point Objective)** = จุดข้อมูลล่าสุดที่ยอมรับได้ว่า "หายไปได้"
(ยิ่ง RPO สั้น = ยอมให้ข้อมูลหายน้อยลง = ต้อง Backup/Replicate ถี่ขึ้น)

```
Timeline ของ Disaster (มุม Data)

──────●──────────────────●─────────────────●───────────────►  เวลา
   Backup ล่าสุด      Disaster เกิดขึ้น         ระบบกลับมาใช้งานได้
      │                    │                        │
      │◄──── RPO ─────────►│                        │
      │  (ข้อมูลที่หายไป =                            │
      │   Backup ล่าสุด ถึง                           │
      │   วินาทีที่ Disaster เกิด)                     │
                           │◄──────── RTO ───────────►│
                           │  (เวลาที่ระบบ Down ทั้งหมด)
```

### 911.3 RTO/RPO เมื่อนำมาใช้กับ Network Config/State — คนละมุมกับ Data โดยสิ้นเชิง

นี่คือประเด็นสำคัญที่สุดของ Step นี้: เมื่อพูดถึง **Network DR**, RTO/RPO ไม่ได้วัด "ข้อมูลลูกค้า
หายไปแค่ไหน" แต่วัด **"Network State/Config หายไปแค่ไหน และต้องใช้เวลานานแค่ไหนกว่า Network จะ
กลับมา Forward Traffic ได้ปกติ"**

| แนวคิด | มุม Data/Server DR (คุ้นเคย) | มุม Network DR (Part นี้) |
|---|---|---|
| **RPO วัดอะไร** | ข้อมูล Transaction/Record ที่ยอมให้หายได้ (เช่น "ยอมเสีย Transaction 15 นาทีสุดท้าย") | **Config Drift ที่ยอมรับได้**: ถ้า Backup Config ล่าสุดเก่ากว่าการเปลี่ยนแปลงจริงบนอุปกรณ์ (เช่น มี ACL ใหม่ที่เพิ่งเพิ่มเมื่อเช้าแต่ยังไม่ Commit เข้า Backup) Rebuild จาก Backup จะได้ Network ที่ "เก่ากว่าความจริง" — ROP สั้น = ต้อง Backup/Commit ถี่ (ผูกกับ Part 47/79) |
| **RTO วัดอะไร** | เวลาที่ Application/Database กลับมา Query ได้ | **เวลาที่ Traffic เริ่ม Forward ผ่าน Path ใหม่ได้จริง** — รวมทั้ง (1) เวลา Detect ว่า Site ล่ม (2) เวลาตัดสินใจ Declare Disaster (3) เวลา Reconfigure/Redirect Traffic จริง (4) เวลา Propagate การเปลี่ยนแปลงนั้นให้ Client ทุกตัวรู้ (DNS TTL ตาม Step 915) |
| **สิ่งที่ "หายไป" ถ้า RPO ไม่ดี** | Transaction ล่าสุด | **Running-config ที่ไม่มีใน Backup** — ถ้าอุปกรณ์ถูกทำลายจริง (ไฟไหม้ Rack) และ Backup ล่าสุดเก่ากว่าการเปลี่ยน VLAN/ACL ครั้งล่าสุด ต้องมานั่งจำ/สร้างส่วนต่างนั้นเองด้วยมือ ซึ่งเป็นความเสี่ยง Human Error สูงมากตอนกำลังเร่งกู้ระบบ |

> **ตัวอย่างที่ทำให้เห็นภาพชัด**: DC1 ที่ Part 84-85 สร้างไว้มี `DC-LEAF-1/2`, `DC-BGW-1/2`
> ที่มี Config เฉพาะทาง (VNI Mapping, Route-Target, EVPN Policy) จำนวนมาก — ถ้า `DC-LEAF-1` ถูก
> ทำลายจริง (Hardware Failure ร้ายแรง/ไฟไหม้ Rack) และแผนก Network ไม่มี Backup Config ที่ทันสมัย
> ของอุปกรณ์ตัวนี้เลย **RTO ของการกู้ Site นี้จะยาวขึ้นทันทีเป็นชั่วโมงถึงวัน** เพราะต้อง Reverse-
> Engineer Config จากอุปกรณ์ข้างเคียง/ความจำของวิศวกร แทนที่จะ Restore จาก Backup ได้ในไม่กี่นาที
> (รายละเอียดเต็มอยู่ใน Step 913)

### 911.4 ตาราง RTO/RPO ตามระดับความสำคัญของระบบ (ตัวอย่างที่องค์กรใช้จริง)

| Tier | ตัวอย่างระบบ | RTO เป้าหมาย | RPO เป้าหมาย | Network Design ที่ต้องมี |
|---|---|---|---|---|
| Tier 0 (Mission-critical) | Core Banking, EVPN Multisite Fabric เอง | < 5 นาที | ~0 (Real-time) | Hot Site, Active-Active, Automated Failover |
| Tier 1 (Business-critical) | ERP, DC-SERVERS Workload | < 1 ชั่วโมง | < 15 นาที | Hot/Warm Site, Runbook + Manual Trigger |
| Tier 2 (Important) | Internal Tools, File Server | < 8 ชั่วโมง (Business hours) | < 4 ชั่วโมง | Warm/Cold Site |
| Tier 3 (Non-critical) | Dev/Test Environment | < 3 วัน | < 24 ชั่วโมง | Cold Site หรือ Rebuild จาก Config Backup |

> **ข้อคิดสำคัญ**: RTO/RPO ของ Network ไม่ควร**ยาวกว่า**ระบบ Tier ที่สูงสุดที่ Network นั้นรองรับ
> เด็ดขาด — การตั้ง Application RTO = 5 นาที แต่ Network RTO = 2 ชั่วโมง เป็นการวางแผนที่ผิดตั้งแต่
> ต้น เพราะ Network คือ **Dependency ร่วม (Shared Dependency)** ของทุก Application ที่วิ่งอยู่บน
> Site เดียวกัน — Network RTO ต้อง**สั้นกว่าหรือเท่ากับ**ค่าที่ Application ที่ Critical ที่สุด
> ต้องการเสมอ

---

## Step 912 — DR Site Model: Hot / Warm / Cold Site

### 912.1 สามระดับของ DR Site และหลักการแบ่ง

DR Site (Site สำรองที่รับ Workload เมื่อ Primary Site ล่ม) แบ่งออกเป็น 3 ระดับตามว่า **"พร้อมรับ
งานทันทีแค่ไหน"** — ยิ่งพร้อมมากเท่าไร ค่าใช้จ่ายในการดูแลให้พร้อมตลอดเวลาก็สูงขึ้นตามไปด้วย
นี่คือ Trade-off หลักของการออกแบบ DR ทุกองค์กร:

```
                    Cost & Complexity เพิ่มขึ้น ────────────────►
Cold Site  ──────────────────►  Warm Site  ──────────────────►  Hot Site
RTO: วัน-สัปดาห์                  RTO: ชั่วโมง                     RTO: นาที (หรือ Real-time)
```

### 912.2 Hot Site — Fully Active, Real-time Sync

**Hot Site** คือ DR Site ที่ **มี Infrastructure พร้อมทำงานตลอดเวลา และ Sync State กับ Primary
Site แบบ Real-time (หรือใกล้เคียง)** — เมื่อ Primary Site ล่ม แทบไม่ต้องทำอะไรเพิ่มเลยนอกจาก
Redirect Traffic เข้า DR Site (บางองค์กรทำ **Active-Active** จริง คือทั้งสอง Site รับ Traffic
พร้อมกันตลอดเวลาอยู่แล้ว ไม่ต้อง "Failover" เลยด้วยซ้ำ)

**เชื่อมกับ Part 85**: DC1/DC2 EVPN Multisite Fabric ที่ Part 84-85 สร้างไว้คือ **Network-layer
Enabler ของ Hot Site Pattern โดยตรง** — VNI 30130 (DC-SERVERS) ถูก **Stretch** ข้าม DC1/DC2 ผ่าน
BGW ทำให้ Subnet เดียวกัน (`10.130.30.0/24`) มีอยู่พร้อมกันทั้งสอง Site และ Host/VM ย้ายข้าม Site
ได้แบบ Live Migration โดย IP ไม่เปลี่ยน — นี่คือคุณสมบัติที่ทำให้ Hot Site แบบ Active-Active เป็นไป
ได้จริงในทางเทคนิค (ตามที่ Part 85 Step 849 บอกไว้ว่า "L2 Stretch เหมาะกับ Active-Active/DR
แบบ Near-zero RTO")

### 912.3 Warm Site — Standby, ต้อง Activation

**Warm Site** คือ DR Site ที่ **มี Infrastructure พร้อมอยู่แล้ว (Hardware ติดตั้งเรียบร้อย, มี
Config Baseline) แต่ไม่ได้รับ Traffic จริงตลอดเวลา** — ต้องมีขั้นตอน **Activation** (เปิด Config
บางส่วน, Sync ข้อมูลรอบล่าสุด, Redirect DNS/Routing) ก่อนจะรับ Production Traffic ได้จริง ตาม
Pattern **Active-Standby** ที่ Part 85 Step 849 อธิบายไว้ (ใช้ **L3-only** DCI ส่วนใหญ่ — DC2 มี
Subnet ของตัวเอง เช่น `10.141.30.0/24` (VNI 30141, DC-APP-DC2) ที่ไม่ Stretch กับ DC1 เพราะการ
Failover ทำที่ Application/DNS Layer แทน)

### 912.4 Cold Site — Bare Infrastructure

**Cold Site** คือ Site ที่มีแค่ **โครงสร้างพื้นฐานทางกายภาพ** (พื้นที่, ไฟ, แอร์, บางครั้งมี
Rack/Cabling พร้อม) แต่ **ไม่มี Hardware ที่ Config ไว้รอเลย** — ต้องสั่งซื้อ/ติดตั้ง/Config
อุปกรณ์ใหม่ทั้งหมดตอนเกิด Disaster จริง (หรือมี Hardware สำรองอยู่ในคลังแต่ยังไม่ได้ Rack/Config)

### 912.5 ตารางเปรียบเทียบเต็มรูปแบบ

| ประเด็น | Hot Site | Warm Site | Cold Site |
|---|---|---|---|
| Infrastructure | ติดตั้งพร้อม Running | ติดตั้งพร้อม แต่ Standby | โครงสร้างพื้นฐานอย่างเดียว (พื้น/ไฟ/แอร์) |
| Data/State Sync | Real-time (Continuous) | Periodic (นาทีถึงชั่วโมง) | ไม่มี (ต้อง Restore จาก Backup ตอนเกิดเหตุ) |
| Network Design | EVPN Multisite L2 Stretch, Active-Active | L3-only DCI, Route Advertisement พร้อมแต่ปิดอยู่ | ไม่มี Network Path ที่ Config ไว้ล่วงหน้าเลย |
| RTO | นาที (หรือ ~0 ถ้า Active-Active) | ชั่วโมง (2-24 ชม. ทั่วไป) | วัน-สัปดาห์ |
| RPO | ~0 ถึงไม่กี่นาที | นาทีถึงชั่วโมง (ขึ้นกับรอบ Sync) | ขึ้นกับรอบ Backup ล่าสุด (อาจเป็นวัน) |
| Cost (Capex+Opex ต่อเนื่อง) | สูงมาก (จ่ายค่า Infra 2 เท่าตลอดเวลา) | กลาง (จ่าย Infra แต่ประหยัด Operating Cost บางส่วน) | ต่ำ (จ่ายแค่พื้นที่/สัญญาเช่า) |
| เหมาะกับ Tier | Tier 0 | Tier 1-2 | Tier 2-3 |

> **กับดักที่พบบ่อย**: หลายองค์กร "พูดว่า" มี Hot Site แต่จริงๆทำแค่ Warm (เช่น มี DR Site ที่
> Config Network ไว้พร้อมแล้ว แต่ Application ยังต้อง Deploy ใหม่ตอนเกิดเหตุ) — ต้องระบุ RTO/RPO
> เป็นตัวเลขจริงเสมอ ไม่ใช่ใช้ชื่อ "Hot/Warm/Cold" อย่างหลวมๆ เพราะผู้บริหารมักเข้าใจผิดว่า "มี
> Hot Site แล้ว = ปลอดภัย 100%" ทั้งที่ Network Layer อาจเป็น Hot จริง แต่ Application Layer
> ยังเป็น Cold อยู่

---

## Step 913 — Network Configuration DR: อุปกรณ์ที่ถูกทำลายต้อง "สร้างใหม่ได้เร็ว"

### 913.1 มุมที่มักถูกมองข้าม: "Network ไม่ล่ม" ≠ "Network สร้างใหม่ได้"

Step 911-912 พูดถึง DR ในมุม "ทำให้ Traffic ยังวิ่งต่อได้ที่ DR Site" — แต่มีอีกมุมที่แยกกันชัดเจน
และมักถูกมองข้าม: **ถ้าอุปกรณ์ Network ตัวหนึ่ง (หรือทั้ง Site) ถูกทำลายจริง (ไม่ใช่แค่ Down
ชั่วคราว) จะสร้างมันขึ้นมาใหม่ได้เร็วแค่ไหน?** นี่คือคำถามที่ต่อยอดจาก Step 911.3 (RPO ของ
Config) โดยตรง

```
สองคำถามที่แยกกัน แต่คนมักปนกัน:

1. "Site ล่ม → Traffic ย้ายไป DR Site ได้ไหม?"      ← Step 912 (Hot/Warm/Cold Site)
2. "อุปกรณ์ถูกทำลาย → สร้าง Config มันใหม่ได้เร็วไหม?"  ← Step นี้ (Config DR)

องค์กรจำนวนมากตอบข้อ 1 ได้ดี (มี DR Site) แต่ตอบข้อ 2 ไม่ได้เลย
(ไม่มี Backup Config ที่ทันสมัยของอุปกรณ์ตัวที่ถูกทำลาย)
```

### 913.2 ต่อยอดจาก Part 17/47: Automated Backup ต้องครบ ต้องถี่ ต้อง Verify

[Part 17](part-017-ntp-syslog-snmp.md) แนะนำ `copy running-config startup-config` และ
[Part 47 Step 468](part-047-automation-python-advanced.md#step-468--git-integration-สำหรับ-config-version-control)
สร้าง **Automated Backup Script** ที่ดึง Running-config ทุกอุปกรณ์เข้า Git Repository พร้อม
Timestamp — สำหรับ Network DR โดยเฉพาะ ต้องเพิ่มเงื่อนไข 3 ข้อที่ทำให้ Backup นี้ **ใช้กู้คืน Site
ทั้งหมดได้จริง** ไม่ใช่แค่ "มีไฟล์ Backup อยู่":

| เงื่อนไข | เหตุผล |
|---|---|
| **ความถี่ (Frequency)** ต้องสอดคล้องกับ RPO ที่ตั้งไว้ (Step 911.3) | ถ้า RPO ของ Network Config คือ "ยอมเสีย Config ที่เปลี่ยนไม่เกิน 1 ชั่วโมง" ต้อง Backup ทุก Config Change (Event-driven ผ่าน Syslog/Webhook — ไม่ใช่แค่ Cron รายวัน) |
| **ครอบคลุมทุกอุปกรณ์ ไม่มีข้อยกเว้น** | อุปกรณ์ใหม่ (เช่น `DC-BGW-1/2` ที่เพิ่มเข้ามาใน Part 85) ต้องถูกเพิ่มเข้า Inventory ของ Backup Script ทันทีที่ Deploy — ไม่ใช่ "นึกออกทีหลัง" |
| **Restore ได้จริง ไม่ใช่แค่มี Backup** | ต้องทดสอบ Restore Config จาก Git ไปยังอุปกรณ์เปล่า (Factory Default) จริงเป็นระยะ — Backup ที่ไม่เคยถูกทดสอบ Restore คือความเสี่ยงที่ซ่อนอยู่ (คล้ายกับ Data Backup ที่ไม่เคยทดสอบ Restore) |

### 913.3 ต่อยอดจาก Part 79: Git ไม่ใช่แค่ Version Control — คือ "แผนผังการสร้างใหม่ทั้ง Fabric"

[Part 79](part-079-fullstack-automation-cicd.md) วาง **Git เป็น Single Source of Truth** สำหรับ
Config/Playbook/Template ทั้งหมดของ Network CI/CD Pipeline — สำหรับ DR โดยเฉพาะ คุณสมบัตินี้มี
ความหมายเพิ่มขึ้นอีกชั้น: **ถ้า Git Repository ยังอยู่ (แม้ Site ที่มีอุปกรณ์จริงถูกทำลายหมด)
องค์กรก็ยังมี "แผนผัง" สำหรับสร้าง Network ทั้ง Fabric ขึ้นมาใหม่จากศูนย์ได้** — นี่คือเหตุผลที่
**Git Repository ต้อง Host อยู่นอก Site ที่กำลังป้องกัน DR ให้เสมอ** (Cloud-hosted Git, หรือ
Mirror ไปยัง DR Site อีกชุด) — ถ้า Git Server เอง Host อยู่ที่ DC1 และ DC1 ถูกทำลายพร้อมกับ Git
Server ด้วย แผนผังทั้งหมดจะหายไปพร้อมกับอุปกรณ์จริง ซึ่งขัดกับเจตนาของ DR โดยสิ้นเชิง

```
Config DR Chain ที่ต้องครบทุกข้อ (ต่อยอด Part 17 → 47 → 79 → Part นี้)

Device Running-config
       │  (ทุก Config Change, Event-driven ตาม RPO)
       ▼
Automated Backup Script (Part 47 Step 468)
       │
       ▼
Git Repository ── ต้อง Host แยกจาก Site ที่ป้องกัน DR (Part 79 ต่อยอด)
       │             เช่น Cloud-hosted Git หรือ Mirror ไปยัง DR Site
       ▼
Zero-Touch Provisioning / Golden Config Template
       │  (สำหรับ Rebuild อุปกรณ์ใหม่ทดแทนตัวที่ถูกทำลาย)
       ▼
อุปกรณ์ใหม่ (Replacement Hardware) ── Config กลับมาเหมือนก่อน Disaster ภายในไม่กี่นาที
                                        (ไม่ใช่ Reverse-engineer จากความจำวิศวกร)
```

### 913.4 Worked Scenario: `DC-LEAF-1` ถูกทำลายจริง (ไฟไหม้ Rack)

```
สถานการณ์: Rack ที่ตั้ง DC-LEAF-1 ไฟไหม้เสียหายทั้ง Hardware — ต้องเปลี่ยนเป็น Chassis ใหม่

ไม่มี Config DR ที่ดี:                          มี Config DR ที่ดีตาม Step นี้:
1. เบิก Hardware ใหม่ (ชั่วโมง-วัน)               1. เบิก Hardware ใหม่ (เวลาเท่ากัน — เป็นข้อจำกัด
2. Config จากศูนย์ด้วยมือ                            ทาง Physical ที่ Config DR แก้ไม่ได้)
   - ต้องถาม/จำ VNI Mapping, Route-Target,       2. Boot อุปกรณ์ใหม่ด้วย Day-0 Bootstrap (ZTP/
     BGP AS, Interface ที่เคย Config              POAP) → ดึง Golden Config ล่าสุดจาก Git
   - เสี่ยงพิมพ์ผิด/ลืมบางส่วน (เช่น ACL ที่          ตรงกับ Hostname `DC-LEAF-1` โดยอัตโนมัติ
     เพิ่มไว้เมื่อสัปดาห์ก่อนแต่ไม่มีใครจำ)         3. Config ตรงกับก่อน Disaster 100% (ตาม RPO ล่าสุด
3. RTO: หลายชั่วโมงถึงวัน (ขึ้นกับว่าวิศวกร            ที่ Backup ไว้) — ต้องตรวจสอบเฉพาะส่วนที่เปลี่ยน
   คนที่จำ Config ได้อยู่ว่างไหม)                     หลัง Backup ล่าสุดเท่านั้น (ตาม RPO Window)
                                                 4. RTO: จำกัดอยู่ที่เวลาเบิก/ติดตั้ง Hardware
                                                    ไม่ใช่เวลา Config ใหม่ทั้งหมด
```

> **บทเรียน**: Config DR ไม่ได้ทำให้ Hardware มาถึงเร็วขึ้น (ยังต้องรอ Physical Lead Time
> เหมือนกัน) แต่ทำให้ **ส่วนที่เป็น Human/Software Process (Config)** ไม่ใช่ Bottleneck ของ RTO
> อีกต่อไป — ในองค์กรจริงหลายแห่ง เวลาที่เสียไปกับ "จำ/สร้าง Config ใหม่" มากกว่าเวลาที่เสียไปกับ
> "รอ Hardware มาถึง" เสียอีก เพราะ Hardware สำรอง (Cold Spare) มักมีอยู่ในคลังแล้วสำหรับ Platform
> สำคัญ แต่ Config ที่ถูกต้อง 100% มักไม่มีใครมั่นใจว่า Backup ล่าสุดตรงกับความจริงแค่ไหน

---

## Step 914 — WAN/DCI Circuit Diversity: Single Point of Failure ที่ซ่อนอยู่ใน Physical Path

### 914.1 ทวนหลักการจาก Part 72 Step 716 แล้วขยายไปที่ WAN/DCI

[Part 72 Step 716](part-072-large-campus-fabric-design.md#step-716--การเชื่อมต่อระหว่างอาคาร-multi-building-interconnection)
สอนกฎเหล็กเรื่อง **Conduit Diversity** สำหรับการเชื่อมต่อระหว่างอาคารภายใน Campus เดียวกัน: ต่อ
Fiber 2 เส้นเพื่อ Redundancy แต่เดินท่อร้อยสาย (Conduit) เดียวกัน = Redundancy ปลอมทันที (Backhoe/
ท่อแตกจุดเดียวตัดทั้งคู่พร้อมกัน) — หลักการเดียวกันนี้ **สำคัญกว่าเดิมอีกหลายเท่า** เมื่อขยับไปที่
ระดับ **WAN Circuit และ DCI (Data Center Interconnect)** ระหว่าง DC1 ↔ DC2 เพราะระยะทางไกลกว่า
มาก (ข้ามเมือง/ข้ามประเทศตาม [Part 85 Step 847](part-085-vxlan-evpn-multisite-advanced.md))
และผ่านมือ **ISP/Carrier หลายเจ้าที่องค์กรควบคุม Physical Path ไม่ได้เองโดยตรง**

### 914.2 สามระดับของ Diversity ที่ต้อง Verify แยกกัน — ไม่ใช่แค่ "มี 2 สาย"

```
ระดับที่ 1 — Logical/Provider Diversity
   ใช้ ISP/Carrier คนละเจ้ากัน (เช่น DCI Circuit A จาก Provider X, Circuit B จาก Provider Y)
   ตรวจสอบง่ายที่สุด แต่ ไม่พอ ถ้า Provider ทั้งสองยังใช้ Physical Infrastructure ร่วมกัน
   (เช่น เช่า Dark Fiber จาก Provider เดียวกันที่เป็นเจ้าของท่อจริง)
        │
ระดับที่ 2 — Physical Path Diversity
   ตรวจสอบว่า Circuit ทั้งสองเดินทางไปคนละเส้นทางจริง (ไม่ทับกันแม้แต่ Segment เดียว)
   ต้องขอ "Physical Route Map" จาก Carrier จริง ไม่ใช่เชื่อคำพูดฝ่ายขายอย่างเดียว
        │
ระดับที่ 3 — Facility/Conduit Diversity (จุดที่พลาดกันบ่อยที่สุด — ตาม Part 72 Step 716)
   แม้ Path ต่างกันเป็นระยะไกล แต่จุด "เข้า-ออกอาคาร" (Building Entrance Facility / Meet-Me-Room)
   มักใช้ท่อร้อยสายเดียวกันหรือช่องเดียวกันเสมอ เพราะอาคารมักมีจุดเข้า Fiber ได้จำกัดจุด
```

### 914.3 บทเรียนจากของจริง: "สองวงจรที่คิดว่า Diverse จริงๆใช้ Conduit เดียวกัน"

```
เหตุการณ์ (Composite จากรูปแบบที่เกิดขึ้นจริงในอุตสาหกรรม):

องค์กรเช่า DCI Circuit 2 วงจรระหว่าง DC1-BGW ↔ DC2-BGW จาก Provider คนละเจ้า (Provider A, B)
ตามหลักการ Provider Diversity (ระดับที่ 1) — ผ่านการ Sign-off จากทีม Procurement ว่า "Redundant"

DR Test ประจำปี: จำลอง Circuit A ขาด (Manual Shutdown ที่ปลาย DC-BGW)
→ Traffic Failover ไป Circuit B สำเร็จ ตามคาด — ทีมมั่นใจว่า Diversity ใช้ได้จริง

6 เดือนต่อมา: ผู้รับเหมาก่อสร้างขุดถนนใกล้ Meet-Me-Room ของอาคาร DC1 โดยไม่ทราบตำแหน่งท่อจริง
→ Backhoe ตัดท่อร้อยสายจุดเดียว → Circuit A **และ** Circuit B ขาดพร้อมกันทั้งคู่

สาเหตุที่พบหลัง Investigation:
Provider A และ Provider B เช่า Conduit จาก "เจ้าของท่อจริง" (Conduit Owner) รายเดียวกัน
ที่จุดเข้าอาคาร (Last-mile Entrance) — แม้ Backbone Network ของสอง Provider แยกกันจริงทั้งเส้น
แต่ระยะ 200 เมตรสุดท้ายก่อนเข้าอาคาร ทั้งสอง Provider ใช้ท่อร้อยสายเดียวกัน (Shared Duct)
เพราะเป็นท่อเดียวที่มีอยู่บริเวณนั้น — ไม่มีใครใน Procurement/Network Team ตรวจสอบระดับนี้เลย
```

> **บทเรียน**: Diversity ต้องถูกตรวจสอบ**ลึกถึงระดับ Physical Route Map ที่ Carrier ต้องยืนยันเป็น
> เอกสาร** (บางครั้งเรียกว่า **"Diverse Route Certificate"** หรือ **KMZ/GIS Map ของเส้นทางจริง**)
> ไม่ใช่แค่เชื่อว่า "Provider คนละเจ้า = Diverse" — และควรมี **Clause ในสัญญา** ที่ระบุชัดว่าห้าม
> ใช้ Facility/Conduit ร่วมกันที่จุดเข้าอาคารทั้งสองฝั่ง (DC1 และ DC2) พร้อมขอสิทธิ์ Audit เส้นทาง
> เป็นระยะ เพราะ Provider อาจเปลี่ยนเส้นทางจริงเมื่อไหร่ก็ได้โดยไม่แจ้งลูกค้า (Provider ทำ Network
> Optimization ของตัวเองโดยไม่รู้ว่าลูกค้ารายนี้ต้องการ Diversity ระดับ Conduit)

### 914.4 Checklist สำหรับ WAN/DCI Diversity Design

| หัวข้อตรวจสอบ | คำถามที่ต้องถาม Provider/Procurement |
|---|---|
| Provider Diversity | Circuit ทั้งสองมาจาก Legal Entity คนละบริษัทจริงหรือไม่ (ไม่ใช่ Reseller ของเจ้าเดียวกัน) |
| Physical Route Diversity | ขอ Route Map/KMZ จาก Provider ทั้งสองเจ้า — เปรียบเทียบว่าทับกันช่วงไหนหรือไม่ |
| Building Entrance Facility | จุดเข้า-ออกอาคารทั้ง DC1 และ DC2 ใช้ท่อ/ช่องเข้าคนละจุดหรือไม่ (Diverse Entry Point) |
| Central Office/POP Diversity | Circuit ทั้งสองผ่าน Central Office/POP (Point of Presence) ของ Provider คนละที่หรือไม่ |
| Regular Re-audit | มี Clause สิทธิ์ขอ Route Map ใหม่เป็นระยะ (Provider เปลี่ยนเส้นทางได้โดยไม่แจ้ง) |
| Live Test เป็นระยะ | ทดสอบ Failover จริง (ไม่ใช่แค่จำลองบน Paper) ตามที่ Step 917 จะสอน |

---

## Step 915 — DNS และ Resilience: TTL Tuning และ Health-check-based Failover

### 915.1 DNS เป็น Dependency ที่ถูกมองข้ามบ่อยที่สุดใน DR Design

[Part 16](part-016-dhcp-dns.md) สอน DNS ในมุม Infrastructure พื้นฐาน (Zone, Record Type, Forward/
Reverse Lookup) — สำหรับ DR โดยเฉพาะ DNS มีบทบาทที่มักถูกมองข้าม: **ไม่ว่า Network/DC2 จะพร้อมรับ
Traffic เร็วแค่ไหน ถ้า Client ยังใช้ IP เดิม (DC1) อยู่ในการเชื่อมต่อเพราะ DNS ยังไม่ Update
Failover ก็ยังไม่เกิดขึ้นจริงในมุมของผู้ใช้งาน** — นี่คือเหตุผลที่ Pattern **Active-Standby DR**
ตาม [Part 85 Step 849](part-085-vxlan-evpn-multisite-advanced.md#step-849--multisite-และ-disaster-recovery-design)
ระบุไว้ชัดว่า "Failover เปลี่ยน DNS/Application config ให้ชี้ไป DC2" — DNS คือกลไกหลักที่ทำให้
Failover แบบ L3-only (ไม่ Stretch Subnet) ใช้งานได้จริงในทางปฏิบัติ

### 915.2 TTL Tuning — Trade-off ระหว่างความเร็วในการ Failover กับภาระของ DNS Infrastructure

**TTL (Time To Live)** ของ DNS Record กำหนดว่า Resolver/Client จะ Cache คำตอบไว้นานแค่ไหนก่อน
ถาม DNS Server ใหม่ — TTL สั้น = Failover เร็วขึ้น (Client ถามใหม่บ่อย เห็นการเปลี่ยนแปลงเร็ว)
แต่ TTL สั้นก็มีต้นทุน:

| TTL | ผลต่อ RTO ของ DNS Failover | ผลข้างเคียง |
|---|---|---|
| ยาว (เช่น 24 ชั่วโมง — Default ทั่วไป) | Failover ใช้เวลานานมาก (Client บางตัว Cache คำตอบเดิมได้ถึง 24 ชม. แม้ DNS Server จะ Update แล้ว) | ภาระ DNS Infrastructure ต่ำ (Query น้อย) |
| สั้น (เช่น 30-60 วินาที) | Failover เร็ว (Client ถามใหม่บ่อย เห็นค่าใหม่เร็ว) | ภาระ DNS Query สูงขึ้นมาก, Cache ที่ Resolver กลางทาง (ISP Resolver, Public DNS) บางตัวไม่เชื่อ TTL สั้นจริง (Cache นานกว่าที่กำหนดเพื่อลดภาระตัวเอง — เป็นความเสี่ยงที่ต้อง Design รับมือ) |

> **แนวทางปฏิบัติที่แนะนำ**: สำหรับ Record ที่เกี่ยวข้องกับ DR โดยเฉพาะ (เช่น VIP ของ Application
> ที่ต้อง Failover ข้าม DC1/DC2) ให้ตั้ง TTL ต่ำล่วงหน้า (เช่น 60 วินาที) **ตั้งแต่ก่อนเกิด
> Disaster** — ไม่ใช่มาลด TTL ตอนกำลังจะ Failover เพราะ Client ที่ Cache ค่า TTL เดิม (ยาว) ไว้
> ก่อนหน้าจะยังใช้ค่าเดิมต่อไปตามอายุ Cache เดิม การลด TTL มีผลแค่กับ Query ใหม่ที่เกิด**หลัง**
> การเปลี่ยนเท่านั้น

### 915.3 Health-check-based DNS Failover — ระบบที่ตรวจสุขภาพและสลับ Record อัตโนมัติ

DNS แบบพื้นฐาน (Static A/AAAA Record) ต้องมีคนไป Update Record เองตอนเกิด Disaster — ระบบ DNS
ระดับ Enterprise/Cloud (มักเรียกรวมว่า **GSLB — Global Server Load Balancing**, หรือ Feature
Health Check ของ DNS Provider) ทำ 3 อย่างเพิ่มจาก DNS ปกติ:

```
DNS Health-check-based Failover — กลไกทำงาน

1. Health Check Probe ──► ยิง HTTP/TCP/ICMP Probe ไปยัง Endpoint ที่ DC1 และ DC2 เป็นระยะ
                            (เช่น ทุก 10-30 วินาที)
2. Decision Engine     ──► ถ้า Probe ไปยัง DC1 Fail ติดต่อกันครบ Threshold (เช่น 3 ครั้งติด)
                            ตัดสินใจว่า DC1 "Unhealthy"
3. Automatic Record Update ──► เปลี่ยนคำตอบของ DNS Query ใหม่ทั้งหมดให้ชี้ไปที่ DC2 ทันที
                                 (ไม่ต้องรอ Human สั่ง Update Record เอง)
```

> **จุดที่ต้องระวัง**: Health Check ต้อง Probe ที่ **จุดที่สะท้อนสุขภาพจริงของ Application** ไม่ใช่
> แค่ Ping อุปกรณ์ Network ว่าตอบหรือไม่ (Network อาจ Up แต่ Application ข้างหลังพังก็ได้ หรือ
> ตรงกันข้าม — Application ยังรันอยู่แต่ Network Path ไปหามันขาด) และต้อง Probe **จากภายนอก
> Site นั้น** (เช่น จาก Cloud-based Health Check Location) ไม่ใช่ Probe จากภายใน Site เดียวกัน
> เพราะถ้า Site ทั้ง Site ล่มจริง (ไฟดับทั้งอาคาร) ตัว Health Check Agent ที่อยู่ใน Site เดียวกัน
> ก็จะล่มไปด้วยและไม่มีใครรายงานผลกลับมาเลย

### 915.4 ทำไม DNS Resilience มักถูกมองข้าม

| เหตุผล | ผลกระทบถ้าไม่ทำ |
|---|---|
| ทีม Network คิดว่า DNS เป็นเรื่องของทีม Server/App | ไม่มีใคร Review TTL ของ Record ที่เกี่ยวกับ DR เลยจนกว่าจะสายเกินไป |
| DNS Server เองบางครั้งตั้งอยู่ที่ Primary Site เท่านั้น | ถ้า DC1 ล่ม DNS Server ที่ต้องใช้ตอบคำถาม Failover ก็ล่มไปด้วย (ต้องมี DNS Server/Secondary ที่ DC2 หรือ Cloud-based DNS ที่ไม่ผูกกับ Site ใด Site หนึ่ง) |
| TTL มักถูกตั้งแบบ Default ยาวโดยไม่มีใครแก้ | Failover ที่วางแผนไว้อย่างดีระดับ Network ใช้เวลาจริงยาวกว่าที่คาดมาก เพราะ Client ยัง Cache ค่าเดิมอยู่ |

---

## Step 916 — Runbook-driven DR: โครงสร้าง Runbook มาตรฐานสำหรับ "Primary DC ล่มทั้ง Site"

### 916.1 "มี Redundancy" ไม่เท่ากับ "จะรับมือได้จริง" — ทำไมต้องมี Runbook ที่เขียนไว้และทดสอบแล้ว

ความเข้าใจผิดที่พบบ่อยที่สุดในองค์กรคือ **"เรามี DC2, มี EVPN Multisite, มี Redundancy ครบแล้ว
เราจะรับมือ Disaster ได้แน่นอน"** — นี่คือความเข้าใจผิดที่อันตรายมาก เพราะ **Technology พร้อม
ไม่ได้แปลว่า Process/คนพร้อม** เมื่อเกิด Disaster จริง (มักเกิดตอนดึก/วันหยุด/ภาวะตื่นตระหนก)
วิศวกรที่ต้อง Response ไม่มีเวลาคิดหาวิธีทำทีละขั้นตอนเอง — ต้องมี **Runbook ที่เขียนไว้ล่วงหน้า
เป็นเอกสาร ทดสอบมาแล้วจริง** ให้ทำตามได้ทันที

### 916.2 โครงสร้าง Runbook มาตรฐาน 6 ส่วน

```
โครงสร้าง DR Runbook มาตรฐาน (ใช้กับสถานการณ์ "Primary DC ล่มทั้ง Site")

1. Scope & Trigger Criteria      ← สถานการณ์ไหนที่ Runbook นี้ใช้ได้ และใครมีสิทธิ์ Declare
2. Roles & Responsibilities      ← ใครทำอะไร (ผูกกับ War-Room ตาม Step 918)
3. Pre-check / Readiness Verify   ← ตรวจสอบว่า DR Site พร้อมรับจริงก่อนสั่ง Failover
4. Step-by-step Execution         ← ขั้นตอนทำจริง ทีละขั้น พร้อมคำสั่ง/Screenshot ที่คาดว่าจะเห็น
5. Validation / Post-check        ← ยืนยันว่า Failover สำเร็จจริง ไม่ใช่แค่ "สั่งไปแล้ว"
6. Rollback / Failback Plan       ← แผนย้อนกลับถ้า Failover ไม่สำเร็จ หรือแผนกลับ Primary
                                     เมื่อ Site เดิมซ่อมเสร็จ
```

### 916.3 ตัวอย่างโครงร่าง Runbook สำหรับ "DC1 ล่มทั้ง Site" บน DC1/DC2 Multisite Fabric

| ส่วน | เนื้อหาที่ต้องมีในสถานการณ์นี้ (อ้างอิง DC1/DC2 จาก Part 84-85) |
|---|---|
| **1. Scope & Trigger** | ใช้เมื่อ: DC1 ทั้ง Site ไม่ตอบสนอง (ไม่ใช่แค่ `DC-LEAF-1` ตัวเดียว — กรณีนั้นใช้ Redundancy ปกติของ Part 81/34 แก้ได้เอง) — Trigger: `DC-BGW-1/2` ทั้งคู่ไม่ตอบ Health Check จาก DC2 ติดต่อกันเกิน Threshold (ดู Step 920 สำหรับเกณฑ์ละเอียด) |
| **2. Roles** | Incident Commander (สั่ง Declare), Network Lead (Execute BGP/Route change ที่ `DC2-BGW-1/2`), DNS/App Lead (Update DNS Record ตาม Step 915), Communication Lead (ตาม Step 918) |
| **3. Pre-check** | `DC2-LEAF-1/2` และ `DC2-BGW-1/2` Up และ Health ปกติ, VNI 30130 (Stretch Subnet) ที่ DC2 มี Local Host พร้อมรับ Traffic, ตรวจสอบ Application ที่ DC2 (VNI 30141, `10.141.30.0/24`) พร้อม Serve จริง (ไม่ใช่แค่ Standby เฉยๆ) |
| **4. Execution** | (ก) ยืนยัน DC1 ล่มจริงตาม Criteria Step 920 (ข) Advertise เส้นทางที่ DC2-BGW ให้เป็น Preferred Path ไปยัง Campus/WAN (ค) Update DNS Record ของ Application VIP ให้ชี้ Endpoint ที่ DC2 (ง) แจ้ง Application Team ให้ยืนยัน Service เริ่มรับ Request จาก DC2 |
| **5. Validation** | Ping/Trace จาก Campus ไปยัง Application VIP ใหม่ผ่าน Path DC2, ตรวจ `show bgp l2vpn evpn` ที่ DC2-BGW ว่า Route ที่เคย Advertise จาก DC1 หายไปแล้วจริง (ไม่มี Stale Route ค้าง), Monitor Error Rate ของ Application จาก DC2 |
| **6. Rollback/Failback** | เมื่อ DC1 ซ่อมเสร็จ: ตรวจ Config/Hardware ของ DC1 ผ่าน Config DR (Step 913) ก่อน Bring-up, Sync ข้อมูลกลับจาก DC2 → DC1 ก่อน, แล้วค่อย Failback แบบ Controlled (ไม่ Rush — Failback ที่เร่งเกินไปเป็นสาเหตุ Incident รอบสองที่พบบ่อย) |

> **กฎเหล็ก**: Runbook ต้องเขียนให้ **คนที่ไม่ใช่ผู้เขียน Runbook เอง** ทำตามได้สำเร็จ — ถ้า
> Runbook ต้องพึ่งความรู้เฉพาะตัวของวิศวกรคนเดียว (Tribal Knowledge) แปลว่า Runbook นั้นยังไม่
> สมบูรณ์ วิธีตรวจสอบที่ดีที่สุดคือให้วิศวกรที่ไม่คุ้นเคยกับ DC1/DC2 Fabric ลองทำตาม Runbook ใน
> Tabletop Exercise (Step 917) ดูว่าติดขัดตรงไหน

---

## Step 917 — DR Testing: Tabletop Exercise vs Live Failover Test

### 917.1 "สร้างไว้แล้วหวังว่าจะทำงาน" คือความผิดพลาดที่พบมากที่สุดของ DR

DR Design ที่ไม่เคยถูกทดสอบมีโอกาสสูงมากที่จะ **ทำงานไม่ได้จริงตอนต้องใช้งานจริง** — สาเหตุที่พบ
บ่อยไม่ใช่ Design ผิดตั้งแต่ต้น แต่เป็น **Config Drift** (Config เปลี่ยนไปทีละนิดหลังจาก Design
ครั้งแรกจนไม่ตรงกับที่ Runbook สมมติไว้), **Dependency ที่ไม่คาดคิด** (เช่น DNS Server ที่ใช้
Failover ดันอยู่ที่ Site เดียวกับที่กำลังจำลองว่าล่ม), หรือ **Human Factor** (คนที่เขียน Runbook
ลาออกไปแล้ว ไม่มีใครรู้รายละเอียดจริง)

### 917.2 สองรูปแบบหลักของ DR Testing

| รูปแบบ | ลักษณะ | ข้อดี | ข้อจำกัด |
|---|---|---|---|
| **Tabletop Exercise** | ทีมนั่งประชุมพร้อมกัน จำลองสถานการณ์ด้วยคำพูด/เอกสาร ("สมมติว่า DC1 ล่มตอนนี้ ทุกคนจะทำอะไร") ไม่แตะอุปกรณ์จริง | ทำได้บ่อย, ไม่มีความเสี่ยงต่อ Production เลย, เหมาะสำหรับฝึก Decision-making และ Communication Flow | ไม่พิสูจน์ว่า Technical Step ทำงานได้จริง (คนอาจตอบถูกในทฤษฎีแต่ Config จริงมีปัญหาที่ไม่มีใครเห็นจนกว่าจะลงมือทำจริง) |
| **Live Failover Test** | สั่ง Failover จริง (เช่น `redundancy force-switchover` ระดับ Chassis ตาม Part 81 Step 809, หรือ Redirect Traffic จริงไปยัง DC2 เต็มรูปแบบ) | พิสูจน์ว่า Technical Step ทำงานได้จริง 100%, เจอปัญหาที่ Tabletop ไม่เจอ (เช่น Timing/Race Condition จริง) | มีความเสี่ยงต่อ Production จริง ถ้า Failover ไม่สำเร็จตามคาด อาจกลายเป็น Incident ของจริงซ้อนเข้ามา |

### 917.3 ความขัดแย้งที่ต้องบริหาร: ทดสอบให้ครบถ้วนที่สุด vs ความเสี่ยงต่อ Production

```
สเปกตรัมของการทดสอบ DR

Tabletop เท่านั้น ──────────────────────────────────────► Live Full-scale Failover
(ปลอดภัยสูงสุด                                              (พิสูจน์จริงสูงสุด
 แต่พิสูจน์ได้น้อยสุด)                                        แต่เสี่ยงสูงสุด)

ระดับกลางที่องค์กรส่วนใหญ่ใช้จริง:
  - Partial Live Test (ทดสอบเฉพาะบางส่วน เช่น Failover เฉพาะ 1 Application ที่ Risk ต่ำ)
  - Live Test ใน Maintenance Window ที่แจ้งล่วงหน้า พร้อม Rollback Plan พร้อมทำทันที
  - Live Test บน Non-production Replica ของ Fabric (ถ้าทำได้ — ต้นทุนสูงแต่ปลอดภัยที่สุด)
```

### 917.4 ตารางแนะนำความถี่และรูปแบบการทดสอบตาม Tier (ต่อยอด Step 911.4)

| Tier | Tabletop | Partial Live Test | Full Live Failover Test |
|---|---|---|---|
| Tier 0 | ทุกไตรมาส | ทุก 6 เดือน | ทุกปี (Maintenance Window ที่วางแผนละเอียด) |
| Tier 1 | ทุก 6 เดือน | ทุกปี | ทุก 1-2 ปี |
| Tier 2-3 | ทุกปี | ตามความเหมาะสม | ไม่บังคับ (ขึ้นกับความเสี่ยงที่ยอมรับได้) |

> **หลักปฏิบัติสำคัญ**: ทุกครั้งที่ Test (ไม่ว่า Tabletop หรือ Live) ต้องมี **After-action Report**
> ที่บันทึกว่า Runbook ส่วนไหนใช้ไม่ได้จริง แล้ว **แก้ Runbook ทันที** ก่อนที่จะลืม — Test ที่ทำแล้ว
> ไม่มีการปรับปรุง Runbook ตามผลลัพธ์คือการเสียเวลาทดสอบไปโดยไม่ได้ประโยชน์ที่แท้จริง (คล้ายกับ
> Post-Incident Review ใน Part 91 ที่เน้นว่าการเรียนรู้จาก Incident ต้องถูกนำไปแก้ Process จริง
> ไม่ใช่แค่เขียนรายงานเก็บไว้)

---

## Step 918 — การสื่อสารและสายบังคับบัญชาระหว่างเกิด DR Event

### 918.1 ทวนโครงสร้าง War-Room จาก Part 69 แล้วขยายเป็นระดับ Business Continuity

[Part 69 Step 682](part-069-mega-lab-1.md#step-682--triage-และ-war-room-process) สอนโครงสร้าง
**War-Room** สำหรับ Major Incident ระดับ Sev-1 ที่ Incident Commander (IC) เป็นผู้นำ พร้อมกฎเหล็ก
เช่น "ห้าม Rollback/แก้ Config ก่อนแจ้ง War Room" — สำหรับ **Disaster ระดับ Site (Business
Continuity Event)** โครงสร้างเดิมยังใช้เป็นฐานได้ แต่ต้องขยาย **ขนาดและ Stakeholder** ให้กว้างขึ้น
มากเพราะผลกระทบไม่ได้จำกัดอยู่แค่ IT/Network เท่านั้น:

| มิติ | War-Room ระดับ Incident (Part 69) | War-Room ระดับ Disaster/Business Continuity (Part นี้) |
|---|---|---|
| ผู้เข้าร่วม | ทีม Technical (Network, Server, Security) | เพิ่ม **ผู้บริหารระดับสูง (C-level), ทีม Legal/Compliance, ทีม PR/Corporate Communication, ทีม HR** (ถ้ากระทบพนักงาน/ความปลอดภัย) |
| Scope การสื่อสาร | ภายในทีม IT และผู้ใช้งานภายใน | เพิ่ม **ลูกค้าภายนอก, คู่ค้า, สื่อ/สาธารณะ** (ถ้ากระทบ Service ที่ลูกค้าเห็น), และในบางกรณี **หน่วยงานกำกับดูแล (Regulator)** ที่ต้องแจ้งตามกฎหมาย |
| ผู้มีสิทธิ์ Declare | IC ระดับ Incident | มักต้องมี **DR Coordinator/Business Continuity Manager** ที่แยกจาก IC ของ Incident ปกติ เพราะการ Declare Disaster มีผลทางธุรกิจ/สัญญา/ประกันภัยด้วย |
| ความถี่ Update | Sync ทุก 15 นาทีระหว่างทีม Technical | เพิ่มชั้น **Executive Briefing** (Update สั้นเป็นภาษาธุรกิจ ไม่ใช่ Technical Detail ทุก 30-60 นาที) แยกจาก Technical Sync |

### 918.2 หลักการสื่อสารที่สำคัญที่สุด: แยก "Technical Channel" กับ "Business/Executive Channel"

```
โครงสร้างการสื่อสารระหว่าง DR Event ขนาดใหญ่

           ┌─────────────────────────────┐
           │   DR Coordinator / BC Manager │  ← จุดตัดสินใจ Declare/Failback สุดท้าย
           └───────────┬─────────────────┘
              ┌─────────┴─────────┐
              ▼                   ▼
   ┌────────────────────┐  ┌────────────────────────┐
   │  Technical War-Room  │  │  Executive/Business     │
   │  (Network/Server/    │  │  Communication Channel  │
   │   App/Security Lead) │  │  (Status ภาษาธุรกิจ,     │
   │  → Execute Runbook   │  │   ไม่ใช่ Log/Command)   │
   │    (Step 916)         │  │  → แจ้งลูกค้า/สื่อ/      │
   └────────────────────┘  │    Regulator ถ้าจำเป็น   │
                            └────────────────────────┘

กฎสำคัญ: Technical War-Room ไม่ควรถูกขัดจังหวะด้วยคำถามจาก Executive ตลอดเวลา
        (เสียสมาธิจากงาน Execute) — ให้ DR Coordinator เป็นคนกลาง Update Executive Channel
        เป็นระยะแทน ไม่ให้ Executive เข้ามาคุยตรงกับ Engineer ที่กำลัง Execute Runbook
```

### 918.3 Checklist การสื่อสารที่ต้องเตรียมไว้ล่วงหน้า (ก่อนเกิด Disaster จริง)

| รายการ | เหตุผล |
|---|---|
| **Contact List ที่ Update เสมอ** พร้อม Backup Contact (เผื่อคนหลักติดต่อไม่ได้) | ระหว่าง Disaster จริง Contact List เก่า/ผิด = เสียเวลาหาคนที่ถูกต้องนานเกินจำเป็น |
| **Communication Channel ที่ไม่ผูกกับ Infrastructure ที่กำลังล่ม** | ถ้าใช้ Chat/Email ที่ Host อยู่ใน Site เดียวกับที่ล่ม จะติดต่อกันไม่ได้พอดีตอนต้องใช้มากที่สุด — ต้องมี Channel สำรอง (เช่น Phone Bridge, SMS, Chat ที่ Host แยก Site/Cloud) |
| **Template ข้อความสื่อสารล่วงหน้า** (สำหรับลูกค้า/สื่อ ถ้าจำเป็น) | ลดเวลาคิดคำพูดตอนกำลังเครียด และลดความเสี่ยงพูดผิดพลาด/ให้ข้อมูลที่ Legal ไม่อนุญาต |
| **นิยามชัดเจนว่าใครมีสิทธิ์พูดกับสื่อ/ลูกค้าภายนอก** | ป้องกัน Engineer ที่กำลังแก้ปัญหาให้สัมภาษณ์สื่อโดยไม่ตั้งใจ (ข้อมูลผิดพลาด/สร้างความเข้าใจผิดเพิ่ม) |

---

## Step 919 — Cyber-incident-driven "Disaster": Ransomware และ Network Isolation

### 919.1 DR ไม่ใช่แค่เรื่อง "ภัยธรรมชาติ/Hardware พัง" — Cyber Incident ก็เป็น Disaster ได้

Step 911-918 พูดถึง DR ในมุม Physical Disaster เป็นหลัก (ไฟไหม้, ไฟดับ, Hardware พัง) แต่ในโลก
จริงปัจจุบัน **สาเหตุของการต้อง Declare Disaster ที่พบมากขึ้นเรื่อยๆ คือ Cyber Incident** โดยเฉพาะ
**Ransomware ที่แพร่กระจายข้าม Network** — ความแตกต่างสำคัญคือ Physical Disaster ทำให้ **ต้องหา
ทางให้ Traffic วิ่งต่อได้** ในขณะที่ Ransomware บางครั้งทำให้ **ต้องตัด Traffic ออกโดยเจตนา
(Isolation)** เพื่อหยุดการแพร่กระจาย — เป้าหมายตรงข้ามกันโดยสิ้นเชิงในบางมิติ แต่ยังใช้ Framework
DR เดียวกัน (RTO/RPO, Runbook, War-Room) ในการบริหารจัดการ

### 919.2 เชื่อมกับ Part 78: Rapid Threat Containment เป็น "DR-like Response" ระดับ Endpoint/Segment

[Part 78 Step 776](part-078-security-integration-ise-trustsec-ftd-umbrella.md#step-776--threat-response-integration-securexxdr-และ-rapid-threat-containment)
สอน **Rapid Threat Containment (RTC)**: ISE Quarantine Endpoint อัตโนมัติผ่าน **ANC (Adaptive
Network Control) + pxGrid + CoA** ทันทีที่ตรวจพบสัญญาณ Compromise — นี่คือ DR-like Response ที่
ทำงานในสเกล **Endpoint เดียวหรือ Segment เล็ก** ระบบ RTC เดิมออกแบบมาสำหรับ "จับ Endpoint ที่ติด
เชื้อแยกออกจากเครือข่ายก่อนแพร่กระจาย" — เมื่อ Ransomware ระบาดรุนแรงจนกลายเป็นสเกล **ทั้ง Site/
ทั้งองค์กร** RTC เพียงอย่างเดียวไม่พอ ต้องยกระดับเป็น **Network-wide Isolation Decision** ซึ่งเป็น
จุดที่ DR Framework (RTO/RPO, Runbook, War-Room ระดับ Business Continuity) เข้ามาทำงานร่วมกับ
Security Incident Response

### 919.3 ตารางเปรียบเทียบ: Physical Disaster DR vs Ransomware DR-like Response

| ประเด็น | Physical Disaster (Step 911-918) | Ransomware/Major Cyber Incident |
|---|---|---|
| เป้าหมายหลักตอนเริ่ม Response | ทำให้ Traffic วิ่งต่อได้เร็วที่สุด (RTO ต่ำ) | **ตัด Traffic/Isolate ก่อน** เพื่อหยุดการแพร่กระจาย แม้จะทำให้ RTO แย่ลงชั่วคราว |
| DR Site (DC2) มีบทบาทอย่างไร | รับ Traffic แทน DC1 ทันที | **ต้อง Verify ว่า DC2/Backup ไม่ได้ติดเชื้อไปด้วยแล้ว** ก่อน Failover ไปใช้ (ถ้า Ransomware กระจายไปถึง DC2 ผ่าน Replication Link ตั้งแต่ก่อนตรวจพบ การ Failover ไป DC2 อาจนำ Malware ไปด้วย) |
| Backup/Config DR (Step 913) มีบทบาทอย่างไร | Restore Config เพื่อสร้างอุปกรณ์ที่ถูกทำลายทาง Physical ใหม่ | **Restore จาก Backup ที่ยืนยันว่า "สะอาด" ก่อนติดเชื้อ (Point-in-time ก่อน Compromise)** — ต้องรู้ว่า Ransomware เริ่ม Compromise ตั้งแต่เมื่อไหร่ (มักนานกว่าที่คิด — Dwell Time หลายสัปดาห์เป็นเรื่องปกติ) |
| War-Room (Step 918) มีบทบาทอย่างไร | Coordinate Technical Response + Communication | เพิ่ม **Legal/Law Enforcement, Cyber Insurance, Forensics Team** เข้าร่วม War-Room ตั้งแต่ต้น (ต่างจาก Physical Disaster ที่มักไม่ต้องมี Forensics) |
| ความเร่งด่วนของการ Isolate | ต่ำ (ไม่มีอะไรต้อง "หยุดไม่ให้แพร่กระจาย") | **สูงสุด** — ทุกนาทีที่ Isolate ช้าลง = Segment ติดเชื้อเพิ่มขึ้น |

### 919.4 หลักปฏิบัติสำคัญ: อย่า Failover ไป DR Site ทันทีโดยไม่ตรวจสอบก่อนในกรณี Cyber Incident

```
ลำดับการตัดสินใจที่ถูกต้องเมื่อสงสัยว่าเป็น Ransomware/Major Cyber Incident
(แตกต่างจาก Physical Disaster Runbook ที่ Failover ได้ทันทีตาม Step 916)

1. Isolate ส่วนที่สงสัยว่าติดเชื้อก่อน (RTC ตาม Part 78 ถ้าเป็นสเกลเล็ก,
   Network-wide Segmentation Isolation ถ้าสเกลใหญ่)
        │
2. เรียก Forensics/Security Team ตรวจสอบ Scope ของการติดเชื้อจริง
   (ติดแค่ Endpoint หรือลามไปถึง Server/Backup Infrastructure แล้ว)
        │
3. ยืนยันว่า DR Site (DC2)/Backup ที่จะใช้ Restore "สะอาด" จริง
   (ตรวจ Log ว่า Replication/Backup ที่มาจาก DC1 ไม่ได้พา Malware ไปด้วยตั้งแต่ก่อนตรวจพบ)
        │
4. ค่อย Failover/Restore ไปยัง DR Site หรือ Point-in-time Backup ที่ยืนยันแล้วว่าสะอาด
```

> **สรุปใจความของ Step นี้**: DR และ Security Incident Response เป็นสอง Discipline ที่ **Framework
> คล้ายกัน (RTO/RPO, Runbook, War-Room)** แต่ **เป้าหมายทันทีตอนเริ่มเหตุการณ์อาจตรงข้ามกัน** —
> Physical Disaster ต้องการ "เร็วที่สุด" ในการทำให้ Traffic วิ่งต่อ ขณะที่ Cyber Incident ต้องการ
> "หยุดก่อน แล้วค่อยตรวจสอบว่าอะไรสะอาดจริง" ก่อนจะ Failover — องค์กรที่มี DR Runbook เดียวสำหรับ
> ทุกสถานการณ์โดยไม่แยกกรณีนี้มีความเสี่ยงสูงที่จะ Failover เร็วเกินไปในกรณี Ransomware แล้วนำ
> Malware ติดไปยัง DR Site ด้วย

---

## Step 920 — Lab เต็มรูปแบบ: DR Runbook Document สำหรับสถานการณ์ "DC1 ล่มทั้ง Site"

### 920.1 เป้าหมายของ Lab

สร้างเอกสาร **DR Runbook** ฉบับสมบูรณ์ที่ใช้งานได้จริง (Deliverable-style Document) สำหรับ
สถานการณ์ **"DC1 ล่มทั้ง Site แบบ Physical Disaster"** โดยใช้ DC1/DC2 EVPN Multisite Fabric จาก
Part 84-85 เป็นฐาน Infrastructure — เอกสารนี้รวบรวมทุก Step ของ Part นี้ (911-919) เข้าเป็น
เอกสารเดียวที่ทีม Operations ใช้ได้ทันทีเมื่อเกิดเหตุจริง

---

<div align="center">

**DR RUNBOOK — DC1 TOTAL SITE FAILURE**
**เอกสารควบคุม: DR-RB-DC1-001 | Classification: Internal — Business Continuity**

</div>

### ส่วนที่ 1 — Pre-failure Readiness Checklist

> ใช้ตรวจสอบเป็นประจำ (ตามความถี่ที่กำหนดใน Step 917.4) **ก่อน**เกิดเหตุจริง — ถ้ารายการใดไม่ผ่าน
> ต้องแก้ไขก่อน ห้ามรอจนกว่าจะเกิด Disaster

| # | รายการตรวจสอบ | เกณฑ์ผ่าน | ผู้รับผิดชอบ |
|---|---|---|---|
| 1 | `DC2-SPINE-1/2`, `DC2-LEAF-1/2`, `DC2-BGW-1/2` สถานะ Operational | `show interface brief` ทุกตัว = Up/Up, ไม่มี Alarm ค้าง | Network Ops |
| 2 | EVPN Multisite BGP Session ระหว่าง `DC-BGW-1/2` ↔ `DC2-BGW-1/2` | `show bgp l2vpn evpn summary` = Established ทั้งคู่ (ตาม Part 85 Step 843) | Network Ops |
| 3 | VNI 30130 (DC-SERVERS, Stretch) เห็น Route จาก DC1 ที่ DC2 | `show l2route evpn mac all` มี Entry จาก Remote VTEP (BGW Anycast VIP `1.1.1.109`) | Network Ops |
| 4 | Config Backup ของทุกอุปกรณ์ DC1/DC2 เป็นปัจจุบัน (ตาม Step 913) | Git Commit ล่าสุดไม่เก่ากว่า RPO ที่กำหนด (Tier 0/1 ตาม Step 911.4) | Automation Team |
| 5 | Git Repository ที่เก็บ Golden Config Host อยู่นอก DC1 | ตรวจสอบ Hosting Location ทุกไตรมาส | Automation Team |
| 6 | DNS Record ของ Application VIP ตั้ง TTL ต่ำไว้ล่วงหน้า (ตาม Step 915.2) | TTL ≤ 60 วินาที สำหรับ Record Tier 0/1 | DNS/App Team |
| 7 | DNS Server สำรองมีอยู่ที่ DC2 หรือ Cloud-hosted (ไม่ผูกกับ DC1 เท่านั้น) | ตรวจสอบ Zone Transfer/Secondary ทำงานปกติ | DNS/App Team |
| 8 | WAN/DCI Circuit Diversity ยืนยันด้วย Route Map ล่าสุด (ตาม Step 914.4) | Re-audit ไม่เกิน 12 เดือนที่ผ่านมา | Network Ops + Procurement |
| 9 | Application ที่ DC2 (VNI 30141, `10.141.30.0/24`) พร้อม Serve จริง ไม่ใช่แค่ Standby เฉยๆ | Health Check Endpoint ที่ DC2 ตอบ 200 OK | App Team |
| 10 | Contact List และ Communication Channel สำรอง Update ล่าสุด (ตาม Step 918.3) | Review ทุกไตรมาส | DR Coordinator |
| 11 | Runbook ฉบับนี้ผ่าน Tabletop Exercise ล่าสุดไม่เกิน 6 เดือน (Tier 0/1 ตาม Step 917.4) | มี After-action Report แนบ | DR Coordinator |

### ส่วนที่ 2 — Failure Detection / Disaster Declaration Criteria

> ป้องกันการ Declare Disaster ผิดพลาด (False Positive ทำให้ Failover โดยไม่จำเป็น) และป้องกันการ
> ช้าเกินไป (True Positive ที่ไม่มีใครกล้าตัดสินใจ)

| เกณฑ์ | รายละเอียด |
|---|---|
| **Signal 1 — Multisite BGP Session Down ทั้งคู่** | `DC2-BGW-1` และ `DC2-BGW-2` ทั้งคู่เห็น eBGP Session ไปยัง `DC-BGW-1/2` เป็น Idle/Active (ไม่ Established) พร้อมกันต่อเนื่องเกิน **5 นาที** |
| **Signal 2 — Anycast VIP ของ DC1 ไม่ตอบสนอง** | Health Check จาก DC2/External Monitoring ไปยัง `1.1.1.109` (DC-BGW Anycast VIP) และ Underlay Loopback อื่นของ DC1 ไม่ตอบทั้งหมดต่อเนื่องเกิน **5 นาที** |
| **Signal 3 — ยืนยันด้วย Out-of-band** | ติดต่อ Site DC1 ผ่านช่องทางอื่น (Phone, Site Security, ไม่ใช่ Network) ยืนยันว่ามีเหตุ Physical จริง (ไฟไหม้/ไฟดับ/น้ำท่วม) — **ห้าม Declare Disaster จาก Signal 1-2 อย่างเดียวโดยไม่พยายาม Verify ด้วย Signal 3 อย่างน้อย 1 ช่องทาง** เพื่อกันกรณี Monitoring/BGP เสียเองโดย DC1 จริงๆยังปกติ |
| **ผู้มีสิทธิ์ Declare** | DR Coordinator (หรือผู้ที่ได้รับมอบหมายตาม On-call Rotation) เท่านั้น — Engineer ระดับปฏิบัติการแจ้งสัญญาณขึ้นมา แต่ไม่ Declare เอง |
| **เกณฑ์ที่ทำให้ "ไม่ Declare"** | ถ้า Signal 1-2 เกิดแต่ Signal 3 ยืนยันว่า DC1 ยัง Physical ปกติ (เช่น เป็นปัญหา DCI Circuit เพียงจุดเดียวที่ Diversity อีกเส้นยังรับได้ตาม Step 914) ให้ใช้ Incident Runbook ปกติ (Part 91) แทน ไม่ต้อง Declare Disaster ระดับ Site |

### ส่วนที่ 3 — Step-by-step Failover Procedure

> ดำเนินการตามลำดับ โดย Network Lead ภายใต้การประสานของ DR Coordinator (War-Room ตาม Step 918)

```
ขั้นที่ 1 — ยืนยัน Disaster Declaration (ตามส่วนที่ 2) และเปิด War-Room
   - DR Coordinator ประกาศ "DR-RB-DC1-001 ACTIVATED"
   - เปิด Technical War-Room Channel (แยกจาก Executive Channel ตาม Step 918.2)
   - แจ้ง Executive Channel ด้วยข้อความ Template ที่เตรียมไว้ล่วงหน้า

ขั้นที่ 2 — ตรวจสอบสถานะ DC2 ล่าสุดก่อนทำอะไรเพิ่ม (Pre-check ซ้ำ ณ เวลาจริง)
   DC2-BGW-1# show bgp l2vpn evpn summary
   DC2-LEAF-1# show nve peers
   ยืนยันว่า DC2 Fabric เองปกติสมบูรณ์ 100% ก่อนจะรับ Traffic เพิ่ม

ขั้นที่ 3 — ตัด Route ที่ Advertise จาก DC1 ออกจาก Global Routing (ถ้ายังเห็น Stale Route ค้าง)
   DC2-BGW-1# show bgp l2vpn evpn | include 1.1.1.109
   (ถ้ายังเห็น Route จาก DC1 Next-hop 1.1.1.109 ค้างอยู่ทั้งที่ DC1 ล่มจริง
    ให้ Clear BGP Session ไปยัง DC-BGW เพื่อบังคับล้าง Stale Route)
   DC2-BGW-1# clear bgp l2vpn evpn neighbor <DC-BGW-1_IP>
   DC2-BGW-2# clear bgp l2vpn evpn neighbor <DC-BGW-2_IP>

ขั้นที่ 4 — ยืนยันว่า Local Route ที่ DC2 (VNI 30130 Stretch) กลายเป็น Best Path เพียงหนึ่งเดียว
   DC2-LEAF-1# show l2route evpn mac all
   (ต้องเห็น MAC ของ Host สำคัญที่เคย Active ที่ DC1 เปลี่ยนมาเรียนรู้จาก Local VTEP ของ DC2 แล้ว
    หรือ Application Team ยืนยันว่าได้ Bring-up Instance ใหม่ที่ DC2 เรียบร้อย)

ขั้นที่ 5 — Update DNS Record ของ Application VIP ให้ชี้ Endpoint ที่ DC2 (ตาม Step 915)
   (ถ้ามี Health-check-based DNS Failover อยู่แล้ว ระบบอาจสลับให้อัตโนมัติตั้งแต่ Signal 2
    ของส่วนที่ 2 — ขั้นนี้คือการ Verify ว่าสลับสำเร็จจริง ไม่ใช่แค่สั่งไปแล้วไม่ตรวจสอบ)
   nslookup app-vip.internal.example.com
   (ผลลัพธ์ที่คาด: IP ที่ตอบกลับต้องเป็น Endpoint ของ DC2 เท่านั้น)

ขั้นที่ 6 — แจ้ง Application/DB Team ให้ยืนยัน Service พร้อม Serve Request จาก DC2 เต็มรูปแบบ
   (รวมถึงตรวจสอบว่า Database ที่ DC2 มีข้อมูลล่าสุดตาม RPO ที่กำหนด — ไม่ใช่ขอบเขตของ
    Network Runbook นี้โดยตรง แต่ Network Lead ต้องรอ Confirm จาก Application Team
    ก่อนประกาศว่า Failover "เสร็จสมบูรณ์" เพราะ Network พร้อมอย่างเดียวไม่พอ)

ขั้นที่ 7 — Update War-Room และ Executive Channel ว่า Failover เสร็จสมบูรณ์
   บันทึกเวลาที่ Failover สำเร็จจริง (สำหรับคำนวณ RTO จริงเทียบกับเป้าหมาย Step 911.4)
```

### ส่วนที่ 4 — Post-failover Validation Checklist

| # | รายการตรวจสอบ | วิธีตรวจสอบ |
|---|---|---|
| 1 | Traffic จาก Campus ไปยัง Application VIP ผ่าน Path DC2 เท่านั้น (ไม่มี Path ค้างไป DC1) | `traceroute` จาก Campus Test Host ไปยัง VIP — ตรวจ Next-hop |
| 2 | ไม่มี Stale BGP EVPN Route ค้างที่ Reference DC1 ({`1.1.1.109`}) | `show bgp l2vpn evpn` ที่ `DC2-BGW-1/2` |
| 3 | Application Error Rate/Latency กลับสู่ปกติ (ภายใน SLA ที่กำหนด) | Application Monitoring Dashboard |
| 4 | DNS Query ใหม่ทั้งหมดได้ IP ของ DC2 (ไม่มี Client ที่ยัง Cache ค่าเก่าเกินคาด) | Monitor DNS Query Log/Resolver Log |
| 5 | ปริมาณ Traffic ที่ DC2 ไม่เกิน Capacity ที่ DC2 Fabric รองรับได้จริง (ตรวจ Bandwidth/CPU ของ `DC2-LEAF/SPINE/BGW`) | Telemetry Dashboard (ต่อยอด Part 80) |
| 6 | เวลา Failover จริง (จาก Declaration ถึง Validation สำเร็จ) บันทึกไว้เทียบกับ RTO เป้าหมาย | เอกสาร Post-incident Timeline |
| 7 | สื่อสารสถานะล่าสุดไปยังผู้เกี่ยวข้องทุกกลุ่มตาม Step 918 (Executive, ลูกค้า ถ้าจำเป็น) | Confirmation จาก Communication Lead |
| 8 | เริ่มวางแผน Failback (แยก Runbook ต่างหาก — ไม่ Rush ตาม Step 916.3 ส่วนที่ 6) | นัดประชุมวางแผน Failback ภายใน 24-48 ชั่วโมง |

> **หมายเหตุปิด Runbook**: Runbook ฉบับนี้ต้องผ่านการทบทวนและ Tabletop/Live Test ตามความถี่ใน
> Step 917.4 เสมอ — เอกสารที่ไม่ได้ Update ให้ตรงกับ Fabric จริง (เช่น IP/VNI เปลี่ยนไปตาม Part
> ที่เขียนเพิ่มหลังจากนี้) คือความเสี่ยงเทียบเท่ากับไม่มี Runbook เลย

---

## Quiz ทบทวน Part 92

1. RTO และ RPO เมื่อนำมาใช้กับ **Network Config/State** (ไม่ใช่ Data ของ Application) แต่ละตัว
   วัดอะไร และเพราะเหตุใด Network RTO ไม่ควรยาวกว่า RTO ของ Application ที่ Critical ที่สุดที่วิ่ง
   อยู่บน Network นั้น?
2. Hot Site, Warm Site, และ Cold Site ต่างกันอย่างไรในมุม Network Design (L2 Stretch vs L3-only
   vs ไม่มี Path เตรียมไว้เลย) และ EVPN Multisite จาก Part 85 สนับสนุน Pattern ไหนเป็นหลัก?
3. เพราะเหตุใด "มี Circuit สองเส้นจาก Provider คนละเจ้า" ยังไม่เพียงพอที่จะเรียกว่า WAN/DCI
   Diversity ที่สมบูรณ์ และต้องตรวจสอบอะไรเพิ่มเติมตาม Step 914?
4. ในกรณี Ransomware/Major Cyber Incident เพราะเหตุใดลำดับการตัดสินใจ (Isolate ก่อน แล้วค่อย
   Verify ความสะอาดของ DR Site ก่อน Failover) จึงต่างจาก Physical Disaster Runbook ที่ Failover
   ไปยัง DR Site ได้ทันที?
5. Config DR (Step 913) แก้ปัญหาอะไรที่แตกต่างจากการมี DR Site (Step 912) และเพราะเหตุใด Git
   Repository ที่เก็บ Golden Config จึงต้อง Host อยู่นอก Site ที่กำลังป้องกัน DR เสมอ?

**เฉลย:**

1. RTO ของ Network วัดเวลาตั้งแต่ Detect ว่า Site ล่ม จนถึง Traffic เริ่ม Forward ผ่าน Path ใหม่
   ได้จริง (รวม Detection + Decision + Reconfigure + Propagation เช่น DNS TTL) — RPO ของ Network
   วัด Config Drift ที่ยอมรับได้ (ความต่างระหว่าง Backup ล่าสุดกับ Config จริง ณ เวลาที่ Disaster
   เกิด) ไม่ใช่ Transaction/ข้อมูลแบบ Data DR — Network ต้อง RTO สั้นกว่าหรือเท่ากับ Application
   ที่ Critical ที่สุดเพราะ Network เป็น Shared Dependency ของทุก Application บน Site เดียวกัน
   ถ้า Network RTO ยาวกว่า ต่อให้ Application/Data พร้อมเร็วแค่ไหนก็ไปไม่ถึงผู้ใช้งานอยู่ดี
2. Hot Site ใช้ L2 Stretch (Subnet เดียวกันมีอยู่ทั้งสอง Site ผ่าน BGW Next-Hop Rewrite) รองรับ
   Active-Active/RTO ใกล้ 0 — Warm Site ใช้ L3-only (Subnet แยกกันคนละ Site, Failover ทำที่
   Application/DNS Layer) RTO ระดับชั่วโมง — Cold Site ไม่มี Network Path เตรียมไว้ล่วงหน้าเลย
   ต้อง Config ใหม่ทั้งหมดตอนเกิดเหตุ RTO ระดับวัน-สัปดาห์ — EVPN Multisite (Part 85) รองรับได้
   ทั้ง L2 Stretch (Hot/Active-Active) และ L3-only (Warm/Active-Standby) แต่ที่ทำให้เป็น "Enabler"
   ของ Hot Site โดยเฉพาะคือ VNI ที่ Stretch ข้าม Site ผ่าน BGW
3. เพราะ Provider Diversity เป็นแค่ระดับที่ 1 จาก 3 ระดับ — Provider คนละเจ้าอาจยังเช่า Conduit/
   Facility ร่วมกันที่จุดเข้า-ออกอาคาร (Building Entrance) หรือใช้ Central Office/POP เดียวกัน
   ต้องขอ Physical Route Map/KMZ จาก Carrier จริงเพื่อยืนยัน Physical Path และ Building Entrance
   Facility ไม่ทับกัน พร้อม Re-audit เป็นระยะเพราะ Provider เปลี่ยนเส้นทางได้โดยไม่แจ้ง
4. เพราะเป้าหมายทันทีของ Ransomware ต่างจาก Physical Disaster โดยสิ้นเชิง — Physical Disaster
   ต้องการให้ Traffic วิ่งต่อเร็วที่สุด (Failover ทันทีปลอดภัย) แต่ Ransomware ยังมีความเสี่ยงว่า
   Malware อาจกระจายไปถึง DR Site/Backup ผ่าน Replication Link ไปแล้วตั้งแต่ก่อนตรวจพบ (Dwell
   Time มักนานหลายสัปดาห์) การ Failover/Restore ทันทีโดยไม่ตรวจสอบก่อนอาจนำ Malware ติดไปด้วย
   จึงต้อง Isolate ก่อน ตรวจ Scope ด้วย Forensics แล้วยืนยันว่า DR Site/Backup สะอาดจริงก่อน
5. DR Site (Step 912) แก้ปัญหา "ทำให้ Traffic ยังวิ่งได้ที่ Site อื่น" แต่ Config DR (Step 913)
   แก้ปัญหาคนละชั้น: "ถ้าอุปกรณ์ถูกทำลายจริง จะสร้าง Config ของมันขึ้นมาใหม่ได้เร็วแค่ไหน" (ไม่
   ใช่แค่ Network ไม่ล่ม แต่ Network ต้อง "สร้างใหม่ได้") — Git ต้อง Host นอก Site ที่ป้องกัน DR
   เพราะถ้า Git Server เอง Host อยู่ใน Site ที่ถูกทำลายพร้อมอุปกรณ์จริง แผนผัง/Golden Config
   ทั้งหมดจะหายไปพร้อมกัน ทำให้ Config DR ใช้งานไม่ได้เลยตอนที่ต้องการมากที่สุด

---

## สรุป Part 92

Part นี้ยกระดับแนวคิด High Availability ที่หลักสูตรสอนมาตลอด (Device/Chassis-level ใน
[Part 81](part-081-high-availability-nsf-sso-issu.md), Link/FHRP-level ใน
[Part 18/35](part-035-advanced-fhrp-campus-ha.md)) ไปสู่ **ระดับ Site/Business Continuity** ที่
[Part 85 Step 849](part-085-vxlan-evpn-multisite-advanced.md#step-849--multisite-และ-disaster-recovery-design)
เปิดประเด็นไว้: เริ่มจาก **RTO/RPO ที่นิยามใหม่สำหรับ Network Config/State** (ไม่ใช่แค่ Data),
**DR Site Model สามระดับ (Hot/Warm/Cold)** พร้อมเชื่อมกับ EVPN Multisite ว่าเป็น Network-layer
Enabler ของ Pattern เหล่านี้, **Config DR** ที่ต่อยอด Automated Backup (Part 17/47) และ Git as
Source of Truth (Part 79) ให้ครอบคลุมถึงการ Rebuild อุปกรณ์ที่ถูกทำลายจริง, **WAN/DCI Circuit
Diversity** ที่ขยาย Conduit Diversity Rule จาก Part 72 Step 716 ไปสู่ระดับ Carrier/Facility พร้อม
บทเรียนจากของจริงเรื่อง Shared Conduit, **DNS Resilience** ที่ต่อยอด Part 16 ด้วย TTL Tuning และ
Health-check-based Failover ซึ่งเป็น Dependency ที่มักถูกมองข้าม, **Runbook-driven DR** พร้อม
โครงสร้างมาตรฐาน 6 ส่วนและตัวอย่างเต็มรูปแบบ, **DR Testing Discipline** ที่ชั่งน้ำหนักระหว่าง
Tabletop Exercise กับ Live Failover Test, **War-Room ระดับ Business Continuity** ที่ขยายจาก
Part 69 ให้ครอบคลุม Executive/Legal/PR, และปิดท้ายด้วยเส้นแบ่งสำคัญระหว่าง **Physical Disaster DR**
กับ **Cyber-incident-driven Response** ที่ต่อยอด Rapid Threat Containment จาก Part 78 — จบด้วย
**DR Runbook Document ฉบับสมบูรณ์**สำหรับสถานการณ์ "DC1 ล่มทั้ง Site" ที่ใช้งานได้จริงบน DC1/DC2
Multisite Fabric

หลักสูตรตอนนี้ได้เตรียม Enterprise Network ให้พร้อมรับมือความล้มเหลวในทุกระดับ ตั้งแต่ Chassis
เดียว (Part 81) จนถึง Data Center ทั้ง Site (Part นี้) — Part ถัดไปจะเปลี่ยนมุมมองจาก "ความอยู่รอด
เมื่อเกิดภัย" ไปสู่ "การขยายขอบเขต Enterprise Network ออกไปยัง Public Cloud": **Cloud Connectivity
กับ AWS/Azure/GCP** ที่จะสอนการเชื่อมต่อ On-premises Network เข้ากับ Cloud Provider หลักทั้งสาม
ผ่าน Direct Connect/ExpressRoute/Cloud Interconnect, Hybrid Routing, และการออกแบบที่ทำให้ Cloud
กลายเป็นส่วนขยายตามธรรมชาติของ Enterprise Fabric ที่สร้างมาตลอด 92 Part ที่ผ่านมา

**ไปต่อ:** [Part 93 — Cloud Connectivity: AWS, Azure, GCP →](part-093-cloud-connectivity-aws-azure-gcp.md)
