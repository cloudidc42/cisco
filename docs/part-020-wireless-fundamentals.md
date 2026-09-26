# Part 20 — Wireless LAN Fundamentals (WLC/AP)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 191–200 จาก 1000**

> ต่อจาก [Part 19 — WAN Technologies →](part-019-wan-technologies.md) ที่ปิดท้ายเรื่อง Wired
> Enterprise Network (WAN Edge, MPLS/Internet, Routing) ในที่สุด Part 20 นี้จะพาเราออกจากโลกของ
> สายทองแดง/ไฟเบอร์ เข้าสู่ **Wireless LAN (WLAN)** — เทคโนโลยีที่ผู้ใช้ปลายทางสัมผัสมากที่สุดใน
> Enterprise Network ยุคปัจจุบัน เราจะเพิ่ม **WLC-1 (Catalyst 9800-CL)** และ **AP-1/AP-2
> (Catalyst 9130)** เข้าไปใน Lab Topology หลักของหลักสูตร (ตาม `docs/00-ip-address-plan.md`)
> และคอนฟิกให้ Client เชื่อมต่อ Wi-Fi เข้า **VLAN 40 (WIFI, 10.10.40.0/24)** ได้จริง
>
> **หมายเหตุ**: Part นี้ปูพื้นฐาน Wireless ให้ครบสำหรับ CCNA เท่านั้น ส่วน 802.1X/WPA3-Enterprise
> แบบละเอียด, RADIUS/ISE integration, Advanced RF Design, และ High Availability ของ WLC
> จะกลับมาเรียนเชิงลึกใน **Part 37 (Wireless Security)** และ **Part 44 (Advanced Wireless/ISE)**

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 191 | Wireless พื้นฐาน: 802.11 Standards, ความถี่, Channel |
| 192 | Autonomous AP vs Lightweight AP (Split-MAC) และ CAPWAP |
| 193 | WLC Deployment Modes: Appliance vs 9800-CL, Centralized vs FlexConnect |
| 194 | CAPWAP Discovery & Join Process |
| 195 | คอนฟิก WLC พื้นฐาน (Catalyst 9800): WLAN Profile, Policy/Site/RF Tag |
| 196 | ลงทะเบียน AP และกำหนด Tag ให้ AP-1/AP-2 |
| 197 | Wireless Security พื้นฐาน: WPA2-Personal vs Enterprise |
| 198 | แนวคิด Client Roaming: L2 Roaming, L3 Roaming, Mobility Group |
| 199 | Verification & Troubleshooting WLC/AP |
| 200 | Lab เต็มรูปแบบ: ตั้งค่า WLC-1, ลงทะเบียน AP-1/AP-2, ทดสอบ Client บน VLAN 40 |

---

## Step 191 — Wireless พื้นฐาน: 802.11 Standards, ความถี่, Channel

### ทำไมต้องเรียน Wireless ทั้งที่เรียน Wired มา 19 Part แล้ว?

Enterprise Network สมัยใหม่แทบทุกที่เป็น **Wireless-first**: พนักงานใช้ Laptop/มือถือเชื่อมต่อ
Wi-Fi เป็นหลัก สาย Ethernet เหลือไว้สำหรับ Server, Printer, Desktop บางจุด, และ AP เอง
ตาม Lab Topology ของหลักสูตรนี้ ACCESS-SW3/ACCESS-SW4 มี VLAN 40 (WIFI) รองรับอยู่แล้วตั้งแต่
Part 1 — Part นี้คือ Part ที่เราจะ "เติม" AP และ WLC เข้าไปใช้งาน VLAN นั้นจริง

### มาตรฐาน IEEE 802.11 และการเทียบกับชื่อทางการตลาด (Wi-Fi Alliance)

Wi-Fi Alliance เปลี่ยนมาใช้ชื่อ "Wi-Fi generation" ตั้งแต่ปี 2018 เพื่อให้ผู้ใช้ทั่วไปเข้าใจง่ายขึ้น
แทนชื่อมาตรฐาน IEEE ที่จำยาก

| มาตรฐาน IEEE | ชื่อ Wi-Fi (Marketing) | ปีที่ออก | ความถี่ | Max Theoretical Speed | Channel Width สูงสุด |
|---|---|---|---|---|---|
| 802.11a | (ไม่มีชื่อย้อนหลัง) | 1999 | 5 GHz | 54 Mbps | 20 MHz |
| 802.11b | (ไม่มีชื่อย้อนหลัง) | 1999 | 2.4 GHz | 11 Mbps | 20 MHz |
| 802.11g | (ไม่มีชื่อย้อนหลัง) | 2003 | 2.4 GHz | 54 Mbps | 20 MHz |
| 802.11n | **Wi-Fi 4** | 2009 | 2.4 GHz + 5 GHz | 600 Mbps (MIMO) | 40 MHz |
| 802.11ac | **Wi-Fi 5** | 2013 | 5 GHz | 6.9 Gbps (MU-MIMO) | 80/160 MHz |
| 802.11ax | **Wi-Fi 6** | 2019 | 2.4 GHz + 5 GHz | 9.6 Gbps | 20/40/80/160 MHz |
| 802.11ax (6 GHz) | **Wi-Fi 6E** | 2021 | 6 GHz (ใหม่) | 9.6 Gbps | 20/40/80/160 MHz |
| 802.11be | **Wi-Fi 7** | 2024 | 2.4/5/6 GHz | 46 Gbps (ทฤษฎี) | 320 MHz |

> **Catalyst 9130** ที่ใช้ใน Lab นี้เป็น AP รุ่น **Wi-Fi 6 (802.11ax)** รองรับทั้ง 2.4 GHz และ 5 GHz
> (มีรุ่น 9130AXI/9130AXE) — เพียงพอสำหรับ Enterprise ทั่วไปในปี 2026 โดยไม่ต้องใช้ Wi-Fi 6E/7

### ความถี่ 2.4 GHz vs 5 GHz vs 6 GHz — ต้องเข้าใจ Trade-off

| คุณสมบัติ | 2.4 GHz | 5 GHz | 6 GHz |
|---|---|---|---|
| ระยะทำการ (Range) | ไกลกว่า ทะลุกำแพงดีกว่า | สั้นกว่า | สั้นที่สุด |
| ความเร็ว (Throughput) | ต่ำกว่า | สูงกว่า | สูงสุด |
| จำนวน Channel ที่ไม่ทับซ้อน | น้อย (3 channel) | มาก (20+ channel) | มากที่สุด (59 channel) |
| ความหนาแน่นของสัญญาณรบกวน | สูงมาก (Bluetooth, Microwave, IoT) | ต่ำกว่า | ต่ำที่สุด (ยังใหม่) |
| Backward Compatibility | รองรับ Client เก่าทุกรุ่น | Client ใหม่เท่านั้น (802.11a ขึ้นไป) | Wi-Fi 6E/7 เท่านั้น |

**หลักการออกแบบจริงใน Enterprise**: ใช้ 5 GHz เป็นหลักสำหรับ Data-heavy client (Laptop) และเปิด
2.4 GHz ไว้เพื่อ backward compatibility กับ IoT/Legacy device เท่านั้น — เทคนิคนี้เรียกว่า
**Band Steering** (บังคับ Client ที่รองรับทั้งสอง band ให้ไปใช้ 5 GHz ก่อนเสมอ)

### Channel และ Non-Overlapping Channel (2.4 GHz)

2.4 GHz มี Channel 1-11 (US)/1-13 (ส่วนใหญ่ของโลก) แต่ **Channel ที่ไม่ทับซ้อนกันจริง (Non-
Overlapping) มีเพียง 3 channel: 1, 6, 11** เพราะ Channel width 22 MHz แต่ channel spacing
เพียง 5 MHz ทำให้ channel ที่ห่างกันน้อยกว่า 5 ช่องจะทับซ้อนกัน (Adjacent Channel Interference)

```
2.4 GHz Spectrum (แสดง Channel 1, 6, 11 ที่ไม่ทับซ้อนกัน)

Ch1  Ch2  Ch3  Ch4  Ch5  Ch6  Ch7  Ch8  Ch9  Ch10 Ch11
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
█████████████                                          <- Channel 1 (2412 MHz ± 11MHz)
     █████████████                                     <- Channel 2 (ทับกับ 1 และ 3)
          █████████████                                <- Channel 3 (ทับกับ 1,2,4,5)
                    █████████████                      <- Channel 6 (ไม่ทับกับ 1)
                                        █████████████   <- Channel 11 (ไม่ทับกับ 6)

สรุป: ใช้เฉพาะ Channel 1, 6, 11 ในการออกแบบ AP หลายตัวเพื่อลด Co-Channel Interference
```

