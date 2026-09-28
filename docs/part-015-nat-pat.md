# Part 15 — NAT / PAT (Network Address Translation)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 141–150 จาก 1000**

> ย้อนกลับไป [Part 14 — Access Control Lists](part-014-access-control-lists.md) ที่เราใช้ ACL
> ควบคุมว่า Traffic ใดผ่านได้/ไม่ได้ ใน Part นี้เราจะนำ ACL แบบเดิมกลับมาใช้ใหม่ในบทบาทที่ต่างออกไป
> — เป็นตัวเลือก (selector) ว่า Traffic ชุดใดจะถูก **แปลง Address (Translate)** ก่อนออกอินเทอร์เน็ต
> เนื้อหาหลักของ Part นี้คือ **NAT (Network Address Translation)** และ **PAT (Port Address
> Translation)** บน `WAN-EDGE-1` ซึ่งเป็นจุดที่ Private Address ของทุก VLAN (10/20/30/40/99)
> ถูกแปลงเป็น Public Address เพื่อออกสู่ Internet ผ่าน ISP-RTR รวมถึงการเปิดบริการ Server1
> (Web Server ภายใน VLAN 30) ให้โลกภายนอกเข้าถึงได้ผ่าน Static NAT

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 141 | NAT คืออะไร ทำไมต้องมี NAT และคำศัพท์ Inside/Outside Local/Global |
| 142 | Static NAT — Mapping แบบ 1:1 ตายตัว (`ip nat inside source static`) |
| 143 | Dynamic NAT — Mapping แบบ Pool (`ip nat pool`, `ip nat inside source list ... pool`) |
| 144 | PAT / NAT Overload — 1 Public IP รองรับหลาย Host พร้อมกัน |
| 145 | การกำหนด Inside/Outside Interface (`ip nat inside` / `ip nat outside`) |
| 146 | NAT Order of Operations — ความสัมพันธ์กับ Routing และ ACL |
| 147 | Port Forwarding — Static NAT พร้อม Port Translation |
| 148 | ACL สำหรับ NAT vs ACL สำหรับ Security — อย่าสับสนบทบาท |
| 149 | Verification & Troubleshooting NAT (`show ip nat translations`, `debug ip nat`) |
| 150 | Lab เต็มรูปแบบ: Static NAT + PAT บน WAN-EDGE-1 พร้อม Verification |

---

## Step 141 — NAT คืออะไร ทำไมต้องมี NAT และคำศัพท์สำคัญ

### 141.1 ปัญหาที่ NAT แก้ไข

IPv4 มีขนาด 32 bit รองรับได้ประมาณ 4.3 พันล้าน Address เท่านั้น ในขณะที่จำนวนอุปกรณ์ที่ต้องต่อ
อินเทอร์เน็ตทั่วโลกมีมากกว่านั้นหลายเท่า ปัญหานี้เรียกว่า **IPv4 Address Exhaustion**

วิธีแก้ที่ IETF กำหนดไว้ตั้งแต่ปี 1996 คือ:

1. **RFC 1918 — Private Addressing**: กันช่วง Address ส่วนหนึ่งไว้ใช้ภายในองค์กรเท่านั้น
   (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) — องค์กรนับล้านแห่งสามารถใช้ช่วงเดียวกันซ้ำกันได้
   โดยไม่ชนกัน เพราะ Address เหล่านี้ **ไม่ถูก Route บน Internet สาธารณะเลย**
2. **RFC 3022 — NAT (Network Address Translation)**: อุปกรณ์ที่อยู่ขอบเขตระหว่างเครือข่ายภายใน (Private)
   กับ Internet (Public) จะทำหน้าที่ **แปลง Source/Destination IP Address** ของ Packet ที่ผ่านไปมา
   เพื่อให้ Private Address ภายในสามารถ "ปลอมตัว" เป็น Public Address เวลาสื่อสารกับ Internet

ในหลักสูตรนี้ ทุก VLAN (`10.10.10.0/24` SALES, `10.10.20.0/24` VOICE, `10.10.30.0/24` SERVERS,
`10.10.40.0/24` WIFI, `10.10.99.0/24` MGMT) เป็น RFC 1918 Private Address ทั้งหมด — **จำเป็นต้องผ่าน
NAT ที่ `WAN-EDGE-1` เสมอ** ก่อนจะไปถึง `ISP-RTR` และ Internet

### 141.2 ตำแหน่งของ NAT ใน Topology

```
   ┌────────────┐       ┌────────────┐       ┌────────────┐        ┌────────────┐
   │ VLAN 10/20 │       │            │       │            │        │            │
   │ 30/40/99   │──────►│  CORE-SW1  │──────►│ WAN-EDGE-1 │═══════►│  ISP-RTR   │───► Internet
   │  (Private) │       │            │       │  (ทำ NAT)   │        │  (Public)  │
   └────────────┘       └────────────┘       └────────────┘        └────────────┘
   10.10.x.0/24                              inside            outside
                                              10.10.254.1        203.0.113.2
                                              (ฝั่ง CORE)         (ฝั่ง ISP)
```

`WAN-EDGE-1` คือ NAT Boundary ของทั้งองค์กร — ฝั่ง LAN (CORE-SW1 และ VLAN ทั้งหมด) เป็น **Inside**,
ฝั่ง `ISP-RTR` (ผ่าน `203.0.113.0/30`) เป็น **Outside**

> **อ้างอิงจาก IP Address Plan**: Link `ISP-RTR ↔ WAN-EDGE-1` คือ `203.0.113.0/30` โดย
> `WAN-EDGE-1` = `203.0.113.2` และ `ISP-RTR` = `203.0.113.1` — นี่คือ Public-facing Interface
> ของทั้งองค์กร (จำลองเป็น Internet Edge) นอกจากนี้ ISP ยังจัดสรร Public Block เพิ่มให้
> `WAN-EDGE-1` อีกก้อนหนึ่งคือ `203.0.113.8/29` (usable: `.9`–`.14`) สำหรับใช้ทำ **Static NAT
> แบบ 1:1** ให้กับ Server ภายในที่ต้องเปิดให้ Internet เข้าถึงได้ (เช่น Web Server) — ก้อนนี้ไม่ใช่
> Address ของ Interface ใดๆ แต่เป็น "NAT Pool" หรือ "Static Mapping Pool" ที่ `WAN-EDGE-1` เป็น
> เจ้าของ (ISP ต้อง Route Block นี้มาให้ WAN-EDGE-1 ด้วย Static Route หรือ BGP ในสภาพแวดล้อมจริง)

### 141.3 คำศัพท์ NAT ทั้ง 4 แบบ — ต้องเข้าใจให้แม่นเพราะ Cisco ใช้คำนี้ตลอด

Cisco แบ่งมุมมอง IP Address ออกเป็น 4 ประเภทตามว่า "อยู่ฝั่งไหน" (Inside/Outside) และ
"มองจากมุมไหน" (Local = มุมมองภายในเครือข่าย, Global = มุมมองจาก Internet)

| คำศัพท์ | คำนิยาม |
|---|---|
| **Inside Local** | IP Address ของ Host ภายใน **ตามที่ Host นั้นถูก config ไว้จริง** (มักเป็น Private/RFC1918) |
| **Inside Global** | IP Address ของ Host ภายใน **ตามที่โลกภายนอก (Internet) มองเห็น** (Public Address ที่ถูกแปลงมา) |
| **Outside Local** | IP Address ของ Host ภายนอก **ตามที่เครือข่ายภายในมองเห็น** (ปกติเหมือน Outside Global เพราะไม่ได้แปลง) |
| **Outside Global** | IP Address ของ Host ภายนอก **ตามที่ถูก config ไว้จริงบน Internet** (Public Address จริง) |

**เทคนิคจำ**: อ่านจากหลังมาหน้า — "Local" คือมุมมองจาก **ฝั่งเดียวกับที่ยืนอยู่ (ภายใน)**
"Global" คือมุมมองจาก **ฝั่งอินเทอร์เน็ตทั้งใบ** ส่วนคำแรก (Inside/Outside) บอกว่า Host
นั้นเป็นของฝั่งใน หรือฝั่งนอกกันแน่ (ไม่เกี่ยวกับว่ากำลังมองจากตรงไหน)

### 141.4 Worked Packet-Flow Diagram

