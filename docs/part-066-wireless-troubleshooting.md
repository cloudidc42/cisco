# Part 66 — Wireless Troubleshooting
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 651–660 จาก 1000 | ระดับ CCNP ENARSI**

> ต่อจาก [Part 65 — Multicast Troubleshooting](part-065-multicast-troubleshooting.md) ที่เจาะ
> Troubleshoot PIM/IGMP/RP Failure ในระดับ ENARSI Part นี้จะพา Wireless LAN ที่ปูพื้นฐานไว้ตั้งแต่
> [Part 20](part-020-wireless-fundamentals.md) (CCNA — WLC/AP พื้นฐาน), ลงลึกสถาปัตยกรรมใน
> [Part 36](part-036-wireless-architecture-deep-dive.md) (CCNP ENCOR — CAPWAP, FlexConnect, RF
> Design, Roaming, WLC HA) และยกระดับความปลอดภัยใน [Part 37](part-037-wireless-security-advanced.md)
> (802.1X/WPA2-Enterprise, Dynamic VLAN, Guest CWA) กลับมาสู่มุมมอง **Troubleshooting เชิงระบบ**
> ตาม ENARSI Blueprint — Lab เดิมทั้งหมด (**WLC-1** Catalyst 9800-CL, **AP-1/AP-2** Catalyst 9130,
> SSID **ENTERPRISE-WIFI** (PSK), **ENTERPRISE-SECURE** (802.1X), **GUEST-WIFI** (CWA),
> FlexConnect บน AP-2, 802.11r Fast Roaming, WLC HA Pair กับ WLC-1-STANDBY) ยังคง Config เดิมทุก
> จุด — Part นี้จะจำลอง **8 สถานการณ์ที่พังจริงในสนามงาน** ทีละอย่าง พร้อมกระบวนการวินิจฉัยแบบเป็น
> ระบบ (Systematic Diagnosis) ที่ข้อสอบ ENARSI ชอบถามที่สุด: **"AP join WLC หรือยัง? ถ้า join แล้ว
> ปัญหาอยู่ที่ RF/Coverage หรือ Config? ถ้า Client associate ไม่ได้ ติดที่ Stage ไหน (Probe/Auth/
> Assoc/DHCP/RADIUS)? ถ้า associate ผ่านแล้ว Traffic ไหลจริงหรือไม่?"**

## สารบัญ Step ในเนื้อหา Part นี้

| Step | หัวข้อ |
|---|---|
| 651 | Wireless Troubleshooting Methodology — Layered Check ตั้งแต่ AP Join จนถึง Traffic Flow |
| 652 | สถานการณ์ 1 — AP-2 Join WLC-1 ไม่ได้หลัง Firmware Upgrade: DHCP Option 43 ชี้ IP ผิด |
| 653 | สถานการณ์ 2 — Client Associate ได้แต่ไม่ได้ IP: VLAN Mapping ผิดที่ Policy Profile |
| 654 | สถานการณ์ 3 — 802.1X Authentication ล้มเหลวเป็นพักๆ: RADIUS Overload หรือ Certificate หมดอายุ |
| 655 | สถานการณ์ 4 — FlexConnect AP-2 หลุดหมดตอน WAN ล่ม: Central DHCP ที่ยังไม่ได้ Local Switch |
| 656 | สถานการณ์ 5 — 802.11r Fast Roaming ไม่ทำงานหลัง Config Audit: FT AKM ถูกลบออกโดยไม่ตั้งใจ |
| 657 | สถานการณ์ 6 — WLC HA Failover ทำ AP Unregistered นานผิดปกติ: Redundancy Port Version Mismatch |
| 658 | สถานการณ์ 7 — Wi-Fi ช้าทั้งชั้น: Co-Channel Interference จาก Rogue AP กับ RRM ที่ถูก Override |
| 659 | สถานการณ์ 8 — Guest CWA Portal ค้างหน้าขาว: Pre-Auth ACL บล็อก DNS/Portal โดยไม่ตั้งใจ |
| 660 | Lab เต็มรูปแบบ — Help Desk Surge วันเดียวเจอทั้ง VLAN Mapping และ 802.11r พังพร้อมกัน |

---

## Step 651 — Wireless Troubleshooting Methodology: Layered Check ตั้งแต่ AP Join จนถึง Traffic Flow

### 651.1 ทำไม Wireless Troubleshooting ต้องมี "ลำดับ" ที่ตายตัว

Wired Troubleshooting (Part 56-63) มี Layer ที่ชัดเจนตาม OSI — แต่ **Wireless มี Layer เพิ่มเข้ามา
ก่อนถึง Layer 3 อีกหลายชั้น** ที่ไม่มีใน Wired เลย: RF Propagation, 802.11 Association State
Machine, และ CAPWAP Tunnel ระหว่าง AP กับ WLC ก่อนที่ Client แม้แต่จะเริ่ม DHCP ได้ — ถ้าไล่ตรวจ
ผิดจุด (เช่น ไปแก้ ACL ทันทีที่ได้ยินคำว่า "Wi-Fi ช้า" โดยไม่เช็คว่า AP Join WLC หรือยัง) จะเสียเวลา
มากและอาจแก้ผิดที่โดยไม่รู้ตัว — ENARSI Blueprint จึงต้องการให้จำ **ลำดับ Layer ที่ตายตัว** นี้ไว้
ก่อนลงมือทุกครั้ง

```
Layer 0: Physical/Power    — AP มีไฟ, สาย/PoE เข้าปกติ, Link Up ที่ Access Switch หรือยัง?
              │
              ▼
Layer 1: CAPWAP Join       — AP Join WLC-1 สำเร็จหรือยัง? (Discovery → DTLS → Join → Config → Run)
              │  (ดู Step 652 — ถ้าติดตรงนี้ ปัญหาไม่ใช่ RF เลย เป็นเรื่อง Discovery/Network)
              ▼
Layer 2: RF/Coverage       — Client เห็น SSID (Probe Response) ไหม? RSSI/SNR พอไหม?
              │  (ดู Step 658 — ถ้า AP Join แล้วแต่ Client เห็นสัญญาณอ่อน/ไม่เห็นเลย)
              ▼
Layer 3: 802.11 Assoc      — Client Authenticate/Associate ผ่านไหม? (Open/PSK/802.1X ตรงกันไหม)
              │  (ดู Step 654, 656 — Security Policy Mismatch, EAP Failure, FT ไม่ทำงาน)
              ▼
Layer 4: DHCP/VLAN         — Client ได้ IP ที่ VLAN ถูกต้องไหม?
              │  (ดู Step 653, 655 — VLAN Mapping ผิด, Central DHCP พังตอน WAN ล่ม)
              ▼
Layer 5: RADIUS/Policy     — ถ้าเป็น 802.1X/CWA — RADIUS ตอบถูกต้องไหม? ACL/Redirect ทำงานไหม?
              │  (ดู Step 654, 659)
              ▼
Layer 6: Traffic Flow      — Client ได้ IP แล้ว Ping/Application ใช้งานได้จริงไหม? (ACL/Routing/QoS)
```

### 651.2 คำถามตัดสินใจ (Decision Tree) ที่ต้องถามทุกครั้งก่อนลงมือแก้

| ลำดับ | คำถาม | ถ้า "ไม่" ให้ดูที่ | ถ้า "ใช่" ให้ไปคำถามถัดไป |
|---|---|---|---|
| 1 | AP ขึ้น `Registered` ใน `show ap summary` ไหม? | Layer 0-1 (Power/CAPWAP/DHCP Option 43) — **ยังไม่ต้องแตะ RF/Security เลย** | ข้อ 2 |
| 2 | Client เห็น SSID ใน Scan list ไหม? | Layer 2 (RF Coverage, Channel, Power, RF Profile ผิด AP Group) | ข้อ 3 |
| 3 | Client Authenticate/Associate สำเร็จไหม? (`show wireless client summary` ไม่มี Error State) | Layer 3 (Security Policy Mismatch, EAP Failure, Cert หมดอายุ, FT/AKM ผิด) | ข้อ 4 |
| 4 | Client ได้ IP Address ที่ VLAN ถูกต้องไหม? | Layer 4 (VLAN Mapping ผิดที่ Policy Profile, DHCP Relay/Pool, Central DHCP) | ข้อ 5 |
| 5 | (ถ้าใช้ 802.1X/CWA) RADIUS/Portal ตอบและ Redirect ถูกต้องไหม? | Layer 5 (RADIUS Down/Overload, Pre-Auth ACL ผิด) | ข้อ 6 |
| 6 | Traffic ไปถึงปลายทางจริงไหม? (Ping/Application) | Layer 6 (ACL ปลายทาง, Routing, QoS, Rate-Limit) | จบ — ไม่มีปัญหา |

### 651.3 คำสั่งหลักที่ใช้ในแต่ละ Layer (Cheat Sheet)

| Layer | คำสั่งหลักบน WLC-1 (Catalyst 9800 IOS-XE) |
|---|---|
| CAPWAP Join | `show ap summary`, `show ap uptime`, `show wireless stats capwap`, `show ap join stats summary` |
| RF/Coverage | `show ap dot11 5ghz summary`, `show wireless dot11 5ghz channel`, `show wireless client mac-address <mac> detail \| include RSSI\|SNR` |
| 802.11 Assoc/Security | `show wireless client summary`, `show wireless client mac-address <mac> detail`, `debug client mac-address <mac>` |
| DHCP/VLAN | `show wireless profile policy detailed <policy>`, `show ip dhcp binding` (บน Distribution Switch), `show wireless client mac-address <mac> detail \| include VLAN` |
| RADIUS/Policy | `show aaa servers`, `test aaa group <grp> <user> <pass> legacy`, `show wireless client mac-address <mac> detail \| include EAP\|Policy Manager State` |
| Traffic Flow | `ping`/`traceroute` จาก Client, `show ip access-lists`, `show wireless client mac-address <mac> detail \| include ACL` |

> **หลักการทองของ ENARSI Wireless Troubleshooting**: ไล่จาก **Layer ที่ใกล้ Physical ที่สุดก่อน
> เสมอ** — ถ้า AP ยังไม่ Join WLC เลย การไปเช็ค RADIUS หรือ ACL คือการเสียเวลาโดยเปล่าประโยชน์
> เพราะ Client ยังไม่มีทางเห็น SSID ตั้งแต่แรก 8 สถานการณ์ที่เหลือของ Part นี้แต่ละอันจะสะท้อน
> Layer หนึ่งใน Decision Tree นี้ให้เห็นภาพจริงชัดเจน

