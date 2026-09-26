# Part 16 — DHCP & DNS Services
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 151–160 จาก 1000**

> ต่อจาก [Part 15 — NAT/PAT](part-015-nat-pat.md) ที่สอนการแปลง Private IP เป็น Public IP
> เพื่อออก Internet แล้ว Part นี้จะพาไปดูอีกด้านของ IP Address Management ที่วิศวกร Enterprise
> ต้องดูแลทุกวัน คือ **การแจกจ่าย IP Address ให้ Host อัตโนมัติ (DHCP)** และ **การแปลงชื่อ
> เป็น IP Address (DNS)** — เราจะแนะนำอุปกรณ์ใหม่เข้าสู่ Lab Topology คือ **DHCP-DNS-SRV**
> (10.10.30.20 ใน VLAN 30 – SERVERS) ที่ทำหน้าที่เป็น DHCP + DNS Server กลางของทั้งองค์กร
> และเรียนรู้ว่า **DIST-SW1–4** ต้องทำหน้าที่ **DHCP Relay Agent** อย่างไรเพื่อส่ง Request ของ
> PC ใน VLAN 10 (SALES) และ VLAN 40 (WIFI) ข้าม Subnet ไปหา Server ตัวนี้ให้ได้ — ปิดท้ายด้วย
> Lab เต็มรูปแบบที่ PC1 ได้รับ IP Address จริงผ่าน DHCP Relay ทั่ว Topology

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 151 | DHCP Fundamentals — DORA Process, Lease และปัญหา Broadcast ข้าม Subnet |
| 152 | Cisco IOS เป็น DHCP Server (Local Pool) — ตัวอย่าง VLAN 20 VOICE |
| 153 | DHCP Relay (`ip helper-address`) — ข้าม Subnet ด้วย Relay Agent |
| 154 | DHCP Excluded Address — กันที่อยู่ Static ออกจาก Pool |
| 155 | DHCP Snooping เบื้องต้น — Trusted/Untrusted Port |
| 156 | DNS Fundamentals — Recursive/Iterative Query และ Record Types |
| 157 | Cisco IOS เป็น DNS Server/Proxy — `ip dns server` และ `ip host` |
| 158 | DHCPv6 — SLAAC vs Stateful DHCPv6 (ต่อจาก Part 8) |
| 159 | Verification & Troubleshooting DHCP/DNS |
| 160 | Lab เต็มรูปแบบ: DHCP Relay ทั่ว Topology + PC1 รับ Lease จริง |

---

## Step 151 — DHCP Fundamentals: DORA Process, Lease และปัญหา Broadcast ข้าม Subnet

### 151.1 ทำไมต้องมี DHCP

ก่อนหน้านี้ทุก Interface ในหลักสูตรนี้ (SVI, Routed Port, Loopback) เราตั้ง IP Address แบบ
**Static** ด้วยมือทั้งหมด ซึ่งใช้ได้ดีกับอุปกรณ์ Network (Switch/Router) ที่มีจำนวนน้อยและ
เปลี่ยนแปลงไม่บ่อย แต่สำหรับ **End-user Host** (PC, Laptop, มือถือ, Printer, IP Phone) ที่มี
จำนวนหลักร้อยหลักพันเครื่องในองค์กรจริง การตั้ง IP ด้วยมือทีละเครื่องเป็นไปไม่ได้ในทางปฏิบัติ —
**DHCP (Dynamic Host Configuration Protocol)** คือโปรโตคอลที่แก้ปัญหานี้ โดยแจก IP Address,
Subnet Mask, Default Gateway, DNS Server และ Option อื่นๆ ให้ Host โดยอัตโนมัติทันทีที่เสียบสาย
หรือเชื่อมต่อ Wi-Fi

DHCP ทำงานอยู่บน **UDP** เสมอ (ไม่ใช่ TCP เพราะตอนขอ IP ยังไม่มี IP จะเปิด TCP Session ไม่ได้)
โดยใช้ 2 Port มาตรฐาน:

| Port | ใช้โดย |
|---|---|
| UDP 67 | DHCP Server (bootps) |
| UDP 68 | DHCP Client (bootpc) |

### 151.2 DORA Process — 4 ขั้นตอนหลักของ DHCP

กระบวนการขอ IP Address ครั้งแรก (Initial Lease) มี 4 ขั้นตอน จำง่ายๆ ด้วยตัวย่อ **D-O-R-A**:

```
Client (ยังไม่มี IP)                                    DHCP Server
     │                                                        │
     │  1. DHCPDISCOVER                                       │
     │  src 0.0.0.0:68 → dst 255.255.255.255:67 (L2+L3 broadcast)
     │───────────────────────────────────────────────────────►│
     │           "มีใครเป็น DHCP Server บ้าง ขอ IP ที"          │
     │                                                        │
     │  2. DHCPOFFER                                          │
     │◄───────────────────────────────────────────────────────│
     │      Server เสนอ IP + Lease Time + Option ทั้งหมด        │
     │      (ถ้ามี DHCP Server หลายตัว Client จะได้ Offer       │
     │       หลายใบ แล้วเลือกรับใบแรกที่มาถึง)                  │
     │                                                        │
     │  3. DHCPREQUEST                                        │
     │───────────────────────────────────────────────────────►│
     │      Client ยืนยันว่า "ขอรับ Offer นี้" — ยังส่งเป็น       │
     │      Broadcast เพื่อให้ DHCP Server ตัวอื่นที่เสนอ Offer   │
     │      ไปด้วยรู้ว่า Client เลือกตัวไหนแล้ว จะได้คืน IP กลับพูล │
     │                                                        │
     │  4. DHCPACK                                            │
     │◄───────────────────────────────────────────────────────│
     │      Server ยืนยันให้ IP นั้นอย่างเป็นทางการ              │
     │      Lease Timer เริ่มนับจากขณะนี้                       │
```

> **จำง่าย**: D-O-R-A = **D**iscover → **O**ffer → **R**equest → **A**cknowledge
> Client เริ่มก่อนเสมอ (Discover) และ Server เป็นฝ่ายตอบเสมอ (Offer, Ack) สลับกันไปมา

### 151.3 DHCP Message Type อื่นๆ ที่ต้องรู้

| Message | ผู้ส่ง | ใช้เมื่อ |
|---|---|---|
| DHCPDISCOVER | Client | ขอ IP ครั้งแรก (broadcast) |
| DHCPOFFER | Server | เสนอ IP ให้ Client |
| DHCPREQUEST | Client | ยืนยันรับ Offer หรือขอ Renew Lease เดิม |
| DHCPACK | Server | ยืนยันให้ IP (ทั้งครั้งแรกและ Renew) |
| DHCPNAK | Server | ปฏิเสธ (เช่น Client ขอ Renew IP ที่ไม่ตรงกับ Subnet ปัจจุบันแล้ว) |
| DHCPDECLINE | Client | Client เจอว่า IP ที่ได้มาซ้ำกับเครื่องอื่น (ตรวจด้วย Gratuitous ARP) จึงปฏิเสธ |
| DHCPRELEASE | Client | Client คืน IP ก่อนหมด Lease (เช่น ปิดเครื่อง/ถอดสาย LAN) |
| DHCPINFORM | Client | Client มี IP Static อยู่แล้ว แต่ขอ Option อื่น (DNS, Domain) จาก DHCP Server |

### 151.4 แนวคิด DHCP Lease และการ Renew

**Lease** คือ "ระยะเวลาเช่า" ที่ Client มีสิทธิ์ใช้ IP Address นั้น — DHCP **ไม่ได้ให้ IP แบบถาวร**
เพื่อให้ Pool IP Address ถูกนำกลับมาใช้ใหม่ได้เมื่อ Host เลิกใช้งานไปแล้ว (ปิดเครื่อง, ย้ายเครือข่าย)

```
Lease Timeline (ตัวอย่าง Lease = 8 วัน)
0% ─────────────────── 50% (T1) ────────── 87.5% (T2) ────────── 100% (Expire)
│  Lease เริ่ม              │ พยายาม Renew      │ พยายาม Renew          │ IP หมดอายุ
│  ใช้งานปกติ                │ กับ Server เดิม   │ กับ Server ใดก็ได้      │ ต้องทำ DORA ใหม่
                            (Unicast DHCPREQUEST) (Broadcast DHCPREQUEST)
```

- **T1 (50% ของ Lease)**: Client พยายามยิง `DHCPREQUEST` แบบ **Unicast** ไปที่ DHCP Server
  ตัวเดิมที่ให้ IP นี้มา เพื่อขอ Renew (ได้ IP เดิม ต่อ Lease ใหม่)
- **T2 (87.5% ของ Lease)**: ถ้า Server เดิมไม่ตอบ Client จะเปลี่ยนไปยิง `DHCPREQUEST` แบบ
  **Broadcast** เพื่อขอ Renew กับ DHCP Server **ตัวใดก็ได้** ที่ตอบสนอง
- **100% (Expire)**: ถ้ายังไม่มีใครตอบเลย IP จะถูกปล่อยคืน (Release) และ Client ต้องเริ่ม DORA
  process ใหม่ทั้งหมดเหมือนขอ IP ครั้งแรก

> Lease ที่สั้นเหมาะกับเครือข่ายที่ Host เปลี่ยนบ่อย (เช่น Guest Wi-Fi) เพราะ IP หมุนกลับมาใช้ใหม่
> ได้เร็ว ส่วน Lease ที่ยาวเหมาะกับ Host ที่อยู่ประจำ (เช่น Desktop PC ในสำนักงาน, IP Phone)
> เพราะลด Traffic การ Renew ที่ไม่จำเป็น

### 151.5 ปัญหาสำคัญ: DHCP Broadcast ไม่ข้าม Subnet/VLAN

