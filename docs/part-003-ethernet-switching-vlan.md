# Part 3 — Ethernet Switching & VLAN
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 21–30 จาก 1000**

> ต่อจาก [Part 2](part-002-ios-cli-basics.md) ที่สอน IOS CLI และ Initial Setup Script
> Part นี้เราจะลงลึกเรื่อง **การทำงานของ Switch ระดับ Layer 2** ตั้งแต่วิธี forward frame,
> แนวคิด Collision/Broadcast Domain, ไปจนถึงการสร้างและ config **VLAN** จริงบน `ACCESS-SW1–4`
> ตาม VLAN Plan ที่ประกาศไว้ใน Part 1 — Part นี้เป็นพื้นฐานสำคัญที่สุดก่อนเข้า Trunking (Part 4)

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 21 | Switch Forwarding Fundamentals: Flooding, MAC Learning, Aging, Table Overflow |
| 22 | Collision Domain vs Broadcast Domain |
| 23 | แนวคิด VLAN และความสัมพันธ์ VLAN : IP Subnet |
| 24 | การสร้าง VLAN บน Cisco IOS และ `show vlan brief` |
| 25 | Configuring Access Port บน ACCESS-SW1–4 ตาม VLAN Plan |
| 26 | Voice VLAN สำหรับ IP Phone บน ACCESS-SW1/ACCESS-SW2 |
| 27 | แนวคิด Native VLAN และความเสี่ยงของ VLAN 1 |
| 28 | VLAN Troubleshooting: ปัญหาที่พบบ่อยและคำสั่ง Verify |
| 29 | การลบ/ย้าย VLAN อย่างปลอดภัย, VLAN Range, Reserved VLAN |
| 30 | Lab: VLAN Configuration ฉบับสมบูรณ์สำหรับ ACCESS-SW1–4 |

---

## Step 21 — Switch Forwarding Fundamentals: Flooding, MAC Learning, Aging, Table Overflow

Switch (Layer 2 Bridge) ตัดสินใจ forward frame โดยใช้ **MAC Address Table** (เรียกอีกชื่อว่า
CAM Table — Content Addressable Memory) เท่านั้น ไม่สนใจ IP Address เลย กระบวนการทำงาน
ของ switch แบ่งเป็น 3 ขั้นตอนหลักที่ต้องเข้าใจให้แม่นเพราะเป็นพื้นฐานของทุกหัวข้อ L2 ต่อจากนี้:

### 1. Learning (การเรียนรู้ MAC Address)

ทุกครั้งที่ switch รับ frame เข้ามาที่ port ใด switch จะอ่าน **Source MAC Address** ของ frame
นั้น แล้วจับคู่กับ port ที่รับเข้ามา บันทึกลง MAC Address Table — ถ้า MAC นั้นมีอยู่แล้วแต่ port
เปลี่ยน (เช่น ย้าย PC ไปเสียบ port อื่น) switch จะ **update** entry ทันที (MAC move)

### 2. Forwarding Decision (การตัดสินใจส่ง frame)

Switch อ่าน **Destination MAC Address** แล้วเทียบกับ MAC Address Table:

| กรณี | Switch ทำอะไร |
|---|---|
| **Known Unicast** (Dest MAC มีใน table) | Forward ออกเฉพาะ port ที่ตรงกับ table (unicast forwarding) |
| **Unknown Unicast** (Dest MAC ไม่มีใน table) | **Flood** ออกทุก port ใน VLAN เดียวกัน ยกเว้น port ที่รับเข้ามา (ingress port) |
| **Broadcast** (Dest MAC = `FFFF.FFFF.FFFF`) | Flood ออกทุก port ใน VLAN เดียวกันเสมอ (ไม่มีการเรียนรู้ broadcast address) |
| **Multicast** (ไม่มี IGMP Snooping กำกับ) | Flood เหมือน broadcast ในเบื้องต้น (ปรับปรุงด้วย IGMP Snooping — Part 41) |

> Flooding เกิดขึ้นเสมอสำหรับ unknown unicast และ broadcast — นี่คือเหตุผลที่ **VLAN ยิ่งใหญ่
> ยิ่งมี broadcast traffic มาก** เพราะทุก broadcast frame ต้องส่งไปทุก port ในทุก VLAN นั้น

### 3. Aging (การหมดอายุของ MAC Table Entry)

MAC Address Table ไม่ได้เก็บ entry ตลอดไป — แต่ละ entry แบบ dynamic (เรียนรู้เอง) จะมี
**Aging Timer** ถ้าไม่มี frame จาก MAC นั้นเข้ามาอีกภายในเวลาที่กำหนด entry จะถูกลบทิ้ง
(ป้องกัน table เต็มไปด้วย MAC ที่ไม่ได้ใช้งานแล้ว เช่น PC ที่ปิดเครื่องไปแล้ว)

```
CORE-SW1# show mac address-table aging-time
Vlan    Aging Time
----    ----------
   1        300
  10        300
  20        300
  30        300

! ปรับ aging time ทั้ง switch (default = 300 วินาที = 5 นาที)
CORE-SW1(config)# mac address-table aging-time 600

! ปรับเฉพาะ VLAN
CORE-SW1(config)# mac address-table aging-time 180 vlan 10

! ลบ MAC table ทั้งหมดด้วยตนเอง (ใช้เวลา troubleshoot)
CORE-SW1# clear mac address-table dynamic
```

### Static MAC Entry (ไม่หมดอายุ — ใช้กับ server สำคัญ)

```
CORE-SW1(config)# mac address-table static 0050.5678.abcd vlan 30 interface GigabitEthernet1/0/1
```

### MAC Address Table Overflow (CAM Table Overflow / MAC Flooding Attack)

MAC Address Table เก็บอยู่ใน **TCAM (Ternary Content Addressable Memory)** ซึ่งมีขนาดจำกัด
(Catalyst 9200/9300 รองรับได้ประมาณ 8,000–32,000 entries ขึ้นกับรุ่น) เมื่อ table เต็ม:

- Switch **ไม่สามารถเรียนรู้ MAC ใหม่** ได้อีก
- Frame ที่ dest MAC ไม่อยู่ใน table (เพราะ table เต็มจนไม่ถูกเรียนรู้) จะถูก**flood ออกทุก port**
  เสมือน switch กลายเป็น hub ชั่วคราว — เรียกพฤติกรรมนี้ว่า **fail-open**
