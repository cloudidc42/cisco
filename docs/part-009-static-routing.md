# Part 9 — Static Routing
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 81–90 จาก 1000**

> ย้อนกลับ: [Part 8 — IPv6 Fundamentals](part-008-ipv6-fundamentals.md)

> ต่อจาก Part 7 (IPv4 Addressing/Subnetting) และ Part 8 (IPv6 Fundamentals) ที่ปูพื้นฐาน
> Logical Addressing ทั้ง IPv4/IPv6 ให้ครบแล้ว Part นี้เราจะเริ่มทำให้ **Router** ใน Lab
> Topology "คุยกันได้จริง" ข้าม Network ด้วย **Static Routing** — เทคนิคการกำหนดเส้นทางแบบ manual
> ที่เป็นรากฐานของทุก Routing Protocol (OSPF, EIGRP, BGP) ที่จะเรียนในภายหลัง ก่อนเข้าสู่ Part 10
> (Inter-VLAN Routing) เราจะเปิดใช้งาน Layer 3 บน **CORE-SW1/CORE-SW2** เป็นครั้งแรกด้วย
> SVI เบื้องต้น (รายละเอียดเต็มเรื่อง Router-on-a-Stick/SVI จะอยู่ใน Part 10) เพื่อให้ Lab
> End-to-End ใน Step 90 ใช้งานได้จริงตั้งแต่ PC1 ถึง Internet

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 81 | Routing Fundamentals: Routing Table, Longest Prefix Match |
| 82 | Basic Static Route Syntax และตัวอย่างจริงบน WAN-EDGE-1 |
| 83 | Next-Hop IP vs Exit-Interface: ปัญหา Recursive Lookup |
| 84 | Default Route (Gateway of Last Resort) |
| 85 | Administrative Distance (AD) ตารางเต็มและการใช้งาน |
| 86 | Floating Static Route (Backup Path) |
| 87 | Host Route (/32) vs Summary Static Route |
| 88 | IPv6 Static Routing (เทียบกับ Part 8) |
| 89 | Troubleshooting Static Route |
| 90 | Lab เต็มรูปแบบ: Static Routing ทั้ง Enterprise Lab Topology |

---

## Step 81 — Routing Fundamentals: Routing Table และ Longest Prefix Match

### Routing Table คืออะไร

**Routing Table** (หรือ RIB — Routing Information Base) คือฐานข้อมูลที่ Router ทุกตัวเก็บไว้ใน RAM
เพื่อใช้ตัดสินใจว่า packet ที่มี destination IP หนึ่งๆ ควรถูกส่งออกไปทาง interface ไหน และผ่าน
next-hop ตัวใด Router จะเติม routing table จาก 3 แหล่งหลัก:

1. **Directly Connected Networks** — Network ที่ interface ของ Router เชื่อมต่ออยู่โดยตรงและ `no shutdown`
   พร้อมมี IP Address (Code `C`) และ Local host route ของ interface นั้นเอง (Code `L`)
2. **Static Route** — ผู้ดูแลระบบกำหนดเส้นทางเอง (Code `S`) ← หัวข้อหลักของ Part นี้
3. **Dynamic Routing Protocol** — Router เรียนรู้เส้นทางจากกันเองอัตโนมัติ (OSPF=`O`, EIGRP=`D`,
   BGP=`B`, RIP=`R`) ← จะเรียนใน Part 11 เป็นต้นไป

### วิธีที่ Router ตัดสินใจ Forward Packet — Longest Prefix Match (LPM)

เมื่อ Router ได้รับ packet และต้องหาเส้นทางที่ "match" ที่สุดกับ destination IP address ใน routing
table Router จะใช้กฎ **Longest Prefix Match (LPM)** เสมอ ไม่ว่าจะมี Route หลายเส้นทางที่ match
ปลายทางเดียวกัน Router จะเลือก Route ที่มี **subnet mask ยาวที่สุด (prefix length มากที่สุด)** ก่อน
โดยไม่สนใจ Administrative Distance หรือ Metric เลย (AD/Metric ใช้ตัดสินเมื่อ prefix ยาวเท่ากันเท่านั้น
— ดูรายละเอียดใน Step 85)

**ตัวอย่าง**: ถ้า Router มีทั้ง 3 เส้นทางนี้อยู่ในตาราง และ packet ปลายทางคือ `10.10.10.55`

| Route ในตาราง | Match กับ 10.10.10.55 หรือไม่ | Prefix Length |
|---|---|---|
| `0.0.0.0/0` (default route) | Match (ครอบคลุมทุก IP) | /0 |
| `10.10.0.0/16` | Match | /16 |
| `10.10.10.0/24` | Match | /24 ← **ยาวที่สุด ชนะ!** |

Router จะเลือก `10.10.10.0/24` เสมอ แม้ AD ของ default route จะต่ำกว่าก็ตาม เพราะ LPM ตัดสินก่อน
AD เสมอ — **นี่คือกฎที่สำคัญที่สุดข้อหนึ่งของการทำ Routing และเป็นคำถามข้อสอบยอดฮิตทุกระดับ**

### โครงสร้าง Routing Table ที่ต้องอ่านให้เป็น

```
WAN-EDGE-1# show ip route

Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
       n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route, + - replicated route, % - next hop override

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 6 subnets, 2 masks
S       10.10.10.0/24 [1/0] via 10.10.254.2
S       10.10.20.0/24 [1/0] via 10.10.254.2
S       10.10.30.0/24 [1/0] via 10.10.254.2
S       10.10.40.0/24 [1/0] via 10.10.254.2
S       10.10.99.0/24 [1/0] via 10.10.254.2
C       10.10.254.0/30 is directly connected, GigabitEthernet0/0/1
L       10.10.254.1/32 is directly connected, GigabitEthernet0/0/1
      203.0.113.0/24 is variably subnetted, 2 subnets, 2 masks
C       203.0.113.0/30 is directly connected, GigabitEthernet0/0/0
L       203.0.113.2/32 is directly connected, GigabitEthernet0/0/0
```

**อ่านแต่ละ column ให้เป็น:**

| ส่วนของ Entry | ความหมาย |
|---|---|
| `S` (ตัวหน้าสุด) | Code — แหล่งที่มาของ route (ดู Codes legend) |
| `10.10.10.0/24` | Network ปลายทาง และ subnet mask (prefix length) |
| `[1/0]` | `[Administrative Distance/Metric]` — AD=1 (static), Metric=0 (static ไม่มี metric จริง) |
| `via 10.10.254.2` | Next-hop IP Address ที่จะส่ง packet ต่อไป |
| `GigabitEthernet0/0/1` | Outgoing interface (แสดงเมื่อเป็น connected หรือระบุ exit-interface) |
| `Gateway of last resort` | บอกว่า default route (ถ้ามี) ชี้ไปทางไหน — มี `*` กำกับที่ code เสมอ |

> **หมายเหตุเรื่อง CEF**: ตาราง `show ip route` ที่เห็นคือ RIB (Control Plane) แต่การ forward
> packet จริงบน Cisco IOS สมัยใหม่ทั้งหมดใช้ **CEF (Cisco Express Forwarding)** ซึ่งสร้าง FIB
> (Forwarding Information Base) และ Adjacency Table แยกไว้ใน Data Plane เพื่อความเร็วสูงสุด
> (ไม่ query routing table ทีละ packet) ตรวจสอบได้ด้วย `show ip cef` และ `show adjacency detail`
> — จะเรียนละเอียดเรื่อง CEF ใน Part 91 (Advanced Troubleshooting Methodology)

