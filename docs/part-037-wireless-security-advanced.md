# Part 37 — Wireless Security Advanced (802.1X, WPA2/3-Enterprise, TrustSec Preview)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 361–370 จาก 1000**

> ต่อจาก [Part 36 — Wireless Architecture Deep Dive →](part-036-wireless-architecture-deep-dive.md)
> ที่เจาะลึกสถาปัตยกรรม WLC ระดับ CCNP แล้ว (High Availability, N+1 Redundancy, Mobility Group
> ขั้นสูง, RF Design เชิงลึก) Part นี้จะกลับมาที่หัวข้อ **Wireless Security** ที่ [Part 20]
> (part-020-wireless-fundamentals.md) ทิ้งไว้เป็นการบ้าน — ตอนนั้นเราใช้ **WPA2-Personal (PSK)**
> เป็น Baseline เพื่อพิสูจน์ว่า Client เชื่อมต่อ VLAN 40 (WIFI) ได้จริงเท่านั้น และ [Part 22]
> (part-022-aaa-device-hardening.md) ได้สร้าง **AAA-SRV (10.10.99.60)** ขึ้นมาเป็น TACACS+
> Server สำหรับดูแล Device Administration ของ CORE/DIST Switch
>
> Part นี้คือจุดที่ทั้งสองเส้นเรื่องมาบรรจบกัน: เราจะ (1) เปิดใช้งาน **RADIUS** บน AAA-SRV ตัวเดิม
> เพื่อทำหน้าที่ **Network Access Authentication** ให้ Wireless Client ผ่าน **802.1X**, (2) Upgrade
> SSID จาก PSK ไปเป็น **WPA2/WPA3-Enterprise** เต็มรูปแบบ, (3) เรียนรู้ **Dynamic VLAN Assignment**
> และ **Guest CWA Design**, และ (4) ปิดท้ายด้วยการแนะนำ **Cisco TrustSec** แบบ Preview ก่อนที่จะ
> เจอ **SD-Access** เต็มรูปแบบใน Part 38

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 361 | WPA2/WPA3-Enterprise Architecture: บทบาทของ 802.1X Framework และเปรียบเทียบ EAP Method |
| 362 | RADIUS Deep Dive สำหรับ Wireless: ทำไมต้อง RADIUS ไม่ใช่ TACACS+ |
| 363 | คอนฟิก 802.1X/WPA2-Enterprise WLAN บน Catalyst 9800: สร้าง SSID "ENTERPRISE-SECURE" |
| 364 | WPA3-Enterprise และ WPA3-Enterprise 192-bit Mode |
| 365 | Certificate-based EAP-TLS เชิงลึก: PKI Trust Chain และบทบาทของ Cisco ISE |
| 366 | Dynamic VLAN Assignment ผ่าน RADIUS Attribute (Tunnel-Private-Group-ID) |
| 367 | Guest Wireless Design: Central Web Authentication (CWA) และ Guest Anchor WLC |
| 368 | Wireless Intrusion Prevention (wIPS) พื้นฐาน: Rogue AP Detection/Classification/Containment |
| 369 | Verification & Troubleshooting: 802.1X, RADIUS, Client Authentication |
| 370 | Lab เต็มรูปแบบ: 802.1X/RADIUS + Dynamic VLAN + Guest CWA + Rogue AP Detection |

---

## Step 361 — WPA2/WPA3-Enterprise Architecture: บทบาทของ 802.1X Framework และเปรียบเทียบ EAP Method

### ทบทวนจาก Part 20: ทำไม PSK ไม่พอสำหรับ Enterprise จริง

[Part 20 Step 197](part-020-wireless-fundamentals.md) สรุปไว้แล้วว่า WPA2-Personal (PSK) มีจุดอ่อน
สำคัญคือ **ทุกคนใช้ Password เดียวกัน** — Part นี้จะลงรายละเอียดของทางแก้ที่ Enterprise ใช้จริง
คือ **802.1X Port-Based Network Access Control** (IEEE 802.1X-2010) ซึ่งเป็น **Framework**
ไม่ใช่ Protocol เดียว — มันกำหนด "บทบาท" ของอุปกรณ์ 3 ตัวที่ต้องทำงานร่วมกัน

### 802.1X Framework: 3 บทบาทหลัก

```
┌─────────────────┐        ┌──────────────────┐        ┌──────────────────────┐
│   SUPPLICANT      │        │   AUTHENTICATOR    │        │  AUTHENTICATION       │
│  (ซอฟต์แวร์ใน       │        │   (WLC-1 / AP)      │        │  SERVER               │
│   Client Device)  │        │                    │        │  (AAA-SRV — RADIUS)   │
│                    │        │                    │        │                       │
│  - เก็บ Credential   │ EAPOL  │  - เป็น "ยาม" หน้า   │ RADIUS │  - ตรวจสอบ Credential  │
│    (user/pass หรือ  │◄──────►│    ประตูเครือข่าย     │◄──────►│    จริงกับ AD/LDAP/   │
│    Certificate)    │ 802.1X │  - ไม่ตัดสินใจเอง     │ UDP    │    Internal Database  │
│  - รัน EAP method   │        │  - แค่ "ส่งต่อ" EAP   │ 1812   │  - ตอบ Accept/Reject   │
│    ที่ Server รองรับ  │        │    frame ระหว่าง     │        │  - แถม Attribute เช่น  │
│                    │        │    Supplicant กับ    │        │    VLAN, ACL, Timeout  │
│                    │        │    Auth Server       │        │    กลับมาด้วย (Step366)│
└─────────────────┘        └──────────────────────┘        └──────────────────────┘
     Laptop/Mobile              WLC-1 (Catalyst 9800)            AAA-SRV (10.10.99.60)
```

**จุดสำคัญที่สุดที่ต้องเข้าใจ**: **Authenticator (WLC-1) ไม่รู้จัก Username/Password ของใครเลย** —
มันทำหน้าที่เป็นแค่ **ตัวกลาง (Pass-through)** ที่ห่อหุ้ม EAP frame จาก Client (ส่งมาทาง
**EAPOL** — EAP over LAN บน 802.11 Association) แล้วแปลงเป็น RADIUS packet ส่งไปให้ AAA-SRV
ตัดสินใจจริง — นี่คือเหตุผลที่ WLC-1 ไม่ต้อง sync user database กับ Active Directory หรือระบบใดๆ
เลย ทุกอย่างอยู่ที่ AAA-SRV (หรือ Cisco ISE ในระบบจริงขนาดใหญ่)

### EAP (Extensible Authentication Protocol) คืออะไร

**EAP** คือ Protocol กลางที่ 802.1X ใช้ "ขนส่ง" ข้อมูล Authentication — ตัว EAP เองไม่ได้กำหนด
วิธี Authenticate ตายตัว แต่เปิดให้เลือก **EAP Method** ได้หลายแบบ (คำว่า Extensible มาจากจุดนี้)
สามแบบที่ Enterprise ใช้บ่อยที่สุดคือ:

| EAP Method | ต้องมี Client Certificate? | ต้องมี Server Certificate? | Credential ที่ใช้ | ระดับความปลอดภัย | ความยากในการ Deploy |
|---|---|---|---|---|---|
| **EAP-TLS** | ✅ ต้องมี | ✅ ต้องมี | Certificate ทั้งสองฝั่ง (Mutual TLS) | **สูงสุด** — ไม่มี Password ให้ขโมยเลย | **สูงสุด** — ต้องมี PKI/CA ทำงานสมบูรณ์ + แจก Cert ให้ Client ทุกเครื่อง |
| **PEAP-MSCHAPv2** | ❌ ไม่ต้อง | ✅ ต้องมี | Username/Password (มักผูกกับ Active Directory) | สูง — Server Cert สร้าง TLS Tunnel ห่อ Password ไว้ก่อนส่ง | กลาง — ต้องมี Server Cert เท่านั้น ไม่ต้องแจก Client Cert |
| **EAP-FAST** | ❌ ไม่ต้อง (option) | ❌ ไม่ต้อง (ใช้ PAC แทน) | Username/Password + PAC (Protected Access Credential) | กลาง-สูง | **ต่ำสุด** — ไม่ต้องมี PKI เลยถ้าใช้ Auto PAC Provisioning |

```
เปรียบเทียบ Tunnel ที่แต่ละ EAP Method สร้างก่อนส่ง Credential

EAP-TLS:
  Supplicant ──[Client Cert + Server Cert, Mutual TLS]──► Auth Server
  (ไม่มีการส่ง Password เลย ทั้งสองฝั่งพิสูจน์ตัวตนด้วย Certificate)

PEAP-MSCHAPv2:
  Supplicant ──[TLS Tunnel จาก Server Cert]──► ┌─────────────────────┐
                                                  │ MSCHAPv2 (user/pass) │  <- ส่งอยู่ "ใน" Tunnel
                                                  └─────────────────────┘   ที่เข้ารหัสแล้ว

EAP-FAST:
  Supplicant ──[PAC ที่แจกไว้ล่วงหน้า]──► สร้าง Tunnel แบบเบาไม่ต้องมี Cert ──► Credential
```

> **หลักการเลือก EAP Method ในงานจริง**: EAP-TLS ปลอดภัยที่สุดแต่ deploy ยากที่สุด (ต้องมี PKI) —
> เหมาะกับ Managed Device ที่ IT ควบคุมได้เต็มที่ (Corporate Laptop) ส่วน PEAP-MSCHAPv2 คือ
> **มาตรฐานที่ Enterprise ส่วนใหญ่ใช้จริง** เพราะ balance ระหว่างความปลอดภัยกับความง่ายในการ deploy
> (ผูกกับ Active Directory ที่มีอยู่แล้ว) EAP-FAST เป็นของ Cisco proprietary ที่ปัจจุบันถูกแทนที่
> ด้วย PEAP เป็นส่วนใหญ่ แต่ยังพบใน Legacy deployment บางองค์กร

