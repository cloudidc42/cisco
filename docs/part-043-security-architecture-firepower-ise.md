# Part 43 — Security Architecture (Firepower, ISE Intro)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 421–430 จาก 1000**

> ต่อจาก [Part 42 — Multicast Advanced →](part-042-multicast-advanced.md) ที่ปิดท้ายเรื่อง IP
> Multicast เชิงลึกไปแล้ว Part นี้จะพาเรากลับไปสะสางของค้างสำคัญ 2 เรื่องที่หลักสูตรนี้ปูทางไว้
> ตั้งแต่ต้น:
>
> 1. **FW-1** — ตั้งแต่ [Part 1](part-001-networking-fundamentals.md) เราวาง FW-1 (ASAv/FTDv) ไว้
>    ที่ตำแหน่ง Edge ระหว่าง WAN-EDGE-1/2 กับ CORE-SW1/2 ในทุก Diagram Topology ของหลักสูตร แต่ยัง
>    ไม่เคย Config อะไรกับมันจริงจังเลยแม้แต่บรรทัดเดียว — Part นี้คือจุดที่เรา "เปิดกล่อง" FW-1
>    และแปลงจาก ASA แนวคิดเดิมไปเป็น **Firepower Threat Defense (FTD)** ที่บริหารผ่าน
>    **Firepower Management Center (FMC)** เต็มรูปแบบ
> 2. **Cisco ISE** — [Part 22](part-022-aaa-device-hardening.md) สร้าง **AAA-SRV** (10.10.99.60)
>    ไว้เป็น TACACS+ Server แบบ Generic สำหรับ Device Administration, [Part 37]
>    (part-037-wireless-security-advanced.md) เปิด RADIUS บน Server ตัวเดิมสำหรับ 802.1X/Wireless
>    และพูดถึง **Cisco ISE** ไว้เป็น Preview สั้นๆ 2 ครั้ง (Step 365 เรื่อง PKI/EAP-TLS, และ
>    "รายละเอียดเต็มรูปแบบอยู่ที่ Part 43/78") — Part นี้คือจุดที่เราเปิด ISE เต็มรูปแบบเป็นครั้งแรก
>    ในฐานะ **Network Access Control (NAC) Platform** ที่ใหญ่กว่า RADIUS ธรรมดามาก
>
> ทั้งสองเรื่องนี้คือ**รากฐาน**ที่ [Part 44 — 802.1X & TrustSec →](part-044-dot1x-trustsec.md) จะ
> ต่อยอดทันที: ISE Policy Set ที่เราออกแบบใน Part นี้จะถูกใช้จริงกับ 802.1X แบบ Wired เต็มรูปแบบ
> และ SGT/TrustSec (ที่ [Part 38](part-038-sd-access-fundamentals.md) Preview ไว้ว่า "รายละเอียด
> เชิงลึกเต็มรูปแบบอยู่ที่ Part 44") จะถูก Distribute ผ่าน ISE ตัวเดียวกันนี้เอง

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 421 | ภาพรวม Enterprise Security Architecture — Defense-in-Depth หลายชั้น และตำแหน่งของ FW-1/ISE/TrustSec |
| 422 | สถาปัตยกรรม Cisco Firepower NGFW — FTD vs ASA, FMC vs FDM |
| 423 | FTD Deployment Mode — Routed vs Transparent และตำแหน่งของ FW-1 ใน Topology |
| 424 | พื้นฐาน FTD Security Policy — Access Control Policy, Intrusion Prevention, File/Malware Policy |
| 425 | Cisco ISE พื้นฐาน — จาก RADIUS ธรรมดาไปสู่ Full NAC |
| 426 | สถาปัตยกรรม ISE Deployment — PAN, MnT, PSN และ Standalone vs Distributed |
| 427 | ISE Authentication & Authorization Policy — Policy Set เต็มรูปแบบแทน AAA-SRV |
| 428 | ISE Profiling และ Posture Assessment |
| 429 | ISE Guest และ BYOD Workflow |
| 430 | Lab: ออกแบบสถาปัตยกรรม Security เต็มรูปแบบ FTD+FMC+ISE |

---

## Step 421 — ภาพรวม Enterprise Security Architecture: Defense-in-Depth หลายชั้น

### ทบทวนแนวคิด Defense-in-Depth ที่หลักสูตรนี้ใช้มาตลอด

หลักสูตรนี้ใช้แนวคิด **Defense-in-Depth** (ความปลอดภัยหลายชั้นซ้อนกัน) มาตั้งแต่ [Part 14]
(part-014-access-control-lists.md) (ACL), [Part 21](part-021-switch-security.md) (Port Security/
DHCP Snooping/DAI), และ [Part 22](part-022-aaa-device-hardening.md) (AAA/Device Hardening) —
แต่ทุก Part ที่ผ่านมาโฟกัสแค่**ชั้นเดียว**ในแต่ละครั้ง Part นี้จะยกภาพรวมทั้งหมดขึ้นมาดูพร้อมกัน
เป็นครั้งแรก เพื่อให้เห็นว่าแต่ละเทคโนโลยีที่เรียนมาแล้ว **อยู่ตรงไหนของภาพใหญ่**

### 5 ชั้นของ Enterprise Security Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  LAYER 5 — MONITORING & RESPONSE                                          │
│  Syslog (Part 17), FMC Dashboard, ISE Live Log, SOC/SIEM                   │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 4 — ENDPOINT                                                       │
│  Antivirus/EDR, Posture Compliance (ISE ตรวจก่อนปล่อยเข้า Network)         │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 3 — SEGMENTATION                                                   │
│  VLAN (Part 3), ACL (Part 14), TrustSec/SGT (Part 38 Preview → Part 44)    │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 2 — NAC / IDENTITY                                                 │
│  802.1X (Part 37 Preview → Part 44), RADIUS/TACACS+ (Part 22/37) → ISE    │
│  (Part นี้)                                                                │
├─────────────────────────────────────────────────────────────────────────┤
│  LAYER 1 — PERIMETER FIREWALL                                             │
│  FW-1 (ASAv → FTDv, Part นี้), IPS/Snort, File/Malware Policy              │
└─────────────────────────────────────────────────────────────────────────┘
      ▲ Traffic จาก Internet/WAN เจอชั้นนี้ก่อนเสมอ (Layer 1 ล่างสุด = ชั้นแรกที่ Traffic ปะทะ)
```

**หลักการสำคัญของ Defense-in-Depth**: ไม่มีชั้นใดชั้นเดียวที่ "พอ" — ถ้า Attacker ผ่าน Perimeter
Firewall มาได้ (เช่น Phishing ทำให้ Malware เข้ามาจาก Inside) ชั้น NAC/Segmentation/Endpoint ต้อง
ยังจับได้อยู่ นี่คือเหตุผลที่องค์กรระดับ Enterprise ไม่เคยพึ่ง Firewall ตัวเดียวเป็นเครื่องป้องกัน
เพียงอย่างเดียว

### ตำแหน่งของ FW-1, ISE, และ TrustSec ใน Topology หลักของหลักสูตร

```
                                   ┌───────────────┐
                                   │   INTERNET     │
                                   └───────┬───────┘
                                   ┌───────┴───────┐
                                   │  ISP-RTR       │
                                   └───────┬───────┘
                        ┌──────────────────┴──────────────────┐
                        │         WAN-EDGE-1 / WAN-EDGE-2        │
                        └──────────────────┬──────────────────┘
                             LAYER 1 ┌──────┴──────┐
                                     │    FW-1      │  <- FTDv + FMC (Step 422-424)
                                     │ (Transparent) │
                                     └──────┬──────┘
        ┌──────────────────────────────────┴──────────────────────────────────┐
        │                          CORE-SW1 <===LACP Po1===> CORE-SW2           │
        └────┬─────────────────────────────────────────────────────────┬────┘
   ┌──────────┴──────────┐                                   ┌──────────┴──────────┐
   │  DIST-SW1 / DIST-SW2 │                                   │  DIST-SW3 / DIST-SW4 │
   └────┬────────────┬───┘                                   └────┬────────────┬───┘
┌────────┴───┐  ┌─────┴──────┐                             ┌────────┴───┐  ┌─────┴──────┐
│ ACCESS-SW1 │  │ ACCESS-SW2  │  LAYER 2 (NAC) — 802.1X    │ ACCESS-SW3 │  │ ACCESS-SW4  │
│ VLAN 10,20 │  │ VLAN 10,20  │  ทุก Port เชื่อม ISE         │ VLAN 30,40 │  │ VLAN 30,40  │
└────────────┘  └────────────┘  (Preview Part 44)          └────────────┘  └────────────┘
                                          │
                          ┌───────────────┴───────────────┐
                          │   ISE-1 (PAN+MnT)  10.10.99.65  │  <- Step 425-429
                          │   ISE-2 (PSN)       10.10.99.66  │      แทนที่บทบาท RADIUS
                          │   VLAN 99 (MGMT)                │      ของ AAA-SRV (Part 37)
                          └────────────────────────────────┘
```

| องค์ประกอบ | เดิม (Part 1-42) | ใหม่/เพิ่มเข้ามาใน Part นี้ |
|---|---|---|
| FW-1 | ASAv (แนวคิด, ยังไม่ Config) | **FTDv** managed by **FMC** — Access Control Policy, IPS, File Policy |
| Network Access Auth | AAA-SRV (Generic RADIUS/TACACS+, Part 22/37) | **Cisco ISE** — NAC เต็มรูปแบบ (Profiling, Posture, Guest, TrustSec) |
| Segmentation | VLAN + ACL (Part 3, 14) | เตรียมพื้นฐานให้ **TrustSec/SGT** (รายละเอียดเต็ม Part 44) |

> **จุดที่ต้องเข้าใจให้ชัด**: Part นี้**ไม่ได้ลบ** AAA-SRV หรือ config เดิมจาก Part 22/37 ออกไป —
> TACACS+ Device Admin ของ AAA-SRV ยังทำงานเหมือนเดิมได้ (ISE รองรับ TACACS+ ได้ด้วยถ้าต้องการ
> Migrate ในอนาคต) สิ่งที่ Part นี้ทำคือ**เพิ่ม ISE เข้ามาเป็นตัวเลือกที่แข็งแรงกว่า**สำหรับบทบาท
> Network Access (RADIUS) ที่ AAA-SRV ทำอยู่แบบ "Flat Config" ใน Part 37 — และเปิดใช้งาน FW-1 ที่
> เคย "มีอยู่ในไดอะแกรมเฉยๆ" ให้ทำงานจริงเป็นครั้งแรก

---

## Step 422 — สถาปัตยกรรม Cisco Firepower NGFW: FTD vs ASA, FMC vs FDM

### จาก ASA สู่ FTD: ทำไม Cisco เปลี่ยนสถาปัตยกรรม

**ASA (Adaptive Security Appliance)** คือ Firewall รุ่นเดิมของ Cisco ที่ทำ Stateful Firewall
+ VPN ได้ดีมาก แต่ไม่มี **Next-Generation Firewall (NGFW)** Feature ในตัว (IPS, Application
Visibility, URL Filtering, Malware/File Inspection ต้องซื้อกล่องแยก เช่น Cisco Firepower
Appliance รุ่นเก่าที่ทำงานเป็น IPS Module เสริม) — Cisco จึงรวมทั้งสองโลกเข้าด้วยกันเป็น
**Firepower Threat Defense (FTD)**: **Unified Software Image** เดียวที่มีทั้ง ASA Data Plane
(Lina Engine — ยังใช้ CLI คล้าย ASA อยู่ข้างใต้) และ Firepower Services (Snort IPS Engine,
File/Malware, URL Filtering, Application Control) รวมอยู่ในอิมเมจเดียว

```
สถาปัตยกรรมภายในของ FTD (Unified Image)

┌─────────────────────────────────────────────────────────────────┐
│                    FTD (Firepower Threat Defense)                  │
│  ┌───────────────────────┐        ┌───────────────────────────┐  │
│  │   Lina Engine            │        │   Snort Engine               │  │
│  │  (มาจาก ASA เดิม)          │◄──────►│  (IPS/Application/File       │  │
│  │  - Stateful Firewall     │        │   Detection/Malware)         │  │
│  │  - Routing/NAT/VPN       │        │  - Deep Packet Inspection     │  │
│  │  - Interface/Zone        │        │  - Signature-based Detection  │  │
│  └───────────────────────┘        └───────────────────────────┘  │
│              ทั้งสอง Engine ทำงานร่วมกันใน Data Path เดียว              │
└─────────────────────────────────────────────────────────────────┘
```

### ตารางเปรียบเทียบ ASA vs FTD

| คุณสมบัติ | ASA (แนวคิดเดิมที่ FW-1 อ้างถึงใน Part 1-42) | FTD (ที่ Part นี้แปลง FW-1 ไปใช้) |
|---|---|---|
| Firewall Engine | Stateful Firewall (Lina) เท่านั้น | Stateful Firewall (Lina) **+** IPS/Application/File (Snort) |
| Management | ASDM (Java-based GUI) หรือ CLI โดยตรง | **FMC** (Centralized) หรือ **FDM** (Local) |
| IPS | ต้องมี Module/Appliance แยก (FirePOWER Services เดิม) | **มีในตัว** (Snort Engine) |
| URL Filtering / App Visibility | ไม่มีในตัว | มีในตัว (ต้องมี License แยกตามฟีเจอร์) |
| Multi-context (Virtual Firewall) | รองรับ | ไม่รองรับ (FTD ยังไม่มี Multi-context ในหลาย Version) |
| CLI พื้นฐาน | ASA CLI (`show conn`, `packet-tracer`) | **CLI คล้าย ASA มาก** (Lina อยู่ข้างใต้ ใช้ `show connection` ได้เหมือนกัน) |
| License Model | Smart License (รุ่นใหม่) | Smart License (บังคับ, ผูกกับ Cisco Smart Software Manager) |

> **สรุปสำหรับข้อสอบ CCNP ENCOR**: ข้อสอบชอบถามว่า "FTD ต่างจาก ASA อย่างไร" — คำตอบหลักคือ
> **FTD = ASA Engine (Lina) + Firepower Services (Snort) ในอิมเมจเดียว จัดการผ่าน FMC/FDM** ไม่ใช่
> Product คนละตัวที่ไม่เกี่ยวกัน

### FMC vs FDM: เลือก Management Platform อย่างไร

| คุณสมบัติ | **FMC** (Firepower Management Center) | **FDM** (Firepower Device Manager) |
|---|---|---|
| รูปแบบ | Centralized — จัดการ FTD หลายตัวพร้อมกันจากจุดเดียว | Local — Web GUI ฝังอยู่ใน FTD ตัวนั้นเอง จัดการทีละตัว |
| เหมาะกับ | Enterprise ที่มี Firewall หลายตัว/หลาย Site (เช่น FW-1 + Firewall สาขาในอนาคต) | Branch/Small Site ที่มี FTD ตัวเดียว ไม่ต้องการ Server จัดการแยก |
| ฟีเจอร์ที่ครบกว่า | **ครบกว่า** — Correlation Policy, Multi-domain, Advanced IPS Tuning, Long-term Analytics | จำกัดกว่า — ฟีเจอร์พื้นฐานพอสำหรับ Site เดียว |
| ต้องมี Appliance/VM แยก? | **ต้องมี** FMC Server/VM แยกต่างหาก | ไม่ต้อง — ใช้ Web GUI ของ FTD เองได้ทันที |
| การ Deploy Policy หลาย Site พร้อมกัน | ทำได้ (Push Policy เดียวไปหลาย FTD) | ทำไม่ได้ (ต้อง Config ทีละตัว) |

> **การเลือกใช้กับ FW-1**: เพราะ Topology ของหลักสูตรนี้มีแนวโน้มขยายเป็น Multi-site ใน Part หลัง
> (SD-WAN จาก [Part 39](part-039-sd-wan-fundamentals.md) เตรียมพื้นฐานไว้แล้ว) เราจึงเลือก **FMC**
> เป็น Management Platform ของ FW-1 ตั้งแต่ต้น — ถ้าอนาคตเพิ่ม Firewall ที่ Branch อีกตัว จะ Push
> Access Control Policy เดียวกันไปทั้งสองตัวได้จากจุดเดียว

### แปลง FW-1 จาก ASA (แนวคิด) ไปเป็น FTD — Checklist เชิงแนวคิด

| # | ขั้นตอน | รายละเอียด |
|---|---|---|
| 1 | ติดตั้ง FTDv Image บน Hypervisor เดิมที่รัน ASAv | FTDv ใช้ Resource Requirement (vCPU/RAM) สูงกว่า ASAv เพราะมี Snort Engine เพิ่ม — ต้องเผื่อ RAM ขั้นต่ำ 8-28 GB ตาม Throughput ที่ต้องการ |
| 2 | กำหนด Management Interface | FTD แยก Management Interface ออกจาก Data Interface เสมอ (ไม่เหมือน ASA ที่ใช้ Data Interface เป็น Management ได้) — ใน Lab นี้ใช้ `10.10.99.70/24` (VLAN 99 MGMT) |
| 3 | ติดตั้ง/เตรียม FMC (VM หรือ Appliance) | FMC ต้องมี IP แยกใน MGMT VLAN เช่นกัน — ใช้ `10.10.99.71/24` |
| 4 | Register FTD เข้า FMC | ใส่ Registration Key ทั้งสองฝั่ง (บน FTD CLI ด้วย `configure manager add`, และบน FMC GUI ด้วย Device serial/IP) |
| 5 | สร้าง Access Control Policy บน FMC แล้ว Deploy ไปยัง FTD | รายละเอียดเต็มใน Step 424 |

```
! ตัวอย่าง Concept: คำสั่งฝั่ง FTD CLI (Lina-based) เพื่อ Register เข้า FMC
> configure manager add 10.10.99.71 CiscoLab-Reg-Key2026
Manager successfully configured.
Please make note of reg_key as this will be required while adding
Device in FMC.

> show managers
Type                       : Manager
Host                       : 10.10.99.71
Registration               : Pending
```

---

## Step 423 — FTD Deployment Mode: Routed vs Transparent และตำแหน่งของ FW-1

### สองโหมดการวาง FTD ใน Network

| คุณสมบัติ | **Routed Mode** | **Transparent Mode** |
|---|---|---|
| การทำงาน | FTD เป็น **L3 Hop** เต็มรูปแบบ — มี IP ของตัวเองบนทุก Interface, ทำ Routing จริง | FTD เป็น **Bridge (L2)** — Interface ทั้งสองฝั่งอยู่ Subnet เดียวกัน, FTD "โปร่งใส" ต่อ Routing |
| ต้องเปลี่ยน IP Addressing เดิมไหม | **ต้องเปลี่ยน** — อุปกรณ์ฝั่งใดฝั่งหนึ่งต้องมี Next-hop เป็น FTD | **ไม่ต้องเปลี่ยน** — Subnet เดิมยังใช้ได้ ไม่กระทบ Routing Protocol ที่ Config ไว้แล้ว |
| Routing Protocol วิ่งผ่านได้ไหม | FTD เข้าร่วม Routing เอง (Static/Connected เท่านั้น — FTD ไม่รัน OSPF/BGP Process เต็มรูปแบบแบบ Router) | **วิ่งผ่านได้โดยไม่มี FTD เข้าร่วม** — Hello/Update Packet ถูก Bridge ผ่านไปเหมือน Switch (ต้องเปิด ACL อนุญาต Protocol นั้นด้วย) |
| Interface Count ต่อ Bridge Group | 1 Interface = 1 Subnet | ต้องมี **อย่างน้อย 2 Interface** ต่อ Bridge Group (BVI — Bridge Virtual Interface) |
| เหมาะกับ | Deploy ใหม่ที่ยังไม่มี Topology เดิม, ต้องการทำ NAT/VPN Termination | **แทรกเข้า (Insert)** ใน Topology ที่มี Routing/Addressing ทำงานสมบูรณ์อยู่แล้ว โดยไม่อยากรื้อ |
| Multicast/Broadcast | Forward ตาม Routing table ปกติ | Bridge ทุกอย่างที่ไม่ถูก ACL บล็อค (รวม Broadcast/ARP) |

### เหตุผลที่ Cisco (และหลักสูตรนี้) แนะนำ Transparent Mode สำหรับ FW-1

Topology หลักของเราตั้งแต่ [Part 11](part-011-ospfv2-fundamentals.md) ถึง [Part 31]
(part-031-route-redistribution.md) ได้สร้าง **OSPF Area 0** และ **BGP Peering** ให้ WAN-EDGE-1 ↔
CORE-SW1 (`10.10.254.0/30`) และ WAN-EDGE-2 ↔ CORE-SW2 (`10.10.254.4/30`) ทำงานสมบูรณ์แล้วตาม
[IP Address Plan](00-ip-address-plan.md) — ถ้าเราวาง FW-1 แบบ **Routed Mode** ตรงกลางเส้นนี้
จะต้อง**รื้อ Subnet เดิมทั้งหมด** (แยกเป็น 2 Subnet ใหม่คนละฝั่งของ FW-1) และแก้ OSPF/BGP Neighbor
Relationship ที่ผ่านมาทุก Part ก่อนหน้า

```
Transparent Mode: FW-1 แทรกเข้าไปโดยไม่แก้ Subnet เดิม

WAN-EDGE-1                    FW-1 (Bridge Group 1)                CORE-SW1
10.10.254.1/30  ◄──────────────────────────────────────────────►  10.10.254.2/30
                    │  Gi0/0 "OUTSIDE"      Gi0/1 "INSIDE"  │
                    │        (ไม่มี IP — เป็น Bridge Port)      │
                    │           BVI1 = 10.10.254.x           │  <- IP เดียวสำหรับ
                    │        (management/monitoring เท่านั้น)   │     Diagnostic เท่านั้น
                    └───────────────────────────────────────┘
      OSPF Hello / BGP Update วิ่งผ่าน FW-1 ได้ปกติทุกประการ
      (ตราบใดที่ Access Control Policy อนุญาต Protocol 89/TCP 179 — ดู Step 424)

Routed Mode: ต้องรื้อ Subnet เดิม (ทางเลือกสำหรับ Site ใหม่ในอนาคต เช่น Branch ที่ยังไม่มี Topology)

WAN-EDGE-1               FW-1 (2 IP, เป็น Router จริง)              CORE-SW1
10.x.x.1/30 (Subnet A) ──────►  Gi0/0=.2 (Subnet A)                     │
                                Gi0/1=.1 (Subnet B) ──────────────► 10.y.y.2/30 (Subnet B)
      ต้อง Renumber ทั้งสอง Subnet + แก้ OSPF/BGP Neighbor ที่ทำมาแล้วทุก Part
```

| จุดตัดสินใจ | Transparent Mode | Routed Mode |
|---|---|---|
| **FW-1 ในหลักสูตรนี้** (แทรกเข้า Topology ที่มี Routing สมบูรณ์แล้วจาก Part 11-31) | ✅ **เลือกใช้** | ไม่เลือก — เสียของ Config เดิมโดยไม่จำเป็น |
| Firewall ตัวใหม่ที่ Site/Branch ใหม่ที่ยังไม่มี Routing เดิม | ไม่จำเป็นต้องใช้ | ✅ เหมาะกว่า — ได้ประโยชน์จาก NAT/VPN Termination เต็มที่ |

> **ข้อจำกัดของ Transparent Mode ที่ต้องรู้**: NAT ทำได้จำกัดกว่า Routed Mode มาก (ไม่รองรับ PAT
> แบบเต็มรูปแบบในหลาย Use Case) และ **ไม่รองรับ Dynamic Routing Protocol ให้ FTD เข้าร่วมเอง** —
> เหมาะกับบทบาท "ตัวกรอง Traffic ที่ผ่าน" มากกว่าบทบาท "L3 Gateway/VPN Endpoint" ถ้าในอนาคต FW-1
> ต้องทำ Site-to-Site VPN Termination เอง (ต่อยอดจาก GRE Tunnel ที่ [Part 19]
> (part-019-wan-technologies.md) ทำไว้บน WAN-EDGE-1) จะต้องพิจารณาสลับไป Routed Mode หรือเพิ่ม
> Firewall ตัวใหม่ที่ Edge แยกออกมา

### ตำแหน่ง Interface ของ FW-1 หลัง Config

| Interface | Zone | เชื่อมต่อไปยัง | หมายเหตุ |
|---|---|---|---|
| Gi0/0 | OUTSIDE (WAN Side) | WAN-EDGE-1 / WAN-EDGE-2 (ผ่าน Bridge Group แยกฝั่งละ 1 Group) | รับ Traffic จาก Internet/WAN ก่อนถึง CORE |
| Gi0/1 | INSIDE (Campus Side) | CORE-SW1 / CORE-SW2 | Traffic ที่ผ่านการกรองแล้วเท่านั้นไปถึง Campus |
| Management (dedicated NIC) | MGMT | VLAN 99 (10.10.99.70/24) | ใช้คุยกับ FMC เท่านั้น ไม่รับ Data Traffic |

---

## Step 424 — พื้นฐาน FTD Security Policy: Access Control Policy, IPS, File/Malware Policy

### Access Control Policy (ACP) — นโยบายหลักที่ควบคุมทุก Traffic ที่ผ่าน FW-1

**Access Control Policy (ACP)** คือ Policy หลักบน FMC ที่แทนที่แนวคิด `access-list` แบบ ASA เดิม
ด้วยโครงสร้างที่ละเอียดกว่า — แต่ละ **Rule** ใน ACP จับคู่ **Condition** (Zone, Network, VLAN,
Port, Application, URL Category, User/Group จาก ISE) เข้ากับ **Action** (Allow, Trust, Block,
Block with Reset, Monitor) และยังผูก **IPS Policy** / **File Policy** ต่อท้าย Rule ที่ Allow ได้

```
โครงสร้าง Access Control Policy บน FMC

┌─────────────────────────────────────────────────────────────────────┐
│  ACCESS CONTROL POLICY: "FW1-EDGE-POLICY"                              │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 1: ALLOW-ROUTING-PROTOCOLS                                       │
│    Condition: Zone(OUTSIDE↔INSIDE), Protocol=OSPF(89)/BGP(TCP 179)     │
│    Action: Allow (Trust — ไม่ต้อง Inspect ลึก เพื่อลด Latency)          │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 2: ALLOW-OUTBOUND-WEB                                            │
│    Condition: Zone(INSIDE→OUTSIDE), Port=80/443, Application=Web-Browsing│
│    Action: Allow + IPS Policy "BALANCED-SECURITY" + File Policy        │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 3: ALLOW-INBOUND-DMZ-WEB (Server1 HTTPS จาก Part 15 NAT)         │
│    Condition: Zone(OUTSIDE→INSIDE), Dest=Server1(Static NAT), Port=443 │
│    Action: Allow + IPS Policy "SECURITY-OVER-CONNECTIVITY"             │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 4 (สุดท้ายเสมอ): DEFAULT-DENY                                     │
│    Condition: any any                                                  │
│    Action: Block with Reset + Log                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### ลำดับการประมวลผล Rule — First-Match เหมือน ACL แต่ละเอียดกว่า

FTD ประมวลผล ACP **จากบนลงล่าง** และหยุดที่ Rule แรกที่ตรงเงื่อนไข (First-Match — เหมือนหลักการ
ACL ที่เรียนใน [Part 14](part-014-access-control-lists.md)) ดังนั้น **ลำดับ Rule สำคัญมาก**:
ต้องวาง Rule ที่ Specific ที่สุดไว้บนสุด และ Rule กว้างๆ (เช่น Default Deny) ไว้ล่างสุดเสมอ — จุดที่
ต่างจาก ACL ธรรมดาคือ ACP มี **Category แยกกลุ่ม** (เช่น "Mandatory" อยู่บนสุดเสมอไม่ว่าใครจะเพิ่ม
Rule ใหม่ตรงไหน, "Default" อยู่ล่างสุดเสมอ) ทำให้ควบคุม Rule จำนวนมากได้เป็นระบบกว่า

> **เหตุผลที่ Rule 1 ต้องมาก่อนและเป็น "Trust"**: OSPF/BGP Traffic ระหว่าง WAN-EDGE ↔ CORE-SW ที่
> วิ่งผ่าน FW-1 ใน Transparent Mode (Step 423) ต้องได้รับอนุญาตโดยไม่ถูก Inspect ลึกเกินไป
> (ลด Latency ของ Routing Protocol ที่ Sensitive ต่อ Delay) — ถ้าลืม Rule นี้ OSPF/BGP Neighbor
> ที่ทำงานมาตั้งแต่ Part 11-31 จะ **Drop ทันที** เพราะ Default-Deny ที่ Rule สุดท้าย

### Intrusion Prevention (Snort Engine) — เมื่อ Rule "Allow" ยังไม่ปลอดภัยพอ

Rule ที่ Action = Allow ใน ACP สามารถผูก **Intrusion Policy** เพิ่มได้ — Intrusion Policy ทำงานบน
**Snort Engine** ตรวจ Payload ของ Packet ที่ผ่าน Rule นั้นด้วย **Signature-based Detection**
เทียบกับฐาน Rule ของ Talos (Cisco Threat Intelligence) — ถ้าพบรูปแบบ Attack (เช่น SQL Injection,
Buffer Overflow, Known Exploit) จะ Block/Drop Packet นั้นแม้ ACP จะ Allow ผ่านไปแล้วก็ตาม

| Intrusion Policy ระดับ (Default Template ของ Cisco) | ระดับความเข้ม | ใช้เมื่อ |
|---|---|---|
| Connectivity Over Security | เข้มน้อยที่สุด | ต้องการความเร็ว/Uptime สูงสุด ยอมรับความเสี่ยง |
| **Balanced Security and Connectivity** | กลาง (ค่า Default แนะนำ) | ใช้กับ Rule 2 (Outbound Web) — สมดุลระหว่างความปลอดภัยกับ Performance |
| Security Over Connectivity | เข้มกว่า | ใช้กับ Rule 3 (Inbound DMZ) — Traffic จาก Internet เข้ามาต้องเข้มกว่า |
| Maximum Detection | เข้มที่สุด | Zone ที่ Sensitive สูงสุด (เช่น PCI Compliance Segment) — แลกกับ False Positive ที่มากขึ้น |

### File & Malware Policy — ตรวจไฟล์ที่ Transfer ผ่าน FW-1

**File Policy** ทำงานคู่กับ ACP Rule เพื่อตรวจไฟล์ที่ถูก Download/Upload ผ่าน Protocol เช่น HTTP,
FTP, SMTP, SMB — แบ่งการตรวจเป็น 2 ระดับ:

| ระดับ | วิธีตรวจ | ตัวอย่าง |
|---|---|---|
| **File Type Control** | บล็อค/อนุญาตตาม Type ของไฟล์ (ดูจาก Magic Number จริง ไม่ใช่แค่ Extension) | บล็อค `.exe`, `.bat` ที่ Download จาก Internet เข้า Inside |
| **Malware Cloud Lookup (AMP)** | ส่ง Hash (SHA-256) ของไฟล์ไปเทียบกับ Cisco AMP Cloud Threat Intelligence แบบ Real-time | ไฟล์ `.pdf` ที่ Hash ตรงกับ Malware ที่รู้จักแล้ว → Block ทันทีแม้ Extension จะดูปลอดภัย |

```
FMC Concept: File Policy Rule
┌────────────────────────────────────────────────────────────┐
│  File Policy: "BLOCK-MALWARE-DOWNLOADS"                       │
│    Rule 1: Application Protocol=HTTP, Direction=Download,     │
│            File Type=Executables → Action: Block               │
│    Rule 2: Application Protocol=Any, Direction=Any,           │
│            Malware Cloud Lookup → Action: Block Malware         │
└────────────────────────────────────────────────────────────┘
ผูกเข้ากับ Rule 2 (ALLOW-OUTBOUND-WEB) ของ ACP ด้านบน
```

> **ลำดับการประมวลผลจริงเมื่อ Packet ผ่าน FW-1**: ACP Rule Match ก่อน (Zone/Network/Port/App) →
> ถ้า Action=Allow และมี Intrusion Policy ผูกไว้ → Snort ตรวจ Payload → ถ้าผ่าน และมี File Policy
> ผูกไว้ → ตรวจไฟล์/Malware → ผ่านครบทุกชั้นจึง Forward Packet จริง (นี่คือ Defense-in-Depth
> **ภายใน FW-1 ตัวเดียว** ซ้อนกันอีกหลายชั้นย่อย)

---

## Step 425 — Cisco ISE พื้นฐาน: จาก RADIUS ธรรมดาไปสู่ Full NAC

### AAA-SRV (Part 22/37) ทำอะไรได้ และทำอะไรไม่ได้

[Part 22](part-022-aaa-device-hardening.md) และ [Part 37](part-037-wireless-security-advanced.md)
ใช้ **AAA-SRV** แบบ Generic (`10.10.99.60`) ทำหน้าที่เป็น "กล่องดำ" ที่ตอบ Accept/Reject
ตาม Username/Password หรือ Group ที่ตั้งไว้ล่วงหน้าแบบ**คงที่ (Static)** — ทำงานได้ดีสำหรับ
Use Case พื้นฐาน (TACACS+ Device Admin, RADIUS 802.1X ธรรมดา) แต่มีข้อจำกัดชัดเจนเมื่อ Enterprise
โตขึ้น:

| ข้อจำกัดของ RADIUS/TACACS+ แบบ Generic (AAA-SRV) | สิ่งที่ ISE เพิ่มเข้ามา |
|---|---|
| รู้จัก Client แค่จาก Username/Password/MAC ที่ตั้งไว้ตายตัว | **Profiling** — จำแนก "ประเภทอุปกรณ์" อัตโนมัติ (Step 428) โดยไม่ต้องมีใครลงทะเบียนล่วงหน้า |
| ไม่ตรวจสอบ "สุขภาพ" ของ Endpoint ก่อนปล่อยเข้า Network | **Posture Assessment** — เช็ค AV/Patch/Compliance ก่อนให้สิทธิ์เต็ม (Step 428) |
| Guest ต้องตั้ง Account/SSID แยกด้วยมือ (Part 37 ทำแค่ Local Web-Auth บน WLC) | **Guest/Sponsor Portal เต็มรูปแบบ** ที่จัดการ Lifecycle ของ Guest Account ได้ (Step 429) |
| ไม่มี Concept ของ "กลุ่ม" ที่ผูกกับ Policy แบบ Dynamic ข้าม VLAN/Subnet | **TrustSec/SGT** — แจก Security Group Tag ให้ Policy ทำงานแยกจาก IP/VLAN เลย (Part 44) |
| Policy เขียนเป็น Flat Config บนอุปกรณ์แต่ละตัว (ตาม Group ที่ Admin ผูกไว้เอง) | **Policy Set** — เขียน Policy จากมุมมอง "Identity" ครั้งเดียว ใช้ได้ทั้ง Wired/Wireless/VPN |

### ISE คืออะไรกันแน่ — นิยามที่ต้องจำ

**Cisco Identity Services Engine (ISE)** คือ **Policy Engine กลาง** สำหรับ Network Access
Control (NAC) ที่รวมความสามารถ 5 อย่างไว้ในแพลตฟอร์มเดียว:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CISCO ISE — 5 ความสามารถหลัก                    │
├─────────────────────────────────────────────────────────────────────┤
│ 1) AAA Server        — ทำ RADIUS + TACACS+ ได้ในตัวเดียว (แทน AAA-SRV) │
│ 2) Profiling          — จำแนกประเภทอุปกรณ์อัตโนมัติ (Step 428)          │
│ 3) Posture             — ตรวจสุขภาพ Endpoint ก่อนให้สิทธิ์เต็ม (Step 428) │
│ 4) Guest/BYOD          — Sponsor Portal, Self-Reg, My Devices (Step 429)│
│ 5) TrustSec/SGT Server — แจก Security Group Tag Policy (Part 44)       │
└─────────────────────────────────────────────────────────────────────┘
```

> **คำเปรียบเทียบที่จำง่าย**: **AAA-SRV = ยามที่ถาม Password ตรงประตู, ISE = ระบบ Security ทั้ง
> อาคารที่รู้ว่าใครเข้ามา, ถืออะไรมา, สุขภาพเป็นอย่างไร, ควรไปโซนไหน, และปรับสิทธิ์ Real-time ได้
> โดยไม่ต้องให้เดินออกจากอาคารไปใหม่ (CoA — Change of Authorization ที่ [Part 37]
> (part-037-wireless-security-advanced.md#step-367) แนะนำไว้แล้ว)**

### ISE รองรับทั้ง RADIUS และ TACACS+ ในตัวเดียว (Dual-Protocol เหมือน AAA-SRV แต่ลึกกว่า)

เหมือนที่ [Part 37 Step 362](part-037-wireless-security-advanced.md) บอกไว้ว่า AAA-SRV รัน
TACACS+ และ RADIUS พร้อมกันได้บน Server เดียว (เพราะ Port/Transport ต่างกัน) — ISE ก็ทำแบบเดียวกัน
แต่มี **Policy Engine เดียว** ที่ทั้งสอง Protocol ใช้ร่วมกันได้ (ไม่ต้องเขียน Policy แยกกันคนละชุด
เหมือน Generic Server สองระบบที่ไม่คุยกัน)

| บทบาท | AAA-SRV (Part 22/37) | ISE (Part นี้) |
|---|---|---|
| TACACS+ Device Admin (Part 22) | ✅ ทำอยู่แล้ว | ✅ ทำได้เหมือนกัน (Migration เป็น Optional — ดู Step 430) |
| RADIUS 802.1X/Wireless (Part 37) | ✅ ทำอยู่แล้ว แบบ Flat Config | ✅ **แทนที่ด้วย Policy Set** (Step 427) |
| Profiling/Posture/Guest/TrustSec | ❌ ไม่มี | ✅ มีครบ |

---

## Step 426 — สถาปัตยกรรม ISE Deployment: PAN, MnT, PSN

### 3 Persona (บทบาท) ของ ISE Node

ISE แบ่งการทำงานออกเป็น **Persona** (บทบาท) ที่ Node หนึ่งตัวรับได้มากกว่า 1 บทบาทพร้อมกัน หรือ
แยกกันเป็น Node คนละตัวก็ได้ตามขนาดองค์กร:

```
┌───────────────────────────────────────────────────────────────────────┐
│  PAN — Policy Administration Node                                        │
│  "สมองที่คิด Policy"                                                      │
│  - จุดเดียวที่ Admin Login เข้าไป Config Policy Set/Profiling/Posture       │
│  - Sync Configuration ไปยัง Node อื่นทั้งหมดในระบบ                          │
│  - มีได้ 1 Primary PAN + 1 Secondary PAN (สำหรับ Redundancy) เท่านั้น         │
├───────────────────────────────────────────────────────────────────────┤
│  MnT — Monitoring Node (Monitoring and Troubleshooting)                   │
│  "ความจำที่บันทึกเหตุการณ์"                                                 │
│  - เก็บ Log ทั้งหมด (RADIUS Live Log, TACACS+ Live Log, Alarm)               │
│  - เป็นจุดที่ Admin ดู Dashboard/Report/Troubleshoot Authentication ผิดพลาด   │
├───────────────────────────────────────────────────────────────────────┤
│  PSN — Policy Service Node (Policy Service Node)                          │
│  "มือที่ลงมือทำงานจริง"                                                    │
│  - **รับ RADIUS/TACACS+ Request จริงจากอุปกรณ์ Network** (WLC, Switch)      │
│  - ประเมิน Policy Set ที่ PAN สั่งมา แล้วตอบ Accept/Reject/CoA               │
│  - ทำ Profiling, Posture, Guest Portal ให้บริการจริงที่ Node นี้             │
│  - **สามารถมีหลายตัวพร้อมกัน** เพื่อกระจายโหลดตามภูมิภาค/Site                │
└───────────────────────────────────────────────────────────────────────┘
```

### Standalone (All-in-One) vs Distributed Deployment

| รูปแบบ | ลักษณะ | เหมาะกับ |
|---|---|---|
| **Standalone** | ทุก Persona (PAN+MnT+PSN) รวมอยู่ใน **Node เดียว** | Lab/องค์กรขนาดเล็ก-กลาง (< 2,000 Endpoint) — ใช้ ISE-1 ตัวเดียวทำทุกอย่าง |
| **Distributed** | แยก Persona ออกเป็นหลาย Node ตามหน้าที่ | Enterprise ขนาดใหญ่/หลาย Site — เพิ่ม PSN ได้ตามจำนวน Site โดยไม่ต้องเพิ่ม PAN |

### การออกแบบ ISE Deployment สำหรับหลักสูตรนี้

Lab ของหลักสูตรนี้ (Topology 1 Campus, ยังไม่ Multi-site เต็มรูปแบบ) เลือกใช้ **Distributed
เบื้องต้น** เพื่อสอนแนวคิดการแยก Persona ให้เห็นภาพ แม้ Scale จริงยังเล็กพอจะใช้ Standalone ได้:

| Node | Persona | IP (VLAN 99 MGMT) |
|---|---|---|
| **ISE-1** | PAN (Primary) + MnT (Primary) | `10.10.99.65/24` |
| **ISE-2** | PSN | `10.10.99.66/24` |

```
                     ┌─────────────────────────┐
                     │        ISE-1               │
                     │   PAN + MnT (Primary)      │   <- Admin Config Policy ที่นี่
                     │   10.10.99.65               │       ดู Live Log/Report ที่นี่
                     └────────────┬────────────┘
                                   │ Sync Config (ทุก Node ต้อง Sync กับ PAN)
                     ┌────────────┴────────────┐
                     │        ISE-2               │
                     │   PSN                       │   <- รับ RADIUS/TACACS+ จริง
                     │   10.10.99.66               │       จาก WLC-1/CORE-SW/DIST-SW
                     └──┬──────────┬──────────┬──┘
              RADIUS 1812/1813  TACACS+ 49    CoA 1700
                     │              │              │
                ┌────┴────┐   ┌─────┴─────┐   ┌────┴────┐
                │  WLC-1   │   │ CORE/DIST  │   │ ACCESS-SW│
                │(Part 37) │   │(Part 22)   │   │(Part 44) │
                └─────────┘   └───────────┘   └─────────┘
