# Part 50 — Cisco DNA Center / Catalyst Center
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 491–500 จาก 1000**

> ต่อจาก [Part 49 — APIs, YANG, NETCONF/RESTCONF Deep Dive](part-049-netconf-restconf-yang-deep-dive.md)
> ที่เราเรียนรู้วิธีคุยกับอุปกรณ์ Cisco ทีละตัวด้วย Model-Driven Programmability (YANG/NETCONF/
> RESTCONF) และปิดท้ายด้วย Lab เปรียบเทียบ CLI/Netmiko vs NETCONF vs RESTCONF บนอุปกรณ์ **ตัวเดียว**
> — Part นี้จะยกระดับคำถามขึ้นไปอีกขั้น: **ถ้าต้อง Automate อุปกรณ์ "ทั้งองค์กร" หลายสิบหลายร้อยตัว
> พร้อมกัน จะเขียน Script เองทุกจุดไหวหรือไม่?** คำตอบของ Cisco คือ **Cisco DNA Center** (ปัจจุบัน
> Rebrand เป็น **Cisco Catalyst Center** ตั้งแต่ปี 2024 เป็นต้นไป แต่แนวคิด/Feature เหมือนเดิมทุก
> ประการ — บทความนี้จะใช้ทั้งสองชื่อสลับกันไปตามบริบท) ซึ่งเป็น Controller ระดับ Enterprise ที่ใช้
> NETCONF/RESTCONF (Part 49) และ YANG Model (Part 49) **เป็นเครื่องมือเบื้องหลัง** ในการ Automate
> ทั้ง Network โดยวิศวกรไม่ต้องเขียน Python เองทุกบรรทัด — และเป็น Platform เดียวกันที่ [Part 38
> Step 376](part-038-sd-access-fundamentals.md) ได้แนะนำบทบาทของมันไว้แล้วแบบสั้นๆ ในบริบทของ
> SD-Access เท่านั้น **Part นี้คือ Part ที่จะเจาะลึก DNA Center ในฐานะ Platform เต็มรูปแบบ** ครอบคลุม
> ทุก Workflow หลัก ไม่จำกัดแค่ SD-Access Fabric

> **สำคัญมาก — ขอบเขตของ Part นี้**: เนื้อหานี้เป็น **GUI/Platform-driven** เกือบทั้งหมด ต่างจาก Part
> ก่อนๆ ที่เน้น CLI เป็นหลัก เพราะ DNA Center **ถูกออกแบบมาให้ไม่ต้องพิมพ์ CLI เอง** — เราจะบรรยาย
> Workflow, หน้าจอ, และแนวคิดเชิงสถาปัตยกรรมเป็นหลัก พร้อมตัวอย่าง API เพียงส่วนที่จำเป็น (Step 497)
> Lab ของหลักสูตร (CORE-SW1/2, DIST-SW1–4, ACCESS-SW1–4 ตามที่ [`00-ip-address-plan.md`](00-ip-address-plan.md)
> กำหนดไว้) จะถูกใช้เป็น "กรณีศึกษา" ตลอด Part นี้ว่าถ้าจะ Adopt เข้า DNA Center จะต้องทำอะไรบ้าง

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 491 | DNA Center / Catalyst Center คืออะไร — "Single Pane of Glass" และ 4 Workflow หลัก |
| 492 | Design Workflow — Network Hierarchy, Credential Profile, IP Pool, Image Repository |
| 493 | Policy Workflow — Group-Based Access Control (GBAC), Application Policy, Virtual Network |
| 494 | Provision Workflow — Plug and Play (PnP), LAN Automation, Fabric Provisioning |
| 495 | Assurance — Network Health, Client Health, 360 View, Issues, Path Trace |
| 496 | Templates & Day-N Configuration — CLI Template, Composite Template |
| 497 | DNA Center Intent API — Northbound REST API สำหรับ External Automation |
| 498 | Software Image Management (SWIM) — Golden Image, Scheduled Upgrade |
| 499 | Multivendor, SD-WAN/Meraki Integration, และ Licensing Model |
| 500 | Lab เชิงแนวคิดเต็มรูปแบบ: แผน Adoption DNA Center สำหรับ Lab Part 1–49 |

---

## Step 491 — DNA Center / Catalyst Center คืออะไร: "Single Pane of Glass"

### จากปัญหาที่สะสมมาตลอด 49 Part สู่คำตอบเดียว

ตลอดหลักสูตรนี้ตั้งแต่ Part 2 เราทำสิ่งเดียวกันซ้ำแล้วซ้ำอีก: **SSH เข้าไปที่อุปกรณ์ทีละตัว พิมพ์
CLI ทีละบรรทัด** — Part 47–49 พยายามแก้ปัญหานี้ด้วย Python/Ansible/NETCONF แต่ก็ยังต้องมีคนเขียน
และดูแล Script เอง ยังไม่มี "ที่เดียว" ที่รวมทั้ง **มองเห็น (Visibility)**, **ควบคุม (Control)**, และ
**วิเคราะห์ปัญหา (Assurance)** ของทั้งเครือข่ายไว้ในหน้าเดียว

**Cisco DNA Center (DNAC)** คือคำตอบของ Cisco — เป็น **Network Controller/Management Platform**
ระดับ Enterprise ที่ทำงานเป็น **"Single Pane of Glass"**: หน้าจอเดียวที่วิศวกรใช้ **ออกแบบ
(Design)**, **กำหนด Policy**, **Provision อุปกรณ์**, และ **Monitor สุขภาพเครือข่าย** ของทั้ง Campus/
Branch/WAN โดยไม่ต้องสลับไปเปิด SSH Session ทีละหน้าต่างอีกต่อไป

```
ก่อนมี DNA Center (Part 1-49 ตามที่เราทำมา):

  วิศวกร ──SSH──► CORE-SW1     วิศวกร ──SSH──► DIST-SW1     วิศวกร ──SSH──► ACCESS-SW1
  วิศวกร ──SSH──► CORE-SW2     วิศวกร ──SSH──► DIST-SW2     วิศวกร ──SSH──► ACCESS-SW2
                               วิศวกร ──SSH──► DIST-SW3     วิศวกร ──SSH──► ACCESS-SW3
                               วิศวกร ──SSH──► DIST-SW4     วิศวกร ──SSH──► ACCESS-SW4
  (10 Session แยกกัน ต่างคน ต่างจำ Syntax ต่างจำ IP เอง — ไม่มีจุดรวมข้อมูล)

หลังมี DNA Center:

                         ┌─────────────────────────────┐
                         │   Cisco DNA Center / Catalyst  │
                         │        Center (GUI เดียว)      │
                         └───────────────┬───────────────┘
                                REST API / NETCONF / SNMP / Syslog / Telemetry
              ┌──────────────┬───────────┼───────────┬──────────────┐
              ▼              ▼           ▼           ▼              ▼
          CORE-SW1       CORE-SW2    DIST-SW1-4   ACCESS-SW1-4   WLC-1/AP
        (1 หน้าจอ เห็น Inventory, Health, Config, Policy ของทุกตัวพร้อมกัน)
```

### DNA Center คือใครในสถาปัตยกรรมของหลักสูตรนี้

สังเกตว่า DNA Center **ไม่ใช่อุปกรณ์เครือข่ายเอง** — มันคือ **Software Appliance** (รันบน Cisco
DN2-HW Appliance เฉพาะ หรือ Virtual บน ESXi ขนาดใหญ่) ที่เชื่อมต่อเข้ากับ Management Network
(VLAN 99 MGMT ตาม [`00-ip-address-plan.md`](00-ip-address-plan.md)) แล้วคุยกับทุกอุปกรณ์ผ่าน
Protocol ที่เรียนมาแล้วทั้ง Part 17 (SNMP), Part 23/47-48 (SSH/Netmiko เบื้องหลัง), และ Part 49
(NETCONF/RESTCONF) — DNAC **ไม่ได้คิด Protocol ใหม่** มันแค่ **ห่อ Protocol เดิมทั้งหมดไว้ใน GUI
เดียว** แล้วเพิ่ม "สมอง" (Intent-based Automation + AI/ML Analytics) เข้าไป

### DNA Center vs Catalyst Center: การ Rebrand

| ประเด็น | Cisco DNA Center (ชื่อเดิม) | Cisco Catalyst Center (ชื่อใหม่ ตั้งแต่ปี 2024) |
|---|---|---|
| Feature/Workflow | เหมือนกันทั้งหมด | เหมือนกันทั้งหมด |
| API Endpoint | `/dna/intent/api/...` | ยังคงใช้ Path เดิม (`dna`) เพื่อ Backward Compatibility |
| เหตุผลที่ Rebrand | — | จัดกลุ่ม Portfolio ใหม่ให้เข้ากับชื่อ "Catalyst" (ตระกูล Switch/AP หลักของ Cisco) และเชื่อมกับ Cisco Networking Cloud |
| สิ่งที่ข้อสอบ CCNP ENCOR ปัจจุบันใช้ | อาจเจอทั้งสองชื่อในโจทย์ ให้ถือว่าเป็น Platform เดียวกัน | เหมือนกัน |

### 4 Workflow หลักของ DNA Center — ทบทวนจาก Part 38 ให้ลึกขึ้น

