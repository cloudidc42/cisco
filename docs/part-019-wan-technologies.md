# Part 19 — WAN Technologies (PPP, MPLS Intro, VPN Intro)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 181–190 จาก 1000**

> ต่อจาก [Part 18 — First Hop Redundancy Protocols (HSRP/VRRP/GLBP)](part-018-fhrp.md) ที่แก้ปัญหา
> Redundancy ของ Default Gateway ภายใน Campus แล้ว Part นี้จะขยับมุมมองออกจาก Campus ไปสู่
> **WAN (Wide Area Network)** — เส้นทางที่เชื่อมต่อ HQ ของเราไปยังโลกภายนอกผ่าน **WAN-EDGE-1**
> และ **WAN-EDGE-2** (ที่ประกาศไว้ตั้งแต่ [00-ip-address-plan.md](00-ip-address-plan.md))
> เราจะเรียนรู้ Layer 2 WAN Protocol ที่สำคัญที่สุด (**PPP**), แนวคิดของ **MPLS L3VPN** จาก
> มุมมองลูกค้า (CE), และพื้นฐานของ **Site-to-Site VPN** ด้วย GRE Tunnel — เพื่อเตรียมทาง
> ไปสู่ IPsec/DMVPN แบบเต็มรูปแบบใน Part 51-53 และ MPLS L3VPN ระดับ Provider ใน Part 86-87
>
> Part นี้จะแนะนำอุปกรณ์จำลองใหม่ 1 ตัว คือ **BRANCH-RTR** — Router ของสำนักงานสาขาขนาดเล็ก
> ที่ต้องการเชื่อมต่อ WAN กลับมายัง HQ ผ่าน **WAN-EDGE-1** (ใช้เฉพาะการสอนเรื่อง WAN link ใน
> Part นี้และ Part ที่เกี่ยวข้องกับ VPN/MPLS ในอนาคต ไม่ได้อยู่ใน Core Lab Topology หลักของ
> Campus ที่ประกาศใน Part 1)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 181 | ภาพรวมเทคโนโลยี WAN — Leased Line, MPLS, Metro Ethernet, Internet VPN, SD-WAN |
| 182 | PPP พื้นฐาน — LCP/NCP Phase, Frame Format, การรองรับ Authentication |
| 183 | คอนฟิก PPP บน Serial/Point-to-Point Interface — WAN-EDGE-1 ↔ BRANCH-RTR |
| 184 | PPP Authentication — PAP vs CHAP |
| 185 | PPP Multilink (MLPPP) — รวมหลาย Physical Link เป็น Logical Link เดียว |
| 186 | MPLS พื้นฐาน — Label Switching, LSR/LER, มุมมอง L3VPN จากฝั่งลูกค้า |
| 187 | คอนฟิก MPLS L3VPN จากมุมมอง CE (WAN-EDGE-1 peering กับ PE) |
| 188 | Site-to-Site VPN พื้นฐาน — GRE Tunnel ระหว่าง WAN-EDGE-1 ↔ BRANCH-RTR |
| 189 | IPsec พื้นฐาน — ISAKMP/IKE Phase 1-2 และแนวคิด GRE-over-IPsec |
| 190 | Lab เต็มรูปแบบ: PPP+CHAP over Leased Line + GRE Tunnel + OSPF over Tunnel |

---

## Step 181 — ภาพรวมเทคโนโลยี WAN

**WAN (Wide Area Network)** คือเครือข่ายที่เชื่อมต่อ Site ที่อยู่ห่างกันทางภูมิศาสตร์ (คนละเมือง
คนละประเทศ) ต่างจาก LAN ที่อยู่ในอาคาร/พื้นที่เดียวกัน องค์กรจริงมักมีหลาย Site (HQ, Branch,
Data Center) ที่ต้องคุยกันผ่าน WAN ที่เช่าจาก **Service Provider (SP)** เพราะไม่มีองค์กรใดวางสาย
ไฟเบอร์ข้ามเมืองข้ามประเทศเอง

### ตัวเลือกเทคโนโลยี WAN หลักที่ต้องรู้จัก

| เทคโนโลยี | ลักษณะการทำงาน | ค่าใช้จ่าย | ความน่าเชื่อถือ (SLA) | Bandwidth ทั่วไป | Use Case หลัก |
|---|---|---|---|---|---|
| **Leased Line** (P2P Circuit) | สายเช่าเฉพาะจุดต่อจุด ไม่แชร์กับใคร (TDM/T1/E1 สมัยก่อน, ปัจจุบันมักเป็น Ethernet-based) | สูงมาก | สูงสุด (Dedicated, SLA แน่น) | 1.5 Mbps – 10 Gbps | Backbone ระหว่าง HQ-DC ที่ต้องการความแน่นอนสูงสุด |
| **MPLS L3VPN** | SP สร้าง Virtual Full-Mesh ให้ผ่าน Label Switching, ลูกค้าเห็นเหมือนมี Router เดียวคุยกับทุก Site | สูง (ต่ำกว่า Leased Line) | สูง (SLA จาก SP, มี QoS class) | 2 Mbps – หลาย Gbps | Enterprise ที่มีหลาย Site จำนวนมาก ต้องการ QoS/Any-to-Any |
| **Metro Ethernet** | Ethernet-based WAN ในระดับเมือง (MAN) ใช้ VLAN/Q-in-Q แยก Customer | กลาง | สูง (SLA ระดับ Metro) | 10 Mbps – 10 Gbps | เชื่อม Site ภายในเมืองเดียวกัน/ใกล้กัน ราคาคุ้มกว่า Leased Line |
| **Internet VPN** (Site-to-Site หรือ DMVPN) | ใช้ Internet สาธารณะ + เข้ารหัส IPsec เพื่อสร้าง Virtual Private Network | ต่ำมาก | ปานกลาง (best-effort, ไม่มี SLA จาก ISP ทั่วไป) | ขึ้นกับ Internet link (หลัก Mbps – Gbps) | Branch เล็ก, Backup Link, องค์กรที่งบจำกัด |
| **SD-WAN** | Overlay ที่รวมหลาย Underlay (MPLS+Internet+4G/5G) บริหารจาก Controller กลาง, เลือก Path อัตโนมัติตาม Application | กลาง-สูง (ROI จากการลด MPLS) | สูง (Active/Active หลาย Underlay + Dynamic Path Selection) | รวมทุก Underlay | Enterprise สมัยใหม่ที่ต้องการลดต้นทุน MPLS แต่คง Reliability |

> **หมายเหตุสำคัญ**: Part นี้เน้น **PPP (Layer 2 บน Leased Line)**, **MPLS L3VPN แนวคิดพื้นฐาน**,
> และ **Site-to-Site VPN (GRE + แนวคิด IPsec)** เท่านั้น — รายละเอียดเชิงลึกของแต่ละเทคโนโลยีจะ
> กลับมาเรียนอีกครั้งในระดับที่สูงขึ้น:
> - **IPsec/DMVPN แบบเต็มรูปแบบ** → Part 51-53
> - **MPLS L3VPN ระดับ Provider (PE-CE, VRF, Route-Target, MP-BGP)** → Part 86-87 (ระดับ CCIE)
> - **SD-WAN (Cisco Catalyst SD-WAN/Viptela)** → Part 39 (ภาพรวม) และ Part 76 (เชิงลึก)

### ทำไมต้องรู้จักเทคโนโลยีเหล่านี้ทั้งหมด

ในความเป็นจริง Enterprise เกือบทุกแห่งใช้ **หลายเทคโนโลยีผสมกัน**: MPLS เป็น Primary Link
ไปยัง Branch สำคัญ, Internet VPN เป็น Backup Link, และ Data Center เชื่อมกันด้วย Leased Line
หรือ Metro Ethernet ความเข้าใจ **trade-off ด้านต้นทุนกับความน่าเชื่อถือ** คือสิ่งที่ทำให้
Network Engineer ออกแบบ WAN ได้อย่างเหมาะสมกับงบประมาณและ SLA ที่ธุรกิจต้องการ

---

## Step 182 — PPP พื้นฐาน (Point-to-Point Protocol)

