# Part 18 — First Hop Redundancy Protocols (HSRP/VRRP/GLBP)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 171–180 จาก 1000**

> ต่อจาก [Part 17 — NTP, Syslog, SNMP](part-017-ntp-syslog-snmp.md) ที่สอนการทำ Time
> Synchronization และ Centralized Monitoring แล้ว Part นี้จะกลับมาแก้ปัญหาสำคัญที่ถูก
> "ค้างไว้" ตั้งแต่ [Part 10 — Inter-VLAN Routing](part-010-inter-vlan-routing.md): **Host
> ในแต่ละ VLAN ยังต้องตั้ง Default Gateway ไปที่ IP จริงของ DIST-SW ตัวใดตัวหนึ่งเพียงตัวเดียว
> (`.2` หรือ `.3`) โดยไม่มี Redundancy อัตโนมัติเลยถ้า Switch ตัวนั้นล่ม** — เราจะแก้ปัญหานี้ด้วย
> **First Hop Redundancy Protocol (FHRP)** ทั้ง 3 แบบที่ Cisco Exam Blueprint กำหนด คือ
> **HSRP, VRRP, GLBP** โดยเน้น HSRP เป็นหลักเพราะเป็น Protocol ที่ Lab หลักของหลักสูตรนี้จะใช้จริง
> ปิดท้ายด้วย Lab เต็มรูปแบบที่เติมค่า `.1` (HSRP Virtual IP) เข้าไปในทุก VLAN ตามที่
> [00-ip-address-plan.md](00-ip-address-plan.md) จองไว้ล่วงหน้าตั้งแต่ต้น

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 171 | ทำไมต้องมี FHRP — Single Point of Failure ที่ Default Gateway |
| 172 | HSRP พื้นฐาน — State, Timer, Virtual MAC |
| 173 | คอนฟิก HSRP พื้นฐาน — Load-Sharing ระหว่าง DIST-SW1/DIST-SW2 |
| 174 | HSRP Interface Tracking — Failover ตาม Uplink |
| 175 | HSRPv1 vs HSRPv2 |
| 176 | VRRP (RFC 3768) — มาตรฐานเปิด |
| 177 | GLBP — Active/Active Load Balancing แบบ Cisco |
| 178 | เลือก FHRP ให้ถูกกับงาน |
| 179 | Verification และ Troubleshooting FHRP |
| 180 | Lab เต็มรูปแบบ: HSRPv2 ทั้ง 5 VLAN บน DIST-SW1–4 |

---

## Step 171 — ทำไมต้องมี FHRP

ย้อนกลับไปดู [Step 96 ของ Part 10](part-010-inter-vlan-routing.md) เราคอนฟิก SVI บน DIST-SW1
เป็น `10.10.10.2` และ DIST-SW2 เป็น `10.10.10.3` สำหรับ VLAN 10 — ทั้งสอง Switch **ทำงานเป็น
Gateway อิสระจากกัน คนละ IP กันคนละหมายเลข** ปัญหาคือ **Host แต่ละเครื่องต้องเลือกตั้ง Default
Gateway ไปที่ IP ใด IP หนึ่งเพียง IP เดียวเท่านั้น** (PC จะมีค่า Default Gateway ได้ทีละค่า)

### ปัญหาที่เกิดขึ้นจริงถ้าไม่มี FHRP

```
        ก่อนมี FHRP (Part 10)
        PC1 (10.10.10.10) ตั้ง Default Gateway = 10.10.10.2  (DIST-SW1 เท่านั้น)

        ┌──────────┐         ┌──────────┐
        │ DIST-SW1 │         │ DIST-SW2 │
        │ .2  ACTIVE│        │ .3  ไม่ได้ใช้เลย │
        └────┬─────┘         └────┬─────┘
             │ PC1 ใช้เส้นนี้        │ ว่างเปล่า แม้ Link ปกติ
        ┌────┴──────────────────────┴────┐
        │           ACCESS-SW1            │
        └──────────────┬───────────────────┘
                       PC1

   ❌ DIST-SW1 reload / สาย uplink หลุด / Power Supply เสีย
      → PC1 ARP หา 10.10.10.2 ไม่มีใครตอบ
      → "Destination host unreachable" ทันที แม้ DIST-SW2 (.3) ยังทำงานปกติ 100%
        และยังมีเส้นทางไป Core ผ่าน DIST-SW2 อยู่เฉยๆ โดยไม่ได้ใช้ประโยชน์เลย
```

นี่คือ **Single Point of Failure ที่ระดับ Default Gateway** — แม้ทั้ง Distribution Layer จะ
ออกแบบให้ Redundant (มี DIST-SW1 และ DIST-SW2 คู่กัน ตาม 3-Tier Design ของ Part 1) แต่ถ้า Host
ผูกติดกับ IP จริงของ Switch เพียงตัวเดียว **ความ Redundant ของ Hardware ก็ไม่มีประโยชน์กับ Host
เลย** เพราะ Host ไม่รู้จักและไม่ Failover ไปใช้ Gateway ตัวสำรองเองอัตโนมัติ

### แนวคิดหลักของ FHRP — Virtual IP + Virtual MAC

**First Hop Redundancy Protocol (FHRP)** คือกลุ่ม Protocol ที่ทำให้ Router/Switch หลายตัว
**ทำงานร่วมกันเป็น Gateway เสมือนหนึ่งเดียว (Virtual Gateway)** โดย:

1. สร้าง **Virtual IP Address (VIP)** ที่ไม่ผูกกับ Interface จริงของอุปกรณ์ใดตัวหนึ่ง — Host
   ทุกเครื่องตั้ง Default Gateway เป็น VIP ตัวนี้เพียงค่าเดียว ตลอดไป ไม่ต้องเปลี่ยนอีก
2. สร้าง **Virtual MAC Address** คู่กับ VIP — เมื่อ Host ARP หา VIP จะได้ Virtual MAC นี้กลับมา
   ไม่ใช่ MAC จริงของ Switch ตัวใดตัวหนึ่ง
3. อุปกรณ์ที่เข้าร่วมกลุ่ม (เรียกว่า **FHRP Group**) จะเลือกกันเองว่าตัวไหนเป็น **ตัว Forward
   Traffic จริง** (Active/Master) ส่วนตัวอื่นๆ อยู่ในสถานะสำรอง (Standby/Backup) พร้อมรับหน้าที่
   ต่อทันทีถ้าตัว Active หลุดจาก Network
4. ทั้งหมดนี้เกิดขึ้น **โดย Host ไม่ต้องรับรู้เลย** — Host เห็นแค่ VIP+Virtual MAC เดียวตลอดเวลา
   แม้ Physical Switch ที่ตอบ ARP จะเปลี่ยนตัวไปมาเบื้องหลังก็ตาม

```
        หลังมี FHRP (Part 18 — Part นี้)
        PC1 ตั้ง Default Gateway = 10.10.10.1  (Virtual IP — ไม่ผูกกับ Switch ตัวใดตัวหนึ่ง)

        ┌──────────┐         ┌──────────┐
        │ DIST-SW1 │◄═══════►│ DIST-SW2 │   Hello ทุก 3 วิ (สถานะ Active/Standby)
        │ .2 ACTIVE │  VIP:.1│ .3 STANDBY│
        └────┬─────┘         └────┬─────┘
             │  PC1 forward ผ่านตัวนี้ตอนนี้    │  พร้อมรับหน้าที่ทันทีถ้า .2 ล่ม
        ┌────┴──────────────────────┴────┐
        │           ACCESS-SW1            │
        └──────────────┬───────────────────┘
                       PC1

   ✅ DIST-SW1 ล่ม → DIST-SW2 ตรวจจับได้ภายใน ~10 วินาที (Hold Timer) → รับ VIP .1 ต่อทันที
      → PC1 ไม่ต้องเปลี่ยน Default Gateway เลย ARP Cache เดิมยังใช้ Virtual MAC เดิมได้ต่อ
```

> **สรุป Step นี้**: FHRP ไม่ได้แก้ปัญหา Bandwidth หรือ Routing Table (นั่นคืองานของ Dynamic
> Routing Protocol อย่าง OSPF ใน Part 11) แต่แก้ปัญหาเฉพาะเจาะจงที่ **First Hop** — จุดแรกที่
> Traffic ของ Host ต้องออกจาก Subnet ของตัวเอง — ให้มี Redundancy โดย Host ไม่ต้องรู้ตัวเลย
> Cisco รองรับ FHRP 3 แบบหลักที่จะเรียนใน Part นี้: **HSRP** (Cisco Proprietary, ใช้ใน Lab นี้),
> **VRRP** (Open Standard), **GLBP** (Cisco Proprietary, Active/Active)

---

## Step 172 — HSRP พื้นฐาน: State, Timer, Virtual MAC

**HSRP (Hot Standby Router Protocol)** เป็น FHRP ตัวแรกที่ Cisco พัฒนาขึ้น (Proprietary — ใช้ได้
เฉพาะอุปกรณ์ Cisco) และเป็น FHRP ที่ใช้กันแพร่หลายที่สุดใน Enterprise Network ที่ใช้ Cisco
ทั้ง Topology เหมือนหลักสูตรนี้

### 172.1 State Machine ของ HSRP

อุปกรณ์ที่รัน HSRP ในกลุ่มเดียวกัน (Group) จะอยู่ในสถานะใดสถานะหนึ่งต่อไปนี้เสมอ:

| State | ความหมาย |
|---|---|
| **Initial** | เพิ่งเปิด HSRP ยังไม่พร้อมทำงาน (รอ Interface up) |
| **Learn** | ยังไม่รู้ Virtual IP (กรณีไม่ได้ config IP ไว้ตรงๆ — พบใน Config แบบ Learn จาก Active) |
| **Listen** | รู้ Virtual IP แล้ว แต่ไม่ใช่ Active หรือ Standby — คอยฟัง Hello จากตัวอื่น (ใน Group ที่มี > 2 สมาชิก) |
| **Speak** | ส่ง Hello ของตัวเอง และร่วม Election หา Active/Standby |
| **Standby** | ผู้สมัครลำดับถัดไปที่จะเป็น Active — คอยรับ Hello จาก Active ตลอดเวลา |
| **Active** | **ตัวที่ Forward Traffic จริงให้ VIP ตอนนี้** — ตอบ ARP Request ด้วย Virtual MAC และ route packet ที่ส่งมาที่ VIP |

> **สำหรับข้อสอบ CCNA/CCNP**: ในกลุ่มที่มีอุปกรณ์แค่ 2 ตัว (แบบ Lab นี้) จะเห็นแค่ **Active** และ
> **Standby** เท่านั้น — State **Listen** จะปรากฏเมื่อมีอุปกรณ์ตัวที่ 3 ขึ้นไปใน Group เดียวกัน
> (เช่น 3 Switch แข่งกันเป็น Gateway ของ VLAN เดียว) ซึ่งไม่ใช่ Design ของหลักสูตรนี้ (ใช้ 2 ตัว
> ต่อ VLAN เสมอ ตาม 3-Tier Design)

### 172.2 Timer — Hello และ Hold

```
Active ────Hello (ทุก 3 วินาที)────► Standby
Active ────Hello (ทุก 3 วินาที)────► Standby
Active ─────X─── (Active ล่ม ไม่ส่ง Hello อีก)

Standby: รอครบ Hold Time (Default 10 วินาที = Hello x ~3.33 รอบ) แล้วไม่ได้รับ Hello เลย
         → ประกาศตัวเองเป็น Active ใหม่ทันที → ตอบ ARP ด้วย Virtual MAC เดิม
```

| Timer | ค่า Default | ความหมาย |
|---|---|---|
| **Hello Timer** | 3 วินาที | ความถี่ที่ Active/Standby ส่ง Hello Multicast บอกสถานะตัวเองให้กลุ่มรู้ |
| **Hold Timer** | 10 วินาที | เวลาที่ Standby รอ Hello จาก Active ก่อนจะสรุปว่า Active หายไปแล้ว (ปกติ = Hello x 3 + เผื่อ) |

> **Downtime โดยประมาณเมื่อ Active ล่ม (ค่า Default)**: ระหว่าง 0–10 วินาที ก่อนที่ Standby จะ
> เข้ามาทำหน้าที่ Active แทน (เร็วกว่านี้ได้ถ้าปรับ Timer ให้ต่ำลง เช่น `standby 10 timers 1 4`
> จะลด Downtime ลงเหลือประมาณ 4 วินาที แต่แลกมาด้วย CPU/Bandwidth ที่ใช้ส่ง Hello ถี่ขึ้น — ไม่
> จำเป็นสำหรับ Lab นี้เพราะ Default ก็เพียงพอแล้ว)

### 172.3 Virtual MAC Address ของ HSRP

HSRP ไม่ได้ใช้ MAC Address จริงของ Interface ตัวใดตัวหนึ่งเป็น MAC ของ VIP แต่จะ**สร้าง Virtual
MAC ขึ้นมาใหม่** ตาม Format ที่คำนวณจากเลข Group:

```
        0000.0C9F.F0XX   (HSRPv2 — ใช้ในหลักสูตรนี้ทั้งหมด)
        └──┬──┘└┬┘└┬┘
           │    │   └── XX = เลข Group แปลงเป็น Hexadecimal (3 หลัก รวมกับ F ด้านหน้า)
           │    └────── OUI ส่วนที่ 2 เฉพาะของ HSRPv2 (07 ใน v1 / 9F ใน v2)
           └─────────── OUI ของ Cisco (0000.0C = Cisco-registered OUI)
```

| HSRP Group | เลข Hex ของ Group | Virtual MAC (HSRPv2) |
|---|---|---|
| 10 (VLAN10-SALES) | 00A | `0000.0C9F.F00A` |
| 20 (VLAN20-VOICE) | 014 | `0000.0C9F.F014` |
| 30 (VLAN30-SERVERS) | 01E | `0000.0C9F.F01E` |
| 40 (VLAN40-WIFI) | 028 | `0000.0C9F.F028` |
| 99 (VLAN99-MGMT) | 063 | `0000.0C9F.F063` |

> **สำคัญ**: MAC นี้เกิดขึ้น**อัตโนมัติ**ตามเลข Group ที่คอนฟิก — ไม่ต้องกำหนดเองด้วยมือ (แม้จะทำได้
> ด้วยคำสั่ง `standby <group> mac-address <mac>` ในบางกรณีพิเศษ เช่น Migration จาก Router ตัวเก่า
> ที่ Host จำ MAC เดิมไว้ใน ARP Cache แต่ไม่ใช่ค่า Default และไม่ใช้ใน Lab นี้)

---

## Step 173 — คอนฟิก HSRP พื้นฐาน: Load-Sharing ระหว่าง DIST-SW1/DIST-SW2

### 173.1 Syntax คำสั่งหลัก 3 ตัว

| คำสั่ง | ทำที่ Mode | ความหมาย |
|---|---|---|
| `standby <group> ip <vip>` | `interface vlan X` | สร้าง HSRP Group และกำหนด Virtual IP |
| `standby <group> priority <0-255>` | `interface vlan X` | กำหนดความสำคัญ — **ค่าสูงกว่าชนะ Election เป็น Active** (Default = 100) |
| `standby <group> preempt` | `interface vlan X` | อนุญาตให้อุปกรณ์ที่ Priority สูงกว่า **ยึด Active คืน** ทันทีที่มันกลับมาออนไลน์ (Default = **ปิด**!) |

> **กฎเหล็กที่มักถูกลืม**: ถ้าไม่ใส่ `preempt` แม้จะตั้ง `priority` สูงกว่าคู่ก็ตาม — เมื่ออุปกรณ์นั้น
> Reload กลับมา มันจะ**ไม่ยึด Active คืนอัตโนมัติ** จะรอเป็น Standby ตลอดไปจนกว่า Active ตัว
> ปัจจุบันจะล่มเองอีกครั้ง — เป็นสาเหตุ Bug ที่พบบ่อยที่สุดของ HSRP ในสนามจริง (ดูเพิ่มใน Step 179)

### 173.2 เป้าหมายการออกแบบ: Load-Sharing ระหว่าง DIST-SW1 กับ DIST-SW2

Lab นี้จะไม่ให้ DIST-SW1 เป็น Active ทุก VLAN (จะทำให้ Traffic ทั้งหมดไปกระจุกที่ตัวเดียว เสีย
ประโยชน์ของการมี Distribution 2 ตัว) แต่จะ**สลับบทบาท Active ตาม VLAN** เพื่อกระจาย Load:

```
                     VLAN 10 (SALES)              VLAN 20 (VOICE)
                     VIP: 10.10.10.1               VIP: 10.10.20.1

   ┌──────────────┐                          ┌──────────────┐
   │  DIST-SW1     │   Priority 150           │  DIST-SW1     │   Priority 100 (default)
   │  .2  = ACTIVE │ ◄── สำหรับ VLAN 10        │  .2  = STANDBY│ ◄── สำหรับ VLAN 20
   └──────────────┘                          └──────────────┘
   ┌──────────────┐                          ┌──────────────┐
   │  DIST-SW2     │   Priority 100 (default) │  DIST-SW2     │   Priority 150
   │  .3  = STANDBY│ ◄── สำหรับ VLAN 10        │  .3  = ACTIVE │ ◄── สำหรับ VLAN 20
   └──────────────┘                          └──────────────┘

   ผลลัพธ์: Traffic VLAN 10 ทั้งหมดวิ่งผ่าน DIST-SW1, Traffic VLAN 20 ทั้งหมดวิ่งผ่าน DIST-SW2
            ทั้งสอง Uplink ไป Core ถูกใช้งานจริงพร้อมกัน ไม่มีตัวใดว่างเปล่า
```

### 173.3 คอนฟิกจริงบน DIST-SW1 — Active สำหรับ VLAN 10, Standby สำหรับ VLAN 20

> ต่อยอดจาก Config เดิมของ [Step 96, Part 10](part-010-inter-vlan-routing.md) — เพิ่มเฉพาะคำสั่ง
> HSRP เข้าไปใน Interface ที่มีอยู่แล้ว ไม่ต้องสร้าง SVI ใหม่

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby version 2
DIST-SW1(config-if)# standby 10 ip 10.10.10.1
DIST-SW1(config-if)# standby 10 priority 150
DIST-SW1(config-if)# standby 10 preempt
DIST-SW1(config-if)# exit

