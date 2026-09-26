# Part 36 — Wireless Architecture Deep Dive (CAPWAP, Roaming)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 351–360 จาก 1000**

> ต่อจาก [Part 35 — Advanced FHRP & Campus HA →](part-035-advanced-fhrp-campus-ha.md) ที่ปิดท้าย
> เรื่อง High Availability ของ Campus Network ระดับ Wired (FHRP, Dual-homed Distribution) Part 36
> นี้จะพา Wireless LAN ที่เริ่มไว้ใน **Part 20 (ระดับ CCNA)** ให้ลงลึกถึงระดับ **CCNP ENCOR**:
> กลไกจริงของ CAPWAP (DTLS, Fragmentation, Keepalive), FlexConnect สำหรับ Branch Office, RF
> Design เชิงวิศวกรรม (RSSI/SNR/RRM), Client Roaming แบบ L2/L3 พร้อม Mobility Group ระหว่าง WLC
> หลายตัว, Fast Roaming Protocol (802.11r/k/v) สำหรับ VoWLAN, และ High Availability ของ WLC เอง
> (N+1 และ AP SSO) — ทุก Step ในนี้คือเนื้อหา**ใหม่ทั้งหมด** ที่ Part 20 ยังไม่ได้แตะ
>
> **หมายเหตุสำคัญเรื่อง Scope**: เพื่อสอน Mobility Group และ WLC High Availability ให้ครบ
> ต้องมี "WLC ตัวที่สอง" เข้ามาในสมการ — Part นี้จึงสมมติเพิ่มอุปกรณ์ 2 ตัวเข้ามา **เฉพาะในบริบท
> ของ Part 36 เท่านั้น**: **WLC-1-STANDBY** (คู่ HA ของ WLC-1 ที่ Campus เดิม) และ **WLC-2**
> (ตัวแทน Controller ของ "อาคารที่สอง" สำหรับสอน Mobility Group) ที่อยู่นอกเหนือ
> `docs/00-ip-address-plan.md` (ไม่มีการแก้ไขเอกสารนั้น) — VLAN/Subnet ที่ใช้กับอุปกรณ์ทั้งสองนี้
> (VLAN 199, 140 และ Link 10.10.253.0/30) เป็น Placeholder สำหรับสอนแนวคิดเท่านั้น ส่วน Lab หลัก
> ของหลักสูตร (WLC-1, AP-1, AP-2, VLAN 40/99) ยังคงเป็นไปตาม Canonical Plan ทุกจุด

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 351 | CAPWAP Protocol Deep Dive: DTLS, Control/Data Tunnel, Fragmentation, Keepalive |
| 352 | FlexConnect Deep Dive: Local vs Central Switching และแปลง AP-2 เป็น Branch AP |
| 353 | RF Fundamentals for Design: RSSI, SNR, Cell Overlap, RRM (DCA/TPC) |
| 354 | Client Roaming Deep Dive: L2/L3 Roaming และ Mobility Group กับ WLC-2 |
| 355 | Fast Roaming Protocols: 802.11r (FT), 802.11k, 802.11v |
| 356 | WLC High Availability: N+1 HA และ AP SSO กับ WLC-1-STANDBY |
| 357 | Multi-WLC Mobility Architecture at Scale: Mobility Controller vs Mobility Agent |
| 358 | RF Profiles และ AP Groups สำหรับแยก RF Setting ตามอาคาร/ชั้น |
| 359 | Verification & Troubleshooting เชิงลึก |
| 360 | Lab เต็มรูปแบบ: FlexConnect + 802.11r + WLC HA/AP SSO พร้อม Roaming Test |

---

## Step 351 — CAPWAP Protocol Deep Dive: DTLS, Control/Data Tunnel, Fragmentation, Keepalive

### ทวนโครงสร้าง CAPWAP จาก Part 20 แบบสั้นๆ ก่อนลงลึก

Part 20 (Step 192, 194) สอนไปแล้วว่า CAPWAP แยกเป็น **Control tunnel (UDP 5246)** และ
**Data tunnel (UDP 5247)** และ Control tunnel เข้ารหัสด้วย DTLS เสมอ — Part นี้จะอธิบาย
**กลไกภายใน** ของแต่ละส่วนแบบละเอียดถึงระดับ Packet

### DTLS (Datagram Transport Layer Security) — ทำไมต้องเป็น DTLS ไม่ใช่ TLS ธรรมดา

TLS ทั่วไป (ที่ใช้กับ HTTPS) วิ่งอยู่บน TCP ซึ่งมี Retransmission/Ordering ในตัวอยู่แล้ว แต่ CAPWAP
วิ่งอยู่บน **UDP** (เพื่อ Latency ต่ำ เหมาะกับ Real-time traffic อย่างเสียง/วิดีโอ) — DTLS คือ
เวอร์ชันของ TLS ที่ถูกดัดแปลงให้ทำงานบน UDP ได้ (เพิ่ม Sequence Number และ Retransmission
Timer ของตัวเองเข้าไป เพราะ UDP ไม่มีให้)

### DTLS Handshake ระหว่าง AP กับ WLC (ตอน Join)

```
AP (Client role ของ DTLS)                          WLC-1 (Server role ของ DTLS)
      │                                                    │
      │──── 1. ClientHello (Cipher Suite ที่รองรับ) ───────►│
      │◄─── 2. ServerHello + Certificate (WLC) ────────────│
      │         + ServerKeyExchange + CertificateRequest    │
      │──── 3. Certificate (AP - ใช้ MIC*) ─────────────────►│
      │      + ClientKeyExchange + CertificateVerify        │
      │      + ChangeCipherSpec + Finished                  │
      │◄─── 4. ChangeCipherSpec + Finished ─────────────────│
      │                                                    │
      │======= DTLS Tunnel พร้อมใช้งาน (Encrypted) =========│
      │──── 5. CAPWAP Join Request (ผ่าน Tunnel) ──────────►│
      │◄─── 6. CAPWAP Join Response ────────────────────────│
```

> **MIC (Manufacturer Installed Certificate)** — AP ทุกตัวจาก Cisco มี Certificate ติดตัวมาจาก
> โรงงาน (คล้าย SSL Certificate ของเว็บ) ใช้ยืนยันตัวตนกับ WLC ว่าเป็น AP ของ Cisco แท้ ไม่ใช่
> อุปกรณ์ปลอม — นี่คือเหตุผลที่ **NTP ต้อง sync กันระหว่าง AP กับ WLC เสมอ** (ตามที่เตือนไว้ใน
> Part 20 Step 199) เพราะ Certificate มีวันหมดอายุ (Validity Period) ต้องเทียบเวลาปัจจุบันด้วย

### Cipher Suite ที่ CAPWAP DTLS ใช้ (IOS-XE ปัจจุบัน)

| Cipher Suite | ระดับความปลอดภัย | เริ่มรองรับ |
|---|---|---|
| TLS_RSA_WITH_AES_128_CBC_SHA | มาตรฐาน (Default เดิม) | ทุกเวอร์ชัน |
| TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 | สูง (Perfect Forward Secrecy) | IOS-XE 17.x ขึ้นไป |

```
! ดู Cipher Suite ที่ WLC-1 เสนอให้ AP ปัจจุบัน
WLC-1# show wireless management interface | include Cipher
```

### Control Tunnel vs Data Tunnel — เจาะรายละเอียด Message Type

| Tunnel | Message Type หลักที่วิ่งอยู่ใน Tunnel | DTLS |
|---|---|---|
| **Control (UDP 5246)** | Discovery Req/Resp, Join Req/Resp, Configuration Status/Update Request, WTP Event Request (AP รายงานสถานะ), Echo Request/Response | **บังคับเข้ารหัสเสมอ ปิดไม่ได้** |
| **Data (UDP 5247)** | 802.11 Data Frame ที่ Encapsulate เป็น CAPWAP Data Payload (Client traffic จริง) | **Option** — เปิด/ปิดได้ |

การเปิด DTLS บน **Data tunnel** เพิ่ม CPU Overhead ทั้งฝั่ง AP และ WLC (ต้อง Encrypt/Decrypt
ทุก Client Packet) — ค่า Default ของ Cisco คือ **ปิด Data DTLS** เพราะถือว่า VLAN 99 (MGMT)
ที่ CAPWAP วิ่งอยู่เป็น Network ภายในที่ควบคุมได้ (Trusted) แต่ถ้า AP ต้องวิ่งผ่าน Network ที่ไม่
น่าเชื่อถือ (เช่น Internet สำหรับ Remote Teleworker AP) **ควรเปิด Data DTLS** เสมอ

```
! เปิด CAPWAP Data DTLS Encryption ผ่าน AP Join Profile (คำสั่งจริงบน Catalyst 9800)
WLC-1(config)# ap profile default-ap-profile
WLC-1(config-ap-profile)# link-encryption
WLC-1(config-ap-profile)# exit
```

### CAPWAP Fragmentation — ทำไม MTU มีผลกับ Wireless

CAPWAP เพิ่ม Header ทับ Frame เดิมของ Client หลายชั้น: IP Header (Outer) + UDP Header +
CAPWAP Header + (DTLS Header ถ้าเปิด) + 802.11 Frame ของ Client เอง — รวมแล้วเพิ่ม Overhead
ประมาณ **36-60 byte** ต่อ Packet

```
Ethernet MTU ปกติ = 1500 byte
Client ส่ง Frame ขนาดใกล้ 1500 byte (เช่น TCP MSS เต็มขนาด)
        │
        ▼
AP ห่อ CAPWAP Header + IP/UDP Header ทับ → รวมอาจเกิน 1500 byte
        │
        ▼
ถ้า Path ระหว่าง AP ↔ WLC มี MTU = 1500 พอดี → CAPWAP ต้อง Fragment เป็น 2 IP Packet
        │
        ▼
ผลกระทบ: Throughput ลดลง (Overhead ของการ Fragment/Reassemble), Latency เพิ่ม,
         และถ้า Router/Firewall กลางทาง Drop Fragment (บางองค์กร Block ด้วยเหตุผล Security)
         → Client จะเจอปัญหา Throughput ต่ำผิดปกติแบบอธิบายไม่ได้
```