**PPP (Point-to-Point Protocol, RFC 1661)** คือ Layer 2 WAN Protocol มาตรฐานเปิดที่ใช้บน
Serial Link / Leased Line แบบจุดต่อจุด (Point-to-Point) แทน HDLC (ซึ่งเป็น Cisco proprietary
default บน Serial interface) เพราะ PPP รองรับความสามารถที่ HDLC ไม่มี เช่น **Authentication**,
**Multilink**, **Compression**, และการทำงานร่วมกับอุปกรณ์ต่างยี่ห้อ (Multi-vendor)

### PPP ทำงานผ่าน 2 Phase หลัก: LCP และ NCP

```
      Link ขึ้น (Physical Layer พร้อม)
                │
                ▼
    ┌───────────────────────────┐
    │   LCP (Link Control        │   <-- เจรจาคุณสมบัติของ Link เอง
    │   Protocol) Phase          │       (MTU, Authentication, Magic Number,
    │                            │        Multilink, Compression)
    └─────────────┬─────────────┘
                  │ LCP สำเร็จ
                  ▼
    ┌───────────────────────────┐
    │   Authentication Phase     │   <-- ถ้าตกลงใช้ PAP/CHAP (optional)
    │   (PAP หรือ CHAP)          │
    └─────────────┬─────────────┘
                  │ Authen ผ่าน (หรือไม่ได้ตั้ง auth)
                  ▼
    ┌───────────────────────────┐
    │   NCP (Network Control     │   <-- เจรจาคุณสมบัติของ Layer 3
    │   Protocol) Phase          │       เช่น IPCP (สำหรับ IPv4),
    │   IPCP / IPv6CP            │       IPv6CP (สำหรับ IPv6)
    └─────────────┬─────────────┘
                  │ NCP สำเร็จ
                  ▼
         Link พร้อมส่ง Data (Network Layer up)
```

- **LCP (Link Control Protocol)**: จัดการเปิด/ปิด/ทดสอบ Data Link — ตกลงพารามิเตอร์ เช่น
  MRU (Maximum Receive Unit), Authentication Protocol ที่จะใช้, Magic Number (ป้องกัน
  Loopback link), Multilink capability
- **NCP (Network Control Protocol)**: มีหนึ่งตัวต่อหนึ่ง Network Layer Protocol — ที่ใช้บ่อย
  ที่สุดคือ **IPCP (IP Control Protocol)** สำหรับเจรจา IP Address บน Link (ใช้เยอะในงาน Dial-up
  แบบเก่าที่ ISP แจก IP ให้ Client แบบ dynamic) และ **IPv6CP** สำหรับ IPv6

### PPP Frame Format

```
┌─────────┬──────────┬──────────┬───────────────┬─────┐
│  Flag   │ Address  │ Control  │  Protocol      │Data │ FCS │
│ (1 byte)│ (1 byte)  │ (1 byte) │  (2 byte)      │(var)│(2B) │
│ 0x7E    │ 0xFF      │ 0x03     │ e.g. 0xC021=LCP│     │     │
└─────────┴──────────┴──────────┴───────────────┴─────┴─────┘
```

- **Flag**: `0x7E` เป็นตัวบอกจุดเริ่ม/จุดสิ้นสุดของ Frame (เหมือน HDLC)
- **Protocol Field**: บอกว่า Payload เป็น Protocol อะไร เช่น `0xC021` = LCP, `0x8021` = IPCP,
  `0x0021` = IP datagram (data จริง)
- PPP เป็น **bit-oriented protocol** คล้าย HDLC แต่มี field และ Phase การเจรจาที่ซับซ้อนกว่า
  เพื่อรองรับ feature ต่างๆ ข้างต้น

### PPP รองรับ Authentication 2 แบบ (จะลงรายละเอียดใน Step 184)

| Authentication | ลักษณะ | ความปลอดภัย |
|---|---|---|
| **PAP** (Password Authentication Protocol) | ส่ง Username/Password แบบ cleartext, 2-way handshake | ต่ำ |
| **CHAP** (Challenge Handshake Authentication Protocol) | ใช้ Challenge + Hash (MD5), 3-way handshake, ไม่ส่ง password ตรงๆ | สูงกว่า |

---

## Step 183 — คอนฟิก PPP บน Serial/Point-to-Point Interface

เราจะจำลอง Leased Line ระหว่าง **WAN-EDGE-1** (HQ) กับ **BRANCH-RTR** (สำนักงานสาขาใหม่ที่
แนะนำใน Part นี้) โดยใช้ Serial interface แบบ back-to-back (ใน Lab จริงมักจำลองด้วย
Serial cable back-to-back หรือ Frame Relay Switch เสมือนบน GNS3/CML — แนวคิดการคอนฟิกเหมือนกัน
ไม่ว่าจะเป็น Physical Serial หรือ WAN Emulation)

### IP Addressing สำหรับ Leased Line นี้ (เฉพาะ Scenario สอน WAN ใน Part นี้)

| Link | Subnet | IP ปลาย A | IP ปลาย B |
|---|---|---|---|
| WAN-EDGE-1 ↔ BRANCH-RTR (Leased Line, Serial) | 172.16.199.0/30 | WAN-EDGE-1 Se0/0/0 = .1 | BRANCH-RTR Se0/0/0 = .2 |

> IP Range `172.16.199.0/30` ถูกเลือกจาก Private Class B (RFC 1918) เพื่อไม่ชนกับ Subnet
> Plan หลักใน [00-ip-address-plan.md](00-ip-address-plan.md) — ใช้เฉพาะสำหรับ Scenario จำลอง
> WAN/MPLS/VPN ใน Part นี้และ Part ที่เกี่ยวข้องในอนาคต

### ตรวจสอบ Encapsulation เริ่มต้นก่อนเปลี่ยน

Cisco Router ที่มี Serial interface จะใช้ **HDLC (Cisco proprietary)** เป็นค่า Default:

```
WAN-EDGE-1# show interface Serial0/0/0
Serial0/0/0 is up, line protocol is up
  Hardware is GT96K
  Description: ** Leased Line to BRANCH-RTR **
  Internet address is 172.16.199.1/30
  MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,
  Encapsulation HDLC, crc 16, loopback not set
```

### คอนฟิก PPP บน WAN-EDGE-1

```
WAN-EDGE-1(config)# interface Serial0/0/0
WAN-EDGE-1(config-if)# description ** Leased Line to BRANCH-RTR **
WAN-EDGE-1(config-if)# ip address 172.16.199.1 255.255.255.252
WAN-EDGE-1(config-if)# encapsulation ppp
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
```

### คอนฟิก PPP บน BRANCH-RTR

```
BRANCH-RTR(config)# interface Serial0/0/0
BRANCH-RTR(config-if)# description ** Leased Line to WAN-EDGE-1 (HQ) **
BRANCH-RTR(config-if)# ip address 172.16.199.2 255.255.255.252
BRANCH-RTR(config-if)# encapsulation ppp
BRANCH-RTR(config-if)# no shutdown
BRANCH-RTR(config-if)# exit
```

### Verify ว่า PPP LCP/NCP เจรจาสำเร็จ

```
WAN-EDGE-1# show interface Serial0/0/0
Serial0/0/0 is up, line protocol is up
  Encapsulation PPP, LCP Open, multilink Closed
  Open: IPCP, CDPCP

WAN-EDGE-1# show ppp all
PPP Serial Interface Serial0/0/0
Session id [1]
    LCP: Open
    IPCP: Open
    CDPCP: Open
```

- `LCP: Open` = LCP Phase เจรจาสำเร็จ (Link พร้อมทำงานที่ Layer 2)
- `IPCP: Open` = NCP Phase สำหรับ IPv4 เจรจาสำเร็จ (Layer 3 พร้อมส่ง IP packet)
- `CDPCP: Open` = PPP ยังรองรับส่ง CDP ผ่าน Link ได้ด้วย (Cisco extension)

