# Part 81 — High Availability (NSF/SSO, ISSU)
**หลักสูตร Cisco CCNA → CCNP → CCIE | Step 801–810 จาก 1000**

> ต่อจาก [Part 80 — Model-Driven Telemetry & Streaming Telemetry](part-080-model-driven-telemetry-streaming.md)
> ที่สอนให้ Network มอง**เห็นตัวเอง**ผ่าน gRPC/gNMI Streaming Telemetry — Part นี้จะกลับมาที่คำถาม
> ที่ค้างไว้นานที่สุดของหลักสูตรนี้: ถ้า Network มองเห็นตัวเองแล้ว แต่ตัว **Hardware ที่ทำหน้าที่
> Forward/Route** ยังล่มได้อยู่ดี Design ที่ดีจะรับมืออย่างไร? [Part 26 Step 256](part-026-advanced-ospf.md#step-256--ospf-graceful-restart--nsf-สำหรับ-high-availability-platform)
> แนะนำ **OSPF Graceful Restart/NSF** ไว้แบบสั้นๆ, [Part 35 Step 345-346](part-035-advanced-fhrp-campus-ha.md#step-345--nsf-และ-sso-supervisor-redundancy-ภายในเครื่องเดียว)
> Preview แนวคิด **SSO + Redundancy Group** ไว้ในระดับ Concept เพราะ Platform ของ Lab (Catalyst
> 9300/9500 แบบ Fixed-Configuration) ไม่มี Hardware Dual-Supervisor ให้ Demo จริง และ
> [Part 74 Step 736](part-074-advanced-bgp-wan-edge.md#step-736--graceful-restart-และ-bgp-nsf)
> ปิดช่องว่างของ BGP Graceful Restart ไปแล้วอีกส่วน — **Part นี้คือ Part ที่เก็บหนี้ทั้งหมดนั้น**:
> เจาะกลไก **SSO (Stateful Switchover)** และ **NSF (Non-Stop Forwarding)** ให้ลึกถึงระดับ
> Data-Plane/Control-Plane แยกกันจริง, สอน **ISSU/eFSU** สำหรับ Upgrade Software แบบไม่มี Outage,
> เชื่อมกลับไปที่ **StackWise Virtual (Part 34)** ว่า SSO ทำงานข้าม 2 Chassis ได้อย่างไร, และปิดท้าย
> ด้วยกลไก HA อีกแบบสำหรับ Platform ที่ไม่มี Dual-RP เลย (WAN Edge Router) พร้อม Lab เต็มรูปแบบที่
> พิสูจน์ด้วยของจริงว่า **NSF ทำให้ Neighbor รอบข้างไม่รู้เลยว่า RP กำลังสลับตัว**

## สารบัญ Step ใน Part นี้

| Step | หัวข้อ |
|---|---|
| 801 | Chassis-level HA พื้นฐาน — Dual Supervisor/RP, RPR vs SSO: อะไรถูกเก็บ อะไรไม่ถูกเก็บ |
| 802 | SSO เจาะลึก — Sync อะไรระหว่าง Active/Standby RP, Config `redundancy` เต็มรูปแบบ |
| 803 | NSF เจาะลึก — Graceful Restart ทำให้ Neighbor ไม่รู้ตัวได้อย่างไร (NSF-capable vs NSF-aware) |
| 804 | SSO+NSF Failover Sequence — Timeline ทุกวินาทีของ RP Switchover จริง |
| 805 | ISSU / eFSU — Upgrade IOS-XE บน Dual-RP/SVL แบบไม่มี Service Outage |
| 806 | ISSU Compatibility Matrix, ข้อจำกัด, และ Rollback Procedure |
| 807 | StackWise Virtual + SSO Interaction — SSO ข้าม 2 Chassis ต่างจาก Dual-RP ในเครื่องเดียวอย่างไร |
| 808 | HA สำหรับ WAN Edge Router — Chassis-internal (NSF/SSO) vs Router-external (HSRP/VRRP) |
| 809 | Testing/Validating HA — Planned Failover Test, `redundancy force-switchover`, Maintenance Runbook |
| 810 | Lab เต็มรูปแบบ: NSF+SSO บน CORE-SW1/CORE-SW2 (SVL), ISSU Walkthrough, Live Failover Test |

---

## Step 801 — Chassis-level HA พื้นฐาน: Dual Supervisor/RP, RPR vs SSO

### 801.1 ปัญหาที่ Chassis-level HA แก้ — คนละชั้นกับ FHRP และคนละชั้นกับ StackWise Virtual

หลักสูตรนี้สอน HA มาแล้วหลายชั้น: **HSRP/VRRP** (Part 18/35) แก้ปัญหา "อุปกรณ์ทั้งเครื่องล่ม แล้วมี
อุปกรณ์คู่แข่งอีกเครื่องรับหน้าที่แทน", **StackWise Virtual** (Part 34) แก้ปัญหา "2 Chassis แยกกัน
อยากให้ดูเป็น 1 Logical Switch เพื่อทำ MEC" — Step นี้แก้ปัญหาที่**เล็กกว่านั้นอีกชั้น**: Platform
ระดับ Modular Chassis (เช่น Catalyst 9600, 6807-XL รุ่นเก่า, หรือ ASR1000 Series ที่มี Slot RSP คู่)
รองรับการเสียบ **Route Processor (RP) / Supervisor Module สองตัวในแชสซีเดียวกัน** — ถ้า RP ตัว
Active ล่มหรือต้อง Reload (เช่น ทำ Software Upgrade) แต่ Line Card/Power/Fan ของแชสซีเดียวกันยัง
ทำงานปกติทุกอย่าง คำถามคือ: **RP สำรองจะเข้ารับหน้าที่ต่อได้เร็วแค่ไหน และ "เร็ว" นั้นแปลว่าอะไรกันแน่?**

```
Modular Chassis ที่มี Dual RP (แนวคิด — ไม่มีใน Lab หลักของหลักสูตรนี้)

        ┌─────────────────────────────────────────────────┐
        │                    CHASSIS                        │
        │  ┌───────────┐        ┌───────────┐               │
        │  │  RP/Sup A  │◄──────►│  RP/Sup B  │  <- Sync Link │
        │  │  (ACTIVE)  │        │ (STANDBY)  │     ภายใน     │
        │  └───────────┘        └───────────┘               │
        │        │                                           │
        │  ┌─────┴──────────────────────────────────┐        │
        │  │     Line Card 1 / 2 / 3 / ... (ASIC)     │  <- Data Plane
        │  │     ไม่ผ่าน RP โดยตรง Forward ด้วย CEF/ASIC │        │
        │  └──────────────────────────────────────────┘        │
        └─────────────────────────────────────────────────┘
```

> **หมายเหตุ Platform ของ Lab หลักสูตรนี้**: CORE-SW1/CORE-SW2 (Catalyst 9500) และ DIST-SW1–4
> (Catalyst 9300) ตาม [00-ip-address-plan.md](00-ip-address-plan.md) เป็น **Fixed-Configuration
> Platform ที่มี Supervisor ตัวเดียวในตัวเอง** — ไม่มี Slot สำหรับ RP สำรองแบบ Modular Chassis
> จริง Step 801-806 ของ Part นี้จึงสอนแนวคิดในระดับ Modular Chassis (Catalyst 9600/ASR1000) ตามที่
> CCNP ENCOR/CCIE Exam ถามถึง ส่วน Step 807 จะพากลับมาที่ Platform จริงของ Lab ผ่าน **StackWise
> Virtual** ซึ่งทำให้ CORE-SW1+CORE-SW2 ได้ผลลัพธ์ "เหมือน" Dual-RP แม้จะเป็น 2 กล่องแยกกันจริง

### 801.2 สองโหมดของ Redundancy: RPR vs SSO

Cisco IOS/IOS-XE รองรับ **Redundancy Mode** สำหรับ Dual-RP Platform อยู่ 2 แบบหลัก (ไม่นับ RPR+
ที่เป็นรุ่นกลางบน Platform เก่าบางตัวซึ่งเลิกใช้ไปแล้ว):

| Redundancy Mode | ชื่อเต็ม | ลักษณะการทำงาน |
|---|---|---|
| **RPR** | Route Processor Redundancy | Standby RP **บู๊ต IOS Image และโหลด Startup-config ไว้ล่วงหน้า** เท่านั้น (Cold Standby) — ไม่ Sync Runtime State ต่อเนื่อง |
| **SSO** | Stateful Switchover | Standby RP **Sync State กับ Active RP แบบ Real-time ตลอดเวลา** (Hot Standby) ผ่าน Checkpoint/IPC Message |

### 801.3 ตารางเปรียบเทียบเต็มรูปแบบ: อะไร "ถูกเก็บ" และอะไร "ไม่ถูกเก็บ"

นี่คือตารางที่สำคัญที่สุดของ Step นี้ — ข้อสอบ CCNP/CCIE มักถามตรงๆว่า "RPR เก็บอะไรไว้บ้าง" ซึ่งคำตอบ
คือ **น้อยมาก** เทียบกับ SSO:

| สิ่งที่ต้องพิจารณา | RPR (Cold/Slow) | SSO (Hot/Fast) |
|---|---|---|
| Standby RP โหลด IOS Image ไว้ล่วงหน้า | ✅ ใช่ | ✅ ใช่ |
| Startup-config ถูก Sync ไปยัง Standby | ✅ ใช่ | ✅ ใช่ |
| Running-config เปลี่ยนแปลง Real-time Sync ไป Standby | ❌ ไม่ (ต้อง `write memory` ก่อน) | ✅ ใช่ (Sync อัตโนมัติทุกคำสั่ง) |
| Interface State (Up/Down, IP) | ❌ ไม่ (ต้องตรวจใหม่หลัง Switchover) | ✅ ใช่ |
| L2 Forwarding Table (MAC/CAM) | ❌ ไม่ | ✅ ใช่ |
| ARP Table | ❌ ไม่ | ✅ ใช่ |
| L3 CEF/FIB Table | ❌ ไม่ (ต้องสร้างใหม่จาก RIB) | ✅ ใช่ (Pre-built บน Standby ตลอดเวลา) |
| Line Card ถูก Reset ระหว่าง Switchover | ✅ ใช่ (Reload/Re-init) | ❌ ไม่ (Data Plane ทำงานต่อเนื่อง) |
| Routing Protocol Adjacency (OSPF/EIGRP/BGP Neighbor) | ❌ ไม่ (ต้องสร้าง Adjacency ใหม่ทั้งหมด) | ❌ **ไม่เช่นกัน — ต้องพึ่ง NSF เพิ่ม** (ดู Step 802-803) |
| Traffic Forwarding ระหว่าง Switchover | ❌ หยุด (วินาทีถึงนาที) | ✅ ต่อเนื่อง (Sub-second ถึงไม่กี่วินาที) |
| Switchover Time โดยประมาณ | หลายสิบวินาทีถึงหลายนาที (เหมือน Reload บางส่วน) | ต่ำกว่า 1 วินาทีถึงไม่กี่วินาที |

> **ประเด็นที่ถูกเข้าใจผิดบ่อยที่สุด**: หลายคนคิดว่า SSO เก็บทุกอย่างไว้ 100% รวมถึง Routing
> Protocol Adjacency ด้วย — **ผิด** SSO เก็บ "Forwarding State" (L2/L3 Data Plane) กับ "Chassis/
> Interface State" ไว้เท่านั้น ส่วน Routing Protocol Neighbor Relationship เป็น **Control Plane
> State ที่ซับซ้อนและเปลี่ยนเร็วเกินกว่าจะ Sync ได้ทั้งหมดแบบ Real-time** — นี่คือเหตุผลที่ต้องมี
> **NSF** เป็น Layer เพิ่มเติมสำหรับ Routing Protocol โดยเฉพาะ (Step 802.3 และ 803 จะอธิบายเหตุผล
> เชิงเทคนิคว่าทำไม Sync ทั้งหมดไม่ได้)

---

## Step 802 — SSO เจาะลึก: อะไร Sync บ้าง และ Config เต็มรูปแบบ

### 802.1 กลไก Checkpointing — SSO Sync อะไรจริงๆ

SSO ใช้กลไกที่เรียกว่า **Checkpoint Facility (CF)** ส่ง State จาก Active RP ไปยัง Standby RP ผ่าน
IPC (Inter-Process Communication) ภายใน Backplane ของ Chassis (หรือผ่าน SVL Link ถ้าเป็น StackWise
Virtual ตาม Step 807) โดยแบ่งเป็น 3 ประเภทข้อมูลหลักที่ Sync:

```
                    Active RP                          Standby RP
              ┌──────────────────┐              ┌──────────────────┐
              │  Routing Protocol │              │  Routing Protocol │
              │  Process (OSPF,   │   ✗ ไม่ Sync  │  Process ยังไม่รัน │
              │  EIGRP, BGP)      │──────X───────│  จริง (ต้องพึ่ง NSF)│
              ├──────────────────┤              ├──────────────────┤
              │  RIB (Routing     │   ✗ ไม่ Sync  │  RIB ว่างเปล่า      │
              │  Information Base)│──────X───────│  ตอนเริ่มต้น         │
              ├──────────────────┤              ├──────────────────┤
              │  CEF/FIB Table    │   ✓ Sync ตลอด │  CEF/FIB Table     │
              │  (Forwarding)     │──────✓───────►│  (พร้อม Forward ทันที)│
              ├──────────────────┤              ├──────────────────┤
              │  Interface State  │   ✓ Sync ตลอด │  Interface State   │
              │  (Line Protocol)  │──────✓───────►│  (Mirror ทันที)     │
              ├──────────────────┤              ├──────────────────┤
              │  L2 MAC/ARP Table │   ✓ Sync ตลอด │  L2 MAC/ARP Table  │
              └──────────────────┘──────✓───────►└──────────────────┘
```

**เหตุผลที่ CEF/FIB Sync ได้ แต่ RIB/Routing Protocol Sync ไม่ได้**: CEF/FIB เป็นโครงสร้างข้อมูล
**ผลลัพธ์สุดท้าย** (Flat Table ที่บอกแค่ "Prefix นี้ Forward ไปทาง Interface/Next-hop ไหน") ซึ่งมี
ขนาดคงที่และอัปเดตแบบ Incremental ได้ง่าย ส่วน Routing Protocol (LSDB ของ OSPF, Topology Table
ของ EIGRP, BGP Table พร้อม Path Attribute ทั้งหมด) เป็น **State Machine ที่ซับซ้อนมาก มี Timer,
Neighbor FSM, และ Database ที่ต้องเชื่อมโยงกับ Neighbor จริงตลอดเวลา** — การ Sync ทั้ง State
Machine นี้แบบ Real-time ต้องแทบจะรัน Protocol Instance สองชุดพร้อมกัน (ซึ่งมี Platform บางรุ่นที่
เริ่มทำแบบนี้จริงในปีหลังๆ เรียกว่า "Distributed" Routing Protocol แต่ยังไม่ใช่ Default ของ IOS-XE
ทั่วไป) — Cisco เลือกแก้ปัญหานี้ด้วยวิธีที่ฉลาดกว่า: **ไม่ต้อง Sync Routing Protocol State เลย
แต่หลอก Neighbor ให้รอจนกว่า Protocol จะสร้างตัวเองใหม่เสร็จ (NSF/Graceful Restart)**

### 802.2 Config `redundancy` เต็มรูปแบบ

```
! บน Modular Chassis ที่มี Dual RP ติดตั้งจริง (แนวคิด — Catalyst 9600/ASR1000)
CORE-SW1(config)# redundancy
CORE-SW1(config-red)# mode sso
CORE-SW1(config-red)# main-cpu
CORE-SW1(config-r-mc)# standby console enable
CORE-SW1(config-r-mc)# exit
CORE-SW1(config-red)# exit
```

- `mode sso` — เปลี่ยน Redundancy Mode จาก RPR (Default บน Platform เก่าบางรุ่น) เป็น SSO
- `standby console enable` — เปิด Console Access ไปยัง Standby RP โดยตรง (ปกติ Login เข้า Active
  RP อย่างเดียว Standby ไม่มี CLI Session แยกให้ใช้งานตามปกติ)
- **คำสั่งนี้มีผลทันทีโดยไม่ต้อง Reload** (ต่างจาก StackWise Virtual ที่ต้อง Reload ตาม Part 34
  Step 333) เพราะเป็นการเปลี่ยน Redundancy Facility ภายในเครื่องเดียวที่ยังทำงานอยู่ ไม่ใช่การ
  รวม 2 Chassis เป็น 1

### 802.3 Verification

```
CORE-SW1# show redundancy states
       my state = 13 -ACTIVE
     peer state = 8  -STANDBY HOT
           Mode = Duplex
           Unit = Secondary
        Unit ID = 2

  Redundancy Mode (Operational) = sso
  Redundancy Mode (Configured)  = sso
  Redundancy State              = sso
     Maintenance Mode = Disabled
    Manual Swact = enabled
     Communications = Up

   client count = 76
 client_notification_TMR = 30000 milliseconds
          keep_alive TMR = 9000 milliseconds
        keep_alive count = 1
    keep_alive threshold = 18
           RF debug mask = 0x0
```

- **peer state = 8 -STANDBY HOT** คือค่าที่ต้องเห็นเสมอก่อนจะไว้ใจ SSO ได้ — ถ้าเห็น `STANDBY COLD`
  หรือ `DISABLED` แปลว่า Standby ยังไม่ Sync สมบูรณ์ (Switchover ตอนนี้จะกลายเป็นแบบ RPR โดยไม่รู้ตัว)
- `Communications = Up` คือ IPC Link ระหว่าง 2 RP ยังทำงานปกติ — ถ้าเห็น `Down` ต้องแก้ก่อน SSO
  จะไม่มีทาง Sync อะไรได้เลย

```
CORE-SW1# show redundancy history | include SWITCHOVER|STATE
```

ใช้ตรวจสอบว่ามี Switchover เกิดขึ้นก่อนหน้านี้หรือไม่ และ State Transition ผ่านขั้นไหนบ้าง (ใช้อ่าน
Log หลัง Failover Test ใน Step 809/810)

---

## Step 803 — NSF เจาะลึก: Graceful Restart ทำให้ Neighbor ไม่รู้ตัวได้อย่างไร

### 803.1 ทวนกลไก Grace-LSA จาก Part 26 แล้วขยายให้ครบทุก Protocol

[Part 26 Step 256](part-026-advanced-ospf.md#step-256--ospf-graceful-restart--nsf-สำหรับ-high-availability-platform)
สอน Grace-LSA (RFC 3623) ของ OSPF ไว้แล้ว — Step นี้ขยายให้เห็นภาพเต็มว่าทุก Routing Protocol
หลักของหลักสูตรนี้มีกลไกแบบเดียวกัน แต่ Implementation Detail ต่างกัน:

| Protocol | ชื่อกลไก | RFC/มาตรฐาน | สิ่งที่ Restarting Router ส่งออกไป | Config Command |
|---|---|---|---|---|
| **OSPF** | Graceful Restart | RFC 3623 (+ Cisco Proprietary) | **Grace-LSA** (Type 9 Opaque LSA) | `nsf cisco` หรือ `nsf ietf` ใต้ `router ospf` |
| **EIGRP** | NSF | Cisco Proprietary | **Hello Packet พร้อม Restart (RS) Bit** ตั้งค่าไว้ | `nsf` ใต้ `router eigrp` |
| **BGP** | Graceful Restart | RFC 4724 | **Graceful Restart Capability + R-bit** ใน OPEN/Update Message | `bgp graceful-restart` ใต้ `router bgp` (สอนเต็มใน [Part 74 Step 736](part-074-advanced-bgp-wan-edge.md#step-736--graceful-restart-และ-bgp-nsf)) |

หลักการร่วมของทั้ง 3 Protocol เหมือนกันทุกตัว: **Restarting Router บอก Neighbor ล่วงหน้า (หรือทันที
ที่ฟื้นตัว) ว่า "อย่าตัด Adjacency/Session และอย่า Withdraw Route ของฉัน ฉันแค่กำลัง Restart Control
Plane ภายใน Data Plane เดิมยัง Forward อยู่"** — Neighbor ที่เข้าใจกลไกนี้จะ**คง Route เดิมไว้
เป็น Stale ชั่วคราว**ระหว่าง Grace/Restart Period แทนที่จะประกาศ Topology Change ทันที

### 803.2 บทบาท 2 ฝั่ง: NSF-capable (Restarting) vs NSF-aware (Helper)

| บทบาท | คำนิยาม | เงื่อนไข Hardware |
|---|---|---|
| **NSF-capable (Restarting Router)** | Router ที่**ทำ RP Switchover เอง**และประกาศ Grace-LSA/Capability ออกไป | ต้องมี **Dual-RP + SSO** จริง (ไม่มี SSO = Data Plane หยุดตอน Switchover อยู่ดี ต่อให้ Config NSF ไว้ก็ไม่มีประโยชน์) |
| **NSF-aware (Helper Router)** | Router ข้างเคียงที่**รับรู้และรอ**ให้ Restarting Router ฟื้นตัว | **ไม่ต้องมี Dual-RP เลย** แค่รองรับ Protocol Extension (เข้าใจ Grace-LSA/RS-bit/GR Capability) ก็เพียงพอ — เป็น Default Behavior ของ IOS ทุกตัวที่รองรับอยู่แล้ว ไม่ต้อง Config เพิ่ม |

### 803.3 Worked Scenario เต็มรูปแบบ: CORE (NSF-capable) กับ DIST-SW1 (NSF-aware Helper)

```
สถานการณ์: CORE (Dual-RP สมมติ, SSO Active) ทำ RP Switchover ขณะมี OSPF Adjacency กับ DIST-SW1

ขั้นที่ 1 — ก่อน Switchover (Planned):
  CORE (Active RP) ส่ง Grace-LSA ไปยัง DIST-SW1
  Grace-LSA บอก: "Grace Period = 120 วินาที, สาเหตุ = Software Reload/Switchover"
        │
ขั้นที่ 2 — DIST-SW1 (Helper) รับ Grace-LSA:
  ตรวจสอบว่าตัวเองรองรับ NSF-aware (Default = รองรับ)
  เข้าสู่ Helper Mode: "คง LSDB และ Adjacency เดิมไว้กับ CORE เป็น FULL"
  ไม่ Re-run SPF, ไม่ประกาศ Router LSA ใหม่ (Topology ไม่ได้เปลี่ยนจริง)
        │
ขั้นที่ 3 — Switchover เกิดขึ้นจริงบน CORE:
  Standby RP กลายเป็น Active RP ใหม่
  Data Plane (CEF/FIB ที่ Sync ไว้แล้วตาม Step 802) Forward ต่อเนื่องไม่หยุด
  OSPF Process เริ่มต้นใหม่บน RP ที่เพิ่งเป็น Active (LSDB ว่าง ต้องเรียนรู้ใหม่)
        │
ขั้นที่ 4 — Re-synchronization ภายใน Grace Period:
  OSPF Process ใหม่ส่ง Hello ออกไป DIST-SW1 ตอบกลับทันที (จำ CORE ไว้เป็น FULL อยู่แล้วจาก Helper Mode)
  ทั้งสองฝั่งแลกเปลี่ยน Database Description แบบเบา (DIST-SW1 มี LSDB เดิมอยู่แล้ว ไม่ต้อง Flood ใหม่
  ถ้าไม่มีอะไรเปลี่ยนจริง) — Adjacency กลับสู่ FULL ภายในไม่กี่วินาที
        │
ขั้นที่ 5 — จบ Grace Period สำเร็จ:
  CORE ส่ง Router LSA ตามปกติ, DIST-SW1 ออกจาก Helper Mode
  ผลลัพธ์สุทธิ: ไม่มีการ Re-run SPF ทั่ว Area แม้แต่ครั้งเดียว, Traffic ไม่หยุดตลอดกระบวนการ
```

### 803.4 ความเสี่ยงที่ต้องรู้: ถ้า Neighbor ไม่ใช่ NSF-aware

```
CORE (NSF-capable) ส่ง Grace-LSA ไปยัง Router รุ่นเก่า/Vendor อื่นที่ไม่เข้าใจ Type 9 Opaque LSA

ผลลัพธ์: Neighbor ไม่รู้จัก Grace-LSA เลย มองเป็น LSA แปลกที่ไม่เข้าใจ (เพิกเฉยหรือ Log Error)
         → เมื่อ OSPF Process เดิมของ CORE หายไปจริง (ตอน Switchover) Hold Timer ของ Neighbor
           จะหมดตามปกติ → Adjacency หลุด → SPF Re-run ตามปกติเหมือนไม่มี NSF เลย

บทเรียน: NSF ต้อง "ทั้งสองฝั่ง" เข้าใจกลไกเดียวกันจึงจะได้ผล ฝั่ง Restarting Router Config
`nsf cisco/ietf` อย่างเดียวไม่พอ ถ้า Environment มี Router ที่เก่ามากจริงๆ ควร Verify ด้วย
`show ip ospf neighbor detail` ว่าเห็น Capability ตรงกันก่อนไว้ใจ NSF ใน Production
```

---

## Step 804 — SSO+NSF Failover Sequence: Timeline ทุกวินาทีของ RP Switchover จริง

### 804.1 ภาพรวม — Data Plane กับ Control Plane แยกกันสมบูรณ์ระหว่าง Switchover

หัวใจของ SSO+NSF คือการแยก 2 เรื่องนี้ออกจากกันอย่างเด็ดขาด:

```
   Data Plane (Hardware/ASIC บน Line Card)  ───►  ไม่หยุดเลยตลอดกระบวนการ (SSO CEF/FIB Sync ไว้แล้ว)
   Control Plane (Routing Protocol Software) ───►  Restart ใหม่ทั้งหมด แต่ NSF "ซ่อน" การ Restart
                                                     นี้จาก Neighbor รอบข้าง
```

### 804.2 Timeline แบบละเอียด (RP Switchover ที่ไม่ได้ Planned — Active RP ล่มกะทันหัน)

| เวลา (สมมติ) | เหตุการณ์ | Data Plane | Control Plane |
|---|---|---|---|
| T+0.000s | Active RP หยุดทำงาน (Crash/Hardware Fault) | Forward ต่อเนื่องด้วย CEF/FIB เดิมที่ Sync ไว้ | OSPF/BGP Process เดิมหายไปทันที |
| T+0.050s | Standby RP ตรวจพบ Keepalive หายไปเกิน Threshold | Forward ต่อเนื่อง | Standby เริ่ม Redundancy Facility Switchover |
| T+0.100–0.300s | Standby ประกาศตัวเองเป็น Active ใหม่ | Forward ต่อเนื่อง (ไม่มี Interface Flap) | ยังไม่เริ่มรัน Routing Protocol Process |
| T+0.5–1s | Active RP ใหม่เริ่ม Init Routing Protocol Process จาก Running-config ที่ Sync ไว้ | Forward ต่อเนื่อง (CEF เดิมยังใช้ได้ เพราะ Topology จริงไม่เปลี่ยน) | OSPF Process ใหม่เริ่ม แต่ LSDB ว่างเปล่า → ส่ง Grace-LSA ทันทีที่เริ่ม (Unplanned NSF) |
| T+1–3s | Neighbor (Helper) ได้รับ Grace-LSA เข้าสู่ Helper Mode | Forward ต่อเนื่อง | Hello Exchange เริ่มใหม่ Adjacency กลับสู่ FULL อย่างรวดเร็ว (ไม่ Flood LSA ใหม่ถ้าไม่มีอะไรเปลี่ยน) |
| T+3–40s | Grace Period ดำเนินต่อ (Default 60-120s ตามที่ Config) | Forward ต่อเนื่อง | Routing Table ถูกสร้างใหม่จน "เหมือนเดิมทุกประการ" ก่อน Grace Period จะหมด |
| T+40s (สมมติจบก่อนกำหนด) | Adjacency ครบทุกตัว, LSDB ตรงกับก่อน Switchover | Forward ต่อเนื่อง (ไม่มีวินาทีใดที่หยุดเลย) | ออกจาก Graceful Restart State — กลับสู่สถานะปกติสมบูรณ์ |

> **ข้อแตกต่างสำคัญระหว่าง Planned กับ Unplanned Switchover**: ถ้าเป็น **Planned** (เช่น
> Operator สั่ง `redundancy force-switchover` ตาม Step 809 หรือ ISSU ตาม Step 805) Routing
> Protocol สามารถส่ง Grace-LSA **ก่อน** Switchover จริง ทำให้ Neighbor เข้าสู่ Helper Mode ล่วงหน้า
> — ถ้าเป็น **Unplanned** (RP Crash กะทันหัน) ไม่มีเวลาส่ง Grace-LSA ก่อน แต่ IOS ยังส่งได้ทันทีที่
> Process ใหม่เริ่มทำงานบน RP ที่กลายเป็น Active (เรียกว่า Grace-LSA แบบ "Unplanned/Unsignaled")
> ซึ่งได้ผลลัพธ์เดียวกันแต่มีโอกาสเสี่ยง Race Condition สูงกว่าเล็กน้อยถ้า Grace Period สั้นเกินไป

### 804.3 Worked Failover Test — อ่านผลจาก Log จริง

```
! ก่อน Switchover — Baseline
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/BDR        00:00:38    10.255.10.2     Te1/0/1
1.1.1.12        1     FULL/BDR        00:00:33    10.255.20.2     Te1/0/2

! สั่ง Switchover แบบ Controlled
CORE-SW1# redundancy force-switchover
This will reload the active unit and force switchover to standby[confirm]

! Log บน RP ที่กลายเป็น Active ใหม่ (Timestamp Service ต้อง Enable ไว้ก่อน — ดู Part 17)
*Mar  3 09:14:02.114: %RF-5-RF_TERMINAL_STATE: Terminal state reached for (SSO)
*Mar  3 09:14:02.201: %OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.11 on Te1/0/1 from LOADING to FULL, Loading Done
*Mar  3 09:14:02.204: %OSPF-5-NSF_NBR_RESYNC: Process 1, Nbr 1.1.1.11 on Te1/0/1: NSF resynchronized

! ทันทีหลัง Switchover — Adjacency ต้องเห็น FULL เหมือนเดิมทุกตัว
CORE-SW1# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/BDR        00:00:35    10.255.10.2     Te1/0/1
1.1.1.12        1     FULL/BDR        00:00:39    10.255.20.2     Te1/0/2
```

- `%OSPF-5-NSF_NBR_RESYNC` คือหลักฐานตรงๆว่า NSF ทำงานสำเร็จ — Neighbor ไม่ได้สร้าง Adjacency
  ใหม่จากศูนย์ แต่ **Resync** กับ State เดิม
- ระหว่างการทดสอบควรมี **Continuous Ping** จาก Management Station วิ่งคู่กันตลอด (`ping ... repeat
  1000`) — ผลลัพธ์ที่ถูกต้องคือ **Packet Loss = 0%** ตลอดกระบวนการทั้งหมด

---

## Step 805 — ISSU/eFSU: Upgrade IOS-XE แบบไม่มี Service Outage

### 805.1 ปัญหาที่ ISSU แก้ — Software Upgrade ที่ปกติต้อง Reload ทั้งเครื่อง

การ Upgrade IOS-XE แบบเดิม (`install add ... activate` ธรรมดา หรือ `boot system` แล้ว `reload`)
ทำให้ Chassis **หยุด Forward Packet ทั้งหมดชั่วขณะ** (นาทีถึงหลายนาทีขึ้นกับขนาด Platform) — สำหรับ
CORE Layer ที่ต้อง Forward Traffic ตลอด 24/7 นี่คือ Maintenance Window ที่มีค่าใช้จ่ายสูง **ISSU
(In-Service Software Upgrade)** ใช้ประโยชน์จาก Dual-RP/SSO ที่เรียนมาแล้วใน Step 801-804: อัปเกรด
RP ทีละตัว แล้วใช้กลไก Switchover เดียวกันสลับ Active Role ระหว่าง Software คนละเวอร์ชันชั่วคราว

### 805.2 eFSU (Enhanced Fast Software Upgrade) — วิวัฒนาการของ ISSU บน IOS-XE

Platform IOS-XE รุ่นใหม่ (รวม Catalyst 9000 Series) ใช้ **Unified Install CLI** (`install add /
activate / commit`) ซึ่งภายในเป็นกลไก **eFSU** — ต่างจาก ISSU รุ่นคลาสสิกบน ASR1000/ISR4000
(คำสั่ง `issu loadversion / runversion / acceptversion / commitversion` แยกกัน) ที่ **ลด Traffic
Outage Window ระหว่างขั้น Switchover จริงให้เหลือระดับ Sub-second ถึงไม่กี่วินาที** โดยอาศัย
SSO+NSF ชุดเดียวกันที่เรียนมาทั้ง Part นี้ — ความแตกต่างสำคัญคือ eFSU **Sync State ระหว่าง RP
เก่ากับ RP ใหม่ (คนละ Software Version)** ได้ล่วงหน้า ทำให้ตอน Switchover จริงไม่ต้องรอ Build
CEF/FIB ใหม่จากศูนย์เหมือน Switchover ปกติ

| ขั้นตอน (Unified Install CLI) | ความหมาย |
|---|---|
| `install add` | ดาวน์โหลด/แตก Package ของ Software เวอร์ชันใหม่เข้าไปใน Standby RP (Active ยังรันเวอร์ชันเดิม) |
| `install activate issu` | Standby RP **Boot ด้วย Software ใหม่** และเริ่ม Sync State กับ Active (เวอร์ชันเก่า) ผ่าน Compatibility Layer — เมื่อ Sync สมบูรณ์ ระบบจะทำ **Switchover อัตโนมัติ**: Active เดิม (Software เก่า) กลายเป็น Standby แล้ว Reload ด้วย Software ใหม่ตามไปด้วย |
| `install commit` | ยืนยันว่า Upgrade สำเร็จสมบูรณ์ ล้าง Rollback Point เดิมทิ้ง (ถ้าไม่ Commit ภายใน Rollback Timer ระบบจะ Auto-Rollback กลับ Software เดิมเอง) |

### 805.3 Worked Upgrade Procedure เต็มรูปแบบ

```
! ขั้นที่ 0 — ตรวจสอบสถานะ SSO ก่อนเริ่ม (ต้องเห็น STANDBY HOT เท่านั้น — ดู Step 802.3)
CORE-SW1# show redundancy states | include peer state

! ขั้นที่ 1 — Copy Image ใหม่เข้า Flash/Bootflash ทั้ง 2 Chassis (SVL Pair สั่งครั้งเดียวจาก Active)
CORE-SW1# copy tftp://10.10.99.50/cat9k_iosxe.17.09.05a.SPA.bin bootflash:
CORE-SW1# copy bootflash:cat9k_iosxe.17.09.05a.SPA.bin bootflash-standby:

! ขั้นที่ 2 — Add Package
CORE-SW1# install add file bootflash:cat9k_iosxe.17.09.05a.SPA.bin
install_add: START Mon Mar  3 09:30:11 UTC 2026
--- Starting installation state synchronization ---
Finished installation state synchronization
[1] Finished install add on all members

! ขั้นที่ 3 — Activate ด้วย ISSU (ขั้นนี้คือขั้นที่มี Switchover จริง)
CORE-SW1# install activate issu
install_activate: START Mon Mar  3 09:34:20 UTC 2026
This operation requires a switchover. Proceed? [y/n] y
--- Starting ISSU state sync ---
[Switch 2] Booting new software on Standby ... done
[Switch 2] Standby state = STANDBY HOT (new version)
--- Performing switchover ---
[Switch 1] Active role transferred to Switch 2
[Switch 1] Reloading with new version ...
SUCCESS: install_activate Mon Mar  3 09:41:55 UTC 2026

! ขั้นที่ 4 — ตรวจสอบว่าทั้งคู่รัน Version ใหม่แล้วและ SSO กลับมา HOT
CORE-SW1# show version | include Version
CORE-SW1# show redundancy states | include peer state

! ขั้นที่ 5 — Commit (สำคัญมาก — ไม่ Commit = Auto-Rollback ตาม Rollback Timer)
CORE-SW1# install commit
install_commit: START Mon Mar  3 09:50:02 UTC 2026
SUCCESS: install_commit Mon Mar  3 09:50:18 UTC 2026
```

> **จุดที่ต้องเข้าใจให้ชัด**: Traffic Forwarding (Data Plane) **ไม่หยุด**ตลอดขั้นที่ 3 เพราะกลไก
> เดียวกับ SSO+NSF ใน Step 804 ทำงานอยู่เบื้องหลัง — Line Card ไม่ถูก Reset ระหว่าง Switchover
> ของ Version, CEF/FIB ที่ Standby (Software ใหม่) Sync ไว้แล้วพร้อม Forward ทันทีที่รับ Active
> Role และ OSPF/BGP ใช้ NSF (Step 803) ซ่อนการ Restart Control Plane จาก Neighbor เหมือนเดิม

---

## Step 806 — ISSU Compatibility Matrix, ข้อจำกัด, และ Rollback Procedure

### 806.1 ไม่ใช่ทุก Upgrade จะเป็น ISSU-compatible

ISSU ทำงานได้เพราะ RP ทั้งสองตัว (Software คนละเวอร์ชัน) ต้อง**เข้าใจ Internal Data Structure
ของกันและกัน**พอที่จะ Sync State ข้าม Version ได้ (Compatibility Layer) — ถ้า Version ห่างกันเกินไป
(เช่น กระโดดข้าม Major Release หลายรุ่น หรือมีการเปลี่ยน Kernel/Infrastructure ภายในแบบ Breaking
Change) Compatibility Layer จะไม่รองรับ และระบบจะปฏิเสธ ISSU ทันที:

```
CORE-SW1# install activate issu
install_activate: START Mon Mar  3 10:02:00 UTC 2026
ERROR: ISSU is not supported for this upgrade path.
       Reason: Infra base version mismatch (source 17.03.04a, target 17.12.01)
       Recommendation: use 'install activate' (non-ISSU, requires reload) instead.
FAILED: install_activate Mon Mar  3 10:02:05 UTC 2026
```

### 806.2 ตารางแนวทางตรวจสอบ Compatibility ก่อนเริ่มจริง

| วิธีตรวจสอบ | รายละเอียด |
|---|---|
| **Cisco Software Checker / Release Notes** | ตรวจ ISSU Compatibility Matrix ที่ Cisco เผยแพร่ต่อ Platform — ระบุคู่ Version ที่รองรับ ISSU อย่างเป็นทางการ (ปกติ ISSU รองรับเฉพาะภายใน "Same Software Train/ไม่กี่ Maintenance Release ห่างกัน" ไม่รองรับการกระโดดข้าม Major Release) |
| **`show version`** | เทียบ Version ปัจจุบันของทั้ง 2 RP/Chassis ให้แน่ใจว่าตรงกันก่อนเริ่ม (ถ้าไม่ตรงกันอยู่แล้วตั้งแต่ต้น = มีปัญหาค้างจาก Upgrade ครั้งก่อน) |
| **`show install summary`** | ดูว่ามี Package ค้าง Uncommitted จากการ Upgrade ครั้งก่อนหรือไม่ (ต้อง Commit หรือ Rollback ให้เรียบร้อยก่อนเริ่ม Upgrade ใหม่เสมอ) |
| **ทดสอบใน Lab ที่จำลอง Version เดียวกันก่อนเสมอ** | เพราะ Compatibility Layer มีรายละเอียดเฉพาะ Platform/Version มาก การอ่าน Documentation อย่างเดียวไม่พอสำหรับ Production ที่ Critical |

### 806.3 กฎทั่วไปที่พอเชื่อได้ (Rule of Thumb)

| ระยะห่างของ Version | โดยทั่วไป ISSU รองรับหรือไม่ |
|---|---|
| Same Major.Minor ต่าง Maintenance Release (เช่น 17.09.04a → 17.09.05) | ✅ รองรับเกือบทุกกรณี |
| ต่าง Minor Release ใกล้กัน (เช่น 17.09.x → 17.12.x) | ⚠️ ขึ้นกับ Platform — ต้องเช็ค Matrix เสมอ ไม่ควรสมมติเอง |
| ต่าง Major Release หรือกระโดดหลาย Release (เช่น 16.x → 17.x, 17.03.x → 17.12.x) | ❌ ส่วนใหญ่ไม่รองรับ ต้องใช้ `install activate` แบบธรรมดา (Disruptive, ต้อง Reload) |

### 806.4 Rollback Procedure ถ้า ISSU ล้มเหลวกลางทาง

```
! กรณีที่ 1 — ยังไม่ Commit และเจอปัญหา (เช่น Neighbor ไม่ขึ้น FULL, Application บน Standby ผิดปกติ)
CORE-SW1# install abort
install_abort: START Mon Mar  3 09:55:00 UTC 2026
--- Reverting to previous running version ---
SUCCESS: install_abort Mon Mar  3 09:58:40 UTC 2026

! กรณีที่ 2 — ไม่ได้ทำอะไรเลยหลัง Activate และไม่ Commit ภายใน Rollback Timer (Default ~120 นาที)
! ระบบ Auto-Rollback กลับ Version เดิมให้เองพร้อม Log แจ้งเตือน
*Mar  3 11:56:20.001: %INSTALL-5-INSTALL_AUTO_ABORT_TIMER_PENDING: Auto abort timer will expire
                       in 5 minutes. Issue 'install commit' to confirm this installation.
*Mar  3 12:01:22.884: %INSTALL-5-INSTALL_AUTO_ABORT_INITIATED: Auto abort timer expired,
                       initiating rollback

! กรณีที่ 3 — Commit ไปแล้วแต่พบปัญหาทีหลัง (ต้องย้อนกลับ Base Version ก่อนหน้า)
CORE-SW1# install rollback to base
```

> **Best Practice**: อย่าปล่อยให้ Auto-Rollback Timer เป็นตัวตัดสินใจแทน — ควรมี Checklist
> Verification (Step 809) ที่ทำให้เสร็จและตัดสินใจ `install commit` เองภายในเวลาที่กำหนดของ
> Maintenance Window เสมอ

---

## Step 807 — StackWise Virtual + SSO Interaction: SSO ข้าม 2 Chassis

### 807.1 ทวนความแตกต่างที่ [Part 34](part-034-stackwise-vss-mec.md) วางไว้

[Part 34 Step 333](part-034-stackwise-vss-mec.md) สอนว่า StackWise Virtual (SVL) รวม CORE-SW1
และ CORE-SW2 (2 Chassis จริง แยกกันคนละกล่อง) ให้เป็น **1 Control Plane เดียว** — คำถามของ Step
นี้คือ: **SSO ที่เรียนมาทั้ง Part นี้ (ออกแบบมาสำหรับ RP 2 ตัวในกล่องเดียวกัน) ทำงานยังไงถ้า "RP"
ทั้งสองตัวนั้นอยู่คนละกล่องกันจริงๆ?**

### 807.2 คำตอบ: SVL "ยืม" Redundancy Infrastructure เดียวกับ SSO มาใช้ข้าม Chassis

เมื่อ SVL Form สำเร็จ Cisco IOS-XE จะตั้ง **Redundancy Mode เป็น SSO โดยอัตโนมัติ** ให้กับ SVL
Domain นั้น — Chassis ที่ได้ Role **Standby** ของ SVL จะทำหน้าที่เหมือน **Standby RP ของ SSO
ทุกประการ** เพียงแต่ Path การ Sync State (Checkpoint Facility ตาม Step 802.1) เดินทางผ่าน **SVL
Link** (สาย Fiber 10G/40G ระหว่าง 2 Chassis) แทนที่จะเป็น Backplane ภายในกล่องเดียว:

```
Dual-RP ในกล่องเดียว (Step 801-804)              StackWise Virtual (SVL) — Part 34 + Part 81

┌───────────────────────┐                        ┌──────────────┐      ┌──────────────┐
│  Chassis เดียว         │                        │  CORE-SW1     │      │  CORE-SW2     │
│  ┌────────┐ ┌────────┐ │                        │  (Switch 1)   │◄────►│  (Switch 2)   │
│  │Active  │◄►│Standby │ │  Checkpoint            │  Role: Active │ SVL  │  Role: Standby│
│  │  RP    │ │  RP    │ │  ผ่าน Backplone         │              │ Link │              │
│  └────────┘ └────────┘ │  ภายในกล่อง             └──────────────┘      └──────────────┘
└───────────────────────┘                          Checkpoint ตัวเดียวกัน ส่งผ่าน SVL Link
                                                    แทน Backplane — Redundancy Mode = SSO
                                                    เหมือนกันทุกประการจาก มุมของ IOS-XE Software
```

```
CORE# show redundancy states
       my state = 13 -ACTIVE
     peer state = 8  -STANDBY HOT
           Mode = Duplex
  Redundancy Mode (Operational) = sso
```

**Output นี้เหมือนกับ Step 802.3 เป๊ะ** — เพราะจากมุมของ Redundancy Facility ใน Software มันคือ
SSO ตัวเดียวกัน ไม่สนใจว่า Physical Path ของ Sync Link จะเป็น Backplane หรือ Fiber ข้ามห้อง

### 807.3 ผลลัพธ์เชิงปฏิบัติ: NSF ทำงานได้เหมือนกันทุกประการ

เพราะ Redundancy Facility เหมือนกัน **NSF (Step 803) ก็ทำงานเหมือนกันทุกประการ** — ถ้า CORE-SW1
(Active) ล่มหรือถูก Switchover, CORE-SW2 (Standby) รับ Active Role ทันทีด้วย CEF/FIB ที่ Sync
ไว้แล้วผ่าน SVL Link, และ OSPF/BGP Process ใหม่บน CORE-SW2 ส่ง Grace-LSA/GR Capability หา DIST-SW1–4
และ WAN-EDGE-1/2 เหมือนที่ Step 803-804 อธิบายไว้ทุกอย่าง — **Router-ID/Loopback0 (1.1.1.1) ของ
Logical Switch ไม่เปลี่ยนแม้ Physical Chassis ที่ Active จะเปลี่ยนไป** เพราะ SVL รวม Identity
เป็นหนึ่งเดียวตั้งแต่ Part 34 Step 333 แล้ว (สังเกตว่านี่คือประเด็นที่ต่างจาก HSRP/VRRP ใน Step
808 อย่างสิ้นเชิง — VIP ของ HSRP เป็น Address ที่แยกจาก Address จริงของแต่ละเครื่อง)

### 807.4 ตารางเปรียบเทียบ: Dual-RP ในกล่องเดียว vs SVL Pair

| ประเด็น | Dual-RP ในกล่องเดียว (Catalyst 9600/ASR1000) | SVL Pair (CORE-SW1+CORE-SW2, Catalyst 9500) |
|---|---|---|
| Physical Location | 2 Card ใน Chassis เดียวกัน | 2 Chassis แยกกันจริง (คนละห้อง/คนละชั้นได้) |
| Sync Path | Internal Backplane | SVL Link (Fiber 10G/40G ระหว่าง 2 Chassis) |
| Redundancy Mode | SSO | SSO (เหมือนกัน — ตั้งอัตโนมัติหลัง SVL Form) |
| ความล้มเหลวที่ป้องกันได้ | RP/Supervisor Card ตัวเดียวเสีย | **ทั้ง Chassis** เสีย (Power/Fan/Line Card ทั้งกล่อง) เพราะ Standby คือกล่องแยกทั้งกล่อง |
| ผลกระทบถ้า Sync Link ขาดทั้งหมด | Standby ถอยเป็น RPR/Cold ชั่วคราว (Rare — Backplane แทบไม่เคยขาดทั้งหมด) | เสี่ยง **Split-Brain** สูงกว่า (2 Chassis คนละที่กันจริง สาย Fiber มีโอกาสขาดมากกว่า Backplane) — ต้องมี **Dual-Active Detection (DAD)** ตาม [Part 34 Step 337](part-034-stackwise-vss-mec.md) เป็นตัวป้องกันเพิ่ม ซึ่ง Dual-RP ในกล่องเดียวไม่ต้องมี |
| MEC/EtherChannel ข้าม Physical Boundary | ไม่เกี่ยวข้อง (เป็นกล่องเดียวอยู่แล้ว) | ทำได้เพราะ SVL รวม LACP System ID เป็นเดียวกัน ([Part 34 Step 335](part-034-stackwise-vss-mec.md)) |

> **สรุปใจความ**: SVL ไม่ได้ "คิดค้น Redundancy กลไกใหม่" แต่ **นำ SSO/NSF ที่มีอยู่แล้วมาขยาย
> Physical Boundary ให้ครอบคลุม 2 กล่อง** — นี่คือเหตุผลที่ Part 34 Step 333 บอกว่าหลัง SVL Form
> "OSPF Process เดียว, Router-ID เดียว" เพราะมันคือ SSO Pair ตัวเดียวกันทุกกลไก เพียงแค่ Sync Link
> เดินสายยาวกว่าปกติเท่านั้นเอง

---

## Step 808 — HA สำหรับ WAN Edge Router: Chassis-internal vs Router-external

### 808.1 ปัญหา: ISR4451/CSR1000v ไม่มี Slot RP สำรอง

WAN-EDGE-1 และ WAN-EDGE-2 ตาม [00-ip-address-plan.md](00-ip-address-plan.md) ใช้ Platform
**ISR4451** (Fixed-Configuration Hardware, RP เดียวในตัว ไม่มี Slot สำหรับ RP สำรอง) หรือ
**CSR1000v** (Virtual Router — เป็น Software ที่รันบน VM, ไม่มีแนวคิด "RP สำรองในเครื่องเดียวกัน"
เลยด้วยซ้ำเพราะไม่มี Hardware Chassis จริง) — ทั้งสอง Platform นี้ **ไม่สามารถทำ SSO/NSF ระดับ
Chassis-internal ได้เลย** ไม่ว่าจะ Config อย่างไรก็ตาม (ไม่มี Standby RP ให้ Sync State ไปด้วย)

### 808.2 ทางออก: HA ระดับ "Router ทั้งเครื่อง" ด้วย HSRP/VRRP

เมื่อไม่มี Redundancy ภายในเครื่องเดียว คำตอบคือย้อนกลับไปที่แนวคิดที่เรียนไว้แล้วใน [Part 18](part-018-fhrp.md)
และเจาะลึกใน [Part 35](part-035-advanced-fhrp-campus-ha.md): ใช้ **Router ทั้งเครื่องสองตัวแยกกัน
จริง** (WAN-EDGE-1 กับ WAN-EDGE-2) แล้วให้ **HSRP/VRRP** เป็นผู้จัดการว่าใครเป็น Active Gateway
ให้กับ Segment ที่มี Shared Subnet:

```
! ตัวอย่างแนวคิด — HSRP บน LAN-facing Interface ของ WAN Edge Router คู่หนึ่งที่ใช้ Shared Subnet
WAN-EDGE-1(config)# interface GigabitEthernet0/0/1
WAN-EDGE-1(config-if)# ip address 10.20.1.2 255.255.255.0
WAN-EDGE-1(config-if)# standby 1 ip 10.20.1.1
WAN-EDGE-1(config-if)# standby 1 priority 150
WAN-EDGE-1(config-if)# standby 1 preempt

WAN-EDGE-2(config)# interface GigabitEthernet0/0/1
WAN-EDGE-2(config-if)# ip address 10.20.1.3 255.255.255.0
WAN-EDGE-2(config-if)# standby 1 ip 10.20.1.1
WAN-EDGE-2(config-if)# standby 1 priority 100
```

> **หมายเหตุสำคัญเกี่ยวกับ Topology จริงของ Lab หลักสูตรนี้**: ตาม [00-ip-address-plan.md](00-ip-address-plan.md)
> WAN-EDGE-1 ↔ CORE-SW1 และ WAN-EDGE-2 ↔ CORE-SW2 เป็น **Routed Point-to-Point Link (/30)** ไม่ใช่
> Shared VLAN/Subnet — จึงไม่มีจุดที่ต้องใช้ HSRP VIP จริงระหว่าง CORE กับ WAN Edge ในโครงสร้างของ
> เรา (HSRP ใช้ตรงจุดที่มี Shared Subnet ให้ Host พึ่งพา Gateway เดียวเท่านั้น) — ฝั่ง **WAN
> (WAN-EDGE↔ISP-RTR)** ของ Lab นี้ใช้ **BGP Dual-Homing** ([Part 74](part-074-advanced-bgp-wan-edge.md))
> เป็นกลไก HA แทน ซึ่งเป็น**แนวคิดเดียวกัน**กับ HSRP ในมุม Design Philosophy (ดู 808.3) แม้จะเป็น
> Protocol คนละตัวก็ตาม — ตัวอย่าง Config ด้านบนคือรูปแบบทั่วไปที่ใช้เมื่อ WAN Edge คู่หนึ่งต้อง
> Front ด้วย Shared LAN Segment จริง (เช่น สาขาที่มี Switch คั่นระหว่าง Router 2 ตัวกับ LAN)

### 808.3 สองปรัชญา HA ที่ต่างกันโดยพื้นฐาน

| มิติ | Chassis-internal HA (NSF/SSO — Step 801-807) | Router-external HA (HSRP/VRRP/BGP Dual-Homing — Step 808) |
|---|---|---|
| อะไรคือ "หน่วยที่ Redundant" | Component ภายในกล่องเดียวกัน (RP/Supervisor) | **อุปกรณ์ทั้งเครื่อง** 2 ตัวแยกกันจริง |
| Neighbor รอบข้างรู้ตัวหรือไม่ว่ามี Failover | **ไม่รู้เลย** (NSF ซ่อนสมบูรณ์ — Adjacency เดิม, Router-ID เดิม) | **รู้เสมอ** — Router ตัวเดิมหายไปจริง (Adjacency กับตัวนั้น Down จริง), มีแค่ Traffic/Gateway ที่เปลี่ยนไปใช้ตัวสำรองแทน |
| Detection Time | เกือบ 0 (Hardware ตรวจจับ Keepalive ภายในกล่อง เร็วมาก) | ขึ้นกับ Timer ของ Protocol (HSRP Hello/Hold, BGP Hold Timer หรือเร็วขึ้นด้วย BFD) — เป็น Sub-second ได้ถ้า Tune ดี แต่ไม่มีทาง "เร็วเท่า" การ Sync ภายในกล่องเดียว |
| ค่าใช้จ่าย Hardware | สูง (ต้องมี Modular Chassis + RP คู่ ราคาแพงกว่ามาก) | ต่ำกว่า (Router ราคาปกติ 2 ตัว หรือแม้แต่ Virtual Router เช่น CSR1000v) |
| ป้องกัน Failure ระดับไหน | RP/Supervisor เสีย (แต่ Chassis ทั้งกล่องยังอยู่ — Power/Fan ปกติ) | **อุปกรณ์ทั้งเครื่องหายไปจริง** (Power ดับทั้งตู้, Hardware Fail ทั้งกล่อง, แม้แต่ Site ทั้ง Site ถ้า Router อยู่คนละที่กัน) |
| ตัวอย่าง Platform ในหลักสูตรนี้ | CORE-SW1/CORE-SW2 ผ่าน SVL (Step 807) | WAN-EDGE-1/WAN-EDGE-2 ผ่าน BGP Dual-Homing (Part 74) หรือ HSRP ตรง Access/Distribution (Part 18/35) |

> **ข้อคิดสำคัญสำหรับข้อสอบ CCIE**: คำถามที่ชอบถามคือ "ทำไมไม่ใช้ HSRP แทน NSF/SSO ไปเลยทุกที่?"
> คำตอบคือ **คนละ Failure Domain กันโดยสิ้นเชิง** — NSF/SSO ป้องกัน Failure ที่**เล็กกว่า**และ
> **เร็วกว่า** (RP เสียแต่กล่องยังอยู่) ในขณะที่ HSRP/VRRP ป้องกัน Failure ที่**ใหญ่กว่า**(กล่อง
> หายไปทั้งกล่อง) — Design ที่ดีใช้ทั้งสองแบบซ้อนกันเป็นชั้น (Defense-in-Depth) เหมือนที่ [Part 35
> Step 347](part-035-advanced-fhrp-campus-ha.md) สรุปไว้ ไม่ใช่เลือกอย่างใดอย่างหนึ่ง

---

## Step 809 — Testing/Validating HA: Planned Failover Test และ Maintenance Runbook

### 809.1 ทำไมต้องทดสอบก่อนไว้ใจใน Production

NSF/SSO/ISSU เป็น Feature ที่**ซับซ้อนและมีเงื่อนไขจำนวนมาก** (Version Compatibility, Neighbor
Capability, Config ที่ต้องตรงกันทั้งคู่) — การ Config ไว้แล้วไม่เคยทดสอบจริงเสี่ยงต่อการพบว่า
"ใช้งานไม่ได้จริง" ตอนที่เกิด Failure โดยไม่ได้ตั้งใจ (ซึ่งเป็นเวลาที่แย่ที่สุดที่จะมารู้ตัว) —
หลักการคือ **ทดสอบ Failover แบบ Controlled ใน Maintenance Window ก่อนเสมอ**

### 809.2 `redundancy force-switchover` — เครื่องมือทดสอบหลัก

```
CORE-SW1# redundancy force-switchover
This will reload the active unit and force switchover to standby[confirm]
```

คำสั่งนี้จำลอง RP Switchover แบบ **Controlled/Planned** (ต่างจาก RP Crash แบบ Unplanned ใน Step
804.2) — Active RP/Chassis ปัจจุบันจะ Reload และมอบ Active Role ให้ Standby ทันที เป็นวิธีทดสอบ
SSO+NSF ที่ปลอดภัยที่สุดเพราะเป็นการทดสอบแบบ **Signaled** (Grace-LSA ส่งออกไปก่อน Switchover จริง)
ซึ่งเป็น Best-case Scenario — ควรทดสอบสิ่งนี้ให้ผ่านก่อนเสมอ ก่อนจะไว้ใจว่า Unplanned Failure
จะได้ผลลัพธ์เดียวกัน

### 809.3 Maintenance Runbook — Checklist ก่อน/ระหว่าง/หลัง ทดสอบ

| ระยะ | รายการที่ต้องเช็ค |
|---|---|
| **ก่อนทดสอบ (Pre-check)** | 1) `show redundancy states` — Peer State ต้องเป็น `STANDBY HOT` เท่านั้น 2) ยืนยัน `nsf cisco/ietf`, `bgp graceful-restart` Config ไว้ครบทุก Routing Protocol ที่ใช้จริง 3) ยืนยันกับทีมว่าอยู่ใน Maintenance Window ที่ได้รับอนุมัติ 4) เริ่ม Continuous Ping/Traceroute จาก Monitoring Host ไปยังปลายทางสำคัญ (End-to-end ผ่าน Prefix ที่พึ่งพา NSF) 5) เปิด `terminal monitor` หรือเตรียม Syslog Server ดู Log Real-time |
| **ระหว่างทดสอบ (Execute)** | 1) สั่ง `redundancy force-switchover` 2) สังเกต Log `%RF-5-RF_TERMINAL_STATE`, `%OSPF-5-NSF_NBR_RESYNC` หรือเทียบเท่าของ Protocol อื่น 3) จับเวลาตั้งแต่สั่งคำสั่งจนถึง Adjacency กลับ FULL/Established ทั้งหมด |
| **หลังทดสอบ (Post-check)** | 1) `show redundancy states` — Role สลับแล้ว, Peer State `STANDBY HOT` อีกครั้ง (Role กลับกัน) 2) `show ip ospf neighbor` / `show ip bgp summary` — ทุก Neighbor ต้อง `FULL`/`Established` ครบ 3) ตรวจผล Ping/Traceroute — **Packet Loss ต้องเป็น 0%** (หรือใกล้ 0% ที่สุด) 4) `show interfaces counters errors` — ต้องไม่มี Error/Drop เพิ่มขึ้นผิดปกติช่วงทดสอบ 5) บันทึกผลลัพธ์ทั้งหมดไว้เป็นหลักฐาน (Timestamp, Loss %, Duration) เพื่อใช้อ้างอิงครั้งถัดไป |
| **ความถี่ที่แนะนำ** | ทดสอบซ้ำหลังทุกครั้งที่มีการเปลี่ยน Software Version สำคัญ, เปลี่ยน Topology (เพิ่ม Neighbor ใหม่), หรืออย่างน้อยปีละครั้งตาม Change Management Policy ขององค์กร |