5 GHz มี Non-Overlapping Channel มากกว่ามาก (channel width 20 MHz แต่ spacing 20 MHz พอดี)
ทำให้สามารถวาง Channel Plan ที่ไม่ทับซ้อนได้ถึง 20+ channel (ขึ้นกับประเทศและ DFS channel)
— รายละเอียดการออกแบบ RF/Channel Plan เต็มรูปแบบจะเรียนใน Part 44

### Channel Width — Trade-off ระหว่างความเร็วกับจำนวน Channel ที่ใช้ได้

| Channel Width | Throughput | จำนวน Non-Overlap Channel ที่เหลือ (5 GHz) | ใช้เมื่อไหร่ |
|---|---|---|---|
| 20 MHz | ต่ำสุด | มากที่สุด | Enterprise ความหนาแน่น AP สูง (Office จำนวนมาก) |
| 40 MHz | กลาง | น้อยลงครึ่งหนึ่ง | พื้นที่ AP ไม่หนาแน่น |
| 80 MHz | สูง | น้อยมาก | ห้องประชุม/พื้นที่เปิดกว้างต้องการ throughput สูง |
| 160 MHz | สูงสุด | เหลือน้อยที่สุด (มัก conflict กับ DFS) | Use case พิเศษ เช่น AR/VR เท่านั้น |

> **กฎทองของ Enterprise Wireless Design**: ยิ่ง Channel Width กว้าง ยิ่งเหลือ Channel ให้ AP
> ข้างเคียงใช้น้อย → เสี่ยง Co-Channel Interference (CCI) มากขึ้น ในพื้นที่ Office ทั่วไปที่มี AP
> จำนวนมาก (High-Density) แนะนำใช้ **20 MHz บน 5 GHz** เป็นค่าเริ่มต้น ไม่ใช่ 80/160 MHz

---

## Step 192 — Autonomous AP vs Lightweight AP (Split-MAC) และ CAPWAP

### Autonomous AP (AP แบบดั้งเดิม)

Autonomous AP คือ AP ที่ทำงาน**อิสระเต็มรูปแบบ** — มี Config ของตัวเอง, ประมวลผล 802.11 MAC
ทั้งหมดเอง (Beacon, Authentication, Encryption, Association), และต้อง config ทีละตัวผ่าน
CLI/Web GUI ของ AP เอง คล้ายกับ Router/Switch แต่ละตัวที่ทำงานแยกกัน

**ปัญหาของ Autonomous AP ในองค์กรขนาดใหญ่:**
- ต้อง config SSID/Security ทีละตัว — ถ้ามี 200 AP ต้อง login เข้า config ทีละตัว
- ไม่มี Central Visibility — ไม่รู้ว่า Client อยู่ AP ไหน, Roaming ระหว่าง AP ทำงานไม่ราบรื่น
- RF Management (Channel/Power) ต้องปรับมือทีละตัว ไม่มี Auto RF
- Firmware Upgrade ต้องทำทีละตัว

### Lightweight AP (LWAP) + WLC — สถาปัตยกรรมที่ Enterprise ใช้จริง

**Lightweight AP** ไม่มี "สมอง" ของตัวเอง — ถูกออกแบบมาให้พึ่งพา **WLC (Wireless LAN
Controller)** ที่ทำงานแบบรวมศูนย์ (Centralized) แนวคิดนี้เรียกว่า **Split-MAC Architecture**:
แยกการประมวลผล 802.11 MAC ออกเป็น 2 ส่วนระหว่าง AP กับ WLC

| หน้าที่ | ทำที่ AP (Real-time functions) | ทำที่ WLC (Non-real-time / Management functions) |
|---|---|---|
| Beacon Frame | ✅ AP ส่งเอง | - |
| 802.11 Encryption/Decryption (WPA2/3) | ✅ AP ทำเอง (Local MAC) หรือ WLC (Central MAC) ขึ้นกับ mode | ขึ้นกับ FlexConnect vs Local mode |
| Association/Authentication (802.11) | รับ Frame มาจาก Client | WLC ตัดสินใจ (Authenticate, RF policy) |
| SSID/Security Policy Config | - | ✅ ตั้งค่าที่ WLC ครั้งเดียว ใช้กับทุก AP |
| RF Management (Channel/Power - RRM) | รับคำสั่งจาก WLC | ✅ คำนวณและสั่งการ (Radio Resource Management) |
| Roaming Coordination | - | ✅ WLC ติดตาม Client ทุกตัวแบบ real-time |
| Client Traffic (Data Plane) | ✅ Encapsulate เป็น CAPWAP tunnel ส่งไป WLC | Decapsulate แล้วส่งเข้า VLAN ปลายทาง |

```
Split-MAC Architecture (Lightweight AP + WLC)

┌─────────────┐                              ┌──────────────────────┐
│   Client     │  802.11 Association/Data     │   Lightweight AP      │
│  (Laptop)    │ ────────────────────────────►│   (AP-1 / AP-2)        │
└─────────────┘                              │  - ส่ง Beacon          │
                                              │  - รับ 802.11 Frame     │
                                              │  - Encapsulate CAPWAP  │
                                              └───────────┬──────────┘
                                                          │ CAPWAP Tunnel
                                                          │ (Control + Data)
                                                          │ ผ่าน VLAN 99 (MGMT)
                                              ┌───────────▼──────────┐
                                              │        WLC-1           │
                                              │  (Catalyst 9800-CL)    │
                                              │  - Config ทุก AP รวมศูนย์│
                                              │  - Decapsulate CAPWAP  │
                                              │  - RRM, Roaming, Policy│
                                              └───────────┬──────────┘
                                                          │ ส่ง Client traffic
                                                          │ เข้า VLAN ปลายทาง
                                              ┌───────────▼──────────┐
                                              │  VLAN 40 (WIFI)        │
                                              │  10.10.40.0/24         │
                                              └────────────────────────┘
```

### CAPWAP (Control And Provisioning of Wireless Access Points) — RFC 5415

CAPWAP คือ Protocol ที่ AP ใช้สื่อสารกับ WLC (พัฒนามาจาก LWAPP รุ่นก่อนของ Cisco) แบ่งเป็น
**2 Tunnel แยกกัน** ระหว่าง AP กับ WLC:

| Tunnel | UDP Port | หน้าที่ | เข้ารหัสหรือไม่ |
|---|---|---|---|
| **CAPWAP Control** | UDP 5246 | ส่งคำสั่ง config, keepalive, join/registration | เข้ารหัสด้วย DTLS เสมอ (บังคับ) |
| **CAPWAP Data** | UDP 5247 | ส่ง Client traffic (802.11 frame ที่ encapsulate) | DTLS เป็น option (เปิดได้ที่ WLC) |

> **ข้อสอบ CCNA ชอบถาม**: CAPWAP ใช้ UDP port อะไร? — ตอบ **5246 (Control) และ 5247 (Data)**
> ต่างจาก LWAPP รุ่นเก่าที่ใช้ port 12222/12223

CAPWAP ทำให้ AP กลายเป็นเหมือน "เสาอากาศบวก NIC การ์ด" ที่ฉลาดพอจะ join WLC เองได้ แต่ Logic
การตัดสินใจทั้งหมดอยู่ที่ WLC — นี่คือเหตุผลที่ Enterprise ใช้โมเดลนี้แทน Autonomous AP

---

## Step 193 — WLC Deployment Modes: Appliance vs 9800-CL, Centralized vs FlexConnect

### WLC มีให้เลือกหลาย Form Factor

| ประเภท WLC | ตัวอย่างรุ่น | ใช้เมื่อไหร่ |
|---|---|---|
| Physical Appliance | Catalyst 9800-40, 9800-80, 9800-L | Data Center/Campus ขนาดใหญ่ ต้องการ Hardware ทุ่มเฉพาะ |
| Virtual (Private Cloud) | **Catalyst 9800-CL** (รันบน ESXi/KVM/CML) | องค์กรที่มี Private Cloud/Virtualization อยู่แล้ว — **ใช้ใน Lab นี้** |
| Embedded (บน Switch) | Catalyst 9800-Embedded Wireless Controller (บน 9300/9500) | สำนักงานสาขาเล็กที่ไม่อยากมีอุปกรณ์เพิ่ม |
| Cloud-managed | Meraki MR (cloud dashboard) | องค์กรที่ต้องการ Cloud-first ไม่มี on-prem controller |

WLC-1 ใน Lab นี้เป็น **Catalyst 9800-CL** ซึ่งรัน **IOS-XE เดียวกันกับ Catalyst 9000 Switch** —
CLI จึงคุ้นเคยกับที่เรียนมาตั้งแต่ Part 2 (ต่างจาก AireOS WLC รุ่นเก่า เช่น 5520/8540 ที่มี CLI คนละแบบ)