สมมติ `PC1` ใน VLAN 10 (`10.10.10.50`) เปิดเว็บไปที่ Server ปลายทางบน Internet ที่ IP
`198.51.100.20` ผ่าน PAT บน `WAN-EDGE-1` (Public IP ของ WAN-EDGE-1 คือ `203.0.113.2`)

```
ขาไป (PC1 -> Internet Server)
┌──────────┐  Src: 10.10.10.50      ┌────────────┐  Src: 203.0.113.2      ┌──────────────┐
│   PC1    │  Dst: 198.51.100.20    │ WAN-EDGE-1 │  Dst: 198.51.100.20    │ Internet Srv │
│          │───────────────────────►│  (NAT here) │───────────────────────►│ 198.51.100.20│
└──────────┘  (ก่อนแปลง)             └────────────┘  (หลังแปลง Source)      └──────────────┘

ขากลับ (Internet Server -> PC1)
┌──────────┐  Src: 198.51.100.20    ┌────────────┐  Src: 198.51.100.20    ┌──────────────┐
│   PC1    │  Dst: 10.10.10.50      │ WAN-EDGE-1 │  Dst: 203.0.113.2      │ Internet Srv │
│          │◄───────────────────────│  (NAT here) │◄───────────────────────│ 198.51.100.20│
└──────────┘  (หลังแปลงกลับ)         └────────────┘  (ก่อนแปลง Destination) └──────────────┘
```

เทียบกับคำศัพท์ 4 แบบในตารางข้างบน สำหรับ Session นี้:

| Address | ค่า | ประเภท |
|---|---|---|
| IP ของ PC1 ตามที่ config จริง | `10.10.10.50` | **Inside Local** |
| IP ของ PC1 ที่โลกภายนอกมองเห็น | `203.0.113.2` (หรือ port เฉพาะถ้าเป็น PAT) | **Inside Global** |
| IP ของ Internet Server ตามที่ PC1 มองเห็น | `198.51.100.20` | **Outside Local** |
| IP ของ Internet Server ตามที่ config จริงบน Internet | `198.51.100.20` | **Outside Global** |

สังเกตว่า **Outside Local = Outside Global** ในกรณีนี้ เพราะ `WAN-EDGE-1` ไม่ได้แปลง Destination
Address เลย (ไม่มี NAT ฝั่ง Outside ในเคสนี้) — นี่คือกรณีปกติที่พบใน 90% ของ Lab จริง
Outside NAT (การแปลง Outside Local ≠ Outside Global) เป็นกรณีพิเศษที่ใช้ตอนมี Overlapping Address
ระหว่างสอง Site ซึ่งจะกล่าวถึงในระดับ CCNP

---

## Step 142 — Static NAT: Mapping แบบ 1:1 ตายตัว

### 142.1 แนวคิด

**Static NAT** คือการ Map **1 Inside Local IP ↔ 1 Inside Global IP แบบตายตัวถาวร** ไม่ว่าจะมี
Traffic ไหลผ่านหรือไม่ Mapping นี้จะยังคงอยู่ใน NAT Table เสมอ (ต่างจาก Dynamic NAT/PAT ที่สร้าง
Entry แบบชั่วคราวตาม Traffic จริง)

**ใช้เมื่อไร**: เมื่อต้องมี Host ภายในที่ต้องถูก "เข้าถึงจากภายนอก (Inbound)" ด้วย IP Address เดิม
ตลอดเวลา เช่น Web Server, Mail Server, VPN Gateway — เพราะ Client ภายนอกต้องรู้ Public IP
คงที่ล่วงหน้าเพื่อเชื่อมต่อเข้ามา

### 142.2 Syntax

```
Router(config)# ip nat inside source static <inside-local-ip> <inside-global-ip>
```

### 142.3 Worked Example — Map Server1 (10.10.30.10) เป็น Public IP 203.0.113.10

`Server1` อยู่ใน VLAN 30 (SERVERS, `10.10.30.0/24`) ที่ Address `10.10.30.10` เราต้องการให้
Internet เข้าถึง Web Service ของ Server1 ผ่าน Public IP `203.0.113.10` (มาจากก้อน
`203.0.113.8/29` ที่ WAN-EDGE-1 ถือไว้สำหรับ Static NAT)

```
WAN-EDGE-1(config)# ip nat inside source static 10.10.30.10 203.0.113.10
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# description LAN-facing to CORE-SW1
WAN-EDGE-1(config-if)# ip nat inside
WAN-EDGE-1(config-if)# exit
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description WAN-facing to ISP-RTR
WAN-EDGE-1(config-if)# ip nat outside
WAN-EDGE-1(config-if)# exit
```

> หมายเหตุ: การ mark `ip nat inside` / `ip nat outside` เป็นเงื่อนไข**บังคับ** — ถ้าไม่ทำ
> NAT statement ทั้งหมดจะไม่ทำงานเลย จะอธิบายรายละเอียดใน Step 145

### 142.4 ตรวจสอบผลลัพธ์

```
WAN-EDGE-1# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
--- 203.0.113.10       10.10.30.10        ---                ---
```

- `Inside local` = `10.10.30.10` (Server1 ตามที่ config จริง)
- `Inside global` = `203.0.113.10` (ตามที่ Internet มองเห็น)
- คอลัมน์ `Outside local/global` เป็น `---` เพราะ Entry นี้คือ Static NAT ที่ยังไม่มี Session
  จริงเชื่อมต่ออยู่ (ไม่ผูกกับ Destination ใดโดยเฉพาะ — Static NAT ทำงานได้กับทุก Outside Host
  ที่พยายามติดต่อมาที่ `203.0.113.10`)

### 142.5 ทำไม Static NAT ไม่ต้องรอ Traffic

Static NAT ถูกเขียนเข้า NAT Table ทันทีที่ใส่คำสั่ง (ไม่เหมือน Dynamic NAT ที่ Entry จะปรากฏใน
`show ip nat translations` เมื่อมี Traffic จริงไหลผ่านเท่านั้น) เพราะ Static NAT ต้องพร้อมรับ
Inbound Connection จากภายนอกได้ตลอดเวลาโดยไม่มีการ "trigger" จากฝั่ง Inside ก่อน

---

## Step 143 — Dynamic NAT: Mapping แบบ Pool

### 143.1 แนวคิด

**Dynamic NAT** คือการแปลง Inside Local เป็น Inside Global โดยเลือก Address จาก **Pool ของ
Public IP ที่กำหนดไว้ล่วงหน้า** — เป็นการ Map แบบ **1:1 แต่ไม่ตายตัว** (ต่างจาก Static NAT ที่
Map คงที่) กล่าวคือ Host A อาจได้ Public IP `.10` วันนี้ แต่พรุ่งนี้อาจได้ `.11` แทน ขึ้นอยู่กับว่า
Address ใดใน Pool ว่างอยู่ ณ ขณะนั้น

**ข้อจำกัดสำคัญ**: Dynamic NAT (แบบไม่มี Overload) ต้องมี **1 Public IP ต่อ 1 Session ที่ Active
พร้อมกัน** ถ้า Pool มี Public IP 5 ตัว แต่มี Host ภายใน 20 เครื่องพยายามออกอินเทอร์เน็ตพร้อมกัน
— มีเพียง 5 เครื่องแรกที่จะได้ Translation สำเร็จ ส่วนที่เหลือจะถูก **Drop** จนกว่า Pool
จะมี Address ว่าง (เมื่อ Session เดิมหมดอายุ/ปิดไป)

> ด้วยข้อจำกัดนี้ Dynamic NAT (ไม่มี Overload) จึงไม่เหมาะกับ Enterprise LAN ที่มี User จำนวนมาก
> — ในทางปฏิบัติสมัยใหม่แทบไม่มีใครใช้ Dynamic NAT แบบล้วนๆแล้ว ส่วนใหญ่ใช้ PAT (Step 144) แทน
> Dynamic NAT ยังพบใช้ในบางกรณีที่ต้องรักษา 1:1 mapping ชั่วคราวไว้ (เช่น ต้องการให้ Session
> คงเส้นทาง port ไม่ถูก multiplex กับ Host อื่น) หรือใช้เป็นความรู้พื้นฐานก่อนเรียน PAT

### 143.2 Syntax

```
Router(config)# ip nat pool <pool-name> <start-ip> <end-ip> netmask <mask>
Router(config)# ip access-list standard <acl-name>
Router(config-std-nacl)# permit <source-network> <wildcard>
Router(config)# ip nat inside source list <acl-name> pool <pool-name>
```