**วิธีแก้ที่ Cisco แนะนำ**: เพิ่ม MTU บน Interface ตลอด Path ระหว่าง AP กับ WLC เป็นอย่างน้อย
**1550-1600 byte** (ถ้า Switch/Router รองรับ Jumbo Frame หรือ System MTU ปรับได้) เพื่อให้
CAPWAP ห่อ Frame เต็มขนาดได้โดยไม่ต้อง Fragment

```
! ตรวจสอบ Fragmentation Statistics บน WLC-1 (ถ้าค่าสูงผิดปกติ = MTU มีปัญหาใน Path)
WLC-1# show wireless stats capwap fragmentation

! ปรับ System MTU บน Distribution Switch (ถ้ารองรับ Jumbo Frame ตลอด Path)
DIST-SW3(config)# system mtu 9198
```

### CAPWAP Keepalive (Echo Request/Response) — กลไกตรวจสอบว่า WLC ยังออนไลน์อยู่

AP ส่ง **CAPWAP Echo Request** ไปที่ WLC ทุก Interval ที่กำหนด (Default 30 วินาที) ผ่าน
Control Tunnel — ถ้า WLC ไม่ตอบภายในเวลาที่กำหนด AP จะลองส่งซ้ำตาม Retry Count (Default
สูงสุด 5 ครั้ง) ก่อนจะประกาศว่า WLC "Unreachable" แล้วเข้าสู่กระบวนการ Discovery ใหม่เพื่อหา
WLC ตัวอื่น (Backup Controller ถ้ามีตั้งไว้ — ดู Step 356)

```
! ปรับ Keepalive Timer และ Discovery Timeout (Global, มีผลกับทุก AP)
WLC-1(config)# ap capwap timers heartbeat-timeout 30
WLC-1(config)# ap capwap timers discovery-timeout 10
```

| Parameter | ค่า Default | ความหมาย |
|---|---|---|
| Heartbeat Interval | 30 วินาที | ความถี่ที่ AP ส่ง Echo Request |
| Heartbeat Timeout | 5 x Interval (~150 วินาที) | เวลารวมก่อนประกาศ WLC Unreachable |
| Discovery Timeout | 10 วินาที | เวลารอ Response ระหว่างขั้นตอน Discovery |

> **ข้อสอบเจาะจง CCNP**: การเพิ่ม Heartbeat Interval ให้สูงขึ้นจะทำให้ AP **ตรวจจับ WLC ล่มได้
> ช้าลง** (Detection Time นานขึ้น) แต่ลด Control-plane Traffic บน Network — เป็น Trade-off
> ระหว่างความเร็วในการ Failover กับ Overhead ของ Network เอง

---

## Step 352 — FlexConnect Deep Dive: Local vs Central Switching และแปลง AP-2 เป็น Branch AP

### ทวน Centralized Mode จาก Part 20 แล้วเจาะ FlexConnect ต่อ

Part 20 (Step 193) บอกไว้สั้นๆ ว่า Centralized Mode เหมาะกับ AP ที่อยู่ Campus เดียวกับ WLC
ส่วน FlexConnect เหมาะกับ Branch Office — Part นี้จะลงคอนฟิกจริงเต็มรูปแบบ

### Local Switching vs Central Switching — เปรียบเทียบระดับ Per-WLAN

FlexConnect ไม่ใช่ All-or-nothing — สามารถกำหนด **ต่อ WLAN** ได้ว่า WLAN ไหนใช้ Local
Switching (Data ออก Local ที่ AP) และ WLAN ไหนยังใช้ Central Switching (Data วิ่งกลับไป WLC)
แม้ AP ตัวนั้นเป็น FlexConnect Mode แล้วก็ตาม

| คุณสมบัติ | Central Switching (บน FlexConnect AP) | Local Switching |
|---|---|---|
| Client Data Path | AP → CAPWAP Data Tunnel → WLC → VLAN ปลายทาง | AP switch ตรงออก Local Access Switch เลย ไม่ผ่าน WLC |
| Authentication | Central (WLC ตัดสินใจ RADIUS/PSK) หรือ Local ก็ได้ | มักใช้ Local Authentication คู่กัน (ลด Dependency ต่อ WAN) |
| เหมาะกับ WLAN ประเภทไหน | SSID ที่ต้องการ Policy รวมศูนย์เข้มงวด (เช่น Guest ที่ต้อง Anchor) | SSID ที่ Client ใช้งาน Local Resource เป็นหลัก (Printer, File Server ในสาขา) |
| ผลถ้า WAN ล่ม | Client หลุดทั้งหมด (Data Plane ขาด) | Client ยังใช้ Local Resource ได้ต่อ (เข้าสู่ "Standalone Mode") |

### Use Case จริง: ทำไม Branch Office ต้องใช้ Local Switching

Branch Office ทั่วไปมี WAN Link ไปยัง Central Site (Data Center/HQ) ที่ **Bandwidth จำกัดและ
Latency สูง** กว่า LAN ภายใน — ถ้า AP ที่ Branch เป็น Centralized Mode ทุก Client Packet
(รวมถึง Traffic ที่ปลายทางอยู่ใน Branch เดียวกัน เช่น Print Job ไปยัง Printer ข้างห้อง) ต้อง
วิ่งผ่าน WAN ไป WLC ที่ Central Site ก่อน แล้วจึงถูก Route กลับมาที่ Branch อีกที (Traffic
"Hair-pinning" ผ่าน WAN 2 รอบ) — สิ้นเปลือง Bandwidth WAN และเพิ่ม Latency โดยไม่จำเป็น
**FlexConnect + Local Switching** แก้ปัญหานี้: AP ที่ Branch จะ Decapsulate 802.11 Frame แล้ว
Switch ออกเข้า VLAN ท้องถิ่นทันที โดยไม่ต้องพึ่ง WLC สำหรับ Data Plane เลย (WLC ยังคุม Control
Plane ผ่าน CAPWAP Control Tunnel อยู่เหมือนเดิม — แค่ Data Tunnel ที่หายไป)

### สถานการณ์ Lab: สมมติ ACCESS-SW4/DIST-SW4 (ที่ AP-2 เสียบอยู่) คือ "อาคารสาขา"

ในสถานการณ์นี้ เราจะสมมติว่า **DIST-SW4/ACCESS-SW4 ที่ AP-2 เสียบอยู่** ถูกย้ายไปตั้งอยู่ที่
**สาขา (Branch Office)** ที่เชื่อมต่อกลับมายัง Campus หลัก (ที่ WLC-1 อยู่) ผ่าน WAN Link เดิม
ที่มีอยู่แล้วตั้งแต่ Part 19 (ผ่าน CORE-SW2 ↔ WAN-EDGE-2) — AP-2 จึงควรเปลี่ยนเป็น FlexConnect
Mode พร้อม Local Switching สำหรับ VLAN 40 เพื่อไม่ให้ Client Data ต้อง Hair-pin ผ่าน WAN

```
Before (Part 20): Centralized Mode                After (Part 36): FlexConnect + Local Switching

Client → AP-2 → CAPWAP Data → WLC-1               Client → AP-2 → Switch Local ทันที
         (ผ่าน WAN ทุก Packet)                              (ไม่ผ่าน WAN เลย สำหรับ Data)
         → VLAN 40 ที่ WLC-1                                 → VLAN 40 ที่ ACCESS-SW4/DIST-SW4
                                                              (Control Plane เท่านั้นที่ยังวิ่งผ่าน
                                                               WAN ไปคุยกับ WLC-1 - CAPWAP Control)
```

> ตาม Access-Layer Mapping ใน `docs/00-ip-address-plan.md` นั้น ACCESS-SW4 Trunk VLAN 30, 40,
> 99 ไปยัง DIST-SW4 อยู่แล้ว และ DIST-SW4 มี SVI VLAN 40 (HSRP Secondary จาก Part 18) พร้อมใช้
> งาน — ทำให้ Local Switching ทำงานได้ทันทีโดยไม่ต้องเพิ่ม VLAN ใหม่ที่ Access Layer

### Config Step-by-Step: สร้าง Flex Profile และแปลง AP-2

```
WLC-1# configure terminal

! ============================================
! 1. สร้าง Flex Profile — กำหนด VLAN ที่ AP-2 (Branch) จะ Local Switch ได้
! ============================================
WLC-1(config)# wireless profile flex FLEX-BRANCH-PROFILE
WLC-1(config-wireless-flex-profile)# description "Branch site - Local Switching VLAN 40"
WLC-1(config-wireless-flex-profile)# vlan-name WIFI vlan-id 40
WLC-1(config-wireless-flex-profile)# native-vlan-id 99
WLC-1(config-wireless-flex-profile)# exit

! ============================================
! 2. สร้าง Policy Profile ใหม่ที่ปิด Central Switching (ให้ AP Switch เอง)
! ============================================
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-wireless-policy)# vlan 40
WLC-1(config-wireless-policy)# no central switching
WLC-1(config-wireless-policy)# central authentication
WLC-1(config-wireless-policy)# central dhcp
WLC-1(config-wireless-policy)# no shutdown
WLC-1(config-wireless-policy)# exit

! ============================================
! 3. สร้าง Policy Tag ใหม่ ผูก WLAN เดิม (ENTERPRISE-WIFI-PROFILE) เข้ากับ Policy Profile ใหม่นี้
! ============================================
WLC-1(config)# wireless tag policy FLEX-BRANCH-TAG
WLC-1(config-policy-tag)# wlan ENTERPRISE-WIFI-PROFILE policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-policy-tag)# exit

! ============================================
! 4. สร้าง Site Tag ที่ผูกกับ Flex Profile (นี่คือจุดที่ทำให้ AP กลายเป็น FlexConnect Mode)
! ============================================
WLC-1(config)# wireless tag site SITE-BRANCH-2
WLC-1(config-site-tag)# flex-profile FLEX-BRANCH-PROFILE
WLC-1(config-site-tag)# exit

! ============================================
! 5. Apply Tag ใหม่ให้ AP-2
! ============================================
WLC-1(config)# ap name AP-2
WLC-1(config-ap-tag)# policy-tag FLEX-BRANCH-TAG
WLC-1(config-ap-tag)# site-tag SITE-BRANCH-2
WLC-1(config-ap-tag)# rf-tag RF-CAMPUS-MAIN
WLC-1(config-ap-tag)# exit

WLC-1(config)# end
WLC-1# copy running-config startup-config
```

