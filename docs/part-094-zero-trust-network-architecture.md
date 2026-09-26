# Part 94 — Zero Trust Network Architecture
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 931–940 จาก 1000 | ระดับ CCIE Enterprise Infrastructure**

> ต่อจาก [Part 93 — Cloud Connectivity: AWS/Azure/GCP](part-093-cloud-connectivity-aws-azure-gcp.md)
> ที่ขยาย Enterprise Lab ออกจาก On-prem ไปเชื่อมต่อ Public Cloud ทั้ง 3 ราย (Direct Connect/
> ExpressRoute/Cloud Interconnect, Transit Gateway/Virtual WAN, และ Hybrid Routing ข้าม Cloud)
> ทำให้ Attack Surface ของ "องค์กร" ที่เราออกแบบมาตลอดหลักสูตรนี้ขยายออกไปไกลกว่าคำว่า
> "เครือข่ายภายใน" อย่างสิ้นเชิงแล้ว — Part นี้คือจุดที่เราต้องหยุดแล้วถามคำถามที่ใหญ่ที่สุดของ
> หลักสูตรทั้งหมด: **"เมื่อ Endpoint, Application, และ Data กระจายอยู่ทุกที่ทั้ง Campus/Branch/
> Cloud แล้ว แนวคิดเรื่อง Trusted Internal Network ที่ [Part 1-10](part-001-networking-fundamentals.md)
> วางไว้เป็นฐานของทั้งหลักสูตร ยังใช้ได้จริงอยู่ไหม"**
>
> คำตอบคือ **ไม่ได้อีกต่อไป** — และ **Zero Trust Network Architecture (ZTA)** คือกรอบความคิดที่
> เข้ามาแทนที่ Perimeter Model แบบเดิม Part นี้เป็น **Part สุดท้ายของสาย "Technology Deep-Dive"**
> ของหลักสูตร (Part 1-94) ก่อนที่ [Part 95-100](part-095-ccie-lab-scenario-1-diagnose.md) จะเปลี่ยน
> รูปแบบทั้งหมดไปเป็น **Integrated CCIE Lab Scenario/Capstone** ที่ไม่มีเทคโนโลยีใหม่ให้เรียนอีกแล้ว
> มีแต่การ **สังเคราะห์** ทุกอย่างที่เรียนมาตลอด 94 Part ให้ทำงานร่วมกันภายใต้ Time Pressure จริง
> — Part นี้จึงทำหน้าที่เป็น **สะพานเชื่อม**: มันไม่ได้สอนเทคโนโลยีใหม่จากศูนย์สักตัวเดียว แต่เป็น
> การเอาชิ้นส่วน Security ที่กระจายอยู่ทั่วทั้งหลักสูตร (Part 22, 37, 43, 44, 51, 54, 76, 77, 78, 80)
> มา **จัดเรียงใหม่ภายใต้กรอบคิด Zero Trust เดียว** และ Step 940 จะเป็น **Capstone Assessment**
> ของ Enterprise Lab ทั้งก้อนที่สร้างมาตั้งแต่ Part 1

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 931 | Zero Trust Fundamentals — "Never Trust, Always Verify" และการล่มสลายของ Perimeter Model |
| 932 | 5 Pillars ของ Zero Trust — แมป Pillar แต่ละตัวเข้ากับ Part ที่หลักสูตรนี้สร้างไว้แล้ว |
| 933 | ZTNA vs Traditional VPN — ทำไม Per-Application Access ถึงมาแทน "ต่อ VPN แล้วเท่ากับอยู่ในเครือข่าย" |
| 934 | Micro-Segmentation คือ Network Pillar ของ Zero Trust — ทวน TrustSec/SGT ให้เป็นภาษา ZT อย่างเป็นทางการ |
| 935 | Continuous Verification & Adaptive Access — จาก "Verify ครั้งเดียว" สู่ "Verify ตลอดเวลา" |
| 936 | Zero Trust บน Campus/LAN — 802.1X + TrustSec คือ ZT ระดับ Wired ที่มีอยู่แล้ว ไม่ต้องรอ Cloud |
| 937 | Zero Trust บน WAN/Branch — SD-WAN Security + Cloud OnRamp for SaaS ในกรอบคิด ZT |
| 938 | Zero Trust Maturity Roadmap — แนวทาง Implement แบบ Incremental ไม่ใช่ Big Bang |
| 939 | Zero Trust และ Compliance/Audit — ทำไม ZT คือ "ผลพลอยได้" ของ Compliance ที่ดี ไม่ใช่ภาระเพิ่ม |
| 940 | Lab: Zero Trust Maturity Assessment & Roadmap สำหรับ Enterprise Lab ทั้งหมด (Part 1-93) |

---

## Step 931 — Zero Trust Fundamentals: "Never Trust, Always Verify" และการล่มสลายของ Perimeter Model

### สมมติฐานที่ฝังอยู่ใน Part 1-10 โดยไม่มีใครพูดถึงตรงๆ

กลับไปดู [Part 1 Step 10](part-001-networking-fundamentals.md) ที่ประกาศ Enterprise Lab Topology
หลักของหลักสูตร — Three-Tier Hierarchical Model (Core/Distribution/Access) ที่มี **FW-1 อยู่ที่ Edge
เพียงจุดเดียว** ระหว่าง Internet กับ Internal Network สถาปัตยกรรมแบบนี้มีสมมติฐานที่ **ไม่เคยเขียน
ไว้ตรงๆ แต่ฝังอยู่ในทุก Design Decision ตั้งแต่ Part 1**:

> **"ทุกอย่างที่อยู่หลัง FW-1 (คือ 'ข้างใน' Core/Distribution/Access) คือ Trusted Zone — ทุกอย่างที่
> อยู่หน้า FW-1 (คือ Internet) คือ Untrusted Zone"**

โมเดลนี้เรียกว่า **Castle-and-Moat Model** (ปราสาทและคูน้ำ) — สร้างกำแพงหนาที่ Perimeter จุดเดียว
แล้วเชื่อทุกอย่างข้างใน เหมือนปราสาทยุคกลางที่มีคูน้ำรอบเดียวและเชื่อทุกคนที่อยู่ในกำแพง

```
Castle-and-Moat Model (สมมติฐานเดิมของ Part 1-10)

                    UNTRUSTED                    TRUSTED
                  (Internet)                  (Internal Network)
                       │                              │
                       │        ┌─────────┐           │
        Attacker ──────┼───────►│  FW-1     │──────────┼──► PC1, Server1, ทุกอุปกรณ์
                       │        │ (Moat)    │           │    "ข้างใน" — ไม่ถูกตรวจสอบซ้ำอีก
                       │        └─────────┘           │    เพราะ "อยู่ในกำแพงแล้ว"
                       │                              │
                  ปัญหา: ถ้า Attacker หลุดผ่าน FW-1 มาได้ครั้งเดียว (Phishing, Malware, USB,
                  Compromised Credential) มันจะเดิน "อย่างอิสระ" ไปทุกที่ในเครือข่ายได้เลย —
                  ไม่มี Checkpoint ที่สองรออยู่ (Lateral Movement ไม่มีอะไรกั้น)
```

### 3 แรงกดดันที่ทำให้ Castle-and-Moat Model ใช้ไม่ได้อีกต่อไปในโลกจริง

| แรงกดดัน | รายละเอียด | เชื่อมโยงกับ Part ที่ผ่านมาของหลักสูตรนี้ |
|---|---|---|
| **1. Remote/Hybrid Work** | พนักงานทำงานจากที่ไหนก็ได้ ไม่ได้เสียบสาย LAN ที่ Office เสมอไป — "ตำแหน่งเครือข่าย" ไม่ใช่ตัวบ่งบอกความน่าเชื่อถืออีกต่อไป | [Part 37 Step 373](part-037-wireless-security-advanced.md) (Roaming), [Part 78 Step 772](part-078-security-integration-ise-trustsec-ftd-umbrella.md) (Umbrella Roaming Client) |
| **2. Cloud & SaaS Adoption** | Application สำคัญไม่ได้อยู่ใน Data Center ขององค์กรแล้ว — อยู่บน AWS/Azure/GCP หรือ SaaS (Office 365, Salesforce) ที่ Traffic ไม่จำเป็นต้องผ่าน FW-1 เลย | [Part 76 Step 755](part-076-sdwan-advanced-design.md) (Direct Internet Access), [Part 93](part-093-cloud-connectivity-aws-azure-gcp.md) (Multi-cloud) |
| **3. Insider Threat / Assume Breach** | Perimeter แข็งแรงแค่ไหนก็ป้องกัน Insider ที่มี Credential ถูกต้องอยู่แล้วไม่ได้ — ต้องสมมติว่า "มี Endpoint ที่ถูก Compromise อยู่แล้วเสมอ" (Assume Breach) | [Part 78 Step 776](part-078-security-integration-ise-trustsec-ftd-umbrella.md) (Rapid Threat Containment) |

### นิยาม Zero Trust — สิ่งที่ต้องแม่นสำหรับข้อสอบ CCIE Security/Enterprise

**Zero Trust** ไม่ใช่ Product ตัวเดียวที่ซื้อมาแล้วติดตั้งจบ — มันคือ **กรอบความคิด (Security
Model)** ที่สรุปได้ด้วยประโยคเดียว:

> **"Never Trust, Always Verify"** — ไม่มี User, Device, Application หรือ Traffic ใดได้รับความ
> ไว้ใจโดยอัตโนมัติ ไม่ว่าจะอยู่ที่ไหนบนเครือข่าย (ข้างในหรือข้างนอก Perimeter เดิม) — **ทุก Request
> การเข้าถึงต้องถูกยืนยันตัวตนและตรวจสอบสิทธิ์ทุกครั้ง** โดยพิจารณาจาก Identity, Device Posture,
> และ Context (เวลา, ตำแหน่ง, Behavior) แบบ Real-time — ไม่ใช่พิจารณาจาก "อยู่ใน Subnet ไหน"

NIST SP 800-207 (มาตรฐานอ้างอิงหลักของ Zero Trust Architecture ที่ข้อสอบ CCIE Security อ้างถึง)
สรุปหลักการเป็น 7 Tenet — แต่สำหรับหลักสูตรนี้ เราสรุปเป็น **4 หลักการปฏิบัติจริง** ที่ทุก Step
ในภาคนี้จะอ้างอิงกลับมาตลอด:

| หลักการ | ความหมาย | ตรงข้ามกับ Castle-and-Moat อย่างไร |
|---|---|---|
| **Verify Explicitly** | ยืนยันตัวตนทุกครั้งด้วย Signal ทั้งหมดที่มี (Identity + Device + Location + Behavior) ไม่เชื่อแค่ตำแหน่งเครือข่าย | Castle-and-Moat เชื่อ "อยู่ข้างใน" = ปลอดภัย ไม่ต้อง Verify ซ้ำ |
| **Least Privilege Access** | ให้สิทธิ์เท่าที่จำเป็นต่องานนั้นจริงๆ (Just-Enough-Access) และจำกัดเวลา (Just-in-Time) | Castle-and-Moat ให้สิทธิ์เต็มเครือข่ายทันทีที่ผ่าน Perimeter |
| **Assume Breach** | ออกแบบเหมือนมี Attacker อยู่ในเครือข่ายแล้วเสมอ — จำกัด Blast Radius ด้วย Micro-Segmentation | Castle-and-Moat ป้องกันแค่ "ไม่ให้เข้ามา" ไม่มีแผนถ้าเข้ามาได้แล้ว |
| **Continuous Monitoring** | ตรวจสอบ Trust ต่อเนื่องตลอด Session ไม่ใช่แค่ตอน Login ครั้งแรก | Castle-and-Moat ตรวจแค่ตอนผ่าน FW-1 ครั้งเดียว หลังจากนั้นไม่ตรวจอีก |

### เปรียบเทียบภาพรวม Perimeter Model vs Zero Trust Model