DIST-SW1(config)# interface vlan 20
DIST-SW1(config-if)# standby version 2
DIST-SW1(config-if)# standby 20 ip 10.10.20.1
DIST-SW1(config-if)# standby 20 priority 100
DIST-SW1(config-if)# standby 20 preempt
DIST-SW1(config-if)# exit
```

### 173.4 คอนฟิกจริงบน DIST-SW2 — Standby สำหรับ VLAN 10, Active สำหรับ VLAN 20

```
DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# standby version 2
DIST-SW2(config-if)# standby 10 ip 10.10.10.1
DIST-SW2(config-if)# standby 10 priority 100
DIST-SW2(config-if)# standby 10 preempt
DIST-SW2(config-if)# exit

DIST-SW2(config)# interface vlan 20
DIST-SW2(config-if)# standby version 2
DIST-SW2(config-if)# standby 20 ip 10.10.20.1
DIST-SW2(config-if)# standby 20 priority 150
DIST-SW2(config-if)# standby 20 preempt
DIST-SW2(config-if)# exit
```

> **สังเกต**: `standby <group> ip <vip>` ต้องคอนฟิก**เหมือนกันทุกตัว**ในกลุ่มเดียวกัน (VIP เดียวกัน,
> เลข Group เดียวกัน) — สิ่งที่ต่างกันคือ **Priority** เท่านั้นที่กำหนดว่าใครเป็น Active

### 173.5 Verify ด้วย `show standby brief`

```
DIST-SW1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   150 P Active  local           10.10.10.3      10.10.10.1
Vl20        20   100 P Standby 10.10.20.3       local           10.10.20.1

DIST-SW2# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   100 P Standby 10.10.10.2       local           10.10.10.1
Vl20        20   150 P Active  local           10.10.20.2      10.10.20.1
```

ผลลัพธ์ตรงตามที่ออกแบบไว้ทุกประการ: DIST-SW1 เป็น **Active** ของ VLAN 10 และ **Standby** ของ
VLAN 20 ส่วน DIST-SW2 กลับกัน — Load ถูกกระจายระหว่าง 2 Switch เรียบร้อย

---

## Step 174 — HSRP Interface Tracking: Failover ตาม Uplink

### 174.1 ปัญหาที่ Priority อย่างเดียวแก้ไม่ได้

สมมติ DIST-SW1 เป็น Active ของ VLAN 10 (Priority 150) ปกติดี **แต่สาย Uplink ไปยัง CORE-SW1
(TenGigabitEthernet1/1/1) หลุดหรือ CORE-SW1 Port นั้นเสีย** — ตัว SVI ของ VLAN 10 (`interface
vlan 10`) เองยังคง **up/up** อยู่ (เพราะ VLAN 10 ยัง Active อยู่ฝั่ง Access Layer ปกติ) HSRP จึง
**ไม่รู้ตัวว่ามีปัญหาเลย** และยังคงเป็น Active ต่อไป — ผลคือ:

```
   PC1 → ARP หา VIP .1 → ตอบโดย DIST-SW1 (Virtual MAC) → ส่ง Packet เข้า DIST-SW1
                                                              │
                                                         ❌ Uplink ไป Core ขาด!
                                                         Packet ค้าง ไม่ไปไหนต่อ
                                                         (แม้ DIST-SW2 มี Uplink ปกติ
                                                          และพร้อมเป็น Active สมบูรณ์)
```

นี่คือ **"Black-Hole"** — HSRP เลือก Active ผิดตัวเพราะดูแค่ Interface ของ VLAN ฝั่ง Access
โดยไม่รู้ว่า "เส้นทางไป Core" (ซึ่งสำคัญกว่าในการทำหน้าที่ Gateway จริง) ขาดไปแล้ว

### 174.2 วิธีแก้: `standby <group> track <interface> decrement <value>`

คำสั่งนี้บอกให้ HSRP **จับตาดู Interface อื่น** (ไม่ใช่ Interface ที่ HSRP รันอยู่) — ถ้า
Interface ที่ Track ไว้ Down ให้**ลด Priority ของ HSRP Group นี้ลงทันที** ตามค่าที่กำหนด ถ้าลด
แล้ว Priority ต่ำกว่าคู่แข่ง → เกิด Election ใหม่ → คู่แข่งกลายเป็น Active แทน

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# exit
```

**คำนวณ**: DIST-SW1 Priority ปกติ = 150 → ถ้า Track เห็น Te1/1/1 Down → ลด 60 → เหลือ **90**
ซึ่ง**น้อยกว่า** Priority ของ DIST-SW2 (100) ทันที → DIST-SW2 ชนะ Election กลายเป็น Active แทน
โดยอัตโนมัติ แม้ตัว SVI VLAN 10 ของ DIST-SW1 เองจะยัง up/up อยู่ตลอดก็ตาม

### 174.3 คอนฟิกให้ครบทั้ง 2 ทิศทาง (Track ทั้งคู่ ป้องกันปัญหาเดียวกันสลับข้าง)

```
! DIST-SW1 — VLAN 10 (ตัว Active) และ VLAN 20 (ตัว Standby) ก็ควร track เช่นกัน
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# exit
DIST-SW1(config)# interface vlan 20
DIST-SW1(config-if)# standby 20 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# exit

! DIST-SW2 — เช่นเดียวกัน track uplink ของตัวเอง
DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# exit
DIST-SW2(config)# interface vlan 20
DIST-SW2(config-if)# standby 20 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# exit
```

> **ทำไม Decrement = 60 ไม่ใช่เลขอื่น?** กฎการเลือกค่าคือ **Priority (Active) − Decrement ต้อง
> น้อยกว่า Priority ของ Standby เสมอ** ในที่นี้ 150 − 60 = 90 < 100 (Standby) ✅ — ถ้าใช้ค่าน้อย
> เกินไป เช่น Decrement 10 → 150−10=140 ยังมากกว่า 100 อยู่ → **HSRP จะไม่ Failover แม้ Uplink
> ขาดไปแล้วก็ตาม** เป็นความผิดพลาด Config ที่พบบ่อยมาก ต้องคำนวณให้ตกลงไปต่ำกว่าคู่แข่งเสมอ

### 174.4 Verify — ก่อนและหลังจำลอง Uplink ล่ม

```
! ก่อน Uplink ล่ม — DIST-SW1 เป็น Active ปกติ พร้อม track uplink อยู่
DIST-SW1# show standby vlan 10
Vlan10 - Group 10 (version 2)
  State is Active
    2 state changes, last state change 00:15:22
  Track object 1 (interface Te1/1/1)
    state Up
  Virtual IP address is 10.10.10.1
  Active virtual MAC address is 0000.0c9f.f00a
  Local virtual MAC address is 0000.0c9f.f00a (v2 default)
  Hello time 3 sec, hold time 10 sec
  Preemption enabled
  Active router is local
  Standby router is 10.10.10.3, priority 100 (expires in 8.912 sec)
  Priority 150 (configured 150)
  Group name is "hsrp-Vl10-10" (default)

! จำลองปัญหาจริง: shutdown สาย uplink ของ DIST-SW1 ไปหา CORE-SW1
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# shutdown

! HSRP เห็นทันทีว่า track interface down → ลด priority → Standby ยึด Active ทันที (ภายในไม่กี่วินาที)
DIST-SW1#
%TRACK-6-STATE: 1 interface Te1/1/1 line-protocol Down -> Up
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Active -> Speak
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Speak -> Standby

DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   90  P Standby  10.10.10.3      local           10.10.10.1
                  └── ลดจาก 150 เหลือ 90 ตาม decrement 60

! ที่ฝั่ง DIST-SW2 — กลายเป็น Active ของ VLAN 10 แทนโดยอัตโนมัติ
DIST-SW2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Active   local           10.10.10.2      10.10.10.1
```

Traffic ของ VLAN 10 ทั้งหมด**สลับไปวิ่งผ่าน DIST-SW2 อัตโนมัติ**ภายในไม่กี่วินาที โดย PC1 ไม่ต้อง
เปลี่ยน Default Gateway หรือ ARP Cache เลย — นี่คือประโยชน์ที่แท้จริงของการ track Uplink Interface
ร่วมกับ HSRP ไม่ใช่แค่การมี Standby Switch เฉยๆ

> เมื่อ `no shutdown` คืน Te1/1/1 ของ DIST-SW1 → Priority กลับเป็น 150 ทันที และเพราะมี `preempt`
> เปิดอยู่ → DIST-SW1 จะ**ยึด Active คืนอัตโนมัติ** กลับสู่สภาวะ Load-Sharing เดิม (รายละเอียดการ
> ทดสอบเต็มรูปแบบอยู่ใน Step 180)

---

## Step 175 — HSRPv1 vs HSRPv2

Cisco ปรับปรุง HSRP จาก v1 เป็น v2 เพื่อแก้ข้อจำกัดเรื่องจำนวน Group และลดโอกาสชนกับ Protocol
อื่นที่ใช้ Multicast Address ใกล้เคียงกัน

| หัวข้อ | HSRPv1 | HSRPv2 |
|---|---|---|
| Multicast Address (Hello) | `224.0.0.2` | `224.0.0.102` |
| UDP Port | 1985 | 1985 |
| Group Number Range | 0–255 | **0–4095** |
| Virtual MAC Format | `0000.0C07.ACxx` (xx = group hex 2 หลัก) | `0000.0C9F.Fxxx` (xxx = group hex 3 หลัก) |
| รองรับ IPv6 | ❌ ไม่รองรับ | ✅ รองรับ (Virtual MAC ต่างออกไปอีกชุดสำหรับ IPv6) |
| ความเข้ากันได้ | Legacy Router รุ่นเก่า | Cisco แนะนำเป็น **Default ตั้งแต่ IOS สมัยใหม่** |
| ปัญหาที่ v1 มี | `224.0.0.2` เป็น Address เดียวกับที่ RIPv1, VRRP รุ่นเก่า, และ Protocol อื่นๆ ใช้ — เสี่ยงชนกันในบาง Design | ใช้ Address เฉพาะของตัวเอง ไม่ชนใคร |

