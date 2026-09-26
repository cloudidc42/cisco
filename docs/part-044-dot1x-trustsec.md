# Part 44 — 802.1X & Cisco TrustSec (Wired Access Control + SGT Micro-Segmentation)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 431–440 จาก 1000**

> ต่อจาก [Part 43 — Security Architecture: Firepower & ISE →](part-043-security-architecture-firepower-ise.md)
> ที่ติดตั้ง **Cisco ISE (Identity Services Engine)** เข้ามาเป็น Policy Engine กลางของทั้งองค์กร
> (แทนที่ AAA-SRV แบบ Generic RADIUS ที่ใช้มาตั้งแต่ [Part 37](part-037-wireless-security-advanced.md))
> และพูดถึง **TrustSec/SGT Policy Distribution** ไว้แบบสั้นๆ ว่า ISE เป็นผู้แจก Security Group Tag
> ให้ทุกอุปกรณ์ผ่าน RADIUS — Part นี้คือจุดที่เราจะ **ลงรายละเอียดเต็มรูปแบบ** ของทั้งสองเรื่องนั้น
>
> [Part 37](part-037-wireless-security-advanced.md) สอน **802.1X ฝั่ง Wireless** ไปแล้ว (Client ↔
> WLC ↔ RADIUS ผ่าน EAPOL บน 802.11 Association) Part นี้จะนำ Framework เดียวกันมาใช้กับ
> **สายแลน (Wired)** บน **ACCESS-SW1-4** — เปลี่ยนจากพอร์ต Access ธรรมดาที่ [Part 3]
> (part-003-ethernet-switching-vlan.md) เปิดไว้ ให้กลายเป็นพอร์ตที่ต้อง **พิสูจน์ตัวตนก่อนเข้าเครือข่าย
> ได้จริง** จากนั้นจะขยายไปสู่ **Cisco TrustSec** — เทคโนโลยี Micro-Segmentation ที่ [Part 38]
> (part-038-sd-access-fundamentals.md) แง้มไว้ว่า "รายละเอียดเชิงลึกเต็มรูปแบบของ TrustSec/SGT
> อยู่ที่ Part 44" — วันนี้คือวันที่เราจะเปิดกล่องนั้น

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 431 | Wired 802.1X Fundamentals — ทวน EAP Framework จาก Part 37 ในบริบทของ Switch Port |
| 432 | Port-Based 802.1X States — Unauthorized vs Authorized, `dot1x port-control auto` vs `authentication port-control auto` |
| 433 | คอนฟิก 802.1X เต็มรูปแบบ (IBNS 2.0) บน ACCESS-SW1 ชี้ไปที่ ISE (Part 43) |
| 434 | Host Mode: Single-Host vs Multi-Host vs Multi-Auth vs Multi-Domain — ทำไม IP Phone ต้องใช้ Multi-Domain |
| 435 | MAB (MAC Authentication Bypass) — Fallback สำหรับ Printer/IoT ที่ทำ 802.1X ไม่ได้ |
| 436 | Critical VLAN / Guest VLAN / Auth-Fail VLAN — Graceful Degradation เมื่อ ISE ล่มหรือ Auth ไม่ผ่าน |
| 437 | Cisco TrustSec Fundamentals — SGT, Dynamic Assignment จาก ISE, Inline Tagging vs SXP |
| 438 | SGACL — แทนที่ IP ACL แบบ Part 14 ด้วย Policy ระดับ SGT-to-SGT |
| 439 | SXP Configuration — ส่ง SGT ไปยังอุปกรณ์ที่ไม่รองรับ Inline Tagging |
| 440 | Lab เต็มรูปแบบ: 802.1X+MAB บน ACCESS-SW1-4 + SGT + SXP + SGACL Enforcement ที่ DIST-SW3 |

---

## Step 431 — Wired 802.1X Fundamentals: ทวน EAP Framework ในบริบทของ Switch Port

### กรอบความคิดเดิมจาก Part 37 ไม่เปลี่ยน เปลี่ยนแค่ "สื่อกลาง"

[Part 37 Step 361](part-037-wireless-security-advanced.md) อธิบายไว้ว่า **802.1X (IEEE 802.1X-2010)**
คือ **Framework** ไม่ใช่ Protocol เดี่ยว — มันนิยาม 3 บทบาทที่ต้องมีเสมอไม่ว่าจะ Wired หรือ
Wireless และ **EAP (Extensible Authentication Protocol)** คือ Protocol กลางที่ขนส่ง Credential
ระหว่าง Supplicant กับ Authentication Server สิ่งที่ต่างกันระหว่าง Wired กับ Wireless มีแค่
**"EAPOL เดินทางบนสื่อกลางอะไร"**

| บทบาท (Role) | Wireless (Part 37) | **Wired (Part 44 — เรื่องใหม่วันนี้)** |
|---|---|---|
| **Supplicant** | ซอฟต์แวร์ใน Laptop/Mobile ที่เชื่อม Wi-Fi | ซอฟต์แวร์ใน PC/Laptop ที่เสียบสาย LAN (Windows มี Wired AutoConfig ในตัว) |
| **Authenticator** | WLC-1 (ผ่าน AP) | **ACCESS-SW1-4** — Switch Port แต่ละพอร์ตทำหน้าที่นี้ **แยกกันเป็นอิสระ** |
| **Authentication Server** | AAA-SRV (Part 37) | **ISE-1 (10.10.99.65)** ที่ [Part 43](part-043-security-architecture-firepower-ise.md) ติดตั้งไว้ |
| EAPOL เดินทางบน | 802.11 Association (Over-the-Air) | **Ethernet Frame ตรงๆ** — EtherType `0x888E` ส่งจาก PC ไปยัง Switch Port ทันทีที่เสียบสาย |
| RADIUS ระหว่าง Authenticator ↔ Server | UDP 1812/1813 | UDP 1812/1813 (เหมือนกันทุกประการ — RFC 3579 ไม่สนใจว่า NAS เป็น Switch หรือ WLC) |

```
Wired 802.1X — 3 บทบาทบน ACCESS-SW1

┌──────────────────┐        ┌──────────────────────┐        ┌──────────────────────┐
│   SUPPLICANT       │        │    AUTHENTICATOR        │        │  AUTHENTICATION       │
│   PC1 (10.10.10.11)│ EAPOL  │    ACCESS-SW1            │ RADIUS │  SERVER                │
│   Windows/Linux    │◄──────►│    Gi1/0/1 (แต่ละพอร์ต   │◄──────►│  ISE-1 (10.10.99.65)   │
│   Supplicant        │ 0x888E │    เป็น Authenticator     │ UDP    │  (Part 43)             │
│                     │ ตรงบน  │    ของตัวเอง — ไม่มีการ   │ 1812   │  - Verify Credential   │
│                     │ สาย LAN│    "แชร์" สถานะข้าม     │        │  - ตอบ Access-Accept   │
│                     │        │    พอร์ต)                │        │    พร้อม VLAN/SGT      │
└──────────────────┘        └──────────────────────┘        └──────────────────────┘
```

### จุดที่ต่างจาก Wireless อย่างมีนัยสำคัญ: "1 พอร์ต = 1 State Machine อิสระ"

บน WLC ตัวเดียวต้องดูแล Client หลายร้อยตัวที่แบ่งกันใช้ Air เดียวกัน (Shared Medium) แต่บน Switch
**แต่ละ Physical Port คือ Collision Domain ของตัวเอง** — 802.1X State Machine (จะเรียนใน Step 432)
จึงรันแยกอิสระทีละพอร์ต การที่พอร์ต Gi1/0/1 Authenticate ไม่ผ่านจะ**ไม่กระทบ**พอร์ต Gi1/0/2 เลย
ต่างจาก Wireless ที่ Client จำนวนมากแย่งกันใช้ RF Channel เดียวกัน — นี่คือเหตุผลที่ Wired 802.1X
"เนียน" กว่าและ Debug ง่ายกว่า Wireless มาก (ไม่มีปัจจัย RF Interference มาปน)

### ทำไมยังต้องเรียน Wired 802.1X ทั้งที่มี Physical Security ของสาย LAN อยู่แล้ว

ข้อโต้แย้งที่พบบ่อย: "สาย LAN อยู่ในตึกที่มี Access Control (บัตรพนักงาน/รปภ.) แล้ว จำเป็นต้องมี
802.1X อีกไหม" — คำตอบคือ **จำเป็นมาก** เพราะ Port Security แบบ MAC-based ([Part 21]
(part-021-switch-security.md)) ป้องกันได้แค่ "MAC Spoofing" แต่ไม่สามารถตอบคำถามที่สำคัญกว่าคือ
**"ใคร (Identity) เสียบสายเข้ามาที่พอร์ตนี้"** — ตัวอย่างจริง: Ports ในห้องประชุม/Lobby ที่ Guest
เดินเข้ามาเสียบสาย Laptop ได้เลยถ้าไม่มี 802.1X, หรือพนักงานที่ลาออกแล้วแต่ยังรู้ MAC/Password
เดิม — 802.1X ผูก Access เข้ากับ **Identity ที่ยืนยันได้จริง** ผ่าน ISE ซึ่งคือรากฐานของ
**Zero Trust Network Access (ZTNA)** ระดับ Campus ที่ [Part 38](part-038-sd-access-fundamentals.md)
สร้างเป็น SD-Access Fabric ไว้

---

## Step 432 — Port-Based 802.1X States: Unauthorized vs Authorized

### 2 สถานะหลักของพอร์ตที่เปิด 802.1X

| State | คำอธิบาย | Traffic ที่ผ่านได้ |
|---|---|---|
| **Unauthorized** (ค่า Default ทันทีที่เปิด 802.1X) | พอร์ต **บล็อกทุกอย่าง** ยกเว้น EAPOL | เฉพาะ EAPOL (Destination MAC `01:80:C2:00:00:03` — IEEE 802.1X PAE Group Address) — ไม่มี DHCP, ARP, ping, หรือ Data Traffic ใดๆผ่านได้เลย |
| **Authorized** (หลัง Authenticate สำเร็จ) | พอร์ตเปิดใช้งานปกติเต็มรูปแบบ | Traffic ทั้งหมดของ VLAN ที่ถูก Assign ให้ (Data VLAN ปกติ หรือ VLAN ที่ RADIUS ส่งมา) |

```
Port State Machine (แนวคิดง่ายของ dot1x/authentication State Machine)

        [Link Up / เสียบสาย]
               │
               ▼
     ┌───────────────────┐
     │   UNAUTHORIZED       │◄────────────────────────────┐
     │  (บล็อกทุกอย่าง       │                              │
     │   ยกเว้น EAPOL)       │                              │ Link Down /
     └──────────┬──────────┘                              │ EAPOL-Logoff /
                │ EAPOL-Start จาก Supplicant                │ authentication
                ▼                                          │ violation
     ┌───────────────────┐        Access-Accept   ┌────────┴────────┐
     │  Authenticating      │───────จาก ISE────────►│   AUTHORIZED       │
     │  (EAP Request/Response│                       │  (Traffic ผ่านได้    │
     │   วนไปมากับ ISE)      │◄──────────────────────│   ปกติตาม VLAN/SGT) │
     └──────────┬──────────┘   Access-Reject          └───────────────┘
                │
                ▼
     กลับไป UNAUTHORIZED (หรือไป Guest/Auth-Fail VLAN — Step 436)
```