```
PERIMETER MODEL (Part 1-10 สมมติฐานเดิม)         ZERO TRUST MODEL (Part 94 — Part นี้)

        Internet                                  Internet    Campus    Branch    Cloud
           │                                          │          │         │         │
        ┌──┴──┐                                       └──────────┼─────────┼─────────┘
        │ FW-1 │  <- Checkpoint เดียว                             │
        └──┬──┘                                       ┌───────────┴───────────┐
           │  หลัง Checkpoint นี้ = Trust เต็มที่          │  POLICY ENGINE กลาง     │
      ┌────┴──────────────┐                          │  (ISE + Context Signal) │
      │  Trusted Zone       │                        └───────────┬───────────┘
      │  (ไม่ตรวจซ้ำอีก)      │                                     │ ทุก Request ต้องผ่าน
      │  ทุกอุปกรณ์ ทุก User   │                          Policy Decision จุดนี้เสมอ
      └────────────────────┘                          ไม่ว่าจะมาจาก "ที่ไหน" บนเครือข่าย

ข้อสังเกต: Zero Trust ไม่ได้ "ลบ" Firewall/Perimeter ทิ้ง — มันเปลี่ยนจาก "1 Checkpoint ใหญ่ที่ Edge"
ไปเป็น "Checkpoint เล็กจำนวนมากกระจายทุกจุดที่มีการเข้าถึงเกิดขึ้น" (ทุก Switch Port, ทุก
Application, ทุก API Call)
```

> **สิ่งที่ต้องเข้าใจให้ถูกก่อนเข้า Step 932**: Zero Trust ไม่ใช่ "เทคโนโลยีใหม่" ที่ต้องซื้อมาติดตั้ง
> จากศูนย์ — มันคือ **กรอบคิดที่จัดระเบียบเทคโนโลยีที่มีอยู่แล้ว** ให้ทำงานเป็นระบบเดียวกัน องค์กร
> ที่ทำ ISE + TrustSec + FTD + Umbrella ครบตามที่ [Part 43](part-043-security-architecture-firepower-ise.md),
> [Part 44](part-044-dot1x-trustsec.md), และ [Part 78](part-078-security-integration-ise-trustsec-ftd-umbrella.md)
> สอนไปแล้ว **ได้ Implement Zero Trust ไปมากแล้วโดยไม่รู้ตัว** — Step 932 จะพิสูจน์เรื่องนี้แบบ
> Pillar-by-Pillar

---

## Step 932 — 5 Pillars ของ Zero Trust: แมป Pillar แต่ละตัวเข้ากับ Part ที่หลักสูตรนี้สร้างไว้แล้ว

### กรอบ Pillar มาตรฐาน (อ้างอิง CISA Zero Trust Maturity Model)

CISA (Cybersecurity and Infrastructure Security Agency) แบ่ง Zero Trust Architecture ออกเป็น
**5 Pillars** ที่ต้อง Mature ไปด้วยกัน (ไม่ใช่ทำ Pillar เดียวแล้วเรียกว่า Zero Trust ได้) — โดยมี
**Cross-cutting Capability** (Visibility & Analytics, Automation & Orchestration, Governance)
คอยเชื่อมทุก Pillar เข้าด้วยกัน

```
                    ┌─────────────────────────────────────────────────┐
                    │           CROSS-CUTTING CAPABILITIES              │
                    │  Visibility & Analytics | Automation | Governance │
                    └─────────────────────────────────────────────────┘
                                            │
        ┌──────────┬──────────┬─────────────┼─────────────┬──────────┐
        ▼          ▼          ▼             ▼             ▼
   ┌─────────┐┌─────────┐┌───────────┐┌─────────────┐┌─────────┐
   │IDENTITY  ││ DEVICE   ││ NETWORK/   ││ APPLICATION/  ││  DATA    │
   │ (ใคร)     ││ (อะไร)   ││ ENVIRONMENT││  WORKLOAD     ││ (อะไรที่  │
   │           ││          ││ (ที่ไหน)    ││  (บริการอะไร)  ││  ป้องกัน) │
   └─────────┘└─────────┘└───────────┘└─────────────┘└─────────┘
```

### แมป Pillar เข้ากับสิ่งที่หลักสูตรนี้สร้างไว้แล้ว — ตารางหลักของ Step นี้

| Pillar | คำถามที่ตอบ | Feature ที่สร้างไว้แล้วในหลักสูตรนี้ | สร้างไว้ที่ | สถานะความสมบูรณ์ |
|---|---|---|---|---|
| **Identity** | ใครคือผู้ใช้/อุปกรณ์ที่ร้องขอเข้าถึง และพิสูจน์ได้แค่ไหน | AAA/TACACS+ (Device Admin), 802.1X/EAP (User Authentication), Cisco ISE เป็น Policy Engine กลาง | [Part 22](part-022-aaa-device-hardening.md), [Part 37](part-037-wireless-security-advanced.md), [Part 43](part-043-security-architecture-firepower-ise.md) | ✅ แข็งแรง — มี Identity Source เดียว (ISE) ที่ทุกระบบอื่นอ้างอิง |
| **Device** | อุปกรณ์นี้ "สุขภาพดี" พอจะเข้าเครือข่ายไหม (Patch, AV, Encryption, Jailbreak) | ISE Posture Assessment — ตรวจสถานะ Endpoint ก่อน/หลัง Authorize, Re-authorize ผ่าน CoA เมื่อสถานะเปลี่ยน | [Part 43 Step 428](part-043-security-architecture-firepower-ise.md) | ✅ มีพื้นฐานแล้ว — แต่ยังเป็น Binary (Compliant/Non-Compliant) ไม่ใช่ Risk Score ต่อเนื่อง (ดู Step 935) |
| **Network/Environment** | จำกัดการเดินทางของ Traffic ไม่ให้ "ไปไหนก็ได้" หลัง Authenticate | TrustSec/SGT + SGACL — Micro-Segmentation ระดับกลุ่ม ไม่ใช่แค่ VLAN/Subnet | [Part 44](part-044-dot1x-trustsec.md), [Part 77](part-077-sd-access-advanced-design.md) | ✅ แข็งแรงมาก — คือ Implementation ของ Network Pillar ที่สมบูรณ์ที่สุดใน Stack นี้ (ดู Step 934) |
| **Application/Workload** | Traffic ที่เข้าถึง Application ต้องถูกตรวจสอบระดับ Layer 4-7 ด้วย Context เดียวกับ Identity/SGT | FTD/FMC ACP ที่ Match ด้วย SGT ผ่าน pxGrid — Firewall Policy ผูกกับ "ตัวตน" ไม่ใช่แค่ IP | [Part 43](part-043-security-architecture-firepower-ise.md), [Part 78 Step 774](part-078-security-integration-ise-trustsec-ftd-umbrella.md) | ✅ มีพื้นฐานแล้วสำหรับ Network-facing App — ยังไม่ครอบคลุม Workload ระดับ Container/Microservice ใน Data Center (ดู Step 938 Gap) |
| **Data** | ข้อมูลถูก Classify, Encrypt, และควบคุมการเข้าถึงตาม Sensitivity หรือไม่ | ยังไม่มี Coverage เต็มรูปแบบในหลักสูตรนี้ — มีแค่ชิ้นเล็กๆ (TLS สำหรับ Management Plane, PKI จาก Part 78 Step 778) | กระจายอยู่หลาย Part แบบไม่เป็นระบบ | ⚠️ **Pillar ที่อ่อนที่สุด** — Part นี้จะเติมแนวคิดพื้นฐานให้ (ดูด้านล่าง) |

### เจาะ Pillar "Data" ที่หลักสูตรนี้ยังไม่ได้สอนอย่างเป็นระบบ — เนื้อหาใหม่ของ Step นี้

ต่างจาก 4 Pillar แรกที่มี Part เฉพาะรองรับอยู่แล้ว **Data Pillar ยังไม่มี Part ไหนสอนตรงๆ** —
นี่คือช่องว่างที่แท้จริงของหลักสูตร (ไม่ใช่แค่ "ยังไม่ถึง Part" แบบ Pillar อื่น) เพราะ Data Security
เป็นเรื่องของ **Data Classification, Encryption, Data Loss Prevention (DLP), Rights Management**
ซึ่งเป็นงานระดับ Application/Endpoint มากกว่า Network Infrastructure — Part นี้จึงให้แค่
**กรอบความคิดพื้นฐาน** เพื่อให้ภาพ 5 Pillar สมบูรณ์ ไม่ลงรายละเอียด Config เพราะเกินขอบเขต
Enterprise Network CCIE:

| องค์ประกอบของ Data Pillar | หลักการ | เชื่อมโยงกับ Network Layer ที่หลักสูตรนี้สอน |
|---|---|---|
| **Data Classification** | แยกข้อมูลตามระดับความอ่อนไหว (Public/Internal/Confidential/Restricted) ก่อนกำหนด Policy อะไรทั้งสิ้น | SERVERS_SGT (Part 44) เป็นตัวแทน "Classification ระดับ Network" อย่างหยาบๆ — ข้อมูลบน Server1-3 ถูกจัดเป็นกลุ่มที่ต้อง Segment แยก แต่ยังไม่ได้ Classify ถึงระดับ "ไฟล์ไหนอ่อนไหวแค่ไหน" |
| **Encryption at Rest** | ข้อมูลที่เก็บอยู่ต้องเข้ารหัส ไม่ใช่แค่ตอนส่งผ่านเครือข่าย | อยู่นอกขอบเขต Network Infrastructure โดยตรง — เป็นหน้าที่ของ Storage/Database Team แต่ Network ต้องรองรับ (เช่น ไม่ Terminate TLS กลางทางโดยไม่จำเป็น) |
| **Encryption in Transit** | ข้อมูลที่วิ่งผ่านเครือข่ายต้องเข้ารหัสเสมอ ไม่เชื่อว่า "อยู่ใน Internal Network แล้วปลอดภัย" | IPsec/GRE-over-IPsec ([Part 51](part-051-vrf-lite-gre-ipsec.md)), TLS สำหรับ Management (NETCONF/RESTCONF — [Part 49](part-049-netconf-restconf-yang-deep-dive.md)), pxGrid Mutual TLS ([Part 78 Step 775](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) |
| **Data Loss Prevention (DLP)** | ตรวจจับและบล็อกการนำข้อมูลอ่อนไหวออกนอกองค์กรโดยไม่ได้รับอนุญาต | Umbrella CASB ([Part 78 Step 772](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) ให้ Visibility ระดับ SaaS App แต่ยังไม่ใช่ DLP เต็มรูปแบบ (ต้องมี Content Inspection ระดับ Document/File) |
| **Rights Management** | ควบคุมว่าใครเปิด/แก้/แชร์ไฟล์ได้ แม้ไฟล์นั้นออกจากองค์กรไปแล้ว | อยู่นอกขอบเขต Network Infrastructure เกือบทั้งหมด (เป็นหน้าที่ของ Microsoft Purview/AIP หรือเทียบเท่า) |

> **ข้อคิดสำคัญของ Step นี้**: การที่ 4 Pillar แรก (Identity/Device/Network/Application) แข็งแรงมาก
> ในหลักสูตรนี้ ไม่ได้แปลว่า Zero Trust "เสร็จสมบูรณ์" — Data Pillar ที่ยังอ่อนคือเหตุผลที่ Step 938
> (Maturity Roadmap) ต้องระบุมันเป็น Priority ชัดเจน ไม่ใช่มองข้ามเพราะ "ไม่ใช่งาน Network"

---

## Step 933 — ZTNA vs Traditional VPN: ทำไม Per-Application Access ถึงมาแทน "ต่อ VPN แล้วเท่ากับอยู่ในเครือข่าย"

### ปัญหาพื้นฐานของ Traditional Remote-Access VPN ในมุม Zero Trust

[Part 51](part-051-vrf-lite-gre-ipsec.md) สอน IPsec VPN เชิงลึก (IKEv1/IKEv2, Crypto Map, VTI,
FlexVPN) เพื่อปกป้อง Site-to-Site Tunnel ระหว่าง WAN-EDGE-1 กับ BRANCH-RTR — โมเดลเดียวกันนี้
(และ FlexVPN/AnyConnect ในโลกจริง) ก็ใช้เป็น **Remote-Access VPN** ให้พนักงานที่ทำงานนอกออฟฟิศ
เชื่อมต่อกลับเข้ามาได้เช่นกัน แต่ไม่ว่าจะเป็น Site-to-Site หรือ Remote-Access ทั้งสองแบบมี
**สมมติฐานเดียวกันที่ขัดกับ Zero Trust โดยตรง**:

> **"เมื่อ Authenticate ผ่าน VPN สำเร็จ (IKE Phase 1 + Phase 2 เสร็จ) Client จะได้รับ Virtual IP
> Address ที่อยู่ใน Subnet ภายในองค์กร — จากจุดนั้นเป็นต้นไป Client ถูกมองว่า 'อยู่ในเครือข่าย'
> เหมือนเสียบสาย LAN ที่ Office จริง (Network-Level Access)"**

ปัญหาคือ **"อยู่ในเครือข่าย" ไม่เท่ากับ "ควรเข้าถึงได้ทุกอย่างในเครือข่าย"** — นี่คือช่องโหว่ที่
Zero Trust ต้องการปิด

```
Traditional VPN Model (Part 51) — Network-Level Access

Remote User ──IKEv2/IPsec Tunnel──► WAN-EDGE-1 ──► ได้ Virtual IP (เช่น 10.10.60.15)
                                                          │
                                          ตั้งแต่จุดนี้ Client "เห็น" ทั้ง Internal Network
                                          เหมือนเสียบสาย LAN จริงที่ Office — ถูกจำกัดแค่ผ่าน
                                          ACL/Firewall Rule แยกอีกชั้น (ถ้ามีคนตั้งใจทำ) แต่
                                          Default คือเข้าถึงได้กว้างกว่าที่จำเป็นเสมอ
                                                          │
                        ┌─────────────────────────────────┼─────────────────────────────────┐
                        ▼                                 ▼                                 ▼
                  App ที่ต้องใช้จริง                  App อื่นที่ไม่เกี่ยวข้อง              Server ทุกตัวใน
                  (เช่น Internal Wiki)                (เช่น HR System)                    Subnet เดียวกัน
                  ✅ ควรเข้าถึงได้                     ❌ ไม่ควรเข้าถึงได้ แต่                ❌ Attack Surface ที่
                                                       "เห็น" อยู่ใน Network เดียวกัน          ไม่จำเป็นต้องเปิด
```

### ZTNA (Zero Trust Network Access) แก้ปัญหานี้อย่างไร

**ZTNA** เปลี่ยนหน่วยของการอนุญาต (Unit of Trust) จาก **"Network"** ไปเป็น **"Application"**
แต่ละครั้ง — Client **ไม่ได้รับ IP Address ที่อยู่ใน Internal Network เลย** สิ่งที่เกิดขึ้นจริงคือ
ZTNA Broker/Gateway ทำหน้าที่เป็น **Reverse Proxy ที่รู้จัก Identity** — สร้าง Connection แยกไปยัง
Application ปลายทางทีละตัวตาม Policy เฉพาะของ User คนนั้น

```
ZTNA Model — Per-Application Access

Remote User ──Identity-Aware Connection──► ZTNA Broker/Gateway (Cloud หรือ On-prem)
                                                          │
                                    ตรวจสอบ Identity + Device Posture + Policy
                                    ก่อนเปิด Connection ไปยัง App แต่ละตัวแยกกัน
                                                          │
                        ┌─────────────────────────────────┼─────────────────────────────────┐
                        ▼                                 ▼                                 ▼
                  App ที่ Policy อนุญาต             App ที่ Policy ไม่อนุญาต            Server อื่นทั้งหมด
                  (เช่น Internal Wiki)               (เช่น HR System)                  ในเครือข่าย
                  ✅ Broker เปิด Micro-Tunnel          ❌ Broker ไม่เปิด Connection         ❌ "มองไม่เห็น" เลย
                  ไปยัง App นี้เท่านั้น                  ไปยัง App นี้ตั้งแต่แรก              (ไม่ได้อยู่ใน Route
                                                                                            Table ของ Client)
```

### ตารางเปรียบเทียบตรงประเด็นที่ข้อสอบ CCIE Enterprise/Security ชอบถาม

| มิติ | Traditional VPN (Part 51) | ZTNA |
|---|---|---|
| **Unit of Trust** | Network (Subnet/VLAN ที่ Virtual IP อยู่) | Application แต่ละตัว (Per-App Session) |
| **หลัง Authenticate สำเร็จ** | ได้ IP ใน Internal Network — "เห็น" Network Topology ทั้งหมด (แม้ ACL จะกันไว้บางส่วน) | ไม่ได้ IP ใน Internal Network เลย — เห็นแค่รายชื่อ App ที่ Policy อนุญาตให้เห็น |
| **Attack Surface ถ้า Credential ถูกขโมย** | สูง — Attacker เห็น Network Topology ทั้งหมด สแกนหา Target อื่นได้ (Lateral Movement ง่ายกว่า) | ต่ำกว่ามาก — Attacker เห็นแค่ App ที่ User นั้นมีสิทธิ์อยู่แล้ว ไม่มี "Network" ให้สแกนต่อ |
| **Device Posture Check** | ส่วนใหญ่ Check แค่ตอน Connect ครั้งแรก (ตอน IKE/SSL Handshake) | Check ต่อเนื่องตลอด Session (เชื่อมกับ Step 935) |
| **Application ใหม่เพิ่มเข้ามา** | ต้องแก้ ACL/Firewall Rule ที่ VPN Concentrator/Firewall ทุกครั้ง | เพิ่ม Policy ที่ Broker ตัวเดียว — ไม่ต้องแก้ Network Config |
| **Scale สำหรับ Remote Workforce ขนาดใหญ่** | VPN Concentrator กลายเป็น Bottleneck (ต้อง Backhaul ทุก Traffic ผ่านจุดเดียว) | Broker เป็น Cloud-delivered ส่วนใหญ่ — Scale ตาม Demand โดยไม่ต้อง Capacity Plan Hardware |

### ZTNA ไม่ได้ "แทนที่" IPsec VPN เสมอไป — จุดที่ต้องแยกให้ถูก

> **ข้อสอบชอบดักจุดนี้**: ZTNA แทนที่ **Remote-Access VPN สำหรับ User เข้าถึง Application** เป็น
> หลัก — แต่ **Site-to-Site VPN** ที่ [Part 51 Step 507-508](part-051-vrf-lite-gre-ipsec.md) สอน
> (WAN-EDGE-1 ↔ BRANCH-RTR ผ่าน GRE-over-IPsec/VTI) ยังจำเป็นอยู่ เพราะมันปกป้อง **Infrastructure
> Traffic ระหว่าง Site** (Routing Protocol, Replication, Backup) ที่ไม่ใช่ "User เข้าถึง App เดียว"
> แบบที่ ZTNA ออกแบบมา — Zero Trust ไม่ได้บอกว่า "อย่าใช้ VPN" มันบอกว่า **"อย่าให้ VPN เป็น
> Checkpoint เดียวที่ตัดสินทุกอย่างหลังจากนั้น"**

### สถานะ ZTNA ในหลักสูตรนี้และ Lab

Lab ของหลักสูตรนี้ยังไม่ได้ Deploy ZTNA Broker จริง (เช่น Cisco Secure Access หรือ Duo Network
Gateway) เพราะเป็น SaaS Product แยกจาก On-prem Infrastructure ที่ Part 1-93 สร้างมา — แต่หลักการ
**"Per-Application, Identity-Verified Access"** ที่ ZTNA ใช้กับ Remote User ก็คือหลักการเดียวกัน
เป๊ะๆ กับที่ **TrustSec/SGACL** (Part 44) ใช้กับ Traffic ภายใน Campus — Step 934 จะพิสูจน์ความ
เชื่อมโยงนี้อย่างเป็นทางการ

---

## Step 934 — Micro-Segmentation คือ Network Pillar ของ Zero Trust: ทวน TrustSec/SGT ให้เป็นภาษา ZT อย่างเป็นทางการ

### ทำไม Step นี้ต้อง "ทวน" TrustSec ทั้งที่สอนไปแล้วใน Part 44 และ 77-78

Part นี้ไม่ได้สอน TrustSec ใหม่ — สิ่งที่ Step นี้ทำคือ **แปลภาษา**: เอา Feature ที่สอนไปแล้วด้วย
คำศัพท์ "Network Engineering" (SGT, SGACL, SXP, Inline Tagging) มาอธิบายใหม่ด้วยคำศัพท์
"Zero Trust Architecture" (Micro-Segmentation, Least Privilege, Policy Enforcement Point) —
เหตุผลที่ต้องทำแบบนี้คือ **ข้อสอบ CCIE และการสนทนากับผู้บริหารระดับ Security Architect มักใช้
ศัพท์ ZT ไม่ใช่ศัพท์ TrustSec โดยตรง** — วิศวกรที่ตอบได้แค่ "SGACL คือ ACL ที่ Match ด้วย SGT"
แต่อธิบายไม่ได้ว่ามันคือ **Micro-Segmentation** จะดูไม่มีมุมมองระดับ Architecture

### Micro-Segmentation คืออะไร — นิยามที่ต้องแม่น

**Micro-Segmentation** คือการแบ่งเครือข่ายออกเป็นส่วนเล็กที่สุดที่จำเป็น (ไม่ใช่แค่ VLAN/Subnet
ระดับ Department) แล้วบังคับ Policy การเข้าถึงระหว่างแต่ละส่วนอย่างเข้มงวด — เป้าหมายคือ **จำกัด
"Blast Radius"**: ถ้า Endpoint ตัวหนึ่งถูก Compromise มันจะแพร่กระจาย (Lateral Movement) ไปยัง
Endpoint อื่นได้ **น้อยที่สุดเท่าที่จะทำได้**

### ตารางแปลศัพท์ TrustSec ↔ Zero Trust Micro-Segmentation

| ศัพท์ TrustSec (Part 44) | ศัพท์ Zero Trust ที่ตรงกัน | ความหมายที่เหมือนกัน |
|---|---|---|
| **SGT (Security Group Tag)** | **Micro-Segment Identity / Workload Identity** | ตัวตนของกลุ่มที่แยกออกจาก IP Address โดยสิ้นเชิง — endpoint ย้าย Location ไปไหน "ตัวตน" ก็ยังติดตัวไปด้วย |
| **SGACL (Security Group ACL)** | **Policy Enforcement Point (PEP)** | จุดที่ตัดสินใจ Allow/Deny จริงตาม Policy — ทำงานที่ระดับ Group-to-Group ไม่ใช่ Network-to-Network |
| **802.1X + ISE (Dynamic SGT Assignment)** | **Policy Decision Point (PDP)** | ระบบที่ "ตัดสินใจ" ว่าใครควรได้ Micro-Segment Identity อะไร — แยกจาก PEP อย่างชัดเจนตามหลัก NIST 800-207 |
| **SXP / Inline Tagging** | **Policy Distribution Mechanism** | วิธีกระจาย Identity/Policy Decision จากจุดที่ตัดสินใจ (PDP) ไปยังจุดที่ Enforce จริง (PEP) ทุกจุดในเครือข่าย |
| **"SGT เดินทางไปกับ Endpoint ไม่ผูกกับ VLAN"** ([Part 44 Step 437](part-044-dot1x-trustsec.md)) | **"Never Trust Based on Network Location Alone"** | นี่คือจุดที่ TrustSec **สาธิตหลักการ ZT ได้ตรงที่สุด** — Endpoint ย้าย VLAN/Subnet/Site ไปไหนก็ยังถูกจำกัดสิทธิ์เท่าเดิมตาม "ตัวตน" ไม่ใช่ตาม "ตำแหน่ง" |

### NIST 800-207 Model: PDP/PEP กับ ISE/TrustSec ของหลักสูตรนี้

```
NIST SP 800-207 — Zero Trust Architecture Component Model

┌───────────────────────────────────────────────────────────────┐
│                     POLICY DECISION POINT (PDP)                     │
│           = ISE-1/ISE-2 (Part 43) — Policy Set + TrustSec Server     │
│   "ตัดสินใจว่า PC1 (employee01) ควรได้ SGT=SALES_SGT (100)"           │
└──────────────────────────────┬────────────────────────────────┘
                        RADIUS Access-Accept + SGT Attribute
                                     │
┌──────────────────────────────▼────────────────────────────────┐
│                  POLICY ENFORCEMENT POINT (PEP)                     │
│    = ACCESS-SW1 (802.1X Port), DIST-SW3 (SGACL), FW-1 (ACP Match SGT) │
│    "บังคับใช้จริง" — Permit/Deny แต่ละ Packet ตาม Policy ที่ PDP ตัดสินใจ │
└───────────────────────────────────────────────────────────────┘

ข้อสังเกตสำคัญ: PDP (ISE) มีจุดเดียว แต่ PEP (จุด Enforce) มีอยู่ "ทุกที่" — ทุก Switch Port,
ทุก Distribution Switch, ทุก Firewall — นี่คือลักษณะเฉพาะของ Zero Trust ที่ต่างจาก Perimeter
Model ที่มี Checkpoint จุดเดียว (FW-1 เท่านั้น)
```

### พิสูจน์ด้วย Worked Example เดิมจาก Part 44/78 — อ่านใหม่ผ่านแว่น Zero Trust

ทวนสถานการณ์จาก [Part 78 Step 774](part-078-security-integration-ise-trustsec-ftd-umbrella.md):
PC1 (SALES_SGT=100) พยายามเข้าถึง Server1 (SERVERS_SGT=120) — คำอธิบายเดิมเน้น "SGACL ทำงาน
อย่างไร" ส่วนคำอธิบายแบบ Zero Trust เน้นคำถามต่างออกไป:

| คำถามแบบ Perimeter Model (ผิด) | คำถามแบบ Zero Trust (ถูก) |
|---|---|
| "PC1 อยู่ใน Subnet ไหน กี่ Hop ถึง Server1" | "PC1 (ในฐานะ SALES_SGT) **ได้รับอนุญาต** ให้คุยกับ Server1 (SERVERS_SGT) หรือไม่ ด้วย Protocol/Port อะไร" |
| "PC1 ผ่าน Firewall มาแล้ว = Trust" | "ทุก Hop ที่ PC1 Traffic ผ่าน (DIST-SW3, FW-1) ต้อง **ตรวจสอบ SGT ซ้ำอิสระจากกัน** — ไม่มี Hop ไหนได้รับสิทธิ์ 'เชื่อ Hop ก่อนหน้า' เฉยๆ" |

> **ข้อสรุปสำคัญที่สุดของ Step นี้**: หลักสูตรนี้ **ไม่ได้รอถึง Part 94 ถึงจะเริ่มทำ Micro-
> Segmentation** — [Part 44](part-044-dot1x-trustsec.md) สร้างมันไว้เต็มรูปแบบแล้วตั้งแต่ระดับ
> CCNP สิ่งที่ Part นี้ทำคือยกระดับให้มองเห็นว่า **สิ่งที่สร้างไปแล้วคือ "Network Pillar" ของ Zero
> Trust Architecture ตัวจริง** ไม่ใช่แค่ "ACL แบบพิเศษที่ใช้ SGT" — มุมมองนี้สำคัญมากตอนต้อง
> Design ระบบใหม่ เพราะจะรู้ว่าเมื่อไหร่ต้องขยาย SGACL Matrix (เมื่อ Micro-Segment ใหม่เกิดขึ้น)
> แทนที่จะคิดแค่ "เพิ่ม VLAN ใหม่"

---

## Step 935 — Continuous Verification & Adaptive Access: จาก "Verify ครั้งเดียว" สู่ "Verify ตลอดเวลา"

### ข้อจำกัดของ Authentication แบบ "จุดเดียว" ที่ Step ก่อนหน้ายังไม่ได้แก้เต็มที่

ทั้ง 802.1X (Part 44) และ ZTNA (Step 933) ยืนยันตัวตน **ตอนเริ่ม Session** ได้ดีมาก แต่มีคำถาม
ที่ยังไม่ถูกตอบ: **"ถ้า PC1 Authenticate ผ่านตอน 9:00 น. แล้วเกิดอะไรขึ้นที่ 9:00.05 (5 วินาที
หลังจากนั้น) — Session ที่ยังเปิดอยู่รู้เรื่องนี้ทันทีไหม"** — คำตอบสำหรับระบบที่ Verify แค่จุดเดียว
คือ **"ไม่รู้จนกว่าจะถึงรอบ Reauthentication ครั้งต่อไป"** ซึ่งอาจนานหลายชั่วโมง — นี่คือช่องโหว่ที่
**Continuous Verification** ต้องปิด

### Adaptive Access คืออะไร — นิยามและ Signal ที่ใช้ตัดสินใจ

**Continuous Verification / Adaptive Access** คือการประเมิน Trust ของ Session **ซ้ำตลอดเวลา**
โดยใช้ Signal ที่เปลี่ยนแปลงได้ระหว่าง Session ไม่ใช่แค่ Signal ตอน Login ครั้งแรก:

| Signal ที่ใช้ Re-evaluate | ตัวอย่างการเปลี่ยนแปลงที่ต้อง Trigger การประเมินใหม่ | ระบบที่สร้างไว้แล้วในหลักสูตรนี้ที่ให้ Signal นี้ |
|---|---|---|
| **Device Posture** | Endpoint ที่เคย Compliant กลายเป็น Non-Compliant กลางทาง (AV หมดอายุ, Patch ที่ต้องมีถูกลบ) | ISE Posture Re-assessment ([Part 43 Step 428](part-043-security-architecture-firepower-ise.md)) |
| **Network Telemetry / Behavior Anomaly** | Endpoint เริ่มส่ง Traffic Pattern แปลกไปจากเดิม (DNS Query จำนวนมากผิดปกติ, Data Exfiltration Pattern) | Model-Driven Telemetry + Anomaly Detection ([Part 54](part-054-network-assurance-telemetry.md), [Part 80 Step 796](part-080-model-driven-telemetry-streaming.md)) |
| **Threat Intelligence** | Endpoint เริ่มติดต่อ Domain/IP ที่ถูกขึ้น Blacklist ใหม่ (Talos Feed Update) | Umbrella/FTD Threat Intelligence ([Part 78 Step 772](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) |
| **Location/Time Anomaly** | User Login จากตำแหน่งที่ไม่สมเหตุสมผล (Login จาก 2 ประเทศห่างกันภายในไม่กี่นาที) | นอกขอบเขต Network Infra ของหลักสูตรนี้ — เป็นหน้าที่ของ Identity Provider (Azure AD/Okta) |

### เชื่อม Telemetry (Part 54/80) เข้ากับ Automated Response (Part 78) — Closed-Loop Adaptive Access

นี่คือจุดที่ Step นี้ **สังเคราะห์ 3 Part เข้าด้วยกันเป็นครั้งแรกในหลักสูตร**: Telemetry ที่ตรวจจับ
ความผิดปกติ (Part 54/80) ไม่ได้จบแค่ "แจ้งเตือน Dashboard" อีกต่อไป — มันต้องต่อเข้ากับ
Automated Response แบบเดียวกับ Rapid Threat Containment ที่ [Part 78 Step 776]
(part-078-security-integration-ise-trustsec-ftd-umbrella.md) สร้างไว้

```
Closed-Loop Continuous Verification — สังเคราะห์ Part 54/80 + Part 78

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  1. TELEMETRY       │     │  2. ANOMALY          │     │  3. RISK RE-SCORE    │     │  4. ADAPTIVE         │
│  MDT/gNMI Stream    │────►│  DETECTION           │────►│  (ISE/SecureX)       │────►│  ENFORCEMENT         │
│  จากทุกอุปกรณ์        │     │  Baseline Deviation   │     │  "Trust Score ของ    │     │  CoA เปลี่ยน SGT      │
│  (Part 54, 80)      │     │  (Part 54 Step 537,  │     │   Session นี้ลดลง"    │     │  ทันที (Part 44,     │
│                     │     │   Part 80 Step 796)   │     │                     │     │  Part 78 Step 776)   │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
      ตัวอย่าง Signal:              ตัวอย่างการตัดสินใจ:            ผลลัพธ์ที่เป็นไปได้:
      Interface Flap ผิดปกติ,       "Endpoint นี้ส่ง DNS Query       - ลดสิทธิ์ (Step-down SGT)
      DNS Query Volume พุ่งขึ้น,     ไป Domain สุ่มจำนวนมาก           - บังคับ Re-authenticate
      Connection Pattern ที่          ผิดปกติจาก Baseline ปกติ         - Quarantine เต็มรูปแบบ
      ไม่ตรง Baseline                 มากกว่า 3 Standard Deviation"    (SGT=999, Part 78 Step 776)
```

### ตารางเปรียบเทียบ: One-Time Authentication vs Continuous Verification

| มิติ | One-Time Authentication (802.1X พื้นฐาน — Part 44) | Continuous Verification (Step นี้) |
|---|---|---|
| **ความถี่ในการตรวจสอบ Trust** | ครั้งเดียวตอนเริ่ม Session (หรือรอบ Reauthentication ที่ตั้งไว้ล่วงหน้า เช่นทุก 8 ชั่วโมง) | ต่อเนื่อง — ทุกครั้งที่มี Signal ใหม่เข้ามา (Posture Change, Anomaly, Threat Intel Update) |
| **จุดที่ Trigger การ Re-evaluate** | เวลาที่ตั้งไว้ล่วงหน้าเท่านั้น (Static Timer) | Event-driven — เกิดขึ้นทันทีที่ Signal เปลี่ยน (Dynamic) |
| **Mechanism ที่ใช้เปลี่ยน Authorization กลางทาง** | ไม่มี — ต้องรอ Session หมดอายุ | **RADIUS CoA** (Change of Authorization — [Part 37 Step 367](part-037-wireless-security-advanced.md), ใช้จริงใน [Part 78 Step 776](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) |
| **ตัวอย่าง Use Case** | PC1 เสียบสาย → Authenticate → ได้ SALES_SGT ตลอด 8 ชั่วโมง | PC1 Authenticate ปกติ → 2 ชั่วโมงต่อมา AMP ตรวจพบ Malware → CoA เปลี่ยนเป็น QUARANTINE_SGT ทันทีโดยไม่ต้องรอ 8 ชั่วโมง |

> **หลักการที่ต้องจำ**: Continuous Verification **ไม่ใช่ Technology ใหม่ตัวหนึ่ง** — มันคือการเอา
> 3 ชิ้นที่มีอยู่แล้ว (Telemetry Pipeline จาก Part 54/80, Policy Engine จาก ISE Part 43, และ
> CoA Mechanism จาก Part 37/78) มา **ต่อเป็น Loop ปิด (Closed Loop)** ที่ไม่ต้องมี Admin
> คอยเฝ้า Dashboard แล้วกดปุ่มเองทุกครั้ง — นี่คือ "Always Verify" ครึ่งหลังของหลักการ Zero Trust
> ที่ Step 931 นิยามไว้ ส่วน "Never Trust" ครึ่งแรกคือ Micro-Segmentation จาก Step 934

---

## Step 936 — Zero Trust บน Campus/LAN: 802.1X + TrustSec คือ ZT ระดับ Wired ที่มีอยู่แล้ว

### ความเข้าใจผิดที่พบบ่อยที่สุดเรื่อง Zero Trust

บทความ/Vendor ส่วนใหญ่พูดถึง Zero Trust ในบริบท **Remote Access และ Cloud** เกือบทั้งหมด
(ZTNA แทน VPN, CASB คุม SaaS) จนวิศวกร Campus Network จำนวนมากเข้าใจผิดว่า **"Zero Trust
เป็นเรื่องของทีม Cloud/Security ไม่เกี่ยวกับงาน Campus LAN ที่เราทำอยู่"** — ความเข้าใจนี้ผิดโดย
สิ้นเชิง และ Step นี้จะพิสูจน์ว่า **Campus LAN ที่ทำ 802.1X + TrustSec ถูกต้องตามที่ Part 44 สอน
คือ Zero Trust Implementation ที่สมบูรณ์กว่า Remote-Access-only ZTNA เสียอีก** เพราะมันครอบคลุม
ทั้ง Identity, Device, และ Network Pillar พร้อมกันในจุดเดียว

### เทียบ 3 Zero Trust Tenet กับ Feature ที่ Wired Campus มีอยู่แล้ว

| Zero Trust Tenet | Feature บน Campus LAN (Part 44) | รายละเอียด |
|---|---|---|
| **Verify Explicitly ก่อนให้ Traffic ผ่านแม้แต่ Packet เดียว** | 802.1X Port State = **Unauthorized by Default** ([Part 44 Step 432](part-044-dot1x-trustsec.md)) | พอร์ตบล็อกทุกอย่างยกเว้น EAPOL ตั้งแต่ก่อนมี DHCP Discover ผ่านได้เลย — นี่คือ "Deny by Default" ที่เข้มกว่า Zero Trust ระดับ Cloud หลายระบบเสียอีก (Cloud ส่วนใหญ่ยัง Allow DNS/ICMP พื้นฐานก่อน Authenticate) |
| **Least Privilege ตาม "กลุ่ม" ไม่ใช่ตาม Subnet** | SGACL ที่ Permit เฉพาะ Protocol/Port ที่จำเป็นระหว่าง SGT ([Part 44 Step 438](part-044-dot1x-trustsec.md)) | PC1 (SALES_SGT) เข้าถึง Server1 (SERVERS_SGT) ได้เฉพาะ Port ที่ Business ต้องใช้จริง — ไม่ Allow ทุก Port เหมือน VLAN-based ACL แบบเดิม |
| **Continuous Verification ผ่าน CoA** | Session Re-authorization เมื่อ Posture เปลี่ยน หรือ ISE สั่ง Quarantine ([Part 78 Step 776](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) | ไม่ต้องรอ Client Disconnect/Reconnect — เปลี่ยน Trust Level กลาง Session ได้ทันที |

### เหตุผลเชิง Physical ที่ทำให้ Campus LAN ZT "แข็งแรง" กว่า Remote Access ZTNA ในบางมิติ

```
เปรียบเทียบ Attack Surface: Wired Campus Port vs Remote ZTNA Connection

Wired Campus (802.1X + TrustSec)              Remote ZTNA
┌─────────────────────────┐                  ┌─────────────────────────┐
│ Physical Access ต้องเข้า    │                  │ Attacker เข้าถึงได้จาก      │
│ ตึกก่อน (Badge/รปภ.)         │                  │ "ที่ไหนก็ได้บนโลก" ที่มี      │
│ + ต้อง Authenticate ผ่าน     │                  │ Internet — Layer ป้องกัน   │
│ 802.1X อีกชั้น (Defense    │                  │ ทางกายภาพไม่มีเลย            │
│ in Depth 2 ชั้น)             │                  │                          │
│                            │                  │ ต้องพึ่ง Device Posture +   │
│ 1 Port = 1 State Machine   │                  │ MFA + Continuous Verify     │
│ อิสระ ([Part 44 Step 431]  │                  │ อย่างเข้มข้นกว่ามาก เพราะ    │
│ (part-044-dot1x-trustsec.md))│                  │ ไม่มี Physical Layer ช่วย   │
└─────────────────────────┘                  └─────────────────────────┘

ข้อสรุป: Zero Trust ไม่ได้แปลว่า "ทุก Location มี Risk เท่ากัน" — Physical Security ของ Campus
ยังเป็น "ชั้นป้องกันเสริม" ที่ใช้ได้จริง แต่ Zero Trust บอกว่า **"ห้ามใช้ Physical Security แทน
Identity Verification โดยสิ้นเชิง"** — ต้องมีทั้งสองชั้นควบคู่กันเสมอ (ตามที่ [Part 44 Step 431]
(part-044-dot1x-trustsec.md) เคยตอบคำถาม "ทำไมยังต้องเรียน 802.1X ทั้งที่มี Physical Security อยู่แล้ว")
```

### เชื่อมกับ SD-Access (Part 38/77) — Fabric คือ ZT Campus ที่ Automate เต็มรูปแบบ

[Part 38](part-038-sd-access-fundamentals.md) และ [Part 77](part-077-sd-access-advanced-design.md)
สร้าง SD-Access Fabric ที่ใช้ VXLAN + LISP + TrustSec ร่วมกัน — ในกรอบคิด Zero Trust, SD-Access
Fabric คือ **"Automated Zero Trust Campus"**: ทุก Endpoint ที่เข้า Fabric ต้อง Authenticate ผ่าน
ISE ก่อนเสมอ (Identity Pillar), ได้ SGT ตาม Policy (Network Pillar), และ Policy นั้น **Consistent
ทุก Site ของ Fabric โดยอัตโนมัติ** ผ่าน Fabric Control Plane — ไม่ต้องไปตั้ง SGACL ทีละ Switch
เหมือน Part 44 สอนแบบ Manual

> **สรุปสำหรับ Step นี้**: อย่าปล่อยให้ใครบอกว่า "Zero Trust คือเรื่องของ Cloud/Remote Work
> เท่านั้น" — วิศวกร Campus/Enterprise ที่ทำ 802.1X + TrustSec ให้ถูกต้องตามมาตรฐานของ Part 44
> **ได้ Deploy Zero Trust Network Access ระดับ Wired LAN ไปแล้วอย่างสมบูรณ์** — สิ่งที่ต้องทำต่อ
> คือขยายกรอบคิดเดียวกันไปให้ครอบคลุม WAN/Branch (Step 937) และ Cloud/Application (Step 933,
> Part 93) เพื่อให้ Zero Trust เป็น "สถาปัตยกรรมเดียว" ไม่ใช่ "Zero Trust แค่บางจุด"

---

## Step 937 — Zero Trust บน WAN/Branch: SD-WAN Security + Cloud OnRamp for SaaS ในกรอบคิด ZT

### ปัญหาเดิมของ WAN แบบ Hub-and-Spoke ในมุม Zero Trust

WAN แบบเดิมที่ [Part 19](part-019-wan-technologies.md) และ [Part 39]
(part-039-sd-wan-fundamentals.md) เริ่มสอน มี Pattern เดียวกับ Castle-and-Moat ระดับ WAN:
**Branch Traffic ทุกอย่างต้อง Backhaul กลับ HQ ผ่าน Hub ก่อนออก Internet** — เหตุผลเดิมคือ
ต้องการให้ Traffic ผ่าน FW-1 ที่ HQ เสมอ (Perimeter เดียว) แต่วิธีนี้มีปัญหา 2 อย่างที่ขัดกับ
Zero Trust โดยตรง:

1. **Latency สูง** — SaaS Traffic (Office 365) ที่ปลายทางอยู่ Cloud อยู่แล้ว ต้องเดินทางไกลผ่าน
   HQ ก่อน แทนที่จะออกตรงจาก Branch
2. **สมมติฐานผิด** — "Backhaul ผ่าน HQ = ปลอดภัยกว่า" คือสมมติฐานแบบ Perimeter Model ที่ Step 931
   บอกไว้ว่าใช้ไม่ได้อีกต่อไป เพราะ FW-1 ที่ HQ ก็ไม่ใช่ Checkpoint ที่ "รู้จัก" User/Device ระดับ
   Identity เท่า ISE เลย

### Cloud OnRamp for SaaS + Umbrella Network Tunnel = Zero Trust ที่ WAN Edge

[Part 76 Step 757](part-076-sdwan-advanced-design.md) สอนไว้แล้วว่า SD-WAN cEdge ฝัง **Enterprise
Firewall, IPS, URL Filtering, และ Umbrella Integration** ไว้ในตัว ทำให้ Branch ทำ **Direct
Internet Access (DIA)** ได้โดยยังมี Security Inspection ระดับ Enterprise — นี่คือ Zero Trust
Principle "**Verify ที่ Edge ไม่ใช่รอ Backhaul ไปที่จุดกลางแล้วเชื่อ**" พูดในภาษา SD-WAN

```
เปรียบเทียบ WAN Model เดิม vs Zero Trust WAN Edge (Part 76 Step 757 + Part 78 Step 773)

WAN แบบเดิม (Hub-and-Spoke Backhaul)          Zero Trust WAN Edge (Cloud OnRamp + Umbrella)

Branch ──Backhaul──► HQ ──► FW-1 ──► Internet   Branch (cEdge) ──┬──DIA──► SaaS App โดยตรง
   (Traffic ทุกอย่างผ่าน HQ ก่อนเสมอ                                     │        (เร็วที่สุด)
    ไม่ว่าปลายทางจะเป็น Internal                                        └──IPsec Tunnel──►
    หรือ Internet ก็ตาม — สมมติฐาน                                          Umbrella Cloud
    "HQ = จุดเดียวที่ Trust ได้")                                          (Verify ก่อนออกจริง
                                                                          ทุกครั้ง — Step 772-773)

ผลลัพธ์: Latency ต่ำ + Security ไม่ลดลง        ผลลัพธ์: Latency ต่ำ + Security Inspection
(เพราะ Trust จุดเดียวที่ HQ)                    เกิดที่ Branch Edge ทุกครั้ง ไม่ต้องพึ่ง HQ เลย
```

### แมป Feature จาก Part 76/78 เข้ากับ ZT Pillar อย่างเป็นระบบ

| Feature ที่ WAN Edge | ZT Pillar ที่ตอบโจทย์ | อ้างอิง |
|---|---|---|
| **Enterprise Firewall (Zone-based) บน cEdge** | Application/Workload | [Part 76 Step 757](part-076-sdwan-advanced-design.md) |
| **IPS (Cisco Talos Signature) บน cEdge** | Application/Workload (Threat Detection) | [Part 76 Step 757](part-076-sdwan-advanced-design.md) |
| **Umbrella DNS-layer + Network Tunnel** | Network/Environment (Verify ก่อนออก Internet จริง) | [Part 78 Step 773](part-078-security-integration-ise-trustsec-ftd-umbrella.md) |
| **VPN Segment (VPN 10/20/30/40) ที่ Consistent ทุก Site** | Network/Environment (Micro-Segmentation ระดับ WAN) | [Part 39 Step 387](part-039-sd-wan-fundamentals.md), [Part 76 Step 758](part-076-sdwan-advanced-design.md) |
| **vManage Centralized Policy Push** | Cross-cutting: Governance/Automation | [Part 76](part-076-sdwan-advanced-design.md) |

### เมื่อไหร่ Branch ยังต้อง Backhaul กลับ HQ — ข้อยกเว้นที่ต้องรู้

[Part 76 Step 757](part-076-sdwan-advanced-design.md) ระบุไว้แล้วว่า Traffic บางประเภท (เช่น
POS/Cardholder Data ตาม PCI-DSS) ยังควร Backhaul ผ่าน Centralized Firewall ที่ HQ เพิ่มอีกชั้น
— ในมุม Zero Trust นี่**ไม่ขัดกัน**เลย เพราะ Zero Trust ไม่ได้บอกว่า "ห้าม Centralize อะไรทั้งสิ้น"
มันบอกว่า **"อย่า Centralize เพราะ Default/ความเคยชิน ให้ Centralize เพราะ Policy/Compliance
Requirement ที่ชัดเจน"** — POS Traffic ที่ต้อง PCI-DSS Compliance คือเหตุผลที่ชัดเจนพอ ส่วน
Traffic ทั่วไป (Web Browsing, SaaS ปกติ) ไม่มีเหตุผลอะไรให้ Backhaul อีกต่อไป

> **ข้อสรุปสำหรับ Step นี้**: WAN/Branch Zero Trust ไม่ใช่ Technology ใหม่ที่ต้องสร้างเพิ่ม — มัน
> คือการอ่าน [Part 76 Step 757](part-076-sdwan-advanced-design.md) และ [Part 78 Step 773]
> (part-078-security-integration-ise-trustsec-ftd-umbrella.md) ใหม่ผ่านแว่น **"Verify at the
> Edge, Not at a Central Choke Point"** — Direct Internet Access ที่เร็วขึ้นและ Umbrella Network
> Tunnel ที่ตรวจสอบก่อนออกจริงทุกครั้ง คือสองด้านของ Zero Trust WAN เดียวกัน

---

## Step 938 — Zero Trust Maturity Roadmap: แนวทาง Implement แบบ Incremental ไม่ใช่ Big Bang

### ทำไม "Big Bang Migration" ไปสู่ Zero Trust ไม่ใช่ทางเลือกที่ทำได้จริง

องค์กรขนาดใหญ่ไม่มีทางปิด Network ทั้งหมดแล้ว Deploy Zero Trust ใหม่หมดในคืนเดียว — Endpoint
เก่าที่ไม่รองรับ 802.1X Supplicant, Application Legacy ที่ไม่รองรับ Modern Authentication,
และ Business ที่ต้องดำเนินต่อเนื่อง ทำให้ **Migration แบบ Incremental (ทีละขั้น)** คือทางเลือก
เดียวที่ใช้ได้จริง — Step นี้ใช้ **CISA Zero Trust Maturity Model (ZTMM)** เป็นกรอบวัดระดับ
(4 ระดับ: Traditional → Initial → Advanced → Optimal) มา Apply กับ Enterprise Lab ของหลักสูตรนี้

### 4 ระดับ Maturity ตาม CISA ZTMM (สรุปสำหรับใช้งานจริง)

| ระดับ | ลักษณะ | ตัวอย่าง |
|---|---|---|
| **Traditional** | Manual Config, Static Policy, Perimeter-based Trust | VLAN-based Access Control, IP-based ACL ล้วนๆ |
| **Initial** | เริ่มมี Automation บางส่วน, Policy เริ่มผูกกับ Attribute (ไม่ใช่แค่ IP) | Dynamic VLAN Assignment ([Part 37 Step 366](part-037-wireless-security-advanced.md)) |
| **Advanced** | Policy ผูกกับ Identity/Group ทั่วทั้ง Domain, Automated Response เริ่มมี | TrustSec/SGT ครบทั้ง Enterprise ([Part 44](part-044-dot1x-trustsec.md)), Rapid Threat Containment ([Part 78 Step 776](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) |
| **Optimal** | Continuous Risk Scoring แบบ Real-time ทุก Session, Policy เปลี่ยนอัตโนมัติตาม Risk ที่คำนวณสด, ครอบคลุมทุก Pillar รวม Data | Closed-Loop Adaptive Access เต็มรูปแบบ (Step 935) ที่ครอบคลุมทุก Pillar |

### 4 Phase Roadmap ที่ต่อยอดจาก "สิ่งที่ Lab นี้มีอยู่แล้ว" — ไม่ใช่เริ่มจากศูนย์

```
PHASE 0 (มีอยู่แล้ว — Part 1-93)          PHASE 1 (Quick Win)         PHASE 2 (Structural)        PHASE 3 (Optimal)
┌─────────────────────┐              ┌─────────────────┐         ┌─────────────────┐        ┌─────────────────┐
│ Identity: AAA/802.1X │              │ ขยาย TrustSec      │         │ ผูก Telemetry       │        │ Data Pillar เต็ม  │
│ (Part 22/37/43)       │──────────────►│ Matrix ให้ครอบคลุม  │─────────►│ (Part 54/80) เข้า   │────────►│ รูปแบบ (DLP,      │
│                       │              │ ทุก Segment ที่      │         │ Loop CoA อัตโนมัติ   │        │ Encryption at Rest,│
│ Network: TrustSec     │              │ ยังเป็น IP-based    │         │ (Step 935)          │        │ Rights Mgmt)       │
│ (Part 44/77)          │              │ ACL อยู่             │         │                     │        │                     │
│                       │              │                    │         │ ขยาย ZTNA ให้ครอบคลุม│        │ Continuous Risk    │
│ Application: FTD +    │              │ Onboard ZTNA        │         │ Remote User ทั้งหมด   │        │ Score รวมทุก Signal │
│ pxGrid (Part 43/78)   │              │ Broker สำหรับ        │         │ (แทน Traditional VPN │        │ (Device+Network+   │
│                       │              │ Remote Access ใหม่   │         │ ทั้งหมด)              │        │ Behavior+Location)  │
│ Automated Response:   │              │ (Step 933)          │         │                     │        │ = Optimal ตาม CISA  │
│ RTC (Part 78 Step 776)│              │                    │         │                     │        │                     │
└─────────────────────┘              └─────────────────┘         └─────────────────┘        └─────────────────┘
   ระดับ: Advanced (บางส่วน)             เป้าหมาย: Advanced เต็ม        เป้าหมาย: Advanced→Optimal    เป้าหมาย: Optimal
```

### ตาราง Roadmap พร้อม Priority — ใช้เป็น Template สำหรับงานจริง

| Priority | งาน | Pillar ที่ได้ประโยชน์ | ต่อยอดจาก Part | ผลลัพธ์ที่วัดได้ |
|---|---|---|---|---|
| **P1 (ทำก่อน — ต้นทุนต่ำ ผลตอบแทนสูง)** | ขยาย TrustSec/SGACL Matrix ให้ครอบคลุมทุก Segment ที่ยังพึ่ง IP-based ACL อยู่ | Network | Part 44 | ลด IP-based ACL ที่ต้องแก้ทุกครั้งที่ Endpoint ย้าย Location |
| **P1** | เปิด ISE Posture Assessment ให้ครอบคลุม Endpoint 100% (ไม่ใช่แค่ Pilot Group) | Device | Part 43 | Non-Compliant Device ถูกจับได้ก่อนเข้าเครือข่ายจริง ไม่ใช่แค่ Endpoint ที่ IT เลือกมา Test |
| **P2 (ต้องมี Structural Change)** | เชื่อม Telemetry Anomaly Detection (Part 54/80) เข้า Loop CoA อัตโนมัติแบบเดียวกับ RTC | Cross-cutting (Automation) | Part 54, 78, 80 | ลดเวลา Mean-Time-to-Contain จากนาที/ชั่วโมง เหลือวินาที |
| **P2** | Deploy ZTNA Broker สำหรับ Remote Access ใหม่ทั้งหมด แทนที่ Traditional Remote-Access VPN | Identity, Network | Part 51 | Attack Surface ของ Remote User ลดลง — ไม่มี "Network-level Access" อีกต่อไป |
| **P3 (ระยะยาว — ต้องมี Team/Tool เพิ่ม)** | เริ่ม Data Classification Program และ Encryption at Rest สำหรับ Data ที่ SERVERS_SGT ครอบคลุม | Data | (ใหม่ — Step 932) | Pillar ที่อ่อนที่สุดของ Stack เริ่มมี Coverage จริง |
| **P3** | สร้าง Continuous Risk Score รวม Signal จากทุก Pillar (ไม่ใช่แค่ Posture Binary) | ทุก Pillar | Step 935 | เข้าสู่ระดับ Optimal ตาม CISA ZTMM |

### ข้อผิดพลาดที่พบบ่อยที่สุดตอนพยายาม Implement Zero Trust ในงานจริง

| ข้อผิดพลาด | ทำไมผิด | วิธีแก้ที่ถูก |
|---|---|---|
| พยายามซื้อ "Zero Trust Platform" ตัวเดียวมาแทนทุกอย่าง | Zero Trust ไม่ใช่ Product เดียว — เป็นกรอบคิดที่ต้องใช้หลาย Component ร่วมกัน (ดู Step 932) | เริ่มจาก Pillar ที่มีพื้นฐานอยู่แล้ว (Network/Identity จาก Part 44) แล้วขยายทีละ Pillar |
| เริ่มจาก Pillar ที่ยากที่สุด (Data) ก่อน | ต้นทุนสูง ผลตอบแทนช้า ทำให้ Momentum ของ Project หายไปก่อนเห็นผล | เริ่มจาก P1 (Network/Device) ที่มีพื้นฐานแล้วให้เห็นผลเร็ว แล้วค่อยขยายไป Data |
| คิดว่า Zero Trust = "ลบ VPN ทิ้งทั้งหมด" | ทำให้ Business Traffic บางประเภท (Site-to-Site Infrastructure) เสียหาย | แยก Use Case ให้ถูกตาม Step 933 — ZTNA แทน Remote-Access User, VPN แบบเดิมยังจำเป็นสำหรับ Infrastructure Traffic |

---

## Step 939 — Zero Trust และ Compliance/Audit: ทำไม ZT คือ "ผลพลอยได้" ของ Compliance ที่ดี

### Zero Trust ไม่ได้ถูกออกแบบมาเพื่อ Compliance โดยตรง — แต่ผลลัพธ์ตรงกันโดยบังเอิญ (ไม่บังเอิญ)

[Part 78 Step 779](part-078-security-integration-ise-trustsec-ftd-umbrella.md) แมป Feature ของ
Security Stack เข้ากับ PCI-DSS Requirement ไว้แล้ว (Requirement 1/7/8/10) — Step นี้จะอธิบาย
**เหตุผลเชิงโครงสร้าง** ว่าทำไม Zero Trust Architecture ถึง "ตอบโจทย์ Compliance ได้เกือบทุก
Framework โดยธรรมชาติ" ไม่ใช่เรื่องบังเอิญ:

> Compliance Framework ส่วนใหญ่ (PCI-DSS, ISO 27001, HIPAA, SOC 2) ถูกออกแบบมาเพื่อตอบคำถาม
> พื้นฐานเดียวกัน: **"ใครเข้าถึงข้อมูลอะไร เมื่อไหร่ และพิสูจน์ได้อย่างไรว่าการเข้าถึงนั้นได้รับ
> อนุญาตจริง"** — คำถามนี้คือคำถามเดียวกันเป๊ะๆ กับที่ Zero Trust Verify Explicitly ตอบอยู่ทุก
> Request

### Continuous Verification ให้ Audit Evidence ที่ดีกว่า Periodic Review อย่างไร

Audit แบบเดิม (Periodic Point-in-Time Review) มีข้อจำกัดที่ Zero Trust แก้ได้โดยตรง:

| มิติ | Periodic Point-in-Time Audit (แบบเดิม) | Continuous Verification (Zero Trust — Step 935) |
|---|---|---|
| **ความถี่ของหลักฐาน** | สุ่มตรวจ 1-2 ครั้งต่อปี (ตาม Audit Cycle) | ทุก Access Request มี Log ทันที (Real-time) |
| **ช่องว่างที่ Auditor มองไม่เห็น** | สิ่งที่เกิดขึ้น "ระหว่าง" รอบ Audit ไม่มีใครรู้ (เช่น Access ที่ผิด Policy ตอนกลางคืนแล้วแก้คืนก่อน Audit มาถึง) | ไม่มีช่องว่าง — ทุก Session ถูกประเมินและ Log ตลอดเวลา |
| **หลักฐานที่ Auditor เชื่อได้แค่ไหน** | เชื่อ Snapshot ที่องค์กรเตรียมไว้ให้ดู (เสี่ยงถูกจัดฉาก) | Cross-validate จากหลายระบบพร้อมกัน (ISE MnT + `show cts role-based counters` + FMC Connection Events — [Part 78 Step 779](part-078-security-integration-ise-trustsec-ftd-umbrella.md)) |

### แมป Zero Trust Pillar เข้ากับ Compliance Framework หลักแบบกว้าง (ไม่ใช่แค่ PCI-DSS)

| ZT Pillar | PCI-DSS | ISO 27001 | HIPAA |
|---|---|---|---|
| **Identity** | Req. 8 (Authenticate ทุกคน) | A.9 (Access Control) | §164.312(d) (Person/Entity Authentication) |
| **Network/Micro-Segmentation** | Req. 1 (Network Segmentation ของ CDE) | A.13 (Network Security Management) | §164.312(a) (Access Control — Technical Safeguards) |
| **Continuous Monitoring** | Req. 10 (Log/Audit Trail) | A.12.4 (Logging and Monitoring) | §164.312(b) (Audit Controls) |
| **Data** | Req. 3-4 (Protect Cardholder Data) | A.8 (Asset Management), A.10 (Cryptography) | §164.312(e) (Transmission Security) |

> **ข้อคิดสำคัญของ Step นี้**: องค์กรที่มองว่า Compliance คือ "ภาระที่ต้องทำเพิ่มต่างหากจาก
> Security" มักจบลงด้วยการทำ Documentation ซ้อนทับ Security Control ที่มีอยู่แล้ว 2 ชุดที่ไม่ตรงกัน
> — องค์กรที่ Design Zero Trust Architecture ให้ถูกตั้งแต่แรก (ตามที่ Part 22-94 สอนมาตลอด) จะพบว่า
> **Compliance Evidence คือผลพลอยได้ที่ระบบสร้างให้อัตโนมัติอยู่แล้ว** ไม่ต้องมี Process แยกมา
> เก็บ Evidence เพิ่มเติมเลย — นี่คือเหตุผลที่ CISO ระดับ Enterprise ผลักดัน Zero Trust ไม่ใช่แค่
> เพราะ "ปลอดภัยกว่า" แต่เพราะ **"ลดต้นทุนการพิสูจน์ Compliance ในระยะยาว"** ด้วย

---

## Step 940 — Lab: Zero Trust Maturity Assessment & Roadmap สำหรับ Enterprise Lab ทั้งหมด (Part 1-93)

> **หมายเหตุสำคัญ**: เหมือนกับ [Part 78 Step 780](part-078-security-integration-ise-trustsec-ftd-umbrella.md)
> Lab นี้คือ **เอกสารสถาปัตยกรรมเชิงแนวคิด (Architectural Assessment Document)** ไม่ใช่ Config
> ใหม่ — เป้าหมายคือสังเคราะห์ **ทุก Part ที่เกี่ยวกับ Security ตลอดหลักสูตร 93 Part** เข้าเป็น
> เอกสารประเมิน Zero Trust Maturity ฉบับเดียวสำหรับ Enterprise Lab ทั้งก้อน — แบบเดียวกับที่
> Security Architect ระดับ CCIE ต้องส่งให้ผู้บริหารจริงในงาน

### ส่วนที่ 1: Current-State Assessment แบบ Pillar-by-Pillar

| Pillar | Component ที่มีอยู่ | สร้างไว้ที่ | ระดับ Maturity (CISA ZTMM) | หลักฐาน (Evidence) |
|---|---|---|---|---|
| **Identity** | ISE-1/ISE-2 เป็น Policy Engine กลาง, 802.1X (Wired+Wireless), TACACS+ สำหรับ Device Admin | Part 22, 37, 43, 44 | **Advanced** | `show authentication sessions` ทุก Access Port แสดง Status=Authorized พร้อม User-Name จริง ไม่ใช่แค่ Port Security แบบ MAC |
| **Device** | ISE Posture Assessment, CoA Re-authorization เมื่อสถานะเปลี่ยน | Part 43 Step 428 | **Advanced** (Binary Compliant/Non-Compliant — ยังไม่ใช่ Risk Score ต่อเนื่อง) | ISE Live Log แสดง Posture Status ต่อ Session |
| **Network/Environment** | TrustSec/SGT ครบ Enterprise, SGACL Enforcement ที่ DIST-SW3, SXP ระหว่าง Site, SD-Access Fabric | Part 44, 77 | **Advanced** — Pillar ที่แข็งแรงที่สุดของ Stack | `show cts role-based counters` แสดง HW-Permitted/Denied จริงตาม SGT Matrix |
| **Application/Workload** | FTD ACP Match ด้วย SGT ผ่าน pxGrid, SD-WAN On-box Security ที่ Branch Edge | Part 43, 76 Step 757, 78 Step 774 | **Advanced** สำหรับ Network-facing App — **Initial** สำหรับ Container/Microservice Workload (ยังไม่มี Coverage) | FMC Connection Events แสดง Source SGT ต่อ Connection |
| **Data** | PKI พื้นฐาน (Part 78 Step 778), TLS สำหรับ Management Plane, Umbrella CASB Visibility | Part 78 Step 778, กระจายหลาย Part | **Initial** — Pillar ที่อ่อนที่สุด | ไม่มี Data Classification Program, ไม่มี DLP เต็มรูปแบบ |
| **Cross-cutting: Automation** | Rapid Threat Containment (ANC+CoA), CI/CD Pipeline สำหรับ Network Config | Part 78 Step 776, Part 79 | **Advanced** สำหรับ Security Response — **Initial** สำหรับเชื่อมกับ Telemetry แบบเต็ม Loop | Worked RTC Scenario ที่ Quarantine อัตโนมัติภายในไม่กี่วินาที |
| **Cross-cutting: Visibility** | Model-Driven Telemetry (Part 54), Enterprise-scale Pipeline (Part 80) | Part 54, 80 | **Advanced** สำหรับ Network Telemetry — **Initial** สำหรับเชื่อมกับ Identity/SGT Context โดยตรง | Dashboard จาก Part 54 Step 538 |

### ส่วนที่ 2: Gap Analysis — สิ่งที่ Enterprise Lab นี้ "ยังไม่มี" อย่างชัดเจน

| Gap | Pillar ที่กระทบ | ผลกระทบถ้าไม่แก้ | เชื่อมโยงกับ Step ที่ระบุไว้ก่อนหน้า |
|---|---|---|---|
| **ไม่มี ZTNA Broker สำหรับ Remote Access** — ยังพึ่ง Traditional IPsec/FlexVPN สำหรับ User เข้าถึง App | Identity, Network | Remote User ยังได้ Network-Level Access เต็มรูปแบบ — Attack Surface สูงกว่าที่ควรเป็น | Step 933 |
| **Telemetry Anomaly Detection ไม่ได้ต่อเข้า CoA Loop โดยตรง** — ยังต้องมี Correlation ผ่าน SecureX/XDR เป็นตัวกลาง ไม่ใช่ Automatic จาก Part 54/80 โดยตรง | Cross-cutting Automation | Mean-Time-to-Contain ยังช้ากว่าที่ Optimal Maturity ต้องการ | Step 935, 938 |
| **Data Pillar ไม่มี Classification/DLP/Encryption-at-Rest Program** | Data | ไม่มีทางพิสูจน์ต่อ Auditor ได้ว่า Data ถูกป้องกันตาม Sensitivity จริง (แค่ Segment ระดับ Network เท่านั้น) | Step 932, 939 |
| **Device Posture เป็น Binary ไม่ใช่ Continuous Risk Score** | Device | ไม่สามารถแยกระดับ Risk ระหว่าง "Non-Compliant เล็กน้อย" กับ "Non-Compliant รุนแรง" ได้ — ตอบสนองแบบเดียวกันหมด | Step 932, 935 |
| **Application/Workload Pillar ไม่ครอบคลุม Container/Microservice** | Application/Workload | ถ้าองค์กรเริ่มมี Internal Application แบบ Microservice ใน Data Center/Cloud (Part 93) จะไม่มี Policy Enforcement ระดับ Workload Identity (เช่น mTLS/SPIFFE) เหมือนที่มีระดับ Network (SGT) | Step 932 |

### ส่วนที่ 3: Prioritized Roadmap — สรุปเป็น Action Plan เดียว

```
Zero Trust Maturity Roadmap — Enterprise Lab (Part 1-93) สรุปรวม

MATURITY SCORE ปัจจุบัน (โดยประมาณ):  Identity ██████████ 80%   Device ███████░░░ 65%
                                     Network  ██████████ 90%   Application ███████░░░ 65%
                                     Data     ███░░░░░░░ 25%   Automation  ███████░░░ 70%

┌─────────────────────────────────────────────────────────────────────────────┐
│  QUARTER 1 (P1 — Quick Win, ต่อยอดจาก Part 44/43 ที่มีอยู่แล้ว)                     │
│  - ตรวจสอบว่าทุก Segment ใน Lab ใช้ SGACL แล้ว ไม่มี IP-based ACL หลงเหลืออยู่           │
│  - เปิด ISE Posture ให้ครอบคลุม Endpoint 100% ของ Lab (ไม่ใช่แค่ Pilot)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  QUARTER 2 (P2 — เชื่อมสิ่งที่มีอยู่ให้เป็น Loop ปิด)                                     │
│  - เชื่อม Part 54/80 Anomaly Detection เข้า ANC/CoA โดยตรง (ไม่ผ่าน Manual Correlation) │
│  - Pilot ZTNA Broker สำหรับ Remote User กลุ่มแรก แทน Traditional VPN                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  QUARTER 3-4 (P3 — เริ่ม Data Pillar และ Workload Identity)                          │
│  - เริ่ม Data Classification Program สำหรับข้อมูลบน Server1-3 (SERVERS_SGT)             │
│  - ประเมิน Workload Identity (mTLS/SPIFFE) สำหรับ Application บน Multi-cloud (Part 93)  │
├─────────────────────────────────────────────────────────────────────────────┤
│  ระยะยาว (Optimal Maturity)                                                          │
│  - Continuous Risk Score รวมทุก Signal (Device+Network+Behavior+Location)             │
│  - Zero Trust Architecture ครอบคลุมทั้ง Campus/Branch/Cloud เป็นสถาปัตยกรรมเดียวสมบูรณ์      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ส่วนที่ 4: Checklist สำหรับใช้เป็น Template ประเมินองค์กรจริง (ไม่ใช่แค่ Lab)

| # | คำถามที่ต้องตอบได้ | Pillar |
|---|---|---|
| 1 | ทุก User/Device ต้อง Authenticate ก่อนได้ Network Access ทุกครั้งหรือไม่ (ไม่มี "Trusted Subnet" ที่ยกเว้น) | Identity |
| 2 | Device ที่ไม่ผ่าน Posture Check ถูกจำกัดสิทธิ์อัตโนมัติหรือไม่ (ไม่ต้องรอ Admin กดปุ่ม) | Device |
| 3 | Policy การเข้าถึงผูกกับ "กลุ่ม/ตัวตน" หรือยังผูกกับ IP/Subnet เป็นหลัก | Network |
| 4 | Firewall/Application Policy ใช้ Context เดียวกับ Network Policy (SGT/Identity) หรือแยกกันคนละชุด | Application/Workload |
| 5 | มี Data Classification Program ที่ระบุได้ว่าข้อมูลไหน Sensitive แค่ไหนหรือไม่ | Data |
| 6 | เมื่อพบ Threat สามารถ Quarantine อัตโนมัติได้ภายในไม่กี่วินาที หรือยังต้องรอ Admin ตอบสนองเอง | Automation |
| 7 | Log จากหลายระบบ Cross-validate กันได้ตอน Audit หรือแต่ละระบบ Log แยก Silo กันอยู่ | Visibility/Governance |

> **ข้อสรุปปิด Lab**: Enterprise Lab ที่หลักสูตรนี้สร้างมาตั้งแต่ [Part 1](part-001-networking-fundamentals.md)
> ผ่านการประเมินแล้วอยู่ในระดับ **Advanced** สำหรับ Identity/Network/Application Pillar และ
> **Automation** — แต่ยังอยู่ระดับ **Initial-to-Advanced** สำหรับ Device (Binary Posture) และ
> **Initial** สำหรับ Data Pillar อย่างชัดเจน — นี่คือระดับที่ **สมเหตุสมผลมากสำหรับหลักสูตรที่
> โฟกัส Enterprise Network Infrastructure** (Data Pillar เป็นงานของ Data/Application Security
> Team มากกว่า Network Team) และ Roadmap ข้างต้นคือแผนที่ **Network/Security Architect ระดับ
> CCIE ควรเสนอต่อผู้บริหาร** เพื่อยกระดับ Maturity ต่อไปในโลกจริง

---

## แบบฝึกหัดทวนความเข้าใจ Part 94

1. อธิบายว่าสมมติฐานของ Castle-and-Moat Model ที่ฝังอยู่ใน Part 1-10 ของหลักสูตรนี้คืออะไร และ
   ทำไมมันถึงใช้ไม่ได้อีกต่อไปในโลกที่มี Remote Work และ Cloud/SaaS
2. จับคู่ 5 Zero Trust Pillar เข้ากับ Part ที่หลักสูตรนี้สร้าง Feature ของ Pillar นั้นไว้แล้ว และ
   ระบุว่า Pillar ใดยังมี Coverage อ่อนที่สุด
3. อธิบายความแตกต่างระหว่าง Traditional Remote-Access VPN กับ ZTNA ในมิติ "Unit of Trust" และ
   ทำไม ZTNA ถึงลด Attack Surface ได้มากกว่าเมื่อ Credential ถูกขโมย
4. อธิบายว่าทำไม TrustSec/SGT ที่สอนใน Part 44 ถึงถือว่าเป็น "Network Pillar ของ Zero Trust"
   ที่สมบูรณ์ — ใช้แนวคิด PDP (Policy Decision Point) และ PEP (Policy Enforcement Point) ของ
   NIST 800-207 ในการอธิบาย
5. ทำไม Continuous Verification จึงให้ Audit Evidence ที่ Auditor เชื่อได้มากกว่า Periodic
   Point-in-Time Review — ยกตัวอย่างจาก Log Cross-validation ของ Part 78 Step 779

**เฉลย:**

1. สมมติฐานคือ "ทุกอย่างที่อยู่หลัง FW-1 (Core/Distribution/Access) คือ Trusted Zone โดยอัตโนมัติ
   ไม่ต้องตรวจสอบซ้ำ ส่วนทุกอย่างนอก FW-1 คือ Untrusted" — มันใช้ไม่ได้อีกต่อไปเพราะ (1) พนักงาน
   ทำงานจากที่ไหนก็ได้ ไม่ได้อยู่หลัง FW-1 เสมอไป ทำให้ "ตำแหน่งเครือข่าย" ไม่ใช่ตัวบ่งบอกความ
   น่าเชื่อถือ (2) Application สำคัญอยู่บน Cloud/SaaS ที่ Traffic ไม่จำเป็นต้องผ่าน FW-1 เลย และ
   (3) ต้อง Assume Breach เสมอ เพราะถ้า Attacker หลุดผ่าน Perimeter มาได้ครั้งเดียว จะเดิน
   Lateral Movement ได้อย่างอิสระถ้าไม่มี Checkpoint ที่สองรออยู่ข้างใน
2. Identity → Part 22 (AAA), Part 37 (802.1X Wireless), Part 43 (ISE) | Device → Part 43 Step 428
   (Posture Assessment) | Network/Environment → Part 44 (TrustSec/SGT), Part 77 (SD-Access) |
   Application/Workload → Part 43/78 Step 774 (FTD ACP Match SGT) | Data → ไม่มี Part เฉพาะ
   กระจายอยู่บางส่วน (PKI จาก Part 78 Step 778, Umbrella CASB) — **Data Pillar อ่อนที่สุด**
   เพราะไม่มี Data Classification/DLP/Encryption-at-Rest Program ที่เป็นระบบ
3. Traditional VPN มี Unit of Trust เป็น "Network" — หลัง Authenticate สำเร็จ Client ได้ Virtual
   IP ในเครือข่ายภายในและ "เห็น" Network Topology ทั้งหมด (แม้จะถูกจำกัดด้วย ACL บางส่วน) ส่วน
   ZTNA มี Unit of Trust เป็น "Application" แต่ละตัว — Client ไม่ได้รับ IP ในเครือข่ายภายในเลย
   เห็นแค่ App ที่ Policy อนุญาตให้เห็นเท่านั้น ถ้า Credential ถูกขโมย Attacker ใน Traditional VPN
   จะเห็น Network ทั้งหมดและสแกนหา Target อื่นได้ (Lateral Movement ง่าย) แต่ใน ZTNA Attacker
   เห็นแค่ App ที่ User นั้นมีสิทธิ์อยู่แล้ว ไม่มี "Network" ให้สแกนต่อเลย
4. TrustSec ตรงกับโมเดล PDP/PEP ของ NIST 800-207 พอดี: ISE ทำหน้าที่ **PDP** (Policy Decision
   Point) ที่ตัดสินใจว่า Endpoint ควรได้ SGT อะไรผ่าน 802.1X/MAB ส่วน ACCESS-SW (Port
   Authorization), DIST-SW (SGACL), และ FW-1 (ACP Match SGT) ทำหน้าที่เป็น **PEP** (Policy
   Enforcement Point) ที่บังคับใช้ Policy จริงในทุกจุดที่มีการเข้าถึงเกิดขึ้น — และเพราะ SGT ผูก
   กับ "ตัวตน" ไม่ใช่ "ตำแหน่งเครือข่าย" (Endpoint ย้าย VLAN/Site ไปไหนก็ยังได้สิทธิ์เท่าเดิม)
   มันจึงสาธิตหลักการ "Never Trust Based on Network Location Alone" ได้ตรงที่สุด
5. Periodic Point-in-Time Review มีช่องว่างที่ Auditor มองไม่เห็นสิ่งที่เกิดขึ้น "ระหว่าง" รอบ Audit
   และต้องเชื่อ Snapshot ที่องค์กรเตรียมให้ดู (เสี่ยงถูกจัดฉาก) ส่วน Continuous Verification สร้าง
   Log ทันทีทุก Access Request แบบ Real-time และให้ Evidence ที่ Cross-validate กันได้จากหลาย
   ระบบพร้อมกัน — ตัวอย่างจาก Part 78 Step 779: ISE MnT ยืนยันว่า employee01 Authenticate ผ่าน
   802.1X ได้ SGT=SALES_SGT, `show cts role-based counters` ที่ DIST-SW3 ยืนยันว่า SGT นั้น
   HW-Permitted เข้าถึง Server จริงในเวลาเดียวกัน, และ FMC Connection Events ยืนยัน SGT/User
   เดียวกันที่ผ่าน FW-1 — 3 แหล่งข้อมูลนี้เล่าเรื่องเดียวกันและยืนยันกันเอง ซึ่งแข็งแรงกว่า Log
   จากแหล่งเดียวมาก

---

## สรุป Part 94

Part นี้ไม่ได้สร้างเทคโนโลยีใหม่จากศูนย์สักตัวเดียว — และนั่นคือประเด็นสำคัญที่สุดของมัน สิ่งที่
Part 94 ทำคือ **ยืนอยู่บนจุดสูงสุดของหลักสูตรแล้วมองย้อนกลับไปทั้ง 93 Part ก่อนหน้า** เพื่อพิสูจน์ว่า
Enterprise Lab ที่เราสร้างมาตั้งแต่ [Part 1](part-001-networking-fundamentals.md) — ผ่าน AAA
(Part 22), 802.1X Wireless (Part 37), ISE+FTD (Part 43), TrustSec/SGT (Part 44), IPsec VPN
(Part 51), Telemetry (Part 54, 80), SD-WAN Security (Part 76), SD-Access (Part 77), และ
Integration เต็มรูปแบบ (Part 78) — **คือ Zero Trust Architecture ที่กำลังก่อตัวขึ้นทีละชิ้นมา
ตลอดทาง โดยไม่มีใครประกาศชื่อมันตรงๆ จนกระทั่งวันนี้**

เราเริ่มจากนิยาม "Never Trust, Always Verify" และพิสูจน์ว่า Perimeter Model ที่ Part 1-10 วางไว้
ใช้ไม่ได้อีกต่อไป (Step 931), แมป 5 Pillar ของ Zero Trust เข้ากับ Part ที่สร้างไว้แล้วพร้อมชี้ว่า
Data Pillar คือช่องว่างที่แท้จริง (Step 932), เปรียบเทียบ ZTNA กับ Traditional VPN ของ Part 51
(Step 933), ยกระดับ TrustSec/SGT ให้เป็นภาษา Micro-Segmentation อย่างเป็นทางการผ่าน PDP/PEP
Model (Step 934), ปิด Loop ระหว่าง Telemetry (Part 54/80) กับ Automated Response (Part 78) ให้
เป็น Continuous Verification (Step 935), พิสูจน์ว่า Campus LAN ทำ Zero Trust ไปแล้วสมบูรณ์ไม่ต้อง
รอ Cloud (Step 936), ขยายกรอบคิดเดียวกันไปที่ WAN/Branch ผ่าน SD-WAN Security (Step 937), วาง
Maturity Roadmap แบบ Incremental ที่ต่อยอดจากสิ่งที่มีอยู่แล้ว (Step 938), เชื่อม Zero Trust กับ
Compliance/Audit ให้เห็นว่ามันคือผลพลอยได้ไม่ใช่ภาระเพิ่ม (Step 939), และปิดท้ายด้วย **Zero Trust
Maturity Assessment เต็มรูปแบบของ Enterprise Lab ทั้งก้อน** พร้อม Gap Analysis และ Prioritized
Roadmap (Step 940)

หลักการที่สำคัญที่สุดของ Part นี้คือ: **Zero Trust ไม่ใช่ปลายทางที่ไปถึงแล้วจบ และไม่ใช่ Product
ที่ซื้อมาแล้วเสร็จ — มันคือทิศทางการออกแบบที่ต่อเนื่อง** องค์กรที่เข้าใจผิดว่า Zero Trust คือ
โปรเจกต์ที่มีวันส่งมอบ จะหยุดพัฒนาทันทีที่ "Deploy ครบ Checklist" — องค์กรที่เข้าใจถูกจะรู้ว่า
Maturity ต้อง**ยกระดับต่อเนื่องตลอดไป** ตามที่ Threat Landscape และ Business Requirement
เปลี่ยนแปลง — นี่คือเหตุผลที่ Step 940 ให้ Roadmap ไม่ใช่ Checklist ปิดจบ

**Part 94 คือ Part สุดท้ายของสาย "Technology/Architecture Deep-Dive"** ของหลักสูตรทั้งหมด
(Part 1-94) — จากจุดนี้ไป **[Part 95-100](part-095-ccie-lab-scenario-1-diagnose.md) จะไม่มี
เทคโนโลยีใหม่ให้เรียนอีกแล้ว** มีแต่ **CCIE Full Lab Scenario** ที่นำทุกอย่างที่เรียนมาตลอด 94
Part — Routing, Switching, Wireless, Automation, Security, SD-WAN, SD-Access, Cloud, และ Zero
Trust Architecture ที่ Part นี้เพิ่งประกอบร่างให้เห็นภาพรวม — มาทดสอบภายใต้ Time Pressure จริง
แบบเดียวกับที่ CCIE Lab Exam วัดผล เริ่มจาก **Diagnose Module** ที่จำลองสถานการณ์ "เครือข่ายมี
ปัญหาอยู่แล้ว หาสาเหตุและแก้ไขให้ตรง Root Cause" บน Enterprise Lab เดียวกันที่เราสร้างมาตั้งแต่
Part 1

**ไปต่อ:** [Part 95 — CCIE Lab Scenario #1: Diagnose Module →](part-095-ccie-lab-scenario-1-diagnose.md)
