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
|---|---|---|
| Login เข้า Router/Switch/Firewall (Device Administration) | **TACACS+** | ต้องการ Command Authorization + Full encryption ของสิ่งที่ Admin พิมพ์ |
| 802.1X Port-based Access Control (Part 21 preview, เต็มรูปแบบ Part 88) | **RADIUS** | เป็น Standard ที่ Supplicant (PC/Phone) รองรับ ไม่ใช่ TACACS+ |
| VPN Remote Access (AnyConnect) | **RADIUS** | ASA/FTD และ VPN Client มาตรฐานคุยกับ RADIUS |
| Wireless (WLC 802.1X/WPA2-Enterprise) | **RADIUS** | มาตรฐานอุตสาหกรรม Wireless ทั้งหมดใช้ RADIUS |
| ต้องการ Audit ว่า Admin คนไหนพิมพ์คำสั่งอะไรบน Switch | **TACACS+** | RADIUS ไม่มี Command Accounting ระดับนี้ |

> **สรุปสั้นที่จำง่าย**: **TACACS+ = ใครดูแลอุปกรณ์ (Device Admin)**,
> **RADIUS = ใครเข้าเครือข่าย (Network Access)** — Part นี้เราจะ config TACACS+ เป็นหลักเพราะ
> โฟกัสคือการบริหารสิทธิ์ Admin เข้า CORE/DIST Switch ส่วน RADIUS จะกลับมาใช้งานจริงตอน 802.1X
> เต็มรูปแบบและ WLC ในภาคหลังของหลักสูตร (Part 88, Part 89)

---

## Step 213 — เปิดใช้งาน AAA ด้วย `aaa new-model` และกฎเหล็กเรื่อง Fallback/Safety Net

### `aaa new-model` คือคำสั่งที่อันตรายที่สุดคำสั่งหนึ่งในหลักสูตรนี้ (ถ้าใช้ไม่ระมัดระวัง)

```
CORE-SW1(config)# aaa new-model
```

คำสั่งนี้เปลี่ยนพฤติกรรม Authentication ของอุปกรณ์ **ทันทีที่กด Enter** — โดยไม่ต้องรอ `commit`
หรือ save ใดๆ กล่าวคือ:

| ก่อน `aaa new-model` | หลัง `aaa new-model` |
|---|---|
| Login ใช้ `password` ที่ตั้งไว้บน `line console`/`line vty` (หรือ `login local` ถ้าตั้งไว้) | ระบบเปลี่ยนไปใช้ **AAA Authentication Model** ทันที |
| Enable ใช้ `enable secret` ตรงๆ | Enable ผ่าน AAA authentication method list เช่นกัน |
| ถ้าไม่มี `aaa authentication login` ระบุไว้ | ระบบจะใช้ **Default Method List** ซึ่งค่า Default คือ... **ใช้ local user database** โดยอัตโนมัติ (ถ้าไม่มีการ config authentication list เพิ่มเติม IOS จะ fallback ไปที่ enable password / local user เดิม) |

**ปัญหาที่เกิดจริงในสนาม**: หลาย Engineer เปิด `aaa new-model` แล้วรีบไป config
`aaa authentication login default group TACACS-GROUP` ในบรรทัดถัดไปโดยไม่มี fallback — ถ้า
TACACS+ Server ตัวนั้น **unreachable** (network ยังไม่ up, firewall block, server down) จะเกิด
**Lockout ทันที**: Login ผ่าน Console/SSH ไม่ได้เลยเพราะไม่มีทางใดพิสูจน์ตัวตนสำเร็จ — ต้องเดินทาง
ไป **Console เข้าเครื่องจริง + factory reset** ถึงจะกู้กลับมาได้ (สำหรับอุปกรณ์ Data Center ที่อยู่
ต่างประเทศ/ต่างเมือง นี่คือหายนะระดับต้องบินไปแก้เอง)

### กฎเหล็ก 3 ข้อก่อนแก้ AAA ทุกครั้ง (ต้องจำให้ขึ้นใจ — ใช้ตลอดอาชีพวิศวกรเครือข่าย)

```
┌───────────────────────────────────────────────────────────────────┐
│  กฎเหล็กการทดสอบ AAA (Golden Rule of AAA Testing)                    │
├───────────────────────────────────────────────────────────────────┤
│  1. ตั้ง Fallback เป็น local เสมอ ก่อนเปิด TACACS+/RADIUS จริง         │
│     aaa authentication login default group TAC-GRP local           │
│     (ถ้า TACACS+ ไม่ตอบ → ใช้ local username ต่อได้)                 │
│                                                                     │
│  2. เปิด Console Line ไว้เป็น "ทางหนีสุดท้าย" ที่ไม่โดน AAA บล็อค       │
│     line console 0                                                 │
│      login local          ! หรือใช้ authentication list แยกที่ไม่พึ่ง│
│                            !   TACACS+ เลย เผื่อ TACACS+ ล่มทั้งระบบ  │
│                                                                     │
│  3. ตั้ง reload in <นาที> ไว้ก่อนทดสอบ AAA เสมอ (Safety Net)          │
│     CORE-SW1# reload in 10                                          │
│     ! ถ้า login ใหม่ผ่าน SSH ไม่ได้ → รอ 10 นาที อุปกรณ์จะ reload       │
│     !   คืนค่า running-config เดิม (ที่ยังไม่ save) อัตโนมัติ          │
│     ! ถ้าทุกอย่างโอเค → CORE-SW1# reload cancel  (ยกเลิกก่อนหมดเวลา) │
└───────────────────────────────────────────────────────────────────┘
```

**ตัวอย่างขั้นตอนที่ปลอดภัย (ทำตามลำดับนี้เสมอ ห้ามสลับ):**

```
! ขั้น 1: ตั้ง safety net ก่อนแก้ไขอะไรเกี่ยวกับ AAA
CORE-SW1# reload in 10
Reload scheduled for 04:32:10 UTC Sat Sep 26 2026 (in 10 minutes)
Proceed with reload? [confirm]

! ขั้น 2: เข้า config mode
CORE-SW1# configure terminal

! ขั้น 3: เปิด AAA
CORE-SW1(config)# aaa new-model

! ขั้น 4: ตั้ง default authentication ให้ TACACS+ เป็น primary, local เป็น fallback
!         (รายละเอียดกลุ่ม TACACS+ อยู่ใน Step 214 — ต้อง config ก่อนบรรทัดนี้จริง)
CORE-SW1(config)# aaa authentication login default group AAA-TACACS local

! ขั้น 5: เก็บ console ไว้ไม่ให้พึ่ง AAA ทั้งหมด (ทางหนีสุดท้ายจริงๆ)
CORE-SW1(config)# aaa authentication login CONSOLE-IN local
CORE-SW1(config)# line console 0
CORE-SW1(config-line)# login authentication CONSOLE-IN
CORE-SW1(config-line)# exit

! ขั้น 6: ทดสอบ SSH เข้าใหม่จาก "อีก session" โดยไม่ปิด session เดิม
!         (เปิด terminal ใหม่ ssh เข้าไปทดสอบ ขณะที่ session เดิมยัง login อยู่)

! ขั้น 7: ถ้าทดสอบผ่านหมดแล้วค่อย save และยกเลิก safety net
CORE-SW1# copy running-config startup-config
CORE-SW1# reload cancel
Reload scheduled for 04:32:10 UTC Sat Sep 26 2026 has been cancelled.
```

