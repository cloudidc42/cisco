# Part 78 — Security Integration (ISE, TrustSec, FTD, Umbrella)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 771–780 จาก 1000**

> ต่อจาก [Part 77 — SD-Access Advanced Design →](part-077-sd-access-advanced-design.md) ที่เจาะลึก
> การออกแบบ SD-Access Fabric ระดับ Advanced (Multi-site Transit, Border Design, Policy ข้าม Fabric)
> Part นี้เปลี่ยนโฟกัสจาก "โครงสร้าง Fabric" มาสู่ **การประกอบร่าง Security Stack ทั้งองค์กรให้เป็น
> สถาปัตยกรรมเดียว** — ตลอดหลักสูตรนี้เราสร้างชิ้นส่วน Security ไว้แยกกันหลาย Part:
> [Part 21](part-021-switch-security.md)/[22](part-022-aaa-device-hardening.md) (Switch Security/AAA),
> [Part 37](part-037-wireless-security-advanced.md) (802.1X Wireless/EAP-TLS PKI Preview),
> [Part 43](part-043-security-architecture-firepower-ise.md) (FTD+FMC, Cisco ISE เต็มรูปแบบ),
> [Part 44](part-044-dot1x-trustsec.md) (802.1X Wired, TrustSec/SGT, SGACL, SXP) — แต่ทุก Part ที่
> ผ่านมา **สอนแต่ละเทคโนโลยีแยกจากกัน** เหมือนแต่ละชิ้นทำงานอยู่ในกล่องของตัวเอง คำถามที่ Part นี้
> จะตอบคือ: **"ทำอย่างไรให้ ISE, TrustSec, FTD ทำงาน 'ร่วมกันจริง' ไม่ใช่แค่ 'อยู่ในไดอะแกรมเดียวกัน'
> และเมื่อเพิ่ม Cisco Umbrella เข้ามาเป็นชั้นป้องกันบน Cloud แล้ว ทั้ง 4 ระบบจะประสานงานกันเป็น
> Automated Response ได้อย่างไร"** — นี่คือระดับความคิดแบบ **CCIE Security/Enterprise** ที่มองข้าม
> การ Config อุปกรณ์ทีละตัว ไปสู่การออกแบบ **Ecosystem** ที่ทุกชิ้นคุยกันผ่าน API/Protocol กลาง
> (pxGrid) และตอบสนองภัยคุกคามแบบอัตโนมัติโดยไม่ต้องมี Admin คอยกดปุ่มเอง

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 771 | ภาพรวม Enterprise Security Stack และแผนผังการ Integration — ISE/TrustSec/FTD/Umbrella ทำงานร่วมกันอย่างไร |
| 772 | Cisco Umbrella พื้นฐาน — DNS-Layer Security, Secure Web Gateway, CASB |
| 773 | Umbrella Deployment Models — Roaming Client, Umbrella for Network, เชื่อมกับ SD-WAN Cloud onRamp for SaaS |
| 774 | ISE + TrustSec + FTD Integrated Enforcement — Worked Example: SGT เดียวใช้ Policy ทั้ง Network และ Firewall |
| 775 | pxGrid (Platform Exchange Grid) — กลไกแชร์ Context แบบ Real-time ระหว่างแพลตฟอร์ม Security |
| 776 | Threat Response Integration — SecureX/XDR และ Rapid Threat Containment (RTC) |
| 777 | Zero Trust Preview — Stack นี้ Implement Zero Trust Principle อะไรไปแล้วบ้าง (เชื่อมโยง Part 94) |
| 778 | Unified PKI Infrastructure สำหรับทั้ง Security Stack |
| 779 | Compliance & Audit — PCI-DSS Segmentation, Logging/Accounting |
| 780 | Lab: เอกสารสถาปัตยกรรม Security แบบบูรณาการเต็มรูปแบบ + Rapid Threat Containment Scenario |

---

## Step 771 — ภาพรวม Enterprise Security Stack และแผนผังการ Integration

### ปัญหาของ "Point Solution" ที่ทุก Part ก่อนหน้าอาจทำให้เข้าใจผิด

ตลอด Part 21-44 เราสร้างเทคโนโลยี Security แต่ละตัวแบบ **เรียงตามลำดับเวลา** (Chronological) —
สร้าง AAA ก่อน (Part 22), ต่อด้วย Wireless 802.1X (Part 37), ต่อด้วย FTD+ISE (Part 43), ปิดท้าย
ด้วย TrustSec/SGT (Part 44) วิธีเรียงแบบนี้ทำให้ผู้เรียนบางคนเข้าใจผิดว่าแต่ละระบบ **ทำงานแยกกัน
เป็นอิสระ** แค่บังเอิญอยู่ใน Topology เดียวกัน — ความจริงคือทั้ง 4 ระบบถูก **ออกแบบมาให้พึ่งพากัน
โดยตรง** และ Part นี้คือจุดที่เราพิสูจน์ว่ามันไม่ใช่ Coincidence

| ระบบ | บทบาทหลัก | สร้างไว้ที่ | สิ่งที่ระบบอื่น "ใช้ต่อ" จากมัน |
|---|---|---|---|
| **Cisco ISE** | Policy Engine กลาง — ตัดสินใจว่าใคร/อะไรควรได้สิทธิ์อะไร | Part 43 | เป็น "แหล่งความจริง" ของ Identity ที่ TrustSec, FTD, และ Umbrella ทั้งหมดอ้างอิง |
| **TrustSec/SGT** | Micro-Segmentation ที่ผูก Policy กับ "กลุ่ม" ไม่ใช่ IP | Part 44 | SGT ที่ ISE แจกให้ ถูกส่งต่อให้ FTD Enforce เป็น Firewall Policy (Step 774) |
| **FTD/FMC** | Perimeter + Internal NGFW — Inspect Traffic เชิงลึก | Part 43 | รับ Context (User/SGT) จาก ISE ผ่าน pxGrid มาช่วยตัดสินใจ Policy แม่นยำกว่า IP-only |
| **Cisco Umbrella** | DNS-layer Security + SWG/CASB บน Cloud | Part นี้ (ใหม่) | ป้องกัน "ก่อน" Connection เกิดขึ้นจริงเลย เป็นชั้นป้องกันที่ FTD มองไม่เห็นด้วยซ้ำ (Step 772) |

### แผนผัง Integration แบบเต็ม — "ใครคุยกับใคร" และ "ผ่าน Protocol/API อะไร"

