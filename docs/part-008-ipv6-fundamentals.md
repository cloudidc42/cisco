# Part 8 — IPv6 Fundamentals
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 71–80 จาก 1000**

> ต่อจาก [Part 7 — IPv4 Addressing & Subnetting](part-007-ipv4-addressing-subnetting.md) ที่วาง
> IPv4 Subnetting ของทุก VLAN ใน Lab Topology หลักไว้แล้ว Part นี้เราจะเพิ่ม **IPv6** เข้าไปแบบ
> **Dual-Stack** (IPv4 + IPv6 วิ่งพร้อมกันบน interface เดียวกัน) ให้กับทุกอุปกรณ์ใน Lab
> ตั้งแต่ทฤษฎี address format, ประเภท address, การสร้าง Interface ID แบบ EUI-64, วิธี assign
> address หลายแบบ, Neighbor Discovery Protocol (แทนที่ ARP), ไปจนถึง Static Routing และคำสั่ง
> verify — ปิดท้ายด้วย Lab จริงที่ config IPv6 ให้ CORE-SW1, CORE-SW2 และ WAN Edge ของหลักสูตร

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 71 | ทำไมต้อง IPv6 และรูปแบบ Address |
| 72 | ประเภทของ IPv6 Address (Global Unicast, Link-Local, ULA, Multicast) |
| 73 | Well-Known Multicast Address และ Solicited-Node Multicast |
| 74 | EUI-64 — การสร้าง Interface ID จาก MAC Address |
| 75 | วิธีกำหนด IPv6 Address (Static, EUI-64, SLAAC, DHCPv6) |
| 76 | การ Config IPv6 บน Cisco IOS |
| 77 | Neighbor Discovery Protocol (NDP) |
| 78 | IPv6 Static Routing เบื้องต้น |
| 79 | คำสั่ง Verify และ Troubleshoot IPv6 |
| 80 | Lab: Dual-Stack IPv6 บน Core Switch และ WAN Edge |

---

## Step 71 — ทำไมต้อง IPv6 และรูปแบบ Address

### ปัญหา IPv4 Address Exhaustion

IPv4 มีขนาด 32 bit รองรับ address ได้สูงสุดทางทฤษฎี **4,294,967,296 (~4.3 พันล้าน)** address
ซึ่งฟังดูเยอะ แต่เมื่อหักส่วนที่ใช้ไม่ได้จริง (Private range ตาม RFC 1918, Multicast, Reserved,
Loopback ฯลฯ) และเทียบกับจำนวนอุปกรณ์ที่ต้องการ IP บน Internet ทั่วโลก (มือถือ, IoT, Cloud VM,
Container) ที่เพิ่มขึ้นแบบทวีคูณ ปริมาณ address ที่ใช้งานได้จริงจึงไม่พอ

| เหตุการณ์ | วันที่ |
|---|---|
| IANA จ่าย IPv4 block สุดท้ายให้ RIR ทั้ง 5 แห่ง | 3 กุมภาพันธ์ 2011 |
| APNIC (เอเชียแปซิฟิก) หมด free pool | เมษายน 2011 |
| RIPE NCC (ยุโรป) หมด free pool | กันยายน 2012 |
| ARIN (อเมริกาเหนือ) หมด free pool | กันยายน 2015 |
| LACNIC (ลาตินอเมริกา) หมด free pool | 2020 |
| AFRINIC (แอฟริกา) หมด free pool | 2021 |

เมื่อ IPv4 หมด ตลาดโลกแก้ปัญหาด้วยเทคนิคยืดอายุ เช่น **NAT/PAT** (Part 15), **CIDR**
(ใช้ address อย่างมีประสิทธิภาพขึ้น — Part 7), และ **CGNAT** ฝั่ง ISP แต่ทั้งหมดนี้เป็นแค่
การประวิงเวลา ทางออกถาวรคือ **IPv6** ซึ่งถูกออกแบบมาตั้งแต่ปี 1998 (RFC 2460 และปัจจุบันคือ
RFC 8200) ให้มี address space ใหญ่พอสำหรับอนาคตอีกหลายสิบปี

### ขนาด Address Space: 128 bit

IPv6 Address มีขนาด **128 bit** เทียบกับ IPv4 ที่มี 32 bit เท่ากับมีจำนวน address สูงสุดถึง

```
2^128 = 340,282,366,920,938,463,463,374,607,431,768,211,456
      ≈ 3.4 × 10^38 address
```

ตัวเลขนี้มากพอที่จะแจก IP ให้กับทุกอะตอมบนพื้นผิวโลกได้หลายรอบ — ในทางปฏิบัติ IPv6 จึงถูกออกแบบ
ให้ "สิ้นเปลือง" address ได้อย่างสบายใจ (เช่น ทุก subnet ปกติจะใช้ /64 เสมอ แม้จะมี host แค่ไม่กี่ตัว)
เพื่อแลกกับความง่ายในการออกแบบ, aggregation, และการทำงานของ Neighbor Discovery / SLAAC

| คุณสมบัติ | IPv4 | IPv6 |
|---|---|---|
| ขนาด Address | 32 bit | 128 bit |
| รูปแบบการเขียน | Dotted Decimal (4 octet) | Hexadecimal (8 group, 16 bit/group) |
| จำนวน Address สูงสุด | ~4.3 พันล้าน | ~3.4 × 10^38 |
| Broadcast | มี | **ไม่มี** (ใช้ Multicast แทน) |
| Header | ซับซ้อนกว่า, มี checksum | เรียบง่ายกว่า, fixed 40 byte, ไม่มี checksum (ให้ Layer บนจัดการ) |
| Address Resolution | ARP (Broadcast) | NDP (Multicast, ผ่าน ICMPv6) |
| Configuration | Static / DHCP | Static / EUI-64 / **SLAAC** / DHCPv6 |
| IPSec | Optional | ออกแบบมาให้รองรับตั้งแต่ต้น (แต่ปัจจุบัน optional เหมือนกันในทางปฏิบัติ) |

### รูปแบบการเขียน IPv6 Address

IPv6 Address เขียนเป็นเลขฐาน 16 (Hexadecimal) แบ่งเป็น **8 กลุ่ม กลุ่มละ 16 bit (Hextet)**
คั่นด้วยเครื่องหมาย colon (`:`) รูปแบบเต็ม (Full/Unabbreviated Form):

```
2001:0DB8:0000:0010:0000:0000:0000:0001
└──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
 16bit×8 กลุ่ม = 128 bit
```

- แต่ละกลุ่มมี 4 hex digit (0-9, A-F) — ไม่สนตัวพิมพ์เล็ก/ใหญ่ (Cisco IOS แสดงผลเป็นตัวพิมพ์เล็กเสมอ)
- คั่นด้วย `:` ระหว่างกลุ่ม (ต่างจาก IPv4 ที่คั่นด้วย `.`)

### กฎการย่อ (Compression Rules) — ต้องจำให้แม่น

**กฎที่ 1 — Leading Zero Omission**: ตัดเลข 0 ที่นำหน้าภายในแต่ละกลุ่มออกได้ (แต่ถ้ากลุ่มนั้นเป็น
`0000` ทั้งหมด ให้เหลืออย่างน้อย 1 ตัว คือ `0`)

```
2001:0DB8:0000:0010:0000:0000:0000:0001
→ 2001:DB8:0:10:0:0:0:1        (ตัด leading zero ของแต่ละกลุ่ม)
```

**กฎที่ 2 — Double Colon (`::`)**: ย่อกลุ่มที่เป็น `0000` ติดต่อกันตั้งแต่ 1 กลุ่มขึ้นไปให้เหลือแค่
`::` ได้ **แค่ 1 ครั้งต่อ address เท่านั้น** (เพราะถ้าใช้ 2 ครั้ง จะไม่รู้ว่าแต่ละฝั่งของ `::`
มีกี่กลุ่ม ทำให้ decompress กลับไม่ได้ — เป็นข้อสอบยอดฮิตของ CCNA)

```
2001:DB8:0:10:0:0:0:1
→ 2001:DB8:0:10::1              (ย่อกลุ่ม 0:0:0 ที่ติดกัน 3 กลุ่มเป็น ::)
```