> **สำคัญ**: ตัวเลือก `no central switching` อยู่ในระดับ **Policy Profile** ไม่ใช่ WLAN Profile
> — เพราะ Policy Profile คือส่วนที่ผูกกับ VLAN/Data Path ตาม Tag-Based Architecture ที่เรียนไว้
> ใน Part 20 (Step 195) ส่วน `central authentication` ที่ยังเปิดอยู่หมายความว่า WLC-1 ยังคง
> ตัดสินใจเรื่อง Authentication (PSK/802.1X) อยู่เหมือนเดิม — มีแค่ Data Plane เท่านั้นที่ Local

### ปรับ Port บน ACCESS-SW4 จาก Access เป็น Trunk (รองรับ VLAN 40 Local Switching)

Part 20 ตั้งค่า Port ที่ AP-2 เสียบเป็น **Access VLAN 99** อย่างเดียว (เพราะตอนนั้น Data
Traffic ทั้งหมดวิ่งผ่าน CAPWAP ไป WLC) — ตอนนี้ AP-2 ต้อง Local Switch VLAN 40 เอง จึงต้อง
เปลี่ยน Port เป็น **Trunk** ที่ยอมทั้ง VLAN 40 (Data) และ VLAN 99 (Control, เป็น Native VLAN)

```
ACCESS-SW4# configure terminal
ACCESS-SW4(config)# interface GigabitEthernet0/3
ACCESS-SW4(config-if)# description ** Uplink to AP-2 (FlexConnect - Local Switching VLAN 40) **
ACCESS-SW4(config-if)# switchport mode trunk
ACCESS-SW4(config-if)# switchport trunk native vlan 99
ACCESS-SW4(config-if)# switchport trunk allowed vlan 40,99
ACCESS-SW4(config-if)# spanning-tree portfast trunk
ACCESS-SW4(config-if)# power inline auto
ACCESS-SW4(config-if)# no shutdown
ACCESS-SW4(config-if)# end
ACCESS-SW4# copy running-config startup-config
```

### ตรวจสอบผลลัพธ์

```
WLC-1# show ap name AP-2 config general | include AP Mode|IP Address
  AP Mode                                       : FlexConnect
  IP Address                                    : 10.10.99.102

WLC-1# show wireless profile flex detailed FLEX-BRANCH-PROFILE | include VLAN
  VLAN Name        VLAN Id
  WIFI             40

WLC-1# show ap name AP-2 flexconnect | include VLAN|Switching
  Local Switching                               : Enabled
  VLAN Name : WIFI, VLAN Id : 40
```

---

## Step 353 — RF Fundamentals for Design: RSSI, SNR, Cell Overlap, RRM (DCA/TPC)

### RSSI (Received Signal Strength Indicator) — วัดความแรงสัญญาณ

RSSI มีค่าเป็น dBm (ค่าลบ ยิ่งใกล้ 0 ยิ่งแรง) วัดจาก Client หรือ AP ว่ารับสัญญาณของอีกฝั่งแรง
แค่ไหน

| RSSI | คุณภาพ | ใช้งานได้กับ |
|---|---|---|
| -30 dBm | ดีเยี่ยม (อยู่ใกล้ AP มาก) | ทุก Application |
| -50 ถึง -60 dBm | ดี | Data + Voice/Video |
| -67 dBm | พอใช้ (ค่า Cell-edge เป้าหมายสำหรับ Voice) | Voice ขั้นต่ำที่แนะนำ |
| -70 ถึง -75 dBm | อ่อน (ค่า Cell-edge เป้าหมายสำหรับ Data ทั่วไป) | Data พื้นฐานเท่านั้น |
| -80 dBm ต่ำกว่า | อ่อนมาก | Connection ไม่นิ่ง เสี่ยงหลุด |
| -90 dBm | ไม่มีสัญญาณจริง | ใช้งานไม่ได้ |

### SNR (Signal-to-Noise Ratio) — ตัวเลขที่สำคัญกว่า RSSI เพียวๆ

```
SNR (dB) = RSSI (dBm) - Noise Floor (dBm)

ตัวอย่าง: RSSI = -65 dBm, Noise Floor = -90 dBm
SNR = -65 - (-90) = 25 dB
```

RSSI แรงอย่างเดียวไม่พอ — ถ้า Noise Floor สูงด้วย (มี Interference มาก) SNR ก็จะต่ำ ทำให้
Data Rate ที่ใช้งานได้จริงต่ำลงแม้ RSSI จะดูดี

| SNR | คุณภาพการเชื่อมต่อ |
|---|---|
| 40 dB ขึ้นไป | ดีเยี่ยม รองรับ Data Rate สูงสุด |
| 25 dB | ขั้นต่ำที่แนะนำสำหรับ **Voice/VoWLAN** |
| 20 dB | ขั้นต่ำที่แนะนำสำหรับ **Data ทั่วไป** |
| ต่ำกว่า 15 dB | Retransmission สูง, Throughput ตกฮวบ |

### Cell Overlap Design — กฎ 15-20%

ในการวาง AP หลายตัวให้ครอบคลุมพื้นที่ (Coverage) ต้องออกแบบให้ **Cell (พื้นที่ครอบคลุมของ AP
แต่ละตัว) ทับซ้อนกันประมาณ 15-20%** ที่ค่า RSSI เป้าหมาย (เช่น -67 dBm) เพื่อให้ Client
Roaming ระหว่าง AP ได้อย่างราบรื่นโดยไม่มี "Dead Zone" (จุดที่สัญญาณจาก AP ทั้งสองอ่อนเกินไป)

```
Cell Overlap ที่เหมาะสม (15-20%)

   AP-1 Cell                    AP-2 Cell
┌──────────────┐          ┌──────────────┐
│              │  ◄─15-20%─►│              │
│      ●AP-1    │░░░░░░░░░░│    ●AP-2      │
│              │  Overlap  │              │
└──────────────┘   Zone    └──────────────┘
                 (Client roam ราบรื่นในโซนนี้)

Overlap น้อยเกินไป (<10%)              Overlap มากเกินไป (>30%)
┌──────────┐  ┌──────────┐         ┌──────────────┐┌──────────────┐
│    ●AP-1  │  │   ●AP-2   │         │      ●AP-1    ││    ●AP-2      │
└──────────┘  └──────────┘         └──────────────┘└──────────────┘
    Dead Zone ระหว่าง AP                  Co-Channel Interference สูง
    (สัญญาณอ่อนเกินไปตรงกลาง)              (AP ใช้ Channel เดียวกันแล้วรบกวนกัน)
```

### Channel และ Power Planning — เป้าหมายคือ Balance ระหว่าง Coverage กับ Interference

- **Channel Plan**: กระจาย Channel ให้ AP ข้างเคียงใช้ Channel ต่างกัน (Non-overlapping — ทวน
  จาก Part 20 Step 191) เพื่อลด **Co-Channel Interference (CCI)**
- **Power Plan**: ปรับ Transmit Power ให้พอดีกับ Cell Overlap ที่ต้องการ — Power สูงเกินไป
  ทำให้ Cell ทับซ้อนกันมากเกินไป (เพิ่ม CCI), Power ต่ำเกินไปทำให้เกิด Dead Zone

### RRM (Radio Resource Management) — ระบบอัตโนมัติที่ WLC ใช้จัดการ Channel/Power

RRM คือฟีเจอร์ที่ WLC ใช้คำนวณ Channel และ Power ให้ AP ทุกตัวโดยอัตโนมัติ แทนที่ Admin ต้อง
ปรับมือทีละตัว (ปัญหาที่ Autonomous AP มีตามที่กล่าวไว้ใน Part 20 Step 192) — ประกอบด้วย 2
Algorithm หลัก

**DCA (Dynamic Channel Assignment)**
- AP ทุกตัวฟัง RF Neighbor รอบข้าง (ผ่าน Neighbor Discovery Protocol - NDP ทาง Over-the-Air)
  แล้วรายงานกลับไปที่ WLC
- WLC คำนวณ "Cost Matrix" ของ Interference ระหว่าง AP ทุกคู่ แล้วเลือก Channel ที่ทำให้
  Co-Channel/Adjacent-Channel Interference โดยรวมต่ำที่สุด
- ทำงานเป็นรอบ (Default Interval = 10 นาที) หรือสั่งรันทันทีได้

**TPC (Transmit Power Control)**
- ปรับ Transmit Power ของ AP แต่ละตัวให้ Cell Edge (พื้นที่ขอบของ Cell) มี RSSI ใกล้เคียงกับ
  ค่า Target ที่ตั้งไว้ (Default Power Threshold ประมาณ -70 dBm)
- ลด Over-coverage (Power แรงเกินจำเป็นจนไปแทรกแซง Cell ข้างเคียง) โดยยังคง Overlap ที่พอดี
  สำหรับ Roaming