[Part 38 Step 376](part-038-sd-access-fundamentals.md) แนะนำ 4 Workflow นี้ไว้แล้วในบริบทของ
SD-Access Fabric เท่านั้น — แต่ในความเป็นจริง **4 Workflow นี้ใช้ได้กับเครือข่ายทุกแบบ** ไม่ว่าจะเป็น
Traditional 3-Tier (Lab หลักของหลักสูตรนี้ Part 1–37, 39–49) หรือ SD-Access Fabric (Part 38) —
DNA Center ไม่บังคับว่าต้องใช้ SD-Access เสมอไป

```
┌──────────┐     ┌──────────┐     ┌────────────┐     ┌─────────────┐
│  DESIGN   │ ──► │  POLICY   │ ──► │  PROVISION  │ ──► │  ASSURANCE   │
│           │     │           │     │             │     │              │
│ Site      │     │ GBAC      │     │ PnP         │     │ Health       │
│ Hierarchy │     │ (SGT/     │     │ LAN         │     │ Dashboard     │
│ IP Pool   │     │ TrustSec) │     │ Automation  │     │ Client 360    │
│ Credential│     │ App QoS   │     │ Fabric      │     │ Issues (AI)   │
│ Image     │     │ Virtual   │     │ Provisioning│     │ Path Trace    │
│           │     │ Network   │     │             │     │              │
│ (Step 492)│     │ (Step 493)│     │ (Step 494)  │     │ (Step 495)   │
└──────────┘     └──────────┘     └────────────┘     └─────────────┘
     ▲                                                       │
     └────────────────── วนกลับเมื่อ Design เปลี่ยน ─────────────┘
```

| Workflow | ตอบคำถามอะไร | ผลลัพธ์ที่ได้ |
|---|---|---|
| **Design** | "เครือข่ายเรามีโครงสร้างอย่างไร และ IP Address แต่ละที่คืออะไร" | Site Hierarchy + IP Pool + Credential Template พร้อมใช้ |
| **Policy** | "ใครคุยกับใครได้ และ Traffic ประเภทไหนสำคัญกว่ากัน" | GBAC Matrix + QoS Policy + VN Definition |
| **Provision** | "จะเอา Config ไปลงอุปกรณ์จริงอย่างไรโดยไม่ต้อง SSH เอง" | อุปกรณ์ Onboard สำเร็จ พร้อม Config ตาม Intent |
| **Assurance** | "ตอนนี้เครือข่ายเราสุขภาพเป็นอย่างไร มีปัญหาอะไรซ่อนอยู่บ้าง" | Dashboard + Alert + Root Cause Suggestion |

> **สำหรับข้อสอบ CCNP ENCOR**: คำว่า **"Single Pane of Glass"** และลำดับ **Design → Policy →
> Provision → Assurance** เป็นคำที่ข้อสอบใช้เป๊ะๆ ต้องจำคำนี้ตรงตัว — และต้องเข้าใจว่า DNA Center
> ใช้ได้กับทั้ง Traditional Network และ SD-Access Fabric ไม่ใช่ผูกกับ SD-Access เท่านั้น

---

## Step 492 — Design Workflow: Network Hierarchy, Credential, IP Pool, Image

### Network Hierarchy: Site → Building → Floor

**Design Workflow** เริ่มต้นด้วยการสร้าง **Network Hierarchy** — โครงสร้างต้นไม้ที่จำลอง
"สถานที่ตั้งทางกายภาพจริง" ขององค์กร เพื่อให้ Policy/Config ทุกอย่างที่กำหนดใน Workflow ถัดไป
สามารถ **ผูกกับสถานที่** ได้ (เช่น "IP Pool นี้ใช้เฉพาะ Floor 3 ของตึก HQ เท่านั้น")

```
Global
  └── Area: Thailand
        └── Site (Building): Bangkok HQ         <- ตรงกับ Lab หลักสูตรนี้ทั้งหมด
              ├── Floor: Floor 1  (ACCESS-SW1, ACCESS-SW2 ตั้งอยู่)
              ├── Floor: Floor 2  (ACCESS-SW3, ACCESS-SW4 ตั้งอยู่)
              └── Floor: Floor 3  (CORE-SW1/2, DIST-SW1-4, WAN-EDGE-1/2 ตั้งอยู่ใน Server Room)
```

| Level | ตัวอย่างในหลักสูตรนี้ | ใช้กำหนดอะไรได้ที่ Level นี้ |
|---|---|---|
| **Area** | "Thailand" | จัดกลุ่ม Site ทางภูมิศาสตร์ (ไม่มี IP/Network Setting ตรงๆ) |
| **Building/Site** | "Bangkok HQ" | Device Credential Profile, NTP/DNS/Syslog Server, AP-Location |
| **Floor** | "Floor 1", "Floor 2", "Floor 3" | IP Address Pool (ผูกกับ VLAN จริง), Floor Map สำหรับ Wireless Heatmap |

### Device Credential Profile

DNA Center ต้องมี **Credential** (Username/Password, Enable Secret, SNMP Community/SNMPv3,
NETCONF/RESTCONF Login — ทบทวนจาก Part 49) เพื่อ Login เข้าอุปกรณ์ทุกตัว แทนที่จะพิมพ์ Credential
ทุกครั้งที่ Provision อุปกรณ์ใหม่ วิศวกรจะสร้าง **Credential Profile** ไว้ล่วงหน้าแล้วผูกกับ Site:

| Credential Type | ตัวอย่างค่าที่ Config ไว้ล่วงหน้า | เทียบกับ Part ก่อนหน้า |
|---|---|---|
| CLI (SSH) | Username `netadmin`, Enable Secret | Part 2 (`enable secret`), Part 47-48 (Netmiko inventory) |
| SNMP | SNMPv3 (`authPriv`, SHA/AES) | Part 17 |
| HTTP(S) — NETCONF/RESTCONF | Username/Password หรือ Token | Part 49 |

> **หมายเหตุ**: Credential Profile ผูกกับ **Site** (เช่น "Bangkok HQ ใช้ Credential Set A")
> ไม่ใช่ผูกทีละอุปกรณ์ — ทำให้เปลี่ยน Password ทั้ง Site ได้ในคลิกเดียว ไม่ต้องไล่เปลี่ยนทุกอุปกรณ์
> ทีละตัวแบบที่ Part 2 สอนไว้

### IP Address Pool Management — Mapping กับแผน VLAN เดิมของหลักสูตร

นี่คือส่วนที่เชื่อมกับ [`00-ip-address-plan.md`](00-ip-address-plan.md) ตรงที่สุด — DNA Center ให้
วิศวกร Define **IP Pool** ไว้ล่วงหน้าที่ Global แล้ว **Reserve** (จัดสรร) ลงไปที่ Site/Floool ใดก็ได้
โดยไม่ต้องคำนวณ Subnet มือเองอีกต่อไปตอน Provision:

| IP Pool Name (DNAC) | Subnet | VLAN ที่ผูก | ตรงกับ VLAN ใน Canonical Plan |
|---|---|---|---|
| `SALES_POOL` | 10.10.10.0/24 | 10 | SALES |
| `VOICE_POOL` | 10.10.20.0/24 | 20 | VOICE |
| `SERVERS_POOL` | 10.10.30.0/24 | 30 | SERVERS |
| `WIFI_POOL` | 10.10.40.0/24 | 40 | WIFI |
| `MGMT_POOL` | 10.10.99.0/24 | 99 | MGMT |

```
DNAC IP Pool Manager (Global) ──Reserve──► Site "Bangkok HQ" ──Assign──► VLAN บน ACCESS-SW ที่ Provision

ตัวอย่าง: ตอน Provision ACCESS-SW1 ผ่าน DNAC วิศวกรแค่เลือก "VLAN 10 = SALES_POOL"
         DNAC จะสร้าง `interface vlan 10` / `ip helper-address` ให้อัตโนมัติ
         ตรงกับที่ Part 3 (สร้าง VLAN) และ Part 16 (DHCP Relay) เคย config ด้วยมือ
```

> **ข้อสังเกตสำคัญ**: DNA Center **ไม่ได้เปลี่ยนแผน IP Address** ของหลักสูตรเลยแม้แต่ Subnet เดียว
> — มันแค่ทำให้การ "Reserve/Assign" Subnet เดิมที่มีอยู่แล้วเป็น GUI Workflow แทนการนั่งคำนวณและ
> พิมพ์ `ip address` มือทุกจุด (Part 3, Part 10)

### Image Repository (นำเข้าสู่ Step 498 แบบสั้นๆ)

Design Workflow ยังมีส่วน **Image Repository** — คลังเก็บไฟล์ IOS-XE Image ทุกเวอร์ชันที่องค์กรใช้
(อัปโหลดเข้า DNAC ครั้งเดียว) เพื่อให้ Workflow อื่น (Provision, SWIM ที่ Step 498) เรียกใช้ได้โดยไม่
ต้อง `copy tftp: flash:` มือทีละอุปกรณ์แบบ Part 2

---

## Step 493 — Policy Workflow: GBAC, Application Policy, Virtual Network

### Group-Based Access Control (GBAC) — หน้า GUI ของ TrustSec/SGT

