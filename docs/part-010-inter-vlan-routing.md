# Part 10 — Inter-VLAN Routing (Router-on-a-Stick, SVI)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 91–100 จาก 1000**

> ต่อจาก [Part 9 — Static Routing](part-009-static-routing.md) ที่สอนการ route ระหว่าง Network
> ด้วย static route บน Router แล้ว Part นี้จะพาไปแก้ปัญหาที่ใกล้ตัวกว่านั้นมาก นั่นคือ
> **การ route ระหว่าง VLAN ภายในองค์กรเดียวกัน** ซึ่งเป็นงานที่วิศวกร Enterprise Network ต้องทำ
> แทบทุกวัน — เราจะเรียนทั้งวิธีเก่า (**Router-on-a-Stick**) และวิธีที่ใช้จริงใน Production
> สมัยใหม่ (**SVI บน Multilayer Switch**) แล้วนำ SVI ไปติดตั้งจริงบน **DIST-SW1–4** ทั้ง 4 ตัว
> ใน Lab Topology หลักของหลักสูตร ปิดท้ายด้วย Lab ทดสอบ ping ข้าม VLAN ทั่วทั้ง Topology จริง
> **นี่คือ Part สุดท้ายของกลุ่มพื้นฐาน CCNA ก่อนเข้าสู่ Dynamic Routing Protocol (OSPF) ใน Part 11**

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 91 | ทำไมต้องมี Inter-VLAN Routing |
| 92 | Router-on-a-Stick (RoaS) — วิธีคลาสสิก |
| 93 | ข้อจำกัดของ Router-on-a-Stick |
| 94 | SVI (Switched Virtual Interface) บน Multilayer Switch |
| 95 | เปิดใช้งาน `ip routing` และรู้จัก Routed Port |
| 96 | คอนฟิก SVI เต็มรูปแบบ — DIST-SW1 และ DIST-SW2 |
| 97 | คอนฟิก SVI เต็มรูปแบบ — DIST-SW3 และ DIST-SW4 |
| 98 | เปรียบเทียบ RoaS vs SVI vs Routed Port |
| 99 | Troubleshooting Inter-VLAN Routing |
| 100 | Lab เต็มรูปแบบ: PC1 (VLAN10) ping ถึง Server1 (VLAN30) ทั่ว Topology |

---

## Step 91 — ทำไมต้องมี Inter-VLAN Routing

ย้อนกลับไป [Part 3](part-003-ethernet-switching-vlan.md) เราเรียนไปแล้วว่า **VLAN แต่ละใบคือ
Broadcast Domain ที่แยกจากกันโดยสมบูรณ์** — Switch (Layer 2) จะไม่ forward Frame ข้าม VLAN
ให้เองเด็ดขาด ไม่ว่า MAC Address ปลายทางจะอยู่ใน MAC Address Table หรือไม่ก็ตาม เพราะ VLAN
ถูกออกแบบมาเพื่อ**แยก** Broadcast Domain ตั้งแต่ต้น (นั่นคือเหตุผลที่มันมีอยู่)

ผลที่ตามมาคือ ถ้า PC1 อยู่ VLAN 10 (10.10.10.0/24) ต้องการคุยกับ Server1 ที่อยู่ VLAN 30
(10.10.30.0/24) — ทั้งสอง IP อยู่คนละ Subnet กัน ตามกฎพื้นฐานของ IP (Part 7) เมื่อปลายทางไม่ได้
อยู่ Subnet เดียวกัน Host จะไม่ ARP หา MAC ปลายทางตรงๆ แต่จะส่ง Packet ไปที่ **Default Gateway**
ก่อนเสมอ — และงานของ Default Gateway นี้ต้องทำที่ **Layer 3** เท่านั้น

```
VLAN 10 (SALES)                          VLAN 30 (SERVERS)
10.10.10.0/24                            10.10.30.0/24
┌─────────┐                              ┌─────────────┐
│  PC1    │   10.10.10.10                │  Server1    │  10.10.30.10
└────┬────┘                              └──────┬──────┘
     │                                          │
┌────┴─────────────────────────────────────────┴──────┐
│              Layer 2 Switch เพียงอย่างเดียว            │
│   MAC Table: รู้ MAC ทุกเครื่อง แต่ "ไม่มีสิทธิ์" forward   │
│   Frame ข้าม VLAN — Broadcast Domain ถูกแยกโดย design  │
└───────────────────────────────────────────────────────┘
        ❌  PC1 ping 10.10.30.10  →  Request timed out
```

### สิ่งที่เกิดขึ้นจริงถ้าไม่มี Layer 3 คั่นกลาง

1. PC1 เทียบ IP ปลายทาง (10.10.30.10) กับ Subnet ตัวเอง (10.10.10.0/24) → **ไม่อยู่ Subnet เดียวกัน**
2. PC1 ส่ง Packet ไปหา Default Gateway ที่ตั้งไว้ (เช่น 10.10.10.2)
3. ถ้าไม่มีอุปกรณ์ใดตอบสนองในฐานะ Gateway (ไม่มี Layer 3 Interface ในบ VLAN 10 เลย) →
   ARP Request หา Gateway จะไม่มีใครตอบ → PC1 แสดง error `Destination host unreachable`
   หรือค้างเป็น `Request timed out`

### บทสรุปของ Step นี้

> **กฎเหล็ก**: VLAN แยก Broadcast Domain แต่ **ไม่ได้แยก** ความต้องการสื่อสารข้าม Department ของ
> องค์กรจริง (Sales ต้องคุยกับ Server, Voice ต้องคุยกับ Internet ผ่าน Core ฯลฯ) ดังนั้นทุกครั้งที่ทำ
> VLAN Segmentation จะต้องมีอุปกรณ์ Layer 3 (Router หรือ Multilayer Switch) วางแผน route ระหว่าง
> VLAN เสมอ — เนื้อหาที่เหลือของ Part นี้คือ 2 วิธีหลักที่ Cisco ใช้ทำสิ่งนี้

---

## Step 92 — Router-on-a-Stick (RoaS) — วิธีคลาสสิก

**Router-on-a-Stick (RoaS)** คือวิธี Inter-VLAN Routing แบบดั้งเดิมที่สุด ใช้ **Router ธรรมดา**
(ไม่ต้องเป็น Multilayer Switch) เชื่อมเข้ากับ Switch ด้วยสาย **Trunk เส้นเดียว** แล้วสร้าง
**Sub-interface** แยกตาม VLAN บน Physical Interface เดียวกันของ Router — ชื่อ "on a Stick"
มาจากภาพที่ Router "เสียบไม้" (สาย Trunk เส้นเดียว) เข้า Switch

### Topology ตัวอย่าง (Lab สาธิตแยกต่างหาก ใช้ WAN-EDGE-1 ทดสอบ)

> เพื่อสาธิตวิธีนี้โดยไม่กระทบ Lab Topology จริงของหลักสูตร (ซึ่งจะใช้ SVI ตั้งแต่ Step 94
> เป็นต้นไป) เราจะสมมติ Lab แยกต่างหาก โดยดึงสาย Trunk ชั่วคราวจาก **WAN-EDGE-1** (ISR4451)
> เข้าไปที่พอร์ตว่างของ **ACCESS-SW1** เพื่อสาธิต concept เท่านั้น

```
                 Trunk เส้นเดียว (dot1Q)
   WAN-EDGE-1 ─────────────────────────── ACCESS-SW1
   Gi0/1                                   Gi0/2 (trunk)
   ├─ Gi0/1.10  (VLAN10 10.10.10.1/24)         │
   ├─ Gi0/1.20  (VLAN20 10.10.20.1/24)         ├─ Gi0/1  PC1   (VLAN10 access)
   └─ Gi0/1.99  (VLAN99 10.10.99.1/24)         └─ Gi0/3  Phone (VLAN20 access)
```