---

## Step 652 — สถานการณ์ 1: AP-2 Join WLC-1 ไม่ได้หลัง Firmware Upgrade

### 652.1 Background

ทีม Network ทำ Maintenance Window อัปเกรด Firmware บน **WLC-1** ตามรอบ Patch ประจำปี — เพื่อไม่
ให้ AP หลุดหมดระหว่างอัปเกรด จึงใช้แผน **Loaner Controller**: สลับ Traffic ของ AP ไปที่ WLC สำรอง
ชั่วคราว (IP `10.10.99.41`) ระหว่างที่ WLC-1 ตัวจริง (`10.10.99.40`) กำลัง Upgrade อยู่ และปรับ
**DHCP Option 43** ที่ DHCP Pool `MGMT-VLAN99` (ตามที่ตั้งไว้ใน [Part 20 Step 194]
(part-020-wireless-fundamentals.md)) ให้ชี้ไปที่ Loaner ตัวนั้นชั่วคราว หลัง Upgrade เสร็จทีมสลับ
กลับมาที่ WLC-1 ตัวจริงแล้ว — **แต่ลืมเปลี่ยน Option 43 กลับ**

**AP-1** ไม่ได้รับผลกระทบเพราะยังมี CAPWAP Session ที่ Join กับ WLC-1 ตัวจริงค้างอยู่ (ไม่ต้อง
Re-discover) แต่ **AP-2** ถูก Power-cycle ระหว่างการสลับ Physical Cable ที่ Branch (Part 36 Step
352 ทำให้ AP-2 เป็น FlexConnect ที่ Branch) จึงต้องวิ่งกระบวนการ **Discovery ใหม่ทั้งหมด**

### 652.2 Symptom

```
WLC-1# show ap summary
Number of APs: 1

AP Name    Slots  AP Model    Ethernet MAC     Radio MAC       Location   Country  IP Address     State
--------------------------------------------------------------------------------------------------------
AP-1       2      C9130AXI    aabb.cc00.1101   aabb.cc00.1102  default    TH       10.10.99.101    Registered

! AP-2 ไม่ปรากฏในรายการเลย — ทั้งที่ Power/Link ที่ ACCESS-SW4 ปกติดี
```

```
! ตรวจสอบที่ DIST-SW4 (Branch) ว่า AP-2 ได้ IP จาก DHCP หรือยัง
DIST-SW4# show ip dhcp binding | include 10.10.99.102
10.10.99.102    aabb.cc00.2101    Sep 26 2026 08:12 AM    Automatic

! AP-2 ได้ IP ปกติ — แปลว่า Layer 0-1 (Physical/DHCP) ผ่านหมดแล้ว ปัญหาต้องอยู่ที่ CAPWAP Discovery
```

```
! Syslog จาก AP-2 (ผ่าน Console/Local Log ก่อนที่ Link จะกลับมาปกติ)
*Sep 26 08:12:31.441: %CAPWAP-3-ERRORLOG: Not able to reach controller from DHCP Option 43: 10.10.99.41
*Sep 26 08:14:02.118: %CAPWAP-3-DISCFAILED: Failed to discover any WLC after all discovery types exhausted
```

### 652.3 Diagnosis

ตาม Decision Tree ใน Step 651.2 — ข้อ 1 คือ "ไม่": AP-2 ไม่ขึ้น Registered แต่ DHCP (Layer 0)
ผ่านแล้ว ปัญหาจึงอยู่ที่ **CAPWAP Discovery** เท่านั้น — ไล่ตรวจตามลำดับที่ [Part 20 Step 194]
(part-020-wireless-fundamentals.md) สอนไว้ (DHCP Option 43 คือวิธี Discovery หลักของ Lab นี้):

```
! ตรวจสอบค่า Option 43 จริงที่ DHCP Pool MGMT-VLAN99 (relay มาจาก Central ผ่าน ip helper-address)
DIST-SW3# show run | section ip dhcp pool MGMT-VLAN99
ip dhcp pool MGMT-VLAN99
 network 10.10.99.0 255.255.255.0
 default-router 10.10.99.1
 dns-server 10.10.99.53
 domain-name lab.local
 option 43 hex f104.0a0a.6329          ! <-- 0a0a.6329 = 10.10.99.41 (Loaner) ไม่ใช่ .40 (ตัวจริง)!
```

**คำนวณ Hex กลับ**: `0a.0a.63.29` = `10.10.99.41` — ยืนยันว่า Option 43 ยังชี้ไปที่ **Loaner WLC**
ที่ปิดไปแล้วหลัง Maintenance Window จบ ไม่ใช่ WLC-1 ตัวจริง (`10.10.99.40` = `0a0a.6328`)

### 652.4 Root Cause

**Stale DHCP Option 43** — ทีมงานสลับ AP ไปใช้ Loaner Controller ระหว่าง Firmware Upgrade แล้ว
**ลืม Revert Option 43 กลับ** เป็น IP ของ WLC-1 ตัวจริงหลัง Maintenance Window เสร็จ AP-1 ไม่มี
อาการเพราะยังไม่ได้ Re-discover (CAPWAP Session เดิมยังใช้งานได้) แต่ AP-2 ที่ถูก Power-cycle ต้อง
Discovery ใหม่ทั้งหมด และไปเจอ Option 43 ที่ผิดพลาดนี้เข้าเต็มๆ

### 652.5 Fix & Verify

```
DIST-SW3# configure terminal
DIST-SW3(config)# ip dhcp pool MGMT-VLAN99
DIST-SW3(dhcp-config)# option 43 hex f104.0a0a.6328     ! กลับไปชี้ WLC-1 ตัวจริง (10.10.99.40)
DIST-SW3(dhcp-config)# exit
DIST-SW3(config)# end
DIST-SW3# copy running-config startup-config

! บังคับให้ AP-2 ปล่อย Lease เดิมแล้วขอใหม่ (หรือ Power-cycle AP-2 อีกครั้ง)
DIST-SW3# clear ip dhcp binding 10.10.99.102
```

```
! ตรวจสอบผลลัพธ์หลัง AP-2 ขอ DHCP ใหม่และ Discover สำเร็จ (ใช้เวลาไม่เกิน 2-3 นาที)
WLC-1# show ap summary
Number of APs: 2

AP Name    Slots  AP Model    Ethernet MAC     Radio MAC       Location   Country  IP Address     State
--------------------------------------------------------------------------------------------------------
AP-1       2      C9130AXI    aabb.cc00.1101   aabb.cc00.1102  default    TH       10.10.99.101    Registered
AP-2       2      C9130AXI    aabb.cc00.2101   aabb.cc00.2102  default    TH       10.10.99.102    Registered
```

> **บทเรียนสำหรับ ENARSI**: ทุกครั้งที่ทำ Maintenance ที่เปลี่ยน "จุดที่ AP ใช้ Discover WLC"
> ชั่วคราว (Loaner Controller, IP Migration, DNS Record) ต้องมี **Checklist Revert กลับ** เป็น
> ขั้นตอนบังคับก่อนปิด Maintenance Window เสมอ — AP ที่ไม่ได้ Reboot ระหว่างนั้นจะไม่มีอาการทันที
> (เพราะ CAPWAP Session เดิมยังอยู่) ทำให้ปัญหาแบบนี้ "ซ่อนตัว" จนกว่าจะมี AP ตัวใดตัวหนึ่ง Reload
> ในอนาคต — เป็นกับดักคลาสสิกที่ผ่านไปเป็นสัปดาห์กว่าจะมีคนสังเกตเห็น

---

## Step 653 — สถานการณ์ 2: Client Associate ได้แต่ไม่ได้ IP — VLAN Mapping ผิดที่ Policy Profile

### 653.1 Background

ทีม Network ทำโครงการ "VLAN Cleanup" เพื่อจัดระเบียบ VLAN ที่เพิ่มขึ้นมาเรื่อยๆ ตั้งแต่ Part 37
(VLAN 45 Contractor, VLAN 50 Guest) — ระหว่างแก้ไข Policy Profile หลายตัวพร้อมกันแบบ Bulk ผ่าน
Script ที่เขียนขึ้นมาเอง มีจุดหนึ่งที่ Copy-paste ผิดพลาด ทำให้ **`ENTERPRISE-WIFI-POLICY`**
(Policy Profile ของ SSID หลัก ENTERPRISE-WIFI จาก Part 20) ถูกเปลี่ยน VLAN จาก 40 (WIFI) ไปเป็น
**45 (CONTRACTOR-QUARANTINE)** โดยไม่ตั้งใจ

### 653.2 Symptom

Helpdesk ได้รับแจ้งจากพนักงานหลายคนพร้อมกันว่า "เชื่อมต่อ Wi-Fi ENTERPRISE-WIFI ได้ (เห็น Password
ถูก ไม่มี Error) แต่เปิดเว็บไม่ได้เลย ไม่มี Internet" — สังเกตว่า Client **Associate สำเร็จ** (ไม่ใช่
ปัญหา RF หรือ Security ตาม Decision Tree ข้อ 3) จึงต้องดูข้อ 4 ต่อ

```
WLC-1# show wireless client summary

Number of Local Clients: 1

MAC Address       AP Name    WLAN    State        Protocol
------------------------------------------------------------
aabb.ccdd.9911     AP-1       1       Associated   11ax
```

```
! เจาะดูรายละเอียด Client ตัวนี้ — จุดสำคัญที่สุดของสถานการณ์นี้
WLC-1# show wireless client mac-address aabb.ccdd.9911 detail
Client MAC Address                          : aabb.ccdd.9911
Client IPv4 Address                         : Unknown                  <- ยังไม่มี IP เลย!
AP Name                                     : AP-1
BSSID                                       : aabb.cc00.1102
Policy Profile                              : ENTERPRISE-WIFI-POLICY
Wireless LAN Id                             : 1
Wireless LAN Name                           : ENTERPRISE-WIFI
Security Policy Completed                   : Yes                     <- WPA2-PSK ผ่านแล้ว
Policy Manager State                        : IP LEARN IN PROGRESS     <- ค้างอยู่ตรงนี้
VLAN                                        : 45                       <- ผิด! ควรเป็น 40
```