```
WAN-EDGE-1# ping 172.16.199.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.199.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

> **สำคัญ**: ถ้า Encapsulation ไม่ตรงกันสองฝั่ง (เช่น ฝั่งหนึ่งเป็น `ppp` อีกฝั่งยังเป็น `hdlc`)
> จะเห็น `show ip interface brief` เป็น **Status = up, Protocol = down** — ปัญหา Layer 2
> Mismatch แบบเดียวกับที่กล่าวถึงใน [Part 2 Step 12](part-002-ios-cli-basics.md)

---

## Step 184 — PPP Authentication: PAP vs CHAP

PPP รองรับการยืนยันตัวตนระหว่างสองฝั่งของ Link ก่อนจะปล่อยให้ NCP เจรจาต่อ — สำคัญมากสำหรับ
Dial-up/Leased Line ที่เช่าจาก SP เพื่อป้องกันไม่ให้อุปกรณ์ที่ไม่ได้รับอนุญาต connect เข้ามา

### PAP (Password Authentication Protocol) — 2-Way Handshake

```
   WAN-EDGE-1                              BRANCH-RTR
      │  ---- Username + Password (cleartext) ---->  │
      │  <----------- Accept / Reject -------------- │
```

- ส่ง Username/Password ครั้งเดียวตอนเริ่มต้น Link เท่านั้น (ไม่ทำซ้ำระหว่าง session)
- ส่งแบบ **cleartext** ผ่าน Link — ถ้ามีคนดักฟัง (sniffing) จะเห็น password ตรงๆ
- **ไม่ป้องกัน Replay Attack** (ดักจับแล้วส่งซ้ำได้)
- ใช้ในกรณีที่ปลายทางไม่รองรับ CHAP เท่านั้น — **ไม่แนะนำสำหรับใช้งานจริง**

### CHAP (Challenge Handshake Authentication Protocol) — 3-Way Handshake

```
   WAN-EDGE-1 (Authenticator)              BRANCH-RTR
      │  -------- 1. Challenge --------------------> │
      │  <------- 2. Response (MD5 Hash) ------------│
      │  -------- 3. Success/Failure --------------->│
```

1. **Challenge**: ฝั่งที่ทำหน้าที่ Authenticator ส่ง Random Challenge value ไปให้อีกฝั่ง
2. **Response**: อีกฝั่งนำ Challenge + Password (Shared Secret) มาผ่าน **MD5 Hash** แล้วส่ง
   Hash ผลลัพธ์กลับไป (ไม่ส่ง Password ตรงๆ)
3. **Success/Failure**: Authenticator คำนวณ Hash แบบเดียวกันด้วย Password ที่ตนเองมี
   ถ้าตรงกัน = Success

- **ปลอดภัยกว่า PAP มาก** เพราะไม่ส่ง Password ผ่าน Link เลย (ส่งแค่ Hash)
- **ป้องกัน Replay Attack** ได้ เพราะ Challenge เปลี่ยนค่าทุกครั้ง (Random) และ CHAP ยังทำ
  Re-challenge เป็นระยะระหว่าง session ได้ด้วย
- ทั้งสองฝั่งต้องรู้ **Password/Secret เดียวกัน** ล่วงหน้า (Shared Secret แบบ Symmetric)

### เปรียบเทียบสรุป

| คุณสมบัติ | PAP | CHAP |
|---|---|---|
| จำนวน Handshake | 2-way | 3-way |
| การส่ง Password | Cleartext | ไม่ส่ง (ส่งแค่ Hash) |
| ป้องกัน Replay Attack | ไม่ป้องกัน | ป้องกัน |
| ทำ Re-authentication ระหว่าง Session | ไม่ทำ | ทำได้ (Re-challenge) |
| ความปลอดภัย | ต่ำ | สูงกว่า |
| คำแนะนำ | ใช้เมื่อจำเป็นเท่านั้น | **แนะนำเป็น Default** |

### คอนฟิก CHAP เต็มรูปแบบระหว่าง WAN-EDGE-1 ↔ BRANCH-RTR

**หลักการสำคัญ**: `username <hostname ของอีกฝั่ง> password <secret>` — ต้องตรงกับ **hostname
ที่อีกฝั่งใช้แสดงตัว** และ **password ต้องเหมือนกันทั้งสองฝั่ง** (CHAP เป็น Symmetric Secret)

**บน WAN-EDGE-1:**

```
WAN-EDGE-1(config)# username BRANCH-RTR password Cisco@WAN123
WAN-EDGE-1(config)# interface Serial0/0/0
WAN-EDGE-1(config-if)# encapsulation ppp
WAN-EDGE-1(config-if)# ppp authentication chap
WAN-EDGE-1(config-if)# exit
```

**บน BRANCH-RTR:**

```
BRANCH-RTR(config)# username WAN-EDGE-1 password Cisco@WAN123
BRANCH-RTR(config)# interface Serial0/0/0
BRANCH-RTR(config-if)# encapsulation ppp
BRANCH-RTR(config-if)# ppp authentication chap
BRANCH-RTR(config-if)# exit
```

### Verify CHAP

```
WAN-EDGE-1# debug ppp authentication
PPP Serial0/0/0: Send CHAP challenge id=1 to remote
PPP Serial0/0/0: CHAP challenge received from BRANCH-RTR
PPP Serial0/0/0: Send CHAP response id=1
PPP Serial0/0/0: CHAP response received from BRANCH-RTR
PPP Serial0/0/0: CHAP peer authentication succeeded for BRANCH-RTR
WAN-EDGE-1# undebug all

WAN-EDGE-1# show interface Serial0/0/0 | include LCP
  Encapsulation PPP, LCP Open, multilink Closed
```

> **ข้อผิดพลาดที่พบบ่อยที่สุด**: ตั้ง `username` ผิด (สลับชื่อตัวเองกับชื่ออีกฝั่ง) หรือ
> Password ไม่ตรงกันทั้งสองด้าน — จะเห็น log แบบ `CHAP peer authentication failed` และ
> `show interface` จะค้างที่ `LCP: Open` แต่ไม่เห็น `IPCP: Open` เพราะ Link จะไม่ผ่านไป Phase NCP

---

## Step 185 — PPP Multilink (MLPPP)

**Multilink PPP (MLPPP, RFC 1990)** คือการรวม **หลาย Physical Link แยกกัน** ให้ทำงานเป็น
**Logical Link เดียว (Bundle)** เพื่อเพิ่ม Bandwidth รวม และทำ Load-Balancing แบบ Per-Packet
ระหว่าง Link ในกลุ่ม — มีประโยชน์มากเมื่อ SP ให้เช่า Leased Line หลายเส้นเล็กๆ (เช่น T1 หลายเส้น)
แทนเส้นใหญ่เส้นเดียว หรือเมื่อต้องการ Redundancy: ถ้า Link หนึ่งล่ม Bundle ยังทำงานต่อได้
(Bandwidth ลดลงแต่ไม่ล่มทั้งหมด)

```
   WAN-EDGE-1                                    BRANCH-RTR
   ┌──────────────┐         Serial0/0/0          ┌──────────────┐
   │              ├───────────────────────────────┤              │
   │  Multilink1  │         Serial0/0/1          │  Multilink1  │
   │  (Logical)   ├───────────────────────────────┤  (Logical)   │
   └──────────────┘                                └──────────────┘
     รวม BW ของทั้งสอง Physical Link เป็น Logical Interface เดียว
```

### คอนฟิกตัวอย่าง MLPPP (2 Serial Link รวมกัน)

**บน WAN-EDGE-1:**

```
WAN-EDGE-1(config)# interface Multilink1
WAN-EDGE-1(config-if)# ip address 172.16.198.1 255.255.255.252
WAN-EDGE-1(config-if)# ppp multilink
WAN-EDGE-1(config-if)# ppp multilink group 1
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# interface range Serial0/0/0 , Serial0/0/1
WAN-EDGE-1(config-if-range)# encapsulation ppp
WAN-EDGE-1(config-if-range)# ppp multilink
WAN-EDGE-1(config-if-range)# ppp multilink group 1
WAN-EDGE-1(config-if-range)# no shutdown
WAN-EDGE-1(config-if-range)# exit
```

**บน BRANCH-RTR:** (คอนฟิกแบบเดียวกัน กลับด้าน IP ให้ตรงกับ Subnet)

```
BRANCH-RTR(config)# interface Multilink1
BRANCH-RTR(config-if)# ip address 172.16.198.2 255.255.255.252
BRANCH-RTR(config-if)# ppp multilink
BRANCH-RTR(config-if)# ppp multilink group 1
BRANCH-RTR(config-if)# exit
!
BRANCH-RTR(config)# interface range Serial0/0/0 , Serial0/0/1
BRANCH-RTR(config-if-range)# encapsulation ppp
BRANCH-RTR(config-if-range)# ppp multilink
BRANCH-RTR(config-if-range)# ppp multilink group 1
BRANCH-RTR(config-if-range)# no shutdown
BRANCH-RTR(config-if-range)# exit
```

### Verify MLPPP

```
WAN-EDGE-1# show ppp multilink
Multilink1
  Bundle name: BRANCH-RTR
  Bundle up for 00:02:15
  0 lost fragments, 0 reordered, 0 unassigned
  Bandwidth (kbps) - Effective: 3088, Load Threshold: 255
  Member links: 2 (max not set, min not set)
    Serial0/0/0, since 00:02:15
    Serial0/0/1, since 00:02:10