### คอนฟิกฝั่ง Switch (ACCESS-SW1) — ทำพอร์ตให้เป็น Trunk

```
ACCESS-SW1(config)# interface GigabitEthernet0/2
ACCESS-SW1(config-if)# description ** Trunk to WAN-EDGE-1 (RoaS demo) **
ACCESS-SW1(config-if)# switchport trunk encapsulation dot1q
ACCESS-SW1(config-if)# switchport mode trunk
ACCESS-SW1(config-if)# switchport trunk allowed vlan 10,20,99
ACCESS-SW1(config-if)# no shutdown
```

### คอนฟิกฝั่ง Router (WAN-EDGE-1) — สร้าง Sub-interface ต่อ VLAN

```
WAN-EDGE-1(config)# interface GigabitEthernet0/1
WAN-EDGE-1(config-if)# description ** Trunk to ACCESS-SW1 (RoaS demo) **
WAN-EDGE-1(config-if)# no ip address
WAN-EDGE-1(config-if)# no shutdown
WAN-EDGE-1(config-if)# exit

! Sub-interface สำหรับ VLAN 10 — ต้องมี encapsulation dot1Q ก่อนใส่ IP เสมอ
WAN-EDGE-1(config)# interface GigabitEthernet0/1.10
WAN-EDGE-1(config-subif)# encapsulation dot1Q 10
WAN-EDGE-1(config-subif)# ip address 10.10.10.1 255.255.255.0
WAN-EDGE-1(config-subif)# exit

! Sub-interface สำหรับ VLAN 20
WAN-EDGE-1(config)# interface GigabitEthernet0/1.20
WAN-EDGE-1(config-subif)# encapsulation dot1Q 20
WAN-EDGE-1(config-subif)# ip address 10.10.20.1 255.255.255.0
WAN-EDGE-1(config-subif)# exit

! Sub-interface สำหรับ VLAN 99 (Native VLAN ของ Trunk เส้นนี้ — ต้องระบุคำว่า native ด้วย)
WAN-EDGE-1(config)# interface GigabitEthernet0/1.99
WAN-EDGE-1(config-subif)# encapsulation dot1Q 99 native
WAN-EDGE-1(config-subif)# ip address 10.10.99.1 255.255.255.0
WAN-EDGE-1(config-subif)# exit
```

> **ทำไม Physical Interface (Gi0/1) ไม่ต้องใส่ IP?** เพราะ Physical Interface ถูกใช้เป็น Trunk
> เปล่าๆ (คุม Encapsulation เท่านั้น) — Traffic ที่ route จริงทั้งหมดวิ่งผ่าน Sub-interface
> `no shutdown` ที่ Physical Interface ยังจำเป็น เพราะถ้า Physical Down, Sub-interface ทุกตัว
> จะ Down ตามไปด้วยทันที

### คำสั่ง Verify

```
WAN-EDGE-1# show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/1         unassigned      YES manual up                    up
GigabitEthernet0/1.10      10.10.10.1      YES manual up                    up
GigabitEthernet0/1.20      10.10.20.1      YES manual up                    up
GigabitEthernet0/1.99      10.10.99.1      YES manual up                    up

WAN-EDGE-1# show ip route connected
      10.0.0.0/24 is subnetted, 3 subnets
C        10.10.10.0 is directly connected, GigabitEthernet0/1.10
C        10.10.20.0 is directly connected, GigabitEthernet0/1.20
C        10.10.99.0 is directly connected, GigabitEthernet0/1.99

WAN-EDGE-1# show vlans   ! ดูสถิติ traffic แยกตาม sub-interface/VLAN
```

ผลลัพธ์: PC1 (VLAN10, gateway 10.10.10.1) และ Phone (VLAN20, gateway 10.10.20.1) สามารถ ping
หากันได้แล้ว เพราะ WAN-EDGE-1 ทำหน้าที่ route ระหว่าง Sub-interface ให้

---

## Step 93 — ข้อจำกัดของ Router-on-a-Stick

RoaS ใช้งานได้จริงและยังพบเห็นได้ใน Branch Office ขนาดเล็กที่มี VLAN ไม่กี่ใบ แต่ **ไม่เหมาะกับ
Enterprise Campus** ด้วยเหตุผลสำคัญ 3 ข้อ:

| ปัญหา | รายละเอียด |
|---|---|
| **Single Point of Failure** | มี Router เพียงตัวเดียวทำหน้าที่ route ทุก VLAN ถ้า Router หรือสาย Trunk เส้นนั้นล่ม **Inter-VLAN Routing หยุดทำงานทั้งหมดทันที** ไม่มี Redundancy |
| **Bandwidth Bottleneck** | Traffic ทุก VLAN ที่ต้องข้าม VLAN ถูกบีบผ่าน **สาย Trunk เส้นเดียว** เช่น ถ้าเป็น Gi (1 Gbps) ต่อให้ Switch มี Backplane หลักร้อย Gbps การ Inter-VLAN Routing ทั้งหมดก็ยังถูกจำกัดที่ 1 Gbps |
| **Latency เพิ่มขึ้น** | Traffic ต้องวิ่งออกจาก Switch ไป Router แล้ววิ่งกลับเข้า Switch อีกครั้ง (Router-arm-round-trip) แทนที่จะ route ได้ทันทีภายใน Switch ตัวเดียว |
| **ไม่ scale ตามจำนวน VLAN** | ยิ่งมี VLAN มาก ยิ่งต้องสร้าง Sub-interface มาก แต่ Bandwidth รวมยังถูกจำกัดที่ Physical Link เส้นเดียวเท่าเดิม |

### ภาพเปรียบเทียบ Bandwidth

```
RoaS:     [Switch Backplane 176 Gbps] ⇄ (1x Gi Trunk = 1 Gbps) ⇄ [Router]
                                          ▲
                                   คอขวดอยู่ตรงนี้ — ไม่ว่า Switch จะแรงแค่ไหนก็ตาม

SVI:      [Multilayer Switch: Switch + Route ภายในตัวเดียวกัน ที่ Backplane เต็มความเร็ว]
                                          ▲
                                   ไม่มีคอขวด เพราะ Routing เกิดใน ASIC ตัวเดียวกับ Switching
```

> **สรุป**: RoaS เหมาะกับ Branch เล็กที่มี Router อยู่แล้ว (ประหยัด ไม่ต้องซื้อ Multilayer
> Switch เพิ่ม) แต่สำหรับ Distribution Layer ของ Enterprise Campus ซึ่งต้อง route Traffic
> ปริมาณสูงระหว่าง VLAN ตลอดเวลา **มาตรฐานคือใช้ SVI บน Multilayer Switch** ซึ่งเป็นหัวข้อของ
> Step ถัดไป และเป็นวิธีที่ Lab หลักของหลักสูตรนี้จะใช้ตั้งแต่นี้เป็นต้นไป

---

## Step 94 — SVI (Switched Virtual Interface) บน Multilayer Switch

**SVI (Switched Virtual Interface)** คือ Interface เสมือน (Virtual) ที่สร้างขึ้นแทน VLAN หนึ่งใบ
บน **Multilayer Switch** (Layer 3 Switch) — Syntax คือ `interface vlan <ID>` แล้วใส่ IP Address
ให้กลายเป็น **Default Gateway ของ VLAN นั้นทันที**

ต่างจาก RoaS ตรงที่ SVI **ไม่ต้องมีสาย Trunk แยกออกไปหา Router ภายนอกเลย** — การ Route
เกิดขึ้น**ภายใน ASIC ตัวเดียวกับที่ทำ Switching** ทำให้ได้ Throughput เท่า Wire-speed
(Line-rate) ของ Backplane Switch ทั้งหมด ไม่ถูกจำกัดด้วย Physical Link เส้นเดียวแบบ RoaS