> **ตัวอย่างที่ผิด**: `2001:DB8::10::1` — ใช้ `::` สองครั้ง **ผิดกฎเสมอ** เพราะ router ไม่รู้ว่า
> จะเติม 0 กี่ตัวให้แต่ละช่อง (ambiguous) — Cisco IOS จะปฏิเสธ input แบบนี้ทันที

### ตัวอย่างการย่อ Address ที่พบบ่อย

| Full Form | Compressed Form | หมายเหตุ |
|---|---|---|
| `2001:0DB8:0000:0000:0000:0000:0000:0001` | `2001:DB8::1` | Global Unicast ตัวอย่างมาตรฐาน (documentation prefix) |
| `FE80:0000:0000:0000:021A:2FFF:FE3B:4C5D` | `FE80::21A:2FFF:FE3B:4C5D` | Link-Local address |
| `0000:0000:0000:0000:0000:0000:0000:0001` | `::1` | Loopback address (เทียบเท่า `127.0.0.1` ของ IPv4) |
| `0000:0000:0000:0000:0000:0000:0000:0000` | `::` | Unspecified address (เทียบเท่า `0.0.0.0` ของ IPv4) |
| `FF02:0000:0000:0000:0000:0000:0000:0001` | `FF02::1` | Multicast all-nodes (จะเรียนละเอียด Step 73) |

### Prefix Length แทน Subnet Mask

IPv6 **ไม่มี Subnet Mask แบบ dotted decimal** เหมือน IPv4 — ใช้ **Prefix Length** แบบ CIDR
(`/64`) เท่านั้น เช่น `2001:DB8:10:10::1/64` หมายถึง 64 bit แรกคือ Network Prefix
(`2001:DB8:10:10::/64`) และ 64 bit หลังคือ Interface ID — **มาตรฐาน Cisco/Enterprise คือใช้
`/64` กับทุก subnet ที่มี host เชื่อมต่อเสมอ** (จำเป็นสำหรับ SLAAC และ EUI-64 ที่ต้องการ 64 bit
สำหรับ Interface ID พอดี)

---

## Step 72 — ประเภทของ IPv6 Address

IPv6 แบ่ง Address ออกเป็น 3 ประเภทหลัก (ไม่มี Broadcast) ตาม RFC 4291:

| ประเภท | Prefix | ลักษณะการใช้งาน | เทียบเท่า IPv4 |
|---|---|---|---|
| **Unicast** | หลาย range | ส่งถึงปลายทางเดียว | Unicast IPv4 |
| **Multicast** | `ff00::/8` | ส่งถึงกลุ่มปลายทางที่ join group | Multicast IPv4 (224.0.0.0/4) |
| **Anycast** | ใช้ range เดียวกับ Unicast | ส่งถึง node ที่ "ใกล้ที่สุด" ใน group เดียวกัน (ตาม routing metric) | ไม่มีใน IPv4 โดยตรง |

Unicast ยังแบ่งย่อยเป็น 3 แบบที่สำคัญที่สุดสำหรับ CCNA:

### 1. Global Unicast Address (GUA) — `2000::/3`

- Address ที่ **routable บน Internet สาธารณะ** เทียบเท่า Public IP ของ IPv4
- Range ปัจจุบันที่ ISP จ่ายจริงเริ่มต้นด้วย `2000::/3` (คือ address ที่ 3 bit แรกเป็น `001`
  ครอบคลุมตั้งแต่ `2000::` ถึง `3FFF:FFFF:...`)
- Format มาตรฐาน: **Global Routing Prefix** (48 bit จาก ISP) + **Subnet ID** (16 bit สำหรับแบ่ง
  VLAN/subnet ภายในองค์กร) + **Interface ID** (64 bit)
- `2001:DB8::/32` คือ prefix สงวนไว้สำหรับ**เอกสาร/ตัวอย่างเท่านั้น** (RFC 3849) ห้ามใช้จริงบน
  Internet — เราจะใช้ prefix นี้ในการจำลองลิงก์ WAN ที่ "เหมือนสาธารณะ" ใน Lab (Step 76, 80)
  เทียบเท่ากับที่ Part 7 ใช้ `203.0.113.0/24` (TEST-NET-3) สำหรับ WAN link ฝั่ง IPv4

### 2. Link-Local Address (LLA) — `fe80::/10`

- **บังคับมีทุก interface ที่เปิดใช้ IPv6 เสมอ โดยอัตโนมัติ** ไม่ว่าจะ config GUA/ULA หรือไม่ก็ตาม
- ใช้สื่อสารเฉพาะภายใน **Link เดียวกัน (segment เดียวกัน)** เท่านั้น **ไม่ถูก route ข้าม subnet**
- เป็น address ที่ NDP, OSPFv3, HSRP for IPv6 ฯลฯ ใช้เป็น next-hop เสมอ (แทนที่จะใช้ GUA)
- สร้างอัตโนมัติจาก prefix `FE80::/10` + Interface ID (ปกติสร้างด้วยวิธี EUI-64 หรือ random
  แบบ Cisco IOS จะ derive จาก MAC ของ interface นั้นโดย default)

```
Router# show ipv6 interface Vlan10 | include link-local
  IPv6 is enabled, link-local address is FE80::21A:2FFF:FE3B:4C5D
```

### 3. Unique Local Address (ULA) — `fc00::/7`

- เทียบเท่า **RFC 1918 Private Address ของ IPv4** (`10.x.x.x`, `172.16-31.x.x`, `192.168.x.x`)
- **ไม่ routable บน Internet สาธารณะ** ใช้ได้เฉพาะภายในองค์กร/lab
- Range เต็มคือ `fc00::/7` แต่แบ่งเป็น 2 ส่วนตาม bit ที่ 8 (L-bit):
  - `fc00::/8` — L-bit = 0 (สงวนไว้สำหรับอนาคต ยังไม่มีมาตรฐานกำหนดวิธี allocate)
  - `fd00::/8` — L-bit = 1 (**Locally Assigned** — ใช้งานได้จริงในปัจจุบัน)
- **หลักสูตรนี้เลือกใช้ ULA prefix `fd00:10:10::/48`** สำหรับ Lab ทั้งหมด (เลข `10:10` ตั้งใจให้
  จำง่าย ล้อกับ IPv4 subnet `10.10.x.0/24` ที่ประกาศไว้ใน Part 1)
- โครงสร้าง ULA: `fd` + Global ID สุ่ม 40 bit (ในทางปฏิบัติ generate แบบสุ่มเพื่อไม่ให้ชนกันเวลา
  merge network) + Subnet ID 16 bit + Interface ID 64 bit — ในหลักสูตรนี้ใช้ Global ID คงที่
  `00:10:10` เพื่อความง่ายในการอ่านเทียบกับ IPv4

### 4. Multicast Address — `ff00::/8`

- ทุก address ที่ขึ้นต้นด้วย `ff` คือ Multicast เสมอ
- **ไม่มี Broadcast ใน IPv6** — ทุกอย่างที่เคยใช้ Broadcast ใน IPv4 (เช่น ARP, DHCP Discover)
  ถูกแทนที่ด้วย **Multicast แบบเจาะจงกลุ่ม** ทำให้อุปกรณ์ที่ไม่เกี่ยวข้องไม่ต้องประมวลผล packet
  ที่ไม่ใช่ของตัวเอง (ลด CPU overhead เมื่อเทียบกับ Broadcast ที่ทุกเครื่องต้อง process)
- รายละเอียด well-known multicast address ดู Step 73

### ตารางสรุปประเภท Address ที่ใช้ตลอดหลักสูตร

| ประเภท | Prefix | ตัวอย่างในหลักสูตรนี้ | Scope |
|---|---|---|---|
| Global Unicast | `2000::/3` | `2001:db8:203:113::2/64` (WAN link จำลอง) | Global (Internet) |
| Link-Local | `fe80::/10` | auto-generated ทุก interface | Link เดียวกันเท่านั้น |
| Unique Local (ULA) | `fd00::/8` | `fd00:10:10:10::/64` (VLAN 10 SALES) | Site/องค์กร (ไม่ routable ออก Internet) |
| Multicast | `ff00::/8` | `ff02::1`, `ff02::5` | ตาม scope ของ address (link-local/site-local) |
| Loopback | `::1/128` | testing บนอุปกรณ์เอง | Host เดียว |
| Unspecified | `::/128` | ใช้เป็น source ตอนยังไม่มี address (DAD) | - |