- นี่คือช่องโหว่ที่ผู้โจมตีใช้ประโยชน์เรียก **MAC Flooding Attack**: ส่ง frame จาก source MAC
  ปลอมจำนวนมหาศาลเพื่อทำให้ table เต็มโดยเจตนา แล้ว sniff traffic ทุก port ที่ถูก flood ออกมา
- **การป้องกัน**: `switchport port-security` จำกัดจำนวน MAC ต่อ port (สอนละเอียดใน **Part 21: Switch Security**)

```
CORE-SW1# show mac address-table count
          Dynamic Address Count:     1847
          Static  Address Count:        3
          Total Mac Addresses  :     1850
          Total Mac Address Space Available: 30722
```

---

## Step 22 — Collision Domain vs Broadcast Domain

สองคำนี้เป็นคำศัพท์พื้นฐานที่สุดของ Layer 1/2 ที่ต้องแยกให้ถูก เพราะอุปกรณ์แต่ละชนิด
มีผลต่อขอบเขตทั้งสองนี้ต่างกัน:

| แนวคิด | คำนิยาม | ใครเป็นคน "แบ่ง" ขอบเขต |
|---|---|---|
| **Collision Domain** | กลุ่มของอุปกรณ์ที่ frame อาจ "ชนกัน" ได้ถ้าส่งพร้อมกัน (เกิดเฉพาะ Half-Duplex) | **Switch** แบ่งทุก port เป็น collision domain ของตัวเอง (Router ก็แบ่งเช่นกัน) |
| **Broadcast Domain** | กลุ่มของอุปกรณ์ที่ได้รับ broadcast frame เดียวกันทั้งหมด | **Router** หรือ **VLAN boundary** เท่านั้นที่แบ่งได้ — Switch ธรรมดา (ไม่มี VLAN) ไม่แบ่ง |

### เปรียบเทียบอุปกรณ์แต่ละชนิด

```
HUB (Layer 1) — 1 Collision Domain, 1 Broadcast Domain
┌─────────────────────────────────┐
│   Hub  (ทุก port share bandwidth  │
│         และ share collision)      │
└──┬───────┬───────┬───────┬──────┘
   PC-A    PC-B    PC-C    PC-D
   └──────────── Collision Domain เดียวกันทั้งหมด ────────────┘
   └──────────── Broadcast Domain เดียวกันทั้งหมด ────────────┘

SWITCH (Layer 2, ไม่มี VLAN) — 4 Collision Domain, 1 Broadcast Domain
┌─────────────────────────────────┐
│           Switch (full-duplex)    │
└──┬───────┬───────┬───────┬──────┘
   PC-A    PC-B    PC-C    PC-D
   [CD1]   [CD2]   [CD3]   [CD4]   <- แต่ละ port = collision domain แยกกัน (full-duplex ไม่มี collision จริง)
   └──────────── Broadcast Domain เดียวกันทั้งหมด (broadcast flood ทุก port) ────────────┘

SWITCH + VLAN — 4 Collision Domain, 2 Broadcast Domain
┌─────────────────────────────────┐
│   Switch: Gi1/0/1-2=VLAN10, Gi1/0/3-4=VLAN20 │
└──┬───────┬───────┬───────┬──────┘
   PC-A    PC-B    PC-C    PC-D
   VLAN10  VLAN10  VLAN20  VLAN20
   └───BD1 (VLAN10)───┘    └───BD2 (VLAN20)───┘   <- broadcast ของ VLAN10 ไม่ไปถึง VLAN20

ROUTER — แบ่ง Broadcast Domain เสมอทุก interface (แต่ละ interface = subnet คนละ broadcast domain)
```

**สรุปกฎที่ต้องจำ:**
- Switch (ธรรมดา) → แบ่ง Collision Domain ทุก port, **ไม่แบ่ง** Broadcast Domain
- Switch + VLAN → แบ่งทั้ง Collision Domain **และ** Broadcast Domain (แบ่งตาม VLAN)
- Router / Layer 3 Switch (SVI) → แบ่ง Broadcast Domain ทุก interface/VLAN เสมอ
- ปัจจุบัน Full-Duplex ทำให้ "collision" ไม่เกิดขึ้นจริงแล้ว แต่คำว่า collision domain ยังใช้อธิบาย
  ขอบเขตทาง logical และยังอยู่ในข้อสอบ CCNA เสมอ

---

## Step 23 — แนวคิด VLAN และความสัมพันธ์ VLAN : IP Subnet

### ทำไมต้องมี VLAN

ถ้าทุก port บน switch อยู่ใน broadcast domain เดียวกันทั้งหมด (Flat Network) จะเกิดปัญหา:

1. **Broadcast Storm กระทบทุกคน** — broadcast จาก host เดียว (เช่น ARP request, DHCP Discover)
   จะถูกส่งไปทุก port ทุก department แม้ไม่เกี่ยวข้องกัน ยิ่งเครือข่ายใหญ่ยิ่งช้า
2. **ไม่มี Security Boundary** — แผนก Sales กับแผนก Server สามารถเห็น broadcast/traffic กันได้
   หมด ทำ ACL แยกไม่ได้ในระดับ L2
3. **Physical Location ผูกกับ Logical Group** — ต้องเสียบสาย PC ตาม department ที่นั่งอยู่จริง
   ย้ายที่นั่งแล้วต้องลากสายใหม่

**VLAN (Virtual LAN)** แก้ปัญหานี้โดยสร้าง **Broadcast Domain แบบ Logical** ที่ไม่ขึ้นกับ
physical port หรือตำแหน่งที่ตั้ง — สอง PC ที่เสียบ switch คนละตัว คนละชั้น สามารถอยู่ VLAN
เดียวกันได้ (ผ่าน Trunk — เรียนใน Part 4) และสอง PC ที่เสียบ switch ตัวเดียวกันติดกัน ก็สามารถ
อยู่คนละ VLAN ได้ (แยกกันสนิทเหมือนอยู่คนละ switch จริง)

### กฎทองคำ: VLAN : IP Subnet = 1 : 1