```

> **เหตุผลที่แยก PSN ออกจาก PAN แม้ Lab ยังเล็ก**: การแยก Persona ตั้งแต่ต้นทำให้**เพิ่ม PSN ตัวที่
> สองในอนาคต**ได้ง่าย (เช่น ถ้าองค์กรขยายไป Branch ใหม่ ก็แค่เพิ่ม PSN ที่ Branch โดยไม่ต้องยุ่งกับ
> PAN/MnT เดิม) — ตรงกับแนวทาง Scale-out ที่หลักสูตรนี้เตรียมไว้ตั้งแต่ [Part 39]
> (part-039-sd-wan-fundamentals.md) เรื่อง SD-WAN Multi-site

### ตรวจสอบสถานะ Deployment (Concept — CLI ของ ISE เป็น Web GUI เป็นหลัก)

```
ISE-1# show application status ise
ise PROCESS NAME               STATE            PROCESS ID
Database Listener               running          1123
Database Server                 running          62 PROCESSES
Application Server              running          2210
Profiler Database               running          2245
AD Connector                    running          2301
M&T Session Database            running          2367
M&T Log Processor               running          2390
ISE Indexing Engine             running          2412
```

---

## Step 427 — ISE Authentication & Authorization Policy: Policy Set แทน AAA-SRV

### จาก "Flat Config" (Part 37) สู่ "Policy Set" (ISE)

[Part 37](part-037-wireless-security-advanced.md) ให้ AAA-SRV ตอบ Accept พร้อม Attribute
คงที่ตาม User Group ที่ตั้งไว้ (เช่น Group "Contractor" → VLAN 45 เสมอ) — วิธีนี้ใช้ได้ แต่ทุก
Policy ใหม่ต้องไปเพิ่มที่ AAA-SRV โดยตรงและไม่มีโครงสร้างที่มองเห็นเป็นภาพรวมง่ายๆ ISE แก้ปัญหานี้
ด้วย **Policy Set**: ชุด Authentication + Authorization Policy ที่จัดกลุ่มตาม Use Case ชัดเจน
และประเมินจาก **Condition หลายมิติพร้อมกัน** (ไม่ใช่แค่ Username/Group อย่างเดียว)

```
โครงสร้าง Policy Set บน ISE