### WPA2-Enterprise vs WPA3-Enterprise — ภาพรวมก่อนลงรายละเอียด Step 364

| คุณสมบัติ | WPA2-Enterprise | WPA3-Enterprise |
|---|---|---|
| Authentication | 802.1X/EAP (เหมือนกัน) | 802.1X/EAP (เหมือนกัน) |
| Key Management | 4-Way Handshake (AES-CCMP) | 4-Way Handshake (AES-GCMP ตัวเลือกที่แข็งแรงกว่า) |
| Protected Management Frame (PMF/802.11w) | Optional (เปิดหรือไม่ก็ได้) | **บังคับ (Mandatory)** — ป้องกัน Deauth Attack |
| 192-bit Mode | ไม่มี | มี — ใช้ Suite B Cryptography สำหรับ Gov/Finance |

---

## Step 362 — RADIUS Deep Dive สำหรับ Wireless: ทำไมต้อง RADIUS ไม่ใช่ TACACS+

### ทบทวนตารางเปรียบเทียบจาก Part 22 — และเจาะจงเหตุผลสำหรับ Wireless

[Part 22 Step 212](part-022-aaa-device-hardening.md) เทียบ TACACS+ กับ RADIUS ไว้ในภาพกว้างแล้วว่า
**"TACACS+ = ดูแลอุปกรณ์, RADIUS = เข้าเครือข่าย"** — Step นี้จะอธิบายเหตุผลเชิงเทคนิคที่ลึกกว่านั้น
ว่าทำไม 802.1X/Wireless **เลือกไม่ได้เลย** นอกจาก RADIUS

| เหตุผล | รายละเอียด |
|---|---|
| **RFC 3579 กำหนด RADIUS ให้ขนส่ง EAP โดยเฉพาะ** | RADIUS มี Attribute `EAP-Message` (Type 79) ที่ออกแบบมาห่อหุ้ม EAP packet ตรงๆ — TACACS+ ไม่มี Attribute แบบนี้ในมาตรฐาน |
| **Supplicant ทุกตัวบนโลก (Windows/macOS/iOS/Android) รองรับ 802.1X ↔ RADIUS เป็นมาตรฐาน** | ไม่มี OS ไหนมี "TACACS+ Supplicant" ในตัว — 802.1X ผูกกับ RADIUS มาตั้งแต่การออกแบบ |
| **RADIUS เป็น UDP แบบ Stateless เหมาะกับ Challenge-Response หลายรอบของ EAP** | EAP handshake (โดยเฉพาะ EAP-TLS/PEAP) ต้องมีการส่ง-ตอบหลายรอบ (Multi-round trip) ก่อนสรุปผล RADIUS ออกแบบมาให้รองรับ pattern นี้ผ่าน Access-Challenge |
| **Vendor-Specific Attribute (VSA) รองรับการส่งค่า Authorization กลับมาพร้อม Accept** | VLAN, ACL, Session-Timeout, QoS ส่งมาพร้อม Access-Accept ได้ในแพ็กเก็ตเดียว (ดู Step 366) |

### RADIUS Packet Flow: Access-Request / Access-Challenge / Access-Accept / Access-Reject

```
Supplicant          WLC-1 (Authenticator/NAS)              AAA-SRV (RADIUS Server, .60)
    │                        │                                        │
    │──EAPOL-Start──────────►│                                        │
    │◄─EAP-Request/Identity──│                                        │
    │──EAP-Response/Identity►│                                        │
    │                        │──Access-Request─────────────────────► │  (ห่อ EAP-Message ไว้ใน
    │                        │  (User-Name, NAS-IP, EAP-Message)     │   Attribute พร้อม Calling-
    │                        │                                        │   Station-Id = Client MAC)
    │                        │◄─────────────────Access-Challenge──────│  (ขอข้อมูลเพิ่ม เช่น TLS
    │◄─EAP-Request (relay)───│                                        │   handshake รอบต่อไป)
    │──EAP-Response (relay)─►│──Access-Request (รอบ 2)───────────────►│
    │                        │        ... (ทำซ้ำหลายรอบสำหรับ          │
    │                        │             EAP-TLS/PEAP handshake) ...│
    │                        │◄──────────────Access-Accept ✅──────────│  (พร้อม Attribute: VLAN,
    │◄─EAP-Success───────────│                                        │   Session-Timeout, ACL)
    │                        │
    │  [4-Way Handshake สร้าง Pairwise/Group Key ต่อจากนี้]              │
    │                        │
```

| Packet Type | ทิศทาง | ความหมาย |
|---|---|---|
| **Access-Request** | NAS (WLC-1) → RADIUS Server | ขออนุญาต พร้อมส่ง Credential/EAP payload |
| **Access-Challenge** | RADIUS Server → NAS | "ยังไม่พอ ขอข้อมูลเพิ่ม" — ใช้บ่อยมากในทุก EAP handshake ที่มีหลายรอบ |
| **Access-Accept** | RADIUS Server → NAS | อนุญาต — แถม Attribute เพิ่มเติม (VLAN, ACL, Timeout) มาด้วยได้ |
| **Access-Reject** | RADIUS Server → NAS | ปฏิเสธ — Credential ผิด หรือ Policy ไม่อนุญาต |

### เปิดใช้งาน RADIUS บน AAA-SRV ตัวเดิมจาก Part 22 (Dual-Role Server)

**AAA-SRV (10.10.99.60)** ที่ [Part 22](part-022-aaa-device-hardening.md) สร้างไว้เป็น TACACS+
Server สำหรับ Device Admin เพียงอย่างเดียว — Part นี้จะเปิด **RADIUS Service เพิ่มบน Server
ตัวเดียวกัน** (Platform เดียวกันสามารถรันทั้งสอง Protocol พร้อมกันได้ เช่น **Cisco ISE** ที่รองรับ
ทั้ง TACACS+ และ RADIUS ในตัว, หรือ FreeRADIUS ที่ต้องรันคู่กับ tac_plus แยก process)

| รายการ | ค่า |
|---|---|
| Hostname | AAA-SRV (ตัวเดิมจาก Part 22) |
| IP Address | 10.10.99.60/24 (VLAN 99 — MGMT) — **ไม่เปลี่ยน** |
| บทบาทเดิม (Part 22) | TACACS+ Server สำหรับ Device Administration (TCP/49) |
| **บทบาทใหม่ (Part 37)** | **RADIUS Server สำหรับ Wireless/802.1X Network Access (UDP/1812-1813)** |
| Shared Secret (RADIUS) | แยกจาก TACACS+ key — ตั้งใหม่เฉพาะ RADIUS |

> **ทำไมใช้ Server ตัวเดียวกันได้**: TACACS+ กับ RADIUS เป็น Protocol คนละ Port คนละ Transport
> (TCP/49 vs UDP/1812-1813) จึงรันคู่กันบน Physical/Virtual Server ตัวเดียวได้สบายๆ — ในองค์กรใหญ่
> จริงมักรวมเป็น **Cisco ISE** ตัวเดียวที่ทำทั้ง Device Admin (TACACS+) และ Network Access
> (RADIUS/802.1X) พร้อม Policy Engine ที่ซับซ้อนกว่านี้มาก (เรียนเต็มรูปแบบใน Part 43/78)

```
CORE-SW1/DIST-SW*  ──TACACS+ (TCP/49)──►┌─────────────────────────┐
    (Device Admin - Part 22)             │        AAA-SRV            │
                                          │      10.10.99.60           │
WLC-1              ──RADIUS (UDP/1812)──►│  (Dual-role: TACACS+ +     │
    (802.1X Wireless - Part 37)          │   RADIUS on same server)   │
                                          └─────────────────────────┘
```

### คอนฟิกฝั่ง WLC-1: ประกาศ RADIUS Server

```
WLC-1# configure terminal

! ============================================
! 1. ประกาศ RADIUS Server (คล้ายกับ tacacs server syntax ใน Part 22)
! ============================================
WLC-1(config)# radius server AAA-SRV-RADIUS
WLC-1(config-radius-server)# address ipv4 10.10.99.60 auth-port 1812 acct-port 1813
WLC-1(config-radius-server)# key CiscoLab-RADIUS-Key2026!
WLC-1(config-radius-server)# timeout 5
WLC-1(config-radius-server)# exit

! ============================================
! 2. สร้าง AAA Server Group (Redundancy pattern เดียวกับ Part 22 Step 214)
! ============================================
WLC-1(config)# aaa group server radius AAA-RADIUS-GRP
WLC-1(config-sg-radius)# server name AAA-SRV-RADIUS
WLC-1(config-sg-radius)# exit

! ============================================
! 3. เปิด AAA บน WLC-1 (ถ้ายังไม่เปิด — จำเป็นสำหรับทั้ง TACACS+ Device Admin และ RADIUS Wireless)
! ============================================
WLC-1(config)# aaa new-model

! ============================================
! 4. Method List สำหรับ 802.1X Authentication และ Network Authorization
! ============================================
WLC-1(config)# aaa authentication dot1x default group AAA-RADIUS-GRP
WLC-1(config)# aaa authorization network default group AAA-RADIUS-GRP
WLC-1(config)# aaa accounting identity default start-stop group AAA-RADIUS-GRP

WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### ตรวจสอบผลลัพธ์

```
WLC-1# show radius server-group AAA-RADIUS-GRP
Server group AAA-RADIUS-GRP:
  Sharecount = 1
  Server (10.10.99.60:1812,1813)

WLC-1# show aaa servers | include AAA-SRV-RADIUS
RADIUS: id 1, priority 1, host 10.10.99.60, auth-port 1812, acct-port 1813
     State: current UP, duration 3600s, previous duration 0s
