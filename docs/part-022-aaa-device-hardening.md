# Part 22 — AAA & Device Hardening
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 211–220 จาก 1000**

> ต่อจาก [Part 21 — Switch Security (Port Security, DHCP Snooping, DAI, IP Source Guard, STP
> Guard) →](part-021-switch-security.md) ซึ่งเราปิดช่องโหว่ที่ Layer 2 ของ Access Layer ไปแล้ว
> Part นี้เราจะขยับขึ้นมาที่ **ความปลอดภัยของตัวอุปกรณ์เอง (Device Security)** — โดยเฉพาะ
> **การบริหารสิทธิ์การเข้าถึงอุปกรณ์แบบรวมศูนย์ด้วย AAA (Authentication, Authorization,
> Accounting)** ซึ่งเป็นก้าวสำคัญจาก "Local User Database" ที่เราตั้งไว้ตั้งแต่ [Part 2]
> (part-002-ios-cli-basics.md) ไปสู่มาตรฐานการบริหาร Network Device ในระดับ Enterprise จริง
> พร้อมทั้ง **Device Hardening Checklist** ฉบับสมบูรณ์ที่ใช้ปิดท้าย

## สารบัญ Step ในภาคนี้

| Step | หัวข้อ |
|---|---|
| 211 | AAA คืออะไร — Authentication, Authorization, Accounting และเหตุผลที่ต้อง Centralized AAA |
| 212 | TACACS+ vs RADIUS — เปรียบเทียบ Protocol และเลือกใช้ให้ถูกงาน |
| 213 | เปิดใช้งาน AAA ด้วย `aaa new-model` และกฎเหล็กเรื่อง Fallback/Safety Net |
| 214 | AAA Authentication Configuration — TACACS+ Primary + Local Fallback |
| 215 | AAA Authorization — Exec Authorization และ Command Authorization |
| 216 | AAA Accounting — บันทึกทุกการกระทำเพื่อ Audit/Compliance |
| 217 | Privilege Level เชิงลึก และ Role-Based CLI Access (`parser view`) |
| 218 | Device Hardening Checklist ฉบับสมบูรณ์ |
| 219 | Control Plane Security ภาพรวม (Preview) และการจำกัด Management Plane |
| 220 | Lab: Deploy AAA เต็มรูปแบบทั้ง Topology + Hardening ทุกอุปกรณ์ |

---

## Step 211 — AAA คืออะไร: Authentication, Authorization, Accounting

ใน [Part 2](part-002-ios-cli-basics.md) เราตั้ง Local User Database ด้วย `username admin
privilege 15 secret ...` และใช้ `login local` บน VTY — วิธีนี้ใช้ได้ดีสำหรับ Lab หรือองค์กร
ขนาดเล็กมาก แต่มีข้อจำกัดร้ายแรงเมื่อ Network โตขึ้นถึงระดับ Enterprise (Topology ของเราตอนนี้มี
CORE-SW1/2, DIST-SW1-4, ACCESS-SW1-4, WAN-EDGE-1/2 — และจะเพิ่มอีกหลายสิบ-หลายร้อยอุปกรณ์ในองค์กร
จริง)

### ปัญหาของ Local-Only Authentication ที่ Enterprise Scale

| ปัญหา | รายละเอียด |
|---|---|
| **ไม่ Scale** | ต้อง config `username` ซ้ำทุกอุปกรณ์ทุกตัว — 50 อุปกรณ์ = ต้อง login 50 ครั้งเพื่อเพิ่ม user 1 คน |
| **การลบสิทธิ์ทำไม่ทัน** | พนักงานลาออก → ต้องเข้าไปลบ `username` ในทุกอุปกรณ์ ถ้าลืมแม้แต่ตัวเดียว = risk ค้างอยู่ |
| **ไม่มี Audit Trail ระดับคำสั่ง** | Local auth รู้แค่ "ใคร login" แต่ไม่รู้ว่า login แล้วไปพิมพ์คำสั่งอะไรบ้าง |
| **Password Rotation ยาก** | เปลี่ยน password ต้องไล่เปลี่ยนทุกอุปกรณ์ หรือใช้ script (เสี่ยง sync ไม่ตรง) |
| **ไม่มี Централized Policy** | ไม่สามารถบังคับ password complexity, MFA, time-of-day restriction จากจุดเดียวได้ |