### เงื่อนไขที่ต้องมีก่อนใช้ SVI ได้

| เงื่อนไข | รายละเอียด |
|---|---|
| 1. Hardware ต้องเป็น Multilayer Switch | ในหลักสูตรนี้คือ **CORE-SW1/2** (Catalyst 9500) และ **DIST-SW1–4** (Catalyst 9300) — **ACCESS-SW1–4** เป็น Layer 2 ล้วน (Catalyst 9200/2960) **ไม่สามารถสร้าง SVI ที่ทำหน้าที่ route ได้** |
| 2. ต้องเปิด `ip routing` แบบ Global | ค่า Default ของ Switch คือปิด (Layer 2 เท่านั้น) — รายละเอียดใน Step 95 |
| 3. VLAN ต้องมีอยู่จริงใน VLAN Database | ต้องสร้างด้วย `vlan <ID>` ก่อน (Part 3) ไม่งั้น SVI จะไม่ยอมขึ้น up/up |
| 4. ต้องมี Access Port ที่ Active อยู่ใน VLAN นั้น | ถ้าไม่มีพอร์ตไหน up อยู่ใน VLAN เลย SVI จะค้างที่ **up/down** เสมอ (รายละเอียดเต็มใน Step 99) |

### Syntax พื้นฐาน

```
DIST-SW1(config)# vlan 10
DIST-SW1(config-vlan)# name SALES
DIST-SW1(config-vlan)# exit
!
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# description ** Gateway for VLAN 10 - SALES **
DIST-SW1(config-if)# ip address 10.10.10.2 255.255.255.0
DIST-SW1(config-if)# no shutdown
```

### ภาพรวม SVI ในสถาปัตยกรรม 3-Tier ของหลักสูตรนี้

```
                     ┌──────────────────────────┐
                     │   CORE-SW1  <=Po1=> CORE-SW2  │   <- Routed only, ไม่มี SVI (ดู Step 98)
                     └─────┬──────────────────┬────┘
              (routed port) │                  │ (routed port)
                     ┌───────┴───────┐  ┌───────┴───────┐
                     │  DIST-SW1     │  │  DIST-SW3     │
                     │  SVI:         │  │  SVI:         │
                     │  Vlan10 .2    │  │  Vlan30 .2    │  <- Default Gateway ของแต่ละ VLAN
                     │  Vlan20 .2    │  │  Vlan40 .2    │     อยู่ที่ DIST Layer เสมอ
                     │  Vlan99 .2    │  │  Vlan99 .4    │
                     └───────┬───────┘  └───────┬───────┘
                        Trunk│                    │Trunk
                     ┌───────┴───────┐  ┌───────┴───────┐
                     │  ACCESS-SW1    │  │  ACCESS-SW3    │   <- Layer 2 ล้วน ไม่มี SVI ที่ route
                     └───────────────┘  └───────────────┘
```

> **นี่คือเหตุผลที่ DIST-SW1–4 ถูกเลือกเป็น Catalyst 9300 (Multilayer) ตั้งแต่ Part 1**
> ในขณะที่ ACCESS-SW1–4 เลือกเป็น Catalyst 9200/2960 (Layer 2 ล้วน) — การวาง SVI ไว้ที่
> **Distribution Layer** คือ Cisco Best Practice มาตรฐานของ Enterprise Campus 3-Tier Design

---

## Step 95 — เปิดใช้งาน `ip routing` และรู้จัก Routed Port

### 95.1 คำสั่ง `ip routing` — สวิตช์เปิด/ปิดความสามารถ Layer 3 ทั้งเครื่อง

Switch ตระกูล Catalyst (แม้เป็นรุ่น Multilayer เช่น 9300/9500) เมื่อแกะกล่องมาใหม่จะทำงานเป็น
**Layer 2 Switch ล้วนโดย Default** ต่อให้สร้าง SVI ใส่ IP ครบแล้ว ก็จะ**ไม่ route packet ระหว่าง
VLAN ให้เลย** จนกว่าจะเปิดคำสั่งนี้:

```
DIST-SW1(config)# ip routing
```

### เปรียบเทียบก่อน/หลังเปิด `ip routing`

```
! ก่อนเปิด ip routing (Layer 2-only mode)
DIST-SW1# show ip route
Codes: L - local, C - connected, S - static, ...
Gateway of last resort is not set
     10.0.0.0/24 is subnetted, 3 subnets
C       10.10.10.0 is directly connected, Vlan10     <- ขึ้นเป็น connected แต่ "ไม่ route" จริง
                                                          ระหว่าง VLAN แม้ SVI จะ up/up ก็ตาม
! เข้า config mode แล้วเปิด
DIST-SW1(config)# ip routing
DIST-SW1(config)# end

! หลังเปิด ip routing — SVI ทุกใบเริ่มทำหน้าที่ Gateway จริง สามารถ route ข้าม VLAN ได้ทันที
DIST-SW1# show ip route
     10.0.0.0/24 is subnetted, 4 subnets
C       10.10.10.0 is directly connected, Vlan10
C       10.10.20.0 is directly connected, Vlan20
C       10.10.99.0 is directly connected, Vlan99
S       10.10.30.0 [1/0] via 10.255.10.1              <- เริ่มเห็น route ไปยัง VLAN อื่นที่ remote
```

> **หมายเหตุ IOS-XE รุ่นใหม่**: Catalyst 9000 บาง Template (เช่น `switch template lanbase-routing`)
> อาจเปิด `ip routing` มาให้ตั้งแต่ต้นแล้ว แต่ **ควรใส่คำสั่งนี้ในทุก Script Config เสมอ** เพื่อไม่ให้
> พลาดเวลาย้าย Platform หรือ Reset อุปกรณ์ — เป็นคำสั่งที่ตรวจสอบง่ายและไม่มีผลเสียถ้าเปิดซ้ำ

### 95.2 Routed Port — อีกทางเลือกนอกจาก SVI (สำคัญมากสำหรับ Link จุดต่อจุด)

SVI เหมาะกับการทำ Gateway ให้ VLAN ที่มีหลาย Host (เช่น VLAN ผู้ใช้งาน) แต่สำหรับ **Link
จุดต่อจุด (Point-to-Point)** ระหว่าง Switch สองตัว เช่น DIST-SW1 ↔ CORE-SW1 การสร้าง VLAN
ใหม่ทั้งใบ + Trunk + SVI แค่เพื่อ Link เดียวถือว่าสิ้นเปลืองทรัพยากรและซับซ้อนเกินจำเป็น

**Routed Port** คือการเอาความเป็น "Switch" ออกจาก Physical Port นั้น ด้วยคำสั่ง `no switchport`
แล้วใส่ IP Address ลงบน Physical Interface ได้โดยตรง เหมือนกับ Interface บน Router ทั่วไป:

```
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# description ** Routed uplink to CORE-SW1 **
DIST-SW1(config-if)# no switchport
DIST-SW1(config-if)# ip address 10.255.10.2 255.255.255.252
DIST-SW1(config-if)# no shutdown
```

### เปรียบเทียบ SVI vs Routed Port สำหรับ Link จุดต่อจุด

| | SVI สำหรับ P2P Link | Routed Port |
|---|---|---|
| ต้องสร้าง VLAN เพิ่มไหม | ต้องสร้าง 1 VLAN ต่อ 1 Link | ไม่ต้อง |
| ต้องทำ Trunk ไหม | ต้อง (แม้มี VLAN เดียว) | ไม่ต้อง — เป็น Access L3 port ตรงๆ |
| ได้รับผลกระทบจาก STP ไหม | ได้ (อยู่ใน STP topology ของ VLAN นั้น) | **ไม่ได้** — ไม่มี STP บน Routed Port เลย |
| ความซับซ้อนของ Config | สูงกว่า | ต่ำกว่า ตรงไปตรงมา |
| ใช้ที่ไหนใน Lab หลักสูตรนี้ | Gateway ของ VLAN ผู้ใช้งาน (10,20,30,40,99) | Uplink ระหว่าง DIST↔CORE และ CORE↔CORE |