---

## Step 82 — Basic Static Route Syntax และตัวอย่างจริงบน WAN-EDGE-1

### Syntax คำสั่งพื้นฐาน

```
Router(config)# ip route <destination-network> <subnet-mask> {<next-hop-ip> | <exit-interface>} [AD] [tag <value>] [name <text>] [permanent]
```

| พารามิเตอร์ | คำอธิบาย |
|---|---|
| `<destination-network>` | Network ปลายทาง เขียนแบบ dotted decimal (ไม่ใช่ CIDR `/24`) |
| `<subnet-mask>` | Subnet mask ของปลายทาง เช่น `255.255.255.0` |
| `<next-hop-ip>` | IP ของ Router ถัดไปที่จะส่ง packet ต่อ (ต้อง reachable ผ่าน connected network) |
| `<exit-interface>` | ชื่อ interface ที่จะส่ง packet ออก แทนการระบุ next-hop IP (ดู Step 83) |
| `[AD]` | Administrative Distance ที่กำหนดเอง (default = 1) ใช้ทำ Floating Static (Step 86) |
| `[tag <value>]` | ใส่ค่า tag ไว้ใช้กับ route-map ตอน redistribute (เรียนใน Part 31) |
| `[name <text>]` | ใส่ชื่ออธิบาย route (ช่วย documentation ใน `show ip route`) |
| `[permanent]` | บังคับให้ route คงอยู่ใน table แม้ exit-interface จะ down (ปกติ route จะหายถ้า interface down) |

### ตัวอย่างจริง: WAN-EDGE-1 ต้องไปถึงทุก VLAN ของ Campus (ผ่าน CORE-SW1)

จาก IP Plan ที่วางไว้ WAN-EDGE-1 เชื่อมต่อกับ CORE-SW1 ผ่าน transit link `10.10.254.0/30`
(WAN-EDGE-1 = `.1`, CORE-SW1 = `.2`) ต้องเพิ่ม static route เพื่อให้ WAN-EDGE-1 รู้จัก VLAN ทั้ง 5
วงที่อยู่หลัง CORE-SW1:

```
WAN-EDGE-1(config)# ip route 10.10.10.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# ip route 10.10.20.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# ip route 10.10.30.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# ip route 10.10.40.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# ip route 10.10.99.0 255.255.255.0 10.10.254.2
```

**Verify:**

```
WAN-EDGE-1# show ip route static
      10.0.0.0/8 is variably subnetted, 7 subnets, 2 masks
S       10.10.10.0/24 [1/0] via 10.10.254.2
S       10.10.20.0/24 [1/0] via 10.10.254.2
S       10.10.30.0/24 [1/0] via 10.10.254.2
S       10.10.40.0/24 [1/0] via 10.10.254.2
S       10.10.99.0/24 [1/0] via 10.10.254.2
```

> **กฎสำคัญ**: next-hop IP ที่ระบุ (`10.10.254.2`) **ต้อง** อยู่ใน network ที่ Router มี interface
> เชื่อมต่อโดยตรง (directly connected) เท่านั้น ถ้าพิมพ์ผิดหรือ next-hop ไม่ reachable
> คำสั่งจะยอมรับ (ไม่ error ตอนพิมพ์) แต่ route จะ**ไม่ถูกติดตั้งใน routing table** — เป็นสาเหตุ
> อันดับ 1 ของปัญหา static route ไม่ทำงาน (ดู Step 89)

### ในทิศทางกลับ — CORE-SW1 ต้องรู้จัก network ฝั่ง WAN-EDGE-1 ด้วย

Routing ต้องทำ**สองทาง**เสมอ (bidirectional) มิฉะนั้น packet ไปถึงปลายทางแต่ reply กลับไม่ได้
(asymmetric routing / one-way traffic):

```
CORE-SW1(config)# ip route 203.0.113.0 255.255.255.252 10.10.254.1
```

ทำให้ CORE-SW1 รู้จักเครือข่าย WAN link ฝั่ง ISP ด้วย ซึ่งจำเป็นสำหรับให้ ping/traceroute
ตอบกลับได้ถูกต้อง

---

## Step 83 — Next-Hop IP vs Exit-Interface: ปัญหา Recursive Lookup

Static route สามารถระบุปลายทางถัดไปได้ 3 แบบ:

| แบบ | ตัวอย่าง | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Next-hop IP เท่านั้น** | `ip route 10.10.10.0 255.255.255.0 10.10.254.2` | อ่านง่าย เปลี่ยน physical interface ได้โดยไม่ต้องแก้ route | เกิด **Recursive Lookup** (ดูด้านล่าง) |
| **Exit-interface เท่านั้น** | `ip route 10.10.10.0 255.255.255.0 GigabitEthernet0/0/1` | ไม่มี recursive lookup, เร็วที่สุดบน P2P link | บน Multi-access (Ethernet) จะเข้าใจผิดว่าทุกปลายทางเป็น "directly connected" ทำให้ Router **ARP ทุก destination IP** — ผิดพฤติกรรมและกิน CPU/ARP table โดยไม่จำเป็น |
| **Fully Specified (interface + next-hop)** | `ip route 10.10.10.0 255.255.255.0 GigabitEthernet0/0/1 10.10.254.2` | ไม่มี recursive lookup **และ** ไม่มีปัญหา ARP ผิดพลาดบน Ethernet | ต้องพิมพ์ยาวขึ้นเล็กน้อย — **แนะนำให้ใช้แบบนี้เป็นค่าเริ่มต้นบน Ethernet ทุกกรณี** |

### ปัญหา Recursive Lookup คืออะไร

เมื่อ static route ระบุแค่ next-hop IP (ไม่ระบุ exit-interface) Router **ไม่รู้ทันทีว่าต้องส่ง packet
ออกทาง interface ไหน** ต้องทำการ **lookup ครั้งที่สอง** ในตัวเอง (recursive) เพื่อหาว่า next-hop
IP นั้นไปถึงได้ทาง interface ใด:

```
ip route 10.10.10.0 255.255.255.0 10.10.254.2
                                        │
                                        ▼
              Router ต้อง lookup อีกครั้งว่า 10.10.254.2 ไปทางไหน
                                        │
                                        ▼
         พบว่า 10.10.254.0/30 connected อยู่ที่ GigabitEthernet0/0/1
                                        │
                                        ▼
                 จึงรู้ว่าต้องส่ง frame ออกทาง Gi0/0/1 พร้อม ARP หา MAC ของ 10.10.254.2
```

การ lookup 2 ชั้นแบบนี้ใช้ CPU cycle เพิ่มขึ้นเล็กน้อยทุกครั้งที่ CEF ต้องสร้าง adjacency ใหม่
(ผลกระทบจริงในทางปฏิบัติต่ำมากบน Router สมัยใหม่ เพราะ CEF cache ผลลัพธ์ไว้ แต่ในระบบเก่าหรือกรณี
routing loop จะเห็นผลชัดเจน — ดู "Recursive Routing Loop" ใน Step 89) และที่สำคัญกว่านั้นคือ
**ถ้า route ที่ recursive lookup ใช้อ้างอิงหายไป (next-hop unreachable) static route
ที่ชี้ไปจะถูกถอดออกจากตารางโดยอัตโนมัติทันที** ซึ่งเป็นพฤติกรรมที่ต้องการ (fail-safe)