Best Practice ของ Cisco (และเป็นสมมติฐานพื้นฐานของ Routing ทุกโปรโตคอล) คือ **1 VLAN ต้องผูก
กับ 1 IP Subnet เสมอ** ห้ามใส่ 2 subnet ปนกันใน VLAN เดียว และห้ามให้ VLAN เดียวกันมีมากกว่า
1 subnet — นี่คือ VLAN Plan ที่เราประกาศไว้ตั้งแต่ Part 1 และจะใช้ตลอดหลักสูตร:

| VLAN ID | ชื่อ VLAN | IP Subnet | ใช้งาน | Access Switch ที่รองรับ |
|---|---|---|---|---|
| 10 | SALES | 10.10.10.0/24 | PC แผนกขาย | ACCESS-SW1, ACCESS-SW2 |
| 20 | VOICE | 10.10.20.0/24 | IP Phone (Voice VLAN) | ACCESS-SW1, ACCESS-SW2 |
| 30 | SERVERS | 10.10.30.0/24 | Server ภายใน (Server1–3) | ACCESS-SW3, ACCESS-SW4 |
| 40 | WIFI | 10.10.40.0/24 | Wireless Client ผ่าน AP/WLC | ACCESS-SW3, ACCESS-SW4 |
| 99 | MGMT | 10.10.99.0/24 | Management ของอุปกรณ์เครือข่ายทุกตัว | ทุกอุปกรณ์ |

> การ route ระหว่าง VLAN (เช่น VLAN10 คุยกับ VLAN30) ต้องผ่าน Layer 3 device (Router หรือ
> Multilayer Switch ที่มี SVI) เท่านั้น — เรียนละเอียดเรื่อง **Inter-VLAN Routing ใน Part 10**
> Part นี้ (Part 3) เราสร้างแค่ VLAN และ assign port เข้า VLAN ก่อน ยังไม่ทำ routing

---

## Step 24 — การสร้าง VLAN บน Cisco IOS และ `show vlan brief`

### สองวิธีสร้าง VLAN บน Cisco IOS

**วิธีที่ 1 (Legacy - ไม่แนะนำแล้ว): VLAN Database Mode**

```
CORE-SW1# vlan database
CORE-SW1(vlan)# vlan 10 name SALES
CORE-SW1(vlan)# apply
CORE-SW1(vlan)# exit
```

> `vlan database` mode ถูก **deprecate** ไปแล้วตั้งแต่ IOS สมัยใหม่ (ไม่มีใน IOS-XE บาง platform)
> **ห้ามใช้ในระบบจริงหรือข้อสอบปัจจุบัน** — ใช้เพื่อรู้จักไว้เผื่อเจอ legacy device เท่านั้น

**วิธีที่ 2 (มาตรฐานปัจจุบัน): Global Configuration Mode → VLAN Config Sub-mode**

```
ACCESS-SW1(config)# vlan 10
ACCESS-SW1(config-vlan)# name SALES
ACCESS-SW1(config-vlan)# exit

ACCESS-SW1(config)# vlan 20
ACCESS-SW1(config-vlan)# name VOICE
ACCESS-SW1(config-vlan)# exit
```

### สร้างหลาย VLAN พร้อมกันด้วย VLAN Range

```
ACCESS-SW1(config)# vlan 10,20,99
ACCESS-SW1(config-vlan)# exit
! หมายเหตุ: สร้างพร้อมกันได้ แต่ตั้งชื่อ (name) ทีละ VLAN เท่านั้น ต้อง vlan <id> ทีละตัวถ้าจะ name
```

### ตรวจสอบด้วย `show vlan brief`

```
ACCESS-SW1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi1/0/5, Gi1/0/6, Gi1/0/7, Gi1/0/8
                                                 Gi1/0/9, Gi1/0/10
10   SALES                            active    Gi1/0/1, Gi1/0/2, Gi1/0/3
20   VOICE                            active
99   MGMT                             active    Gi1/0/23
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
```

- **Status = active** → VLAN ใช้งานได้ปกติ
- **Status = act/unsup** → VLAN legacy (FDDI/Token Ring) ที่ platform ปัจจุบันไม่ support จริง
  (เป็น VLAN สงวนที่ IOS สร้างมาให้อัตโนมัติ เห็นทุกครั้งที่พิมพ์คำสั่งนี้ — ไม่ต้องตกใจ)
- คอลัมน์ **Ports** จะโชว์เฉพาะ port ที่เป็น **access port** ของ VLAN นั้น — port ที่เป็น trunk
  จะไม่โชว์ในคอลัมน์นี้ (เพราะ trunk พา VLAN ได้หลายตัว ดู Part 4)
- VLAN 20 ในตัวอย่างยังไม่มี port แสดง เพราะยังไม่ assign port ใดเข้า VLAN นี้เลย (VLAN ที่สร้าง
  ไว้แต่ไม่มี port ใช้งาน เรียกว่า VLAN "สร้างลอย" — ยังใช้งานได้ปกติ)

### คำสั่ง show ที่เกี่ยวข้องอื่นๆ

```
ACCESS-SW1# show vlan
ACCESS-SW1# show vlan id 10
ACCESS-SW1# show vlan name SALES
ACCESS-SW1# show running-config | section vlan
```

---

## Step 25 — Configuring Access Port บน ACCESS-SW1–4 ตาม VLAN Plan

**Access Port** คือ port ที่ถูก assign ให้อยู่ใน VLAN เดียวเท่านั้น (ตรงข้ามกับ Trunk Port ที่พา
ได้หลาย VLAN — เรียน Part 4) เป็น mode ที่ใช้กับ port ต่อ end device (PC, Server, Printer,
IP Phone) เสมอ คำสั่งหลัก 2 คำสั่งที่ต้องจำ:

```
Switch(config-if)# switchport mode access       ! บังคับ port เป็น access mode (ไม่ negotiate)
Switch(config-if)# switchport access vlan <id>  ! กำหนดว่า access port นี้อยู่ VLAN ไหน
```

> **สำคัญ**: ต้องพิมพ์ `switchport mode access` ก่อนเสมอ ถ้าไม่พิมพ์ port จะอยู่ใน mode
> `dynamic auto`/`dynamic desirable` (ค่า default ของ Catalyst ส่วนใหญ่) ซึ่งจะพยายาม negotiate
> trunk ผ่าน DTP กับอุปกรณ์ปลายทาง — เป็นความเสี่ยงด้าน security (VLAN Hopping) เรียนละเอียด
> ใน Part 4