┌─────────────────────────────────────────────────────────────────────┐
│  POLICY SET 1: "Wireless-Corporate-Access"                             │
│    Condition (เลือก Policy Set): Radius:NAS-IP-Address = 10.10.40.5      │
│                                    (WLC-1) AND SSID = ENTERPRISE-SECURE  │
│  ├─ Authentication Policy: ใช้ Certificate (EAP-TLS) หรือ AD (PEAP)      │
│  └─ Authorization Policy: (Rules ประเมินจากบนลงล่าง)                    │
│       Rule 1: EndPointGroup=Corporate-Laptop AND Time=Office-Hours       │
│                → VLAN 40, dACL=PERMIT-ALL-INTERNAL, SGT=EMPLOYEE (Part44)│
│       Rule 2: EndPointGroup=Contractor-Laptop                            │
│                → VLAN 45, dACL=CONTRACTOR-RESTRICT, SGT=CONTRACTOR       │
│       Rule 3: EndPointGroup=Cisco-IP-Phone (จาก Profiling, Step 428)     │
│                → VLAN 20 (VOICE), dACL=VOICE-ONLY                        │
│       Default: DenyAccess                                                │
├─────────────────────────────────────────────────────────────────────┤
│  POLICY SET 2: "Wired-8021X-Access" (Preview — เต็มรูปแบบ Part 44)       │
├─────────────────────────────────────────────────────────────────────┤
│  POLICY SET 3: "Guest-CWA-Access" (Step 429)                            │
└─────────────────────────────────────────────────────────────────────┘
```

### ตัวอย่าง Condition ที่ ISE ใช้ได้ (มากกว่า Group ธรรมดาของ AAA-SRV มาก)

| ประเภท Condition | ตัวอย่าง | เทียบกับ AAA-SRV (Part 37) |
|---|---|---|
| **Device Type** (จาก Profiling) | `EndPointGroup = Cisco-IP-Phone`, `Workstation`, `Printer` | AAA-SRV ไม่รู้จัก "ประเภทอุปกรณ์" เลย รู้แค่ Username/MAC ที่ลงทะเบียนไว้ |
| **Location** | `RADIUS:NAS-IP-Address = 10.10.40.5` (เฉพาะ WLC-1), หรือ `NAS-Port-Id` เฉพาะ Switch Port ที่ ACCESS-SW3 | ทำได้ยาก ต้องเขียน Config แยกทุกอุปกรณ์ |
| **Time-of-Day** | `Time And Date = 08:00-18:00 Mon-Fri` (จำกัด Contractor เข้าได้แค่ในเวลาทำงาน) | ต้องพึ่ง `login block-for`/Time-range ACL แยกที่อุปกรณ์เอง — ไม่รวมศูนย์ |
| **Active Directory Group** | `AD:memberOf = CN=Employees,OU=Groups` | ทำได้เหมือนกัน แต่ ISE รวมกับ Condition อื่นในกฎเดียวได้ |
| **Certificate Attribute** (EAP-TLS จาก Part 37 Step 365) | `Certificate:Subject Alternative Name = *.company.local` | ทำได้ แต่ ISE ผูกกับ Profiling/Posture ต่อได้ทันที |

### Worked Example: แทนที่ AAA-SRV Flat Config จาก Part 37 ด้วย ISE Policy Set

**สถานการณ์เดิม (Part 37)**: AAA-SRV ตอบ Access-Accept พร้อม `Tunnel-Private-Group-ID=45` แบบ
คงที่ให้ทุก User ที่อยู่ Group "Contractor" — ไม่สนใจว่าเป็นอุปกรณ์อะไร, เวลาไหน, หรือ Endpoint
ผ่าน Posture หรือไม่

**สถานการณ์ใหม่ (ISE)**: Policy Set เดียวกันตรวจสอบ **4 มิติพร้อมกัน** ก่อนตัดสินใจ:

```
Authorization Policy Rule (แทนที่ AAA-SRV Group "Contractor" เดิม)