**จุดสังเกตสำคัญที่สุด**: `Security Policy Completed : Yes` และ `AP Name`/`BSSID` ถูกต้องครบ — นี่
คือหลักฐานว่า Layer 2-3 (RF/Association) **ไม่มีปัญหาเลย** — แต่ `VLAN : 45` คือตัวบอกโดยตรงว่า
Client กำลังถูก Map เข้า VLAN ผิด (Contractor Quarantine) ซึ่งที่ ACCESS-SW3 ไม่มี DHCP Pool/Trunk
ที่ผูกกับ AP-1 ในทางที่ Client จะขอ IP ผ่าน Centralized Mode ได้ตามที่ตั้งใจไว้จริง

### 653.3 Root Cause

```
! ตรวจสอบ Policy Profile ต้นตอ
WLC-1# show wireless profile policy detailed ENTERPRISE-WIFI-POLICY | include VLAN
  VLAN                                     : 45     <- ควรเป็น 40 (WIFI) ตาม docs/00-ip-address-plan.md

! ตรวจ Config จริงยืนยัน
WLC-1# show run | section wireless profile policy ENTERPRISE-WIFI-POLICY
wireless profile policy ENTERPRISE-WIFI-POLICY
 vlan 45
 no shutdown
```

Script Bulk-edit ที่ใช้ระหว่างโครงการ "VLAN Cleanup" รัน `vlan 45` ทับ `vlan 40` เดิมบน
`ENTERPRISE-WIFI-POLICY` โดยพลาด (ตั้งใจจะแก้แค่ Policy Profile ของ Contractor แต่ Script ลูป
ผิดตัวแปร) — Client จึงถูก WLC-1 ยัดเข้า VLAN 45 (CONTRACTOR-QUARANTINE, `10.10.45.0/24`) ที่มี
ACL จำกัดสิทธิ์ตาม [Part 37 Step 366](part-037-wireless-security-advanced.md) แทน VLAN 40 ที่
ถูกต้อง — และเพราะ DHCP Scope/ACL ของ VLAN 45 ถูกออกแบบมาสำหรับ Contractor เท่านั้น (ไม่ครอบคลุม
การใช้งานพนักงานปกติ) Client จึงค้างที่ `IP LEARN IN PROGRESS` ไม่ได้ IP ที่ใช้งานได้จริงเลย

### 653.4 Fix & Verify

```
WLC-1# configure terminal
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY
WLC-1(config-wireless-policy)# vlan 40                      ! แก้กลับเป็น VLAN 40 (WIFI) ตาม IP Plan
WLC-1(config-wireless-policy)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config

! บังคับ Client Reassociate ใหม่เพื่อรับ VLAN ที่ถูกต้อง (ไม่ต้องรอ Client เดินหนีสัญญาณ)
WLC-1# wireless client mac-address aabb.ccdd.9911 delete
```

```
! ตรวจสอบผลลัพธ์หลัง Client เชื่อมต่อใหม่
WLC-1# show wireless client mac-address aabb.ccdd.9911 detail | include VLAN|IPv4|State
Client IPv4 Address                         : 10.10.40.118
VLAN                                        : 40
Policy Manager State                        : RUN
```

> **บทเรียนสำหรับ ENARSI**: `show wireless client mac-address <mac> detail` คือคำสั่งเดียวที่บอก
> ครบทุก Layer ในบรรทัดเดียวกัน (Security Policy Completed, VLAN, Policy Manager State, IPv4
> Address) — เมื่อ `Security Policy Completed : Yes` แต่ `Client IPv4 Address : Unknown` แปลว่า
> ปัญหาไม่ใช่ Security/RF เด็ดขาด **ต้องดูที่ VLAN Mapping กับ DHCP ทันที** ไม่ต้องเสียเวลาไล่ Debug
> 802.11 Association ซ้ำอีก

---

## Step 654 — สถานการณ์ 3: 802.1X Authentication ล้มเหลวเป็นพักๆ — RADIUS Overload หรือ Certificate หมดอายุ

### 654.1 Background

SSID **ENTERPRISE-SECURE** (WPA2-Enterprise/EAP-TLS ตาม [Part 37 Step 363-365]
(part-037-wireless-security-advanced.md)) ถูก Deploy ให้ Laptop บริษัทประมาณ 150 เครื่องที่ Enroll
Certificate ผ่าน SCEP พร้อมกันเมื่อปีที่แล้วในโครงการ "New Hire Onboarding Day" — เช้าวันหนึ่ง
Helpdesk ได้รับแจ้งพร้อมกันหลายสิบ Ticket ว่า Laptop "เชื่อมต่อ Wi-Fi ไม่ได้ Authentication
Failed" แต่ **ไม่ใช่ทุกเครื่อง** — ประมาณ 30% ของ User เท่านั้นที่มีอาการ ส่วนที่เหลือใช้งานปกติ

### 654.2 Diagnosis — ตัดตัวแปรที่เป็นไปได้ทีละข้อ

**สมมติฐานที่ 1: RADIUS Server (AAA-SRV) Overload** — ตรวจสอบก่อนเพราะกระทบวงกว้างได้เร็วที่สุด

```
WLC-1# show aaa servers | include AAA-SRV-RADIUS
RADIUS: id 1, priority 1, host 10.10.99.60, auth-port 1812, acct-port 1813
     State: current UP, duration 315000s, previous duration 0s
     Dead: total time 0s, count 0
     Quarantined: No
     Authen: request 4821, timeouts 3, failover 0, retransmission 3
     Response: accept 4790, reject 25, challenge 3
     Average response time: 0.082s      <- ปกติ ไม่ช้าผิดปกติ

! ทดสอบยิง Request ตรงไป RADIUS โดยไม่ผ่าน Client จริง
WLC-1# test aaa group AAA-RADIUS-GRP testuser Test@Pass2026! legacy
Attempting authentication test to server-group AAA-RADIUS-GRP using radius
User was successfully authenticated.
```

**ผลสรุป**: `timeouts 3` จาก Request กว่า 4800 ครั้ง (~0.06%) และ Response Time เฉลี่ย 0.08 วินาที
ถือว่าปกติมาก — **ตัดสมมติฐาน RADIUS Overload ออกไปได้** (ไม่ใช่ต้นเหตุของ 30% ที่ Fail)

**สมมติฐานที่ 2: Client Certificate หมดอายุ** — เจาะดู Client ตัวอย่างที่มีอาการ

```
WLC-1# show wireless client mac-address ccdd.1234.aa01 detail | include EAP|Policy Manager State
EAP Type                                    : EAP-TLS
Policy Manager State                        : AUTHCHECK           <- ค้าง ไม่ไปถึง RUN

! Syslog บน WLC-1 ที่ตรง MAC เดียวกัน
*Sep 26 09:02:11.220: %DOT1X-5-FAIL: Authentication failed for client ccdd.1234.aa01
*Sep 26 09:02:11.221: %RADIUS-3-ALLDEADSERVER: (ไม่ปรากฏ — ยืนยันว่า RADIUS ไม่ได้ล่ม)
*Sep 26 09:02:11.225: [dot1x] ccdd.1234.aa01 EAP-TLS alert received: certificate_expired (code 45)
```

`certificate_expired (code 45)` คือ **TLS Alert มาตรฐาน** ที่ยืนยันโดยตรงว่า Certificate ฝั่ง
Client หมดอายุแล้ว — ตรวจสอบวันที่ออก Certificate เทียบกับวันนี้:

| รายการ | ค่า |
|---|---|
| วันที่ Enroll Certificate (New Hire Onboarding Day ปีที่แล้ว) | 26 กันยายน 2568 (2025) |
| อายุ Certificate ที่ Template กำหนด | **365 วัน** |
| วันหมดอายุ | **26 กันยายน 2569 (2026) — วันนี้!** |
| จำนวน User ที่ Enroll วันเดียวกันทั้งหมด | ~45 คน (ตรงกับสัดส่วน ~30% ของ 150 เครื่องที่มีอาการ) |

### 654.3 Root Cause

**Certificate หมดอายุพร้อมกันเป็นชุด (Batch Expiry)** — เพราะ Client Certificate ทั้ง 45 ใบถูก
Enroll วันเดียวกันในโครงการ Onboarding เมื่อปีที่แล้ว และ Template ตั้งอายุ Certificate ไว้พอดี
365 วัน (ไม่มี Auto-Renewal ผ่าน SCEP ล่วงหน้าอย่างที่ [Part 37 Step 365]
(part-037-wireless-security-advanced.md) เตือนไว้ว่าองค์กรส่วนใหญ่ "เจ็บ" ที่สุดตรงจุดนี้เมื่อ Scale
ใหญ่ขึ้น) — Certificate ทุกใบจึงหมดอายุพร้อมกันในวันเดียว ทำให้ EAP-TLS Handshake Fail ที่ขั้น
Certificate Validation ก่อนจะถึง RADIUS Policy Decision ด้วยซ้ำ (ไม่เกี่ยวกับ RADIUS Server เลย
ตามที่ตัดสมมติฐานไปแล้วในขั้นตอนก่อนหน้า)

### 654.4 Fix & Verify

แก้ปัญหาระยะสั้น (คืน Access ให้ User ที่ได้รับผลกระทบทันที) และระยะยาว (ป้องกันไม่ให้เกิดซ้ำ):

```
! ระยะสั้น: Trigger SCEP Re-enrollment ทันทีผ่าน MDM ให้กับ 45 เครื่องที่ได้รับผลกระทบ
! (คำสั่งฝั่ง MDM — Concept เท่านั้น ไม่ผูก Platform เฉพาะ)
MDM-Console> push-policy --group "Onboarding-Batch-2025" --action force-scep-renew

! ระยะยาว: ตั้ง SCEP Auto-Renewal Threshold ให้ Renew ล่วงหน้า 30 วันก่อนหมดอายุเสมอ
! (ป้องกัน Batch Expiry ซ้ำในอนาคต — Concept เดียวกับที่ Part 37 Step 365 เตือนไว้)
MDM-Console> set-scep-policy --renew-before-expiry-days 30
```

```
! ตรวจสอบผลลัพธ์หลัง Certificate Renew สำเร็จ
WLC-1# show wireless client mac-address ccdd.1234.aa01 detail | include EAP|Policy Manager State|VLAN
EAP Type                                    : EAP-TLS
Policy Manager State                        : RUN
VLAN                                        : 40
```