นี่คือหัวใจของ Part นี้ — `DHCPDISCOVER` เป็น **L3 Broadcast** (`255.255.255.255`) และ **L2
Broadcast** (`FFFF.FFFF.FFFF`) เสมอ เพราะ Client ยังไม่มี IP และไม่รู้จัก DHCP Server ล่วงหน้า
เลยแม้แต่ IP เดียว

ตามกฎพื้นฐานของ Router (Part 9-10) — **Router (หรือ SVI บน Multilayer Switch) จะไม่ Forward
Broadcast Traffic ข้าม Interface/VLAN ให้เองเด็ดขาด** (นี่คือขอบเขตของ Broadcast Domain ตาม
นิยาม) ดังนั้นถ้า DHCP Server ตั้งอยู่คนละ VLAN กับ Client (ซึ่งเป็นการออกแบบที่พบบ่อยที่สุดใน
Enterprise จริง — DHCP Server มักอยู่รวมกับ Server อื่นๆ ใน VLAN SERVERS) **Broadcast
Request จาก Client จะไปไม่ถึง Server เลย**

```
VLAN 10 (SALES)              ┌─────────────┐              VLAN 30 (SERVERS)
10.10.10.0/24                │  DIST-SW1   │              10.10.30.0/24
┌──────┐                     │  SVI Vlan10 │              ┌──────────────┐
│ PC1  │──DHCPDISCOVER──────►│  = .2       │   ❌ หยุดที่นี่  │ DHCP-DNS-SRV │
└──────┘   (broadcast)       │  ip routing │   ไม่ forward  │  10.10.30.20 │
                             └─────────────┘   broadcast    └──────────────┘
                                                ข้าม VLAN ให้
```

**ผลลัพธ์**: PC1 ยิง `DHCPDISCOVER` ออกไปแล้วไม่มีใครตอบ (Server อยู่ VLAN อื่น ไม่ได้ยิน
Broadcast นี้เลย) → PC1 จะ Fallback เป็น **APIPA** (Automatic Private IP Addressing,
`169.254.x.x/16` บน Windows) หรือค้างที่ "Obtaining IP Address..." แล้วสุดท้าย connectivity
ล้มเหลวทั้งหมด

Cisco แก้ปัญหานี้ด้วย 2 วิธีหลัก ซึ่งเป็นเนื้อหาของ Step ถัดไปทั้งหมด:

| วิธีแก้ | คำอธิบายสั้น | ใช้ใน Step |
|---|---|---|
| **DHCP Server ตัวเดียวกับ Client (Local Pool)** | ตั้ง Cisco IOS เองเป็น DHCP Server ให้ VLAN นั้นตรงๆ ไม่มีปัญหาข้าม Subnet เพราะ Server "อยู่ในตัว Gateway" ของ VLAN นั้นเลย | Step 152 |
| **DHCP Relay Agent** | ให้ Gateway (SVI) ของ VLAN นั้น "แปลง" Broadcast เป็น Unicast แล้วส่งข้าม VLAN ไปหา Server จริงให้ | Step 153 |

---

## Step 152 — Cisco IOS เป็น DHCP Server (Local Pool) — ตัวอย่าง VLAN 20 VOICE

### 152.1 เมื่อไหร่ควรใช้ Cisco IOS เป็น DHCP Server เอง

Cisco IOS/IOS-XE มีความสามารถทำหน้าที่ DHCP Server ในตัวได้เลยโดยไม่ต้องมี Server แยก —
เหมาะกับ **Branch Office ขนาดเล็ก** หรือ **VLAN เฉพาะทางที่มี Scope เล็กและไม่อยากพึ่งพา
Server กลาง** เช่น VLAN เสียง (VOICE) ที่มักต้องการความเรียบง่ายและ Availability สูง (ถ้า
Server กลางล่ม โทรศัพท์ในตึกจะยังขอ IP ได้จาก Switch ตัวเองโดยตรง)

> **ในหลักสูตรนี้**: Lab หลักของ VLAN 10 (SALES) และ VLAN 40 (WIFI) จะใช้ **DHCP-DNS-SRV
> กลาง** ผ่าน DHCP Relay (Step 153) เป็นสถาปัตยกรรมหลัก — ส่วน Step นี้ใช้ **VLAN 20 (VOICE)**
> เป็นตัวอย่างประกอบการสอนแยกต่างหาก เพื่อให้เห็นวิธีตั้ง Cisco IOS DHCP Server แบบ Local Pool
> ด้วยตัวเอง ซึ่งเป็นความรู้ที่ข้อสอบ CCNA ถามตรงๆ บ่อยมาก

### 152.2 คำสั่งหลักของ IOS DHCP Server

| คำสั่ง | ใช้ทำอะไร |
|---|---|
| `ip dhcp pool <name>` | สร้าง DHCP Pool ตั้งชื่อเอง เข้าสู่ `(config-dhcp)#` mode |
| `network <subnet> <mask>` | กำหนด Subnet ที่ Pool นี้จะแจก IP ให้ |
| `default-router <ip> [ip2...]` | Default Gateway ที่จะแจกให้ Client (ใส่ได้หลายตัวเพื่อ Redundancy) |
| `dns-server <ip> [ip2...]` | DNS Server ที่จะแจกให้ Client |
| `domain-name <name>` | Domain name ที่จะแจกให้ Client |
| `lease <days> [hours] [minutes]` / `lease infinite` | ระยะเวลา Lease (Default ถ้าไม่ตั้งคือ 1 วัน) |
| `netbios-name-server <ip>` | WINS Server (ระบบเก่า พบใน Windows Legacy) |
| `option <code> ...` | Option แบบกำหนดเอง เช่น `option 150 ip <tftp-ip>` สำหรับ Cisco IP Phone หา TFTP/CUCM |

### 152.3 Worked Example — DIST-SW2 เป็น DHCP Server ให้ VLAN 20 (VOICE)

DIST-SW2 มี SVI `Vlan20 = 10.10.20.3` อยู่แล้ว (Part 10) และเป็นหนึ่งใน Default Gateway ของ
VLAN นี้ (คู่กับ DIST-SW1 ที่ `10.10.20.2`) — เราจะตั้งให้ DIST-SW2 แจก IP ให้ IP Phone ใน
VLAN 20 ได้เองโดยตรง

```
DIST-SW2(config)# ip dhcp excluded-address 10.10.20.1 10.10.20.9
! กันที่อยู่ Infra ไว้ก่อนเสมอ (รายละเอียดเต็มใน Step 154) — .1 กันไว้ให้ HSRP VIP (Part 18),
! .2 และ .3 คือ SVI จริงของ DIST-SW1/DIST-SW2, ที่เหลือกันสำรองให้ Access Point/Infra เสียง

DIST-SW2(config)# ip dhcp pool VOICE-PHONES
DIST-SW2(dhcp-config)# network 10.10.20.0 255.255.255.0
DIST-SW2(dhcp-config)# default-router 10.10.20.2 10.10.20.3
DIST-SW2(dhcp-config)# dns-server 10.10.30.20
DIST-SW2(dhcp-config)# domain-name lab.local
DIST-SW2(dhcp-config)# lease 7
DIST-SW2(dhcp-config)# exit
```

> **สังเกต `default-router 10.10.20.2 10.10.20.3`**: เพราะยังไม่มี HSRP (Part 18) VLAN นี้ยังมี
> Default Gateway สองตัวแยกกัน (ไม่ใช่ Virtual IP เดียว) — การใส่ทั้งสอง IP ใน Option Default
> Gateway ทำให้ Client ที่รองรับหลาย Gateway (Multi-router Option, RFC 3442-ish behavior)
> มีทางเลือก Fallback ได้บ้าง แต่ **Client ส่วนใหญ่ (รวม Cisco IP Phone) จะใช้แค่ตัวแรกในลิสต์
> เท่านั้น** เป็น Default Gateway จริง — ความ Redundancy แบบสมบูรณ์จะมาพร้อม HSRP ใน Part 18
> ซึ่งตอนนั้นค่านี้จะถูกเปลี่ยนเป็น `default-router 10.10.20.1` (Virtual IP เดียว) แทน

### 152.4 คำสั่งปิดการทำงาน DHCP Server บน Interface ที่ไม่ต้องการ (ป้องกันความสับสน)

ถ้า Switch/Router เปิด `ip dhcp pool` ไว้แต่มี Interface บางตัวที่ไม่อยากให้ตอบ DHCP Request
เลย (เช่น Uplink ที่เป็น Routed Port) ให้ปิดเฉพาะจุด:

```
DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# no ip dhcp server         ! ปิดเฉพาะ Interface นี้ (IOS-XE)
```

> Global command `no service dhcp` จะปิด DHCP Server function **ทั้งเครื่อง** ทันที
> (รวม Relay Agent ด้วย) — ใช้เมื่อต้องการปิด DHCP ทุกบทบาทบน Device นั้นแบบเบ็ดเสร็จเท่านั้น

---

## Step 153 — DHCP Relay (`ip helper-address`) — ข้าม Subnet ด้วย Relay Agent

### 153.1 ทำไมต้องมี DHCP Relay

ตามที่อธิบายใน Step 151.5 — VLAN 10 (SALES) และ VLAN 40 (WIFI) ต้องขอ IP จาก
**DHCP-DNS-SRV (10.10.30.20)** ซึ่งอยู่ **VLAN 30 (SERVERS) คนละ VLAN กันโดยสิ้นเชิง**
Broadcast `DHCPDISCOVER` จาก PC ใน VLAN 10/40 จะไปไม่ถึง Server ตัวนี้เลยถ้าไม่มีตัวช่วย