### 143.3 Worked Example — Dynamic NAT Pool สำหรับ VLAN 10 (สมมติ ก่อนจะเปลี่ยนไปใช้ PAT ใน Step 144)

สมมติเราต้องการให้ VLAN 10 (SALES, `10.10.10.0/24`) ใช้ Pool Public IP 4 ตัวจากก้อน
`203.0.113.8/29` (`.9`–`.14`) เพื่อสาธิต Dynamic NAT ก่อนเปลี่ยนไปใช้ PAT จริงในภาคปฏิบัติ:

```
WAN-EDGE-1(config)# ip nat pool SALES-POOL 203.0.113.9 203.0.113.12 netmask 255.255.255.248
WAN-EDGE-1(config)# ip access-list standard NAT-SALES-ACL
WAN-EDGE-1(config-std-nacl)# permit 10.10.10.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# exit
WAN-EDGE-1(config)# ip nat inside source list NAT-SALES-ACL pool SALES-POOL
```

### 143.4 ตรวจสอบเมื่อมี Traffic

```
WAN-EDGE-1# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
--- 203.0.113.9        10.10.10.51        ---                ---
--- 203.0.113.10       10.10.10.52        ---                ---
```

เมื่อ Pool เต็ม (Host ตัวที่ 5 ของ VLAN 10 พยายามออกอินเทอร์เน็ต) จะเห็น Counter ผิดปกติที่
`show ip nat statistics` ในช่อง `Pool stats drop` เพิ่มขึ้น (ดูรายละเอียดใน Step 149) — นี่คือ
สาเหตุหลักที่ Dynamic NAT ไม่มี Overload ไม่ถูกใช้จริงในองค์กรที่มี User มากกว่าจำนวน Public IP
ที่มี — ซึ่งนำไปสู่ PAT ใน Step ถัดไป

---

## Step 144 — PAT (Port Address Translation) / NAT Overload

### 144.1 แนวคิด

**PAT** (หรือเรียกอีกชื่อว่า **NAT Overload**) คือ Dynamic NAT ที่เพิ่มการแปลง **Port Number**
เข้าไปด้วย ทำให้ **Public IP เพียง 1 ตัว รองรับ Internal Host ได้หลายพันเครื่องพร้อมกัน**
เพราะ TCP/UDP มี Port Number ให้เลือกใช้ได้ถึง 65536 ค่า — Router จะจับคู่
`(Inside Local IP, Inside Local Port)` แต่ละคู่ให้ Map ไปยัง `(Inside Global IP, Inside Global
Port)` ที่ไม่ซ้ำกัน โดยใช้ **Inside Global IP ตัวเดียวกันซ้ำๆ แต่เปลี่ยน Port**

นี่คือรูปแบบ NAT ที่ใช้กันมากที่สุดในโลกจริง — บ้านทุกบ้าน, องค์กรทุกขนาดที่มี Public IP จำกัด
(ปกติมีแค่ 1 ตัวจาก ISP) ใช้ PAT ในการให้ User ภายในทั้งหมดออกอินเทอร์เน็ตพร้อมกัน

### 144.2 ตัวอย่าง Port Multiplexing

| Inside Local (IP:Port) | Inside Global (IP:Port) หลัง PAT |
|---|---|
| `10.10.10.51:52001` | `203.0.113.2:52001` |
| `10.10.20.60:52002` | `203.0.113.2:52002` |
| `10.10.30.10:52003` | `203.0.113.2:1024` (Router เลือก Port ใหม่ถ้าชนกัน) |
| `10.10.40.80:52001` | `203.0.113.2:52004` |

สังเกตว่า Inside Global IP เป็น `203.0.113.2` (IP ของ Interface ขาออกจริงของ WAN-EDGE-1)
**ตัวเดียวกันทุกแถว** — สิ่งที่ทำให้ Router แยกแต่ละ Session ออกจากกันได้คือ **Port Number**
ที่ต่างกัน ไม่ใช่ IP

### 144.3 Syntax

```
Router(config)# ip access-list standard <acl-name>
Router(config-std-nacl)# permit <source-network> <wildcard>
Router(config)# ip nat inside source list <acl-name> interface <outside-interface> overload
```

Keyword `overload` คือสิ่งที่เปลี่ยน Dynamic NAT ธรรมดาให้กลายเป็น PAT — และการใช้
`interface <outside-if>` (แทน `pool <name>`) หมายถึง "ใช้ IP Address ของ Interface ขาออกนั้น
เป็น Inside Global IP" ซึ่งเป็นรูปแบบที่ใช้บ่อยที่สุดเมื่อองค์กรมี Public IP แค่ 1 ตัว

### 144.4 Worked Example — PAT สำหรับทุก VLAN (10, 20, 30, 40) ผ่าน Public IP เดียวของ WAN-EDGE-1

```
WAN-EDGE-1(config)# ip access-list standard NAT-INTERNAL-ACL
WAN-EDGE-1(config-std-nacl)# remark Traffic ที่อนุญาตให้ PAT ออกอินเทอร์เน็ต
WAN-EDGE-1(config-std-nacl)# permit 10.10.10.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# permit 10.10.20.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# permit 10.10.30.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# permit 10.10.40.0 0.0.0.255
WAN-EDGE-1(config-std-nacl)# exit
WAN-EDGE-1(config)# ip nat inside source list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 overload
```

> **หมายเหตุการออกแบบ**: VLAN 99 (MGMT) มักถูก**แยกไว้ไม่ให้ออกอินเทอร์เน็ตโดยตรง** ด้วยเหตุผล
> Security (MGMT Network ควรเข้าถึงได้จากภายในองค์กรเท่านั้น ผ่าน VPN หรือ Jump Host) จึงไม่รวม
> `10.10.99.0/24` เข้าใน ACL นี้ — นี่คือตัวอย่างการออกแบบ NAT ที่สอดคล้องกับ Security Policy
> ขององค์กร ไม่ใช่แค่เรื่อง Technical

### 144.5 ทำไม PAT ไม่มีปัญหา "Pool หมด" แบบ Dynamic NAT

เพราะ Public IP 1 ตัว (พร้อม Port ~64,512 ค่าที่ใช้ได้จริงในช่วง Ephemeral) สามารถรองรับ Session
พร้อมกันได้หลายหมื่น Session — ในทางปฏิบัติ Enterprise ขนาดกลางจะไม่ชน Limit นี้เลย ยกเว้นกรณี
เฉพาะเช่น NAT Table เต็มจริงๆ (ดู Step 149 เรื่อง Translation Table Overflow)

---

## Step 145 — การกำหนด Inside/Outside Interface

### 145.1 ทำไมต้อง Mark Interface

คำสั่ง `ip nat inside source ...` เพียงอย่างเดียว **ยังไม่เพียงพอ** — Cisco IOS ต้องรู้ว่า
Interface ใดของ Router เป็นฝั่ง "ใน" (Inside) และ Interface ใดเป็นฝั่ง "นอก" (Outside) เพื่อ
ตัดสินใจว่า Packet ที่เข้ามาทาง Interface ไหนต้องถูกแปลง Address ไปในทิศทางไหน

```
Router(config-if)# ip nat inside     ! ใส่บน Interface ที่ต่อไปทาง LAN/Private Network
Router(config-if)# ip nat outside    ! ใส่บน Interface ที่ต่อไปทาง Internet/Public Network
```

### 145.2 การ Mark Interface บน WAN-EDGE-1

ตาม IP Address Plan, `WAN-EDGE-1` มี Interface หลักๆ ดังนี้:

| Interface | ต่อไปทาง | Subnet | Role |
|---|---|---|---|
| Gi0/0/0 | ISP-RTR | `203.0.113.0/30` (.2) | **Outside** |
| Gi0/0/1 | CORE-SW1 | `10.10.254.0/30` (.1) | **Inside** |
| Gi0/0/2 | WAN-EDGE-2 (cross-link) | `10.10.254.8/30` (.9) | **Inside** (ยังเป็นเครือข่ายภายในองค์กร) |

```
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description ** OUTSIDE ** to ISP-RTR
WAN-EDGE-1(config-if)# ip address 203.0.113.2 255.255.255.252
WAN-EDGE-1(config-if)# ip nat outside
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# description ** INSIDE ** to CORE-SW1
WAN-EDGE-1(config-if)# ip address 10.10.254.1 255.255.255.252
WAN-EDGE-1(config-if)# ip nat inside
WAN-EDGE-1(config-if)# exit
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/2
WAN-EDGE-1(config-if)# description ** INSIDE ** cross-link to WAN-EDGE-2
WAN-EDGE-1(config-if)# ip address 10.10.254.9 255.255.255.252
WAN-EDGE-1(config-if)# ip nat inside
WAN-EDGE-1(config-if)# exit
```