```

---

## Step 363 — คอนฟิก 802.1X/WPA2-Enterprise WLAN บน Catalyst 9800: สร้าง SSID "ENTERPRISE-SECURE"

### แผนการอัปเกรด: ไม่ลบ SSID เดิม เพิ่ม SSID ใหม่คู่กัน

เพื่อไม่กระทบ Client ที่ยังใช้ PSK (`ENTERPRISE-WIFI` จาก Part 20) อยู่ระหว่างการ Migrate เราจะ
**สร้าง WLAN Profile ใหม่ "ENTERPRISE-SECURE"** ควบคู่กันไปก่อน แล้วค่อย Deprecate SSID เดิม
ทีหลังเมื่อ Client ทุกเครื่อง Migrate มาใช้ 802.1X ได้หมดแล้ว — Pattern นี้เป็นวิธี Migrate ที่ปลอดภัย
ที่สุดในงานจริง (คล้ายกับ Parallel Run ก่อน Cutover)

```
Catalyst 9800 Tag-Based Architecture (ต่อยอดจาก Part 20 Step 195)

┌────────────────────┐   ┌────────────────────┐
│  WLAN Profile #1     │   │  WLAN Profile #2     │
│  ENTERPRISE-WIFI-    │   │  ENTERPRISE-SECURE-  │
│  PROFILE (PSK)       │   │  PROFILE (802.1X)    │  <- ใหม่ใน Part 37
└──────────┬──────────┘   └──────────┬──────────┘
           │                          │
┌──────────▼──────────┐   ┌──────────▼──────────┐
│  Policy Profile #1    │   │  Policy Profile #2    │
│  ENTERPRISE-WIFI-     │   │  ENTERPRISE-SECURE-   │
│  POLICY (VLAN 40)     │   │  POLICY (VLAN 40 +    │  <- aaa-override เปิด (Step 366)
│                       │   │  aaa-override)         │
└──────────┬──────────┘   └──────────┬──────────┘
           │                          │
           └────────────┬────────────┘
                         ▼
              ┌────────────────────┐
              │   Policy Tag         │   <- ใช้ Tag เดิม "ENTERPRISE-TAG" จาก Part 20
              │  "ENTERPRISE-TAG"    │      เพิ่ม mapping ใหม่เข้าไปอีกบรรทัด
              └──────────┬──────────┘
                         │
                  ┌───────┴───────┐
                  ▼                ▼
              AP-1              AP-2   <- Broadcast ทั้ง 2 SSID พร้อมกัน (ไม่ต้องเปลี่ยน Tag ที่ AP)
```

### 1) สร้าง WLAN Profile "ENTERPRISE-SECURE-PROFILE" — WPA2-Enterprise (802.1X)

```
WLC-1# configure terminal

! ============================================
! สร้าง WLAN ใหม่ WLAN ID 2 (ID 1 ถูกใช้ไปแล้วโดย ENTERPRISE-WIFI-PROFILE ใน Part 20)
! ============================================
WLC-1(config)# wlan ENTERPRISE-SECURE-PROFILE 2 ENTERPRISE-SECURE
WLC-1(config-wlan)# security wpa akm dot1x               ! ใช้ 802.1X เป็นวิธี Key Management
WLC-1(config-wlan)# security wpa wpa2                     ! บังคับ WPA2 (WPA3 เพิ่มใน Step 364)
WLC-1(config-wlan)# security ft over-the-ds                ! เปิด Fast Transition สำหรับ Roaming เร็ว
WLC-1(config-wlan)# no security wpa akm dot1x-sha256        ! ปิด SHA256 variant ไว้ก่อน (ใช้ตอน WPA3)
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
```

### 2) สร้าง Policy Profile "ENTERPRISE-SECURE-POLICY" — Map ไปยัง VLAN 40 เดิม

```
WLC-1(config)# wireless profile policy ENTERPRISE-SECURE-POLICY
WLC-1(config-wireless-policy)# vlan 40
WLC-1(config-wireless-policy)# aaa-override                 ! สำคัญ! เปิดไว้ล่วงหน้าเพื่อรองรับ
WLC-1(config-wireless-policy)#                                !   Dynamic VLAN ใน Step 366
WLC-1(config-wireless-policy)# accounting-list AAA-RADIUS-GRP
WLC-1(config-wireless-policy)# no shutdown
WLC-1(config-wireless-policy)# exit
```

### 3) เพิ่ม WLAN ↔ Policy Mapping เข้าไปใน Policy Tag เดิม (ไม่ต้องสร้าง Tag ใหม่)

```
WLC-1(config)# wireless tag policy ENTERPRISE-TAG
WLC-1(config-policy-tag)# wlan ENTERPRISE-SECURE-PROFILE policy ENTERPRISE-SECURE-POLICY
WLC-1(config-policy-tag)# exit
! Policy Tag "ENTERPRISE-TAG" ตอนนี้ Map WLAN 2 ตัวแล้ว: ENTERPRISE-WIFI-PROFILE (เดิม) +
!   ENTERPRISE-SECURE-PROFILE (ใหม่) — เพราะ AP-1/AP-2 ถูกผูก Tag นี้อยู่แล้วจาก Part 20 Step 196
!   จึงไม่ต้องแก้ AP Tag อะไรเลย ทั้งสอง SSID จะปรากฏบน AP-1/AP-2 ทันที

WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### ตรวจสอบผลลัพธ์

```
WLC-1# show wlan summary
Number of WLANs: 2
--------------------------------------------------------------------------------
ID   Profile Name                      SSID                            Status
--------------------------------------------------------------------------------
1    ENTERPRISE-WIFI-PROFILE           ENTERPRISE-WIFI                 UP
2    ENTERPRISE-SECURE-PROFILE         ENTERPRISE-SECURE               UP

WLC-1# show wireless tag policy detailed ENTERPRISE-TAG | include WLAN Profile|Policy Profile
  WLAN Profile Name             : ENTERPRISE-WIFI-PROFILE   Policy Profile Name : ENTERPRISE-WIFI-POLICY
  WLAN Profile Name             : ENTERPRISE-SECURE-PROFILE Policy Profile Name : ENTERPRISE-SECURE-POLICY

WLC-1# show wireless profile policy detailed ENTERPRISE-SECURE-POLICY | include VLAN|AAA Override
  VLAN                                     : 40
  AAA Override                             : Enabled
```

> **ข้อควรระวังเรื่อง WLAN ID**: บน Catalyst 9800 WLAN ID (1-512) ต้อง**ไม่ซ้ำกัน**ทั้งระบบ —
> ควรตั้งมาตรฐานการนับเลข WLAN ID ไว้ล่วงหน้า (เช่น 1-99 = Corporate SSID, 100-199 = Guest SSID)
> เพื่อไม่ให้สับสนเมื่อมี SSID จำนวนมากขึ้นในภาคหลังของหลักสูตร

---

## Step 364 — WPA3-Enterprise และ WPA3-Enterprise 192-bit Mode

### สิ่งใหม่ใน WPA3 เทียบกับ WPA2

| ฟีเจอร์ | WPA2 (เดิม) | WPA3 (ใหม่) |
|---|---|---|
| **-Personal**: Key Exchange | PSK + 4-Way Handshake (เสี่ยง Offline Dictionary Attack) | **SAE (Simultaneous Authentication of Equals)** — ป้องกัน Offline Dictionary Attack ได้เพราะทุก guess ต้อง interact กับ AP สด (Forward Secrecy ด้วย) |
| **-Enterprise**: Authentication | 802.1X/EAP เดิม | 802.1X/EAP เดิม (ไม่เปลี่ยน) แต่เพิ่มการเข้ารหัสที่แข็งแรงขึ้น |
| **PMF (802.11w — Protected Management Frame)** | Optional | **บังคับ (Mandatory)** — เข้ารหัส Management Frame เช่น Deauth/Disassociation ป้องกัน DoS จาก Fake Deauth Attack |
| Encryption Cipher | AES-CCMP-128 | AES-GCMP-256 (แข็งแรงกว่า, เร็วกว่าในบาง Hardware) |
| 192-bit Mode | ไม่มี | มี — สำหรับ Gov/Finance ที่ต้องการมาตรฐาน Suite B |

> **จุดที่มักเข้าใจผิด**: WPA3-Enterprise "ระดับพื้นฐาน" (ไม่ใช่ 192-bit) **ไม่ได้บังคับ**ให้ใช้
> EAP-TLS หรือ Certificate — ยังใช้ PEAP-MSCHAPv2/EAP-FAST ได้ตามปกติ สิ่งที่ WPA3-Enterprise
> บังคับจริงๆ คือ **PMF (802.11w) Mandatory** เท่านั้น — ส่วน 192-bit Mode ต่างหากที่บังคับ
> EAP-TLS กับ Certificate ที่มีความยาว Key ขั้นต่ำตามมาตรฐาน Suite B (ดูด้านล่าง)

### คอนฟิก: อัปเกรด WLAN เดิมเป็น WPA3-Enterprise (มาตรฐาน ไม่ใช่ 192-bit)