### 809.4 ข้อผิดพลาดที่พบบ่อยเวลาทดสอบ

- **ทดสอบตอน Peer State ยังไม่ HOT** — ผลลัพธ์คือ Switchover กลายเป็นเหมือน RPR (Cold) โดยไม่รู้ตัว
  เพราะ Standby ยังไม่ Sync สมบูรณ์ ทำให้สรุปผิดว่า "SSO ใช้ไม่ได้จริง" ทั้งที่ปัญหาคือ Sync ไม่เสร็จ
  ต่างหาก
- **ไม่ได้ Config NSF บนทุก Routing Protocol ที่ใช้จริง** — เช่น Config `nsf cisco` ไว้แต่ EIGRP
  หรือ BGP ไม่ได้ Config `nsf`/`bgp graceful-restart` คู่กัน ทำให้ Protocol นั้นๆ Drop Adjacency
  ตามปกติแม้ OSPF จะรอดก็ตาม
- **ลืมตรวจ Neighbor Capability** — Neighbor รุ่นเก่าหรือ Vendor อื่นที่ไม่รองรับ Extension เดียวกัน
  (ดู Step 803.4) — ทดสอบใน Lab ที่ Neighbor เป็น IOS/IOS-XE รุ่นใหม่ทั้งหมดอาจไม่เจอปัญหานี้ แต่
  Production จริงอาจมี Neighbor ที่ต่างออกไป