---

## Step 73 — Well-Known Multicast Address และ Solicited-Node Multicast

### Multicast Scope

Multicast address ของ IPv6 มี field **Scope** (4 bit ใน address) กำหนดขอบเขตการกระจาย
ที่พบบ่อยที่สุดคือ:

| Scope | ตัวอย่าง Prefix | ความหมาย |
|---|---|---|
| Interface-Local (1) | `ff01::/16` | จำกัดอยู่ใน interface เดียว (loopback เท่านั้น) |
| **Link-Local (2)** | `ff02::/16` | จำกัดอยู่ใน Link/Segment เดียวกัน — **ใช้บ่อยที่สุด** |
| Site-Local (5) | `ff05::/16` | จำกัดอยู่ใน "site" เดียวกัน (ไม่ค่อยใช้จริง) |
| Global (E) | `ffe::/16`* | กระจายได้ทั่ว Internet |

### Well-Known Multicast Address ที่ต้องจำ (ระดับ Link-Local: `ff02::/16`)

| Multicast Address | ชื่อ | ใครฟัง (Join Group) |
|---|---|---|
| `ff02::1` | **All-Nodes** | **ทุก IPv6 node** บน link (host + router) — แทนที่ ARP Broadcast/Ping Broadcast |
| `ff02::2` | **All-Routers** | ทุก IPv6 **router** บน link เท่านั้น (host ไม่ join) |
| `ff02::5` | OSPFv3 All-SPF-Routers | ทุก router ที่รัน OSPFv3 (จะเรียนใน Part 27) |
| `ff02::6` | OSPFv3 All-DR-Routers | เฉพาะ DR/BDR ของ OSPFv3 |
| `ff02::9` | RIPng Routers | ทุก router ที่รัน RIPng |
| `ff02::a` | EIGRP (IPv6) Routers | ทุก router ที่รัน EIGRP for IPv6 (จะเรียนใน Part 28) |
| `ff02::1:2` | All-DHCP-Relay-Agents-and-Servers | DHCPv6 Relay/Server |
| `ff02::1:ffXX:XXXX` | **Solicited-Node Multicast** | ดูรายละเอียดด้านล่าง (ใช้แทน ARP Request) |

```
CORE-SW1# ping ff02::1
Output Interface: Vlan10
Type escape sequence to abort.
Sending 1, 100-byte ICMP Echos to FF02::1, timeout is 2 seconds:
Reply to request 0 received from FE80::21A:2FFF:FE3B:4C5D, 1 ms
```

> `ping ff02::1` เทียบเท่ากับการ ping broadcast address ของ IPv4 (`255.255.255.255`) แต่ทำงานได้
> ถูกต้องและปลอดภัยกว่า เพราะมีแค่ IPv6-enabled node เท่านั้นที่ตอบ

### Solicited-Node Multicast Address — กลไกสำคัญที่แทนที่ ARP

ทุก Unicast/Anycast address ของ IPv6 จะมี **Solicited-Node Multicast Address** คู่กันเสมอ
(สร้างอัตโนมัติโดย IOS) ใช้สำหรับกระบวนการ **Neighbor Solicitation (NS)** แทนที่ ARP Request
ของ IPv4 — ข้อดีคือ router/host อื่นที่ไม่เกี่ยวข้องจะไม่ต้องประมวลผล packet นี้เลย (ต่างจาก
ARP Broadcast ที่ทุกเครื่องต้อง wake up มา process)

**วิธีคำนวณ**: นำ **24 bit ท้ายสุด (low-order)** ของ Unicast address มาต่อท้าย prefix คงที่
`ff02::1:ff00:0/104`

```
Prefix คงที่ (104 bit):     ff02:0000:0000:0000:0000:0001:ff00:0000
                                                              └────┘
                                                         แทนที่ 24 bit นี้
```

**ตัวอย่าง**: host มี Unicast address `2001:db8:10:10::21a:2fff:fe3b:4c5d`

1. เอา 24 bit สุดท้าย (6 hex digit สุดท้าย) มา: `3b:4c:5d`
2. นำไปต่อท้าย prefix `ff02::1:ff00:0000` แทนที่ 24 bit ท้าย
3. ผลลัพธ์: **`ff02::1:ff3b:4c5d`**

```
Router# show ipv6 interface Vlan10
Vlan10 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::21A:2FFF:FE3B:4C5D
  Global unicast address(es):
    2001:DB8:10:10::21A:2FFF:FE3B:4C5D, subnet is 2001:DB8:10:10::/64
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:FF3B:4C5D        <-- Solicited-Node multicast (คู่กับ Global Unicast ข้างบน)
    FF02::1:FF00:1           <-- Solicited-Node multicast (คู่กับ Link-Local)
```

สังเกตว่า address 1 ตัวจะมี Solicited-Node multicast ของตัวเอง 1 กลุ่มเสมอ (ทั้ง Link-Local
และ Global Unicast) — เมื่อ router ต้องการถาม "ใครใช้ address ปลายทางนี้" (เทียบเท่า ARP Request)
มันจะส่ง Neighbor Solicitation ไปที่ Solicited-Node multicast address นี้แทนที่จะ broadcast
ทั้ง segment ทำให้มีแค่เจ้าของ address จริงเท่านั้นที่ต้องตอบกลับ

---

## Step 74 — EUI-64: การสร้าง Interface ID จาก MAC Address

**EUI-64 (Extended Unique Identifier, 64-bit)** เป็นวิธีสร้าง **Interface ID (64 bit หลังของ
IPv6 address)** จาก **MAC Address (48 bit)** ของ interface นั้นโดยอัตโนมัติ ไม่ต้องพิมพ์ address
เต็มเอง — ใช้เมื่อ config ด้วย `ipv6 address <prefix>/64 eui-64` หรือเวลา SLAAC สร้าง
Interface ID (RA บอกแค่ prefix ส่วน Interface ID host คำนวณเองด้วยวิธีนี้)

### ขั้นตอนการแปลง MAC → EUI-64 Interface ID (ตัวอย่างเต็ม)

**MAC Address ตั้งต้น**: `00-1A-2F-3B-4C-5D` (เขียนแบบ Cisco: `001A.2F3B.4C5D`)

**ขั้นที่ 1 — แบ่ง MAC (48 bit) ออกเป็น 2 ส่วน ส่วนละ 24 bit**

```
OUI (24 bit)         Device ID (24 bit)
00-1A-2F             3B-4C-5D
```

**ขั้นที่ 2 — แทรก `FFFE` (16 bit) เข้าไปตรงกลาง** ทำให้ได้ 64 bit

```
00-1A-2F | FF-FE | 3B-4C-5D
     →   00-1A-2F-FF-FE-3B-4C-5D
```

**ขั้นที่ 3 — Flip bit ที่ 7 (U/L bit — Universal/Local bit) ของ byte แรก**

Byte แรกคือ `00` = `0000 0000` ในรูปแบบ binary — bit ที่ 7 (นับจากซ้าย เป็น bit ตำแหน่งที่ 2
จากซ้าย หรือเรียกว่า "7th bit" ตามมาตรฐาน IEEE) คือ bit ที่บอกว่า MAC นี้เป็น **Universally
Administered (0)** หรือ **Locally Administered (1)** — EUI-64 กำหนดให้ **flip (invert) bit นี้**
เสมอตอนแปลงเป็น Interface ID:

```
0000 0000   (0x00)
     ↑ flip bit นี้ (0 → 1)
0000 0010   (0x02)
```

byte แรกจาก `00` กลายเป็น `02`

**ผลลัพธ์สุดท้าย — Interface ID (64 bit)**:

```
MAC เดิม:            00-1A-2F-3B-4C-5D
หลัง insert FFFE:    00-1A-2F-FF-FE-3B-4C-5D
หลัง flip bit 7:     02-1A-2F-FF-FE-3B-4C-5D
                     ─┬─ ─┬─ ─┬─ ─┬─ ─┬─ ─┬─
Interface ID (hex groups 16 bit): 021A : 2FFF : FE3B : 4C5D
```