```
! เปิด RRM Auto สำหรับ DCA และ TPC บน 5 GHz (Default อยู่แล้วปกติ แต่ตรวจสอบให้ชัดเจน)
WLC-1(config)# ap dot11 5ghz rrm channel dca anchor-time 0
WLC-1(config)# ap dot11 5ghz rrm channel dca interval 1
WLC-1(config)# ap dot11 5ghz rrm txpower auto max 17 min 7
```

| Parameter | ความหมาย |
|---|---|
| `dca anchor-time` | ชั่วโมงที่ DCA เริ่มรันในแต่ละวัน (0 = เที่ยงคืน) |
| `dca interval` | ความถี่ของรอบ DCA เป็นชั่วโมง (1 = ทุก 1 ชั่วโมง) |
| `txpower auto max/min` | ขอบบน/ล่างของ Power (dBm) ที่ TPC อนุญาตให้ปรับ |

> รายละเอียดการสร้าง **RF Profile แยกตามพื้นที่** (เช่น High-density ต่างจาก Low-density)
> จะลงคอนฟิกจริงใน Step 358

---

## Step 354 — Client Roaming Deep Dive: L2/L3 Roaming และ Mobility Group กับ WLC-2

### ทวน L2 Roaming จาก Part 20 แบบเจาะกลไก

Part 20 (Step 198) สอนว่า L2 Roaming คือ Client ย้าย AP แต่ VLAN เดิม — Part นี้อธิบายกลไก
ภายในว่า WLC ทำอย่างไร: WLC เก็บ **Client Database** ที่ Map ระหว่าง Client MAC Address ↔
AP ที่ Client Associate อยู่ปัจจุบัน เมื่อ Client Roam จาก AP-1 ไป AP-2 (ที่ join WLC-1
เดียวกัน) WLC-1 แค่ **อัปเดต Entry ใน Database ตัวเอง** ว่า Client ตัวนี้ย้ายไปอยู่ AP-2 แล้ว
โดยไม่ต้องขอ IP ใหม่ เพราะ Client ยังอยู่ใน CAPWAP Tunnel Database ของ WLC-1 ตัวเดิม — เรียก
ว่า **Intra-Controller Roaming**

```
Intra-Controller L2 Roaming (WLC-1 ตัวเดียว - เหมือน Part 20)

Client Database บน WLC-1:
┌─────────────────┬──────────┬──────┐
│ Client MAC        │ AP Name   │ VLAN │
├─────────────────┼──────────┼──────┤
│ aabb.ccdd.1122     │ AP-1 → AP-2│ 40   │  <- แค่เปลี่ยน AP Name ไม่เปลี่ยน VLAN/IP
└─────────────────┴──────────┴──────┘
```

### L3 Roaming — Client ย้ายข้าม WLC ที่ VLAN คนละอัน

เกิดขึ้นเมื่อ Client ย้ายไปยัง AP ที่ join กับ **WLC ตัวอื่น** ซึ่งมี Policy Profile ผูกกับ VLAN
คนละ Subnet (เช่น Multi-building ที่แต่ละอาคารมี WLC ของตัวเอง ผูก VLAN ท้องถิ่น) — ถ้าไม่มี
การจัดการพิเศษ Client ต้องขอ DHCP ใหม่ (IP เปลี่ยน) ทำให้ Session ที่เปิดอยู่ขาด

Cisco แก้ปัญหานี้ด้วย **Inter-Controller Mobility**: เมื่อ Client Roam ไปยัง WLC ปลายทาง
("Foreign Controller") WLC ปลายทางจะสร้าง **Mobility Tunnel (EoIP - Ethernet over IP)**
กลับไปยัง WLC ต้นทาง ("Anchor Controller") เพื่อให้ Client Traffic ยังคง**วิ่งผ่าน VLAN เดิม
ที่ Anchor** — Client จึงรักษา IP เดิมไว้ได้ แม้ตัวจะย้ายไปอยู่ Physical Location คนละที่แล้ว

```
Inter-Controller L3 Roaming (WLC-1 ↔ WLC-2)

Client (VLAN 40, IP 10.10.40.101) associate กับ AP-1 ที่ WLC-1 (Anchor)
              │
              ▼ เดินไปอาคารที่ 2 ที่มี WLC-2 ดูแล (VLAN 140 ท้องถิ่น)
Client roam ไปยัง AP ที่ join WLC-2 (Foreign)
              │
              ▼
WLC-2 ตรวจสอบ Client Database → พบว่า Client มาจาก WLC-1 (ผ่าน Mobility Group)
              │
              ▼
WLC-2 สร้าง Mobility (EoIP) Tunnel กลับไปยัง WLC-1
              │
              ▼
Client Traffic ทั้งหมดยัง Encapsulate ผ่าน Tunnel กลับไปออกที่ VLAN 40 บน WLC-1 เดิม
→ Client รักษา IP 10.10.40.101 ไว้ได้ ไม่มี Session ขาด
```

### Mobility Group — Config เต็มรูปแบบระหว่าง WLC-1 และ WLC-2

> **สมมติสถานการณ์ (เฉพาะ Part 36)**: WLC-2 คือ Controller ของ "อาคารที่ 2" — Management
> IP `10.10.199.40` (VLAN 199, สมมติสำหรับ Part นี้) และมี WLAN ท้องถิ่นของตัวเองผูกกับ
> VLAN 140 (`10.10.140.0/24`, สมมติเช่นกัน) — ไม่อยู่ใน Canonical Plan หลักของหลักสูตร

```
! ============================================
! ฝั่ง WLC-1 (Campus หลัก - Mobility Group Member ที่ 1)
! ============================================
WLC-1# configure terminal
WLC-1(config)# wireless mobility group name CAMPUS-MOBILITY-DOMAIN
WLC-1(config)# wireless mobility group member ip 10.10.199.40 public-ip 10.10.199.40 group CAMPUS-MOBILITY-DOMAIN
WLC-1(config)# end
WLC-1# copy running-config startup-config

! ============================================
! ฝั่ง WLC-2 (อาคาร 2 - Mobility Group Member ที่ 2)
! ============================================
WLC-2# configure terminal
WLC-2(config)# wireless mobility group name CAMPUS-MOBILITY-DOMAIN
WLC-2(config)# wireless mobility group member ip 10.10.99.40 public-ip 10.10.99.40 group CAMPUS-MOBILITY-DOMAIN
WLC-2(config)# end
WLC-2# copy running-config startup-config
```

> **ข้อกำหนดสำคัญ**: ทุก WLC ใน Mobility Group เดียวกันต้องใช้ **ชื่อ Group เดียวกัน**
> (case-sensitive) — และต้องเปิด UDP Port **16666** (Mobility Control Message) และ **16667**
> (Mobility Data ถ้าใช้ Multicast) ระหว่างกันบน Firewall/ACL ที่กั้นอยู่ระหว่าง WLC ทั้งสองด้วย

### ตรวจสอบ Mobility Group

```
WLC-1# show wireless mobility summary
Mobility Summary

Mobility Role                                   : Mobility Controller
Mobility Protocol Port                          : 16666
Mobility Group Name                             : CAMPUS-MOBILITY-DOMAIN
DTLS Mode                                       : Enabled

Controllers configured in the Mobility Group
--------------------------------------------------------------------------
MAC Address       IP Address       Public IP        Group Name              Status
--------------------------------------------------------------------------
00ab.1234.0001     10.10.99.40      10.10.99.40      CAMPUS-MOBILITY-DOMAIN  Up (Self)
00ab.5678.0002     10.10.199.40     10.10.199.40     CAMPUS-MOBILITY-DOMAIN  Up
```

---

## Step 355 — Fast Roaming Protocols: 802.11r (FT), 802.11k, 802.11v

### ทำไม Roaming แบบธรรมดายังไม่พอสำหรับ VoWLAN/Real-time Application

L2 Roaming ที่เรียนใน Step 354 เร็วอยู่แล้ว (มักไม่เกิน 100ms) แต่ปัญหาคือ **ขั้นตอน Re-
authentication** ที่เกิดขึ้นทุกครั้งที่ Client Roam ไปยัง AP ใหม่ — ถ้าใช้ WPA2-Enterprise
(802.1X) เต็มรูปแบบ ต้องทำ **Full EAP Exchange ใหม่กับ RADIUS Server ทุกครั้ง** ซึ่งอาจใช้
เวลา 100-800ms ขึ้นอยู่กับ RTT ไปยัง RADIUS — สำหรับ Data ทั่วไปไม่มีปัญหา แต่สำหรับ
**VoWLAN (Voice over WLAN)** ที่ต้องการ Roam ให้เสร็จภายใน **50ms** (ไม่ให้เสียงสาย/หลุด)
การ Re-authenticate เต็มรูปแบบทุกครั้งที่ Roam คือปัญหาใหญ่

### 802.11r — Fast BSS Transition (FT)

802.11r ลดเวลา Re-authentication โดยให้ Client ทำ **Key derivation ล่วงหน้า (Pre-computed
Key Hierarchy)** ระหว่าง AP ต้นทางกับ AP ปลายทาง ผ่าน 2 วิธี

| วิธี | อธิบาย |
|---|---|
| **Over-the-Air** | Client คุยตรงกับ AP ปลายทางก่อน Roam จริง (ผ่าน RF) เพื่อแลก Key ล่วงหน้า |
| **Over-the-DS (Distribution System)** | Client คุยผ่าน AP ปัจจุบัน (ที่ Associate อยู่) ให้ Forward ข้อความไปยัง AP ปลายทางแทน (ผ่าน Wired backend) — นิยมกว่าเพราะไม่ต้องพึ่ง RF ที่อาจไม่แรงพอ |

ผลคือ ตอน Roam จริง Client ไม่ต้องทำ Full 4-Way Handshake ใหม่ทั้งหมด — ลดเวลา Roaming ลง
เหลือ **ระดับ 10-20ms** ซึ่งเพียงพอสำหรับ VoWLAN

### 802.11k — Neighbor Reports