---

## Step 810 — Lab เต็มรูปแบบ: NSF+SSO บน CORE-SW1/CORE-SW2, ISSU Walkthrough, Live Failover Test

### 810.1 Baseline Topology และสถานะก่อนเริ่ม

CORE-SW1/CORE-SW2 เป็น **StackWise Virtual Pair** ตาม [Part 34 Step 340](part-034-stackwise-vss-mec.md)
อยู่แล้ว (Domain 100, SVL Link 2x10G, DAD Link) — มองจาก DIST-SW1–4 และ WAN-EDGE-1/2 เห็นเป็น
**"CORE" ตัวเดียว** ด้วย Loopback0/Router-ID = 1.1.1.1 (Chassis ที่เป็น Active ตั้งแต่ SVL Form
ครั้งแรก) รัน **OSPF Process 1** กับ DIST-SW1–4 และ **iBGP AS 65001** กับ WAN-EDGE-1/2 (Route
Reflector Client ตาม [Part 74 Step 740](part-074-advanced-bgp-wan-edge.md))

### 810.2 Config NSF+SSO เต็มรูปแบบบน CORE (SVL Pair)

```
! ตรวจสอบว่า Redundancy Mode เป็น SSO อัตโนมัติแล้วหลัง SVL Form (Step 807.2)
CORE# show redundancy states | include Redundancy Mode
  Redundancy Mode (Operational) = sso

! เปิด NSF บน OSPF Process
CORE(config)# router ospf 1
CORE(config-router)# nsf cisco
CORE(config-router)# end

! เปิด Graceful Restart บน BGP Process (ค่านี้มักถูก Config ไว้แล้วตาม Part 74 Step 740 — ตรวจซ้ำ)
CORE(config)# router bgp 65001
CORE(config-router)# bgp graceful-restart
CORE(config-router)# bgp graceful-restart restart-time 120
CORE(config-router)# bgp graceful-restart stalepath-time 300
CORE(config-router)# end
CORE# write memory
```

