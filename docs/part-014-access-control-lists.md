# Part 14 — Access Control Lists (Standard/Extended)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 131–140 จาก 1000**

> ต่อจาก [Part 13 — EIGRP Fundamentals](part-013-eigrp-fundamentals.md) ที่เราปูพื้นฐาน Dynamic
> Routing Protocol ตัวที่สอง Part นี้จะเปลี่ยนโฟกัสไปที่ **Security ระดับ Layer 3/4** — การใช้
> **Access Control List (ACL)** เพื่อกรอง (filter) traffic ระหว่าง VLAN ต่างๆ ใน Enterprise Lab
> ของเรา ตั้งแต่การจำกัดสิทธิ์ Management Access ไปจนถึงการควบคุม traffic ระหว่าง SALES,
> VOICE และ SERVERS VLAN ตาม Security Policy ขององค์กร ACL เป็นพื้นฐานสำคัญที่จะใช้ต่อใน
> **Part 15 (NAT/PAT)**, **Part 22 (AAA)**, และ Firewall Policy ใน **Part 43-45**

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 131 | ACL คืออะไร — แนวคิด Packet Filtering, Top-Down Processing, Implicit Deny, ลำดับ Statement |
| 132 | ทวน Wildcard Mask (เชื่อมกับ Part 7) และการใช้กับ ACL Match Statement |
| 133 | Standard ACL — Syntax, Numbered/Named, จำกัด Telnet/SSH เข้า VTY เฉพาะ MGMT VLAN |
| 134 | Extended ACL — Syntax, Match Protocol/Port, จำกัด SALES เข้าถึง SERVERS เฉพาะ Server1 |
| 135 | Named ACL — ข้อดีเหนือ Numbered ACL, Sequence Number, `remark` |
| 136 | การนำ ACL ไปใช้กับ Interface — `ip access-group in/out` และผลต่างของทิศทาง |
| 137 | การแก้ไข ACL ด้วย Sequence Number — เพิ่ม/ลบบรรทัดโดยไม่ต้องลบทั้ง ACL |
| 138 | ข้อผิดพลาดที่พบบ่อย (Common Pitfalls) และวิธีป้องกันตัวเองล็อกตัวเองออก |
| 139 | การ Verify/Troubleshoot ACL — `show access-lists`, `show ip interface`, `log` keyword |
| 140 | Lab ใหญ่: Security Policy เต็มรูปแบบสำหรับ Enterprise Lab ด้วย ACL |

---

## Step 131 — ACL คืออะไร: Packet Filtering, Top-Down Processing, Implicit Deny

### แนวคิด Packet Filtering

**Access Control List (ACL)** คือชุดของ **Statement (Access Control Entry — ACE)** ที่เรียงกัน
เป็นลำดับ (Ordered List) ซึ่ง Router/Switch (Layer 3) ใช้ตรวจสอบ packet ที่ผ่านเข้า/ออก interface
แล้วตัดสินใจว่าจะ **permit** (อนุญาตให้ผ่าน) หรือ **deny** (บล็อกทิ้ง) ตาม condition ที่กำหนด

ACL ทำงานที่ **Layer 3 (IP Address)** และ **Layer 4 (Port/Protocol)** เป็นหลัก — ต่างจาก Firewall
แบบ Stateful (ASA/FTD ที่จะเรียนใน Part 43-45) ตรงที่ ACL บน Router/Switch เป็น **Stateless**
(ไม่จำสถานะ connection เอง ยกเว้นใช้ keyword `established` ซึ่งจะพูดถึงใน Step 138)

### ACL ใช้งานได้ 3 จุดหลักในหลักสูตรนี้

| จุดใช้งาน | ตัวอย่าง |
|---|---|
| **Traffic Filtering** ระหว่าง Interface/VLAN | จำกัด SALES ไม่ให้เข้าถึง SERVERS (Step 134) |
| **Management Plane Protection** | จำกัด VTY (SSH/Telnet) ให้เข้าได้จาก MGMT VLAN เท่านั้น (Step 133) |
| **Match Traffic ให้ Feature อื่น** | Route-map, NAT (Part 15), QoS, VPN (ใช้ ACL เป็น "ตัวจับคู่ traffic") |

### หลักการทำงาน: Top-Down Sequential Processing

นี่คือกฎที่ **สำคัญที่สุด** ของ ACL ทั้งหมด: Router จะอ่าน ACE **จากบนลงล่าง ทีละบรรทัด**
และ **หยุดทันทีที่เจอบรรทัดแรกที่ match** (ไม่อ่านบรรทัดที่เหลือต่อ ไม่ว่าจะมี match ที่ specific
กว่าอยู่ข้างล่างก็ตาม)

```
Packet เข้ามา
     │
     ▼
┌─────────────────┐
│ ACE line 10      │──── Match? ──Yes──► ทำตามคำสั่ง (permit/deny) แล้วหยุด ไม่ดูบรรทัดต่อไป
└─────────────────┘
     │ No (ไม่ match)
     ▼
┌─────────────────┐
│ ACE line 20      │──── Match? ──Yes──► ทำตามคำสั่ง แล้วหยุด
└─────────────────┘
     │ No
     ▼
    ... (ทำแบบนี้ต่อไปจนหมดรายการ) ...
     │
     ▼
┌─────────────────────────────┐
│ Implicit Deny (deny ip any any) │  <-- ไม่เห็นในการ show run แต่มีอยู่จริงเสมอ!
└─────────────────────────────┘
```

### Implicit Deny ที่ปลาย ACL — กฎที่พลาดบ่อยที่สุด

ทุก ACL ของ Cisco (Standard และ Extended) จะมี **`deny any` (หรือ `deny ip any any`) ที่ซ่อนอยู่
ท้ายรายการเสมอ** แม้จะไม่เห็นบรรทัดนี้ตอนพิมพ์ `show running-config` ก็ตาม

> **ผลที่ตามมา**: ถ้าเราสร้าง ACL ที่มีแค่ `permit` statement เดียว แล้วไม่ได้ตั้งใจให้ traffic
> อื่นทั้งหมดถูก drop — packet ที่ไม่ match บรรทัดใดเลยจะถูก **deny โดยอัตโนมัติ** ทันทีที่ ACL
> ถูก apply เข้า interface นี่คือสาเหตุอันดับ 1 ที่วิศวกรมือใหม่ทำ Management Access ตัวเองหลุด
> (จะพูดถึงวิธีป้องกันใน Step 138)

### ความสำคัญของลำดับ Statement (Order Matters!)

เพราะ ACL ประมวลผลแบบ top-down และหยุดที่ match แรก **ลำดับของ statement มีผลต่อผลลัพธ์
โดยตรง** แม้ว่า statement ทั้งหมดจะเหมือนกัน แต่สลับลำดับกัน ผลลัพธ์ก็ต่างกันได้

**ตัวอย่างที่ผิด (Order ผิด → ACL ใช้งานไม่ได้ตามที่ตั้งใจ)**:
```
access-list 100 deny   ip 10.10.10.0 0.0.0.255 any
access-list 100 permit ip host 10.10.10.5 any
```
ผลลัพธ์: Host `10.10.10.5` **จะถูก deny เสมอ** เพราะบรรทัด `deny` ทั้ง subnet `10.10.10.0/24`
มาก่อน และ `10.10.10.5` อยู่ใน subnet นั้น จึง match บรรทัดแรกและหยุดทันที (ไม่ทันได้ไปถึงบรรทัด
`permit` ที่ specific กว่า)

**ตัวอย่างที่ถูก (กฎทอง: Specific ต้องมาก่อน General เสมอ)**:
```
access-list 100 permit ip host 10.10.10.5 any
access-list 100 deny   ip 10.10.10.0 0.0.0.255 any
```
ผลลัพธ์: `10.10.10.5` match บรรทัดแรกก่อน (permit) แล้วหยุด — Host อื่นๆ ใน `10.10.10.0/24`
ที่ไม่ใช่ `.5` จะไหลลงไป match บรรทัดที่สอง (deny)