### ทำไมต้องใส่ `standby version 2` เสมอในหลักสูตรนี้

แม้ IOS-XE รุ่นใหม่ๆ บาง Platform (เช่น Catalyst 9000 Series ที่ใช้ใน Lab นี้) จะ**เริ่มต้นเป็น
version 2 อยู่แล้ว** แต่ควรใส่คำสั่งนี้ให้เห็นชัดเจนในทุก Script Config เสมอ ด้วยเหตุผล 2 ข้อ:

1. **ความชัดเจนในการ Migrate ระหว่าง Platform** — ถ้าย้ายไป Router รุ่นเก่าที่ Default เป็น v1
   จะได้ Group Number ตรงกับที่ตั้งใจ (โดยเฉพาะ Group 99 ที่เกิน Range ของ v1 อยู่แล้ว — ดูข้อ 2)
2. **Group Number ที่ใช้ในหลักสูตรนี้เกิน 255 ไม่ได้ก็จริง แต่ Group 99 ยังพอดีกับทั้งคู่** — แต่ถ้า
   ในอนาคตขยาย Lab ไปถึง VLAN ที่มีเลขเกิน 255 (เช่น VLAN 300 ในบาง Design จริง) **HSRPv1 จะใช้
   งานไม่ได้เลยทันที** เพราะ Group Number ผูกกับ VLAN ID ตามธรรมเนียมของหลักสูตรนี้

> **กฎที่ใช้ตลอด Lab ของหลักสูตรนี้**: ใส่ `standby version 2` เป็นคำสั่งแรกเสมอก่อนคำสั่ง `standby`
> อื่นๆ ทุกครั้งที่คอนฟิก HSRP บน Interface ใดๆ

---

## Step 176 — VRRP (Virtual Router Redundancy Protocol, RFC 3768)

**VRRP** คือ FHRP แบบ **Open Standard** (กำหนดโดย IETF ใน RFC 3768, ปรับปรุงเป็น RFC 5798 สำหรับ
VRRPv3) ออกแบบมาให้ทำงานได้กับอุปกรณ์**ต่างยี่ห้อกัน** ไม่ผูกกับ Cisco เพียงอย่างเดียวแบบ HSRP

### 176.1 คำศัพท์ที่ต่างจาก HSRP

| แนวคิด | HSRP | VRRP |
|---|---|---|
| ตัวที่ Forward Traffic จริง | **Active** | **Master** |
| ตัวสำรอง | **Standby** | **Backup** |
| ตัวระบุกลุ่ม | Group Number | **VRID (Virtual Router ID)** |
| Priority Election | ค่าสูงสุดชนะ (Default 100) | ค่าสูงสุดชนะ (Default 100, Range 1–254) |

### 176.2 คอนฟิก VRRP (ตัวอย่างสาธิต ไม่ใช้จริงใน Lab หลัก)

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# vrrp 10 ip 10.10.10.1
DIST-SW1(config-if)# vrrp 10 priority 150
DIST-SW1(config-if)# vrrp 10 preempt
```

Syntax หน้าตาคล้าย HSRP มาก (`vrrp` แทน `standby`) แต่มีความต่างสำคัญที่ต้องจำ:

### 176.3 ตารางเปรียบเทียบ VRRP vs HSRP โดยละเอียด

| หัวข้อ | HSRP | VRRP |
|---|---|---|
| เจ้าของ Standard | Cisco Proprietary | **IETF Open Standard (RFC 3768/5798)** |
| ใช้ข้าม Vendor ได้ไหม | ❌ Cisco เท่านั้น | ✅ ได้ (Juniper, Huawei, MikroTik ฯลฯ รองรับ) |
| Hello Timer (Default) | 3 วินาที | 1 วินาที |
| Hold/Dead Timer (Default) | 10 วินาที | 3 วินาที (เร็วกว่า HSRP) |
| Multicast Address | 224.0.0.102 (v2) | 224.0.0.18 |
| Preempt (Default) | **ปิด** | **เปิด** (ต่างจาก HSRP!) |
| ใช้ Real Interface IP เป็น VIP ได้ไหม | ❌ ต้องเป็น IP ที่ไม่ผูกกับ Interface จริงเสมอ | ✅ **ได้** — ถ้า Priority ของอุปกรณ์ที่ตั้ง IP จริงตรงกับ VIP จะถูกบังคับเป็น Priority 255 เสมอ (IP Owner) |
| จำนวน Active พร้อมกันต่อ Group | 1 (Active เดียว) | 1 (Master เดียว) |
| รองรับ Load-Balancing แบบ Active/Active ในตัวไหม | ❌ ไม่ได้ (ต้องใช้หลาย Group สลับ Priority เอง แบบ Step 173) | ❌ ไม่ได้เช่นกัน (เหมือน HSRP) |

> **จุดสังเกตสำคัญสำหรับข้อสอบ**: VRRP **Preempt เปิดเป็น Default** (ตรงข้ามกับ HSRP ที่ปิดเป็น
> Default) และ VRRP อนุญาตให้ใช้ **IP จริงของ Interface เป็น Virtual IP ได้เลย** (เรียกว่า
> "IP Address Owner") ซึ่ง HSRP ทำไม่ได้ — เป็นความต่างที่ข้อสอบ CCNP ENCOR ชอบถามเปรียบเทียบ

### 176.4 ทำไมหลักสูตรนี้ไม่ใช้ VRRP เป็นหลัก

Lab ทั้ง Topology ของหลักสูตรนี้เป็น **Cisco ทั้งหมด** (Catalyst 9300/9500 ตาม
[00-ip-address-plan.md](00-ip-address-plan.md)) จึงไม่มีข้อจำกัดเรื่อง Multi-Vendor ที่ VRRP ถูก
ออกแบบมาแก้ — ในทางกลับกัน HSRP ให้ Feature เสริมที่ Cisco พัฒนาเพิ่มให้ (เช่น การผูกกับ Cisco
IOS Feature อื่นๆ ได้แน่นกว่า) ดังนั้น Lab หลักจะใช้ **HSRP** ต่อไป — รายละเอียดเหตุผลแบบเต็มอยู่ใน
Step 178

---

## Step 177 — GLBP: Active/Active Load Balancing แบบ Cisco

**GLBP (Gateway Load Balancing Protocol)** เป็น FHRP อีกตัวที่ Cisco พัฒนาเพิ่มขึ้นมา (Proprietary
เช่นเดียวกับ HSRP) เพื่อแก้ข้อจำกัดที่ HSRP/VRRP มีเหมือนกันทั้งคู่คือ **มี Router ทำหน้าที่ Forward
Traffic จริงได้แค่ตัวเดียวต่อ Group เท่านั้น** (ตัวอื่นอยู่เฉยๆ รอสำรอง) — GLBP ทำให้**หลายตัว
Forward Traffic พร้อมกันได้จริงโดยใช้ VIP เดียวกัน**

### 177.1 บทบาทใหม่ที่ GLBP เพิ่มมา: AVG และ AVF

| บทบาท | ชื่อเต็ม | หน้าที่ |
|---|---|---|
| **AVG** (Active Virtual Gateway) | ตัวที่ชนะ Election ด้วย Priority สูงสุด — มีแค่ 1 ตัวต่อ Group | ตอบ ARP Request ของ Host **ทุกตัว** แต่จะแจก **Virtual MAC ที่ต่างกัน** ให้ Host แต่ละกลุ่ม (ไม่ใช่ MAC เดียวเหมือน HSRP) |
| **AVF** (Active Virtual Forwarder) | อุปกรณ์ทุกตัวใน Group (รวม AVG เองด้วย) — **ได้ Virtual MAC ของตัวเอง 1 ชุด** | Forward Traffic จริงสำหรับ Host ที่ถูก AVG มอบหมาย MAC ของตนให้ |

### 177.2 หลักการ Load Balancing ของ GLBP — เกิดที่ระดับ ARP

```
                              VIP เดียว: 10.10.10.1  (เหมือน HSRP/VRRP)
                                        │
                     ┌──────────────────┴──────────────────┐
                     │              AVG (เช่น DIST-SW1)       │
                     │   แจก Virtual MAC ต่างกันให้ ARP Reply │
                     └───┬───────────────────────────┬───────┘
              PC1 ARP หา .1 → ได้ MAC-A          PC2 ARP หา .1 → ได้ MAC-B
                     │                                     │
              ┌──────┴──────┐                       ┌──────┴──────┐
              │  DIST-SW1    │ AVF#1 (MAC-A)         │  DIST-SW2    │ AVF#2 (MAC-B)
              │ Forward PC1  │◄── Traffic จาก PC1     │ Forward PC2  │◄── Traffic จาก PC2
              └─────────────┘                        └─────────────┘

   ผลลัพธ์: PC1 และ PC2 ใช้ VIP เดียวกัน (.1) เหมือนกันทุกประการ แต่ Traffic ถูกกระจายไปคนละ
            Physical Switch จริง โดย Host ไม่ต้องตั้งค่าอะไรต่างกันเลย — เกิดจาก ARP Reply
            ที่ AVG แจก MAC สลับกันไปตาม Algorithm ที่เลือก (ไม่ใช่การสลับ Priority แบบ Step 173)