### 810.3 Baseline Verification (ก่อนทำ ISSU หรือ Failover Test ใดๆ)

```
CORE# show switch virtual
Switch Mode : Stackwise Virtual
Switch Virtual Link (SVL) Domain Number : 100
                     Switch 1                Switch 2
--------------------------------------------------------------
Role                 Active                   Standby

CORE# show redundancy states
       my state = 13 -ACTIVE
     peer state = 8  -STANDBY HOT
  Redundancy Mode (Operational) = sso

CORE# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/BDR        00:00:37    10.255.10.2     Po19
1.1.1.12        1     FULL/BDR        00:00:34    10.255.20.2     Po29
1.1.1.13        1     FULL/BDR        00:00:39    10.255.30.2     Po39
1.1.1.14        1     FULL/BDR        00:00:38    10.255.40.2     Po49

CORE# show ip bgp summary | begin Neighbor
Neighbor        V    AS  MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
1.1.1.21        4 65001    88213    88190    41205    0    0 3w2d          214
1.1.1.22        4 65001    88201    88188    41205    0    0 3w2d          214
1.1.1.11        4 65001    12904    12899    41205    0    0 2w6d           88
1.1.1.12        4 65001    12888    12879    41205    0    0 2w6d           88
```

### 810.4 ISSU Walkthrough บน SVL Pair (ทวนจาก Step 805 ในบริบทของ Lab นี้)