### เมื่อไหร่ควรใช้แบบไหน

- **Point-to-Point link (Serial, GRE Tunnel, PPP)**: ใช้ **exit-interface อย่างเดียวได้** เพราะฝั่งตรงข้าม
  มีแค่ปลายทางเดียวเสมอ ไม่มีปัญหา ARP ผิดพลาด
- **Multi-access link (Ethernet, VLAN, Trunk)**: **ห้ามใช้ exit-interface อย่างเดียว** — ใช้
  **next-hop IP** หรือ **Fully Specified** เท่านั้น (เลือก Fully Specified ถ้าต้องการประสิทธิภาพสูงสุด)

```
! ตัวอย่างผิด (บน Ethernet) - จะทำให้ Router ARP ทุก IP ปลายทางราวกับ directly connected
WAN-EDGE-1(config)# ip route 10.10.30.0 255.255.255.0 GigabitEthernet0/0/1     ! ไม่แนะนำ!

! ตัวอย่างถูกต้อง - Fully Specified
WAN-EDGE-1(config)# ip route 10.10.30.0 255.255.255.0 GigabitEthernet0/0/1 10.10.254.2
```

---

## Step 84 — Default Route (Gateway of Last Resort)

**Default Route** คือ static route พิเศษที่ match กับ**ทุก destination IP ที่ไม่มี route ที่เจาะจงกว่า
ในตาราง** เขียนด้วย network `0.0.0.0` mask `0.0.0.0` (หรือ `/0`) เป็นวิธีมาตรฐานที่ Router
ขอบ (Edge Router) ใช้ส่ง traffic ที่ไม่รู้จักปลายทางออกไปยัง ISP

### Syntax และตัวอย่างจริงบน WAN-EDGE-1

```
WAN-EDGE-1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

คำสั่งนี้บอกว่า "packet ปลายทางใดๆ ที่ไม่ match กับ route อื่นในตาราง ให้ส่งไปที่ ISP-RTR
(`203.0.113.1`) ทั้งหมด" — เหมาะสำหรับ Edge Router ที่ไม่จำเป็นต้องรู้จักทุก prefix บน Internet
(มี prefix นับแสนรายการ) จึงใช้ default route แทนการเรียนรู้ full BGP table

**Verify:**

```
WAN-EDGE-1# show ip route | include Gateway
Gateway of last resort is 203.0.113.1 to network 0.0.0.0

WAN-EDGE-1# show ip route static
S*   0.0.0.0/0 [1/0] via 203.0.113.1
```

สังเกตว่า code จะมี `*` ต่อท้าย (`S*`) เสมอเมื่อ route นั้นเป็น candidate default route ที่กำลัง
ถูกใช้งานจริง (Gateway of Last Resort)

### `ip route 0.0.0.0 0.0.0.0` vs `ip default-gateway`

จุดที่มือใหม่สับสนบ่อยมาก:

| คำสั่ง | ใช้กับอุปกรณ์ประเภทใด | เงื่อนไข |
|---|---|---|
| `ip route 0.0.0.0 0.0.0.0 <next-hop>` | Router หรือ Layer 3 Switch **ที่เปิด `ip routing`** | ใช้สำหรับ forward IP traffic ของอุปกรณ์อื่นออก Internet |
| `ip default-gateway <ip>` | Layer 2 Switch ธรรมดา **ที่ไม่ได้เปิด `ip routing`** (เช่น ACCESS-SW1-4) | ใช้แค่ให้**ตัว Switch เอง**ส่ง management traffic (SSH/ping/SNMP จากตัว Switch) ออกไปเท่านั้น ไม่เกี่ยวกับ traffic ของ end device |

CORE-SW1/CORE-SW2 (Multilayer Switch เปิด `ip routing`) ใช้ `ip route 0.0.0.0 0.0.0.0`
ส่วน ACCESS-SW1-4 (L2 switch) ใช้ `ip default-gateway` เท่านั้น

---

## Step 85 — Administrative Distance (AD) ตารางเต็มและการใช้งาน

**Administrative Distance (AD)** คือค่าความน่าเชื่อถือ (0–255) ที่ Cisco IOS กำหนดให้กับ**แหล่งที่มา
ของ route แต่ละประเภท** ใช้ตัดสินใจว่าจะเลือก route จากแหล่งไหน**เมื่อมีหลายแหล่งเรียนรู้ prefix
เดียวกันด้วย prefix length เท่ากันทุกประการ** (ถ้า prefix length ต่างกัน LPM จาก Step 81 ชนะ
เสมอ ไม่เกี่ยวกับ AD) — **ค่า AD ยิ่งต่ำ ยิ่งน่าเชื่อถือ ยิ่งถูกเลือกก่อน**

### ตาราง Administrative Distance เต็ม (ต้องจำให้ขึ้นใจ — ออกข้อสอบทุกระดับ)

| แหล่งที่มาของ Route | Administrative Distance |
|---|---|
| Connected Interface | **0** |
| Static Route | **1** |
| EIGRP Summary Route | **5** |
| eBGP (External BGP) | **20** |
| EIGRP (Internal) | **90** |
| IGRP (เลิกใช้แล้ว) | **100** |
| OSPF | **110** |
| IS-IS | **115** |
| RIP | **120** |
| EIGRP External | **170** |
| iBGP (Internal BGP) | **200** |
| Unknown / ไม่น่าเชื่อถือเลย | **255 (ไม่ถูกติดตั้งใน routing table)** |

### AD ทำงานอย่างไรเมื่อมีหลาย Routing Source แข่งกัน

สมมติ CORE-SW1 เรียนรู้ path ไปยัง `10.10.30.0/24` จาก **2 แหล่ง** พร้อมกัน:

1. **Static Route** ที่ผู้ดูแลระบบพิมพ์เอง → AD = 1
2. **OSPF** ที่เรียนรู้ผ่าน routing protocol (สมมติเปิดใช้แล้วใน Part 11) → AD = 110

แม้ OSPF จะคำนวณ path ที่ "ดีที่สุด" ตาม metric ของมันเอง แต่ **Static Route (AD=1) จะชนะเสมอ**
เพราะ AD ต่ำกว่า — IOS จะติดตั้งเฉพาะ static route ใน RIB และไม่สนใจ OSPF entry เลยสำหรับ
prefix นั้น (OSPF entry ยังคงอยู่ใน OSPF topology table/database แต่ไม่ถูกติดตั้งใน routing table)

> **ข้อควรระวังระดับ Production**: นี่คือเหตุผลที่ static route ที่ทิ้งไว้โดยไม่ได้ตั้งใจ (ลืมลบ)
> มักทำให้ Network Engineer งงว่า "ทำไม OSPF convergence แล้ว traffic ยังวิ่งเส้นทางเดิม"
> คำตอบคือ static route (AD=1) บังทับ path ไว้อยู่ ต้องลบออกก่อนให้ OSPF ทำงานแทน — เทคนิคการ
> ตรวจสอบนี้คือหัวใจของ **Floating Static Route** ใน Step 86

---

## Step 86 — Floating Static Route (Backup Path)

**Floating Static Route** คือ static route ที่ตั้งค่า **Administrative Distance ให้สูงกว่า** AD
ของ route หลัก (primary route) โดยเจตนา ทำให้ route นี้ **"ลอย" อยู่เฉยๆ ไม่ถูกติดตั้งใน routing
table** ตราบใดที่ primary route ยังทำงานอยู่ และจะ**ถูกติดตั้งอัตโนมัติทันทีที่ primary route หายไป**
(interface down หรือ routing protocol สูญเสีย neighbor) — ใช้ทำ **Backup Path** โดยไม่ต้องพึ่ง
Dynamic Routing Protocol ที่ซับซ้อน

### ตัวอย่างที่ 1: Floating Static เป็น WAN Backup Internet Path (WAN-EDGE-2 สำรอง WAN-EDGE-1)

Topology: WAN-EDGE-1 เป็น Internet Path หลัก (เชื่อม ISP-RTR ที่ `203.0.113.0/30`)
ส่วน WAN-EDGE-2 มีลิงก์สำรองไปยัง ISP-RTR อีกเส้นที่ `203.0.113.4/30` (WAN-EDGE-2=`.6`,
ISP-RTR=`.5`) และ WAN-EDGE-1 ↔ WAN-EDGE-2 มี cross-link ที่ `10.10.254.8/30`
(WAN-EDGE-1=`.9`, WAN-EDGE-2=`.10`) สำหรับ route traffic ข้ามไปมาเวลาเกิด failover

```
                          ┌───────────────┐
                          │   ISP-RTR      │
                          └───┬───────┬───┘
                  203.0.113.0/30   203.0.113.4/30
                (.1)  │               │  (.5)
                      │(.2)      (.6) │
              ┌───────┴──────┐ ┌──────┴───────┐
              │  WAN-EDGE-1   │ │  WAN-EDGE-2   │
              │  (PRIMARY)     │ │  (BACKUP)      │
              └───────┬──────┘ └──────┬───────┘
                      │  10.10.254.8/30  │
                (.9)  └──────────────────┘  (.10)