```
                          ┌─────────────────────────────────────────┐
                          │              CISCO UMBRELLA (Cloud)          │
                          │   DNS Security + SWG + CASB (Step 772-773)   │
                          └───────────────┬─────────────┬─────────────┘
                     DNS Query (ทุก Client/Branch)      │ API/Cloud Integration
                                          │              │ (Threat Intel, SecureX — Step 776)
                                          ▼              ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                                  ENTERPRISE CAMPUS/WAN                            │
│                                                                                     │
│   PC/Endpoint ──802.1X/MAB──► ACCESS-SW ──SXP/Inline──► DIST-SW ──► CORE-SW        │
│        │        (Part 44)         │                        │                      │
│        │ RADIUS                    │ SGT Enforcement          │                     │
│        ▼                          ▼ (SGACL — Part 44)         │                     │
│   ┌─────────────────────┐                                     ▼                    │
│   │   ISE (PAN/MnT/PSN)    │◄──────────pxGrid (Step 775)──────► FW-1 (FTD/FMC)       │
│   │   - Identity            │   (Session, SGT, ANC/Quarantine)   - ACP ใช้ SGT        │
│   │   - Posture             │                                    Condition (774)     │
│   │   - TrustSec Server     │◄──────────pxGrid───────────────► SIEM/XDR (SecureX)    │
│   └─────────────────────┘                                     (Step 776)            │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 3 มุมมองที่ทำให้เห็นว่านี่คือ "สถาปัตยกรรมเดียว" ไม่ใช่ 4 ระบบที่บังเอิญอยู่ด้วยกัน

1. **Shared Identity**: ทั้ง TrustSec (SGT), FTD (Policy Condition), และ Umbrella (User/Group
   Reporting ผ่าน AD/ISE Integration) ใช้ "ตัวตน" เดียวกันที่มาจาก ISE — ไม่มีระบบไหนสร้าง Identity
   ของตัวเองซ้ำ
2. **Shared Enforcement Chain**: การบล็อก Endpoint ที่มีปัญหาไม่ได้เกิดจากจุดเดียว — SGACL บล็อกที่
   Layer 3 Segmentation, FTD บล็อกที่ Layer 4-7 Inspection, Umbrella บล็อกที่ DNS Layer ก่อนแม้แต่
   จะเริ่ม Connection — ทั้ง 3 ชั้นทำงานสอดคล้องกันจาก Policy เดียวกัน (Defense-in-Depth ที่ทวนจาก
   [Part 43 Step 421](part-043-security-architecture-firepower-ise.md))
3. **Shared Response Loop**: เมื่อเกิดภัยคุกคาม การตอบสนอง (Quarantine/Block) ไม่ต้องให้ Admin
   เข้าไปกดที่แต่ละระบบทีละตัว — pxGrid + CoA ทำให้ระบบหนึ่งสั่งอีกระบบเปลี่ยน Policy ได้แบบ
   Real-time (Step 776)

> **สรุปหลักการของ Part นี้**: ถ้า Part 43/44 สอน "วิธีสร้าง" ISE/TrustSec/FTD แต่ละตัว Part นี้
> สอน **"วิธีทำให้มันเป็นระบบเดียว"** — ความแตกต่างระหว่าง Enterprise ที่ "มี" Security Product
> หลายตัว กับ Enterprise ที่มี **Security Architecture** ที่แท้จริง อยู่ที่การ Integration นี่เอง

---

## Step 772 — Cisco Umbrella พื้นฐาน: DNS-Layer Security, Secure Web Gateway, CASB

### ปัญหาที่ FTD (Part 43) แก้ไม่ได้เต็มที่: Roaming User และ "ก่อน" Connection เกิด

FTD ทำงานได้ดีมากกับ Traffic ที่ **ผ่าน** ตัวมันจริง (Inline Inspection ที่ Edge ตาม
[Part 43 Step 423](part-043-security-architecture-firepower-ise.md)) แต่มี 2 จุดบอดที่ FTD ตัวเดียว
แก้ไม่ได้:

| จุดบอดของ Perimeter Firewall (FTD) | สิ่งที่ Cisco Umbrella แก้ |
|---|---|
| พนักงานที่ทำงานนอกออฟฟิศ (Roaming User ต่อ Wi-Fi ร้านกาแฟ/บ้าน) ไม่ได้ผ่าน FW-1 เลย | Umbrella ทำงานที่ **DNS Layer** ซึ่งเกิด**ก่อน**การเชื่อมต่อจริงเสมอ ไม่ว่า Client จะอยู่ที่ไหนในโลก |
| FTD ต้อง Inspect Payload เต็ม Packet ก่อนตัดสินใจ Block (ทำงานหลัง Connection เริ่มแล้ว) — ถ้า Malware โหลดเร็วมากอาจเนียนผ่านไปได้บางส่วน | Umbrella บล็อกที่ขั้นตอน **DNS Resolution** — ถ้า Domain เป็นอันตราย Client จะ **ไม่ได้ IP Address ปลายทางเลย** ไม่มีการเชื่อมต่อเกิดขึ้นแม้แต่ Packet เดียว |
| SaaS Traffic (Office 365, Dropbox, Salesforce) ที่ไม่ได้วิ่งผ่าน WAN Edge เดิม (Direct Internet Access จาก Branch) | Umbrella ทำ **CASB** (Cloud Access Security Broker) มองเห็น SaaS App ที่ใช้งานอยู่แม้ Traffic ไม่ผ่าน On-prem Firewall เลย |

### สถาปัตยกรรม 3 ชั้นของ Cisco Umbrella

```
┌───────────────────────────────────────────────────────────────────────────┐
│                            CISCO UMBRELLA (Cloud-Delivered)                    │
├───────────────────────────────────────────────────────────────────────────┤
│  ชั้น 1 — DNS-Layer Security                                                  │
│    ทุก DNS Query ของ Client วิ่งไปที่ Umbrella Recursive DNS ก่อนเสมอ            │
│    เทียบ Domain กับ Cisco Talos Threat Intelligence (ฐานเดียวกับ Snort ของ FTD)  │
│    Malicious Domain → NXDOMAIN/Block Page ทันที, ไม่มี IP ตอบกลับมาเลย            │
├───────────────────────────────────────────────────────────────────────────┤
│  ชั้น 2 — Secure Web Gateway (SWG)                                            │
│    ถ้า Domain "ไม่ชัดเจน" (Risk ปานกลาง) → Proxy Traffic HTTP/HTTPS ผ่าน Umbrella│
│    เพื่อทำ Full Inspection (คล้าย IPS/URL Filtering ของ FTD แต่ทำงานบน Cloud)     │
├───────────────────────────────────────────────────────────────────────────┤
│  ชั้น 3 — CASB (Cloud Access Security Broker)                                 │
│    มองเห็นและควบคุม SaaS App ที่องค์กรใช้ (Shadow IT Discovery, Data Loss        │
│    Prevention ระดับ Cloud App, Block "การใช้ App ส่วนตัว" เช่น Personal Gmail)  │
└───────────────────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบ Umbrella กับ FTD แบบตาราง — "เสริม" ไม่ใช่ "แทน"

| มิติ | FTD (Perimeter/On-prem, Part 43) | Cisco Umbrella (Cloud-delivered) |
|---|---|---|
| ตำแหน่งที่ Inspect | เฉพาะ Traffic ที่ผ่าน FW-1 จริง (Physical Location) | ทุกที่ที่ Client อยู่ (Global — ผ่าน DNS) |
| จุดที่ Block | หลัง Connection เริ่ม (Packet/Session Level) | **ก่อน** Connection เริ่ม (DNS Resolution Level) |
| ความเร็วในการ Deploy Protection ใหม่ | ต้อง Deploy Rule ผ่าน FMC ไปยัง FTD ตัวที่เกี่ยวข้อง | Cloud-wide — Threat Intelligence อัปเดตทันทีทุก Location พร้อมกัน |
| Depth of Inspection | สูงสุด (Full Packet, IPS, File/Malware — Step 424 ของ Part 43) | ปานกลาง-สูง (DNS + SWG Proxy) แต่ไม่เห็น Payload เท่า FTD ถ้า Traffic ไม่ Proxy |
| ใช้กับ Roaming User ได้ไหม | ไม่ได้ (ต้องผ่าน VPN กลับมาที่ FW-1 เท่านั้น) | ได้เต็มรูปแบบผ่าน Roaming Client (Step 773) |
| License/Data Plane | ต้องมี Appliance/VM เอง | Cloud SaaS ล้วนๆ ไม่มี Appliance ที่ต้อง Patch/Maintain (ยกเว้น Virtual Appliance สำหรับ Branch — Step 773) |

> **หลักการที่ต้องจำสำหรับข้อสอบ CCIE Security**: Umbrella **ไม่ได้แทนที่** FTD และ FTD
> **ไม่ได้แทนที่** Umbrella — ทั้งสองเป็นชั้นป้องกันที่ **ซ้อนกัน** ตามหลัก Defense-in-Depth
> Traffic ที่ผ่าน FW-1 ปกติจะถูกกรองทั้ง DNS Layer (Umbrella, ถ้า Config ให้ On-prem DNS Forward
> ไป Umbrella — Step 773) **และ** Packet/Application Layer (FTD) ซ้อนกันสองชั้น — ส่วน Roaming
> User ที่ไม่ผ่าน FW-1 เลยจะยังมี Umbrella คุ้มครองอยู่ชั้นเดียว ดีกว่าไม่มีอะไรเลย

---

## Step 773 — Umbrella Deployment Models: Roaming Client, Umbrella for Network, และ SD-WAN Cloud onRamp

### 2 รูปแบบการ Deploy หลัก ตาม "ตำแหน่ง" ของ Endpoint

| Deployment Model | ใช้กับ | วิธีบังคับ DNS ให้วิ่งไป Umbrella |
|---|---|---|
| **Umbrella Roaming Client** | Laptop พนักงานที่พกออกไปนอกออฟฟิศ (Work From Home, ร้านกาแฟ, เดินทาง) | ติดตั้ง Roaming Client (ส่วนหนึ่งของ **Cisco Secure Client** โมดูลเดียวกับที่ [Part 37]
(part-037-wireless-security-advanced.md) ใช้ทำ Posture) — Client เปลี่ยน DNS Resolver ของ OS ให้ชี้ไป Umbrella โดยตรง ไม่ว่าจะต่อ Network ไหน |
| **Umbrella for Network (Branch/Campus)** | Site ทั้งหมด (Campus, Branch) ที่มี Client จำนวนมากใช้ Infrastructure DNS กลาง | 2 วิธีย่อย: (1) **DNS Forwarding** — DHCP แจก DNS Server เป็น Umbrella Virtual Appliance (VA) ภายใน Site แทน DNS Server เดิม (2) **Network Tunnel (IPsec)** — WAN Edge สร้าง IPsec Tunnel ไปยัง Umbrella Cloud โดยตรง ส่ง Traffic ทั้งหมด (ไม่ใช่แค่ DNS) ไปให้ Umbrella Proxy/Inspect |

### วาง Umbrella Virtual Appliance (VA) ในหลักสูตรนี้ — ต่อจาก DNS Server เดิมของ Part 16

[Part 16](part-016-dhcp-dns.md) สร้าง DNS Server ภายในไว้แล้วสำหรับ Resolve Internal Hostname
(`*.company.local`) — Umbrella VA **ไม่ได้แทนที่** DNS Server นี้ แต่ทำงานเป็น **Forwarder ชั้น
กลาง**: Query สำหรับ Internal Domain ยังตอบจาก DNS Server เดิม แต่ Query สำหรับ Internet Domain
จะถูก VA ส่งต่อไปยัง Umbrella Cloud เพื่อตรวจสอบก่อนตอบกลับ Client

| Component ใหม่ที่เพิ่มใน Part นี้ | IP (VLAN 99 MGMT) | หมายเหตุ |
|---|---|---|
| **Umbrella-VA-1** | `10.10.99.72/24` | Virtual Appliance ตัวหลัก — ตั้งเป็น Conditional Forwarder ที่ DNS Server เดิม (Part 16) |
| **Umbrella-VA-2** | `10.10.99.73/24` | ตัวสำรอง (Cisco แนะนำอย่างน้อย 2 VA เสมอเพื่อ Redundancy — ถ้า VA ตัวเดียวล่ม Client จะ Resolve DNS ไม่ได้เลย) |

```
Flow การ Resolve DNS หลัง Deploy Umbrella VA (Umbrella for Network Model)

PC1 (10.10.10.11) ──DNS Query──► Internal DNS Server (Part 16, 10.10.30.x)
                                            │
                          ┌─────────────────┴─────────────────┐
                          │                                     │
                 Domain = *.company.local              Domain = Internet (เช่น facebook.com)
                          │                                     │
                    ตอบจาก Local Zone                Forward ไปยัง Umbrella-VA-1/2
                    (เหมือน Part 16 เดิม)                   (10.10.99.72/73)
                                                                 │
                                                        Umbrella-VA ส่งต่อ Query ไปยัง
                                                        Umbrella Cloud (Anycast DNS)
                                                                 │
                                                    ┌────────────┴────────────┐
                                                    │                          │
                                            Domain ปลอดภัย              Domain อันตราย/Policy Block
                                                    │                          │
                                            ตอบ IP จริงกลับมา          NXDOMAIN หรือ Block Page IP
                                            (Client เชื่อมต่อได้ปกติ)   (Client เชื่อมต่อไม่ได้เลย)
```