> **จุดสำคัญที่สุดของ Step นี้**: สถานะ Default ของพอร์ตที่เปิด 802.1X คือ **Unauthorized เสมอ**
> — นี่คือหลักการ **"Deny by Default"** ระดับ Layer 2 ที่แตกต่างจาก ACL ([Part 14]
> (part-014-access-control-lists.md)) ที่ทำงานที่ Layer 3/4 — 802.1X บล็อกตั้งแต่ **ก่อน** Client
> จะได้ IP Address เสียอีก (ไม่มี DHCP Discover ผ่านได้เลยจนกว่าจะ Authorized)

### Syntax เปรียบเทียบ: `dot1x port-control auto` (IBNS 1.0 — Legacy) vs `authentication port-control auto` (IBNS 2.0 — ที่ Part นี้จะใช้ตลอด)

| รายการ | IBNS 1.0 (Legacy — Cisco IOS รุ่นเก่า) | **IBNS 2.0 (ปัจจุบัน — ใช้ตลอด Part นี้)** |
|---|---|---|
| เปิด Authentication บน Interface | `dot1x port-control auto` | `authentication port-control auto` |
| จำกัดวิธี Authenticate ได้กี่แบบ | **802.1X เท่านั้น** ต่อพอร์ต | **หลายวิธีผสมกันได้** ผ่าน `authentication order dot1x mab` (Step 435) |
| จัดการ Fallback (MAB/Guest/Critical) | ต้องเขียนแยกเป็นคำสั่งเฉพาะทาง (`dot1x auth-fail vlan`, `dot1x guest-vlan` แบบเก่า) | รวมอยู่ใน `authentication event ...` Framework เดียว (Step 436) |
| Host Mode Command | `dot1x host-mode multi-host` | `authentication host-mode multi-domain` (Step 434) |
| ความยืดหยุ่น | ต่ำ — ออกแบบมาสมัยที่มีแค่ 802.1X วิธีเดียว | สูง — ออกแบบมาให้รองรับ 802.1X + MAB + Web-Auth ผสมกันตั้งแต่ต้น |
| Cisco แนะนำสำหรับ Deployment ใหม่ | ❌ Deprecated | ✅ **ใช้เสมอสำหรับ Config ใหม่** (Catalyst 9200/9300 รองรับทั้งคู่ แต่แนะนำ IBNS 2.0) |

> **ทำไม Part นี้เลือก IBNS 2.0**: เพราะ Lab ของเราต้องรองรับ **ทั้ง 802.1X และ MAB ผสมกัน**
> บนพอร์ตเดียว (Step 435) และต้องมี **Multi-Domain Host Mode** สำหรับ IP Phone (Step 434) —
> IBNS 1.0 ทำได้ยากและอ่านยากกว่ามากถ้าจะ mix วิธีหลายแบบ คำสั่ง `dot1x port-control auto`
> ยังพบใน Cisco Exam/เอกสารเก่าอยู่บ้าง จึงต้องรู้จักไว้ แต่ **ไม่ใช้จริงในหลักสูตรนี้อีกต่อไป**

### คำสั่งพื้นฐานที่สุดที่ต้องมีก่อนเปิด 802.1X บนพอร์ตใดๆ

```
! คำสั่ง Global เดียวที่ต้องเปิดก่อนเสมอ — ไม่มีคำสั่งนี้ ทุกพอร์ตจะเป็น Authorized ตลอดเวลา
! (คือยังไม่มี Enforcement อะไรเกิดขึ้นจริง แม้จะใส่คำสั่งระดับ Interface ไปแล้วก็ตาม)
ACCESS-SW1(config)# dot1x system-auth-control
```

`dot1x system-auth-control` คือ **Master Switch** ของทั้งอุปกรณ์ — ถ้าไม่เปิดคำสั่งนี้
คำสั่ง `authentication port-control auto` ที่ Interface จะไม่มีผลอะไรเลย (พอร์ตจะทำงานเป็น
Access Port ปกติทุกประการ เหมือนไม่มี 802.1X) รายละเอียด Config เต็มรูปแบบอยู่ใน Step 433

---

## Step 433 — คอนฟิก 802.1X เต็มรูปแบบ (IBNS 2.0) บน ACCESS-SW1 ชี้ไปที่ ISE

### ภาพรวม: จาก Part 3 (Access Port ธรรมดา) สู่ Part 44 (802.1X-Enforced Port)

[Part 3](part-003-ethernet-switching-vlan.md) ตั้งค่า ACCESS-SW1 ไว้ให้ Gi1/0/1-3 เป็น Access Port
VLAN 10 (SALES) + Voice VLAN 20 ธรรมดา ไม่มีการพิสูจน์ตัวตนใดๆ — Step นี้จะเพิ่ม 802.1X
เข้าไปบน Config เดิมนั้น **โดยไม่ต้องรื้อ VLAN Assignment ที่มีอยู่แล้ว**

### 1) เปิด AAA และประกาศ ISE เป็น RADIUS Server (Pattern เดียวกับ Part 37 Step 362 ที่ WLC-1 ทำกับ AAA-SRV)

```
ACCESS-SW1# configure terminal

! ============================================
! 1. เปิด AAA (จำเป็นเสมอก่อนใช้ RADIUS/TACACS+ ใดๆ — ทำไปแล้วจาก Part 22 ถ้า Login Local ใช้ TACACS+
!    อยู่แล้ว แต่ต้องตรวจสอบซ้ำเพราะ dot1x ต้องพึ่ง aaa new-model เหมือนกัน)
! ============================================
ACCESS-SW1(config)# aaa new-model

! ============================================
! 2. ประกาศ ISE-1 เป็น RADIUS Server (แทน AAA-SRV เดิมสำหรับงาน Network Access — ISE ติดตั้งจาก Part 43)
! ============================================
ACCESS-SW1(config)# radius server ISE-1
ACCESS-SW1(config-radius-server)# address ipv4 10.10.99.65 auth-port 1812 acct-port 1813
ACCESS-SW1(config-radius-server)# key TrustSec-Dot1X-Key2026!
ACCESS-SW1(config-radius-server)# timeout 5
ACCESS-SW1(config-radius-server)# retransmit 2
ACCESS-SW1(config-radius-server)# exit

! ============================================
! 3. AAA Server Group + Method List สำหรับ 802.1X/MAB โดยเฉพาะ
! ============================================
ACCESS-SW1(config)# aaa group server radius ISE-GRP
ACCESS-SW1(config-sg-radius)# server name ISE-1
ACCESS-SW1(config-sg-radius)# exit

ACCESS-SW1(config)# aaa authentication dot1x default group ISE-GRP
ACCESS-SW1(config)# aaa authorization network default group ISE-GRP
ACCESS-SW1(config)# aaa accounting dot1x default start-stop group ISE-GRP

! ============================================
! 4. บอก Switch ให้ยอมรับ Vendor-Specific Attribute จาก ISE (VLAN, SGT, ACL ที่ส่งมาพร้อม Access-Accept)
! ============================================
ACCESS-SW1(config)# radius-server vsa send authentication
ACCESS-SW1(config)# radius-server vsa send accounting

! ============================================
! 5. เปิด Master Switch ของ 802.1X (Step 432)
! ============================================
ACCESS-SW1(config)# dot1x system-auth-control
```

### 2) Device Tracking — โครงสร้างพื้นฐานที่ 802.1X/MAB/TrustSec ทุกอย่างต้องพึ่ง

IBNS 2.0 ต้องรู้ **IP Address** ของ Client ที่ผ่าน Authentication แล้ว (เพื่อสร้าง IP-SGT Binding
ใน Step 437-440 และเพื่อบังคับ DHCP ให้ Client ต้องขอ IP ใหม่หลัง Authorized) ผ่าน Feature
**Device Tracking** (แทนคำสั่งเก่า `ip device tracking` บน IOS-XE รุ่นใหม่)

```
ACCESS-SW1(config)# device-tracking tracking auto-source
ACCESS-SW1(config)# device-tracking policy IBNS2-POLICY
ACCESS-SW1(config-device-tracking)# limit address-count 4
ACCESS-SW1(config-device-tracking)# tracking enable
ACCESS-SW1(config-device-tracking)# exit
```

### 3) Interface Template พื้นฐานที่สุด (Single-Host, ยังไม่มี MAB/Multi-Domain — จะเติมใน Step 434-436)

ตัวอย่างนี้ใช้ Gi1/0/23 (Local Management Station จาก Part 3 — เป็น PC เดี่ยวไม่มี Phone)
เพื่อสอน Syntax พื้นฐานที่สุดก่อน ส่วนพอร์ตที่มี Phone (Gi1/0/1-3) จะเรียนใน Step 434

```
ACCESS-SW1(config)# interface GigabitEthernet1/0/23
ACCESS-SW1(config-if)# switchport mode access
ACCESS-SW1(config-if)# switchport access vlan 99
ACCESS-SW1(config-if)# device-tracking attach-policy IBNS2-POLICY
ACCESS-SW1(config-if)# access-session port-control auto
ACCESS-SW1(config-if)# dot1x pae authenticator
ACCESS-SW1(config-if)# dot1x timeout tx-period 7
ACCESS-SW1(config-if)# spanning-tree portfast
ACCESS-SW1(config-if)# no shutdown
ACCESS-SW1(config-if)# exit

ACCESS-SW1(config)# end
ACCESS-SW1# copy running-config startup-config
```

> **หมายเหตุเรื่อง Syntax**: บน IOS-XE รุ่นใหม่ (Catalyst 9200/9300) คำสั่งเปิด 802.1X ที่ Interface
> จริงๆมีทั้ง `authentication port-control auto` (Interface Template แบบ Step 432 เดิม) และ
> `access-session port-control auto` (Syntax ล่าสุดที่ผูกกับ `access-session` Feature Set เต็มรูปแบบ)
> — ทั้งสองคำสั่งทำงานเหมือนกันในทางปฏิบัติสำหรับ Lab ระดับนี้ เอกสารนี้จะใช้ `authentication
> port-control auto` เป็นหลักเพราะตรงกับคำศัพท์ที่ข้อสอบ CCNP ENCOR ใช้บ่อยที่สุด แต่ให้รู้จัก
> `access-session` ไว้เผื่อเจอใน `show running-config` ของอุปกรณ์รุ่นใหม่จริง

### ตรวจสอบผลลัพธ์เบื้องต้น

```
ACCESS-SW1# show dot1x all summary
Interface        PAE       Client            Status
Gi1/0/23          AUTH      -                 UNAUTHORIZED

ACCESS-SW1# show aaa servers | include ISE-1
RADIUS: id 1, priority 1, host 10.10.99.65, auth-port 1812, acct-port 1813
     State: current UP, duration 120s, previous duration 0s
```

พอร์ต Gi1/0/23 ตอนนี้อยู่ในสถานะ **UNAUTHORIZED** ทันที (ตามหลักการ Step 432) — PC ที่เสียบอยู่
จะยังไม่ได้ IP เลยจนกว่าจะมี Supplicant ส่ง EAPOL-Start และ Authenticate ผ่าน ISE-1 สำเร็จ

---

## Step 434 — Host Mode: Single-Host vs Multi-Host vs Multi-Auth vs Multi-Domain