**DHCP Relay Agent** คือฟีเจอร์ที่ตั้งไว้บน **Default Gateway ของ VLAN นั้น** (SVI) ให้ทำหน้าที่
"ดักฟัง" Broadcast DHCP ที่มาถึงตัวเอง แล้ว **แปลงเป็น Unicast Packet** ส่งตรงไปยัง DHCP Server
ที่ระบุไว้ ข้าม Subnet ไปได้ตามปกติเหมือน Traffic อื่นๆ ที่ Route ได้

คำสั่งเดียวที่ใช้คือ **`ip helper-address <dhcp-server-ip>`** ใส่ไว้ที่ SVI (หรือ Interface ใดก็ตาม
ที่เป็น Default Gateway ของ Subnet นั้น)

### 153.2 กลไกการทำงานเบื้องหลัง (Packet Flow เต็มรูปแบบ)

```
PC1 (VLAN10)              DIST-SW1 (SVI Vlan10=10.10.10.2)         DHCP-DNS-SRV
10.10.10.10 (ยังไม่มี IP)    ip helper-address 10.10.30.20            10.10.30.20 (VLAN30)
     │                              │                                      │
     │ ① DHCPDISCOVER               │                                      │
     │ src 0.0.0.0:68               │                                      │
     │ dst 255.255.255.255:67       │                                      │
     │──────────────────────────────►                                      │
     │                              │ ② SVI (ip helper-address) จับ        │
     │                              │    Broadcast UDP/67 ได้ แปลง:        │
     │                              │    src = 10.10.10.2  (SVI เอง)       │
     │                              │    dst = 10.10.30.20 (helper addr)   │
     │                              │    ตั้ง giaddr = 10.10.10.2           │
     │                              │    (Gateway IP Address field ใน      │
     │                              │     DHCP Header — บอก Server ว่า     │
     │                              │     Request มาจาก Subnet ไหน)         │
     │                              │──────────────────────────────────────►
     │                              │        (Unicast ข้าม VLAN ได้ปกติ)     │
     │                              │                                      │ ③ Server เห็น giaddr
     │                              │                                      │ = 10.10.10.2 จึงรู้ว่า
     │                              │                                      │ ต้องจ่าย IP จาก Scope
     │                              │                                      │ 10.10.10.0/24 (ไม่ใช่
     │                              │                                      │ Scope ของตัวเอง 30.0/24)
     │                              │◄──────────────────────────────────────
     │                              │ ④ DHCPOFFER (unicast กลับมาที่ .2)    │
     │◄──────────────────────────────                                      │
     │ ⑤ DIST-SW1 broadcast          │                                      │
     │   OFFER กลับลง VLAN10 ให้ PC1  │                                      │
     │   (ทำซ้ำแบบเดียวกันสำหรับ       │                                      │
     │    REQUEST/ACK จนครบ DORA)    │                                      │
```

> **หัวใจสำคัญที่สุดของ Step นี้ (มักออกสอบ)**: `ip helper-address` ต้องตั้งที่ **SVI ของ VLAN
> ต้นทาง** (ที่ Client อยู่) ไม่ใช่ตั้งที่ VLAN ปลายทาง (ที่ Server อยู่) — และ Field **`giaddr`
> (Gateway IP Address)** ใน DHCP Packet คือกลไกที่ทำให้ DHCP Server รู้ว่าต้องแจก IP จาก Scope
> ไหน (Server ดู giaddr แล้วจับคู่กับ Scope/Pool ที่มี Network ตรงกัน)

### 153.3 คำสั่ง `ip helper-address` Forward Protocol อะไรบ้างโดย Default

`ip helper-address` ไม่ได้ Forward แค่ DHCP (UDP/67-68) เท่านั้น — ตาม Default ของ IOS จะ
Forward **UDP Broadcast ทั้งหมด 8 Protocol** นี้ไปยัง Helper Address ที่ตั้งไว้:

| Protocol | UDP Port |
|---|---|
| Time | 37 |
| TACACS | 49 |
| DNS | 53 |
| BOOTP/DHCP Server | 67 |
| BOOTP/DHCP Client | 68 |
| TFTP | 69 |
| NetBIOS Name Server | 137 |
| NetBIOS Datagram Server | 138 |

> เพิ่ม/ลด Protocol ที่ Forward ได้ด้วย `ip forward-protocol udp <port>` (global config) —
> ถ้าต้องการให้ `ip helper-address` Forward **เฉพาะ** DHCP เท่านั้น (ตัด 7 Protocol ที่ไม่ใช้
> ออกเพื่อความปลอดภัย/ลด Broadcast รบกวน) ต้องใช้ `no ip forward-protocol udp <port>` ไล่ปิด
> ทีละ Port ที่ไม่ต้องการ — เป็นรายละเอียดที่ CCNP มักถามเจาะ

### 153.4 Worked Config — VLAN 10 (SALES) บน DIST-SW1 และ DIST-SW2

```
! ===== DIST-SW1 =====
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# description ** Gateway VLAN10-SALES **
DIST-SW1(config-if)# ip helper-address 10.10.30.20
DIST-SW1(config-if)# exit

! ===== DIST-SW2 =====
DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# description ** Gateway VLAN10-SALES **
DIST-SW2(config-if)# ip helper-address 10.10.30.20
DIST-SW2(config-if)# exit
```

> **ต้องตั้งทั้งสองตัว** เพราะ VLAN 10 มี Default Gateway อยู่ 2 จุด (DIST-SW1 `.2` และ
> DIST-SW2 `.3`) — PC ที่ตั้ง Gateway ไปทาง DIST-SW1 จะส่ง Broadcast ไปให้ DIST-SW1 เห็น
> เท่านั้น (ไม่ได้ไปถึง DIST-SW2) ดังนั้นทั้งสองตัวต้องมี `ip helper-address` เหมือนกันเสมอ
> ไม่ว่า PC จะ ARP ไปทางไหน Request ก็ต้องถูก Relay ไปหา Server ได้

### 153.5 Worked Config — VLAN 40 (WIFI) บน DIST-SW3 และ DIST-SW4

```
! ===== DIST-SW3 =====
DIST-SW3(config)# interface vlan 40
DIST-SW3(config-if)# description ** Gateway VLAN40-WIFI **
DIST-SW3(config-if)# ip helper-address 10.10.30.20
DIST-SW3(config-if)# exit

! ===== DIST-SW4 =====
DIST-SW4(config)# interface vlan 40
DIST-SW4(config-if)# description ** Gateway VLAN40-WIFI **
DIST-SW4(config-if)# ip helper-address 10.10.30.20
DIST-SW4(config-if)# exit
```

### 153.6 ทำไม VLAN 30 (SERVERS) และ VLAN 99 (MGMT) ไม่ต้องมี `ip helper-address`

| VLAN | ต้องมี Relay หรือไม่ | เหตุผล |
|---|---|---|
| 10 – SALES | ✅ ต้องมี | Client ต้องขอ DHCP ข้ามไปหา Server ที่ VLAN 30 |
| 20 – VOICE | ❌ ไม่ต้อง | ใช้ Local Pool บน DIST-SW2 เอง (Step 152) — Server "อยู่ในตัว Gateway" แล้ว |
| 30 – SERVERS | ❌ ไม่ต้อง | Server ทุกตัว (รวม DHCP-DNS-SRV เอง) ใช้ **Static IP** เสมอ — Server ไม่ควรพึ่ง DHCP |
| 40 – WIFI | ✅ ต้องมี | เหมือน VLAN 10 |
| 99 – MGMT | ❌ ไม่ต้อง | อุปกรณ์ Network (Switch/Router/AP) ทุกตัวใน MGMT ใช้ Static IP เสมอตาม Best Practice |

> **กฎเหล็กของ Network Design**: **Server และอุปกรณ์ Infrastructure ต้องใช้ Static IP เสมอ**
> (หรืออย่างน้อยก็ DHCP Reservation แบบตายตัว) เพราะถ้า Server เปลี่ยน IP โดยไม่ได้ตั้งใจ
> (เช่น Lease หมดแล้วได้ IP ใหม่) จะกระทบ Service ทั้งองค์กรที่พึ่งพา IP เดิมอยู่ — DHCP เหมาะกับ
> **End-user Device** ที่ไม่แคร์ว่า IP จะเปลี่ยนไปเรื่อยๆ เท่านั้น

---

## Step 154 — DHCP Excluded Address — กันที่อยู่ Static ออกจาก Pool

### 154.1 ทำไมต้อง Exclude

ใน Subnet เดียวกัน มักมีทั้ง Host ที่ใช้ DHCP (PC ทั่วไป) และ Host ที่ต้องใช้ **Static IP** เสมอ
(Server, Printer, Access Point, อุปกรณ์ Network) — ถ้าไม่กันที่อยู่กลุ่มนี้ออกจาก DHCP Pool
ไว้ก่อน มีความเสี่ยงที่ DHCP Server จะแจก IP ตัวเดียวกันซ้ำให้กับ Client DHCP โดยไม่รู้ตัว
เกิด **IP Conflict** ทันที (สอง Host ใช้ IP เดียวกัน = ทั้งสองใช้งานไม่ได้)

คำสั่งที่ใช้คือ `ip dhcp excluded-address <start> <end>` — ต้องอยู่ใน **Global Configuration**
เสมอ (ไม่ใช่ใน `(dhcp-config)#` mode) และควรตั้งไว้**ก่อน**สร้าง `ip dhcp pool` เสมอเพื่อไม่ให้
IP ที่แจกไปแล้วชนกับที่อยู่ Static ที่จะกันย้อนหลัง

### 154.2 Worked Example — SALES Pool กัน `.1–.19` ไว้ให้ Infra