### Centralized (Local) Mode vs FlexConnect Mode

| คุณสมบัติ | Centralized / Local Mode | FlexConnect Mode |
|---|---|---|
| ตำแหน่ง WLC เทียบกับ AP | WLC อยู่ Data Center/Campus เดียวกับ AP (LAN เดียวกัน หรือ WAN latency ต่ำ) | WLC อยู่ Central Site, AP อยู่ Branch Office ไกลออกไปผ่าน WAN |
| Client Data Traffic | ส่งผ่าน CAPWAP tunnel ไปที่ WLC ก่อน แล้วค่อยออกสู่ VLAN ปลายทาง (Central Switching) | AP switch traffic ออก local ที่ branch เอง (Local Switching) — ไม่ต้องผ่าน WAN ทุก packet |
| ผลกระทบถ้า WAN Link ล่ม | Client หลุดทั้งหมด (data plane ขาด) | Client ยังใช้งาน local resource ได้ (FlexConnect "standalone mode") |
| Use Case ทั่วไปในหลักสูตรนี้ | ✅ **ใช้ใน Lab นี้** — AP-1/AP-2 อยู่ใน Campus เดียวกับ WLC-1 | จะกล่าวถึงเพิ่มเติมสำหรับ Branch ใน Part 39 (Multi-site Design) |
| ความซับซ้อนในการ config | ต่ำกว่า | สูงกว่า (ต้อง config local VLAN mapping ที่ AP) |

เนื่องจาก Lab Topology ของหลักสูตรนี้เป็น Single-Campus (WLC-1, AP-1, AP-2 อยู่ในเครือข่ายเดียวกัน
ทั้งหมด ผ่าน DIST-SW3/DIST-SW4) เราจะใช้ **Centralized (Local) Mode** ตลอด Part นี้ — Client
traffic จาก AP จะเดินทางผ่าน CAPWAP Data tunnel ไปที่ WLC-1 ก่อน แล้ว WLC-1 ค่อย forward
เข้า VLAN 40 (WIFI) — นี่คือจุดสำคัญที่ต้องเข้าใจ: **VLAN 40 ที่ Client ได้ IP ไม่ต้องเทรงค์ไปถึง
Access Switch ที่ AP เสียบ (ACCESS-SW3/SW4) เพราะ traffic ถูก tunnel ผ่าน CAPWAP ไปที่ WLC
โดยตรง** — สิ่งที่ต้องมีที่ ACCESS-SW3/SW4 คือแค่ VLAN 99 (MGMT) สำหรับ AP ต่อ Control Plane เท่านั้น

```
Centralized Mode Data Path (สำคัญมาก - เข้าใจผิดบ่อย)

Client (VLAN 40 IP) ── 802.11 ──► AP-1 ── CAPWAP Data (UDP 5247) ──► WLC-1
                                  (ผ่าน VLAN 99 MGMT บน ACCESS-SW3)   │
                                                                       │ Decapsulate
                                                                       ▼
                                                            SVI VLAN 40 (Dynamic Interface)
                                                            ที่ WLC-1 หรือ Trunk ไปยัง DIST-SW
                                                                       │
                                                                       ▼
                                                            10.10.40.0/24 Network

** VLAN 40 ไม่จำเป็นต้องมีบน ACCESS-SW3/SW4 เพื่อให้ AP ทำงานได้ — เพราะ traffic ของ Client
   ถูก tunnel ทับ VLAN 99 (MGMT) ไปหา WLC-1 โดยตรงใน Centralized Mode **
```

---

## Step 194 — CAPWAP Discovery & Join Process

AP ที่แกะกล่องมาใหม่ (Factory Default) ไม่รู้จัก WLC ของตัวเองเลย — ต้องผ่านกระบวนการ
**Discovery** เพื่อหา WLC ก่อน แล้วค่อย **Join** เข้าไปเป็นสมาชิก

### วิธีที่ AP ใช้ค้นหา WLC (Discovery Methods) — เรียงจากที่ใช้บ่อยที่สุด

| วิธี | อธิบาย | ใช้เมื่อไหร่ |
|---|---|---|
| **DHCP Option 43** | DHCP Server ส่ง IP ของ WLC กลับมาพร้อม DHCP Offer (Vendor-Specific Option) | **วิธีมาตรฐานของ Enterprise** — ใช้ใน Lab นี้ |
| **DNS "CISCO-CAPWAP-CONTROLLER"** | AP resolve ชื่อ `CISCO-CAPWAP-CONTROLLER.<domain>` ผ่าน DNS | Backup method ถ้าไม่มี Option 43 |
| **Layer 3 Broadcast** | AP ส่ง Broadcast บน Local Subnet เพื่อหา WLC ที่อยู่ Subnet เดียวกัน | ใช้ได้เฉพาะ AP/WLC อยู่ VLAN เดียวกัน (พบใน Lab เล็ก) |
| **Over-the-Air Provisioning (OTAP)** | AP ที่ join แล้วช่วยบอก AP ตัวใหม่ (ผ่าน RF) | Legacy, ปัจจุบันไม่แนะนำ |
| **Priming (Static/Previously joined)** | AP จำ WLC ตัวก่อนหน้าที่เคย join ไว้ใน NVRAM | AP ที่ถูกย้ายจากที่หนึ่งไปอีกที่ |
| **Manual Static Configuration** | Admin ใส่ IP ของ WLC ตรงๆ ผ่าน AP CLI | Lab/Test environment |

### ตัวอย่าง DHCP Option 43 บน Cisco IOS DHCP Server (ทบทวนจาก Part 16)

```
! DHCP Pool สำหรับ VLAN 99 (MGMT) ที่ AP-1/AP-2 จะขอ IP มาใช้เป็น Control-Plane IP
DIST-SW3(config)# ip dhcp pool MGMT-VLAN99
DIST-SW3(dhcp-config)# network 10.10.99.0 255.255.255.0
DIST-SW3(dhcp-config)# default-router 10.10.99.1
DIST-SW3(dhcp-config)# dns-server 10.10.99.53
DIST-SW3(dhcp-config)# domain-name lab.local
!
! Option 43 ต้องเป็น Hex string พิเศษ: type(0xf1) + length + IP ของ WLC (Hex)
! สูตรคำนวณ: f1 04 <4 byte IP ของ WLC เป็น Hex>
! WLC-1 = 10.10.99.40 = 0A.0A.63.28 (hex)
DIST-SW3(dhcp-config)# option 43 hex f104.0a0a.6328
DIST-SW3(dhcp-config)# exit
```

> **คำนวณ Option 43 แบบ manual**: `10.10.99.40` → แปลงแต่ละ octet เป็น hex →
> 10=0A, 10=0A, 99=63, 40=28 → ตามด้วย prefix `f1` (type) กับ `04` (length=4 byte)
> รวมเป็น `f1040a0a6328` — Cisco IOS ยอมรับในรูปแบบ dotted-hex `f104.0a0a.6328`

### CAPWAP Join State Machine

```
┌──────────────┐    ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐    ┌──────────┐
│  1. Discovery  │───►│  2. Join       │───►│ 3. Image Download  │───►│ 4. Config      │───►│ 5. Run    │
│  (หา WLC)      │    │  (DTLS setup,  │    │  (ถ้า Image ไม่ตรง  │    │  (รับ Config    │    │  (ทำงาน   │
│                │    │   ส่ง Join Req) │    │   version - reboot) │    │   จาก WLC)      │    │   ปกติ)    │
└──────────────┘    └──────────────┘    └──────────────────┘    └──────────────┘    └──────────┘
```

**รายละเอียดแต่ละ State:**

1. **Discovery** — AP ส่ง CAPWAP Discovery Request ไปยัง WLC ที่เจอจากวิธีข้างบน (อาจส่งไปหลาย
   WLC พร้อมกันถ้ามีมากกว่า 1 ตัว) แต่ละ WLC ตอบ Discovery Response กลับมาพร้อมข้อมูล เช่น
   AP-count ปัจจุบัน, Load — AP จะเลือก WLC ที่เหมาะสมที่สุด (โดย default = WLC ที่ load น้อยที่สุด)

2. **Join** — AP สร้าง **DTLS (Datagram TLS)** tunnel กับ WLC ที่เลือกไว้ก่อน (เข้ารหัส Control
   channel ด้วย Certificate ที่ AP มีติดตัวมาจากโรงงาน — MIC: Manufacturer Installed Certificate)
   แล้วส่ง Join Request ผ่าน tunnel ที่เข้ารหัสแล้ว