### ทำไมต้องมี "Host Mode" — ปัญหาที่มันแก้

พอร์ต Switch หนึ่งพอร์ตในทางทฤษฎีมี Endpoint ต่อได้มากกว่า 1 ตัว (ผ่าน Hub, หรือกรณีที่พบบ่อย
ที่สุดคือ **IP Phone ที่มี Switch Port ในตัวให้ PC ต่อผ่าน** — Pattern ที่ [Part 3]
(part-003-ethernet-switching-vlan.md) ตั้งไว้ตั้งแต่แรก: `switchport access vlan 10` +
`switchport voice vlan 20` บนพอร์ตเดียว) **Host Mode** คือ Policy ที่กำหนดว่า "พอร์ตนี้ยอมให้กี่
Endpoint Authenticate พร้อมกัน และปฏิบัติต่อ Endpoint แต่ละตัวยังไง"

### 4 Host Mode เปรียบเทียบ

| Host Mode | จำนวน Endpoint ที่ Authenticate ได้ | ใช้กับอะไร | ระดับความปลอดภัย |
|---|---|---|---|
| **Single-Host** (Default) | **1 ตัวเท่านั้น** — ถ้ามีตัวที่สองมาต่อ (เช่นผ่าน Hub) จะเจอ `security violation` และพอร์ตอาจ err-disable | PC เดี่ยวที่ไม่มี Phone (Gi1/0/23 ใน Step 433) | สูงสุด |
| **Multi-Host** | หลายตัว แต่**ให้ตัวแรกที่ Authenticate สำเร็จ "ปลดล็อก" ให้ทั้งพอร์ต** — ตัวอื่นที่ต่อผ่าน Hub เดียวกันได้สิทธิ์ตามไปด้วยฟรี **ไม่ต้อง Authenticate เอง** | Hub เก่าหลัง 1 Endpoint ที่ Trust ได้ (ปัจจุบันใช้น้อยมาก) | ต่ำ — เป็นช่องโหว่ถ้ามีคนแอบต่อ Hub เพิ่ม |
| **Multi-Auth** | **หลายตัว และทุกตัวต้อง Authenticate แยกกันเอง** ไม่มีการ "แถม" สิทธิ์ | Access Point ที่มี Client หลายเครื่องเชื่อมต่อผ่านมันเข้า Switch (Wireless-behind-Wired), หรือ IP Phone รุ่นเก่าที่ไม่แยก Voice/Data Domain | สูง — แต่ไม่แยก Domain (ดูข้อจำกัดด้านล่าง) |
| **Multi-Domain** ⭐ | **2 ตัวพอดี** — แบ่งเป็น **Data Domain** (1 ตัว) + **Voice Domain** (1 ตัว) ชัดเจน แต่ละ Domain Authenticate อิสระและได้ VLAN คนละอัน | **IP Phone + PC ต่อผ่าน Phone** — Use Case ที่พบบ่อยที่สุดในทุก Enterprise Access Layer | สูง — และถูก Design มาสำหรับ Use Case นี้โดยเฉพาะ |

### ทำไม IP Phone ต้องใช้ Multi-Domain โดยเฉพาะ ไม่ใช่ Multi-Auth

Multi-Auth มองทุก Endpoint เป็น "ระดับเดียวกัน" ไม่รู้จักแนวคิด Voice/Data แยกกัน — ถ้าใช้ Multi-Auth
กับพอร์ตที่มี Phone+PC ปัญหาคือ **Switch ไม่รู้ว่าใครควรได้ Data VLAN (10) และใครควรได้ Voice VLAN
(20)** ต้องพึ่ง Authorization Result (Attribute จาก ISE) ในการแยกเอง ซึ่งเสี่ยงผิดพลาด — **Multi-Domain
แก้ปัญหานี้ตรงจุด**: มันผูก **"Domain" เข้ากับ CDP/LLDP-MED Voice VLAN Discovery ที่ Phone ประกาศ
ตัวเองอยู่แล้ว** (Phone บอก Switch ผ่าน CDP ว่า "ฉันคือ Voice Endpoint") ทำให้ Switch แยก Domain
ได้แม่นยำโดยไม่ต้องพึ่ง MAC OUI Guessing หรือ Attribute พิเศษจาก RADIUS เลย

```
Multi-Domain Host Mode บน Gi1/0/1 (ACCESS-SW1) — PC1 + Phone1

                    ┌─────────────────────────────┐
       PC1 ─────────┤  Gi1/0/1                       │
   (10.10.10.11)    │  authentication host-mode       │
                    │  multi-domain                    │
   Phone1 ──────────┤                                  ├──── Uplink (Trunk) ไป DIST-SW1
  (10.10.20.21)      │  Data Domain  → PC1  → VLAN 10   │
                    │  Voice Domain → Phone1→ VLAN 20   │
                    └─────────────────────────────┘

- PC1 authenticate ผ่าน EAPOL ของตัวเอง → ISE ตอบ Access-Accept (Data Domain) → VLAN 10
- Phone1 authenticate ผ่าน EAPOL ของตัวเอง (แยกจาก PC1 คนละ Session) → ISE ตอบ Access-Accept
  (Voice Domain, มักมี Attribute cisco-av-pair="device-traffic-class=voice" กำกับมาด้วย) → VLAN 20
- ถ้ามี Endpoint ตัวที่ 3 พยายาม Authenticate บนพอร์ตนี้ (เช่น Client แปลกปลอมอ้างเป็น Data อีกตัว)
  → Switch จะปฏิเสธทันทีเพราะ Multi-Domain อนุญาตแค่ 1 Data + 1 Voice เท่านั้น
```

### Config เต็มรูปแบบ: ACCESS-SW1 Gi1/0/1-3 (PC1-3 + Phone1-3)

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1-3
ACCESS-SW1(config-if-range)# switchport mode access
ACCESS-SW1(config-if-range)# switchport access vlan 10
ACCESS-SW1(config-if-range)# switchport voice vlan 20
ACCESS-SW1(config-if-range)# device-tracking attach-policy IBNS2-POLICY
ACCESS-SW1(config-if-range)# authentication host-mode multi-domain
ACCESS-SW1(config-if-range)# authentication port-control auto
ACCESS-SW1(config-if-range)# dot1x pae authenticator
ACCESS-SW1(config-if-range)# dot1x timeout tx-period 7
ACCESS-SW1(config-if-range)# mls qos trust cos
ACCESS-SW1(config-if-range)# spanning-tree portfast
ACCESS-SW1(config-if-range)# no shutdown
ACCESS-SW1(config-if-range)# exit
ACCESS-SW1(config)# end
ACCESS-SW1# copy running-config startup-config
```

> **ACCESS-SW2 ใช้ Config Pattern เดียวกันทุกประการ** บน Gi1/0/1-2 (PC4-5 + Phone4-5 จาก Part 3)
> — เปลี่ยนแค่หมายเลข Interface ตามที่ [Part 3](part-003-ethernet-switching-vlan.md) กำหนดไว้

### ตรวจสอบผลลัพธ์: `show authentication sessions`

```
ACCESS-SW1# show authentication sessions interface GigabitEthernet1/0/1
            Interface:  GigabitEthernet1/0/1
          MAC Address:  0050.5601.aaaa
               IPv6 Address:  Unknown
               IP Address:  10.10.10.11
                User-Name:  PC1-employee01
                   Status:  Authorized
                   Domain:  DATA
          Security Policy:  Should Secure
          Security Status:  Link Unsecure
           Oper host mode:  multi-domain
         Oper control dir:  both
            Authorized By:  Authentication Server
              Vlan Policy:  10

ACCESS-SW1# show authentication sessions interface GigabitEthernet1/0/1 | include Domain|MAC|Vlan
```

**ผลลัพธ์เมื่อทั้ง Data + Voice Domain Authenticate สำเร็จ (สรุป):**
```
ACCESS-SW1# show authentication sessions interface GigabitEthernet1/0/1 details
Domain: DATA    MAC: 0050.5601.aaaa   VLAN: 10   Status: Authorized
Domain: VOICE   MAC: 0050.5602.bbbb   VLAN: 20   Status: Authorized
```

---

## Step 435 — MAB (MAC Authentication Bypass): Fallback สำหรับ Printer/IoT

### ปัญหาที่ MAB แก้: อุปกรณ์ที่ไม่มี Supplicant ให้ Config เลย

Printer, IP Camera, IoT Sensor, และอุปกรณ์ Legacy จำนวนมากไม่มีซอฟต์แวร์ 802.1X Supplicant ติดตั้ง
มาให้ (และส่วนใหญ่ไม่มีหน้าจอให้ตั้งค่าด้วยซ้ำ) — ถ้าบังคับ 802.1X อย่างเดียวบนทุกพอร์ต อุปกรณ์
เหล่านี้จะเข้าเครือข่ายไม่ได้เลย **MAB (MAC Authentication Bypass)** คือ Fallback ที่ให้ Switch
ใช้ **MAC Address ของอุปกรณ์เอง** เป็น "Credential" ส่งไปให้ ISE ตรวจสอบแทน (ISE ต้องมี MAC
เหล่านี้ลงทะเบียนไว้ล่วงหน้าใน Endpoint Database — เรียนเรื่อง Endpoint Profiling ของ ISE ใน
[Part 43](part-043-security-architecture-firepower-ise.md))

```
ลำดับการทำงานเมื่อตั้งค่า authentication order dot1x mab

Endpoint เสียบสาย
       │
       ▼
┌─────────────────────┐
│ รอ EAPOL-Start          │  <- ลอง 802.1X ก่อนเสมอ (ตาม order ที่กำหนด)
│ (dot1x timeout          │
│  tx-period × retry)     │
└──────────┬──────────┘
           │ ไม่มี EAPOL ตอบกลับเลย (Timeout ครบ retry) — แสดงว่าไม่มี Supplicant
           ▼
┌─────────────────────┐
│  Fallback ไป MAB         │  <- Switch อ่าน MAC Address จาก Frame แรกที่ endpoint ส่งมา
│  (ส่ง MAC เป็น              │     (มักเป็น DHCP Discover หรือ ARP) ส่งเป็น
│   Username/Password       │     Access-Request ให้ ISE ตรวจสอบ
│   ไปให้ ISE)                │
└──────────┬──────────┘
           │
           ▼
   Access-Accept (MAC อยู่ใน Endpoint DB) → Authorized, VLAN/SGT ตามที่ ISE กำหนด
   Access-Reject (MAC ไม่รู้จัก) → ไป Guest/Auth-Fail VLAN (Step 436)