### ACCESS-SW1 (Uplink → DIST-SW1, รองรับ VLAN 10 SALES + VLAN 20 VOICE)

```
ACCESS-SW1(config)# vlan 10
ACCESS-SW1(config-vlan)# name SALES
ACCESS-SW1(config-vlan)# exit
ACCESS-SW1(config)# vlan 20
ACCESS-SW1(config-vlan)# name VOICE
ACCESS-SW1(config-vlan)# exit
ACCESS-SW1(config)# vlan 99
ACCESS-SW1(config-vlan)# name MGMT
ACCESS-SW1(config-vlan)# exit
!
interface GigabitEthernet1/0/1
 description ** PC1 - Sales **
 switchport mode access
 switchport access vlan 10
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** PC2 - Sales **
 switchport mode access
 switchport access vlan 10
 no shutdown
!
interface GigabitEthernet1/0/3
 description ** PC3 - Sales **
 switchport mode access
 switchport access vlan 10
 no shutdown
```

### ACCESS-SW2 (Uplink → DIST-SW2, รองรับ VLAN 10 SALES + VLAN 20 VOICE)

```
ACCESS-SW2(config)# vlan 10
ACCESS-SW2(config-vlan)# name SALES
ACCESS-SW2(config-vlan)# exit
ACCESS-SW2(config)# vlan 20
ACCESS-SW2(config-vlan)# name VOICE
ACCESS-SW2(config-vlan)# exit
!
interface GigabitEthernet1/0/1
 description ** PC4 - Sales **
 switchport mode access
 switchport access vlan 10
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** PC5 - Sales **
 switchport mode access
 switchport access vlan 10
 no shutdown
```

### ACCESS-SW3 (Uplink → DIST-SW3, รองรับ VLAN 30 SERVERS + VLAN 40 WIFI)

```
ACCESS-SW3(config)# vlan 30
ACCESS-SW3(config-vlan)# name SERVERS
ACCESS-SW3(config-vlan)# exit
ACCESS-SW3(config)# vlan 40
ACCESS-SW3(config-vlan)# name WIFI
ACCESS-SW3(config-vlan)# exit
!
interface GigabitEthernet1/0/1
 description ** Server1 **
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast          ! port ต่อ server/end-host เปิด PortFast ได้ (เรียนละเอียด Part 5)
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Server2 **
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/3
 description ** Server3 **
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
```

### ACCESS-SW4 (Uplink → DIST-SW4, รองรับ VLAN 30 SERVERS + VLAN 40 WIFI)

```
ACCESS-SW4(config)# vlan 30
ACCESS-SW4(config-vlan)# name SERVERS
ACCESS-SW4(config-vlan)# exit
ACCESS-SW4(config)# vlan 40
ACCESS-SW4(config-vlan)# name WIFI
ACCESS-SW4(config-vlan)# exit
!
interface GigabitEthernet1/0/1
 description ** AP-1 (Wi-Fi VLAN) **
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** AP-2 (Wi-Fi VLAN) **
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
```

> **หมายเหตุเรื่อง Uplink ports (Gi1/0/24 ของทุก access switch ไปยัง DIST-SWx)**: ในหลักสูตรนี้
> เราจะ config ให้เป็น **Trunk Port** ที่พาได้หลาย VLAN พร้อมกัน ซึ่งจะสอนละเอียดใน **Part 4 —
> Trunking, VTP, DTP** ตอนนี้ปล่อย port นี้ไว้ที่ค่า default ก่อน (`switchport mode dynamic
> auto`) ยังไม่ต้อง config เพิ่ม

### Verify การตั้งค่า access port

```
ACCESS-SW1# show interfaces GigabitEthernet1/0/1 switchport
Name: Gi1/0/1
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Administrative Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 10 (SALES)
Trunking Native Mode VLAN: 1 (default)
Voice VLAN: none
```

---

## Step 26 — Voice VLAN สำหรับ IP Phone บน ACCESS-SW1/ACCESS-SW2

โครงสร้างที่พบบ่อยที่สุดในองค์กรจริงคือ **IP Phone เสียบสายเข้า PC และ PC เสียบสายเข้า Phone**
(Daisy-Chain ผ่าน built-in 3-port switch ในตัว IP Phone) — port บน Access Switch เพียง **1
port** ต้องรองรับ **2 VLAN พร้อมกัน**: Data VLAN (PC) และ Voice VLAN (Phone) โดยที่ยังนับเป็น
"access port" อยู่ (ไม่ต้องทำ trunk) — Cisco เรียกฟีเจอร์นี้ว่า **Voice VLAN**

### แนวคิดการทำงาน

```
        ┌─────────────┐        ┌─────────────┐
Wall ───┤  IP Phone    ├── PC ──┤     PC       │
        │ (VLAN 20)    │        │ (VLAN 10)    │
        └──────┬──────┘        └─────────────┘
               │  (สายเดียว - trunk เล็กๆ แบบ 802.1Q เฉพาะ voice tag)
        ┌──────┴──────┐
        │ ACCESS-SW1   │
        │ Gi1/0/1       │
        │ access vlan 10│ <- data traffic (untagged)
        │ voice vlan 20 │ <- voice traffic (tagged 802.1Q ภายใน link เดียวกัน)
        └─────────────┘
```

Switch จะสั่งงาน IP Phone ผ่าน **CDP** ให้ tag traffic เสียงด้วย VLAN 20 (802.1Q tag) ส่วน
traffic จาก PC ที่เสียบผ่าน phone จะถูกส่งแบบ untagged เข้า VLAN 10 ตามปกติ — เป็นกลไกพิเศษ
ที่ทำให้ port เดียวพา 2 VLAN ได้โดยไม่ต้องประกาศเป็น `trunk` เต็มรูปแบบ

### Configuration บน ACCESS-SW1 (แก้จาก Step 25 เพิ่ม voice vlan)