```
WLC-1# configure terminal
WLC-1(config)# wlan ENTERPRISE-SECURE-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# security wpa wpa3                    ! เปิด WPA3
WLC-1(config-wlan)# security wpa akm dot1x-sha256          ! WPA3-Enterprise ใช้ SHA256 variant ของ AKM
WLC-1(config-wlan)# no security wpa akm dot1x              ! ปิด plain dot1x AKM แบบ WPA2 เดิม
WLC-1(config-wlan)# security pmf mandatory                 ! บังคับ PMF (จุดที่ WPA3-Enterprise ต้องมี)
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### WPA3-Enterprise 192-bit Mode — Suite B Cryptography สำหรับความปลอดภัยสูงสุด

**192-bit Mode** เป็น Option เสริมของ WPA3-Enterprise ที่บังคับใช้ **Cryptographic Algorithm
Suite ระดับ NSA Suite B** ทั้งหมด — เหมาะกับองค์กรระดับ **รัฐบาล/การเงิน/Defense** ที่ต้องการ
มาตรฐานความปลอดภัยสูงสุด

| ส่วนประกอบ | มาตรฐาน 192-bit Mode |
|---|---|
| Authentication | **EAP-TLS เท่านั้น** (Certificate-based — ดู Step 365) |
| Encryption | AES-256-GCM (ไม่ใช่ 128-bit อีกต่อไป) |
| Key Derivation | HMAC-SHA-384 |
| Digital Signature | ECDSA ด้วย Elliptic Curve **P-384** (หรือ RSA 3072-bit ขั้นต่ำ) |
| ผลกระทบต่อ Certificate ที่ใช้ | Certificate ทุกใบ (Client + Server + CA Chain) ต้องออกด้วย Key Length/Curve ที่ผ่านมาตรฐานนี้ — Certificate เดิมที่ใช้ RSA-2048 **ใช้ไม่ได้** ต้อง Re-issue ใหม่ |

```
WLC-1# configure terminal
WLC-1(config)# wlan ENTERPRISE-SECURE-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# security wpa wpa3
WLC-1(config-wlan)# security wpa wpa3 192-bit-mode           ! เปิด Suite B 192-bit mode
WLC-1(config-wlan)# security wpa akm 802.1x-suiteb192          ! บังคับ AKM แบบ Suite-B 192-bit
WLC-1(config-wlan)# security pmf mandatory                    ! PMF ยังบังคับเสมอ
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
```

> **หมายเหตุสำคัญ**: Syntax ของ 192-bit Mode อาจต่างกันเล็กน้อยตาม IOS-XE Release Version บน
> Catalyst 9800 จริง (ใช้ `?` ตรวจสอบ Keyword ที่รองรับในเวอร์ชันของท่านเสมอก่อนใช้งานจริง) — และ
> ก่อนเปิดโหมดนี้ **ต้องมี PKI ที่ออก Certificate ตามมาตรฐาน Suite B ให้ Client ทุกเครื่องก่อน**
> ไม่เช่นนั้น Client ที่มี Certificate ไม่ผ่านมาตรฐานจะ Authenticate ไม่ผ่านทั้งหมดทันทีที่เปิดโหมดนี้
> — ในทางปฏิบัติ 192-bit Mode ใช้เฉพาะ Use Case เฉพาะทางเท่านั้น องค์กรทั่วไปใช้ WPA2/WPA3-Enterprise
> มาตรฐานก็เพียงพอแล้ว

---

## Step 365 — Certificate-based EAP-TLS เชิงลึก: PKI Trust Chain และบทบาทของ Cisco ISE

### ทำไม EAP-TLS ปลอดภัยที่สุดแต่ Deploy ยากที่สุด

EAP-TLS ไม่มีการส่ง Password ข้ามเครือข่ายเลย — ทั้ง Client และ Server พิสูจน์ตัวตนด้วย
**Certificate** ที่ผ่านการเซ็นจาก **Certificate Authority (CA)** ที่ทั้งสองฝั่ง **เชื่อถือร่วมกัน**
เท่านั้น (Mutual TLS) นี่คือเหตุผลที่ทำให้ Password Spraying, Credential Phishing, และ
Offline Dictionary Attack **ใช้ไม่ได้เลย** กับ EAP-TLS — แต่ก็ต้องแลกกับความซับซ้อนขององค์กรที่
ต้องมี **PKI (Public Key Infrastructure)** ที่ทำงานได้จริงและดูแลตลอดอายุการใช้งาน

### PKI Trust Chain ที่ EAP-TLS ต้องมี

```
                          ┌─────────────────────┐
                          │      Root CA           │   <- Certificate Authority สูงสุด
                          │  (Offline, เก็บปลอดภัย   │      (มักเป็น Internal CA เช่น
                          │   ที่สุด แทบไม่ใช้งานตรง) │       Microsoft AD CS Root)
                          └──────────┬───────────┘
                                     │ เซ็นรับรอง
                          ┌──────────▼───────────┐
                          │   Intermediate CA       │   <- ใช้งานจริงในการออก Cert
                          │  (Online, ออก Cert ให้    │      รายวัน (แยกจาก Root เพื่อ
                          │   Client/Server จริง)    │       ความปลอดภัย — Root ไม่ต้อง
                          └──────────┬───────────┘      ออนไลน์บ่อย)
                    ┌────────────────┴────────────────┐
                    │ เซ็นรับรอง                          │ เซ็นรับรอง
         ┌──────────▼──────────┐            ┌──────────▼──────────┐
         │   Client Certificate   │            │  RADIUS/ISE Server     │
         │  (ติดตั้งบน Laptop/     │            │  Certificate            │
         │   Mobile ของผู้ใช้)     │            │  (ติดตั้งบน AAA-SRV/    │
         └───────────────────────┘            │   Cisco ISE)             │
                                                └──────────────────────┘

ทั้ง 2 ฝั่งต้องมี "Trust Store" ที่เก็บ Root CA (และ Intermediate CA) ไว้ ถึงจะ Verify
Certificate ของอีกฝั่งได้สำเร็จ — ถ้า Client ไม่เชื่อถือ Root CA ที่เซ็น Server Cert
(เช่น Certificate หมดอายุ, หรือ CA ไม่อยู่ใน Trust Store) EAP-TLS จะ Fail ทันที
```

### สิ่งที่ต้องมีก่อน Deploy EAP-TLS จริง

| องค์ประกอบ | รายละเอียด |
|---|---|
| **CA Infrastructure** | Internal CA (Microsoft AD CS ที่พบบ่อยที่สุดในองค์กรที่ใช้ Active Directory, หรือ CA ในตัวของ Cisco ISE) |
| **Server Certificate** | ออกให้กับ RADIUS/ISE Server (AAA-SRV) — ต้อง Renew ก่อนหมดอายุเสมอ (ลืม Renew = Client ทุกเครื่อง Authenticate ไม่ผ่านพร้อมกัน) |
| **Client Certificate Distribution** | แจก Certificate ให้ Endpoint ทุกเครื่อง — ทำได้ผ่าน **SCEP (Simple Certificate Enrollment Protocol)** อัตโนมัติผ่าน MDM (Intune, JAMF) หรือ Group Policy (สำหรับ Windows AD-joined) |
| **Revocation Checking** | CRL (Certificate Revocation List) หรือ OCSP (Online Certificate Status Protocol) — เพื่อเช็คว่า Certificate ที่ถูกขโมย/เครื่องหาย ถูก Revoke ไปแล้วหรือยัง |
| **Lifecycle Management** | ต้องมี Process ต่ออายุ Certificate ก่อนหมดอายุอัตโนมัติ (มักผ่าน MDM/SCEP renewal) — นี่คือส่วนที่องค์กรส่วนใหญ่ "เจ็บ" ที่สุดเมื่อ Scale ใหญ่ขึ้น |

### บทบาทของ Cisco ISE ในการออก Certificate (Preview)

**Cisco ISE (Identity Services Engine)** ซึ่งจะเรียนเต็มรูปแบบใน **Part 43/78** สามารถทำหน้าที่
เป็นทั้ง **RADIUS Server** และ **Internal CA** ในตัวเดียว — รองรับ:

- ออก Certificate ให้ Endpoint ผ่าน **SCEP** โดยอัตโนมัติตอน Client ทำ **BYOD Onboarding**
  (Native Supplicant Provisioning flow — Client เชื่อมต่อ SSID ชั่วคราวก่อน, ยืนยันตัวตน,
  แล้ว ISE ออก Certificate ให้ทันทีแบบ Self-service)
- ทำหน้าที่เป็น **Intermediate CA** ที่รับ Certificate มาจาก External Root CA ขององค์กร (ไม่ต้อง
  เป็น Root CA เอง)
- ผูก Certificate เข้ากับ Policy: เช่น "Certificate ที่ออกจาก Template นี้เท่านั้นที่ผ่าน
  Authorization Policy เข้า SSID ENTERPRISE-SECURE ได้"

> **Part นี้แค่ Preview แนวคิด ISE เท่านั้น** — ใน Lab ของหลักสูตรนี้ AAA-SRV ยังทำหน้าที่แบบ
> Generic RADIUS Server (ไม่มี CA ในตัว) องค์กรที่จะ Deploy EAP-TLS จริงต้องมี PKI แยกหรือใช้
> ISE's Internal CA ซึ่งจะเรียนการติดตั้งและ Config เต็มรูปแบบใน Part 43/78

---

## Step 366 — Dynamic VLAN Assignment ผ่าน RADIUS Attribute (Tunnel-Private-Group-ID)

### ปัญหาที่ Dynamic VLAN แก้: ทุก User ผ่าน SSID เดียวกันแต่ต้องได้ VLAN ต่างกัน

โจทย์จริงที่พบบ่อย: องค์กรมี SSID เดียว (`ENTERPRISE-SECURE`) ให้ทั้ง **พนักงานประจำ** และ
**Contractor** เชื่อมต่อ แต่ต้องการให้ Contractor ถูก**แยกไปอยู่ VLAN Quarantine ต่างหาก**
โดยอัตโนมัติ (จำกัดสิทธิ์เข้าถึง Resource ภายใน) — วิธีทำโดยไม่ต้องสร้าง SSID แยกคือ
**RADIUS ส่ง VLAN กลับมาพร้อม Access-Accept** แทนการกำหนด VLAN ตายตัวที่ Policy Profile

### RADIUS Attribute ที่ใช้ — ต้องมาครบ 3 ตัวพร้อมกัน (RFC 2868)

| Attribute | Type | ค่าที่ต้องใส่ | หมายเหตุ |
|---|---|---|---|
| `Tunnel-Type` | 64 | `13` (VLAN) | บอกว่านี่คือ VLAN assignment |
| `Tunnel-Medium-Type` | 65 | `6` (802 — คือ Ethernet/IEEE 802) | บอกประเภทสื่อกลาง |
| `Tunnel-Private-Group-ID` | 81 | หมายเลข VLAN หรือชื่อ VLAN (string) | **ค่า VLAN จริงที่จะ Assign ให้** |

> **ข้อควรระวัง**: ทั้ง 3 Attribute ต้องมาครบพร้อมกันเสมอ — ถ้ามาแค่ `Tunnel-Private-Group-ID`
> อย่างเดียวโดยไม่มี Tunnel-Type/Tunnel-Medium-Type ครบ WLC จะ**ไม่ยอมรับ**ค่า VLAN ที่ส่งมา
> (Behavior นี้เหมือนกันทั้ง AireOS และ IOS-XE 9800)

### สร้าง VLAN ใหม่สำหรับ Contractor: VLAN 45 (CONTRACTOR-QUARANTINE)

Lab ของหลักสูตรนี้ยังไม่มี VLAN สำหรับ Contractor ใน [IP Address Plan](00-ip-address-plan.md) —
Part นี้จึงเพิ่ม VLAN ใหม่เป็นส่วนขยายเฉพาะสำหรับ Use Case นี้ (Pattern เดียวกับที่ [Part 22]
(part-022-aaa-device-hardening.md) เพิ่ม AAA-SRV เข้ามาโดยไม่ได้อยู่ใน Canonical Table ตั้งแต่แรก)

| VLAN | ชื่อ | Subnet | Default Gateway | หมายเหตุ |
|---|---|---|---|---|
| **45** | **CONTRACTOR-QUARANTINE** | **10.10.45.0/24** | **10.10.45.1** | ใหม่ใน Part 37 — จำกัดสิทธิ์เข้าถึง Internal Resource ด้วย ACL |

```
! คอนฟิกที่ DIST-SW3 (SVI สำหรับ VLAN 45 — pattern เดียวกับ VLAN อื่นจาก Part 10)
DIST-SW3(config)# vlan 45
DIST-SW3(config-vlan)# name CONTRACTOR-QUARANTINE
DIST-SW3(config-vlan)# exit
DIST-SW3(config)# interface Vlan45
DIST-SW3(config-if)# description ** Contractor Quarantine VLAN - Dynamic VLAN Assignment **
DIST-SW3(config-if)# ip address 10.10.45.1 255.255.255.0
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