```

- **Effective Bandwidth 3088 kbps** = ผลรวมของสอง Serial Link (1544+1544 kbps) — พิสูจน์ว่า
  ทำงานเป็น Logical Link เดียวสำเร็จ
- ใน CCNP/CCIE ระดับสูง MLPPP มักถูกแทนที่ด้วย Multilink บน MPLS/Ethernet หรือ **EtherChannel**
  (ที่เรียนไปแล้วใน [Part 6](part-006-etherchannel.md) — แนวคิด Bundle เพื่อเพิ่ม Bandwidth
  เหมือนกัน แต่ทำที่ Layer 2 Ethernet)

---

## Step 186 — MPLS พื้นฐาน (Multiprotocol Label Switching)

**MPLS (Multiprotocol Label Switching)** คือเทคโนโลยีที่ Service Provider ใช้สร้าง Backbone
Network ขนาดใหญ่ที่ Forward Packet โดยใช้ **Label** (เลขสั้นๆ) แทนการดู IP Destination Address
เต็มๆ ทุก Hop แบบ Traditional IP Routing — ทำให้ Forward เร็วขึ้นและยืดหยุ่นในการทำ Traffic
Engineering / VPN แยก Customer จำนวนมากบน Infrastructure เดียวกัน

### แนวคิด Label Switching

```
Traditional IP Routing:  ทุก Router ต้อง Look up Routing Table เต็มรูปแบบ (Longest Prefix Match)
                          ทุก Hop จนถึงปลายทาง

MPLS Label Switching:    Router ที่ Edge (LER) ติด "Label" (เช่น เลข 100) เข้าไปที่ Packet
                          Router กลาง (LSR) แค่ดู Label แล้ว Swap เป็น Label ใหม่ (Label Swap)
                          ไม่ต้องดู IP Header เต็มๆ อีกจนกว่าจะถึง Edge อีกด้าน
```

```
                    MPLS Provider Backbone (SP Network)
   CE1 ──── PE1 ════[Label 100]════ P1 ════[Label 200]════ PE2 ──── CE2
  (ลูกค้า) (Edge)                  (Core)                 (Edge)  (ลูกค้า)
           LER                     LSR                     LER
```

### บทบาทของอุปกรณ์ใน MPLS Network

| อุปกรณ์ | ชื่อเต็ม | บทบาท |
|---|---|---|
| **LER** (Label Edge Router) | Label Edge Router (คือ PE — Provider Edge) | อยู่ที่ขอบ MPLS Network, ทำหน้าที่ **ติด Label เข้า (Push)** ตอนแพ็กเก็ตเข้า Backbone และ **แกะ Label ออก (Pop)** ตอนออกจาก Backbone — เป็นจุดเดียวที่ต้องดู IP Header เต็มรูปแบบ |
| **LSR** (Label Switch Router) | Label Switch Router (คือ P — Provider Core) | อยู่ใน Core ของ MPLS Network, ทำหน้าที่ **Swap Label** เท่านั้น (เปลี่ยน Label ขาเข้าเป็น Label ขาออกตาม Label Forwarding Table) ไม่ต้องดู IP Header |
| **CE** (Customer Edge) | Customer Edge Router | Router ของลูกค้า (เช่น WAN-EDGE-1 ของเรา) ที่เชื่อมต่อเข้ากับ PE ของ SP — **CE ไม่รู้จัก MPLS เลย มองเห็นแค่ IP Routing ปกติ** |

### มุมมองของลูกค้า (CE) ต่อ MPLS L3VPN: Virtual Full-Mesh

จุดที่ทำให้ MPLS L3VPN ทรงพลังคือ **ลูกค้าไม่ต้องรู้เรื่อง MPLS เลย** — จากมุมมองของ CE
Router ที่แต่ละ Site เห็นแค่ว่า **PE คือ Router เพื่อนบ้านตัวหนึ่ง** ที่ส่ง Route ของ Site อื่นๆ
มาให้ทั้งหมด เสมือนมี **Full-Mesh WAN** เชื่อมทุก Site เข้าด้วยกันโดยตรง ทั้งที่จริงๆ
Traffic เดินทางผ่าน MPLS Backbone ของ SP ที่ซับซ้อนภายใน

```
มุมมองที่ SP มองเห็นจริง (Physical + MPLS Label Switching):

   CE-HQ ── PE1 ═══ MPLS Core (P Routers, VRF, MP-BGP) ═══ PE2 ── CE-Branch1
                              │
                             PE3 ── CE-Branch2

มุมมองที่ CE (ลูกค้า) มองเห็น — เสมือน Full-Mesh Virtual Circuit:

              ┌──────────────┐
        ┌─────┤   CE-HQ       ├─────┐
        │     └──────────────┘     │
        ▼                          ▼
 ┌──────────────┐          ┌──────────────┐
 │  CE-Branch1  │◄────────►│  CE-Branch2  │
 └──────────────┘          └──────────────┘
   (CE-HQ เห็น Route ของ Branch1/Branch2 โดยตรงจาก PE เสมือนต่อสายตรงถึงกัน)
```

- SP ใช้ **VRF (Virtual Routing and Forwarding)** เพื่อแยก Routing Table ของลูกค้าแต่ละราย
  ออกจากกันบน PE Router ตัวเดียว (Multi-tenancy)
- SP ใช้ **MP-BGP (Multiprotocol BGP)** ระหว่าง PE-PE เพื่อแจก VPN Route ข้าม Site ของลูกค้า
  รายเดียวกัน พร้อม **Route-Target** เพื่อควบคุมว่า Site ไหนเห็น Route ของ Site ไหน
- รายละเอียดการคอนฟิก VRF/MP-BGP/Route-Target ทั้งหมดนี้เป็นงานฝั่ง **SP (P/PE Router)**
  ซึ่งเป็นเนื้อหาระดับ **CCIE Service Provider** จะสอนแบบเต็มรูปแบบใน **Part 86-87**

---

## Step 187 — คอนฟิก MPLS L3VPN จากมุมมองลูกค้า (CE)

แม้การคอนฟิกฝั่ง SP (PE/P Router) จะซับซ้อนและเลื่อนไปสอนที่ Part 86-87 แต่สิ่งที่ **CE Router
ของลูกค้าต้องทำ** นั้นเรียบง่ายมาก — เป็นแค่การตั้งค่า Routing Protocol (Static หรือ eBGP)
เพื่อแลก Route กับ PE เท่านั้น ในทางปฏิบัติ CE ไม่จำเป็นต้องคอนฟิก MPLS แม้แต่คำสั่งเดียว

### Scenario: WAN-EDGE-1 เป็น CE เชื่อมต่อกับ PE ของ MPLS Provider

สมมติ SP จัดสรร Link ระหว่าง WAN-EDGE-1 (CE) กับ PE ของเขาที่ subnet `192.0.2.0/30`
(PE = `192.0.2.1`, WAN-EDGE-1 = `192.0.2.2`) และ SP กำหนด Customer AS (สำหรับ eBGP)
เป็น **AS 65001** (Provider AS = 65000)

### วิธีที่ 1 — CE-PE ใช้ Static Route (แบบง่ายที่สุด สำหรับ Site เล็ก)

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/2
WAN-EDGE-1(config-if)# description ** MPLS L3VPN Access Link to Provider PE **
WAN-EDGE-1(config-if)# ip address 192.0.2.2 255.255.255.252
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
! Default Route ไปยัง PE ให้ SP ดูแลเรื่อง Routing ของทุก Site ให้เอง
WAN-EDGE-1(config)# ip route 0.0.0.0 0.0.0.0 192.0.2.1
```