> **กฎการออกแบบที่ใช้ตลอดหลักสูตรนี้**: ใช้ **SVI** สำหรับ Gateway ของ VLAN ผู้ใช้งานที่ Distribution
> Layer และใช้ **Routed Port** สำหรับ Backbone Link ระหว่าง Distribution↔Core และ Core↔Core
> เสมอ — เป็นการออกแบบที่เรียกว่า **"Routed Core"** ซึ่งไม่มี VLAN ใดๆ วิ่งข้าม Core Layer เลย
> ทำให้ Core ไม่ต้องพึ่งพา Spanning-Tree และ Converge เร็วกว่าการทำ Core แบบ Layer 2 มาก

---

## Step 96 — คอนฟิก SVI เต็มรูปแบบ: DIST-SW1 และ DIST-SW2

DIST-SW1 และ DIST-SW2 รับผิดชอบ VLAN ที่ ACCESS-SW1/ACCESS-SW2 trunk ขึ้นมา คือ
**VLAN 10 (SALES), VLAN 20 (VOICE), VLAN 99 (MGMT)** ตาม IP Plan:

| VLAN | Subnet | DIST-SW1 SVI | DIST-SW2 SVI | Virtual Gateway (HSRP, Part 18) |
|---|---|---|---|---|
| 10 – SALES | 10.10.10.0/24 | 10.10.10.2 | 10.10.10.3 | 10.10.10.1 (ยังไม่ config ใน Part นี้) |
| 20 – VOICE | 10.10.20.0/24 | 10.10.20.2 | 10.10.20.3 | 10.10.20.1 (ยังไม่ config ใน Part นี้) |
| 99 – MGMT | 10.10.99.0/24 | 10.10.99.2 | 10.10.99.3 | — |

> **หมายเหตุสำคัญ**: ตอนนี้ PC/Host ในแต่ละ VLAN ต้องตั้ง Default Gateway ไปที่ IP จริงของ
> DIST-SW1 (`.2`) หรือ DIST-SW2 (`.3`) เพียงตัวใดตัวหนึ่งไปก่อน (ไม่มี IP กลางใช้ร่วมกัน) — ยังไม่มี
> Redundancy อัตโนมัติถ้า Switch ตัวที่เป็น Gateway ล่ม จนกว่าจะเรียน **HSRP ใน Part 18**
> ซึ่งจะรวม `.2` และ `.3` เป็น Virtual IP เดียว (`.1`) ให้ Host ใช้งานแทน

### คอนฟิกเต็มบน DIST-SW1

```
DIST-SW1(config)# hostname DIST-SW1
DIST-SW1(config)# ip routing

! สร้าง VLAN Database ก่อนเสมอ (ทวนจาก Part 3)
DIST-SW1(config)# vlan 10
DIST-SW1(config-vlan)# name SALES
DIST-SW1(config-vlan)# exit
DIST-SW1(config)# vlan 20
DIST-SW1(config-vlan)# name VOICE
DIST-SW1(config-vlan)# exit
DIST-SW1(config)# vlan 99
DIST-SW1(config-vlan)# name MGMT
DIST-SW1(config-vlan)# exit

! SVI ของแต่ละ VLAN
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# description ** Gateway VLAN10-SALES **
DIST-SW1(config-if)# ip address 10.10.10.2 255.255.255.0
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

DIST-SW1(config)# interface vlan 20
DIST-SW1(config-if)# description ** Gateway VLAN20-VOICE **
DIST-SW1(config-if)# ip address 10.10.20.2 255.255.255.0
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

DIST-SW1(config)# interface vlan 99
DIST-SW1(config-if)# description ** Gateway VLAN99-MGMT **
DIST-SW1(config-if)# ip address 10.10.99.2 255.255.255.0
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! Trunk ลงหา ACCESS-SW1 (พก VLAN10,20,99 ตาม design)
DIST-SW1(config)# interface GigabitEthernet1/0/1
DIST-SW1(config-if)# description ** Trunk to ACCESS-SW1 **
DIST-SW1(config-if)# switchport trunk encapsulation dot1q
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk allowed vlan 10,20,99
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! Trunk ลงหา ACCESS-SW2 (dual-homed สำหรับ redundancy - STP จัดการ path ตาม Part 5)
DIST-SW1(config)# interface GigabitEthernet1/0/2
DIST-SW1(config-if)# description ** Trunk to ACCESS-SW2 **
DIST-SW1(config-if)# switchport trunk encapsulation dot1q
DIST-SW1(config-if)# switchport mode trunk
DIST-SW1(config-if)# switchport trunk allowed vlan 10,20,99
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! Routed Port ขึ้นไปหา CORE-SW1 (ไม่ใช่ Trunk — ดู Step 95.2)
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# description ** Routed uplink to CORE-SW1 **
DIST-SW1(config-if)# no switchport
DIST-SW1(config-if)# ip address 10.255.10.2 255.255.255.252
DIST-SW1(config-if)# no shutdown
DIST-SW1(config-if)# exit

! Default route ชี้ขึ้น Core (ใช้ static route ไปก่อน จนกว่าจะเปลี่ยนเป็น OSPF ใน Part 11)
DIST-SW1(config)# ip route 0.0.0.0 0.0.0.0 10.255.10.1
```

### คอนฟิกเต็มบน DIST-SW2 (สลับ IP host เป็น `.3` และ uplink อีกวง)

```
DIST-SW2(config)# hostname DIST-SW2
DIST-SW2(config)# ip routing
DIST-SW2(config)# vlan 10
DIST-SW2(config-vlan)# name SALES
DIST-SW2(config-vlan)# exit
DIST-SW2(config)# vlan 20
DIST-SW2(config-vlan)# name VOICE
DIST-SW2(config-vlan)# exit
DIST-SW2(config)# vlan 99
DIST-SW2(config-vlan)# name MGMT
DIST-SW2(config-vlan)# exit

DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# description ** Gateway VLAN10-SALES **
DIST-SW2(config-if)# ip address 10.10.10.3 255.255.255.0
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface vlan 20
DIST-SW2(config-if)# description ** Gateway VLAN20-VOICE **
DIST-SW2(config-if)# ip address 10.10.20.3 255.255.255.0
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface vlan 99
DIST-SW2(config-if)# description ** Gateway VLAN99-MGMT **
DIST-SW2(config-if)# ip address 10.10.99.3 255.255.255.0
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface GigabitEthernet1/0/1
DIST-SW2(config-if)# description ** Trunk to ACCESS-SW1 **
DIST-SW2(config-if)# switchport trunk encapsulation dot1q
DIST-SW2(config-if)# switchport mode trunk
DIST-SW2(config-if)# switchport trunk allowed vlan 10,20,99
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface GigabitEthernet1/0/2
DIST-SW2(config-if)# description ** Trunk to ACCESS-SW2 **
DIST-SW2(config-if)# switchport trunk encapsulation dot1q
DIST-SW2(config-if)# switchport mode trunk
DIST-SW2(config-if)# switchport trunk allowed vlan 10,20,99
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface TenGigabitEthernet1/1/1
DIST-SW2(config-if)# description ** Routed uplink to CORE-SW1 **
DIST-SW2(config-if)# no switchport
DIST-SW2(config-if)# ip address 10.255.20.2 255.255.255.252
DIST-SW2(config-if)# no shutdown
DIST-SW2(config-if)# exit

DIST-SW2(config)# ip route 0.0.0.0 0.0.0.0 10.255.20.1
```