### เชื่อมกับ SD-WAN Cloud onRamp for SaaS (Part 76) — เมื่อ Branch ไม่ผ่าน WAN Edge กลับมา Campus

[Part 76](part-076-sd-wan-advanced-cloud-onramp.md) ออกแบบ **Cloud onRamp for SaaS** ให้ Branch
ที่ใช้ SD-WAN สามารถส่ง Traffic ของ SaaS App (Office 365, Salesforce) **ตรงออก Internet จาก
Branch เอง** (Direct Internet Access — DIA) โดยไม่ต้อง Backhaul ผ่าน FW-1 ที่ Campus — วิธีนี้
เร็วกว่ามากแต่สร้างจุดบอดใหม่: **Traffic DIA นี้ไม่ผ่าน FW-1 เลย** ถ้าไม่มีการป้องกันเพิ่ม

**Umbrella คือคำตอบของช่องโหว่นี้โดยตรง**: WAN Edge Router ที่ทำ Cloud onRamp for SaaS สามารถ
สร้าง **IPsec Tunnel ไปยัง Umbrella Cloud** (Network Tunnel Model) ควบคู่กับ DIA — ทำให้ Traffic
ที่ออกตรงจาก Branch ยังถูก Inspect ที่ DNS/SWG Layer ก่อนออก Internet จริง แม้จะไม่ผ่าน FW-1
ที่ Campus เลยก็ตาม

```
Branch Router (SD-WAN, Cloud onRamp for SaaS — Part 76)
        │
        ├──DIA (Direct)──► Office 365 / SaaS App โดยตรง (เร็วที่สุด)
        │                          ▲
        │                          │ ก่อนออก Internet จริง Traffic ผ่าน Umbrella ก่อนเสมอ
        │                          │
        └──IPsec Tunnel──► Umbrella Cloud (Network Tunnel — Step นี้)
                            - Inspect DNS + Web Traffic ของ DIA ทั้งหมด
                            - ไม่ต้อง Backhaul กลับ Campus/FW-1 เลย
```

> **ข้อสอบ CCIE Enterprise ชอบถามจุดนี้**: "องค์กรเปิด Direct Internet Access (DIA) ที่ Branch
> แล้ว จะป้องกัน Traffic นั้นอย่างไรโดยไม่เสียประโยชน์ด้านความเร็วของ DIA" — คำตอบคือ **Umbrella
> Network Tunnel** เพราะมันให้ Security ระดับ Cloud-delivered โดยไม่ต้อง Backhaul Traffic กลับไป
> ผ่าน Perimeter Firewall แบบเดิม ซึ่งจะทำให้เสียประโยชน์ของ SD-WAN DIA ไปเลย

---

## Step 774 — ISE + TrustSec + FTD Integrated Enforcement: Worked Example เต็มรูปแบบ

### จาก "Co-located" สู่ "Integrated" — ความต่างที่ Step นี้จะพิสูจน์ด้วย Config จริง

[Part 44](part-044-dot1x-trustsec.md) สอนให้ SGT ถูกแจกโดย ISE ผ่าน 802.1X แล้ว Enforce ด้วย
SGACL ที่ระดับ Switch (DIST-SW3) — [Part 43](part-043-security-architecture-firepower-ise.md)
สอนให้ FTD เขียน Access Control Policy (ACP) โดย Match ด้วย Zone/Network/Port ธรรมดา — ถ้าหยุด
อยู่แค่นี้ ทั้งสองระบบก็ยังเป็นแค่ "อยู่ในไดอะแกรมเดียวกัน" เท่านั้น **สิ่งที่ทำให้เป็น
Integration จริง** คือการที่ **FTD เขียน Rule โดย Match ด้วย SGT ตัวเดียวกันที่ ISE แจกให้ผ่าน
802.1X** — ไม่ใช่แค่ IP/Subnet อีกต่อไป

### สถานการณ์: PC1 (SALES_SGT=100) พยายามเข้าถึง Internet ผ่าน FW-1

```
ลำดับเหตุการณ์เต็ม (End-to-End) — จาก 802.1X Authenticate จนถึง FTD ตัดสินใจ

1) PC1 เสียบสาย ACCESS-SW1 Gi1/0/1 → 802.1X Authenticate ผ่าน ISE-1 (Part 44 Step 433-434)
   → ISE-1 ตอบ Access-Accept พร้อม cisco-av-pair="cts:security-group-tag=0064-01" (SGT=100)

2) ACCESS-SW1 สร้าง IP-SGT Binding: 10.10.10.11 = SGT 100 (SALES_SGT)
   → Binding นี้เดินทางผ่าน SXP/Inline Tagging ไปถึง CORE-SW1/2 (Part 44 Step 437-439)

3) ISE-1 (ผ่าน pxGrid — Step 775) ประกาศ Session นี้ให้ FMC รับรู้แบบ Real-time:
   "User=employee01, IP=10.10.10.11, SGT=100 (SALES_SGT), Device=Corporate-Laptop"

4) PC1 เปิด Browser ไป Internet → Traffic ผ่าน FW-1 (Transparent Mode, Part 43 Step 423)
   → FTD ตรวจ Packet: Source IP = 10.10.10.11
   → FTD (ผ่าน Identity/SGT Mapping ที่ได้จาก pxGrid) รู้ว่า Packet นี้มี SGT = 100
   → ACP Rule ที่ Match "Source SGT = SALES_SGT" ถูกใช้ตัดสินใจ (ไม่ใช่ Rule ที่ Match แค่
     Source Network = 10.10.10.0/24 แบบเดิม)
```

### เขียน Access Control Policy บน FMC โดย Match ด้วย SGT (แทนที่แนวคิด IP-based เดิม)

FMC รองรับ Condition ประเภท **"Source SGT/ISE Attributes"** ใน ACP Rule (เพิ่มเข้ามาจาก Rule
เดิมของ [Part 43 Step 424](part-043-security-architecture-firepower-ise.md)) — ตัวอย่าง Rule ใหม่
ที่แทนที่ Rule แบบเดิมที่ Match ด้วย Subnet:

```
โครงสร้าง ACP Rule ใหม่บน FMC — Match ด้วย SGT จาก ISE (ผ่าน pxGrid)

┌─────────────────────────────────────────────────────────────────────┐
│  ACCESS CONTROL POLICY: "FW1-EDGE-POLICY" (ต่อจาก Part 43 Step 424)     │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 2b: SALES-RESTRICTED-WEB (แทรกก่อน Rule 2 เดิม — ต้อง Specific กว่า)│
│    Condition:                                                          │
│      Zone(INSIDE→OUTSIDE)                                              │
│      Source SGT = SALES_SGT (100)     <- Match ด้วย SGT ที่ ISE แจกให้    │
│      URL Category ≠ (Business, News, Reference)                        │
│    Action: Block                                                        │
│    เหตุผล: SALES ต้องเข้า Web ได้เฉพาะหมวดที่เกี่ยวงาน                       │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 2c: CONTRACTOR-WEB-BLOCKED (SGT=CONTRACTOR จาก Part 44)           │
│    Condition: Source SGT = CONTRACTOR, Destination = any                │
│    Action: Block with Reset                                             │
│    เหตุผล: Contractor ไม่ควรออก Internet ผ่าน FW-1 เลย ตาม Policy บริษัท     │
├─────────────────────────────────────────────────────────────────────┤
│  Rule 2 (เดิมจาก Part 43): ALLOW-OUTBOUND-WEB (สำหรับ SGT อื่นที่ไม่ถูกจับ) │
└─────────────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบ: ถ้า PC1 ถูกย้ายไปเสียบที่ ACCESS-SW3 (คนละ Subnet) — พิสูจน์ประโยชน์ของ Integration

| สถานการณ์ | ACP แบบ IP-based (ไม่ Integrate กับ ISE) | ACP แบบ SGT-based (Integrate ผ่าน pxGrid — Step นี้) |
|---|---|---|
| PC1 ย้ายจาก ACCESS-SW1 (VLAN 10) ไป ACCESS-SW3 (สมมติมี VLAN 10 ด้วย แต่คนละ Subnet Site) | ต้องแก้ ACP Rule ใหม่ให้ครอบ Subnet ปลายทางใหม่ | **ไม่ต้องแก้ ACP เลย** — PC1 Authenticate ใหม่ที่พอร์ตใหม่ได้ SGT=100 เหมือนเดิม, FTD เห็น SGT เดิมผ่าน pxGrid ทันที |
| Admin ต้องการดูว่า "employee01" ใช้ Internet อะไรบ้าง | ต้องเทียบ IP กับ DHCP Log แยกต่างหากเอง | FMC แสดง **User/SGT Context** ตรงใน Connection Event เลย (มาจาก pxGrid Identity Mapping) |
| Security Team อยากเพิ่มกลุ่มใหม่ "FINANCE_SGT" ที่ต้อง Block เข้า Internet ทั้งหมด | ต้องรู้ Subnet ของ Finance ทุก Site ก่อนเขียน ACL | เพิ่ม SGT ใหม่ที่ ISE + เขียน ACP Rule เดียว Match ด้วย SGT นั้น ใช้ได้ทุก Site ทันที |

> **จุดสำคัญที่สุดของ Step นี้**: การที่ FTD "รู้จัก" SGT ของ Traffic ที่ผ่านมันได้ **ไม่ใช่เพราะ
> FTD คำนวณเอง** — FTD ไม่มีทางรู้ว่า IP 10.10.10.11 มี SGT อะไรถ้าไม่มีใครบอกมัน สิ่งที่ทำให้มัน
> "รู้" คือ **pxGrid** ที่ ISE Subscribe ข้อมูล Session/SGT ให้ FMC แบบ Real-time — นี่คือกลไกที่
> Step 775 จะอธิบายรายละเอียดเชิงเทคนิคทั้งหมด

---

## Step 775 — pxGrid (Platform Exchange Grid): กลไกแชร์ Context แบบ Real-time

### pxGrid คืออะไร — นิยามที่ต้องแม่นสำหรับข้อสอบ

**Cisco Platform Exchange Grid (pxGrid)** คือ **Framework การแชร์ข้อมูล Context แบบ Publish/
Subscribe** ที่ให้ ISE (ในฐานะ "เจ้าของ" ข้อมูล Identity/Session/SGT) กระจายข้อมูลนั้นไปยัง
Platform Security อื่นๆ แบบ **Real-time** โดยไม่ต้องให้แต่ละ Platform ไป Query ISE ทีละครั้ง
(Polling) ซึ่งช้าและ Scale ไม่ได้

```
สถาปัตยกรรม pxGrid แบบง่าย

                        ┌─────────────────────────────┐
                        │   ISE — pxGrid Node (Persona)   │  <- Persona ที่ 4 เพิ่มจาก
                        │   (เพิ่มจาก PAN/MnT/PSN         │      PAN/MnT/PSN (Part 43 Step 426)
                        │    ของ Part 43 Step 426)        │
                        └───────────────┬─────────────┘
                     Publish Topic (WebSocket/STOMP, TLS Mutual Auth)
              ┌──────────────────┼──────────────────┬──────────────────┐
              ▼                  ▼                  ▼                  ▼
      ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
      │      FMC          │ │  Cisco Umbrella  │ │  SIEM/XDR         │ │  Third-party NAC/  │
      │  (Subscriber)      │ │  (ผ่าน Cloud API,│ │  (SecureX/Splunk) │ │  Threat Intel      │
      │  Step 774           │ │   ไม่ใช่ pxGrid   │ │  Step 776           │ │  (Subscriber)       │
      │                     │ │   โดยตรง)          │ │                     │ │                     │
      └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘
```

> **ข้อสังเกตสำคัญ**: Umbrella ไม่ได้เชื่อมกับ ISE ผ่าน pxGrid โดยตรงแบบ FMC — Umbrella เป็น
> Cloud Service ที่ Integrate ผ่าน **Cloud-to-Cloud API/SecureX** เป็นหลัก (Step 776) ส่วน pxGrid
> เองเป็น Protocol ที่ออกแบบมาสำหรับ Platform ที่อยู่ On-prem/Private เชื่อมกับ ISE โดยตรงเป็นหลัก
> เช่น FMC, SIEM, และ Third-party NAC — ข้อสอบ CCIE Security ชอบแยกจุดนี้ให้ชัดว่า **"pxGrid ≠
> ทุก Integration ของ ISE"**

### Topic หลักที่ pxGrid แจกจ่าย (สิ่งที่ FMC "Subscribe" ไปใช้จริงใน Step 774)

| pxGrid Topic | ข้อมูลที่ส่ง | ใครใช้ประโยชน์ |
|---|---|---|
| **Session Directory** | User/MAC/IP/SGT/Posture Status ของทุก Session ที่ Authenticate ผ่าน ISE แบบ Real-time | FMC (แปลง IP → User/SGT ให้ ACP Match ได้ — Step 774), SIEM (สืบสวน Incident ว่า "IP นี้คือใคร") |
| **TrustSec Metadata (SXP/SGT)** | รายชื่อ SGT ทั้งหมดที่ ISE นิยามไว้ พร้อมชื่อ/คำอธิบาย | ให้ Platform อื่นแสดงชื่อ SGT (เช่น "SALES_SGT") แทนเลข 100 เฉยๆ ในหน้า Report |
| **EndpointProfileMeta** | ผลลัพธ์ Profiling ของ ISE (Part 43 Step 428) — "อุปกรณ์นี้คือ Cisco-IP-Phone" | SIEM ใช้จัดกลุ่ม Alert ตามประเภทอุปกรณ์ |
| **ANC (Adaptive Network Control)** | **ไม่ใช่ข้อมูลที่ ISE Publish** แต่เป็น **Action ที่ Platform อื่นสั่งกลับมาที่ ISE ได้** ผ่าน pxGrid (เช่น "Quarantine MAC นี้ทันที") | จุดที่ทำให้ Step 776 (Rapid Threat Containment) เกิดขึ้นได้จริง |

### ทำไม pxGrid ต้องใช้ Mutual Certificate Authentication เสมอ

pxGrid ไม่ใช่ REST API ธรรมดาที่ยืนยันด้วย API Key — มันบังคับให้ทั้ง **ISE pxGrid Node และ
Client (FMC/SIEM)** ต้องมี **Certificate ที่เชื่อถือร่วมกัน** (Mutual TLS) ก่อนจะ Subscribe/
Publish อะไรได้เลย เหตุผลคือข้อมูลที่ไหลผ่าน pxGrid **มีความ Sensitive สูงมาก** (Identity ของ
พนักงานทุกคนในองค์กร) — ถ้าใครขโมย Credential ธรรมดาไปปลอมตัวเป็น Client ได้ จะเห็น Identity
ทั้งองค์กรทันที การบังคับ Certificate จึงเป็นการป้องกันอีกชั้น (รายละเอียดการออกแบบ PKI ที่รองรับ
Certificate เหล่านี้อยู่ใน Step 778)

```
ขั้นตอน Register FMC เป็น pxGrid Client (Concept)

1) บน ISE: Administration → pxGrid Services → เปิด pxGrid Persona
2) บน FMC: System → Integration → Identity Sources → ระบุ ISE pxGrid Node IP
3) FMC ส่ง Certificate Request (CSR) หรือใช้ Certificate ที่ออกจาก CA เดียวกัน (Step 778)
4) บน ISE pxGrid Services: เห็น Client "FMC" ขอ Subscribe → Admin กด "Approve"
   (ค่า Default ต้อง Approve ด้วยมือครั้งแรกเสมอ — ป้องกัน Client ปลอมมา Subscribe อัตโนมัติ)
5) หลัง Approve: FMC เริ่มรับ Session Directory Update แบบ Real-time ทันที
```

> **จุดที่ข้อสอบชอบถาม**: "เพิ่ม pxGrid Client ใหม่แล้วทำไมยังไม่เห็นข้อมูล" — คำตอบที่พบบ่อยที่สุด
> คือ **ยังไม่ได้ Approve Client บนหน้า pxGrid Services ของ ISE** (Step 4 ด้านบน) ซึ่งเป็นขั้นตอน
> Manual โดยตั้งใจเพื่อความปลอดภัย ไม่ใช่ Bug

---

## Step 776 — Threat Response Integration: SecureX/XDR และ Rapid Threat Containment

### SecureX/XDR — แนวคิด "มองภัยคุกคามเป็นภาพเดียว" ข้าม Platform

ปัญหาที่ Enterprise ขนาดใหญ่เจอบ่อยที่สุด: มี Alert จาก ISE ตัวหนึ่ง, Alert จาก FTD ตัวหนึ่ง,
Alert จาก Endpoint (AMP/Secure Endpoint) อีกตัวหนึ่ง — ทั้ง 3 Alert **จริงๆแล้วคือเหตุการณ์เดียวกัน**
(Endpoint ตัวเดียวกันติด Malware) แต่ปรากฏเป็น 3 Incident แยกกันในระบบที่ต่างกัน ทำให้ SOC
(Security Operations Center) ต้องเสียเวลาสืบสวนต่อจุดเชื่อมเอง

**Cisco SecureX (แนวคิด XDR — Extended Detection and Response)** คือ Platform ที่ **Correlate**
Alert จากหลายแหล่ง (ISE, FTD, Umbrella, Secure Endpoint/AMP, Email Security) เข้าเป็น **Incident
เดียว** โดยอัตโนมัติ พร้อม Timeline และ "Casebook" ที่รวบรวม Indicator ทั้งหมดของเหตุการณ์นั้น
ไว้ในที่เดียว

```
ก่อนมี XDR (แยกกัน)                          หลังมี SecureX/XDR (รวมเป็น Incident เดียว)

ISE Alert: "Endpoint MAC=aaaa                 ┌─────────────────────────────────┐
Posture=Non-Compliant"                        │   INCIDENT #4821                    │
                                               │   Endpoint: 10.10.10.11 (PC1)       │
FTD Alert: "IP=10.10.10.11 ติดต่อ              │   Observables:                       │
C2 Domain evil-c2[.]net"                       │   - ISE: Posture Non-Compliant        │
                                               │   - FTD: Connection to evil-c2.net    │
AMP Alert: "Hash SHA256=xxxx                   │   - AMP: Malware Hash Detected         │
พบใน Endpoint 10.10.10.11"                     │   - Umbrella: DNS Query to evil-c2.net│
                                               │   Confidence: High → Auto-Response    │
(SOC ต้องมานั่งต่อจุดเอง)                        └─────────────────────────────────┘
```

### Rapid Threat Containment (RTC) — Worked Scenario เต็มรูปแบบ

**Rapid Threat Containment** คือกลไกที่ ISE **Quarantine อุปกรณ์โดยอัตโนมัติ** เมื่อได้รับสัญญาณ
ว่ามี Malware บนอุปกรณ์นั้น โดยไม่ต้องรอ Admin เข้าไปกดปิด Port หรือลบ VLAN ด้วยมือ — กลไกนี้ใช้
**ANC (Adaptive Network Control) ผ่าน pxGrid** ที่พูดถึงใน Step 775 ร่วมกับ **CoA (Change of
Authorization)** ที่ทวนจาก [Part 37 Step 367](part-037-wireless-security-advanced.md)

```
Worked Scenario: PC1 (10.10.10.11, SALES_SGT=100) ติด Malware