**เขียนเป็น IPv6 Interface ID (บีบ leading zero)**: `21a:2fff:fe3b:4c5d`

### ประกอบเป็น Full Address

ถ้า interface นี้อยู่ใน VLAN 10 ที่ใช้ prefix ULA `fd00:10:10:10::/64` ของหลักสูตรนี้:

```
Prefix:        fd00:10:10:10::/64
Interface ID:  21a:2fff:fe3b:4c5d
────────────────────────────────────────
Full Address:  fd00:10:10:10:21a:2fff:fe3b:4c5d/64
```

และ Link-Local address ของ interface เดียวกัน (ใช้ EUI-64 เดียวกันเสมอ ต่าง prefix เท่านั้น):

```
fe80::21a:2fff:fe3b:4c5d
```

> **จำง่ายๆ**: EUI-64 = "แหวก MAC ตรงกลาง ยัด FFFE เข้าไป แล้วพลิก bit ที่ 7 ของ byte แรก"
> ข้อสอบ CCNA ชอบถามให้แปลง MAC → EUI-64 Interface ID แบบนี้เป๊ะๆ ต้องฝึกทำมือให้คล่อง

---

## Step 75 — วิธีกำหนด IPv6 Address บนอุปกรณ์/Host

Cisco IOS และ Host รองรับการกำหนด IPv6 Address 4 วิธีหลัก:

### 1. Static (พิมพ์ address เต็มเอง)

```
Router(config-if)# ipv6 address 2001:DB8:10:10::1/64
```

เหมาะกับ Router/Switch/Server ที่ต้องการ address คงที่แน่นอน (infrastructure device) —
เป็นวิธีหลักที่หลักสูตรนี้ใช้กับ SVI และ WAN interface ทุกจุด

### 2. EUI-64 (Static Prefix + Auto-generate Interface ID)

```
Router(config-if)# ipv6 address 2001:DB8:10:10::/64 eui-64
```

IOS จะคำนวณ Interface ID จาก MAC ของ interface นั้นเองตามวิธี Step 74 โดยอัตโนมัติ —
ข้อดีคือพิมพ์สั้น ไม่ต้องคำนวณเอง แต่ **address จะเปลี่ยนถ้าเปลี่ยน physical interface/MAC**
ซึ่งเป็นข้อเสียสำหรับอุปกรณ์ที่ต้องการ address คงที่แน่นอน (จึงไม่ค่อยนิยมใน production
infrastructure แต่เจอบ่อยในข้อสอบ)

### 3. SLAAC (StateLess Address AutoConfiguration)

Host เรียนรู้ **prefix** จาก **Router Advertisement (RA)** ที่ router ส่งมา (ผ่าน NDP,
Step 77) แล้ว host คำนวณ Interface ID เองด้วย EUI-64 หรือ Random (Privacy Extension, RFC 4941 —
Windows/macOS/Linux สมัยใหม่ default ใช้ random เพื่อความเป็นส่วนตัว ไม่ใช้ EUI-64 ที่โยง MAC ตรงๆ)

- Router ต้องเปิด `ipv6 unicast-routing` และมี `ipv6 address` บน interface นั้น จึงจะส่ง RA
  โดยอัตโนมัติทุก 200 วินาที (default) หรือเมื่อได้รับ Router Solicitation จาก host
- Host ไม่ต้องมี DHCPv6 Server เลยก็ได้ address ใช้งานได้ — เหมาะกับ network ที่ต้องการความง่าย

### 4. DHCPv6 (Stateful / Stateless)

| แบบ | Router Advertisement Flag | Host ได้อะไรจาก DHCPv6 Server |
|---|---|---|
| **Stateful DHCPv6** | M-flag (Managed) = 1 | ได้ **ทั้ง IPv6 address และ options อื่น** (DNS, domain) — เหมือน DHCP ของ IPv4 ทุกประการ |
| **Stateless DHCPv6** | O-flag (Other config) = 1, M-flag = 0 | Address มาจาก **SLAAC** ส่วน DHCPv6 Server ให้แค่ **options เพิ่มเติม** เช่น DNS Server, NTP Server |

```
! บน Router (ทำหน้าที่ประกาศ flag ใน RA)
Router(config-if)# ipv6 nd managed-config-flag      ! ตั้ง M-flag = 1 (บอก host ให้ไปขอ DHCPv6 แบบ stateful)
Router(config-if)# ipv6 nd other-config-flag        ! ตั้ง O-flag = 1 (บอก host ให้ขอ options เพิ่มจาก DHCPv6)
```

> DHCPv6 รายละเอียดเต็ม (Server config, Relay, Pool) จะเรียนต่อเนื่องกับ DHCP ของ IPv4 ใน
> **Part 16** เพื่อไม่ให้เนื้อหาซ้ำซ้อน — Part นี้เน้นให้เข้าใจ concept และ flag ที่เกี่ยวข้อง

### ตารางเปรียบเทียบทั้ง 4 วิธี

| วิธี | ใครกำหนด Address | ต้องมี DHCPv6 Server ไหม | Router ต้อง config อะไร | เหมาะกับ |
|---|---|---|---|---|
| Static | Admin พิมพ์เอง | ไม่ต้อง | `ipv6 address X::Y/64` | Router, Switch SVI, Server (infra) |
| EUI-64 | Admin กำหนด prefix, IOS คำนวณ Interface ID | ไม่ต้อง | `ipv6 address X::/64 eui-64` | Router/Switch ที่อยากพิมพ์สั้น |
| SLAAC | Host คำนวณเอง (prefix จาก RA) | ไม่ต้อง | `ipv6 unicast-routing` + `ipv6 address` | End-user host, IoT ที่ไม่มี DHCPv6 |
| DHCPv6 Stateful | DHCPv6 Server แจกเต็ม | **ต้องมี** | `ipv6 nd managed-config-flag` + DHCPv6 pool | Enterprise ที่ต้องการ tracking/control แบบ IPv4 DHCP |
| DHCPv6 Stateless | SLAAC (address) + DHCPv6 (options) | ต้องมี (สำหรับ options) | `ipv6 nd other-config-flag` + DHCPv6 pool (no address) | Network ที่อยากได้ DNS/NTP จาก DHCPv6 แต่ใช้ SLAAC สำหรับ address |

---

## Step 76 — การ Config IPv6 บน Cisco IOS

### คำสั่งพื้นฐาน 3 คำสั่งที่ต้องรู้

| คำสั่ง | Mode | ผล |
|---|---|---|
| `ipv6 unicast-routing` | Global Config | **เปิดการ route IPv6 ทั้งอุปกรณ์** (จำเป็นบน Router/L3 Switch ทุกตัวที่ต้อง forward packet ข้าม subnet และส่ง RA) — คล้าย `ip routing` ของ IPv4 (ซึ่งปกติเปิด default อยู่แล้วบน Router แต่ IPv6 **ต้องเปิดเองเสมอ**) |
| `ipv6 address <address>/<prefix-length>` | Interface Config | กำหนด IPv6 address (Global Unicast หรือ ULA) แบบ static ให้ interface |
| `ipv6 enable` | Interface Config | เปิดใช้ IPv6 บน interface **แบบไม่มี Global/ULA address** — สร้างแค่ Link-Local address อัตโนมัติ ใช้เมื่อต้องการรัน NDP/routing protocol โดยไม่ต้องมี routable address (เช่น point-to-point link ที่ใช้ link-local เป็น next-hop ล้วน) |

> **ข้อสังเกตสำคัญ**: การพิมพ์ `ipv6 address ...` (static หรือ eui-64) จะสร้าง **Link-Local
> address ให้อัตโนมัติด้วยเสมอ** จึงไม่จำเป็นต้องพิมพ์ `ipv6 enable` ซ้ำอีก — ใช้ `ipv6 enable`
> เฉพาะกรณีที่ตั้งใจจะไม่มี Global/ULA address บน interface นั้นเลย

### IPv6 Addressing Plan ของ Lab หลักสูตรนี้