```

**บน WAN-EDGE-1 (primary — ใช้ AD ปกติ = 1):**

```
WAN-EDGE-1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

**บน WAN-EDGE-2 (route สำรอง — ตั้ง AD ให้สูงกว่า 1 แต่ยังต่ำกว่า OSPF ในอนาคต (110) เผื่อไว้):**

```
! Floating static ผ่านลิงก์ ISP สำรองของตัวเอง กำหนด AD=5 (แพงกว่า static ปกติ)
WAN-EDGE-2(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.5 5
```

**บน CORE-SW1/CORE-SW2 (มองไป WAN-EDGE-1 เป็น path หลัก, WAN-EDGE-2 เป็น floating backup):**

```
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 10.10.254.1
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 10.10.254.6 5
```

**ทดสอบ Failover จริง:**

```
! ก่อน shutdown - เห็นแค่ route หลัก (AD=1) เท่านั้นในตาราง
CORE-SW1# show ip route static
S*   0.0.0.0/0 [1/0] via 10.10.254.1

! จำลอง WAN-EDGE-1 ล่ม
CORE-SW1(config)# interface range GigabitEthernet1/0/1
CORE-SW1(config-if-range)# shutdown

! หลัง primary หาย - floating static (AD=5) ถูกติดตั้งอัตโนมัติทันที
CORE-SW1# show ip route static
S*   0.0.0.0/0 [5/0] via 10.10.254.6
```

### ตัวอย่างที่ 2: Floating Static รอ OSPF (เตรียมไว้ล่วงหน้าก่อนเรียน Part 11)

หลักการเดียวกัน — เมื่อเปิด OSPF ใน Part 11 (AD=110) เราสามารถทิ้ง static route ที่ AD สูงกว่า
110 ไว้เป็น "safety net" กรณี OSPF process ล่มทั้งหมด (bug, misconfiguration):

```
CORE-SW1(config)# ip route 0.0.0.0 0.0.0.0 10.10.254.1 200
```

ค่า AD=200 นี้จะไม่ถูกใช้เลยตราบใดที่ OSPF (AD=110) ยังทำงานปกติ แต่จะเข้ามาแทนที่ทันทีถ้า OSPF
หายไปทั้งหมด — เทคนิคนี้เป็นมาตรฐานความปลอดภัยขั้นต่ำในระบบ Production ระดับ Enterprise

---

## Step 87 — Host Route (/32) vs Summary Static Route

### Host Route (/32) — เจาะจงเครื่อง IP เดียว

**Host Route** คือ static route ที่ mask เป็น `255.255.255.255` (`/32`) ใช้เมื่อต้องการควบคุม
เส้นทางไปยัง**เครื่องปลายทางเดียวแบบเจาะจง** โดยไม่กระทบ network วงอื่นที่เหลือ เช่น ต้องการให้
traffic ไปยัง Syslog/TFTP Server ตัวหนึ่งวิ่งผ่านเส้นทางพิเศษ (เช่น ผ่าน VPN หรือ Management
network แยก) ในขณะที่ traffic ไปยัง IP อื่นในวงเดียวกันวิ่งเส้นทางปกติ:

```
! Server บริหารจัดการ (Syslog/TFTP) อยู่ที่ 10.10.99.50 ต้องการ route เจาะจงแยกจาก VLAN99 ปกติ
CORE-SW1(config)# ip route 10.10.99.50 255.255.255.255 10.10.254.1 name MGMT-SERVER-VIA-WAN1
```

เนื่องจาก `/32` เป็น prefix ที่ยาวที่สุดที่เป็นไปได้ (LPM) route นี้จะ**ชนะ**เสมอเหนือ route
`10.10.99.0/24` ปกติ สำหรับ traffic ที่ปลายทางตรงกับ `10.10.99.50` เท่านั้น ส่วน host อื่นในวง
`10.10.99.0/24` จะยังใช้ route ปกติ

### Summary Static Route — รวมหลาย Subnet เป็นเส้นทางเดียว

**Summary Route (Route Aggregation)** คือการรวมหลาย subnet ที่ต่อเนื่องกันทาง binary
ให้เหลือเป็น static route เพียงเส้นเดียว ช่วยลดจำนวนบรรทัด config และลดขนาด routing table
ของ Router ต้นทาง

จากตัวอย่าง Step 82 ที่ WAN-EDGE-1 ต้องมี static route แยก 5 บรรทัดไปยังทุก VLAN ของ Campus
(`10.10.10.0/24`, `10.10.20.0/24`, `10.10.30.0/24`, `10.10.40.0/24`, `10.10.99.0/24`)
เนื่องจากทุก subnet นี้อยู่ภายใต้ `10.10.0.0/16` เหมือนกันหมด (และไม่มี subnet อื่นของ
`10.10.x.x` ที่ WAN-EDGE-1 ต้องแยกแยะเป็นพิเศษ) เราสามารถรวมเป็น **1 บรรทัดเดียว** ได้ทันที:

```
! แทนที่ 5 บรรทัดด้วย summary route เพียง 1 บรรทัด
WAN-EDGE-1(config)# no ip route 10.10.10.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# no ip route 10.10.20.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# no ip route 10.10.30.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# no ip route 10.10.40.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# no ip route 10.10.99.0 255.255.255.0 10.10.254.2
WAN-EDGE-1(config)# ip route 10.10.0.0 255.255.0.0 10.10.254.2
```

**Verify:**

```
WAN-EDGE-1# show ip route static
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
S       10.10.0.0/16 [1/0] via 10.10.254.2
```

> **ข้อควรระวัง**: Summary route ต้องครอบคลุม**เฉพาะ** subnet ที่ next-hop เดียวกันสามารถไปถึงได้
> จริงเท่านั้น ถ้า `10.10.254.0/30` (transit link) หรือ subnet อื่นที่ไม่ได้อยู่หลัง CORE-SW1
> ตัวเดียวกันดันอยู่ในช่วง `10.10.0.0/16` ด้วย จะเกิด **Black Hole Route** (Router คิดว่าส่งไปถึง
> แต่จริงๆ ไม่มีเส้นทางไปจริง) ต้องตรวจสอบ IP Plan ให้ละเอียดก่อน summarize เสมอ

---

## Step 88 — IPv6 Static Routing (เทียบกับ Part 8)

การทำ Static Route บน IPv6 ใช้แนวคิดเดียวกันทุกประการกับ IPv4 ต่างกันแค่ syntax คำสั่งและรูปแบบ
Address — ต่อยอดจาก IPv6 Addressing Plan ที่วางไว้ใน **Part 8** (Global Unicast Prefix
`2001:db8:cafe::/48` แบ่งตาม VLAN ID เป็น subnet `/64`: VLAN10=`2001:db8:cafe:10::/64`,
VLAN20=`2001:db8:cafe:20::/64`, VLAN30=`2001:db8:cafe:30::/64`, VLAN40=`2001:db8:cafe:40::/64`,
VLAN99=`2001:db8:cafe:99::/64`, และ WAN Transit=`2001:db8:cafe:fe::/64`)

### ตารางเทียบ Syntax IPv4 vs IPv6 Static Route

| งาน | IPv4 | IPv6 |
|---|---|---|
| คำสั่งพื้นฐาน | `ip route <net> <mask> <next-hop>` | `ipv6 route <prefix>/<len> <next-hop>` |
| Default Route | `ip route 0.0.0.0 0.0.0.0 <next-hop>` | `ipv6 route ::/0 <next-hop>` |
| Exit-interface | `ip route <net> <mask> <interface>` | `ipv6 route <prefix>/<len> <interface>` |
| Fully Specified | `ip route <net> <mask> <interface> <next-hop>` | `ipv6 route <prefix>/<len> <interface> <next-hop>` |
| Floating Static (custom AD) | `ip route <net> <mask> <next-hop> <AD>` | `ipv6 route <prefix>/<len> <next-hop> <AD>` |
| Host Route | `ip route <ip> 255.255.255.255 <next-hop>` | `ipv6 route <ip>/128 <next-hop>` |
| ดู routing table | `show ip route` | `show ipv6 route` |
| ดูเฉพาะ static | `show ip route static` | `show ipv6 route static` |
| เปิดใช้งาน routing | `ip routing` (เปิด default อยู่แล้วบน Router) | `ipv6 unicast-routing` (**ต้องเปิดเองเสมอ**) |

### ตัวอย่างจริงบน WAN-EDGE-1 (คู่ขนานกับ Step 82/84 ฝั่ง IPv4)

```
WAN-EDGE-1(config)# ipv6 unicast-routing
!
! Summary route ไปยังทุก VLAN ของ Campus (เทียบเท่า Step 87 ฝั่ง IPv4)
WAN-EDGE-1(config)# ipv6 route 2001:DB8:CAFE::/48 2001:DB8:CAFE:FE::2
!
! Default route ออก Internet ผ่าน ISP-RTR (สมมติ ISP แจก IPv6 transit ผ่าน link-local)
WAN-EDGE-1(config)# ipv6 route ::/0 GigabitEthernet0/0/0 FE80::1
```

> **ข้อควรระวังเฉพาะ IPv6**: ถ้า next-hop เป็น **Link-Local Address** (`FE80::/10`) จำเป็นต้อง
> ระบุ **exit-interface ควบคู่เสมอ** (`ipv6 route ::/0 GigabitEthernet0/0/0 FE80::1`) เพราะ
> Link-Local ไม่ใช่ address ที่ routable ข้าม link ได้ — Router ไม่สามารถทำ recursive lookup
> เพื่อหา interface ที่ถูกต้องได้เหมือน Global Unicast Address ปกติ นี่คือความต่างสำคัญจาก
> IPv4 ที่ Step 83 อธิบายไว้

**Verify:**

```
WAN-EDGE-1# show ipv6 route static
S   ::/0 [1/0]
     via FE80::1, GigabitEthernet0/0/0
S   2001:DB8:CAFE::/48 [1/0]
     via 2001:DB8:CAFE:FE::2
```

---

## Step 89 — Troubleshooting Static Route

### คำสั่ง Verify หลักที่ต้องใช้เป็นประจำ

```
! ดูเฉพาะ static route ทั้งหมดในตาราง
WAN-EDGE-1# show ip route static

! ตรวจสอบ route เจาะจงหนึ่งเส้นทาง (บอกด้วยว่า match กับ entry ไหน และทำไม)
WAN-EDGE-1# show ip route 10.10.10.0
Routing entry for 10.10.10.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.10.254.2
      Route metric is 0, traffic share count is 1

! ดูว่า prefix นี้ "ไม่มี" ใน table หรือไม่ (ผลลัพธ์ % Network not in table = ยังไม่ได้ config หรือ next-hop unreachable)
WAN-EDGE-1# show ip route 172.16.5.0
% Network not in table

! ดู CEF FIB ที่ใช้ forward จริง (ต้องตรงกับ RIB)
WAN-EDGE-1# show ip cef 10.10.10.0
10.10.10.0/24
  nexthop 10.10.254.2 GigabitEthernet0/0/1

! ตรวจสอบว่า route มาจาก running-config จริง (ตรง syntax ที่พิมพ์)
WAN-EDGE-1# show running-config | include ip route
```

### ข้อผิดพลาดที่พบบ่อยที่สุด 4 แบบ

**1) Subnet Mask ผิด** — พิมพ์ mask ผิดทำให้ route ครอบคลุม (หรือไม่ครอบคลุม) ปลายทางที่ต้องการ

```
! ผิด - mask /16 ครอบคลุมกว้างเกินไป อาจไป conflict กับ route อื่นที่ควรเจาะจงกว่า
WAN-EDGE-1(config)# ip route 10.10.10.0 255.255.0.0 10.10.254.2   ! ผิด! ตั้งใจจะทำ /24

! ถูก
WAN-EDGE-1(config)# ip route 10.10.10.0 255.255.255.0 10.10.254.2
```

**2) Next-hop ไม่ใช่ Directly Connected** — พิมพ์ next-hop ที่ไม่มีทาง reachable โดยตรง
route จะถูกยอมรับตอน config แต่จะไม่ถูกติดตั้งใน RIB (เงียบ ไม่มี error message เตือน!)