[Part 44](part-044-dot1x-trustsec.md) สอน **Cisco TrustSec/SGT** ผ่าน CLI โดยตรง (`cts role-based
permissions`, `cts manual`, ISE Policy) — **GBAC ใน DNA Center คือหน้าตา GUI ของสิ่งเดียวกันนี้**
ไม่ใช่เทคโนโลยีใหม่ วิศวกรกำหนด **Scalable Group** และ **Policy Matrix** ผ่านตารางคลิกได้ แล้ว
DNAC จะแปลงเป็น `cts role-based permissions` Config เดิมเป๊ะ ส่งไปที่อุปกรณ์ผ่าน NETCONF/RESTCONF
(Part 49) เบื้องหลัง

```
GBAC Policy Matrix (ตัวอย่างในหน้า DNAC — เชิงแนวคิด):

              │ EMPLOYEE │ CONTRACTOR │ SERVERS_GRP │ PRINTER │
──────────────┼──────────┼────────────┼─────────────┼─────────┤
 EMPLOYEE     │  Permit  │  Permit    │   Permit    │ Permit  │
 CONTRACTOR   │  Permit  │  Permit    │   Deny      │ Permit  │  <- คลิกเซลล์นี้ = สร้าง SGACL
 SERVERS_GRP  │  Permit  │  Deny      │   Permit    │  Deny   │
 PRINTER      │  Permit  │  Permit    │    Deny     │ Permit  │

คลิก 1 เซลล์ = DNAC สร้าง `cts role-based permissions from CONTRACTOR to SERVERS_GRP deny_all`
             ให้อัตโนมัติ แล้ว Push ไปทุก Enforcement Point ที่เกี่ยวข้องพร้อมกัน
```

| หัวข้อ | CLI ตรงจาก Part 44 | GBAC บน DNAC |
|---|---|---|
| นิยามกลุ่ม | `cts role-based sgt-map` ทีละบรรทัด | สร้าง "Scalable Group" ผ่าน GUI form |
| นิยาม Policy | `cts role-based permissions from X to Y` | คลิกเซลล์ใน Matrix |
| Push ไปอุปกรณ์ | SSH เข้าไป Apply ทีละตัว | DNAC Push ให้ทุกอุปกรณ์ที่เกี่ยวข้องพร้อมกัน |
| แหล่งข้อมูลกลุ่ม | ISE (Identity Services Engine) ผูกกับ DNAC ผ่าน pxGrid | เหมือนกัน — DNAC ดึง SGT Assignment มาจาก ISE |

### Application Policy — หน้า GUI ของ QoS (MQC)

[Part 40](part-040-advanced-qos.md) สอน **MQC (Modular QoS CLI)**: `class-map` → `policy-map` →
`service-policy` ทีละคำสั่งบน Interface — **Application Policy บน DNAC คือ GUI ที่สร้าง MQC ชุด
เดียวกันนี้ให้อัตโนมัติ** โดยวิศวกรไม่ต้องจำ DSCP Value หรือ Bandwidth Percentage เอง แค่เลือก
"ระดับความสำคัญของ Application" จาก Dropdown

```
Application Policy Workflow บน DNAC:

1. เลือก Application (จาก NBAR2 Signature ที่ DNAC รู้จักในตัว — Cisco Webex, VoIP, SAP, ฯลฯ)
2. จัดเข้ากลุ่ม Business-Relevance: Business-Relevant / Default / Business-Irrelevant
3. DNAC map กลุ่มเข้ากับ Queue/DSCP มาตรฐาน (RFC 4594 model เดียวกับที่ Part 40 สอน)
4. เลือก Site/Device Role ที่จะ Apply แล้วกด Deploy

ผลลัพธ์ (สิ่งที่ DNAC Push ไปให้จริง — เหมือน Part 40 ทุกตัวอักษร):
  class-map match-any VOICE
   match dscp ef
  policy-map WAN-EDGE-POLICY
   class VOICE
    priority percent 10
   class BUSINESS-RELEVANT
    bandwidth remaining percent 60
  interface Gi0/0/0
   service-policy output WAN-EDGE-POLICY
```

> **ข้อสังเกตสำคัญสำหรับข้อสอบ**: Application Policy บน DNAC **ไม่ได้สร้าง QoS Model ใหม่**
> มันยังอิงกรอบ **RFC 4594 12-Class Model** และ **DiffServ/DSCP** เดิมที่ Part 40 สอนไว้ทุกประการ
> เพียงเปลี่ยนวิธี "เขียน" จาก CLI เป็น GUI Form เท่านั้น

### Virtual Network (VN) Policy — สำหรับ SD-Access เท่านั้น

ต่างจาก GBAC และ Application Policy ที่ใช้ได้กับทั้ง Traditional Network และ Fabric, **Virtual
Network (VN)** ตามที่ [Part 38 Step 374-375](part-038-sd-access-fundamentals.md) สอนไว้
**ใช้ได้เฉพาะเมื่อ Site นั้นถูก Provision เป็น SD-Access Fabric แล้วเท่านั้น** (ต้องมี Fabric Border/
Edge/Control Plane Node ตาม Step 494) — Policy Workflow เป็นจุดที่วิศวกร **สร้างชื่อ VN**
(`EMPLOYEE_VN`, `IOT_VN`, `GUEST_VN`) และผูก IP Pool จาก Step 492 เข้ากับ VN นั้นก่อนที่จะนำไป
Provision จริงที่ Step 494

| หัวข้อ | ใช้กับ Traditional Network ได้ไหม | ใช้กับ SD-Access Fabric ได้ไหม |
|---|---|---|
| GBAC (SGT) | ได้ (ผ่าน 802.1X + SGT tagging, Part 44) | ได้ (ฝังใน VXLAN Header, Part 38 Step 374) |
| Application Policy (QoS) | ได้ (MQC ปกติ, Part 40) | ได้ (Apply บน Fabric Border/Edge) |
| Virtual Network (VN) | **ไม่ได้** (ไม่มี VXLAN/LISP Overlay) | ได้ (เป็นแนวคิดเฉพาะของ Fabric) |

---

## Step 494 — Provision Workflow: PnP, LAN Automation, Fabric Provisioning

### Plug and Play (PnP) — Zero-Touch Onboarding

**Plug and Play (PnP)** คือกลไกที่อุปกรณ์ Cisco ตัวใหม่เอี่ยม (ไม่มี Config เลย, Factory Default)
สามารถ **ค้นหา DNA Center เองอัตโนมัติ** ผ่านเครือข่าย แล้วรับ Config เริ่มต้นจาก DNAC โดยที่
วิศวกรไม่ต้องต่อ Console Cable เข้าไป Config ตั้งต้น (`hostname`, `enable secret`, IP Management)
ด้วยมือแบบที่ [Part 2](part-002-ios-cli-basics.md) สอนไว้เลยแม้แต่บรรทัดเดียว

```
ขั้นตอน PnP (แนวคิด):

1. ช่างเสียบสาย Switch ใหม่เข้า Network แล้วเปิดเครื่อง (Factory Default, ไม่มี Startup-config)
2. Switch พยายามหา PnP Server ผ่าน 4 วิธีตามลำดับ:
   a. DHCP Option 43 (ชี้ตรงไปยัง DNAC IP)
   b. DNS "pnpserver.<domain>"
   c. Cisco Cloud Redirection Service (พาไปหา DNAC on-prem ที่ Register ไว้)
   d. USB/Console manual bootstrap (สำรองสุดท้าย)
3. Switch คุยกับ DNAC ผ่าน HTTPS, ส่ง Serial Number ให้ตรวจสอบ
4. DNAC เทียบ Serial Number กับ "Planned Device" ที่วิศวกร Pre-provision ไว้ล่วงหน้าใน GUI
5. DNAC Push: Hostname, Credential, Site Assignment, Template (Step 496), Image (Step 498)
6. Switch Reboot พร้อม Config เต็มรูปแบบ — พร้อมใช้งานจริงโดยไม่มีใคร SSH เข้าไปเลย
```

| ขั้นตอนเดิม (Part 2, มือทั้งหมด) | ขั้นตอนผ่าน PnP |
|---|---|
| ต่อ Console, `enable`, `configure terminal` | ไม่ต้องต่อ Console เลย |
| พิมพ์ `hostname`, `ip address`, `enable secret` ทีละคำสั่ง | DNAC Push อัตโนมัติตาม Template ที่เตรียมไว้ |
| ต้องมีคนอยู่หน้าอุปกรณ์ | ต้องมีคนแค่ "เสียบสาย" — Config ทำจากศูนย์กลาง |

### LAN Automation — Automate สิ่งที่ Part 33 ทำด้วยมือ

[Part 38 Step 376](part-038-sd-access-fundamentals.md) แนะนำ **LAN Automation** ไว้แบบสั้นๆ ใน
บริบท SD-Access Underlay — แต่แนวคิดนี้สำคัญพอที่ต้องเจาะลึกอีกครั้งที่นี่ เพราะมันคือฟีเจอร์ที่
**Automate สิ่งที่ [Part 33](part-033-advanced-stp-campus-design.md) สอนให้ทำด้วยมือทั้งหมด**:
การตั้งค่า Routed Port + Loopback0 + OSPF Area 0 ให้กับ Switch ทุกตัวในโครงสร้าง Core/Distribution