ตั้งแต่ Part นี้เป็นต้นไป ทุก VLAN ใน Lab Topology จะมี **ทั้ง IPv4 (Part 7) และ IPv6 (ULA)
วิ่งคู่กันแบบ Dual-Stack** โดยใช้ **ULA prefix `fd00:10:10::/48`** แบ่ง subnet ตาม VLAN ID
(ตัวเลข subnet ที่ 3 ของ IPv6 ล้อกับตัวเลข octet ที่ 3 ของ IPv4 เพื่อให้จำง่าย):

| VLAN ID | ชื่อ | IPv4 Subnet (Part 7) | IPv6 ULA Subnet (Part 8) |
|---|---|---|---|
| 10 | SALES | 10.10.10.0/24 | `fd00:10:10:10::/64` |
| 20 | VOICE | 10.10.20.0/24 | `fd00:10:10:20::/64` |
| 30 | SERVERS | 10.10.30.0/24 | `fd00:10:10:30::/64` |
| 40 | WIFI | 10.10.40.0/24 | `fd00:10:10:40::/64` |
| 99 | MGMT | 10.10.99.0/24 | `fd00:10:10:99::/64` |

สำหรับ **WAN link ระหว่าง WAN-EDGE-1 ↔ ISP-RTR** ที่ Part 2 ใช้ IPv4 documentation range
`203.0.113.0/30` จำลอง Public IP เราจะใช้ **`2001:db8:203:113::/64`** (Global Unicast
documentation prefix ตาม RFC 3849) จำลอง Public IPv6 ในลักษณะเดียวกัน:

| Device | Interface | IPv6 Address |
|---|---|---|
| WAN-EDGE-1 | GigabitEthernet0/0/0 | `2001:db8:203:113::2/64` |
| ISP-RTR | (interface ฝั่ง WAN-EDGE-1) | `2001:db8:203:113::1/64` |

### Config เต็ม — CORE-SW1 (SVI ทุก VLAN, Dual-Stack)

```
CORE-SW1(config)# ipv6 unicast-routing
!
CORE-SW1(config)# interface Vlan10
CORE-SW1(config-if)# description ** SALES - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.10.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:10::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan20
CORE-SW1(config-if)# description ** VOICE - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.20.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:20::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan30
CORE-SW1(config-if)# description ** SERVERS - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.30.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:30::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan40
CORE-SW1(config-if)# description ** WIFI - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.40.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:40::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan99
CORE-SW1(config-if)# description ** MGMT - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.99.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:99::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
```

### Config เต็ม — WAN-EDGE-1 (link ไป ISP-RTR, Dual-Stack)

```
WAN-EDGE-1(config)# ipv6 unicast-routing
!
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description ** Link to ISP-RTR **
WAN-EDGE-1(config-if)# ip address 203.0.113.2 255.255.255.252
WAN-EDGE-1(config-if)# ipv6 address 2001:db8:203:113::2/64
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
```

### Config บน ISP-RTR (ฝั่งตรงข้าม)

```
ISP-RTR(config)# ipv6 unicast-routing
!
ISP-RTR(config)# interface GigabitEthernet0/0
ISP-RTR(config-if)# description ** Link to WAN-EDGE-1 **
ISP-RTR(config-if)# ip address 203.0.113.1 255.255.255.252
ISP-RTR(config-if)# ipv6 address 2001:db8:203:113::1/64
ISP-RTR(config-if)# no shutdown
ISP-RTR(config-if)# exit
```

> **ข้อควรระวังที่พบบ่อยที่สุด**: ลืมพิมพ์ `ipv6 unicast-routing` แล้วสงสัยว่าทำไม SLAAC/RA
> ไม่ทำงาน หรือ static route ที่ config ไปไม่ขึ้นใน routing table — คำสั่งนี้เป็นคำสั่งแรกที่ต้อง
> ตรวจสอบเสมอเวลา troubleshoot IPv6 ไม่ทำงานทั้งระบบ

---

## Step 77 — Neighbor Discovery Protocol (NDP)

**NDP (Neighbor Discovery Protocol, RFC 4861)** เป็นกลไกหลักของ IPv6 ที่**แทนที่ ARP, ICMP
Router Discovery, และ ICMP Redirect ของ IPv4 ทั้งหมดในโปรโตคอลเดียว** ทำงานอยู่บน **ICMPv6**
(ไม่ใช่ protocol แยกเหมือน ARP ของ IPv4)

### 5 ประเภทข้อความหลักของ NDP (ICMPv6 Type)

| ICMPv6 Type | ชื่อข้อความ | ย่อ | หน้าที่ | เทียบเท่า IPv4 |
|---|---|---|---|---|
| 133 | Router Solicitation | **RS** | Host ถาม router ว่า "มี router อยู่ไหม ช่วยส่ง prefix มาหน่อย" | (ไม่มีตรงๆ — คล้าย DHCP Discover) |
| 134 | Router Advertisement | **RA** | Router ประกาศ prefix, default gateway, M/O flag, MTU ให้ host บน link | (ไม่มีตรงๆ) |
| 135 | Neighbor Solicitation | **NS** | ถาม MAC Address ของ IPv6 address เป้าหมาย (ส่งไปที่ Solicited-Node Multicast) | **ARP Request** |
| 136 | Neighbor Advertisement | **NA** | ตอบกลับ NS ด้วย MAC Address ของตัวเอง | **ARP Reply** |
| 137 | Redirect | - | Router บอก host ว่ามี next-hop ที่ดีกว่าใน subnet เดียวกัน | ICMP Redirect |

### กระบวนการ Address Resolution (NS/NA) แทนที่ ARP

```
Host A (fd00:10:10:10::10)                    Host B (fd00:10:10:10::20)
         │                                               │
         │  1. NS: "ใครคือ fd00:10:10:10::20?"           │
         │     Dst = Solicited-Node Multicast            │
         │     ff02::1:ff00:20                            │
         │  ──────────────────────────────────────────►  │
         │                                               │
         │  2. NA: "ฉันเองคือ fd00:10:10:10::20"          │
         │     MAC = 00:50:56:AA:BB:CC                    │
         │  ◄──────────────────────────────────────────  │
         │                                               │
   Host A บันทึกลง IPv6 Neighbor Cache (เทียบเท่า ARP Table ของ IPv4)
```

```
CORE-SW1# show ipv6 neighbors
IPv6 Address                      Age Link-layer Addr State Interface
fd00:10:10:10::10                   0 0050.56aa.bbcc   REACH Vlan10
fd00:10:10:10::20                   3 0050.56dd.eeff   STALE Vlan10
FE80::21A:2FFF:FE3B:4C5D            5 001a.2f3b.4c5d   STALE Vlan10
```

State ของ Neighbor Cache (คล้าย ARP แต่ละเอียดกว่า): **INCMP** (Incomplete — กำลังส่ง NS รออยู่),
**REACH** (Reachable — ยืนยันแล้วภายในเวลาที่กำหนด), **STALE** (เคยยืนยันแต่หมดเวลาแล้ว รอใช้งาน
ถึงจะ verify ใหม่), **DELAY**, **PROBE**

### Router Discovery (RS/RA) — กลไกเบื้องหลัง SLAAC

```
Host                                          Router (CORE-SW1)
  │  1. RS → ff02::2 (All-Routers)             │
  │  "มี router ไหม ขอ prefix หน่อย"            │
  │  ─────────────────────────────────────►    │
  │                                             │
  │  2. RA → ff02::1 (All-Nodes) หรือ unicast   │
  │  Prefix: fd00:10:10:10::/64                │
  │  Flags: M=0, O=0 (ใช้ SLAAC ล้วน)           │
  │  Default Gateway: link-local ของ router     │
  │  ◄─────────────────────────────────────    │
  │                                             │
  Host คำนวณ Interface ID เอง (EUI-64/Random)
  → ได้ Global/ULA address ใช้งานได้ทันที
```

Router ที่เปิด `ipv6 unicast-routing` และมี `ipv6 address` บน interface จะส่ง RA **แบบ
unsolicited ทุก 200 วินาที (default)** โดยอัตโนมัติ และตอบทันทีเมื่อได้รับ RS จาก host