```
! ผิด - 10.10.254.99 ไม่ได้อยู่ใน subnet ที่ WAN-EDGE-1 connected อยู่ (10.10.254.0/30 มีแค่ .1-.2)
WAN-EDGE-1(config)# ip route 10.10.10.0 255.255.255.0 10.10.254.99

WAN-EDGE-1# show ip route 10.10.10.0
% Network not in table          ← เห็นได้ชัดว่า route ไม่ถูกติดตั้ง แม้ config อยู่ก็ตาม
```

**3) ลืม Default Route หรือ Route กลับทาง (Return Path)** — ping ไปถึงปลายทางได้ (one-way)
แต่ reply กลับไม่ได้ เพราะ Router ปลายทางไม่รู้จักทางกลับ อาการ: `traceroute` ค้างที่ hop กลาง
หรือ ping timeout ทั้งที่ capture เห็น packet ไปถึงปลายทางจริง — เป็นปัญหา **Asymmetric Routing**
ที่พบบ่อยที่สุดอันดับ 1 ของ Static Routing ทั้งหมด

**4) Recursive Routing Loop** — Router สองตัวต่างชี้ default route หากันเอง (มักเกิดตอน config
ผิดพลาดชั่วคราวระหว่างเปลี่ยน topology):

```
RouterA(config)# ip route 0.0.0.0 0.0.0.0 <IP ของ RouterB>
RouterB(config)# ip route 0.0.0.0 0.0.0.0 <IP ของ RouterA>
```

อาการ: packet ที่ไม่มีปลายทางจริงจะถูก "โยนกันไปมา" ระหว่าง Router ทั้งสองจนกว่า TTL จะหมด
(`traceroute` จะแสดง hop สลับกันไปมาระหว่าง 2 IP ซ้ำๆ) วิธีตรวจสอบ:

```
WAN-EDGE-1# traceroute 172.16.99.99
Type escape sequence to abort.
Tracing the route to 172.16.99.99
  1 10.10.254.2 4 msec
  2 10.10.254.1 4 msec
  3 10.10.254.2 4 msec
  4 10.10.254.1 4 msec        ← เห็น pattern วนซ้ำ = recursive loop ชัดเจน
  5 10.10.254.2 4 msec
    ...
```

แก้ไขโดยเพิ่ม **null route (discard route)** ที่ Edge Router เพื่อป้องกัน traffic ที่ไม่รู้จัก
ปลายทางจริงวนลูป: `ip route 0.0.0.0 0.0.0.0 Null0 250` (AD สูงมากเพื่อไม่ให้แทนที่ default
route ปกติ ใช้เป็น safety net สุดท้ายเท่านั้น)

---

## Step 90 — Lab เต็มรูปแบบ: Static Routing ทั้ง Enterprise Lab Topology

### Topology สรุปพร้อม IP Addressing ทั้งหมดที่ใช้ใน Lab นี้

```
                                    ┌────────────────┐
                     (Lo1: 8.8.8.8/32 จำลอง Internet Host)
                                    │    ISP-RTR      │
                                    └───┬────────┬────┘
                       203.0.113.0/30   │        │   203.0.113.4/30
                          (.1)          │        │          (.5)
                                 (.2)   │        │   (.6)
                         ┌──────────────┴┐      ┌┴──────────────┐
                         │   WAN-EDGE-1    │      │   WAN-EDGE-2    │
                         │  (Lo0 1.1.1.21) │══════│  (Lo0 1.1.1.22) │
                         └───────┬────────┘10.10.254.8/30└────────┬───┘
                        10.10.254.0/30(.9)     (.10)  10.10.254.4/30
                            (.1)  │                          │  (.1)
                            (.2)  │                          │  (.2)
                         ┌────────┴───────┐          ┌───────┴────────┐
                         │   CORE-SW1      │══Po1═════│   CORE-SW2      │
                         │  (Lo0 1.1.1.1)  │  10.10.253.0/30  (Lo0 1.1.1.2) │
                         │  SVI Gateway     │  (.1)/(.2)│  (backup gateway)│
                         └────────┬───────┘          └────────────────┘
                                  │
              ┌───────────┬───────┼────────┬───────────┐
          VLAN10/.1   VLAN20/.1  VLAN30/.1 VLAN40/.1  VLAN99/.1
        10.10.10.0/24 10.10.20.0/24 10.10.30.0/24 10.10.40.0/24 10.10.99.0/24
              │             │          │            │            │
          PC1-PC5      Phone/AP    Server1-3      WLC+AP     Mgmt Hosts
```

> **หมายเหตุ**: ในขั้นตอนนี้ยังไม่ได้ insert FW-1 เข้า path ระหว่าง WAN-EDGE-1 กับ CORE-SW1
> (จะทำใน Part 43-45 เรื่อง Security Architecture) และ SVI บน CORE-SW1 เป็นการตั้งค่าเบื้องต้น
> เพื่อให้ Lab ทำงานได้ end-to-end เท่านั้น รายละเอียดเรื่อง Router-on-a-Stick/SVI/Trunk เต็มรูปแบบ
> จะอยู่ใน **Part 10**

### ตาราง Static Route สรุปทั้งหมดที่ต้องมีในแต่ละอุปกรณ์

| อุปกรณ์ | Static Route ที่ต้อง config |
|---|---|
| ISP-RTR | route กลับเข้า `10.10.0.0/16` ผ่าน WAN-EDGE-1 (primary) + WAN-EDGE-2 (floating) |
| WAN-EDGE-1 | default route ไป ISP-RTR, summary route `10.10.0.0/16` ไป CORE-SW1 |
| WAN-EDGE-2 | floating default route ไป ISP-RTR (AD=5), floating summary route ไป CORE-SW2 (AD=5) |
| CORE-SW1 | default route ไป WAN-EDGE-1 (primary), floating default ไป WAN-EDGE-2 (AD=5) |
| CORE-SW2 | floating route สำรองทุกเส้นทาง (AD=5) เผื่อ CORE-SW1 ล่ม |

### Running-Config เต็ม: ISP-RTR

```
hostname ISP-RTR
!
interface Loopback1
 description ** Simulated Internet Host (8.8.8.8) **
 ip address 8.8.8.8 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** Link to WAN-EDGE-1 (Primary) **
 ip address 203.0.113.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** Link to WAN-EDGE-2 (Backup) **
 ip address 203.0.113.5 255.255.255.252
 no shutdown
!
! ISP มองเห็น Enterprise Lab ทั้งวงเป็น summary เดียว ผ่าน WAN-EDGE-1 เป็นหลัก
ip route 10.10.0.0 255.255.0.0 203.0.113.2
! Floating route สำรองผ่าน WAN-EDGE-2 (AD=5 - ใช้เมื่อ WAN-EDGE-1 ล่มเท่านั้น)
ip route 10.10.0.0 255.255.0.0 203.0.113.6 5
!
end
```

### Running-Config เต็ม: WAN-EDGE-1