1) AMP/Secure Endpoint บน PC1 (หรือ FTD ที่ตรวจ Traffic ผ่าน) พบ File Hash ตรงกับ Malware ที่รู้จัก
   → ส่ง Alert ไปยัง SecureX/XDR

2) SecureX Correlate เห็นว่า Endpoint เดียวกันนี้ก็มี DNS Query ไปยัง C2 Domain ที่ Umbrella
   Block ไว้ (ผ่าน Umbrella Cloud API/SecureX Integration) → ยืนยัน Confidence สูงว่าติด Malware จริง

3) SecureX สั่ง Action ผ่าน pxGrid ไปยัง ISE:
   "ANC Quarantine — Apply to Endpoint MAC=0050.5601.aaaa"

4) ISE รับคำสั่ง Quarantine → ส่ง **RADIUS CoA** ไปยัง ACCESS-SW1 (Authenticator ของพอร์ตนั้น)
   → ACCESS-SW1 Re-authorize Session ของ PC1 ใหม่ทันที (ไม่ต้อง Disconnect สาย)
   → คราวนี้ ISE ตอบ Authorization ใหม่: SGT = QUARANTINE_SGT (เช่น SGT=999) แทน SALES_SGT เดิม

5) SGT ใหม่ (QUARANTINE_SGT) เดินทางผ่าน SXP/Inline Tagging เหมือนปกติ (Part 44 Step 437-439)
   → ที่ DIST-SW3 (Enforcement Point เดิมของ Part 44 Step 440):
     cts role-based permissions from 999 to any sgacl DENY-ALL-SGACL
   → PC1 ไม่สามารถคุยกับอุปกรณ์อื่นในเครือข่ายได้เลย (ตัด Lateral Movement ทันที)

6) ในเวลาเดียวกัน FTD (ที่รับ Context ใหม่ผ่าน pxGrid — Step 774) เห็นว่า SGT ของ 10.10.10.11
   เปลี่ยนเป็น QUARANTINE_SGT แล้ว → ACP Rule "QUARANTINE-BLOCK-ALL" (Match Source SGT=999)
   บล็อก Traffic ทุกทางของ PC1 ที่ผ่าน FW-1 ด้วย (สองชั้นซ้อนกัน)

7) Umbrella (ที่มี Threat Intelligence เดียวกับ Talos ของ FTD) บล็อก Domain C2 นั้นสำหรับ
   Endpoint/Network ทั้งหมดอยู่แล้วตั้งแต่ก่อนขั้นตอนที่ 1 — ตัด "ทางออก" ของ Malware แม้ Endpoint
   ตัวอื่นที่ยังไม่ถูก Quarantine จะพยายามติดต่อ C2 Domain เดียวกันก็ตาม

ผลลัพธ์สุดท้าย: PC1 ถูกตัดขาดจากทั้ง Lateral Movement (SGACL) และ Command-and-Control (Umbrella)
ภายในไม่กี่วินาที โดยไม่มี Admin คนไหนต้อง Login เข้าอุปกรณ์สักตัวเดียว
```

### ตารางสรุป: ใครทำหน้าที่อะไรใน RTC Chain นี้

| ระบบ | หน้าที่ใน Rapid Threat Containment |
|---|---|
| AMP/Secure Endpoint หรือ FTD | ตรวจจับสัญญาณแรก (Malware Hash/Suspicious Connection) |
| SecureX/XDR | Correlate สัญญาณจากหลายแหล่งเป็น Incident เดียว, สั่ง Action |
| ISE (ผ่าน pxGrid ANC) | รับคำสั่ง Quarantine, ส่ง CoA ไปยัง Switch, เปลี่ยน SGT |
| ACCESS-SW/DIST-SW (TrustSec) | Enforce SGACL ตาม SGT ใหม่ — ตัด Lateral Movement |
| FTD (ผ่าน pxGrid Context) | เห็น SGT ใหม่ทันที — บล็อก Traffic ที่ผ่าน Perimeter/Internal Firewall |
| Umbrella | บล็อก C2 Domain ที่ระดับ DNS — ตัดทางสื่อสารกับผู้โจมตีจากภายนอก |

---

## Step 777 — Zero Trust Preview: Stack นี้ Implement Principle อะไรไปแล้วบ้าง

### Zero Trust คืออะไรโดยสรุป (รายละเอียดเต็มรูปแบบรอที่ Part 94)

หลักการ **Zero Trust** สรุปสั้นที่สุดคือ **"Never Trust, Always Verify"** — ไม่มี Traffic ไหน
ได้รับความไว้ใจแค่เพราะ "อยู่ในเครือข่ายภายใน" (Trusted Zone แบบเดิม) อีกต่อไป ทุก Request ต้อง
ถูกยืนยันตัวตนและตรวจสอบสิทธิ์**ทุกครั้ง** ไม่ว่าจะมาจากไหน — Part นี้ไม่ได้สอน Zero Trust แบบ
เต็มรูปแบบ (รอ **Part 94**) แต่ Stack ที่เราสร้างมาตั้งแต่ Part 21-44 และประกอบร่างใน Part นี้
**Implement หลักการ Zero Trust ไปแล้วหลายข้อโดยไม่รู้ตัว**

### แมป Feature ที่มีอยู่แล้ว เข้ากับ Zero Trust Principle

| Zero Trust Principle | Feature ในหลักสูตรนี้ที่ Implement แล้ว | สร้างไว้ที่ |
|---|---|---|
| **Verify Explicitly** (ยืนยันตัวตนทุกครั้ง ไม่เชื่อแค่ตำแหน่งเครือข่าย) | 802.1X ที่บล็อก Traffic ทุกอย่างจนกว่าจะ Authenticate ผ่าน (Deny by Default ที่ Layer 2) | Part 44 Step 432 |
| **Least Privilege Access** (ให้สิทธิ์เท่าที่จำเป็นเท่านั้น) | SGACL ที่จำกัด SGT-to-SGT เฉพาะ Protocol/Port ที่ต้องใช้จริง ไม่ Allow กว้างๆ | Part 44 Step 438 |
| **Assume Breach** (สมมติว่ามี Endpoint ที่ถูก Compromise อยู่แล้วเสมอ) | Rapid Threat Containment ที่ Quarantine อัตโนมัติ ไม่รอ Admin ยืนยัน | Step 776 |
| **Micro-Segmentation** (แบ่งเครือข่ายเป็นส่วนเล็กที่สุดที่ทำได้ ไม่ใช่แค่ Perimeter เดียว) | TrustSec/SGT ที่ทำ Segmentation ระดับ "กลุ่ม" ไม่ใช่แค่ VLAN/Subnet | Part 44 Step 437 |
| **Continuous Verification** (ตรวจสอบต่อเนื่อง ไม่ใช่แค่ตอน Login ครั้งแรก) | ISE Posture Assessment ที่ตรวจสุขภาพ Endpoint ต่อเนื่องและ Re-authorize ผ่าน CoA เมื่อสถานะเปลี่ยน | Part 43 Step 428 |
| **Inspect Everywhere, Not Just Perimeter** (ตรวจสอบทุกที่ ไม่ใช่แค่ Edge) | Umbrella คุ้มครอง Roaming User ที่ไม่อยู่หลัง Perimeter Firewall เลย | Step 772-773 |

### สิ่งที่ Stack นี้ "ยังไม่ครบ" สำหรับ Zero Trust เต็มรูปแบบ — จุดที่ Part 94 จะเติม

```
┌─────────────────────────────────────────────────────────────────────┐
│  Zero Trust ที่มีแล้ว (Part 21-44, 78)          Zero Trust ที่ Part 94 จะเติม     │
├─────────────────────────────────────────────────────────────────────┤
│  ✅ Identity-based Network Policy (SGT)        ⬜ Zero Trust Network Access │
│  ✅ Device Posture ก่อนเข้า Network                 (ZTNA) แบบ Application-  │
│  ✅ Cloud-delivered Inspection (Umbrella)           Specific ไม่ใช่แค่ Network│
│  ✅ Automated Response (RTC)                    ⬜ Zero Trust สำหรับ Workload │
│                                                     ใน Data Center/Cloud     │
│                                                  ⬜ Continuous Risk Scoring   │
│                                                     แบบ Real-time ทุก Session │
│                                                  ⬜ Software-Defined Perimeter│
└─────────────────────────────────────────────────────────────────────┘
```

> **สรุปสำหรับ Step นี้**: อย่าเข้าใจผิดว่า "ต้องรอ Part 94 ถึงจะเริ่ม Zero Trust ได้" — ความจริง
> คือองค์กรที่ทำ ISE + TrustSec + Posture + Umbrella ครบตามที่หลักสูตรนี้สอนมาแล้ว **ได้เริ่ม
> เดินทาง Zero Trust ไปมากแล้ว** สิ่งที่ Part 94 จะทำคือ **ขยายหลักการเดียวกันนี้** ไปให้ครอบคลุม
> Application/Workload Level และ Cloud-native Environment ที่ Network-based Segmentation อย่าง
> SGT อย่างเดียวยังไปไม่ถึง (เช่น Microservice ใน Container ที่ไม่มี "Switch Port" ให้ทำ 802.1X)

---

## Step 778 — Unified PKI Infrastructure สำหรับทั้ง Security Stack

### ทำไม PKI ต้องเป็น "โครงสร้างพื้นฐาน" ไม่ใช่ "เรื่องทีหลัง"

ทวนจาก [Part 37 Step 365](part-037-wireless-security-advanced.md): EAP-TLS ต้องมี PKI ที่ทำงาน
ได้จริง — แต่ตลอด Part 37-44 หลักสูตรนี้พูดถึง Certificate แบบแยกส่วนตามแต่ละ Use Case (Client
Cert สำหรับ EAP-TLS, Server Cert สำหรับ ISE Admin) Part นี้คือจุดที่เราต้องมองภาพรวมว่า **CA
เดียวกันหนึ่งชุดต้องออก Certificate ให้กับใครบ้าง** ในทั้ง Security Stack — ถ้าออกแบบ PKI แยกกัน
คนละชุดสำหรับแต่ละ Use Case จะดูแลไม่ไหวและ Trust Chain จะสับสนมาก

### รายการ Certificate ทั้งหมดที่ Enterprise Security Stack ต้องมี

| Certificate ที่ต้องมี | ใช้ทำอะไร | อ้างอิง |
|---|---|---|
| **EAP-TLS Client Certificate** | ออกให้ทุก Endpoint (Laptop พนักงาน) สำหรับ Mutual TLS ตอน 802.1X | Part 37 Step 365, Part 44 |
| **RADIUS/EAP Server Certificate** | ติดตั้งบน ISE PSN — ให้ Client Verify ว่ากำลังคุยกับ ISE ตัวจริง ไม่ใช่ Rogue AP/Server | Part 43 Step 426 |
| **ISE Admin Certificate** | HTTPS สำหรับ Admin Login เข้า ISE GUI (PAN) | Part 43 |
| **pxGrid Certificate** | Mutual TLS ระหว่าง ISE pxGrid Node กับ Client (FMC, SIEM) | Step 775 |
| **FMC/FTD Management Certificate** | HTTPS สำหรับ FMC GUI และการสื่อสาร FTD↔FMC | Part 43 Step 422 |
| **TrustSec Device Identity Certificate** | ยืนยันตัวตนของ Switch/Router ที่เข้าร่วม CTS Domain (PAC หรือ Certificate-based ตาม Mode ที่เลือก) | Part 44 |

### ออกแบบ CA Hierarchy เดียวที่รองรับทุกความต้องการข้างบน

```
                          ┌─────────────────────┐
                          │      ROOT CA           │   <- Offline เก็บปลอดภัยสุด
                          │  CompanyLab-Root-CA     │      (เหมือนหลักการ Part 37 Step 365)
                          └───────────┬───────────┘
                                      │ เซ็นให้ Subordinate CA เท่านั้น (Root ไม่ออก Leaf Cert ตรง)
                    ┌─────────────────┼─────────────────┐
                    ▼                                     ▼
        ┌─────────────────────┐               ┌─────────────────────┐
        │  ISSUING CA #1          │               │  ISSUING CA #2          │
        │  "Endpoint-CA"          │               │  "Infrastructure-CA"    │
        │  (Online — ออก Cert      │               │  (Online — ออก Cert      │
        │   ให้ Endpoint จำนวนมาก) │               │   ให้อุปกรณ์ Network/    │
        │                          │               │   Security Platform)     │
        └───────────┬───────────┘               └───────────┬───────────┘
                    │                                         │
        ┌────────────┼────────────┐                ┌──────────┼──────────┬──────────┐
        ▼            ▼            ▼                 ▼          ▼          ▼          ▼
   Client Cert   Client Cert   Client Cert      ISE PSN     FMC/FTD   pxGrid      TrustSec
   (พนักงาน       (พนักงาน       (BYOD ผ่าน       Server     Mgmt      Node        Device
    Laptop 1)      Laptop 2)     SCEP/ISE         Cert       Cert      Cert         Cert
                                 BYOD Portal —
                                 Part 43 Step 429)