### Verify

```
DIST-SW1# show ip interface brief | exclude unassigned
Interface              IP-Address      OK? Method Status                Protocol
Vlan10                 10.10.10.2      YES manual up                    up
Vlan20                 10.10.20.2      YES manual up                    up
Vlan99                 10.10.99.2      YES manual up                    up
TenGigabitEthernet1/1/1 10.255.10.2    YES manual up                    up

DIST-SW1# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active
10   SALES                            active    Gi1/0/1, Gi1/0/2
20   VOICE                            active    Gi1/0/1, Gi1/0/2
99   MGMT                             active    Gi1/0/1, Gi1/0/2
```

---

## Step 97 — คอนฟิก SVI เต็มรูปแบบ: DIST-SW3 และ DIST-SW4

DIST-SW3 และ DIST-SW4 รับผิดชอบ VLAN ฝั่ง ACCESS-SW3/ACCESS-SW4 คือ
**VLAN 30 (SERVERS), VLAN 40 (WIFI), VLAN 99 (MGMT)**:

| VLAN | Subnet | DIST-SW3 SVI | DIST-SW4 SVI | Virtual Gateway (HSRP, Part 18) |
|---|---|---|---|---|
| 30 – SERVERS | 10.10.30.0/24 | 10.10.30.2 | 10.10.30.3 | 10.10.30.1 (ยังไม่ config ใน Part นี้) |
| 40 – WIFI | 10.10.40.0/24 | 10.10.40.2 | 10.10.40.3 | 10.10.40.1 (ยังไม่ config ใน Part นี้) |
| 99 – MGMT | 10.10.99.0/24 | 10.10.99.4 | 10.10.99.5 | — |

> **หมายเหตุการออกแบบ**: VLAN 99 (MGMT) ใช้ Subnet เดียวกันทั่วทั้งองค์กร (10.10.99.0/24) แต่ใน
> Lab นี้ถูกแบ่งเป็น 2 ขอบเขตทางกายภาพ (บล็อก DIST-SW1/2 กับบล็อก DIST-SW3/4) เนื่องจาก Uplink
> ขึ้น Core เป็น **Routed Port** (ไม่ใช่ Trunk) ทำให้ VLAN 99 ไม่ถูก Bridge ข้าม Core Layer ได้
> เรื่องนี้เป็นข้อจำกัดที่พบได้จริงเวลาออกแบบเครือข่าย — ในระบบ Production จริงมักแก้ด้วยการแยก
> **Out-of-Band Management Network** ต่างหาก (จะเรียนละเอียดใน Part 17) สำหรับตอนนี้ให้เข้าใจว่า
> ทั้งสองบล็อกยัง route หากันได้ปกติผ่าน Core เหมือน VLAN อื่นทุกประการ

### คอนฟิกเต็มบน DIST-SW3

```
DIST-SW3(config)# hostname DIST-SW3
DIST-SW3(config)# ip routing
DIST-SW3(config)# vlan 30
DIST-SW3(config-vlan)# name SERVERS
DIST-SW3(config-vlan)# exit
DIST-SW3(config)# vlan 40
DIST-SW3(config-vlan)# name WIFI
DIST-SW3(config-vlan)# exit
DIST-SW3(config)# vlan 99
DIST-SW3(config-vlan)# name MGMT
DIST-SW3(config-vlan)# exit

DIST-SW3(config)# interface vlan 30
DIST-SW3(config-if)# description ** Gateway VLAN30-SERVERS **
DIST-SW3(config-if)# ip address 10.10.30.2 255.255.255.0
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface vlan 40
DIST-SW3(config-if)# description ** Gateway VLAN40-WIFI **
DIST-SW3(config-if)# ip address 10.10.40.2 255.255.255.0
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface vlan 99
DIST-SW3(config-if)# description ** Gateway VLAN99-MGMT (block 2) **
DIST-SW3(config-if)# ip address 10.10.99.4 255.255.255.0
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet1/0/1
DIST-SW3(config-if)# description ** Trunk to ACCESS-SW3 **
DIST-SW3(config-if)# switchport trunk encapsulation dot1q
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk allowed vlan 30,40,99
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface GigabitEthernet1/0/2
DIST-SW3(config-if)# description ** Trunk to ACCESS-SW4 **
DIST-SW3(config-if)# switchport trunk encapsulation dot1q
DIST-SW3(config-if)# switchport mode trunk
DIST-SW3(config-if)# switchport trunk allowed vlan 30,40,99
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# interface TenGigabitEthernet1/1/1
DIST-SW3(config-if)# description ** Routed uplink to CORE-SW2 **
DIST-SW3(config-if)# no switchport
DIST-SW3(config-if)# ip address 10.255.30.2 255.255.255.252
DIST-SW3(config-if)# no shutdown
DIST-SW3(config-if)# exit

DIST-SW3(config)# ip route 0.0.0.0 0.0.0.0 10.255.30.1
```

### คอนฟิกเต็มบน DIST-SW4

```
DIST-SW4(config)# hostname DIST-SW4
DIST-SW4(config)# ip routing
DIST-SW4(config)# vlan 30
DIST-SW4(config-vlan)# name SERVERS
DIST-SW4(config-vlan)# exit
DIST-SW4(config)# vlan 40
DIST-SW4(config-vlan)# name WIFI
DIST-SW4(config-vlan)# exit
DIST-SW4(config)# vlan 99
DIST-SW4(config-vlan)# name MGMT
DIST-SW4(config-vlan)# exit

DIST-SW4(config)# interface vlan 30
DIST-SW4(config-if)# description ** Gateway VLAN30-SERVERS **
DIST-SW4(config-if)# ip address 10.10.30.3 255.255.255.0
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface vlan 40
DIST-SW4(config-if)# description ** Gateway VLAN40-WIFI **
DIST-SW4(config-if)# ip address 10.10.40.3 255.255.255.0
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface vlan 99
DIST-SW4(config-if)# description ** Gateway VLAN99-MGMT (block 2) **
DIST-SW4(config-if)# ip address 10.10.99.5 255.255.255.0
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface GigabitEthernet1/0/1
DIST-SW4(config-if)# description ** Trunk to ACCESS-SW3 **
DIST-SW4(config-if)# switchport trunk encapsulation dot1q
DIST-SW4(config-if)# switchport mode trunk
DIST-SW4(config-if)# switchport trunk allowed vlan 30,40,99
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface GigabitEthernet1/0/2
DIST-SW4(config-if)# description ** Trunk to ACCESS-SW4 **
DIST-SW4(config-if)# switchport trunk encapsulation dot1q
DIST-SW4(config-if)# switchport mode trunk
DIST-SW4(config-if)# switchport trunk allowed vlan 30,40,99
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# interface TenGigabitEthernet1/1/1
DIST-SW4(config-if)# description ** Routed uplink to CORE-SW2 **
DIST-SW4(config-if)# no switchport
DIST-SW4(config-if)# ip address 10.255.40.2 255.255.255.252
DIST-SW4(config-if)# no shutdown
DIST-SW4(config-if)# exit

DIST-SW4(config)# ip route 0.0.0.0 0.0.0.0 10.255.40.1
```

### Verify DIST-SW3

```
DIST-SW3# show ip interface brief | exclude unassigned
Interface              IP-Address      OK? Method Status                Protocol
Vlan30                 10.10.30.2      YES manual up                    up
Vlan40                 10.10.40.2      YES manual up                    up
Vlan99                 10.10.99.4      YES manual up                    up
TenGigabitEthernet1/1/1 10.255.30.2   YES manual up                    up
```

---