```
CORE# install add file bootflash:cat9k_iosxe.17.09.05a.SPA.bin
[1] Finished install add on all members

CORE# install activate issu
This operation requires a switchover. Proceed? [y/n] y
[Switch 2] Booting new software on Standby ... done
[Switch 2] Standby state = STANDBY HOT (new version)
[Switch 1] Active role transferred to Switch 2
[Switch 1] Reloading with new version ...
SUCCESS: install_activate

CORE# show version | include Version
CORE# show redundancy states | include peer state
     peer state = 8  -STANDBY HOT

CORE# install commit
SUCCESS: install_commit
```

ระหว่างขั้น `install activate issu` **ทั้ง OSPF Neighbor (DIST-SW1-4) และ BGP Neighbor
(WAN-EDGE-1/2) ยังคง FULL/Established ตลอดเวลา** เพราะกลไก NSF (Step 803) ทำงานเบื้องหลังเหมือน
RP Switchover ปกติทุกประการ — Switch 2 กลายเป็น Active ใหม่ (Software ใหม่) ในขณะที่ Switch 1
Reload ด้วย Software ใหม่ตามไปด้วย โดย MEC (Po19/29/39/49) ที่ DIST-SW เห็นไม่เคย Down เลยสักครั้ง

### 810.5 Live SSO Failover Test พร้อม Timestamp พิสูจน์ NSF