```

### 177.3 Load-Balancing Algorithm ของ GLBP (เลือกได้ 3 แบบ)

| Algorithm | คำสั่ง | วิธีทำงาน |
|---|---|---|
| **Round-Robin** (Default) | `glbp <group> load-balancing round-robin` | แจก MAC สลับกันไปเรื่อยๆ ตามลำดับ Host ที่ ARP เข้ามา |
| **Weighted** | `glbp <group> load-balancing weighted` | แจกตามสัดส่วนที่ตั้งไว้ (Router ที่แรงกว่าตั้ง Weight สูงกว่า ได้ Host มากกว่า) |
| **Host-Dependent** | `glbp <group> load-balancing host-dependent` | Host เดิมได้ MAC เดิมเสมอ (สำคัญเมื่อ Application ผูกกับ Path เดิมตลอด เช่น NAT/Firewall Session-based) |

### 177.4 คอนฟิกตัวอย่าง (สาธิต Concept — ไม่ใช้จริงใน Lab หลักของหลักสูตรนี้)

```
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# glbp 10 ip 10.10.10.1
DIST-SW1(config-if)# glbp 10 priority 150
DIST-SW1(config-if)# glbp 10 preempt
DIST-SW1(config-if)# glbp 10 load-balancing round-robin

DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# glbp 10 ip 10.10.10.1
DIST-SW2(config-if)# glbp 10 priority 100
DIST-SW2(config-if)# glbp 10 preempt
```

### 177.5 ตรวจสอบ (ตัวอย่างผลลัพธ์)

```
DIST-SW1# show glbp brief
Interface   Grp  Fwd Pri State    Address         Active router   Standby router
Vl10        10   -   150 Active   10.10.10.1       local            10.10.10.3
Vl10        10   1   150 Active   0007.b400.0a01   local            -
Vl10        10   2   100 Listen   0007.b400.0a02   10.10.10.3       -
```

สังเกตว่า GLBP มี **2 Forwarder (Fwd 1 และ Fwd 2)** พร้อมกันภายใต้ VIP เดียว — นี่คือความต่างหลัก
จาก HSRP/VRRP ที่มี Active ได้แค่ 1 ตัวต่อ VIP เท่านั้น (ต้องใช้หลาย Group แยกกันแบบ Step 173
ถ้าจะ Load-Share ด้วย HSRP)

> **ข้อจำกัดของ GLBP ที่ต้องรู้**: การ Load Balance เกิดที่ **ระดับ ARP/MAC เท่านั้น** ถ้า Host
> Cache ARP ไว้แล้ว (ปกติ ARP Cache อยู่ได้หลายนาทีถึงชั่วโมง) จะใช้ AVF ตัวเดิมตลอดจนกว่า ARP
> Cache หมดอายุ — ไม่ใช่การ Load Balance แบบ Real-time ต่อ Packet เหมือน ECMP ของ Routing Protocol

---

## Step 178 — เลือก FHRP ให้ถูกกับงาน

### 178.1 ตารางตัดสินใจ (Decision Table)

| สถานการณ์ | FHRP ที่เหมาะสม | เหตุผล |
|---|---|---|
| Network เป็น Cisco ทั้งหมด, ต้องการ Control สูง, ทีมคุ้นเคย Cisco Feature | **HSRP** | Native บน Cisco IOS/IOS-XE ทุกรุ่น, เข้ากับ Feature อื่นของ Cisco ได้แน่นที่สุด, เสถียรและมีมานานที่สุด |
| Network มีหลาย Vendor ผสมกัน (เช่น Core เป็น Cisco แต่ Firewall/Load Balancer เป็นยี่ห้ออื่นที่ต้องเข้าร่วม FHRP Group เดียวกัน) | **VRRP** | เป็น Open Standard ตาม RFC — อุปกรณ์ทุกยี่ห้อที่รองรับ RFC 3768/5798 เข้ากลุ่มเดียวกันได้ |
| ต้องการใช้ Bandwidth ของ Uplink **ทุกเส้นพร้อมกันจริง** โดยไม่ต้องแยก VLAN/Group เพื่อสลับ Active เอง | **GLBP** | Active/Active ในตัวโดยไม่ต้องคอนฟิกหลาย Group ต่อ VLAน — Forwarder หลายตัว Active พร้อมกันได้จาก VIP เดียว |
| ต้องการใช้ IP จริงของ Router เป็น Virtual IP เลย (ไม่อยากมี VIP แยก) | **VRRP** (รองรับ IP Address Owner) | HSRP/GLBP ไม่รองรับแนวคิดนี้ — VIP ต้องเป็นคนละ IP จาก Interface จริงเสมอ |

### 178.2 ทำไม Lab หลักของหลักสูตรนี้เลือก HSRP

1. **Topology ทั้งหมดเป็น Cisco 100%** (Catalyst 9300/9500 ตาม
   [00-ip-address-plan.md](00-ip-address-plan.md)) — ไม่มีความจำเป็นต้องใช้ Open Standard เพื่อ
   ความเข้ากันได้ข้าม Vendor เลย
2. **HSRP คือ FHRP ที่ข้อสอบ CCNA/CCNP ENCOR ให้น้ำหนักมากที่สุด** เพราะเป็น Protocol ที่ Cisco
   สอนและใช้อ้างอิงในเอกสารทางการเป็นหลัก การเรียนรู้ HSRP อย่างลึกให้ประโยชน์สูงสุดต่อ Certification
3. **Design แบบ Load-Sharing ด้วยหลาย Group สลับ Priority (Step 173)** ก็ให้ผลลัพธ์การกระจาย
   Traffic ที่เพียงพอสำหรับ Enterprise Campus ทั่วไปอยู่แล้ว โดยไม่ต้องเพิ่มความซับซ้อนของ GLBP
   (AVG/AVF Election ซับซ้อนกว่า HSRP Election ธรรมดามาก และ Troubleshoot ยากกว่า)
4. **ความเรียบง่ายของ Config และ Verification** — `show standby brief` อ่านง่ายกว่า
   `show glbp brief` มากสำหรับทีมงานที่ไม่ได้เชี่ยวชาญ GLBP มาก่อน

> **กฎสรุปสำหรับสนามจริงและข้อสอบ**: ถ้าโจทย์ไม่ได้พูดถึง Multi-Vendor หรือความต้องการ
> Active/Active แบบเจาะจง → คำตอบ Default ที่ปลอดภัยที่สุดคือ **HSRP** เสมอ (โดยเฉพาะใน
> Cisco-only Environment) — Lab นี้จึงใช้ HSRPv2 ทั้ง 5 VLAN ใน Step 180

---

## Step 179 — Verification และ Troubleshooting FHRP

### 179.1 คำสั่ง Verify หลักของแต่ละ Protocol

| Protocol | คำสั่งสรุป (Brief) | คำสั่งละเอียด |
|---|---|---|
| HSRP | `show standby brief` | `show standby [interface] [group]` |
| VRRP | `show vrrp brief` | `show vrrp [interface]` |
| GLBP | `show glbp brief` | `show glbp [interface] [group]` |

```
! HSRP — ภาพรวมทุก Group บนเครื่อง (ใช้บ่อยที่สุดในการตรวจสุขภาพ FHRP ทั้งเครื่อง)
DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           10.10.10.3      10.10.10.1
Vl20        20   100 P Standby  10.10.20.3       local           10.10.20.1
Vl99        99   150 P Active   local           10.10.99.3      10.10.99.1

! HSRP — ละเอียดเฉพาะ Group เดียว (ใช้ debug เจาะปัญหา)
DIST-SW1# show standby vlan 10
Vlan10 - Group 10 (version 2)
  State is Active
  Virtual IP address is 10.10.10.1
  Active virtual MAC address is 0000.0c9f.f00a
  Hello time 3 sec, hold time 10 sec
  Preemption enabled
  Priority 150 (configured 150)
  Track object 1 (interface Te1/1/1), state Up, decrement 60
  Standby router is 10.10.10.3, priority 100 (expires in 8.500 sec)