ตาม IP Plan ของ VLAN 10 (SALES, 10.10.10.0/24) องค์กรมีการจอง Range ล่างของ Subnet ไว้ให้
อุปกรณ์ Static เสมอ (Gateway, Access Point ของ Floor นั้น, Printer เครือข่าย) — ตัวอย่างนี้แสดง
Syntax ของคำสั่งบน Cisco IOS โดยตรง (ใช้ Concept เดียวกับที่ VOICE Pool ใน Step 152):

```
DIST-SW1(config)# ip dhcp excluded-address 10.10.10.1 10.10.10.19
```

| Range ที่กัน | จองไว้ให้ |
|---|---|
| `10.10.10.1` | Default Gateway/HSRP VIP (Part 18) |
| `10.10.10.2` – `10.10.10.3` | SVI จริงของ DIST-SW1/DIST-SW2 (ก่อน HSRP) |
| `10.10.10.4` – `10.10.10.9` | สำรองสำหรับอุปกรณ์ Network เพิ่มเติมในอนาคต |
| `10.10.10.10` – `10.10.10.19` | Printer/อุปกรณ์ Static อื่นๆ ของแผนก Sales |
| `10.10.10.20` – `10.10.10.250` | **Pool จริงที่แจกให้ PC ผ่าน DHCP** |
| `10.10.10.251` – `10.10.10.254` | สำรองปลาย Range (Broadcast คือ `.255`, Network คือ `.0`) |

> **หมายเหตุสำคัญเรื่องสถาปัตยกรรมจริงของ Lab นี้**: VLAN 10 (SALES) ใช้ **DHCP-DNS-SRV
> กลาง** เป็น Server จริง (ผ่าน Relay ใน Step 153) ไม่ใช่ Cisco IOS ดังนั้น **ค่า Exclude/Scope
> นี้จะถูกตั้งค่าจริงที่ตัว DHCP-DNS-SRV เอง** (ไม่ใช่ที่ DIST-SW1/DIST-SW2) — แต่ Concept และ
> ตัวเลข Range เหมือนกันทุกประการ เพียงแค่วิธีตั้งค่าต่างกันตาม Platform ของ Server:

```
# ตัวอย่างฝั่ง Linux ISC-DHCP-Server (dhcpd.conf) บน DHCP-DNS-SRV
subnet 10.10.10.0 netmask 255.255.255.0 {
  range 10.10.10.20 10.10.10.250;
  option routers 10.10.10.2;
  option domain-name-servers 10.10.30.20;
  option domain-name "lab.local";
  default-lease-time 691200;   # 8 วัน (เป็นวินาที)
  max-lease-time 691200;
}
# หมายเหตุ: ISC-DHCP ไม่มีคำสั่ง exclude ตรงๆ — ทำโดย "ไม่รวม" Range .1-.19 ไว้ใน `range` ด้านบน

# ตัวอย่างฝั่ง Windows Server DHCP (PowerShell) บน DHCP-DNS-SRV
Add-DhcpServerv4Scope -Name "SALES" -StartRange 10.10.10.20 -EndRange 10.10.10.250 `
    -SubnetMask 255.255.255.0 -LeaseDuration 8.00:00:00
Add-DhcpServerv4ExclusionRange -ScopeId 10.10.10.0 -StartRange 10.10.10.1 -EndRange 10.10.10.19
Set-DhcpServerv4OptionValue -ScopeId 10.10.10.0 -Router 10.10.10.2 -DnsServer 10.10.30.20 -DnsDomain lab.local
```

### 154.3 Scope ของ VLAN 40 (WIFI) — แนวคิดเดียวกัน

```
# Exclude .1-.19 เหตุผลเดียวกับ VLAN 10 (Gateway, AP-1/AP-2 ที่อาจตั้ง Static, สำรอง Infra)
range 10.10.40.20 10.10.40.250;
option routers 10.10.40.2;
option domain-name-servers 10.10.30.20;
option domain-name "lab.local";
default-lease-time 14400;    # 4 ชั่วโมง — WiFi Client หมุนเวียนบ่อยกว่า Wired PC มาก
```

> **หลักการเลือก Lease Time**: VLAN ที่ Host เปลี่ยนบ่อย (WiFi, Guest Network) → Lease สั้น
> (ชั่วโมง) เพื่อให้ IP หมุนคืน Pool เร็ว ไม่เกิด Pool Exhaustion — VLAN ที่ Host อยู่ประจำ
> (Wired PC, IP Phone) → Lease ยาว (วัน) เพื่อลด DHCP Traffic ที่ไม่จำเป็นจากการ Renew บ่อยๆ

### 154.4 ผลที่เกิดถ้า Exclude Range แคบเกินไป

ถ้า Range ที่ Exclude เล็กเกินไป (เช่น Exclude แค่ `.1-.5` แต่จริงๆ มีอุปกรณ์ Static ไปจนถึง
`.30`) — DHCP Server จะแจก IP ที่ซ้อนทับกับอุปกรณ์ Static เหล่านั้นออกไปให้ PC เครื่องใหม่ที่
ขอ DHCP โดยไม่รู้ตัว เกิด **IP Conflict** ทันทีที่ IP นั้นถูกแจกออกไปจริง (รายละเอียดการตรวจจับ
Conflict อยู่ใน Step 159)

---

## Step 155 — DHCP Snooping เบื้องต้น — Trusted/Untrusted Port

### 155.1 ปัญหา: Rogue DHCP Server

DHCP ไม่มี Authentication ในตัวเอง — **ใครก็ตามที่เสียบอุปกรณ์เข้ากับ Access Port แล้วเปิด
DHCP Server ปลอม (Rogue DHCP Server) โดยตั้งใจหรือไม่ตั้งใจก็ตาม (เช่น พนักงานเอา Home
Router มาเสียบผิดที่) สามารถแจก IP/Gateway/DNS ปลอมให้ Client ในเครือข่ายได้ทันที** เพราะ
Client จะรับ `DHCPOFFER` ใบแรกที่มาถึงโดยไม่ตรวจสอบว่ามาจาก Server ที่ถูกต้องหรือไม่

ผลกระทบที่รุนแรงที่สุดคือ **Man-in-the-Middle**: Rogue Server แจก Gateway/DNS เป็น IP ของ
ตัวเอง ทำให้ Traffic ทั้งหมดของ Client วิ่งผ่านเครื่องปลอมก่อนออกอินเทอร์เน็ตจริง

### 155.2 แนวคิด DHCP Snooping — Trusted vs Untrusted Port

**DHCP Snooping** คือ Layer 2 Security Feature ที่ทำให้ Switch "ฟัง" DHCP Traffic ที่วิ่งผ่านตัวเอง
และตัดสินใจว่าจะให้ Message บางประเภทผ่านหรือไม่ ตามสถานะของ Port:

| ประเภท Port | อนุญาต Message อะไร | ใช้กับ Port แบบไหน |
|---|---|---|
| **Trusted** | อนุญาตทั้ง `DHCPOFFER`/`DHCPACK` (จาก Server) และ Request จาก Client | Uplink/Trunk ที่นำไปสู่ DHCP Server หรือ Relay Agent ที่ถูกต้องเท่านั้น (เช่น Trunk ขึ้น Core, Port ที่ต่อ DHCP-DNS-SRV) |
| **Untrusted** (Default ของทุก Port) | อนุญาตแค่ Message จาก Client (`DHCPDISCOVER`, `DHCPREQUEST`) — **บล็อก `DHCPOFFER`/`DHCPACK` ที่มาจาก Port นี้ทันที** | Access Port ทุกจุดที่ต่อ PC/User (เพราะ User ไม่ควรมี DHCP Server อยู่หลัง Port ของตัวเอง) |

```
                     ┌─────────────────────────────┐
                     │           DIST-SW1            │
   Trusted Port ────►│  (Trunk/Uplink ไปทาง Server)  │
   (อนุญาต OFFER/ACK)  │                                │◄──── Untrusted Port (Default)
                     │                                │      (บล็อก OFFER/ACK ที่มาจากทางนี้)
                     └─────────────────────────────┘             │
                                                                  │
                                                          PC1 (Access Port)
                                              ❌ ถ้ามีคนเสียบ Rogue DHCP Server
                                                 ที่ Port นี้ → OFFER ปลอมถูกบล็อกทันที
```

### 155.3 เหตุผลที่ Part นี้พูดถึงแค่ Concept — รายละเอียดเต็มอยู่ Part 21

DHCP Snooping เป็นฟีเจอร์ด้าน **Layer 2 Security** (คู่กับ Dynamic ARP Inspection และ IP Source
Guard) มากกว่าเป็นเรื่อง Availability/Configuration ของ DHCP โดยตรง — Config เต็มรูปแบบ
(`ip dhcp snooping`, `ip dhcp snooping vlan <list>`, `ip dhcp snooping trust`,
`ip dhcp snooping limit rate <pps>`, DHCP Snooping Binding Database) รวมถึงการผูกกับ
Dynamic ARP Inspection จะอยู่ใน **Part 21 (Layer 2 Security)** ซึ่งเป็น Part ที่รวม Security
Feature ระดับ Access Layer ทั้งหมดไว้ด้วยกัน — ที่นี่ให้จำแค่หลักการ **Trusted = ทางไป Server
จริง, Untrusted = ทุกอย่างที่เหลือ (Default)** ไปก่อน

> **ข้อสอบ CCNA มักถามแนวนี้**: "Port ใดควรตั้งเป็น Trusted ใน DHCP Snooping" → คำตอบคือ
> **Port/Trunk ที่นำไปสู่ DHCP Server หรือ Relay Agent ที่ถูกต้องเท่านั้น** ไม่ใช่ Access Port
> ของ End-user เด็ดขาด

---

## Step 156 — DNS Fundamentals: Recursive/Iterative Query และ Record Types

### 156.1 ทำไมต้องมี DNS

**DNS (Domain Name System)** แปลงชื่อที่มนุษย์จำง่าย (เช่น `dist-sw1.lab.local`,
`www.example.com`) ให้เป็น IP Address ที่อุปกรณ์ Network ใช้ Route จริง — โดยไม่มี DNS
มนุษย์ต้องจำ IP Address ทุกตัวที่ต้องการติดต่อ ซึ่งเป็นไปไม่ได้ในทางปฏิบัติ

### 156.2 Recursive Query vs Iterative Query

```
Client                  Recursive Resolver             Root/TLD/Authoritative Server
(ลูกค้า)                  (เช่น DHCP-DNS-SRV)              (เช่น Internet DNS Hierarchy)
  │                              │                                    │
  │ ① Query แบบ Recursive         │                                    │
  │   "www.example.com คือ IP    │                                    │
  │    อะไร — ตอบมาให้ครบเลย"      │                                    │
  │──────────────────────────────►                                    │
  │                              │ ② Query แบบ Iterative หลายรอบ        │
  │                              │   (ถาม Root → ถาม .com TLD →         │
  │                              │    ถาม Authoritative ของ example.com)│
  │                              │───────────────────────────────────►│
  │                              │◄───────────────────────────────────│
  │                              │      (ทำซ้ำหลาย Server จนได้คำตอบ)    │
  │◄──────────────────────────────                                    │
  │ ③ ได้คำตอบสุดท้ายกลับมาเลย     │                                    │