! ACL จำกัดสิทธิ์ Contractor: อนุญาตแค่ Internet ผ่าน NAT, ปฏิเสธเข้า VLAN Internal ทั้งหมด
DIST-SW3(config)# ip access-list extended CONTRACTOR-RESTRICT
DIST-SW3(config-ext-nacl)# deny ip 10.10.45.0 0.0.0.255 10.10.10.0 0.0.0.255
DIST-SW3(config-ext-nacl)# deny ip 10.10.45.0 0.0.0.255 10.10.30.0 0.0.0.255
DIST-SW3(config-ext-nacl)# deny ip 10.10.45.0 0.0.0.255 10.10.99.0 0.0.0.255
DIST-SW3(config-ext-nacl)# permit ip any any
DIST-SW3(config-ext-nacl)# exit
DIST-SW3(config)# interface Vlan45
DIST-SW3(config-if)# ip access-group CONTRACTOR-RESTRICT in
DIST-SW3(config-if)# exit
```

### ตั้งค่าฝั่ง RADIUS Server (Concept — Attribute ที่ AAA-SRV ต้องตอบกลับสำหรับ Group "Contractor")

```
! ตัวอย่าง Concept การตั้งค่าฝั่ง AAA-SRV (ไม่ผูก Platform เฉพาะ — เหมือนแนวทาง Generic ของ Part 22)
! User Group: "Contractor" -> Access-Accept พร้อม Attribute:
   Tunnel-Type            = 13 (VLAN)
   Tunnel-Medium-Type     = 6  (802)
   Tunnel-Private-Group-ID = 45
```

### ฝั่ง WLC-1: Policy Profile ต้องเปิด `aaa-override` (ทำไปแล้วใน Step 363)

WLC-1 จะ**ไม่ยอมรับ**ค่า VLAN ที่ RADIUS ส่งมาเลย ถ้า Policy Profile ไม่เปิด `aaa-override` —
เพราะ Default Behavior คือ "เชื่อ VLAN ที่ Admin ตั้งไว้ Local เท่านั้น ไม่ยอมให้ RADIUS มา Override"

```
! ยืนยันว่า aaa-override เปิดอยู่ (ทำไปแล้วใน Step 363 — ตรวจสอบซ้ำที่นี่)
WLC-1# show wireless profile policy detailed ENTERPRISE-SECURE-POLICY | include AAA Override|VLAN
  VLAN                                     : 40      <- ค่า Default (fallback) ถ้า RADIUS ไม่ส่ง VLAN มา
  AAA Override                             : Enabled
```

### ผลลัพธ์: Contractor Login → ถูกวาง VLAN 45 อัตโนมัติ, พนักงานปกติ → ยังได้ VLAN 40 ตามเดิม

```
Flow การทำงานเต็ม

Contractor Laptop ──802.1X/EAP──► WLC-1 ──RADIUS Access-Request──► AAA-SRV
                                                                       │
                                              ตรวจสอบ User Group = "Contractor"
                                                                       │
                                        Access-Accept + Tunnel-Private-Group-ID=45
                                                                       │
                    WLC-1 ◄───────────────────────────────────────────┘
                       │
                 aaa-override = Enabled → ใช้ VLAN 45 (ไม่ใช้ VLAN 40 ที่ Policy Profile ตั้งไว้)
                       │
                       ▼
            Client ได้ IP จาก DHCP Pool VLAN 45 (10.10.45.0/24)