```
สิ่งที่ Part 33 สอนให้ทำด้วยมือ (ต่ออุปกรณ์ใหม่ 1 ตัวเข้า Backbone):

CORE-SW1(config)# interface TenGigabitEthernet1/0/3
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.50.1 255.255.255.252
CORE-SW1(config-if)# exit
CORE-SW1(config)# router ospf 1
CORE-SW1(config-router)# network 10.255.50.0 0.0.0.3 area 0
! ... แล้วต้อง SSH เข้า Switch ใหม่ (NEW-DIST-SW5) ทำ Loopback0 + OSPF ซ้ำอีกฝั่ง

สิ่งที่ LAN Automation ทำให้อัตโนมัติ (คลิกเดียวจาก DNAC):

1. เลือก CORE-SW1 เป็น "Seed Device" (มี IP Reachability ไปยัง DNAC อยู่แล้ว)
2. เสียบ NEW-DIST-SW5 (Factory Default) เข้าพอร์ตว่างของ CORE-SW1
3. DNAC สั่ง CORE-SW1 เปิด CDP + DHCP Snooping ชั่วคราว → "เห็น" NEW-DIST-SW5 ทันทีที่เปิดเครื่อง
4. DNAC แจก IP ชั่วคราวผ่าน DHCP, Push Loopback0 + Underlay Routing Protocol (Default: IS-IS แต่
   ปรับเป็น OSPF ได้ให้ตรงกับ Part 33) + IP บน Link ที่เชื่อมกับ Seed Device ให้อัตโนมัติ
5. NEW-DIST-SW5 เข้า Underlay สมบูรณ์ — Route ถึง CORE-SW1 ได้ทันที ไม่มีใคร SSH เข้าไปเลย
```

> **ข้อสังเกตสำคัญสำหรับข้อสอบ**: LAN Automation **Default ใช้ IS-IS** เป็น Underlay Routing
> Protocol (ไม่ใช่ OSPF ที่ Part 33 สอน) — แต่วิศวกรสามารถเลือกเปลี่ยนได้ในบาง Version ของ DNAC
> ข้อสอบชอบถามว่า Protocol Default คือ IS-IS เพื่อทดสอบว่าไม่ได้จำผิดว่าเป็น OSPF ไปโดยอัตโนมัติ

### Fabric Provisioning — ปิดวง SD-Access จาก Part 38

หลังจาก Design (Step 492) กำหนด Site/IP Pool และ Policy (Step 493) กำหนด VN/GBAC เรียบร้อยแล้ว
**Fabric Provisioning** คือขั้นตอนที่วิศวกร **Assign Fabric Role** (Edge, Border, Control Plane
Node ตามที่ [Part 38 Step 372](part-038-sd-access-fundamentals.md) สอนไว้) ให้อุปกรณ์แต่ละตัวผ่าน
การคลิกเลือกบนแผนที่ Topology แล้วกด "Provision Device" — DNAC จะคำนวณและ Push LISP + VXLAN
Config ที่ซับซ้อนทั้งหมดให้อัตโนมัติ ซึ่งเป็น Config ระดับที่ **ไม่มีใครแนะนำให้เขียนด้วยมือ**
เพราะซับซ้อนเกินกว่าจะ Maintain เองในระยะยาว

| ขั้นตอน Provision (GUI) | สิ่งที่เกิดขึ้นเบื้องหลัง |
|---|---|
| เลือก CORE-SW1/2 → Assign "Border Node" | Push Config LISP Border + VXLAN Tunnel Endpoint (VTEP) |
| เลือก DIST-SW1–4 → Assign "Control Plane Node" (ร่วมกับ Border) | Push Config LISP Map-Server/Map-Resolver |
| เลือก ACCESS-SW1–4 → Assign "Edge Node" (ถ้า Hardware รองรับ, ดูข้อจำกัดที่ Part 38 Step 379) | Push Config VXLAN VTEP + SGACL Enforcement |
| กด "Deploy" | DNAC Compile Intent ทั้งหมดเป็น Device Config จริง ส่งผ่าน NETCONF/RESTCONF พร้อมกันทุกตัว |

---

## Step 495 — Assurance: Network Health, Client Health, 360 View, Issues, Path Trace

### Network Health Dashboard

หน้า **Network Health** คือภาพรวมของทั้ง Site ในหน้าเดียว แสดง Health Score (0-10) ของ Device
ทุก Role (Core, Distribution, Access, Wireless) พร้อม Trend ย้อนหลัง 24 ชั่วโมง/7 วัน

```
Network Health Dashboard (เชิงแนวคิด):

  Overall Health Score: 8/10  ▲ (ดีขึ้นจากเมื่อวาน)

  ┌─────────────┬─────────────┬─────────────┬─────────────┐
  │  CORE (2)    │  DIST (4)    │  ACCESS (4)  │  WIRELESS(1)│
  │  Score: 10   │  Score: 9    │  Score: 6 ⚠  │  Score: 10   │
  │  0 issues    │  0 issues    │  1 issue     │  0 issues    │
  └─────────────┴─────────────┴─────────────┴─────────────┘
                                      │
                                      └─ ACCESS-SW3: Interface flapping (ดูรายละเอียด Step 500)
```

Health Score คำนวณจากหลาย Metric รวมกัน (CPU, Memory, Interface Error Rate, Uptime, Reachability
to DNAC) — ต่างจาก `show processes cpu` หรือ `show interfaces` ที่ต้อง SSH เข้าไปดูทีละตัวแบบ
Part 1–37 ตรงที่ **DNAC รวบรวมและให้คะแนนสรุปแทนวิศวกร** โดยอัตโนมัติแบบ Real-time

### Client Health

คล้ายกับ Network Health แต่มองจากมุม **End Device** (PC, Phone, AP Client) แบ่งเป็น Wired/Wireless
แสดง Onboarding Time (เวลาที่ Client ใช้ Authenticate + รับ DHCP กว่าจะ Online เต็มรูปแบบ — เชื่อม
กับ Part 44 dot1x และ Part 16 DHCP) และ Connectivity Score

### 360 View — มุมมองเจาะลึกรายอุปกรณ์/รายไคลเอนต์

เมื่อคลิกเข้าไปที่อุปกรณ์หรือ Client ตัวใดตัวหนึ่ง DNAC จะแสดง **360 View** — Timeline และข้อมูล
ทั้งหมดของตัวนั้นในหน้าเดียว:

| ส่วนของ 360 View | ข้อมูลที่แสดง | เทียบกับคำสั่ง CLI เดิม |
|---|---|---|
| Device Info | Platform, IOS Version, Uptime, Site | `show version` |
| Health Timeline | กราฟ CPU/Memory/Interface Error ย้อนหลัง | `show processes cpu history` (ไม่มีจริงบน IOS ปกติ) |
| Connected Clients | รายชื่อ Client ทั้งหมดที่ต่ออยู่ ณ ขณะนี้ | `show mac address-table`, `show ip dhcp binding` |
| Event/Issue History | เหตุการณ์ผิดปกติที่เคยเกิดกับอุปกรณ์นี้ | `show logging` (แต่จัดกลุ่ม/สรุปให้แล้ว) |
| Physical Topology | ตำแหน่งใน Topology และ Neighbor | `show cdp neighbors` |

### Issues — AI-Driven Root Cause Suggestion

หน้า **Issues** คือรายการปัญหาที่ DNAC ตรวจพบเองจาก Telemetry (SNMP, Syslog, NetFlow, Streaming
Telemetry — แอบดูใน [Part 49 Step 489](part-049-netconf-restconf-yang-deep-dive.md)) โดยไม่ต้อง
รอให้วิศวกรมาเจอปัญหาเอง — จุดเด่นคือ DNAC ใช้ **Machine Learning บน Baseline ปกติของเครือข่าย**
เพื่อ **เสนอ Root Cause ที่เป็นไปได้** ให้ทันที ไม่ใช่แค่แจ้ง Alert เฉยๆ

```
ตัวอย่าง Issue Card บน DNAC:

┌───────────────────────────────────────────────────────────┐
│ ⚠ Issue: Interface Flapping                                   │
│ Device: ACCESS-SW3   Interface: GigabitEthernet1/0/12         │
│ Priority: P2 (High)   First Seen: 09:14   Occurrences: 47      │
│                                                                 │
│ Suggested Root Cause:                                         │
│  "Interface มี Error Rate สูงผิดปกติ ร่วมกับ Duplex Mismatch    │
│   ที่ตรวจพบจาก Counter — ตรวจสอบ Cable และ Speed/Duplex          │
│   Setting ของอุปกรณ์ปลายทาง (ทบทวน Part 1 Step 6)"              │
│                                                                 │
│ [ View Details ]  [ Run Path Trace ]  [ Mark Resolved ]        │
└───────────────────────────────────────────────────────────┘
```

| Priority | ความหมาย |
|---|---|
| P1 | กระทบผู้ใช้จำนวนมากทันที (เช่น Site ทั้ง Site หลุดจาก DNAC) |
| P2 | กระทบเฉพาะจุด แต่มีผลต่อบริการ (เช่น Interface Flapping ที่ Access Layer) |
| P3 | Warning เชิงป้องกัน (เช่น Memory ใกล้เต็ม, ยังไม่กระทบผู้ใช้) |

### Path Trace — Traceroute เวอร์ชันมี Telemetry รายทาง

**Path Trace** คือ Tool ที่ให้วิศวกรเลือก Source/Destination (เช่น PC1 ↔ Server1) แล้ว DNAC จะ
คำนวณเส้นทางจริงที่ Traffic ใช้ ณ ขณะนี้ (อ่านจาก Routing Table/CEF ของทุก Hop จริง ไม่ใช่แค่ยิง
ICMP แบบ `traceroute` ปกติ) พร้อมแสดง **Telemetry ของทุก Hop**: Latency, Interface Utilization,
QoS Queue ที่ Packet ใช้, และ ACL/Policy ที่ Match ระหว่างทาง