```

| ประเภท | ใครทำงานหนัก | ลักษณะ |
|---|---|---|
| **Recursive Query** | Server ที่รับ Query (Resolver) ต้องหาคำตอบให้ครบก่อนตอบ Client | Client → Resolver: ถามครั้งเดียว ได้คำตอบสุดท้ายเลย (ไม่สน Process เบื้องหลัง) |
| **Iterative Query** | ผู้ถามต้องไล่ถามไปเรื่อยๆ ทีละ Server จนกว่าจะเจอคำตอบ | Resolver → Root/TLD/Authoritative: ถามแล้วได้แค่ "ไปถามต่อที่นี่" ซ้ำไปเรื่อยๆ |

> โดยทั่วไป **Client ถาม Resolver แบบ Recursive เสมอ** (ขี้เกียจ รอคำตอบสุดท้ายอย่างเดียว)
> ส่วน **Resolver ไปถามต่อกับ DNS Server อื่นแบบ Iterative** (ไล่ถามทีละชั้นจาก Root ลงมา)
> — Cache ที่ Resolver เก็บไว้ (Caching) ช่วยลดจำนวนรอบ Iterative Query ในการถามครั้งต่อไป

### 156.3 DNS Record Types ที่วิศวกร Network ต้องรู้จัก

| Record | ชื่อเต็ม | ใช้ทำอะไร | ตัวอย่าง |
|---|---|---|---|
| **A** | Address (IPv4) | แปลงชื่อ → IPv4 Address | `dist-sw1.lab.local → 10.10.99.11` |
| **AAAA** | Address (IPv6) | แปลงชื่อ → IPv6 Address | `dist-sw1.lab.local → fd00:10:10:99::11` |
| **PTR** | Pointer (Reverse) | แปลง IP Address → ชื่อ (ย้อนทาง — ใช้ตรวจสอบ/Log) | `10.10.99.11 → dist-sw1.lab.local` |
| **CNAME** | Canonical Name (Alias) | ทำให้ชื่อหนึ่งเป็น "นามแฝง" ของอีกชื่อ | `ntp.lab.local → dhcp-dns-srv.lab.local` |
| **SRV** | Service | บอก Host+Port ของ Service เฉพาะ (เช่น VoIP, LDAP, Active Directory) | `_sip._tcp.lab.local → cucm.lab.local:5060` |
| **MX** | Mail Exchange | ระบุ Mail Server ของ Domain (เกี่ยวกับ Network น้อย แต่ควรรู้จัก) | `lab.local → mail.lab.local` |
| **NS** | Name Server | ระบุว่า Domain นี้ใช้ DNS Server ตัวไหนเป็น Authoritative | `lab.local → dhcp-dns-srv.lab.local` |

> **PTR Record สำคัญกับงาน Network Operation มากกว่าที่คิด**: หลายระบบ Logging/SNMP/Syslog
> (Part 17) จะแสดงชื่อ Host แทน IP ถ้า Reverse Lookup (PTR) สำเร็จ ทำให้ Log อ่านง่ายขึ้นมาก
> เช่น เห็น `dist-sw1.lab.local: Interface Vlan10 changed state to up` แทนที่จะเห็นแค่ IP ดิบๆ

### 156.4 คำสั่ง `ip domain-name` และ `ip name-server` บน Cisco IOS

Cisco Device เองก็เป็น "Client" ของ DNS ได้เช่นกัน เมื่อต้องอ้างถึง Hostname ในคำสั่งต่างๆ
(เช่น `ping ntp-server.lab.local`, หรือ NTP/Syslog Server ที่ตั้งด้วยชื่อแทน IP ใน **Part 17**)
ต้องตั้งค่า 2 คำสั่งนี้:

```
DIST-SW1(config)# ip domain-name lab.local
! ตั้ง Domain Name ของ Device เอง — ใช้เติมท้ายชื่อที่ไม่มี . (FQDN) โดย Auto

DIST-SW1(config)# ip name-server 10.10.30.20
! ตั้ง DNS Server ที่ Device จะส่ง Query ไปถาม (รองรับใส่ได้สูงสุด 6 ตัว)
DIST-SW1(config)# ip name-server 10.10.30.20 8.8.8.8
```

> **ย้อนกลับไป Part 2**: เราสั่ง `no ip domain-lookup` ไว้กับทุกอุปกรณ์ เพื่อป้องกัน CLI ค้าง
> เวลาพิมพ์คำสั่งผิด (IOS จะพยายาม DNS Lookup คำที่พิมพ์ผิดเป็นเวลานานก่อน Timeout) — ถ้าต้องการ
> ให้ Device Resolve Hostname ได้จริง (เพื่อใช้ใน `ping <hostname>` หรืออ้าง Hostname ใน NTP/
> Syslog Config ที่ Part 17) **ต้องเปิดกลับด้วย `ip domain-lookup`** เสมอ คู่กับตั้ง
> `ip name-server` ให้ถูกต้อง — ไม่มี `ip name-server` ตั้งไว้ ต่อให้เปิด `ip domain-lookup`
> ก็ Resolve ไม่ได้อยู่ดี (ไม่รู้จะถามใคร)

```
DIST-SW1(config)# ip domain-lookup
DIST-SW1(config)# ip domain-name lab.local
DIST-SW1(config)# ip name-server 10.10.30.20

DIST-SW1# ping dhcp-dns-srv.lab.local
Translating "dhcp-dns-srv.lab.local"...domain server (10.10.30.20) [OK]
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.30.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

---

## Step 157 — Cisco IOS เป็น DNS Server/Proxy — `ip dns server` และ `ip host`

### 157.1 ทำไมต้องมี Cisco IOS DNS Server/Proxy สำหรับ Lab

การพึ่งพา DNS Server กลาง (DHCP-DNS-SRV) เพียงจุดเดียวทั้งองค์กร มีความเสี่ยงถ้า Server
ตัวนั้น Down หรือ WAN Link ไปยัง Segment นั้นขาด — Cisco IOS มีความสามารถทำหน้าที่เป็น
**DNS Server/Proxy** ในตัวได้เช่นกัน โดยใช้ **ตารางชื่อ Static (`ip host`)** ที่ Admin กำหนดไว้
ล่วงหน้า ซึ่งเหมาะมากกับ **Internal Lookup Table ของอุปกรณ์ Network เอง** เพราะรายชื่อ
Hostname ของ Router/Switch ในองค์กรไม่ได้เปลี่ยนบ่อยและควบคุมได้แน่นอน 100% (ไม่พึ่งพา
DNS Server ภายนอกที่อาจ Down)

### 157.2 คำสั่ง `ip host` — Static Host Entry

```
Router(config)# ip host <hostname> <ip-address>
```

เทียบเท่ากับไฟล์ `/etc/hosts` บน Linux หรือ `C:\Windows\System32\drivers\etc\hosts` บน
Windows — เป็นการ Map ชื่อ → IP แบบ Static ที่ Device เก็บไว้ในตัวเอง ไม่ต้องยิง Query ออกไป
หา DNS Server ภายนอกเลย

### 157.3 คำสั่ง `ip dns server` — เปิดให้ Device ตัวนี้ตอบ DNS Query จากเครื่องอื่นได้

```
Router(config)# ip dns server
```

เมื่อเปิดคำสั่งนี้ Router/Switch จะเริ่ม **ฟัง DNS Query ที่ Port UDP/53** และตอบกลับด้วยข้อมูล
จาก `ip host` Table ของตัวเอง (ถ้าไม่พบใน Table และมีการตั้ง `ip name-server` ไว้ด้วย มันจะ
Forward Query นั้นต่อไปยัง Server ที่ตั้งไว้ — ทำหน้าที่เป็น **DNS Proxy/Forwarder**)

### 157.4 Worked Example — ตั้ง WAN-EDGE-1 เป็น Internal Lookup Table ของ Lab ทั้งหมด