> **บทเรียนสำหรับ ENARSI**: เมื่อ 802.1X Fail เป็น "พักๆ" หรือ "เป็นกลุ่ม" (ไม่ใช่ 100% หรือ 0%)
> ให้สงสัย **Certificate Lifecycle** หรือ **RADIUS Server Load/Health** เป็นอันดับแรก — วิธีแยก
> สองสมมติฐานนี้คือดู `show aaa servers` (Response Time/Timeout Count ผิดปกติไหม) เทียบกับ
> `show wireless client mac-address <mac> detail` ของ Client ตัวอย่างที่ Fail (มี EAP Alert Code
> เจาะจงไหม) — Batch Expiry เป็นสาเหตุที่พบบ่อยมากในองค์กรที่ Deploy EAP-TLS แบบ Bulk Enrollment
> ครั้งเดียวโดยไม่มี Auto-Renewal

---

## Step 655 — สถานการณ์ 4: FlexConnect AP-2 หลุดหมดตอน WAN ล่ม — Central DHCP ที่ยังไม่ได้ Local Switch

### 655.1 Background

[Part 36 Step 352](part-036-wireless-architecture-deep-dive.md) แปลง AP-2 เป็น **FlexConnect
Mode** พร้อม **Local Switching VLAN 40** เพื่อไม่ให้ Client Data ต้อง Hair-pin ผ่าน WAN — Config
เดิมตั้งไว้ว่า `no central switching` (Data Plane Local) แต่ยังคง `central authentication` และ
**`central dhcp`** ไว้ (WLC-1 ยังตัดสินใจเรื่อง Authentication และยัง Relay DHCP ให้จากศูนย์กลาง)
ทีม Network มีคืน Maintenance ที่ต้องตัด Fiber WAN ระหว่าง `CORE-SW2` กับ `WAN-EDGE-2` เพื่อ
เปลี่ยนอุปกรณ์ ใช้เวลาประมาณ 45 นาที — คาดหวังว่า Client ที่ Branch (ผ่าน AP-2) จะยังใช้งาน Local
Resource ได้ต่อเนื่องตามหลักการ FlexConnect Standalone Mode ที่ Part 36 สอนไว้

### 655.2 Symptom

ผลจริงกลับแย่กว่าที่คาด — **Client ทุกตัวที่ AP-2 หลุดหมดภายใน 5-10 นาทีแรกของ Maintenance
Window** ไม่ใช่แค่ Client ใหม่ที่พยายาม Join ระหว่างนั้น แต่รวมถึง Client ที่ Associate อยู่ก่อน
แล้วด้วย — ตรงข้ามกับที่ Part 36 อธิบายไว้ว่า FlexConnect Local Switching ควรทำให้ Client ที่ใช้
Local Resource ยังทำงานต่อได้แม้ WAN ล่ม

หลัง Fiber กลับมาใช้งานได้และ AP-2 Rejoin WLC-1 สำเร็จ ทีมตรวจสอบ Log/History เพื่อหาสาเหตุ:

```
! ตรวจสอบสถานะ FlexConnect ของ AP-2 (หลัง Link กลับมาแล้ว)
WLC-1# show ap name AP-2 flexconnect
AP Name                                       : AP-2
Flex Profile Name                             : FLEX-BRANCH-PROFILE
Central DHCP                                  : Enabled            <- ตัวการหลัก!
Central Switching                             : Disabled
VLAN Name : WIFI, VLAN Id : 40

! Syslog ที่บันทึกไว้บน ACCESS-SW4 ระหว่างช่วง WAN ล่ม (จาก Local Syslog Buffer)
*Sep 26 01:15:02: %DHCP-4-DECLINE_CONFLICT: (ไม่ปรากฏ) ...
*Sep 26 01:16:40: (Client บน AP-2 ส่ง DHCPREQUEST ต่ออายุ Lease แต่ไม่มี DHCPACK ตอบกลับเลย)
```

### 655.3 Root Cause

ตรวจสอบ Config เดิมที่ [Part 36 Step 352](part-036-wireless-architecture-deep-dive.md) ตั้งไว้
พบว่า Policy Profile `ENTERPRISE-WIFI-POLICY-FLEX` เปิด **`central dhcp`** ไว้ (ตั้งใจไว้ตอนแรก
เพื่อให้ IT ที่ Central Site มองเห็น DHCP Lease ของ Branch ทั้งหมดจากจุดเดียว) — ค่านี้หมายความว่า
**แม้ Data Plane จะ Local Switch แล้ว แต่ DHCP Request/Renew ของทุก Client ยังต้องเดินทางผ่าน
CAPWAP กลับไปหา WLC-1 เพื่อขอ Relay อยู่ดี** — เมื่อ WAN ล่ม CAPWAP Tunnel (ทั้ง Control และ Data)
ขาดหายไปด้วย ทำให้ Client ตัวใดก็ตามที่ DHCP Lease Timer ครบกำหนด Renew ระหว่างนั้น (Default DHCP
Lease Time ส่วนใหญ่ไม่เกิน 24 ชั่วโมง — โอกาสชนกับหน้าต่าง Maintenance 45 นาทีนี้สูงมากถ้ามี
Client จำนวนมาก) จะไม่ได้รับ DHCPACK และหลุด IP ไปในที่สุด แม้ว่า 802.11 Data Path ของมันจะยัง
Local Switch ได้ตามปกติทุกอย่างก็ตาม

นี่คือสาเหตุที่ทำให้ผลลัพธ์ "แย่กว่าที่คาด" — วิศวกรที่ตั้งค่าไว้ตั้งแต่ Part 36 เข้าใจว่า Local
Switching อย่างเดียวเพียงพอสำหรับ Resilience ตอน WAN ล่ม แต่ลืมว่า **DHCP เป็นคนละ Setting กับ
Switching** บน Policy Profile ของ Catalyst 9800

### 655.4 Fix & Verify

```
WLC-1# configure terminal
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY-FLEX
WLC-1(config-wireless-policy)# no central dhcp                    ! เปลี่ยนเป็น Local DHCP
WLC-1(config-wireless-policy)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

```
! ต้องมี DHCP Pool สำหรับ VLAN 40 อยู่ที่ DIST-SW4 (Branch) เองด้วย ไม่ใช่พึ่ง Relay ไปศูนย์กลางแล้ว
DIST-SW4# configure terminal
DIST-SW4(config)# ip dhcp pool WIFI-VLAN40-BRANCH
DIST-SW4(dhcp-config)# network 10.10.40.0 255.255.255.0
DIST-SW4(dhcp-config)# default-router 10.10.40.1
DIST-SW4(dhcp-config)# dns-server 10.10.99.53
DIST-SW4(dhcp-config)# exit
DIST-SW4(config)# end
DIST-SW4# copy running-config startup-config
```

```
! ตรวจสอบผลลัพธ์
WLC-1# show ap name AP-2 flexconnect | include Central DHCP|Central Switching
Central DHCP                                  : Disabled
Central Switching                             : Disabled

! จำลอง WAN ล่มอีกครั้ง (Maintenance ครั้งถัดไป) — Client ที่ Renew DHCP ระหว่างนั้นยังได้ IP ปกติ
! จาก DIST-SW4 Local Pool โดยไม่ต้องพึ่ง WLC-1 เลย
```

> **บทเรียนสำหรับ ENARSI**: FlexConnect มี Setting แยกกัน **3 ชั้น** ที่ต้องตรวจสอบให้ครบทุกครั้ง
> — **Switching** (Data Path Local/Central), **Authentication** (Central/Local), และ **DHCP**
> (Central/Local) — การเปิด Local Switching อย่างเดียวโดยยังเหลือ Central DHCP หรือ Central
> Authentication ไว้ จะทำให้ Resilience ตอน WAN ล่ม "ไม่สมบูรณ์" อย่างที่คาดไว้ ต้องออกแบบทั้ง 3
> ชั้นให้สอดคล้องกับ Requirement ของ Branch จริงเสมอ

---

## Step 656 — สถานการณ์ 5: 802.11r Fast Roaming ไม่ทำงานหลัง Config Audit — FT AKM ถูกลบโดยไม่ตั้งใจ

### 656.1 Background

[Part 36 Step 355](part-036-wireless-architecture-deep-dive.md) เปิด **802.11r (FT — Fast BSS
Transition)** บน SSID ENTERPRISE-WIFI ไว้แล้ว (`security ft over-the-ds`) เพื่อรองรับ VoWLAN/
Real-time Application ที่ต้องการ Roaming ไม่เกิน 50ms — ทีม Security ทำ **Config Audit ประจำ
ไตรมาส** เพื่อตรวจสอบ WLAN Security ทุกตัวให้ตรงตาม Standard และรัน Script ที่ Normalize AKM
Setting ของทุก WLAN ให้เป็นรูปแบบเดียวกัน — ระหว่างนั้น Script ไปสั่ง `no security ft` บน WLAN
`ENTERPRISE-WIFI-PROFILE` โดยไม่ตั้งใจ (Logic ของ Script เข้าใจผิดว่า `ft` เป็นส่วนหนึ่งของ AKM ที่
ต้อง Normalize ทิ้งแล้ว Enable ใหม่ตาม Template แต่ Template ไม่มีบรรทัด `security ft` อยู่เลย)

### 656.2 Symptom

ทีม Facility แจ้งว่า VoWLAN Phone (Softphone บน Laptop ที่เดินระหว่างห้องประชุม) มีอาการ "เสียง
กระตุกทุกครั้งที่เดินข้ามพื้นที่ AP-1 ไป AP-2" — ทดสอบ Roaming Time ด้วยเครื่องมือ Wireless
Analyzer พบว่าใช้เวลานานกว่าที่ควรมาก:

```
Roaming Test Result (ก่อนแก้ไข):
  Client Laptop เดินจาก AP-1 (RSSI ลดลง) ไป AP-2 (RSSI เริ่มแรงขึ้น)
  Roaming Time ที่วัดได้: 187ms   <- ควรอยู่ที่ 10-20ms ถ้า 802.11r ทำงานถูกต้อง
```

```
! ตรวจสอบสถานะ 802.11r บน WLAN จริง
WLC-1# show wlan name ENTERPRISE-WIFI-PROFILE | include 802.11r|FT|Fast Transition
  802.11r Fast Transition                   : Disabled            <- ควรเป็น Enabled!
  Over the DS                               : Disabled