3. **Image Download** — WLC ตรวจสอบ AP Software version เทียบกับ version ที่ WLC เก็บไว้
   ถ้าไม่ตรงกัน AP จะโหลด Image ใหม่จาก WLC ผ่าน CAPWAP แล้ว **reboot ตัวเอง** (ทำให้รอบแรกที่
   AP join ใหม่มักใช้เวลานานกว่าปกติ)

4. **Configuration** — WLC ส่ง Config ทั้งหมดให้ AP: SSID ที่ต้อง broadcast, Channel/Power ที่
   RRM คำนวณให้, Security Policy — AP ไม่มี Config เก็บถาวรของตัวเอง (Stateless)

5. **Run** — AP เข้าสถานะทำงานปกติ พร้อมรับ Client, ส่ง Keepalive (CAPWAP Echo Request/Response)
   ไปที่ WLC ทุก ~30 วินาทีเพื่อยืนยันว่ายังออนไลน์อยู่

> **ข้อสอบเจาะจง**: DTLS ใช้เข้ารหัส **CAPWAP Control channel เสมอ (บังคับ ปิดไม่ได้)** ส่วน
> CAPWAP Data channel (Client traffic) เป็น **option** — เปิด/ปิดได้ที่ WLC configuration

---

## Step 195 — คอนฟิก WLC พื้นฐาน (Catalyst 9800): WLAN Profile, Policy/Site/RF Tag

### สถาปัตยกรรม "Tag-Based" ของ Catalyst 9800 (IOS-XE) — ต่างจาก AireOS รุ่นเก่าโดยสิ้นเชิง

WLC รุ่นเก่า (AireOS เช่น 5520/8540/2504) ใช้แนวคิด "WLAN → Interface (VLAN) → AP Group"
แบบง่ายๆ แต่ **Catalyst 9800 (IOS-XE)** ใช้สถาปัตยกรรมใหม่ที่แยก **Configuration** ออกจาก
**Policy** อย่างชัดเจน เพื่อให้ Scale ได้ดีขึ้นในองค์กรใหญ่ — ประกอบด้วย 4 ส่วนหลัก:

| Component | หน้าที่ | เทียบเท่า (AireOS เดิม) |
|---|---|---|
| **WLAN Profile** | นิยาม SSID, Security (WPA2/WPA3, PSK/802.1X) | WLAN |
| **Policy Profile** | นิยามว่า Client ของ WLAN นี้จะไปที่ VLAN ไหน, QoS, ACL, Session Timeout | Interface + AP Group mapping |
| **Policy Tag** | จับคู่ (Map) ระหว่าง WLAN Profile ↔ Policy Profile — เป็นตัวที่ผูกกับ AP | AP Group |
| **Site Tag** | กำหนดว่า AP กลุ่มนี้เป็น Local mode หรือ FlexConnect, และ join ผ่าน AP Join Profile ไหน | AP Group (บางส่วน) |
| **RF Tag** | กำหนด RF Profile (Channel/Power policy) สำหรับ 2.4GHz/5GHz ให้ AP กลุ่มนี้ | RF Profile |

```
Catalyst 9800 Tag-Based Architecture (Diagram)

┌────────────────┐        ┌───────────────────┐
│  WLAN Profile    │        │  Policy Profile     │
│  "ENTERPRISE-    │        │  "ENTERPRISE-       │
│   WIFI-PROFILE"  │        │   WIFI-POLICY"      │
│  SSID:            │        │  VLAN: 40 (WIFI)     │
│  ENTERPRISE-WIFI  │        │  10.10.40.0/24       │
│  Security: WPA2   │        │                     │
└────────┬─────────┘        └────────┬───────────┘
         │                             │
         └───────────┬─────────────────┘
                      ▼
            ┌───────────────────┐
            │   Policy Tag         │   <- จับคู่ WLAN ↔ Policy Profile
            │  "ENTERPRISE-TAG"    │
            └──────────┬──────────┘
                        │
     ┌──────────────────┼──────────────────┐
     ▼                                       ▼
┌───────────┐                          ┌───────────┐
│ Site Tag    │  <- Local/FlexConnect     │  RF Tag     │  <- Channel/Power Policy
│ "SITE-1"    │                          │  "RF-1"     │
└─────┬─────┘                          └─────┬─────┘
      │                                        │
      └───────────────┬────────────────────────┘
                       ▼
              ┌─────────────────┐
              │   AP-1 / AP-2      │  <- ทั้ง 3 Tag ถูก assign ให้ AP ใน Step 196
              └─────────────────┘
```

> **จำง่ายๆ**: WLAN Profile บอกว่า "SSID ชื่ออะไร ใส่ Password แบบไหน" ส่วน Policy Profile บอกว่า
> "Client ที่เข้ามาทาง SSID นี้จะได้ VLAN อะไร" แล้ว Policy Tag คือตัวเชื่อมทั้งสองเข้าด้วยกัน
> และผูกเข้ากับ AP อีกที ผ่าน Site Tag (โหมดการทำงาน) และ RF Tag (RF Policy)

### Step-by-Step: สร้าง SSID "ENTERPRISE-WIFI" แบบเต็ม (WPA2-Enterprise พื้นฐาน)

```
WLC-1# configure terminal

! ============================================
! 1. สร้าง VLAN 40 บน WLC-1 ก่อน (ต้องมีทั้งสองฝั่ง)
! ============================================
WLC-1(config)# vlan 40
WLC-1(config-vlan)# name WIFI
WLC-1(config-vlan)# exit

! ============================================
! 2. สร้าง WLAN Profile (นิยาม SSID + Security)
!    Syntax: wlan <profile-name> <wlan-id 1-512> <ssid-name>
! ============================================
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE 1 ENTERPRISE-WIFI
WLC-1(config-wlan)# security wpa akm dot1x            ! ใช้ 802.1X (รายละเอียดเต็มใน Part 37)
WLC-1(config-wlan)# security wpa wpa2                 ! บังคับ WPA2 (WPA3 จะเรียนเพิ่มใน Part 37)
WLC-1(config-wlan)# no security wpa akm dot1x-sha256   ! ปิด option ที่ไม่ใช้ในตอนนี้
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit

! ============================================
! 3. สร้าง Policy Profile (กำหนด VLAN ปลายทาง)
! ============================================
WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY
WLC-1(config-wireless-policy)# vlan 40
WLC-1(config-wireless-policy)# no shutdown
WLC-1(config-wireless-policy)# exit

! ============================================
! 4. สร้าง Policy Tag แล้ว Map WLAN Profile <-> Policy Profile
! ============================================
WLC-1(config)# wireless tag policy ENTERPRISE-TAG
WLC-1(config-policy-tag)# wlan ENTERPRISE-WIFI-PROFILE policy ENTERPRISE-WIFI-POLICY
WLC-1(config-policy-tag)# exit

! ============================================
! 5. สร้าง Site Tag (Centralized/Local Mode - ไม่ใช่ FlexConnect ตามที่ตัดสินใจใน Step 193)
! ============================================
WLC-1(config)# wireless tag site SITE-CAMPUS-MAIN
WLC-1(config-site-tag)# no flex-profile               ! ไม่ใช้ FlexConnect
WLC-1(config-site-tag)# no local-site                 ! หมายเหตุ: default AP join profile ใช้ได้เลย
WLC-1(config-site-tag)# exit

! ============================================
! 6. สร้าง RF Tag (ใช้ RF Profile default ของ Cisco เป็นจุดเริ่มต้น)
! ============================================
WLC-1(config)# wireless tag rf RF-CAMPUS-MAIN
WLC-1(config-rf-tag)# 24ghz-rf-policy Typical_Client_Density_rf_24gHz
WLC-1(config-rf-tag)# 5ghz-rf-policy Typical_Client_Density_rf_5gHz
WLC-1(config-rf-tag)# exit

WLC-1(config)# end
WLC-1# copy running-config startup-config
```

### ตรวจสอบผลลัพธ์

```
WLC-1# show wlan summary
Number of WLANs: 1
--------------------------------------------------------------------------------
ID   Profile Name                      SSID                            Status
--------------------------------------------------------------------------------
1    ENTERPRISE-WIFI-PROFILE           ENTERPRISE-WIFI                 UP

WLC-1# show wireless profile policy detailed ENTERPRISE-WIFI-POLICY | include VLAN
  VLAN                                     : 40
```

---

## Step 196 — ลงทะเบียน AP และกำหนด Tag ให้ AP-1/AP-2

Tag ที่สร้างใน Step 195 (Policy Tag, Site Tag, RF Tag) ยังไม่มีผลอะไรจนกว่าจะถูก**ผูกเข้ากับ AP
จริง** — มี 2 วิธีในการ assign tag:

1. **Static** (ผูกด้วยชื่อ AP หรือ MAC Address ล่วงหน้า ก่อน AP join) — แนะนำสำหรับ Production
2. **AP Filter / Tag Source Priority** (ผูกแบบ dynamic ตาม rule) — ใช้ในองค์กรใหญ่ที่มี AP จำนวนมาก

Lab นี้ใช้วิธี Static เพราะมี AP เพียง 2 ตัว ทำให้ตรวจสอบและสอนเข้าใจง่ายที่สุด

### คอนฟิก: ผูก Tag ให้ AP-1 (เชื่อมต่อ ACCESS-SW3) และ AP-2 (เชื่อมต่อ ACCESS-SW4)

```
WLC-1# configure terminal

! ต้องรู้ AP-name หรือ Ethernet MAC Address ของ AP ก่อน — เช็คจาก AP ที่ join มาแล้วแบบ default
! (AP join ด้วย default tag ก่อน แล้วเราค่อยเปลี่ยนเป็น tag ที่ถูกต้อง)

! ============================================
! AP-1 (เชื่อมต่อ ACCESS-SW3, VLAN 99 สำหรับ control-plane)
! ============================================
WLC-1(config)# ap name AP-1
WLC-1(config-ap-tag)# policy-tag ENTERPRISE-TAG
WLC-1(config-ap-tag)# site-tag SITE-CAMPUS-MAIN
WLC-1(config-ap-tag)# rf-tag RF-CAMPUS-MAIN
WLC-1(config-ap-tag)# exit

! ============================================
! AP-2 (เชื่อมต่อ ACCESS-SW4, VLAN 99 สำหรับ control-plane)
! ============================================
WLC-1(config)# ap name AP-2
WLC-1(config-ap-tag)# policy-tag ENTERPRISE-TAG
WLC-1(config-ap-tag)# site-tag SITE-CAMPUS-MAIN
WLC-1(config-ap-tag)# rf-tag RF-CAMPUS-MAIN
WLC-1(config-ap-tag)# exit

WLC-1(config)# end
WLC-1# copy running-config startup-config
```

> **หมายเหตุ**: การเปลี่ยน Tag บน AP ที่ join อยู่แล้วจะทำให้ **AP restart CAPWAP session
> ชั่วคราว** (join ใหม่พร้อม tag ที่ถูกต้อง) — ปกติใช้เวลาไม่เกิน 1-2 นาที ควรทำนอกเวลาทำงาน
> จริงในระบบ Production

### ตั้งชื่อ AP ให้ตรงกับ Hostname convention ของ Lab (ถ้า AP ยัง join มาด้วยชื่อ default)

```
! บาง AP join มาด้วยชื่อ default ตาม MAC Address เช่น "AP70b3.1234.5678"
! เปลี่ยนชื่อให้ตรงกับ Lab Topology (AP-1, AP-2) ด้วยคำสั่งนี้จาก WLC
WLC-1# ap name AP70b3.1234.5678 name AP-1
WLC-1# ap name AP70b3.1234.abcd name AP-2
```

### ผลลัพธ์ที่คาดหวังหลัง Tag ถูก apply

```
WLC-1# show ap tag summary
Number of APs: 2
------------------------------------------------------------------------------------------
AP Name    Policy Tag Name   Site Tag Name        RF Tag Name        Misconfigured Tag
------------------------------------------------------------------------------------------
AP-1       ENTERPRISE-TAG    SITE-CAMPUS-MAIN      RF-CAMPUS-MAIN     No
AP-2       ENTERPRISE-TAG    SITE-CAMPUS-MAIN      RF-CAMPUS-MAIN     No
```

---

## Step 197 — Wireless Security พื้นฐาน: WPA2-Personal vs Enterprise

> **สำคัญ**: Part นี้สอนแค่ระดับพื้นฐานที่ต้องรู้สำหรับ CCNA เท่านั้น รายละเอียด 802.1X แบบเต็ม
> (RADIUS Server, EAP methods, Certificate-based auth), WPA3-Enterprise 192-bit mode, และการ
> integrate กับ Cisco ISE จะเรียนแบบละเอียดใน **Part 37 (Wireless Security)** และ **Part 44
> (Advanced Wireless/ISE)**

### เปรียบเทียบโหมดความปลอดภัย Wireless หลัก

| โหมด | การ Authenticate | Key Management | ใช้เมื่อไหร่ | ระดับความปลอดภัย |
|---|---|---|---|---|
| **Open** | ไม่มี | ไม่เข้ารหัสเลย | Guest network สาธารณะเท่านั้น | ต่ำสุด (ไม่ควรใช้กับ Enterprise data) |
| **WPA2-Personal (PSK)** | Pre-Shared Key (Password เดียวใช้ร่วมกันทุกคน) | 4-Way Handshake จาก PSK | SOHO, บ้าน, IoT device ที่ทำ 802.1X ไม่ได้ | กลาง — ทุกคนรู้ password เดียวกัน เสี่ยงถ้า password หลุด |
| **WPA3-Personal (SAE)** | Simultaneous Authentication of Equals (แก้ปัญหา offline dictionary attack ของ WPA2 PSK) | SAE Handshake | บ้าน/SOHO ที่ต้องการความปลอดภัยสูงกว่า PSK | สูงกว่า WPA2-Personal |
| **WPA2-Enterprise (802.1X)** | Username/Password หรือ Certificate ผ่าน RADIUS Server | Dynamic key ต่อ user/session (ไม่มี key เดียวที่ทุกคนรู้) | **Enterprise ทั้งหมด — มาตรฐานองค์กร** | สูง — key เปลี่ยนต่อ session, revoke user ได้ทันที |
| **WPA3-Enterprise** | เหมือน 802.1X แต่บังคับ Protected Management Frame (PMF) และรองรับ 192-bit mode | Dynamic + PMF บังคับ | Enterprise ที่ต้องการมาตรฐานสูงสุด (Gov/Finance) | สูงสุด |

### ทำไม Enterprise ต้องใช้ 802.1X แทน PSK?

- **PSK เดียวใช้ร่วมกันทั้งองค์กร** — ถ้าพนักงานลาออกแล้วยังจำ password ได้ ต้องเปลี่ยน PSK
  ทั้งองค์กร (กระทบ Client ที่เชื่อมต่ออยู่ทุกเครื่อง)
- **802.1X ระบุตัวบุคคลได้** — RADIUS Server (เช่น Cisco ISE) รู้ว่า "user A" login มาจาก AP ไหน
  เวลาไหน ตรวจสอบ Compliance ของอุปกรณ์ได้ (Posture Assessment) ก่อนอนุญาตเข้าเครือข่าย
- **Revoke สิทธิ์รายบุคคลได้ทันที** — ปิด Account ที่ Active Directory/RADIUS โดยไม่กระทบคนอื่น

### คอนฟิก PSK-based WLAN สำหรับ Lab (Baseline ทดสอบเร็ว ก่อนเรียน 802.1X เต็มใน Part 37)

เนื่องจากใน Lab เรายังไม่มี RADIUS Server (จะสร้างใน Part 22 - AAA และ Part 37) เราจะใช้
**WPA2-Personal (PSK)** เป็น Baseline ทดสอบว่า Client เชื่อมต่อ VLAN 40 ได้จริงก่อน แล้วจะ
Upgrade เป็น 802.1X ใน Part 37

```
WLC-1# configure terminal

! แก้ WLAN Profile เดิมจาก Step 195 ให้ใช้ PSK แทน 802.1X ชั่วคราว
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE
WLC-1(config-wlan)# shutdown
WLC-1(config-wlan)# no security wpa akm dot1x
WLC-1(config-wlan)# security wpa akm psk set-key ascii 0 P@ssw0rdEnterprise2026!
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit
WLC-1(config)# end
WLC-1# copy running-config startup-config
```

> **หมายเหตุ**: `set-key ascii 0` หมายถึง key เป็น plaintext ตอนใส่ (Cisco จะเก็บแบบเข้ารหัสใน
> running-config โดย default เป็น type 6 บน IOS-XE) — ห้ามใช้ PSK ที่คาดเดาง่ายในระบบจริง
> ควรมีความยาวขั้นต่ำ 12-16 ตัวอักษร ผสมตัวเลข/สัญลักษณ์

---

## Step 198 — แนวคิด Client Roaming: L2 Roaming, L3 Roaming, Mobility Group

### Client Roaming คืออะไร

**Roaming** คือกระบวนการที่ Client (เช่น มือถือที่กำลังเดิน) เปลี่ยนการเชื่อมต่อจาก AP ตัวหนึ่ง
ไปยัง AP อีกตัวโดยที่ **Session ไม่ขาด** (เช่น กำลัง Video Call อยู่ ต้องไม่หลุด)

### Layer 2 Roaming — Client ย้าย AP แต่ยังอยู่ VLAN เดิม