**AAA (Authentication, Authorization, Accounting)** คือ Framework ของ Cisco (และอุตสาหกรรม
เครือข่ายทั้งหมด) สำหรับควบคุมว่า **ใครเข้าระบบได้ (Authentication)**, **เข้าแล้วทำอะไรได้บ้าง
(Authorization)**, และ **ทำอะไรไปแล้วบ้าง (Accounting)** — โดยแยกทั้ง 3 ฟังก์ชันนี้ออกจากกันอย่าง
ชัดเจน แม้ในทางปฏิบัติมักถูกกำหนดพร้อมกันในคำสั่งเดียวก็ตาม

### 3 ฟังก์ชันของ AAA อธิบายแยกทีละตัว

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AAA — 3 ฟังก์ชันที่แยกกันเด็ดขาด               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1) AUTHENTICATION  "คุณเป็นใคร และคุณพิสูจน์ได้ไหม"                    │
│     ตรวจสอบ Identity ด้วย Username/Password (หรือ Certificate,        │
│     Token, MFA) — ตอบคำถาม "ใครกำลังจะเข้าระบบ"                        │
│                                                                       │
│  2) AUTHORIZATION   "คุณทำอะไรได้บ้างหลังจากพิสูจน์ตัวตนแล้ว"           │
│     กำหนดสิทธิ์ เช่น เข้าได้ที่ Privilege Level เท่าไหร่,               │
│     รันคำสั่งไหนได้/ไม่ได้ — ตอบคำถาม "ทำอะไรได้บ้าง"                    │
│                                                                       │
│  3) ACCOUNTING      "คุณทำอะไรไปแล้วบ้าง เมื่อไหร่ นานแค่ไหน"           │
│     บันทึก Log ทุก Session/ทุกคำสั่งที่รันไปยัง Server กลาง             │
│     — ตอบคำถาม "เกิดอะไรขึ้นไปแล้ว" (สำหรับ Audit/Forensics)           │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

**จุดสำคัญที่สอบและงานจริงชอบถาม**: ทั้ง 3 ฟังก์ชันนี้**อิสระจากกัน**ในทางทฤษฎี — อุปกรณ์สามารถ
ทำ Authentication อย่างเดียวโดยไม่มี Authorization ก็ได้ (ทุกคน login แล้วได้ full access เท่ากัน)
หรือทำ Authentication + Authorization แต่ไม่ Accounting ก็ได้ แต่ **Enterprise ที่ดีควรทำครบทั้ง 3
เสมอ** เพื่อ Security-in-Depth (แนวคิดเดียวกับที่เราใช้ตลอดหลักสูตรนี้ตั้งแต่ Part 21)

### เหตุผลที่ต้อง Centralized AAA — สถาปัตยกรรมใหม่ที่จะสร้างใน Part นี้

```
                         ┌─────────────────────────┐
                         │   AAA-SRV (RADIUS/       │
                         │   TACACS+ Server)         │   VLAN 99 (MGMT)
                         │   10.10.99.60             │   10.10.99.0/24
                         │   (Cisco ISE / FreeRADIUS/│
                         │    tac_plus — generic)    │
                         └────────────┬────────────┘
                                      │  TACACS+ (TCP/49) หรือ RADIUS (UDP/1812-1813)
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
        ┌─────┴─────┐          ┌──────┴──────┐          ┌─────┴──────┐
        │ CORE-SW1/2 │          │ DIST-SW1-4  │          │ WAN-EDGE-*  │
        └───────────┘          └─────────────┘          └────────────┘

   Admin login → ยิงไปถาม AAA-SRV ทุกครั้ง (แทนที่จะเช็ค local username ในอุปกรณ์)
   เพิ่ม/ลบ user 1 ครั้งที่ AAA-SRV = มีผลกับทุกอุปกรณ์ในองค์กรทันที
```

จาก Step นี้เป็นต้นไป เราจะแนะนำอุปกรณ์ใหม่เข้าสู่ Lab Topology: **AAA-SRV** ที่ IP `10.10.99.60`
ใน VLAN 99 (MGMT) ตาม [IP Address Plan](00-ip-address-plan.md) — เป็น Server ที่รัน AAA Protocol
(เช่น **Cisco ISE (Identity Services Engine)** สำหรับ Enterprise เต็มรูปแบบ, หรือ **FreeRADIUS**
/ **tac_plus (TACACS+.net)** สำหรับ Lab/Open-source) หลักสูตรนี้จะอ้างอิงแบบ **Generic** เพราะ
คำสั่งฝั่ง Cisco IOS ที่อุปกรณ์ Network ใช้คุยกับ Server เหล่านี้เหมือนกันไม่ว่า Server จะเป็น
Platform ใด