### 145.3 ตรวจสอบว่า Interface ถูก Mark ครบหรือยัง

```
WAN-EDGE-1# show ip nat statistics
Total translations: 4 (0 static, 4 dynamic; 4 extended)
Outside interfaces:
  GigabitEthernet0/0/0
Inside interfaces:
  GigabitEthernet0/0/1, GigabitEthernet0/0/2
Hits: 15234  Misses: 3
...
```

### 145.4 ข้อผิดพลาดที่พบบ่อยที่สุด: ลืม Mark Interface

**อาการ**: ใส่คำสั่ง `ip nat inside source ...` ครบถูกต้องทุกอย่าง แต่ Ping/Traffic จาก Internal
Host ไปอินเทอร์เน็ต **ไม่ผ่าน** หรือ `show ip nat translations` **ไม่มี Entry ปรากฏขึ้นเลย**
แม้จะมี Traffic วิ่งอยู่จริง

**สาเหตุที่พบบ่อยที่สุด**:
1. ลืมใส่ `ip nat inside` บน Interface ฝั่ง LAN (หรือใส่ผิด Interface เช่น ใส่ที่ Sub-interface
   ผิดตัวในกรณี Router-on-a-Stick)
2. ลืมใส่ `ip nat outside` บน Interface ฝั่ง WAN
3. ใส่ทั้งสอง Interface เป็น `ip nat inside` เหมือนกัน (สลับ Logic) — Cisco IOS จะไม่ Error
   ตอน config แต่ NAT จะไม่ทำงานตามที่ตั้งใจ

**วิธี Verify อย่างเร็ว**: ใช้ `show ip nat statistics` เพื่อดูว่า Interface ใดถูกจัดอยู่ใน
"Inside interfaces" และ "Outside interfaces" ตรงกับที่ตั้งใจไว้หรือไม่ — ถ้า Section ใด Section
หนึ่งว่างเปล่า แสดงว่ายังไม่ได้ Mark เลย

---

## Step 146 — NAT Order of Operations: ความสัมพันธ์กับ Routing และ ACL

### 146.1 หลักการสำคัญ: NAT เกิด "หลัง Routing Decision" เสมอสำหรับ Traffic ขาออก

Cisco IOS ประมวลผล Packet ตามลำดับ (Order of Operations) ที่ตายตัว จุดที่สำคัญที่สุดสำหรับ
บทนี้คือ:

```
Traffic ขาออก (Inside -> Outside):
  Packet เข้า Inside Interface
        │
        ▼
  [Inbound ACL บน Inside Interface (ถ้ามี)]
        │
        ▼
  [Routing Decision — เลือก Exit Interface จาก Routing Table]  <-- (1) Routing เกิดก่อน
        │
        ▼
  [NAT Translation — แปลง Source IP เป็น Inside Global]         <-- (2) NAT เกิดหลัง Routing
        │
        ▼
  [Outbound ACL บน Outside Interface (ถ้ามี)]
        │
        ▼
  ส่งออกทาง Outside Interface


Traffic ขาเข้า (Outside -> Inside):
  Packet เข้า Outside Interface
        │
        ▼
  [Inbound ACL บน Outside Interface (ถ้ามี)]
        │
        ▼
  [NAT Translation — แปลง Destination IP กลับเป็น Inside Local]  <-- (1) NAT เกิดก่อน Routing
        │
        ▼
  [Routing Decision — เลือก Exit Interface ไปยัง Inside Local IP]  <-- (2) Routing เกิดหลัง NAT
        │
        ▼
  [Outbound ACL บน Inside Interface (ถ้ามี)]
        │
        ▼
  ส่งออกทาง Inside Interface
```

**สรุปเป็นหลักจำง่าย**:

| ทิศทาง | ลำดับ |
|---|---|
| Inside → Outside (ขาออก) | Routing ก่อน → NAT ทีหลัง (**Route-then-Translate**) |
| Outside → Inside (ขาเข้า) | NAT ก่อน → Routing ทีหลัง (**Translate-then-Route**) |

เหตุผลของลำดับนี้: Router ต้อง**รู้ก่อนว่าจะส่ง Packet ออกทาง Interface ไหน** เพื่อเลือกว่าจะแปลง
เป็น Inside Global IP ตัวไหน (ในกรณีมีหลาย Outside Interface/Pool) ส่วนขาเข้า Router ต้อง
**แปลง Destination กลับเป็น Inside Local ก่อน** จึงจะรู้ว่าต้อง Route ไปทาง Interface ไหนภายใน
LAN (เพราะ Routing Table ไม่มี Entry สำหรับ Public IP `203.0.113.10` เจาะจงไปที่ Server1 เลย
มีแต่ Entry ของ Local Subnet `10.10.30.0/24`)

### 146.2 ผลกระทบต่อการ Troubleshoot

1. **ACL ที่ apply บน Inside Interface ทิศทาง `in`** จะเห็น Packet ที่ **ยังไม่ถูก NAT**
   (เป็น Source = Inside Local IP เช่น `10.10.30.10`)
2. **ACL ที่ apply บน Outside Interface ทิศทาง `out`** จะเห็น Packet ที่ **ถูก NAT แล้ว**
   (เป็น Source = Inside Global IP เช่น `203.0.113.2`)
3. เมื่อเขียน ACL เพื่อกรอง Traffic (Security ACL ตาม Part 14) ต้องคิดให้ถูกว่า ณ จุดที่ ACL
   นั้น apply อยู่ — Packet ถูกแปลง Address ไปแล้วหรือยัง ถ้าคิดผิดจุด ACL จะ Match ผิด Address
   และดูเหมือนกฎไม่ทำงาน (ทั้งที่ Syntax ถูกทุกอย่าง)
4. คำสั่ง `debug ip nat` และ `debug ip packet` (ระมัดระวังการใช้บน Production เพราะกิน CPU
   มาก) ช่วยยืนยันว่า ณ จุดใดของ Path Packet มี Address เป็นอะไรกันแน่

### 146.3 ตัวอย่างที่มักทำให้งง

สมมติ Security ACL (Part 14 style) ต้องการบล็อกไม่ให้ VLAN 40 (WIFI) เข้าถึง VLAN 30 (SERVERS)
โดยตรง — ACL แบบนี้ apply บน Interface **ภายใน LAN** (เช่น SVI บน DIST-SW) ซึ่งไม่มี NAT
เกี่ยวข้องเลย เพราะ Traffic ระหว่าง VLAN ภายในองค์กรไม่เคยผ่าน NAT Boundary (`WAN-EDGE-1`)
— นี่คือเหตุผลที่ต้องแยกให้ชัดว่า ACL ตัวไหนอยู่ "ก่อน NAT Boundary" (Internal Security ACL)
และ ACL ตัวไหนอยู่ "ที่ NAT Boundary" (NAT Selector ACL หรือ Internet Edge Security ACL)
รายละเอียดการแยกบทบาท ACL ทั้งสองแบบนี้จะอธิบายเจาะจงใน Step 148

---

## Step 147 — Port Forwarding: Static NAT พร้อม Port Translation

### 147.1 แนวคิด

Static NAT ปกติ (Step 142) แปลงทั้ง IP Address โดยไม่แตะ Port — หมายความว่า Public IP นั้น
ถูก "ยึด" ไว้ทั้งตัวสำหรับ Internal Host เดียว ไม่สามารถใช้ Public IP ตัวเดียวกันกับบริการอื่นได้

**Port Forwarding (Static NAT with Port Translation)** แก้ปัญหานี้โดยระบุ **Protocol +
Port** ลงไปในคำสั่ง Static NAT ด้วย ทำให้:

1. เปิดเผยเฉพาะ Port ที่ต้องการ (เช่น 443 สำหรับ HTTPS) ไม่ใช่ทุก Port ของ Host นั้น
   (**Security ดีกว่า** Static NAT ธรรมดา)
2. **ใช้ Public IP ตัวเดียวกันได้กับหลาย Internal Host** โดยแยกด้วย Port ต่างกัน — เช่น
   `203.0.113.10:443` ไป Server1 แต่ `203.0.113.10:22` ไป Server2 (ถ้าต้องการ)

### 147.2 Syntax