```

> **เทคนิคจริงในองค์กร**: การจำแนก User Group (Employee vs Contractor) ทำที่ **RADIUS Server**
> ผ่านการ Query Active Directory Group Membership (เช่น AD Group "Contractors") — WLC-1
> ไม่จำเป็นต้องรู้เรื่อง Group เลย มันแค่เชื่อ Attribute ที่ RADIUS ส่งมาเท่านั้น (Separation of
> Concerns ระหว่าง Policy Decision ที่ Server กับ Policy Enforcement ที่ WLC)

---

## Step 367 — Guest Wireless Design: Central Web Authentication (CWA) และ Guest Anchor WLC

### ทำไม Guest Wireless ต้องออกแบบต่างจาก Corporate SSID

Guest ไม่มี Account ในระบบ AD/RADIUS ขององค์กร (หรือมีแค่ชั่วคราว) — จึงใช้ 802.1X ไม่ได้โดยตรง
วิธีมาตรฐานคือ **Central Web Authentication (CWA)**: Client เชื่อมต่อ SSID แบบ Open ก่อน แล้วถูก
"ดัก" (Redirect) ไปหน้า Captive Portal เพื่อ Login/ยอมรับเงื่อนไข ก่อนได้สิทธิ์ใช้งานอินเทอร์เน็ตจริง

### CWA Flow แบบละเอียด

```
Guest Device                    WLC-1                      RADIUS/ISE (Portal)
    │                              │                                │
    │──Associate SSID "GUEST-WIFI"►│                                │
    │◄───────────Open Auth OK──────│  (ไม่มี Password, แค่ Associate)  │
    │                              │──Access-Request (MAC Filter)──►│
    │                              │◄──Access-Accept + redirect-url─│  (RADIUS ส่ง URL Portal
    │                              │     + Access-List "REDIRECT"    │   กลับมาพร้อม ACL ที่บีบ
    │                              │                                │   ให้ไปได้แค่ Portal เท่านั้น)
    │──HTTP Request (เว็บไซต์ใดๆ)──►│                                │
    │◄─────HTTP 302 Redirect───────│  (WLC ดัก HTTP แล้ว Redirect     │
    │                              │   ไปหน้า Login Guest Portal)     │
    │──เปิดหน้า Portal, กรอกข้อมูล──►│───────────────────────────────►│  (Login/ยอมรับเงื่อนไข)
    │                              │◄────RADIUS CoA (Change of──────│  (Portal สั่ง WLC "ปลด ACL
    │                              │      Authorization): Access-   │   จำกัดออก ให้สิทธิ์เต็ม")
    │                              │      Accept ใหม่, ACL ใหม่       │
    │◄────Internet Access เต็มรูปแบบ─│                                │
```

**จุดสำคัญ**: CWA ใช้ **RADIUS CoA (Change of Authorization — RFC 5176)** เพื่อ "อัปเดต" สิทธิ์
ของ Client ที่ Associate อยู่แล้วโดย**ไม่ต้อง Disconnect/Reconnect ใหม่** — Server (มักเป็น
Cisco ISE ในการ Deploy จริง) เป็นผู้ส่ง CoA Request มาสั่ง WLC หลังจาก Guest ผ่านหน้า Portal แล้ว

### สร้าง Guest WLAN บน WLC-1 (ระดับ WLC เท่านั้น — Portal เต็มรูปแบบต้องพึ่ง ISE ในระบบจริง)

```
WLC-1# configure terminal

WLC-1(config)# wlan GUEST-WIFI-PROFILE 3 GUEST-WIFI
WLC-1(config-wlan)# no security wpa                        ! ปิด WPA ทั้งหมด — ใช้ Open + MAC Filter
WLC-1(config-wlan)# security web-auth                       ! เปิด Web Authentication
WLC-1(config-wlan)# security web-auth authentication-list default
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit

WLC-1(config)# wireless profile policy GUEST-WIFI-POLICY
WLC-1(config-wireless-policy)# vlan 50                       ! VLAN ใหม่สำหรับ Guest (ดูตารางด้านล่าง)
WLC-1(config-wireless-policy)# no shutdown
WLC-1(config-wireless-policy)# exit

WLC-1(config)# wireless tag policy ENTERPRISE-TAG
WLC-1(config-policy-tag)# wlan GUEST-WIFI-PROFILE policy GUEST-WIFI-POLICY
WLC-1(config-policy-tag)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

| VLAN | ชื่อ | Subnet | หมายเหตุ |
|---|---|---|---|
| **50** | **GUEST-DMZ** | **10.10.50.0/24** | ใหม่ใน Part 37 — Guest Traffic ทั้งหมดควรออกทาง DMZ ไม่ผ่าน Internal Network เลย |

### Guest Anchor WLC — แนวคิดสำหรับแยก Guest Traffic ออกจาก Internal Network โดยสมบูรณ์

ใน Enterprise ขนาดใหญ่ที่มีหลาย WLC วิธีที่ปลอดภัยที่สุดสำหรับ Guest คือ **Anchor WLC**:
WLC ตัวหนึ่งอยู่ที่ **DMZ** โดยเฉพาะ (แยก Physical/Network segment จาก Internal WLC ทั้งหมด)
ทำหน้าที่เป็น "จุดปล่อย" Guest Traffic ออกอินเทอร์เน็ตเท่านั้น

```
Local WLC (WLC-1, Internal Network)          Guest Anchor WLC (DMZ)
┌────────────────────────┐                 ┌─────────────────────────┐
│  AP-1/AP-2 broadcast     │                 │  ไม่มี AP ต่อตรง            │
│  SSID "GUEST-WIFI"       │                 │  รับ Traffic ผ่าน           │
│                          │  EoIP Mobility  │  Mobility Tunnel เท่านั้น  │
│  Guest Client Associate  │═════Tunnel═════►│                          │
│  ที่ AP-1 (Local)         │  (คล้าย L3       │  Terminate Traffic ที่นี่   │
│                          │   Roaming        │  แล้วปล่อยออก Internet     │
│                          │   Part 20 198)   │  โดยตรงจาก DMZ             │
└────────────────────────┘                 └─────────────────────────┘

ผลลัพธ์: Guest Traffic "ไม่แตะ" Internal Network ของ WLC-1 เลยแม้แต่ Packet เดียว —
   Client Associate ที่ AP Local แต่ Data Plane ทั้งหมดถูก Tunnel ไปที่ DMZ ก่อนออก Internet
```

> **สถานะใน Lab ของหลักสูตรนี้**: เช่นเดียวกับ Mobility Group ใน [Part 20 Step 198]
> (part-020-wireless-fundamentals.md) — Lab นี้มี WLC-1 เพียงตัวเดียว จึงยังไม่มี Anchor WLC
> จริงให้ Config Mobility Tunnel ได้ ส่วน CWA ระดับนี้ทำได้แค่ Local Web-Auth เบื้องต้นบน WLC-1
> เอง — การเพิ่ม WLC ตัวที่สองเข้า DMZ พร้อม Mobility Anchor Config เต็มรูปแบบ (`wireless
> mobility group member ... anchor`) จะกลับมาเรียนเมื่อ Lab ขยายเป็น Multi-Site ใน Part 39/44

### Rate-Limiting / Bandwidth Contract สำหรับ Guest SSID

Guest ไม่ควรแบ่ง Bandwidth เท่ากับพนักงาน — ใช้ QoS Policy จำกัด Rate ต่อ Client:

```
! สร้าง Policy Map จำกัด Bandwidth ต่อ Client บน Guest SSID (Per-Client Rate Limit)
WLC-1# configure terminal
WLC-1(config)# wireless profile policy GUEST-WIFI-POLICY
WLC-1(config-wireless-policy)# service-policy input GUEST-RATE-LIMIT-IN
WLC-1(config-wireless-policy)# service-policy output GUEST-RATE-LIMIT-OUT
WLC-1(config-wireless-policy)# exit
```

| ทิศทาง | Rate Limit แนะนำ (ตัวอย่าง) | เหตุผล |
|---|---|---|
| Upstream (Client → AP) | 2 Mbps ต่อ Client | Guest มักแค่ Browse/Video Call — ไม่ต้องการ Upload มาก |
| Downstream (AP → Client) | 5 Mbps ต่อ Client | เพียงพอสำหรับ Web Browsing/Video Streaming ระดับ SD |

---

## Step 368 — Wireless Intrusion Prevention (wIPS) พื้นฐาน: Rogue AP Detection/Classification/Containment

### Rogue AP คืออะไร และทำไมอันตราย

**Rogue AP** คือ Access Point ที่**ไม่ได้รับอนุญาต**จาก IT ให้เข้าร่วมเครือข่าย — อาจเป็น:
- พนักงานเอา Home Router มาต่อเข้า Switch Port เอง (ตั้งใจดี แต่เปิดช่องโหว่)
- Attacker วาง AP ปลอมที่ SSID เหมือนองค์กร (Evil Twin) เพื่อดัก Credential ของ Client
- AP จากองค์กรข้างเคียงที่สัญญาณล้นเข้ามา (ไม่อันตราย แต่ต้อง Classify แยกออก)

### RRM ทำหน้าที่ตรวจจับ Rogue ผ่าน Off-Channel Scanning

AP ทุกตัว (แม้กำลัง Serve Client ปกติอยู่) จะสละเวลาสั้นๆ เป็นระยะ **สลับ Channel ไปฟัง (Scan)**
Channel อื่นๆ ที่ตัวเองไม่ได้ให้บริการอยู่ — เพื่อฟังว่ามี Beacon Frame จาก AP ที่ไม่รู้จักหรือไม่
(WLC-1 จะเทียบ MAC/BSSID ที่ตรวจพบกับรายการ AP ที่ตนเองบริหารอยู่ทั้งหมด)

### Rogue Classification — จัดกลุ่มความเสี่ยงอัตโนมัติ

| Classification | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Friendly** | Rogue ที่ Admin ยืนยันว่าไม่เป็นอันตราย (Manual Ack หรือ Auto ถ้าตรวจพบว่าอยู่ใน Wired Network เดียวกันแบบที่รู้จัก) | AP ของทีมอื่นในตึกเดียวกันที่รู้จักกันแล้ว |
| **Malicious** | Rogue ที่ตรวจพบว่าเชื่อมต่อเข้ากับ **Wired Network ภายในองค์กรเดียวกัน** (Rogue on Wire) หรือมี SSID เลียนแบบ (Honeypot/Evil Twin) | AP ปลอมที่ผู้บุกรุกเสียบเข้า Switch Port ในสำนักงาน |
| **Unclassified** | ตรวจพบทาง RF แต่ยังไม่รู้ว่าเชื่อมกับ Wired Network ขององค์กรหรือไม่ | AP จากร้านกาแฟข้างตึกที่สัญญาณล้นเข้ามา |
| **Ad-hoc** | Client ที่เปิด Wireless Hotspot/Ad-hoc Network เอง (ไม่ใช่ AP จริง) | Laptop ที่เปิด Mobile Hotspot แชร์เน็ตให้เพื่อน |

### Containment — การ "ปิด" Rogue AP ทาง RF (ต้องระมัดระวังทางกฎหมาย)

**Containment** คือการสั่งให้ AP ที่ควบคุมอยู่ส่ง **Deauthentication Frame ปลอม** ไปยัง Client
ที่เชื่อมต่อกับ Rogue AP นั้น เพื่อบังคับให้หลุดการเชื่อมต่อซ้ำๆ จน Client เลิกพยายามเชื่อมกับ
Rogue ตัวนั้น

> **คำเตือนสำคัญ**: การทำ Containment กับ AP ที่ไม่ได้เป็นของตัวเองในทางกฎหมาย (เช่น AP ของร้าน
> ข้างเคียงที่ Classify เป็น Unclassified แต่ไม่ใช่ Malicious จริง) **อาจผิดกฎหมายโทรคมนาคม**ใน
> หลายประเทศ (รบกวนการสื่อสารของผู้อื่นโดยไม่มีอำนาจ) — ควร Containment เฉพาะที่ยืนยันแล้วว่าเป็น
> **Malicious (Rogue on Wire)** เท่านั้น และควรมี Policy/กระบวนการอนุมัติชัดเจนก่อนใช้งานจริง

### คอนฟิกเปิด Rogue Detection และดูผลลัพธ์

```
WLC-1# configure terminal
WLC-1(config)# wireless wps rogue detection                 ! เปิด Rogue Detection ทั้งระบบ (มัก Default อยู่แล้ว)
WLC-1(config)# wireless wps rogue ap notify-min-rssi -70      ! แจ้งเตือนเฉพาะ Rogue ที่สัญญาณแรงกว่า -70dBm
WLC-1(config)# end

! ดูสรุป Rogue AP ที่ตรวจพบทั้งหมด
WLC-1# show wireless wps rogue ap summary

Rogue AP Summary
=================
Detail Report for Rogue AP:
MAC Address       Class            State           # APs   # Clients   RSSI    Last Seen
--------------------------------------------------------------------------------------------
0011.2233.4455    Malicious        Contained       1        2           -55     2026-09-26 10:15:22
aabb.ccee.7788    Unclassified     Alert           1        0           -78     2026-09-26 10:14:05

! ดูรายละเอียด Rogue ตัวเดียว
WLC-1# show wireless wps rogue ap detailed 0011.2233.4455
Rogue BSSID              : 0011.2233.4455
Classification           : Malicious
Detected on AP           : AP-1
On Wired Network         : Yes  (พบ MAC เดียวกันบน Switch CAM Table — ยืนยันว่าเป็น Rogue on Wire)
State                    : Contained
```

---

## Step 369 — Verification & Troubleshooting: 802.1X, RADIUS, Client Authentication

### คำสั่งหลักตรวจสอบ Client ที่ Authenticate ผ่าน 802.1X

```
! ดู Client Summary พร้อม WLAN/State (คำสั่งพื้นฐานจาก Part 20)
WLC-1# show wireless client summary

! ดูรายละเอียดเต็มของ Client ตัวเดียว — จุดสำคัญที่สุดสำหรับ Troubleshoot 802.1X
WLC-1# show wireless client mac-address aabb.ccdd.2222 detail
```

### ตัวอย่าง Output จริงที่ต้องอ่านให้เป็น (แสดง EAP Method และผล RADIUS)

```
WLC-1# show wireless client mac-address aabb.ccdd.2222 detail
Client MAC Address                          : aabb.ccdd.2222
Client IPv4 Address                         : 10.10.40.115
AP Name                                     : AP-1
Policy Profile                              : ENTERPRISE-SECURE-POLICY
Wireless LAN Id                             : 2
Wireless LAN Name                           : ENTERPRISE-SECURE
Security Policy Completed                   : Yes
EAP Type                                    : PEAP                     <- EAP Method ที่ใช้จริง
Authentication Algorithm                    : Open System
Policy Manager State                        : RUN
Client State                                : Associated
Re-Authentication Timeout                   : 1800
VLAN                                        : 40
Session Timeout                             : 1800
Authentication Server                       : 10.10.99.60             <- ยืนยันว่ายิงไปหา AAA-SRV ตัวถูก
```

### `debug client mac-address` — ดู Flow แบบ Real-time ตอน Client กำลัง Authenticate

```
WLC-1# debug client mac-address aabb.ccdd.2222

! ตัวอย่างสิ่งที่จะเห็น (ย่อ) ตอน Client เชื่อมต่อสำเร็จ
*Sep 26 10:20:01.001: [client-orch-state] aabb.ccdd.2222 Association received
*Sep 26 10:20:01.045: [dot1x] aabb.ccdd.2222 Sending EAP-Request/Identity
*Sep 26 10:20:01.120: [dot1x] aabb.ccdd.2222 Received EAP-Response, forwarding to RADIUS
*Sep 26 10:20:01.310: [aaa] aabb.ccdd.2222 RADIUS Access-Challenge received (round 2)
*Sep 26 10:20:01.980: [aaa] aabb.ccdd.2222 RADIUS Access-Accept received, VLAN=40
*Sep 26 10:20:02.010: [client-orch-state] aabb.ccdd.2222 Moving to RUN state
```

### `test aaa group radius` — ทดสอบว่า RADIUS Server ตอบสนองถูกต้องโดยไม่ต้องรอ Client จริง

```
! ทดสอบ Username/Password ตรงกับ RADIUS Server กลุ่ม AAA-RADIUS-GRP
WLC-1# test aaa group AAA-RADIUS-GRP testuser Test@Pass2026! legacy

Attempting authentication test to server-group AAA-RADIUS-GRP using radius
User was successfully authenticated.
```

| ผลลัพธ์ | ความหมาย | สาเหตุที่เป็นไปได้ |
|---|---|---|
| `User was successfully authenticated` | RADIUS Server ตอบ Accept ปกติ — Server/Network เชื่อมกันได้ดี | - |
| `User rejected` | RADIUS Server ตอบ Reject — Credential ผิดจริง หรือ Policy ไม่อนุญาต | ตรวจ Username/Password และ Policy ฝั่ง Server |
| `ERROR: Timed out waiting for server` | WLC-1 คุยกับ AAA-SRV ไม่ได้เลย | ตรวจ `show radius server-group`, Shared Secret ตรงกันไหม, ACL/Firewall block UDP 1812 หรือไม่ |

### อาการที่พบบ่อยที่สุดจาก Supplicant Misconfiguration

| อาการ | สาเหตุที่พบบ่อยที่สุด | วิธีแก้ |
|---|---|---|
| Client แสดง "Authentication Failed" ทันทีที่พิมพ์ Password | Client เลือก EAP Method ผิด (เช่น เลือก EAP-TLS แต่ Server รอ PEAP) | ตรวจ Network Profile ของ Client ให้ตรงกับ EAP Method ที่ WLAN Config ไว้ |
| Client ค้างที่ "Validating identity..." นานผิดปกติแล้ว Fail | Client **ไม่เชื่อถือ (Trust) Server Certificate** ของ RADIUS/ISE (พบบ่อยที่สุดกับ PEAP) | ติดตั้ง Root CA Certificate ที่เซ็น Server Cert ลงใน Trust Store ของ Client ก่อน หรือปิด "Validate server certificate" ชั่วคราวเพื่อ Test (ไม่ควรทำใน Production) |
| Username/Password ถูกแต่ยัง Reject | รูปแบบ Username ผิด (ลืม Domain Prefix เช่น `DOMAIN\username` หรือ UPN `user@domain.com`) | ตรวจ Format Username ที่ RADIUS Server คาดหวังให้ตรงกับ AD |
| Machine เชื่อมได้แต่ User Login แล้วหลุด | Policy กำหนด **Machine Authentication** อย่างเดียว ไม่รองรับ User Authentication ต่อจากนั้น (หรือ Client ปิด "Authenticate as computer when information is available") | เปิด Both Machine + User Authentication ที่ Client Supplicant Setting |
| RADIUS Reject ทุกครั้งแม้ Password ถูก 100% | เวลา (Clock) ของ AAA-SRV กับ WLC-1 ไม่ตรงกัน ทำให้ Certificate/Timestamp Validation Fail (สำหรับ EAP-TLS/PEAP) | ตรวจ NTP sync ทุกอุปกรณ์ (ทบทวน [Part 17](part-017-ntp-syslog-snmp.md)) |

---

## Step 370 — Lab เต็มรูปแบบ: 802.1X/RADIUS + Dynamic VLAN + Guest CWA + Rogue AP Detection

Lab นี้รวบรวมทุก Step ของ Part 37 เข้าเป็น Configuration เดียวที่สมบูรณ์ พร้อม Verification
เต็มรูปแบบ — ต่อยอดจาก Lab ของ [Part 20](part-020-wireless-fundamentals.md) และ [Part 22]
(part-022-aaa-device-hardening.md) โดยไม่ลบ Config เดิม

### สรุป Address/Device เพิ่มเติมที่ใช้ใน Part นี้

| รายการ | ค่า |
|---|---|
| AAA-SRV (RADIUS role ใหม่) | 10.10.99.60 — UDP 1812 (Auth) / 1813 (Accounting) |
| WLAN ใหม่ #1 | `ENTERPRISE-SECURE` (WLAN ID 2) — WPA2/WPA3-Enterprise 802.1X → VLAN 40 (default) หรือ VLAN 45 (Contractor, ผ่าน Dynamic VLAN) |
| WLAN ใหม่ #2 | `GUEST-WIFI` (WLAN ID 3) — Web-Auth (CWA) → VLAN 50 (GUEST-DMZ) |
| VLAN ใหม่ #1 | VLAN 45 — CONTRACTOR-QUARANTINE, 10.10.45.0/24 |
| VLAN ใหม่ #2 | VLAN 50 — GUEST-DMZ, 10.10.50.0/24 |

### 1) Full Running-Config บน WLC-1 (ส่วนที่เพิ่มจาก Part 37 — ต่อยอด Part 20)

```
! ============================================================
!  WIRELESS SECURITY ADVANCED — FULL TEMPLATE (WLC-1)
!  ต่อยอดจาก Part 20 (WLC พื้นฐาน) และ Part 22 (AAA-SRV) — ห้ามลบ config เดิม
! ============================================================
configure terminal

! --- RADIUS Server & Group (Step 362) ---
radius server AAA-SRV-RADIUS
 address ipv4 10.10.99.60 auth-port 1812 acct-port 1813
 key CiscoLab-RADIUS-Key2026!
 timeout 5
 exit
!
aaa group server radius AAA-RADIUS-GRP
 server name AAA-SRV-RADIUS
 exit

! --- เปิด AAA และ Method List สำหรับ 802.1X (Step 362) ---
aaa new-model
aaa authentication dot1x default group AAA-RADIUS-GRP
aaa authorization network default group AAA-RADIUS-GRP
aaa accounting identity default start-stop group AAA-RADIUS-GRP

! --- WLAN #2: ENTERPRISE-SECURE — WPA2/WPA3-Enterprise 802.1X (Step 363-364) ---
wlan ENTERPRISE-SECURE-PROFILE 2 ENTERPRISE-SECURE
 security wpa akm dot1x
 security wpa wpa2
 security wpa wpa3
 security wpa akm dot1x-sha256
 security ft over-the-ds
 security pmf mandatory
 no shutdown
 exit
!
wireless profile policy ENTERPRISE-SECURE-POLICY
 vlan 40
 aaa-override
 accounting-list AAA-RADIUS-GRP
 no shutdown
 exit

! --- WLAN #3: GUEST-WIFI — Central Web Authentication (Step 367) ---
wlan GUEST-WIFI-PROFILE 3 GUEST-WIFI
 no security wpa
 security web-auth
 security web-auth authentication-list default
 no shutdown
 exit
!
wireless profile policy GUEST-WIFI-POLICY
 vlan 50
 service-policy input GUEST-RATE-LIMIT-IN
 service-policy output GUEST-RATE-LIMIT-OUT
 no shutdown
 exit

! --- Map ทั้ง 2 WLAN ใหม่เข้า Policy Tag เดิม (Step 363, 367) ---
wireless tag policy ENTERPRISE-TAG
 wlan ENTERPRISE-SECURE-PROFILE policy ENTERPRISE-SECURE-POLICY
 wlan GUEST-WIFI-PROFILE policy GUEST-WIFI-POLICY
 exit

! --- Rogue AP Detection (Step 368) ---
wireless wps rogue detection
wireless wps rogue ap notify-min-rssi -70

end
copy running-config startup-config
```

### 2) VLAN 45 (Contractor) และ VLAN 50 (Guest) บน DIST-SW3

```
DIST-SW3# configure terminal

vlan 45
 name CONTRACTOR-QUARANTINE
 exit
interface Vlan45
 description ** Contractor Quarantine — Dynamic VLAN Assignment via RADIUS **
 ip address 10.10.45.1 255.255.255.0
 no shutdown
 exit

vlan 50
 name GUEST-DMZ
 exit
interface Vlan50
 description ** Guest Wireless DMZ — CWA **
 ip address 10.10.50.1 255.255.255.0
 no shutdown
 exit

! DHCP Pool สำหรับทั้งสอง VLAN
ip dhcp pool CONTRACTOR-VLAN45
 network 10.10.45.0 255.255.255.0
 default-router 10.10.45.1
 exit
ip dhcp pool GUEST-VLAN50
 network 10.10.50.0 255.255.255.0
 default-router 10.10.50.1
 exit

end
copy running-config startup-config
```

### 3) Verification ครบ 3 จุดสำคัญ

**(A) Client Authentication สำเร็จ — Employee ปกติ (802.1X, VLAN 40)**

```
WLC-1# show wireless client mac-address aabb.ccdd.2222 detail | include EAP Type|VLAN|State|Authentication Server
EAP Type                                    : PEAP
VLAN                                        : 40
Client State                                : Associated
Authentication Server                       : 10.10.99.60

! Syslog ยืนยัน Authentication สำเร็จ
*Sep 26 10:20:02.010: %DOT1X-5-SUCCESS: Authentication successful for client aabb.ccdd.2222
```

**(B) Contractor ถูก Dynamic VLAN ไปยัง VLAN 45 อัตโนมัติ**

```
WLC-1# show wireless client mac-address ccdd.eeff.3333 detail | include VLAN|AAA Override|State
AAA Override                                : Applied  (Tunnel-Private-Group-ID = 45)
VLAN                                        : 45
Client State                                : Associated

! ตรวจสอบจาก DHCP Binding ยืนยันว่า Client ได้ IP ใน Subnet ที่ถูกต้อง
DIST-SW3# show ip dhcp binding | include 10.10.45
10.10.45.101    ccdd.eeff.3333    Sep 26 2026 10:22 AM    Automatic
```

**(C) Guest CWA และ Rogue AP Detection Demo**

```
! Guest Client ผ่านหน้า Portal สำเร็จ (สถานะเปลี่ยนจาก WEBAUTH_REQD ไป RUN)
WLC-1# show wireless client mac-address 1122.3344.5566 detail | include Policy Manager State|VLAN
Policy Manager State                        : RUN
VLAN                                        : 50

! Rogue AP ที่ตรวจพบระหว่างการทดสอบ (จำลอง Laptop เปิด Mobile Hotspot ชื่อคล้าย SSID องค์กร)
WLC-1# show wireless wps rogue ap summary
Rogue AP Summary
=================
MAC Address       Class            State           # APs   # Clients   RSSI    Last Seen
--------------------------------------------------------------------------------------------
7788.99aa.bbcc     Unclassified     Alert           1        1           -65     2026-09-26 10:25:40
```

Lab นี้ยืนยันครบทุกจุดที่ต้องการ: **802.1X/RADIUS Authentication ทำงานผ่าน AAA-SRV จริง**,
**Dynamic VLAN Assignment วาง Contractor ไปยัง VLAN 45 อัตโนมัติโดยไม่ต้องสร้าง SSID แยก**,
**Guest CWA ทำงานผ่าน Captive Portal และวาง Guest ไปยัง VLAN 50 (DMZ)**, และ **wIPS ตรวจพบ
Rogue AP ที่ไม่รู้จักได้ทันที** ✅

---

## แบบฝึกหัดทวนความเข้าใจ Part 37

1. อธิบายบทบาทของ Supplicant, Authenticator, และ Authentication Server ใน 802.1X Framework —
   แต่ละตัวทำหน้าที่อะไร และตัวไหน "ตัดสินใจ" จริงว่า Client จะได้เข้าเครือข่ายหรือไม่?
2. เพราะเหตุใด Wireless 802.1X จึงต้องใช้ RADIUS เท่านั้น ใช้ TACACS+ แทนไม่ได้ — อ้างอิงเหตุผล
   เชิงเทคนิคของ Protocol
3. RADIUS Attribute ใดบ้าง (ครบ 3 ตัว) ที่ต้องส่งมาพร้อมกันเพื่อทำ Dynamic VLAN Assignment
   และ Policy Profile บน WLC ต้องเปิด Option ใดก่อนจึงจะยอมรับค่าที่ RADIUS ส่งมา?
4. อธิบายความแตกต่างระหว่าง WPA2-Enterprise กับ WPA3-Enterprise — ฟีเจอร์ใดที่ WPA3-Enterprise
   บังคับใช้ (Mandatory) ที่ WPA2-Enterprise เป็นแค่ Option?
5. ทำไม EAP-TLS จึงถือว่าปลอดภัยที่สุดในบรรดา EAP Method แต่ Deploy ยากที่สุด — ต้องมีองค์ประกอบ
   อะไรบ้างก่อนใช้งานจริงได้?

**เฉลย:**

1. **Supplicant** (ซอฟต์แวร์ใน Client) เก็บ Credential และรัน EAP Method; **Authenticator**
   (WLC-1/AP) ทำหน้าที่เป็นตัวกลางส่งต่อ EAP Frame ระหว่าง Supplicant กับ Server เท่านั้น
   **ไม่ได้ตัดสินใจเอง**; **Authentication Server** (AAA-SRV/RADIUS) คือตัวที่**ตัดสินใจจริง**
   ว่า Accept หรือ Reject โดยตรวจสอบ Credential กับ Database ที่มี
2. เพราะ RADIUS มี Attribute `EAP-Message` (RFC 3579) ที่ออกแบบมาห่อหุ้ม EAP Packet โดยเฉพาะ,
   Supplicant ทุก OS รองรับ 802.1X ↔ RADIUS เป็นมาตรฐานสากล, และ RADIUS ออกแบบมารองรับ
   Challenge-Response หลายรอบที่ EAP Handshake ต้องการ — TACACS+ ไม่มี Attribute หรือ
   Mechanism แบบนี้ในมาตรฐาน
3. ต้องส่งครบ 3 ตัว: `Tunnel-Type` (64) = 13 (VLAN), `Tunnel-Medium-Type` (65) = 6 (802),
   และ `Tunnel-Private-Group-ID` (81) = หมายเลข/ชื่อ VLAN ที่ต้องการ Assign — และ Policy Profile
   บน WLC ต้องเปิด `aaa-override` ก่อน ไม่เช่นนั้นจะไม่ยอมรับค่า VLAN ที่ RADIUS ส่งมา
4. WPA3-Enterprise บังคับ **PMF (Protected Management Frame / 802.11w) เป็น Mandatory** ซึ่ง
   WPA2-Enterprise เป็นแค่ Option เปิด/ปิดได้ — ส่วน 802.1X/EAP Authentication ยังเหมือนเดิมทั้ง
   สองแบบ (WPA3 เพิ่ม AES-GCMP-256 และ 192-bit Mode เป็น Option เสริมสำหรับความปลอดภัยสูงสุด)
5. EAP-TLS ปลอดภัยที่สุดเพราะไม่มีการส่ง Password ข้ามเครือข่ายเลย — ทั้ง Client และ Server
   พิสูจน์ตัวตนด้วย Certificate (Mutual TLS) แต่ต้องมี **PKI ที่ทำงานได้จริง**: CA Infrastructure,
   Server Certificate บน RADIUS, การแจก Client Certificate ให้ทุก Endpoint (ผ่าน SCEP/MDM),
   ระบบ Revocation Checking (CRL/OCSP), และ Process การ Renew Certificate ก่อนหมดอายุ

---

## สรุป Part 37

Part นี้ยกระดับความปลอดภัย Wireless จาก **WPA2-Personal (PSK)** ที่เป็น Baseline ใน [Part 20]
(part-020-wireless-fundamentals.md) ไปสู่ **WPA2/WPA3-Enterprise เต็มรูปแบบ** ด้วย 802.1X
Framework และ RADIUS — โดยเปิดใช้งาน RADIUS บน **AAA-SRV (10.10.99.60)** ตัวเดิมจาก [Part 22]
(part-022-aaa-device-hardening.md) ให้ทำหน้าที่คู่กับ TACACS+ (Dual-role Server), สร้าง SSID
**ENTERPRISE-SECURE** ที่รองรับทั้ง EAP-TLS/PEAP-MSCHAPv2/EAP-FAST, เจาะลึก PKI Trust Chain
ของ EAP-TLS พร้อม Preview บทบาท Cisco ISE, ทำ **Dynamic VLAN Assignment** แยก Contractor ไป
VLAN 45 อัตโนมัติผ่าน RADIUS Attribute, ออกแบบ **Guest Wireless** ด้วย CWA + Anchor WLC Concept,
และปิดท้ายด้วย **wIPS พื้นฐาน** สำหรับตรวจจับ Rogue AP ✅

**พร้อมสำหรับ Part 38**: เราจะเปลี่ยนมุมมองจาก "Wireless Security เฉพาะทาง" ไปสู่ภาพใหญ่ของ
Enterprise Network สมัยใหม่ — **SD-Access (Software-Defined Access)**: แนวคิด Fabric-based
Network, Policy-based Segmentation ที่ทำงานร่วมกับสิ่งที่เราสร้างวันนี้ (Dynamic VLAN,
Identity-based Access) ในรูปแบบที่ Scale ได้กว้างกว่าและอัตโนมัติมากขึ้น รวมถึงจะแนะนำ
**Cisco TrustSec/SGT (Security Group Tag)** ที่ต่อยอดจากแนวคิด Group-based Policy ที่เราเริ่ม
เห็นเค้าลางแล้วใน Dynamic VLAN Assignment ของ Part นี้

**ไปต่อ:** [Part 38 — SD-Access Fundamentals →](part-038-sd-access-fundamentals.md)