```

```
! ยืนยันด้วย debug ระหว่าง Client Roam จริง — เห็น Full Re-authentication ไม่ใช่ FT
WLC-1# debug client mac-address aabb.ccdd.7744
*Sep 26 11:05:20.100: [client-orch-state] aabb.ccdd.7744 Roaming detected: AP-1 -> AP-2
*Sep 26 11:05:20.105: [dot1x] aabb.ccdd.7744 Starting FULL 4-Way Handshake      <- ไม่ใช่ FT!
*Sep 26 11:05:20.287: [client-orch-state] aabb.ccdd.7744 Moving to RUN state
```

### 656.3 Root Cause

```
! เปรียบเทียบ Config ปัจจุบันกับที่ Part 36 Step 355 ตั้งไว้
WLC-1# show run | section wlan ENTERPRISE-WIFI-PROFILE
wlan ENTERPRISE-WIFI-PROFILE 1 ENTERPRISE-WIFI
 security wpa akm dot1x
 security wpa wpa2
 no shutdown
 !  <- ไม่มีบรรทัด "security ft over-the-ds" อยู่เลย — ถูกลบไปแล้ว
```

Config Audit Script ของทีม Security รัน `no security ft` บน `ENTERPRISE-WIFI-PROFILE` โดยเข้าใจ
ผิดว่าเป็นส่วนหนึ่งของ AKM ที่ต้อง Normalize — ผลคือ Client ทุกตัวที่ Roam ระหว่าง AP-1/AP-2 ต้อง
ทำ **Full Re-authentication ใหม่ทั้งหมด** (4-Way Handshake เต็มรูปแบบ) ทุกครั้งที่ Roam แทนที่จะ
ใช้ Pre-computed Key ตามแนวคิด Fast Transition ที่ [Part 36 Step 355]
(part-036-wireless-architecture-deep-dive.md) อธิบายไว้ — ทำให้ Roaming Time เพิ่มจาก
10-20ms เป็นเกือบ 200ms ซึ่งเกินขีดจำกัดของ VoWLAN (50ms) ไปมาก

### 656.4 Fix & Verify

```
WLC-1# configure terminal
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# security ft over-the-ds              ! เปิด FT กลับคืนตามที่ Part 36 ตั้งไว้เดิม
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

```
! ตรวจสอบผลลัพธ์
WLC-1# show wlan name ENTERPRISE-WIFI-PROFILE | include 802.11r|FT|Fast Transition
  802.11r Fast Transition                   : Enabled
  Over the DS                               : Enabled

! ทดสอบ Roaming ซ้ำ
Roaming Test Result (หลังแก้ไข):
  Roaming Time ที่วัดได้: 14ms    <- กลับมาอยู่ในเกณฑ์ที่ VoWLAN ต้องการแล้ว
```

> **บทเรียนสำหรับ ENARSI**: Config Audit/Normalization Script ที่เขียนขึ้นมาเองมีความเสี่ยงสูงที่
> จะ "แก้เกินขอบเขต" ถ้าไม่เข้าใจว่าแต่ละ Keyword ทำหน้าที่อะไรจริงๆ — `security ft` เป็นฟีเจอร์
> **Roaming Optimization แยกจาก AKM โดยสิ้นเชิง** (AKM คือวิธี Key Management หลัก เช่น `dot1x`/
> `psk`, ส่วน `ft` คือ Fast Transition Extension ที่ทำงานร่วมกับ AKM ตัวไหนก็ได้) ทุกครั้งที่ทำ
> Audit ควรมี **Before/After Diff ของทุก `show wlan` ที่สำคัญ** เก็บไว้เทียบ ไม่ใช่เชื่อ Script
> อย่างเดียวโดยไม่ Verify ผลจริงหลังรัน

---

## Step 657 — สถานการณ์ 6: WLC HA Failover ทำ AP Unregistered นานผิดปกติ — Redundancy Port Version Mismatch

### 657.1 Background

[Part 36 Step 356](part-036-wireless-architecture-deep-dive.md) จับคู่ **WLC-1** กับ
**WLC-1-STANDBY** เป็น **AP SSO HA Pair** ผ่าน Redundancy Port — จุดขายหลักของ AP SSO คือ
Failover แบบ **Stateful**: AP ไม่ต้อง Rejoin เลยตอน Active ล่ม เพราะ CAPWAP Session ถูก Sync ไว้
ที่ Standby ล่วงหน้าตลอดเวลา ทีมงานทำ **Failover Drill** ตามรอบ DR Test ประจำครึ่งปี (สั่ง Reload
WLC-1 Active ด้วยมือเพื่อทดสอบว่า Standby รับช่วงต่อได้จริง) — ผลที่คาดหวังคือ AP ไม่มี Downtime
เลยหรือมีน้อยมาก (ระดับ Sub-second ถึงไม่กี่วินาที)

### 657.2 Symptom

ผลจริงกลับพบว่า AP-1/AP-2 หลุดจากสถานะ Registered ไปนานถึง **4 นาที** ก่อนกลับมา Registered
ใหม่ — ไม่ตรงกับพฤติกรรม Stateful Switchover ที่ควรจะเป็น

```
! ตรวจสอบสถานะ HA Pair ทันทีหลัง Failover
WLC-1-STANDBY# show redundancy states
       my state = 13 -ACTIVE                  <- ขึ้นเป็น Active สำเร็จ (ตัวเดิม WLC-1 ล่มไปแล้ว)
     peer state = 1  -DISABLED                <- WLC-1 (เดิม Active) ยังไม่กลับมา (คาดไว้)
           Mode = Simplex                     <- ผิดปกติ! ควรเป็น Duplex ตอน HA ทำงานสมบูรณ์
     Manual Swact = disabled

! ตรวจสอบ AP ระหว่างที่ WLC-1-STANDBY เป็น Active แล้ว
WLC-1-STANDBY# show ap summary
Number of APs: 0                              <- AP ทั้งหมดหายไปหมด! ไม่ตรงกับ AP SSO ที่ควร Sync ไว้
```

### 657.3 Root Cause

```
! ตรวจสอบ Version ของทั้งสองตัวหลังกู้คืน WLC-1 ขึ้นมาได้แล้ว
WLC-1# show version | include Version
Cisco IOS XE Software, Version 17.12.3

WLC-1-STANDBY# show version | include Version
Cisco IOS XE Software, Version 17.12.5      <- ไม่ตรงกัน!
```

ตรวจ Log ประวัติ Patch พบว่าทีมงานทำ **Quick Security Patch** บน `WLC-1-STANDBY` แยกตัวเดียวเมื่อ
สัปดาห์ก่อน (เข้าใจว่าเป็น "ตัวสำรอง แก้ก่อนได้ไม่กระทบ Production") โดยไม่ได้อัปเกรด `WLC-1`
(Active) ให้ตรงกันตามรอบ Maintenance ปกติที่วางแผนไว้ — **AP SSO กำหนดไว้ชัดเจนว่า WLC ทั้งคู่ต้อง
เป็น Platform/Software Version เดียวกันเป๊ะ** (ตามตารางเปรียบเทียบใน [Part 36 Step 356]
(part-036-wireless-architecture-deep-dive.md)) เมื่อ Version ไม่ตรงกัน HA Pair จะไม่สามารถทำ
Stateful Switchover ได้จริง (Mode กลายเป็น `Simplex` ไม่ใช่ `Duplex`) แม้ Redundancy Port จะยัง Up
อยู่ก็ตาม — ทำให้ Client/AP State ที่ควร Sync ไว้ล่วงหน้าไม่ได้ถูก Sync จริง Standby ที่ขึ้นมาเป็น
Active จึงกลายเป็นเหมือน WLC ตัวใหม่ที่ไม่รู้จัก AP เลย และ AP ต้องผ่านกระบวนการ Discovery + Join
ใหม่ทั้งหมด (พฤติกรรมแบบ N+1 HA ไม่ใช่ AP SSO) จึงใช้เวลานานถึง 4 นาที

### 657.4 Fix & Verify

```
! ต้องอัปเกรด WLC-1 (Active ที่กู้คืนมา) ให้เป็น Version เดียวกับ WLC-1-STANDBY ก่อนจับคู่ HA ใหม่
WLC-1# install add file bootflash:cat9k_iosxe.17.12.05.SPA.bin activate commit
% Reload อัตโนมัติหลัง Install สำเร็จ

! หลัง Reload ตรวจสอบ Version ทั้งคู่ตรงกันแล้ว
WLC-1# show version | include Version
Cisco IOS XE Software, Version 17.12.5

WLC-1-STANDBY# show version | include Version
Cisco IOS XE Software, Version 17.12.5

! ตรวจสอบ HA Pair กลับมาเป็น Duplex/AP SSO สมบูรณ์
WLC-1# show redundancy states
       my state = 13 -ACTIVE
     peer state = 8  -STANDBY HOT
           Mode = Duplex                       <- กลับมาถูกต้องแล้ว

WLC-1# show ap redundancy summary
AP SSO State                                  : Enabled
AP Client SSO                                 : Enabled
```

```
! ทดสอบ Failover ซ้ำอีกครั้งเพื่อยืนยัน — AP ควรไม่หลุดสถานะ Registered เลย
WLC-1-STANDBY# show ap summary
Number of APs: 2
AP Name    State
--------------------
AP-1       Registered      <- ไม่มี Gap เลยตอนนี้ (Stateful Switchover ทำงานจริง)
AP-2       Registered
```

> **บทเรียนสำหรับ ENARSI**: "ตัวสำรองแก้ก่อนได้ไม่กระทบ Production" เป็นความเข้าใจผิดที่พบบ่อยมาก
> สำหรับระบบ HA แบบ **Stateful** (AP SSO, HSRP กับ Version-sensitive Feature, Stack ที่ต้อง
> Software เดียวกัน) — การ Patch ข้าง Standby ฝั่งเดียวโดยไม่ Sync Active พร้อมกันจะทำให้ HA Pair
> เสียคุณสมบัติ Stateful ไปทันที แม้จะดูเหมือนว่า "ทำงานได้" (Redundancy Port Up, Election สำเร็จ)
> ต้องตรวจสอบ `show redundancy states` ว่า `Mode = Duplex` เสมอ ไม่ใช่ดูแค่ Role Active/Standby

---