### Duplicate Address Detection (DAD)

ก่อนอุปกรณ์จะเริ่มใช้ IPv6 address ใดๆ (ไม่ว่าจะได้จาก static, EUI-64, หรือ SLAAC) ต้องผ่าน
**DAD (Duplicate Address Detection)** ก่อนเสมอ เพื่อยืนยันว่าไม่มีใครใน link เดียวกันใช้
address นี้ซ้ำ:

```
1. อุปกรณ์กำหนด address ชั่วคราวเป็นสถานะ "Tentative" (ยังใช้งานจริงไม่ได้)
2. ส่ง Neighbor Solicitation (NS) โดยใช้ source address = :: (Unspecified Address)
   ไปยัง Solicited-Node Multicast ของ address ตัวเอง
3. รอฟัง Neighbor Advertisement (NA) ตอบกลับ
   - ถ้า "ไม่มี" ใครตอบ NA ภายในเวลาที่กำหนด → address ใช้งานได้ (เปลี่ยนสถานะเป็น "Preferred")
   - ถ้า "มี" NA ตอบกลับ → แปลว่า address ซ้ำ (Duplicate) → interface หยุดใช้ address นั้น
     และแจ้ง error "DAD: duplicate address detected"
```

```
%IPV6-4-DUPLICATE: Duplicate address FD00:10:10:10::1 on Vlan10
```

> **ข้อสังเกต**: DAD คือเหตุผลที่ต้องใช้ **Unspecified Address (`::`)** เป็น source แทนที่จะใช้
> address ที่กำลังตรวจสอบเอง เพราะ address นั้นยังไม่ได้รับการยืนยันว่าใช้งานได้จริง

---

## Step 78 — IPv6 Static Routing เบื้องต้น

### Syntax คำสั่ง `ipv6 route`

```
Router(config)# ipv6 route <destination-prefix>/<prefix-length> {<next-hop-address> | <exit-interface> | <exit-interface> <next-hop-address>} [administrative-distance]
```

| Parameter | ความหมาย |
|---|---|
| `destination-prefix/prefix-length` | Network ปลายทาง เช่น `fd00:10:10:30::/64` |
| `next-hop-address` | IPv6 address ของ next-hop (ใช้ **Link-Local address** ก็ได้ ถ้าระบุ exit-interface ควบคู่ไปด้วย) |
| `exit-interface` | Interface ขาออก — **จำเป็นเมื่อใช้ Link-Local address เป็น next-hop** เพราะ Link-Local ไม่ unique ข้าม interface |
| `administrative-distance` | ค่า AD (default = 1 เหมือน static route ของ IPv4) |

### ตัวอย่าง Static Route

```
! ระบุ next-hop ด้วย Global Unicast/ULA address (ไม่ต้องระบุ exit-interface)
CORE-SW2(config)# ipv6 route fd00:10:10:99::/64 fd00:10:10:1::1

! ระบุ next-hop ด้วย Link-Local address (ต้องระบุ exit-interface เสมอ)
CORE-SW2(config)# ipv6 route fd00:10:10:99::/64 GigabitEthernet1/0/1 FE80::1A2B:3CFF:FE4D:5E6F

! ระบุแค่ exit-interface (ใช้ได้เฉพาะ point-to-point link เท่านั้น)
WAN-EDGE-1(config)# ipv6 route 2001:DB8:30:30::/64 GigabitEthernet0/0/0
```

### Default Route (`::/0`)

เทียบเท่า `ip route 0.0.0.0 0.0.0.0` ของ IPv4 — ใช้ prefix **`::/0`** หมายถึง "ทุก destination
ที่ไม่ match route อื่นเลยในตาราง"

```
! WAN-EDGE-1 ส่ง traffic ที่ไม่รู้จักออกไปหา ISP-RTR ทั้งหมด
WAN-EDGE-1(config)# ipv6 route ::/0 2001:db8:203:113::1
```

### `show ipv6 route`

```
CORE-SW2# show ipv6 route
IPv6 Routing Table - default - 12 entries
Codes: C - Connected, L - Local, S - Static, R - RIP, B - BGP
       U - Per-user Static route
       O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1, OE2 - OSPF ext 2
       ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2

C   fd00:10:10:10::/64 [0/0]
     via Vlan10, directly connected
L   fd00:10:10:10::2/128 [0/0]
     via Vlan10, receive
S   fd00:10:10:99::/64 [1/0]
     via fd00:10:10:1::1
L   FF00::/8 [0/0]
     via Null0, receive
```

- **C** = Connected (subnet ที่ต่อตรงกับ interface ของอุปกรณ์)
- **L** = Local (host route `/128` ของตัว address เอง — IPv6 สร้าง Local route แยกจาก
  Connected route เสมอ ต่างจาก IPv4 ที่รวมกันเป็นบรรทัดเดียว)
- **S** = Static (route ที่ admin พิมพ์เอง — ตัวย่อ AD/Metric ในวงเล็บเหมือน IPv4 `[AD/Metric]`)

> ตัวย่อ Routing Protocol Code (O=OSPF, B=BGP) จะเริ่มมีบรรทัดจริงตั้งแต่ Part 11 (OSPFv2)
> และ Part 27 (OSPFv3 for IPv6) เป็นต้นไป

---

## Step 79 — คำสั่ง Verify และ Troubleshoot IPv6

| คำสั่ง | ใช้ตรวจสอบอะไร |
|---|---|
| `show ipv6 interface brief` | สถานะ Up/Down และ address สรุปของทุก interface (คู่กับ `show ip interface brief` ของ IPv4) |
| `show ipv6 interface [interface]` | รายละเอียดเต็ม: address ทั้งหมด, MTU, joined multicast group, ND parameter |
| `show ipv6 neighbors` | IPv6 Neighbor Cache (เทียบเท่า `show ip arp`) |
| `show ipv6 route` | Routing table ของ IPv6 |
| `ping <ipv6-address>` | ทดสอบ connectivity (IOS ตรวจจาก format address อัตโนมัติ ไม่ต้องพิมพ์ `ping ipv6` ก็ได้ถ้าพิมพ์ address ตรงๆ) |
| `ping ipv6 <host/address>` | บังคับ ping แบบ IPv6 อย่างชัดเจน (มีประโยชน์เวลาพิมพ์ hostname ที่ resolve ได้ทั้ง A/AAAA record) |
| `traceroute ipv6 <address>` | ดู path แบบ hop-by-hop ของ IPv6 (เทียบเท่า `traceroute` ของ IPv4) |
| `show ipv6 protocols` | สรุป Routing Protocol IPv6 ที่กำลังทำงาน |
| `debug ipv6 nd` | Debug กระบวนการ Neighbor Discovery แบบ real-time |

### ตัวอย่าง `show ipv6 interface brief`

```
CORE-SW1# show ipv6 interface brief
Vlan10                 [up/up]
    FE80::21A:2FFF:FE3B:4C5D
    FD00:10:10:10::1
Vlan20                 [up/up]
    FE80::21A:2FFF:FE3B:4C5D
    FD00:10:10:20::1
Vlan30                 [up/up]
    FE80::21A:2FFF:FE3B:4C5D
    FD00:10:10:30::1
Vlan40                 [up/up]
    FE80::21A:2FFF:FE3B:4C5D
    FD00:10:10:40::1
Vlan99                 [up/up]
    FE80::21A:2FFF:FE3B:4C5D
    FD00:10:10:99::1
GigabitEthernet0/1     [up/up]
    FE80::21A:2FFF:FE3C:0102
    unassigned
```

> สังเกตว่าทุก interface ที่เปิด IPv6 จะมี **2 บรรทัดเสมอ**: Link-Local (ขึ้นต้น `FE80::`)
> กับ Global/ULA (ถ้ามี config) — ถ้าเห็นแค่ `unassigned` แปลว่า interface นั้นเปิด IPv6 ไว้
> (มี Link-Local) แต่ยังไม่ได้กำหนด Global/ULA address

### ตัวอย่าง `show ipv6 interface` (แบบละเอียด)