```

### เหตุผลที่แยก 2 Issuing CA (ไม่ใช้ Issuing CA เดียวรวมทุกอย่าง)

| เหตุผล | รายละเอียด |
|---|---|
| **Blast Radius เมื่อ CA ถูก Compromise** | ถ้า Endpoint-CA ถูกแฮ็ก (เช่น จาก BYOD Onboarding Flow ที่มีจุดเสี่ยงมากกว่า) Revoke ได้เฉพาะ Endpoint-CA โดยไม่กระทบ Certificate ของ Infrastructure Device (ISE/FMC/Switch) ที่ยังใช้งานต่อได้ |
| **Certificate Lifecycle ต่างกัน** | Client Cert ของพนักงานอาจ Renew ทุก 1 ปี (ตาม Turnover พนักงาน) ส่วน Infrastructure Cert (ISE/FMC) อาจ Renew ทุก 3-5 ปี — แยก CA ทำให้ตั้ง Policy อายุ Certificate ต่างกันได้ง่ายกว่า |
| **จำนวน Certificate ที่ต้องออก** | Endpoint-CA ต้องออก Certificate นับพัน-หมื่นใบ (ตามจำนวนพนักงาน) ส่วน Infrastructure-CA ออกไม่กี่สิบใบ (ตามจำนวนอุปกรณ์) — แยกภาระงานและ CRL/OCSP Load ออกจากกัน |

### Revocation Checking — สิ่งที่มักถูกมองข้ามแต่จำเป็นมาก

Certificate ที่ออกไปแล้วต้อง **เพิกถอนได้** เมื่อพนักงานลาออกหรือ Device สูญหาย — ISE รองรับทั้ง
2 วิธี:

| วิธี | หลักการ | ข้อดี/ข้อเสีย |
|---|---|---|
| **CRL (Certificate Revocation List)** | ISE Download รายการ Certificate ที่ถูก Revoke เป็นไฟล์ตามรอบเวลา | ง่าย แต่ **มี Delay** — ถ้า Revoke ไปแล้วแต่ CRL ยังไม่ Update รอบใหม่ Certificate นั้นยังใช้งานได้อยู่ |
| **OCSP (Online Certificate Status Protocol)** | ISE ถาม CA แบบ Real-time ทุกครั้งที่มี Client ยื่น Certificate | **Real-time** ไม่มี Delay แต่ต้องมี OCSP Responder พร้อมทำงานตลอดเวลา (ถ้า OCSP Responder ล่ม ต้องมี Fallback Policy — Fail Open หรือ Fail Closed) |

> **คำแนะนำสำหรับ Enterprise Design**: ใช้ **OCSP เป็นหลัก** สำหรับ Certificate ที่ Sensitive
> สูง (EAP-TLS Client Cert, pxGrid Cert) เพราะ Delay ของ CRL อาจหมายถึงพนักงานที่ถูกไล่ออกยังใช้
> Certificate เข้าเครือข่ายได้อยู่หลายชั่วโมง — CRL ใช้เป็น Fallback รองเมื่อ OCSP Responder
> เข้าไม่ถึงชั่วคราว

---

## Step 779 — Compliance และ Audit Considerations

### ทำไม Security Integration ต้องผูกกับ Compliance Framework เสมอ

Enterprise ส่วนใหญ่ไม่ได้ทำ Security Integration เพราะ "อยากทำ" เท่านั้น แต่เพราะต้อง**พิสูจน์
ต่อ Auditor** ว่ามีการควบคุมที่เพียงพอตาม Framework ที่ธุรกิจต้อง Compliance (PCI-DSS สำหรับ
ธุรกิจที่รับบัตรเครดิต, ISO 27001, HIPAA สำหรับ Healthcare) — ข่าวดีคือ Stack ที่เราสร้างมาตั้งแต่
Part 21-44 และ Integrate ใน Part นี้ **ตอบโจทย์ Compliance Requirement ทั่วไปได้เกือบทั้งหมด
โดยไม่ต้องซื้อ Tool เพิ่ม**

### แมป Feature เข้ากับ PCI-DSS Requirement (ตัวอย่างที่พบบ่อยที่สุด)

| PCI-DSS Requirement (สรุป) | Feature ในหลักสูตรนี้ที่ตอบโจทย์ |
|---|---|
| **Requirement 1**: แบ่ง Network Segmentation ระหว่าง Cardholder Data Environment (CDE) กับส่วนอื่น | TrustSec/SGT (Part 44) — SERVERS_SGT ที่แยก Server ที่เก็บข้อมูลบัตรออกจากกลุ่มอื่นด้วย SGACL, และ FTD ACP ที่ Match ด้วย SGT เดียวกัน (Step 774) ทำให้ Segmentation สอดคล้องกันทั้ง Network และ Firewall Layer |
| **Requirement 7**: จำกัดสิทธิ์การเข้าถึงตาม "Need to Know" (Least Privilege) | SGACL ที่ Permit เฉพาะ Port ที่จำเป็น (เช่น HTTPS/443 เท่านั้นระหว่าง SALES→SERVERS จาก Part 44 Step 438) ตรงกับหลักการ Least Privilege โดยตรง |
| **Requirement 8**: ยืนยันตัวตนผู้ใช้ทุกคนที่เข้าถึง System ที่เกี่ยวกับ Cardholder Data | 802.1X (Part 44) + ISE Policy Set (Part 43 Step 427) ที่บังคับ Authenticate ก่อนเข้าเครือข่ายเสมอ |
| **Requirement 10**: บันทึก Log และ Audit Trail ของการเข้าถึงทั้งหมด | **TACACS+ Accounting** ([Part 22](part-022-aaa-device-hardening.md)) สำหรับ Device Admin (บันทึกว่า Admin คนไหนพิมพ์คำสั่งอะไรบน Switch/Router) รวมกับ **ISE MnT** (Part 43 Step 426) ที่บันทึก RADIUS Live Log ทุก Authentication Event — สองระบบรวมกันให้ Audit Trail ครบทั้ง "ใครดูแลอุปกรณ์" และ "ใครเข้าเครือข่าย" |
| **Requirement 11**: ทดสอบ Security ของระบบอย่างสม่ำเสมอ (Vulnerability Scanning, IPS) | FTD Intrusion Policy (Snort Engine, Part 43 Step 424) ทำหน้าที่ IPS ต่อเนื่อง, `show cts role-based counters` (Part 44 Step 440) ให้ข้อมูลยืนยันว่า Policy ทำงานจริง |

### รวม Log จาก 2 แหล่งให้เป็น Audit Trail เดียว — Pattern ที่ Auditor มักขอเห็น

```
คำถามที่ Auditor PCI-DSS มักถามจริง: "แสดงหลักฐานว่าใครเข้าถึง Server1 (Cardholder Data) เมื่อไหร่"