802.11k ให้ AP ส่ง **Neighbor Report** (รายชื่อ AP ข้างเคียงพร้อม Channel/RSSI ที่ควรพิจารณา)
ให้ Client ทราบล่วงหน้า — Client ที่รองรับ 802.11k จะรู้ว่า "ถ้าจะ Roam ควรไปที่ AP ตัวไหน"
โดยไม่ต้องเสียเวลา Scan ทุก Channel เองแบบสุ่ม (ซึ่งช้ากว่ามาก) — ลด Roaming Decision Time

### 802.11v — BSS Transition Management (BTM)

802.11v ให้ **WLC/AP สามารถ "แนะนำ" Client ให้ Roam ไปยัง AP ที่ดีกว่า** ได้เชิงรุก (ไม่ต้อง
รอ Client ตัดสินใจเองจาก RSSI ที่อ่อนลง) เช่น ถ้า WLC เห็นว่า Client ตัวหนึ่งอยู่ AP ที่มี Client
หนาแน่นเกินไป แต่มี AP ข้างเคียงที่ Load น้อยกว่าและสัญญาณดีพอ WLC สามารถส่ง BSS Transition
Request ให้ Client ย้ายไปแบบ Proactive — ช่วยเรื่อง Load Balancing ระหว่าง AP ด้วย

### ทำไม 3 Protocol นี้ต้องใช้ร่วมกัน

```
802.11k (รู้ว่ามี AP ไหนน่าไปบ้าง - Neighbor Report)
        │
        ▼
802.11v (WLC แนะนำ/สั่งให้ Client ย้ายไปยัง AP ที่เหมาะสมที่สุด - Proactive)
        │
        ▼
802.11r (Roam ไปยัง AP ที่เลือกแล้วแบบเร็วที่สุด - Fast Key Exchange, ไม่ Re-auth เต็ม)
```

### Config: เปิด 802.11r บน WLAN "ENTERPRISE-WIFI"

```
WLC-1# configure terminal
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# security ft
WLC-1(config-wlan)# security ft over-the-ds
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
```

### Config: เปิด 802.11k (Neighbor List) และ 802.11v (BSS Transition)

```
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE
WLC-1(config-wlan)# assisted-roaming neighbor-list
WLC-1(config-wlan)# assisted-roaming dual-list
WLC-1(config-wlan)# bss-transition
WLC-1(config-wlan)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### ตรวจสอบผลลัพธ์

```
WLC-1# show wlan name ENTERPRISE-WIFI-PROFILE | include 802.11r|FT|BSS Transition|Neighbor
  802.11r Fast Transition                   : Enabled
  Over the DS                               : Enabled
  Neighbor List                             : Enabled
  BSS Transition                            : Enabled
```

> **ข้อควรระวัง**: Client รุ่นเก่าบางตัวที่ไม่รองรับ 802.11r อาจ Associate ไม่ผ่าน ถ้าตั้งค่า
> `security ft` เป็น **Mandatory** — ทางแก้คือใช้โหมด **Adaptive** (`security ft adaptive`)
> ที่รองรับทั้ง Client เก่าและใหม่ในเวลาเดียวกันบน SSID เดียวกัน

---

## Step 356 — WLC High Availability: N+1 HA และ AP SSO กับ WLC-1-STANDBY

### สองโมเดล HA ของ Catalyst 9800 ที่ต้องแยกให้ออก

| โมเดล | กลไก | ผลตอน WLC ล่ม | AP ต้องเป็น HW เดียวกันไหม |
|---|---|---|---|
| **N+1 HA** | WLC หลายตัว **เป็นอิสระจากกัน** — AP กำหนด Primary/Secondary/Tertiary Controller ไว้เอง | AP ต้อง **CAPWAP Rejoin ใหม่** กับ WLC ตัวถัดไป (ไม่ Stateful) — Client หลุดชั่วครู่ | ไม่ต้อง (แม้อยู่คนละที่ทางภูมิศาสตร์ก็ได้) |
| **AP SSO (Stateful Switchover)** | WLC สองตัวจับคู่เป็น **HA Pair เดียว** (Active/Standby-Hot) เชื่อมด้วย Redundancy Port (RP) โดยตรง | Failover แบบ **Stateful** — AP **ไม่ต้อง Rejoin เลย** (CAPWAP Session ถูก Sync ไปที่ Standby ไว้ล่วงหน้า) | ต้องเป็น Platform/Version เดียวกัน |

**AP SSO คือ HA ที่แท้จริงในความหมาย "Stateful"** — WLC สองตัวทำงานเป็นแนวคิดเดียวกับ HSRP ที่
เรียนใน Part 18 แต่ที่ระดับ Session-state เต็มรูปแบบ (Sync Client Database, AP Database,
Mobility State ทั้งหมดไปที่ Standby ตลอดเวลา) ส่วน N+1 HA คือการมี WLC สำรองแบบ "Cold" ที่ AP
รู้จักไว้เผื่อตัวหลักหาย แต่ต้อง Rejoin ใหม่ทุกครั้ง (คล้าย Static Backup มากกว่า Live Failover)

### Worked Config: จับคู่ WLC-1 กับ WLC-1-STANDBY เป็น AP SSO HA Pair

> **สมมติสถานการณ์ (เฉพาะ Part 36)**: เพิ่ม **WLC-1-STANDBY** (Catalyst 9800-CL รุ่น/เวอร์ชัน
> เดียวกับ WLC-1) เข้ามาที่ Campus เดียวกัน เชื่อมต่อกันด้วย Link ตรง (Redundancy Port) ผ่าน
> Subnet `10.10.253.0/30` — Subnet นี้เป็น Link เฉพาะระหว่าง WLC สองตัวเท่านั้น ไม่ผ่าน Switch
> ตัวอื่นในเครือข่าย จึงไม่ปรากฏใน `docs/00-ip-address-plan.md` (เหมือนแนวทางที่ IP Plan ใช้กับ
> Cross-link ของ WAN-EDGE-1 ↔ WAN-EDGE-2)

```
Topology ของ AP SSO HA Pair

┌──────────────┐  Gi3 (Redundancy Port)   ┌──────────────────┐
│   WLC-1        │◄────10.10.253.0/30─────►│  WLC-1-STANDBY     │
│  (Active)      │      Heartbeat Link      │  (Standby-Hot)     │
│  10.10.99.40   │                          │  10.10.99.41 (RMI)  │
└──────┬───────┘                          └────────┬─────────┘
       │                                             │
       └───────────────┬─────────────────────────────┘
                        ▼
              Wireless Management IP (RMI) ร่วม: 10.10.99.40
              (AP ทุกตัว join ที่ IP นี้เพียง IP เดียว — ไม่รู้เลยว่ามี WLC 2 ตัว)
```

```
! ============================================
! ทำบน WLC-1 (จะกลายเป็น Active)
! ============================================
WLC-1# configure terminal
WLC-1(config)# interface GigabitEthernet3
WLC-1(config-if)# ip address 10.10.253.1 255.255.255.252
WLC-1(config-if)# no shutdown
WLC-1(config-if)# exit
WLC-1(config)# chassis redundancy ha-interface GigabitEthernet3 local-ip 10.10.253.1 remote-ip 10.10.253.2
% ระบบจะสั่ง Reload อัตโนมัติเพื่อเริ่มกระบวนการจับคู่ HA

! ============================================
! ทำบน WLC-1-STANDBY (จะกลายเป็น Standby-Hot)
! ============================================
WLC-1-STANDBY# configure terminal
WLC-1-STANDBY(config)# interface GigabitEthernet3
WLC-1-STANDBY(config-if)# ip address 10.10.253.2 255.255.255.252
WLC-1-STANDBY(config-if)# no shutdown
WLC-1-STANDBY(config-if)# exit
WLC-1-STANDBY(config)# chassis redundancy ha-interface GigabitEthernet3 local-ip 10.10.253.2 remote-ip 10.10.253.1
% ระบบจะสั่ง Reload อัตโนมัติเพื่อเริ่มกระบวนการจับคู่ HA
```

หลัง Reload ทั้งสองตัวจะ Negotiate กันผ่าน Redundancy Port เอง — ตัวที่ชนะ Election จะกลาย
เป็น **Active** (รับ Config ทั้งหมด, ตอบ CAPWAP ที่ RMI IP เดิมของ WLC-1 คือ `10.10.99.40`)
ส่วนอีกตัวกลายเป็น **Standby-Hot** (Sync State ตลอดเวลา, ไม่ตอบ CAPWAP เอง จนกว่า Active
จะล่ม) — จากมุมมองของ AP และ Network ภายนอก **เห็น WLC เพียงตัวเดียว** ที่ IP `10.10.99.40`
เท่านั้น (คล้าย VIP ของ HSRP)

### AP SSO ทำงานอย่างไรตอน Active ล่ม

```
ก่อน Failover: WLC-1 (Active) กำลังให้บริการ AP-1/AP-2 อยู่ปกติ
       │
       ▼ WLC-1 (Active) ล่ม (Power/Software Crash)
WLC-1-STANDBY ตรวจจับได้ทันทีผ่าน Redundancy Port (Heartbeat หายไป)
       │
       ▼ Stateful Switchover (SSO)
WLC-1-STANDBY เลื่อนขึ้นเป็น Active ทันที พร้อม State ที่ Sync มาแล้วล่วงหน้า:
  - CAPWAP Session ของ AP-1/AP-2 (ไม่ต้อง Rejoin!)
  - Client Database (Client ไม่ต้อง Re-associate/Re-DHCP)
  - Mobility State, RRM State
       │
       ▼