```

### 179.2 ปัญหาที่พบบ่อยที่สุด 3 อันดับ

| ปัญหา | อาการที่เห็น | สาเหตุ | วิธีแก้ |
|---|---|---|---|
| **1. Priority ผิด ทำให้ Active อยู่ผิดตัว** | Traffic ทุก VLAN กระจุกที่ Switch ตัวเดียว ไม่กระจาย Load ตามที่ตั้งใจ | ลืมตั้ง Priority หรือใส่สลับกันระหว่างสอง Switch (Copy-paste Config ผิด) | `show standby brief` เทียบ Priority ทั้งสองฝั่งว่าตรงตาม Design หรือไม่ แล้วแก้ด้วย `standby <group> priority <value>` |
| **2. ไม่ได้ตั้ง `preempt` ทำให้ Failback ไม่เกิดอัตโนมัติ** | Active กลับมาออนไลน์แล้ว (Priority สูงกว่าคู่แข่งแล้ว) แต่ยังค้างเป็น Standby ตลอดไป ไม่ยึด Active คืน | `standby <group> preempt` ไม่ได้ config ไว้ (Default HSRP = ปิด) | เพิ่ม `standby <group> preempt` แล้วรอ (หรือ `clear standby group <name>` เพื่อ Force Re-election ทันที) |
| **3. VIP ชนกับ Host IP จริงที่มีอยู่แล้ว** | Host บางเครื่อง Ping VIP ไม่ได้ หรือ ARP ได้ MAC ผิดสลับกันไปมา (Duplicate IP Address บน Network) | ตั้ง `standby <group> ip` เป็น IP ที่มี Host ใช้งานอยู่แล้วโดยไม่ได้ตรวจสอบก่อน | ตรวจสอบด้วย `show ip arp` / `ping` ก่อน config ทุกครั้งว่า VIP ที่จะใช้ยังไม่มีใครถือครองอยู่ — ในหลักสูตรนี้ `.1` ถูกจองไว้เป็น VIP ตั้งแต่ต้นแล้วตาม [00-ip-address-plan.md](00-ip-address-plan.md) จึงไม่มีปัญหานี้ |

### 179.3 ลำดับ Troubleshooting มาตรฐานสำหรับ FHRP

```
1. show standby brief (หรือ vrrp/glbp brief)  → ดูภาพรวมทุก Group ก่อนว่า State ตรงตาม Design ไหม
2. show standby <interface>                    → ดู Priority, Preempt, Track Object ละเอียด
3. show track <object-number>                  → ถ้าใช้ Interface Tracking ตรวจว่า Track Object
                                                   เห็นสถานะ Interface ที่ Track ถูกต้องหรือไม่
4. show ip interface brief                     → ยืนยันว่า SVI ของ VLAN นั้น up/up จริง (ไม่งั้น
                                                   HSRP จะไม่ทำงานเลยตั้งแต่ต้น — ทวน Part 10)
5. debug standby [terse]                       → ดู State Transition แบบ Real-time เมื่อ Failover
                                                   เกิดขึ้น (ใช้ระมัดระวังใน Production เพราะมี CPU
                                                   Load เพิ่มขึ้นเมื่อ Debug เปิดอยู่)
```

> **เช็คลิสต์ก่อนเริ่ม Deploy HSRP ทุกครั้ง**: (1) `standby version 2` ใส่ก่อนคำสั่งอื่นเสมอ,
> (2) เลข Group ตรงกันทั้งสองฝั่ง, (3) VIP ตรงกันทั้งสองฝั่ง, (4) Priority ตั้งใจให้ต่างกันชัดเจน
> (ไม่ควรเท่ากันเว้นแต่ตั้งใจ Load-Balance แบบสุ่ม), (5) `preempt` เปิดทั้งสองฝั่งเสมอเพื่อให้ Failback
> เกิดอัตโนมัติ, (6) Track Object กับ Decrement คำนวณให้ Priority ตกลงต่ำกว่าคู่แข่งจริง

---

## Step 180 — Lab เต็มรูปแบบ: HSRPv2 ทั้ง 5 VLAN บน DIST-SW1–4

นี่คือ Lab สรุปของ Part 18 — เติมค่า HSRP Virtual IP (`.1`) ให้ครบทุก VLAN ที่
[00-ip-address-plan.md](00-ip-address-plan.md) จองไว้ตั้งแต่ต้น พร้อม Interface Tracking และ
Load-Sharing เต็มรูปแบบ ทั้ง 2 บล็อกของ Distribution Layer

### 180.1 แผนผัง Active/Standby เต็มรูปแบบทั้ง Topology

```
   บล็อก 1: DIST-SW1 / DIST-SW2  (รับผิดชอบ VLAN 10, 20, 99)     uplink → CORE-SW1

   ┌────────────────────────────┐        ┌────────────────────────────┐
   │        DIST-SW1             │        │        DIST-SW2             │
   │  Vlan10 .2   HSRP10: ACTIVE  │◄──────►│  Vlan10 .3   HSRP10: STANDBY │
   │  Vlan20 .2   HSRP20: STANDBY │  Hello │  Vlan20 .3   HSRP20: ACTIVE  │
   │  Vlan99 .2   HSRP99: ACTIVE  │ v2 3s  │  Vlan99 .3   HSRP99: STANDBY │
   │  Te1/1/1 → CORE-SW1 (tracked)│        │  Te1/1/1 → CORE-SW1 (tracked)│
   └────────────────────────────┘        └────────────────────────────┘
       VIP VLAN10 = 10.10.10.1     VIP VLAN20 = 10.10.20.1    VIP VLAN99(บล็อก1) = 10.10.99.1

   บล็อก 2: DIST-SW3 / DIST-SW4  (รับผิดชอบ VLAN 30, 40, 99)     uplink → CORE-SW2

   ┌────────────────────────────┐        ┌────────────────────────────┐
   │        DIST-SW3             │        │        DIST-SW4             │
   │  Vlan30 .2   HSRP30: ACTIVE  │◄──────►│  Vlan30 .3   HSRP30: STANDBY │
   │  Vlan40 .2   HSRP40: STANDBY │  Hello │  Vlan40 .3   HSRP40: ACTIVE  │
   │  Vlan99 .4   HSRP99: ACTIVE  │ v2 3s  │  Vlan99 .5   HSRP99: STANDBY │
   │  Te1/1/1 → CORE-SW2 (tracked)│        │  Te1/1/1 → CORE-SW2 (tracked)│
   └────────────────────────────┘        └────────────────────────────┘
       VIP VLAN30 = 10.10.30.1     VIP VLAN40 = 10.10.40.1    VIP VLAN99(บล็อก2) = 10.10.99.1