```

### `authentication order` vs `authentication priority` — คนละหน้าที่กัน (จุดที่สับสนบ่อยที่สุด)

| คำสั่ง | ควบคุมอะไร | ตัวอย่าง |
|---|---|---|
| `authentication order dot1x mab` | **ลำดับที่ Switch จะ "ลอง"** วิธี Authenticate — ลองอันแรกก่อน ถ้า Timeout/ไม่มี Response ค่อยลองอันต่อไป | ลอง 802.1X ก่อน ถ้าไม่มี Supplicant เลยค่อย fallback ไป MAB |
| `authentication priority dot1x mab` | **ถ้าทั้งสองวิธี "แข่งกันสำเร็จพร้อมกัน"** (เช่น MAB สำเร็จไปแล้วระหว่างรอ 802.1X Timeout แล้ว Supplicant ค่อยส่ง EAPOL-Start มาทีหลัง) — วิธีไหนมี Priority สูงกว่าจะ **แทนที่** ผลลัพธ์เดิม | ให้ 802.1X ชนะ MAB เสมอถ้า Supplicant โผล่มาทีหลัง (เพราะ 802.1X ยืนยันตัวบุคคลได้แน่นอนกว่า MAC ที่ Spoof ได้ง่าย) |

> **กฎจำง่าย**: `order` = "ลองอะไรก่อน" (Sequence), `priority` = "เชื่อผลของอะไรมากกว่า" (Trust
> Ranking) — ตั้งสองคำสั่งนี้**เหมือนกันได้**ในกรณีส่วนใหญ่ (`dot1x mab` ทั้งคู่) แต่มันคือคำสั่ง
> คนละตัวที่ทำหน้าที่ต่างกันเสมอ

### Timing Tuning — ทำให้ Fallback ไป MAB เร็วขึ้น (ค่า Default รอนานเกินไปสำหรับ Printer)

ค่า Default ของ `dot1x timeout tx-period` คือ 30 วินาที และ Switch จะส่ง EAPOL-Request/Identity
ซ้ำ **`dot1x max-reauth-req`** ครั้ง (Default 2) ก่อนยอม Fallback ไป MAB — รวมแล้ว Printer
อาจต้องรอเกือบ **60-90 วินาทีกว่าจะได้ IP** ถ้าไม่ Tune ค่านี้ ซึ่งช้าเกินไปสำหรับงานจริง

```
ACCESS-SW3(config-if)# dot1x timeout tx-period 3
ACCESS-SW3(config-if)# dot1x max-reauth-req 2
! ผลลัพธ์: Switch รอ 3 วินาที x 2 ครั้ง = 6 วินาที ก่อน Fallback ไป MAB (เร็วขึ้นมากจาก Default)
```

### Config เต็มรูปแบบ: Printer1 บน ACCESS-SW3 Gi1/0/4 (Single-Host + MAB Fallback)

Printer1 เป็นอุปกรณ์ใหม่ที่เพิ่มเข้ามาต่อพอร์ตที่ [Part 3](part-003-ethernet-switching-vlan.md)
ทิ้งว่างไว้ (`Gi1/0/4 notconnect` จาก `show interfaces status`) — วาง VLAN 30 (SERVERS) เดิม
เพราะ Printer อยู่ใน Segment เดียวกับ Server ตามธรรมเนียม Print Server ขององค์กร

```
ACCESS-SW3(config)# interface GigabitEthernet1/0/4
ACCESS-SW3(config-if)# description ** Printer1 - MAB Fallback (no 802.1X supplicant) **
ACCESS-SW3(config-if)# switchport mode access
ACCESS-SW3(config-if)# switchport access vlan 30
ACCESS-SW3(config-if)# device-tracking attach-policy IBNS2-POLICY
ACCESS-SW3(config-if)# authentication host-mode single-host
ACCESS-SW3(config-if)# authentication order dot1x mab
ACCESS-SW3(config-if)# authentication priority dot1x mab
ACCESS-SW3(config-if)# authentication port-control auto
ACCESS-SW3(config-if)# dot1x pae authenticator
ACCESS-SW3(config-if)# dot1x timeout tx-period 3
ACCESS-SW3(config-if)# dot1x max-reauth-req 2
ACCESS-SW3(config-if)# mab
ACCESS-SW3(config-if)# spanning-tree portfast
ACCESS-SW3(config-if)# no shutdown
ACCESS-SW3(config-if)# exit
ACCESS-SW3(config)# end
ACCESS-SW3# copy running-config startup-config
```

> คำสั่ง **`mab`** เดี่ยวๆ (ไม่มี Argument) คือตัวเปิดใช้งาน MAB บน Interface — ถ้าไม่มีคำสั่งนี้
> ต่อให้ตั้ง `authentication order dot1x mab` ไว้ Switch ก็จะไม่รู้จักวิธี MAB เลย (Order บอกแค่
> "ลำดับ" แต่ต้องเปิดแต่ละ Method เองด้วยคำสั่งเฉพาะของมัน เหมือนกับที่ `dot1x pae authenticator`
> ต้องมีคู่กับ 802.1X)

### ACCESS-SW4 Gi1/0/1-2 (AP-1/AP-2) — Config Pattern เดียวกัน

AP-1/AP-2 ก็ไม่มี 802.1X Supplicant ในตัว (มันเป็น Infrastructure Device ที่ IT ควบคุม ไม่ใช่
End-user Endpoint) ใช้ MAB Fallback Pattern เดียวกันกับ Printer1 ทุกประการ เปลี่ยนแค่ VLAN
เป็น 40 (WIFI) ตามที่ [Part 3](part-003-ethernet-switching-vlan.md) กำหนดไว้

```
ACCESS-SW4(config)# interface range GigabitEthernet1/0/1-2
ACCESS-SW4(config-if-range)# switchport mode access
ACCESS-SW4(config-if-range)# switchport access vlan 40
ACCESS-SW4(config-if-range)# device-tracking attach-policy IBNS2-POLICY
ACCESS-SW4(config-if-range)# authentication host-mode single-host
ACCESS-SW4(config-if-range)# authentication order dot1x mab
ACCESS-SW4(config-if-range)# authentication priority dot1x mab
ACCESS-SW4(config-if-range)# authentication port-control auto
ACCESS-SW4(config-if-range)# dot1x pae authenticator
ACCESS-SW4(config-if-range)# dot1x timeout tx-period 3
ACCESS-SW4(config-if-range)# mab
ACCESS-SW4(config-if-range)# spanning-tree portfast
ACCESS-SW4(config-if-range)# no shutdown
ACCESS-SW4(config-if-range)# exit
```

### ตรวจสอบ MAB ทำงานถูกต้อง

```
ACCESS-SW3# show authentication sessions interface GigabitEthernet1/0/4 details
            Interface:  GigabitEthernet1/0/4
          MAC Address:  001a.a0aa.1234
                   Method:  mab
                   Status:  Authorized
              Authorized By:  Authentication Server
              Vlan Policy:  30

ACCESS-SW3# show mab all
Interface   MAC Address     Method   Status
Gi1/0/4     001a.a0aa.1234  mab      AUTHORIZED
```

---

## Step 436 — Critical VLAN / Guest VLAN / Auth-Fail VLAN: Graceful Degradation

### ทำไมต้องมี "แผนสำรอง" ทั้ง 3 แบบ — 3 สถานการณ์ที่ต่างกันโดยสิ้นเชิง

| สถานการณ์ | อาการ | คำตอบที่ถูก |
|---|---|---|
| **ISE เข้าไม่ถึงเลย** (Network ล่ม, ISE Down) | Switch ส่ง RADIUS Access-Request ไปแล้ว **ไม่มี Response กลับมาเลย** (ไม่ใช่ Reject — คือ Server หายไปจากโลก) | **Critical VLAN** — Authorize ไปก่อนแบบจำกัดสิทธิ์ ดีกว่าปิดพอร์ตทั้งบริษัทเพราะ ISE ล่มตัวเดียว |
| **Endpoint ไม่มี Supplicant และ MAC ไม่อยู่ใน Endpoint DB ของ ISE เลย** | 802.1X Timeout ไปแล้ว, MAB ก็ถูก Reject เพราะ ISE ไม่รู้จัก MAC นี้ | **Guest VLAN** — ให้สิทธิ์ Internet-only แบบจำกัด (คล้าย CWA ฝั่ง Wireless จาก [Part 37 Step 367](part-037-wireless-security-advanced.md) แต่ไม่มี Web Portal ที่ระดับนี้) |
| **มี Supplicant ส่ง EAPOL มา แต่ Credential ผิด** (เช่น Password พนักงานที่ลาออกแล้ว) | ISE ตอบ **Access-Reject ชัดเจน** (ไม่ใช่ Timeout) | **Auth-Fail VLAN** — แยกไปกักไว้ต่างหาก ไม่ใช่ปฏิเสธเข้าเครือข่ายเลย (ป้องกัน Help Desk ถูกโทรถามรัวๆ แต่ยังจำกัดสิทธิ์เต็มที่) |

### เพิ่ม VLAN ใหม่ 3 ตัวสำหรับ Use Case นี้ (ส่วนขยายเฉพาะ Part 44 — Pattern เดียวกับที่ Part 37 เพิ่ม VLAN 45/50)

Lab ของหลักสูตรนี้ยังไม่มี VLAN สำหรับ Use Case นี้ใน [IP Address Plan](00-ip-address-plan.md) —
เพิ่มเข้ามาที่ **DIST-SW1** (จุดที่ ACCESS-SW1 Uplink ไปตาม [Access-Layer Mapping]
(00-ip-address-plan.md) — ACCESS-SW2 ทำ Pattern เดียวกันกับ DIST-SW2)

| VLAN | ชื่อ | Subnet | หมายเหตุ |
|---|---|---|---|
| **91** | **DOT1X-CRITICAL** | **10.10.91.0/24** | ISE เข้าไม่ถึง — จำกัดสิทธิ์เข้า Internal Resource แต่ยังทำงานพื้นฐานได้ |
| **92** | **DOT1X-GUEST** | **10.10.92.0/24** | ไม่มี Supplicant/MAC ไม่รู้จัก — Internet-only |
| **93** | **DOT1X-AUTHFAIL** | **10.10.93.0/24** | Credential ผิด — กักไว้ต่างหาก รอ Help Desk แก้ไข |

```
DIST-SW1(config)# vlan 91
DIST-SW1(config-vlan)# name DOT1X-CRITICAL
DIST-SW1(config)# vlan 92
DIST-SW1(config-vlan)# name DOT1X-GUEST
DIST-SW1(config)# vlan 93
DIST-SW1(config-vlan)# name DOT1X-AUTHFAIL
DIST-SW1(config-vlan)# exit

DIST-SW1(config)# interface Vlan91
DIST-SW1(config-if)# ip address 10.10.91.1 255.255.255.0
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan92
DIST-SW1(config-if)# ip address 10.10.92.1 255.255.255.0
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface Vlan93
DIST-SW1(config-if)# ip address 10.10.93.1 255.255.255.0
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! ตัดสิทธิ์ VLAN 91/93 ไม่ให้เข้า Internal Network เลย (คล้าย CONTRACTOR-RESTRICT จาก Part 37 Step 366)
DIST-SW1(config)# ip access-list extended DOT1X-FALLBACK-RESTRICT
DIST-SW1(config-ext-nacl)# deny ip 10.10.91.0 0.0.0.255 10.10.10.0 0.0.0.255
DIST-SW1(config-ext-nacl)# deny ip 10.10.91.0 0.0.0.255 10.10.30.0 0.0.0.255
DIST-SW1(config-ext-nacl)# deny ip 10.10.92.0 0.0.0.255 10.10.10.0 0.0.0.255
DIST-SW1(config-ext-nacl)# deny ip 10.10.92.0 0.0.0.255 10.10.30.0 0.0.0.255
DIST-SW1(config-ext-nacl)# deny ip 10.10.93.0 0.0.0.255 10.10.10.0 0.0.0.255
DIST-SW1(config-ext-nacl)# deny ip 10.10.93.0 0.0.0.255 10.10.30.0 0.0.0.255
DIST-SW1(config-ext-nacl)# permit ip any any
DIST-SW1(config-ext-nacl)# exit
DIST-SW1(config)# interface range Vlan91,Vlan92,Vlan93
DIST-SW1(config-if-range)# ip access-group DOT1X-FALLBACK-RESTRICT in
DIST-SW1(config-if-range)# exit