AP-1/AP-2 ยังคง "Registered" ต่อเนื่อง — เห็นแค่ WLC IP เดิม (10.10.99.40) ไม่เคยเปลี่ยน
Client ที่ใช้งานอยู่ (เช่น กำลัง Video Call) **ไม่หลุด** เพราะ Session ทั้งหมดถูก Sync ไว้แล้ว
```

> **นี่คือความต่างสำคัญจาก N+1 HA**: ถ้าเป็น N+1 (AP มี Primary/Secondary Controller
> แยกกันคนละ IP) ตอน Primary ล่ม AP ต้องออกจาก CAPWAP Session เดิม แล้ว **Discovery + Join
> ใหม่ทั้งหมด** กับ Secondary Controller — ใช้เวลาหลักสิบวินาทีถึงหลักนาที และ Client ทุกตัว
> จะหลุดแล้วต้อง Re-associate ใหม่ (Non-stateful)

### ตรวจสอบสถานะ HA Pair

```
WLC-1# show chassis
Chassis/Stack Mac Address : 001a.2b3c.4d5e - Local Mac Address
Mac persistency wait time: Indefinite
                                             H/W   Current
 Chassis#  Role       Mac Address     Priority Version  State
 -------------------------------------------------------------
 *1        Active     001a.2b3c.4d5e   1       V01      Ready
  2        Standby    001a.2b3c.4d5f   1       V01      Ready

WLC-1# show redundancy states
       my state = 13 -ACTIVE
     peer state = 8  -STANDBY HOT
           Mode = Duplex
   Split Mode = Disabled
 Manual Swact = enabled
 Communications = Up

WLC-1# show ap redundancy summary
AP SSO State                                  : Enabled
AP Client SSO                                 : Enabled
```

---

## Step 357 — Multi-WLC Mobility Architecture at Scale: Mobility Controller vs Mobility Agent

### ปัญหาของ Mobility Group แบบ Flat เมื่อองค์กรใหญ่ขึ้น

Mobility Group ที่สอนใน Step 354 (ทุก WLC รู้จักกันหมดแบบ Full-mesh ผ่าน UDP 16666) มี
**ขีดจำกัด 24 WLC ต่อ Group** — องค์กรขนาดใหญ่ (หลายสิบ/หลายร้อยอาคาร) ที่มี WLC เป็นจำนวน
มากจะเกินขีดจำกัดนี้ได้ง่าย จึงต้องมีสถาปัตยกรรมแบบลำดับชั้น (Hierarchical) เข้ามาช่วย Scale

### Mobility Group vs Mobility Domain

| แนวคิด | ขอบเขต | ใช้เมื่อไหร่ |
|---|---|---|
| **Mobility Group** | WLC สูงสุด 24 ตัว Full-mesh กันโดยตรง แชร์ Client Database เต็มรูปแบบ | องค์กรขนาดกลาง ที่ WLC ไม่เกิน 24 ตัว |
| **Mobility Domain** (RF Group รวมหลาย Mobility Group) | รวมหลาย Mobility Group เข้าด้วยกัน — Roaming ข้าม Group ทำงานได้ แต่ไม่ Full-mesh ทุกตัว (ลด Overhead) | องค์กรขนาดใหญ่ระดับหลายร้อย WLC (Campus จำนวนมาก, Multi-site ทั่วประเทศ) |

### Mobility Controller (MC) vs Mobility Agent (MA)

ในสถาปัตยกรรม Wireless แบบ Scale ใหญ่ (เช่น SD-Access Wireless) Catalyst 9800 แต่ละตัว
สามารถทำหน้าที่ได้ 2 Role

| Role | หน้าที่ | เหมาะกับ |
|---|---|---|
| **Mobility Controller (MC)** | เก็บ Client Database "Global" ของทั้ง Domain, ตัดสินใจ Mobility เต็มรูปแบบ, พูดคุยกับ MC ตัวอื่นในระดับ Domain | WLC หลักที่อยู่ตำแหน่งรวมศูนย์ (เช่น Data Center/Border ของ SD-Access Fabric) |
| **Mobility Agent (MA)** | ดูแล AP/Client เฉพาะพื้นที่ตัวเอง (Local เท่านั้น) แล้ว **Forward คำถาม Mobility ไปหา MC ที่ตนสังกัด** แทนที่จะคุยกับ WLC อื่นตรงๆ | WLC ที่กระจายอยู่ตามอาคาร/สาขา จำนวนมาก (Fabric Edge) |

```
สถาปัตยกรรมแบบ Hierarchical MC/MA (สำหรับองค์กรขนาดใหญ่มาก)

                    ┌─────────────────────┐
                    │  Mobility Controller  │  <- รู้จัก Client ทุกตัวใน Domain (Global View)
                    │      (เช่น WLC กลาง)   │
                    └──────────┬───────────┘
           ┌────────────────────┼────────────────────┐
           ▼                    ▼                    ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │ Mobility Agent │    │ Mobility Agent │    │ Mobility Agent │
   │   (อาคาร A)     │    │   (อาคาร B)     │    │   (อาคาร C)     │
   └──────────────┘    └──────────────┘    └──────────────┘
   Client Roam ระหว่าง MA ต่างตัว → ถาม MC กลางเป็นตัวกลางเสมอ (ไม่ Full-mesh กันเอง)
```

> **ในบริบทหลักสูตรนี้** (WLC-1 กับ WLC-2 ใน Step 354) ทั้งสองตัวทำหน้าที่เป็น **Mobility
> Controller แบบ Standalone** (Default Role ของ 9800 ทุกตัวเมื่อยังไม่ตั้งเป็น MA) และคุยกัน
> ตรงๆ ผ่าน Mobility Group แบบ Flat — เพียงพอสำหรับ Lab ระดับ 2 อาคาร สถาปัตยกรรม MC/MA แบบ
> เต็มรูปแบบจะมีความหมายจริงเมื่อ WLC มีจำนวนมากถึงระดับ SD-Access Campus Fabric ทั้งองค์กร

```
! ตั้ง Role ของ WLC ให้เป็น Mobility Agent (ชี้ไปยัง MC ที่กำหนด) - ตัวอย่าง Syntax แนวคิด
WLC-2(config)# wireless mobility controller ip 10.10.99.40
```

---

## Step 358 — RF Profiles และ AP Groups สำหรับแยก RF Setting ตามอาคาร/ชั้น

### ทำไมต้องแยก RF Profile ตามพื้นที่

RF Tag เดียว (`RF-CAMPUS-MAIN`) ที่ตั้งไว้ใน Part 20 ใช้ RF Profile Default ของ Cisco
(`Typical_Client_Density`) กับ AP-1/AP-2 ทั้งคู่ — แต่พื้นที่จริงมักมีความหนาแน่นต่างกัน เช่น
ห้องประชุมใหญ่ (Client เยอะ ต้องการ Cell เล็กลง Power ต่ำลง) เทียบกับ Corridor/Warehouse
(Client น้อย ต้องการ Coverage ไกล Power สูงขึ้น) — การใช้ RF Profile เดียวทั่วทั้งองค์กรจึงไม่
เหมาะกับการออกแบบจริง

### สร้าง RF Profile แบบ Custom สำหรับพื้นที่ความหนาแน่นสูง (High-Density)

```
WLC-1# configure terminal

WLC-1(config)# ap dot11 5ghz rf-profile RF-HIGH-DENSITY
WLC-1(config-rf-profile)# description "Auditorium / ห้องประชุมใหญ่ - Client หนาแน่น"
WLC-1(config-rf-profile)# tx-power-min 10
WLC-1(config-rf-profile)# tx-power-max 14
WLC-1(config-rf-profile)# channel-width 20
WLC-1(config-rf-profile)# no shutdown
WLC-1(config-rf-profile)# exit

WLC-1(config)# ap dot11 24ghz rf-profile RF-HIGH-DENSITY
WLC-1(config-rf-profile)# tx-power-min 7
WLC-1(config-rf-profile)# tx-power-max 11
WLC-1(config-rf-profile)# no shutdown
WLC-1(config-rf-profile)# exit
```

### สร้าง RF Profile สำหรับพื้นที่ Coverage กว้าง (Low-Density / Warehouse)

```
WLC-1(config)# ap dot11 5ghz rf-profile RF-LOW-DENSITY
WLC-1(config-rf-profile)# description "Warehouse / Corridor - ต้องการระยะไกล"
WLC-1(config-rf-profile)# tx-power-min 14
WLC-1(config-rf-profile)# tx-power-max 20
WLC-1(config-rf-profile)# channel-width 40
WLC-1(config-rf-profile)# no shutdown
WLC-1(config-rf-profile)# exit
```

### ผูก RF Profile ใหม่เข้ากับ RF Tag ที่แยกตามอาคาร/พื้นที่

```
! RF Tag สำหรับพื้นที่ความหนาแน่นสูง
WLC-1(config)# wireless tag rf RF-TAG-HIGH-DENSITY
WLC-1(config-rf-tag)# 24ghz-rf-policy RF-HIGH-DENSITY
WLC-1(config-rf-tag)# 5ghz-rf-policy RF-HIGH-DENSITY
WLC-1(config-rf-tag)# exit

! RF Tag สำหรับ Warehouse/Corridor
WLC-1(config)# wireless tag rf RF-TAG-LOW-DENSITY
WLC-1(config-rf-tag)# 5ghz-rf-policy RF-LOW-DENSITY
WLC-1(config-rf-tag)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### AP Groups (แนวคิดคู่กับ RF Tag) — จัดกลุ่ม AP ตาม Physical Location

บน Catalyst 9800 การ "จัดกลุ่ม AP ตามพื้นที่" ทำผ่านการ Assign **Tag ชุดต่างกัน (Policy/Site/
RF Tag) ให้กับ AP แต่ละกลุ่ม** ตามตำแหน่งจริง — ตัวอย่างสมมติถ้า AP-1 อยู่ในโซนห้องประชุมใหญ่