```
Client เชื่อมกับ AP-1 (VLAN 40, IP 10.10.40.50) เดินไปอีกฝั่งอาคาร
              │
              ▼ (RF สัญญาณ AP-1 อ่อนลง, AP-2 สัญญาณแรงขึ้น)
Client roam ไปเชื่อมกับ AP-2 (VLAN 40 เดียวกัน)
              │
              ▼
Client ยังคง IP 10.10.40.50 เดิม — ไม่ต้องขอ DHCP ใหม่ ไม่มี Session ขาด
```

เนื่องจากทั้ง AP-1 และ AP-2 ใน Lab นี้ join WLC-1 ตัวเดียวกัน และใช้ Policy Profile เดียวกัน
(VLAN 40) — WLC-1 รู้อยู่แล้วว่า Client ตัวนี้เคย associate กับ AP-1 มาก่อน จึงทำการ**ย้าย Client
Database Entry ภายใน WLC เอง** (Intra-Controller Roaming) โดยไม่ต้องขอ IP ใหม่ — Roaming
แบบนี้เร็วมาก (มักไม่เกิน 100ms) เพราะ Layer 3 (IP subnet) ไม่เปลี่ยน

### Layer 3 Roaming — Client ย้ายข้าม WLC ที่อยู่ VLAN/Subnet คนละอัน

L3 Roaming เกิดขึ้นเมื่อ Client ย้ายไปยัง AP ที่ join กับ **WLC ตัวอื่น** ที่มี Policy Profile
ผูกกับ **VLAN คนละอัน** (เช่น กรณี Multi-Campus ที่มี WLC หลายตัว แต่ละที่ผูก VLAN ท้องถิ่นตัวเอง)

- ถ้าไม่มีการจัดการพิเศษ Client จะต้องขอ DHCP ใหม่ (IP เปลี่ยน) → Session ที่เปิดอยู่ (เช่น
  TCP connection เดิม) จะขาด เพราะ IP source เปลี่ยนกลางทาง
- Cisco แก้ปัญหานี้ด้วย **Mobility Tunnel** ระหว่าง WLC (EoIP - Ethernet over IP tunnel):
  WLC ปลายทางจะ tunnel traffic ของ Client กลับไปยัง WLC ต้นทางเพื่อให้ Client "เก็บ IP เดิม"
  ไว้ได้แม้ย้าย Physical location — เรียกว่า **Auto Anchor / Mobility Anchor** design

### Mobility Group — แนวคิดสำหรับ Multi-WLC (ใน Lab นี้มี WLC-1 เพียงตัวเดียว)

**Mobility Group** คือกลุ่มของ WLC หลายตัวที่ตั้งค่าให้รู้จักกัน (แชร์ Mobility Message
Exchange ผ่าน UDP 16666) เพื่อให้ Roaming ระหว่าง WLC ทำงานได้ราบรื่น — WLC ทุกตัวใน Group
เดียวกันจะ sync ข้อมูล Client Database ระหว่างกัน

```
! ตัวอย่าง config Mobility Group (concept - ใน Lab นี้ยังไม่ต้องทำเพราะมี WLC-1 ตัวเดียว)
WLC-1(config)# wireless mobility group member ip <WLC-2-IP> public-ip <WLC-2-IP> group <group-name>
```

> **ใน Lab ของหลักสูตรนี้มี WLC-1 เพียงตัวเดียว** จึงมีแต่ **Intra-Controller L2 Roaming**
> ระหว่าง AP-1 ↔ AP-2 เท่านั้น — เนื้อหา Multi-WLC Mobility Group และ Inter-Controller L3
> Roaming แบบเต็มรูปแบบ (รวม N+1 HA, SSO) จะกลับมาเรียนเมื่อ Lab ขยายเป็น Multi-Site ใน
> **Part 39 (Multi-site Design)** และ **Part 44 (Advanced Wireless)**

---

## Step 199 — Verification & Troubleshooting WLC/AP

### คำสั่ง Verify หลักบน Catalyst 9800

```
! สรุปสถานะ Wireless โดยรวมทั้งระบบ
WLC-1# show wireless summary

! สรุป AP ทั้งหมดที่ join (คำสั่งที่ใช้บ่อยที่สุด)
WLC-1# show ap summary

! สรุป WLAN ทั้งหมด
WLC-1# show wlan summary

! สรุป Client ที่เชื่อมต่ออยู่
WLC-1# show wireless client summary

! รายละเอียดของ AP ตัวเดียว
WLC-1# show ap name AP-1 config general

! ดู Tag ที่ผูกกับ AP
WLC-1# show ap tag summary

! ดู CAPWAP session detail ของ AP (debug join process)
WLC-1# show ap name AP-1 capwap detail
```

### ตัวอย่าง Output จริงที่ต้องอ่านให้เป็น

```
WLC-1# show wireless summary
Number of Access Points                : 2
Configured Country                     : TH  (Thailand)
Total Radios Count                     : 4
Total Client Count                     : 0

WLC-1# show ap summary
Number of APs: 2

AP Name    Slots  AP Model     Ethernet MAC     Radio MAC       State
-------------------------------------------------------------------------
AP-1       2      C9130AXI     70b3.1712.aa01   70b3.1712.aa00  Registered
AP-2       2      C9130AXI     70b3.1712.bb01   70b3.1712.bb00  Registered

WLC-1# show wireless client summary
Number of Local Clients: 1

MAC Address     AP Name    WLAN  State      Protocol
------------------------------------------------------
aabb.ccdd.1122   AP-1       1     Run        11ax(5GHz)
```

### สาเหตุที่พบบ่อยที่สุดเมื่อ AP ไม่ Join (Registered) กับ WLC

| อาการ | State ที่เห็น | สาเหตุที่พบบ่อยที่สุด | วิธีแก้ |
|---|---|---|---|
| AP ไม่เจอ WLC เลย | ไม่ปรากฏใน `show ap summary` | VLAN 99 (MGMT) ไม่ถูก trunk มาถึง Access Port ที่ AP เสียบ, หรือ AP อยู่ VLAN ผิด | เช็ค `show vlan brief`, `show interfaces trunk` บน ACCESS-SW3/SW4 |
| AP ไม่ได้ IP | State = ค้างที่ boot, ไม่มี IP บน AP | DHCP Pool VLAN 99 ไม่ทำงาน, Option 43 ผิด | `show ip dhcp binding`, ตรวจ Option 43 hex string |
| AP ได้ IP แต่ join ไม่ได้ | State = Discovery ค้าง | DNS record `CISCO-CAPWAP-CONTROLLER` ไม่มี, Option 43 ชี้ IP ผิด | ping จาก AP ไปยัง WLC Management IP, ตรวจ Option 43 |
| AP join ได้แต่หลุดๆ | State = Registered ↔ Disconnect สลับ | Certificate mismatch (เวลา AP/WLC ไม่ sync — ต้องมี NTP!), MTU/Firewall block UDP 5246/5247 | `show ap name <ap> capwap detail`, ตรวจ NTP (Part 17), ACL ที่ block CAPWAP port |
| Client เชื่อม Wi-Fi ไม่ได้ IP | Client associate สำเร็จแต่ไม่มี IP | VLAN 40 ไม่มี DHCP Pool, หรือ Policy Profile ผูก VLAN ผิด | `show wireless profile policy detailed`, ตรวจ DHCP Pool VLAN 40 |
| Client เห็น SSID แต่ Authentication fail | Client แสดง "Authentication failed" | PSK/802.1X credential ผิด, RADIUS Server ไม่ตอบ (ถ้าใช้ Enterprise mode) | `show wireless client mac-address <mac> detail`, ตรวจ RADIUS reachability |

> **กฎสำคัญที่มักลืม**: WLC และ AP ต้องมี **เวลาตรงกัน (NTP)** เสมอ เพราะ DTLS Certificate
> validation ตรวจสอบเวลาด้วย — ถ้า Clock skew มากเกินไป AP จะ join ไม่ได้แม้ Network ปกติทุกอย่าง
> (ทบทวน NTP configuration จาก **Part 17**)

---

## Step 200 — Lab เต็มรูปแบบ: ตั้งค่า WLC-1, ลงทะเบียน AP-1/AP-2, ทดสอบ Client บน VLAN 40

Lab นี้รวบรวมทุก Step ของ Part 20 เข้าเป็น Configuration เดียวที่สมบูรณ์ พร้อม verification
เต็มรูปแบบ ตาม Canonical Address Plan (`docs/00-ip-address-plan.md`):