```
ACCESS-SW1(config)# interface range GigabitEthernet1/0/1 - 3
ACCESS-SW1(config-if-range)# switchport mode access
ACCESS-SW1(config-if-range)# switchport access vlan 10
ACCESS-SW1(config-if-range)# switchport voice vlan 20
ACCESS-SW1(config-if-range)# mls qos trust cos                 ! เชื่อ CoS marking ที่ phone ส่งมา (QoS พื้นฐาน)
ACCESS-SW1(config-if-range)# spanning-tree portfast
ACCESS-SW1(config-if-range)# exit
```

### Configuration บน ACCESS-SW2

```
ACCESS-SW2(config)# interface range GigabitEthernet1/0/1 - 2
ACCESS-SW2(config-if-range)# switchport mode access
ACCESS-SW2(config-if-range)# switchport access vlan 10
ACCESS-SW2(config-if-range)# switchport voice vlan 20
ACCESS-SW2(config-if-range)# mls qos trust cos
ACCESS-SW2(config-if-range)# spanning-tree portfast
ACCESS-SW2(config-if-range)# exit
```

> `switchport voice vlan <id>` ใช้ได้เฉพาะ port ที่เป็น `switchport mode access` เท่านั้น
> (ไม่สามารถใช้กับ trunk port) และ port ยังคงถูกมองว่าเป็น access port ใน `show vlan brief`
> (จะปรากฏใน column ของทั้ง data VLAN และจะไม่ปรากฏใน voice VLAN เพราะ voice VLAN แสดงแยกใน
> คำสั่งด้านล่าง)

### Verify Voice VLAN

```
ACCESS-SW1# show interfaces GigabitEthernet1/0/1 switchport
Name: Gi1/0/1
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Access Mode VLAN: 10 (SALES)
Voice VLAN: 20 (VOICE)

ACCESS-SW1# show interfaces GigabitEthernet1/0/1 trunk
   (ไม่แสดงผล เพราะ port นี้ไม่ใช่ trunk แม้จะพา 2 VLAN)

ACCESS-SW1# show mac address-table interface GigabitEthernet1/0/1
Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
  10    0050.7966.1a2b    DYNAMIC     Gi1/0/1        <- MAC ของ PC1
  20    0050.7966.99cc    DYNAMIC     Gi1/0/1        <- MAC ของ IP Phone
```

สังเกตว่า MAC Address Table เห็น **2 MAC บน port เดียวกันแต่คนละ VLAN** — นี่คือหลักฐานว่า
Voice VLAN ทำงานถูกต้อง (PC และ Phone ถูกแยก broadcast domain กันสนิทแม้เสียบสายเดียวกัน)

---

## Step 27 — แนวคิด Native VLAN และความเสี่ยงของ VLAN 1

**Native VLAN** เป็นแนวคิดที่ผูกกับ **Trunk Port** (802.1Q) โดยตรง — เราจะเรียน config
ละเอียดใน **Part 4** แต่ควรทำความเข้าใจแนวคิดพื้นฐานไว้ตั้งแต่ตอนนี้เพราะเกี่ยวกับ VLAN โดยตรง:

- ปกติเมื่อ trunk ส่ง frame ของ VLAN ใดๆ จะแปะ **802.1Q tag** (4 byte) บอกเลข VLAN ไปด้วยเสมอ
- แต่มี VLAN พิเศษ 1 ตัวต่อ trunk ที่จะ**ไม่แปะ tag** เรียกว่า **Native VLAN** — frame ของ
  native VLAN จะถูกส่งแบบ untagged ผ่าน trunk
- Native VLAN มีไว้เพื่อรองรับอุปกรณ์รุ่นเก่าที่ไม่รู้จัก 802.1Q (ปัจจุบันแทบไม่มีความจำเป็นแล้ว
  แต่ยังเป็นค่า default ที่ทุก trunk ต้องมี)
- **ค่า default ของ Native VLAN คือ VLAN 1** บนทุก trunk port ของ Cisco IOS

### ทำไม VLAN 1 เป็น Native VLAN ถึงเป็นความเสี่ยงด้านความปลอดภัย

1. **VLAN 1 เป็นค่า default ของทุก port ที่ยังไม่ configure** (access port ใหม่ทุก port จะอยู่
   VLAN 1 โดย default) ทำให้ VLAN 1 เต็มไปด้วย traffic broadcast จาก port ที่ไม่ได้ตั้งใจ
2. **VLAN 1 พา Control-Plane Protocol หลายตัวโดย default** เช่น CDP, VTP, STP BPDU (native)
   ทำให้เป็น VLAN ที่ "รู้" ข้อมูลของทุก switch ในเครือข่ายมากเกินจำเป็น
3. **VLAN Hopping Attack แบบ Double-Tagging**: ผู้โจมตีสามารถส่ง frame ที่มี 802.1Q tag ซ้อน
   กัน 2 ชั้น (tag ชั้นนอก = native VLAN ของ trunk ที่เชื่อมต่อ, tag ชั้นใน = VLAN เป้าหมาย)
   เมื่อ switch ตัวแรกดึง tag ชั้นนอกออก (เพราะเป็น native VLAN จึงไม่ tag) frame ที่เหลือจะถูก
   forward ไปยัง VLAN เป้าหมายที่ tag ไว้ชั้นใน — ทำให้ผู้โจมตี "กระโดด" ข้าม VLAN ได้โดยไม่ผ่าน
   Layer 3 device เลย ถ้า Native VLAN ของ attacker ตรงกับ native VLAN ของ trunk

### แนวทางป้องกัน (รายละเอียดเต็มใน Part 4)

- **เปลี่ยน Native VLAN ของทุก trunk ให้เป็น VLAN ที่ไม่ได้ใช้งานจริง** (เช่น VLAN 999 "BLACKHOLE")
  ไม่ใช้ VLAN 1
- **ห้ามใช้ VLAN 1 กับ user/device ใดๆ เลย** ปล่อยให้เป็น VLAN ระบบภายในอย่างเดียว
- ใช้คำสั่ง `switchport trunk native vlan <id>` บน trunk port ทุกตัว (สอนเต็มใน Part 4)
- Cisco Best Practice ปัจจุบัน: "VLAN 1 should carry no user traffic and ideally should not
  even be the native VLAN of any trunk"

---

## Step 28 — VLAN Troubleshooting: ปัญหาที่พบบ่อยและคำสั่ง Verify