```
WAN-EDGE-1(config)# ip dns server
WAN-EDGE-1(config)# ip domain-name lab.local

! ===== Core Layer =====
WAN-EDGE-1(config)# ip host core-sw1.lab.local 1.1.1.1
WAN-EDGE-1(config)# ip host core-sw2.lab.local 1.1.1.2

! ===== Distribution Layer =====
WAN-EDGE-1(config)# ip host dist-sw1.lab.local 1.1.1.11
WAN-EDGE-1(config)# ip host dist-sw2.lab.local 1.1.1.12
WAN-EDGE-1(config)# ip host dist-sw3.lab.local 1.1.1.13
WAN-EDGE-1(config)# ip host dist-sw4.lab.local 1.1.1.14

! ===== WAN Edge =====
WAN-EDGE-1(config)# ip host wan-edge-1.lab.local 1.1.1.21
WAN-EDGE-1(config)# ip host wan-edge-2.lab.local 1.1.1.22

! ===== Server / Services =====
WAN-EDGE-1(config)# ip host dhcp-dns-srv.lab.local 10.10.30.20

! ===== Access Layer (ใช้ MGMT IP เพราะไม่มี Loopback) =====
WAN-EDGE-1(config)# ip host access-sw1.lab.local 10.10.99.21
WAN-EDGE-1(config)# ip host access-sw2.lab.local 10.10.99.22
WAN-EDGE-1(config)# ip host access-sw3.lab.local 10.10.99.23
WAN-EDGE-1(config)# ip host access-sw4.lab.local 10.10.99.24

! ===== Wireless =====
WAN-EDGE-1(config)# ip host wlc-1.lab.local 10.10.99.30
```

> **หมายเหตุ IP ของ ACCESS-SW1-4/WLC-1**: ตัวเลข MGMT IP เฉพาะของ Step นี้เป็นตัวอย่าง
> ประกอบการสอน `ip host` เท่านั้น — ค่า MGMT IP จริงที่ใช้ทำงานตลอดหลักสูตรให้ยึดตาม
> [`00-ip-address-plan.md`](00-ip-address-plan.md) เป็นหลักเสมอถ้ามีการระบุ MGMT IP
> รายอุปกรณ์เพิ่มเติมใน Part หลังๆ

### 157.5 ทดสอบการ Resolve จากอุปกรณ์อื่นที่ชี้ `ip name-server` มาที่ WAN-EDGE-1

```
DIST-SW1(config)# ip name-server 1.1.1.21
DIST-SW1(config)# ip domain-lookup

DIST-SW1# ping dist-sw3.lab.local
Translating "dist-sw3.lab.local"...domain server (1.1.1.21) [OK]
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/3/5 ms
```

### 157.6 ตรวจสอบ `ip host` Table ที่ตั้งไว้

```
WAN-EDGE-1# show hosts
Default domain is lab.local
Name/address lookup uses domain service
Name servers are 255.255.255.255

Codes: UN - unknown, EX - expired, OK - OK, ?? - revalidate
       temp - temporary, perm - permanent
       NA - Not Applicable  NB - No Bind  NBW - No Bind Wait

Host                       Port  Flags      Age Type   Address(es)
core-sw1.lab.local         None  (perm, OK)   0  IP     1.1.1.1
core-sw2.lab.local         None  (perm, OK)   0  IP     1.1.1.2
dist-sw1.lab.local         None  (perm, OK)   0  IP     1.1.1.11
dist-sw2.lab.local         None  (perm, OK)   0  IP     1.1.1.12
dist-sw3.lab.local         None  (perm, OK)   0  IP     1.1.1.13
dist-sw4.lab.local         None  (perm, OK)   0  IP     1.1.1.14
dhcp-dns-srv.lab.local     None  (perm, OK)   0  IP     10.10.30.20
```

> **`(perm, OK)`** หมายถึง Entry นี้เป็น Static (Permanent) ที่ตั้งไว้ด้วย `ip host` เอง — ต่างจาก
> Entry ที่ได้จากการ Query DNS Server จริง ซึ่งจะโชว์ `(temp, OK)` และมี Age/TTL ที่ลดลงเรื่อยๆ
> จนหมดอายุ (Expire) แล้วต้อง Query ใหม่

---

## Step 158 — DHCPv6: SLAAC vs Stateful DHCPv6 (ต่อจาก Part 8)

### 158.1 ทวนความจำจาก Part 8

[Part 8](part-008-ipv6-fundamentals.md) แนะนำ IPv6 Addressing และ **SLAAC (Stateless
Address Autoconfiguration)** ไปแล้วว่า Host สามารถสร้าง IPv6 Address ของตัวเองได้จาก
**Router Advertisement (RA)** ที่ Router/SVI ส่งออกมาเป็นระยะ โดยไม่ต้องมี DHCP Server เลย
(ต่าง IPv4 ที่ต้องพึ่ง DORA เสมอ) — Part นี้จะขยายความว่า DHCPv6 เข้ามาเสริม SLAAC ได้อย่างไร

### 158.2 เปรียบเทียบ 3 รูปแบบการแจก IPv6 Address

| รูปแบบ | Address มาจากไหน | Option อื่น (DNS ฯลฯ) มาจากไหน | RA Flag ที่เกี่ยว |
|---|---|---|---|
| **SLAAC ล้วน** | Host สร้างเอง (Prefix จาก RA + Interface ID) | ไม่มี (Host ต้องรู้ DNS Server เองผ่านทางอื่น) | `M=0, O=0` |
| **SLAAC + Stateless DHCPv6** | Host สร้างเอง (เหมือนซ้าย) | ขอ DNS/Domain **เพิ่มเติม** จาก DHCPv6 Server (ไม่ขอ Address) | `M=0, O=1` |
| **Stateful DHCPv6** | ขอทั้ง Address **และ** Option ทั้งหมดจาก DHCPv6 Server (คล้าย DORA ของ IPv4) | ขอจาก DHCPv6 Server เหมือนกัน | `M=1` |

- **M Flag (Managed Address Configuration)** = 1 → บอก Host ว่า "ให้ไปขอ Address จาก
  DHCPv6 Server แบบ Stateful เต็มรูปแบบ ห้ามใช้ SLAAC สร้างเอง"
- **O Flag (Other Configuration)** = 1 → บอก Host ว่า "Address ให้ใช้ SLAAC สร้างเอง แต่ให้ไป
  ขอ Option อื่น (เช่น DNS Server ผ่าน RDNSS หรือ DHCPv6) เพิ่มเติม"

### 158.3 DHCPv6 Process — SARR (คล้าย DORA แต่ไม่เหมือนกัน)

```
Client                                          DHCPv6 Server
  │  ① SOLICIT   (multicast ff02::1:2 "All_DHCP_Relay_Agents_and_Servers")
  │───────────────────────────────────────────────────►│
  │  ② ADVERTISE                                        │
  │◄───────────────────────────────────────────────────│
  │  ③ REQUEST                                          │
  │───────────────────────────────────────────────────►│
  │  ④ REPLY                                            │
  │◄───────────────────────────────────────────────────│
```

**S-A-R-R**: **S**olicit → **A**dvertise → **R**equest → **R**eply — แนวคิดเหมือน DORA ของ
IPv4 ทุกประการ (Discover↔Solicit, Offer↔Advertise, Request↔Request, Ack↔Reply) ต่างกัน
แค่ **IPv6 ไม่มี Broadcast เลย** จึงใช้ **Multicast** (`ff02::1:2`) แทน Broadcast ตลอดกระบวนการ

### 158.4 `ipv6 dhcp relay` — แนวคิด Relay สำหรับ DHCPv6

เหมือนกับ IPv4 ที่ DHCP Broadcast ไม่ข้าม Subnet ได้เอง — DHCPv6 Multicast ก็มีขอบเขตจำกัด
เช่นกัน (Link-local Scope) ถ้า Stateful DHCPv6 Server ไม่ได้อยู่ในวง Subnet เดียวกับ Client
ก็ต้องมี Relay เช่นกัน คำสั่งที่ใช้คือ:

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# ipv6 dhcp relay destination <dhcpv6-server-ipv6-address>
```

> Syntax และรายละเอียดเชิงลึกของ `ipv6 dhcp relay`, `ipv6 dhcp pool`, และการตั้งค่า
> Stateful DHCPv6 Server เต็มรูปแบบบน Cisco IOS จะอยู่ใน Part ที่กลับมาขยาย IPv6 Services
> โดยเฉพาะช่วงหลังของหลักสูตร (Advanced IPv6) — ที่นี่ให้เข้าใจแค่ว่า **แนวคิด Relay เดียวกันกับ
> IPv4 ถูกนำมาใช้กับ DHCPv6 เช่นกัน เพียงเปลี่ยนจาก Broadcast เป็น Multicast Scope และเปลี่ยน
> คำสั่งจาก `ip helper-address` เป็น `ipv6 dhcp relay destination`**

---

## Step 159 — Verification & Troubleshooting DHCP/DNS

### 159.1 คำสั่ง Verify DHCP บน Cisco IOS (กรณีเป็น Local Pool เช่น VOICE VLAN 20)

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show ip dhcp binding` | รายการ IP ที่แจกไปแล้วทั้งหมด พร้อม MAC Address และเวลาที่ Lease จะหมด |
| `show ip dhcp pool [name]` | สถานะของ Pool: จำนวน IP ที่แจกไปแล้ว, จำนวนที่เหลือ, Range |
| `show ip dhcp conflict` | รายการ IP ที่ตรวจพบว่า **ชนกัน** (Conflict) ระหว่าง Static Host กับ DHCP Pool |
| `show ip dhcp server statistics` | สถิติ DORA Message ทั้งหมดที่ Server ประมวลผล (จำนวน Discover/Offer/Request/Ack) |
| `debug ip dhcp server events` | Debug แบบสรุป Event (แนะนำใช้ก่อนเสมอ กิน CPU น้อยกว่า) |
| `debug ip dhcp server packet` | Debug แบบดู Packet DORA ทุกใบแบบละเอียด (กิน CPU สูง ใช้เมื่อจำเป็นจริงๆ) |

### 159.2 ตัวอย่าง Output จริงที่ต้องอ่านให้เป็น (VOICE Pool บน DIST-SW2)