```
Router(config)# ip nat inside source static tcp <inside-local-ip> <inside-port> \
                <inside-global-ip> <global-port> extendable
```

Keyword `extendable` จำเป็นเมื่อมี Static NAT Entry มากกว่าหนึ่งตัวที่อาจดูเหมือนซ้ำกัน
(เช่น ต่างกันแค่ Port) — IOS ใช้ Flag นี้เพื่อแยกความกำกวมของ Translation Table internally

### 147.3 Worked Example — เปิดเฉพาะ HTTPS (443) ของ Server1 ให้ Internet เข้าถึง

ต้องการเปิดเฉพาะ Port 443 (HTTPS) ของ `Server1` (`10.10.30.10`) ให้ Internet เข้าถึงผ่าน
`203.0.113.10:443` โดย**ไม่เปิด Port อื่นเลย** (ปิด SSH, RDP, หรือ Port อื่นทั้งหมดของ Server1
จากภายนอก):

```
WAN-EDGE-1(config)# no ip nat inside source static 10.10.30.10 203.0.113.10
WAN-EDGE-1(config)# ip nat inside source static tcp 10.10.30.10 443 203.0.113.10 443 extendable
```

> บรรทัดแรก `no ip nat ...` คือการถอน Static NAT แบบเดิม (Whole-IP) ที่ทำไว้ใน Step 142
> ออกก่อน เพื่อไม่ให้ชนกับ Port-based Static NAT ตัวใหม่ที่แคบกว่า — ใน Lab จริงควรเลือกใช้
> แบบใดแบบหนึ่งให้สอดคล้องกับ Security Requirement ขององค์กร (Whole-IP NAT เหมาะกับ Server ที่
> ต้องเปิดทุก Port เช่น VPN Concentrator, ส่วน Port-based NAT เหมาะกับ Web/App Server ที่ควร
> เปิดเฉพาะ Port ที่จำเป็น)

### 147.4 เสริม Security ด้วย ACL ที่ Outside Interface (Best Practice)

Port Forwarding เพียงอย่างเดียวยังไม่ได้จำกัดว่า **ใคร** จากภายนอกเข้าถึงได้ — ควรใส่ Extended
ACL (ตาม Part 14) ที่ Outside Interface ร่วมด้วยเพื่อควบคุมอีกชั้น (Defense in Depth):

```
WAN-EDGE-1(config)# ip access-list extended OUTSIDE-IN-ACL
WAN-EDGE-1(config-ext-nacl)# permit tcp any host 203.0.113.10 eq 443
WAN-EDGE-1(config-ext-nacl)# deny ip any host 203.0.113.10
WAN-EDGE-1(config-ext-nacl)# permit ip any any
WAN-EDGE-1(config-ext-nacl)# exit
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# ip access-group OUTSIDE-IN-ACL in
WAN-EDGE-1(config-if)# exit
```

ACL นี้ทำงานที่ **Outside Global Address** (`203.0.113.10`) เพราะ apply ที่ Outside Interface
ทิศทาง `in` — ตาม Order of Operations ใน Step 146 คือ **ก่อน NAT แปลง Destination กลับเป็น
Inside Local** ดังนั้น ACL ต้อง Match กับ Public IP (`203.0.113.10`) ไม่ใช่ Private IP
(`10.10.30.10`)

### 147.5 ตรวจสอบ

```
WAN-EDGE-1# show ip nat translations
Pro Inside global          Inside local         Outside local        Outside global
tcp 203.0.113.10:443       10.10.30.10:443      ---                  ---
```

---

## Step 148 — NAT ACL vs Security ACL: อย่าสับสนบทบาท

### 148.1 สองบทบาทที่แตกต่างกันโดยสิ้นเชิงของ ACL

Part 14 สอน ACL ในบทบาท **Security/Filtering** (permit/deny ว่า Traffic ผ่านได้หรือไม่)
Part นี้ใช้ ACL ในบทบาทที่ต่างออกไปโดยสิ้นเชิง — เป็น **Selector/Classifier** สำหรับ NAT ว่า
Traffic กลุ่มใดจะถูกนำไปแปลง Address ผ่าน `ip nat inside source list <acl> ...`

| หัวข้อเปรียบเทียบ | Security ACL (Part 14) | NAT Selector ACL (Part นี้) |
|---|---|---|
| ผลของ `permit` | ปล่อยให้ Traffic ผ่าน | นำ Traffic ไปแปลง Address (Translate) |
| ผลของ `deny` | บล็อก Traffic ทันที (drop) | **ไม่แปลง Address** — Traffic ยังคงถูกส่งต่อไปตามปกติถ้าไม่มี ACL อื่นบล็อกไว้ (ไม่ถูก drop โดย NAT ACL) |
| Apply ที่ไหน | `ip access-group <acl> in/out` บน Interface | อ้างอิงใน `ip nat inside source list <acl> ...` (ไม่ apply ตรงบน Interface) |
| Implicit deny ท้าย ACL | มีผล บล็อก Traffic ที่ไม่ Match | มีผลแค่ "ไม่ต้องแปลง Address" ให้ Traffic นั้น — Traffic นั้นไม่ได้ถูก Drop จาก ACL นี้ |

> **จุดที่ทำให้วิศวกรสับสนบ่อยที่สุด**: เห็น `deny` ใน NAT ACL แล้วเข้าใจผิดว่า Traffic นั้นถูก
> บล็อกไปเลย ทั้งที่ความจริงคือ Traffic นั้นแค่ **"ไม่ถูก PAT/NAT"** เท่านั้น — ถ้าไม่มี Security
> ACL อื่นมาบล็อกซ้ำ Traffic นั้นจะยังถูกส่งออกไปด้วย Source IP เดิม (Private Address) ซึ่งจะถูก
> Router ปลายทางบน Internet Drop เพราะ Private Address ไม่ถูก Route บน Internet อยู่แล้ว
> (ผลลัพธ์ปลายทางคล้าย "ถูกบล็อก" แต่สาเหตุจริงคือ Routing ไม่ใช่ ACL)

### 148.2 ตัวอย่างการใช้ `deny` ใน NAT ACL อย่างมีเจตนา

กรณีจริงที่ใช้ `deny` ใน NAT Selector ACL คือเมื่อมี Traffic บางประเภทที่**ไม่ควรถูก PAT**
เช่น Traffic ระหว่าง Site-to-Site VPN (ที่ต้องคง Source เป็น Private IP เดิมเพื่อให้ Encrypt
ถูกก่อน NAT) — เทคนิคนี้เรียกว่า **NAT Exemption**

```
WAN-EDGE-1(config)# ip access-list extended NAT-INTERNAL-ACL-EXT
WAN-EDGE-1(config-ext-nacl)# remark ยกเว้น Traffic ไป Site อื่นผ่าน VPN ไม่ให้ถูก PAT
WAN-EDGE-1(config-ext-nacl)# deny ip 10.10.0.0 0.0.255.255 10.20.0.0 0.0.255.255
WAN-EDGE-1(config-ext-nacl)# permit ip 10.10.10.0 0.0.0.255 any
WAN-EDGE-1(config-ext-nacl)# permit ip 10.10.20.0 0.0.0.255 any
WAN-EDGE-1(config-ext-nacl)# permit ip 10.10.30.0 0.0.0.255 any
WAN-EDGE-1(config-ext-nacl)# permit ip 10.10.40.0 0.0.0.255 any
WAN-EDGE-1(config)# ip nat inside source list NAT-INTERNAL-ACL-EXT interface GigabitEthernet0/0/0 overload
```

Traffic ที่ปลายทางเป็น `10.20.0.0/16` (สมมติเป็น Remote Site ผ่าน VPN ในอนาคต) จะ Match บรรทัด
`deny` แถวแรก → **ไม่ถูก PAT** และถูกส่งไปตาม Routing/Crypto Map ปกติด้วย Source เดิม
ส่วน Traffic อื่นที่ไปอินเทอร์เน็ตทั่วไป (`permit ip any`) จะถูก PAT ตามปกติ — Site-to-Site VPN
จะกล่าวถึงโดยละเอียดใน Part ที่เกี่ยวกับ IPsec VPN ของหลักสูตรระดับ CCNP

### 148.3 กฎการออกแบบที่ควรยึดถือ

1. **แยกชื่อ ACL ให้ชัดเจนตามบทบาท** เช่น `NAT-*` สำหรับ NAT Selector, `SEC-*` หรือ
   `*-IN`/`*-OUT` สำหรับ Security ACL — ป้องกันความสับสนตอน Troubleshoot ทีมงานหลายคน