```
! เริ่ม Continuous Ping จาก Management Station ไปยัง DIST-SW1 Loopback0 คู่กับ WAN-EDGE-1 Loopback0
mgmt-host$ ping -i 0.2 1.1.1.11 &
mgmt-host$ ping -i 0.2 1.1.1.21 &

! สั่ง Controlled Switchover
CORE# redundancy force-switchover
This will reload the active unit and force switchover to standby[confirm]

! Log ที่ปรากฏบน Switch ที่กลายเป็น Active ใหม่
*Mar  3 14:02:11.402: %RF-5-RF_TERMINAL_STATE: Terminal state reached for (SSO)
*Mar  3 14:02:11.588: %OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.11 on Po19 from LOADING to FULL, Loading Done
*Mar  3 14:02:11.591: %OSPF-5-NSF_NBR_RESYNC: Process 1, Nbr 1.1.1.11 on Po19: NSF resynchronized
*Mar  3 14:02:12.104: %BGP-5-NBR_RESTART: neighbor 1.1.1.21 graceful restart, forwarding state retained

! ทันทีหลัง Switchover
CORE# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.11        1     FULL/BDR        00:00:33    10.255.10.2     Po19
1.1.1.12        1     FULL/BDR        00:00:36    10.255.20.2     Po29
1.1.1.13        1     FULL/BDR        00:00:31    10.255.30.2     Po39
1.1.1.14        1     FULL/BDR        00:00:35    10.255.40.2     Po49

CORE# show ip bgp summary | begin Neighbor
Neighbor        V    AS  MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  State/PfxRcd
1.1.1.21        4 65001    88214    88191    41206    0    0 3w2d          214
1.1.1.22        4 65001    88202    88189    41206    0    0 3w2d          214
```