| หัวข้อ | `traceroute` (Part 1, CLI ปกติ) | Path Trace (DNAC) |
|---|---|---|
| วิธีทำงาน | ส่ง ICMP TTL Exceeded ทีละ Hop | อ่าน CEF/Routing Table จริงของทุก Hop จาก DNAC Inventory |
| ข้อมูลที่ได้ | แค่ IP และ RTT ของแต่ละ Hop | Latency, Bandwidth Utilization, QoS Class, ACL Match/Drop ต่อ Hop |
| เห็น Drop เพราะ ACL ไหม | ไม่เห็น (Packet แค่ไม่ผ่าน ไม่รู้เหตุผล) | เห็นชัดว่า Drop ที่ Hop ไหน เพราะ ACL/Policy ข้อไหน |
| ตัวอย่างการใช้จริง | Debug เบื้องต้น | ตัวอย่างเต็มรูปแบบดูที่ Step 500 |

---

## Step 496 — Templates & Day-N Configuration

### CLI Template — Jinja2 บน GUI

[Part 48](part-048-automation-ansible-advanced.md) สอนการเขียน **Jinja2 Template** เพื่อ Generate
Config หลายอุปกรณ์จาก Template เดียว — **CLI Template บน DNA Center ใช้ Syntax Jinja2 เดียวกัน
เป๊ะ** แค่เขียนและจัดเก็บผ่าน GUI ของ DNAC แทน Text Editor/Git Repository แล้ว DNAC เป็นผู้ Push
Config ที่ Render แล้วไปยังอุปกรณ์ผ่าน NETCONF/RESTCONF ([Part 49](part-049-netconf-restconf-yang-deep-dive.md))
โดยตรง ไม่ต้องพึ่ง Ansible/Netmiko แยกอีกชั้น

```jinja
! ตัวอย่าง CLI Template บน DNAC (Syntax เหมือน Jinja2 จาก Part 48 ทุกตัวอักษร)
hostname {{hostname}}
!
interface Vlan{{vlan_id}}
 description {{vlan_name}}
 ip address {{ip_address}} {{subnet_mask}}
 standby 1 ip {{hsrp_vip}}
 standby 1 priority {{hsrp_priority}}
!
{% if enable_dhcp_relay %}
interface Vlan{{vlan_id}}
 ip helper-address {{dhcp_server}}
{% endif %}
```

| หัวข้อ | Jinja2 + Ansible (Part 48) | CLI Template (DNAC) |
|---|---|---|
| ที่เก็บ Template | Git Repository / Ansible Role | DNAC Template Editor (มี Version Control ในตัว) |
| แหล่ง Variable | `host_vars`/`group_vars` (YAML) | DNAC Inventory + ค่าที่ใส่ตอน Provision |
| วิธี Push | Ansible Playbook → Netmiko/NETCONF | DNAC เรียก NETCONF/RESTCONF ให้อัตโนมัติ (Part 49) |
| การ Approve ก่อน Push | Manual `ansible-playbook --check` | DNAC "Preview Config Diff" ก่อนกด Deploy จริง |

### Composite Template — รวม Template หลายชุดเข้าด้วยกัน

**Composite Template** คือการเอา CLI Template หลายชุด (เช่น Template VLAN + Template QoS +
Template ACL) มา **เรียงลำดับการ Push** เป็นชุดเดียว รับประกันว่า Config จะถูก Apply ตามลำดับที่
ถูกต้อง (เช่น ต้องสร้าง VLAN ก่อน แล้วค่อย Apply ACL ที่อ้างถึง VLAN นั้น) — เทียบได้กับการเรียก
หลาย Ansible Role ตามลำดับใน 1 Playbook ของ Part 48

### Day-0 vs Day-N Configuration

| ประเภท | ทำเมื่อไหร่ | ตัวอย่าง | เครื่องมือใน DNAC |
|---|---|---|---|
| **Day-0** | ตอน Onboard อุปกรณ์ใหม่ครั้งแรก | Hostname, Management IP, Credential เริ่มต้น | PnP Onboarding Template (Step 494) |
| **Day-N** | หลังอุปกรณ์ Online ใช้งานจริงแล้ว ต้องแก้/เพิ่ม Config ต่อเนื่อง | เพิ่ม VLAN ใหม่, ปรับ QoS Policy, แก้ ACL | CLI Template / Composite Template (Step 496 นี้) |

> **สำหรับข้อสอบ CCNP ENCOR**: คำว่า **"Day-N"** หมายถึงการแก้ Config **หลัง** Onboarding เสร็จแล้ว
> เท่านั้น (ตรงข้ามกับ Day-0 ที่หมายถึงตอน Onboard ครั้งแรก) — เป็นคำที่ Blueprint ใช้ตรงตัว

---

## Step 497 — DNA Center Intent API: Northbound REST API

### แนวคิด Northbound API

ทุก Workflow ที่บรรยายมาใน Step 491–496 เป็นการใช้งานผ่าน **GUI** — แต่ DNA Center เองก็เปิด
**Northbound REST API (เรียกว่า Intent API)** ให้ระบบภายนอก (Script Python, ServiceNow, CI/CD
Pipeline จาก [Part 48](part-048-automation-ansible-advanced.md)) เรียกใช้ Workflow เดียวกันได้โดย
ไม่ต้องเปิด GUI เลย — นี่คือจุดที่ Skill จาก [Part 47](part-047-automation-python-advanced.md)
(Python + `requests`) และ [Part 49](part-049-netconf-restconf-yang-deep-dive.md) (RESTCONF
concept) มาบรรจบกันพอดี: **Syntax การเรียก REST API คล้ายกันมาก แต่ปลายทางเปลี่ยนจาก "อุปกรณ์
ตัวเดียว" เป็น "DNA Center ที่ควบคุมทั้งเครือข่าย"**

```
Southbound API (Part 49):     Python Script ──RESTCONF──► Router ตัวเดียว (1:1)

Northbound API (Part นี้):    Python Script ──REST API──► DNA Center ──(ควบคุมภายใน)──►
                                                              ทุกอุปกรณ์ในองค์กร (1:many)
```

### ขั้นตอนพื้นฐาน: Authentication + เรียก Intent API

DNA Center Intent API ใช้ **Token-based Authentication** (คล้าย OAuth) — ต้อง Authenticate ก่อน
เพื่อรับ Token แล้วใส่ Token นั้นใน Header ของ Request ถัดไปทุกครั้ง

```bash
# Step 1: Authenticate เพื่อรับ Token (Basic Auth ครั้งเดียว)
curl -s -k -X POST \
  "https://dnac.company.local/dna/system/api/v1/auth/token" \
  -H "Content-Type: application/json" \
  -u "netadmin:P@ssw0rd123" \
  | jq -r '.Token'

# ตัวอย่าง Response:
# { "Token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9...." }
```

```bash
# Step 2: เรียก Intent API — ตัวอย่าง get-device-list (ดึงรายชื่ออุปกรณ์ทั้งหมดใน Inventory)
curl -s -k -X GET \
  "https://dnac.company.local/dna/intent/api/v1/network-device" \
  -H "X-Auth-Token: eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9...."
```

```python
# ตัวอย่าง Python (ต่อจาก Skill Part 47 — ใช้ requests เหมือน Part 49 Step 487)
import requests
requests.packages.urllib3.disable_warnings()

DNAC = "https://dnac.company.local"

def get_token():
    resp = requests.post(
        f"{DNAC}/dna/system/api/v1/auth/token",
        auth=("netadmin", "P@ssw0rd123"),
        verify=False,
    )
    return resp.json()["Token"]

def get_device_list(token):
    headers = {"X-Auth-Token": token}
    resp = requests.get(
        f"{DNAC}/dna/intent/api/v1/network-device",
        headers=headers, verify=False,
    )
    return resp.json()["response"]

token = get_token()
devices = get_device_list(token)
for d in devices:
    print(f"{d['hostname']:20s} {d['managementIpAddress']:16s} {d['reachabilityStatus']}")

# ตัวอย่าง Output ที่คาดหวัง (จาก Lab หลักสูตรนี้ ถ้า Adopt เข้า DNAC ตาม Step 500):
# CORE-SW1             10.10.99.11      Reachable
# CORE-SW2             10.10.99.12      Reachable
# DIST-SW1             10.10.99.21      Reachable
# DIST-SW2             10.10.99.22      Reachable
```

### ตัวอย่าง Intent API อื่นที่สำคัญ (Reference)

| Intent API Endpoint | ใช้ทำอะไร | เชื่อมกับ Workflow ใน Part นี้ |
|---|---|---|
| `GET /dna/intent/api/v1/network-device` | ดึงรายชื่อ + สถานะอุปกรณ์ทั้งหมด | Provision (Step 494) |
| `GET /dna/intent/api/v1/network-health` | ดึง Health Score รวม | Assurance (Step 495) |
| `POST /dna/intent/api/v1/flow-analysis` | สั่ง Path Trace ผ่าน API (ไม่ต้องคลิก GUI) | Assurance Path Trace (Step 495) |
| `POST /dna/intent/api/v1/template-programmer/project/{id}/template` | Deploy CLI Template ผ่าน API | Templates (Step 496) |
| `POST /dna/intent/api/v1/pnp-device` | เพิ่ม Planned Device เข้า PnP ผ่าน API | Provision PnP (Step 494) |