2. **ห้ามใช้ ACL ตัวเดียวกันทำสองบทบาท** แม้ Syntax จะอนุญาตให้ทำได้ก็ตาม เพราะการแก้ไข ACL
   เพื่อวัตถุประสงค์หนึ่งจะกระทบอีกวัตถุประสงค์โดยไม่ตั้งใจ
3. Standard ACL เพียงพอสำหรับ NAT Selector ส่วนใหญ่ (Match แค่ Source) แต่ถ้าต้องเจาะจงถึง
   Destination ด้วย (เช่น NAT Exemption ข้างบน) ต้องใช้ Extended ACL

---

## Step 149 — Verification & Troubleshooting NAT

### 149.1 คำสั่งหลักที่ต้องใช้เป็นประจำ

| คำสั่ง | ใช้ทำอะไร |
|---|---|
| `show ip nat translations` | แสดง NAT Table ปัจจุบันทั้งหมด (Static + Dynamic + PAT) |
| `show ip nat translations verbose` | เหมือนข้างบน แต่โชว์รายละเอียดเพิ่ม เช่น Timeout ที่เหลือของแต่ละ Entry |
| `show ip nat statistics` | สรุปจำนวน Translation, Inside/Outside Interface, Hit/Miss Counter |
| `clear ip nat translation *` | ล้าง Dynamic NAT Entry ทั้งหมด (Static NAT ไม่ถูกลบ) — ใช้ Force ให้สร้าง Session ใหม่ |
| `clear ip nat translation <ip>` | ล้าง Entry เฉพาะของ IP นั้น |
| `debug ip nat` | Real-time debug แสดง Packet ที่ถูก NAT ทุกตัว (ใช้ระมัดระวังบน Production — กิน CPU) |
| `debug ip nat detailed` | เหมือนข้างบนแต่แสดงรายละเอียดมากขึ้น (เช่น เหตุผลที่ NAT ไม่ทำงาน) |

### 149.2 ตัวอย่าง `show ip nat translations` ที่มี Session จริง

```
WAN-EDGE-1# show ip nat translations
Pro Inside global          Inside local         Outside local        Outside global
tcp 203.0.113.10:443       10.10.30.10:443      ---                  ---
tcp 203.0.113.2:52011      10.10.10.51:52011     198.51.100.20:443    198.51.100.20:443
udp 203.0.113.2:52012      10.10.20.60:52012     8.8.8.8:53           8.8.8.8:53
tcp 203.0.113.2:52013      10.10.40.80:52013     93.184.216.34:80     93.184.216.34:80
```

### 149.3 ตัวอย่าง `show ip nat statistics`

```
WAN-EDGE-1# show ip nat statistics
Total translations: 4 (1 static, 3 dynamic; 3 extended)
Peak translations: 812, occurred 02:14:07 ago
Outside interfaces:
  GigabitEthernet0/0/0
Inside interfaces:
  GigabitEthernet0/0/1, GigabitEthernet0/0/2
Hits: 48221  Misses: 0
CEF Translated packets: 48221, CEF Punted packets: 0
Expired translations: 3980
Dynamic mappings:
-- Inside Source
[Id: 1] access-list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 refcount 3
Queued Packets: 0
```

**ค่าที่ต้องดูเป็นประจำ**:
- `Misses` ที่สูงผิดปกติเทียบกับ `Hits` → อาจแปลว่า ACL ที่ใช้เป็น NAT Selector ไม่ครอบคลุม
  Traffic จริงบางส่วน (Traffic นั้นเข้ามาแต่ไม่ Match ACL จึงไม่ถูกแปลง)
- `Total translations` ที่โตขึ้นเรื่อยๆไม่มีลด → อาจมี Session ที่ไม่ปิดตัวเอง (เช่น UDP ที่ไม่มี
  FIN/RST เหมือน TCP) หรือ Timeout ตั้งไว้นานเกินไป

### 149.4 ปัญหาที่พบบ่อยและวิธีแก้

| ปัญหา | อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|---|
| **Asymmetric Routing** | NAT Session ถูกสร้างที่ Router A แต่ Traffic ขากลับวิ่งผ่าน Router B (เช่น มี WAN-EDGE-2 เป็น Path สำรอง) — Router B ไม่มี NAT State ของ Session นั้น จึงไม่รู้จะแปลง Destination กลับเป็น Inside Local อะไร | Packet ขากลับถูก Drop เพราะไม่มี NAT Entry ที่ตรงกัน | ต้องออกแบบ Routing ให้ Traffic ไป-กลับผ่าน NAT Device ตัวเดียวกันเสมอ (Symmetric Path) หรือใช้ NAT บนทั้งสอง Router พร้อม State Sync ในสภาพแวดล้อมที่ต้องการ Redundancy จริง |
| **Translation Table เต็ม** | Host ใหม่ไม่สามารถออกอินเทอร์เน็ตได้ ทั้งที่ Config ถูกต้อง, `show ip nat statistics` แสดง `Total translations` สูงมาก | Session เก่าไม่ถูกเคลียร์ (Timeout ยาวเกินไป, หรือมี Application ที่เปิด Connection จำนวนมากผิดปกติ) | ปรับ Timeout ด้วย `ip nat translation timeout <sec>` (Default 86400s สำหรับ TCP ทั่วไป, `ip nat translation tcp-timeout` เฉพาะ TCP), หรือ `clear ip nat translation *` เพื่อบังคับล้าง (ระวังจะตัด Session ที่ Active อยู่จริงด้วย) |
| **Dynamic NAT Pool หมด** | Host บางเครื่องใน VLAN ที่ใช้ Dynamic NAT (ไม่ใช่ PAT) ออกอินเทอร์เน็ตไม่ได้เป็นบางเครื่อง | Pool มี Address น้อยกว่าจำนวน Host ที่ Active พร้อมกัน — ดู Step 143 | เปลี่ยนไปใช้ PAT (`overload`) แทน Dynamic NAT ล้วนๆ เพราะ PAT ไม่มีข้อจำกัดนี้ |
| **Overlapping NAT Pool** | Public IP Range ที่ใช้ใน NAT Pool ซ้อนทับกับ Range ที่ใช้งานจริงอยู่แล้ว (เช่น ซ้ำกับ Static NAT Entry อื่น หรือซ้ำกับ IP ของ Interface) | Config ผิดพลาดจากการวางแผน Address ไม่รอบคอบ | ตรวจสอบด้วย `show ip nat statistics` + `show running-config \| section ip nat` ให้แน่ใจว่า Static NAT, Dynamic Pool, และ Interface IP ไม่ทับกันเลย — ในหลักสูตรนี้เราแยกไว้ชัดแล้ว: Interface ใช้ `203.0.113.0/30`, Static NAT Pool ใช้ `203.0.113.8/29` |
| **ลืม mark inside/outside** | ไม่มี Entry ปรากฏใน NAT Table เลย แม้มี Traffic วิ่งจริง | ดู Step 145.4 | ตรวจสอบ `show ip nat statistics` ว่า Interface อยู่ใน Section ที่ถูกต้อง |

### 149.5 ตัวอย่าง `debug ip nat` เมื่อ PAT ทำงานถูกต้อง

```
WAN-EDGE-1# debug ip nat
IP NAT debugging is on
WAN-EDGE-1#
*Sep 26 10:12:03.221: NAT*: s=10.10.10.51->203.0.113.2, d=198.51.100.20 [1024]
*Sep 26 10:12:03.223: NAT*: s=198.51.100.20, d=203.0.113.2->10.10.10.51 [1024]
```

บรรทัดแรก (`s=...->...`) คือขาออก (Source ถูกแปลง), บรรทัดที่สอง (`d=...->...`) คือขากลับ
(Destination ถูกแปลงกลับ) — ถ้าเห็นแค่บรรทัดแรกแต่ไม่มีบรรทัดที่สองตามมา แสดงว่า Traffic ขากลับ
ไม่ได้วิ่งผ่าน Router ตัวนี้ (สงสัย Asymmetric Routing ตามตารางข้างบน) **ควรปิด debug ด้วย
`no debug ip nat` หรือ `undebug all` ทันทีหลังใช้งาน** เพราะกิน CPU สูงมากถ้ามี Traffic จำนวนมาก

---

## Step 150 — Lab เต็มรูปแบบ: Static NAT + PAT บน WAN-EDGE-1