```
mgmt-host$ tail -f ping-results.log
--- ผลสรุป Ping ระหว่างช่วง Switchover (14:02:00 - 14:02:30) ---
1.1.1.11 : 150 packets transmitted, 150 received, 0% packet loss
1.1.1.21 : 150 packets transmitted, 150 received, 0% packet loss
```

### 810.6 บทสรุปของผลการทดสอบ

| หลักฐาน | ผลลัพธ์ |
|---|---|
| `show ip ospf neighbor` ก่อน/หลัง Switchover | **FULL ทุกตัวเหมือนกันทุกประการ** ไม่มี LOADING/DOWN ปรากฏเลย |
| `show ip bgp summary` ก่อน/หลัง Switchover | **Established ต่อเนื่อง** (Up/Down Timer ไม่ Reset — พิสูจน์ว่า Session ไม่ได้ขาดจริง) |
| Log `%OSPF-5-NSF_NBR_RESYNC`, `%BGP-5-NBR_RESTART...forwarding state retained` | หลักฐานตรงจาก IOS-XE ว่า NSF/Graceful Restart ทำงานจริง ไม่ใช่แค่ "ดูเหมือนไม่กระทบ" |
| Ping ต่อเนื่องระหว่างทดสอบ | **0% Packet Loss** — Data Plane ไม่หยุดแม้แต่ Packet เดียว |
| `show switch virtual` หลังทดสอบ | Role Active/Standby สลับกัน (Switch 2 เป็น Active ใหม่) — Logical Switch Identity (Router-ID) ไม่เปลี่ยน |