! เพิ่ม VLAN ใหม่เข้า Trunk ระหว่าง ACCESS-SW1 ↔ DIST-SW1
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# switchport trunk allowed vlan add 91,92,93
DIST-SW1(config-if)# exit
```

### Config ที่ ACCESS-SW1: เพิ่ม `authentication event` เข้า Interface Template จาก Step 434

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1-3
! --- Critical VLAN: เมื่อ ISE ทุกตัวใน Server Group ไม่ตอบสนอง ---
ACCESS-SW1(config-if-range)# authentication event server dead action authorize vlan 91
ACCESS-SW1(config-if-range)# authentication event server dead action authorize voice
ACCESS-SW1(config-if-range)# authentication event server alive action reinitialize
! --- Guest VLAN: ไม่มี Supplicant เลย (802.1X timeout หมด และ MAB ไม่ match) ---
ACCESS-SW1(config-if-range)# authentication event no-response action authorize vlan 92
! --- Auth-Fail VLAN: ISE ตอบ Access-Reject ชัดเจน (ลองซ้ำ 2 ครั้งก่อนยอมแพ้) ---
ACCESS-SW1(config-if-range)# authentication event fail retry 2 action authorize vlan 93
ACCESS-SW1(config-if-range)# exit
```

**อธิบายแต่ละบรรทัด:**
- `server dead action authorize vlan 91` — ตรวจจับผ่าน RADIUS Dead-Server Detection (Switch
  Mark Server เป็น Dead หลังไม่ตอบครบ `retransmit` รอบตามที่ตั้งไว้ใน Step 433) แล้ว **Authorize
  ทันที** เข้า VLAN 91 โดยไม่ต้องรอ Authenticate จริง — คู่กับ `server alive action reinitialize`
  ที่บอกให้ **ลอง Authenticate ใหม่ทันที** ที่ ISE กลับมาออนไลน์ (ไม่ต้องรอ Client Reboot/Unplug)
- `authorize voice` แยกไว้ต่างหาก เพราะ Multi-Domain Host Mode ต้องมี Policy สำหรับ Voice Domain
  ด้วย ไม่ใช่แค่ Data Domain — ถ้าไม่ใส่บรรทัดนี้ Phone จะไม่ได้ Voice VLAN ตอน ISE ล่ม
- `no-response` = ไม่มี EAPOL ตอบ **และ** MAB ไม่ Match เลย (คนละเคสกับ Auth-Fail)
- `fail retry 2` = ให้ Client พิมพ์ Password ผิดได้ 2 ครั้งก่อนเข้า Auth-Fail VLAN (กัน Fat-finger
  Typo ธรรมดาไม่ให้โดนกักทันทีในครั้งแรก)

### ตรวจสอบผลลัพธ์

```
ACCESS-SW1# show authentication sessions interface GigabitEthernet1/0/1 details | include Status|Vlan
                   Status:  Authorized
              Vlan Policy:  91          <- ตัวอย่างตอน ISE-1 ล่มชั่วคราว

! Log เมื่อ ISE-1 กลับมาออนไลน์
*Sep 26 09:12:03.114: %AUTHMGR-5-START: Starting 'dot1x' for client (0050.5601.aaaa) on
Interface Gi1/0/1 AuditSessionID 0A0A630B00000012345678
```

---

## Step 437 — Cisco TrustSec Fundamentals: SGT, Dynamic Assignment, Inline Tagging vs SXP

### ทวนจาก Part 38: SGT คืออะไร

[Part 38 Step 375](part-038-sd-access-fundamentals.md) แนะนำแนวคิดไว้สั้นๆว่า **Security Group Tag
(SGT)** คือ Tag ตัวเลข (16-bit, ค่า 2-65519 สำหรับใช้งานจริง) ที่ผูกกับ **"กลุ่ม" ของ Endpoint**
แทนการผูกกับ IP Address/Subnet — และบอกไว้ว่า "รายละเอียดเชิงลึกเต็มรูปแบบอยู่ที่ Part 44" นี่คือ
Part นั้น

```
แนวคิดหลัก: SGT แยก "ตัวตน (Identity/Group)" ออกจาก "ตำแหน่งบนเครือข่าย (IP/VLAN)" อย่างสิ้นเชิง

              วิธีเดิม (IP-based ACL — Part 14)              วิธีใหม่ (SGT-based SGACL — Step 438)
        ┌─────────────────────────┐             ┌─────────────────────────┐
        │  Policy ผูกกับ IP Subnet    │             │  Policy ผูกกับ SGT (กลุ่ม)   │
        │  "10.10.10.0/24 ห้ามคุย     │             │  "SALES_SGT ห้ามคุยกับ       │
        │   กับ 10.10.30.0/24"       │             │   SERVERS_SGT"              │
        └─────────────────────────┘             └─────────────────────────┘
        Endpoint ย้าย VLAN/Subnet             Endpoint ย้าย VLAN/Subnet ไปไหนก็ได้
        → Policy ต้องเขียนใหม่ทั้งหมด          → SGT ติดตัวไปด้วย ไม่ต้องแก้ Policy เลย
```

### SGT ที่จะใช้ตลอด Part นี้ (กำหนดขึ้นใหม่สำหรับ Lab — ไม่ผูกกับเลข VLAN โดยตั้งใจ)

| SGT (Decimal) | SGT (Hex) | ชื่อ | กลุ่มที่ครอบคลุม | วิธี Assign |
|---|---|---|---|---|
| 100 | 0x0064 | **SALES_SGT** | PC1-5 (VLAN 10, ผ่าน 802.1X) | Dynamic — ISE ส่งมาพร้อม Access-Accept |
| 110 | 0x006E | **VOICE_SGT** | Phone1-5 (VLAN 20, Voice Domain) | Dynamic — ISE ส่งมาพร้อม Access-Accept (Voice Domain) |
| 120 | 0x0078 | **SERVERS_SGT** | Server1-3 (VLAN 30) | **Static** — Server ไม่ทำ 802.1X เอง (ดูด้านล่าง) |
| 130 | 0x0082 | **PRINTER_SGT** | Printer1 (MAB) | Dynamic — ISE ส่งมาพร้อม Access-Accept ของ MAB |

> **จงใจเลือกเลข SGT ให้ไม่เหมือนเลข VLAN**: เพื่อตอกย้ำหลักการสำคัญที่สุดของ TrustSec — **SGT
> เป็นคนละ Namespace กับ VLAN โดยสิ้นเชิง** VLAN 10 (SALES) กับ SGT 100 (SALES_SGT) เป็นแนวคิด
> คนละชั้นที่บังเอิญ**คุมกลุ่มเดียวกัน**ในตอนนี้ แต่ถ้าอนาคตมีคนย้าย PC ไป VLAN อื่น (เช่น
> เสียบที่ ACCESS-SW3 แทน) SGT 100 ยังติดตัวไปเหมือนเดิมทันทีที่ Authenticate สำเร็จ

### SGT ถูก Assign แบบ Dynamic ได้อย่างไร: Cisco AV-Pair ใน RADIUS Access-Accept

เมื่อ PC1 Authenticate ผ่าน 802.1X สำเร็จ (Step 433-434) ISE-1 จะตอบ **Access-Accept** กลับมา
พร้อม **Cisco AV-Pair** พิเศษที่บอก SGT — Attribute ตัวนี้คือกลไกเดียวกับ `Tunnel-Private-Group-ID`
ที่ทำ Dynamic VLAN ใน [Part 37 Step 366](part-037-wireless-security-advanced.md) แต่เปลี่ยนมาใช้
Cisco AV-Pair เฉพาะของ TrustSec แทน

```
! Concept RADIUS Access-Accept ที่ ISE-1 ส่งกลับมาให้ ACCESS-SW1 ตอน PC1 Authenticate สำเร็จ
Access-Accept
  Tunnel-Private-Group-ID = 10                       (VLAN Assignment ปกติ)
  cisco-av-pair = "cts:security-group-tag=0064-01"    (SGT = 100 ในเลข Hex — SALES_SGT)
```

ACCESS-SW1 อ่าน Attribute นี้แล้วสร้าง **IP-SGT Binding** ในตารางภายในตัวเอง (ผูก IP ที่ Client
ได้จาก DHCP หลัง Authorized เข้ากับ SGT ที่ได้รับมา) — Binding นี้คือหัวใจของทุกอย่างที่จะเกิดขึ้น
ต่อจากนี้ ไม่ว่าจะเป็นการ Enforce SGACL (Step 438) หรือส่งผ่าน SXP (Step 439)

### Server1-3 ไม่ทำ 802.1X — ต้อง Assign SGT แบบ Static ที่ DIST-SW3

Server ส่วนใหญ่ (รวมถึง Server1-3 ใน Lab นี้) ไม่รัน 802.1X Supplicant และมักไม่เหมาะทำ MAB
เพราะ Downtime ระหว่างรอ Authenticate ไม่คุ้มกับความเสี่ยง — วิธีที่ใช้จริงคือ **Static IP-SGT
Mapping** ที่ทำที่ Switch ตัวที่เป็น SGT-capable โดยตรง (DIST-SW3 ในกรณีนี้ เพราะเป็น Gateway
ของ VLAN 30 อยู่แล้ว)

```
DIST-SW3(config)# cts role-based sgt-map 10.10.30.10 sgt 120
DIST-SW3(config)# cts role-based sgt-map 10.10.30.20 sgt 120
DIST-SW3(config)# cts role-based sgt-map 10.10.30.30 sgt 120
```

### 2 วิธีที่ SGT "เดินทาง" ข้ามอุปกรณ์: Inline Tagging vs SXP

| วิธี | หลักการ | ใช้ตอนไหน |
|---|---|---|
| **Inline Tagging (Native/Layer 2)** | แปะ SGT ลงใน **Cisco Meta Data (CMD)** field ของ Ethernet Frame โดยตรง บน Trunk ที่ทั้ง 2 ฝั่งเป็น **CTS-Capable Hardware** — SGT เดินทางไปกับ Data Plane จริงๆ ไม่ต้องมี Protocol แยก | Trunk ระหว่างอุปกรณ์ที่รองรับ CTS Hardware ทั้งคู่ (ในหลักสูตรนี้: **DIST-SW1-4 ↔ CORE-SW1/2**, Catalyst 9300 ↔ 9500 ทุกตัวรองรับ) |
| **SXP (SGT Exchange Protocol)** | ส่ง **IP-SGT Binding Table** (ไม่ใช่ตัว Tag ในทุก Packet) ผ่าน **TCP Port 64999** ไปให้อุปกรณ์ปลายทางที่ไม่มี Hardware รองรับ Inline Tagging — ปลายทางเก็บ Binding ไว้ แล้วใช้ตอน Lookup SGT จาก Source IP เอาเอง | Link ที่มีอุปกรณ์ไม่รองรับ CTS Hardware อยู่ฝั่งใดฝั่งหนึ่ง (ในหลักสูตรนี้: **ACCESS-SW1-4 ↔ DIST-SW1-4** — ACCESS-SW เป็น Catalyst 9200/2960 ที่ไม่มี Hardware Inline Tagging) |