| รายการ | ค่า |
|---|---|
| Hostname | AAA-SRV |
| IP Address | 10.10.99.60/24 (VLAN 99 — MGMT) |
| Default Gateway | 10.10.99.1 (HSRP VIP ของ VLAN 99) |
| บทบาท | TACACS+ Server (หลัก) / RADIUS Server (สำหรับ 802.1X, VPN ในภาคหลัง) |
| ตัวอย่าง Platform | Cisco ISE, FreeRADIUS, tac_plus (ระบุ generic — เลือก platform จริงตามงบ/นโยบายองค์กร) |

---

## Step 212 — TACACS+ vs RADIUS: เปรียบเทียบ Protocol และเลือกใช้ให้ถูกงาน

ก่อน config ต้องเข้าใจก่อนว่า AAA ไม่ใช่ Protocol เดียว — มันเป็น **Framework** ที่ทำงานผ่าน
Protocol การสื่อสารกับ Server กลางได้ 2 แบบหลัก คือ **TACACS+** และ **RADIUS**

### ตารางเปรียบเทียบแบบละเอียด

| คุณสมบัติ | TACACS+ (Terminal Access Controller Access-Control System Plus) | RADIUS (Remote Authentication Dial-In User Service) |
|---|---|---|
| Transport Layer | **TCP** port 49 (Connection-oriented, reliable) | **UDP** port 1812 (Auth) / 1813 (Accounting) — บาง legacy ใช้ 1645/1646 |
| การเข้ารหัส Packet | **เข้ารหัสทั้ง Packet Body** (Full encryption) | เข้ารหัสเฉพาะ **Password field** เท่านั้น ส่วน header อื่นเป็น plaintext |
| AuthN/AuthZ | **แยกกันชัดเจน** (Authentication และ Authorization เป็นขั้นตอนคนละอันได้) | **รวมกัน** ใน Access-Accept เดียว (แยกไม่ได้ในทางปฏิบัติ) |
| Command Authorization | **ทำได้** (`aaa authorization commands`) — จุดเด่นสำคัญที่สุด | **ทำไม่ได้** โดยธรรมชาติของ Protocol |
| Vendor | **Cisco Proprietary** (แต่ก็มี Open-source อย่าง tac_plus/shrubbery) | **Open Standard** (RFC 2865/2866) — รองรับหลายผู้ผลิต |
| Use Case หลัก | **Device Administration** (SSH/console login เข้า Router/Switch) | **Network Access Control** (802.1X, VPN Remote Access, Wi-Fi Authentication) |
| Multi-protocol Support | รองรับ AppleTalk, NetBIOS, Novell ASAP, X.25 (ในเวอร์ชันเก่า) — ปัจจุบันโฟกัส IP | โฟกัส IP-based access เท่านั้น |
| Accounting | ละเอียดกว่า (แยก start/stop/watchdog ต่อคำสั่ง) | มี แต่หยาบกว่า (per-session) |

### ทำไมต้องรู้ความต่างนี้ให้แม่น (ข้อสอบ CCNA/CCNP ชอบถามเปรียบเทียบนี้)

```
TACACS+ Packet:                         RADIUS Packet:
┌───────────────────────┐               ┌───────────────────────┐
│ Header (เข้ารหัส)       │               │ Header (plaintext)      │
│ ┌───────────────────┐ │               │ - Code, ID, Length      │
│ │ Body ทั้งหมด         │ │               │ - Attributes (plaintext)│
│ │ (เข้ารหัสด้วย shared │ │               │ ┌─────────────────┐   │
│ │  secret key)        │ │               │ │ Password field    │   │
│ └───────────────────┘ │               │ │ (เข้ารหัสเท่านั้น)   │   │
└───────────────────────┘               │ └─────────────────┘   │
                                          └───────────────────────┘
   → ปลอดภัยกว่าสำหรับ                        → attribute อื่น เช่น username,
     Device Admin (เห็น command                 authorization data ยังโดน sniff
     ที่พิมพ์ด้วยถ้าไม่เข้ารหัส)                  เห็นได้ถ้าไม่มี IPsec/TLS คลุมอีกชั้น
```

### กฎการเลือกใช้ (Best Practice มาตรฐานอุตสาหกรรม)

| สถานการณ์ | Protocol ที่แนะนำ | เหตุผล |