## Step 658 — สถานการณ์ 7: Wi-Fi ช้าทั้งชั้น — Co-Channel Interference จาก Rogue AP กับ RRM ที่ถูก Override

### 658.1 Background

[Part 36 Step 353](part-036-wireless-architecture-deep-dive.md) และ [Step 358]
(part-036-wireless-architecture-deep-dive.md) ตั้งค่า **RRM (DCA/TPC)** ให้ทำงานอัตโนมัติเพื่อ
เลือก Channel/Power ที่ดีที่สุดให้ AP-1/AP-2 เสมอ — เมื่อหลายเดือนก่อน มี Ticket Support แจ้งว่า
Video Conference บนชั้นที่ AP-1 ให้บริการมีอาการหลุดเป็นระยะ วิศวกรที่ตอบ Ticket ตอนนั้น (ที่ลาออก
ไปแล้ว) แก้ปัญหาเฉพาะหน้าด้วยการ **Fix Channel ของ AP-1 แบบ Static** (`ap dot11 5ghz channel 36`)
เพื่อ "ล็อกไม่ให้ RRM เปลี่ยน Channel กลางอากาศตอนประชุม" — แต่ไม่ได้เอากลับมาเปิด Auto RRM คืน
หลังจากนั้น

### 658.2 Symptom

พนักงานทั้งชั้นแจ้งว่า Wi-Fi "ช้าทั้งชั้น" ในช่วงเวลาที่คนใช้งานหนาแน่น — ทดสอบด้วย Client
Wireless Analyzer พบว่า RSSI ที่วัดได้ยังดี (`-52 dBm`) แต่ Throughput ต่ำผิดปกติมาก

```
! ตรวจสอบ Channel Assignment ปัจจุบันของ AP-1
WLC-1# show ap dot11 5ghz summary
AP Name    Channel   Channel Width   Tx Power Level   Channel Assignment
----------------------------------------------------------------------------
AP-1       36        20 MHz          3                Custom             <- ผิดปกติ! ควรเป็น Auto
AP-2       149       20 MHz          2                Auto
```

```
! ตรวจสอบว่ามี AP อื่นแย่ง Channel เดียวกันอยู่ใกล้ๆ หรือไม่
WLC-1# show wireless wps rogue ap summary
Rogue AP Summary
=================
MAC Address       Class            State           Channel   RSSI    Last Seen
-----------------------------------------------------------------------------------
7788.99aa.cc10     Unclassified     Alert           36        -48     2026-09-26 13:40:02   <- ชนกันเป๊ะ!
```

Rogue AP ตัวนี้ (คาดว่าเป็น Access Point ส่วนตัวที่พนักงานแอบเอามาต่อเพิ่มสัญญาณให้แรงขึ้นในพื้นที่
ตัวเอง) กำลัง Broadcast อยู่ที่ **Channel 36 เดียวกันกับ AP-1 เป๊ะ** และ RSSI ที่ WLC-1 ตรวจจับได้
(-48 dBm) แรงกว่า AP-1 เองที่ Client บางจุดรับได้ (-52 dBm) ด้วยซ้ำ — เกิด **Co-Channel
Interference (CCI)** เต็มรูปแบบตามที่ [Part 36 Step 353](part-036-wireless-architecture-deep-dive.md)
อธิบายไว้ (RSSI ดีไม่พอ ถ้า Noise/Interference สูง SNR จะตกและ Throughput ก็ตกตามไปด้วย)

### 658.3 Root Cause

**AP-1 ถูก Fix Channel แบบ Static (`Channel Assignment: Custom`) มาตั้งแต่การแก้ปัญหาเฉพาะหน้า
เมื่อหลายเดือนก่อน** — ทำให้ RRM/DCA **ไม่สามารถขยับ AP-1 หนี Rogue AP ตัวใหม่ที่มาแย่ง Channel
36 ทีหลังได้เลย** (ถ้า AP-1 ยังเป็น Auto ตามปกติ DCA รอบถัดไปจะตรวจพบ Interference สูงที่ Channel
36 แล้วสั่งย้าย AP-1 ไป Channel อื่นที่ปลอดจาก CCI เองโดยอัตโนมัติ) — นี่คือผลของการ "แก้เฉพาะหน้า
แล้วไม่ได้ Revert" แบบเดียวกับที่เห็นใน Step 652 — Static Override ที่ตั้งใจแก้ปัญหาหนึ่งใน
อดีต กลายเป็นสาเหตุของปัญหาใหม่ที่ไม่เกี่ยวข้องกันในภายหลัง

### 658.4 Fix & Verify

```
WLC-1# configure terminal
WLC-1(config)# ap name AP-1 dot11 5ghz channel 0     ! 0 = คืนกลับเป็น Auto (RRM ควบคุมเอง)
WLC-1(config)# end

! บังคับให้ DCA รันรอบใหม่ทันที ไม่ต้องรอ Interval ปกติ (Default 1 ชั่วโมงตาม Step 353)
WLC-1# ap dot11 5ghz rrm channel dca invoke
```

```
! ตรวจสอบผลลัพธ์หลัง DCA รันใหม่
WLC-1# show ap dot11 5ghz summary
AP Name    Channel   Channel Width   Tx Power Level   Channel Assignment
----------------------------------------------------------------------------
AP-1       44        20 MHz          2                Auto                <- ย้ายหนี Channel 36 แล้ว
AP-2       149       20 MHz          2                Auto

! ทดสอบ Throughput ซ้ำในพื้นที่เดิม — กลับมาปกติ
```

**สำหรับ Rogue AP ที่ตรวจพบ**: เนื่องจากยัง Classify เป็น `Unclassified` (ยังไม่ยืนยันว่าเชื่อมกับ
Wired Network ขององค์กรหรือไม่) ต้องส่งทีม Security ไปสืบหาตำแหน่งจริงก่อน (Physical Investigation
หรือ RF Direction Finding) แล้วยืนยัน Classification ให้ถูกต้องตาม [Part 37 Step 368]
(part-037-wireless-security-advanced.md) ก่อนจะพิจารณา Containment — **ไม่ควร Contain ทันทีที่
เจอ** เพราะยังไม่ยืนยันว่าเป็น Malicious จริง

> **บทเรียนสำหรับ ENARSI**: RF Symptom ที่ดูเหมือน "Coverage ไม่พอ" (RSSI ดี แต่ช้า) มักไม่ใช่
> ปัญหา Coverage เลย แต่เป็น **Interference/SNR** — คำสั่งแรกที่ควรเช็คเสมอคือ
> `show ap dot11 5ghz summary` ดูคอลัมน์ **Channel Assignment**: ถ้าเจอ `Custom` ที่ไม่ได้ตั้งใจ
> ไว้ (ไม่มีเหตุผลทางธุรกิจชัดเจนรองรับ) ให้สงสัยว่าเป็น Static Override ที่ค้างมาจากการแก้ปัญหา
> เก่าที่ไม่มีการ Revert เสมอ

---

## Step 659 — สถานการณ์ 8: Guest CWA Portal ค้างหน้าขาว — Pre-Auth ACL บล็อก DNS/Portal โดยไม่ตั้งใจ

### 659.1 Background

[Part 37 Step 367](part-037-wireless-security-advanced.md) ตั้งค่า SSID **GUEST-WIFI** ด้วย
**Central Web Authentication (CWA)** — Guest ต้อง Associate แบบ Open ก่อน แล้วถูก Redirect ไปหน้า
Portal ผ่าน **Pre-Auth ACL** (ACL ที่อนุญาตเฉพาะ DNS + HTTP/HTTPS ไปยัง Portal เท่านั้น ปิดกั้น
ทุกอย่างอื่นจนกว่าจะ Login สำเร็จ) — ทีม Security ทำ Hardening Pass เพิ่มความปลอดภัยให้ Guest
Network โดยเพิ่ม Deny Statement บล็อก Protocol ที่ไม่จำเป็นออกจาก ACL เดิม แต่แทรก ACE ใหม่โดย
**ไม่ระบุ Sequence Number** ทำให้ ACE ใหม่ไปแทรกอยู่ที่ตำแหน่งบนสุดของ ACL (ก่อนกฎที่จำเป็นสำหรับ
CWA เอง)

### 659.2 Symptom

Guest แจ้งว่า Associate SSID GUEST-WIFI ได้ปกติ (ไม่มี Password) ได้ IP Address ปกติ แต่เปิด
Browser แล้ว **ค้างที่หน้าขาว โหลดไม่จบ ไม่ Redirect ไปหน้า Portal เลย**

```
WLC-1# show wireless client mac-address 1122.33aa.bb01 detail | include Policy Manager State|VLAN|IPv4
Client IPv4 Address                         : 10.10.50.87
VLAN                                        : 50
Policy Manager State                        : WEBAUTH_REQD          <- ค้างอยู่ตรงนี้ ไม่ไป RUN เลย
```

`WEBAUTH_REQD` แปลว่า Client Associate และได้ IP แล้ว (Layer 1-4 ผ่านหมด) แต่ยังไม่ผ่าน Web
Authentication — ตาม Decision Tree ข้อ 5 (RADIUS/Policy) ต้องตรวจ ACL/Redirect ต่อ

### 659.3 Diagnosis & Root Cause

```
! ตรวจสอบ ACL ที่ Apply กับ Guest Pre-Auth
WLC-1# show ip access-lists GUEST-PREAUTH-ACL
Extended IP access list GUEST-PREAUTH-ACL
    10 deny udp any any eq 53                          <- ACE ใหม่ที่แทรกเข้ามาโดยไม่ตั้งใจ!
    20 permit udp any any eq 53                         (Sequence เดิมที่อนุญาต DNS สำหรับ CWA)
    30 permit tcp any host 10.10.99.70 eq 443            (Portal Server)
    40 deny ip any any
```

**เจอต้นเหตุทันที**: ACE ที่ Sequence `10` (`deny udp any any eq 53`) ถูกแทรกเข้ามาอยู่ **เหนือ**
ACE ที่ Sequence `20` (`permit udp any any eq 53`) ซึ่งเป็นกฎที่จำเป็นสำหรับให้ Guest Resolve DNS
เพื่อไปหา Portal ได้ — เพราะ Router/WLC ประมวลผล ACL **จากบนลงล่างและหยุดที่ Match แรก** ACE ที่
Sequence 10 จึง Match และ **Deny DNS ทุก Query ก่อนที่จะถึงกฎ Permit ที่ Sequence 20 เลย** — ผลคือ
Guest Client Resolve Domain Name อะไรไม่ได้เลย (แม้แต่ Domain ของ Portal เอง) ทำให้ Browser ค้าง
ที่หน้าขาวรอ DNS Timeout ไปเรื่อยๆ โดยไม่มีทาง Redirect ไปหา Portal ได้