```
WLC-1(config)# ap name AP-1
WLC-1(config-ap-tag)# policy-tag ENTERPRISE-TAG
WLC-1(config-ap-tag)# site-tag SITE-CAMPUS-MAIN
WLC-1(config-ap-tag)# rf-tag RF-TAG-HIGH-DENSITY
WLC-1(config-ap-tag)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### ตรวจสอบ RF Profile ที่ Apply จริง

```
WLC-1# show ap rf-profile summary
Number of RF-profiles: 4
RF Profile Name         Band     Description
------------------------------------------------------------
Typical_Client_Density_rf_5gHz   5GHz    Cisco Default
RF-HIGH-DENSITY                  5GHz    Auditorium / ห้องประชุมใหญ่
RF-HIGH-DENSITY                  24GHz   Auditorium / ห้องประชุมใหญ่
RF-LOW-DENSITY                   5GHz    Warehouse / Corridor

WLC-1# show ap dot11 5ghz summary | include AP-1
AP-1     RF-TAG-HIGH-DENSITY     14 dBm (Max)     Ch 100
```

---

## Step 359 — Verification & Troubleshooting เชิงลึก

### คำสั่งตรวจสอบ Radio และ Channel Plan

```
! สรุป Radio 2.4 GHz ทุกตัว
WLC-1# show ap dot11 24ghz summary

! สรุป Radio 5 GHz ทุกตัว
WLC-1# show ap dot11 5ghz summary

! ดู Channel ปัจจุบันที่ DCA เลือกให้แต่ละ AP บน 5 GHz
WLC-1# show ap dot11 5ghz channel
```

ตัวอย่าง Output:

```
WLC-1# show ap dot11 5ghz summary
AP Name    Slot   Admin State   Oper State   Channel   Width   Tx Power Level
--------------------------------------------------------------------------------
AP-1       1      Enabled       Up           100       20MHz   3 (14 dBm)
AP-2       1      Enabled       Up           149       40MHz   2 (17 dBm)

WLC-1# show ap dot11 5ghz channel
Automatic Channel Assignment
  Channel Assignment Mode              : AUTO
  Channel Update Interval              : 1 (hour)
  Anchor time (Hour of the day)        : 0
  Channel Update Contribution          : Noise, Interference, Load, CleanAir Event
--------------------------------------------------------------------------------
AP Name          Channel        Channel Energy (dBm)    Interference (%)
--------------------------------------------------------------------------------
AP-1             100            -85                     2
AP-2             149            -88                     1
```

### คำสั่งตรวจสอบ Mobility และ Roaming History

```
! สรุปสถานะ Mobility Group ทั้งหมด
WLC-1# show wireless mobility summary

! ประวัติ Roaming ของ Client ตัวหนึ่ง (ดูว่า Roam ผ่าน AP ไหนมาบ้าง, ใช้ FT หรือไม่)
WLC-1# show wireless client mobility history mac-address aabb.ccdd.1122
```

ตัวอย่าง Output:

```
WLC-1# show wireless client mobility history mac-address aabb.ccdd.1122
Client MAC Address : aabb.ccdd.1122
Number of entries : 2

Timestamp                    Type    Roam Type      AP Name   BSSID              Auth
----------------------------------------------------------------------------------------
09/26/2026 08:15:22.123       Assoc   Initial        AP-1      70b3.1712.aa00     Complete
09/26/2026 08:17:40.556       Assoc   FT Roam (11r)  AP-2      70b3.1712.bb00     Complete
```

### สาเหตุที่พบบ่อยที่สุดของ Roaming Failure

| อาการ | State/Log ที่เห็น | สาเหตุที่พบบ่อยที่สุด | วิธีแก้ |
|---|---|---|---|
| Client Roam แล้วหลุด (ต้อง Reconnect ใหม่) | `show wireless client mobility history` ไม่มี Entry แบบ FT Roam | Client ไม่รองรับ 802.11r หรือ FT ตั้งเป็น Mandatory แล้ว Client เก่าเข้าไม่ได้ | ใช้ `security ft adaptive` แทน Mandatory |
| Client Roam ช้า (มากกว่า 500ms) | Log แสดง Full 802.1X Re-authentication ทุกครั้ง | 802.11r ไม่ได้เปิด หรือ RADIUS Server ไกลเกินไป (RTT สูง) | เปิด `security ft`, ตรวจ RTT ไปยัง RADIUS |
| Roaming ข้าม WLC ไม่ทำงาน (L3 Roam) | `show wireless mobility summary` แสดง Peer WLC "Down" | Mobility Group Name ไม่ตรงกัน, UDP 16666 ถูก Block, IP ที่ config ผิด | ตรวจ Group Name ให้ตรงตัวใหญ่เล็ก, ตรวจ ACL/Firewall UDP 16666 |
| AP ไม่ Rejoin ตอน WLC ล่ม (แม้ตั้ง AP SSO) | `show ap redundancy summary` แสดง AP SSO Disabled | HA Pair ยังไม่ Form สำเร็จ (Redundancy Port ไม่ขึ้น) | ตรวจสาย/Interface RP ทั้งสองฝั่ง, `show redundancy states` |
| DCA เปลี่ยน Channel บ่อยเกินไป จน Client หลุดตอนเปลี่ยน | Channel เปลี่ยนถี่ใน `show ap dot11 5ghz channel` | DCA Interval ตั้งไว้ถี่เกินไป หรือ Interference สูงจริงจากอุปกรณ์นอกระบบ | ปรับ `dca interval` ให้นานขึ้น, ตรวจ CleanAir Event ว่ามี Rogue/Interferer จริงไหม |

> **กฎสำคัญของ FlexConnect Troubleshooting**: ถ้า Client Local Switching ไม่ได้ IP ให้ตรวจ
> ก่อนว่า **Trunk ที่ AP เสียบ (เช่น ACCESS-SW4 Gi0/3) อนุญาต VLAN Data (40) ผ่านหรือยัง** —
> เป็นจุดที่ Admin มักลืมเปลี่ยนจาก Access เป็น Trunk ตอน Migrate AP จาก Centralized ไป
> FlexConnect (ตามที่ทำใน Step 352)

---

## Step 360 — Lab เต็มรูปแบบ: FlexConnect + 802.11r + WLC HA/AP SSO พร้อม Roaming Test

Lab นี้รวบรวมทุก Step ของ Part 36 เข้าเป็นชุดคอนฟิกเดียวที่สมบูรณ์ ต่อยอดจาก Lab ของ Part 20
โดยไม่แก้ไข Canonical Address Plan (`docs/00-ip-address-plan.md`)

### 1) FlexConnect บน AP-2 (Branch, Local Switching VLAN 40) — สรุปจาก Step 352

```
WLC-1# configure terminal
WLC-1(config)# wireless profile flex FLEX-BRANCH-PROFILE
WLC-1(config-wireless-flex-profile)# vlan-name WIFI vlan-id 40
WLC-1(config-wireless-flex-profile)# native-vlan-id 99
WLC-1(config-wireless-flex-profile)# exit

WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-wireless-policy)# vlan 40
WLC-1(config-wireless-policy)# no central switching
WLC-1(config-wireless-policy)# central authentication
WLC-1(config-wireless-policy)# no shutdown
WLC-1(config-wireless-policy)# exit

WLC-1(config)# wireless tag policy FLEX-BRANCH-TAG
WLC-1(config-policy-tag)# wlan ENTERPRISE-WIFI-PROFILE policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-policy-tag)# exit

WLC-1(config)# wireless tag site SITE-BRANCH-2
WLC-1(config-site-tag)# flex-profile FLEX-BRANCH-PROFILE
WLC-1(config-site-tag)# exit

WLC-1(config)# ap name AP-2
WLC-1(config-ap-tag)# policy-tag FLEX-BRANCH-TAG
WLC-1(config-ap-tag)# site-tag SITE-BRANCH-2
WLC-1(config-ap-tag)# rf-tag RF-CAMPUS-MAIN
WLC-1(config-ap-tag)# exit
```

```
ACCESS-SW4(config)# interface GigabitEthernet0/3
ACCESS-SW4(config-if)# switchport mode trunk
ACCESS-SW4(config-if)# switchport trunk native vlan 99
ACCESS-SW4(config-if)# switchport trunk allowed vlan 40,99
ACCESS-SW4(config-if)# no shutdown
ACCESS-SW4(config-if)# exit
```

### 2) 802.11r Fast Roaming บน ENTERPRISE-WIFI — สรุปจาก Step 355

```
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# security ft adaptive
WLC-1(config-wlan)# security ft over-the-ds
WLC-1(config-wlan)# assisted-roaming neighbor-list
WLC-1(config-wlan)# bss-transition
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
```

### 3) WLC HA Pair + AP SSO (WLC-1 ↔ WLC-1-STANDBY) — สรุปจาก Step 356

```
WLC-1(config)# interface GigabitEthernet3
WLC-1(config-if)# ip address 10.10.253.1 255.255.255.252
WLC-1(config-if)# no shutdown
WLC-1(config-if)# exit
WLC-1(config)# chassis redundancy ha-interface GigabitEthernet3 local-ip 10.10.253.1 remote-ip 10.10.253.2
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

```
WLC-1-STANDBY(config)# interface GigabitEthernet3
WLC-1-STANDBY(config-if)# ip address 10.10.253.2 255.255.255.252
WLC-1-STANDBY(config-if)# no shutdown
WLC-1-STANDBY(config-if)# exit
WLC-1-STANDBY(config)# chassis redundancy ha-interface GigabitEthernet3 local-ip 10.10.253.2 remote-ip 10.10.253.1
WLC-1-STANDBY(config)# end
WLC-1-STANDBY# copy running-config startup-config
```

### 4) Verification เต็มรูปแบบ — ยืนยันว่าทุกอย่างทำงาน