> **สำหรับข้อสอบ CCNP ENCOR**: จำ 2-Step Pattern ให้แม่น — **(1) Authenticate เพื่อรับ Token
> จาก `/dna/system/api/v1/auth/token`** → **(2) ใส่ Token ใน Header `X-Auth-Token` ทุก Request
> ถัดไปที่เรียก `/dna/intent/api/...`** — Path ที่มีคำว่า **"intent"** คือจุดที่แยก Northbound API
> (คุมทั้งเครือข่าย) จาก Southbound API อย่าง RESTCONF ของ Part 49 (คุมอุปกรณ์เดียว) อย่างชัดเจน

---

## Step 498 — Software Image Management (SWIM)

### ปัญหาเดิมจาก Part 2: Manual Image Upgrade ทีละอุปกรณ์

[Part 2](part-002-ios-cli-basics.md) สอนคำสั่งพื้นฐานของการอัปเกรด IOS: `copy tftp: flash:` แล้ว
ตั้ง `boot system flash:...` ใหม่ ทำทีละอุปกรณ์ — ถ้าองค์กรมี Switch 100 ตัวที่ต้อง Upgrade ตัว
เดียวกัน วิศวกรต้องทำ 100 รอบ **แต่ละรอบเสี่ยง Human Error** และไม่มีทางรับประกันว่าทุกอุปกรณ์
ได้ Version เดียวกันจริง

### SWIM Workflow: Golden Image + Scheduled Upgrade

**Software Image Management (SWIM)** คือ Feature ของ DNA Center ที่จัดการ Lifecycle ของ IOS-XE
Image ทั้งองค์กรจากจุดเดียว

```
SWIM Workflow (แนวคิด):

1. Import Image เข้า DNAC (ครั้งเดียว) — Upload จากไฟล์ หรือ Download ตรงจาก Cisco.com ผ่าน DNAC
2. Mark Image เป็น "Golden Image" สำหรับ Platform นั้น
   เช่น "IOS-XE 17.12.4 = Golden Image สำหรับ Catalyst 9300 ทุกตัว"
3. DNAC จะแจ้งเตือนทุกอุปกรณ์ที่ยัง "ไม่ตรงกับ Golden Image" ว่า Non-Compliant
4. เลือกกลุ่มอุปกรณ์ (เช่น DIST-SW1-4 ทั้ง 4 ตัวพร้อมกัน) → กด "Upgrade"
5. ตั้ง Schedule (เช่น "Upgrade คืนวันเสาร์ 02:00 น. เพื่อลด Impact ต่อผู้ใช้")
6. DNAC ทำ Distribute (โอน Image ไปเก็บที่ Flash ของอุปกรณ์ล่วงหน้า) แยกจาก Activate
   (Reboot จริง) — ลด Downtime ช่วง Maintenance Window ให้สั้นที่สุด
7. DNAC ทำทีละ Batch ตาม Concurrency ที่ตั้งไว้ (เช่น Upgrade DIST-SW1/DIST-SW3 ก่อน, รอเช็ค Health
   ผ่านแล้วค่อยทำ DIST-SW2/DIST-SW4 เพื่อไม่ให้ Redundancy Pair ล้มพร้อมกัน)
```

| หัวข้อ | Manual (`copy tftp: flash:`, Part 2) | SWIM (DNAC) |
|---|---|---|
| จำนวนอุปกรณ์ต่อรอบ | 1 ตัว | หลายสิบ/หลายร้อยตัวพร้อมกัน (เป็น Batch) |
| ตรวจสอบ Compliance | ต้อง `show version` เทียบมือทีละตัว | DNAC ขึ้น Dashboard "Compliant / Non-Compliant" อัตโนมัติ |
| แยก Distribute/Activate | ไม่มี (โอนไฟล์แล้ว Reboot ทันที) | แยกชัด — ลด Maintenance Window |
| Rollback ถ้า Upgrade ล้มเหลว | ต้องทำมือ (`copy` Image เดิมกลับ) | DNAC มี Automatic Rollback ถ้า Activate ไม่สำเร็จ |
| กำหนดเวลา | ไม่มี (ทำตอนนั้นเลย) | ตั้ง Schedule ล่วงหน้าได้ (Maintenance Window) |

> **สำหรับข้อสอบ CCNP ENCOR**: จำคำ 2 คำนี้ให้แม่น — **Distribute** (โอนไฟล์ Image ไปไว้ที่อุปกรณ์
> ล่วงหน้า ยังไม่ Reboot) และ **Activate** (สั่ง Boot ด้วย Image ใหม่จริง ทำให้เกิด Downtime) —
> การแยก 2 ขั้นนี้คือประโยชน์หลักของ SWIM ที่ Manual Process ทำไม่ได้

---

## Step 499 — Multivendor, SD-WAN/Meraki Integration, และ Licensing Model

### วิวัฒนาการของ Platform: จาก Cisco-only สู่ Multivendor

DNA Center เดิมออกแบบมาเพื่อ Cisco Catalyst/IOS-XE เป็นหลัก แต่ในการ Rebrand เป็น **Catalyst
Center** พร้อมการรวมเข้ากับ **Cisco Networking Cloud** ทิศทางของ Cisco ได้ขยายไปสู่การรองรับ
อุปกรณ์ Third-Party (Multivendor) มากขึ้นเรื่อยๆ ผ่านกลไกดังนี้:

| กลไก | รายละเอียด |
|---|---|
| **SNMP-based Discovery** | อุปกรณ์ Third-Party ที่รองรับ SNMP มาตรฐาน สามารถถูก Discover เข้า Inventory ของ DNAC ได้ในระดับ Visibility พื้นฐาน (เห็น Status, Interface) แต่ **ไม่สามารถทำ Provision/Fabric Role** แบบอุปกรณ์ Cisco เต็มรูปแบบ |
| **Telemetry Ingestion แบบเปิด** | DNAC (และ Cisco Networking Cloud ที่กว้างกว่า) เริ่มรับ Streaming Telemetry จากอุปกรณ์ Third-Party ผ่าน Standard Model (OpenConfig — ทบทวนจาก [Part 49 Step 482](part-049-netconf-restconf-yang-deep-dive.md)) เพื่อขึ้น Dashboard Assurance ร่วมกันได้ |
| **API-first Integration** | ผ่าน Intent API (Step 497) ทำให้ระบบ Third-Party (เช่น ServiceNow, Splunk) เชื่อมต่อแบบ Bi-directional ได้ |

### SD-WAN และ Meraki Integration

Cisco มี Controller หลายตัวสำหรับ Domain ต่างกัน — ทิศทางปัจจุบันคือรวม Visibility ของทุก
Controller เข้า **Cisco Networking Cloud** เดียว โดย DNA Center/Catalyst Center ยังคงเป็นศูนย์กลาง
ของ Campus/Branch (Wired/Wireless) เป็นหลัก

| Controller | Domain ที่ดูแล | ความสัมพันธ์กับ DNA Center |
|---|---|---|
| **DNA Center / Catalyst Center** | Campus LAN, Wireless (WLC/AP) | ตัวหลักของ Part นี้ |
| **Cisco SD-WAN (vManage)** | WAN Edge Router ([Part 39](part-039-sd-wan-fundamentals.md)) | Controller แยก แต่ Telemetry รวมศูนย์ผ่าน Cisco Networking Cloud/ThousandEyes ได้ |
| **Cisco Meraki Dashboard** | Meraki-specific Hardware (MX, MS, MR) | Cloud-native Controller แยกต่างหาก แต่ทิศทาง Cisco คือให้ข้อมูล Assurance ไหลมารวมที่ Cisco Networking Cloud เดียวกัน |

> **สำหรับข้อสอบ CCNP ENCOR**: ข้อสอบปัจจุบันเน้นให้เข้าใจว่า **DNA Center/Catalyst Center,
> SD-WAN (vManage), และ Meraki Dashboard เป็น Controller แยกกันคนละ Domain** — ไม่ใช่ Platform
> เดียวที่ทำทุกอย่าง แต่ Cisco มีทิศทางระยะยาวรวม Visibility เข้าด้วยกันผ่าน Cisco Networking Cloud

### Licensing Model: DNA Essentials / Advantage / Premier

Cisco ขาย License ของ Catalyst/DNA Center แยกเป็นระดับ (Tier) — License นี้ผูกกับ **ตัวอุปกรณ์
Cisco แต่ละตัว** (ไม่ใช่ผูกกับ DNAC เอง) และเป็นตัวกำหนดว่า **Feature ไหนใช้ได้บ้างบน DNAC สำหรับ
อุปกรณ์ตัวนั้น**

| Tier | Feature หลักที่ปลดล็อก |
|---|---|
| **DNA Essentials** (ระดับพื้นฐาน) | Day-0/Day-N Provisioning, Software Image Management (SWIM), Network Health พื้นฐาน |
| **DNA Advantage** | เพิ่ม SD-Access Fabric เต็มรูปแบบ, Advanced Assurance (AI-Driven Issues, Path Trace เต็มรูปแบบ), Application Policy (QoS) |
| **DNA Premier / Cisco Networking Cloud Premier** (ชื่อ Bundle เปลี่ยนตามยุค) | รวม Security Feature เพิ่มเติม (เช่น Cisco ISE Advantage, Umbrella Integration) |