### 150.1 โจทย์ของ Lab

1. VLAN ทั้งหมด (10 SALES, 20 VOICE, 30 SERVERS, 40 WIFI) ต้องออกอินเทอร์เน็ตได้ผ่าน PAT
   โดยใช้ IP ของ Interface ขาออก (`203.0.113.2`) ของ `WAN-EDGE-1` เพียงตัวเดียว
2. VLAN 99 (MGMT) **ห้าม** ออกอินเทอร์เน็ตโดยตรง (ตามหลัก Security Best Practice)
3. `Server1` (`10.10.30.10`) ต้องเปิดให้ Internet เข้าถึงบริการ **HTTPS (443) เท่านั้น** ผ่าน
   Public IP `203.0.113.10`
4. ต้อง Verify ด้วย `show ip nat translations`, ทดสอบ Trace จาก Internal Host ไปยัง External
   Address ตัวแทน, และจำลอง Inbound Connection ไปยัง Server1

### 150.2 Running-Config เต็มของ WAN-EDGE-1 (ส่วนที่เกี่ยวกับ NAT)

```
hostname WAN-EDGE-1
!
ip nat translation timeout 86400
ip nat translation tcp-timeout 3600
ip nat translation udp-timeout 300
!
interface GigabitEthernet0/0/0
 description ** OUTSIDE ** to ISP-RTR
 ip address 203.0.113.2 255.255.255.252
 ip access-group OUTSIDE-IN-ACL in
 ip nat outside
 negotiation auto
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** INSIDE ** to CORE-SW1
 ip address 10.10.254.1 255.255.255.252
 ip nat inside
 negotiation auto
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** INSIDE ** cross-link to WAN-EDGE-2
 ip address 10.10.254.9 255.255.255.252
 ip nat inside
 negotiation auto
 no shutdown
!
ip access-list standard NAT-INTERNAL-ACL
 remark PAT selector - VLAN 10/20/30/40 เท่านั้น (ไม่รวม VLAN 99 MGMT)
 permit 10.10.10.0 0.0.0.255
 permit 10.10.20.0 0.0.0.255
 permit 10.10.30.0 0.0.0.255
 permit 10.10.40.0 0.0.0.255
!
ip access-list extended OUTSIDE-IN-ACL
 remark อนุญาตเฉพาะ HTTPS ไปยัง Server1 public IP เท่านั้น
 permit tcp any host 203.0.113.10 eq 443
 deny ip any host 203.0.113.10
 permit icmp any host 203.0.113.2 echo-reply
 permit icmp any host 203.0.113.2 time-exceeded
 permit icmp any host 203.0.113.2 unreachable
 deny ip any any log
!
ip nat inside source static tcp 10.10.30.10 443 203.0.113.10 443 extendable
ip nat inside source list NAT-INTERNAL-ACL interface GigabitEthernet0/0/0 overload
!
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

> **หมายเหตุ**: `ip route 0.0.0.0 0.0.0.0 203.0.113.1` คือ Default Route ไปยัง `ISP-RTR`
> ซึ่งเป็นสิ่งจำเป็นคู่กับ NAT เสมอ — NAT ไม่ได้แทนที่ Routing แต่ทำงานร่วมกับ Routing Table
> ตามลำดับที่อธิบายใน Step 146 (Traffic ต้องหา Path ออกทาง Interface ที่ถูกต้องก่อน NAT
> จะเลือก Translation ให้สอดคล้องกับ Interface นั้น)

### 150.3 Verification Step 1 — ตรวจสอบ Interface Role และ ACL

```
WAN-EDGE-1# show ip nat statistics
Total translations: 1 (1 static, 0 dynamic; 1 extended)
Outside interfaces:
  GigabitEthernet0/0/0
Inside interfaces:
  GigabitEthernet0/0/1, GigabitEthernet0/0/2
Hits: 0  Misses: 0

WAN-EDGE-1# show access-lists
Standard IP access list NAT-INTERNAL-ACL
    10 permit 10.10.10.0, wildcard bits 0.0.0.255
    20 permit 10.10.20.0, wildcard bits 0.0.0.255
    30 permit 10.10.30.0, wildcard bits 0.0.0.255
    40 permit 10.10.40.0, wildcard bits 0.0.0.255
Extended IP access list OUTSIDE-IN-ACL
    10 permit tcp any host 203.0.113.10 eq 443
    20 deny ip any host 203.0.113.10
    30 permit icmp any host 203.0.113.2 echo-reply
    40 permit icmp any host 203.0.113.2 time-exceeded
    50 permit icmp any host 203.0.113.2 unreachable
    60 deny ip any any log
```

### 150.4 Verification Step 2 — จำลอง Traffic ขาออกจากทุก VLAN (PAT)

ทดสอบจาก Host ตัวแทนของแต่ละ VLAN ไปยัง External Address ตัวแทน (สมมติทดสอบด้วย extended ping
จาก `WAN-EDGE-1` เอง โดยระบุ Source เป็น SVI ของแต่ละ VLAN เพื่อจำลอง — ในการทดสอบจริงควรทำจาก
Host ปลายทางโดยตรง):

```
! ทดสอบจาก PC ใน VLAN 10 (SALES) ไปยัง Public DNS ของ Google
PC1> ping 8.8.8.8

Pinging 8.8.8.8 with 32 bytes of data:
Reply from 8.8.8.8: bytes=32 time=18ms TTL=118
Reply from 8.8.8.8: bytes=32 time=17ms TTL=118
Reply from 8.8.8.8: bytes=32 time=19ms TTL=118
Reply from 8.8.8.8: bytes=32 time=18ms TTL=118

Ping statistics for 8.8.8.8:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

ตรวจสอบที่ `WAN-EDGE-1` ทันทีหลัง Ping:

```
WAN-EDGE-1# show ip nat translations
Pro Inside global          Inside local         Outside local        Outside global
tcp 203.0.113.10:443       10.10.30.10:443      ---                  ---
icmp 203.0.113.2:5         10.10.10.51:5        8.8.8.8:5             8.8.8.8:5
```

จำลองต่อจาก Host ใน VLAN 20 (VOICE), 30 (SERVERS), 40 (WIFI) เรียก curl ไปยัง Web Server
สาธารณะ (`93.184.216.34`, ตัวแทน External HTTP endpoint):

```
Voice-Phone20> curl -I http://93.184.216.34
Server-VLAN30-Host> curl -I http://93.184.216.34
WiFi-Client40> curl -I http://93.184.216.34
```

```
WAN-EDGE-1# show ip nat translations
Pro Inside global          Inside local         Outside local        Outside global
tcp 203.0.113.10:443       10.10.30.10:443      ---                  ---
icmp 203.0.113.2:5         10.10.10.51:5        8.8.8.8:5             8.8.8.8:5
tcp 203.0.113.2:52201      10.10.20.60:52201    93.184.216.34:80     93.184.216.34:80
tcp 203.0.113.2:52202      10.10.30.55:52202    93.184.216.34:80     93.184.216.34:80
tcp 203.0.113.2:52203      10.10.40.80:52203    93.184.216.34:80     93.184.216.34:80
```

สังเกต **Inside Global IP เป็น `203.0.113.2` เดียวกันทุกแถว แต่ Port ต่างกัน** — นี่คือ PAT
ทำงานถูกต้องตามที่ออกแบบไว้ Host จาก 3 VLAN ที่ต่างกัน (20, 30, 40) ทั้งหมดใช้ Public IP
ตัวเดียวกันพร้อมกันได้โดยไม่ชนกัน

### 150.5 Verification Step 3 — ยืนยันว่า VLAN 99 (MGMT) ไม่ถูก PAT (ตามที่ตั้งใจ)