```
เส้นทางการเดินทางของ SGT เต็ม Topology (Lab Part 44)

PC1 ──802.1X──► ACCESS-SW1 ──SXP (TCP 64999)──► DIST-SW1 ──Inline Tagging (CMD)──► CORE-SW1
(SALES_SGT=100  (เรียนรู้ Binding    (SXP Listener      (Trunk Native CTS      (Trunk Native CTS
 มาจาก ISE)      จาก 802.1X Session   รับ Binding มา       ทั้งสองฝั่งเป็น         ทั้งสองฝั่งเป็น
                 ของตัวเอง แล้ว        แล้ว "รู้" ว่า        Catalyst 9300)         Catalyst 9500)
                 Speak ออกไป)          10.10.10.11=100)
                                                                                        │
                                                                              CORE-SW2 ◄─┘
                                                                                 │ Inline Tagging
                                                                                 ▼
                                                                            DIST-SW3 (Enforcement
                                                                             Point — Step 438-440)
```

> **จุดสำคัญที่ต้องเข้าใจให้แม่น**: SXP **ไม่ใช่**การส่ง Tag ไปกับ Data Plane — มันเป็นแค่
> Control-Plane Protocol ที่บอก "IP นี้ = SGT อะไร" ให้อุปกรณ์ปลายทางรู้ไว้ล่วงหน้าเท่านั้น
> เมื่อ ACCESS-SW1 ส่ง Frame ธรรมดาไปให้ DIST-SW1 (ไม่มี CMD Tag ติดมา เพราะ ACCESS-SW1 ไม่มี
> Hardware ทำ Inline Tagging) DIST-SW1 จะ **Lookup Source IP ของ Frame นั้นในตาราง SXP Binding
> ที่เรียนมา** แล้วรู้ว่าต้องแปะ SGT อะไรต่อเมื่อส่งออก Trunk ที่เป็น Inline Tagging ต่อไป (ไปทาง
> CORE-SW1) — นี่คือเหตุผลที่ SXP ต้องมี Device-Tracking (Step 433) ทำงานถูกต้องเสมอ เพราะ SXP
> Binding มาจาก IP-SGT Table ที่ Device-Tracking สร้างไว้

---

## Step 438 — SGACL: แทนที่ IP ACL แบบ Part 14 ด้วย Policy ระดับ SGT-to-SGT

### ทวน ACL แบบเดิมจาก Part 14 ก่อนเทียบ

[Part 14 Step 140](part-014-access-control-lists.md) สร้าง Extended ACL ชื่อ `SALES-TO-SERVERS`
ไว้บน DIST-SW3 เพื่อจำกัด **VLAN 10 (SALES) เข้า Server1 ได้เฉพาะ HTTPS** โดย Match ด้วย
**IP Subnet** ตรงๆ:

```
! ACL เดิมจาก Part 14 (ยังใช้งานอยู่จริงถ้าไม่ได้ยกเลิก)
ip access-list extended SALES-TO-SERVERS
 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
 deny   ip  10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
```

**ปัญหาของวิธีนี้ที่ TrustSec แก้**: ถ้าวันหนึ่งบริษัทมี SALES Office สาขาใหม่ที่ใช้ Subnet
`10.20.10.0/24` (คนละ Subnet กับ VLAN 10 เดิม) ACL ชุดนี้ **ต้องเขียนเพิ่มบรรทัดใหม่ทันที**
มิฉะนั้น SALES สาขาใหม่จะเข้า SERVERS ไม่ได้เลย (หรือแย่กว่านั้นคือ ไม่ได้ Deny ตามที่ควร) —
ยิ่งองค์กรมี Subnet เยอะขึ้น ACL แบบนี้ยิ่ง**ระเบิดจำนวนบรรทัด**แบบ Quadratic (N กลุ่ม × M
Subnet ต่อกลุ่ม) จนดูแลไม่ไหว

### SGACL แก้ปัญหานี้อย่างไร: Match ด้วย SGT ไม่ใช่ IP

**SGACL (Security Group ACL)** คือ ACL ที่ไม่มี Source/Destination IP อยู่ในตัวมันเองเลย —
มันถูก "ผูก" เข้ากับคู่ **(Source SGT → Destination SGT)** แยกต่างหากในตาราง **SGT Matrix**
ทำให้ ACE ภายในสั้นและอ่านง่ายกว่าเดิมมาก

```
Router(config)# ip access-list role-based <name>
Router(config-rb-acl)# {permit|deny} <protocol> [operator port] [log]
```

> **สังเกต**: ไม่มีการระบุ Source/Destination Address เลยในบรรทัด ACE — เพราะ "Address"
> ในความหมายของ SGACL **คือคู่ SGT ที่ผูกไว้ต่างหาก** ไม่ใช่ IP

### สร้าง SGACL แทนที่ `SALES-TO-SERVERS` เดิม: `SALES-TO-SERVERS-SGACL`

**Requirement เดิมจาก Part 14** (SALES เข้า Server1 ได้เฉพาะ HTTPS/SSH) แปลงเป็น SGACL:

```
DIST-SW3(config)# ip access-list role-based SALES-TO-SERVERS-SGACL
DIST-SW3(config-rb-acl)# permit tcp dst eq 443
DIST-SW3(config-rb-acl)# permit tcp dst eq 22
DIST-SW3(config-rb-acl)# deny ip log
DIST-SW3(config-rb-acl)# exit
```

**เทียบ ACE ทีละบรรทัดกับต้นฉบับ Part 14:**

| Part 14 (IP ACL) | Part 44 (SGACL) | สังเกต |
|---|---|---|
| `permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443` | `permit tcp dst eq 443` | ไม่มี IP เลย — Source/Dest ถูกกำหนดจากคู่ SGT ที่ Map ไว้แทน |
| `deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log` | `deny ip log` | สั้นลงมาก เพราะ "SALES→SERVERS" คือความหมายของ SGT-pair ไปแล้ว ไม่ต้องพิมพ์ Subnet ซ้ำ |

### Map SGACL เข้ากับคู่ SGT ที่ต้องการ (SGT Matrix Cell)

```
DIST-SW3(config)# cts role-based permissions from 100 to 120 sgacl SALES-TO-SERVERS-SGACL
!                                    │        │
!                          SALES_SGT (Source)  SERVERS_SGT (Destination)
```

เพิ่ม Policy สำหรับ VOICE_SGT และ PRINTER_SGT เข้า SERVERS_SGT ด้วย (ทวน Requirement เดิมจาก
Part 14 ที่ VOICE เข้า SERVERS ได้เต็มที่เฉพาะ SIP 5060):

```
DIST-SW3(config)# ip access-list role-based VOICE-TO-SERVERS-SGACL
DIST-SW3(config-rb-acl)# permit tcp dst eq 5060
DIST-SW3(config-rb-acl)# permit udp dst eq 5060
DIST-SW3(config-rb-acl)# deny ip log
DIST-SW3(config-rb-acl)# exit
DIST-SW3(config)# cts role-based permissions from 110 to 120 sgacl VOICE-TO-SERVERS-SGACL

! PRINTER_SGT ไม่ควรคุยกับ SERVERS_SGT เลย (Printer ไม่มีเหตุผลต้องเข้า Server โดยตรง)
DIST-SW3(config)# ip access-list role-based DENY-ALL-SGACL
DIST-SW3(config-rb-acl)# deny ip log
DIST-SW3(config-rb-acl)# exit
DIST-SW3(config)# cts role-based permissions from 130 to 120 sgacl DENY-ALL-SGACL
```

### เปิด Enforcement (ถ้าไม่เปิดคำสั่งนี้ SGACL จะถูก Config ไว้เฉยๆ ไม่ Enforce จริง — เหมือน `dot1x system-auth-control` ใน Step 432)

```
DIST-SW3(config)# cts role-based enforcement
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# cts role-based enforcement
DIST-SW3(config-if)# exit
```

### สรุปประโยชน์เชิง Scalability — ตัวอย่างที่จับได้ชัดที่สุด

| สถานการณ์ | IP ACL (Part 14) ต้องทำอะไร | SGACL (Part 44) ต้องทำอะไร |
|---|---|---|
| PC1 ถูกย้ายจาก ACCESS-SW1 ไปเสียบที่ ACCESS-SW3 (คนละ Subnet/VLAN) | เขียน ACE ใหม่ทั้งหมดที่อ้าง Subnet ปลายทางใหม่ | **ไม่ต้องทำอะไรเลย** — SALES_SGT ติดตัว PC1 ไปตาม 802.1X Authenticate ใหม่ ณ พอร์ตใหม่ |
| SALES Office สาขาใหม่เปิด Subnet `10.20.10.0/24` | เพิ่ม ACE ใหม่ครอบ Subnet ใหม่ในทุก ACL ที่เกี่ยวข้อง | **ไม่ต้องทำอะไรเลย** — PC ที่สาขาใหม่ Authenticate แล้วได้ SALES_SGT เหมือนกัน Policy เดิมใช้ได้ทันที |
| เพิ่ม Endpoint กลุ่มใหม่ 20 กลุ่ม (ตาม Business ที่โตขึ้น) | ต้องออกแบบ Subnet ใหม่ 20 ชุด + เขียน ACL ใหม่ครอบทุกคู่ที่เกี่ยวข้อง | เพิ่ม SGT ใหม่ 20 ตัว + เขียน SGACL Policy ในตาราง Matrix เท่านั้น ไม่ต้องยุ่งกับ IP/VLAN เลย |

> ตรงกับที่ [Part 38 Step 375](part-038-sd-access-fundamentals.md) สรุปไว้ล่วงหน้าว่า **SGT
> ไม่ผูกกับ IP เลย — Endpoint ย้าย Subnet/VLAN ได้ SGT ยังติดตัวเหมือนเดิม** — Step นี้คือการ
> พิสูจน์ด้วย Config จริงว่าคำพูดนั้นแปลเป็นการปฏิบัติได้อย่างไร

---

## Step 439 — SXP Configuration: ส่ง SGT ไปยังอุปกรณ์ที่ไม่รองรับ Inline Tagging

### ทำไม ACCESS-SW1-4 ต้องใช้ SXP (ทวนจาก Step 437)

ตาม [Device Platform Reference](00-ip-address-plan.md) ACCESS-SW1-4 เป็น Catalyst 9200/2960
ที่ในหลักสูตรนี้ถือว่า **ไม่มี Hardware รองรับ Inline SGT Tagging** บน Uplink Trunk (ต่างจาก
DIST-SW1-4 และ CORE-SW1/2 ที่เป็น Catalyst 9300/9500 ซึ่งรองรับเต็มรูปแบบ) — SXP คือทางออก
เดียวที่ทำให้ SGT ที่ ACCESS-SW1 เรียนรู้จาก 802.1X/MAB **เดินทางต่อไปถึง DIST-SW1** ได้

### Syntax หลักของ SXP

```
Router(config)# cts sxp enable
Router(config)# cts sxp default password <shared-secret>
Router(config)# cts sxp default source-ip <local-ip>
Router(config)# cts sxp connection peer <peer-ip> password default mode {local|peer} {speaker|listener}
```