> **ข้อสังเกตสำคัญ**: ฟีเจอร์ **SD-Access Fabric (Step 494 ส่วน Fabric Provisioning)** และ
> **GBAC/VN Policy (Step 493)** ต้องมี License **DNA Advantage เป็นอย่างน้อย** — ถ้าองค์กรซื้อแค่
> Essentials จะใช้ได้แค่ Design/Provision/SWIM พื้นฐาน ไม่สามารถทำ SD-Access ได้ ข้อสอบมักถามใน
> เชิง "องค์กรต้องการ Feature X ต้องซื้อ License Tier ใดเป็นอย่างน้อย"

---

## Step 500 — Lab เชิงแนวคิดเต็มรูปแบบ: แผน Adoption DNA Center สำหรับ Lab Part 1–49

Lab นี้เป็น **Lab เชิงแนวคิด/ระดับออกแบบ (Conceptual/Design-Level)** ไม่ใช่ CLI Lab แบบ Part
ก่อนๆ — เป้าหมายคือแสดงแผนการ **Adopt DNA Center** เข้ากับ Lab หลักของหลักสูตรที่สร้างมาตั้งแต่
Part 1 อย่างเป็นระบบ ครบทั้ง 4 Workflow

### ส่วนที่ 1 — Design: Site Hierarchy ตรงกับ Topology จริง

```
Global
  └── Area: Thailand
        └── Site: Bangkok HQ
              ├── Floor: Core & Distribution Room
              │     - CORE-SW1 (1.1.1.1), CORE-SW2 (1.1.1.2)
              │     - DIST-SW1 (1.1.1.11), DIST-SW2 (1.1.1.12)
              │     - DIST-SW3 (1.1.1.13), DIST-SW4 (1.1.1.14)
              │     - WAN-EDGE-1 (1.1.1.21), WAN-EDGE-2 (1.1.1.22)
              ├── Floor: Sales & Voice Zone
              │     - ACCESS-SW1, ACCESS-SW2
              │     - PC1-PC5 (VLAN 10 SALES), Phone/AP (VLAN 20 VOICE)
              └── Floor: Server & Wireless Zone
                    - ACCESS-SW3, ACCESS-SW4
                    - Server1-3 (VLAN 30 SERVERS), WLC-1 + AP-1/AP-2 (VLAN 40 WIFI)
```

**IP Pool ที่ Reserve ให้ตรงกับ [`00-ip-address-plan.md`](00-ip-address-plan.md) เป๊ะ ไม่เปลี่ยนแม้
Subnet เดียว**:

| IP Pool | Subnet | Reserve ที่ Site/Floor |
|---|---|---|
| `SALES_POOL` | 10.10.10.0/24 | Sales & Voice Zone |
| `VOICE_POOL` | 10.10.20.0/24 | Sales & Voice Zone |
| `SERVERS_POOL` | 10.10.30.0/24 | Server & Wireless Zone |
| `WIFI_POOL` | 10.10.40.0/24 | Server & Wireless Zone |
| `MGMT_POOL` | 10.10.99.0/24 | ทุก Floor (Global Pool) |

### ส่วนที่ 2 — Provision: Onboard CORE-SW1/2 และ DIST-SW1–4 ผ่าน PnP

```
แผน PnP Onboarding (แนวคิด):

1. Pre-provision ใน DNAC: เพิ่ม CORE-SW1, CORE-SW2, DIST-SW1-4 เป็น "Planned Device"
   ระบุ Serial Number จริงของแต่ละตัว + Site ที่จะ Assign (Core & Distribution Room)

2. ตั้ง DHCP Option 43 บน DHCP Server ของ VLAN 99 (MGMT) ให้ชี้ไปที่ DNAC IP
   (10.10.99.254 สมมติเป็น DNAC Management IP)

3. เปิดเครื่อง CORE-SW1/2, DIST-SW1-4 (Factory Default ตามสมมติฐาน — ในทางปฏิบัติของหลักสูตรนี้
   อุปกรณ์เหล่านี้มี Config อยู่แล้วจาก Part 1-49 ดังนั้นแผนจริงคือใช้ "Day-N Import"
   แทน PnP เต็มรูปแบบ — PnP เหมาะกับ Greenfield Deployment เท่านั้น)

4. DNAC Discover อุปกรณ์ผ่าน Serial Number → จับคู่กับ Planned Device → Push Credential
   Profile ที่เตรียมไว้ที่ Step 492 → Assign เข้า Site ที่ถูกต้อง

5. ตรวจสอบผลผ่าน Provision > Inventory: ทุกอุปกรณ์ควรแสดงสถานะ "Reachable" และ
   "Managed" ครบทั้ง 6 ตัว
```

> **ข้อสังเกตสำคัญ**: เพราะ Lab หลักของหลักสูตรนี้ Config มาแล้วตั้งแต่ Part 2–37 ด้วยมือ (ไม่ใช่
> Greenfield) ขั้นตอนที่ถูกต้องในทางปฏิบัติคือ **"Inventory Import"** (DNAC Discover อุปกรณ์ที่มี
> Config อยู่แล้วผ่าน SNMP/SSH Credential แล้วดึง Config ปัจจุบันเข้ามาอ่าน) แทนที่จะ PnP แบบ
> Factory Default เต็มรูปแบบ — PnP เหมาะกับอุปกรณ์ใหม่ที่ยังไม่เคย Config เท่านั้น

### ส่วนที่ 3 — Assurance Dashboard Walkthrough: จำลองปัญหา Interface Flapping

สมมติสถานการณ์: หลัง Adopt เข้า DNAC 1 สัปดาห์ ระบบ Assurance ตรวจพบปัญหาที่ **ACCESS-SW3**
(อุปกรณ์ที่ Server1-3 เชื่อมต่ออยู่ ตาม Access-Layer Mapping ใน [`00-ip-address-plan.md`](00-ip-address-plan.md))

```
Timeline การ Troubleshoot ผ่าน DNAC Assurance:

09:14  Network Health Dashboard: ACCESS Layer Score ตกจาก 10 → 6
       └─► คลิกเข้าไปดู ACCESS-SW3 โดยตรง

09:15  360 View ของ ACCESS-SW3:
       - Interface GigabitEthernet1/0/12 (พอร์ตที่ต่อกับ DIST-SW3 หรือ Server1)
       - Event Log แสดง "link-up / link-down" สลับกันถี่ผิดปกติ (47 ครั้งใน 10 นาที)

09:16  หน้า Issues แสดง Issue Card:
       "Interface Flapping — GigabitEthernet1/0/12 บน ACCESS-SW3"
       Priority: P2
       Suggested Root Cause: "Physical Layer Error Rate สูงผิดปกติ ตรวจพบ CRC Error เพิ่มขึ้น
       ต่อเนื่อง — อาจเกิดจากสาย Fiber/UTP เสีย หรือ Transceiver ผิดปกติ (ทบทวน Part 1 Step 6)"

09:18  วิศวกรกด [Run Path Trace] จาก Issue Card เพื่อดูผลกระทบจริงต่อ Traffic
       (รายละเอียด Path Trace ที่ส่วนที่ 4 ด้านล่าง)

09:25  ส่งช่างไปเปลี่ยนสาย/Transceiver ตามคำแนะนำ — Interface กลับมา Stable
       Network Health Score ของ ACCESS Layer กลับขึ้นเป็น 10 ภายใน 5 นาทีถัดมา
```

เทียบกับวิธี Manual แบบ Part 1–37: วิศวกรต้อง SSH เข้า ACCESS-SW3 เอง พิมพ์ `show interfaces
GigabitEthernet1/0/12` และ `show logging | include UPDOWN` ไล่ดูเอง ถึงจะรู้ว่ามี Flapping —
DNAC ทำให้ **รู้ก่อนที่จะมีคนร้องเรียน** เพราะ Telemetry ไหลเข้า DNAC ต่อเนื่องอยู่แล้ว

### ส่วนที่ 4 — Path Trace: PC1 → Server1 พร้อม QoS/Latency รายทาง

```
Path Trace Request: Source = PC1 (10.10.10.5)   Destination = Server1 (10.10.30.10)

ผลลัพธ์ (เชิงแนวคิด):

┌─────────┐   ┌────────────┐   ┌──────────┐   ┌──────────┐   ┌───────────┐
│  PC1     │──►│ ACCESS-SW1  │──►│ DIST-SW1  │──►│ CORE-SW1  │──►│ CORE-SW2   │
│10.10.10.5│   │(L2 Access)  │   │(1.1.1.11) │   │(1.1.1.1)  │   │(1.1.1.2)   │
└─────────┘   └────────────┘   └──────────┘   └──────────┘   └───────────┘
                   Latency:2ms    Latency:1ms    Latency:1ms    Latency:1ms
                   Util: 12%      Util: 8%       Util: 15%      Util: 18%
                   QoS: n/a       Queue: Default  Queue: Default Queue: Default
                                                                       │
                                                                       ▼
                                                              ┌────────────┐   ┌─────────┐
                                                              │ DIST-SW3    │──►│ACCESS-SW3│──► Server1
                                                              │(1.1.1.13)   │   │          │  10.10.30.10
                                                              └────────────┘   └─────────┘
                                                              Latency:1ms      Latency:2ms
                                                              Util: 10%        Util: 9%
                                                              Queue: Default    QoS: n/a

Total End-to-End Latency: ~8ms      ACL/Policy Match: ไม่มี Deny ระหว่างทาง (Permit ตลอดเส้นทาง)
Path Status: ✅ Healthy — ไม่พบ Hop ที่ Utilization เกิน Threshold หรือ Error Rate ผิดปกติ
```