> **อย่าปิด Session เดิมเด็ดขาดจนกว่าจะยืนยันว่า Session ใหม่ login ได้สำเร็จ** — นี่คือกฎที่ใช้ได้
> กับการแก้ไข AAA/ACL/Firewall Policy ทุกครั้งตลอดอาชีพ ไม่ใช่แค่ Cisco IOS

---

## Step 214 — AAA Authentication Configuration: TACACS+ Primary + Local Fallback

ตอนนี้เราจะ config การเชื่อมต่อไปยัง **AAA-SRV (10.10.99.60)** จริงแบบทีละขั้น

### 1) ประกาศ TACACS+ Server

```
CORE-SW1(config)# tacacs server AAA-SRV1
CORE-SW1(config-server-tacacs)# address ipv4 10.10.99.60
CORE-SW1(config-server-tacacs)# key CiscoLab-TACACS-Key2026!
CORE-SW1(config-server-tacacs)# timeout 5
CORE-SW1(config-server-tacacs)# exit
```

| Parameter | ความหมาย |
|---|---|
| `address ipv4` | IP ของ AAA-SRV ใน VLAN 99 |
| `key` | Shared Secret ที่ต้องตรงกับที่ config ไว้บน AAA-SRV (Cisco ISE/tac_plus/FreeRADIUS ฝั่ง server) — ใช้เข้ารหัส TACACS+ packet ทั้งหมด |
| `timeout` | เวลารอ response จาก Server ก่อนถือว่า "ไม่ตอบ" (ค่า default 5 วินาที) — ค่านี้สำคัญกับ Fallback เพราะถ้าตั้งนานเกินไป ผู้ใช้จะรอ login นานเกินจำเป็นตอน Server ล่ม |

### 2) สร้าง AAA Server Group (รวม Server หลายตัวเข้าด้วยกันเพื่อ Redundancy)

```
CORE-SW1(config)# aaa group server tacacs+ AAA-TACACS
CORE-SW1(config-sg-tacacs+)# server name AAA-SRV1
CORE-SW1(config-sg-tacacs+)# exit

! กำหนด source interface สำหรับ TACACS+ traffic ทั้งหมด (ให้ใช้ Loopback0 เพื่อความคงที่
!  ไม่ว่า interface ทางออกจริงจะเปลี่ยนไปเส้นทางไหนก็ตาม — pattern เดียวกับ Routing Protocol
!  ใน Part 11-14 ที่ใช้ Loopback0 เป็น Router-ID)
CORE-SW1(config)# ip tacacs source-interface Loopback0
```

> **หมายเหตุ Redundancy**: ในระบบจริงมักมี AAA-SRV สำรองอีกตัว (เช่น `AAA-SRV2` ที่ Data Center
> อีกฝั่ง) — เพิ่มเข้า group เดียวกันด้วย `server name AAA-SRV2` อีกบรรทัด อุปกรณ์จะ query ตัวแรก
> ก่อน ถ้า timeout จึงลองตัวถัดไปตามลำดับที่ประกาศ

### 3) เปิด AAA และตั้ง Authentication Method List

```
CORE-SW1(config)# aaa new-model

! Default method list สำหรับ Login (ใช้กับทุก line ที่ไม่ระบุ method list อื่นชัดเจน)
! ลำดับความสำคัญ: 1) ลองถาม TACACS+ group ก่อน  2) ถ้า TACACS+ ไม่ตอบ (server down/unreachable)
!                  ให้ fallback ไปใช้ local username database
CORE-SW1(config)# aaa authentication login default group AAA-TACACS local

! Authentication สำหรับ enable mode (เผื่อยังมีคนใช้ enable password style เดิมอยู่)
CORE-SW1(config)# aaa authentication enable default group AAA-TACACS enable
```

**จุดที่ต้องเข้าใจให้แม่น**: คำว่า **"local" ท้ายสุด** ใน method list ไม่ได้แปลว่า "ใช้ local
เสมอ" — มันหมายถึง **"ใช้เมื่อ TACACS+ ทุก Server ในกลุ่ม unreachable เท่านั้น"** ถ้า TACACS+
ตอบกลับว่า **reject** (username/password ผิด) ระบบจะ**ไม่ fallback ไป local** — มันจะปฏิเสธ
login ทันที (Fallback ทำงานเฉพาะกรณี Server ไม่ตอบ ไม่ใช่กรณี Server ตอบว่า "ไม่อนุญาต")

```
Method List Behavior:

TACACS+ Server ตอบ "Accept"     → login สำเร็จ (จบ ไม่ไป local)
TACACS+ Server ตอบ "Reject"     → login ล้มเหลว (จบ ไม่ไป local!)
TACACS+ Server "ไม่ตอบ/Timeout" → fallback ไป local ✅ (นี่คือกรณีเดียวที่ local จะถูกใช้)
```

### 4) Local User ที่ยังต้องมีไว้เป็น Fallback (ต่อจาก Part 2)

```
CORE-SW1(config)# username admin-local privilege 15 secret Fallback-Adm1n2026!
```

> ตั้งชื่อ local user แยกจาก TACACS+ user ให้ชัดเจน (เช่น `admin-local`) เพื่อไม่ให้สับสนว่า
> login สำเร็จเพราะ TACACS+ หรือเพราะ Fallback — และควรเปลี่ยน secret ของ local fallback account
> นี้เป็นระยะ เก็บไว้ในที่ปลอดภัย (Password Vault) เพราะเป็น "กุญแจฉุกเฉิน" ของทุกอุปกรณ์

### ตรวจสอบผลลัพธ์

```
CORE-SW1# show aaa method-lists authentication
Authen method lists:
     default   : group AAA-TACACS local

CORE-SW1# show tacacs
Server: AAA-SRV1/10.10.99.60/49
        Socket opens:               12
        Socket closes:              12
        Socket aborts:               0
        Socket errors:               0
        Total packets sent:         24
        Total packets recv:         24
        Expected replies:            0
```

---

## Step 215 — AAA Authorization: Exec Authorization และ Command Authorization