| Role | ความหมาย |
|---|---|
| **Speaker** | ฝั่งที่ **"มี" Binding อยู่แล้ว** (มาจาก 802.1X/MAB Session จริงบนพอร์ตของมัน) และส่งออกไปให้เพื่อน |
| **Listener** | ฝั่งที่ **"รับ" Binding** เข้ามาเก็บไว้ในตารางของตัวเอง เพื่อใช้ Enforce หรือส่งต่อด้วย Inline Tagging |

### Config: ACCESS-SW1 (Speaker) ↔ DIST-SW1 (Listener)

ใช้ Management SVI (VLAN 99) ที่มีอยู่แล้วตั้งแต่ [Part 3](part-003-ethernet-switching-vlan.md)
เป็น Source/Peer IP ของ SXP (ไม่ต้องสร้าง Interface ใหม่)

```
! ============ ACCESS-SW1 (10.10.99.11) — SXP Speaker ============
ACCESS-SW1(config)# cts sxp enable
ACCESS-SW1(config)# cts sxp default password TrustSec-SXP-Key2026!
ACCESS-SW1(config)# cts sxp default source-ip 10.10.99.11
ACCESS-SW1(config)# cts sxp connection peer 10.10.99.2 password default mode local speaker
ACCESS-SW1(config)# end
ACCESS-SW1# copy running-config startup-config
```

```
! ============ DIST-SW1 (10.10.99.2) — SXP Listener ============
DIST-SW1(config)# cts sxp enable
DIST-SW1(config)# cts sxp default password TrustSec-SXP-Key2026!
DIST-SW1(config)# cts sxp default source-ip 10.10.99.2
DIST-SW1(config)# cts sxp connection peer 10.10.99.11 password default mode local listener
DIST-SW1(config)# end
DIST-SW1# copy running-config startup-config
```

> **ต้องตั้ง Password ให้ตรงกันทั้ง 2 ฝั่งเสมอ** (`cts sxp default password`) — SXP ใช้ Password
> นี้เพื่อสร้าง MD5-signed TCP Session (คล้ายหลักการของ BGP MD5 Authentication ที่ [Part 29]
> (part-029-bgp-fundamentals.md) สอนไว้) ถ้า Password ไม่ตรงกัน Connection จะไม่ขึ้นเลย

### ทำ Pattern เดียวกันให้ครบทุกคู่ ACCESS-SW ↔ DIST-SW

| ACCESS-SW (Speaker) | DIST-SW (Listener) | Peer IP ที่ใช้ |
|---|---|---|
| ACCESS-SW1 (10.10.99.11) | DIST-SW1 (10.10.99.2) | ทำไปแล้วด้านบน |
| ACCESS-SW2 (10.10.99.12) | DIST-SW2 (10.10.99.3) | Config Pattern เดียวกัน |
| ACCESS-SW3 (10.10.99.13) | DIST-SW3 (10.10.99.4) | Config Pattern เดียวกัน — **สำคัญที่สุดสำหรับ Step 440** เพราะ DIST-SW3 คือ Enforcement Point |
| ACCESS-SW4 (10.10.99.14) | DIST-SW4 (10.10.99.5) | Config Pattern เดียวกัน |

### ตรวจสอบผลลัพธ์: `show cts sxp connections` และ `show cts sxp sgt-map`

```
DIST-SW1# show cts sxp connections
SXP                 : Enabled
Highest Version Supported: 4
Default Password    : Set
Default Source IP   : 10.10.99.2
Connection retry open period: 120 secs
Reconcile period: 120 secs
Retry open timer is not running

Peer IP       Source IP      Conn Status  Duration
10.10.99.11   10.10.99.2     On            2:15:33
                Peer Role: Speaker  Local Role: Listener

DIST-SW1# show cts sxp sgt-map
SXP Node ID(generated):0x0A0A6302(10.10.99.2)
IP-SGT Mappings as follows:
IPv4,SGT     : 10.10.10.11,100
IPv4,SGT     : 10.10.20.21,110
Total number of IP-SGT Mappings : 2
```

Binding `10.10.10.11 → SGT 100` และ `10.10.20.21 → SGT 110` ที่ DIST-SW1 เห็น **มาจาก ACCESS-SW1
ผ่าน SXP ทั้งหมด** — DIST-SW1 ไม่ได้รัน 802.1X กับ PC1/Phone1 เองเลย มันแค่ "เชื่อ" Binding
ที่ ACCESS-SW1 Speak มาให้ ซึ่งพิสูจน์หลักการ Separation of Concerns ระหว่าง **Authentication
Point** (ACCESS-SW1) กับ **Policy Enforcement Point** (DIST-SW3 ใน Step 440) ที่ TrustSec
ออกแบบมาให้แยกจากกันได้อย่างสมบูรณ์

---

## Step 440 — Lab เต็มรูปแบบ: 802.1X+MAB บน ACCESS-SW1-4 + TrustSec SGT + SXP + SGACL Enforcement

### Topology สรุปทั้งหมดของ Part นี้

```
                              ISE-1 (10.10.99.65) — Part 43
                                        │ RADIUS (UDP 1812/1813)
              ┌─────────────────────────┼─────────────────────────┐
              │                         │                         │
        ACCESS-SW1                ACCESS-SW3                ACCESS-SW4
     (802.1X Multi-Domain)      (802.1X+MAB Single-Host)   (MAB Single-Host)
        │Gi1/0/1-3│                 │Gi1/0/1-3│  │Gi1/0/4│    │Gi1/0/1-2│
     PC1-3+Phone1-3            Server1-3(static)  Printer1     AP-1/AP-2
     SALES_SGT=100             SERVERS_SGT=120    PRINTER_SGT   MAB→infra
     VOICE_SGT=110              (static mapping)    =130         profile
              │                         │                         │
        SXP Speaker               SXP Speaker               SXP Speaker
              │ TCP 64999                │ TCP 64999                │ TCP 64999
              ▼                          ▼                          ▼
         DIST-SW1                   DIST-SW3 ⭐                DIST-SW4
      SXP Listener              SXP Listener +              SXP Listener
                                 Enforcement Point
                                 (SGACL: SALES/VOICE/PRINTER → SERVERS)
              │                          │                          │
              └──────────Inline Tagging (CTS Native, Trunk)─────────┘
                          via CORE-SW1 <===LACP===> CORE-SW2
```

*(ACCESS-SW2 ↔ DIST-SW2 ใช้ Pattern เดียวกันกับ ACCESS-SW1 ↔ DIST-SW1 ทุกประการ — ไม่แสดงซ้ำ)*

### สรุป Config ที่ต้องทำครบทุกจุด (Checklist)

| # | จุดที่ Config | สิ่งที่ทำ | อ้างอิง Step |
|---|---|---|---|
| 1 | ACCESS-SW1-4 (Global) | `aaa new-model`, `radius server ISE-1`, `dot1x system-auth-control`, `device-tracking policy` | 433 |
| 2 | ACCESS-SW1/2 Gi1/0/1-3 | `authentication host-mode multi-domain` + `authentication event` (Critical/Guest/Auth-Fail) | 434, 436 |
| 3 | ACCESS-SW3 Gi1/0/4, ACCESS-SW4 Gi1/0/1-2 | `authentication order/priority dot1x mab` + `mab` | 435 |
| 4 | DIST-SW1 | เพิ่ม VLAN 91/92/93 + ACL `DOT1X-FALLBACK-RESTRICT` | 436 |
| 5 | ACCESS-SW1-4 ↔ DIST-SW1-4 | `cts sxp enable` + `cts sxp connection peer ... speaker` (ACCESS) / `... listener` (DIST) | 439 |
| 6 | DIST-SW3 | `cts role-based sgt-map` (Static SERVERS_SGT), SGACL x3, `cts role-based permissions from ... to ...`, `cts role-based enforcement` | 437, 438 |
| 7 | DIST-SW1-4 ↔ CORE-SW1/2 | Trunk Native เปิด Inline Tagging (Default บน Catalyst 9300/9500 ที่เปิด `cts sxp`/`cts role-based` แล้ว ไม่ต้อง Config เพิ่ม) | 437 |

### Verification เต็มรูปแบบ — 4 คำสั่งหลักที่ต้องรู้ผลลัพธ์ให้ขึ้นใจ

**1) `show authentication sessions` — ยืนยันว่า Endpoint ทุกตัว Authorized ถูก Domain/VLAN**

```
ACCESS-SW1# show authentication sessions
Interface    MAC Address     Method   Domain   Status       Fg   Session ID
Gi1/0/1      0050.5601.aaaa  dot1x    DATA     Authorized        0A0A630B00000012
Gi1/0/1      0050.5602.bbbb  dot1x    VOICE    Authorized        0A0A630B00000013
Gi1/0/2      0050.5603.cccc  dot1x    DATA     Authorized        0A0A630B00000014
Gi1/0/2      0050.5604.dddd  dot1x    VOICE    Authorized        0A0A630B00000015

ACCESS-SW3# show authentication sessions
Interface    MAC Address     Method   Domain   Status       Session ID
Gi1/0/4      001a.a0aa.1234  mab      DATA     Authorized   0A0A630D00000021
```

**2) `show cts role-based sgt-map` — ยืนยันว่าทุก IP มี SGT ผูกถูกต้อง (บน DIST-SW3 ซึ่งเห็นทั้งฝั่ง Static และฝั่ง SXP)**

```
DIST-SW3# show cts role-based sgt-map all
Active IP-SGT Bindings Information

IP Address              SGT     Source
============================================
10.10.10.11              100     SXP
10.10.10.12              100     SXP
10.10.20.21              110     SXP
10.10.30.10              120     CLI (Static)
10.10.30.20              120     CLI (Static)
10.10.30.30              120     CLI (Static)
10.10.30.40              130     SXP

IP-SGT Active Bindings Summary
============================================
Total number of CLI bindings    = 3
Total number of SXP bindings    = 4
Total number of active bindings = 7
```

**3) `show cts role-based permissions` — ยืนยัน SGACL Matrix ที่ตั้งไว้ครบ**

```
DIST-SW3# show cts role-based permissions
IPv4 Role-based permissions default:
    Permit IP-00

From SALES_SGT(100) to SERVERS_SGT(120):
    SALES-TO-SERVERS-SGACL-00

From VOICE_SGT(110) to SERVERS_SGT(120):
    VOICE-TO-SERVERS-SGACL-00

From PRINTER_SGT(130) to SERVERS_SGT(120):
    DENY-ALL-SGACL-00
```

**4) `show cts role-based counters` — ยืนยันว่า Policy ทำงานจริง (มี Match Counter หลังทดสอบ Traffic)**

```
DIST-SW3# show cts role-based counters
Role-based IPv4 counters
From    To      SW-Denied      SW-Permitted    HW-Denied    HW-Permitted
100     120     18             1               0            0
110     120     0              1               0            0
130     120     6              0               0            0
*       *       0              48210           0            0
```