```
WLC-1# show ap summary
Number of APs: 2
AP Name    Slots  AP Model     Ethernet MAC     Radio MAC       State        AP Mode
------------------------------------------------------------------------------------
AP-1       2      C9130AXI     70b3.1712.aa01   70b3.1712.aa00  Registered   Local
AP-2       2      C9130AXI     70b3.1712.bb01   70b3.1712.bb00  Registered   FlexConnect

WLC-1# show wlan name ENTERPRISE-WIFI-PROFILE | include FT|BSS Transition
  802.11r Fast Transition                   : Adaptive
  BSS Transition                            : Enabled

WLC-1# show chassis
 Chassis#  Role       Mac Address     Priority   State
 -------------------------------------------------------
 *1        Active     001a.2b3c.4d5e   1         Ready
  2        Standby    001a.2b3c.4d5f   1         Ready

WLC-1# show ap redundancy summary
AP SSO State                                  : Enabled
AP Client SSO                                 : Enabled
```

### 5) ทดสอบ Roaming จริง — Client เดินจาก AP-1 ไป AP-2 พร้อม Fast Transition

```
[บน Client Laptop - เชื่อมต่อ AP-1 อยู่ก่อน, IP 10.10.40.101]
C:\> ping -t 10.10.40.1
Reply from 10.10.40.1: bytes=32 time=1ms TTL=255
Reply from 10.10.40.1: bytes=32 time=1ms TTL=255
[--- เดินจาก AP-1 ไปยังพื้นที่ AP-2 ---]
Reply from 10.10.40.1: bytes=32 time=1ms TTL=255      <- ไม่มี Packet Loss ระหว่าง Roam
Reply from 10.10.40.1: bytes=32 time=1ms TTL=255

WLC-1# show wireless client mobility history mac-address aabb.ccdd.1122
Timestamp                    Type    Roam Type      AP Name   Auth
------------------------------------------------------------------
09/26/2026 09:02:10.001       Assoc   Initial        AP-1      Complete
09/26/2026 09:04:33.877       Assoc   FT Roam (11r)  AP-2      Complete
```

> Client ยังคง IP `10.10.40.101` เดิมตลอด (L2 Roam ภายใน WLC-1) และ Log ยืนยันว่า Roam ครั้งนี้
> เป็น **FT Roam (11r)** — ไม่มี Full Re-authentication เต็มรูปแบบ ยืนยันว่า 802.11r ทำงานจริง

### 6) ทดสอบ WLC Failover — ปิด WLC-1 (Active) แล้วดูว่า AP ยังอยู่ผ่าน AP SSO

```
[จำลอง WLC-1 (Active) ล่ม - ตัด Power/Reload]
WLC-1# reload
...

[ตรวจสอบที่ WLC-1-STANDBY ทันทีหลัง Failover]
WLC-1-STANDBY# show chassis
 Chassis#  Role       Mac Address     Priority   State
 -------------------------------------------------------
  1        Standby    001a.2b3c.4d5e   1         Init      (WLC-1 กำลัง Reload กลับมา)
 *2        Active     001a.2b3c.4d5f   1         Ready     (เลื่อนขึ้นเป็น Active ทันที)

WLC-1-STANDBY# show ap summary
Number of APs: 2
AP Name    State        AP Mode
--------------------------------
AP-1       Registered   Local        <- ยัง Registered ต่อเนื่อง ไม่ต้อง Rejoin ใหม่
AP-2       Registered   FlexConnect  <- ยัง Registered ต่อเนื่อง ไม่ต้อง Rejoin ใหม่

WLC-1-STANDBY# show wireless client summary
Number of Local Clients: 1
MAC Address     AP Name    WLAN  State
----------------------------------------
aabb.ccdd.1122   AP-2       1     Run     <- Client ไม่หลุด ไม่ต้อง Re-DHCP
```

Lab นี้ยืนยันครบทุกจุดที่ Part 36 ต้องการพิสูจน์: **AP-2 ทำงานเป็น FlexConnect ด้วย Local
Switching สำเร็จ**, **802.11r Fast Transition ทำงานจริงตอน Client Roam ระหว่าง AP-1/AP-2**,
และ **AP SSO ทำให้ AP ทั้งสองตัวไม่ต้อง Rejoin และ Client ไม่หลุดแม้ WLC-1 (Active) ล่มไปจริง**
— ครบทั้ง 3 แกนหลักของ Wireless Architecture ระดับ CCNP ที่ Part 20 (CCNA) ยังไม่ได้สอน

---

## แบบฝึกหัดทวนความเข้าใจ Part 36

1. เพราะเหตุใด CAPWAP Control Tunnel จึงบังคับเข้ารหัสด้วย DTLS เสมอ แต่ CAPWAP Data Tunnel
   เป็น Option — และคำสั่งใดบน Catalyst 9800 ที่ใช้เปิด DTLS สำหรับ Data Tunnel?
2. ในสถานการณ์ FlexConnect Local Switching ทำไม Port บน Access Switch ที่ AP เสียบต้องเปลี่ยน
   จาก Access VLAN เดียวเป็น Trunk — ต่างจาก Centralized Mode อย่างไร?
3. อธิบายความแตกต่างระหว่าง N+1 HA กับ AP SSO ในมุมของ "Stateful" — เพราะเหตุใด AP SSO จึงทำให้
   Client ไม่หลุดตอน WLC ล่ม แต่ N+1 HA ทำไม่ได้?
4. 802.11k, 802.11v, และ 802.11r ทำงานร่วมกันอย่างไรในกระบวนการ Roaming ของ Client หนึ่งตัว
   — แต่ละ Protocol มีหน้าที่ตรงไหน?
5. ตามกฎ Cell Overlap 15-20% เหตุใด Overlap ที่มากเกินไป (>30%) จึงเป็นปัญหา ทั้งที่ดูเหมือนจะ
   ทำให้ Roaming ราบรื่นกว่า?

**เฉลย:**
1. Control Tunnel บังคับเข้ารหัสเพราะบรรจุคำสั่ง Config/Join ที่สำคัญต่อความปลอดภัยของทั้งระบบ
   (ถ้าถูกดักฟัง/ปลอมแปลงได้ อันตรายมาก) ส่วน Data Tunnel เป็น Client Traffic ทั่วไปที่ถือว่า
   วิ่งอยู่บน Network ภายในที่ควบคุมได้ — คำสั่งเปิด DTLS Data คือ `link-encryption` ภายใต้
   `ap profile <name>`
2. Centralized Mode ส่ง Client Data ผ่าน CAPWAP Data Tunnel ไปที่ WLC เสมอ Port ที่ AP เสียบจึง
   ต้องมีแค่ VLAN Control-plane (99) เท่านั้น แต่ FlexConnect Local Switching ให้ AP Switch
   Traffic ออก VLAN Data (40) เองที่ Local จึงต้องเปิด Trunk ให้ทั้ง VLAN Control และ VLAN Data
   ผ่านได้
3. AP SSO คือ HA Pair เดียวที่ Sync State (CAPWAP Session, Client Database) แบบ Real-time
   ผ่าน Redundancy Port ตลอดเวลา ทำให้ Standby พร้อมรับงานต่อทันทีโดยไม่ต้องให้ AP/Client เริ่ม
   ใหม่ ส่วน N+1 HA คือ WLC อิสระที่ไม่ Sync State กัน AP จึงต้อง Discovery/Join ใหม่ทั้งหมดกับ
   WLC สำรอง ทำให้ Client หลุดและต้อง Re-associate
4. 802.11k ให้ Client รู้ AP ข้างเคียงที่น่า Roam ไปโดยไม่ต้อง Scan สุ่ม, 802.11v ให้ WLC/AP
   แนะนำหรือสั่งให้ Client ย้ายไปยัง AP ที่เหมาะสมแบบ Proactive (Load Balancing), และ 802.11r
   ทำให้ขั้นตอน Roam จริงเร็วขึ้นด้วยการเตรียม Key ล่วงหน้าไม่ต้อง Re-authenticate เต็มรูปแบบ
5. Overlap ที่มากเกินไปทำให้ AP หลายตัวที่อยู่ใกล้กันต้องแย่งใช้ Channel เดียวกันในพื้นที่
   ที่ทับซ้อนกันมาก เกิด Co-Channel Interference (CCI) สูง ลด Throughput โดยรวมของทั้งพื้นที่
   ลงไปมากกว่าที่จะได้ประโยชน์จาก Roaming ที่ราบรื่นขึ้นเพียงเล็กน้อย

---

## สรุป Part 36

Part นี้พา Wireless LAN Architecture ที่ Part 20 ปูพื้นฐานไว้ในระดับ CCNA ให้ลงลึกถึงระดับ
CCNP ENCOR อย่างเต็มรูปแบบ: กลไกจริงของ **CAPWAP** (DTLS Handshake, Control/Data Tunnel
Separation, Fragmentation, Keepalive), **FlexConnect** สำหรับ Branch Office พร้อมแปลง AP-2
เป็น Local Switching จริง, **RF Design เชิงวิศวกรรม** (RSSI/SNR/Cell Overlap/RRM), **Client
Roaming** ทั้ง L2/L3 พร้อม Mobility Group ระหว่าง WLC-1 กับ WLC-2, **Fast Roaming Protocol**
(802.11r/k/v) สำหรับ VoWLAN, **High Availability** ของ WLC เอง (N+1 HA เทียบกับ AP SSO ที่
Stateful จริง) พร้อมจับคู่ WLC-1 กับ WLC-1-STANDBY, สถาปัตยกรรม **Mobility Controller/Agent**
สำหรับ Scale ระดับองค์กรใหญ่, และปิดท้ายด้วย **RF Profile/AP Group** ที่แยกตามลักษณะพื้นที่จริง

สิ่งที่ Part 36 **ยังไม่ได้แตะ**: รายละเอียด Wireless Security เชิงลึก (802.1X Deep Dive,
WPA3-Enterprise 192-bit, Cisco ISE Integration, Rogue AP/WIDS/WIPS) — จะเรียนเต็มรูปแบบใน
**Part 37 (Wireless Security Advanced)** ต่อจากนี้

**ไปต่อ:** [Part 37 — Wireless Security Advanced →](part-037-wireless-security-advanced.md)