ทีม Security ที่แก้ ACL ตั้งใจจะ Deny DNS เฉพาะที่ไม่ได้ชี้ไปยัง DNS Server ที่กำหนด (Data
Exfiltration ผ่าน DNS Tunneling Prevention) แต่ลืมระบุ Sequence Number ให้แทรกอยู่ **หลัง** กฎ
Permit ที่จำเป็นสำหรับ CWA และลืมระบุ `host` ให้เจาะจงเฉพาะปลายทางที่ไม่ใช่ DNS Server ที่ถูกต้อง —
กลายเป็น Deny แบบ Blanket ที่บล็อกทุกอย่าง

### 659.4 Fix & Verify

```
WLC-1# configure terminal
WLC-1(config)# ip access-list extended GUEST-PREAUTH-ACL
WLC-1(config-ext-nacl)# no 10                                    ! ลบ ACE ที่ผิดตำแหน่งออกก่อน
! แก้ไขให้ถูกต้อง: Deny DNS ไปยังปลายทางที่ไม่ใช่ DNS Server ที่กำหนดเท่านั้น แทรกไว้หลัง Permit ที่จำเป็น
WLC-1(config-ext-nacl)# 25 deny udp any any neq 53
WLC-1(config-ext-nacl)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

```
! ตรวจสอบ ACL ที่ถูกต้องหลังแก้ไข — Permit DNS/Portal (Sequence 20-30) ต้องมาก่อน Deny ใหม่เสมอ
WLC-1# show ip access-lists GUEST-PREAUTH-ACL
Extended IP access list GUEST-PREAUTH-ACL
    20 permit udp any any eq 53
    25 deny udp any any neq 53
    30 permit tcp any host 10.10.99.70 eq 443
    40 deny ip any any
```

```
! ตรวจสอบผลลัพธ์ฝั่ง Client — Guest Login ผ่าน Portal สำเร็จ
WLC-1# show wireless client mac-address 1122.33aa.bb01 detail | include Policy Manager State
Policy Manager State                        : RUN
```

> **บทเรียนสำหรับ ENARSI**: ทุกครั้งที่แก้ไข ACL ที่มีอยู่แล้วโดยเฉพาะ ACL ที่เกี่ยวกับ Redirect/
> Pre-Auth Flow **ต้องระบุ Sequence Number เสมอ** และตรวจสอบลำดับหลังแก้ไขทุกครั้งด้วย `show ip
> access-lists <name>` — การ Insert ACE โดยไม่ระบุ Sequence จะไปต่อท้าย ACL เสมอ **ยกเว้น**บาง
> Platform/Mode ที่ Auto-renumber อาจทำให้ตำแหน่งไม่เป็นไปตามที่คาด (ตามที่ Part 14 เตือนไว้เรื่อง
> ACL Editing) — `Policy Manager State : WEBAUTH_REQD` ที่ค้างอยู่นานคือสัญญาณบอกตรงว่าปัญหาอยู่ที่
> ACL/Redirect ไม่ใช่ RF หรือ DHCP

---

## Step 660 — Lab เต็มรูปแบบ: Help Desk Surge วันเดียวเจอทั้ง VLAN Mapping และ 802.11r พังพร้อมกัน

### 660.1 สถานการณ์เปิดเรื่อง

เช้าวันจันทร์หลัง Maintenance Window ของสุดสัปดาห์ (ที่มีทั้งโครงการ "VLAN Cleanup" และ "Config
Audit" รันพร้อมกันในคืนเดียวโดยทีมที่ต่างกัน) Help Desk เจอ Ticket จำนวนมากพุ่งขึ้นพร้อมกัน 2
ลักษณะที่แตกต่างกันชัดเจน:

| กลุ่ม Ticket | อาการ | จำนวน |
|---|---|---|
| กลุ่ม A | "เชื่อม Wi-Fi ENTERPRISE-WIFI ได้ แต่ Internet ใช้ไม่ได้" | ~40 Ticket |
| กลุ่ม B | "VoWLAN/Softphone เสียงกระตุกตอนเดินระหว่างห้อง" | ~8 Ticket |

วิศวกรที่รับเรื่องต้อง **วินิจฉัยแบบเป็นระบบ** ตาม Decision Tree ของ Step 651 เพื่อแยกว่าทั้งสอง
กลุ่ม Ticket เป็นปัญหาเดียวกันหรือคนละปัญหา — และนี่คือ Lab เต็มรูปแบบที่รวม **สถานการณ์ 2 (Step
653 — VLAN Mapping)** และ **สถานการณ์ 5 (Step 656 — 802.11r ถูกลบ)** ที่เกิดขึ้น**พร้อมกันจริง**
ในคืนเดียว

### 660.2 ขั้นตอนวินิจฉัยแบบเป็นระบบ — Before Snapshot

**ขั้นที่ 1: ยืนยันว่า AP ทั้งคู่ Registered ปกติก่อน (Layer 0-1)**

```
WLC-1# show ap summary
AP Name    IP Address     State
------------------------------------
AP-1       10.10.99.101   Registered
AP-2       10.10.99.102   Registered
```

ผ่าน — ไม่ใช่ปัญหา CAPWAP/Discovery แบบ Step 652 ตัดออกไปได้เลย

**ขั้นที่ 2: สุ่ม Client จากกลุ่ม Ticket A ตรวจ Layer 3-4**

```
WLC-1# show wireless client mac-address aabb.ccdd.5501 detail | include Security Policy|VLAN|IPv4|Policy Manager State
Security Policy Completed                   : Yes
Client IPv4 Address                         : Unknown
VLAN                                        : 45                          <- ผิด! เหมือน Step 653 เป๊ะ
Policy Manager State                        : IP LEARN IN PROGRESS
```

**ขั้นที่ 3: สุ่ม Client จากกลุ่ม Ticket B ตรวจ Roaming (Layer 3, ส่วน FT)**

```
WLC-1# show wlan name ENTERPRISE-WIFI-PROFILE | include 802.11r|Fast Transition
  802.11r Fast Transition                   : Disabled                    <- เหมือน Step 656 เป๊ะ
```

**สรุปก่อนแก้ไข**: ทั้งสองกลุ่ม Ticket มาจาก **คนละ Root Cause กันโดยสิ้นเชิง** ที่บังเอิญเกิดใน
คืนเดียวกัน — กลุ่ม A คือ `ENTERPRISE-WIFI-POLICY` ถูกตั้ง VLAN ผิดเป็น 45 (จาก VLAN Cleanup
Script), กลุ่ม B คือ `ENTERPRISE-WIFI-PROFILE` ถูกลบ `security ft` ออก (จาก Config Audit
Script) — **นี่คือเหตุผลที่ต้องไล่ตาม Decision Tree ทุกครั้ง แทนที่จะสรุปเหมาเข่งว่าเป็นปัญหาเดียว**
เพราะทั้งสองอาการดูเผินๆ อาจถูกเข้าใจผิดว่าเป็น "SSID เดียวกันมีปัญหา" ทั้งที่จริงคือสองจุดที่ต่างกัน
บน WLAN/Policy Profile เดียวกัน

### 660.3 แก้ไขทั้ง 2 จุดพร้อมกันในรอบ Maintenance เดียว

```
WLC-1# configure terminal

! === แก้ปัญหากลุ่ม A: VLAN Mapping (เหมือน Step 653) ===
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY
WLC-1(config-wireless-policy)# vlan 40
WLC-1(config-wireless-policy)# exit

! === แก้ปัญหากลุ่ม B: 802.11r FT (เหมือน Step 656) ===
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# security ft over-the-ds
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit

WLC-1(config)# end
WLC-1# copy running-config startup-config

! บังคับให้ Client ที่ค้าง VLAN ผิด Reassociate ใหม่ทั้งหมด (Client ที่ Roam ปกติจะรับ FT ใหม่เองอัตโนมัติ)
WLC-1# wireless client mac-address aabb.ccdd.5501 delete
```

### 660.4 Verification — After Snapshot เทียบกับ Before

**(A) VLAN Mapping — กลุ่ม Ticket A**

```
WLC-1# show wireless profile policy detailed ENTERPRISE-WIFI-POLICY | include VLAN
  VLAN                                     : 40                          <- ถูกต้องแล้ว

WLC-1# show wireless client mac-address aabb.ccdd.5501 detail | include VLAN|IPv4|Policy Manager State
Client IPv4 Address                         : 10.10.40.144
VLAN                                        : 40
Policy Manager State                        : RUN
```

**(B) 802.11r Fast Transition — กลุ่ม Ticket B**

```
WLC-1# show wlan name ENTERPRISE-WIFI-PROFILE | include 802.11r|Fast Transition|Over the DS
  802.11r Fast Transition                   : Enabled
  Over the DS                               : Enabled

! ทดสอบ Roaming ซ้ำกับ Client VoWLAN ตัวอย่าง
WLC-1# debug client mac-address aabb.ccdd.7744
*Sep 26 14:02:10.010: [client-orch-state] aabb.ccdd.7744 Roaming detected: AP-2 -> AP-1
*Sep 26 14:02:10.023: [dot1x] aabb.ccdd.7744 FT Key derivation from cache — skip full 4-Way Handshake
*Sep 26 14:02:10.031: [client-orch-state] aabb.ccdd.7744 Moving to RUN state