> **กฎทองของการเขียน ACL**: เรียง statement จาก **specific ที่สุดไปหา general ที่สุด**
> (host เดี่ยว → subnet เล็ก → subnet ใหญ่ → any) เสมอ

---

## Step 132 — ทวน Wildcard Mask และการใช้กับ ACL Match Statement

Part 7 (Step 67) ได้สอน Wildcard Mask ไว้แล้วว่าเป็นค่า**ตรงข้าม (Inverse)** ของ Subnet Mask
แบบ bit-by-bit — Step นี้จะทวนสั้นๆ และเจาะไปที่การใช้งานจริงใน ACL statement โดยเฉพาะ
เพราะ **ACL ทุกตัวใน Cisco IOS ใช้ Wildcard Mask เท่านั้น ไม่ใช้ Subnet Mask**

### ทวนกฎ Wildcard Mask

| Bit ใน Wildcard Mask | ความหมาย |
|---|---|
| **0** | ต้องตรงกัน (must match) กับ Address ที่ระบุใน ACE |
| **1** | ไม่สนใจค่า (don't care) — ยอมรับทั้ง 0 และ 1 ที่ตำแหน่งนั้น |

**สูตรแปลง**: `Wildcard = 255.255.255.255 − Subnet Mask`

### ตาราง Wildcard Mask ที่ใช้บ่อยที่สุดใน ACL (อ้างอิงจาก IP Plan ของหลักสูตร)

| Subnet ที่ต้องการ Match | Prefix | Subnet Mask | Wildcard Mask ที่ใช้ใน ACL |
|---|---|---|---|
| Host เดี่ยว (เช่น Server1) | /32 | 255.255.255.255 | `0.0.0.0` (หรือใช้ keyword `host`) |
| VLAN 10 SALES | /24 | 255.255.255.0 | `0.0.0.255` |
| VLAN 20 VOICE | /24 | 255.255.255.0 | `0.0.0.255` |
| VLAN 30 SERVERS | /24 | 255.255.255.0 | `0.0.0.255` |
| VLAN 40 WIFI | /24 | 255.255.255.0 | `0.0.0.255` |
| VLAN 99 MGMT | /24 | 255.255.255.0 | `0.0.0.255` |
| ทั้งหมด (any address) | 0.0.0.0/0 | 0.0.0.0 | `255.255.255.255` (หรือใช้ keyword `any`) |

### Worked Example: แปลงและเขียน ACE จริง

**ตัวอย่าง 1** — Match ทั้ง subnet MGMT VLAN (`10.10.99.0/24`):
```
Subnet Mask /24     = 255.255.255.0
255.255.255.255
−  255.255.255.  0
------------------
=    0.  0.  0.255
```
→ ACE: `permit 10.10.99.0 0.0.0.255`

**ตัวอย่าง 2** — Match เฉพาะ Host เดียว `10.10.30.10` (Server1):
```
Subnet Mask /32     = 255.255.255.255
255.255.255.255
−  255.255.255.255
------------------
=    0.  0.  0.  0
```
→ ACE: `permit 10.10.30.10 0.0.0.0` เขียนแบบย่อด้วย keyword `host` ได้เป็น
`permit host 10.10.30.10` (ผลลัพธ์เดียวกัน 100%)

**ตัวอย่าง 3** — Match ครึ่งบนของ VLAN 30 (`10.10.30.0/25`, host `.1`–`.126`):
```
Subnet Mask /25     = 255.255.255.128
255.255.255.255
−  255.255.255.128
------------------
=    0.  0.  0. 127
```
→ ACE: `permit 10.10.30.0 0.0.0.127`

### Keyword ลัดที่ต้องจำ

| Keyword | เทียบเท่ากับ | ใช้เมื่อ |
|---|---|---|
| `host <IP>` | `<IP> 0.0.0.0` | ต้องการ match เฉพาะ IP เดียว |
| `any` | `0.0.0.0 255.255.255.255` | ต้องการ match ทุก Address |

> **ข้อควรระวังที่พบบ่อย**: Wildcard Mask กับ Subnet Mask มีรูปแบบตัวเลขคล้ายกันมาก
> (`0.0.0.255` vs `255.255.255.0`) วิศวกรมือใหม่มักพิมพ์สลับกันโดยไม่ตั้งใจ ผลคือ ACL match
> ผิดกลุ่ม IP ทั้งหมด — ควร**ตรวจทาน ACE ทุกบรรทัดด้วย `show access-lists` ก่อน apply จริงเสมอ**

---

## Step 133 — Standard ACL: Syntax, Numbered vs Named, จำกัด VTY เฉพาะ MGMT VLAN

### Standard ACL คืออะไร

**Standard ACL** เป็น ACL รุ่นแรกสุดของ Cisco IOS ตรวจสอบ (match) ได้ **เฉพาะ Source IP
Address เท่านั้น** ไม่สามารถ match Destination IP, Protocol, หรือ Port ได้เลย
ด้วยความจำกัดนี้ Standard ACL จึงเหมาะกับงานง่ายๆ เช่น **การกรอง Management Access
(VTY line)** หรือ **การ match traffic ให้ Route-map/NAT** มากกว่าการกรอง traffic ทั่วไป

### Syntax: Numbered Standard ACL

```
Router(config)# access-list <1-99 หรือ 1300-1999> {permit|deny} <source> [source-wildcard]
```

| Range Number | ประเภท |
|---|---|
| 1–99 | Standard ACL (range ดั้งเดิม) |
| 1300–1999 | Standard ACL (Expanded range) |
| 100–199 | Extended ACL (range ดั้งเดิม — Step 134) |
| 2000–2699 | Extended ACL (Expanded range) |

### Worked Example: จำกัด Telnet/SSH เข้า VTY เฉพาะจาก MGMT VLAN

**Requirement**: ให้ทุก DIST-SW และ CORE-SW **รับ SSH/Telnet เข้า VTY ได้เฉพาะจาก
`10.10.99.0/24` (MGMT VLAN) เท่านั้น** — จาก VLAN อื่นห้ามเข้า VTY โดยตรงเด็ดขาด

```
DIST-SW1(config)# access-list 10 permit 10.10.99.0 0.0.0.255
DIST-SW1(config)# access-list 10 deny   any log
```

> สังเกตว่าเราพิมพ์ `deny any log` เพิ่มเข้ามาเอง (ทั้งที่มี Implicit Deny อยู่แล้ว) เพื่อให้
> keyword `log` ทำงาน (ดู Step 139) — Implicit Deny ที่ซ่อนอยู่ **ไม่มี log** ให้เราเห็น

**นำ ACL ไปผูกกับ VTY line ด้วยคำสั่ง `access-class` (ไม่ใช่ `ip access-group` — คำสั่งนั้นใช้กับ
interface ทั่วไป ส่วน VTY ใช้ `access-class` เสมอ)**:

```
DIST-SW1(config)# line vty 0 15
DIST-SW1(config-line)# access-class 10 in
DIST-SW1(config-line)# transport input ssh
DIST-SW1(config-line)# login local
DIST-SW1(config-line)# exit
```

**ทดสอบผลลัพธ์**:
```
! จาก PC ใน VLAN 99 (MGMT) - ควรเข้าได้
C:\> ssh admin@10.10.99.10
Password: ***********
DIST-SW1>

! จาก PC1 ใน VLAN 10 (SALES) - ควรถูกปฏิเสธ
C:\> ssh admin@10.10.99.10
% Connection refused by remote host
```

### Named Standard ACL (แนะนำมากกว่า Numbered — ดูรายละเอียดข้อดีใน Step 135)

```
DIST-SW1(config)# ip access-list standard MGMT-ONLY-VTY
DIST-SW1(config-std-nacl)# permit 10.10.99.0 0.0.0.255
DIST-SW1(config-std-nacl)# deny   any log
DIST-SW1(config-std-nacl)# exit
DIST-SW1(config)# line vty 0 15
DIST-SW1(config-line)# access-class MGMT-ONLY-VTY in
```

### ข้อจำกัดสำคัญของ Standard ACL

| ข้อจำกัด | รายละเอียด |
|---|---|
| Match ได้แค่ Source IP | ไม่สามารถ match Destination, Protocol (TCP/UDP/ICMP), หรือ Port |
| ตำแหน่งการ apply | ควร apply **ใกล้ Destination ที่สุด** (เพราะ match ได้แค่ source จะ deny
  traffic ทั้งหมดจาก source นั้นไปทุกที่ ถ้า apply ผิดตำแหน่งอาจ block traffic ที่ไม่ตั้งใจ) |
| การใช้งานที่เหมาะสมในหลักสูตรนี้ | VTY access-class, Route-map match, NAT source match (Part 15) |

> เปรียบเทียบกับ **Extended ACL** (Step 134) ที่ match ได้ทั้ง Source+Destination+Protocol+Port
> จึง**ควร apply ใกล้ Source ที่สุด** เพื่อ drop traffic ที่ไม่ต้องการตั้งแต่ทางเข้า
> ประหยัด bandwidth บน backbone

---

## Step 134 — Extended ACL: Match Protocol/Port, จำกัด SALES เข้า SERVERS เฉพาะ Server1

### Extended ACL คืออะไร

**Extended ACL** match ได้ครบทั้ง **Source IP + Destination IP + Protocol (ip/tcp/udp/icmp)
+ Port Number/Operator** — เป็น ACL ที่ใช้งานจริงมากที่สุดในการกรอง traffic ระหว่าง VLAN

### Syntax เต็มรูปแบบ

```
Router(config)# access-list <100-199 หรือ 2000-2699> {permit|deny} <protocol> \
    <source> <source-wildcard> [operator source-port] \
    <destination> <destination-wildcard> [operator destination-port] \
    [established] [log]
```

| ส่วนประกอบ | คำอธิบาย | ตัวอย่าง |
|---|---|---|
| `<protocol>` | `ip` (ทุก protocol), `tcp`, `udp`, `icmp`, `ospf`, `eigrp` เป็นต้น | `tcp` |
| `<source>` `<wildcard>` | IP + Wildcard ต้นทาง (หรือ `any` / `host x.x.x.x`) | `10.10.10.0 0.0.0.255` |
| `<destination>` `<wildcard>` | IP + Wildcard ปลายทาง | `host 10.10.30.10` |
| `operator port` | `eq` (equal), `neq`, `gt`, `lt`, `range` + หมายเลข port หรือชื่อ | `eq 443` |
| `established` | (เฉพาะ TCP) match เฉพาะ return traffic ของ session ที่เริ่มจากอีกฝั่ง | ดู Step 138 |

### Worked Example: SALES เข้าถึง SERVERS ได้เฉพาะ HTTPS/SSH ไปที่ Server1

**Requirement (Security Policy ของ Enterprise Lab)**:
1. VLAN 10 (SALES, `10.10.10.0/24`) เข้าถึง **Server1 (`10.10.30.10`)** ได้เฉพาะ **HTTPS (443/TCP)**
   และ **SSH (22/TCP)** เท่านั้น
2. VLAN 10 ห้ามเข้าถึง Server อื่น (Server2, Server3) หรือ port อื่นของ Server1 ทั้งหมด
3. Traffic อื่นทั้งหมดจาก VLAN 10 ไป VLAN 30 (SERVERS) ให้ deny

```
DIST-SW3(config)# access-list 100 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
DIST-SW3(config)# access-list 100 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
DIST-SW3(config)# access-list 100 deny   ip  10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
DIST-SW3(config)# access-list 100 permit ip  any any
```

**อธิบายทีละบรรทัด**:
- บรรทัด 1-2: permit เฉพาะ HTTPS และ SSH จาก SALES ทั้ง subnet ไปยัง Server1 host เดียว
  (เขียนก่อนเสมอ ตามกฎ "specific มาก่อน general")
- บรรทัด 3: deny traffic ที่เหลือทั้งหมดจาก SALES ไปยัง SERVERS ทั้ง subnet (ครอบคลุม
  Server2, Server3, และ port อื่นของ Server1 ที่ไม่ match บรรทัด 1-2) — ใส่ `log` ไว้ดู
  ว่ามีใครพยายามเข้าถึงที่ไม่ได้รับอนุญาต
- บรรทัด 4: **สำคัญมาก** — `permit ip any any` ปลายรายการ เพื่อไม่ให้ traffic อื่นที่ไม่เกี่ยวกับ
  SALES→SERVERS (เช่น traffic ปกติของ VLAN อื่นที่ผ่าน interface เดียวกัน) ถูก Implicit Deny
  บล็อกไปด้วยโดยไม่ตั้งใจ (รายละเอียดเพิ่มเติมเรื่องนี้ใน Step 138)

### ตัวอย่าง Operator Port ที่ใช้บ่อย

| Operator | ความหมาย | ตัวอย่าง |
|---|---|---|
| `eq` | เท่ากับ port นี้เท่านั้น | `eq 443` |
| `neq` | ไม่เท่ากับ port นี้ | `neq 23` |
| `gt` | มากกว่า port นี้ | `gt 1023` |
| `lt` | น้อยกว่า port นี้ | `lt 1024` |
| `range` | ช่วง port (ระบุ 2 ค่า) | `range 5060 5061` |

### เปรียบเทียบ Standard vs Extended ACL

| คุณสมบัติ | Standard ACL | Extended ACL |
|---|---|---|
| Match ได้ | Source IP เท่านั้น | Source IP, Dest IP, Protocol, Port |
| Range Number | 1-99, 1300-1999 | 100-199, 2000-2699 |
| ตำแหน่งที่ควร apply | ใกล้ Destination | ใกล้ Source |
| ความละเอียดในการกรอง | หยาบ | ละเอียด (ใช้จริงมากที่สุด) |
| ใช้กับ VTY (`access-class`) | ได้ (นิยมใช้) | ได้ (แต่ overkill สำหรับงานนี้) |

---

## Step 135 — Named ACL: ข้อดีเหนือ Numbered ACL และการใช้ `remark`

### ทำไมต้องใช้ Named ACL

Numbered ACL (Step 133-134) มีข้อจำกัดสำคัญ: **ไม่สามารถแก้ไข/ลบ statement เดี่ยวๆ ได้ง่าย**
ใน IOS รุ่นเก่า ถ้าต้องการแก้ต้อง `no access-list 100` (ลบทั้งหมด) แล้วพิมพ์ใหม่ทั้งชุด — เสี่ยง
เกิดช่วงเวลาที่ไม่มี ACL ป้องกันอยู่เลย (traffic ทั้งหมดจะถูก permit ชั่วคราวเพราะไม่มี ACL ผูกกับ
interface ระหว่างพิมพ์ใหม่)

**Named ACL** แก้ปัญหานี้ได้ทั้งหมด และมี **Sequence Number อัตโนมัติ** ทำให้แก้ไขบรรทัดเดียว
ได้โดยไม่กระทบบรรทัดอื่น (รายละเอียดการแก้ไขจริงอยู่ใน Step 137)

| ข้อดีของ Named ACL | อธิบาย |
|---|---|
| ชื่อสื่อความหมาย | `SALES-TO-SERVERS` อ่านง่ายกว่า `access-list 100` มาก |
| แก้ไขบรรทัดเดี่ยวได้ | ใช้ sequence number `no 20` / `15 permit ...` โดยไม่ลบทั้ง ACL |
| รองรับ `remark` | เขียนคำอธิบายแนบไว้ในตัว ACL เอง (Numbered ACL ก็มี remark ได้เหมือนกัน
  แต่ไม่นิยมเพราะยังต้องจำหมายเลข) |
| Syntax เดียวกันทั้ง Standard/Extended | `ip access-list standard <name>` หรือ
  `ip access-list extended <name>` |

### Syntax

```
Router(config)# ip access-list standard <name>
Router(config-std-nacl)# ...

Router(config)# ip access-list extended <name>
Router(config-ext-nacl)# ...
```

### แปลง ACL จาก Step 134 เป็น Named ACL พร้อม `remark`

```
DIST-SW3(config)# ip access-list extended SALES-TO-SERVERS
DIST-SW3(config-ext-nacl)# remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
DIST-SW3(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
DIST-SW3(config-ext-nacl)# permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
DIST-SW3(config-ext-nacl)# remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
DIST-SW3(config-ext-nacl)# deny   ip  10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
DIST-SW3(config-ext-nacl)# remark ==== Explicit permit for everything else (do not block unrelated traffic) ====
DIST-SW3(config-ext-nacl)# permit ip any any
DIST-SW3(config-ext-nacl)# exit
```

**ตรวจสอบว่า sequence number ถูกใส่ให้อัตโนมัติ (step ทีละ 10):**
```
DIST-SW3# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    30 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
    40 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
    50 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
    60 remark ==== Explicit permit for everything else (do not block unrelated traffic) ====
    70 permit ip any any
```

> **หมายเหตุ**: `remark` เองก็ได้ sequence number ของตัวเองด้วย (10, 40, 60 ในตัวอย่างนี้) —
> IOS เว้นช่องว่างทีละ 10 เสมอเพื่อเผื่อพื้นที่แทรกบรรทัดใหม่ในอนาคต (ดู Step 137)

> **Best Practice ของหลักสูตรนี้ตั้งแต่ Part 14 เป็นต้นไป**: ใช้ **Named ACL เสมอ**
> ไม่ใช้ Numbered ACL อีก ยกเว้นตัวอย่างสอนพื้นฐานเพื่อความเข้าใจ syntax ดั้งเดิม

---

## Step 136 — การนำ ACL ไปใช้กับ Interface: `ip access-group in/out`

### Syntax

```
Router(config-if)# ip access-group <name หรือ number> {in|out}
```

- **`in`**: ตรวจสอบ packet **ทันทีที่เข้ามาใน interface นั้น** ก่อนที่ Router จะ route/ตัดสินใจใดๆ
- **`out`**: ตรวจสอบ packet **ก่อนที่จะออกจาก interface นั้น** หลังจาก Router routing แล้ว

### ความแตกต่างของทิศทาง — เรื่องที่ทำให้วิศวกรมือใหม่สับสนมากที่สุด

```
                         ROUTER / L3 SWITCH
                    ┌──────────────────────────┐
   Packet เข้า      │                            │      Packet ออก
  ───────────►│ IN ACL check │  Routing  │ OUT ACL check │───────────►
                    │  (ก่อน route)  │  Decision  │  (หลัง route)   │
                    └──────────────────────────┘

  "in"  = ทิศทางที่มองจาก "packet วิ่งเข้า interface นี้"  (ทิศทางเทียบกับ interface เสมอ
          ไม่ใช่เทียบกับทิศทางของ traffic flow โดยรวม)
  "out" = ทิศทางที่มองจาก "packet วิ่งออกจาก interface นี้"
```

| ทิศทาง | ประมวลผลตอนไหน | ผลกระทบต่อ CPU | ใช้เมื่อ |
|---|---|---|---|
| `in` | ก่อน Routing Decision | ประหยัดกว่า (drop เร็ว ไม่เสีย CPU routing) | ต้องการ deny ตั้งแต่ทางเข้า, ต้องการรู้ source interface ที่แท้จริง |
| `out` | หลัง Routing Decision | เสีย CPU routing ไปแล้วแม้จะ deny ทีหลัง | ต้องการกรองตามข้อมูลที่รู้เฉพาะหลัง routing (เช่น ปลายทางตาม output interface) |

> **กฎที่จำง่าย**: `in`/`out` อ้างอิงกับ**ตัว interface ที่ apply** เสมอ ไม่ใช่อ้างอิงกับทิศทางของ
> traffic โดยรวมในเครือข่าย — ต้องดูว่า packet **กำลังเข้า** หรือ **กำลังออก** จาก interface
> ตัวนั้นโดยเฉพาะ

### Worked Example: apply ACL จาก Step 135 บน DIST-SW3

DIST-SW3 เป็น Distribution Switch ที่มี SVI ของ VLAN 30 (SERVERS) ตาม IP Plan
(`10.10.30.0/24`) เนื่องจากเราต้องการ**กรอง traffic ที่มาจาก SALES ก่อนจะเข้าไปถึง SERVERS**
วิธีที่ถูกต้องที่สุดคือ apply ACL แบบ `in` บน **SVI VLAN 30 (interface ที่ traffic มาจาก SALES
ไหลเข้าไปยัง SERVERS)** — เพราะ Extended ACL ควร apply **ใกล้ Source ที่สุดเท่าที่ทำได้**
แต่ในกรณีนี้ SVI VLAN 30 คือจุดที่ traffic จาก SALES (ผ่านการ route ข้าม VLAN มาแล้ว) จะไป
ถึง SERVERS ก่อนจะ forward ออกไปยัง Access Layer ของ SERVERS VLAN

```
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# description ** SERVERS VLAN 30 SVI - filters inbound traffic to servers **
DIST-SW3(config-if)# ip access-group SALES-TO-SERVERS in
DIST-SW3(config-if)# exit
```

**ทำไมต้อง `in` บน SVI VLAN 30 แทน `out` บน SVI VLAN 10?**

| ตัวเลือก | ข้อดี/ข้อเสีย |
|---|---|
| `in` บน SVI VLAN 30 (เลือกใช้) | ตรวจ traffic **ทุกเส้นทาง**ที่จะเข้า SERVERS (ไม่ว่าจะมาจาก SALES, VOICE, WIFI VLAN ใดก็ตาม) ด้วย ACL เดียว — ง่ายต่อการดูแลรักษาระยะยาว |
| `out` บน SVI VLAN 10 เท่านั้น | ต้องทำ ACL แยกทุก VLAN ต้นทางที่ต้องการกรอง (VLAN 10, 20, 40 ฯลฯ) — ดูแลยากกว่า และ ACL statement จะต้องเขียนซ้ำ Destination เดิมทุกครั้ง |

> **ข้อควรจำ**: interface หนึ่งทิศทางหนึ่ง (`in` หรือ `out`) รองรับได้ **ACL เดียวเท่านั้นต่อ
> protocol** (เช่น apply ACL IPv4 ได้ 1 ตัวสำหรับ `in` และอีก 1 ตัวสำหรับ `out` บน interface
> เดียวกัน) ถ้า apply ACL ตัวใหม่ทับ ACL เดิมที่มีอยู่แล้วในทิศทางเดียวกัน ACL เดิมจะถูกแทนที่ทันที

---

## Step 137 — การแก้ไข ACL ด้วย Sequence Number

### ปัญหาของ Numbered ACL เมื่อต้องแก้ไข

```
Router(config)# access-list 100 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
Router(config)# access-list 100 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255
```
ถ้าต้องการแทรก `permit` สำหรับ SSH (22) เข้าไป**ระหว่าง**บรรทัดสองบรรทัดนี้ — Numbered ACL
รุ่นเก่าทำไม่ได้โดยตรง ต้องลบทั้งหมดแล้วพิมพ์ใหม่ (เสี่ยงเปิดช่องโหว่ชั่วคราว)

### วิธีแก้ไขที่ถูกต้องด้วย Named ACL + Sequence Number

**ตรวจสอบ sequence number ปัจจุบันก่อนแก้ไขเสมอ:**
```
DIST-SW3# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    30 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
    40 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
    50 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
    60 remark ==== Explicit permit for everything else ====
    70 permit ip any any
```

**ตัวอย่าง 1: แทรกบรรทัดใหม่ระหว่าง line 20 และ 30** (เพิ่ม permit RDP 3389 เข้า Server1
สำหรับทีม IT Support จาก SALES VLAN โดยเฉพาะ)

```
DIST-SW3(config)# ip access-list extended SALES-TO-SERVERS
DIST-SW3(config-ext-nacl)# 25 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 3389
```
เพราะ 25 อยู่ระหว่าง 20 และ 30 IOS จะแทรกบรรทัดนี้ **เข้าไปตรงกลาง** โดยไม่กระทบบรรทัดอื่นเลย

**ตัวอย่าง 2: ลบบรรทัดเดียวโดยไม่กระทบบรรทัดอื่น** (สมมติว่าทีม Security สั่งยกเลิกสิทธิ์ RDP
ที่เพิ่งเพิ่มไป)

```
DIST-SW3(config)# ip access-list extended SALES-TO-SERVERS
DIST-SW3(config-ext-nacl)# no 25
```

**ตรวจสอบผลลัพธ์หลังแก้ไข:**
```
DIST-SW3# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    30 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
    40 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
    50 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
    60 remark ==== Explicit permit for everything else ====
    70 permit ip any any
```
บรรทัด 25 ถูกลบไปแล้ว บรรทัดอื่นทั้งหมดยังคง sequence number เดิมทุกบรรทัด — **ไม่มี traffic
gap ระหว่างการแก้ไขเลย** (ACL ยัง apply อยู่กับ interface ตลอดเวลาที่แก้ไข)

### สรุปเปรียบเทียบการแก้ไข: Numbered vs Named ACL

| งานที่ต้องทำ | Numbered ACL | Named ACL (Sequence Number) |
|---|---|---|
| ลบบรรทัดเดียว | ต้องลบทั้ง ACL แล้วพิมพ์ใหม่ทั้งหมด | `no <seq#>` บรรทัดเดียว |
| แทรกบรรทัดกลาง | ต้องลบทั้ง ACL แล้วพิมพ์ใหม่ทั้งหมดตามลำดับ | `<seq#> permit/deny ...` |
| ความเสี่ยง downtime ระหว่างแก้ | สูง (ACL หายไปชั่วคราวระหว่างพิมพ์ใหม่) | ไม่มี — ACL ยัง active ตลอด |
| เหมาะกับ Production | ไม่แนะนำ | แนะนำเสมอ |

> **สรุป**: นี่คือเหตุผลหลักที่หลักสูตรนี้ (และ Best Practice ระดับ Enterprise จริง) ใช้ **Named ACL
> เท่านั้น** ตั้งแต่ Part 14 เป็นต้นไป

---

## Step 138 — ข้อผิดพลาดที่พบบ่อย (Common ACL Pitfalls)

### Pitfall 1: ลืม `permit ip any any` ตอนที่ตั้งใจให้ traffic อื่นผ่านได้

ถ้าเขียน ACL ที่มีแต่ `deny` โดยไม่มี `permit` ปลายรายการ (และตั้งใจให้ traffic นอกเหนือจากที่
deny ผ่านได้) — **Implicit Deny จะบล็อก traffic ทั้งหมดที่ไม่ match บรรทัดใดเลย**

```
! ผิด - ตั้งใจ deny เฉพาะ FTP แต่ไม่มี permit ปลายรายการ
ip access-list extended BLOCK-FTP-ONLY
 deny tcp any any eq 21
 ! ไม่มีบรรทัดต่อ -> Implicit Deny จะ block traffic TCP/UDP/ICMP อื่นทั้งหมดด้วย!
```

```
! ถูก - เติม permit ip any any ปลายรายการเพื่อให้ traffic อื่นผ่านได้ตามที่ตั้งใจ
ip access-list extended BLOCK-FTP-ONLY
 deny tcp any any eq 21
 permit ip any any
```

### Pitfall 2: Apply ACL ผิด Interface หรือผิดทิศทาง

Apply ACL ที่ออกแบบมาสำหรับกรอง SALES→SERVERS เข้ากับ interface ผิดตัว (เช่น apply เข้า
SVI VLAN 10 แบบ `out` แทน `in` บน SVI VLAN 30) จะทำให้ ACL **ดูเหมือนใช้งานได้แต่จริงๆ
ไม่กรอง traffic ตามที่ตั้งใจเลย** เพราะ Source/Destination Address ใน ACE ไม่สัมพันธ์กับทิศทาง
ที่ apply จริง

**วิธีป้องกัน**: ทุกครั้งหลัง apply ACL ให้ตรวจสอบด้วย `show ip interface <name>` (ดู Step 139)
และทดสอบจริงด้วย `ping`/`telnet`/`ssh` จากทั้งฝั่งที่ควร permit และฝั่งที่ควร deny

### Pitfall 3: ล็อกตัวเองออกจาก Management Access โดยไม่ตั้งใจ

นี่คือ**อุบัติเหตุที่เกิดขึ้นบ่อยที่สุด**ในงานจริง — วิศวกร apply ACL ผิดพลาด (เช่น ลืม permit
subnet ของตัวเอง หรือ apply ผิด interface) แล้ว SSH/Telnet session ของตัวเองถูกตัดทันที
โดยไม่มีทางเข้าไปแก้ไขอีก (ถ้าไม่มี console access ทางกายภาพ)

**เทคนิคป้องกันตัวเอง (Safety Net) ที่ต้องใช้ทุกครั้งก่อน apply ACL ที่มีผลต่อ Management
Access — คำสั่ง `reload in <นาที>`:**

```
DIST-SW3# reload in 10
Reload scheduled for 14:32:07 UTC Sat Sep 26 2026 (in 10 minutes) by admin
Proceed with reload? [confirm]

! ทำการ apply ACL ที่มีความเสี่ยง
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# ip access-group SALES-TO-SERVERS in
DIST-SW3(config-if)# end

! ทดสอบว่ายังเข้าถึงอุปกรณ์ได้ปกติหรือไม่ (SSH session ใหม่, ไม่ใช่ session เดิม)
! ถ้าเข้าได้ปกติ -> ยกเลิก reload ที่ตั้งไว้
DIST-SW3# reload cancel
Reload scheduled for 14:32:07 UTC Sat Sep 26 2026 (in 10 minutes) by admin is cancelled.

! ถ้าเข้าไม่ได้ -> ไม่ต้องทำอะไร รอ 10 นาที อุปกรณ์จะ reload กลับไปใช้ startup-config เดิม
! (เงื่อนไข: ต้องยังไม่ทำ 'copy running-config startup-config' ทับ config ที่ผิดพลาด!)
```

> **กฎเหล็ก**: `reload in <นาที>` ต้องตั้งไว้**ก่อน**เริ่มแก้ไข ACL ที่มีความเสี่ยงต่อ Management
> Access เสมอ และ**ห้าม** `write memory`/`copy running-config startup-config` จนกว่าจะยืนยัน
> แล้วว่า config ใหม่ทำงานถูกต้อง — ไม่เช่นนั้น reload จะโหลด config ที่ผิดพลาดกลับมาเหมือนเดิม

### Pitfall 4: ไม่ใช้ `established` keyword ทำให้ Return Traffic ถูก block

เมื่อ Extended ACL apply แบบ `in` ที่ interface ทางเข้าของ session TCP ฝั่งหนึ่ง แต่ไม่มี ACL
ที่ยอมรับ **return traffic** (traffic ตอบกลับ) ที่ interface อีกฝั่ง — session TCP จะเปิดไม่ได้
เพราะ SYN-ACK/ACK ที่ตอบกลับมาถูก deny โดย ACL ฝั่งรับ

**ตัวอย่าง**: อนุญาตให้ Server1 ตอบกลับ (return traffic) เฉพาะ session ที่ SALES เป็นผู้เริ่มต้น
เท่านั้น (ไม่ใช่ session ใหม่ที่ Server1 เริ่มเข้าหา SALES เอง):

```
DIST-SW3(config)# ip access-list extended SERVERS-TO-SALES-RETURN
DIST-SW3(config-ext-nacl)# permit tcp host 10.10.30.10 10.10.10.0 0.0.0.255 established
DIST-SW3(config-ext-nacl)# deny   ip any any log
```

`established` จะ match packet ที่มี **ACK หรือ RST flag ตั้งอยู่** เท่านั้น (คือ traffic ที่เป็น
ส่วนหนึ่งของ session ที่มีอยู่แล้ว ไม่ใช่ packet เริ่ม session ใหม่ที่มีแค่ SYN flag) — เหมาะสำหรับ
Extended ACL ที่ apply แบบ `out` ฝั่ง Server เพื่อไม่ให้ Server เริ่ม connection ใหม่เข้าหา Client
เอง แต่ยังตอบกลับ session ที่ Client เริ่มได้ตามปกติ

> **ข้อจำกัดของ `established`**: ใช้ได้กับ **TCP เท่านั้น** (UDP ไม่มี flag ให้ตรวจสอบ) และเป็นการ
> ตรวจสอบแบบ **Stateless** (ดู flag ของ packet เดี่ยวๆ ไม่ได้จำ state ของ session จริงแบบ Firewall)
> ดังนั้นยังมีช่องโหว่ที่ผู้โจมตีปลอมแปลง ACK flag ได้ — งานที่ต้องการความปลอดภัยสูงจริงต้องใช้
> Stateful Firewall (ASA/FTD, Zone-Based Firewall) ที่จะเรียนใน Part 43-45

### สรุปตาราง Pitfalls ทั้งหมด

| Pitfall | ผลกระทบ | วิธีป้องกัน |
|---|---|---|
| ลืม `permit` ปลายรายการ | Traffic ที่ไม่เกี่ยวข้องถูก deny ทั้งหมด | ตรวจสอบเจตนาก่อนเขียน ACL เสมอว่าต้องการ default-permit หรือ default-deny |
| Apply ผิด interface/ทิศทาง | ACL ดูเหมือนทำงานแต่ไม่กรองตามที่ตั้งใจ | ทดสอบจริงหลัง apply ทุกครั้ง + `show ip interface` |
| ล็อกตัวเองออก Management | เข้าอุปกรณ์ไม่ได้เลย (ต้อง console เข้าไปแก้) | `reload in 10` ก่อน apply เสมอ |
| ไม่ใส่ `established` | Return traffic ของ session ถูก block | ใส่ `established` บน ACL ฝั่งที่ apply กับ return path |

---

## Step 139 — การ Verify/Troubleshoot ACL

### `show access-lists` — ดู ACE ทั้งหมดพร้อม Match Counter

```
DIST-SW3# show access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443 (234 matches)
    30 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22 (12 matches)
    40 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
    50 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log (87 matches)
    60 remark ==== Explicit permit for everything else ====
    70 permit ip any any (15420 matches)
```

**ตัวเลข `(N matches)`** คือจำนวนครั้งที่ packet match บรรทัดนี้สะสมตั้งแต่ boot หรือตั้งแต่
`clear access-list counters` ครั้งล่าสุด — เป็นข้อมูลสำคัญที่สุดในการยืนยันว่า **ACL ทำงานตรง
ตามที่ตั้งใจหรือไม่**: ถ้าบรรทัด `permit https` มี match count เพิ่มขึ้นจริงตามที่ทดสอบ แสดงว่า
ACL ทำงานถูกต้อง แต่ถ้าบรรทัด `deny` มี count สูงผิดปกติ อาจแปลว่ามี traffic ที่ไม่ควรเกิดขึ้น
พยายามเข้าถึง (หรือ Server1/Client ตั้ง IP ผิด subnet)

**ล้าง counter เพื่อเริ่มนับใหม่ (มีประโยชน์มากตอน troubleshoot สด):**
```
DIST-SW3# clear access-list counters SALES-TO-SERVERS
DIST-SW3# clear access-list counters
```

### `show ip interface` — ดูว่า interface ไหนผูก ACL ทิศทางใดอยู่

```
DIST-SW3# show ip interface Vlan30
Vlan30 is up, line protocol is up
  Internet address is 10.10.30.2/24
  Broadcast address is 255.255.255.255
  Address determined by setup command
  MTU is 1500 bytes
  Helper address is not set
  Directed broadcast forwarding is disabled
  Outgoing Common access list is not set
  Outgoing access list is not set
  Inbound  Common access list is not set
  Inbound  access list is SALES-TO-SERVERS
  Proxy ARP is enabled
  ...
```

บรรทัด **`Inbound access list is SALES-TO-SERVERS`** คือหลักฐานยืนยันว่า ACL ตัวนี้ผูกอยู่กับ
interface นี้ในทิศทาง `in` จริง — เป็นคำสั่งที่ควรใช้ตรวจสอบทุกครั้งหลัง apply ACL ใหม่
(แก้ปัญหา Pitfall 2 จาก Step 138 ได้โดยตรง)

### `show running-config | section access-list` — ดู ACL ทั้งหมดใน config รวดเดียว

```
DIST-SW3# show running-config | section access-list
ip access-list standard MGMT-ONLY-VTY
 permit 10.10.99.0 0.0.0.255
 deny   any log
ip access-list extended SALES-TO-SERVERS
 remark ==== Allow SALES -> Server1 HTTPS/SSH only ====
 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 22
 remark ==== Deny all other SALES -> SERVERS traffic, log attempts ====
 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
 remark ==== Explicit permit for everything else ====
 permit ip any any
```

> คำสั่ง `| section <keyword>` เป็นเทคนิค filter output ที่ใช้บ่อยมากในงานจริง เทียบเท่ากับ
> `| include` แต่จะดึงเอา**ทั้ง block ที่เกี่ยวข้อง** (ตั้งแต่บรรทัดแรกที่ match ไปจนถึงก่อนบรรทัด
> level เดียวกันบรรทัดต่อไป) ไม่ใช่แค่บรรทัดเดียวที่ match

### `log` keyword — ดู Traffic ที่ Match แบบ Real-time

การใส่ `log` ต่อท้าย ACE จะทำให้ IOS ส่ง syslog message ทุกครั้งที่มี packet match บรรทัดนั้น
(มีประโยชน์มากสำหรับ deny statement เพื่อดูว่าใครพยายามเข้าถึงสิ่งที่ไม่ได้รับอนุญาต)

```
DIST-SW3(config)# ip access-list extended SALES-TO-SERVERS
DIST-SW3(config-ext-nacl)# 50 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
```

**ตัวอย่าง syslog message ที่เกิดขึ้นเมื่อมี traffic ถูก deny และ log:**
```
*Sep 26 14:05:12.221: %SEC-6-IPACCESSLOGP: list SALES-TO-SERVERS denied tcp
10.10.10.55(51022) -> 10.10.30.20(3389), 1 packet
```

จาก message นี้อ่านได้ว่า: PC ที่ IP `10.10.10.55` (SALES) พยายามเชื่อมต่อ RDP (port 3389)
ไปที่ `10.10.30.20` (Server2) และถูก ACL `SALES-TO-SERVERS` deny ไป 1 packet — ข้อมูลนี้
มีประโยชน์มากทั้งสำหรับ **Security Monitoring** (ตรวจจับความผิดปกติ) และ **Troubleshooting**
(ผู้ใช้แจ้งว่าเข้าถึงไม่ได้ แล้วเราต้องยืนยันว่าเป็นเพราะ ACL block จริงหรือปัญหาอื่น)

> **ข้อควรระวัง**: `log` ใช้ CPU มากกว่า ACE ที่ไม่มี `log` (ต้อง generate syslog ทุก match)
> ไม่ควรใส่ `log` บน ACE ที่คาดว่าจะมี match จำนวนมากอย่างต่อเนื่อง (เช่น `permit` บรรทัดที่รับ
> traffic ปกติทั่วไป) ควรใส่เฉพาะ `deny` ที่ต้องการเฝ้าดู หรือใช้ระหว่าง troubleshoot ชั่วคราวเท่านั้น

### สรุปคำสั่ง Verify/Troubleshoot ACL ทั้งหมด

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show access-lists [name]` | เนื้อหา ACE ทั้งหมด + match counter |
| `show ip access-lists [name]` | เหมือนข้างบน (alias, เจาะจง IP ACL) |
| `show ip interface <intf>` | ACL ที่ผูกกับ interface และทิศทาง (in/out) |
| `show running-config \| section access-list` | ACL configuration ทั้งหมดใน running-config |
| `clear access-list counters [name]` | รีเซ็ต match counter เพื่อเริ่มนับใหม่ |
| `debug ip packet [access-list <name>]` | ดู real-time packet processing (ใช้อย่างระมัดระวัง กิน CPU สูง) |

---

## Step 140 — Lab ใหญ่: Security Policy เต็มรูปแบบด้วย ACL สำหรับ Enterprise Lab

### Security Policy ที่ต้อง Implement

| # | Requirement |
|---|---|
| 1 | จำกัด VTY/SSH Management Access ให้เข้าได้เฉพาะจาก **MGMT VLAN (`10.10.99.0/24`)** บนทุก Switch/Router |
| 2 | จำกัด **SALES VLAN (10)** ไม่ให้เข้าถึง **SERVERS VLAN (30)** ยกเว้น **HTTPS ไปที่ Server1 (`10.10.30.10`)** เท่านั้น |
| 3 | อนุญาตให้ **VOICE VLAN (20)** เข้าถึง **SERVERS VLAN (30)** ได้เต็มที่เฉพาะ **port ของ Voice-Signaling Application (SIP, 5060/TCP+UDP)** |

### ภาพรวม Topology ที่เกี่ยวข้องกับ Lab นี้

```
                         CORE-SW1 <===LACP Po1===> CORE-SW2
                        /                                  \
                DIST-SW1/DIST-SW2                    DIST-SW3/DIST-SW4
                (VLAN 10 SALES, 20 VOICE)            (VLAN 30 SERVERS, 40 WIFI)
                       │                                     │
              ACCESS-SW1/2 (10,20,99)              ACCESS-SW3/4 (30,40,99)
                       │                                     │
              PC1-PC5 (SALES)                        Server1 10.10.30.10  <- HTTPS target
              Phone/AP (VOICE)                       Server2 10.10.30.20
                                                       Server3 10.10.30.30
```

ACL สำหรับ Requirement #2 และ #3 (traffic ระหว่าง VLAN) จะ apply บน **DIST-SW3** ที่ SVI
VLAN 30 (SERVERS) แบบ `in` เหมือนหลักการที่อธิบายไว้ใน Step 136 — เพราะเป็นจุดเดียวที่
traffic จากทั้ง SALES และ VOICE ต้องผ่านก่อนไปถึง SERVERS

### ส่วนที่ 1 — Standard ACL: จำกัด VTY เฉพาะ MGMT VLAN (Apply ทุก Switch)

```
! ทำเหมือนกันทุกตัว: CORE-SW1, CORE-SW2, DIST-SW1-4, ACCESS-SW1-4
ip access-list standard MGMT-ONLY-VTY
 remark ==== Allow SSH/Telnet management only from MGMT VLAN 99 ====
 permit 10.10.99.0 0.0.0.255
 deny   any log
!
line vty 0 15
 access-class MGMT-ONLY-VTY in
 transport input ssh
 login local
 exec-timeout 10 0
```

### ส่วนที่ 2 — Extended ACL: SALES → SERVERS (เฉพาะ HTTPS ไปที่ Server1) — Apply บน DIST-SW3

```
ip access-list extended SALES-TO-SERVERS
 remark ==== Requirement 2: SALES can reach Server1 via HTTPS only ====
 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
 remark ==== Deny all other SALES -> SERVERS traffic, log for security review ====
 deny   ip  10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
```

### ส่วนที่ 3 — Extended ACL: VOICE → SERVERS (เต็มที่เฉพาะ Voice-Signaling Port 5060) — เพิ่มเข้า ACL เดียวกัน

```
ip access-list extended SALES-TO-SERVERS
 remark ==== Requirement 3: VOICE full access to SERVERS for SIP signaling (5060) ====
 permit tcp 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 eq 5060
 permit udp 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 eq 5060
 remark ==== Deny all other VOICE -> SERVERS traffic, log for security review ====
 deny   ip  10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 log
 remark ==== Explicit permit for everything else (WIFI, MGMT, unrelated traffic) ====
 permit ip any any
```

> **หมายเหตุเรื่องลำดับ**: ต้องเรียงบรรทัด `permit` ของ Requirement #2 และ #3 (specific ที่สุด)
> ไว้ **ก่อน** บรรทัด `deny` ของแต่ละ VLAN เสมอ ตามกฎ Step 131 — ในตัวอย่างนี้เราแยก `deny`
> เป็นสองบรรทัด (แยกตาม source VLAN) เพื่อให้ log message แยกแยะได้ว่า traffic ที่ถูก block
> มาจาก SALES หรือ VOICE ได้ชัดเจนกว่าการรวมเป็น `deny ip any 10.10.30.0 0.0.0.255` บรรทัดเดียว

### ACL ฉบับสมบูรณ์พร้อม Sequence Number บน DIST-SW3

```
DIST-SW3# show ip access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Requirement 2: SALES can reach Server1 via HTTPS only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443
    30 remark ==== Deny all other SALES -> SERVERS traffic, log for security review ====
    40 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log
    50 remark ==== Requirement 3: VOICE full access to SERVERS for SIP signaling (5060) ====
    60 permit tcp 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 eq 5060
    70 permit udp 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 eq 5060
    80 remark ==== Deny all other VOICE -> SERVERS traffic, log for security review ====
    90 deny ip 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 log
    100 remark ==== Explicit permit for everything else ====
    110 permit ip any any
```

### Apply เข้า Interface (พร้อม Safety Net `reload in`)

```
DIST-SW3# reload in 10
Reload scheduled for 15:10:00 UTC Sat Sep 26 2026 (in 10 minutes) by admin
Proceed with reload? [confirm]

DIST-SW3# configure terminal
DIST-SW3(config)# interface Vlan30
DIST-SW3(config-if)# description ** SERVERS VLAN 30 SVI - inbound security policy applied **
DIST-SW3(config-if)# ip access-group SALES-TO-SERVERS in
DIST-SW3(config-if)# end

! ทดสอบ management access ยังใช้ได้ปกติผ่าน session ใหม่ก่อนยกเลิก reload
DIST-SW3# reload cancel
Reload scheduled for 15:10:00 UTC Sat Sep 26 2026 is cancelled.

DIST-SW3# copy running-config startup-config
```

### การทดสอบ (Verification Test) — Blocked vs Permitted Traffic

**Test 1: SALES → Server1 HTTPS (ควรผ่าน — Requirement 2)**
```
! จาก PC1 (10.10.10.11) ทดสอบเปิด HTTPS ไปที่ Server1
PC1> curl -k https://10.10.30.10
HTTP/1.1 200 OK ...   <- สำเร็จ
```

**Test 2: SALES → Server1 SSH (ควรถูก block — ไม่อยู่ใน Requirement 2)**
```
PC1> ssh admin@10.10.30.10
% Connection timed out   <- ถูก deny โดย ACL
```

**Test 3: SALES → Server2 HTTPS (ควรถูก block — Requirement 2 อนุญาตแค่ Server1 เท่านั้น)**
```
PC1> curl -k https://10.10.30.20
curl: (28) Failed to connect to 10.10.30.20 port 443: Connection timed out
```

**Test 4: VOICE (Phone) → SERVERS port 5060 (ควรผ่าน — Requirement 3)**
```
Phone1 (10.10.20.21)> sip-signaling-test 10.10.30.30:5060
SIP 200 OK   <- สำเร็จ (SIP signaling ผ่าน)
```

**Test 5: VOICE → SERVERS port อื่นที่ไม่ใช่ 5060 (ควรถูก block — Requirement 3 จำกัดแค่ signaling port)**
```
Phone1 (10.10.20.21)> curl -k https://10.10.30.10
curl: (28) Failed to connect to 10.10.30.10 port 443: Connection timed out
```

### ตรวจสอบผลลัพธ์ด้วย Match Counter หลังทดสอบ

```
DIST-SW3# show access-lists SALES-TO-SERVERS
Extended IP access list SALES-TO-SERVERS
    10 remark ==== Requirement 2: SALES can reach Server1 via HTTPS only ====
    20 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443 (1 match)
    30 remark ==== Deny all other SALES -> SERVERS traffic, log for security review ====
    40 deny ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255 log (2 matches)
    50 remark ==== Requirement 3: VOICE full access to SERVERS for SIP signaling (5060) ====
    60 permit tcp 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 eq 5060 (0 matches)
    70 permit udp 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 eq 5060 (1 match)
    80 remark ==== Deny all other VOICE -> SERVERS traffic, log for security review ====
    90 deny ip 10.10.20.0 0.0.0.255 10.10.30.0 0.0.0.255 log (1 match)
    100 remark ==== Explicit permit for everything else ====
    110 permit ip any any (48210 matches)
```

**อ่านผลลัพธ์**: line 20 มี 1 match (Test 1 ที่ HTTPS ผ่านสำเร็จ), line 40 มี 2 matches
(Test 2 SSH + Test 3 HTTPS ไป Server2 ที่ถูก block), line 70 มี 1 match (Test 4 SIP over
UDP ที่ผ่านสำเร็จ), line 90 มี 1 match (Test 5 ที่ถูก block) — ตัวเลขทั้งหมดตรงกับพฤติกรรม
ที่ Security Policy ต้องการทุกประการ ยืนยันว่า ACL ทำงานถูกต้อง 100%

### Syslog ที่ควรเห็นจากการทดสอบ (เพราะใส่ `log` ไว้บน deny statement)

```
*Sep 26 15:22:01.114: %SEC-6-IPACCESSLOGP: list SALES-TO-SERVERS denied tcp
10.10.10.11(52210) -> 10.10.30.10(22), 1 packet
*Sep 26 15:22:45.887: %SEC-6-IPACCESSLOGP: list SALES-TO-SERVERS denied tcp
10.10.10.11(52233) -> 10.10.30.20(443), 1 packet
*Sep 26 15:23:30.502: %SEC-6-IPACCESSLOGP: list SALES-TO-SERVERS denied tcp
10.10.20.21(52260) -> 10.10.30.10(443), 1 packet
```

### สรุปการปฏิบัติตาม Security Policy ครบทั้ง 3 ข้อ

| # | Requirement | Implementation | ยืนยันด้วย |
|---|---|---|---|
| 1 | VTY เข้าได้เฉพาะจาก MGMT VLAN | Standard ACL `MGMT-ONLY-VTY` ผูกกับ `access-class` ทุก Switch | ทดสอบ SSH จาก VLAN อื่น → refused |
| 2 | SALES เข้า SERVERS ได้เฉพาะ HTTPS ไป Server1 | Extended ACL `SALES-TO-SERVERS` line 20+40 บน DIST-SW3 SVI VLAN 30 (`in`) | Test 1-3 + match counter line 20/40 |
| 3 | VOICE เข้า SERVERS เต็มที่เฉพาะ port SIP 5060 | Extended ACL `SALES-TO-SERVERS` line 60-90 บน DIST-SW3 SVI VLAN 30 (`in`) | Test 4-5 + match counter line 60/70/90 |

---

## แบบฝึกหัดทวนความเข้าใจ Part 14

1. เพราะเหตุใด ACL ที่มีแต่ `deny` statement (ไม่มี `permit` ปลายรายการ) และตั้งใจให้ traffic
   ที่เหลือผ่านได้ตามปกติ จึงมักทำให้ traffic ทั้งหมดถูกบล็อกโดยไม่ตั้งใจ?
2. Standard ACL และ Extended ACL ต่างกันอย่างไรในเรื่องสิ่งที่สามารถ match ได้ และควร apply
   ใกล้ Source หรือ Destination ตามลำดับ?
3. คำสั่ง `access-list 100 permit tcp 10.10.10.0 0.0.0.255 host 10.10.30.10 eq 443` มี
   Wildcard Mask อะไร และหมายถึงกลุ่ม Source IP ใด?
4. ก่อนจะ apply ACL ที่มีผลต่อ VTY/Management Access ควรใช้คำสั่งใดเป็น Safety Net และทำงาน
   อย่างไร?
5. คำสั่งใดใช้ตรวจสอบว่า ACL ตัวใดถูกผูกกับ interface ใดในทิศทาง `in` หรือ `out` และคำสั่งใด
   ใช้ดู match counter ของแต่ละ ACE?

**เฉลย:**
1. เพราะทุก ACL มี **Implicit Deny (`deny ip any any`)** ซ่อนอยู่ท้ายรายการเสมอ (แม้ไม่แสดงใน
   `show run`) — packet ที่ไม่ match บรรทัดใดเลยที่เขียนไว้จะตกไปโดน Implicit Deny แทน ถ้าไม่ได้
   เติม `permit` ปลายรายการเพื่อดักไว้ก่อน
2. Standard ACL match ได้เฉพาะ Source IP เท่านั้น ควร apply ใกล้ **Destination** ที่สุด
   ส่วน Extended ACL match ได้ทั้ง Source, Destination, Protocol, Port ควร apply ใกล้
   **Source** ที่สุดเพื่อ drop traffic ที่ไม่ต้องการตั้งแต่ทางเข้า ประหยัด bandwidth บน backbone
3. Wildcard Mask คือ `0.0.0.255` หมายถึงกลุ่ม Source IP ทั้ง subnet `10.10.10.0/24`
   (SALES VLAN ทั้งหมด) เพราะ 3 octet แรกต้อง match ตรง (bit 0) และ octet สุดท้ายไม่สนใจ (bit 1)
4. คำสั่ง `reload in <นาที>` — ตั้งเวลาให้อุปกรณ์ reload อัตโนมัติกลับไปใช้ startup-config เดิม
   ถ้าไม่ได้ยกเลิกด้วย `reload cancel` ภายในเวลาที่กำหนด ใช้เป็น Safety Net เผื่อ ACL ที่ apply
   ไปทำให้ Management Access หลุด (ต้องยังไม่ `write memory` ทับ config เดิม)
5. `show ip interface <interface>` ดูว่า ACL ผูกอยู่ทิศทางใด (บรรทัด "Inbound access list is ...")
   และ `show access-lists [name]` ใช้ดู match counter ของแต่ละ ACE

---

## สรุป Part 14

Part นี้ครอบคลุม **Access Control List** ตั้งแต่แนวคิดพื้นฐาน (Top-Down Processing, Implicit
Deny, ความสำคัญของลำดับ statement), การทวน Wildcard Mask ที่เชื่อมกับ Part 7, Standard ACL
และ Extended ACL พร้อม worked example จริงในบริบทของ Enterprise Lab, การใช้ Named ACL
กับ Sequence Number เพื่อแก้ไข config อย่างปลอดภัย, หลักการ apply ACL เข้า interface ให้ถูก
ทิศทาง, ข้อผิดพลาดที่พบบ่อยพร้อมเทคนิค `reload in` ป้องกันตัวเองล็อกตัวเองออก, คำสั่ง verify/
troubleshoot ที่ใช้งานจริงทุกวัน และปิดท้ายด้วย Lab ใหญ่ที่ implement Security Policy ครบ
ทั้ง 3 ข้อของ Enterprise Lab พร้อมการทดสอบและยืนยันผลลัพธ์ด้วย match counter จริง ✅

ACL คือทักษะพื้นฐานที่จะใช้ควบคู่กับทุก Feature ด้านความปลอดภัยตลอดหลักสูตรนี้ ทั้งใน
Route-map, QoS classification, VPN traffic selector, และ Zone-Based Firewall

**พร้อมสำหรับ Part 15**: เราจะเรียนรู้ **NAT/PAT (Network Address Translation)** — วิธีที่
Private IP Address (10.10.0.0/16) ใน Enterprise Lab ของเราสามารถออก Internet ผ่าน WAN-EDGE
ได้จริง ทั้ง Static NAT, Dynamic NAT, และ PAT (NAT Overload) ที่ใช้กันมากที่สุดในโลกจริง

**ไปต่อ:** [Part 15 — NAT/PAT →](part-015-nat-pat.md)