**Authorization** ตอบคำถามว่า "หลัง login สำเร็จแล้ว user คนนี้ทำอะไรได้บ้าง" — นี่คือจุดที่
TACACS+ เหนือกว่า RADIUS อย่างชัดเจนที่สุด (ตาม Step 212)

### 1) Exec Authorization — กำหนดว่าจะได้ Privilege Level เท่าไหร่ตอน Login

```
CORE-SW1(config)# aaa authorization exec default group AAA-TACACS local
```

เมื่อ user login ผ่าน TACACS+ สำเร็จ (Authentication OK) ระบบจะยิง query อีกครั้งไปถาม AAA-SRV ว่า
"user นี้ authorize ให้เข้า exec mode ระดับไหน" — ฝั่ง AAA-SRV (เช่นใน Cisco ISE) จะตอบกลับด้วย
**Attribute-Value (AV) Pair** เช่น `priv-lvl=15` ทำให้ user คนนั้น landing ที่ privilege 15 ทันที
โดยไม่ต้องพิมพ์ `enable` เพิ่ม (ถ้า Server ตั้งไว้แบบนั้น) หรือได้ privilege 5 ถ้าเป็น Helpdesk
(ดูรายละเอียดใน Step 217)

### 2) Command Authorization — จุดเด่นที่ RADIUS ทำไม่ได้

```
! ตรวจสอบทุกคำสั่งที่ privilege level 15 (คำสั่ง config/สำคัญ) ก่อนอนุญาตให้รัน
CORE-SW1(config)# aaa authorization commands 15 default group AAA-TACACS local

! (ตัวเลือกเสริม) ตรวจสอบคำสั่งที่ privilege level 1 ด้วย ถ้าต้องการควบคุมละเอียดสุด
CORE-SW1(config)# aaa authorization commands 1 default group AAA-TACACS local
```

**การทำงานจริง**: ทุกครั้งที่ Admin พิมพ์คำสั่งใดๆ ที่อยู่ระดับ privilege 15 (เช่น
`configure terminal`, `interface`, `no shutdown`, `write memory`) — IOS จะส่ง**คำสั่งเต็มบรรทัด
นั้น** ไปถาม AAA-SRV ว่า "user คนนี้อนุญาตให้รันคำสั่งนี้หรือไม่" **ก่อน**ที่จะรันจริง

```
                     Command Authorization Flow
┌──────────┐  1) พิมพ์คำสั่ง       ┌──────────┐  2) ส่งคำสั่งไปถาม   ┌──────────┐
│  Admin    │ ──────────────────► │ IOS      │ ──────────────────► │ AAA-SRV  │
│ (SSH)     │  "interface Gi0/1"  │ (Switch) │   TACACS+ query      │10.10.99.60│
└──────────┘                     └────┬─────┘                     └────┬─────┘
                                        │  3) Permit/Deny ตอบกลับ           │
                                        │ ◄─────────────────────────────────┘
                                  ┌─────┴─────┐
                                  │ ถ้า Permit → รันคำสั่งจริง                │
                                  │ ถ้า Deny   → "% Command authorization    │
                                  │              failed"                     │
                                  └───────────┘
```

**ตัวอย่างจริงบน AAA-SRV (concept)**: กำหนดให้ user group "Helpdesk" ได้รับอนุญาตเฉพาะคำสั่งที่
ขึ้นต้นด้วย `show` และ `ping` เท่านั้น — ถ้า Helpdesk พิมพ์ `configure terminal` แม้จะ login เข้า
มาสำเร็จ (Authentication ผ่าน) แต่ Authorization จะ **Deny** ทันที

### ผลลัพธ์ตอน Deny (สิ่งที่ Admin จะเห็นจริงบนหน้าจอ)

```
helpdesk1@CORE-SW1> enable
Password: ****
CORE-SW1# configure terminal
Command authorization failed.
CORE-SW1#
```

---

## Step 216 — AAA Accounting: บันทึกทุกการกระทำเพื่อ Audit/Compliance

**Accounting** คือการบันทึกว่า "ใครทำอะไร เมื่อไหร่ นานแค่ไหน" ไปเก็บไว้ที่ AAA-SRV — สำคัญมาก
สำหรับการตรวจสอบภายหลัง (Post-incident Forensics), การทำ Compliance (PCI-DSS, ISO 27001) และ
การพิสูจน์ความรับผิด (Accountability) ของ Admin แต่ละคน

### Exec Accounting — บันทึกทุก Session Login

```
CORE-SW1(config)# aaa accounting exec default start-stop group AAA-TACACS
```

- **start** — ส่ง record ทันทีที่ session (login) เริ่มต้น
- **stop** — ส่ง record ทันทีที่ session จบ (logout/disconnect)
- ข้อมูลที่บันทึก: username, เวลาเริ่ม/จบ, source IP ที่ login เข้ามา, ระยะเวลาที่ใช้งาน

### Command Accounting — บันทึกทุกคำสั่งที่รันสำเร็จ

```
CORE-SW1(config)# aaa accounting commands 15 default start-stop group AAA-TACACS
CORE-SW1(config)# aaa accounting commands 1 default start-stop group AAA-TACACS
```

ทุกคำสั่งที่ authorize ผ่านและถูกรันจริง จะถูกส่ง record ไปเก็บที่ AAA-SRV — นี่คือสิ่งที่ทำให้
TACACS+ เป็นมาตรฐานทองคำสำหรับ Compliance Audit ของอุปกรณ์เครือข่าย

### ตัวอย่าง Accounting Log ที่จะเห็นฝั่ง AAA-SRV (concept — รูปแบบขึ้นกับ platform จริง)

```
Sep 26 04:15:02 CORE-SW1 admin1  tty1  10.10.99.100  start  task_id=101
Sep 26 04:15:10 CORE-SW1 admin1  tty1  10.10.99.100  cmd    "configure terminal"
Sep 26 04:15:22 CORE-SW1 admin1  tty1  10.10.99.100  cmd    "interface GigabitEthernet1/0/5"
Sep 26 04:15:31 CORE-SW1 admin1  tty1  10.10.99.100  cmd    "shutdown"
Sep 26 04:16:45 CORE-SW1 admin1  tty1  10.10.99.100  stop   task_id=101 elapsed_time=103
```

> ข้อมูลนี้ตอบคำถามที่ Manager/Auditor ถามบ่อยที่สุดเวลา Incident เกิดขึ้น: **"ใคร shutdown
> port นี้ และเมื่อไหร่?"** — โดยไม่ต้องพึ่ง syslog ของอุปกรณ์เพียงอย่างเดียว (ซึ่ง Admin ที่ตั้งใจ
> ก่อกวนอาจลบ log local ทิ้งได้ แต่ลบ Accounting record ที่อยู่บน AAA-SRV แยก server ไม่ได้)

### ตรวจสอบ AAA Method List ทั้งหมดในคำสั่งเดียว