```
CORE-SW1# show ipv6 interface Vlan10
Vlan10 is up, line protocol is up
  IPv6 is enabled, link-local address is FE80::21A:2FFF:FE3B:4C5D
  No Virtual link-local address(es):
  Global unicast address(es):
    FD00:10:10:10::1, subnet is FD00:10:10:10::/64
  Joined group address(es):
    FF02::1
    FF02::2
    FF02::1:FF00:1
    FF02::1:FF3B:4C5D
  MTU is 1500 bytes
  ICMP error messages limited to one every 100 milliseconds
  ICMP redirects are enabled
  ICMP unreachables are sent
  ND DAD is enabled, number of DAD attempts: 1
  ND reachable time is 30000 milliseconds (using 30000)
  ND advertised reachable time is 0 (unspecified)
  ND advertised retransmit interval is 0 (unspecified)
  ND router advertisements are sent every 200 seconds
  ND router advertisements live for 1800 seconds
  Hosts use stateless autoconfig for addresses.
```

### ตัวอย่าง Ping และ Traceroute

```
CORE-SW1# ping fd00:10:10:30::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FD00:10:10:30::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms

CORE-SW1# ping ipv6 2001:db8:203:113::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:203:113::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms

WAN-EDGE-1# traceroute ipv6 2001:db8:30:30::10
Type escape sequence to abort.
Tracing the route to 2001:DB8:30:30::10

  1 2001:DB8:203:113::1 4 msec 2 msec 2 msec
  2 FD00:10:10:1::1 6 msec 5 msec 4 msec
  3 2001:DB8:30:30::10 8 msec 7 msec 6 msec
```

### Checklist Troubleshoot IPv6 ที่ไม่ทำงาน

1. `show run | include ipv6 unicast-routing` — เปิดหรือยัง? (ปัญหาที่พบบ่อยที่สุด)
2. `show ipv6 interface brief` — interface up/up ไหม มี address หรือ unassigned?
3. `show ipv6 neighbors` — resolve MAC ของปลายทางได้ไหม (เทียบเท่าเช็ค ARP)
4. `show ipv6 route` — มี route ไป destination นั้นไหม (Connected/Static/Dynamic)
5. `ping` ไปที่ Link-Local ของ next-hop ก่อน แล้วค่อยไล่ไป Global/ULA ปลายทางจริง (แยกปัญหา
   Layer 2 ออกจาก Layer 3/Routing)

---

## Step 80 — Lab: Dual-Stack IPv6 บน Core Switch และ WAN Edge

Lab นี้จะ config **IPv6 แบบ Dual-Stack (คู่กับ IPv4 เดิม)** ให้ครบทุกจุดตาม Addressing Plan
ของ Step 76: **CORE-SW1**, **CORE-SW2** (SVI ทั้ง 5 VLAN) และ **WAN-EDGE-1 ↔ ISP-RTR**
พร้อม static route เชื่อมทุกอย่างเข้าด้วยกัน แล้ว verify ให้ครบ

### Topology ของ Lab นี้

```
                              ┌───────────────┐
                              │   ISP-RTR      │
                              │ 2001:db8:203:113::1/64
                              └───────┬───────┘
                                      │  (2001:db8:203:113::/64)
                              ┌───────┴───────┐
                              │  WAN-EDGE-1    │
                              │ 2001:db8:203:113::2/64
                              └───────┬───────┘
                                      │
        ┌─────────────────────────────┴─────────────────────────────┐
        │           CORE-SW1 <===LACP Po1===> CORE-SW2                │
        │  Vlan10: fd00:10:10:10::1/64   Vlan10: fd00:10:10:10::2/64   │
        │  Vlan20: fd00:10:10:20::1/64   Vlan20: fd00:10:10:20::2/64   │
        │  Vlan30: fd00:10:10:30::1/64   Vlan30: fd00:10:10:30::2/64   │
        │  Vlan40: fd00:10:10:40::1/64   Vlan40: fd00:10:10:40::2/64   │
        │  Vlan99: fd00:10:10:99::1/64   Vlan99: fd00:10:10:99::2/64   │
        └───────────────────────────────────────────────────────────┘
```

### Config เต็ม — CORE-SW1

```
CORE-SW1(config)# ipv6 unicast-routing
!
CORE-SW1(config)# interface Vlan10
CORE-SW1(config-if)# description ** SALES - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.10.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:10::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan20
CORE-SW1(config-if)# description ** VOICE - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.20.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:20::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan30
CORE-SW1(config-if)# description ** SERVERS - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.30.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:30::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan40
CORE-SW1(config-if)# description ** WIFI - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.40.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:40::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
CORE-SW1(config)# interface Vlan99
CORE-SW1(config-if)# description ** MGMT - Dual-Stack SVI **
CORE-SW1(config-if)# ip address 10.10.99.1 255.255.255.0
CORE-SW1(config-if)# ipv6 address fd00:10:10:99::1/64
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit
!
! Default route ฝั่ง IPv6 ชี้ไปทาง WAN ผ่าน WAN-EDGE-1 (สมมติ CORE-SW1 มี P2P link ไป WAN-EDGE-1 ที่ fd00:10:10:1::/64)
CORE-SW1(config)# ipv6 route ::/0 fd00:10:10:1::2
CORE-SW1(config)# copy running-config startup-config
```

### Config เต็ม — CORE-SW2 (คู่ HA กับ CORE-SW1 — ใช้ ::2 ทุก VLAN)

```
CORE-SW2(config)# ipv6 unicast-routing
!
CORE-SW2(config)# interface Vlan10
CORE-SW2(config-if)# description ** SALES - Dual-Stack SVI **
CORE-SW2(config-if)# ip address 10.10.10.2 255.255.255.0
CORE-SW2(config-if)# ipv6 address fd00:10:10:10::2/64
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit
!
CORE-SW2(config)# interface Vlan20
CORE-SW2(config-if)# description ** VOICE - Dual-Stack SVI **
CORE-SW2(config-if)# ip address 10.10.20.2 255.255.255.0
CORE-SW2(config-if)# ipv6 address fd00:10:10:20::2/64
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit
!
CORE-SW2(config)# interface Vlan30
CORE-SW2(config-if)# description ** SERVERS - Dual-Stack SVI **
CORE-SW2(config-if)# ip address 10.10.30.2 255.255.255.0
CORE-SW2(config-if)# ipv6 address fd00:10:10:30::2/64
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit
!
CORE-SW2(config)# interface Vlan40
CORE-SW2(config-if)# description ** WIFI - Dual-Stack SVI **
CORE-SW2(config-if)# ip address 10.10.40.2 255.255.255.0
CORE-SW2(config-if)# ipv6 address fd00:10:10:40::2/64
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit
!
CORE-SW2(config)# interface Vlan99
CORE-SW2(config-if)# description ** MGMT - Dual-Stack SVI **
CORE-SW2(config-if)# ip address 10.10.99.2 255.255.255.0
CORE-SW2(config-if)# ipv6 address fd00:10:10:99::2/64
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit
!
CORE-SW2(config)# ipv6 route ::/0 fd00:10:10:1::2
CORE-SW2(config)# copy running-config startup-config
```

> **หมายเหตุ**: HSRP for IPv6 (Virtual Link-Local Address, Anycast Gateway) ที่ทำให้ทั้งสอง
> Core Switch ใช้ Default Gateway เดียวกันแบบ redundant จะเรียนละเอียดร่วมกับ HSRP/VRRP/GLBP
> ของ IPv4 ใน **Part 18** — ตอนนี้ทั้งสองตัวมี address จริงแยกกัน (`::1` / `::2`) ไว้ก่อน

### Config เต็ม — WAN-EDGE-1 ↔ ISP-RTR

```
! WAN-EDGE-1
WAN-EDGE-1(config)# ipv6 unicast-routing
WAN-EDGE-1(config)# interface GigabitEthernet0/0/0
WAN-EDGE-1(config-if)# description ** Link to ISP-RTR **
WAN-EDGE-1(config-if)# ip address 203.0.113.2 255.255.255.252
WAN-EDGE-1(config-if)# ipv6 address 2001:db8:203:113::2/64
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit
!
! Default route ออก Internet ผ่าน ISP-RTR
WAN-EDGE-1(config)# ipv6 route ::/0 2001:db8:203:113::1
!
! Static route กลับเข้า Lab ภายใน (สรุปรวม ULA ทั้ง /48 เส้นเดียว ผ่าน CORE-SW1)
WAN-EDGE-1(config)# ipv6 route fd00:10:10::/48 fd00:10:10:1::1
WAN-EDGE-1(config)# copy running-config startup-config
```