### ปัญหาที่พบบ่อยที่สุดเรื่อง VLAN

| อาการ | สาเหตุที่เป็นไปได้ | วิธีตรวจสอบ |
|---|---|---|
| PC ต่อ port แล้ว ping ไม่ได้เลยแม้แต่ default gateway | Port อยู่ VLAN ผิด หรือ VLAN ที่ port อ้างอิงยังไม่ถูกสร้าง | `show vlan brief`, `show run int <port>` |
| Port ขึ้น "inactive" ใน `show vlan brief` | อ้างอิง VLAN ที่ **ไม่มีอยู่จริง** ใน VLAN database ของ switch นั้น | `show vlan id <id>` — ถ้าไม่มี ต้องสร้างก่อน |
| PC สอง department เห็น broadcast กันได้ (ไม่ควรเห็น) | Port ถูก assign VLAN ผิด (อยู่ VLAN เดียวกันโดยไม่ตั้งใจ) | `show vlan brief` เช็ค port อยู่ VLAN ไหนจริง |
| Voice ใช้งานไม่ได้ แต่ data ปกติ | ลืม `switchport voice vlan` หรือ Phone ไม่รับ CDP (ปิด CDP ไว้) | `show interfaces ... switchport`, `show cdp neighbors` |
| ต่อ trunk ระหว่าง switch แล้ว VLAN บางตัวใช้ไม่ได้ข้าม switch | **VLAN Mismatch**: VLAN นั้นถูกสร้างไว้ฝั่งหนึ่งแต่ไม่มีอีกฝั่ง (ปัญหานี้จะสอนลึกเรื่อง trunk ใน Part 4) | `show vlan brief` เทียบทั้งสองฝั่ง |
| Native VLAN mismatch warning ใน log | Native VLAN ของ trunk สองฝั่งตั้งไว้คนละเลข | `show interfaces trunk` (Part 4) |

### กรณีศึกษา: "Port ในสถานะ inactive"

```
ACCESS-SW1# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
10   SALES                            active    Gi1/0/1, Gi1/0/2
50   ???                              inactive  Gi1/0/5
```

Port `Gi1/0/5` ถูกสั่งให้ `switchport access vlan 50` แต่ **VLAN 50 ยังไม่ถูกสร้าง** ด้วย
`vlan 50` — วิธีแก้:

```
ACCESS-SW1(config)# vlan 50
ACCESS-SW1(config-vlan)# name GUEST
ACCESS-SW1(config-vlan)# exit
```

หลังสร้าง VLAN แล้ว port จะเปลี่ยนสถานะเป็น `active` ทันทีโดยไม่ต้องแก้ interface อะไรเพิ่ม

### คำสั่ง `show mac address-table` สำหรับ Troubleshoot

```
ACCESS-SW1# show mac address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
  10    0050.7966.1a2b    DYNAMIC     Gi1/0/1
  10    0050.7966.1a2c    DYNAMIC     Gi1/0/2
  20    0050.7966.99cc    DYNAMIC     Gi1/0/1
  99    001a.a2bb.cc10    DYNAMIC     Gi1/0/23

ACCESS-SW1# show mac address-table vlan 10
ACCESS-SW1# show mac address-table interface GigabitEthernet1/0/2
ACCESS-SW1# show mac address-table dynamic
```

ใช้เพื่อเช็คว่า: MAC ของ end device เห็นอยู่ **VLAN ที่คาดไว้จริงหรือไม่** และเห็นที่
**port ที่ถูกต้องหรือไม่** — ถ้า MAC ที่ควรอยู่ VLAN 10 กลับไปโผล่ VLAN อื่น แสดงว่า port
config ผิด หรือ PC ต่อผิดสาย

### คำสั่ง `show interfaces switchport` สำหรับ Troubleshoot

```
ACCESS-SW1# show interfaces GigabitEthernet1/0/2 switchport
Name: Gi1/0/2
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Administrative Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 10 (SALES)
Trunking Native Mode VLAN: 1 (default)
Administrative Native VLAN tagging: enabled
Voice VLAN: none
Administrative private-vlan host-association: none
Administrative private-vlan mapping: none
Operational private-vlan: none
Trunking VLANs Enabled: ALL
Capture Mode Disabled
Capture VLANs Allowed: ALL

Protected: false
Unknown unicast blocked: disabled
Unknown multicast blocked: disabled
Appliance trust: none
```

Field ที่ต้องดูเวลา troubleshoot:
- **Administrative Mode** ต้องตรงกับที่ตั้งใจ config ไว้ (`static access` = access port ที่บังคับแล้ว)
- **Operational Mode** คือสถานะจริงที่ port ทำงานอยู่ ณ ขณะนี้ — ถ้าไม่ตรงกับ Administrative Mode
  แสดงว่ามีปัญหาการ negotiate (มักเกิดกับ trunk — เรียน Part 4)
- **Access Mode VLAN** ต้องตรงกับ VLAN ที่ต้องการ

---

## Step 29 — การลบ/ย้าย VLAN อย่างปลอดภัย, VLAN Range, Reserved VLAN

### VLAN Range ทั้งหมดที่ Cisco IOS รองรับ

| Range | ชื่อเรียก | ใช้งานได้กับ | เก็บใน |
|---|---|---|---|
| 0, 4095 | Reserved (สงวนโดยมาตรฐาน 802.1Q) | ห้ามใช้ | - |
| 1 | Default VLAN | มีอยู่แล้ว ลบไม่ได้ | vlan.dat |
| 2–1001 | **Normal Range** | สร้าง/ลบ/name ได้ตามปกติ (VLAN Plan ของเราทั้งหมดอยู่ในช่วงนี้) | vlan.dat (sync ผ่าน VTP ได้ — Part 4) |
| 1002–1005 | **Reserved (Legacy)** — FDDI/Token Ring default | มีอยู่แล้วอัตโนมัติ แก้ไข/ลบไม่ได้ | vlan.dat |
| 1006–4094 | **Extended Range** | สร้าง/ใช้งานได้ (Catalyst รุ่นใหม่ทุกรุ่นรองรับ) แต่**ไม่ sync ผ่าน VTP รุ่น 1/2** ได้ (ต้องใช้ VTP v3 — Part 4) | running-config เท่านั้น (ไม่ใช่ vlan.dat ถ้า VTP v1/2) |