## Step 98 — เปรียบเทียบ RoaS vs SVI vs Routed Port

| หัวข้อ | Router-on-a-Stick | SVI | Routed Port |
|---|---|---|---|
| ต้องมี VLAN หรือไม่ | ต้อง (Trunk ทุก VLAN) | ต้อง | ไม่ต้อง |
| ทำงานบนอุปกรณ์ | Router ทั่วไป (ราคาถูกกว่า L3 Switch) | Multilayer Switch เท่านั้น | Multilayer Switch เท่านั้น |
| Throughput | จำกัดที่ Physical Trunk เส้นเดียว | เต็มความเร็ว Backplane ของ Switch | เต็มความเร็ว Port นั้นๆ |
| Redundancy | ต่ำ (Single Point of Failure) | สูง (คู่กับ HSRP/VRRP — Part 18) | สูง (คู่กับ Routing Protocol — Part 11+) |
| ความซับซ้อน Config | ปานกลาง (encapsulation dot1Q ต่อ VLAN) | ต่ำ (interface vlan ตรงไปตรงมา) | ต่ำที่สุด (ไม่มี VLAN/Trunk เกี่ยวข้อง) |
| ใช้ทำอะไร | Gateway ของหลาย VLAN บน Branch เล็ก | Gateway ของ VLAN ผู้ใช้งานที่ Distribution | Backbone Link ระหว่าง Switch (P2P) |
| ใช้ที่ไหนใน Lab หลักสูตรนี้ | สาธิตเท่านั้น (Step 92) — ไม่ใช้ใน Production Lab | **DIST-SW1–4** ทุกตัว (Step 96-97) | **DIST↔CORE** และ **CORE↔CORE** |
| Cost (ทั่วไป) | ต่ำที่สุด (ใช้ Router ที่มีอยู่แล้ว) | สูงกว่า (ต้องซื้อ L3 Switch) | รวมอยู่ใน Cost ของ L3 Switch แล้ว |

### กฎการเลือกใช้ในงานออกแบบ Enterprise 3-Tier จริง

```
                Access Layer  →  Layer 2 ล้วน (ไม่มี Routing เลย)
                Distribution  →  SVI  (Gateway ของทุก VLAN + HSRP/VRRP)   <- ใช้ SVI ที่นี่
                Core Layer    →  Routed Port ล้วน (ไม่มี VLAN ข้าม Core) <- ใช้ Routed Port ที่นี่
```

> **สรุปสำหรับข้อสอบ CCNA**: คำถามแนว "องค์กรมี VLAN 20 ใบ ต้องการ Inter-VLAN Routing ที่
> Throughput สูงและมี Redundancy" → คำตอบที่ถูกต้องคือ **SVI บน Multilayer Switch** เสมอ ไม่ใช่
> Router-on-a-Stick ยกเว้นโจทย์จะระบุชัดว่าเป็น **สาขาเล็ก (Branch) ที่มี VLAN ไม่กี่ใบ และมี Router
> อยู่แล้วโดยไม่มี Multilayer Switch** เท่านั้นที่ RoaS จะเป็นคำตอบที่เหมาะสมกว่า

---

## Step 99 — Troubleshooting Inter-VLAN Routing

เมื่อ Inter-VLAN Routing ไม่ทำงานตามที่ควร ให้ตรวจสอบตามลำดับนี้เสมอ (Layer 1 → Layer 3):

### ลำดับการตรวจสอบมาตรฐาน

```
1. show vlan brief              → VLAN มีอยู่จริงใน Database หรือไม่ และมีพอร์ตอยู่ใน VLAN นั้นบ้าง
2. show interfaces trunk        → Trunk พก (allow) VLAN ที่ต้องการอยู่จริงหรือไม่
3. show ip interface brief      → SVI ขึ้น up/up หรือไม่ (ดูรายละเอียดด้านล่าง)
4. show ip route                → มี route ไปยัง Subnet ปลายทางหรือไม่ (connected/static/dynamic)
5. show running-config          → ตรวจ ip routing ถูกเปิดจริงหรือไม่ ip address ถูกต้องหรือไม่
```

### ปัญหาที่พบบ่อยที่สุด: SVI ค้างที่ `up/down`

```
DIST-SW3# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
Vlan30                 10.10.30.2      YES manual up                    down     <-- ผิดปกติ!
```

**Status = up, Protocol = down** สำหรับ SVI (ต่างจาก Physical Interface) มีสาเหตุหลักเสมอ 2 อย่าง:

| สาเหตุ | วิธีตรวจสอบ | วิธีแก้ |
|---|---|---|
| **VLAN ไม่มีอยู่ใน VLAN Database** | `show vlan brief` ไม่เห็น VLAN 30 เลย | `vlan 30` แล้ว `name SERVERS` ใน global config |
| **ไม่มีพอร์ตไหน Active (up) อยู่ใน VLAN นั้นเลย** | `show vlan brief` เห็น VLAN 30 แต่ช่อง Ports ว่างเปล่า หรือพอร์ตที่มีอยู่ทั้งหมด down | เสียบสาย/เปิด `no shutdown` ที่พอร์ต Access หรือ Trunk ที่พก VLAN นั้นอย่างน้อย 1 เส้นให้ up จริง |

### ตัวอย่างจริง — ก่อนแก้และหลังแก้

```
! ก่อนแก้: VLAN 30 มีอยู่ แต่ยังไม่มี Trunk ใดพกมันขึ้นมาที่ DIST-SW3 เลย
DIST-SW3# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
30   SERVERS                          active                    <- ไม่มีพอร์ตเลย!

DIST-SW3# show ip interface brief | include Vlan30
Vlan30                 10.10.30.2      YES manual up                    down

! แก้โดยเปิด Trunk ที่พก VLAN 30 ให้ up จริง (ทำตาม Step 97)
DIST-SW3(config)# interface GigabitEthernet1/0/1
DIST-SW3(config-if)# switchport trunk allowed vlan 30,40,99
DIST-SW3(config-if)# no shutdown

! หลังแก้: Trunk ขึ้น + ACCESS-SW3 มีอย่างน้อย 1 พอร์ตที่ active ใน VLAN 30
DIST-SW3# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
30   SERVERS                          active    Gi1/0/1

DIST-SW3# show ip interface brief | include Vlan30
Vlan30                 10.10.30.2      YES manual up                    up      <- แก้แล้ว
```

### เช็คลิสต์ Troubleshooting เพิ่มเติมที่ต้องจำ

- `ip routing` ต้องเปิดเสมอ — ถ้าลืม จะเห็น SVI up/up ทุกใบ แต่ traffic ไม่ข้าม VLAN เลย (ไม่มี error
  ชัดเจน ต้องเช็คด้วย `show running-config | include ip routing`)
- Host ปลายทาง Default Gateway ต้องตรงกับ IP ของ SVI จริง (`.2` หรือ `.3` ไม่ใช่ `.1` เพราะยังไม่มี
  HSRP ใน Part นี้)
- Native VLAN Mismatch บน Trunk (Part 4) ทำให้บาง VLAN Route ไม่ได้ทั้งที่ Trunk ขึ้น up/up
- `show ip route` ไม่เห็น Subnet ปลายทางเลย = ปัญหาอยู่ที่ Static/Dynamic Routing ไม่ใช่ VLAN/SVI
  แล้ว (ต้องแยกให้ออกว่าปัญหาอยู่ "ใน VLAN" หรือ "ข้าม Distribution/Core")

---

## Step 100 — Lab เต็มรูปแบบ: PC1 (VLAN10) ping ถึง Server1 (VLAN30) ทั่ว Topology