```
hostname WAN-EDGE-1
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.21 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** Link to ISP-RTR (Primary Internet) **
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** Link to CORE-SW1 **
 ip address 10.10.254.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** Cross-link to WAN-EDGE-2 (Backup Path) **
 ip address 10.10.254.9 255.255.255.252
 no shutdown
!
! Default route ออก Internet ผ่าน ISP-RTR โดยตรง (primary, AD=1)
ip route 0.0.0.0 0.0.0.0 203.0.113.1
!
! Summary route กลับเข้า Campus ทุก VLAN ผ่าน CORE-SW1 (Fully Specified - อยู่บน Ethernet)
ip route 10.10.0.0 255.255.0.0 GigabitEthernet0/0/1 10.10.254.2
!
! Floating route สำรอง - ถ้า link ไป CORE-SW1 ตายให้อ้อมผ่าน WAN-EDGE-2 แทน (AD=5)
ip route 10.10.0.0 255.255.0.0 GigabitEthernet0/0/2 10.10.254.10 5
!
end
```

### Running-Config เต็ม: WAN-EDGE-2

```
hostname WAN-EDGE-2
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.22 255.255.255.255
!
interface GigabitEthernet0/0/0
 description ** Link to ISP-RTR (Backup Internet) **
 ip address 203.0.113.6 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/1
 description ** Link to CORE-SW2 **
 ip address 10.10.254.5 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0/2
 description ** Cross-link to WAN-EDGE-1 (Backup Path) **
 ip address 10.10.254.10 255.255.255.252
 no shutdown
!
! Floating default route ออก Internet ผ่าน ISP-RTR ของตัวเอง (AD=5 - สำรอง WAN-EDGE-1)
ip route 0.0.0.0 0.0.0.0 203.0.113.5 5
!
! Floating summary route กลับเข้า Campus ผ่าน CORE-SW2 (AD=5)
ip route 10.10.0.0 255.255.0.0 GigabitEthernet0/0/1 10.10.254.6 5
!
! เผื่อ CORE-SW2 เองก็ล่ม ให้ยังมีทาง cross-link ไปหา WAN-EDGE-1 ได้ (AD=10)
ip route 10.10.0.0 255.255.0.0 GigabitEthernet0/0/2 10.10.254.9 10
!
end
```

### Running-Config เต็ม: CORE-SW1 (ส่วนที่เกี่ยวกับ Routing/SVI)

```
hostname CORE-SW1
!
ip routing
!
interface Loopback0
 description ** Router-ID / Management Loopback **
 ip address 1.1.1.1 255.255.255.255
!
interface Vlan10
 description ** SALES Gateway **
 ip address 10.10.10.1 255.255.255.0
!
interface Vlan20
 description ** VOICE Gateway **
 ip address 10.10.20.1 255.255.255.0
!
interface Vlan30
 description ** SERVERS Gateway **
 ip address 10.10.30.1 255.255.255.0
!
interface Vlan40
 description ** WIFI Gateway **
 ip address 10.10.40.1 255.255.255.0
!
interface Vlan99
 description ** MGMT Gateway **
 ip address 10.10.99.1 255.255.255.0
!
interface GigabitEthernet1/0/1
 description ** Link to WAN-EDGE-1 **
 no switchport
 ip address 10.10.254.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/0/24
 description ** Routed link to CORE-SW2 (redundancy) **
 no switchport
 ip address 10.10.253.1 255.255.255.252
 no shutdown
!
! Default route ออก Internet ผ่าน WAN-EDGE-1 (primary, AD=1 - Fully Specified)
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.1
!
! Floating default สำรอง - ถ้า WAN-EDGE-1 ตาย ให้อ้อมผ่าน CORE-SW2 ไปออก WAN-EDGE-2 แทน (AD=5)
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/24 10.10.253.2 5
!
! Return path กลับไปหา WAN Transit / ISP subnet (จำเป็นสำหรับ traceroute/ping ตอบกลับถูกต้อง)
ip route 203.0.113.0 255.255.255.252 10.10.254.1
ip route 10.10.254.4 255.255.255.252 10.10.253.2 5
ip route 10.10.254.8 255.255.255.252 10.10.254.1
!
end
```

### Running-Config เต็ม: CORE-SW2 (สำรอง — SVI เดียวกัน แต่เป็น Backup Gateway)

```
hostname CORE-SW2
!
ip routing
!
interface Loopback0
 ip address 1.1.1.2 255.255.255.255
!
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
interface Vlan20
 ip address 10.10.20.2 255.255.255.0
interface Vlan30
 ip address 10.10.30.2 255.255.255.0
interface Vlan40
 ip address 10.10.40.2 255.255.255.0
interface Vlan99
 ip address 10.10.99.2 255.255.255.0
!
interface GigabitEthernet1/0/1
 description ** Link to WAN-EDGE-2 **
 no switchport
 ip address 10.10.254.6 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/0/24
 description ** Routed link to CORE-SW1 (redundancy) **
 no switchport
 ip address 10.10.253.2 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/1 10.10.254.5
ip route 0.0.0.0 0.0.0.0 GigabitEthernet1/0/24 10.10.253.1 5
!
end
```

> **หมายเหตุ**: การมี SVI IP คนละใบ (`.1` บน CORE-SW1, `.2` บน CORE-SW2) แบบนี้ยังไม่ใช่ HA
> Gateway ที่สมบูรณ์ (PC ยังต้องเลือก default gateway เดียวตายตัว) — จะแก้ให้เป็น Active/Standby
> Virtual Gateway ตัวเดียวด้วย **HSRP** ใน **Part 18** ในตอนนี้ PC1-PC5 และ Server1-3 ทุกเครื่อง
> ตั้ง default gateway เป็น CORE-SW1 (`.1`) เป็นหลัก

### Verification: ตรวจสอบ Routing Table ทุกจุด

```
CORE-SW1# show ip route
Gateway of last resort is 10.10.254.1 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 10.10.254.1, GigabitEthernet1/0/1
      10.0.0.0/8 is variably subnetted, 9 subnets, 3 masks
C       10.10.10.0/24 is directly connected, Vlan10
L       10.10.10.1/32 is directly connected, Vlan10
C       10.10.20.0/24 is directly connected, Vlan20
L       10.10.20.1/32 is directly connected, Vlan20
C       10.10.30.0/24 is directly connected, Vlan30
L       10.10.30.1/32 is directly connected, Vlan30
C       10.10.40.0/24 is directly connected, Vlan40
C       10.10.99.0/24 is directly connected, Vlan99
C       10.10.253.0/30 is directly connected, GigabitEthernet1/0/24
C       10.10.254.0/30 is directly connected, GigabitEthernet1/0/1
S       10.10.254.4/30 [5/0] via 10.10.253.2, GigabitEthernet1/0/24
S       10.10.254.8/30 [1/0] via 10.10.254.1, GigabitEthernet1/0/1
      203.0.113.0/24 is variably subnetted, 1 subnets, 1 masks
S       203.0.113.0/30 [1/0] via 10.10.254.1, GigabitEthernet1/0/1
```

```
WAN-EDGE-1# show ip route
Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*   0.0.0.0/0 [1/0] via 203.0.113.1, GigabitEthernet0/0/0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
S       10.10.0.0/16 [1/0] via 10.10.254.2, GigabitEthernet0/0/1
C       10.10.254.0/30 is directly connected, GigabitEthernet0/0/1
C       10.10.254.8/30 is directly connected, GigabitEthernet0/0/2
      203.0.113.0/24 is variably subnetted, 2 subnets, 2 masks
C       203.0.113.0/30 is directly connected, GigabitEthernet0/0/0
```