- **WLC-1**: Management IP `10.10.99.40` (VLAN 99, MGMT)
- **AP-1**: เชื่อมต่อ ACCESS-SW3 → รับ DHCP/Static ใน VLAN 99 สำหรับ Control-Plane
- **AP-2**: เชื่อมต่อ ACCESS-SW4 → รับ DHCP/Static ใน VLAN 99 สำหรับ Control-Plane
- **SSID**: `ENTERPRISE-WIFI` → mapped ไปยัง **VLAN 40 (WIFI, 10.10.40.0/24)**

### 1) เตรียม Access Switch — ACCESS-SW3 (ที่ AP-1 เสียบอยู่)

```
ACCESS-SW3# configure terminal
ACCESS-SW3(config)# interface GigabitEthernet0/3
ACCESS-SW3(config-if)# description ** Uplink to AP-1 (Catalyst 9130) **
ACCESS-SW3(config-if)# switchport mode access
ACCESS-SW3(config-if)# switchport access vlan 99
ACCESS-SW3(config-if)# switchport voice vlan none
ACCESS-SW3(config-if)# spanning-tree portfast
ACCESS-SW3(config-if)# power inline auto
ACCESS-SW3(config-if)# no shutdown
ACCESS-SW3(config-if)# end
ACCESS-SW3# copy running-config startup-config
```

> AP รุ่น Catalyst 9130 กินไฟผ่าน **PoE+ (802.3at)** เท่านั้น (ต้องใช้ Switch ที่รองรับ PoE+
> ขั้นต่ำ 30W ต่อพอร์ต) — คำสั่ง `power inline auto` เปิดให้ Switch จ่ายไฟอัตโนมัติตามที่ AP ขอ

### 2) เตรียม Access Switch — ACCESS-SW4 (ที่ AP-2 เสียบอยู่) — เหมือนกันทุกจุด

```
ACCESS-SW4# configure terminal
ACCESS-SW4(config)# interface GigabitEthernet0/3
ACCESS-SW4(config-if)# description ** Uplink to AP-2 (Catalyst 9130) **
ACCESS-SW4(config-if)# switchport mode access
ACCESS-SW4(config-if)# switchport access vlan 99
ACCESS-SW4(config-if)# switchport voice vlan none
ACCESS-SW4(config-if)# spanning-tree portfast
ACCESS-SW4(config-if)# power inline auto
ACCESS-SW4(config-if)# no shutdown
ACCESS-SW4(config-if)# end
ACCESS-SW4# copy running-config startup-config
```

### 3) ตั้งค่า Management IP บน WLC-1 (Catalyst 9800-CL, VLAN 99)

```
WLC-1# configure terminal
WLC-1(config)# hostname WLC-1
WLC-1(config)# no ip domain-lookup
WLC-1(config)# ip domain-name lab.local
WLC-1(config)# vlan 99
WLC-1(config-vlan)# name MGMT
WLC-1(config-vlan)# exit

WLC-1(config)# interface Vlan99
WLC-1(config-if)# description ** WLC-1 Management/Control-Plane Interface **
WLC-1(config-if)# ip address 10.10.99.40 255.255.255.0
WLC-1(config-if)# no shutdown
WLC-1(config-if)# exit

! ตั้งค่า Wireless Management Interface ให้ชี้ไปที่ VLAN 99 (จำเป็นสำหรับ CAPWAP)
WLC-1(config)# wireless management interface Vlan99

! ตั้ง Default Gateway (อ้างอิง HSRP VIP ของ VLAN 99 ตาม Part 18)
WLC-1(config)# ip route 0.0.0.0 0.0.0.0 10.10.99.1

! NTP - จำเป็นสำหรับ DTLS Certificate validation (Step 199)
WLC-1(config)# ntp server 10.10.99.50

! Enable Secret + SSH (ตาม Convention จาก Part 2)
WLC-1(config)# enable secret Cisco123!
WLC-1(config)# username admin privilege 15 secret StrongP@ssw0rd!
WLC-1(config)# crypto key generate rsa modulus 2048
WLC-1(config)# ip ssh version 2
WLC-1(config)# line vty 0 15
WLC-1(config-line)# transport input ssh
WLC-1(config-line)# login local
WLC-1(config-line)# exit
```

### 4) VLAN 40 (WIFI) — สร้างบน WLC-1 และ Trunk ที่เชื่อมต่อ WLC-1 ↔ DIST-SW3/DIST-SW4

```
WLC-1(config)# vlan 40
WLC-1(config-vlan)# name WIFI
WLC-1(config-vlan)# exit

! Uplink ของ WLC-1 ต้อง Trunk ทั้ง VLAN 99 (control) และ VLAN 40 (client data - Centralized Mode)
WLC-1(config)# interface TenGigabitEthernet0/0/1
WLC-1(config-if)# description ** Uplink to DIST-SW3 - Trunk VLAN 40,99 **
WLC-1(config-if)# switchport mode trunk
WLC-1(config-if)# switchport trunk allowed vlan 40,99
WLC-1(config-if)# no shutdown
WLC-1(config-if)# exit
```

**ฝั่ง DIST-SW3 (ทบทวนจาก Part 3/10 — ต้อง Trunk รับ WLC-1 เข้ามาด้วย):**

```
DIST-SW3(config)# interface GigabitEthernet1/0/24
DIST-SW3(config-if)# description ** Uplink to WLC-1 - Trunk VLAN 40,99 **
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk allowed vlan 40,99
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit
```

### 5) DHCP Pool สำหรับ VLAN 40 (WIFI Client) และ VLAN 99 (AP Control-Plane) — ทบทวน Part 16

```
DIST-SW3(config)# ip dhcp excluded-address 10.10.40.1 10.10.40.10
DIST-SW3(config)# ip dhcp excluded-address 10.10.99.1 10.10.99.39
DIST-SW3(config)#
DIST-SW3(config)# ip dhcp pool WIFI-VLAN40
DIST-SW3(dhcp-config)# network 10.10.40.0 255.255.255.0
DIST-SW3(dhcp-config)# default-router 10.10.40.1
DIST-SW3(dhcp-config)# dns-server 10.10.99.53
DIST-SW3(dhcp-config)# lease 8
DIST-SW3(dhcp-config)# exit
DIST-SW3(config)#
DIST-SW3(config)# ip dhcp pool MGMT-VLAN99-AP
DIST-SW3(dhcp-config)# network 10.10.99.0 255.255.255.0
DIST-SW3(dhcp-config)# default-router 10.10.99.1
DIST-SW3(dhcp-config)# dns-server 10.10.99.53
DIST-SW3(dhcp-config)# domain-name lab.local
DIST-SW3(dhcp-config)# option 43 hex f104.0a0a.6328
DIST-SW3(dhcp-config)# exit
```

### 6) สร้าง SSID, Policy Profile, Tags ทั้งหมด (รวม Step 195-197)

```
WLC-1(config)# wlan ENTERPRISE-WIFI-PROFILE 1 ENTERPRISE-WIFI
WLC-1(config-wlan)# security wpa akm psk set-key ascii 0 P@ssw0rdEnterprise2026!
WLC-1(config-wlan)# security wpa wpa2
WLC-1(config-wlan)# no shutdown
WLC-1(config-wlan)# exit

WLC-1(config)# wireless profile policy ENTERPRISE-WIFI-POLICY
WLC-1(config-wireless-policy)# vlan 40
WLC-1(config-wireless-policy)# no shutdown
WLC-1(config-wireless-policy)# exit

WLC-1(config)# wireless tag policy ENTERPRISE-TAG
WLC-1(config-policy-tag)# wlan ENTERPRISE-WIFI-PROFILE policy ENTERPRISE-WIFI-POLICY
WLC-1(config-policy-tag)# exit

WLC-1(config)# wireless tag site SITE-CAMPUS-MAIN
WLC-1(config-site-tag)# no flex-profile
WLC-1(config-site-tag)# exit

WLC-1(config)# wireless tag rf RF-CAMPUS-MAIN
WLC-1(config-rf-tag)# 24ghz-rf-policy Typical_Client_Density_rf_24gHz
WLC-1(config-rf-tag)# 5ghz-rf-policy Typical_Client_Density_rf_5gHz
WLC-1(config-rf-tag)# exit
```

### 7) ลงทะเบียน AP-1 และ AP-2 พร้อม Tag (รวม Step 196)

```
WLC-1(config)# ap name AP-1
WLC-1(config-ap-tag)# policy-tag ENTERPRISE-TAG
WLC-1(config-ap-tag)# site-tag SITE-CAMPUS-MAIN
WLC-1(config-ap-tag)# rf-tag RF-CAMPUS-MAIN
WLC-1(config-ap-tag)# exit

WLC-1(config)# ap name AP-2
WLC-1(config-ap-tag)# policy-tag ENTERPRISE-TAG
WLC-1(config-ap-tag)# site-tag SITE-CAMPUS-MAIN
WLC-1(config-ap-tag)# rf-tag RF-CAMPUS-MAIN
WLC-1(config-ap-tag)# exit

WLC-1(config)# end
WLC-1# copy running-config startup-config
Building configuration...
[OK]
```