นี่คือข้อพิสูจน์ครบวงจรว่า **NSF (Step 803) ซ่อนการ Restart Control Plane จาก Neighbor ได้จริง**,
**SSO (Step 802) ทำให้ Data Plane ไม่หยุดจริง**, และ **StackWise Virtual (Step 807) ทำให้กลไก
ทั้งหมดนี้ทำงานข้าม 2 Chassis จริงได้เหมือนเป็น Dual-RP ในกล่องเดียว** — สิ่งที่ Part 26 และ Part
35 Preview ไว้แบบ Concept-only ตอนนี้พิสูจน์ได้ครบด้วย Log และผลทดสอบจริงแล้ว

---

## แบบฝึกหัดทวนความเข้าใจ Part 81

1. RPR และ SSO ต่างกันอย่างไรในเรื่อง CEF/FIB Table และ Line Card ระหว่าง Switchover?
2. ทำไม SSO Sync CEF/FIB Table ได้ แต่ Sync Routing Protocol Adjacency (Neighbor Table) ไม่ได้?
   NSF เข้ามาแก้ปัญหานี้อย่างไร?
3. ในการทดสอบ `redundancy force-switchover` ถ้า `show redundancy states` แสดง Peer State เป็น
   `STANDBY COLD` ก่อนทดสอบ จะเกิดอะไรขึ้นกับผลการทดสอบ NSF/SSO?
4. ISSU ต่างจากการ Upgrade Software แบบ `install add/activate` ธรรมดา (ไม่ใช้ ISSU) อย่างไร และ
   เพราะเหตุใดการกระโดดข้าม Major Release มักไม่รองรับ ISSU?
5. เพราะเหตุใด WAN-EDGE-1/WAN-EDGE-2 (ISR4451/CSR1000v) จึงใช้ HSRP/VRRP หรือ BGP Dual-Homing
   เป็นกลไก HA หลัก แทนที่จะใช้ NSF/SSO แบบที่ CORE-SW1/CORE-SW2 ใช้?

**เฉลย:**

1. RPR: Standby RP ไม่มี CEF/FIB Table พร้อมล่วงหน้า ต้องสร้างใหม่จาก RIB หลัง Switchover และ
   Line Card ถูก Reset/Re-init ระหว่างนั้น (Data Plane หยุด) — SSO: CEF/FIB Table ถูก Sync ไปยัง
   Standby ตลอดเวลาแบบ Real-time และ Line Card ไม่ถูก Reset เลย ทำให้ Forward Packet ต่อเนื่องได้
   ทันทีที่ Standby รับ Active Role
2. CEF/FIB เป็น Flat Table ผลลัพธ์สุดท้ายที่มีขนาดคงที่ Sync แบบ Incremental ได้ง่าย ส่วน Routing
   Protocol Adjacency เป็น State Machine ที่ซับซ้อน (Timer, Neighbor FSM, LSDB/Topology Table ที่
   ต้องเชื่อมโยงกับ Neighbor จริง) การ Sync ทั้งหมดแบบ Real-time ทำได้ยากมาก — NSF แก้ปัญหานี้ด้วย
   การไม่ Sync เลยแต่ใช้ Grace-LSA/GR Capability บอก Neighbor ให้รอ (Helper Mode) จนกว่า Routing
   Protocol Process ใหม่จะสร้าง Adjacency กลับมาสมบูรณ์เหมือนเดิม
3. Switchover จะกลายเป็นเหมือน RPR (Cold) โดยไม่รู้ตัว เพราะ Standby ยังไม่มี CEF/FIB/State ที่
   Sync สมบูรณ์ให้ใช้ Forward ทันที — Traffic จะหยุดชั่วขณะระหว่าง Switchover แม้จะ Config NSF ไว้
   ก็ตาม เพราะ NSF พึ่งพา Data Plane ที่ต่อเนื่อง (SSO) เป็นพื้นฐานอยู่ก่อนแล้ว
4. ISSU/eFSU ใช้กลไก SSO+NSF สลับ Active Role ระหว่าง RP/Chassis คนละ Software Version ทำให้
   Traffic ไม่หยุด (Outage Window เหลือ Sub-second ถึงไม่กี่วินาที) ในขณะที่ Upgrade ธรรมดาต้อง
   Reload ทั้งเครื่อง (หยุด Forward Packet นาทีถึงหลายนาที) — การกระโดดข้าม Major Release มักมี
   การเปลี่ยน Internal Data Structure/Infrastructure แบบ Breaking Change ทำให้ Compatibility
   Layer ที่ ISSU ต้องใช้ Sync State ข้าม Version ไม่สามารถแปลความหมายของ State ระหว่าง 2 Version
   ที่ต่างกันมากได้ ระบบจึงปฏิเสธ ISSU และบังคับให้ใช้วิธี Disruptive แทน
5. ISR4451 เป็น Fixed-Configuration Hardware ไม่มี Slot RP สำรอง และ CSR1000v เป็น Virtual Router
   ที่ไม่มีแนวคิด Hardware RP สำรองเลย ทั้งสอง Platform จึงไม่สามารถทำ SSO/NSF ระดับ Chassis-
   internal ได้ไม่ว่าจะ Config อย่างไร ทางออกคือใช้ Router ทั้งเครื่องสองตัวแยกกันจริงและให้
   HSRP/VRRP (จุดที่มี Shared Subnet) หรือ BGP Dual-Homing (จุดที่เป็น Routed P2P อย่างในโครงสร้าง
   ของหลักสูตรนี้) เป็นผู้จัดการ Failover ระดับ "อุปกรณ์ทั้งเครื่อง" แทน

---

## สรุป Part 81

Part นี้ปิดหนี้ที่ค้างมาตั้งแต่ [Part 26 Step 256](part-026-advanced-ospf.md) และ [Part 35 Step
345-346](part-035-advanced-fhrp-campus-ha.md) ด้วยการเจาะลึก High Availability ระดับ Chassis
อย่างครบวงจร: **RPR vs SSO** พร้อมตารางเปรียบเทียบว่าอะไรถูกเก็บ/ไม่ถูกเก็บระหว่าง Switchover,
**SSO Mechanics** ที่ Sync CEF/FIB/Interface/ARP แต่ไม่ Sync Routing Protocol Adjacency (และ
เหตุผลเชิงเทคนิคว่าทำไม), **NSF/Graceful Restart** ที่ใช้ Grace-LSA/RS-bit/GR Capability ซ่อนการ
Restart Control Plane จาก Neighbor ได้สำเร็จทั้ง OSPF/EIGRP/BGP, **Timeline ละเอียดทุกวินาที**ของ
SSO+NSF Failover จริง, **ISSU/eFSU** สำหรับ Upgrade Software แบบไม่มี Outage พร้อม Compatibility
Matrix และ Rollback Procedure, การเชื่อมโยง **StackWise Virtual (Part 34)** เข้ากับ SSO ว่าเป็น
Redundancy Facility เดียวกันที่ขยาย Physical Boundary ข้าม 2 Chassis, ความแตกต่างเชิงปรัชญาระหว่าง
**Chassis-internal HA** กับ **Router-external HA (HSRP/VRRP/BGP Dual-Homing)** สำหรับ Platform
อย่าง WAN Edge ที่ไม่มี Dual-RP, แนวทางการทดสอบ HA ก่อนไว้ใจใน Production ด้วย `redundancy
force-switchover` และ Maintenance Runbook, และปิดท้ายด้วย **Lab เต็มรูปแบบ**ที่พิสูจน์ด้วย Log
และผลทดสอบจริงว่า OSPF/BGP Neighbor ไม่หลุดแม้แต่ครั้งเดียวระหว่าง RP Switchover และ ISSU Upgrade
บน CORE-SW1/CORE-SW2 (SVL Pair)

Campus/Enterprise Network ของหลักสูตรนี้ตอนนี้มี HA ที่ครบทุกระดับ ตั้งแต่ Physical Chassis
(StackWise Virtual), RP/Supervisor ภายในเครื่อง (NSF/SSO), Software Upgrade (ISSU/eFSU), จนถึง
Router ทั้งเครื่อง (HSRP/VRRP/BGP Dual-Homing) — Part ถัดไปจะเปลี่ยนโฟกัสจาก "ความพร้อมใช้งาน"
ไปสู่ "คุณภาพของ Traffic ที่วิ่งอยู่": **End-to-End Enterprise QoS Design** ที่รวม Classification,
Marking, Queuing, และ Policy ตั้งแต่ Access Layer จนถึง WAN Edge เข้าเป็นแบบแผนเดียวที่สอดคล้องกัน
ทั้งองค์กร

**ไปต่อ:** [Part 82 — End-to-End Enterprise QoS Design →](part-082-end-to-end-enterprise-qos-design.md)