```
CORE-SW1# show running-config | include ^aaa
aaa new-model
aaa authentication login default group AAA-TACACS local
aaa authentication enable default group AAA-TACACS enable
aaa authorization exec default group AAA-TACACS local
aaa authorization commands 15 default group AAA-TACACS local
aaa authorization commands 1 default group AAA-TACACS local
aaa accounting exec default start-stop group AAA-TACACS
aaa accounting commands 15 default start-stop group AAA-TACACS
aaa accounting commands 1 default start-stop group AAA-TACACS
```

---

## Step 217 — Privilege Level เชิงลึก และ Role-Based CLI Access (`parser view`)

[Part 2](part-002-ios-cli-basics.md) แนะนำว่า Privilege Level มีตั้งแต่ 0-15 (User EXEC=1,
Privileged EXEC=15) และบอกไว้ว่าเรื่องนี้จะเรียนละเอียดคู่กับ AAA ใน Part นี้ — ถึงเวลานั้นแล้ว

### ตาราง Privilege Level มาตรฐาน

| Level | ความหมาย | Default Commands |
|---|---|---|
| 0 | จำกัดสุด | `disable`, `enable`, `exit`, `help`, `logout` |
| 1 | User EXEC (ค่า default หลัง login) | `show` บางคำสั่งพื้นฐาน, `ping`, `traceroute` |
| 2-14 | **ไม่มี default commands กำหนดไว้** — เป็นช่องสำหรับ Custom Role | (กำหนดเองด้วย `privilege exec level`) |
| 15 | Privileged EXEC (เทียบเท่า `enable` เต็มรูปแบบ) | ทุกคำสั่ง รวม `configure terminal` |

### สร้าง Custom Privilege Level 5 สำหรับ "Helpdesk Role"

โจทย์: Helpdesk ต้อง**ดูสถานะอุปกรณ์และทดสอบ connectivity ได้** แต่**ห้าม config อะไรเด็ดขาด**

```
CORE-SW1(config)# privilege exec level 5 show
CORE-SW1(config)# privilege exec level 5 show running-config
CORE-SW1(config)# privilege exec level 5 ping
CORE-SW1(config)# privilege exec level 5 traceroute
CORE-SW1(config)# privilege exec level 5 show ip interface brief
```

> **ข้อควรระวัง**: `privilege exec level 5 show running-config` เป็นคำสั่งที่ต้อง**ระวังมาก**
> เพราะ running-config มักมี TACACS+ key, SNMP community, หรือ password ที่ยังไม่ encrypt เต็ม
> รูปแบบ (แม้เปิด `service password-encryption` แล้วก็ยังเป็น Type 7 ที่ decrypt ได้ง่าย) —
> Enterprise จริงมักจะ**ไม่**ให้ level ต่ำกว่า 15 เห็น running-config เต็มไฟล์ แต่จะให้เห็นเฉพาะ
> `show running-config | section interface` หรือใช้ `parser view` (ดูหัวข้อถัดไป) ซึ่งควบคุมได้
> ละเอียดกว่า

```
! Map local user เข้ากับ privilege level 5 (fallback account สำหรับ helpdesk)
CORE-SW1(config)# username helpdesk1 privilege 5 secret Help2026-Local!

! Enable Secret เฉพาะสำหรับ level 5 (ถ้าต้องการให้ enable แยกจาก level 15)
CORE-SW1(config)# enable secret level 5 Help-Enable2026!
```

การ login ผ่าน **TACACS+** ก็ทำงานเช่นเดียวกัน — ฝั่ง AAA-SRV จะตอบ AV-Pair `priv-lvl=5` กลับมา
ให้กับ user group "Helpdesk" โดยที่ IOS ไม่ต้องมี `username` local สำหรับทุกคนเลย (มีแค่
`helpdesk1` ไว้เป็น fallback ตามกฎ Step 213-214)

### Role-Based CLI Access — `parser view` (ละเอียดกว่า Privilege Level)

Privilege Level มีข้อจำกัด: มันเป็น**ลำดับขั้น (Hierarchical)** — ถ้า level 5 ได้คำสั่ง A, B, C
แล้ว level 10 ต้องได้ A, B, C ทั้งหมดของ level ต่ำกว่าด้วยเสมอ (ปรับแต่งแยกกันแบบสมบูรณ์ไม่ได้)
**Role-Based CLI Access (`parser view`)** แก้ปัญหานี้ด้วยการสร้าง "View" ที่เลือกคำสั่งได้อิสระ
ไม่ผูกกับลำดับขั้นใดๆ

```
! ต้องเปิด aaa new-model ก่อนเสมอ (ทำไปแล้วใน Step 213)
CORE-SW1(config)# parser view HELPDESK-VIEW
CORE-SW1(config-view)# secret HelpdeskView2026!
CORE-SW1(config-view)# commands exec include show
CORE-SW1(config-view)# commands exec include ping
CORE-SW1(config-view)# commands exec include traceroute
CORE-SW1(config-view)# commands exec include-exclusive show running-config
CORE-SW1(config-view)# exit
```

**การใช้งาน (เข้า View):**

```
CORE-SW1> enable view HELPDESK-VIEW
Password: ****
CORE-SW1#
% ตอนนี้ CLI จะเห็นเฉพาะคำสั่งที่ประกาศไว้ใน view เท่านั้นตอนกด ?
```

| เปรียบเทียบ | Privilege Level | Parser View (RBAC) |
|---|---|---|
| ลำดับขั้น | ต้องเรียงลำดับ (Hierarchical) | อิสระ ไม่ผูกลำดับ |
| ความละเอียด | หยาบกว่า (per-command เท่านั้น) | ละเอียดกว่า (รวม keyword/argument ได้) |
| ใช้คู่กับ AAA | ได้ (`priv-lvl` AV-Pair) | ได้ (`aaa authorization ... local`/TACACS+ ก็ map view ได้) |
| ความซับซ้อนในการดูแล | ง่ายกว่า เหมาะกับ Role น้อย | ซับซ้อนกว่าแต่ Enterprise ขนาดใหญ่ที่มีหลาย Role ควรใช้ |

> **แนวทางปฏิบัติ**: Lab/องค์กรขนาดเล็ก-กลางที่มี Role ไม่เกิน 3-4 แบบ (Admin/NetOps/Helpdesk/
> Auditor) ใช้ **Custom Privilege Level** ก็เพียงพอและง่ายต่อการดูแล — องค์กรขนาดใหญ่ที่มี Role
> ซับซ้อนมากและต้องควบคุม argument ระดับคำสั่งย่อยจริงๆ ค่อยลงทุนกับ `parser view`

---

## Step 218 — Device Hardening Checklist ฉบับสมบูรณ์