นี่คือ Lab สรุปของ Part 10 — พิสูจน์ว่า Inter-VLAN Routing ทำงานได้จริง**ทั่วทั้ง Topology**
ไม่ใช่แค่ภายใน Distribution Block เดียว โดย PC1 (VLAN10, ผูกกับ DIST-SW1/2) ต้อง ping ไปถึง
Server1 (VLAN30, ผูกกับ DIST-SW3/4) ซึ่งอยู่คนละฝั่งของ Core ทั้งหมด

### Topology เต็มพร้อม IP Address ที่ใช้จริงใน Step นี้

```
PC1 10.10.10.10/24 gw .2                                  Server1 10.10.30.10/24 gw .2
        │                                                            │
   ACCESS-SW1 (VLAN10,20,99 trunk up)                     ACCESS-SW3 (VLAN30,40,99 trunk up)
        │                                                            │
   ┌────┴─────┐                                              ┌───────┴───┐
   │ DIST-SW1 │ Vlan10=.2  Vlan20=.2  Vlan99=.2               │ DIST-SW3  │ Vlan30=.2 Vlan40=.2 Vlan99=.4
   └────┬─────┘  Te1/1/1: 10.255.10.2/30                      └─────┬─────┘  Te1/1/1: 10.255.30.2/30
        │ routed                                                     │ routed
   ┌────┴──────────────────────┐                          ┌─────────┴────────┐
   │       CORE-SW1            │  Po1 (routed, no switchport)      │
   │  Te1/0/1: 10.255.10.1/30  │◄══ 10.255.0.0/30 ══►│  CORE-SW2         │
   │  Te1/0/2: 10.255.20.1/30  │  .1              .2  │  Te1/0/1: 10.255.30.1/30│
   └────────────────────────────┘                       │  Te1/0/2: 10.255.40.1/30│
                                                          └──────────────────┘
```

### 1) คอนฟิก Static Route ที่ CORE-SW1 และ CORE-SW2

Core Layer ในการออกแบบนี้เป็น **Routed Port ล้วน** (Step 95.2) ไม่มี SVI เลย และเชื่อมต่อกันเองผ่าน
`Port-channel1` (ตั้งไว้เป็น LACP ตั้งแต่ Part 6) ซึ่งตอนนี้ถูกแปลงเป็น Routed Port-channel:

```
! ===== CORE-SW1 =====
CORE-SW1(config)# ip routing
CORE-SW1(config)# interface Port-channel1
CORE-SW1(config-if)# description ** Routed backbone link to CORE-SW2 **
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.0.1 255.255.255.252
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/1
CORE-SW1(config-if)# description ** Routed downlink to DIST-SW1 **
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.10.1 255.255.255.252
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

CORE-SW1(config)# interface TenGigabitEthernet1/0/2
CORE-SW1(config-if)# description ** Routed downlink to DIST-SW2 **
CORE-SW1(config-if)# no switchport
CORE-SW1(config-if)# ip address 10.255.20.1 255.255.255.252
CORE-SW1(config-if)# no shutdown
CORE-SW1(config-if)# exit

! Static route: subnet ของฝั่ง DIST-SW3/4 อยู่หลัง CORE-SW2 ทั้งหมด
CORE-SW1(config)# ip route 10.10.30.0 255.255.255.0 10.255.0.2
CORE-SW1(config)# ip route 10.10.40.0 255.255.255.0 10.255.0.2
CORE-SW1(config)# ip route 10.10.99.4 255.255.255.254 10.255.0.2

! ===== CORE-SW2 =====
CORE-SW2(config)# ip routing
CORE-SW2(config)# interface Port-channel1
CORE-SW2(config-if)# description ** Routed backbone link to CORE-SW1 **
CORE-SW2(config-if)# no switchport
CORE-SW2(config-if)# ip address 10.255.0.2 255.255.255.252
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/1
CORE-SW2(config-if)# description ** Routed downlink to DIST-SW3 **
CORE-SW2(config-if)# no switchport
CORE-SW2(config-if)# ip address 10.255.30.1 255.255.255.252
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

CORE-SW2(config)# interface TenGigabitEthernet1/0/2
CORE-SW2(config-if)# description ** Routed downlink to DIST-SW4 **
CORE-SW2(config-if)# no switchport
CORE-SW2(config-if)# ip address 10.255.40.1 255.255.255.252
CORE-SW2(config-if)# no shutdown
CORE-SW2(config-if)# exit

! Static route: subnet ของฝั่ง DIST-SW1/2 อยู่หลัง CORE-SW1 ทั้งหมด
CORE-SW2(config)# ip route 10.10.10.0 255.255.255.0 10.255.0.1
CORE-SW2(config)# ip route 10.10.20.0 255.255.255.0 10.255.0.1
CORE-SW2(config)# ip route 10.10.99.2 255.255.255.254 10.255.0.1
```

> **สังเกต**: ต้องเขียน static route ทีละคู่ Subnet ด้วยมือ ยิ่งมี VLAN/Site มาก ยิ่งต้องเขียน
> route จำนวนมาก (N×M) และถ้า Topology เปลี่ยน (เพิ่ม Link, Link ล่ม) static route **ไม่ปรับตัว
> เองเลย** ต้องเข้าไปแก้ทุกจุดด้วยมือ — **นี่คือข้อจำกัดที่แท้จริงของ Static Routing ในสเกล
> Enterprise และเป็นเหตุผลหลักที่ Part 11 เป็นต้นไปจะเปลี่ยนมาใช้ Dynamic Routing Protocol
> (OSPFv2) ซึ่งจะเรียนรู้ Route เหล่านี้เองอัตโนมัติทั้งหมด**

### 2) คอนฟิกฝั่ง Host (สำหรับ Lab จำลองด้วย PC/Server จริงหรือ Packet Tracer)

```
PC1>   ip address 10.10.10.10 255.255.255.0
PC1>   ip default-gateway 10.10.10.2

Server1>   ip address 10.10.30.10 255.255.255.0
Server1>   ip default-gateway 10.10.30.2
```

### 3) ทดสอบด้วย ping จาก PC1

```
PC1> ping 10.10.30.10

Pinging 10.10.30.10 with 32 bytes of data:
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125
Reply from 10.10.30.10: bytes=32 time=1ms TTL=125
Reply from 10.10.30.10: bytes=32 time=1ms TTL=125
Reply from 10.10.30.10: bytes=32 time=2ms TTL=125

Ping statistics for 10.10.30.10:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

### 4) ยืนยัน Path ด้วย traceroute — เห็นทุก Hop ที่แพ็กเก็ตวิ่งผ่านจริง

```
PC1> tracert 10.10.30.10

Tracing route to 10.10.30.10 over a maximum of 30 hops:

  1    1 ms    1 ms    1 ms   10.10.10.2       <- SVI ของ DIST-SW1 (Default Gateway)
  2    1 ms    1 ms    1 ms   10.255.0.1        <- Port-channel1 ของ CORE-SW1 (ไม่ต้องผ่านไป
                                                    Te1/0/1 เพราะปลายทางอยู่หลัง CORE-SW2)
  3    2 ms    1 ms    2 ms   10.255.30.2       <- Routed uplink ของ DIST-SW3
  4    2 ms    2 ms    1 ms   10.10.30.10       <- Server1 ถึงปลายทาง

Trace complete.
```

### 5) ตรวจ Routing Table ที่จุดสำคัญเพื่อยืนยันภาพรวม

```
DIST-SW1# show ip route
Gateway of last resort is 10.255.10.1 to network 0.0.0.0

C       10.10.10.0/24 is directly connected, Vlan10
C       10.10.20.0/24 is directly connected, Vlan20
C       10.10.99.0/24 is directly connected, Vlan99
C       10.255.10.0/30 is directly connected, TenGigabitEthernet1/1/1
S*      0.0.0.0/0 [1/0] via 10.255.10.1