### วิธีที่ 2 — CE-PE ใช้ eBGP (แนะนำสำหรับ Enterprise ที่มีหลาย Site/ต้องการ Redundancy)

```
WAN-EDGE-1(config)# router bgp 65001
WAN-EDGE-1(config-router)# bgp router-id 1.1.1.21
WAN-EDGE-1(config-router)# neighbor 192.0.2.1 remote-as 65000
WAN-EDGE-1(config-router)# neighbor 192.0.2.1 description ** eBGP to Provider PE (MPLS L3VPN) **
!
! Advertise Prefix ของ HQ เข้าไปใน MPLS VPN เพื่อให้ Site อื่นเห็น Route ของเรา
WAN-EDGE-1(config-router)# network 10.10.0.0 mask 255.255.0.0
WAN-EDGE-1(config-router)# exit
```

### Verify การรับ Route จาก Site อื่นผ่าน MPLS L3VPN (มุมมอง CE)

```
WAN-EDGE-1# show ip bgp summary
BGP router identifier 1.1.1.21, local AS number 65001
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.0.2.1       4 65000      45      44        8    0    0 00:15:20        3

WAN-EDGE-1# show ip route bgp
     172.20.0.0/16 [20/0] via 192.0.2.1, 00:15:20   <- Route ของ Branch Site อื่น (มาจากไซต์ที่อยู่ใน MPLS VPN เดียวกัน)
     172.21.0.0/16 [20/0] via 192.0.2.1, 00:15:20
```

- **ทั้ง 2 Route (`172.20.0.0/16`, `172.21.0.0/16`) มาจาก Site อื่นขององค์กรเดียวกัน** ที่
  เชื่อมต่อ MPLS L3VPN เดียวกันกับเรา — WAN-EDGE-1 (CE) มองเห็นเพียง "Next-Hop คือ PE"
  โดยไม่รู้เลยว่าภายใน Backbone ของ SP มี Label Switching, VRF, MP-BGP ทำงานอยู่กี่ชั้น
- นี่คือแนวคิดหลักของ Step 186 ที่พิสูจน์ในทางปฏิบัติ: **จากมุมมอง CE, MPLS L3VPN คือ Full-Mesh
  ธรรมดาที่ Routing Protocol เดียว (eBGP หรือ Static) ก็เพียงพอ**

---

## Step 188 — Site-to-Site VPN พื้นฐาน และ GRE Tunnel

### ทำไม VPN ผ่าน Internet ถึงเป็นตัวเลือกที่ถูกกว่า MPLS

จาก Step 181 เราเห็นว่า MPLS มีค่าใช้จ่ายสูงกว่า Internet มาก เพราะ SP ต้องรับประกัน SLA และ
ดูแล Backbone เฉพาะสำหรับลูกค้า ในขณะที่ **Internet Access เป็นบริการที่ทุกที่มีอยู่แล้วและ
ราคาต่ำกว่ามาก** — แนวทางแก้คือ **สร้าง Virtual Private Network (VPN) ทับบน Internet** เพื่อ
เชื่อมต่อ Site ต่างๆ โดยไม่ต้องเช่า Circuit เฉพาะจาก SP รายเดียว

| ประเด็น | MPLS | Internet VPN |
|---|---|---|
| ต้นทุนรายเดือน | สูง | ต่ำกว่ามาก (ใช้ Internet link ปกติ) |
| SLA | มี (จาก SP) | ไม่มี (best-effort บน Internet สาธารณะ) |
| ความปลอดภัยของข้อมูล | Private (แต่ไม่ได้เข้ารหัสโดย default) | ต้องเข้ารหัสเอง (IPsec) เพราะวิ่งบน Internet สาธารณะ |
| ความง่ายในการขยาย Site ใหม่ | ต้องรอ SP ติดตั้ง Circuit ใหม่ | เปิดใช้ Internet ที่ Site ใหม่แล้ว Config VPN ได้ทันที |

### GRE Tunnel — จุดเริ่มต้นของ Site-to-Site VPN (ยังไม่เข้ารหัส)

**GRE (Generic Routing Encapsulation, RFC 2784)** คือ Protocol ที่ห่อหุ้ม (Encapsulate)
Packet ใดๆ (รวมถึง Routing Protocol Multicast/Broadcast เช่น OSPF Hello) ไว้ภายใน IP Packet
อีกชั้น เพื่อสร้าง **Virtual Point-to-Point Tunnel** ระหว่างสอง Router ผ่าน Network กลาง
(เช่น Internet) — ข้อสำคัญคือ **GRE เพียงอย่างเดียวไม่มีการเข้ารหัสใดๆ** (plain tunnel)
เป็นแค่การ "ห่อหุ้ม" ไม่ใช่ "ปกปิดเนื้อหา"

```
                    Internet / Public Network
   WAN-EDGE-1 ═══════════════════════════════════════ BRANCH-RTR
   Physical: 203.0.113.2 (WAN-EDGE-1 ↔ ISP-RTR)      Physical: Public IP ของสาขา
                     │                                          │
                     └──────── GRE Tunnel (Tunnel0) ────────────┘
                          Virtual P2P Link ทับบน Internet
                     Tunnel source/destination = Physical IP
                     Tunnel IP = IP วง private ใหม่ (172.16.200.0/30)
```

### IP Addressing สำหรับ GRE Tunnel Scenario

| รายการ | ค่า |
|---|---|
| WAN-EDGE-1 Physical (Tunnel source) | 203.0.113.2 (Interface ที่ต่อ ISP-RTR ตาม [00-ip-address-plan.md](00-ip-address-plan.md)) |
| BRANCH-RTR Physical (Tunnel destination, จำลอง Public IP ของสาขา) | 198.51.100.2/30 (จำลอง — สาขาต่อ Internet ผ่าน ISP อื่น) |
| Tunnel0 Subnet (Virtual) | 172.16.200.0/30 |
| WAN-EDGE-1 Tunnel0 IP | 172.16.200.1 |
| BRANCH-RTR Tunnel0 IP | 172.16.200.2 |

### คอนฟิก GRE Tunnel บน WAN-EDGE-1

```
WAN-EDGE-1(config)# interface Tunnel0
WAN-EDGE-1(config-if)# description ** GRE Tunnel to BRANCH-RTR over Internet **
WAN-EDGE-1(config-if)# ip address 172.16.200.1 255.255.255.252
WAN-EDGE-1(config-if)# tunnel source GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# tunnel destination 198.51.100.2
WAN-EDGE-1(config-if)# tunnel mode gre ip
WAN-EDGE-1(config-if)# exit
```

### คอนฟิก GRE Tunnel บน BRANCH-RTR

```
BRANCH-RTR(config)# interface Tunnel0
BRANCH-RTR(config-if)# description ** GRE Tunnel to WAN-EDGE-1 (HQ) over Internet **
BRANCH-RTR(config-if)# ip address 172.16.200.2 255.255.255.252
BRANCH-RTR(config-if)# tunnel source GigabitEthernet0/0/1
BRANCH-RTR(config-if)# tunnel destination 203.0.113.2
BRANCH-RTR(config-if)# tunnel mode gre ip
BRANCH-RTR(config-if)# exit
```

> **หมายเหตุ**: `tunnel mode gre ip` เป็นค่า Default อยู่แล้วบน Cisco IOS เมื่อสร้าง Tunnel
> interface (สามารถละคำสั่งนี้ได้) แต่แนะนำให้พิมพ์ชัดเจนเพื่อ documentation ที่อ่านง่าย

### Verify GRE Tunnel

```
WAN-EDGE-1# show interface Tunnel0
Tunnel0 is up, line protocol is up
  Internet address is 172.16.200.1/30
  Tunnel source 203.0.113.2, destination 198.51.100.2
  Tunnel protocol/transport GRE/IP, key disabled, sequencing disabled

WAN-EDGE-1# ping 172.16.200.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.200.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/6/9 ms
```