**อ่านผลลัพธ์**: `From 100 To 120` (SALES→SERVERS) มี **18 SW-Denied** (การพยายามเข้าถึง Port
อื่นที่ไม่ใช่ 443/22 ถูก Deny ตาม SGACL) และ **1 SW-Permitted** (HTTPS ผ่านสำเร็จ) — ตรงกับ
Test Case แบบเดียวกับที่ [Part 14 Step 140](part-014-access-control-lists.md) ทำไว้ แต่ตอนนี้
Policy Enforce ด้วย **SGT-Pair** แทน IP Subnet ทั้งหมดแล้ว — `From 130 To 120` (PRINTER→SERVERS)
มี 6 SW-Denied และ 0 SW-Permitted ยืนยันว่า Printer1 ถูกบล็อกไม่ให้เข้า Server เลยตามที่ตั้งใจ

### Test เปรียบเทียบ: ปิด SXP ชั่วคราวเพื่อพิสูจน์ว่า Policy พังจริงถ้าไม่มี SXP

```
ACCESS-SW1(config)# interface GigabitEthernet1/0/24
ACCESS-SW1(config-if)# shutdown          ! จำลอง Link ACCESS-SW1↔DIST-SW1 ขาด (SXP Connection หลุดตามไปด้วย)

DIST-SW1# show cts sxp connections | include Status
Connection Status: Off (was On)

DIST-SW3# show cts role-based sgt-map all | include 10.10.10.11
! ไม่มี Entry นี้อีกต่อไป — Binding หายไปจาก DIST-SW3 เพราะ SXP Chain ขาดตอน

! ผลลัพธ์: Traffic จาก PC1 ที่มาถึง DIST-SW3 จะถูกมองเป็น SGT "Unknown" (0) โดย Default
! → นโยบายที่ไม่ได้ Match บรรทัดไหนใน Matrix เลยจะถูก Deny โดย Default Policy (Deny IP)
! (พิสูจน์ว่า SXP เป็น "จุดเดียว" ที่ทำให้ Policy Enforcement ทำงานได้ต่อเนื่องจาก Edge ถึง Core)

ACCESS-SW1(config-if)# no shutdown       ! คืนสถานะปกติหลังทดสอบ
```

---

## แบบฝึกหัดทวนความเข้าใจ Part 44

1. ทำไม Multi-Auth Host Mode จึงไม่เหมาะกับพอร์ตที่มี IP Phone + PC ต่อพ่วงกัน แม้จะรองรับ
   Endpoint ได้หลายตัวเหมือนกับ Multi-Domain?
2. อธิบายความแตกต่างระหว่าง `authentication order dot1x mab` กับ `authentication priority
   dot1x mab` — คำสั่งไหนตัดสินใจ "ลองอะไรก่อน" และคำสั่งไหนตัดสินใจ "เชื่อผลของอะไรมากกว่า"?
3. Endpoint ตัวหนึ่งเสียบสายเข้า ACCESS-SW3 แล้วไม่มี IP เลยแม้จะรอนาน — `show authentication
   sessions` แสดง Status เป็น `Authorized` และ `Vlan Policy: 91` จงอธิบายว่าเกิดอะไรขึ้น และ
   นี่ถือเป็น Behavior ที่ถูกต้องหรือไม่ ทำไม?
4. เพราะเหตุใด SGACL ที่สร้างใน Step 438 (`permit tcp dst eq 443`) จึงไม่มีการระบุ Source/
   Destination IP Address เลย ต่างจาก IP ACL ใน Part 14 อย่างไร และอะไรคือสิ่งที่ทำหน้าที่
   "แทน" IP Address ใน SGACL?
5. ทำไม ACCESS-SW1-4 ต้องใช้ SXP ในการส่ง SGT ไปยัง DIST-SW1-4 แต่ DIST-SW1-4 ↔ CORE-SW1/2
   ไม่ต้องใช้ SXP เลย?

**เฉลย:**

1. Multi-Auth มองทุก Endpoint บนพอร์ตเป็น "ระดับเดียวกัน" ไม่มีแนวคิด Voice/Data Domain แยกกัน
   ทำให้ Switch ไม่รู้ว่าใครควรได้ VLAN 10 (Data) และใครควรได้ VLAN 20 (Voice) อย่างแน่นอน
   ต้องพึ่ง Attribute พิเศษจาก RADIUS ในการแยกเอง เสี่ยงผิดพลาด ส่วน Multi-Domain ผูก "Domain"
   เข้ากับ CDP/LLDP-MED Voice VLAN Discovery ที่ Phone ประกาศตัวเองอยู่แล้ว ทำให้แยก Data/Voice
   Domain ได้แม่นยำและจำกัดไว้แค่ 1 Data + 1 Voice ต่อพอร์ตพอดี ตรงกับ Use Case ของ IP Phone
2. `authentication order dot1x mab` กำหนด**ลำดับที่ Switch จะลอง**วิธี Authenticate (ลอง 802.1X
   ก่อน ถ้า Timeout ไม่มี Response ค่อย Fallback ไป MAB) ส่วน `authentication priority dot1x
   mab` กำหนดว่า**ถ้าทั้งสองวิธีสำเร็จแข่งกัน** (เช่น MAB สำเร็จไปก่อนแล้ว Supplicant ค่อยส่ง
   EAPOL มาทีหลัง) ผลของวิธีไหนจะ "ชนะ" และแทนที่ผลเดิม — Order คือ Sequence, Priority คือ
   Trust Ranking เป็นคำสั่งคนละหน้าที่กันที่มักตั้งค่าเหมือนกันแต่ไม่ใช่คำสั่งเดียวกัน
3. Endpoint นี้ถูก Authorize เข้า **Critical VLAN (91)** เพราะ Switch ตรวจพบว่า **ISE-1 (RADIUS
   Server) ไม่ตอบสนองเลย** (Dead Server Detection ทำงาน) จึงใช้ `authentication event server
   dead action authorize vlan 91` Authorize เข้า VLAN สำรองแบบจำกัดสิทธิ์ทันทีโดยไม่รอ ISE — ถือ
   เป็น Behavior ที่ถูกต้องตามการออกแบบ (Graceful Degradation) เพราะดีกว่าปิดพอร์ตทั้งหมดจนกว่า
   ISE จะกลับมาออนไลน์ ซึ่งจะทำให้ทั้งบริษัทเข้าเครือข่ายไม่ได้เพียงเพราะ Server ตัวเดียวล่ม —
   ปัญหาที่แท้จริงคือต้องไปตรวจสอบว่าทำไม ISE-1 ถึงเข้าไม่ถึง ไม่ใช่ที่พอร์ตนี้
4. SGACL ไม่ระบุ IP Address เพราะมันไม่ได้ผูกกับ IP/Subnet เลยตามหลักการของ TrustSec — สิ่งที่
   ทำหน้าที่ "แทน" IP Address คือ **คู่ SGT (Source SGT → Destination SGT)** ที่ถูก Map ไว้
   แยกต่างหากด้วยคำสั่ง `cts role-based permissions from <src-sgt> to <dst-sgt> sgacl <name>`
   — ACE ภายใน SGACL จึงมีแค่ Protocol/Port เท่านั้น เพราะ "ใครคุยกับใคร" ถูกตัดสินไปแล้วตั้งแต่
   ขั้นตอน Mapping SGT-Pair ก่อนที่จะมาถึงการตรวจ ACE เลย ต่างจาก IP ACL ของ Part 14 ที่ต้อง
   ระบุ Source/Destination Address ทุกบรรทัดเพราะ Policy ผูกกับ IP โดยตรง
5. เพราะ ACCESS-SW1-4 เป็น Catalyst 9200/2960 ที่ไม่มี Hardware รองรับ Inline SGT Tagging
   (ไม่สามารถแปะ Cisco Meta Data ลงใน Ethernet Frame ได้) จึงต้องใช้ SXP ส่ง IP-SGT Binding
   Table ผ่าน Control-Plane (TCP 64999) แทน ในขณะที่ DIST-SW1-4 และ CORE-SW1/2 ทั้งหมดเป็น
   Catalyst 9300/9500 ที่รองรับ CTS Hardware เต็มรูปแบบ จึงสามารถแปะ SGT ลงใน Data Plane
   จริงผ่าน Inline Tagging บน Trunk ได้โดยตรง ไม่ต้องพึ่ง Protocol แยกอย่าง SXP อีก

---

## สรุป Part 44

Part นี้นำ **802.1X Framework** จาก [Part 37](part-037-wireless-security-advanced.md) มาใช้กับ
**สายแลนบน ACCESS-SW1-4** เต็มรูปแบบ ตั้งแต่ทวนบทบาท Supplicant/Authenticator/Authentication
Server ในบริบท Wired (Step 431), เจาะลึก Port State (Unauthorized/Authorized) และเปรียบเทียบ
Syntax IBNS 1.0/2.0 (Step 432), คอนฟิกจริงชี้ไปที่ **ISE-1** ที่ [Part 43]
(part-043-security-architecture-firepower-ise.md) ติดตั้งไว้ (Step 433), เลือก **Host Mode
Multi-Domain** ให้พอร์ตที่มี IP Phone+PC (Step 434), เพิ่ม **MAB** เป็น Fallback สำหรับ Printer/
AP ที่ไม่มี Supplicant (Step 435), และปิดท้ายด้วย **Critical/Guest/Auth-Fail VLAN** เพื่อให้
เครือข่ายยัง Graceful Degrade ได้เมื่อ ISE ล่มหรือ Auth ไม่ผ่าน (Step 436)

ครึ่งหลังของ Part นี้เปิดกล่อง **Cisco TrustSec** ที่ [Part 38](part-038-sd-access-fundamentals.md)
แง้มไว้ตั้งแต่แรก — **SGT** ที่ ISE แจกแบบ Dynamic ผ่าน 802.1X/MAB (Step 437), **SGACL**
ที่แทนที่ IP ACL แบบ `SALES-TO-SERVERS` ของ [Part 14](part-014-access-control-lists.md) ด้วย
Policy ระดับ SGT-to-SGT ที่ไม่ต้องแก้เมื่อ IP/Subnet เปลี่ยน (Step 438), **SXP** สำหรับส่ง SGT
ผ่าน Link ที่ไม่รองรับ Inline Tagging (Step 439), และ Lab เต็มรูปแบบที่พิสูจน์ทุกอย่างทำงาน
ร่วมกันจริงด้วย `show authentication sessions`, `show cts role-based sgt-map`, และ
`show cts role-based counters` (Step 440)

หลักการ Micro-Segmentation ด้วย SGT ที่เราเพิ่งสร้างเสร็จนี้ยังทำงานคู่กับ ACL แบบเดิมของ
[Part 14](part-014-access-control-lists.md) อยู่ — คำถามที่ยังไม่ได้ตอบคือ ACL แบบ Stateless
ธรรมดายังพอไหมสำหรับ Security Policy ระดับ Enterprise ในปี 2026 หรือต้องมี Feature ที่ฉลาดกว่า
เช่น การ Inspect Traffic แบบ Stateful หรือแบ่ง Zone ทั้งเครือข่ายออกเป็นโซนความปลอดภัยชัดเจน —
**Part 45 — Advanced ACL & Zone-Based Firewall** จะพา ACL ที่เรารู้จักมาตั้งแต่ Part 14 ก้าวข้าม
ขีดจำกัดเดิม ไปสู่ Feature ระดับ CCNP ที่ใช้งานจริงในองค์กรขนาดใหญ่

**ไปต่อ:** [Part 45 — Advanced ACL & Zone-Based Firewall →](part-045-advanced-acl-zbfw.md)