CORE-SW1# show ip route static
S       10.10.30.0/24 [1/0] via 10.255.0.2
S       10.10.40.0/24 [1/0] via 10.255.0.2
S       10.10.99.4/31 [1/0] via 10.255.0.2

DIST-SW3# show ip route
C       10.10.30.0/24 is directly connected, Vlan30
C       10.10.40.0/24 is directly connected, Vlan40
C       10.10.99.0/24 is directly connected, Vlan99
C       10.255.30.0/30 is directly connected, TenGigabitEthernet1/1/1
S*      0.0.0.0/0 [1/0] via 10.255.30.1
```

Ping และ Traceroute สำเร็จ + Routing Table ทุกจุดมี Route ครบตามที่ออกแบบไว้ → **Inter-VLAN
Routing ทำงานได้ทั่วทั้ง Topology ของ Enterprise Lab หลักสูตรนี้แล้วอย่างสมบูรณ์**

---

### สรุปภาพรวม CCNA Part 1–10 — สิ่งที่ Topology ของเราทำได้แล้ว ณ จุดนี้

| Part | หัวข้อ | สิ่งที่เพิ่มเข้า Topology |
|---|---|---|
| 1 | Networking Fundamentals | ประกาศ Topology หลัก + IP/VLAN Plan ทั้งหมด |
| 2 | IOS CLI Basics | Initial Setup Script (hostname, SSH, banner) ทุกอุปกรณ์ |
| 3 | Ethernet Switching & VLAN | สร้าง VLAN 10/20/30/40/99 บนทุก Switch |
| 4 | Trunking, VTP, DTP | Trunk ระหว่าง Access↔Distribution |
| 5 | STP/RSTP/MSTP | ป้องกัน Loop บน Trunk ที่ Redundant |
| 6 | EtherChannel | Po1 ระหว่าง CORE-SW1↔CORE-SW2 (LACP) |
| 7 | IPv4 Addressing & Subnetting | ออกแบบ Subnet /24 ต่อ VLAN + /30 สำหรับ Link P2P |
| 8 | IPv6 Fundamentals | เตรียมพื้นฐาน Dual-Stack (ยังไม่ deploy จริงใน Lab) |
| 9 | Static Routing | Static Route พื้นฐานระหว่าง Router |
| **10** | **Inter-VLAN Routing** | **SVI ที่ DIST-SW1–4 ครบทุก VLAN + Routed Port ที่ Core + Static Route เชื่อมทั้ง Topology — PC1 ping Server1 ได้แล้วทั่ว Enterprise Lab** |

> ตอนนี้ Lab ของเรามี **Layer 2 (VLAN/Trunk/STP/EtherChannel)** และ **Layer 3 พื้นฐาน
> (IP Plan/Static Routing/Inter-VLAN Routing)** ครบสมบูรณ์แล้ว — พร้อมสำหรับการเปลี่ยนจาก
> Static Route ที่ต้องเขียนเองทุกจุด ไปเป็น **Dynamic Routing Protocol** ที่เรียนรู้ Topology
> และปรับตัวเองอัตโนมัติ ซึ่งเป็นหัวข้อของ Part 11 เป็นต้นไป

---

## แบบฝึกหัดทวนความเข้าใจ Part 10

1. เพราะเหตุใด Layer 2 Switch เพียงอย่างเดียวจึงไม่สามารถ forward Frame ข้าม VLAN ให้ได้ แม้จะรู้
   MAC Address ปลายทางอยู่แล้วก็ตาม?
2. ข้อจำกัดที่สำคัญที่สุด 2 ข้อของ Router-on-a-Stick คืออะไร และเพราะเหตุใด Enterprise Campus
   จึงไม่นิยมใช้วิธีนี้ที่ Distribution Layer?
3. คำสั่งใดที่ต้องเปิดแบบ Global ก่อน SVI บน Multilayer Switch จะสามารถ route ข้าม VLAN ได้จริง?
4. ถ้า `show ip interface brief` แสดง SVI ของ VLAN หนึ่งเป็น `up/down` (Status=up, Protocol=down)
   สาเหตุที่เป็นไปได้มากที่สุด 2 ข้อคืออะไร และแก้อย่างไร?
5. ในการออกแบบ 3-Tier ของหลักสูตรนี้ เพราะเหตุใด Link ระหว่าง DIST-SW1 กับ CORE-SW1 จึงถูกทำ
   เป็น **Routed Port** แทนที่จะเป็น **SVI + Trunk**?

**เฉลย:**
1. เพราะ VLAN ถูกออกแบบมาเพื่อแยก Broadcast Domain ที่ Layer 2 โดยเจตนา — Switch จะ block
   การ forward ข้าม VLAN เสมอไม่ว่า MAC Table จะมีข้อมูลหรือไม่ ต้องมีอุปกรณ์ Layer 3 (Router
   หรือ SVI บน Multilayer Switch) ทำหน้าที่ route แทน
2. (1) Single Point of Failure — Router ตัวเดียวล่มแล้ว Inter-VLAN Routing หยุดทั้งหมด และ
   (2) Bandwidth Bottleneck — Traffic ทุก VLAN ถูกบีบผ่าน Trunk เส้นเดียว ไม่ scale กับ
   ปริมาณ Traffic ระดับ Enterprise
3. `ip routing` (Global Configuration Mode)
4. สาเหตุ: (1) VLAN นั้นไม่มีอยู่จริงใน VLAN Database (ต้องสร้างด้วย `vlan <ID>`) หรือ
   (2) ไม่มีพอร์ตใดที่ Active (up) อยู่ใน VLAN นั้นเลย — แก้โดยตรวจสอบ `show vlan brief` แล้ว
   เปิด/เสียบสาย Trunk หรือ Access Port ที่พก VLAN นั้นให้ up จริงอย่างน้อย 1 พอร์ต
5. เพราะ Link ระหว่าง Distribution↔Core เป็น Point-to-Point ที่ไม่มี Host หลายเครื่องต้องใช้
   ร่วมกัน การใช้ Routed Port ไม่ต้องสร้าง VLAN/Trunk เพิ่ม ไม่ผูกกับ Spanning-Tree และ Converge
   เร็วกว่า สอดคล้องกับหลักการออกแบบ "Routed Core" ของ Cisco Enterprise Campus

---

## สรุป Part 10

Part นี้ปิดกลุ่มพื้นฐาน CCNA ด้าน Switching/Layer 2-3 อย่างสมบูรณ์ — เราเรียนรู้ทำไมต้องมี
Inter-VLAN Routing, สอง Method หลัก (**Router-on-a-Stick** สำหรับ Branch เล็ก และ **SVI** สำหรับ
Enterprise Campus), เปิดใช้งาน `ip routing` และ Routed Port, แล้วนำ SVI ไป**คอนฟิกจริงบน
DIST-SW1–4 ทั้ง 4 ตัว** พร้อม Routed Port ที่ Core Layer จนสามารถ **ping ข้าม VLAN ได้ทั่วทั้ง
Enterprise Lab Topology** ✅

Static Route ที่ใช้เชื่อม Distribution↔Core ใน Step 100 แสดงให้เห็นชัดเจนว่ายิ่ง Topology โต
ขึ้น การเขียน Static Route ด้วยมือยิ่งซับซ้อนและไม่ยืดหยุ่น — **Part 11 เป็นต้นไปจะเข้าสู่ระดับ
Routing Protocol เต็มรูปแบบ** เริ่มจาก **OSPFv2 Fundamentals** ซึ่งจะมาแทนที่ Static Route
ทั้งหมดในหลักสูตรนี้ด้วย Dynamic Routing ที่เรียนรู้และปรับตัวเองอัตโนมัติ

**ไปต่อ:** [Part 11 — OSPFv2 Fundamentals →](part-011-ospfv2-fundamentals.md)