```

> **หมายเหตุสำคัญเรื่อง VLAN 99 (MGMT) ที่ต้องเข้าใจให้ชัด**: ตามที่ [Step 97, Part 10]
> (part-010-inter-vlan-routing.md) ได้ระบุไว้แล้วว่า VLAN 99 ใช้ Subnet เดียวกันทั่วองค์กร
> (`10.10.99.0/24`) แต่ถูกแบ่งเป็น **2 ขอบเขต Layer 2 ที่ไม่เชื่อมถึงกันทางกายภาพ** (เพราะ Core
> เป็น Routed Port ล้วน ไม่มี VLAN ใดถูก Bridge ข้าม Core) — ผลคือ HSRP Group 99 ของบล็อก 1 และ
> บล็อก 2 เป็น**คนละ Broadcast Domain กันโดยสมบูรณ์** ต่างฝ่ายต่างรัน HSRP อิสระจากกันโดยไม่มี
> Hello Packet ชนกันในทาง Protocol (Multicast Hello ไม่ข้าม Broadcast Domain อยู่แล้ว) การใช้ VIP
> `.1` ซ้ำกันทั้ง 2 บล็อกจึงไม่ก่อ Conflict ในทาง HSRP เอง แต่**ยังเป็นข้อจำกัดด้าน IP Design ที่
> สืบทอดมาจาก Part 10** (Subnet เดียวกันไม่ควรมี 2 ขอบเขต L3 ที่แยกจากกันจริง) — การแก้ไขระยะยาว
> คือแยก **Out-of-Band Management Network** เป็นคนละ Subnet ต่อบล็อกอย่างเหมาะสม ซึ่งเป็นงาน
> Design ระดับ CCNP ENCORE ที่จะอยู่นอกขอบเขตของ Part นี้ — สำหรับ Lab นี้ให้เข้าใจว่า HSRP ทำงาน
> ถูกต้องสมบูรณ์ **ภายในแต่ละบล็อก** ซึ่งเพียงพอสำหรับเป้าหมายการสอน FHRP ของ Part นี้

### 180.2 Running-Config เต็ม — DIST-SW1

```
DIST-SW1(config)# standby version 2
!
DIST-SW1(config)# interface vlan 10
DIST-SW1(config-if)# description ** Gateway VLAN10-SALES (HSRP Active) **
DIST-SW1(config-if)# ip address 10.10.10.2 255.255.255.0
DIST-SW1(config-if)# standby version 2
DIST-SW1(config-if)# standby 10 ip 10.10.10.1
DIST-SW1(config-if)# standby 10 priority 150
DIST-SW1(config-if)# standby 10 preempt
DIST-SW1(config-if)# standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# interface vlan 20
DIST-SW1(config-if)# description ** Gateway VLAN20-VOICE (HSRP Standby) **
DIST-SW1(config-if)# ip address 10.10.20.2 255.255.255.0
DIST-SW1(config-if)# standby version 2
DIST-SW1(config-if)# standby 20 ip 10.10.20.1
DIST-SW1(config-if)# standby 20 priority 100
DIST-SW1(config-if)# standby 20 preempt
DIST-SW1(config-if)# standby 20 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# exit
!
DIST-SW1(config)# interface vlan 99
DIST-SW1(config-if)# description ** Gateway VLAN99-MGMT block1 (HSRP Active) **
DIST-SW1(config-if)# ip address 10.10.99.2 255.255.255.0
DIST-SW1(config-if)# standby version 2
DIST-SW1(config-if)# standby 99 ip 10.10.99.1
DIST-SW1(config-if)# standby 99 priority 150
DIST-SW1(config-if)# standby 99 preempt
DIST-SW1(config-if)# standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW1(config-if)# exit
!
! Interface และ Static Route ที่เหลือคงเดิมตาม Part 10 (Trunk ไป ACCESS-SW1/2, Routed Uplink
! TenGigabitEthernet1/1/1 = 10.255.10.2/30 ไป CORE-SW1, ip route 0.0.0.0 0.0.0.0 10.255.10.1)
```

### 180.3 Running-Config เต็ม — DIST-SW2

```
DIST-SW2(config)# standby version 2
!
DIST-SW2(config)# interface vlan 10
DIST-SW2(config-if)# description ** Gateway VLAN10-SALES (HSRP Standby) **
DIST-SW2(config-if)# ip address 10.10.10.3 255.255.255.0
DIST-SW2(config-if)# standby version 2
DIST-SW2(config-if)# standby 10 ip 10.10.10.1
DIST-SW2(config-if)# standby 10 priority 100
DIST-SW2(config-if)# standby 10 preempt
DIST-SW2(config-if)# standby 10 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# exit
!
DIST-SW2(config)# interface vlan 20
DIST-SW2(config-if)# description ** Gateway VLAN20-VOICE (HSRP Active) **
DIST-SW2(config-if)# ip address 10.10.20.3 255.255.255.0
DIST-SW2(config-if)# standby version 2
DIST-SW2(config-if)# standby 20 ip 10.10.20.1
DIST-SW2(config-if)# standby 20 priority 150
DIST-SW2(config-if)# standby 20 preempt
DIST-SW2(config-if)# standby 20 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# exit
!
DIST-SW2(config)# interface vlan 99
DIST-SW2(config-if)# description ** Gateway VLAN99-MGMT block1 (HSRP Standby) **
DIST-SW2(config-if)# ip address 10.10.99.3 255.255.255.0
DIST-SW2(config-if)# standby version 2
DIST-SW2(config-if)# standby 99 ip 10.10.99.1
DIST-SW2(config-if)# standby 99 priority 100
DIST-SW2(config-if)# standby 99 preempt
DIST-SW2(config-if)# standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW2(config-if)# exit
!
! Interface และ Static Route ที่เหลือคงเดิมตาม Part 10 (Uplink Te1/1/1 = 10.255.20.2/30
! ไป CORE-SW1, ip route 0.0.0.0 0.0.0.0 10.255.20.1)
```

### 180.4 Running-Config เต็ม — DIST-SW3

```
DIST-SW3(config)# standby version 2
!
DIST-SW3(config)# interface vlan 30
DIST-SW3(config-if)# description ** Gateway VLAN30-SERVERS (HSRP Active) **
DIST-SW3(config-if)# ip address 10.10.30.2 255.255.255.0
DIST-SW3(config-if)# standby version 2
DIST-SW3(config-if)# standby 30 ip 10.10.30.1
DIST-SW3(config-if)# standby 30 priority 150
DIST-SW3(config-if)# standby 30 preempt
DIST-SW3(config-if)# standby 30 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW3(config-if)# exit
!
DIST-SW3(config)# interface vlan 40
DIST-SW3(config-if)# description ** Gateway VLAN40-WIFI (HSRP Standby) **
DIST-SW3(config-if)# ip address 10.10.40.2 255.255.255.0
DIST-SW3(config-if)# standby version 2
DIST-SW3(config-if)# standby 40 ip 10.10.40.1
DIST-SW3(config-if)# standby 40 priority 100
DIST-SW3(config-if)# standby 40 preempt
DIST-SW3(config-if)# standby 40 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW3(config-if)# exit
!
DIST-SW3(config)# interface vlan 99
DIST-SW3(config-if)# description ** Gateway VLAN99-MGMT block2 (HSRP Active) **
DIST-SW3(config-if)# ip address 10.10.99.4 255.255.255.0
DIST-SW3(config-if)# standby version 2
DIST-SW3(config-if)# standby 99 ip 10.10.99.1
DIST-SW3(config-if)# standby 99 priority 150
DIST-SW3(config-if)# standby 99 preempt
DIST-SW3(config-if)# standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW3(config-if)# exit
!
! Interface และ Static Route ที่เหลือคงเดิมตาม Part 10 (Uplink Te1/1/1 = 10.255.30.2/30
! ไป CORE-SW2, ip route 0.0.0.0 0.0.0.0 10.255.30.1)
```

### 180.5 Running-Config เต็ม — DIST-SW4

```
DIST-SW4(config)# standby version 2
!
DIST-SW4(config)# interface vlan 30
DIST-SW4(config-if)# description ** Gateway VLAN30-SERVERS (HSRP Standby) **
DIST-SW4(config-if)# ip address 10.10.30.3 255.255.255.0
DIST-SW4(config-if)# standby version 2
DIST-SW4(config-if)# standby 30 ip 10.10.30.1
DIST-SW4(config-if)# standby 30 priority 100
DIST-SW4(config-if)# standby 30 preempt
DIST-SW4(config-if)# standby 30 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW4(config-if)# exit
!
DIST-SW4(config)# interface vlan 40
DIST-SW4(config-if)# description ** Gateway VLAN40-WIFI (HSRP Active) **
DIST-SW4(config-if)# ip address 10.10.40.3 255.255.255.0
DIST-SW4(config-if)# standby version 2
DIST-SW4(config-if)# standby 40 ip 10.10.40.1
DIST-SW4(config-if)# standby 40 priority 150
DIST-SW4(config-if)# standby 40 preempt
DIST-SW4(config-if)# standby 40 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW4(config-if)# exit
!
DIST-SW4(config)# interface vlan 99
DIST-SW4(config-if)# description ** Gateway VLAN99-MGMT block2 (HSRP Standby) **
DIST-SW4(config-if)# ip address 10.10.99.5 255.255.255.0
DIST-SW4(config-if)# standby version 2
DIST-SW4(config-if)# standby 99 ip 10.10.99.1
DIST-SW4(config-if)# standby 99 priority 100
DIST-SW4(config-if)# standby 99 preempt
DIST-SW4(config-if)# standby 99 track TenGigabitEthernet1/1/1 decrement 60
DIST-SW4(config-if)# exit
!
! Interface และ Static Route ที่เหลือคงเดิมตาม Part 10 (Uplink Te1/1/1 = 10.255.40.2/30
! ไป CORE-SW2, ip route 0.0.0.0 0.0.0.0 10.255.40.1)
```

### 180.6 เปลี่ยน Default Gateway ของ Host ทุกเครื่องเป็น VIP

หลังคอนฟิก HSRP เสร็จครบทุก Switch แล้ว **ขั้นตอนสุดท้ายที่จำเป็น** คือเปลี่ยน Default Gateway
ของ Host/Server ทุกเครื่องในทุก VLAN จาก IP จริงของ DIST-SW (`.2`/`.3`/`.4`/`.5`) ให้เป็น **VIP
`.1` ของ VLAN ตัวเอง** เท่านั้น:

| VLAN | Default Gateway เดิม (Part 10) | Default Gateway ใหม่ (Part 18) |
|---|---|---|
| 10 – SALES | `10.10.10.2` หรือ `10.10.10.3` | **`10.10.10.1`** |
| 20 – VOICE | `10.10.20.2` หรือ `10.10.20.3` | **`10.10.20.1`** |
| 30 – SERVERS | `10.10.30.2` หรือ `10.10.30.3` | **`10.10.30.1`** |
| 40 – WIFI | `10.10.40.2` หรือ `10.10.40.3` | **`10.10.40.1`** |
| 99 – MGMT | `10.10.99.2/.3` หรือ `10.10.99.4/.5` | **`10.10.99.1`** (ตามบล็อกที่ Host นั้นอยู่) |

### 180.7 Verify ทั่วทั้ง Topology หลัง Config เสร็จสมบูรณ์

```
DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           10.10.10.3      10.10.10.1
Vl20        20   100 P Standby  10.10.20.3       local           10.10.20.1
Vl99        99   150 P Active   local           10.10.99.3      10.10.99.1

DIST-SW2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Standby  10.10.10.2       local           10.10.10.1
Vl20        20   150 P Active   local           10.10.20.2      10.10.20.1
Vl99        99   100 P Standby  10.10.99.2       local           10.10.99.1

DIST-SW3# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl30        30   150 P Active   local           10.10.30.3      10.10.30.1
Vl40        40   100 P Standby  10.10.40.3       local           10.10.40.1
Vl99        99   150 P Active   local           10.10.99.5      10.10.99.1

DIST-SW4# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl30        30   100 P Standby  10.10.30.2       local           10.10.30.1
Vl40        40   150 P Active   local           10.10.40.2      10.10.40.1
Vl99        99   100 P Standby  10.10.99.4       local           10.10.99.1

! ทดสอบจาก PC1 (VLAN10) ด้วย Default Gateway ใหม่
PC1> ping 10.10.10.1
Reply from 10.10.10.1: bytes=32 time=1ms TTL=255
Reply from 10.10.10.1: bytes=32 time=1ms TTL=255

PC1> arp -a
Interface: 10.10.10.10
  Internet Address    Physical Address    Type
  10.10.10.1           00-00-0c-9f-f0-0a   dynamic    <- Virtual MAC ของ HSRP Group 10 ตรงตาม Step 172
```

### 180.8 ทดสอบ Failover จริง — Shutdown Uplink ของ DIST-SW1 แล้วสังเกต DIST-SW2 รับหน้าที่ต่อ

```
! สถานะก่อนทดสอบ (ปกติ) — DIST-SW1 Active ของ VLAN10
DIST-SW1# show standby vlan 10 | include State|Priority
  State is Active
  Priority 150 (configured 150)

! === จำลอง Failure จริง: สาย/พอร์ต Uplink ไป CORE-SW1 ขาด ===
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# shutdown