### Verification: Traceroute จาก PC1 (VLAN10) ไปยัง Internet และ Server1

```
PC1> traceroute 8.8.8.8

Tracing route to 8.8.8.8 over a maximum of 30 hops:
  1   1 ms   1 ms   1 ms   10.10.10.1        <- CORE-SW1 (SVI Gateway VLAN10)
  2   2 ms   2 ms   2 ms   10.10.254.1        <- WAN-EDGE-1
  3   3 ms   3 ms   3 ms   203.0.113.1        <- ISP-RTR
  4   3 ms   3 ms   3 ms   8.8.8.8            <- ปลายทาง (Loopback1 บน ISP-RTR)

Trace complete.
```

```
PC1> traceroute 10.10.30.50

Tracing route to 10.10.30.50 over a maximum of 30 hops:
  1   1 ms   1 ms   1 ms   10.10.10.1        <- CORE-SW1 (SVI Gateway VLAN10)
  2   1 ms   1 ms   1 ms   10.10.30.50        <- Server1 (routed ผ่าน SVI VLAN30 บน CORE-SW1 เดียวกัน)

Trace complete.
```

```
PC1> ping 8.8.8.8

Pinging 8.8.8.8 with 32 bytes of data:
Reply from 8.8.8.8: bytes=32 time=3ms TTL=252
Reply from 8.8.8.8: bytes=32 time=3ms TTL=252
Reply from 8.8.8.8: bytes=32 time=3ms TTL=252
Reply from 8.8.8.8: bytes=32 time=3ms TTL=252

Ping statistics for 8.8.8.8:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

Lab นี้ยืนยันว่าการ config Static Route แบบครบวงจร (Default Route + Summary Route + Floating
Static + SVI Gateway เบื้องต้น) ทำให้ **PC1 (VLAN10)** สามารถ:
1. คุยกับ **Server1 (VLAN30)** ได้ผ่าน CORE-SW1 (routing ระหว่าง VLAN ภายในเครื่องเดียว)
2. ออกไปถึง **Internet (จำลองด้วย 8.8.8.8 บน ISP-RTR)** ได้ผ่าน CORE-SW1 → WAN-EDGE-1 → ISP-RTR
3. ยังคงใช้งานได้ต่อเนื่องแม้ WAN-EDGE-1 หรือเส้นทางหลักล่ม เพราะมี **Floating Static Route**
   สำรองไว้ทุกจุดของ Topology

---

## แบบฝึกหัดทวนความเข้าใจ Part 9

1. ถ้า Router มี route `10.10.0.0/16` (AD=1, static) และ `10.10.10.0/24` (AD=110, OSPF) อยู่ใน
   ตารางพร้อมกัน และ packet ปลายทางคือ `10.10.10.55` Router จะเลือก route ไหน เพราะเหตุใด?
2. เพราะเหตุใดการใช้ static route แบบระบุ **exit-interface อย่างเดียว** บน Ethernet link
   จึงเป็นแนวทางที่ไม่แนะนำ ควรแก้ไขอย่างไร?
3. Administrative Distance ของ Static Route, OSPF, eBGP, และ iBGP มีค่าเท่าใดตามลำดับ?
4. Floating Static Route ทำงานอย่างไร และต้องตั้งค่าพารามิเตอร์ใดให้ต่างจาก static route ปกติ?
5. คำสั่งใดใช้เปิดใช้งาน IPv6 routing บน Router ก่อนที่จะ config `ipv6 route` ได้ และเพราะเหตุใด
   next-hop แบบ Link-Local ของ IPv6 static route จึงต้องระบุ exit-interface เสมอ?

**เฉลย:**
1. Router จะเลือก `10.10.10.0/24` (OSPF, AD=110) เพราะกฎ **Longest Prefix Match** ตัดสินก่อน
   AD เสมอ — prefix `/24` ยาวกว่า `/16` จึง match เจาะจงกว่าและชนะ แม้ AD จะแพงกว่าก็ตาม
   (AD ใช้ตัดสินเฉพาะกรณี prefix length เท่ากันเท่านั้น)
2. เพราะบน Multi-access network (Ethernet) Router จะเข้าใจผิดว่าทุก destination IP เป็น
   directly connected และพยายาม ARP หา MAC ของทุกปลายทาง ทำให้เกิดการทำงานผิดพลาดและ
   สิ้นเปลือง ARP table/CPU โดยไม่จำเป็น ควรแก้ไขด้วยการระบุ next-hop IP หรือใช้แบบ
   Fully Specified (exit-interface + next-hop IP พร้อมกัน) แทน
3. Static = 1, OSPF = 110, eBGP = 20, iBGP = 200
4. Floating Static Route ทำงานโดยตั้งค่า Administrative Distance ให้**สูงกว่า** AD ของ route
   หลักโดยเจตนา ทำให้ route นี้ไม่ถูกติดตั้งใน routing table ตราบใดที่ route หลักยังทำงานอยู่
   และจะถูกติดตั้งอัตโนมัติทันทีเมื่อ route หลักหายไป — ต้องระบุค่า AD ต่อท้ายคำสั่ง
   `ip route` (พารามิเตอร์ตัวเลขหลัง next-hop) ให้สูงกว่า AD ของแหล่งที่มาหลัก
5. คำสั่ง `ipv6 unicast-routing` ต้องเปิดเองเสมอ (ต่างจาก IPv4 ที่ `ip routing` เปิด default
   อยู่แล้วบน Router) ส่วน Link-Local Address ไม่ใช่ address ที่ routable ข้าม link ได้
   Router จึงไม่สามารถทำ recursive lookup หา exit-interface เองได้ ต้องระบุ exit-interface
   มาให้พร้อมกับ next-hop link-local เสมอ

---

## สรุป Part 9

Part นี้ปูพื้นฐาน **Static Routing** ครบทุกมิติที่จำเป็นก่อนเข้าสู่ Dynamic Routing Protocol:
Routing Table และ Longest Prefix Match, Syntax พื้นฐานและ Fully Specified Route,
Default Route, ตาราง Administrative Distance เต็ม, Floating Static สำหรับ Backup Path,
Host Route vs Summary Route, IPv6 Static Routing เทียบกับ Part 8, การ Troubleshoot
ปัญหาที่พบบ่อย และปิดท้ายด้วย Lab เต็มรูปแบบที่เชื่อม **ISP-RTR → WAN-EDGE-1/2 → CORE-SW1/2**
เข้าด้วยกันจนใช้งานได้จริงแบบ end-to-end ✅

**พร้อมสำหรับ Part 10**: เราจะเจาะลึกเรื่อง **Inter-VLAN Routing** อย่างเป็นทางการ ทั้ง
Router-on-a-Stick (Sub-interface + 802.1Q) และ SVI (Switched Virtual Interface) บน
Multilayer Switch ที่ Part นี้แตะไว้เพียงผิวเผิน พร้อมอธิบาย `ip routing`, Trunk ระหว่าง
CORE-SW1/2 กับ DIST-SW1-4 อย่างละเอียด

**ไปต่อ:** [Part 10 — Inter-VLAN Routing (RoaS, SVI) →](part-010-inter-vlan-routing.md)