นอกจาก AAA แล้ว มี Service/Feature อีกหลายตัวที่เปิดอยู่โดย Default บน Cisco IOS ซึ่งไม่จำเป็น
สำหรับการทำงานปกติ และอาจเป็นช่องโหว่ (Attack Surface) — ต้องปิดทิ้งเป็นมาตรฐานทุกอุปกรณ์

### Checklist เต็มรูปแบบ

| # | คำสั่ง | เหตุผล |
|---|---|---|
| 1 | `no ip http server` | ปิด Web GUI แบบ HTTP (plaintext) — ไม่จำเป็นถ้าบริหารผ่าน CLI/API เป็นหลัก |
| 2 | `no ip http secure-server` | ปิด Web GUI แบบ HTTPS ด้วย ถ้าไม่ได้ใช้ GUI จัดการอุปกรณ์เลย (ถ้าต้องใช้ WebUI ให้เปิดเฉพาะ secure-server และจำกัดด้วย ACL) |
| 3 | `no cdp run` (ที่ external-facing interface หรือทั้งอุปกรณ์ตาม Part 2) | ป้องกันข้อมูลอุปกรณ์ภายในรั่วไหลออกไปยัง WAN/Internet-facing link |
| 4 | `no service pad` | ปิด Packet Assembler/Disassembler (X.25 legacy) — ไม่มีใครใช้แล้วในปัจจุบัน แต่เปิด default บน IOS รุ่นเก่า |
| 5 | `no ip source-route` | ป้องกัน IP Source Routing ซึ่งใช้ bypass routing policy/security ได้ (attacker กำหนด path เองผ่าน header) |
| 6 | `no ip finger` (หรือ `no service finger`) | ปิด Finger Protocol (TCP/79) ที่เปิดเผยข้อมูล user ที่ login อยู่ให้คนนอก |
| 7 | `service password-encryption` | เข้ารหัส password ทุกตัวใน running-config เป็น Type 7 (weak แต่ดีกว่า plaintext — ย้ำจาก Part 2) |
| 8 | `no service tcp-small-servers` / `no service udp-small-servers` | ปิด Legacy service เช่น echo, chargen, discard (เป้าหมาย DoS/Amplification attack เก่า) |
| 9 | `no service dhcp` (บนอุปกรณ์ที่ไม่ได้ทำ DHCP Server จริง) | ป้องกัน rogue DHCP relay/server function โดยไม่ตั้งใจ |
| 10 | `login block-for X attempts Y within Z` | ป้องกัน Brute-force password guessing (รายละเอียดด้านล่าง) |
| 11 | `banner motd` / `banner login` | Legal Notice ก่อน login (ทำไปแล้วใน Part 2 — ย้ำว่าเป็นข้อบังคับทางกฎหมายในหลายประเทศก่อนดำเนินคดี unauthorized access) |
| 12 | `exec-timeout` บนทุก line | ตัดการเชื่อมต่อ session ที่ idle นานเกินไปอัตโนมัติ (ทำไปแล้ว Part 2 — ย้ำว่าต้องมีทุก line ไม่ใช่แค่ console) |
| 13 | `no ip domain-lookup` | ป้องกัน CLI ค้างพยายาม DNS resolve คำสั่งที่พิมพ์ผิด (ทำไปแล้ว Part 2) |

### `login block-for` — ป้องกัน Brute-Force Password Guessing โดยละเอียด

```
CORE-SW1(config)# login block-for 120 attempts 3 within 60
```

อ่านว่า: "ถ้ามีการ login ผิด **3 ครั้ง** ภายในเวลา **60 วินาที** ให้ **บล็อคการ login ทั้งหมด**
เป็นเวลา **120 วินาที**" — Block นี้มีผลกับ**ทุก IP** ที่พยายาม login เข้ามา (ไม่เฉพาะ IP ที่ทำผิด)
เพื่อป้องกัน Distributed Brute-force ด้วย

```
! เสริม: ยกเว้น Management Subnet ของเราเองจากการถูก Block (ป้องกัน Admin ตัวเองโดนบล็อคด้วย)
CORE-SW1(config)# ip access-list standard NO-BLOCK-MGMT
CORE-SW1(config-std-nacl)# permit 10.10.99.0 0.0.0.255
CORE-SW1(config-std-nacl)# exit
CORE-SW1(config)# login quiet-mode access-class NO-BLOCK-MGMT

! เพิ่ม delay ระหว่างความพยายาม login แต่ละครั้ง (ชะลอ automated brute-force script)
CORE-SW1(config)# login delay 1

! เปิด logging ทุกครั้งที่มีการ login สำเร็จ/ล้มเหลว (สำหรับตรวจสอบ pattern โจมตี)
CORE-SW1(config)# login on-success log
CORE-SW1(config)# login on-failure log
```

**ตรวจสอบสถานะ:**

```
CORE-SW1# show login
     A login delay of 1 second is applied.
     Quiet-Mode access list NO-BLOCK-MGMT is applied.
     All successful login is logged.
     All failed login is logged.

     Router NOT enabled to watch for login Attacks
```

### คำสั่งปิด Service ที่ไม่ใช้ (Copy-Paste Block เดียว)

```
CORE-SW1(config)# no ip http server
CORE-SW1(config)# no ip http secure-server
CORE-SW1(config)# no service pad
CORE-SW1(config)# no ip source-route
CORE-SW1(config)# no ip finger
CORE-SW1(config)# no service tcp-small-servers
CORE-SW1(config)# no service udp-small-servers
CORE-SW1(config)# no service dhcp
CORE-SW1(config)# service password-encryption
```

---

## Step 219 — Control Plane Security ภาพรวม (Preview) และการจำกัด Management Plane

ทุกอุปกรณ์ Cisco แบ่งการทำงานภายในออกเป็น 3 "Plane" — เข้าใจโครงสร้างนี้จะช่วยให้เห็นภาพว่า AAA
และ Hardening ที่ทำมาทั้งหมดใน Part นี้ปกป้อง **Plane ไหน**

```
┌─────────────────────────────────────────────────────────────┐
│  MANAGEMENT PLANE  — SSH, SNMP, TACACS+/AAA, Syslog, NTP       │  <- Part นี้ปกป้องส่วนนี้เป็นหลัก
├─────────────────────────────────────────────────────────────┤
│  CONTROL PLANE     — OSPF/EIGRP/BGP Hello, STP BPDU, ARP, CDP  │  <- ป้องกันด้วย CoPP (Part 46)
├─────────────────────────────────────────────────────────────┤
│  DATA PLANE        — Traffic ของ User ที่ Forward ผ่านอุปกรณ์  │  <- ป้องกันด้วย ACL/Firewall (Part 43-45)
└─────────────────────────────────────────────────────────────┘
```