### ข้อดีสำคัญที่สุดของ GRE: รองรับ Routing Protocol ผ่าน Tunnel ได้

เพราะ GRE ห่อหุ้ม Packet ได้ทุกประเภทรวม **Multicast/Broadcast** ทำให้ Routing Protocol ที่ต้องใช้
Multicast Hello เช่น **OSPF (224.0.0.5/224.0.0.6)** หรือ **EIGRP (224.0.0.10)** ที่เราเรียนไปแล้ว
ใน [Part 11-14](part-011-ospfv2-fundamentals.md) สามารถทำงานผ่าน GRE Tunnel ได้เหมือนเป็น Link
ปกติ — นี่คือจุดที่ IPsec แบบ site-to-site ธรรมดา (ไม่มี GRE) ทำไม่ได้ เพราะ IPsec Tunnel
มาตรฐาน (Tunnel Mode) ไม่รองรับ Multicast/Routing Protocol โดยตรง (จะอธิบายต่อใน Step 189)

---

## Step 189 — IPsec พื้นฐาน และแนวคิด GRE-over-IPsec

### ปัญหาของ GRE เพียงอย่างเดียว: ไม่มีการเข้ารหัส

จาก Step 188 GRE Tunnel ที่สร้างขึ้นนั้น **Traffic ทั้งหมดวิ่งเป็น Cleartext** ผ่าน Internet
สาธารณะ — ถ้ามีคนดักจับ Traffic ระหว่างทางจะเห็นข้อมูลทั้งหมดตรงๆ (เพียงแค่ถูก "ห่อหุ้ม" ด้วย
GRE Header ไม่ใช่ "เข้ารหัส") จึงต้องใช้ **IPsec** เพื่อเข้ารหัสข้อมูลจริงบน Tunnel นี้

### IPsec คืออะไร

**IPsec (IP Security)** เป็น Suite ของ Protocol สำหรับเข้ารหัสและยืนยันความถูกต้อง (Authenticate)
ของ IP Traffic ทำงานที่ Layer 3 ประกอบด้วย 2 Phase หลักที่ใช้ **ISAKMP/IKE (Internet Key
Exchange)** ในการเจรจา

```
                    IKE Phase 1                              IKE Phase 2
   ┌─────────────────────────────────┐      ┌─────────────────────────────────┐
   │  สร้าง ISAKMP SA (Secure,        │      │  สร้าง IPsec SA (ต่อ Traffic     │
   │  Encrypted Management Channel)   │ ───► │  จริงที่จะเข้ารหัส) โดยใช้        │
   │  ระหว่างสอง Peer ก่อน             │      │  Transform Set ที่ตกลงกันไว้      │
   │  (Authenticate ตัวตนของ Peer)     │      │  (Quick Mode)                    │
   └─────────────────────────────────┘      └─────────────────────────────────┘
```

| Phase | ชื่อ | หน้าที่ |
|---|---|---|
| **IKE Phase 1** | Main Mode (หรือ Aggressive Mode) | สร้าง **ISAKMP SA (Security Association)** — Peer ทั้งสองยืนยันตัวตนกัน (Pre-shared Key หรือ Certificate) และเจรจา Encryption/Hash Algorithm สำหรับช่องทางบริหาร (Management Channel) เอง |
| **IKE Phase 2** | Quick Mode | สร้าง **IPsec SA** จริงที่จะใช้เข้ารหัส Data Traffic โดยอ้างอิง **Transform Set** (ชุด Algorithm ที่จะใช้เข้ารหัส/Hash Traffic จริง เช่น AES-256 + SHA-256) |

### แนวคิด Transform Set (ภาพรวม — รายละเอียดคอนฟิกจริงใน Part 51-53)

```
crypto ipsec transform-set MY-TSET esp-aes 256 esp-sha256-hmac
```

- **ESP (Encapsulating Security Payload)**: ให้ทั้ง Encryption + Authentication (ใช้บ่อยที่สุด)
- **AH (Authentication Header)**: ให้แค่ Authentication ไม่เข้ารหัส (ใช้น้อยกว่ามาก)
- Transform Set ต้อง **ตรงกันทั้งสองฝั่ง Peer** เหมือนหลักการ Match parameter ของ PPP LCP

### ทำไมต้องรวม GRE + IPsec (GRE-over-IPsec)

IPsec แบบ **Tunnel Mode ล้วนๆ** (ไม่มี GRE) เข้ารหัส IP Packet ได้ดี แต่ **ไม่รองรับ
Multicast/Broadcast** จึงใช้กับ Routing Protocol แบบ Dynamic (OSPF/EIGRP) ที่ต้องพึ่ง
Multicast Hello ไม่ได้โดยตรง — วิธีแก้คือ **GRE-over-IPsec**:

```
┌─────────────────────────────────────────────────────────────┐
│  1. GRE ห่อหุ้ม Packet (รวม Routing Protocol Multicast) ก่อน  │  <- ให้ Routing Protocol ทำงานได้
├─────────────────────────────────────────────────────────────┤
│  2. IPsec เข้ารหัส GRE Packet ทั้งก้อนอีกชั้น                  │  <- ให้ Traffic ปลอดภัย
└─────────────────────────────────────────────────────────────┘

           [ Original IP Packet ]
                    │  GRE encapsulate
                    ▼
        [ GRE Header [ Original IP Packet ] ]
                    │  IPsec encrypt (ESP)
                    ▼
   [ New IP Header [ ESP [ GRE [ Original IP Packet ] ] ] ]
```

**ผลลัพธ์**: ได้ทั้ง **Dynamic Routing Protocol Support** (จาก GRE) และ **Encryption** (จาก
IPsec) ในคราวเดียว — นี่คือรูปแบบพื้นฐานที่นำไปสู่ **DMVPN (Dynamic Multipoint VPN)** ซึ่งเป็น
วิวัฒนาการของ GRE-over-IPsec ที่รองรับ Hub-and-Spoke ขนาดใหญ่แบบ Dynamic (ไม่ต้องคอนฟิก Tunnel
คู่ต่อคู่ทีละ Site) — **จะสอนคอนฟิกเต็มรูปแบบทั้ง IPsec Phase 1/2, Crypto Map/IKEv2 Profile,
และ DMVPN Phase 1-3 ใน Part 51-53**

---

## Step 190 — Lab เต็มรูปแบบ: PPP+CHAP over Leased Line + GRE Tunnel + OSPF over Tunnel

Lab นี้รวมทุกแนวคิดของ Part นี้เข้าด้วยกัน โดยจำลองสถานการณ์จริง: **BRANCH-RTR เชื่อมต่อกับ
WAN-EDGE-1 (HQ) ด้วย 2 เส้นทางพร้อมกัน**:

1. **Leased Line หลัก** ผ่าน PPP + CHAP Authentication (Serial0/0/0, subnet `172.16.199.0/30`
   จาก Step 183-184)
2. **GRE Tunnel สำรอง** ผ่าน Internet (Tunnel0, subnet `172.16.200.0/30` จาก Step 188) —
   รัน **OSPF Area 0** ผ่าน Tunnel เพื่อแลก Route กับ HQ แบบ Dynamic (เชื่อมโยงกับ OSPF
   ที่เรียนไปแล้วใน [Part 11-12](part-011-ospfv2-fundamentals.md))

```
                              ┌──────────────────┐
                              │     ISP-RTR       │
                              └────────┬─────────┘
                       203.0.113.0/30  │
                              ┌────────┴─────────┐
                              │    WAN-EDGE-1      │  Lo0: 1.1.1.21/32
                              │      (HQ)          │
                              └───┬─────────┬─────┘
                    Se0/0/0 (PPP) │         │ Gi0/0/0 (tunnel source)
              172.16.199.1/30    │         │  203.0.113.2
                                  │         │
                                  │    ╔════╧═══════════════════╗
                                  │    ║  GRE Tunnel0 (over      ║
                                  │    ║  Internet, backup path) ║
                                  │    ║  172.16.200.0/30        ║
                                  │    ╚════╤═══════════════════╝
                    Se0/0/0 (PPP) │         │ Gi0/0/1 (tunnel destination)
              172.16.199.2/30    │         │  198.51.100.2
                              ┌───┴─────────┴─────┐
                              │    BRANCH-RTR       │  Lo0: 172.20.0.1/32
                              │  (สำนักงานสาขา)      │
                              └────────────────────┘
```