```
DIST-SW2# show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address       Client-ID/               Lease expiration        Type
                 Hardware address/
                 User name
10.10.20.15      0011.2233.4455           Sep 27 2026 08:14 AM     Automatic
10.10.20.16      0011.2233.4466           Sep 27 2026 08:16 AM     Automatic

DIST-SW2# show ip dhcp pool VOICE-PHONES
Pool VOICE-PHONES :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/last)       : 0 / 0
 Total addresses                : 245
 Leased addresses                : 2
 Pending event                   : none
  1 subnet is currently in the pool :
  Current index        IP address range                    Leased addresses
  10.10.20.16           10.10.20.10  - 10.10.20.254           2

DIST-SW2# show ip dhcp conflict
IP address       Detection method     Detection time             VRF
10.10.20.12      Ping                 Sep 26 2026 11:02 AM

DIST-SW2# show ip dhcp server statistics
Memory usage                     24521
Address pools                    1
Database agents                  0
Automatic bindings                2
Manual bindings                   0
Expired bindings                  0
Malformed messages                0
Secure arp entries                0

Message                          Received
BOOTREQUEST                       0
DHCPDISCOVER                      3
DHCPREQUEST                       2
DHCPDECLINE                       0
DHCPRELEASE                       0
DHCPINFORM                        0

Message                          Sent
BOOTREPLY                         0
DHCPOFFER                         3
DHCPACK                           2
DHCPNAK                           0
```

### 159.3 ตรวจสอบ Relay Configuration ที่ SVI (กรณี VLAN 10/40 ที่ Relay ไปหา Server กลาง)

DIST-SW1–4 ไม่ได้เป็น DHCP Server เองสำหรับ VLAN 10/40 จึงไม่มี `show ip dhcp binding`
ให้ดู (Binding อยู่ที่ DHCP-DNS-SRV) — สิ่งที่ต้องตรวจสอบฝั่ง Cisco คือ **Helper Address ถูกตั้ง
ถูกต้องหรือไม่**:

```
DIST-SW1# show ip interface vlan 10 | include Helper
  Helper address is 10.10.30.20

DIST-SW1# show running-config interface vlan 10
Building configuration...

interface Vlan10
 description ** Gateway VLAN10-SALES **
 ip address 10.10.10.2 255.255.255.0
 ip helper-address 10.10.30.20
end
```

> ฝั่ง DHCP-DNS-SRV (Server กลาง) ให้ตรวจสอบจาก Console/Lease Table ของ Platform นั้นโดย
> ตรง เช่น Windows Server DHCP Console → Address Leases, หรือ Linux `cat /var/lib/dhcp/
> dhcpd.leases` — เห็น Lease ของ Subnet `10.10.10.0/24` ปรากฏอยู่แม้ Server ตัวนี้อยู่ VLAN 30
> คนละวงกับ Client เลย = Relay ทำงานถูกต้องแล้ว

### 159.4 ทดสอบ DNS แบบเทียบเท่า `nslookup`

Cisco IOS ไม่มีคำสั่ง `nslookup` ตรงๆ แต่ใช้คำสั่งที่ Trigger DNS Lookup แทนได้ (เช่น `ping`,
`telnet`, `traceroute` ด้วย Hostname) หรือใช้ Client จริงบน PC:

```
! ฝั่ง Cisco IOS — Trigger Lookup ผ่านคำสั่งที่รองรับ Hostname
DIST-SW1# ping dhcp-dns-srv.lab.local
Translating "dhcp-dns-srv.lab.local"...domain server (10.10.30.20) [OK]

! ฝั่ง PC (Windows/Linux/Mac) — คำสั่งมาตรฐานสำหรับทดสอบ DNS โดยตรง
C:\> nslookup dhcp-dns-srv.lab.local
Server:   10.10.30.20
Address:  10.10.30.20#53
Name:     dhcp-dns-srv.lab.local
Address:  10.10.30.20

$ dig dhcp-dns-srv.lab.local          (Linux/Mac — ให้รายละเอียด Record ครบกว่า nslookup)
```

### 159.5 ปัญหาที่พบบ่อยที่สุดและวิธีแก้

| ปัญหา | อาการที่เห็น | วิธีแก้ |
|---|---|---|
| **`ip helper-address` ชี้ผิด IP** | PC ไม่ได้รับ IP เลย (APIPA/169.254.x.x) แม้ Server ทำงานปกติ | ตรวจ `show running-config interface vlan X` ว่า Helper Address ตรงกับ IP จริงของ DHCP-DNS-SRV (`10.10.30.20`) หรือไม่ — พิมพ์ผิดเป็น IP อื่นบ่อยมาก |
| **ลืมตั้ง `ip helper-address` ที่ SVI ตัวที่สอง** | PC บางเครื่องได้ IP บางเครื่องไม่ได้ (ขึ้นกับว่า ARP ไปทาง Gateway ตัวไหน) | ตรวจสอบว่าตั้ง Helper Address ครบทั้ง DIST-SW1 **และ** DIST-SW2 (หรือ SW3/SW4) ไม่ใช่แค่ตัวเดียว |
| **Exclude Range แคบเกินไป → Pool Exhaustion เร็ว หรือ IP Conflict** | `show ip dhcp conflict` มี Entry, หรือ Static Device ใช้งานไม่ได้เพราะถูกแจกซ้ำ | ขยาย Exclude Range ให้ครอบคลุม Static Device ทั้งหมดจริง แล้ว Clear Binding ที่ Conflict ด้วย `clear ip dhcp conflict *` |
| **DHCP Conflict (สอง Host ใช้ IP เดียวกัน)** | Host ทั้งสองเครื่อง Connectivity ใช้งานไม่ได้แบบสุ่มๆ | `show ip dhcp conflict` ดู IP ที่ชนกัน แก้ที่ต้นเหตุ (มักเป็น Static IP ที่ไม่ได้ Exclude) แล้ว `clear ip dhcp conflict <ip>` |
| **Firewall/ACL บล็อก UDP/67-68 ระหว่างทาง** | Relay ตั้งถูกแล้วแต่ยังไม่มี Binding เกิดขึ้นที่ Server เลย | ตรวจสอบ ACL บน Interface ระหว่างทาง (Part 13-14) ว่าอนุญาต UDP/67, 68 ผ่านหรือไม่ |
| **`ip domain-lookup` ปิดอยู่ (ค่า Default จาก Part 2)** | คำสั่งที่ใช้ Hostname (`ping <name>`) ค้างหรือ Error `Unrecognized host` ทันที | เปิด `ip domain-lookup` และตรวจว่ามี `ip name-server` ตั้งไว้ถูกต้อง |
| **`show ip dhcp binding` ไม่เห็น Client ที่ควรมี** | มักเป็นเพราะ Client นั้นถูก Relay ไปหา Server อื่น ไม่ใช่ Local Pool ของ Switch ตัวนี้ | ตรวจว่า VLAN นั้นใช้ Local Pool หรือ Relay — ถ้า Relay ต้องไปดู Binding ที่ Server ปลายทาง ไม่ใช่ที่ Switch |

---

## Step 160 — Lab เต็มรูปแบบ: DHCP Relay ทั่ว Topology + PC1 รับ Lease จริง

### 160.1 Topology สรุปของ Lab นี้

```
PC1 (VLAN10-SALES)                                    DHCP-DNS-SRV
10.10.10.10 (ขอ DHCP)                                  10.10.30.20/24
  gateway .2  DNS .20                                  gateway 10.10.30.2 (DIST-SW3)
      │                                                       │
 ACCESS-SW1 (trunk 10,20,99)                          ACCESS-SW3 (trunk 30,40,99)
      │                                                       │
 ┌────┴─────┐   Vlan10=.2 helper→.20         Vlan30=.2  ┌─────┴─────┐
 │ DIST-SW1 │───────────────────┐                       │ DIST-SW3  │
 └────┬─────┘                    │                       └─────┬─────┘
 ┌────┴─────┐   Vlan10=.3 helper→.20        (ปกติไม่ Relay - Server อยู่ VLAN นี้เลย)
 │ DIST-SW2 │───────────────────┤
 └────┬─────┘                    │
      │                          ▼
 ┌────┴──────────────────────────────────┐
 │              CORE-SW1 ⇄ CORE-SW2       │  (Routed Port ล้วน — ดู Part 10)
 └────┬──────────────────────────┬────────┘
 ┌────┴─────┐               ┌────┴─────┐
 │ DIST-SW3 │               │ DIST-SW4 │   Vlan40=.2/.3 helper→10.10.30.20
 └────┬─────┘               └────┬─────┘
      │                          │
 ACCESS-SW3/4 (trunk 30,40,99)   │
      │                          │
   Server1 (VLAN30, Static)   PC-WIFI (VLAN40, ขอ DHCP)
```

### 160.2 Running-config ที่เพิ่มเข้าไปใน DIST-SW1–4 (ส่วนที่เกี่ยวกับ DHCP เท่านั้น)

ต่อยอดจาก Config เต็มของ Part 10 (Step 96-97) — เพิ่มแค่ 1 บรรทัดในแต่ละ SVI ที่เกี่ยวข้อง:

```
! ===== DIST-SW1 =====
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# ip helper-address 10.10.30.20
DIST-SW1(config-if)# exit
DIST-SW1(config)# ip domain-name lab.local
DIST-SW1(config)# ip name-server 10.10.30.20
DIST-SW1(config)# end
DIST-SW1# copy running-config startup-config

! ===== DIST-SW2 =====
DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# ip helper-address 10.10.30.20
DIST-SW2(config-if)# exit
DIST-SW2(config)# ip domain-name lab.local
DIST-SW2(config)# ip name-server 10.10.30.20
DIST-SW2(config)# end
DIST-SW2# copy running-config startup-config

! ===== DIST-SW3 =====
DIST-SW3(config)# interface vlan 40
DIST-SW3(config-if)# ip helper-address 10.10.30.20
DIST-SW3(config-if)# exit
DIST-SW3(config)# ip domain-name lab.local
DIST-SW3(config)# ip name-server 10.10.30.20
DIST-SW3(config)# end
DIST-SW3# copy running-config startup-config

! ===== DIST-SW4 =====
DIST-SW4(config)# interface vlan 40
DIST-SW4(config-if)# ip helper-address 10.10.30.20
DIST-SW4(config-if)# exit
DIST-SW4(config)# ip domain-name lab.local
DIST-SW4(config)# ip name-server 10.10.30.20
DIST-SW4(config)# end
DIST-SW4# copy running-config startup-config
```