| จุดสังเกตจากผล Path Trace | ความหมาย |
|---|---|
| เส้นทางผ่าน CORE-SW1 → CORE-SW2 | ยืนยันว่า Traffic ข้าม Core ผ่าน Port-channel1 (Part 6/33) ตามที่ออกแบบ — Single-homed DIST↔CORE ตาม [`00-ip-address-plan.md`](00-ip-address-plan.md) หมายเหตุ Redundancy |
| Queue: Default ทุก Hop | ไม่มี QoS Policy พิเศษสำหรับ Traffic ประเภทนี้ (ถ้าเป็น VoIP ควรเห็น Queue: Priority ตาม Part 40) |
| Latency รวม ~8ms | ปกติสำหรับ Traffic ภายใน Campus เดียวกัน (เทียบกับ WAN ที่ Part 39 ที่มี Latency สูงกว่ามาก) |
| ไม่มี ACL Deny ระหว่างทาง | ยืนยันว่า Server1 (VLAN 30 SERVERS) เข้าถึงได้จาก PC1 (VLAN 10 SALES) ตาม Policy ที่ตั้งใจไว้ |

> ถ้าในอนาคตมีการเพิ่ม SGACL (Step 493) ที่ห้าม `CONTRACTOR` เข้าถึง `SERVERS_GRP` และ Path Trace
> ถูกรันจากอุปกรณ์ที่อยู่ในกลุ่ม `CONTRACTOR` แทน PC1 ผลลัพธ์จะแสดง **"Path Status: ❌ Blocked at
> Hop: ACCESS-SW1 — Denied by SGACL"** ทันที ซึ่งเป็นข้อมูลที่ `traceroute` ธรรมดาไม่มีทางบอกได้เลย

---

## แบบฝึกหัดทวนความเข้าใจ Part 50

1. 4 Workflow หลักของ DNA Center คืออะไร เรียงลำดับที่ถูกต้อง และทำไมต้องเรียงลำดับนี้เท่านั้น?
2. LAN Automation Default ใช้ Underlay Routing Protocol ตัวใด และต่างจาก Protocol ที่
   [Part 33](part-033-advanced-stp-campus-design.md) สอนไว้อย่างไร?
3. Path Trace ให้ข้อมูลอะไรที่ `traceroute` แบบดั้งเดิมให้ไม่ได้ อย่างน้อย 2 ข้อ?
4. SWIM แยกขั้นตอน Distribute และ Activate ออกจากกันเพื่อประโยชน์อะไร?
5. Virtual Network (VN) Policy ใช้ได้กับเครือข่ายแบบใดเท่านั้น และต้องมี License Tier ใดเป็น
   อย่างน้อยจึงจะใช้ Feature นี้ได้?

**เฉลย:**

1. **Design → Policy → Provision → Assurance** — ต้องเรียงลำดับนี้เพราะ Provision ต้องมี IP
   Pool/Site (จาก Design) และ Policy (VN/GBAC) พร้อมอยู่ก่อนจึงจะผูก Config ให้อุปกรณ์ได้ถูกต้อง
   ส่วน Assurance เป็นการ Monitor ผลลัพธ์ของสิ่งที่ Provision ไปแล้ว จึงต้องมาหลังสุดเสมอ
2. LAN Automation **Default ใช้ IS-IS** เป็น Underlay Routing Protocol ต่างจาก Part 33 ที่สอน
   **OSPF Area 0** — ถ้าต้องการให้ LAN Automation ใช้ OSPF ให้ตรงกับ Underlay เดิมของหลักสูตร
   ต้องเลือกปรับ Setting นี้ตอน Config LAN Automation Workflow โดยเฉพาะ ไม่ใช่ค่า Default
3. Path Trace ให้ (ก) **Telemetry รายทาง** เช่น Latency, Interface Utilization, QoS Queue ที่
   Packet ใช้จริงในแต่ละ Hop และ (ข) **เหตุผลที่ Packet ถูก Drop** เช่นเจอ ACL/SGACL Deny ที่ Hop
   ไหน ซึ่ง `traceroute` แบบเดิมเห็นได้แค่ IP และ RTT ของแต่ละ Hop ไม่รู้เหตุผลเบื้องหลังเลย
4. แยกเพื่อ **ลด Maintenance Window (Downtime)** — Distribute (โอนไฟล์ Image ไปเก็บที่ Flash ของ
   อุปกรณ์ล่วงหน้า) ทำได้ตอนกลางวันโดยไม่กระทบผู้ใช้เลย ส่วน Activate (Reboot ด้วย Image ใหม่จริง)
   ทำแค่ช่วงสั้นๆ ใน Maintenance Window ที่ตั้งเวลาไว้เท่านั้น ลดเวลาที่อุปกรณ์ต้อง Offline จริง
5. Virtual Network (VN) Policy ใช้ได้ **เฉพาะเครือข่ายที่ Provision เป็น SD-Access Fabric แล้ว
   เท่านั้น** (ต้องมี Fabric Border/Edge/Control Plane Node ตาม Part 38) ใช้กับ Traditional
   3-Tier Network ไม่ได้ และต้องมี License **DNA Advantage เป็นอย่างน้อย** จึงจะใช้ Feature
   SD-Access Fabric/VN Policy ได้ (DNA Essentials รองรับแค่ Design/Provision/SWIM พื้นฐาน)

---

## สรุป Part 50

Part นี้เจาะลึก **Cisco DNA Center / Catalyst Center** ในฐานะ **"Single Pane of Glass"** ที่รวม
การจัดการเครือข่ายทั้งองค์กรไว้ในหน้าเดียว ครอบคลุมทั้ง 4 Workflow หลัก: **Design** (Network
Hierarchy, IP Pool, Credential, Image ที่ Mapping ตรงกับ [`00-ip-address-plan.md`](00-ip-address-plan.md)),
**Policy** (GBAC ที่เป็น GUI ของ TrustSec/SGT จาก [Part 44](part-044-dot1x-trustsec.md),
Application Policy ที่เป็น GUI ของ QoS/MQC จาก [Part 40](part-040-advanced-qos.md), และ Virtual
Network สำหรับ SD-Access จาก [Part 38](part-038-sd-access-fundamentals.md)), **Provision**
(Plug and Play, LAN Automation ที่ Automate สิ่งที่ [Part 33](part-033-advanced-stp-campus-design.md)
ทำด้วยมือ, และ Fabric Provisioning), และ **Assurance** (Network/Client Health, 360 View, AI-Driven
Issues, และ Path Trace) เราต่อยอดไปถึง **Templates** (Jinja2-based CLI Template ต่อจาก
[Part 48](part-048-automation-ansible-advanced.md)), **Intent API** (Northbound REST API ที่ใช้
Pattern การ Authenticate + เรียก API แบบเดียวกับ [Part 47](part-047-automation-python-advanced.md)/
[Part 49](part-049-netconf-restconf-yang-deep-dive.md) แต่ควบคุมทั้งเครือข่ายพร้อมกัน), **SWIM**
(Golden Image + Scheduled Upgrade ที่แก้ปัญหา Manual Upgrade จาก [Part 2](part-002-ios-cli-basics.md)),
และปิดท้ายด้วยภาพรวม **Multivendor/SD-WAN/Meraki Integration และ Licensing Tier** (Essentials/
Advantage/Premier) ก่อนจบด้วย **Lab เชิงแนวคิดเต็มรูปแบบ**: แผน Adopt DNA Center เข้ากับ Lab
Part 1–49 ทั้ง Site Hierarchy, IP Pool, PnP Onboarding, Assurance Walkthrough (จำลอง Interface
Flapping ที่ ACCESS-SW3), และ Path Trace จาก PC1 ไปยัง Server1

**ประเด็นสำคัญที่ต้องจำ**: DNA Center **ไม่ได้สร้างเทคโนโลยีใหม่** — มันคือ **Layer ของ
Automation/Orchestration/Analytics ที่ครอบอยู่บนเทคโนโลยีเดิมทั้งหมดที่เรียนมาตั้งแต่ Part 1**
(VLAN, OSPF, QoS, TrustSec, SD-Access, NETCONF/RESTCONF) โดยเปลี่ยนวิธี "สั่งงาน" จาก CLI ทีละ
บรรทัดเป็น GUI Workflow/API ที่ควบคุมทั้งองค์กรพร้อมกัน — Part ถัดไปจะพาเรากลับไปที่ CLI อีกครั้ง
เพื่อเรียนเทคนิคขั้นสูงของการทำ **Multi-tenancy บนอุปกรณ์ Router/Switch โดยตรง**: **VRF-Lite,
GRE Tunnel, และ IPsec** — สามเทคนิคพื้นฐานที่ยังต้องเขียนด้วยมือ ก่อนที่จะเจาะลึก MPLS L3VPN และ
Site-to-Site VPN เต็มรูปแบบในกลุ่ม Part ถัดไป

**ไปต่อ:** [Part 51 — VRF-Lite, GRE Tunnel & IPsec →](part-051-vrf-lite-gre-ipsec.md)