### 8) Verification เต็มรูปแบบ — ยืนยันว่าทุกอย่างทำงาน

```
WLC-1# show wlan summary
Number of WLANs: 1
--------------------------------------------------------------------------------
ID   Profile Name                      SSID                            Status
--------------------------------------------------------------------------------
1    ENTERPRISE-WIFI-PROFILE           ENTERPRISE-WIFI                 UP

WLC-1# show ap summary
Number of APs: 2

AP Name    Slots  AP Model     Ethernet MAC     Radio MAC       State
-------------------------------------------------------------------------
AP-1       2      C9130AXI     70b3.1712.aa01   70b3.1712.aa00  Registered
AP-2       2      C9130AXI     70b3.1712.bb01   70b3.1712.bb00  Registered

WLC-1# show ap tag summary
Number of APs: 2
------------------------------------------------------------------------------------------
AP Name    Policy Tag Name   Site Tag Name        RF Tag Name        Misconfigured Tag
------------------------------------------------------------------------------------------
AP-1       ENTERPRISE-TAG    SITE-CAMPUS-MAIN      RF-CAMPUS-MAIN     No
AP-2       ENTERPRISE-TAG    SITE-CAMPUS-MAIN      RF-CAMPUS-MAIN     No

WLC-1# show ap name AP-1 config general | include Ethernet Address|IP Address|State
  Ethernet Address           : 70b3.1712.aa01
  IP Address                 : 10.10.99.101
  AP State                   : Registered

WLC-1# show ap name AP-2 config general | include Ethernet Address|IP Address|State
  Ethernet Address           : 70b3.1712.bb01
  IP Address                 : 10.10.99.102
  AP State                   : Registered
```

### 9) ทดสอบ Client เชื่อมต่อจริง — Client บนมือถือ/Laptop เชื่อม SSID "ENTERPRISE-WIFI"

```
[บน Client Laptop]
1. เปิด Wi-Fi Scan -> เจอ SSID "ENTERPRISE-WIFI"
2. เชื่อมต่อด้วย Password: P@ssw0rdEnterprise2026!
3. Client ได้รับ IP จาก DHCP Pool VLAN 40:

C:\> ipconfig
Wireless LAN adapter Wi-Fi:
   Connection-specific DNS Suffix  . : lab.local
   IPv4 Address. . . . . . . . . . . : 10.10.40.101
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.40.1

C:\> ping 10.10.40.1
Reply from 10.10.40.1: bytes=32 time=1ms TTL=255
Reply from 10.10.40.1: bytes=32 time=1ms TTL=255

C:\> ping 10.10.30.10          [ทดสอบไปยัง Server ใน VLAN 30 - ผ่าน Inter-VLAN Routing]
Reply from 10.10.30.10: bytes=32 time=2ms TTL=254
```

**ตรวจสอบจากฝั่ง WLC-1 ว่าเห็น Client จริง:**

```
WLC-1# show wireless client summary
Number of Local Clients: 1

MAC Address     AP Name    WLAN  State      Protocol
------------------------------------------------------
aabb.ccdd.1122   AP-1       1     Run        11ax(5GHz)

WLC-1# show wireless client mac-address aabb.ccdd.1122 detail | include IPv4 Address|AP Name|VLAN|State
  Client IPv4 Address                   : 10.10.40.101
  AP Name                               : AP-1
  VLAN                                  : 40
  State                                 : Associated
```

Lab นี้ยืนยันครบทุกจุดที่ต้องการ: **AP-1 และ AP-2 Registered เข้า WLC-1**, **SSID
ENTERPRISE-WIFI ทำงาน**, **Client ได้ IP ใน VLAN 40 (10.10.40.0/24) ตาม Canonical Address
Plan**, และ **Client สามารถ Route ข้าม VLAN ไปยัง Server (VLAN 30) ได้ผ่าน Inter-VLAN Routing
ที่ตั้งไว้ตั้งแต่ Part 10**

---

## แบบฝึกหัดทวนความเข้าใจ Part 20

1. CAPWAP ใช้ UDP Port อะไรสำหรับ Control channel และ Data channel ตามลำดับ และช่องไหนที่
   บังคับต้องเข้ารหัสด้วย DTLS เสมอ?
2. อธิบายความแตกต่างระหว่าง WLAN Profile, Policy Profile, และ Policy Tag บน Catalyst 9800
   — แต่ละอย่างทำหน้าที่อะไร?
3. เพราะเหตุใด Enterprise Network จึงเลือกใช้ 5 GHz เป็นหลักแทน 2.4 GHz ทั้งที่ 2.4 GHz มี
   ระยะทำการไกลกว่า?
4. ในสถาปัตยกรรม Centralized (Local) Mode ของ WLC เหตุใด VLAN 40 (WIFI) จึงไม่จำเป็นต้องถูก
   Trunk ไปถึง ACCESS-SW3/SW4 ที่ AP เสียบอยู่?
5. หาก AP-1 ได้ IP จาก DHCP ใน VLAN 99 แล้ว แต่ State ยังค้างอยู่ที่ "Discovery" ไม่ขึ้น
   "Registered" ควรตรวจสอบอะไรเป็นลำดับแรก?

**เฉลย:**
1. Control channel = UDP 5246 (บังคับเข้ารหัส DTLS เสมอ), Data channel = UDP 5247
   (DTLS เป็น option เปิด/ปิดได้)
2. WLAN Profile นิยาม SSID และ Security (WPA2/PSK/802.1X); Policy Profile นิยามว่า Client
   ของ WLAN นั้นจะไปที่ VLAN ไหน/QoS/ACL; Policy Tag คือตัวจับคู่ (Map) ระหว่าง WLAN Profile
   กับ Policy Profile แล้วนำไปผูกกับ AP
3. 5 GHz มี Channel ที่ไม่ทับซ้อนกันมากกว่า (20+ channel เทียบกับ 3 channel ของ 2.4 GHz) และ
   มีสัญญาณรบกวนจากอุปกรณ์อื่นน้อยกว่า ทำให้ Throughput สูงกว่าและ Interference น้อยกว่า
   เหมาะกับ Data-heavy client แม้ระยะทำการจะสั้นกว่า
4. เพราะใน Centralized Mode, Client Data traffic ถูก encapsulate เป็น CAPWAP Data tunnel
   (UDP 5247) ส่งผ่าน VLAN 99 (MGMT) ไปที่ WLC-1 โดยตรง แล้ว WLC-1 เป็นผู้ decapsulate และ
   ส่งเข้า VLAN 40 เอง — ACCESS-SW3/SW4 จึงต้องมีแค่ VLAN 99 เท่านั้น
5. ตรวจสอบ DNS record "CISCO-CAPWAP-CONTROLLER" หรือค่า Option 43 ว่าชี้ไป WLC IP ที่ถูกต้อง
   หรือไม่ (`10.10.99.40`), ตรวจ Firewall/ACL ที่อาจ block UDP 5246, และตรวจว่า NTP sync
   ตรงกันระหว่าง AP กับ WLC หรือไม่ (Clock skew ทำให้ DTLS Certificate validation fail)

---

## สรุป Part 20

Part นี้พาเราเข้าสู่โลก Wireless LAN อย่างเต็มรูปแบบ: ตั้งแต่ทฤษฎี 802.11/ความถี่/Channel,
สถาปัตยกรรม Lightweight AP + WLC แบบ Split-MAC ผ่าน CAPWAP tunnel, การเลือก Deployment Mode
ที่เหมาะกับ Lab (Centralized), กระบวนการ Discovery/Join ของ AP, ไปจนถึงการคอนฟิก **Catalyst
9800 WLC-1** แบบ Tag-based Architecture (WLAN Profile → Policy Profile → Policy/Site/RF Tag)
และปิดท้ายด้วย Lab เต็มรูปแบบที่ทำให้ **AP-1/AP-2 Registered สำเร็จ** และ **Client เชื่อมต่อ
SSID "ENTERPRISE-WIFI" ได้ IP ใน VLAN 40 (10.10.40.0/24) จริง** ✅

Wireless Security แบบละเอียด (802.1X, WPA3-Enterprise, RADIUS/ISE Integration) ยังไม่ได้เรียน
เต็มรูปแบบใน Part นี้ — เราใช้ PSK เป็น Baseline ทดสอบเท่านั้น จะกลับมาเรียนเชิงลึกใน **Part 37**

**ไปต่อ:** [Part 21 — Switch Security →](part-021-switch-security.md)