```
! ISP-RTR
ISP-RTR(config)# ipv6 unicast-routing
ISP-RTR(config)# interface GigabitEthernet0/0
ISP-RTR(config-if)# description ** Link to WAN-EDGE-1 **
ISP-RTR(config-if)# ip address 203.0.113.1 255.255.255.252
ISP-RTR(config-if)# ipv6 address 2001:db8:203:113::1/64
ISP-RTR(config-if)# no shutdown
ISP-RTR(config-if)# exit
!
! ISP-RTR รู้ทาง (จำลอง) กลับเข้า Enterprise ULA ผ่าน WAN-EDGE-1
ISP-RTR(config)# ipv6 route fd00:10:10::/48 2001:db8:203:113::2
ISP-RTR(config)# copy running-config startup-config
```

### Verification: ตรวจสอบผลลัพธ์ทั้งหมด

**1) ตรวจ interface ทุกจุดขึ้น `up/up` และมี address ครบ**

```
CORE-SW1# show ipv6 interface brief | exclude unassigned
Vlan10                 [up/up]
    FD00:10:10:10::1
Vlan20                 [up/up]
    FD00:10:10:20::1
Vlan30                 [up/up]
    FD00:10:10:30::1
Vlan40                 [up/up]
    FD00:10:10:40::1
Vlan99                 [up/up]
    FD00:10:10:99::1
```

**2) ตรวจ Routing Table ของ CORE-SW1 เห็น connected ครบ 5 VLAN + default route**

```
CORE-SW1# show ipv6 route
IPv6 Routing Table - default - 15 entries
Codes: C - Connected, L - Local, S - Static

C   FD00:10:10:10::/64 [0/0]
     via Vlan10, directly connected
C   FD00:10:10:20::/64 [0/0]
     via Vlan20, directly connected
C   FD00:10:10:30::/64 [0/0]
     via Vlan30, directly connected
C   FD00:10:10:40::/64 [0/0]
     via Vlan40, directly connected
C   FD00:10:10:99::/64 [0/0]
     via Vlan99, directly connected
S   ::/0 [1/0]
     via FD00:10:10:1::2
```

**3) ทดสอบ ping ข้าม VLAN ภายใน (SALES → SERVERS) และ ping ออก WAN**

```
CORE-SW1# ping fd00:10:10:30::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FD00:10:10:30::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/4 ms

WAN-EDGE-1# ping 2001:db8:203:113::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:203:113::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms
```

**4) ตรวจ Neighbor Cache หลัง ping สำเร็จ (ยืนยันว่า NDP ทำงาน)**

```
CORE-SW1# show ipv6 neighbors
IPv6 Address                      Age Link-layer Addr State Interface
FD00:10:10:30::1                    0 001a.2f4a.0102   REACH Vlan30
FD00:10:10:1::2                     1 001a.2f5b.0304   REACH GigabitEthernet1/0/1
```

Lab นี้ยืนยันว่า **Dual-Stack ทำงานได้สมบูรณ์**: ทุก SVI ตอบทั้ง `ping <ipv4>` และ
`ping <ipv6>`, Static Route ของ IPv6 ทำงานถูกต้องทั้งขาเข้า-ขาออก และ NDP resolve
MAC Address ได้ปกติ — โครงสร้างนี้จะถูกใช้ต่อเนื่องเป็นฐานของ **OSPFv3** ใน Part 27
และ **EIGRP for IPv6** ใน Part 28

---

## แบบฝึกหัดทวนความเข้าใจ Part 8

1. เพราะเหตุใด IPv6 จึงไม่มี Broadcast Address และใช้อะไรทดแทนการทำงานที่ Broadcast เคยทำใน IPv4?
2. จงย่อ (compress) IPv6 address เต็มรูปแบบนี้ให้สั้นที่สุดตามกฎ: `2001:0DB8:0000:0000:0000:0000:0000:0001`
3. Prefix `fc00::/7` (โดยเฉพาะ `fd00::/8`) ใช้แทนที่ concept ใดของ IPv4 และต่างจาก
   Global Unicast Address (`2000::/3`) อย่างไร?
4. จงแปลง MAC Address `00-50-56-AA-BB-CC` เป็น EUI-64 Interface ID (แสดงทุกขั้นตอน)
5. คำสั่งใดที่ **ต้อง** เปิดก่อนเสมอบน Router/L3 Switch เพื่อให้ IPv6 routing และ SLAAC/RA
   ทำงานได้ และถ้าลืมเปิดจะเกิดอาการอะไร?

**เฉลย:**

1. เพราะ Broadcast ทำให้ทุกอุปกรณ์ใน segment ต้องประมวลผล packet แม้ไม่เกี่ยวข้อง (สิ้นเปลือง
   CPU) IPv6 จึงใช้ **Multicast แบบเจาะจงกลุ่ม** แทน เช่น `ff02::1` (All-Nodes) แทน Broadcast
   ทั่วไป และ **Solicited-Node Multicast** แทน ARP Request แบบ Broadcast
2. `2001:DB8::1` — ตัด leading zero ของแต่ละกลุ่มออกก่อน ได้ `2001:DB8:0:0:0:0:0:1`
   จากนั้นย่อกลุ่ม 0 ที่ติดกัน 5 กลุ่มด้วย `::` ได้ `2001:DB8::1`
3. `fd00::/8` เทียบเท่า **RFC 1918 Private Address** ของ IPv4 (`10.x.x.x` เป็นต้น) คือใช้ได้
   เฉพาะภายในองค์กร ไม่ routable บน Internet สาธารณะ ต่างจาก Global Unicast (`2000::/3`)
   ที่ ISP จ่ายให้และ routable บน Internet จริง
4. แบ่ง MAC เป็น `00-50-56` และ `AA-BB-CC` → แทรก `FFFE` ตรงกลาง ได้ `00-50-56-FF-FE-AA-BB-CC`
   → flip bit ที่ 7 ของ byte แรก (`00` = `00000000` → `02` = `00000010`) → ผลลัพธ์
   Interface ID = `0250:56FF:FEAA:BBCC` เขียนแบบย่อ = `250:56ff:feaa:bbcc`
5. คำสั่ง `ipv6 unicast-routing` (Global Config) — ถ้าลืมเปิด อุปกรณ์จะไม่ forward IPv6 packet
   ข้าม subnet, ไม่ส่ง Router Advertisement (ทำให้ SLAAC บน host ไม่ทำงาน), และ static route
   IPv6 ที่ config ไว้จะไม่ถูกติดตั้งลง routing table

---

## สรุป Part 8

Part นี้ปูพื้นฐาน IPv6 ครบทุกด้านที่จำเป็นสำหรับ CCNA: รูปแบบ address และกฎการย่อ, ประเภท
address ทั้ง 4 แบบ (GUA/Link-Local/ULA/Multicast), well-known multicast และ Solicited-Node
Multicast ที่แทนที่ ARP, วิธีคำนวณ EUI-64, วิธี assign address ทั้ง 4 แบบ (Static/EUI-64/
SLAAC/DHCPv6), การ config บน Cisco IOS, Neighbor Discovery Protocol, Static Routing, คำสั่ง
verify ครบชุด และปิดท้ายด้วย Lab จริงที่ทำให้ **CORE-SW1, CORE-SW2 และ WAN-EDGE-1 ↔ ISP-RTR
วิ่งแบบ Dual-Stack (IPv4 + IPv6) พร้อมกันสมบูรณ์** ✅ — Addressing Plan ของ ULA
(`fd00:10:10::/48`) ที่ประกาศใน Part นี้จะถูกใช้ซ้ำตลอดหลักสูตร ทุกครั้งที่เพิ่ม Routing
Protocol หรือ Service ใหม่ที่รองรับ IPv6

**ไปต่อ:** [Part 9 — Static Routing →](part-009-static-routing.md)