Rule: "Contractor-Restricted-Access"
  Condition:
    (AD:memberOf = Contractors)
    AND (EndPointGroup = Corporate-Laptop OR Windows10-Workstation)  <- ต้องเป็นเครื่อง Managed
    AND (Time And Date = 08:00-18:00 Mon-Fri)                        <- นอกเวลานี้ = ปฏิเสธ
    AND (Network Access:UseCase = Host Lookup OR Posture Status = Compliant)  <- ต้องผ่าน Posture
  Permissions (Authorization Result):
    - VLAN/dACL: Tunnel-Private-Group-ID = 45 (VLAN 45, เหมือน Part 37)
    - SGT = CONTRACTOR (เตรียมไว้สำหรับ TrustSec เต็มรูปแบบใน Part 44)
    - Session Timeout = 28800 (8 ชั่วโมง — บังคับ Re-authenticate ทุกวัน)
```

| เปรียบเทียบ | AAA-SRV Flat Config (Part 37) | ISE Policy Set (Part นี้) |
|---|---|---|
| จำนวน Condition ที่ตรวจ | 1 (Group เดียว) | 4 (AD Group + Device Type + Time + Posture) พร้อมกัน |
| ถ้า Contractor ใช้เครื่องส่วนตัว (ไม่ Managed) | ยังได้ VLAN 45 เหมือนเดิม (ตรวจแค่ Username) | ถูกจับด้วย Condition ที่ 2 → ไปอีก Rule/Deny (ตรวจ "ใครถือมา" ด้วย) |
| ถ้า Contractor Login ตอนเที่ยงคืน | ยังผ่านได้ (ไม่มี Time Condition) | ถูก Time Condition บล็อคอัตโนมัติ |
| แก้ Policy ในอนาคต | ต้องแก้ที่ AAA-SRV โดยตรง ไม่มี "มุมมอง" Policy รวม | แก้ที่ Rule เดียวใน Policy Set ที่ ISE-1 (PAN) เห็นภาพรวมทั้งหมด |

---

## Step 428 — ISE Profiling และ Posture Assessment

### Profiling: ISE "รู้จัก" อุปกรณ์ได้อย่างไรโดยไม่ต้องมีใครลงทะเบียนล่วงหน้า

**Profiling** คือกระบวนการที่ ISE จำแนกประเภทอุปกรณ์ (Cisco IP Phone, Printer, Windows
Workstation, iPhone, ฯลฯ) โดยอัตโนมัติจาก "ร่องรอย" ที่อุปกรณ์ส่งออกมาเองตามปกติ — ไม่ต้องมีใคร
มา Manual Registration ทีละเครื่อง

| Probe (แหล่งข้อมูล) | ข้อมูลที่ได้ | ตัวอย่าง |
|---|---|---|
| **DHCP Fingerprint** | Option field ใน DHCP Request ที่แต่ละ OS/อุปกรณ์ส่งแบบเฉพาะตัว | Option 55 List ที่ Windows ส่งต่างจาก iOS/Android |
| **HTTP User-Agent** | String ที่ Browser ส่งมาตอนเข้าเว็บ (ผ่าน Redirect/Portal) | `Mozilla/5.0 (Windows NT 10.0...)` vs `Mozilla/5.0 (iPhone...)` |
| **MAC OUI** | 3 Byte แรกของ MAC Address (ทบทวนจาก [Part 1 Step 7](part-001-networking-fundamentals.md)) | OUI ของ Cisco = ผลิตอุปกรณ์ Cisco (เช่น IP Phone) |
| **CDP/LLDP (ผ่าน Device Sensor)** | Switch ส่ง CDP/LLDP Neighbor Info ที่เห็นต่อให้ ISE | Switch เห็น CDP จาก "Cisco IP Phone 8845" ต่ออยู่ที่ Port นั้น |
| **NMAP Active Scan** | ISE สแกน Port/OS Fingerprint เอง (Active — ต้องเปิดใช้ระมัดระวัง) | เจอ Port 515 (Printer) เปิดอยู่ → เดาว่าเป็น Printer |
| **RADIUS Attribute** | Calling-Station-Id, NAS-Port-Type ที่มาพร้อม Access-Request | บอกว่าเชื่อมผ่าน Wireless หรือ Wired |

```
Profiling Flow แบบง่าย