### สรุป IP Addressing ทั้งหมดสำหรับ Lab นี้

| Device | Interface | IP Address | หมายเหตุ |
|---|---|---|---|
| WAN-EDGE-1 | Serial0/0/0 | 172.16.199.1/30 | PPP + CHAP ไปยัง BRANCH-RTR |
| WAN-EDGE-1 | GigabitEthernet0/0/0 | 203.0.113.2/30 | Physical ไปยัง ISP-RTR (ตาม IP Plan หลัก) |
| WAN-EDGE-1 | Tunnel0 | 172.16.200.1/30 | GRE Tunnel source=Gi0/0/0, destination=198.51.100.2 |
| WAN-EDGE-1 | Loopback0 | 1.1.1.21/32 | Router-ID (ตาม IP Plan หลัก) |
| BRANCH-RTR | Serial0/0/0 | 172.16.199.2/30 | PPP + CHAP ไปยัง WAN-EDGE-1 |
| BRANCH-RTR | GigabitEthernet0/0/1 | 198.51.100.2/30 | Physical ไปยัง Internet (จำลอง Public IP สาขา) |
| BRANCH-RTR | Tunnel0 | 172.16.200.2/30 | GRE Tunnel source=Gi0/0/1, destination=203.0.113.2 |
| BRANCH-RTR | Loopback0 | 172.20.0.1/32 | จำลอง LAN ของสาขา (advertise เข้า OSPF) |

### Running-Config เต็มรูปแบบ — WAN-EDGE-1

```
hostname WAN-EDGE-1
!
username BRANCH-RTR password Cisco@WAN123
!
interface Loopback0
 description ** Router-ID / Management **
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** Link to ISP-RTR (Internet path for GRE) **
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface Serial0/0/0
 description ** Leased Line (Primary Path) to BRANCH-RTR **
 ip address 172.16.199.1 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 no shutdown
!
interface Tunnel0
 description ** GRE Tunnel (Backup Path over Internet) to BRANCH-RTR **
 ip address 172.16.200.1 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 198.51.100.2
 tunnel mode gre ip
!
router ospf 1
 router-id 1.1.1.21
 network 172.16.200.0 0.0.0.3 area 0
 passive-interface default
 no passive-interface Tunnel0
!
! Leased Line ใช้ Static Route แบบ Floating (Administrative Distance สูงกว่า OSPF)
! เป็น Backup ให้กันในทิศทางตรงข้าม ในสถานการณ์จริงอาจใช้ OSPF บน Serial ด้วยเช่นกัน
ip route 172.20.0.1 255.255.255.255 172.16.199.2 100
!
end
```

### Running-Config เต็มรูปแบบ — BRANCH-RTR

```
hostname BRANCH-RTR
!
username WAN-EDGE-1 password Cisco@WAN123
!
interface Loopback0
 description ** จำลอง LAN ของสำนักงานสาขา **
 ip address 172.20.0.1 255.255.255.255
!
interface GigabitEthernet0/0/1
 description ** Link to Internet (Public IP จำลอง) **
 ip address 198.51.100.2 255.255.255.252
 no shutdown
!
interface Serial0/0/0
 description ** Leased Line (Primary Path) to WAN-EDGE-1 (HQ) **
 ip address 172.16.199.2 255.255.255.252
 encapsulation ppp
 ppp authentication chap
 no shutdown
!
interface Tunnel0
 description ** GRE Tunnel (Backup Path over Internet) to WAN-EDGE-1 (HQ) **
 ip address 172.16.200.2 255.255.255.252
 tunnel source GigabitEthernet0/0/1
 tunnel destination 203.0.113.2
 tunnel mode gre ip
!
router ospf 1
 router-id 172.20.0.1
 network 172.16.200.0 0.0.0.3 area 0
 network 172.20.0.1 0.0.0.0 area 0
 passive-interface default
 no passive-interface Tunnel0
!
ip route 1.1.1.21 255.255.255.255 172.16.199.1 100
!
end
```

### Verification ทั้งหมด

**1) ตรวจสอบ Leased Line (PPP + CHAP):**

```
WAN-EDGE-1# show interfaces Serial0/0/0
Serial0/0/0 is up, line protocol is up
  Description: ** Leased Line (Primary Path) to BRANCH-RTR **
  Internet address is 172.16.199.1/30
  MTU 1500 bytes, BW 1544 Kbit/sec, DLY 20000 usec,
  Encapsulation PPP, LCP Open, multilink Closed
  Open: IPCP, CDPCP
  Last input 00:00:03, output 00:00:03, output hang never
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec

WAN-EDGE-1# show ppp all
PPP Serial Interface Serial0/0/0
Session id [1]
    Remote host: BRANCH-RTR
    LCP: Open
    IPCP: Open
    CDPCP: Open
    Authentication: CHAP, remote passed, local passed
```

**2) ตรวจสอบ GRE Tunnel:**

```
WAN-EDGE-1# show interfaces Tunnel0
Tunnel0 is up, line protocol is up
  Description: ** GRE Tunnel (Backup Path over Internet) to BRANCH-RTR **
  Internet address is 172.16.200.1/30
  MTU 1476 bytes, BW 100 Kbit/sec, DLY 50000 usec,
  Tunnel source 203.0.113.2, destination 198.51.100.2
  Tunnel protocol/transport GRE/IP, key disabled, sequencing disabled
  Tunnel TTL 255
```

> สังเกตว่า **MTU ของ Tunnel0 เหลือแค่ 1476 byte** (ต่ำกว่า Ethernet ปกติ 1500 byte) เพราะ
> GRE Header กิน Overhead ไปประมาณ 24 byte — เป็นจุดที่ต้อง troubleshoot Fragmentation บ่อย
> ในงานจริง ถ้า Application ส่ง Packet ขนาดใหญ่เกิน MTU ของ Tunnel

**3) ตรวจสอบ OSPF Neighbor ผ่าน Tunnel:**

```
WAN-EDGE-1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
172.20.0.1        1   FULL/  -        00:00:38    172.16.200.2    Tunnel0
```

- **State = FULL** ยืนยันว่า OSPF Adjacency ผ่าน GRE Tunnel สำเร็จสมบูรณ์ — Interface
  ที่เห็นคือ **Tunnel0** (Virtual interface) ไม่ใช่ Physical interface เลย ตรงตามแนวคิด
  Step 188 ที่ GRE ทำให้ Routing Protocol มองเห็น Tunnel เป็น Link ปกติ
- Neighbor ID `172.20.0.1` มาจาก Router-ID ของ BRANCH-RTR (ใช้ Loopback0 ตาม convention
  ที่วางไว้ตั้งแต่ [Part 2 Step 20](part-002-ios-cli-basics.md))

**4) ตรวจสอบ Routing Table รวม (ทั้ง PPP Static และ OSPF over GRE):**

```
WAN-EDGE-1# show ip route | begin Gateway
Gateway of last resort is not set

      1.0.0.0/32 is subnetted, 1 subnets
C        1.1.1.21 is directly connected, Loopback0
      172.16.0.0/16 is variably subnetted, 4 subnets, 2 masks
C        172.16.199.0/30 is directly connected, Serial0/0/0
L        172.16.199.1/32 is directly connected, Serial0/0/0
C        172.16.200.0/30 is directly connected, Tunnel0
L        172.16.200.1/32 is directly connected, Tunnel0
      172.20.0.0/16 is variably subnetted, 1 subnets, 1 mask
O        172.20.0.1/32 [110/1001] via 172.16.200.2, 00:12:44, Tunnel0
```

- Route ไปยัง `172.20.0.1/32` (Loopback0 ของ BRANCH-RTR) มาจาก **O (OSPF)** ผ่าน Tunnel0 —
  หมายเลข `[110/1001]` คือ **[Administrative Distance/Metric]**: 110 = AD ของ OSPF, 1001 =
  Cost (สูงเพราะ Bandwidth ของ Tunnel ต่ำตาม `BW 100 Kbit/sec` ที่เห็นใน `show interfaces
  Tunnel0` — ค่า Cost ที่สูงนี้ทำให้ในสถานการณ์จริงที่มี Path อื่น Cost ต่ำกว่า OSPF จะเลือก
  Path นั้นเป็นหลักและ Tunnel นี้เป็น Backup โดยอัตโนมัติ)