! Console ของ DIST-SW1 แสดง Log ทันที
DIST-SW1#
%LINK-3-UPDOWN: Interface TenGigabitEthernet1/1/1, changed state to administratively down
%TRACK-6-STATE: 1 interface Te1/1/1 line-protocol Down -> Down
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Active -> Speak
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Speak -> Standby

! Priority ตกจาก 150 เหลือ 90 (150-60) ทันที
DIST-SW1# show standby vlan 10 | include State|Priority
  State is Standby
  Priority 90 (configured 150)

! ฝั่ง DIST-SW2 เห็น Active หายไป (ไม่มี Hello เข้ามาอีก) และ Priority 90 < 100 ของตัวเอง
! → เข้ายึด Active ทันที
DIST-SW2#
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Standby -> Active

DIST-SW2# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Active   local           10.10.10.2      10.10.10.1
                            ▲
                    รับหน้าที่ Active ของ VLAN 10 แทน DIST-SW1 แล้วสำเร็จ

! ทดสอบจาก PC1 ระหว่าง Failover — เห็น Delay เล็กน้อย (Loss ไม่กี่ Packet) แล้วกลับมาปกติ
PC1> ping 10.10.30.10 -t     (ping ต่อเนื่องไปหา Server1 ข้าม VLAN ผ่าน Gateway)
Reply from 10.10.30.10: bytes=32 time=2ms TTL=254
Reply from 10.10.30.10: bytes=32 time=2ms TTL=254
Request timed out.                                    <- ช่วง Failover เกิด Loss ประมาณ 1 Packet
Reply from 10.10.30.10: bytes=32 time=3ms TTL=254      <- กลับมาปกติ (Path ใหม่ผ่าน DIST-SW2)
Reply from 10.10.30.10: bytes=32 time=3ms TTL=254

! คืนสภาพ: no shutdown แล้ว preempt จะทำให้ DIST-SW1 ยึด Active คืนอัตโนมัติ
DIST-SW1(config)# interface TenGigabitEthernet1/1/1
DIST-SW1(config-if)# no shutdown
DIST-SW1#
%LINK-3-UPDOWN: Interface TenGigabitEthernet1/1/1, changed state to up
%TRACK-6-STATE: 1 interface Te1/1/1 line-protocol Down -> Up
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Standby -> Active   <- Preempt ทำงาน ยึด Active คืน

DIST-SW1# show standby brief
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           10.10.10.3      10.10.10.1
```

**สรุปผลการทดสอบ**: เมื่อ Uplink ของ DIST-SW1 ขาด ระบบ Failover ไปยัง DIST-SW2 โดยอัตโนมัติ
ภายในไม่กี่วินาที (สูญเสีย Packet เพียง 1-2 Packet ระหว่างช่วง Transition) โดย PC1 **ไม่ต้อง
เปลี่ยน Default Gateway หรือ ARP Cache เลยแม้แต่ครั้งเดียว** — และเมื่อ Uplink กลับมาปกติ ระบบ
ก็คืนสู่สภาวะ Load-Sharing เดิมโดยอัตโนมัติเช่นกัน (เพราะ `preempt` เปิดอยู่ทั้งสองฝั่ง) — นี่คือ
เป้าหมายสูงสุดของ FHRP ที่ทำให้ Distribution Layer มี **High Availability ที่แท้จริงในระดับ
Default Gateway**

---

## แบบฝึกหัดทวนความเข้าใจ Part 18

1. เพราะเหตุใด แม้ Distribution Layer จะมี DIST-SW1 และ DIST-SW2 คู่กันเพื่อ Redundancy (ตาม
   3-Tier Design) แต่ก่อนเรียน FHRP ใน Part นี้ Redundancy นั้นก็ยังไม่มีประโยชน์กับ Host เลย?
2. HSRP Group หมายเลข 30 (VLAN30-SERVERS) จะมี Virtual MAC Address เป็นค่าใดตาม Format ของ
   HSRPv2?
3. ถ้าคอนฟิก `standby 10 priority 150` บน DIST-SW1 แต่**ไม่ได้**ใส่ `standby 10 preempt` จะเกิด
   อะไรขึ้นหลังจาก DIST-SW1 Reload กลับมาออนไลน์ใหม่ ทั้งที่ Priority สูงกว่า DIST-SW2 อยู่แล้ว?
4. เพราะเหตุใด HSRP Interface Tracking (`standby track ... decrement ...`) จึงจำเป็น แม้ตัว SVI
   ของ VLAN นั้นจะยัง up/up อยู่ตลอดเวลาก็ตาม?
5. ในสถานการณ์ที่องค์กรมี Firewall ยี่ห้ออื่น (ไม่ใช่ Cisco) ต้องเข้าร่วม FHRP Group เดียวกันกับ
   Cisco Switch ควรเลือกใช้ FHRP ตัวใด และเพราะเหตุใด GLBP หรือ HSRP จึงไม่เหมาะกับสถานการณ์นี้?

**เฉลย:**
1. เพราะ Host แต่ละเครื่องตั้ง Default Gateway ได้เพียง IP เดียว — ถ้าตั้งเป็น IP จริงของ
   DIST-SW1 (`.2`) เพียงตัวเดียว เมื่อ DIST-SW1 ล่ม Host จะ ARP หา Gateway ไม่ได้เลย แม้
   DIST-SW2 (`.3`) จะยังทำงานปกติสมบูรณ์ก็ตาม เพราะ Host ไม่รู้จักและไม่ Failover ไปหา IP อื่น
   เองอัตโนมัติ — ต้องมี FHRP เพื่อสร้าง Virtual IP ร่วมกันให้ Host ใช้แทน
2. Group 30 แปลงเป็น Hexadecimal = `01E` → Virtual MAC = `0000.0C9F.F01E`
3. DIST-SW1 จะกลับมาเป็น **Standby** เท่านั้น (ไม่ยึด Active คืน) เพราะไม่มี `preempt` แม้ Priority
   จะสูงกว่าก็ตาม ระบบจะยังใช้ DIST-SW2 เป็น Active ต่อไปจนกว่า DIST-SW2 จะล่มเองอีกครั้ง หรือมีคน
   สั่ง `clear standby` เพื่อ Force Re-election
4. เพราะ SVI ของ VLAN ฝั่ง Access อาจ up/up ปกติ แต่ปัญหาจริงเกิดที่ **Uplink ไปยัง Core** ขาด
   ทำให้ Switch นั้นกลายเป็น "Black-Hole" (รับ Traffic จาก Host แต่ไม่สามารถส่งต่อไปที่อื่นได้) —
   Interface Tracking ทำให้ HSRP รับรู้ปัญหานี้และลด Priority เพื่อสละ Active ให้ตัวที่ Uplink
   ยังปกติแทน
5. ควรใช้ **VRRP** เพราะเป็น Open Standard ตาม RFC 3768/5798 ที่อุปกรณ์ทุกยี่ห้อที่รองรับ
   Standard นี้เข้าร่วมกลุ่มเดียวกันได้ — **HSRP เป็น Cisco Proprietary ใช้ได้เฉพาะ Cisco เท่านั้น**
   ส่วน **GLBP ก็เป็น Cisco Proprietary เช่นกัน** ไม่ได้ถูกออกแบบมาให้ทำงานร่วมกับอุปกรณ์ต่างยี่ห้อ
   ดังนั้นทั้งสองตัวนี้ไม่เหมาะกับสถานการณ์ Multi-Vendor

---

## สรุป Part 18

Part นี้ปิดช่องว่างสำคัญที่ค้างมาตั้งแต่ Part 10 — **Default Gateway ของทุก VLAN ในหลักสูตรนี้
มี Redundancy ที่แท้จริงแล้ว** ด้วย **HSRPv2** เราเรียนรู้ State Machine (Active/Standby/Listen),
Timer (Hello 3s/Hold 10s), Virtual MAC Format, การ Load-Share Active Role ระหว่าง DIST-SW คู่กัน
ด้วยการสลับ Priority ตาม VLAN, การป้องกัน Black-Hole ด้วย Interface Tracking, และเปรียบเทียบกับ
FHRP อีก 2 ตัวคือ **VRRP** (Open Standard สำหรับ Multi-Vendor) และ **GLBP** (Active/Active
Load-Balancing แบบ Cisco) ปิดท้ายด้วย Lab เต็มรูปแบบที่เติมค่า HSRP Virtual IP `.1` เข้าไปในทุก
VLAN ของ DIST-SW1–4 ทั้ง 4 ตัว พร้อมพิสูจน์การ Failover จริงด้วยการ Shutdown Uplink

ตอนนี้ Distribution Layer ของ Enterprise Lab มี **Redundancy ครบทั้ง Hardware (คู่ Switch) และ
Logical Gateway (HSRP VIP)** แล้ว — สิ่งที่ยังไม่ได้ทำคือการเชื่อมต่อ Enterprise Network นี้ออกไป
สู่โลกภายนอกผ่าน WAN ซึ่งเป็นหัวข้อของ **Part 19 เป็นต้นไป** ที่จะเข้าสู่ระดับ CCNP ENCOR/ENARSI
เต็มรูปแบบ เริ่มจาก **WAN Technologies พื้นฐาน** (MPLS, Internet, Leased Line, และการเลือก
Technology ให้เหมาะกับ Requirement ขององค์กร)

**ไปต่อ:** [Part 19 — WAN Technologies →](part-019-wan-technologies.md)