- **Management Plane**: ทุกอย่างที่ใช้ "บริหารจัดการ" ตัวอุปกรณ์เอง — SSH, TACACS+, SNMP —
  Part นี้ (AAA + Hardening) และ [Part 14](part-014-acl-fundamentals.md) (ACL) คือแนวป้องกันหลัก
- **Control Plane**: Process ภายในที่ทำให้อุปกรณ์ "รู้ทาง" เช่น Routing Protocol, STP — ถ้าถูก
  โจมตีด้วย Packet จำนวนมาก (เช่น ปลอม OSPF Hello ถี่ๆ) CPU อุปกรณ์อาจพังจนควบคุมไม่ได้เลย
- **Data Plane**: Traffic ของ User ทั่วไปที่วิ่งผ่านอุปกรณ์ (Forwarding เฉยๆ ไม่แตะ CPU มากถ้าใช้
  Hardware Switching/CEF)

### Control Plane Policing (CoPP) — Preview

**Control Plane Policing** คือการใช้ QoS Policy (`class-map`/`policy-map`) จำกัด **Rate ของ
Traffic ที่จะเข้าสู่ CPU ของอุปกรณ์เอง** (ไม่ใช่ traffic ที่ผ่านๆ ไป) — ป้องกันไม่ให้ Traffic
ปลอม (เช่น ARP Flood, ปลอม Routing Protocol Packet จำนวนมาก) ทำให้ CPU พุ่งจนอุปกรณ์ล่มหรือ
ตอบสนอง Login/SSH ไม่ได้เลย

```
! ตัวอย่าง concept เท่านั้น (รายละเอียดเต็มรูปแบบใน Part 46)
CORE-SW1(config)# class-map match-any COPP-CRITICAL
CORE-SW1(config-cmap)#  match access-group name OSPF-BGP-TRAFFIC
CORE-SW1(config)# policy-map COPP-POLICY
CORE-SW1(config-pmap)#  class COPP-CRITICAL
CORE-SW1(config-pmap-c)#   police 512000
CORE-SW1(config)# control-plane
CORE-SW1(config-cp)#  service-policy input COPP-POLICY
```

> **หมายเหตุ**: นี่เป็นแค่ Preview เพื่อให้เห็นภาพรวมของ Security-in-Depth ครบทุก Plane —
> การออกแบบ Class-map/Policy-map สำหรับ CoPP แบบละเอียด (แยก class ตาม traffic type: routing
> protocol, management, monitoring, default) จะเรียนเต็มรูปแบบใน **Part 46**

### จำกัด Management Plane ด้วย VTY Access-Class (ทบทวนจาก Part 14)

หลักการ Defense-in-Depth ที่ดีที่สุดคือ**ไม่ให้ Network Access เข้ามาถึง TACACS+/SSH ได้ตั้งแต่แรก
เว้นแต่มาจาก MGMT VLAN เท่านั้น** — แม้ AAA จะแข็งแรงแค่ไหน ก็ควรมีชั้นป้องกันนี้เสริมไว้เสมอ

```
CORE-SW1(config)# ip access-list standard MGMT-ONLY-ACCESS
CORE-SW1(config-std-nacl)# remark ** อนุญาตเฉพาะ MGMT VLAN 99 เข้า VTY ได้ **
CORE-SW1(config-std-nacl)# permit 10.10.99.0 0.0.0.255
CORE-SW1(config-std-nacl)# deny   any log
CORE-SW1(config-std-nacl)# exit

CORE-SW1(config)# line vty 0 15
CORE-SW1(config-line)# access-class MGMT-ONLY-ACCESS in
CORE-SW1(config-line)# exit
```

ผลลัพธ์: แม้ Attacker จะรู้ Username/Password ของ TACACS+ หรือ Local Fallback ก็ตาม แต่ถ้า
Source IP ไม่ได้มาจาก `10.10.99.0/24` — Connection จะถูก**ปฏิเสธที่ระดับ TCP/IP ก่อนถึงขั้น
Authentication ด้วยซ้ำ** (ACL ทำงานก่อน AAA เสมอในลำดับการประมวลผลของ VTY line)

---

## Step 220 — Lab: Deploy AAA เต็มรูปแบบทั้ง Topology + Hardening ทุกอุปกรณ์