```
MGMT-Host99> ping 8.8.8.8

Pinging 8.8.8.8 with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 8.8.8.8:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

```
WAN-EDGE-1# show ip nat translations | include 10.10.99
! (ไม่มีผลลัพธ์ - ไม่มี Entry ของ VLAN 99 เลย ตามที่ตั้งใจ)
```

Traffic จาก VLAN 99 ไปถึง `WAN-EDGE-1` และพยายาม Route ออกทาง Default Route ปกติ (Routing
เกิดก่อน NAT ตาม Step 146) แต่เมื่อถึงขั้น NAT, ACL `NAT-INTERNAL-ACL` ไม่ Match Source
`10.10.99.0/24` เลย (ไม่มีบรรทัด permit ให้) — Packet จึงถูกส่งออกทาง Interface Outside
โดย **ไม่ถูกแปลง Source Address** (ยังเป็น `10.10.99.x` ซึ่งเป็น Private Address) ผลคือ `ISP-RTR`
และอุปกรณ์บน Internet จะ Drop Packet นี้เพราะไม่รู้จัก Route กลับมาที่ Private Address —
Ping จึง Timeout ตามที่คาดหวัง (ตรงกับหลักการที่อธิบายไว้ใน Step 148.1)

### 150.6 Verification Step 4 — จำลอง Inbound Connection จาก Internet เข้า Server1

จำลองจากมุมมองของ Client บน Internet (ผ่าน `ISP-RTR`) เชื่อมต่อไปยัง `203.0.113.10:443`:

```
! บน ISP-RTR (จำลอง Internet Client)
ISP-RTR# telnet 203.0.113.10 443
Trying 203.0.113.10, 443 ... Open
```

ตรวจสอบที่ `WAN-EDGE-1`:

```
WAN-EDGE-1# show ip nat translations
Pro Inside global          Inside local         Outside local        Outside global
tcp 203.0.113.10:443       10.10.30.10:443      203.0.113.1:52999    203.0.113.1:52999
```

Entry ใหม่ปรากฏขึ้น — `Outside local`/`Outside global` ทั้งสองเท่ากันเพราะไม่มี Outside NAT
เกี่ยวข้อง (ตามที่อธิบายใน Step 141.4) `Inside global` คือ `203.0.113.10:443` (Public-facing)
ส่วน `Inside local` คือ `10.10.30.10:443` (Server1 จริง) ทดสอบว่าเข้าถึง Port อื่นไม่ได้:

```
ISP-RTR# telnet 203.0.113.10 22
Trying 203.0.113.10, 22 ...
% Connection timed out; remote host not responding
```

Connection ไป Port 22 ล้มเหลวเพราะ `OUTSIDE-IN-ACL` มีบรรทัด
`deny ip any host 203.0.113.10` บล็อกทุก Protocol/Port อื่นไปยัง `203.0.113.10` ยกเว้น
TCP/443 ที่อนุญาตไว้บรรทัดแรก — ตรงตามโจทย์ที่ต้องการเปิดเฉพาะ HTTPS เท่านั้น

### 150.7 สรุปผล Lab

| ข้อกำหนด | ผลการทดสอบ | สถานะ |
|---|---|---|
| VLAN 10/20/30/40 ออกอินเทอร์เน็ตผ่าน PAT ด้วย IP เดียว | `show ip nat translations` เห็น Inside Global `203.0.113.2` ซ้ำกันหลาย Entry ต่าง Port | ผ่าน |
| VLAN 99 ห้ามออกอินเทอร์เน็ต | Ping จาก MGMT Host Timeout, ไม่มี NAT Entry ของ `10.10.99.x` | ผ่าน |
| Server1 เปิดเฉพาะ HTTPS (443) | Telnet ไป 443 สำเร็จ, ไป 22 ถูกบล็อก | ผ่าน |
| Static NAT ของ Server1 ถาวรไม่ว่ามี Traffic หรือไม่ | Entry `203.0.113.10:443` ปรากฏใน `show ip nat translations` ตั้งแต่ก่อนมี Session จริง (Step 150.3) | ผ่าน |

---

## แบบฝึกหัดทวนความเข้าใจ Part 15

1. อธิบายความแตกต่างระหว่าง **Inside Local** และ **Inside Global** โดยยกตัวอย่าง Address จริง
   จาก IP Address Plan ของหลักสูตร (เช่น Server1)
2. เพราะเหตุใด Dynamic NAT (แบบไม่มี `overload`) จึงไม่เหมาะกับองค์กรที่มี User จำนวนมาก
   เมื่อเทียบกับ Public IP ที่มีจำกัด?
3. ถ้าลืมใส่คำสั่ง `ip nat outside` บน Interface ที่ต่อไปยัง `ISP-RTR` จะเกิดอะไรขึ้น และจะ
   ตรวจสอบด้วยคำสั่งอะไรว่า Interface ถูก Mark ถูกต้องหรือไม่?
4. อธิบาย NAT Order of Operations สำหรับ Traffic **ขาเข้า (Outside → Inside)** — NAT เกิดก่อน
   หรือหลัง Routing Decision? และทำไม?
5. ในคำสั่ง `ip nat inside source static tcp 10.10.30.10 443 203.0.113.10 443 extendable`
   Keyword `extendable` มีไว้เพื่ออะไร และถ้าไม่ใส่จะมีผลอย่างไรเมื่อมี Static NAT Entry
   หลายตัว?

**เฉลย:**

1. **Inside Local** คือ IP ตามที่ Host นั้น config จริงภายในองค์กร เช่น Server1 = `10.10.30.10`
   (Private/RFC1918) ส่วน **Inside Global** คือ IP ตามที่โลกภายนอกมองเห็น Server1 นั้น เช่น
   `203.0.113.10` (Public, จากการทำ Static NAT ใน Step 142)
2. เพราะ Dynamic NAT (ไม่มี Overload) ทำ Mapping แบบ **1 Public IP ต่อ 1 Session พร้อมกัน**
   เท่านั้น — ถ้า Pool มี Public IP น้อยกว่าจำนวน Host ที่ต้องการออกอินเทอร์เน็ตพร้อมกันจริง
   Host ส่วนเกินจะไม่ได้ Translation และถูก Drop จนกว่า Pool จะมี Address ว่าง
3. Traffic จะไม่ถูก NAT เลย (เพราะ IOS ไม่รู้ว่า Interface นั้นเป็นฝั่ง Outside) — Ping/Traffic
   จาก Internal Host จะไม่ผ่าน หรือ `show ip nat translations` จะไม่มี Entry ปรากฏขึ้นแม้มี
   Traffic วิ่งจริง ตรวจสอบด้วย `show ip nat statistics` แล้วดูว่า Interface นั้นอยู่ใน Section
   "Outside interfaces" หรือไม่
4. **NAT เกิดก่อน Routing Decision** สำหรับ Traffic ขาเข้า เพราะ Router ต้องแปลง Destination
   IP (Inside Global เช่น `203.0.113.10`) กลับเป็น Inside Local (`10.10.30.10`) เสียก่อน จึงจะ
   สามารถค้นหา Routing Table เพื่อหา Path ไปยัง Subnet ภายใน (`10.10.30.0/24`) ได้ถูกต้อง —
   Routing Table ไม่มี Entry เจาะจงสำหรับ Public IP นั้นโดยตรง
5. `extendable` ช่วยให้ IOS แยกความกำกวมของ Translation Table เมื่อมี Static NAT Entry มากกว่า
   หนึ่งตัวที่อาจดูเหมือนซ้ำกันบางส่วน (เช่น ต่างกันแค่ Port หรือ Protocol) ถ้าไม่ใส่ IOS อาจ
   ปฏิเสธการเพิ่ม Static NAT Entry ใหม่ หรือ Behavior ของ Translation อาจไม่ตรงตามที่ต้องการ
   เมื่อมี Entry หลายตัวที่เกี่ยวข้องกับ Address/Port ที่ใกล้เคียงกัน

---

## สรุป Part 15

Part นี้ครอบคลุม NAT/PAT ตั้งแต่พื้นฐานคำศัพท์ (Inside/Outside Local/Global), Static NAT
แบบ 1:1, Dynamic NAT แบบ Pool, PAT/NAT Overload ที่ใช้จริงในทุกองค์กร, การ Mark Inside/Outside
Interface, NAT Order of Operations ที่กระทบการ Troubleshoot, Port Forwarding สำหรับเปิดบริการ
เฉพาะ Port, การแยกบทบาท ACL ระหว่าง NAT Selector กับ Security ACL, และปิดท้ายด้วย Lab เต็ม
รูปแบบบน `WAN-EDGE-1` ที่รวม Static NAT (Server1 HTTPS), PAT (ทุก VLAN ยกเว้น MGMT), และ
Security ACL เข้าด้วยกันอย่างสอดคล้อง — ระบบ NAT ที่วางไว้ใน Part นี้จะเป็นฐานสำคัญที่ Part
ถัดๆไปเกี่ยวกับ VPN, Firewall (FW-1), และ Cloud Connectivity จะอ้างอิงถึงตลอดหลักสูตร

**ไปต่อ:** [Part 16 — DHCP & DNS →](part-016-dhcp-dns.md)