> VLAN Plan ของหลักสูตรนี้ (10, 20, 30, 40, 99) อยู่ใน **Normal Range** ทั้งหมด — เพียงพอสำหรับ
> การออกแบบ Enterprise ระดับ Campus ทั่วไป Extended Range มักใช้ในงาน Service Provider หรือ
> Data Center ที่ต้องการ VLAN จำนวนมากเกิน 1000

### ทำไม VLAN 1002–1005 ถึงลบไม่ได้

```
ACCESS-SW1(config)# no vlan 1002
Default VLANs (1002-1005) cannot be deleted.
```

VLAN เหล่านี้เป็นมาตรฐานเก่าที่ผูกกับ FDDI/Token Ring ซึ่ง IOS สร้างให้อัตโนมัติทุกครั้งและ
ล็อกไว้ไม่ให้ลบ (ปัจจุบันไม่มีใครใช้ FDDI/Token Ring จริงแล้ว แต่ IOS ยังคง backward-compatible
ไว้เสมอ)

### ขั้นตอนการลบ VLAN อย่างปลอดภัย

**อันตรายที่ต้องระวัง**: ถ้าลบ VLAN ที่ยังมี port ผูกอยู่ (`switchport access vlan <id>`)
port เหล่านั้นจะกลายเป็น **inactive ทันที** (ตัดขาดจากเครือข่ายจริง แม้ physical link ยัง up)

```
! ขั้นตอนที่ถูกต้อง: ย้าย port ออกจาก VLAN ก่อนเสมอ แล้วจึงลบ VLAN
ACCESS-SW1# show vlan id 50                       ! เช็คก่อนว่ามี port อะไรอยู่บ้าง
ACCESS-SW1(config)# interface range Gi1/0/5 - 6
ACCESS-SW1(config-if-range)# switchport access vlan 999    ! ย้ายไป VLAN parking/อื่นก่อน
ACCESS-SW1(config-if-range)# exit
ACCESS-SW1(config)# no vlan 50                    ! ลบ VLAN ได้อย่างปลอดภัยแล้ว
```

### การย้าย Port ระหว่าง VLAN (VLAN Move)

```
ACCESS-SW3(config)# interface GigabitEthernet1/0/1
ACCESS-SW3(config-if)# switchport access vlan 40      ! เปลี่ยนจาก VLAN 30 เป็น VLAN 40 ได้ทันที
ACCESS-SW3(config-if)# exit
```

- คำสั่งใหม่จะ **overwrite** ค่าเดิมทันที ไม่ต้อง `no switchport access vlan 30` ก่อน
- MAC Address ที่เคย learn ไว้ใน VLAN เดิมจาก port นี้จะยังอยู่ใน table จนกว่า aging timer
  จะหมดอายุ (ควร `clear mac address-table dynamic interface <port>` เพื่อเคลียร์ทันทีถ้าต้องการ)

```
ACCESS-SW3# clear mac address-table dynamic interface GigabitEthernet1/0/1
```

### เปลี่ยนชื่อ VLAN (ไม่กระทบ port ที่ผูกอยู่)

```
ACCESS-SW1(config)# vlan 10
ACCESS-SW1(config-vlan)# name SALES-DEPT
ACCESS-SW1(config-vlan)# exit
```

---

## Step 30 — Lab: VLAN Configuration ฉบับสมบูรณ์สำหรับ ACCESS-SW1–4

ต่อไปนี้คือ configuration ฉบับสมบูรณ์ที่รวมทุกอย่างจาก Step 24–26 เข้าด้วยกัน สำหรับ 4 Access
Switch ตาม Lab Topology หลัก — ใช้ต่อกับ Initial Setup Script จาก Part 2 ได้ทันที (คัดลอกวางต่อ
จาก `hostname` ของ template เดิม)

### ACCESS-SW1 — Full VLAN Config

```
hostname ACCESS-SW1
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface GigabitEthernet1/0/1
 description ** PC1 + IP Phone1 - Sales/Voice **
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** PC2 + IP Phone2 - Sales/Voice **
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/3
 description ** PC3 + IP Phone3 - Sales/Voice **
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/23
 description ** Local Management Station **
 switchport mode access
 switchport access vlan 99
 no shutdown
!
interface GigabitEthernet1/0/24
 description ** Uplink to DIST-SW1 (Trunk config ใน Part 4) **
 no shutdown
!
interface Vlan99
 description ** Management SVI **
 ip address 10.10.99.11 255.255.255.0
 no shutdown
!
end
copy running-config startup-config
```

### ACCESS-SW2 — Full VLAN Config

```
hostname ACCESS-SW2
!
vlan 10
 name SALES
vlan 20
 name VOICE
vlan 99
 name MGMT
!
interface GigabitEthernet1/0/1
 description ** PC4 + IP Phone4 - Sales/Voice **
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** PC5 + IP Phone5 - Sales/Voice **
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/24
 description ** Uplink to DIST-SW2 (Trunk config ใน Part 4) **
 no shutdown
!
interface Vlan99
 description ** Management SVI **
 ip address 10.10.99.12 255.255.255.0
 no shutdown
!
end
copy running-config startup-config
```

### ACCESS-SW3 — Full VLAN Config

```
hostname ACCESS-SW3
!
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
interface GigabitEthernet1/0/1
 description ** Server1 **
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** Server2 **
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/3
 description ** Server3 **
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/24
 description ** Uplink to DIST-SW3 (Trunk config ใน Part 4) **
 no shutdown
!
interface Vlan99
 description ** Management SVI **
 ip address 10.10.99.13 255.255.255.0
 no shutdown
!
end
copy running-config startup-config
```

### ACCESS-SW4 — Full VLAN Config

```
hostname ACCESS-SW4
!
vlan 30
 name SERVERS
vlan 40
 name WIFI
vlan 99
 name MGMT
!
interface GigabitEthernet1/0/1
 description ** AP-1 (Wi-Fi VLAN) **
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/2
 description ** AP-2 (Wi-Fi VLAN) **
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
!
interface GigabitEthernet1/0/24
 description ** Uplink to DIST-SW4 (Trunk config ใน Part 4) **
 no shutdown
!
interface Vlan99
 description ** Management SVI **
 ip address 10.10.99.14 255.255.255.0
 no shutdown
!
end
copy running-config startup-config
```