- Static Route ที่ตั้งไว้บน Serial ผ่าน PPP (`ip route 172.20.0.1 ... 172.16.199.2 100`)
  มี Administrative Distance = 100 ซึ่ง **ดีกว่า** OSPF (110) — แต่ในตัวอย่างนี้ไม่ปรากฏใน
  Routing Table เพราะ Static Route ปลายทางเดียวกันถูกกำหนดผ่าน Interface เดียวเจาะจง
  (`172.16.199.2`) และไม่มี Recursive Lookup ปัญหา — ในการออกแบบจริงควรเลือกให้ AD ของ Static
  Route (Leased Line) **ต่ำกว่า** OSPF (GRE/Internet) เสมอ เพื่อให้ Leased Line เป็น Primary
  Path และ GRE Tunnel เป็น Backup ตามที่ตั้งใจออกแบบไว้ในหัวข้อนี้

**5) ทดสอบ Failover: ปิด Leased Line แล้วดูว่า Traffic เปลี่ยนไปใช้ GRE Tunnel**

```
WAN-EDGE-1(config)# interface Serial0/0/0
WAN-EDGE-1(config-if)# shutdown
WAN-EDGE-1(config-if)# end

WAN-EDGE-1# show ip route 172.20.0.1
Routing entry for 172.20.0.1/32
  Known via "ospf 1", distance 110, metric 1001, type intra area
  Last update from 172.16.200.2 on Tunnel0, 00:00:02 ago
  Routing Descriptor Blocks:
  * 172.16.200.2, from 172.20.0.1, 00:00:02 ago, via Tunnel0
      Route metric is 1001, traffic share count is 1

WAN-EDGE-1# ping 172.20.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.20.0.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 8/11/15 ms
```

Traffic ยังคงไปถึงปลายทางได้สำเร็จผ่าน GRE Tunnel แม้ Leased Line (PPP) ถูกปิดไปแล้ว —
พิสูจน์ว่า **Design แบบ Dual-Path (Leased Line + Internet VPN) ให้ Redundancy ได้จริง**
ซึ่งเป็นเหตุผลสำคัญที่สุดที่องค์กรจริงมักไม่พึ่งพา WAN Technology เพียงชนิดเดียว

---

## แบบฝึกหัดทวนความเข้าใจ Part 19

1. PPP แบ่งการทำงานเป็น 2 Phase หลักคืออะไร และ Phase ใดที่ทำหน้าที่เจรจา IP Address/Layer 3?
2. เพราะเหตุใด CHAP จึงปลอดภัยกว่า PAP โดยละเอียด (อธิบายกลไก 3-way handshake)?
3. จากมุมมองของ CE Router ลูกค้าเห็น MPLS L3VPN เป็นอย่างไร และ CE ต้องคอนฟิก MPLS Label
   ด้วยตนเองหรือไม่?
4. เพราะเหตุใด GRE เพียงอย่างเดียวจึงไม่เพียงพอสำหรับ Site-to-Site VPN ที่ต้องการความปลอดภัย
   และทำไมต้องรวมกับ IPsec (GRE-over-IPsec)?
5. ในการออกแบบ Dual-Path WAN (Leased Line + GRE Tunnel ผ่าน Internet) เราควรตั้งค่า
   Administrative Distance อย่างไรเพื่อให้ Leased Line เป็น Path หลักเสมอ?

**เฉลย:**
1. **LCP (Link Control Protocol)** และ **NCP (Network Control Protocol)** — NCP (โดยเฉพาะ
   IPCP) ทำหน้าที่เจรจา Layer 3/IP Address, ส่วน LCP เจรจาคุณสมบัติของ Link เอง (MTU,
   Authentication, Magic Number) ก่อน NCP จะทำงานได้
2. CHAP ใช้ **3-way handshake (Challenge → Response → Success/Failure)** และส่งแค่ **MD5
   Hash ของ Password** ไม่ส่ง Password ตรงๆ ผ่าน Link (ต่างจาก PAP ที่ส่ง Password แบบ
   cleartext) นอกจากนี้ CHAP ยังใช้ Random Challenge ที่เปลี่ยนทุกครั้งทำให้ **ป้องกัน Replay
   Attack** ได้ และสามารถทำ Re-authentication ระหว่าง Session ได้ด้วย
3. CE เห็น MPLS L3VPN เป็นเสมือน **Virtual Full-Mesh** — มองเห็น Route ของทุก Site ผ่าน PE
   เหมือนต่อสายตรงถึงกัน โดย **ไม่ต้องคอนฟิก MPLS Label ใดๆ เลย** CE แค่ทำ Routing ปกติ
   (Static Route หรือ eBGP) กับ PE เท่านั้น ส่วน Label Switching/VRF/MP-BGP ทั้งหมดเป็นงาน
   ฝั่ง SP (P/PE Router)
4. GRE เป็นเพียงการ **ห่อหุ้ม (Encapsulate)** Packet ไม่ใช่การเข้ารหัส — Traffic ที่วิ่งผ่าน
   GRE Tunnel บน Internet สาธารณะยังเป็น Cleartext ถ้าถูกดักจับจะเห็นข้อมูลตรงๆ ต้องรวมกับ
   IPsec เพื่อเข้ารหัส (Confidentiality) — และต้องใช้ GRE ควบคู่ไปด้วยเพราะ IPsec เพียงอย่าง
   เดียว (Tunnel Mode) ไม่รองรับ Multicast/Broadcast ที่ Routing Protocol อย่าง OSPF/EIGRP
   ต้องใช้
5. ตั้งค่า **Administrative Distance ของ Static Route บน Leased Line ให้ต่ำกว่า AD ของ OSPF
   ที่รันอยู่บน GRE Tunnel** (ตัวอย่างในหลักสูตรนี้ใช้ Static AD=100 ซึ่งต่ำกว่า OSPF AD=110
   อยู่แล้ว) ทำให้ Router เลือก Path ที่มี AD ต่ำกว่าเป็นหลักเสมอเมื่อทั้งสอง Path พร้อมใช้งาน
   และสลับไปใช้ GRE Tunnel โดยอัตโนมัติเมื่อ Leased Line ล่ม (ตามที่พิสูจน์ใน Step 190 ข้อ 5)

---

## สรุป Part 19

Part นี้ขยายขอบเขตของหลักสูตรจาก Campus Network ไปสู่ **WAN (Wide Area Network)** อย่างเต็ม
รูปแบบ — เริ่มจากภาพรวมเทคโนโลยี WAN ทั้ง 5 ประเภท, ลงลึกที่ **PPP** (LCP/NCP, PAP/CHAP,
Multilink), แนวคิด **MPLS L3VPN** จากมุมมองลูกค้าที่มองเห็นเป็น Virtual Full-Mesh, และปิดท้าย
ด้วยพื้นฐาน **Site-to-Site VPN** ผ่าน GRE Tunnel พร้อมแนวคิด IPsec/GRE-over-IPsec ✅ Lab
สุดท้ายรวมทุกอย่างเข้าด้วยกันเป็น Dual-Path WAN Design ระหว่าง **WAN-EDGE-1 (HQ)** และ
**BRANCH-RTR (สาขาใหม่)** ที่ใช้งานได้จริงและมี Redundancy อัตโนมัติ

หัวข้อที่ถูก "ค้างไว้" อย่างตั้งใจใน Part นี้ (จะกลับมาสอนเต็มรูปแบบในระดับที่สูงขึ้น):
- **SD-WAN** → Part 39 (ภาพรวม), Part 76 (เชิงลึก)
- **IPsec/DMVPN แบบเต็มรูปแบบ** → Part 51-53
- **MPLS L3VPN ระดับ Provider (VRF, MP-BGP, Route-Target)** → Part 86-87

**ไปต่อ:** [Part 20 — Wireless Fundamentals →](part-020-wireless-fundamentals.md)