Endpoint ──DHCP Request──► Switch/WLC ──ส่งต่อ (Device Sensor)──► ISE Profiler
                                                                       │
                                              เทียบกับ Profiling Policy/Feed Library
                                                     (Cisco Feed Service อัปเดตจาก Cloud)
                                                                       │
                                                    ─────────────────────
                                                    │  ผลลัพธ์: EndPointGroup =        │
                                                    │  "Cisco-IP-Phone-8845" (Confidence│
                                                    │   Score สูง ≥ Threshold)          │
                                                    ─────────────────────
                                                                       │
                                          Policy Set (Step 427) ใช้ EndPointGroup นี้
                                          เป็น Condition ตัดสินใจ VLAN/dACL ต่อไป
```

> **จุดสำคัญที่สอบชอบถาม**: Profiling ไม่ได้ "รับรอง" ตัวตนผู้ใช้ (นั่นคือหน้าที่ของ Authentication)
> — มันแค่บอกว่า **"อุปกรณ์นี้น่าจะเป็นอะไร"** ด้วย Confidence Score ที่คำนวณจากหลาย Probe ร่วมกัน
> ยิ่งมี Probe ยืนยันตรงกันมากเท่าไหร่ ความมั่นใจก็สูงขึ้น (เช่น DHCP Fingerprint + MAC OUI + CDP
> ยืนยันตรงกันหมดว่าเป็น Cisco IP Phone = Confidence สูงมาก)

### Posture Assessment: ตรวจ "สุขภาพ" ก่อนให้สิทธิ์เต็ม

**Posture** คือการตรวจสอบว่า Endpoint ที่ผ่าน Authentication มาแล้ว **มีสถานะปลอดภัยเพียงพอ**
ก่อนได้รับสิทธิ์เข้าถึง Resource เต็มรูปแบบหรือไม่ — ทำผ่าน **Posture Agent** (มักเป็น Module
ของ Cisco Secure Client/AnyConnect) ที่ติดตั้งบน Endpoint

| ตัวอย่างเงื่อนไข Posture ที่ตรวจ | ผลลัพธ์ถ้าไม่ผ่าน |
|---|---|
| Antivirus ต้องเป็นเวอร์ชันล่าสุด และ Signature ไม่เก่ากว่า 7 วัน | Non-Compliant |
| Windows Update ต้อง Patch ระดับ Critical ครบ | Non-Compliant |
| ต้องมี Registry Key เฉพาะที่องค์กรกำหนด (ยืนยันว่าเป็นเครื่อง Corporate Image) | Non-Compliant |
| Firewall (Windows Defender Firewall) ต้องเปิดอยู่ | Non-Compliant |

```
Posture Flow (ต่อจาก Authentication สำเร็จ)