### 160.3 Scope Configuration บน DHCP-DNS-SRV (สรุปทั้ง 2 Subnet ที่ Relay มาถึง)

| Scope | Subnet | Range ที่แจกจริง | Excluded | Gateway | DNS | Lease |
|---|---|---|---|---|---|---|
| SALES | 10.10.10.0/24 | .20 – .250 | .1 – .19 | 10.10.10.2 | 10.10.30.20 | 8 วัน |
| WIFI | 10.10.40.0/24 | .20 – .250 | .1 – .19 | 10.10.40.2 | 10.10.30.20 | 4 ชั่วโมง |

### 160.4 Verify ทีละจุดตามลำดับ (Layer 1 → Layer 7)

```
! 1) ยืนยัน Helper Address ถูกตั้งครบทั้ง 4 ตัว DIST-SW
DIST-SW1# show ip interface vlan 10 | include Helper
  Helper address is 10.10.30.20
DIST-SW2# show ip interface vlan 10 | include Helper
  Helper address is 10.10.30.20
DIST-SW3# show ip interface vlan 40 | include Helper
  Helper address is 10.10.30.20
DIST-SW4# show ip interface vlan 40 | include Helper
  Helper address is 10.10.30.20

! 2) ยืนยัน Route ไปถึง VLAN 30 (SERVERS) จากทุกจุด — ถ้า Route ไม่มี Relay ก็ไปไม่ถึง
DIST-SW1# show ip route 10.10.30.0
Routing entry for 10.10.30.0/24
  Known via "ospf 1", distance 110, metric 20     (ตั้งแต่ Part 11 เป็น OSPF แล้ว)
  Last update from 10.255.10.1 on TenGigabitEthernet1/1/1

! 3) ทดสอบจาก PC1 (VLAN10) — ขอ DHCP ใหม่
C:\> ipconfig /release
C:\> ipconfig /renew

Ethernet adapter Ethernet:
   Connection-specific DNS Suffix  . : lab.local
   IPv4 Address. . . . . . . . . . . : 10.10.10.24
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.10.2
   DHCP Server . . . . . . . . . . . : 10.10.30.20
   DNS Servers . . . . . . . . . . . : 10.10.30.20
   Lease Obtained. . . . . . . . . . : Saturday, September 26, 2026 10:00:00 AM
   Lease Expires . . . . . . . . . . : Sunday, October 4, 2026 10:00:00 AM

! 4) ทดสอบจาก PC-WIFI (VLAN40) — ขอ DHCP ใหม่ผ่าน AP-1/WLC-1
C:\> ipconfig /release
C:\> ipconfig /renew

Wireless LAN adapter Wi-Fi:
   Connection-specific DNS Suffix  . : lab.local
   IPv4 Address. . . . . . . . . . . : 10.10.40.31
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.40.2
   DHCP Server . . . . . . . . . . . : 10.10.30.20
   DNS Servers . . . . . . . . . . . : 10.10.30.20
   Lease Obtained. . . . . . . . . . : Saturday, September 26, 2026 10:05:00 AM
   Lease Expires . . . . . . . . . . : Saturday, September 26, 2026 02:05:00 PM

! 5) ยืนยัน DNS ทำงานได้จริงจาก IP ที่ได้รับมา
C:\> nslookup dhcp-dns-srv.lab.local
Server:   10.10.30.20
Address:  10.10.30.20#53
Name:     dhcp-dns-srv.lab.local
Address:  10.10.30.20

! 6) ping ข้าม VLAN ยืนยัน Connectivity สมบูรณ์ปลายทางถึงปลายทาง (ทวน Part 10-11)
C:\> ping 10.10.30.10
Reply from 10.10.30.10: bytes=32 time=3ms TTL=125
```

**ผลลัพธ์**: PC1 (VLAN 10) และ PC-WIFI (VLAN 40) ได้รับ IP Address, Default Gateway และ
DNS Server ที่ถูกต้องครบถ้วนจาก DHCP-DNS-SRV (10.10.30.20) ทั้งที่ Server อยู่คนละ VLAN
กันโดยสิ้นเชิง เป็นเครื่องพิสูจน์ว่า **DHCP Relay ผ่าน `ip helper-address` ทำงานถูกต้องทั่ว
Topology ทั้งหมด** ตั้งแต่ Distribution Layer ทั้ง 4 ตัว ผ่าน Core Layer ไปจนถึง Server จริง

---

## แบบฝึกหัดทวนความเข้าใจ Part 16

1. เรียงลำดับ DORA Process ให้ถูกต้อง และอธิบายว่าขั้นตอนใดที่ Client เป็นฝ่ายเริ่มก่อน
2. เพราะเหตุใด DHCP Request จาก PC ใน VLAN 10 จึงไปไม่ถึง DHCP Server ที่อยู่ใน VLAN 30
   ได้เองโดยไม่มีการตั้งค่าเพิ่มเติม?
3. คำสั่งใดใน Cisco IOS ใช้แก้ปัญหาในข้อ 2 และต้องตั้งไว้ที่ Interface ใด (ฝั่ง Client หรือฝั่ง
   Server)?
4. `ip dhcp excluded-address` มีไว้เพื่ออะไร และถ้าตั้ง Range ที่ Exclude แคบเกินไปจะเกิดปัญหา
   อะไรตามมา?
5. ความแตกต่างระหว่าง SLAAC ล้วน, SLAAC + Stateless DHCPv6, และ Stateful DHCPv6 คืออะไร
   และ RA Flag ตัวไหนที่บอก Host ว่าต้องขอ Address จาก DHCPv6 Server แบบเต็มรูปแบบ?

**เฉลย:**

1. **D**iscover → **O**ffer → **R**equest → **A**cknowledge — Client เริ่มก่อนด้วย
   `DHCPDISCOVER` (ขั้นที่ 1) และ `DHCPREQUEST` (ขั้นที่ 3) เสมอ ส่วน Server เป็นฝ่ายตอบด้วย
   `DHCPOFFER` และ `DHCPACK`
2. เพราะ `DHCPDISCOVER` เป็น Broadcast (L2 และ L3) เสมอ และ Router/SVI จะไม่ Forward
   Broadcast ข้าม VLAN/Subnet ให้เองตามนิยามของ Broadcast Domain — Server ที่อยู่ VLAN อื่น
   จึงไม่ได้ยิน Request นี้เลย
3. คำสั่ง `ip helper-address <dhcp-server-ip>` ต้องตั้งไว้ที่ **SVI ของ VLAN ต้นทาง** (ฝั่งที่
   Client อยู่ เช่น Vlan10 บน DIST-SW1/DIST-SW2) ไม่ใช่ตั้งที่ฝั่ง Server
4. ใช้กันที่อยู่ที่จองไว้ให้อุปกรณ์ Static (Server, Printer, Network Device) ไม่ให้ DHCP Server
   แจกซ้ำให้ Client อื่นโดยไม่ตั้งใจ — ถ้า Exclude Range แคบเกินไป DHCP อาจแจก IP ที่ชนกับ
   Static Device เหล่านั้น เกิด **IP Conflict** ทำให้ทั้งสองฝั่งใช้งานไม่ได้
5. SLAAC ล้วน = Host สร้าง Address เองจาก RA และไม่ขอ Option อื่นเลย (`M=0,O=0`);
   SLAAC + Stateless DHCPv6 = Host สร้าง Address เองแต่ขอ Option อื่น (เช่น DNS) เพิ่มจาก
   DHCPv6 Server (`M=0,O=1`); Stateful DHCPv6 = ขอทั้ง Address และ Option ทั้งหมดจาก
   DHCPv6 Server เต็มรูปแบบ (`M=1`) — Flag ที่บอกว่าต้องขอ Address แบบ Stateful คือ
   **M Flag (Managed Address Configuration) = 1**

---

## สรุป Part 16

Part นี้ครอบคลุมพื้นฐาน DHCP ตั้งแต่ DORA Process, Lease, การตั้ง Cisco IOS เป็น DHCP Server
เอง (Local Pool), **DHCP Relay ผ่าน `ip helper-address`** ซึ่งเป็นหัวใจสำคัญที่สุดของ Part นี้
สำหรับการแจก IP ข้าม VLAN ไปยัง **DHCP-DNS-SRV (10.10.30.20)** กลาง, การกันที่อยู่ Static
ด้วย Excluded Address, แนวคิด DHCP Snooping เบื้องต้น, พื้นฐาน DNS ทั้ง Recursive/Iterative
Query และ Record Types, การตั้ง Cisco IOS เป็น DNS Server/Proxy ด้วย `ip host`, และปิดท้าย
ด้วยการทวนความเข้าใจ DHCPv6 ที่ต่อเนื่องจาก Part 8 — ปิด Part ด้วย Lab เต็มรูปแบบที่พิสูจน์ว่า
PC1 (VLAN10) และ PC-WIFI (VLAN40) รับ IP Address ได้จริงข้าม VLAN ทั่วทั้ง Topology ✅

**ไปต่อ:** [Part 17 — NTP, Syslog & SNMP →](part-017-ntp-syslog-snmp.md)