### Verification: ตรวจสอบผลลัพธ์รวมทั้ง 4 Switch

**1. `show vlan brief` บน ACCESS-SW1:**

```
ACCESS-SW1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi1/0/4, Gi1/0/5, Gi1/0/6
10   SALES                            active    Gi1/0/1, Gi1/0/2, Gi1/0/3
20   VOICE                            active
99   MGMT                             active    Gi1/0/23, Vl99
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
```

**2. `show interfaces status` บน ACCESS-SW3 (สรุปทุก port):**

```
ACCESS-SW3# show interfaces status

Port      Name               Status       Vlan       Duplex  Speed Type
Gi1/0/1   Server1            connected    30         a-full  a-1000 10/100/1000BaseTX
Gi1/0/2   Server2            connected    30         a-full  a-1000 10/100/1000BaseTX
Gi1/0/3   Server3            connected    30         a-full  a-1000 10/100/1000BaseTX
Gi1/0/4                      notconnect   1          auto    auto   10/100/1000BaseTX
Gi1/0/24  Uplink to DIST-SW3 connected    1          a-full  a-1000 10/100/1000BaseTX
```

**3. `show ip interface brief` เช็ค Management SVI ทั้ง 4 ตัว (เข้าที่ตัวใดตัวหนึ่งด้วย SSH ตาม Part 2):**

```
ACCESS-SW1# show ip interface brief | include Vlan
Vlan99                 10.10.99.11     YES manual up                    up

ACCESS-SW4# show ip interface brief | include Vlan
Vlan99                 10.10.99.14     YES manual up                    up
```

**4. ทดสอบ ping ระหว่าง Management SVI (ต้องผ่านได้ ถ้า DIST/CORE เชื่อมโยงกันแล้วจาก Part ก่อนหน้า):**

```
ACCESS-SW1# ping 10.10.99.13
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.99.13, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

> **หมายเหตุสำคัญ**: ในสถานะปัจจุบัน (จบ Part 3) การ ping ข้าม access switch ที่ต่างกันจะยัง
> **ไม่สำเร็จ** ถ้า uplink port (Gi1/0/24) ยังไม่ถูกตั้งเป็น trunk เพื่อพา VLAN 99 ข้าม switch
> — นี่คือเหตุผลที่ **Part 4 (Trunking)** ต้องเรียนต่อทันที เพื่อให้ VLAN ทั้งหมดสื่อสารข้าม
> switch ได้จริงตาม topology ที่ออกแบบไว้

---

## แบบฝึกหัดทวนความเข้าใจ Part 3

1. เมื่อ switch ได้รับ frame ที่ **destination MAC ไม่มีใน MAC Address Table** switch จะทำอะไร
   กับ frame นั้น และเรียกพฤติกรรมนี้ว่าอะไร?
2. Switch ธรรมดา (ไม่มี VLAN) แบ่ง **Collision Domain** และ **Broadcast Domain** อย่างไร —
   ต่างจาก Hub อย่างไร?
3. เพราะเหตุใดจึงต้องยึดกฎ "1 VLAN : 1 IP Subnet" และถ้าฝ่าฝืนกฎนี้จะเกิดปัญหาอะไร?
4. คำสั่ง 2 คำสั่งที่ต้อง config บน interface เพื่อทำให้เป็น access port ของ VLAN 30 คืออะไร?
5. ทำไม VLAN 1 จึงถือเป็นความเสี่ยงด้านความปลอดภัยเมื่อถูกใช้เป็น Native VLAN ของ trunk?

**เฉลย:**
1. Switch จะ **flood** frame นั้นออกทุก port ในวง VLAN เดียวกัน ยกเว้น port ที่รับเข้ามา
   เรียกพฤติกรรมนี้ว่า **Unknown Unicast Flooding**
2. Switch ธรรมดาแบ่ง **Collision Domain ทุก port** (แต่ละ port เป็น collision domain ของตัวเอง)
   แต่ **ไม่แบ่ง Broadcast Domain** (broadcast ยัง flood ไปทุก port เหมือน Hub) — ต่างจาก Hub
   ที่ไม่แบ่งทั้งสองอย่างเลย (ทุก port share collision domain และ broadcast domain เดียวกัน)
3. เพราะ Routing Protocol และการออกแบบเครือข่ายทั้งหมดอ้างอิงว่า 1 subnet = 1 broadcast domain
   ถ้าใส่หลาย subnet ใน VLAN เดียวจะทำให้ ARP/broadcast ปนกันข้าม subnet และการทำ Access Control
   ตาม VLAN ทำไม่ได้แม่นยำ
4. `switchport mode access` และ `switchport access vlan 30`
5. เพราะ VLAN 1 เป็นค่า default ของทุก port ที่ยังไม่ถูก config, พา control-plane protocol
   หลายตัวโดย default, และเปิดช่องให้เกิด **VLAN Hopping ผ่าน Double-Tagging Attack** ได้ง่าย
   ถ้า native VLAN ของ attacker ตรงกับ native VLAN ของ trunk

---

## สรุป Part 3

Part นี้ปูพื้นฐาน **Layer 2 Switching** ตั้งแต่กลไก flood/learn/age ของ MAC Address Table,
แนวคิด Collision/Broadcast Domain, ไปจนถึงการสร้างและ config **VLAN** จริงบน `ACCESS-SW1–4`
ครบทั้ง Access Port, Voice VLAN, และการ troubleshoot ปัญหาที่พบบ่อยที่สุด ✅ ตอนนี้ทุก Access
Switch มี VLAN และ port assignment ครบตาม Plan แล้ว แต่ยัง **สื่อสารข้าม switch ไม่ได้** เพราะ
uplink ยังไม่ใช่ trunk — **พร้อมสำหรับ Part 4**: เราจะ config **Trunking (802.1Q), VTP, และ
DTP** เพื่อให้ VLAN ทั้งหมดพาข้าม switch ได้จริงตาม topology ทั้งระบบ

**ไปต่อ:** [Part 4 — Trunking, VTP, DTP →](part-004-trunking-vtp-dtp.md)