1) Client Authenticate สำเร็จ (802.1X/RADIUS) → ได้ VLAN "Quarantine" ชั่วคราวก่อน (จำกัดสิทธิ์)
2) Posture Agent บน Client เช็คสถานะ AV/Patch/Registry ส่งผลกลับไปยัง ISE PSN
3a) ผ่าน (Compliant)     → ISE ส่ง RADIUS CoA (ทบทวนจาก Part 37 Step 367) เปลี่ยนสิทธิ์เป็นเต็มรูปแบบ
3b) ไม่ผ่าน (Non-Compliant) → ค้างอยู่ VLAN Quarantine, Redirect ไปหน้า Remediation
                              (บอกให้ไป Update AV/Patch ก่อน แล้ว Re-check อัตโนมัติ)
```

> **สถานะใน Lab ของหลักสูตรนี้**: Part นี้อธิบาย Posture เชิงแนวคิดเท่านั้น (ต้องมี Cisco Secure
> Client License + Endpoint จริงที่ติดตั้ง Agent เพื่อทดสอบ) — การ Deploy Posture Policy เต็มรูปแบบ
> พร้อม Remediation Action จริงเป็นหัวข้อระดับ CCIE Security ที่อยู่นอกสโคปของ ENCOR

---

## Step 429 — ISE Guest และ BYOD Workflow

### ทบทวนจาก Part 37: Guest CWA แบบพื้นฐานที่ทำได้แค่ระดับ WLC

[Part 37 Step 367](part-037-wireless-security-advanced.md) ทำ Guest Wireless ด้วย **Local
Web-Auth บน WLC-1 เอง** — Client Associate SSID Open แล้ว WLC Redirect ไปหน้า Login ที่ WLC
สร้างเอง (Portal พื้นฐาน ไม่ปรับแต่งได้มาก, ไม่มีระบบจัดการ Guest Account แยก, Admin ต้องสร้าง
Local User ให้ Guest ด้วยมือ) — ISE ยกระดับ Guest Workflow ทั้งหมดนี้ให้เป็นระบบรวมศูนย์

### 3 รูปแบบ Guest/BYOD Workflow บน ISE

```
┌───────────────────────────────────────────────────────────────────────┐
│  1) SPONSORED GUEST ACCESS                                               │
│     พนักงาน (Sponsor) Login เข้า Sponsor Portal → สร้าง Guest Account      │
│     ให้ผู้มาเยือน (กำหนดวันหมดอายุ, สิทธิ์การเข้าถึง) → ส่ง Credential ให้    │
│     Guest ผ่าน SMS/Email อัตโนมัติ                                        │
├───────────────────────────────────────────────────────────────────────┤
│  2) SELF-REGISTRATION PORTAL                                             │
│     Guest ลงทะเบียนตัวเอง (กรอกชื่อ/อีเมล/เบอร์โทร) ผ่าน Captive Portal     │
│     ที่ ISE PSN ให้บริการ (แทนที่ WLC Local Web-Auth เดิม) → ได้รับ         │
│     Temporary Credential ทันที ไม่ต้องรอ Sponsor อนุมัติ                    │
├───────────────────────────────────────────────────────────────────────┤
│  3) MY DEVICES PORTAL (BYOD Onboarding)                                  │
│     พนักงานนำอุปกรณ์ส่วนตัว (Laptop/Mobile) มาลงทะเบียนเองผ่าน Portal       │
│     → ISE ทำ Native Supplicant Provisioning (ตั้งค่า 802.1X Profile        │
│     ให้อัตโนมัติ) และออก Certificate ผ่าน SCEP (ทบทวนจาก [Part 37]         │
│     (part-037-wireless-security-advanced.md#step-365))                   │
└───────────────────────────────────────────────────────────────────────┘
```

### CWA Flow เดิม (Part 37) เทียบกับ ISE-Centralized CWA

```
Part 37 (WLC-only Local Web-Auth)          Part นี้ (ISE-Centralized CWA)

Guest ──Associate──► WLC-1                 Guest ──Associate──► WLC-1
         │  Redirect ไปหน้า Portal                    │  RADIUS Access-Request
         │  ที่ WLC สร้างเอง                          ▼
         ▼                                    ISE-2 (PSN) ตอบ Access-Accept
   Login สำเร็จ → Internet                    + redirect-url ไปยัง ISE Guest Portal
   (ไม่มี Sponsor/Self-Reg แยก)                          │
                                              Guest Login/Register ที่ ISE Portal
                                                          │
                                              ISE ส่ง RADIUS CoA ให้ WLC-1
                                              (เหมือนหลักการ Part 37 Step 367
                                               แต่ Portal/Account อยู่ที่ ISE
                                               แทน WLC — จัดการรวมศูนย์ได้)
                                                          ▼
                                                   Internet Access เต็มรูปแบบ
```

| คุณสมบัติ | WLC Local Web-Auth (Part 37) | ISE Centralized Guest (Part นี้) |
|---|---|---|
| Portal Customization | จำกัดมาก (Template พื้นฐานของ WLC) | ปรับแต่งได้เต็มรูปแบบ (Logo, เงื่อนไข, ภาษา, Branding) |
| Guest Account Lifecycle | Admin สร้าง Local User ด้วยมือทุกครั้ง | Sponsor Portal / Self-Reg — จัดการวันหมดอายุอัตโนมัติ |
| ใช้ได้กับ Wired ด้วยไหม | ไม่ได้ (WLC ทำได้แค่ Wireless) | **ได้** — Policy Set เดียวใช้ได้ทั้ง Wired 802.1X (Part 44) และ Wireless |
| Reporting/Audit ว่าใครเป็น Guest เมื่อไหร่ | จำกัด (ต้องดู Log ที่ WLC เอง) | รวมศูนย์ที่ ISE MnT (Step 426) — ดู Guest Session ทั้งองค์กรจากจุดเดียว |

> **VLAN GUEST-DMZ (`10.10.50.0/24`) ที่ [Part 37]**(part-037-wireless-security-advanced.md#step-367)
> **สร้างไว้ยังใช้ต่อได้เหมือนเดิม** — Part นี้เปลี่ยนแค่ "ใครเป็นผู้ควบคุม Portal/Account" จาก WLC-1
> ไปเป็น ISE-2 (PSN) เท่านั้น ไม่ต้องรื้อ VLAN/Subnet ที่ทำไว้แล้ว

---

## Step 430 — Lab: ออกแบบสถาปัตยกรรม Security เต็มรูปแบบ FTD+FMC+ISE

> **หมายเหตุสำคัญ**: FMC และ ISE เป็น Appliance ที่บริหารผ่าน **Web GUI เป็นหลัก** (ไม่ใช่ IOS CLI
> แบบ Router/Switch) — Lab นี้จึงเป็น **แผนสถาปัตยกรรมเชิงแนวคิด (Conceptual/Architectural)** ที่
> สรุปทุก Decision จาก Step 421-429 ให้เป็นภาพเดียว พร้อมคำสั่ง Verification ที่ใช้งานจริงได้บน
> FTD CLI (ซึ่งยังมี Lina/ASA-style CLI อยู่จริง)

### ภาพรวมสถาปัตยกรรมสุดท้าย

```
                                   ┌───────────────┐
                                   │   INTERNET     │
                                   └───────┬───────┘
                                   ┌───────┴───────┐
                                   │  ISP-RTR       │
                                   └───────┬───────┘
                        ┌──────────────────┴──────────────────┐
                        │         WAN-EDGE-1 / WAN-EDGE-2        │
                        └──────────────────┬──────────────────┘
                                     ┌──────┴──────┐
                                     │    FW-1      │  FTDv, Transparent Mode
                                     │  Managed by  │  ACP: "FW1-EDGE-POLICY"
                                     │  FMC(.71)    │  (Step 422-424)
                                     └──────┬──────┘
        ┌──────────────────────────────────┴──────────────────────────────────┐
        │                          CORE-SW1 <===LACP Po1===> CORE-SW2           │
        └────┬─────────────────────────────────────────────────────────┬────┘
   ┌──────────┴──────────┐                                   ┌──────────┴──────────┐
   │  DIST-SW1 / DIST-SW2 │                                   │  DIST-SW3 / DIST-SW4 │
   └────┬────────────┬───┘                                   └────┬────────────┬───┘
┌────────┴───┐  ┌─────┴──────┐                             ┌────────┴───┐  ┌─────┴──────┐
│ ACCESS-SW1 │  │ ACCESS-SW2  │  802.1X (Preview Part 44)  │ ACCESS-SW3 │  │ ACCESS-SW4  │
└─────┬──────┘  └──────┬─────┘                             └─────┬──────┘  └──────┬─────┘
      │                │                                          │                │
   PC1-PC5          Phone/AP                                   Server1-3        WLC-1+AP
                                                                                     │
                                                          RADIUS (แทนที่ AAA-SRV Wireless Role)
                                                                                     │
                          ┌──────────────────────────────────────────────────────┴────┐
                          │   ISE-1 (PAN+MnT, .65)   +   ISE-2 (PSN, .66)                │
                          │   VLAN 99 (MGMT)                                             │
                          │   Policy Set: Wireless-Corporate / Wired-8021X / Guest-CWA    │
                          └──────────────────────────────────────────────────────────────┘

   AAA-SRV (10.10.99.60) ยังทำ TACACS+ Device Admin ต่อจาก Part 22 (Migration เป็น Phase 2 — ดูหมายเหตุ)
```

### ตาราง IP/Component สรุปทั้งหมดที่ Part นี้เพิ่มเข้ามา

| Component | IP (VLAN 99 MGMT) | บทบาท |
|---|---|---|
| FW-1 (FTDv) — Management | `10.10.99.70/24` | Data Plane อยู่ที่ Bridge Group ระหว่าง WAN-EDGE ↔ CORE-SW (Transparent Mode) |
| FMC | `10.10.99.71/24` | บริหาร FW-1 แบบ Centralized |
| ISE-1 (PAN + MnT) | `10.10.99.65/24` | Config Policy Set, ดู Live Log/Report |
| ISE-2 (PSN) | `10.10.99.66/24` | รับ RADIUS/TACACS+ Request จริงจาก WLC-1/Switch |
| AAA-SRV (เดิม) | `10.10.99.60/24` | ยังทำ TACACS+ Device Admin ต่อ (Part 22) — Optional Migrate ไป ISE ในอนาคต |

### แผนออกแบบ Policy Set เต็มรูปแบบ (สรุปจาก Step 427-429)

| Policy Set | ใช้กับ | Authentication | Authorization ผลลัพธ์ |
|---|---|---|---|
| **Wireless-Corporate-Access** | SSID `ENTERPRISE-SECURE` (Part 37) | EAP-TLS/PEAP ผ่าน AD | VLAN 40 (Employee) / VLAN 45+dACL (Contractor) / VLAN 20 (Cisco-IP-Phone จาก Profiling) |
| **Wireless-Guest-CWA** | SSID `GUEST-WIFI` (Part 37) | Guest Portal (Self-Reg/Sponsor) | VLAN 50 (GUEST-DMZ) + Redirect ACL จนกว่าจะ Login ผ่าน Portal |
| **Wired-8021X-Access** (Preview) | ACCESS-SW1-4 ทุก Port | 802.1X (รายละเอียดเต็ม Part 44) | VLAN ตาม Profiling + SGT (TrustSec เต็มรูปแบบ Part 44) |
| **Device-Admin-TACACS** | CORE-SW/DIST-SW SSH/Console | TACACS+ (ถ้า Migrate จาก AAA-SRV) | priv-lvl ตาม Role (เหมือน Part 22 Step 215-217) |

### Verification & Monitoring ที่วิศวกรต้องตรวจสอบจริง

**1) ฝั่ง FW-1 (FTD CLI — Lina-based เหมือน ASA)**

```
> show connection
5 in use, 12 most used
TCP OUTSIDE 203.0.113.10:443 INSIDE 10.10.30.30:52344, idle 0:00:02, bytes 4820, flags UIO

> show access-control-config
================[ FW1-EDGE-POLICY ]================
Rule 1: ALLOW-ROUTING-PROTOCOLS  — Hit Count: 184203
Rule 2: ALLOW-OUTBOUND-WEB       — Hit Count: 98211
Rule 3: ALLOW-INBOUND-DMZ-WEB    — Hit Count: 4102
Rule 4: DEFAULT-DENY             — Hit Count: 771 (Log)

> show asp drop
Flow drop:
  ips-fail-close                                                 0
  invalid-tcp-hdr-length                                         2
```

**2) ฝั่ง FMC Dashboard (Web GUI — คำอธิบายสิ่งที่วิศวกรดูทุกวัน)**

| หน้าจอ (Menu) | ใช้ตรวจสอบอะไร |
|---|---|
| **Analysis → Connection Events** | Traffic ทุก Flow ที่ผ่าน FW-1 พร้อม Rule ที่ Match — ใช้ Troubleshoot ว่า Traffic ถูก Block ที่ Rule ไหน |
| **Analysis → Intrusion Events** | Alert จาก Snort Engine (Step 424) — ดูว่ามี Attack Signature ไหนถูก Trigger บ่อยที่สุด |
| **Analysis → Files** | ประวัติไฟล์ที่ถูกตรวจ/Block จาก File & Malware Policy |
| **Health → Monitor** | สถานะ CPU/Memory/Interface ของ FW-1 เอง (เทียบเท่า `show processes cpu` บน IOS) |
| **Overview → Dashboard** | ภาพรวม Top Applications, Top Talkers, Threat Summary แบบ Real-time |

**3) ฝั่ง ISE Live Log (Web GUI — Operations Menu)**

| หน้าจอ (Menu) | ใช้ตรวจสอบอะไร |
|---|---|
| **Operations → RADIUS → Live Logs** | ทุก Authentication Request แบบ Real-time พร้อม Policy Set/Rule ที่ Match, ผลลัพธ์ Accept/Reject, VLAN/dACL ที่ได้รับ |
| **Operations → TACACS → Live Logs** | เหมือนกันแต่สำหรับ Device Admin (ถ้า Migrate มาจาก AAA-SRV) |
| **Context Visibility → Endpoints** | รายการ Endpoint ทั้งหมดที่ Profiling จับได้ พร้อม EndPointGroup ปัจจุบัน |
| **Operations → Reports → Guest Access** | ประวัติ Guest ทั้งหมดที่เคย Login ผ่าน Sponsor/Self-Reg Portal |

**ตัวอย่าง RADIUS Live Log ที่วิศวกรจะเห็นจริง (Concept):**

```
Time            Status   Identity        Endpoint ID        Policy Set              Authorization Rule
10:42:15.203    Pass     jsmith          AA:BB:CC:11:22:33   Wireless-Corporate       Rule1-Employee-Full
10:42:18.771    Pass     contractor01    AA:BB:CC:44:55:66   Wireless-Corporate       Rule2-Contractor-Restricted
10:42:30.055    Fail     guest_unknown   AA:BB:CC:77:88:99   Wireless-Guest-CWA       Default-DenyAccess
10:43:01.918    Pass     (unauthenticated) AA:BB:CC:AA:BB:CC Wireless-Corporate       Rule3-Cisco-IP-Phone
```

> แถวสุดท้ายคือตัวอย่างผลลัพธ์ของ **Profiling (Step 428)** ที่ทำงานร่วมกับ Policy Set —
> อุปกรณ์ที่ยังไม่ผ่าน 802.1X Authentication เต็มรูปแบบ (Cisco IP Phone ที่ใช้ MAB — MAC
> Authentication Bypass) ก็ยังถูกจัดกลุ่มและได้ Authorization Result ที่ถูกต้องจาก EndPointGroup
> ที่ Profiler จำแนกให้ — รายละเอียดเต็มรูปแบบของ MAB และ 802.1X บน Wired จะเรียนต่อใน **Part 44**

### สรุป Checklist Deploy สำหรับ Lab ของหลักสูตรนี้

| # | งาน | Step ที่อ้างอิง |
|---|---|---|
| 1 | ติดตั้ง FTDv แทน ASAv บน FW-1, ตั้ง Management IP `10.10.99.70` | Step 422 |
| 2 | ติดตั้ง FMC ที่ `10.10.99.71`, Register FW-1 เข้า FMC | Step 422 |
| 3 | ตั้ง FW-1 เป็น Transparent Mode, สร้าง Bridge Group ระหว่าง WAN-EDGE ↔ CORE-SW | Step 423 |
| 4 | สร้าง ACP "FW1-EDGE-POLICY" พร้อม Rule อนุญาต OSPF/BGP เป็นอันดับแรก | Step 424 |
| 5 | ผูก Intrusion Policy + File Policy เข้ากับ Rule ที่ Allow | Step 424 |
| 6 | ติดตั้ง ISE-1 (PAN+MnT, `.65`) และ ISE-2 (PSN, `.66`) | Step 426 |
| 7 | เขียน Policy Set แทน AAA-SRV Flat Config ของ Part 37 | Step 427 |
| 8 | เปิด Profiling Feed Service + สร้าง Policy รองรับ Cisco-IP-Phone/Printer | Step 428 |
| 9 | Migrate Guest Portal จาก WLC Local Web-Auth ไปยัง ISE Sponsor/Self-Reg Portal | Step 429 |
| 10 | เตรียม Policy Set "Wired-8021X-Access" ไว้เป็นโครง (ยังไม่ Enable จริง) รอ Part 44 | Step 430 |

---

## แบบฝึกหัดทวนความเข้าใจ Part 43

1. FTD ต่างจาก ASA อย่างไร และ FMC ต่างจาก FDM อย่างไร?
2. เพราะเหตุใด Part นี้จึงเลือกวาง FW-1 ในโหมด Transparent แทน Routed — อธิบายโดยอ้างอิงถึง
   OSPF/BGP ที่ Config ไว้ตั้งแต่ Part 11-31?
3. Access Control Policy (ACP) บน FTD ประเมิน Rule ตามลำดับอย่างไร และทำไม Rule ที่อนุญาต
   OSPF/BGP ต้องอยู่บนสุด?
4. อธิบายบทบาทของ PAN, MnT, และ PSN ใน ISE Deployment — Node ไหนที่รับ RADIUS Request จริงจาก
   WLC-1?
5. ISE Policy Set ต่างจาก AAA-SRV Flat Config (Part 37) อย่างไร ในเรื่องจำนวน Condition ที่ใช้
   ตัดสินใจ Authorization?

**เฉลย:**

1. **FTD** คือ Unified Image ที่รวม ASA Engine เดิม (Lina — Stateful Firewall/Routing/VPN) กับ
   Firepower Services (Snort Engine — IPS/Application/File/Malware) ไว้ในตัวเดียว ต่างจาก ASA ที่
   มีแค่ Stateful Firewall — **FMC** คือ Management Platform แบบ Centralized จัดการ FTD หลายตัว
   จากจุดเดียว เหมาะกับ Enterprise หลาย Site ส่วน **FDM** คือ Web GUI ในตัว FTD เอง จัดการได้
   ทีละตัว เหมาะกับ Site เดียว/Branch เล็ก
2. เพราะ Topology ตั้งแต่ Part 11-31 มี OSPF Area 0 และ BGP Peering ที่ WAN-EDGE-1/2 ↔
   CORE-SW1/2 ทำงานสมบูรณ์อยู่แล้วบน Subnet เดิมตาม IP Address Plan — ถ้าใช้ Routed Mode ต้อง
   Renumber Subnet และแก้ Neighbor Relationship ทั้งหมดที่ทำมา แต่ Transparent Mode ทำให้ FW-1
   เป็น Bridge ที่ "แทรกเข้า" ได้โดยไม่ต้องแก้ Subnet/Routing Config เดิมเลย
3. ACP ประเมิน Rule **จากบนลงล่างแบบ First-Match** (หยุดที่ Rule แรกที่ตรง) — Rule ที่อนุญาต
   OSPF/BGP ต้องอยู่บนสุดเพราะถ้า Routing Protocol Traffic ไปติด Default-Deny (ที่อยู่ล่างสุดเสมอ)
   ก่อน OSPF/BGP Neighbor Relationship ที่ทำมาตั้งแต่ Part 11-31 จะขาดทันที
4. **PAN (Policy Administration Node)** เป็นจุดที่ Admin Config Policy Set/Profiling/Posture
   และ Sync Config ไปยัง Node อื่น; **MnT (Monitoring Node)** เก็บ Log และเป็นจุดดู Live
   Log/Report/Troubleshoot; **PSN (Policy Service Node)** คือ Node ที่**รับ RADIUS/TACACS+
   Request จริง**จาก WLC-1/Switch แล้วประเมิน Policy Set ตอบ Accept/Reject/CoA — ในหลักสูตรนี้
   คือ ISE-2 (`10.10.99.66`)
5. AAA-SRV Flat Config ตัดสินใจจาก Condition เดียว (Username/Group) ทำให้ทุก User ใน Group
   เดียวกันได้ผลลัพธ์เหมือนกันเสมอไม่ว่าจะใช้อุปกรณ์อะไร เวลาไหน หรือผ่าน Posture หรือไม่ — ISE
   Policy Set ตรวจสอบ**หลาย Condition พร้อมกัน**ในกฎเดียว (เช่น AD Group + Device Type จาก
   Profiling + Time-of-Day + Posture Status) ทำให้ Authorization Result แม่นยำและปลอดภัยกว่ามาก
   โดยไม่ต้องสร้าง Group ใหม่ทุกครั้งที่มีเงื่อนไขเพิ่ม

---

## สรุป Part 43

Part นี้เปิดใช้งานสองส่วนที่หลักสูตรนี้เตรียมไว้ตั้งแต่ต้นให้ทำงานจริงเป็นครั้งแรก: (1) แปลง
**FW-1** จากแนวคิด ASA ให้เป็น **Firepower Threat Defense (FTD)** ที่บริหารผ่าน **FMC** เต็ม
รูปแบบ วางในโหมด **Transparent** เพื่อแทรกเข้า Topology ที่มี OSPF/BGP ทำงานอยู่แล้วโดยไม่ต้อง
Renumber, สร้าง **Access Control Policy** พร้อม Intrusion Prevention และ File/Malware Policy
ปกป้อง WAN Edge, และ (2) ยกระดับบทบาท RADIUS ของ **AAA-SRV** (Part 22/37) ไปสู่ **Cisco ISE**
เต็มรูปแบบ — ด้วย **PAN/MnT/PSN Architecture**, **Policy Set** ที่ประเมินหลาย Condition พร้อมกัน
แทน Flat Config เดิม, **Profiling/Posture** ที่จำแนกและตรวจสุขภาพ Endpoint อัตโนมัติ, และ
**Guest/BYOD Workflow** ที่รวมศูนย์กว่าการทำ Local Web-Auth บน WLC ✅

**พร้อมสำหรับ Part 44**: เราจะนำ ISE ที่สร้างเสร็จใน Part นี้ไปใช้งานจริงกับ **802.1X Port-Based
Access Control แบบ Wired เต็มรูปแบบ** บน ACCESS-SW1-4 (ต่อยอดจาก Preview ใน [Part 21]
(part-021-switch-security.md) และ [Part 37](part-037-wireless-security-advanced.md)) พร้อมเจาะลึก
**Cisco TrustSec/SGT** ที่ [Part 38](part-038-sd-access-fundamentals.md) บอกไว้ว่า "รายละเอียด
เชิงลึกเต็มรูปแบบอยู่ที่ Part 44" — ปิดท้ายด้วยการทำให้ Segmentation ของทั้งองค์กรทำงานจาก
"Identity" ล้วนๆ โดยไม่ผูกกับ IP/VLAN อีกต่อไป

**ไปต่อ:** [Part 44 — 802.1X & TrustSec →](part-044-dot1x-trustsec.md)