ตอนนี้เรารวมทุกอย่างจาก Step 211-219 เข้าเป็น Configuration Template เดียวที่ Deploy ได้จริงกับ
**CORE-SW1, CORE-SW2, DIST-SW1, DIST-SW2, DIST-SW3, DIST-SW4** ทั้งหมด — เปลี่ยนแค่ hostname และ
IP ตามตารางด้านล่าง (Pattern เดียวกับ Initial Setup Script ใน [Part 2 Step 20]
(part-002-ios-cli-basics.md#step-20))

### ตัวแปรที่ต้องเปลี่ยนต่ออุปกรณ์

| Hostname | Loopback0 (Router-ID) | MGMT SVI IP (VLAN 99) |
|---|---|---|
| CORE-SW1 | 1.1.1.1/32 | 10.10.99.10/24 |
| CORE-SW2 | 1.1.1.2/32 | 10.10.99.11/24 |
| DIST-SW1 | 1.1.1.11/32 | 10.10.99.21/24 |
| DIST-SW2 | 1.1.1.12/32 | 10.10.99.22/24 |
| DIST-SW3 | 1.1.1.13/32 | 10.10.99.23/24 |
| DIST-SW4 | 1.1.1.14/32 | 10.10.99.24/24 |

> IP เหล่านี้เป็น Host Address ภายใน VLAN 99 (`10.10.99.0/24`) ตาม [IP Address Plan]
> (00-ip-address-plan.md) — ไม่ชนกับ Default Gateway HSRP VIP (`.1`), TFTP Server (`.50` ตาม
> Part 2), หรือ **AAA-SRV (`.60`)** ที่แนะนำใน Part นี้

### Full Running-Config Template — ใช้กับ CORE-SW1 เป็นตัวอย่างสมบูรณ์

```
! ============================================================
!  AAA & DEVICE HARDENING — FULL TEMPLATE (CORE-SW1)
!  ต่อยอดจาก Initial Setup Script (Part 2) — ห้ามลบ config เดิม
! ============================================================
configure terminal

! --- Safety Net ก่อนแก้ AAA (Step 213) ---
! ทำที่ privileged exec ก่อนเข้า config mode จริง:
!   CORE-SW1# reload in 10

! --- TACACS+ Server & Group (Step 214) ---
tacacs server AAA-SRV1
 address ipv4 10.10.99.60
 key CiscoLab-TACACS-Key2026!
 timeout 5
 exit
!
aaa group server tacacs+ AAA-TACACS
 server name AAA-SRV1
 exit
!
ip tacacs source-interface Loopback0

! --- เปิด AAA (Step 213) ---
aaa new-model

! --- Authentication: TACACS+ primary, local fallback (Step 214) ---
aaa authentication login default group AAA-TACACS local
aaa authentication enable default group AAA-TACACS enable
aaa authentication login CONSOLE-IN local

! --- Authorization: Exec + Command-level (Step 215) ---
aaa authorization exec default group AAA-TACACS local
aaa authorization commands 15 default group AAA-TACACS local
aaa authorization commands 1 default group AAA-TACACS local

! --- Accounting: Exec + Command-level (Step 216) ---
aaa accounting exec default start-stop group AAA-TACACS
aaa accounting commands 15 default start-stop group AAA-TACACS
aaa accounting commands 1 default start-stop group AAA-TACACS

! --- Local Fallback Users (Step 214, 217) ---
username admin-local privilege 15 secret Fallback-Adm1n2026!
username helpdesk1 privilege 5 secret Help2026-Local!

! --- Custom Privilege Level 5: Helpdesk Role (Step 217) ---
privilege exec level 5 show
privilege exec level 5 show running-config
privilege exec level 5 ping
privilege exec level 5 traceroute
privilege exec level 5 show ip interface brief
enable secret level 5 Help-Enable2026!

! --- Console Line: ไม่พึ่ง TACACS+ เต็มรูปแบบ (Step 213) ---
line console 0
 login authentication CONSOLE-IN
 exec-timeout 10 0
 logging synchronous
 exit

! --- VTY Line: ใช้ default AAA method list + จำกัดด้วย ACL (Step 219) ---
ip access-list standard MGMT-ONLY-ACCESS
 remark ** อนุญาตเฉพาะ MGMT VLAN 99 เข้า VTY ได้ **
 permit 10.10.99.0 0.0.0.255
 deny   any log
 exit
!
line vty 0 15
 access-class MGMT-ONLY-ACCESS in
 transport input ssh
 exec-timeout 10 0
 exit

! --- Login Brute-Force Protection (Step 218) ---
ip access-list standard NO-BLOCK-MGMT
 permit 10.10.99.0 0.0.0.255
 exit
!
login block-for 120 attempts 3 within 60
login quiet-mode access-class NO-BLOCK-MGMT
login delay 1
login on-success log
login on-failure log

! --- Device Hardening Checklist (Step 218) ---
no ip http server
no ip http secure-server
no service pad
no ip source-route
no ip finger
no service tcp-small-servers
no service udp-small-servers
no service dhcp
service password-encryption

end
copy running-config startup-config

! --- ยกเลิก safety net หลังยืนยันว่า login ผ่านหมดแล้ว ---
!   CORE-SW1# reload cancel
```

> Apply Template นี้กับ **CORE-SW2, DIST-SW1, DIST-SW2, DIST-SW3, DIST-SW4** ทุกตัว โดยเปลี่ยน
> เฉพาะค่าที่ผูกกับ Loopback0/Hostname ที่ตั้งไว้แล้วตั้งแต่ Part 2/Part 11-14 — ส่วน AAA-SRV,
> Key, Method List Name, ACL ทั้งหมดใช้**ค่าเดียวกันทุกอุปกรณ์**เพื่อความสอดคล้อง (Consistency)
> ทั้ง Topology ตามหลักการ Centralized AAA ใน Step 211

### Verification: `show aaa servers`

```
CORE-SW1# show aaa servers
RADIUS: id 0, priority 1, host 10.10.99.60, auth-port 1812, acct-port 1813
     State: current UP, duration 9s, previous duration 0s
     Dead: total time 0s, count 0

TACACS+: id 1, priority 1, host 10.10.99.60, port 49
     State: current UP, duration 3625s, previous duration 0s
     Dead: total time 0s, count 0
     Quiet time left: 0
     Flags: (0x0)
     Current server state: UP, duration 3625s, previous duration 0s
     Total connections: 24
     Total requests sent: 48
     Total responses received: 48
     Total sync requests sent: 0
     Total send errors: 0
     Total receive errors: 0
     Total tries for current server: 0
```

### Verification: `test aaa` — ทดสอบ Authentication แบบ On-Demand โดยไม่ต้อง Login จริง

```
CORE-SW1# test aaa group AAA-TACACS admin1 CorrectPassword123! legacy
Attempting authentication test to server-group AAA-TACACS using tacacs+
User was successfully authenticated

CORE-SW1# test aaa group AAA-TACACS admin1 WrongPassword legacy
Attempting authentication test to server-group AAA-TACACS using tacacs+
User authentication request was rejected by server
```

> คำสั่ง `test aaa` คือเครื่องมือที่ **สำคัญที่สุด**สำหรับ Debug AAA — ใช้ทดสอบว่า Server ตอบ
> ถูกต้องหรือไม่ **โดยไม่กระทบ Session ที่ login อยู่จริง** ควรใช้คำสั่งนี้ทดสอบทุกครั้งก่อนตัด
> Local Fallback ทิ้ง

### Debug: ดู TACACS+ Exchange แบบ Real-time

```
CORE-SW1# debug tacacs
CORE-SW1# debug aaa authentication
CORE-SW1# terminal monitor

! ตัวอย่าง output ตอน login (ย่อ):
TPLUS: Queuing AAA Authentication request 142 for processing
TPLUS: processing authentication start request id 142
TPLUS: Authentication start packet created for 142()
TPLUS: Using server 10.10.99.60
TPLUS(00000142)/0/NB_WAIT/-1201282740: Recv'd size 12 for 3 bytes data
TPLUS(00000142)/0/NB_WAIT: TCP Wait, cont=1
TPLUS(00000142)/0/NB_WAIT/-1201282740: Recv'd size 27 for 21 bytes data
TPLUS(00000142)/0/NB_WAIT: Processing the reply packet
TPLUS: received authen response status = PASS
AAA/AUTHEN (142): status = PASS

! ปิด debug ทันทีหลังเช็คจบ (กฎเหล็กจาก Part 2)
CORE-SW1# undebug all
```

### Simulated Login: TACACS+ Authentication + Command Authorization ทำงานพร้อมกัน

```
$ ssh admin1@10.10.99.10
Password: **************

===================================================
  Enterprise Lab - Cisco CCNA/CCNP/CCIE Training Course
  Unauthorized access is prohibited.
===================================================

CORE-SW1# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     10.10.99.10     YES manual up                    up
Vlan99                 10.10.99.10     YES manual up                    up
Loopback0              1.1.1.1         YES manual up                    up

CORE-SW1# configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
CORE-SW1(config)# interface GigabitEthernet1/0/10
CORE-SW1(config-if)# description ** Test command authorization **
CORE-SW1(config-if)# end
CORE-SW1#
```

*(สำหรับ `admin1` ที่ AAA-SRV กำหนด priv-lvl=15 และอนุญาตทุกคำสั่ง — Authorization ผ่านทุก
ขั้นแบบ transparent ผู้ใช้ไม่รู้สึกถึงความช้าเพิ่มขึ้นเลย)*

**เปรียบเทียบกับ `helpdesk1` ที่ AAA-SRV กำหนด priv-lvl=5 และอนุญาตเฉพาะ show/ping:**

```
$ ssh helpdesk1@10.10.99.10
Password: **************

CORE-SW1# show version
Cisco IOS XE Software, Version 17.09.04a
...

CORE-SW1# configure terminal
Command authorization failed.

CORE-SW1# ping 10.10.30.5

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.30.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

Accounting Record ที่ AAA-SRV จะเห็นจาก 2 session นี้ (concept):

```
admin1     10.10.99.10  start  "SSH session"
admin1     10.10.99.10  cmd    "configure terminal"           PERMIT
admin1     10.10.99.10  cmd    "interface GigabitEthernet1/0/10" PERMIT
admin1     10.10.99.10  cmd    "description ..."               PERMIT
admin1     10.10.99.10  stop

helpdesk1  10.10.99.10  start  "SSH session"
helpdesk1  10.10.99.10  cmd    "show version"                  PERMIT
helpdesk1  10.10.99.10  cmd    "configure terminal"             DENY
helpdesk1  10.10.99.10  cmd    "ping 10.10.30.5"                PERMIT
helpdesk1  10.10.99.10  stop
```

ทั้ง 2 Session นี้ยืนยันว่า Architecture ที่เราสร้างขึ้นทำงานได้ครบทั้ง 3 ฟังก์ชันของ AAA:
**Authentication** (พิสูจน์ตัวตนผ่าน TACACS+ สำเร็จทั้งคู่), **Authorization** (admin1 ทำได้ทุก
อย่าง, helpdesk1 ถูกบล็อคคำสั่ง config), และ **Accounting** (ทุก action ถูกบันทึกไว้ที่ AAA-SRV
แยกตาม username ชัดเจน)

---

## แบบฝึกหัดทวนความเข้าใจ Part 22

1. อธิบายความแตกต่างระหว่าง Authentication, Authorization, และ Accounting ใน AAA แต่ละอย่างตอบ
   คำถามอะไร?
2. เพราะเหตุใด TACACS+ จึงเหมาะกับ Device Administration มากกว่า RADIUS โดยเฉพาะเรื่อง
   Command Authorization?
3. ในคำสั่ง `aaa authentication login default group AAA-TACACS local` คำว่า `local` จะถูกใช้
   งานเมื่อไหร่ — เมื่อ TACACS+ ตอบ Reject หรือเมื่อ TACACS+ ไม่ตอบสนอง?
4. ก่อนทดสอบ AAA configuration ใหม่บนอุปกรณ์ Production ควรทำอะไรเป็น Safety Net เพื่อป้องกัน
   Lockout?
5. Privilege Level แบบดั้งเดิม (0-15) ต่างจาก Role-Based CLI Access (`parser view`) อย่างไร
   ในเรื่องความละเอียดของการควบคุมสิทธิ์?

**เฉลย:**

1. **Authentication** = พิสูจน์ตัวตนว่า "คุณเป็นใคร" (username/password ถูกไหม), **Authorization**
   = กำหนดว่า "ทำอะไรได้บ้าง" หลัง login สำเร็จ (privilege level, คำสั่งที่รันได้), **Accounting**
   = บันทึกว่า "ทำอะไรไปแล้วบ้าง เมื่อไหร่ นานแค่ไหน" สำหรับ Audit
2. เพราะ TACACS+ แยก Authentication และ Authorization เป็นขั้นตอนอิสระกัน ทำให้ตรวจสอบ
   **คำสั่งแต่ละคำสั่ง (Command Authorization)** ก่อนอนุญาตให้รันได้ — RADIUS รวม AuthN/AuthZ
   เข้าด้วยกันในขั้นตอนเดียว จึงทำ Command-level Authorization ไม่ได้โดยธรรมชาติ
3. `local` จะถูกใช้ **เมื่อ TACACS+ Server ทุกตัวในกลุ่มไม่ตอบสนอง (Timeout/Unreachable)** เท่านั้น
   — ถ้า TACACS+ ตอบว่า Reject (username/password ผิด) ระบบจะปฏิเสธ login ทันที ไม่ fallback
   ไปที่ local
4. ตั้ง `reload in <นาที>` ไว้ก่อนแก้ไข AAA เสมอ (Safety Net ที่จะ reload คืนค่า config เดิมถ้า
   Lockout เกิดขึ้น) พร้อมทั้งเปิด Console Line ที่ไม่พึ่ง AAA ทั้งหมดไว้เป็นทางหนีสุดท้าย และ
   ห้ามปิด session เดิมจนกว่าจะยืนยันว่า session ใหม่ login ได้สำเร็จ
5. Privilege Level เป็นแบบ **ลำดับขั้น (Hierarchical)** — Level สูงกว่าต้องรวมคำสั่งทั้งหมดของ
   Level ต่ำกว่าเสมอ ปรับแยกอิสระไม่ได้ ส่วน **Parser View** อิสระจากลำดับขั้น เลือกคำสั่ง/
   argument ได้ละเอียดกว่าโดยไม่ผูกเงื่อนไขว่าต้องมีคำสั่งของ view อื่นรวมอยู่ด้วย

---

## สรุป Part 22

Part นี้ยกระดับความปลอดภัยของ**ตัวอุปกรณ์เอง**จาก Local-Only Authentication (Part 2) ไปสู่
**Centralized AAA** ด้วย TACACS+ ผ่าน AAA-SRV (10.10.99.60) ครบทั้ง 3 ฟังก์ชัน — Authentication,
Authorization (รวม Command Authorization ระดับคำสั่ง), และ Accounting — พร้อมกฎเหล็กเรื่อง
Safety Net ที่ต้องใช้ทุกครั้งก่อนแก้ไข AAA จริง, Custom Privilege Level และ Role-Based CLI Access
สำหรับ Helpdesk Role, และปิดท้ายด้วย Device Hardening Checklist ฉบับสมบูรณ์ที่ Deploy ได้ทันที
กับทุกอุปกรณ์ใน Topology ✅

**พร้อมสำหรับ Part 23**: เราจะเริ่มเข้าสู่โลกของ **Network Automation** — ทำไมต้อง Automate,
พื้นฐาน Python สำหรับ Network Engineer, และการเขียน Script แรกเพื่อดึงข้อมูลจากอุปกรณ์ Cisco
หลายตัวพร้อมกัน (แทนที่จะพิมพ์คำสั่งซ้ำๆ ทีละอุปกรณ์แบบที่ทำมาตลอด 22 Part ที่ผ่านมา)

**ไปต่อ:** [Part 23 — Automation Basics →](part-023-automation-basics.md)