ตอบด้วยการรวม Log 3 ชั้น:

1) ISE MnT — RADIUS Live Log ยืนยันว่า "employee01" Authenticate ผ่าน 802.1X ที่ ACCESS-SW1
   Gi1/0/1 เวลา 10:42:15 ได้ SGT=SALES_SGT

2) DIST-SW3 — `show cts role-based counters` (Part 44 Step 440) ยืนยันว่า SALES_SGT→SERVERS_SGT
   มี HW-Permitted บน Port 443 ในช่วงเวลาเดียวกัน (คือ employee01 เข้าถึง Server1 ผ่าน HTTPS จริง)

3) FMC — Connection Events (Step 774) ยืนยัน Source SGT/User ที่ผ่าน FW-1 พร้อม Timestamp
   ตรงกันกับสองแหล่งข้อมูลด้านบน

3 แหล่งข้อมูลนี้ "ยืนยันกันเอง" (Cross-validation) — Auditor เชื่อได้มากกว่า Log จากแหล่งเดียว
เพราะพิสูจน์ว่า Identity → Network Enforcement → Firewall Enforcement สอดคล้องกันตลอดสาย
```

> **ข้อคิดสำหรับงานจริง**: การออกแบบให้ Log จากหลายระบบ "เล่าเรื่องเดียวกัน" ได้แบบนี้คือผลพลอยได้
> โดยตรงจากการทำ Integration ใน Step 771-776 — ถ้าแต่ละระบบทำงานแยกกันเป็น Silo (Point Solution)
> Log ของแต่ละระบบจะไม่สามารถ Cross-validate กันได้เลย และ Auditor จะต้องเชื่อ Log แต่ละแหล่ง
> แบบแยกส่วน ซึ่งอ่อนกว่ามาก

---

## Step 780 — Lab: เอกสารสถาปัตยกรรม Security แบบบูรณาการเต็มรูปแบบ

> **หมายเหตุสำคัญ**: เหมือนกับ [Part 43 Step 430](part-043-security-architecture-firepower-ise.md)
> Lab นี้เป็น **เอกสารสถาปัตยกรรมเชิงแนวคิด (Architectural/Design-level)** ที่รวมทุก Decision
> จาก Step 771-779 เข้าเป็นภาพเดียว — ระบบส่วนใหญ่ (ISE/FMC/Umbrella) บริหารผ่าน Web GUI/Cloud
> Portal เป็นหลัก คำสั่ง Verification ที่ให้ไว้คือส่วนที่ยืนยันได้จริงบน Network Device (IOS-XE/
> FTD CLI)

### แผนผัง Deployment เต็มรูปแบบของทั้ง Security Stack

```
                          ┌─────────────────────────────┐
                          │       CISCO UMBRELLA (Cloud)     │
                          │  DNS Security + SWG + CASB         │
                          └──────┬──────────────┬───────────┘
                    Umbrella-VA          Network Tunnel (IPsec)
              (10.10.99.72/73)         จาก WAN-EDGE (Cloud onRamp
                     │                   for SaaS — Part 76)
                     ▼                              ▲
┌───────────────────────────────────────────────────┼───────────────────────┐
│                          INTERNET / WAN                                       │
└──────────────────────────────┬──────────────────────────────────────────────┘
                                 ┌──────┴──────┐
                                 │    FW-1      │  FTDv (Transparent, Part 43)
                                 │  FMC(.71)    │  ACP Match SGT (Step 774)
                                 └──────┬──────┘  pxGrid Subscriber (Step 775)
        ┌────────────────────────────┴────────────────────────────┐
        │                    CORE-SW1 <===LACP===> CORE-SW2           │
        └────┬───────────────────────────────────────────────┬────┘
   ┌──────────┴──────────┐                          ┌──────────┴──────────┐
   │  DIST-SW1 / DIST-SW2 │                          │  DIST-SW3 ⭐ / DIST-SW4│
   │  (SXP Listener)      │                          │  (SGACL Enforcement,   │
   └────┬────────────┬───┘                          │   Part 44 Step 438)     │
┌────────┴───┐  ┌─────┴──────┐                       └────┬────────────┬───┘
│ ACCESS-SW1 │  │ ACCESS-SW2  │  802.1X (Part 44)   ┌────────┴───┐  ┌─────┴──────┐
│ SXP Speaker│  │ SXP Speaker  │                       │ ACCESS-SW3 │  │ ACCESS-SW4  │
└─────┬──────┘  └──────┬─────┘                       └─────┬──────┘  └──────┬─────┘
      │                │                                     │                │
  PC1-5(SALES_SGT)  Phone(VOICE_SGT)                    Server1-3        AP/Printer
                                                       (SERVERS_SGT      (PRINTER_SGT,
                                                        static)           MAB)
                                          │
                          ┌──────────────┴──────────────────────┐
                          │   ISE-1 (PAN+MnT+pxGrid, .65)           │
                          │   ISE-2 (PSN, .66)                       │
                          │   Policy Set + TrustSec Server + ANC     │
                          └──────────────┬──────────────────────┘
                                          │ pxGrid (Step 775)
                          ┌──────────────┴──────────────┐
                          ▼                              ▼
                  FMC (Subscriber)              SecureX/XDR (Step 776)
                  ACP Match SGT                 Correlate ISE+FTD+AMP+Umbrella