Roaming Time ที่วัดได้: 12ms      <- กลับมาปกติ, ตรงกับที่ Step 656 ยืนยันไว้
```

**สรุปตาราง Before/After ทั้งระบบ**

| จุดตรวจ | ก่อนแก้ | หลังแก้ |
|---|---|---|
| `ENTERPRISE-WIFI-POLICY` VLAN | 45 (ผิด) | 40 (ถูกต้อง) |
| Client IPv4 Address | Unknown | 10.10.40.144 |
| Policy Manager State | IP LEARN IN PROGRESS | RUN |
| WLAN 802.11r Fast Transition | Disabled | Enabled |
| Roaming Time (VoWLAN Test) | 187ms | 12ms |
| Ticket กลุ่ม A/B ที่เหลือ | ~48 Ticket | 0 Ticket (ปิดหมดหลัง Verify) |

> **บทเรียนสำคัญที่สุดของ Part นี้**: เมื่อเจอ Ticket จำนวนมากพร้อมกันหลัง Maintenance Window ที่มี
> หลายทีมทำงานคาบเกี่ยวกัน **ห้ามสรุปว่าเป็นปัญหาเดียวโดยไม่ตรวจสอบ** — ต้องไล่ตาม Decision Tree
> (Step 651) ทีละกลุ่ม Ticket แยกกัน เพราะ Config เปลี่ยนพร้อมกันหลายจุดในคืนเดียวมักหมายความว่ามี
> Root Cause มากกว่า 1 จุดจริง — การแก้ถูกทั้ง 2 จุดพร้อมกันในรอบ Maintenance เดียวช่วยลด Downtime
> รวมของ Help Desk ได้มากกว่าการแก้ทีละจุดแยก Window

---

## แบบฝึกหัดทวนความเข้าใจ Part 66

1. ตาม Wireless Troubleshooting Decision Tree ของ Step 651 — ถ้า `show ap summary` ไม่แสดง AP
   ตัวหนึ่งเลย ควรตรวจสอบ Layer ไหนก่อน และเพราะเหตุใดจึง **ไม่ควร** เริ่มตรวจ RADIUS หรือ Security
   Policy ก่อน?
2. ในสถานการณ์ Step 653 เหตุใด Output `Security Policy Completed : Yes` ร่วมกับ
   `Client IPv4 Address : Unknown` จึงเป็นหลักฐานที่ชี้ตรงไปที่ปัญหา VLAN Mapping ไม่ใช่ปัญหา RF
   หรือ Security?
3. อธิบายความแตกต่างระหว่าง `central switching`, `central authentication`, และ `central dhcp`
   บน FlexConnect Policy Profile — เพราะเหตุใด Step 655 จึงเน้นว่าการเปิด Local Switching อย่าง
   เดียวไม่เพียงพอสำหรับ Resilience เต็มรูปแบบตอน WAN ล่ม?
4. เพราะเหตุใด AP SSO (Step 657) จึงกำหนดว่า WLC ทั้งคู่ใน HA Pair ต้องมี Software Version
   เดียวกันเป๊ะ และถ้า Version ไม่ตรงกัน ผลลัพธ์ตอน Failover จะต่างจาก AP SSO ปกติอย่างไร?
5. ใน Step 658 เพราะเหตุใดการ Fix Channel แบบ Static (`Channel Assignment: Custom`) ที่ตั้งไว้เพื่อ
   แก้ปัญหาหนึ่งในอดีต จึงกลายเป็นสาเหตุของปัญหา Co-Channel Interference ใหม่ในภายหลังได้ และวิธี
   แก้ที่ถูกต้องคืออะไร?

**เฉลย:**

1. ควรตรวจสอบ **Layer 0-1 (Physical/CAPWAP Join)** ก่อนเสมอ ผ่านคำสั่งเช่น `show ap summary`,
   `show ip dhcp binding` (ที่ Access/Distribution Switch), และตรวจ DHCP Option 43 — เพราะถ้า AP
   ยังไม่ Join WLC เลย Client จะไม่มีทางเห็น SSID หรือ Authenticate ได้ตั้งแต่แรก การไปตรวจ RADIUS
   หรือ Security Policy ในจุดที่ AP ยังไม่ Join คือการเสียเวลาโดยไล่ผิด Layer ทั้งหมด
2. `Security Policy Completed : Yes` ยืนยันว่า Client ผ่าน 802.11 Association/Authentication
   (WPA2-PSK หรือ 802.1X) สำเร็จแล้ว 100% — ตัด Layer 2-3 (RF/Security) ออกจากสมการได้เลย ส่วน
   `Client IPv4 Address : Unknown` ชี้ตรงไปที่ Layer 4 (DHCP/VLAN) — เมื่อสอง Field นี้ปรากฏพร้อมกัน
   จึงชี้เจาะจงไปที่ VLAN Mapping ผิดหรือ DHCP Pool/Relay มีปัญหา ไม่ใช่ RF หรือ Security
3. `central switching` ควบคุม Data Plane ของ Client Traffic (Local ที่ AP หรือ Central ที่ WLC),
   `central authentication` ควบคุมว่า WLC ตัดสินใจ Authentication เอง (ต้องมี CAPWAP Control
   Tunnel), และ `central dhcp` ควบคุมว่า DHCP Request ต้อง Relay ผ่าน WLC หรือ Local ที่ AP —
   ทั้ง 3 ค่าเป็น Setting แยกอิสระกัน การเปิด Local Switching อย่างเดียวโดยยังเหลือ Central DHCP
   ไว้ทำให้ Data Path Local จริง แต่ Client ที่ต้อง Renew DHCP Lease ระหว่าง WAN ล่มจะยังหลุด IP
   เพราะ DHCP ยังต้องพึ่ง CAPWAP Tunnel ไปหา WLC อยู่ ต้องปิด Central ทั้ง 3 จุดที่เกี่ยวข้องให้
   สอดคล้องกับ Resilience Requirement จริง
4. เพราะ AP SSO ต้อง Sync State (Client Database, AP Database, CAPWAP Session) แบบ Real-time
   ระหว่าง Active/Standby ซึ่งอาศัย Internal Data Structure ที่ผูกกับ Software Version — ถ้า
   Version ไม่ตรงกัน HA Pair จะไม่สามารถทำ Stateful Sync ได้จริง (Mode กลายเป็น Simplex ไม่ใช่
   Duplex) ทำให้ตอน Failover พฤติกรรมจริงกลายเป็นแบบ N+1 HA (AP ต้อง Discovery + Join ใหม่
   ทั้งหมด ใช้เวลาหลักนาที) ไม่ใช่ AP SSO ที่ควรมี Downtime ระดับ Sub-second
5. เพราะ RRM/DCA เป็นระบบที่ปรับ Channel แบบ**อัตโนมัติต่อเนื่อง**ตามสภาพ RF ที่เปลี่ยนไปเรื่อยๆ —
   การ Fix Channel แบบ Static คือการ "หยุด" ไม่ให้ AP ตัวนั้นปรับตัวตามสภาพแวดล้อมที่เปลี่ยนไปอีก
   เลย เมื่อมี Rogue AP ใหม่มาแย่ง Channel เดียวกันในภายหลัง AP ที่ Fix Channel ไว้จะไม่สามารถขยับ
   หนีได้เอง ต่างจาก AP ที่เป็น Auto ซึ่ง DCA จะตรวจพบ Interference แล้วย้าย Channel ให้อัตโนมัติ
   — วิธีแก้คือคืนค่า Channel Assignment กลับเป็น Auto (`channel 0`) แล้วบังคับรัน DCA รอบใหม่
   (`ap dot11 5ghz rrm channel dca invoke`)

---

## สรุป Part 66

Part นี้นำ Wireless LAN ที่วางพื้นฐานไว้ตั้งแต่ [Part 20](part-020-wireless-fundamentals.md),
ลงลึกสถาปัตยกรรมใน [Part 36](part-036-wireless-architecture-deep-dive.md), และยกระดับความปลอดภัย
ใน [Part 37](part-037-wireless-security-advanced.md) กลับมาฝึก **Troubleshooting เชิงระบบ** ตาม
แนวทาง ENARSI ผ่าน **Layered Methodology** (CAPWAP Join → RF/Coverage → 802.11 Association →
DHCP/VLAN → RADIUS/Policy → Traffic Flow) และ 8 สถานการณ์จริงที่ครอบคลุมทุก Layer: **DHCP Option
43 ค้างจากการสลับ Loaner Controller** (Step 652), **VLAN Mapping ผิดที่ Policy Profile** (Step
653), **Certificate หมดอายุเป็น Batch บน EAP-TLS** (Step 654), **Central DHCP ที่บั่นทอน
FlexConnect Resilience** (Step 655), **802.11r FT ถูกลบระหว่าง Config Audit** (Step 656),
**Redundancy Port/Version Mismatch ที่ทำลาย AP SSO** (Step 657), **Co-Channel Interference จาก
Static Channel Override** (Step 658), และ **Pre-Auth ACL Sequence ผิดที่บล็อก Guest CWA** (Step
659) ปิดท้ายด้วย Lab เต็มรูปแบบที่รวม 2 สถานการณ์เกิดพร้อมกันจริง (Step 660) เพื่อฝึกการแยก Root
Cause หลายจุดออกจากกันอย่างเป็นระบบ — บทเรียนที่ย้ำซ้ำหลายครั้งใน Part นี้คือ **การแก้ปัญหาเฉพาะ
หน้าที่ไม่ได้ Revert กลับ** (Loaner IP, Static Channel) และ **Config Script ที่แก้เกินขอบเขตที่
ตั้งใจ** (VLAN Cleanup, Config Audit) คือสาเหตุของ Incident ส่วนใหญ่ในโลกจริง มากกว่า Hardware
หรือ Protocol ที่พังเอง ✅

**พร้อมสำหรับ Part 67**: เราจะปิดท้ายสาย Troubleshooting เชิงเทคโนโลยีเฉพาะทางด้วย
**Automation Troubleshooting** — เมื่อ Network สมัยใหม่พึ่งพา NETCONF/RESTCONF/YANG (ทวนจาก
[Part 49](part-049-netconf-restconf-yang-deep-dive.md)) และ Ansible/Python Automation (ทวนจาก
[Part 47](part-047-automation-python-advanced.md)/[Part 48](part-048-automation-ansible-advanced.md))
มากขึ้นเรื่อยๆ **Automation เองก็กลายเป็นแหล่งที่มาของ Incident ได้เช่นกัน** — ทั้ง Script ที่ Push
Config ผิดพลาดโดยไม่มี Guard Rail, Idempotency ที่ทำงานผิดคาด, และ API Rate Limit ที่ทำให้ระบบ
Monitoring ล่มเป็นลูกโซ่ — เนื้อหาที่ต่อเนื่องโดยตรงจากบทเรียนเรื่อง "Script แก้เกินขอบเขต" ที่เห็น
ซ้ำแล้วซ้ำอีกใน Part นี้

**ไปต่อ:** [Part 67 — Automation Troubleshooting →](part-067-automation-troubleshooting.md)