```

### ตาราง Component สรุปทั้งหมดที่ Part นี้เพิ่มเข้ามา

| Component | IP/ตำแหน่ง | บทบาทใหม่ที่เพิ่มจาก Part นี้ |
|---|---|---|
| ISE-1 | `10.10.99.65/24` (เดิมจาก Part 43) | เพิ่ม **pxGrid Persona** ให้ FMC/SecureX Subscribe (Step 775) |
| FMC | `10.10.99.71/24` (เดิมจาก Part 43) | Register เป็น pxGrid Client, เพิ่ม ACP Rule Match SGT (Step 774) |
| Umbrella-VA-1/2 | `10.10.99.72/73` (ใหม่) | DNS Forwarder สำหรับ Umbrella for Network (Step 773) |
| DIST-SW3 | Enforcement Point เดิม (Part 44) | เพิ่ม SGACL สำหรับ QUARANTINE_SGT (Step 776) |
| WAN-EDGE-1/2 | เดิมจาก Part 19/39 | เพิ่ม Network Tunnel (IPsec) ไปยัง Umbrella Cloud สำหรับ Cloud onRamp for SaaS (Step 773) |
| SecureX/XDR | Cloud Portal | Correlate Alert จาก ISE/FTD/AMP/Umbrella, สั่ง ANC ผ่าน pxGrid (Step 776) |

### Worked Rapid Threat Containment Scenario — ทวนแบบสรุปเป็น Checklist

สถานการณ์: **Malware ถูกพบบน Endpoint ที่ SALES VLAN (VLAN 10, SGT=100)**

| # | เหตุการณ์ | ระบบที่ทำงาน | ผลลัพธ์ |
|---|---|---|---|
| 1 | Endpoint ติดต่อ C2 Domain ผ่าน DNS | Umbrella | Query ถูก Log ไว้เป็น Threat Indicator |
| 2 | AMP/FTD ตรวจพบ File Hash อันตรายบน Endpoint เดียวกัน | AMP + FTD | Alert ส่งไปยัง SecureX |
| 3 | Correlate Alert จาก DNS Query + File Hash เป็น Incident เดียว | SecureX/XDR | ยืนยัน Confidence สูง สั่ง Auto-Response |
| 4 | สั่ง ANC Quarantine ผ่าน pxGrid | SecureX → ISE | ISE เตรียม Authorization ใหม่ (QUARANTINE_SGT) |
| 5 | ส่ง CoA ไปยัง ACCESS-SW ที่ Endpoint เสียบอยู่ | ISE → ACCESS-SW | Session ถูก Re-authorize ทันที ไม่ต้อง Disconnect |
| 6 | SGT ใหม่เดินทางผ่าน SXP/Inline Tagging ไปถึง DIST-SW3 | TrustSec (Part 44) | SGACL บล็อก Lateral Movement ทุกทาง |
| 7 | FTD เห็น SGT ใหม่ผ่าน pxGrid | FMC/FTD | ACP Rule QUARANTINE-BLOCK-ALL บล็อก Traffic ผ่าน Perimeter |
| 8 | Umbrella บล็อก C2 Domain ต่อไปสำหรับ Endpoint อื่นทั้งองค์กร | Umbrella | ตัดโอกาสระบาดผ่าน Domain เดียวกัน |

### PKI Design ที่รองรับทั้ง Stack — สรุปจาก Step 778

| CA Layer | ออก Certificate ให้ | จำนวนโดยประมาณ | Renewal Cycle |
|---|---|---|---|
| Root CA (Offline) | Issuing CA ทั้ง 2 ตัวเท่านั้น | 2 | 10+ ปี |
| Endpoint-CA | Client Cert พนักงาน/BYOD (EAP-TLS) | ตามจำนวนพนักงาน/Device | 1 ปี |
| Infrastructure-CA | ISE PSN, FMC/FTD, pxGrid Node, TrustSec Device | ตามจำนวนอุปกรณ์ | 3 ปี |

### Checklist Deploy สำหรับ Lab ของหลักสูตรนี้

| # | งาน | อ้างอิง Step |
|---|---|---|
| 1 | เปิด pxGrid Persona บน ISE-1, Register FMC เป็น pxGrid Client + Approve | Step 775 |
| 2 | เพิ่ม ACP Rule บน FMC ให้ Match ด้วย Source SGT (SALES/CONTRACTOR/QUARANTINE) | Step 774 |
| 3 | Deploy Umbrella-VA-1/2 ที่ `10.10.99.72/73`, ตั้งเป็น Conditional Forwarder จาก DNS Server เดิม (Part 16) | Step 773 |
| 4 | ตั้ง Network Tunnel (IPsec) จาก WAN-EDGE ไปยัง Umbrella Cloud สำหรับ Cloud onRamp for SaaS Traffic | Step 773 |
| 5 | สร้าง QUARANTINE_SGT ใหม่บน ISE + SGACL `DENY-ALL-SGACL` ที่ DIST-SW3 สำหรับ ANC | Step 776 |
| 6 | เชื่อม SecureX/XDR กับ ISE (pxGrid ANC), FTD, และ Umbrella สำหรับ Correlation | Step 776 |
| 7 | ออกแบบ CA Hierarchy: Root CA + Endpoint-CA + Infrastructure-CA ตาม Step 778 | Step 778 |
| 8 | ตรวจสอบ Log Cross-validation ระหว่าง ISE MnT / `show cts role-based counters` / FMC Connection Events | Step 779 |

---

## แบบฝึกหัดทวนความเข้าใจ Part 78

1. Cisco Umbrella แก้ปัญหา "จุดบอด" อะไรของ FTD ที่ FTD ตัวเดียวทำไม่ได้ และเพราะเหตุใด?
2. อธิบายว่า pxGrid ทำให้ FTD "รู้จัก" SGT ของ Traffic ที่ผ่านมันได้อย่างไร — ทำไม FTD จึงไม่
   สามารถคำนวณ SGT เองได้โดยไม่มี pxGrid?
3. ในกรณี Rapid Threat Containment (Step 776) เมื่อ ISE ได้รับคำสั่ง Quarantine ผ่าน ANC มันใช้
   Mechanism อะไรส่งคำสั่งไปยัง Switch เพื่อเปลี่ยน SGT ของ Endpoint โดยไม่ต้อง Disconnect สาย?
4. เพราะเหตุใด PKI Design ของทั้ง Security Stack จึงควรแยก Issuing CA เป็น 2 ตัว (Endpoint-CA
   กับ Infrastructure-CA) แทนที่จะใช้ Issuing CA เดียวรวมทุกอย่าง?
5. ยกตัวอย่าง PCI-DSS Requirement 1 ข้อหนึ่ง และอธิบายว่า Feature ใดในหลักสูตรนี้ตอบโจทย์
   Requirement นั้นได้อย่างไร?

**เฉลย:**

1. Umbrella แก้ปัญหา **Roaming User ที่ไม่ผ่าน FW-1 เลย** (พนักงานทำงานนอกออฟฟิศ) เพราะ FTD
   Inspect ได้เฉพาะ Traffic ที่ผ่านตัวมันจริงที่ Perimeter เท่านั้น — Umbrella ทำงานที่ DNS Layer
   ซึ่งเกิดก่อนการเชื่อมต่อจริงและทำงานได้ไม่ว่า Client จะอยู่ที่ไหนในโลก จึงคุ้มครอง Roaming
   User และ Direct Internet Access Traffic ที่ไม่ผ่าน On-prem Firewall ได้
2. FTD ไม่มีทางรู้ SGT ของ Traffic ได้เองเพราะ SGT เป็นข้อมูลที่ ISE เป็นผู้ตัดสินใจและแจกให้
   ผ่าน 802.1X/MAB เท่านั้น ไม่ได้อยู่ใน Packet ที่ FTD เห็นตรงๆ (นอกจากผ่าน Trunk ที่ทำ Inline
   Tagging) — pxGrid คือ Protocol ที่ ISE ใช้ Publish ข้อมูล Session Directory (User/IP/SGT)
   ให้ FMC Subscribe แบบ Real-time ทำให้ FTD "รู้" ว่า IP ใดมี SGT อะไรโดยไม่ต้องคำนวณเอง
3. ISE ใช้ **RADIUS CoA (Change of Authorization)** ส่งไปยัง Authenticator (ACCESS-SW) เพื่อ
   สั่ง Re-authorize Session ของ Endpoint นั้นใหม่ทันที โดยไม่ต้อง Disconnect สายหรือรอ Endpoint
   Reboot — ครั้งนี้ ISE ตอบ Authorization Result ใหม่เป็น QUARANTINE_SGT แทน SGT เดิม
4. เพราะ (1) จำกัด Blast Radius — ถ้า Endpoint-CA ถูก Compromise สามารถ Revoke ได้โดยไม่กระทบ
   Certificate ของ Infrastructure Device ที่ยังต้องทำงานต่อเนื่อง (2) Certificate Lifecycle
   ต่างกัน — Client Cert อาจ Renew บ่อยกว่าตาม Turnover พนักงาน ส่วน Infrastructure Cert Renew
   ห่างกว่ามาก และ (3) แยกภาระงาน CRL/OCSP ออกจากกันเพราะจำนวน Certificate ที่ Endpoint-CA
   ต้องออกมีมากกว่า Infrastructure-CA หลายเท่า
5. **PCI-DSS Requirement 1** กำหนดให้แบ่ง Network Segmentation ระหว่าง Cardholder Data
   Environment (CDE) กับส่วนอื่นของเครือข่าย — **TrustSec/SGT** (Part 44) ตอบโจทย์นี้ได้โดยแยก
   SERVERS_SGT ออกจากกลุ่มอื่นด้วย SGACL ที่ Permit เฉพาะ Port ที่จำเป็น และเมื่อรวมกับ FTD ACP
   ที่ Match ด้วย SGT เดียวกัน (Step 774) ทำให้ Segmentation สอดคล้องกันทั้งที่ Network Layer
   และ Firewall Layer ซึ่งเป็นหลักฐานที่ Auditor ยอมรับได้ว่า CDE ถูกแยกออกจริง

---

## สรุป Part 78

Part นี้ไม่ได้สร้างเทคโนโลยีใหม่จากศูนย์เหมือน Part ก่อนๆ ส่วนใหญ่ — สิ่งที่ทำคือ **ประกอบร่าง**
ทุกชิ้นส่วน Security ที่หลักสูตรนี้สร้างไว้แยกกันตั้งแต่ [Part 21](part-021-switch-security.md)
ถึง [Part 44](part-044-dot1x-trustsec.md) ให้กลายเป็น **สถาปัตยกรรมเดียวที่พึ่งพากันจริง** — เริ่ม
จากภาพรวม Integration Map (Step 771), เปิดตัว **Cisco Umbrella** เป็นชั้นป้องกันบน Cloud ที่เสริม
(ไม่แทนที่) FTD ทั้งในรูปแบบ Roaming Client และ Umbrella for Network ที่เชื่อมกับ SD-WAN Cloud
onRamp for SaaS ของ [Part 76](part-076-sd-wan-advanced-cloud-onramp.md) (Step 772-773), พิสูจน์
ด้วย Worked Example ว่า **ISE + TrustSec + FTD** Integrate กันจริงผ่าน **pxGrid** โดยให้ FTD
เขียน Firewall Policy ด้วย SGT เดียวกันที่ ISE แจกผ่าน 802.1X (Step 774-775), สร้าง **Rapid
Threat Containment** ที่ Quarantine Endpoint อัตโนมัติผ่าน SecureX/XDR + ANC + CoA (Step 776),
มองย้อนว่า Stack นี้ Implement Zero Trust Principle ไปมากแล้วก่อนเข้า Part 94 (Step 777),
ออกแบบ **Unified PKI** ที่รองรับ Certificate ทุกประเภทในสถาปัตยกรรมนี้ (Step 778), และปิดท้าย
ด้วยการแมป Feature ทั้งหมดเข้ากับ **PCI-DSS/Compliance Requirement** จริง (Step 779) ก่อนสรุป
เป็นเอกสารสถาปัตยกรรมเต็มรูปแบบพร้อม Worked RTC Scenario (Step 780)

หลักการที่สำคัญที่สุดของ Part นี้คือ: **Security ที่แข็งแรงไม่ได้มาจากการมี "Product" หลายตัว
แต่มาจากการทำให้ Product เหล่านั้น "คุยกัน" และ "ตอบสนองร่วมกัน" ได้แบบ Real-time** — Enterprise
ที่ทำ Integration แบบนี้สำเร็จจะตอบสนองภัยคุกคามได้เร็วกว่าองค์กรที่มี Point Solution เดียวกัน
ทุกตัวแต่แยกกันทำงานเป็น Silo หลายเท่าตัว

เราใช้เวลาส่วนใหญ่ของหลักสูตรนี้ (Part 1-78) สร้าง **Infrastructure และ Security** ของ Enterprise
Network ให้แข็งแรงในทุกมิติ — แต่ทุกอย่างที่สร้างมาจนถึงตอนนี้ยังต้อง Config ผ่าน CLI/GUI ทีละ
ระบบอยู่ ยังไม่มี **Automation Pipeline** ที่ทำให้การ Deploy/Test/Rollback Config เปลี่ยนแปลง
ทั้งหมดนี้เป็นกระบวนการที่ทำซ้ำได้และปลอดภัยในระดับ Production จริง **Part 79 — Full-Stack
Automation & CI/CD** จะนำ Python Automation ที่เรียนมาแล้วใน Part 23/47/48 มาต่อยอดเป็น
CI/CD Pipeline เต็มรูปแบบสำหรับ Network Infrastructure ทั้งหมดที่เราสร้างมาตลอด 78 Part นี้

**ไปต่อ:** [Part 79 — Full-Stack Automation & CI/CD →](part-079-fullstack-automation-cicd.md)
